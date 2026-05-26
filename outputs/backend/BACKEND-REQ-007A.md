---
doc_type: backend_spec
req_id: REQ-007A-pc
version: 0.5.0
status: draft
generated_from: REQ-007A-pc.md@0.5.0
generated_at: 2026-05-25
owner: ""
---

# 后端开发说明 — PC 端内部审批 Todo、详情查看与原文件下载

> **来源需求**: [REQ-007A-pc.md](../../requirements/pc/REQ-007A-pc.md) @ v0.5.0
> **依赖后端文档**: [BACKEND-REQ-007.md](./BACKEND-REQ-007.md)（图纸两级审批基础后端）
> **产品**: SMART SITE SYSTEM
> **服务模块**: `drawing-service`
> **生成日期**: 2026-05-25

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007A-pc @ v0.5.0 |
| 依赖文档 | BACKEND-REQ-007.md（内部审批通过/驳回核心逻辑） |
| 覆盖 Story | US-007A-001 / 002 / 003 / 004 / 005 / 006 |
| 覆盖 AC | AC-007A-001 ~ AC-007A-023 |

---

## 1. 功能概述

本文档在 BACKEND-REQ-007 基础上，聚焦以下 PC 端内部审批相关后端能力：

| 功能 | 说明 | 覆盖 AC |
|------|------|---------|
| 内部审批 Todo 列表接口 | 仅返回 assigneeId = 当前用户的内部审批待办 | AC-007A-002 |
| 图纸版本详情接口 | 返回详情侧滑弹框所需全量字段 | AC-007A-004 / 005 / 006 |
| 文件 inline 访问接口 | 生成 Content-Disposition: inline 预签名 URL | AC-007A-011 |
| 文件强制下载接口 | 生成 Content-Disposition: attachment 预签名 URL + 行级权限校验 | AC-007A-007 / 008 / 009 / 010 |
| 内部审批通过接口 | 含 DC 配置二次校验（兜底）、状态流转、副作用 | AC-007A-015 / 016 / 017 / 018 / 019 / 020 |
| 内部审批驳回接口 | Comment 必填校验、状态流转、通知设计人员 | AC-007A-021 / 022 / 023 |
| DC 配置查询接口 | 前端 Approve 前置检查使用 | AC-007A-014 |

> **注意**：内部审批通过/驳回的核心状态流转逻辑已在 BACKEND-REQ-007 §2.2 / §2.3 定义，本文档在此基础上补充 PC 端特有的行级权限、下载接口和 DC 配置查询实现细节。

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

沿用 BACKEND-REQ-007 全部技术约定（语言、框架、数据库、对象存储、消息队列等）。

### 2.2 本需求新增依赖

| 库 / 服务 | 用途 | 评估 |
|-----------|------|------|
| 对象存储 SDK（OSS / S3） | 生成预签名 URL（view & download） | 已在 BACKEND-REQ-003 引入 |

---

## 3. API 接口定义

### 3.1 `GET /todo/list` — 内部审批待办列表

**覆盖 AC**: AC-007A-002 / AC-007A-003

**鉴权**: JWT，需携带 `Authorization` / `X-Tenant-Id` / `Project-Id`

**查询参数**:

| 参数 | 类型 | 必填 | 说明 |
|-----|------|-----|------|
| `type` | string | 是 | `DRAWING_INTERNAL_APPROVAL` |
| `projectId` | string | 是 | 当前项目 ID |

**过滤逻辑**:

```sql
WHERE todo.type = 'DRAWING_INTERNAL_APPROVAL'
  AND todo.assignee_id = :currentUserId   -- 仅自己名下
  AND todo.project_id = :projectId
  AND todo.status = 'PENDING'
ORDER BY todo.created_at DESC
```

**响应字段**（每条 TodoItem）:

| 字段 | 类型 | 来源 | 说明 |
|-----|------|------|------|
| `id` | string | todo.id | Todo 任务 ID |
| `type` | string | todo.type | `DRAWING_INTERNAL_APPROVAL` |
| `drawingVersionId` | string | todo.drawing_version_id | — |
| `drawingCode` | string | drawing.code | 图纸编号 |
| `drawingName` | string | drawing.name | 图纸名称 |
| `versionNo` | string | drawing_version.version_no | 例：V3 |
| `uploaderName` | string | drawing_version.uploader_name | — |
| `uploadTime` | string | drawing_version.upload_time | ISO 8601 |
| `versionNote` | string\|null | drawing_version.version_note | 可为 null |

