---
doc_type: backend_spec
req_id: REQ-003E-pc
version: 0.2.0
status: draft
generated_from: REQ-003E-pc.md@0.3.1
data_contract_ref: data-contract.md@0.1.0
generated_at: 2026-05-24
updated_at: 2026-05-25
owner: ""
---

# 后端开发说明：PC 端 — 图纸上传 AI 自动识别页信息

> **本文档供后端开发工程师及其 agent 使用**。
>
> ⚠️ **重要约定**：
> - 数据模型与 API 字段定义完全引用 `data-contract.md`，本文档不重复定义。
> - 业务规则、状态转换、事务边界、幂等性、审计要求在本文档详述。
> - 代码注释中必须标注覆盖的 AC ID。
> - API 风格：Action-based（非 RESTful），遵循 `backend-rules.md §8`。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003E-pc @ v0.3.1 |
| 数据契约 | data-contract.md @ v0.1.0 |
| 覆盖 Story | US-003E-001、US-003E-002 |
| 覆盖 AC | AC-003E-001 ~ AC-003E-013 |

---

## 1. 功能概述

本次后端需交付以下能力：

1. **文件上传接收**：接收前端上传的 PDF 文件，存储到对象存储（OSS/S3），返回文件 URL 与识别任务 ID。
2. **AI 识别任务管理**：创建 `AIRecognitionJob` 记录，异步调用 AI 识别服务，维护任务状态（PENDING → PROCESSING → DONE / FAILED）。
3. **识别结果查询**：提供轮询接口，返回任务状态及各页识别结果（页码、缩略图 URL、Drawing No、Drawing Name、置信度）。
4. **Drawing 创建（含 AI 关联）**：复用 REQ-003A 创建接口，扩展支持传入 `ai_recognition_job_id`，在创建 Drawing 记录时持久化关联来源识别任务。
5. **Drawing 详情查询（含 AI 页面信息）**：Drawing 详情接口返回关联的 AI 识别页面信息（页码、Drawing No、Drawing Name）。

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

- 语言：Java
- 框架：Spring Cloud（微服务）
- 数据库：MySQL
- 缓存：Redis
- 消息队列：RabbitMQ
- 对象存储：OSS / S3
- API 风格：Action-based
- 鉴权：JWT

### 2.2 本需求新增依赖

| 库 / 服务 | 用途 | 评估 |
|---------|------|------|
| AI 识别服务（内部 / 第三方，见 OQ-006） | 识别 PDF 页面图框中的 Drawing No / Name | 接口约定需与 AI 团队联调 |
| PDF 缩略图生成（如 Apache PDFBox） | 为每页生成低分辨率预览图并上传 OSS | 可在 AI 识别任务执行时同步完成 |

---

## 3. 业务逻辑

### 3.1 文件上传与识别任务创建（覆盖 AC-003E-001）

```
1. 校验权限：JWT 解析用户，校验具有 drawing:create 权限
2. 校验输入：
   - 文件类型必须为 application/pdf（MIME 白名单）// AC-003E-010
   - 文件大小 ≤ 50MB // AC-003E-011
3. 上传文件到 OSS/S3，获取 fileUrl
4. 为每页生成缩略图并上传 OSS，获取 thumbnailUrls[]（可异步，识别完成前完成）
5. 创建 AIRecognitionJob 记录：
   - status = PENDING
   - fileUrl = 上传后 URL
   - projectId = 当前项目 ID
   - creatorId = 当前用户 ID
6. 发布消息到 MQ（queue: ai.recognition.job.created），payload: { jobId, fileUrl }
7. 返回 { jobId, status: 'PENDING' }
```

### 3.2 AI 识别任务异步处理（覆盖 AC-003E-001、AC-003E-002、AC-003E-006）

```
[消费者：ai.recognition.job.created]
1. 更新 AIRecognitionJob.status = PROCESSING
2. 调用 AI 识别服务（传入 fileUrl），等待结果（超时阈值见 OQ-001，建议 30s）
3. AI 识别成功（DONE）：
   - 将各页结果写入 AIRecognitionJob.pages JSON：
     { pageNo, thumbnailUrl, drawingNo, drawingName, confidence }
   - 更新 status = DONE
   - 发布 WebSocket 事件（如支持）或等待前端轮询
4. AI 识别失败（超时 / 异常）：
   - 更新 status = FAILED，记录 failureReason
   - 发布 WebSocket 事件或等待前端轮询
```

