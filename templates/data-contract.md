---
doc_type: data_contract
req_id: REQ-XXXX                    # 关联需求 ID
version: 0.1.0
status: draft
generated_from: requirement.md@0.1.0  # 溯源:从哪个需求文档版本生成
generated_at: YYYY-MM-DD
owner: ""                           # 通常是后端架构师 / Tech Lead
---

# 数据契约文档:[需求标题]

> **本文档是前后端联调与 QA 接口测试的单一事实源**。
>
> - 前端 agent 从本文档生成 TS 类型、API client、mock。
> - 后端 agent 从本文档生成 controller 骨架、DTO、参数校验。
> - QA agent 从本文档生成接口测试用例与契约测试。
>
> ⚠️ **任何字段定义只允许在本文档出现一次。** 其他文档(frontend-spec、backend-spec)只允许引用,不允许重复定义。

---

## 1. 实体数据模型(Data Models)

> 对应需求文档第 4 节的实体清单。

### 1.1 [实体名]

**对应实体**: ENT-XXX

| 字段名 | 类型 | 必填 | 默认值 | 约束 | 描述 |
|-------|------|-----|-------|------|------|
| id | string(uuid) | ✅ | - | 主键 | 系统生成 |
| | | | | | |
| created_at | timestamp | ✅ | now() | ISO 8601 | 创建时间 |
| updated_at | timestamp | ✅ | now() | ISO 8601 | 更新时间 |

**索引**:
- 主键:`id`
- 唯一索引:<!-- 例如 (parent_id, name) -->
- 普通索引:<!-- 例如 (status, created_at) -->

**软删除**: 是 / 否(若是,加字段 `deleted_at`)

---

### 1.2 [实体名]

<!-- 按相同结构展开 -->

---

## 2. 枚举与字典(Enums)

> 所有枚举值集中定义,前后端 QA 共用。

### 2.1 [枚举名]

| 值 | 显示名(zh-CN) | 显示名(en) | 描述 |
|---|--------------|-----------|------|
| VALUE_A | | | |
| VALUE_B | | | |

---

## 3. 状态机定义(State Machines)

> 对应需求文档 §5,在此用机器可解析格式定义。

### 3.1 [实体名] 状态机

```yaml
state_machine: [实体名]_status
initial_state: [初始状态]

transitions:
  - id: T-001
    from: [状态 A]
    to: [状态 B]
    action: [动作名]
    guard:
      - [前置条件 1]
      - [前置条件 2]
    side_effects:
      - [副作用 1]
      - [副作用 2]
    ac_refs: [AC-XXXX-001]

forbidden_transitions:
  - from: [状态 X]
    to: "*"
    reason: "[禁止原因]"
```

---

## 4. API 接口契约(REST)

> 采用 OpenAPI 风格简化描述。生产时建议导出为 OpenAPI 3.0 yaml。

### 4.1 接口清单(目录)

| API ID | 方法 | 路径 | 用途 | 关联 AC |
|--------|------|------|------|--------|
| API-001 | | | | |
| API-002 | | | | |

---

### 4.2 接口详细规范模板

#### API-XXX:[接口名]

**用途**: <!-- 一句话描述 -->

**幂等性**: ✅ 是 / ❌ 否(必须明确)
**鉴权**: ✅ 是 / ❌ 否
**权限**: <!-- 哪些角色可调用 -->

**请求**:

```
[METHOD] [/path]
Headers:
  [必要 header,如 Idempotency-Key]
Body / Query / Path Params:
  - [字段]: 类型, 必填?, 约束
```

**响应:成功**

```json
{
  "...": "..."
}
```

**响应:错误**

| HTTP | 错误码 | 含义 | 触发条件 |
|------|-------|------|---------|
| 400 | [CODE] | | |
| 401 | UNAUTHORIZED | 未登录 | - |
| 403 | FORBIDDEN | 无权限 | |
| 409 | [CODE] | | |
| 429 | RATE_LIMITED | 频率限制 | |
| 500 | INTERNAL_ERROR | 服务器错误 | - |

**统一错误响应格式**:

```json
{
  "error": {
    "code": "[ERROR_CODE]",
    "message": "[用户可见提示]",
    "details": { },
    "request_id": "req-..."
  }
}
```

**业务校验**:
<!-- 列出后端必须做的业务校验,例如唯一性、引用完整性、事务边界 -->

---

<!-- 其余 API 按相同结构展开 -->

---

## 5. 异步任务与事件(Async / Events)

> 哪些操作是异步的、走消息队列的、需要补偿的。

| 任务 ID | 触发时机 | 处理逻辑 | 失败重试 | 关联 AC |
|--------|---------|---------|---------|--------|
| TASK-001 | | | | |

### 事件总线消息格式

```json
{
  "event_type": "[domain].[action]",
  "event_id": "evt-uuid",
  "occurred_at": "ISO 8601",
  "payload": {
    "...": "..."
  },
  "trace_id": "trace-..."
}
```

---

## 6. 鉴权与权限模型

### 6.1 鉴权

- 方式:<!-- JWT / Session / OAuth2 -->
- Token 来源:
- Token 过期:
- Refresh:支持 / 不支持

### 6.2 权限校验点(Permission Checks)

> 后端 agent 必须在每个接口实现权限校验。本表是清单。

| API ID | 校验类型 | 校验逻辑 |
|--------|---------|---------|
| API-001 | <!-- 角色 / 资源所有者 / 团队成员 --> | |

---

## 7. 数据约束总览

> 集中列出所有"魔法数字",方便前后端 QA 共享常量。

```yaml
constants:
  rate_limit:
    api_per_minute: 600
  pagination:
    default_page_size: 20
    max_page_size: 100
  text_length:
    # 字段长度上限
  retention:
    # 数据保留期限
```

---

## 8. 契约校验机制

> 用于前后端联调和 QA 自动化测试的对齐机制。

- **契约文件输出**: 本文档维护人在 PR 合并时,需同步导出 OpenAPI yaml 至 `schemas/openapi.yaml`
- **前端校验**: 前端构建时基于 OpenAPI 生成 TS 类型,类型不匹配则编译失败
- **后端校验**: 后端启动时校验路由与 OpenAPI 一致,不一致则启动失败
- **契约测试**: QA 在 CI 中运行 schemathesis / dredd 类工具,验证实际接口符合契约

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 | Breaking? |
|-----|------|-------|---------|-----------|
| 0.1.0 | YYYY-MM-DD | | 初稿 | - |

> ⚠️ **Breaking Change 规则**: 字段重命名、删除、类型变更、必填变化均为 breaking,必须提升 major 版本(1.0.0 → 2.0.0),并通知所有下游。
