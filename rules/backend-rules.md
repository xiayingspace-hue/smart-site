# 后端开发说明生成规则(Backend Rules)

> **本文件定义如何从需求/契约生成 `backend-spec.md`**。
>
> 后端 agent 的工作:把 data-contract 翻译成"业务逻辑 + 状态机实现 + 事务/幂等/审计/并发"决策。
> 这一份是 4 份角色 rules 中**信息密度最高**的,因为后端决策面最广,踩坑面最大。

继承:`global-rules.md`、`traceability-rules.md`

---

## 1. 输入契约

| 输入 | 用于哪一节 |
|-----|---------|
| `requirement.md §3` 权限矩阵 | §13.2 鉴权权限 |
| `requirement.md §5` 状态机 | §4 状态机实现 |
| `requirement.md §7` 功能需求 | §3 业务逻辑 |
| `requirement.md §8` AC | §15 AC 覆盖 |
| `requirement.md §9.1` 性能 | §12 性能要求 |
| `requirement.md §9.2` 安全 | §13 安全要求 |
| `requirement.md §10` 数据量级 | §8.1 索引、§9 缓存策略 |
| `requirement.md §11` 依赖系统 | §2.2 新增依赖 |
| `requirement.md §13` 灰度 | §17 部署回滚 |
| `data-contract.md §1` 数据模型 | §8 数据库设计(只列建表/索引要点) |
| `data-contract.md §3` 状态机 | §4 状态机实现 |
| `data-contract.md §4` API | §3 业务逻辑(每个 API → 1 个业务点) |
| `data-contract.md §5` 异步任务 | §7 任务执行规范 |
| `data-contract.md §6` 鉴权权限 | §13.2 |
| `data-contract.md §7` constants | 引用,不重复 |

---

## 2. 输出章节

| 章节 | 必填? |
|-----|------|
| 0 溯源块 | ✅ |
| 1 功能概述 | ✅ |
| 2 技术栈 | ✅ |
| 3 业务逻辑 | ✅ |
| 4 状态机实现 | ✅(若有状态机) |
| 5 事务边界 | ✅ |
| 6 幂等性要求 | ✅ |
| 7 异步任务与事件 | ⭕(若有) |
| 8 数据库设计 | ✅ |
| 9 缓存策略 | ✅ |
| 10 并发控制 | ✅ |
| 11 审计日志 | ✅ |
| 12 性能要求 | ✅ |
| 13 安全要求 | ✅ |
| 14 可观测性 | ✅ |
| 15 AC 覆盖检查表 | ✅ |
| 16 测试要求 | ✅ |
| 17 部署与回滚 | ⭕ |
| 18 验收条件 | ✅ |
| 19 变更历史 | ✅ |

---

## 3. 转换规则

### 3.1 业务逻辑(§3)

对 `data-contract.md §4` 每个 API,生成 1 个 §3.X 业务逻辑小节。

**模板**:

```markdown
### 3.X [业务点名](覆盖 AC-XXX-XXX)

[在事务内 / 不在事务内]
1. 校验权限:[引用 data-contract §6.2 的校验规则]
2. 校验输入:[必填、范围、类型]
3. 业务校验:[唯一性、引用完整性、状态机前置]
4. 持久化:[操作哪些表]
5. 副作用:[审计、事件、外部调用]
[事务提交]
6. 异步动作:[发 MQ、调外部 API]
```

#### 3.1.1 必填决策

每个业务点必须显式回答 6 个问题(缺一即不合格):

| 决策项 | 问题 |
|-------|------|
| 事务边界 | 哪些步骤必须原子? |
| 幂等性 | 重复调用是否产生不同结果? |
| 审计 | 是否需要留痕? |
| 并发控制 | 是否会冲突?用什么锁? |
| 一致性模型 | 强一致还是最终一致? |
| 失败补偿 | 失败如何回滚 / 补偿? |

### 3.2 状态机实现(§4)

#### 3.2.1 强制约束

- 所有状态变更必须通过 `{Entity}StateMachine.transition(entity, action)` 入口
- 业务代码**不允许**直接 `entity.status = X`
- 非法转换必须抛 `IllegalStateTransitionException`,接口层映射 422

#### 3.2.2 实现伪代码(必须输出)