> **幂等要求**：消费者处理前检查 `status != PENDING` 时跳过（防止重复消费）。

### 3.3 识别任务状态查询（覆盖 AC-003E-001、AC-003E-002、AC-003E-006、AC-003E-009）

```
API: POST /api/drawing/recognition/queryJob
Request: { jobId }
1. 校验权限：JWT 用户 + 项目归属校验（只能查询本项目 / 本人创建的 job）
2. 查询 AIRecognitionJob 记录
3. 若不存在：返回 404
4. 返回 {
     jobId,
     status,           // PENDING / PROCESSING / DONE / FAILED
     totalPages,       // status = DONE 时有效
     pages: [          // status = DONE 时有效
       { pageNo, thumbnailUrl, drawingNo, drawingName, confidence }
     ],
     failureReason     // status = FAILED 时有效
   }
```

### 3.4 Drawing 创建（扩展 REQ-003A 接口，覆盖 AC-003E-005、AC-003E-012、AC-003E-013）

```
API: POST /api/drawing/create（复用 REQ-003A，扩展字段）
新增字段：aiRecognitionJobId（可选）

1. 校验权限：drawing:create
2. 校验输入：
   - Drawing Code 格式 + 唯一性（项目内）
   - Drawing Name 非空
   - Category 存在
   - Internal Approver 存在且有 drawing:approve 权限
3. 若 aiRecognitionJobId 非空：
   - 校验该 job 存在且归属当前项目（防越权）
   - 校验 job.status = DONE 或 FAILED（不允许识别进行中 PENDING/PROCESSING 时提交）
     - DONE：AI 识别成功，Drawing 关联 job 并持久保存识别页面信息 // AC-003E-012
     - FAILED：AI 识别失败但用户选择手动填写，Drawing 仍关联 job（可溯源失败原因），job 字段非 null
     - 若 status = PENDING / PROCESSING：返回 422，`AI_RECOGNITION_IN_PROGRESS`
4. 开启事务：
   a. INSERT Drawing（含 ai_recognition_job_id 字段）
   b. INSERT DrawingVersion（V0，status = PENDING_INTERNAL）
   c. INSERT 内部审批任务 → Todo 推送
5. 提交事务
6. 审计：记录 actor_id, created_drawing_code, ai_recognition_job_id, timestamp
7. 返回 { drawingId, drawingCode, status: 'PENDING_INTERNAL' }
```

> - `aiRecognitionJobId` 为 null 时（降级完全手动输入，未上传或用户未触发识别）：正常创建，`Drawing.ai_recognition_job_id` 为 NULL // AC-003E-013
> - `aiRecognitionJobId` 非 null 且 status=DONE 时：Drawing 持久关联识别任务，详情页展示 AI 页面信息 // AC-003E-012
> - `aiRecognitionJobId` 非 null 且 status=FAILED 时：Drawing 记录关联该 job（`ai_recognition_job_id` 非空），但详情页行为与 AC-003E-013 一致（不展示 AI 识别结果区域，因无有效 pages 数据）

### 3.5 Drawing 详情查询（扩展，覆盖 AC-003E-012、AC-003E-013）

```
API: POST /api/drawing/getDetail（扩展响应）
新增响应字段：aiRecognitionInfo（可选）

1. 查询 Drawing 记录
2. 若 Drawing.ai_recognition_job_id 非 null：
   - JOIN AIRecognitionJob 获取 status 和 pages 信息
   - 若 job.status = DONE（有有效 pages）：
     响应中包含 aiRecognitionInfo: {
       jobId, totalPages,
       pages: [{ pageNo, drawingNo, drawingName }]
     }
   - 若 job.status = FAILED（无有效 pages）：
     响应中 aiRecognitionInfo = null（前端不展示 AI 识别结果区域，同 AC-003E-013 行为）
3. 若 ai_recognition_job_id 为 null：
   - 响应中 aiRecognitionInfo = null // AC-003E-013
```

---

## 4. 状态机实现

### 4.1 AIRecognitionJob 状态机