**空列表**：返回 `{ "data": [] }`，HTTP 200。

---

### 3.2 `GET /drawing/version/{versionId}/detail` — 图纸版本详情

**覆盖 AC**: AC-007A-004 / AC-007A-005 / AC-007A-006

**路径参数**: `versionId` — DrawingVersion ID

**权限校验**:
- 当前用户须具备 `drawing:approve` 权限
- 当前用户须为该版本的 `assigneeId`（行级权限），否则返回 403

**响应字段**:

| 字段 | 类型 | 来源 | 说明 |
|-----|------|------|------|
| `versionId` | string | drawing_version.id | — |
| `drawingCode` | string | drawing.code | — |
| `drawingName` | string | drawing.name | — |
| `category` | string | drawing.category | — |
| `description` | string\|null | drawing.description | 无则返回 null |
| `versionNo` | string | drawing_version.version_no | 例：V3 |
| `versionNote` | string\|null | drawing_version.version_note | 无则返回 null |
| `uploaderName` | string | drawing_version.uploader_name | — |
| `uploadTime` | string | drawing_version.upload_time | ISO 8601 |
| `approverName` | string | drawing_version.approver_name | 被指派的内部审批人姓名 |
| `fileUrl` | string\|null | drawing_version.file_url | 原始文件 OSS key；为 null 时表示文件不可用 |
| `fileName` | string\|null | drawing_version.file_name | 原始文件名（含扩展名） |
| `fileExt` | string\|null | drawing_version.file_ext | 文件扩展名（小写，不含点），例：`pdf` |
| `todoStatus` | string | todo.status | `PENDING` / `APPROVED` / `REJECTED` |

**注**：`description` / `versionNote` 为 null 时，前端显示 `—`（AC-007A-006）。

---

### 3.3 `GET /drawing/version/{versionId}/view` — 文件 inline 预签名 URL

**覆盖 AC**: AC-007A-011

**权限校验**：同 §3.2（`drawing:approve` + `assigneeId` 行级权限）

**响应**:

```json
{
  "url": "https://oss.example.com/bucket/path/file.pdf?X-Amz-Expires=300&...",
  "expiresIn": 300
}
```

**预签名 URL 参数**:
- `Content-Disposition: inline`（浏览器在标签页中打开）
- 有效期：5 分钟（300 秒）

**注**：若 `fileUrl` 为空，返回 404：`{ "code": "FILE_NOT_FOUND", "message": "Original file not available." }`

---

### 3.4 `GET /drawing/version/{versionId}/download` — 文件强制下载预签名 URL

**覆盖 AC**: AC-007A-007 / AC-007A-008 / AC-007A-009 / AC-007A-010

**权限校验**：
- 当前用户须具备 `drawing:approve` 权限
- 当前用户须为该版本的 `assigneeId`（**行级权限**，防越权下载），否则返回 **403**（AC-007A-010）

**响应**:

```json
{
  "url": "https://oss.example.com/bucket/path/file.pdf?X-Amz-Expires=300&response-content-disposition=attachment...",
  "fileName": "ARCH-001-V3-original.pdf",
  "expiresIn": 300
}
```

**预签名 URL 参数**:
- `Content-Disposition: attachment; filename="{drawingCode}-V{versionNo}-original.{ext}"`（强制下载）
- 文件命名规则：`{drawingCode}-V{versionNo}-original.{ext}`（AC-007A-007）
- 有效期：5 分钟（300 秒）

**审计日志**：每次调用须记录操作日志（actor_id、drawing_version_id、timestamp、action=`DOWNLOAD_ORIGINAL`）（AC-007A-010，见 §11）

**注**：若 `fileUrl` 为空，返回 404。

---

### 3.5 `GET /project/{projectId}/dc-config` — 查询项目 DC 配置

**覆盖 AC**: AC-007A-014 / AC-007A-020

**用途**：前端 Approve 前置检查；内部审批通过接口后端兜底使用同一查询逻辑。

**权限**：项目成员均可查询（含内部审批人）

**响应**:

```json
{
  "data": [
    { "userId": "u001", "userName": "王五", "email": "wangwu@example.com" }
  ]
}
```

空列表时返回 `{ "data": [] }`（AC-007A-014 前端据此判断是否弹出无 DC 警告）。

---

### 3.6 `POST /drawing/approve` — 内部审批通过 / 驳回

**覆盖 AC**: AC-007A-015 ~ AC-007A-023（通过 BACKEND-REQ-007 §2.2 / §2.3 已定义核心逻辑，本节补充 PC 端专项）

**请求体**:

```json
{
  "versionId": "v001",
  "phase": "INTERNAL",
  "action": "APPROVE",        // "APPROVE" | "REJECT"
  "comment": "驳回原因"        // action=REJECT 时必填
}
```

**校验**:

| 校验项 | 规则 | 错误码 |
|-------|------|-------|
| `versionId` 存在 | DrawingVersion 必须存在 | `404` |
| Todo 未已处理 | `todo.status == PENDING` | `409` `TASK_ALREADY_COMPLETED` |
| `assigneeId` 匹配 | `drawing_version.assignee_id == currentUserId` | `403` |
| DC 配置存在（action=APPROVE） | `ProjectDcConfig.count > 0`（兜底，AC-007A-020） | `1003007012` |
| Comment 必填（action=REJECT） | comment 非空且长度 ≤ 500 字符 | `400` `COMMENT_REQUIRED` |

**成功响应（APPROVE）**: HTTP 200

```json
{
  "code": 0,
  "message": "Internal approval completed.",
  "data": { "versionId": "v001", "newStatus": "INTERNAL_APPROVED" }
}
```

**成功响应（REJECT）**: HTTP 200

```json
{
  "code": 0,
  "message": "Internal approval rejected.",
  "data": { "versionId": "v001", "newStatus": "INTERNAL_REJECTED" }
}
```

**错误响应**:

| HTTP | 错误码 | 场景 |
|------|-------|------|
| 403 | `FORBIDDEN` | 非本人 assignee |
| 404 | `NOT_FOUND` | versionId 不存在 |
| 409 | `TASK_ALREADY_COMPLETED` | Todo 已被处理 |
| 422 | `1003007012` | 无 DC 配置（action=APPROVE） |
| 400 | `COMMENT_REQUIRED` | action=REJECT 且 comment 为空 |

---

## 4. 业务逻辑详述

### 4.1 内部审批通过业务逻辑（覆盖 AC-007A-015 ~ AC-007A-020）

```
1. 校验 JWT + 权限（drawing:approve）
2. 加载 DrawingVersion（versionId），校验存在性 → 404
3. 校验 todo.status == PENDING → 409（防并发 / 重复操作）
4. 校验 assigneeId == currentUserId → 403
5. 【兜底】查询 ProjectDcConfig 列表，若为空 → 返回 1003007012（AC-007A-020）
6. 加锁（行级锁 / 乐观锁），防并发重复审批
7. [事务开始]
   a. DrawingVersion.approvalStatus → INTERNAL_APPROVED
   b. Drawing.status → PENDING_EXTERNAL
   c. 创建 DrawingApproval（phase=INTERNAL, status=APPROVED, approverId, approvedAt）
   d. Todo.status → APPROVED
   e. 为每个 DC 用户创建 Todo（type=DRAWING_EXTERNAL_APPROVAL）
   f. INSERT outbox_events（type=INTERNAL_APPROVED，payload={versionId, dcUserIds}）
8. [事务提交]
9. 异步（Outbox 调度器）：向每个 DC 发送站内通知
```

**关键约束**：
- 步骤 3（todo.status 检查）与步骤 6（加锁）须配合，防止并发双重审批
- 步骤 5（DC 兜底校验）必须在事务外执行，避免事务内长查询
- 版本不立即生效（`isCurrent` 不变、不推送 SE、不生成 QR）（AC-007A-017）

### 4.2 内部审批驳回业务逻辑（覆盖 AC-007A-021 ~ AC-007A-023）