```pseudo
function transition(entity, action):
  matched = find_transition(entity.status, action)
  if not matched: throw IllegalAction
  for guard in matched.guard:
    if not evaluate(guard, entity): throw GuardFailed(guard)
  acquire_lock(entity.id)
  entity.status = matched.to
  entity.updated_at = now()
  entity.save()
  for effect in matched.side_effects:
    schedule(effect)        # 走 outbox,见 §7.3
  emit_event(entity_type + '.' + action)
  release_lock()
```

#### 3.2.3 状态字段持久化

- 状态字段为 varchar(20~50),存枚举值字符串(不存 int)
- 索引:`(parent_id, status)` 用于按状态过滤
- 不允许在 DB 里存"状态历史",历史走 audit_logs

### 3.3 事务边界(§5)

#### 3.3.1 必须同一事务

| 场景 | 理由 |
|-----|------|
| 父子实体同时创建 | 否则有"父无子"或"子无父"脏数据 |
| 状态变更 + 关联实体更新 | 一致性 |
| 状态变更 + 审计写入 | 审计必须可信 |
| 唯一约束相关的多表写入 | 否则约束失效 |

#### 3.3.2 必须拆开

| 场景 | 理由 |
|-----|------|
| 业务写入 + 外部 API 调用 | 外部调用慢、可能失败 |
| 业务写入 + MQ 发布 | 用 outbox 模式而非直接发,见 §7.3 |
| 业务写入 + 缓存更新 | 缓存失败不应回滚业务 |
| 业务写入 + 文件 I/O | I/O 慢 |

### 3.4 幂等性(§6)

对 `data-contract.md §4` 每个 API:

| API 类型 | 幂等策略推断 |
|---------|----------|
| GET / HEAD | 自然幂等,不需额外处理 |
| POST 创建类 | 必须有 Idempotency-Key header,Redis 存 24h |
| POST 状态机动作 | 用唯一约束(action_log 表 + 唯一键) |
| 文件上传 | 基于 file_hash 去重 |
| 删除/归档 | 检查当前状态是否已为目标状态 → 直接返回成功 |
| PATCH/PUT | 同样输入应产生同样最终状态 |

输出格式:

```markdown
| API | 幂等键 | 存储 | 过期 |
|-----|-------|------|-----|
| API-001 | `Idempotency-Key` header | Redis | 24h |
| API-005 | `(parent_id, action, action_id)` 唯一约束 | DB | 永久 |
```

**幂等响应一致性**:重复请求**必须返回与首次完全相同**的响应(包括 status code 和 body)。

### 3.5 异步任务与事件(§7)

#### 3.5.1 任务规范

对 `data-contract.md §5` 每个任务,生成:

```markdown
| 任务 ID | 队列 | 重试策略 | 幂等性 | 死信处理 | SLA |
|--------|------|---------|------|---------|-----|
| TASK-XXX | `xxx-q` | 3 次,指数退避(1m/5m/30m) | 必须 | DLQ + 告警 | < 5min |
```

**强制规则**:消费者必须幂等,因为消息中间件普遍是"至少一次"语义。

#### 3.5.2 Outbox 模式(强制)

凡是"业务写入 + 事件发布"的场景,**必须**用 outbox:

```pseudo
[业务事务内]
1. 业务写入
2. INSERT outbox_events(event_type, payload, status='PENDING')
[事务提交]

[独立调度器,每秒扫描]
3. SELECT * FROM outbox_events WHERE status='PENDING' LIMIT 100
4. 发到 MQ
5. UPDATE status='SENT', sent_at=NOW()
6. 失败:retry_count++,达上限转 FAILED + 告警
```

理由:防止"业务成功但事件丢失"或"事件发了但业务回滚"。

#### 3.5.3 事件命名

格式:`{domain}.{action}`,小写点分,如 `order.published`、`payment.refunded`。

### 3.6 数据库设计(§8)

#### 3.6.1 建表语句生成规则

不重复 data-contract §1 的字段定义,只输出:

- 表创建 SQL(主键 + 约束)
- 索引 SQL
- 数据库迁移文件命名

```sql
CREATE TABLE {table} (
  id            UUID PRIMARY KEY,
  -- 字段引用 data-contract §1.X
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at    TIMESTAMPTZ,
  CONSTRAINT uk_xxx UNIQUE (...)
);

CREATE INDEX CONCURRENTLY idx_xxx ON {table}(...) WHERE deleted_at IS NULL;
```