**实现方式**：应用层状态机，通过 `AiRecognitionJobStateMachine.transition(job, action)` 入口

**关键约束**：
- 禁止业务代码直接修改 `status` 字段
- 非法转换抛 `IllegalStateTransitionException`，映射为 422
- `DONE → PROCESSING` 和 `FAILED → DONE` 为非法转换（见 REQ-003E §5.3）

**状态转换伪代码**：

```pseudo
function transition(job, action):
  allowed = {
    (PENDING,    START_PROCESSING) -> PROCESSING,
    (PROCESSING, COMPLETE)         -> DONE,
    (PROCESSING, FAIL)             -> FAILED,
    (PENDING,    TIMEOUT)          -> FAILED,
  }
  target = allowed.get((job.status, action))
  if target is None: throw IllegalStateTransitionException
  acquire_lock(job.id)
  job.status = target
  job.save()
  emit_event(action + ".completed", { jobId: job.id })
  release_lock()
```

---

## 5. 事务边界

| 操作 | 事务边界 | 说明 |
|-----|---------|------|
| 上传文件 + 创建 AIRecognitionJob | ❌ 拆开（OSS 在事务外） | OSS 上传与 DB 写入分离；OSS 成功后写 DB；OSS 失败直接返回错误 |
| AI 识别完成 → 更新 Job 状态 + 写 pages JSON | ✅ 同一事务 | 防止状态更新成功但 pages 写入失败导致数据不一致 |
| 创建 Drawing + DrawingVersion + Todo 推送 | ✅ 同一事务（Outbox 模式） | Drawing/Version 写入 + outbox_events 写入同一事务；Todo 推送由独立调度器消费 outbox |

---

## 6. 幂等性要求

| API | 幂等键 | 策略 |
|-----|-------|-----|
| `drawing/create` | 客户端传入 `clientRequestId`（UUID） | Redis key = `idem:drawing:create:{clientRequestId}`，TTL 24h；首次写入并返回，重复请求返回原结果 |
| AI 识别任务消费者 | `jobId` | 消费前检查 `status != PENDING` 则跳过 |
| OSS 文件上传 | 文件 MD5 + projectId | 相同文件相同项目复用已有 URL（可选优化） |

---

## 7. 异步任务与事件

### 7.1 任务执行规范

| 任务 ID | 触发 | 队列 | 重试 | 死信处理 | SLA |
|--------|------|-----|------|---------|-----|
| TASK-AI-RECOGNITION | 文件上传成功后发布 MQ 消息 | `ai.recognition.job.created` | 最多 3 次，指数退避 | 死信后将 job status 置为 FAILED | 30s（OQ-001 待确认）|
| TASK-TODO-NOTIFY | Drawing 创建成功后 outbox 消费 | `drawing.created` | 最多 5 次 | 告警人工介入 | 5s |

### 7.2 Outbox 模式（Drawing 创建 → Todo 推送）

```
[在业务事务内]
1. INSERT drawing
2. INSERT drawing_version
3. INSERT outbox_events(event_type='DRAWING_CREATED', payload={drawingId, approverId}, status=PENDING)
[事务提交]
[独立调度器]
4. 扫描 outbox_events WHERE status=PENDING
5. 调用消息通知服务推送 Todo
6. UPDATE outbox_events SET status=SENT
```

---

## 8. 数据库设计

### 8.1 新增表：ai_recognition_jobs

```sql
-- 引用 data-contract.md §ENT-003
CREATE TABLE ai_recognition_jobs (
  id              VARCHAR(36) PRIMARY KEY,
  project_id      VARCHAR(36) NOT NULL,
  creator_id      VARCHAR(36) NOT NULL,
  file_url        VARCHAR(1024) NOT NULL,
  status          ENUM('PENDING','PROCESSING','DONE','FAILED') NOT NULL DEFAULT 'PENDING',
  total_pages     INT NULL,
  pages_json      JSON NULL COMMENT '各页识别结果: [{pageNo, thumbnailUrl, drawingNo, drawingName, confidence}]',
  failure_reason  VARCHAR(500) NULL,
  created_at      DATETIME NOT NULL DEFAULT NOW(),
  updated_at      DATETIME NOT NULL DEFAULT NOW() ON UPDATE NOW(),
  deleted_at      DATETIME NULL
);

CREATE INDEX idx_ai_jobs_project_status ON ai_recognition_jobs(project_id, status)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_ai_jobs_creator ON ai_recognition_jobs(creator_id)
  WHERE deleted_at IS NULL;
```

