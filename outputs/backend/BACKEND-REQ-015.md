---
doc_type: backend_spec
req_id: REQ-015
version: 0.2.0
status: draft
generated_from: requirement.md@0.5.0
data_contract_ref: data-contract.md@0.1.0
generated_at: 2026-05-18
owner: ""
---

# 后端开发说明：BCA 月度人力数据提交

> ⚠️ 重要约定：
> - 数据模型与 API 字段定义完全引用 `data-contract.md`，本文档不重复定义。
> - 业务规则、状态转换、事务边界、幂等性、审计要求在本文档详述。
> - 代码注释中必须标注覆盖的 AC ID。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-015 @ v0.5.0 |
| 数据契约 | data-contract.md @ v0.1.0 |
| 覆盖 Story | US-001、US-002、US-003、US-004、US-005 |
| 覆盖 AC | AC-015-001 ~ AC-015-011 |

---

## 1. 功能概述

新增 BCA 月度人力数据提交能力，包含：

1. **提交任务管理**：创建/查询 Submission Task，拆分批次，驱动批次异步并发发送
2. **批次发送引擎**：异步队列并发调用 BCA 外部接口，状态实时回写；Task 直接落地为终态（Success / Partial Failed / Failed）
3. **批次级重试**：支持对失败批次整批手动重试（**不支持单条记录重试**，BCA 接口限制）
4. **自动提交调度**：每月定时为已开启的项目自动创建提交任务
5. **数据快照**：创建任务时对当月数据做快照，保证提交内容不随源数据变更

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

- 语言：（与项目现有后端语言一致）
- 框架：（与项目现有框架一致）
- 数据库：PostgreSQL
- 缓存：Redis
- 消息队列：（项目现有 MQ，如 RabbitMQ / Kafka）
- 调度：Cron Job（项目现有调度框架）

### 2.2 本需求新增依赖

| 库 / 服务 | 用途 | 评估 |
|----------|------|------|
| BCA Manpower Reporting API（外部） | 接收人力数据上报 | 需向 BCA 申请 API Key，确认接口文档（OQ-001~004） |

---

## 3. 业务逻辑

### 3.1 创建提交任务（覆盖 AC-015-001、AC-015-002、AC-015-007）

```
1. 权限校验：当前用户对 project_id 有 Site Admin 或 Platform Admin 权限，否则 403
2. 防重复校验：
   SELECT count(*) FROM submission_tasks
   WHERE project_id = ? AND submission_month = ?
   若 count > 0 → 返回业务错误 BCA_SUBMISSION_DUPLICATE（422）  // AC-015-002
   （任何状态的已有任务均不可重复发起）
3. 数据存在性校验：
   SELECT count(*) FROM attendance_records
   WHERE project_id = ? AND attendance_month = ?
   若 count = 0 → 返回 BCA_NO_ATTENDANCE_DATA（422）  // AC-015-007
4. 事务开始：
   a. INSERT submission_tasks（status=pending, trigger_type=manual）
   b. 查询当月人员通行数据，按 batch_size（默认 100，可配置）拆分
   c. 对每条通行记录做快照，INSERT batch_records（含全量字段快照）  // AC-015-010
   d. INSERT batches（status=pending，batch_index 从 1 开始递增）
5. 事务提交
6. 发布异步任务消息：触发批次发送队列
7. 返回 submission_task（含 task_id、status=pending、batch_count）
```

### 3.2 批次发送引擎（覆盖 AC-015-003、AC-015-004）