#### 3.6.2 索引推断规则

| 来源 | 索引 |
|-----|------|
| data-contract §1 标注的"唯一" | UNIQUE INDEX |
| data-contract §1 标注的"外键" | 单列索引 |
| 列表查询常用筛选字段 | 复合索引(scope_id + filter_field) |
| 排序字段 | 复合索引最后一列 |
| 软删除 | 索引加 `WHERE deleted_at IS NULL` |
| 大表(> 100w 行预期) | 全部加 `CONCURRENTLY`(PG)防锁 |

#### 3.6.3 迁移规则

- 工具:Flyway / Liquibase / Alembic(按团队)
- 命名:`V{时间戳}__{描述}.sql`
- **只前进,不回退**
- 新增列:必须有默认值或允许 NULL
- 删除列:先发"忽略该列"的版本,后续版本再删
- 重命名:先加新列双写,迁移完删旧列

### 3.7 缓存策略(§9)

#### 3.7.1 决策规则

| 数据特征 | 是否缓存 | TTL |
|---------|--------|-----|
| 高频读、低频写 | ✅ | 30s ~ 5min |
| 强一致要求 | ❌ | - |
| 用户隔离的列表 | ✅(按 user 分片 key) | 60s |
| 全局配置 | ✅ | 5min |
| 实时数据 | ❌ | - |

#### 3.7.2 必备防护

| 风险 | 防护 |
|-----|------|
| 缓存击穿(热 key 过期) | mutex 锁防止并发回源 |
| 缓存穿透(查不存在数据) | 缓存空值 60s |
| 缓存雪崩(大量同时过期) | TTL 加随机抖动 ±10% |
| 缓存不一致 | 主动失效 + TTL 双保险 |

### 3.8 并发控制(§10)

| 场景特征 | 推荐策略 |
|--------|---------|
| 同一资源短时间内最多变更几次 | 乐观锁(version 字段) |
| 高频并发同一行 | 悲观锁(SELECT FOR UPDATE) |
| 跨资源协调 | 分布式锁(Redis / 数据库咨询锁) |
| 唯一性保证 | DB 唯一约束(最强、最简单) |
| 状态机转换 | 行级锁 + 乐观版本号 |

**默认偏好**:DB 唯一约束 > 乐观锁 > 悲观锁 > 分布式锁。复杂度低优先。

### 3.9 审计日志(§11)

#### 3.9.1 强制审计的操作

| 操作类型 | 强制审计 |
|---------|---------|
| 创建/修改业务实体 | ✅ |
| 状态变更 | ✅ |
| 权限相关(分享、转移) | ✅ |
| 删除/归档 | ✅ |
| 大批量操作 | ✅ |
| 跨用户/跨组织操作 | ✅ |
| 普通查询 | ❌(量太大) |
| 高敏数据查询 | ✅ |

#### 3.9.2 审计字段(强制)

```sql
CREATE TABLE audit_logs (
  id            BIGSERIAL PRIMARY KEY,
  occurred_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  actor_id      UUID NOT NULL,
  actor_role    VARCHAR(50),
  action        VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50),
  resource_id   UUID,
  payload       JSONB,                 -- 变更前后差异
  ip            INET,
  user_agent    TEXT,
  request_id    VARCHAR(64),
  trace_id      VARCHAR(64)
);
CREATE INDEX ON audit_logs (resource_type, resource_id);
CREATE INDEX ON audit_logs (actor_id, occurred_at DESC);
```

#### 3.9.3 写入时机

- 同一事务写入(防业务成功审计丢失)
- 异步写入(若审计 DB 独立)→ 用 outbox

### 3.10 性能(§12)

#### 3.10.1 默认指标

若 requirement §9.1 未指定:

| 接口类型 | P95 默认目标 |
|---------|-----------|
| 列表查询 | < 200ms |
| 详情查询 | < 100ms |
| 创建/更新 | < 300ms |
| 复杂业务 | < 500ms |
| 文件上传(per MB) | < 50ms |

#### 3.10.2 必备优化

| 触发条件 | 优化措施 |
|--------|---------|
| 任何列表查询 | 必须有覆盖索引或限制扫描行数 |
| 任何 N+1 风险点 | 批查询(IN 或 JOIN) |
| 大字段 | 列表查询不返回(详情才返回) |
| 计数查询 | 维护 counter 而非 COUNT(*) |
| 分页深度 | 用 cursor 而非 OFFSET |