```
1. 校验 JWT + 权限（drawing:approve）
2. 加载 DrawingVersion，校验存在性 → 404
3. 校验 todo.status == PENDING → 409
4. 校验 assigneeId == currentUserId → 403
5. 校验 comment 非空（后端兜底）→ 400
6. 加锁，防并发
7. [事务开始]
   a. DrawingVersion.approvalStatus → INTERNAL_REJECTED
   b. 若图纸存在已生效旧版本（isCurrent=true），Drawing.status 保持 ACTIVE（AC-007A-023）
      否则 Drawing.status → INTERNAL_REJECTED
   c. 创建 DrawingApproval（phase=INTERNAL, status=REJECTED, comment, approverId, approvedAt）
   d. Todo.status → REJECTED
   e. INSERT outbox_events（type=INTERNAL_REJECTED，payload={versionId, designerId, comment}）
8. [事务提交]
9. 异步：向设计人员发送站内消息（含驳回原因）（AC-007A-022）
```

---

## 5. 状态机实现

> 引用 BACKEND-REQ-007 §2 的状态流转定义，本节补充行级并发控制要点。

**合法转换（本需求涉及）**:

| from | action | to | 守卫条件 |
|------|--------|----|---------|
| `PENDING_INTERNAL` | APPROVE | `INTERNAL_APPROVED` | assigneeId 匹配 + DC 存在 |
| `PENDING_INTERNAL` | REJECT | `INTERNAL_REJECTED` | assigneeId 匹配 + comment 非空 |

**非法转换（必须拒绝）**:
- `INTERNAL_APPROVED` → APPROVE/REJECT：任何重复审批操作 → 409
- 非 assignee 用户触发 APPROVE/REJECT → 403

---

## 6. 事务边界

| 操作 | 事务边界 | 说明 |
|-----|---------|------|
| 内部审批通过（状态变更 + Todo 关闭 + 新 DC Todo 创建 + Outbox 写入） | ✅ 同一事务 | 保证原子性 |
| 向 DC 发送站内通知（Outbox 调度） | ❌ 事务外异步 | 失败可重试，不影响审批结果 |
| 内部审批驳回（状态变更 + Todo 关闭 + Outbox 写入） | ✅ 同一事务 | — |
| 向设计人员发送通知 | ❌ 事务外异步 | — |
| 预签名 URL 生成（view / download） | ❌ 不涉及事务 | 仅读 OSS，无 DB 写入 |

---

## 7. 幂等性要求

| API | 幂等键 | 策略 |
|-----|-------|-----|
| `POST /drawing/approve` | `versionId + phase + action` | DB 唯一约束（DrawingApproval 不允许同一 versionId+phase 重复写入）；Todo 状态 PENDING 检查 |
| `GET /drawing/version/{versionId}/download` | — | 只读（生成预签名 URL），天然幂等 |
| `GET /drawing/version/{versionId}/view` | — | 只读，天然幂等 |

---

## 8. 并发控制

| 场景 | 策略 | 说明 |
|-----|------|------|
| 同一版本被两个请求同时 Approve | 行级锁（`SELECT ... FOR UPDATE`）+ `todo.status == PENDING` 检查 | 第一个事务提交后，第二个查到 status=APPROVED，返回 409 |
| 并发 Approve + Reject | 同上 | 任一先到者成功，另一返回 409 |

---

## 9. 异步任务与事件

### 9.1 Outbox 事件

| 事件类型 | 触发时机 | Payload | 消费者动作 |
|--------|---------|---------|-----------|
| `DRAWING_INTERNAL_APPROVED` | 内部审批通过事务提交后 | `{ versionId, projectId, dcUserIds[] }` | 向各 DC 发站内通知 + Push |
| `DRAWING_INTERNAL_REJECTED` | 内部审批驳回事务提交后 | `{ versionId, projectId, designerId, comment }` | 向设计人员发站内消息（含驳回原因） |

### 9.2 发件箱模式

```
[事务内]
INSERT outbox_events(event_type, payload, status='PENDING', created_at)

[独立调度器，每 5s 扫描]
SELECT * FROM outbox_events WHERE status='PENDING' LIMIT 50
→ 推送消息总线
→ UPDATE status='SENT'
→ 失败时 status='FAILED'，重试 3 次后告警
```

---

## 10. 数据库

### 10.1 无新增表

本需求不引入新数据表，使用 BACKEND-REQ-007 已定义的：
- `drawing_versions`
- `drawings`
- `drawing_approvals`
- `drawing_todos`
- `project_dc_config`
- `outbox_events`

### 10.2 索引确认