```
// 由异步队列 Worker 执行，每次处理一个批次
1. 从队列取出 batch_id
2. 加行级锁：SELECT FOR UPDATE batch WHERE id=batch_id
3. 校验批次状态为 pending，否则幂等跳过
4. UPDATE batch SET status='sending', sent_at=NOW()
5. 读取批次内所有 batch_records（快照数据）
6. 调用 BCA 接口：
   POST {BCA_API_BASE}/manpower-attendance
   Body: { records: [...] }  // 字段见 data-contract
   超时：30s
7. 解析 BCA 响应：
   若整批成功：
     UPDATE batch SET status='success'
     UPDATE batch_records SET status='success'
   若 BCA 返回逐条结果（部分成功/失败）：
     按记录逐条更新 batch_records.status
     若有任意 record 失败 → batch.status='failed'，记录 error_code 和 error_message
     若全部成功 → batch.status='success'
   若接口超时或网络错误：
     UPDATE batch SET status='failed', error_message='接口超时'
     UPDATE batch_records SET status='failed', error_message='接口超时'
8. 发布事件：BATCH_COMPLETED(batch_id, status)
9. 触发任务状态重评估（见 3.3）
```

> ⚠️ OQ-002：BCA 接口幂等性待确认。若不幂等，批次重试时需评估去重策略。

### 3.3 任务状态重评估（覆盖 AC-015-003、AC-015-004）

```
// 每次批次状态变更后触发，在同一事务内执行
// 仅在所有批次均已落地（无 sending 状态）时才写入 Task 终态
function reEvaluateTaskStatus(task_id):
  batches = SELECT status FROM batches WHERE task_id=? AND deleted_at IS NULL
  if any(b.status == 'sending' for b in batches):
    return  // 仍有批次在发送中，不写入任务状态，等待全部响应
  if all(b.status == 'success' for b in batches):
    task.status = 'success'       // AC-015-003
  elif all(b.status == 'failed' for b in batches):
    task.status = 'failed'        // AC-015-004
  else:
    task.status = 'partial_failed'  // 有成功有失败，无进行中
  UPDATE submission_tasks SET status=task.status WHERE id=task_id
```

### 3.4 批次级重试（覆盖 AC-015-005、AC-015-011）

```
1. 权限校验：同 3.1 步骤 1
2. 校验批次状态为 failed，否则：
   - 若为 sending → 返回 BCA_BATCH_SENDING（409）  // AC-015-011
   - 其他 → 422
3. 事务开始：
   UPDATE batch SET status='pending', error_message=NULL, error_code=NULL
   UPDATE batch_records SET status='pending', error_message=NULL
     WHERE batch_id=? （⚠️ 整批重发：含已成功记录一并重置，等待 OQ-002 确认幂等后决定是否仅重置失败记录）
4. 事务提交
5. 重新入队：发布批次发送任务消息
6. 触发任务状态重评估
```

### 3.5 单条记录重试（不支持）

> BCA 接口不支持单条记录提交，**不提供单条记录重试 API**（覆盖 AC-015-006）。
>
> 对应的 `POST .../records/:recordId/retry` 接口**不实现**，前端不展示单条重试入口。
> 失败记录须通过所属批次的批次级重试（§3.4）整批重发覆盖。

### 3.6 自动提交调度（覆盖 AC-015-008、AC-015-009）

```
// Cron Job，每天凌晨运行，检查当天是否为某项目的触发日
1. 查询所有已开启自动提交的配置：
   SELECT * FROM auto_submit_configs WHERE enabled=true
     AND trigger_day = DAY_OF_MONTH(NOW())
     AND trigger_time <= TIME(NOW())
     AND last_triggered_month < LAST_MONTH()
2. 对每条配置：
   a. 检查该项目上月是否已有任务（status != failed）
      若已有 → 跳过（AC-015-009）
      若无 → 进入创建流程（同 3.1，trigger_type='auto'）
   b. UPDATE auto_submit_configs SET last_triggered_month=LAST_MONTH()
3. 记录调度日志
```

> ⚠️ 为防止 Cron Job 重复触发（多实例部署），需在步骤 1 加分布式锁或 DB 唯一约束保护。

---

## 4. 状态机实现

> 引用需求 §5 的状态机定义。

### 4.1 Submission Task 状态机

**实现方式**：应用层状态机，通过 `reEvaluateTaskStatus()` 函数集中管理状态转换，禁止直接修改 `status` 字段。

**关键约束**：
- 任何状态转换必须通过 `reEvaluateTaskStatus()` 或 `batchStateMachine.transition()` 入口
- 非法转换抛 `IllegalStateTransitionException`，映射为 HTTP 422
- 并发重试同一批次：行级锁（`SELECT FOR UPDATE`）保护