### 8.2 变更表：drawings（扩展列）

```sql
-- 引用 data-contract.md §ENT-001
ALTER TABLE drawings
  ADD COLUMN ai_recognition_job_id VARCHAR(36) NULL
    COMMENT 'AI 识别任务 ID，降级手动上传时为 NULL',
  ADD INDEX idx_drawings_ai_job(ai_recognition_job_id);
```

### 8.3 数据库迁移

- 工具：Flyway
- 迁移文件命名：
  - `V{timestamp}__add_ai_recognition_jobs_table.sql`
  - `V{timestamp}__add_ai_recognition_job_id_to_drawings.sql`
- `ai_recognition_job_id` 列允许 NULL，存量数据不受影响

---

## 9. 缓存策略

| 数据 | 缓存层 | TTL | 失效时机 |
|-----|-------|-----|---------|
| AIRecognitionJob 状态查询 | Redis | 5s | job status 变更时主动失效 |
| Drawing 详情（含 AI 页面信息） | Redis | 60s | Drawing 更新时失效 |

---

## 10. 并发控制

| 场景 | 策略 | 说明 |
|-----|------|------|
| 同一 jobId 被多次消费 | MQ 消费者幂等（检查 status） | 防止 AI 识别重复触发 |
| 同一 Drawing Code 并发创建 | DB 唯一约束（project_id + drawing_code） | 重复时抛 409 |
| AI 识别任务状态并发更新 | 行级锁（SELECT ... FOR UPDATE） | 防止状态机并发跳转 |

---

## 11. 审计日志

| 操作 | 必须审计 | 审计内容 |
|-----|---------|---------|
| 上传 PDF 文件 | ✅ | actor_id, project_id, file_name, file_size, job_id, timestamp |
| AI 识别完成 | ✅ | job_id, total_pages, status=DONE, timestamp |
| AI 识别失败 | ✅ | job_id, failure_reason, timestamp |
| 创建 Drawing | ✅ | actor_id, drawing_code, ai_recognition_job_id（可 null）, timestamp |

**保留期限**：遵循项目审计日志策略（待 PM 确认，建议 ≥ 3 年）。

---

## 12. 性能要求

| 接口 | P95 延迟 | QPS | 测试方式 |
|-----|---------|-----|---------|
| 文件上传接口（50MB） | ≤ 60s | — | 实测，正常网络 |
| 识别任务状态查询 | ≤ 200ms | ≤ 100 | 后端压测 |
| Drawing 创建接口 | ≤ 500ms | ≤ 50 | 后端压测 |
| Drawing 详情接口（含 AI 页面信息） | ≤ 500ms | ≤ 200 | 后端压测 |

**关键优化**：
- `ai_recognition_jobs.pages_json` 字段使用 MySQL JSON 类型，避免额外关联表
- 识别任务状态查询加 Redis 缓存（TTL 5s），减少 DB 轮询压力
- 缩略图生成与 AI 识别并行进行（异步子任务）

---

## 13. 安全要求

### 13.1 输入校验

- 文件类型：服务端检查 MIME + 文件头魔数（`%PDF`），双重校验，防绕过
- 文件大小：Nginx/Gateway 层限制 + Spring 层限制，均设为 50MB
- AI 识别服务凭证：仅在服务端持有，不向前端暴露
- OSS 访问：使用预签名 URL（过期时间 ≤ 1h），不直接暴露 bucket 路径

### 13.2 鉴权与权限

- 所有接口必须校验 JWT
- `queryJob` 接口：校验 job 的 `project_id` 与当前用户所在项目匹配（防越权）
- `createDrawing` 接口：校验用户具有 `drawing:create` 权限

### 13.3 限流

- 文件上传接口：单用户 ≤ 10 次/min（防止 AI 资源滥用）
- 识别任务状态查询：单用户 ≤ 60 次/min（轮询兜底限流）

---

## 14. 可观测性

### 14.1 日志

结构化 JSON 日志，必含：`trace_id`、`request_id`、`user_id`、`project_id`

