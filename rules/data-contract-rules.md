# 数据契约生成规则(Data Contract Rules)

> **本文件定义如何从需求文档生成 `data-contract.md`**。
>
> data-contract 是前端/后端/QA 三方协调的枢纽,生成质量直接决定下游闭环可行性。
> 本规则强调"机器可消费"——生成出来的内容应能被 OpenAPI 工具、TS codegen、契约测试工具解析。
>
> ⚠️ 本 agent 的产出**应由后端架构师 / Tech Lead 评审后再启用**,不建议完全自动放行。

继承:`global-rules.md`、`traceability-rules.md`

---

## 1. 输入契约(Inputs)

| 输入 | 用于生成本文档的哪一节 |
|-----|-------------------|
| `requirement.md §4` 实体清单 | §1 数据模型 |
| `requirement.md §5` 状态机 | §3 状态机定义 |
| `requirement.md §6` 业务流程 | §4 接口清单(每个流程节点 ≈ 一个 API) |
| `requirement.md §7` 功能需求 | §4 接口详细规范 |
| `requirement.md §8` AC | §3、§4 的 ac_refs 字段 |
| `requirement.md §9.1` 性能 | §7 constants(分页等) |
| `requirement.md §9.2` 安全 | §6 鉴权与权限 |
| `requirement.md §11` 依赖系统 | §5 异步任务(若涉及外部调用) |
| `requirement.md §3` 权限矩阵 | §6.2 权限校验点 |
| `glossary.md §5.2` 字段命名规范 | 全文字段命名 |
| `glossary.md §5.3` 枚举命名规范 | §2 枚举 |

---

## 2. 输出契约(Outputs)

生成的 `data-contract.md` 必须包含且仅包含以下章节(不允许删除/重排/重命名):

| 章节 | 必填? | 内容来源 |
|-----|------|---------|
| 0 front matter | ✅ | 模板 |
| 1 实体数据模型 | ✅(若 requirement §4 非空) | 转换 §4 |
| 2 枚举与字典 | ✅(若有枚举字段) | 从字段类型抽取 |
| 3 状态机定义 | ✅(若 requirement §5 非空) | 转换 §5,机器可解析 yaml |
| 4 API 接口契约 | ✅ | 主要内容 |
| 5 异步任务与事件 | ⭕(可空,但需保留章节) | 从功能需求识别异步项 |
| 6 鉴权与权限模型 | ✅ | 转换 §3 + §9.2 |
| 7 数据约束总览 | ✅ | 集中所有"魔法数字" |
| 8 契约校验机制 | ✅ | 模板内容,不需推导 |
| 9 变更历史 | ✅ | 首版填初稿 |

---

## 3. 转换规则

### 3.1 实体 → 数据模型(§1)

对 `requirement.md §4.1` 的每个实体,生成 `data-contract.md §1.X`:

| 输入元素 | 输出字段 | 推导规则 |
|---------|---------|---------|
| 实体名 | 章节标题 | 直接使用 |
| 实体 ID | 标记 `**对应实体**: ENT-XXX` | 双向引用 |
| 关键属性(业务语义) | 字段表行 | 每个属性 → 1 行,推断技术类型 |
| - | `id` 字段 | 强制添加,UUID 主键 |
| - | `created_at` / `updated_at` | 强制添加 |
| 提到"删除/归档" | `deleted_at` 字段 | 软删除时添加 |
| 提到"父子/归属" | 外键字段 + 索引 | 例如 `parent_id` |
| 业务唯一标识 | 唯一索引 | 如 `(parent_id, business_key)` |

**类型推断规则**:

| 业务描述含 | 推断类型 |
|----------|---------|
| 编号/序号/号 | `string` 或 `integer`(看是否含字母),长度上限引用 constants |
| 日期 | `date`(无时分秒) |
| 时间 | `timestamp`(ISO 8601) |
| 数量/计数 | `integer`,非负 |
| 金额/百分比 | `decimal(精度,标度)`,**禁止 float** |
| 长文本/说明 | `text`,长度上限引用 constants |
| 短文本/名称 | `varchar(N)`,N 引用 constants |
| 是否/可见性 | `boolean` |
| 列表(如多接收方) | `array<elem_type>` |
| 结构化嵌套 | `jsonb` 或子表(优先子表,避免 jsonb 难索引) |

**降级**:类型无法推断时,标注 `<!-- 需 TL 确认类型 -->` 后用 `string`。