### 4.2 Batch 状态机

**非法转换防护**：

| 尝试操作 | 当前状态 | 拦截响应 |
|---------|---------|---------|
| 重试批次 | sending | 409 BCA_BATCH_SENDING |
| 重试批次 | success | 422 非法操作 |
| 重试批次 | pending | 422 非法操作 |

---

## 5. 事务边界

| 操作 | 事务边界 | 说明 |
|-----|---------|------|
| 创建任务 + 生成批次 + 生成批次记录快照 | ✅ 同一事务 | 三者必须原子，否则出现任务无批次的脏数据 |
| 批次状态更新 + 记录状态更新 + 任务状态重评估 | ✅ 同一事务 | 保证三层状态一致性 |
| 入队消息（发布到 MQ） | ❌ 事务提交后再发 | 使用 Outbox 模式，见 §7.3 |
| 批次重试重置状态 + 入队 | ✅ 状态重置在同一事务；入队走 Outbox | — |
| 单条记录重试 + 状态更新 + 批次/任务状态重评估 | ✅ 同一事务 | — |

---

## 6. 幂等性要求

| API | 幂等键 | 策略 |
|-----|-------|-----|
| `POST /bca-submissions`（创建任务） | `(project_id, submission_month)` | DB 唯一约束（任何已有任务均返回 422） |
| 批次发送队列消费 | `batch_id` | 消费前校验 batch.status=pending；非 pending 则幂等跳过（锁+check） |
| `POST .../batches/:batchId/retry` | `batch_id` | 操作前 `SELECT FOR UPDATE` 校验状态，sending 时返回 409 |
| BCA 接口调用幂等 | 待 OQ-002 确认 | 若 BCA 不幂等，重试前需评估去重策略 |

**幂等存储**：
- 批次发送幂等：DB 行级锁（不用 Redis，批次状态已在 DB 中）
- Cron 调度幂等：`last_triggered_month` + DB 唯一约束防重复创建

---

## 7. 异步任务与事件

### 7.1 任务清单

| 任务 ID | 触发方式 | 队列 | 重试次数 | 死信处理 | SLA |
|--------|---------|------|---------|---------|-----|
| TASK-BATCH-SEND | 创建任务后 / 批次重试后入队 | `bca-batch-send` | 0（不自动重试，由人工触发重试） | 死信队列记录，告警通知 | 单批次 30s 超时 |
| TASK-AUTO-SUBMIT | Cron Job 每日触发 | 无队列，直接调用 Service | 0 | Cron 失败告警 | — |

> ⚠️ 批次发送**不自动重试**（需求 §9.3 明确）：失败后由人工在页面触发重试，避免频繁调用 BCA 接口。

### 7.2 发件箱模式（Outbox）

批次入队消息使用 Outbox 模式保证「业务事务提交」和「消息发送」原子性：

```
[业务事务内]
1. 更新 submission_tasks、batches、batch_records
2. INSERT outbox_events(event_type='BATCH_ENQUEUE', payload={batch_id}, status='PENDING')
[事务提交]
[独立 Outbox 调度器，每秒扫描]
3. SELECT * FROM outbox_events WHERE status='PENDING' ORDER BY id LIMIT 100
4. 发送到 MQ 队列 bca-batch-send
5. UPDATE outbox_events SET status='SENT'
```

---

## 8. 数据库设计

### 8.1 表结构概览

> 字段详细定义见 data-contract.md §1。以下仅列建表要点和索引策略。