以下索引须确认已存在（BACKEND-REQ-007 应已创建）：

```sql
-- 按 assignee + type + status 查询内部审批待办
CREATE INDEX IF NOT EXISTS idx_drawing_todos_assignee_type_status
  ON drawing_todos(assignee_id, type, status)
  WHERE deleted_at IS NULL;

-- 按 version + phase 查询审批记录（防重）
CREATE UNIQUE INDEX IF NOT EXISTS uidx_drawing_approvals_version_phase
  ON drawing_approvals(drawing_version_id, phase)
  WHERE deleted_at IS NULL;
```

---

## 11. 审计日志

| 操作 | 必须审计 | 审计内容 |
|-----|---------|---------|
| 内部审批通过 | ✅ | actor_id, version_id, action=APPROVE, timestamp, request_id |
| 内部审批驳回 | ✅ | actor_id, version_id, action=REJECT, comment, timestamp, request_id |
| 下载原始文件（`/download`） | ✅ | actor_id, version_id, action=DOWNLOAD_ORIGINAL, file_name, timestamp, ip |
| 查看 inline 文件（`/view`） | ✅ | actor_id, version_id, action=VIEW_ORIGINAL, timestamp |

**审计表**：沿用 BACKEND-REQ-007 的 `audit_logs` 表结构。

**下载操作日志保留期限**：与业务审计要求一致（建议 ≥ 1 年）。

---

## 12. 安全要求

### 12.1 行级权限校验（重点）

- **`/view` 和 `/download` 接口**：后端须校验 `drawing_version.assignee_id == currentUserId`，非指派审批人一律返回 **403**（AC-007A-010）
- **`POST /drawing/approve`**：同上行级校验
- 禁止通过接口参数遍历其他版本的文件（防 IDOR）

### 12.2 预签名 URL 安全

- 预签名 URL 有效期严格控制在 5 分钟（300 秒）
- URL 不包含永久凭证；过期后须重新请求接口
- OSS Bucket 不开放公共读，所有文件访问必须经由预签名 URL

### 12.3 输入校验

- `versionId` 使用 UUID 格式校验，防路径穿越
- `comment` 最大长度 500 字符，超出返回 400

### 12.4 限流

- `GET /drawing/version/{versionId}/download`：用户级 60 次/分钟（防爬）
- `POST /drawing/approve`：用户级 30 次/分钟

---

## 13. 性能要求

| 接口 | P95 延迟目标 | 说明 |
|-----|------------|------|
| `GET /todo/list` | ≤ 1.5s | 含分页 |
| `GET /drawing/version/{versionId}/detail` | ≤ 800ms | JOIN 2 表 |
| `GET /drawing/version/{versionId}/view` | ≤ 500ms | 主要耗时：OSS 预签名 URL 生成 |
| `GET /drawing/version/{versionId}/download` | ≤ 500ms | 同上 |
| `POST /drawing/approve` | P95 ≤ 2s | 含事务 + Outbox 写入 |
| `GET /project/{projectId}/dc-config` | ≤ 500ms | 简单查询 |

**关键优化**：
- Todo 列表使用 `idx_drawing_todos_assignee_type_status` 覆盖索引，避免全表扫描
- 详情接口 JOIN `drawings` + `drawing_versions` 确认索引已覆盖 `drawing_version_id`
- 预签名 URL 生成在应用层调用 OSS SDK，非数据库操作；若 OSS 延迟高可考虑缓存（TTL < 4min）

---

## 14. 可观测性

### 14.1 关键 Metrics

| 指标 | 说明 |
|-----|------|
| `drawing.approve.internal.success_total` | 内部审批通过总次数 |
| `drawing.approve.internal.reject_total` | 内部审批驳回总次数 |
| `drawing.original.download_total` | 原文件下载总次数 |
| `drawing.original.download_fail_total` | 下载失败次数 |
| `drawing.approve.dc_not_found_total` | 无 DC 配置被拒绝次数 |

### 14.2 告警

| 告警项 | 阈值 | 等级 |
|-------|------|-----|
| `POST /drawing/approve` 失败率 | > 5% | P2 |
| `GET .../download` 失败率 | > 5% | P2 |
| Outbox 事件堆积（status=FAILED） | > 10 条 | P1 |

---

## 15. AC 覆盖检查表