### 3.11 安全(§13)

#### 3.11.1 输入校验

- 所有外部输入走 DTO + 校验框架(Bean Validation / pydantic)
- SQL 必须参数化
- 文件名 / 路径必须校验,防穿越

#### 3.11.2 权限校验点

引用 `data-contract.md §6.2`,**每个接口必做**:

| 校验类型 | 实现位置 |
|---------|---------|
| 鉴权(已登录) | 中间件 |
| 角色 | 注解 / 装饰器(`@RequireRole`) |
| 资源所有权 | Service 层(查询时强制注入归属过滤) |
| 行级权限 | 在 DAO 层注入,而非 Controller |

#### 3.11.3 限流

引用 `data-contract.md §7` constants:

| 维度 | 实现 |
|-----|------|
| 用户级全局 | Redis 滑动窗口 |
| 接口级 | 注解 / 装饰器 |
| IP 级(防爬) | 网关层 |

### 3.12 可观测性(§14)

#### 3.12.1 必含字段

每条日志:`trace_id`, `request_id`, `user_id`, 业务关键 ID。

#### 3.12.2 metrics 自动推断

| 来源 | 生成 metric |
|-----|-----------|
| 每个 API | qps、延迟分位、错误率 |
| 每个状态机 | 状态转换计数(by from-to) |
| 每个异步任务 | 队列深度、消费延迟、重试次数 |
| requirement §14 成功指标 | 业务计数器 |

#### 3.12.3 告警规则(默认)

| 触发 | 等级 |
|-----|------|
| 接口 5xx 率 > 1% 持续 5min | P1 |
| 核心接口 P99 > 阈值 持续 5min | P2 |
| MQ 队列堆积 > N | P1 |
| outbox PENDING 数 > N | P1(事件丢失风险) |
| DB 连接池满 | P1 |
| 慢查询 > 1s 持续 1min | P2 |

### 3.13 测试(§16)

> 团队当前未强制要求单元测试，以下为建议项，由开发自行评估引入。

| 层级 | 建议程度 | 说明 |
|-----|---------|------|
| 单元 | 建议（非强制） | 状态机转换逻辑、复杂业务规则优先考虑 |
| 集成 | 建议（非强制） | 核心接口的完整业务流 |
| 契约 | 暂不要求 | 待前后端协作模式稳定后再引入 |
| 性能 | 建议（非强制） | 覆盖 §12 指标中的关键接口 |

---

## 4. AC 覆盖反查(§15)

```markdown
| AC ID | 实现位置 | 测试 | 状态 |
|------|---------|-----|------|
| AC-XXX-001 | `OrderService.create()` | OrderServiceTest | TODO |
| AC-XXX-007 | DB 唯一约束 + Service 校验 | integration test | TODO |
```

---

## 5. 降级与待定

| 情况 | 处理 |
|-----|------|
| 技术栈未指定 | 用团队默认,标注"待 TL 确认" |
| 性能未指定 | 用 §3.10.1 默认 |
| 异步策略未指定 | 默认非关键路径走 outbox + MQ |
| 缓存策略未指定 | 默认不缓存,标注"如有性能问题再加" |
| 审计需求未指定 | 默认核心写操作全审计 |
| 数据保留期未指定 | 标注 OQ,等待法务/PM 确认 |

---

## 6. 校验清单(自检)

- [ ] 每个 API 有对应 §3.X 业务逻辑小节
- [ ] 每个业务点显式回答 6 个决策问题(§3.1.1)
- [ ] 状态机有伪代码 + 强制约束声明
- [ ] §5 事务边界覆盖所有跨表操作
- [ ] §6 幂等表覆盖所有 POST/PATCH/DELETE
- [ ] §7 异步任务有重试 + 死信 + SLA
- [ ] 涉及业务事件 → 有 outbox 设计
- [ ] §8 每张表有索引推断
- [ ] §11 每个核心写操作有审计
- [ ] §13 每个接口有权限校验点
- [ ] §14 metrics 与 §15 AC 关联
- [ ] AC 覆盖表全部处理
- [ ] 没有重复定义字段(必须引用 data-contract)

---

## 7. 不要做的事