```sql
-- 提交任务表
CREATE TABLE submission_tasks (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id        UUID NOT NULL,
  submission_month  CHAR(7) NOT NULL,          -- 格式 YYYY-MM
  trigger_type      VARCHAR(10) NOT NULL,       -- 'manual' | 'auto'
  status            VARCHAR(20) NOT NULL DEFAULT 'processing', -- 内部过渡状态，所有批次响应后写入 success/partial_failed/failed
  total_batches     INT NOT NULL DEFAULT 0,
  success_batches   INT NOT NULL DEFAULT 0,
  failed_batches    INT NOT NULL DEFAULT 0,
  created_by        UUID,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- 防重：同月同项目只允许一条任务（任何状态均不可重复）
CREATE UNIQUE INDEX uq_submission_active
  ON submission_tasks(project_id, submission_month);

-- 批次表
CREATE TABLE batches (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  task_id       UUID NOT NULL REFERENCES submission_tasks(id),
  batch_index   INT NOT NULL,
  status        VARCHAR(20) NOT NULL DEFAULT 'pending',
  record_count  INT NOT NULL DEFAULT 0,
  error_code    VARCHAR(100),
  error_message TEXT,
  sent_at       TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_batches_task_id ON batches(task_id);
CREATE INDEX idx_batches_status ON batches(task_id, status);

-- 批次记录快照表
CREATE TABLE batch_records (
  id                                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  batch_id                                UUID NOT NULL REFERENCES batches(id),
  task_id                                 UUID NOT NULL,
  -- 人员通行快照字段（完整镜像，见 data-contract §1.3）
  submission_entity                       INT,
  submission_month                        CHAR(7),
  project_reference_number                VARCHAR(100),
  project_title                           TEXT,
  project_location_description            TEXT,
  main_contractor_company_name            TEXT,
  main_contractor_company_unique_entity_number  VARCHAR(50),
  person_id_no                            VARCHAR(50),
  person_id_and_work_pass_type            VARCHAR(20),
  person_trade                            VARCHAR(20),
  person_employer_company_name            TEXT,
  person_employer_company_unique_entity_number  VARCHAR(50),
  person_employer_company_trade           JSONB,
  person_employer_client_company_name     TEXT,
  person_employer_client_company_unique_entity_number  VARCHAR(50),
  person_attendance_date                  DATE,
  person_attendance_details               JSONB,   -- [{time_in, time_out}]
  -- 发送状态
  status                                  VARCHAR(20) NOT NULL DEFAULT 'pending',
  error_message                           TEXT,
  created_at                              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_batch_records_batch_id ON batch_records(batch_id);
CREATE INDEX idx_batch_records_status   ON batch_records(batch_id, status);

-- 自动提交配置表
CREATE TABLE auto_submit_configs (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id            UUID NOT NULL UNIQUE,
  enabled               BOOLEAN NOT NULL DEFAULT FALSE,
  trigger_day           INT NOT NULL DEFAULT 1,    -- 1~7
  trigger_time          TIME NOT NULL DEFAULT '02:00:00',
  last_triggered_month  CHAR(7),
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Outbox 事件表（如项目尚未建立，需新增）
CREATE TABLE outbox_events (
  id          BIGSERIAL PRIMARY KEY,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  event_type  VARCHAR(100) NOT NULL,
  payload     JSONB NOT NULL,
  status      VARCHAR(20) NOT NULL DEFAULT 'PENDING'
);
CREATE INDEX idx_outbox_pending ON outbox_events(status, id) WHERE status='PENDING';
```

### 8.2 数据库迁移

- 工具：项目现有迁移工具（Flyway / Liquibase / Alembic）
- 命名：`V{时间戳}__add_bca_submission_tables.sql`
- 新增索引使用 `CREATE INDEX CONCURRENTLY`（零停机）
- 新字段均有默认值或允许 NULL

---

## 9. 缓存策略

| 数据 | 缓存层 | TTL | 失效时机 |
|-----|-------|-----|---------|
| 任务列表（筛选结果） | 不缓存 | — | 数据实时性要求高，轮询场景下不适合缓存 |
| BCA API Key / Secret | Redis / 加密存储 | 永久（人工配置） | 仅后台更新时失效 |
| 分布式锁（批次发送） | Redis | 批次 SLA + buffer（60s） | 锁释放时删除 |

---

## 10. 并发控制