### 3.2 状态机 → §3

直接转换为 yaml 块,字段名固定:

```yaml
state_machine: <实体名>_status
initial_state: <来自需求 §5.1 第一个非终态>
transitions:
  - id: T-001                         # 来自需求 §5.2 顺序编号
    from: <状态 ID>
    to: <状态 ID>
    action: <动作名,snake_case>      # 来自需求 §5.2 触发动作
    guard: <数组,来自守卫条件,每条一行>
    side_effects: <数组,来自副作用>
    ac_refs: [AC-XXX]                 # 反查所有 Given/When/Then 中提到该转换的 AC
forbidden_transitions:
  - from: <状态>
    to: "*"                           # 或具体状态
    reason: <来自需求 §5.3>
```

**guard 表达式规范**:
- 使用 `<左>` `==/>=/!=` `<右>` 简单表达式
- 多条件用数组(逻辑 AND);OR 用 `(A or B)` 写在单行
- 引用其他实体字段:`<实体>.<字段>`

### 3.3 业务流程 + 功能需求 → API(§4)

#### 3.3.1 API 识别启发式

从 `requirement.md §6.1` 流程步骤识别 API。每个"系统响应"步骤通常对应 1 个 API。

| 流程动作 | 推断 HTTP 方法 |
|---------|--------------|
| 创建/新增/上传/提交 | POST |
| 查询/列表/搜索/查看 | GET |
| 修改/编辑/更新部分字段 | PATCH |
| 替换整个资源 | PUT |
| 删除/归档/失效 | POST(归档建议 POST 而非 DELETE,便于审计) |
| 状态机操作(发布/取消/确认) | POST `/resource/{id}/{action}` |

**路径命名**:
- RESTful 资源用复数:`/api/orders`(不是 `/api/order`)
- 状态机动作:`/api/{resource}/{id}/{action}`,action 用 kebab-case
- 嵌套 ≤ 2 层,再深就拍平

#### 3.3.2 每个 API 必填字段

```yaml
# 必填项,缺一 fail
api_id: API-XXX
method: POST | GET | PATCH | PUT | DELETE
path: /api/...
purpose: <一句话描述>
idempotent: true | false           # 必须显式给值,不允许省略
auth_required: true | false        # 几乎都是 true
permission: <角色或所有者校验>      # 引用 ROLE-XXX
ac_refs: [AC-XXX]                  # 至少 1 个

request:
  headers: { ... }
  body / query / path: { ... }     # 字段类型引用 §1 的实体定义

response_success:
  status: 200 | 201 | 204
  body: { ... }

response_errors:
  - status: 400/401/403/404/409/429/500
    code: <UPPER_SNAKE>
    when: <触发条件,引用业务规则>

business_rules:
  - <用伪代码或自然语言>
```

#### 3.3.3 错误码生成规则

每个 API **必须**包含以下错误响应(若适用):

| 业务情况 | 强制错误码 |
|---------|---------|
| 鉴权失败 | 401 UNAUTHORIZED |
| 权限不足 | 403 FORBIDDEN |
| 资源不存在 | 404 NOT_FOUND |
| 唯一性冲突 | 409 {RESOURCE}_CONFLICT |
| 状态机非法转换 | 422 ILLEGAL_STATE_TRANSITION |
| 输入校验失败 | 400 INVALID_INPUT(细分子码) |
| 速率限制 | 429 RATE_LIMITED |
| 服务器错误 | 500 INTERNAL_ERROR |

错误码命名:**全大写 + 下划线**,语义化(如 `FILE_TOO_LARGE` 而非 `ERR_001`)。

#### 3.3.4 幂等性判定规则

| API 类型 | 默认幂等? |
|---------|---------|
| GET / HEAD | ✅ 自然幂等 |
| 创建类(POST 资源) | ❌ 默认不幂等,**必须**用 `Idempotency-Key` header 实现 |
| 状态机动作(POST `/x/{id}/action`) | ✅ 必须幂等 |
| 文件上传 | ✅ 基于 hash 幂等 |
| 删除/归档 | ✅ 必须幂等(再次删已删的不报错) |
| 修改类(PATCH/PUT) | ✅ 必须幂等(同样输入 → 同样状态) |

**任何不幂等的 API 必须显式标注理由**。

### 3.4 性能/分页/限流 → §7 constants

从 `requirement.md §9.1` 抽取所有数值,集中放入 §7:

```yaml
constants:
  rate_limit:
    api_per_minute: <数值>
    <action>_per_minute: <数值>
  pagination:
    default_page_size: 20
    max_page_size: 100
  text_length:
    <field>_max: <数值>
  file:
    max_size_bytes: <数值>
  retention:
    <data_type>_days: <数值>
```

### 3.5 角色权限 → §6.2

对 `requirement.md §3` 的权限矩阵,反向构造"每个 API 谁可以调用":

| 矩阵中的操作 | 推断为 API | 权限 |
|-----------|----------|-----|
| "查看 X" | GET 类 API | 校验团队/项目成员身份 |
| "创建/上传 X" | POST 类 API | 校验角色 ∈ 允许的角色集合 |
| 其他动作 | 状态机动作 API | 校验角色 + 资源所有权 |

输出格式:

```markdown
| API ID | 校验类型 | 校验逻辑 |
|--------|---------|---------|
| API-001 | 角色 | user.role IN [ROLE-XXX, ROLE-YYY] |
| API-002 | 角色+所有权 | role OK AND resource.owner == user |
```

---

## 4. 异步任务识别(§5)

从 `requirement.md` 识别异步任务的信号:

| 关键词 | 推断为异步 |
|-------|---------|
| "通知/推送/发送" | ✅ 走 MQ |
| "异步/后台/调度" | ✅ |
| "解析/识别/转换" | ✅ 长耗时计算 |
| "缩略图/预览生成" | ✅ |
| "导出/批量处理" | ✅ |
| "对接外部系统" | ✅ 隔离失败 |

每个异步任务输出:

```yaml
- task_id: TASK-XXX
  trigger: <事件名>
  description: <做什么>
  retry_policy: <重试策略,默认 3 次指数退避>
  failure_handling: <死信队列 + 告警>
  sla: <时间预期>
  ac_refs: [AC-XXX]
```

---

## 5. 降级与待定

| 情况 | 处理 |
|-----|------|
| 实体无字段说明,只有名字 | 仅输出 id/created_at/updated_at,标注 `<!-- TODO: 字段待 PM 补充 -->` |
| 状态机不完整(只列状态没列转换) | 输出状态枚举,转换部分标注 TODO |
| 业务规则模糊("看情况") | 不编造逻辑,标注 OQ |
| API 错误响应未指定 | 用 §3.3.3 默认错误码,标注 `<!-- 默认错误码,请 review -->` |
| 类型无法推断 | string + TODO |
| 性能数值未指定 | constants 中标注 `<!-- 默认值,需压测后调整 -->` |

---

## 6. 校验清单(自检)

- [ ] 每个实体在 §1 有完整字段定义(至少 id/created_at/updated_at)
- [ ] 每个枚举字段在 §2 有枚举值定义
- [ ] 每个状态在 §3 出现,每个转换有 ac_refs
- [ ] 每个 API 有完整的 method/path/idempotent/auth/permission/ac_refs
- [ ] 每个 API 至少有 401/403/500 错误响应
- [ ] 每个 POST 创建类 API 有 Idempotency-Key 处理
- [ ] §6.2 权限校验点覆盖所有 API
- [ ] §7 constants 覆盖所有"具体数值"
- [ ] 所有字段命名遵循 glossary.md §5.2
- [ ] 所有枚举命名遵循 glossary.md §5.3
- [ ] 没有重复定义(同一字段在多个实体下重复)
- [ ] 至少 1 个 transition 引用了 ac_refs(否则状态机和 AC 失联)

---

## 7. 输出后置动作

agent 完成生成后:

1. 输出文件至 `requirements/{REQ_ID}/data-contract.md`
2. 同步导出 OpenAPI yaml 至 `schemas/{REQ_ID}/openapi.yaml`(若有 schema 工具)
3. 在终端打印生成摘要:实体数、API 数、状态机数、未解决 OQ 数

---

## 8. 不要做的事

- ❌ 凭空发明字段(只因"通常会有")
- ❌ 用 `int` 表示金额
- ❌ 用 `string` 表示枚举(必须是明确枚举类型)
- ❌ 接口路径用 camelCase
- ❌ 在错误响应里返回不同 schema(成功是 `{data}`,错误是 `{msg}` 这种不一致)
- ❌ 把鉴权信息放在 query string
- ❌ 在 GET 接口里写 body
- ❌ 让一个 API "既创建又更新"(应拆 POST 和 PATCH)
