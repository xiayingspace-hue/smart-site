---
doc_type: backend_spec
req_id: REQ-007D-pc
version: 0.1.0
status: draft
generated_from: REQ-007D-pc.md@0.1.0
generated_at: 2026-05-27
owner: ""
---

# 后端开发说明 — PC 端 DC 配置页面

> **来源需求**: [REQ-007D-pc.md](../../requirements/pc/REQ-007D-pc.md) @ v0.1.0
> **依赖后端文档**: [BACKEND-REQ-007.md](./BACKEND-REQ-007.md)（图纸两级审批基础后端）
> **产品**: SMART SITE SYSTEM
> **服务模块**: `drawing-service`
> **生成日期**: 2026-05-27

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007D-pc @ v0.1.0 |
| 覆盖 Story | US-007D-001 |
| 覆盖 AC | AC-007D-001 ~ AC-007D-010 |

---

## 1. 功能概述

本文档聚焦 DC 配置功能的后端实现：

| 功能 | 说明 | 覆盖 AC |
|------|------|---------|
| 查询 DC 列表 | 返回当前项目已配置的 DC 人员 | AC-007D-002 |
| 查询 Available Users | 返回有 `drawing:external-approval` 权限且未配置为 DC 的用户 | AC-007D-003 |
| 添加 DC | 单条添加，立即生效，后续内部审批通过时自动通知 | AC-007D-004 / 005 |
| 移除 DC | 软删除（或硬删除），历史审批记录保留 | AC-007D-007 / 008 |
| 无 DC 兜底校验 | 内部审批通过时后端校验项目是否有有效 DC | AC-007D-010（集成） |

---

## 2. 技术栈

### 2.1 已有技术栈（继承自 BACKEND-REQ-007）

沿用 BACKEND-REQ-007 全部技术约定（语言、框架、数据库、缓存、消息队列等）。

### 2.2 本需求新增依赖

无新增外部依赖。

---

## 3. 数据模型

### 3.1 新增表：`project_dc_config`

```sql
CREATE TABLE project_dc_config (
  id            VARCHAR(36)  NOT NULL PRIMARY KEY,   -- UUID
  project_id    VARCHAR(36)  NOT NULL,
  dc_user_id    VARCHAR(36)  NOT NULL,
  dc_user_name  VARCHAR(100) NOT NULL,               -- 冗余存储，减少联查
  configured_by VARCHAR(36)  NOT NULL,               -- 操作人 user_id
  configured_by_name VARCHAR(100) NOT NULL,          -- 操作人姓名（冗余）
  configured_at TIMESTAMP    NOT NULL DEFAULT NOW(),
  deleted_at    TIMESTAMP    NULL,                   -- 软删除标记
  UNIQUE KEY uq_project_dc (project_id, dc_user_id, deleted_at)
);
CREATE INDEX idx_pdc_project ON project_dc_config(project_id);
```

> **软删除说明**：`deleted_at IS NULL` 表示有效配置；移除 DC 时设置 `deleted_at = NOW()`，历史审批任务记录（`drawing_approval`）通过 `dc_user_id` 关联，不受影响。

---

## 4. API 接口定义

### 4.1 `GET /project/{projectId}/dc-config` — 查询已配置 DC 列表

**覆盖 AC**: AC-007D-002

**鉴权**: JWT，需携带 `Authorization` / `X-Tenant-Id` / `Project-Id`

**权限**: `drawing:dc-config`（无权限返回 403）

**覆盖 AC**: AC-007D-001（无权限时菜单隐藏 + URL 直访返回 403）

**响应字段**（数组，每条 DcConfigItem）:

| 字段 | 类型 | 来源 | 说明 |
|-----|------|------|------|
| `id` | string | project_dc_config.id | 配置记录 ID |
| `dcUserId` | string | project_dc_config.dc_user_id | DC 用户 ID |
| `dcUserName` | string | project_dc_config.dc_user_name | DC 姓名 |
| `configuredByName` | string | project_dc_config.configured_by_name | 操作人姓名 |
| `configuredAt` | string | project_dc_config.configured_at | ISO 8601，格式 `YYYY-MM-DD` |

**SQL 核心逻辑**:

```sql
SELECT id, dc_user_id, dc_user_name, configured_by_name, configured_at
FROM project_dc_config
WHERE project_id = :projectId
  AND deleted_at IS NULL
ORDER BY configured_at ASC;
```

**空列表**：返回 `{ "data": [] }`，HTTP 200。

---

### 4.2 `GET /project/{projectId}/dc-config/available` — 查询可添加用户列表

**覆盖 AC**: AC-007D-003

**权限**: `drawing:dc-config`

**查询参数**:

| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| `keyword` | string | 否 | 姓名模糊搜索，为空时返回全部 |

**过滤逻辑**:

1. 查询该项目中拥有 `drawing:external-approval` 权限的所有用户（来自权限系统）
2. 排除已在 `project_dc_config` 中有效配置（`deleted_at IS NULL`）的用户
3. 若 `keyword` 非空，按 `LIKE %keyword%` 过滤姓名

**响应字段**（数组，每条 UserItem）:

| 字段 | 类型 | 说明 |
|-----|------|------|
| `userId` | string | 用户 ID |
| `userName` | string | 用户姓名 |

**空列表**：返回 `{ "data": [] }`，HTTP 200（前端展示 "All eligible users have been configured as DC."）。

---

### 4.3 `POST /project/{projectId}/dc-config` — 添加 DC

**覆盖 AC**: AC-007D-004 / AC-007D-005

**权限**: `drawing:dc-config`

**Request Body**:

```json
{
  "dcUserId": "user-uuid-xxxx"
}
```

**业务逻辑**:

```
1. 校验权限：当前用户须具备 drawing:dc-config，否则 403
2. 校验目标用户具备 drawing:external-approval 权限，否则返回:
   { "code": 1003007011, "message": "用户无 DC 权限" }
3. 校验 (projectId, dcUserId) 未配置（deleted_at IS NULL），若已存在返回 409
4. 查询目标用户姓名（来自用户服务）
5. INSERT INTO project_dc_config (id, project_id, dc_user_id, dc_user_name,
     configured_by, configured_by_name, configured_at)
6. 记录审计日志（actor_id、dc_user_id、action=DC_ADD、timestamp）
7. 返回新创建的 DcConfigItem
```

**响应**: HTTP 201，返回 DcConfigItem（结构同 §4.1）

**错误码**:

| HTTP | code | 说明 |
|------|------|------|
| 403 | — | 无 `drawing:dc-config` 权限 |
| 409 | 1003007012 | 该用户已配置为 DC |
| 422 | 1003007011 | 目标用户无 `drawing:external-approval` 权限 |

---

### 4.4 `DELETE /project/{projectId}/dc-config/{dcUserId}` — 移除 DC

**覆盖 AC**: AC-007D-006 / AC-007D-007 / AC-007D-008

**权限**: `drawing:dc-config`

**业务逻辑**:

```
1. 校验权限：当前用户须具备 drawing:dc-config，否则 403
2. 查询 project_dc_config WHERE project_id = :projectId AND dc_user_id = :dcUserId AND deleted_at IS NULL
   若不存在返回 404
3. 执行软删除：UPDATE project_dc_config SET deleted_at = NOW() WHERE id = :id
4. 记录审计日志（actor_id、dc_user_id、action=DC_REMOVE、timestamp）
5. 返回 HTTP 204
```

> **注意**：移除操作不阻止（即使列表变为空），后续内部审批通过时若无 DC 则由 BACKEND-REQ-007A §3.5 的兜底校验拦截并提示。（AC-007D-008）

**正在进行中的 Todo 不受影响**：已发送给该 DC 的 `PENDING` 外部审批 Todo 记录不因配置移除而消失或变更，由 DC 自行处理完结。

**响应**: HTTP 204 No Content

**错误码**:

| HTTP | code | 说明 |
|------|------|------|
| 403 | — | 无权限 |
| 404 | 1003007013 | 未找到有效 DC 配置 |

---

## 5. 业务逻辑补充

### 5.1 内部审批通过时的 DC 兜底校验（集成点）

> 本节是对 BACKEND-REQ-007A 内部审批通过接口的补充约束。

当内部审批人调用 `POST /drawing/version/{versionId}/approve` 时，后端执行以下额外校验：

```
查询 project_dc_config WHERE project_id = :projectId AND deleted_at IS NULL
若结果为空（count = 0）：
  返回 HTTP 422，错误码 1003007020
  message: "No DC configured for this project. Please configure a DC before approving."
```

这是兜底校验，防止通过内部审批后系统无法推进外部审批流程。（AC-007D-010）

### 5.2 添加 DC 后的幂等性

- `(project_id, dc_user_id)` 联合唯一约束（`deleted_at IS NULL`）
- 若同一用户先被移除（软删除），再次添加时创建新记录（新 id、新 `configured_at`），历史记录保留

---

## 6. 权限编码

| 权限标识 | 说明 | 适用角色 |
|--------|------|---------|
| `drawing:dc-config` | 查看/添加/移除 DC 配置 | 项目管理员、业务人员 |
| `drawing:external-approval` | 作为 DC 执行外部审批 | Document Controller |

---

## 7. 错误码汇总

| 错误码 | HTTP | 说明 |
|------|------|------|
| 1003007011 | 422 | 目标用户无 `drawing:external-approval` 权限 |
| 1003007012 | 409 | 该用户已配置为本项目 DC |
| 1003007013 | 404 | 未找到有效 DC 配置记录 |
| 1003007020 | 422 | 项目无 DC 配置，内部审批无法通过 |

---

## 8. 审计日志

每次添加/移除 DC 须写入审计日志表（复用既有审计机制）：

| 字段 | 说明 |
|-----|------|
| `actor_id` | 操作人 user_id |
| `actor_name` | 操作人姓名 |
| `action` | `DC_ADD` / `DC_REMOVE` |
| `target_user_id` | 被操作的 DC user_id |
| `target_user_name` | 被操作的 DC 姓名 |
| `project_id` | 项目 ID |
| `timestamp` | 操作时间 |

---

## 9. 非功能需求

### 9.1 性能

| 接口 | 目标 P95 |
|------|---------|
| GET /dc-config | ≤ 500ms |
| GET /dc-config/available | ≤ 800ms（需跨服务查询权限） |
| POST /dc-config | ≤ 1s |
| DELETE /dc-config/{dcUserId} | ≤ 1s |

### 9.2 安全

- 所有接口须携带 `Authorization`（JWT）、`X-Tenant-Id`、`Project-Id`
- `drawing:dc-config` 权限校验在 Controller 层统一拦截
- 行级校验：`project_id` 必须与 JWT 中的当前项目匹配

### 9.3 数据迁移

- 新增 `project_dc_config` 表，无历史数据迁移
- 建议上线初期由运维执行初始化脚本，将各项目 DC 人员录入

---

## 10. 上线前置条件

- [ ] `drawing:dc-config` 权限码已创建并绑定到管理员角色
- [ ] `drawing:external-approval` 权限码已创建并绑定到 DC 角色
- [ ] `project_dc_config` 表及索引已 migrate
- [ ] BACKEND-REQ-007A `POST /drawing/version/{versionId}/approve` 已加入兜底校验（§5.1）

---

## 11. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-27 | agent | 从 REQ-007D-pc.md 初稿生成 |