| 场景 | 策略 | 说明 |
|-----|------|------|
| 同时重试同一批次 | DB 行级锁（`SELECT FOR UPDATE`）+ status 校验 | 第二个请求取锁后发现 status=sending，返回 409 |
| 同月同项目并发创建任务 | DB 唯一索引（`uq_submission_active`） | 后者 INSERT 抛唯一约束异常，映射为 422 |
| 批次队列并发消费 | `SELECT FOR UPDATE` 保证单消费者处理 | 多 Worker 时防重复发送 |
| Cron Job 多实例 | 分布式锁（Redis）+ `last_triggered_month` 约束 | 防止同月重复调度 |

---

## 11. 审计日志

| 操作 | 必须审计 | 审计内容 |
|-----|---------|---------|
| 创建提交任务（手动） | ✅ | actor_id、project_id、submission_month、task_id |
| 批次重试 | ✅ | actor_id、task_id、batch_id、retry_at |
| 自动任务创建（系统触发） | ✅ | actor_id=system、project_id、submission_month、task_id |
| 保存自动提交配置 | ✅ | actor_id、project_id、config_diff |
| 查询任务/批次列表 | ❌ | 读操作不审计 |

**审计表**：复用项目现有 `audit_logs` 表（含 `actor_id`、`action`、`resource_type`、`resource_id`、`payload`）。

**保留期限**：≥ 3 年（合规要求，BCA 相关数据需长期保留）。

---

## 12. 性能要求

| 接口 | P95 延迟 | 目标 QPS | 测试方式 |
|-----|---------|---------|---------|
| `GET /bca-submissions`（列表） | < 500ms | 20 | 压测 |
| `GET /bca-submissions/:id`（详情） | < 500ms | 50（轮询高频） | 压测 |
| `GET .../batches`（批次列表） | < 500ms | 50 | 压测 |
| `POST /bca-submissions`（创建任务含拆批） | < 5s（万级数据） | 5 | 压测 |
| `POST .../retry`（批次/记录重试） | < 500ms（不含 BCA 调用） | 10 | 压测 |

**关键优化**：
- 批次列表查询：`idx_batches_task_id` 覆盖索引
- 创建任务拆批：批量 INSERT（`INSERT ... VALUES (...), (...)` 方式，避免 N 次单条插入）
- 任务详情轮询：`updated_at` 字段支持增量判断，减少不必要的全量渲染
- 记录列表分页：`GET .../records?page=1&page_size=100`，前端懒加载

---

## 13. 安全要求

### 13.1 输入校验

- `submission_month` 格式：`/^\d{4}-(0[1-9]|1[0-2])$/`，且不可超过当月
- `trigger_day` 范围：1 ~ 7（整数）
- `batch_size` 系统配置项，不对外暴露，不允许客户端传入

### 13.2 鉴权与权限

- 所有接口必须携带有效 JWT，否则 401
- Site Admin：仅可操作/查询自己有权限的 project_id（后端行级过滤）
- Platform Admin：可操作所有 project_id
- BCA API Key：仅后端读取，不通过任何前端 API 返回

### 13.3 外部接口安全

- BCA API Key 加密存储（AES-256 或 KMS），不明文入库
- 调用 BCA 接口时通过 HTTPS，TLS 1.2+
- 超时 30s，不重试（由人工触发重试）

### 13.4 限流

- 创建任务接口：每项目每分钟最多 5 次（防脚本刷）
- 重试接口：每批次每分钟最多 3 次
- 实现：Redis + 滑动窗口

---

## 14. 可观测性

### 14.1 日志

- 结构化日志（JSON），必含：`trace_id`、`request_id`、`user_id`、`task_id`、`batch_id`
- 关键节点日志：任务创建、批次入队、批次发送开始/结束、BCA 响应（脱敏）、状态转换

### 14.2 Metrics

| 指标 | 类型 | 说明 |
|-----|------|------|
| `bca_submission_task_created_total` | Counter | 按 trigger_type 区分 |
| `bca_batch_send_duration_seconds` | Histogram | BCA 接口调用耗时 |
| `bca_batch_send_result_total` | Counter | 按 status(success/failed) 区分 |
| `bca_batch_retry_total` | Counter | 批次重试次数 |
| `bca_submission_queue_depth` | Gauge | 待发送批次队列深度 |

