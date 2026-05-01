---
doc_type: backend_spec
req_id: REQ-XXXX
version: 0.1.0
status: draft
generated_from: requirement.md@0.1.0
data_contract_ref: data-contract.md@0.1.0
generated_at: YYYY-MM-DD
owner: ""
---

# 后端开发说明:[需求标题]

> **本文档供后端开发工程师及其 agent 使用**。
>
> ⚠️ **重要约定**:
> - 数据模型与 API 字段定义完全引用 `data-contract.md`,本文档不重复定义。
> - 业务规则、状态转换、事务边界、幂等性、审计要求在本文档详述。
> - 代码注释中必须标注覆盖的 AC ID。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-XXXX @ v0.1.0 |
| 数据契约 | data-contract.md @ v0.1.0 |
| 覆盖 Story | US-XXX |
| 覆盖 AC | AC-XXXX-XXX |

---

## 1. 功能概述

<!-- 本次后端要交付的能力 -->

---

## 2. 技术栈

### 2.1 已有技术栈(继承)

<!-- 列出团队既定栈,例如:
- 语言
- 框架
- 数据库
- 缓存
- 消息队列
- 对象存储
-->

### 2.2 本需求新增依赖

| 库 | 用途 | 评估 |
|---|------|------|
| | | |

---

## 3. 业务逻辑

> 每个核心业务点用清晰的伪代码或步骤描述,对应 AC 标注。

### 3.1 [业务点名称](覆盖 AC-XXXX-XXX)

```
1. 校验权限:[校验逻辑]
2. 校验输入:[校验逻辑]
3. [核心步骤]
4. [副作用]
5. 返回结果
```

### 3.2 [其他业务点]

<!-- 按相同结构展开 -->

---

## 4. 状态机实现

> 引用 data-contract §3 的状态机定义。本节只描述实现要点。

### 4.1 [实体名] 状态机

**实现方式**: 应用层状态机(不存数据库存储过程)

**关键约束**:
- 任何状态变更必须通过 `[XxxStateMachine].transition(entity, action)` 入口
- 禁止业务代码直接修改 status 字段(代码 review 拦截)
- 非法转换抛 `IllegalStateTransitionException`,接口层映射为 422

**伪代码**:

```pseudo
function transition(entity, action):
  matched = find_transition(entity.status, action)
  if not matched: throw IllegalAction
  for guard in matched.guard:
    if not evaluate(guard, entity): throw GuardFailed(guard)
  acquire_lock(entity.id)         # 防并发
  entity.status = matched.to
  entity.save()
  for effect in matched.side_effects:
    schedule(effect)
  emit_event(action.completed)
  release_lock()
```

---

## 5. 事务边界

> 哪些操作必须在同一事务,哪些可以拆开。错了会出现脏数据。

| 操作 | 事务边界 | 说明 |
|-----|---------|------|
| | ✅ 同一事务 / ❌ 拆开 | |

---

## 6. 幂等性要求

> ⚠️ 漏做幂等会出生产事故。本表必须严格执行。

| API | 幂等键 | 策略 |
|-----|-------|-----|
| | | |

**幂等存储**:
- 短期(< 24h):Redis,key = `idem:{key}`,value = 原响应序列化
- 长期(永久):DB 唯一约束

---

## 7. 异步任务与事件

> 引用 data-contract §5 的任务清单,本节是实现细节。

### 7.1 任务执行规范

| 任务 ID | 触发 | 队列 | 重试 | 死信处理 | SLA |
|--------|------|-----|------|---------|-----|
| TASK-XXX | | | | | |

### 7.2 事件发布

- 事件总线:<!-- Kafka / RabbitMQ -->
- 消息格式见 data-contract §5
- 至少一次语义(at-least-once),消费者必须幂等

### 7.3 发件箱模式(Outbox)

> 业务事务和事件发布必须原子。采用 Outbox 模式:

```
[在业务事务内]
1. 业务写入
2. INSERT outbox_events(event_type, payload, status=PENDING)
[事务提交]
[独立调度器]
3. 扫描 outbox_events WHERE status=PENDING
4. 发到事件总线
5. UPDATE outbox_events SET status=SENT
```

---

## 8. 数据库设计

### 8.1 表结构

> 详细字段定义见 data-contract.md §1。本节列出建表要点。