关键业务日志点：
- 文件上传开始 / 成功 / 失败
- AI 识别任务创建 / 进入 PROCESSING / DONE / FAILED
- Drawing 创建成功 / 失败

### 14.2 Metrics

| 指标 | 类型 | 标签 |
|-----|------|------|
| `ai_recognition_job_total` | Counter | status=(DONE/FAILED) |
| `ai_recognition_duration_seconds` | Histogram | — |
| `drawing_create_total` | Counter | ai_used=(true/false) |
| `ai_recognition_failure_rate` | Gauge | project_id |

### 14.3 告警

| 告警项 | 阈值 | 等级 |
|-------|------|-----|
| AI 识别失败率 | > 20%（5 分钟滑动窗口）| P2 |
| AI 识别平均耗时 | > 20s（P95）| P2 |
| Drawing 创建失败率 | > 5% | P2 |

---

## 15. AC 覆盖检查表

| AC ID | 实现位置 | 测试覆盖 | 状态 |
|------|---------|---------|------|
| AC-003E-001 | 文件上传接口 + AIRecognitionJob 创建 | 集成测试 | TODO |
| AC-003E-002 | AI 识别消费者 → status=DONE + pages 写入 | 单元测试（状态机）+ 集成测试 | TODO |
| AC-003E-003 | 前端逻辑（后端不涉及）| — | N/A |
| AC-003E-004 | `pages_json` 中 drawingNo/Name 允许为空，前端渲染 | 集成测试 | TODO |
| AC-003E-005 | `drawing/create` 创建单条 Drawing + 单条 Todo | 集成测试 + 事务测试 | TODO |
| AC-003E-006 | AI 识别消费者 → status=FAILED | 单元测试 | TODO |
| AC-003E-007 | `drawing/create` 支持 ai_recognition_job_id=null | 集成测试 | TODO |
| AC-003E-008 | 前端逻辑（后端不涉及）| — | N/A |
| AC-003E-009 | 前端逻辑（后端不涉及）| — | N/A |
| AC-003E-010 | 文件上传接口 MIME + 魔数校验 | 单元测试 | TODO |
| AC-003E-011 | 文件上传接口大小校验 | 单元测试 | TODO |
| AC-003E-012 | `drawing/getDetail` 返回 aiRecognitionInfo | 集成测试 | TODO |
| AC-003E-013 | `drawing/getDetail` 当 job_id=null 时返回 null | 集成测试 | TODO |

---

## 16. 测试要求

| 层级 | 框架 | 范围 | 覆盖率目标 |
|-----|------|------|-----------|
| 单元 | JUnit 5 + Mockito | 状态机、业务逻辑、文件校验 | ≥ 80% |
| 集成 | Testcontainers（MySQL + Redis）| 上传流程、识别任务流转、创建接口 | 核心路径 100% |
| 契约 | schemathesis | 所有新增 / 扩展接口 | 全部接口 |
| 状态机 | 单元 | AIRecognitionJob 全部合法 + 非法转换 | 100% |

---

## 17. 部署与回滚

- DB 迁移：先发含 `ai_recognition_job_id` 允许 NULL 的兼容版本，再部署新服务
- 功能开关：`feature.ai-recognition.enabled`，关闭时上传接口跳过 AI 识别任务创建（降级为 REQ-003A 原始流程）
- 回滚：关闭功能开关，存量 Drawing 记录不受影响（`ai_recognition_job_id` 为 NULL 的记录正常展示）

---

## 18. 验收条件

后端开发完成的判定：

- [ ] 所有 AC 在 §15 表中标记完成
- [ ] 单元 + 集成测试通过，覆盖率达标
- [ ] 契约测试与前端通过
- [ ] 压测达到 §12 目标
- [ ] 安全扫描无 high 项
- [ ] 已部署到 dev，可与前端联调

---

## 19. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-24 | agent | 初稿，从 REQ-003E-pc@0.3.1 派生 |
| 0.2.0 | 2026-05-25 | agent | §3.4 补充 job status=FAILED 时仍可关联的逻辑及 PENDING/PROCESSING 422 错误码；§3.5 补充 FAILED job 对应 `aiRecognitionInfo=null` 的返回行为；注释更精确区分三种 aiRecognitionJobId 场景 |