### 14.3 告警

| 告警项 | 阈值 | 等级 |
|-------|------|-----|
| BCA 接口错误率 | > 10%（5 分钟内） | P1 |
| 批次队列堆积 | > 100 条 pending 超过 10 分钟 | P2 |
| Cron Job 未按时执行 | 触发日 +1 小时内无执行记录 | P2 |
| 任务创建失败 | 任意 500 错误 | P1 |

---

## 15. AC 覆盖检查表

| AC ID | 实现位置 | 测试覆盖 | 状态 |
|------|---------|---------|------|
| AC-015-001 | `SubmissionService.createTask()` | 单元 + 集成 | TODO |
| AC-015-002 | `SubmissionService.createTask()` → 防重复校验 | 单元 + 集成 | TODO |
| AC-015-003 | `BatchSendWorker` → 成功分支 + `reEvaluateTaskStatus()` | 集成 | TODO |
| AC-015-004 | `BatchSendWorker` → 失败分支 + `reEvaluateTaskStatus()` | 集成 | TODO |
| AC-015-005 | `BatchService.retryBatch()` | 单元 + 集成 | TODO |
| AC-015-006 | 不提供单条记录重试 API（`POST .../records/:recordId/retry` 不实现） | 契约测试：该路径返回 404 | TODO |
| AC-015-007 | `SubmissionService.createTask()` → 无数据校验 | 单元 | TODO |
| AC-015-008 | `AutoSubmitCronJob.run()` → 自动创建任务 | 集成 | TODO |
| AC-015-009 | `AutoSubmitCronJob.run()` → 已有任务时跳过 | 单元 + 集成 | TODO |
| AC-015-010 | `SubmissionService.createTask()` → 快照写入 `batch_records` | 集成 | TODO |
| AC-015-011 | `BatchService.retryBatch()` → sending 状态返回 409 | 单元 | TODO |

---

## 16. 测试要求

| 层级 | 框架 | 范围 | 覆盖率目标 |
|-----|------|------|-----------|
| 单元 | 项目现有框架 | 业务逻辑（状态机转换、防重校验、无数据校验、幂等判断） | ≥ 80% |
| 集成 | Testcontainers + PostgreSQL | 完整创建任务流程、批次发送流程、重试流程、Cron Job | 主流程 100% |
| 契约 | schemathesis / Pact | 所有对外 REST API Schema | 全部接口 |
| 状态机 | 单元 | 所有合法转换 + 所有非法转换 | 100% |

---

## 17. 部署与回滚

- 部署方式：K8s 滚动发布
- DB 迁移：先部署迁移（新表、新索引），再发布应用（新表无旧数据，向下兼容）
- 回滚：通过 Feature Flag 关闭 BCA 数据提交入口；Cron Job 可通过配置禁用；DB 新增表回滚时执行 DROP TABLE（数据为新功能数据，可接受）
- 灰度：见需求 §14

---

## 18. 验收条件

- [ ] 所有 AC 在 §15 表中标记完成
- [ ] 单元 + 集成测试通过，覆盖率达标
- [ ] 契约测试与前端通过
- [ ] 压测接口延迟达到 §12 目标
- [ ] BCA API Key 加密存储验证通过
- [ ] 安全扫描无 high 项
- [ ] 已部署到 dev，可联调
- [ ] 审计日志写入验证通过

---

## 19. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-17 | | 初稿，包含任务创建、批次发送引擎、重试机制、自动调度、数据快照、DB 设计 |
| 0.2.0 | 2026-05-18 | | 同步 REQ-015 v0.2.0~v0.5.0：1) 防重校验改为任何已有任务均拦截，移除 status 过滤；2) DB 唯一索引去掉 WHERE 条件；3) 移除 §3.5 单条记录重试，改为不支持说明；4) reEvaluateTaskStatus 移除 in_progress，仅在所有批次落地后写 Task 终态；5) 幂等表移除单条重试条目；6) 审计日志移除单条重试行；7) Metrics 移除 bca_record_retry_total；8) submission_tasks 初始状态改为 processing |