- ❌ 在 backend-spec 重新定义字段(必须引用 data-contract)
- ❌ 让业务代码直接改 status 字段(必须走状态机入口)
- ❌ 把 MQ 发布写在业务事务里(应走 outbox)
- ❌ 跳过幂等设计(POST 创建类不带 Idempotency-Key)
- ❌ 用 application 层事务跨多个微服务(应改成 Saga)
- ❌ 在 Controller 写权限校验逻辑(应在 Service / DAO 层)
- ❌ 给所有读接口加缓存(强一致场景不该缓存)
- ❌ 用 OFFSET 分页深翻(用 cursor)
- ❌ 在审计日志里只存"做了什么",不存"做了之后值是什么"(无法回溯)
- ❌ 为每个微服务单独搞限流(应在 API 网关统一)

---

## 8. 接口路径规范

### 8.1 路径结构

所有接口路径遵循以下固定格式：

```
https://smartsite.mcc.sg/{网关前缀}/{服务}/{模块}[/{资源}]/{动作}
```

> 资源层级可选，层级数量由业务复杂度决定，通常为 3～4 段（不含网关前缀）。

**各段说明**：

| 段 | 说明 | 示例 |
|---|------|------|
| 网关前缀 | 固定值，区分管理端 / 用户端 | `admin-api` / `app-api` |
| 服务 | 后端服务名，全小写，见 §8.4 服务清单 | `system` / `staff` / `worker` |
| 模块 | 服务内的业务模块，小驼峰 | `drawing` / `planning` / `equipment` |
| 资源（可选）| 模块下的子资源，小驼峰，可多级 | `processSetting` / `menu` |
| 动作 | 操作动词，小驼峰 | `page` / `get` / `create` |

**示例**：
```
GET    /admin-api/system/user/menu/getList
GET    /admin-api/staff/drawing/page
GET    /admin-api/staff/planning/processSetting/page
POST   /admin-api/worker/equipment/create
PUT    /admin-api/base/alarm/rule/update
DELETE /admin-api/system/role/delete
```

### 8.2 命名规则

- **网关前缀**：`admin-api`（管理后台）、`app-api`（移动端/用户端）
- **服务**：全小写，取服务注册名去掉 `blade-` / `mcc-` 前缀，见 §8.4
- **模块 / 资源**：小驼峰
- **动作**：小驼峰，使用以下标准动词：

| 动作 | HTTP 方法 | 说明 |
|------|---------|------|
| `page` | GET | 分页查询 |
| `getList` | GET | 全量列表查询（不分页） |
| `get` | GET | 单条查询 |
| `create` | POST | 新建 |
| `update` | PUT | 全量更新 |
| `updatePartial` | PATCH | 部分更新 |
| `delete` | DELETE | 删除 |
| `[业务动词]` | POST | 业务操作，如 `enable` / `disable` / `import` / `export` |


### 8.4 服务清单

服务注册名与路径段的对应关系（去掉 `blade-` / `mcc-` 前缀）：

| 服务注册名 | 路径段 | 说明 |
|-----------|-------|------|
| `blade-system` | `system` | 系统管理（用户、角色、菜单、权限） |
| `blade-user` | `user` | 用户服务 |
| `blade-auth` | `auth` | 认证授权 |
| `blade-admin` | `admin` | 管理端基础服务 |
| `blade-gateway` | `gateway` | 网关 |
| `blade-resource` | `resource` | 资源管理（文件、附件） |
| `blade-report` | `report` | 报表 |
| `blade-desk` | `desk` | 工作台 / 通知 |
| `blade-flow` | `flow` | 工作流 |
| `blade-xxljob` | `xxljob` | 定时任务 |
| `mcc-staff` | `staff` | 员工 / 项目业务 |
| `mcc-base` | `base` | 基础业务数据 |
| `mcc-worker` | `worker` | 作业 / 设备业务 |
| `mcc-open` | `open` | 开放平台 / 对外接口 |

> 新增服务时，在此表追加，路径段命名遵循：取注册名去掉前缀，全小写，单词间不加连字符。

### 8.3 在 data-contract.md 中的写法

生成 API 定义时，路径必须完整包含所有层级，不得省略网关前缀：

```yaml
api:
  - id: API-001
    method: GET
    path: /admin-api/project/drawing/page
    summary: 分页查询图纸列表
```