```sql
-- 引用 data-contract §1.X
CREATE TABLE [table_name] (
  id            UUID PRIMARY KEY,
  -- ...
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at    TIMESTAMPTZ
);

CREATE INDEX [idx_name] ON [table_name]([cols])
  WHERE deleted_at IS NULL;
```

### 8.2 数据库迁移

- 工具:<!-- Flyway / Liquibase / Alembic -->
- 命名:`V{时间戳}__{描述}.sql`
- 原则:**只前进,不回退**,新增字段必须有默认值或允许 NULL
- 索引创建用 `CONCURRENTLY`(PG)

---

## 9. 缓存策略

| 数据 | 缓存层 | TTL | 失效时机 |
|-----|-------|-----|---------|
| | | | |

**缓存击穿防护**:
- 热 key 用 mutex 防止重复回源
- 缓存空值短期(60s),防穿透

---

## 10. 并发控制

| 场景 | 策略 | 说明 |
|-----|------|------|
| | DB 唯一约束 / 行级锁 / 分布式锁 / 乐观锁 | |

---

## 11. 审计日志

> 哪些操作必须留痕。

| 操作 | 必须审计 | 审计内容 |
|-----|---------|---------|
| | ✅/❌ | who, when, what |

**审计表**:
```sql
CREATE TABLE audit_logs (
  id          BIGSERIAL PRIMARY KEY,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  actor_id    UUID NOT NULL,
  actor_role  VARCHAR(50),
  action      VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50),
  resource_id UUID,
  payload     JSONB,
  ip          INET,
  user_agent  TEXT,
  request_id  VARCHAR(64)
);
```

**保留期限**: <!-- 例如 N 年(依业务/法规) -->

---

## 12. 性能要求

> 引用需求 §9.1。本节给出实现指标。

| 接口 | P95 延迟 | QPS | 测试方式 |
|-----|---------|-----|---------|
| | | | |

**关键优化**:
<!-- 列出本需求采用的优化手段:覆盖索引、批查询、N+1 防护等 -->

---

## 13. 安全要求

### 13.1 输入校验

- 所有外部输入走 DTO 校验
- SQL 用参数化查询,禁止字符串拼接
- 路径/文件名校验,防路径穿越

### 13.2 鉴权与权限

- 鉴权:<!-- JWT / Session -->,中间件统一校验
- 权限校验**每个接口必做**,引用 data-contract §6.2 表格
- 行级权限:列表查询时强制注入归属过滤

### 13.3 限流

- 用户级:<!-- 600 req/min -->
- 接口级:<!-- 高耗资源接口单独限制 -->
- 实现:Redis + 滑动窗口

---

## 14. 可观测性

### 14.1 日志

- 结构化日志(JSON)
- 必含:trace_id, request_id, user_id
- 日志级别:INFO(业务关键路径)、WARN(可重试错误)、ERROR(需告警)

### 14.2 Metrics

- 业务指标:<!-- 列出关键业务计数器/分位数 -->
- 技术指标:接口 QPS/延迟/错误率,DB 连接池,MQ 堆积

### 14.3 Tracing

- 全链路 trace,接 OpenTelemetry
- 跨服务 trace_id 透传

### 14.4 告警

| 告警项 | 阈值 | 等级 |
|-------|------|-----|
| | | |

---

## 15. AC 覆盖检查表

| AC ID | 实现位置 | 测试覆盖 | 状态 |
|------|---------|---------|------|
| AC-XXXX-XXX | | | TODO |

---

## 16. 测试要求

| 层级 | 框架 | 范围 | 覆盖率目标 |
|-----|------|------|-----------|
| 单元 | | 业务逻辑、工具类 | |
| 集成 | Testcontainers | DB/缓存/MQ 真实联调 | |
| 契约 | Pact / schemathesis | 与前端契约 | 全部接口 |
| 状态机 | 单元 | 所有合法+非法转换 | 100% |

---

## 17. 部署与回滚

- 部署方式:<!-- K8s 蓝绿 / 滚动 -->
- DB schema 变更:先发兼容版本(读旧写新),再迁移
- 回滚:配置开关 + DB 兼容性
- 灰度:见需求 §13

---

## 18. 验收条件

后端开发完成的判定:

- [ ] 所有 AC 在 §15 表中标记完成
- [ ] 单元 + 集成测试通过,覆盖率达标
- [ ] 契约测试与前端通过
- [ ] 压测达到 §12 目标
- [ ] 安全扫描无 high 项
- [ ] 已部署到 dev,可联调

---

## 19. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | YYYY-MM-DD | | 初稿 |