| AC ID | 实现位置 | 状态 |
|------|---------|------|
| AC-007A-001 | 前端（TodoPanel.vue）；后端仅提供数据 | TODO |
| AC-007A-002 | `GET /todo/list` — SQL WHERE assignee_id 过滤（§3.1） | TODO |
| AC-007A-003 | `GET /todo/list` 返回空列表，前端处理空状态 | TODO |
| AC-007A-004 | `GET /drawing/version/{versionId}/detail`（§3.2） | TODO |
| AC-007A-005 | `GET /drawing/version/{versionId}/detail` 返回全量字段（§3.2） | TODO |
| AC-007A-006 | `description` / `versionNote` 为 null 时前端显示 `—` | TODO |
| AC-007A-007 | `GET .../download` 响应 fileName = `{code}-V{no}-original.{ext}`（§3.4） | TODO |
| AC-007A-008 | 前端 loading 实现；后端接口幂等（AC 主体在前端） | TODO |
| AC-007A-009 | `fileUrl` 为空时 `GET .../download` 返回 404；前端据此 disabled 按钮 | TODO |
| AC-007A-010 | `GET .../view` 和 `GET .../download` 行级权限校验 → 403（§3.3 / §3.4 / §12.1） | TODO |
| AC-007A-011 | `GET .../view` 生成 inline 预签名 URL（§3.3） | TODO |
| AC-007A-012 | 前端（DrawingApprovalDetailDrawer.vue）辅助文案 | TODO |
| AC-007A-013 | 后端返回 5xx 时前端展示 Retry；后端本身无特殊实现 | TODO |
| AC-007A-014 | `GET /project/{projectId}/dc-config` 返回空列表（§3.5） | TODO |
| AC-007A-015 | `POST /drawing/approve` action=APPROVE + DC 已配置路径（§3.6 / §4.1） | TODO |
| AC-007A-016 | `POST /drawing/approve` 成功响应（§3.6）；Toast 在前端 | TODO |
| AC-007A-017 | 内部审批通过后 `isCurrent` 不变，版本状态为 `INTERNAL_APPROVED`（§4.1） | TODO |
| AC-007A-018 | 创建 DC Todo（DRAWING_EXTERNAL_APPROVAL）（§4.1 步骤 e）| TODO |
| AC-007A-019 | 失败时返回错误码，前端保留对话框可重试；后端正常返回 | TODO |
| AC-007A-020 | `POST /drawing/approve` 无 DC 时返回 `1003007012`（§3.6 / §4.1 步骤 5） | TODO |
| AC-007A-021 | `POST /drawing/approve` action=REJECT + comment 空时返回 400（§3.6 / §4.2） | TODO |
| AC-007A-022 | 驳回成功后 Outbox 触发设计人员通知（§4.2 / §9.1） | TODO |
| AC-007A-023 | 驳回时若有 ACTIVE 旧版本，Drawing.status 保持 ACTIVE（§4.2 步骤 b） | TODO |

---

## 16. 测试要求

| 层级 | 框架 | 范围 | 覆盖率目标 |
|-----|------|------|-----------|
| 单元 | JUnit / pytest | 业务逻辑（§4.1 / §4.2）、状态机流转 | ≥ 80% |
| 集成 | Testcontainers | DB 行级权限、并发审批（§8）、Outbox 写入 | 所有合法 + 非法转换 |
| 契约 | schemathesis | 全部接口（§3.1 ~ §3.6） | 全部接口 |
| 状态机 | 单元 | 所有合法 + 非法转换（§5） | 100% |

---

## 17. 验收条件

- [ ] 所有 AC 在 §15 表中标记完成
- [ ] 单元 + 集成测试通过，覆盖率达标
- [ ] 契约测试与前端通过
- [ ] 行级权限越权测试通过（AC-007A-010）
- [ ] 并发审批场景测试通过（§8）
- [ ] 已部署到 dev，可联调

---

## 18. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.5.0 | 2026-05-25 | agent | 基于 REQ-007A-pc v0.5.0（合并 REQ-007E-pc 后）重新生成；新增 view/download 接口（§3.3/3.4）、DC 配置查询（§3.5）、详情接口全量字段（§3.2）、行级权限安全说明（§12.1）、下载审计日志（§11）、AC 覆盖完整映射（§15） |
