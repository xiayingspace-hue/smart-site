---
doc_type: backend_spec
req_id: REQ-003F-pc
version: 0.5.0
status: draft
generated_from: REQ-003F-pc.md@0.5.0
generated_at: 2026-05-26
owner: ""
---

# 后端开发说明 — PC 端上传新版本

> **来源需求**: [REQ-003F-pc.md](../../../requirements/pc/REQ-003F-pc.md) @ v0.5.0
> **依赖共享规范**: [REQ-003-shared](../../../requirements/shared/REQ-003-shared.md)、[REQ-007-shared](../../../requirements/shared/REQ-007-shared.md)
> **产品**: SMART SITE SYSTEM
> **服务模块**: `drawing-service`
> **生成日期**: 2026-05-26

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003F-pc @ v0.5.0 |
| 覆盖 Story | US-003F-001 |
| 覆盖 AC | AC-003F-001 ~ AC-003F-013 |

---

## 1. 功能概述

本模块实现"上传新版本"的全后端链路：

| 功能 | 说明 |
|------|------|
| 文件预上传 | 接收 PDF 文件、校验、写入对象存储，返回临时文件 ID 与 AI 任务 ID |
| AI 识别（异步） | 服务端发起 AI 识别任务，识别 PDF 各页的 Drawing Code / Drawing Name |
| AI 状态查询 | 前端轮询接口，返回 PROCESSING / DONE / FAILED 及识别结果页列表 |
| 提交新版本 | 校验状态守卫、创建 DrawingVersion、更新图纸状态、触发内部审批 Todo |
| 内部审批通知 | 新版本提交成功后，异步通知指定内部审批人 |

---

## 2. 数据模型

### 2.1 涉及实体

```
Drawing (主记录)
├── id, drawingCode, drawingName, category, description
└── currentVersionId → DrawingVersion

DrawingVersion (版本记录)
├── id, drawingId, versionNo (V0/V1/V2...)
├── fileId (→ ObjectStorage), fileUrl (预签名)
├── aiJobId (→ AIRecognitionJob)
├── versionNote
├── internalApproverId
├── approvalStatus: PENDING_INTERNAL | PENDING_EXTERNAL | ACTIVE
│                  | INTERNAL_REJECTED | EXTERNAL_REJECTED | DEPRECATED
├── uploaderId, uploadedAt
└── updatedAt

AIRecognitionJob (AI 识别任务)
├── id, drawingVersionId
├── status: PROCESSING | DONE | FAILED
├── pages: AIRecognitionPage[]
└── createdAt, completedAt

AIRecognitionPage (识别结果页)
├── jobId, pageNo
├── thumbnailUrl (低分辨率预览图 URL)
├── drawingCode (识别到的图纸编号，可为 null)
└── drawingName (识别到的图纸名称，可为 null)
```

### 2.2 版本号规则

- 版本号格式：`V{n}`，n 从 0 开始，每次新版本 +1
- 查询当前图纸最大版本号：`SELECT MAX(versionNo) FROM drawing_version WHERE drawing_id = ?`
- 新版本号 = MAX + 1（并发场景加数据库唯一约束，详见 §5.3）

### 2.3 状态守卫（AC-003F-011 对应后端校验）

提交新版本前，后端须检查当前图纸是否存在进行中版本：

```sql
SELECT COUNT(*) FROM drawing_version
WHERE drawing_id = :drawingId
  AND approval_status IN ('PENDING_INTERNAL', 'PENDING_EXTERNAL')
```

如果 COUNT > 0，返回 **HTTP 409 Conflict**：
```json
{
  "code": "DRAWING_UNDER_REVIEW",
  "message": "This drawing is now under review."
}
```

---

## 3. 接口设计

### 3.1 接口总览

| Method | Path | 说明 | Auth |
|--------|------|------|------|
| `POST` | `/drawing/{drawingId}/version/file` | Step 1：上传 PDF 文件，触发 AI 识别 | JWT + drawing:upload |
| `GET` | `/ai/job/{aiJobId}/status` | Step 2：AI 识别状态轮询（前端每 2s 调用） | JWT |
| `POST` | `/drawing/{drawingId}/version` | Step 3：提交新版本（创建 DrawingVersion） | JWT + drawing:upload |
| `GET` | `/project/{projectId}/approvers` | 获取有 `drawing:approve` 权限的用户列表 | JWT |

---

### 3.2 `POST /drawing/{drawingId}/version/file`

**功能**：接收 PDF 文件 → 写入对象存储 → 创建 AIRecognitionJob → 异步调用 AI 服务

**请求**：`multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|:---:|------|
| `file` | File | ✅ | PDF 文件，≤ 50MB |
| `drawingId` | string | ✅ | 图纸 ID |

**后端校验**：

1. `file` MIME type 必须为 `application/pdf`（前后端双重校验，AC-003F-008）
2. `file.size ≤ 50 * 1024 * 1024`（AC-003F-009）
3. `drawingId` 对应 Drawing 存在且当前用户有 `drawing:upload` 权限

**处理流程**：

```
1. 校验文件类型与大小
2. 生成唯一文件 Key：{drawingId}/versions/temp/{uuid}.pdf
3. 上传至对象存储（OSS/S3），记录 objectKey
4. 创建 AIRecognitionJob 记录（status=PROCESSING）
5. 发送消息至 AI 识别队列（payload: { jobId, objectKey }）
6. 返回 { versionFileId, aiJobId }
```

**响应 200**：
```json
{
  "versionFileId": "file-uuid-001",
  "aiJobId": "job-uuid-001"
}
```

**错误响应**：

| 状态码 | code | 说明 |
|--------|------|------|
| 400 | `INVALID_FILE_TYPE` | 非 PDF 文件 |
| 400 | `FILE_TOO_LARGE` | 超过 50MB |
| 403 | `PERMISSION_DENIED` | 无 drawing:upload 权限 |
| 404 | `DRAWING_NOT_FOUND` | drawingId 不存在 |

---

### 3.3 `GET /ai/job/{aiJobId}/status`

**功能**：返回 AI 识别任务当前状态及识别结果页列表

**权限**：JWT 鉴权，用户须对该任务所属图纸有读权限

**响应 200（PROCESSING）**：
```json
{
  "status": "PROCESSING",
  "pages": []
}
```

**响应 200（DONE）**：
```json
{
  "status": "DONE",
  "pages": [
    {
      "pageNo": 1,
      "thumbnailUrl": "https://cdn.example.com/thumbs/job-uuid-001/page1.jpg",
      "drawingCode": "ARCH-001",
      "drawingName": "Ground Floor Plan"
    },
    {
      "pageNo": 2,
      "thumbnailUrl": "https://cdn.example.com/thumbs/job-uuid-001/page2.jpg",
      "drawingCode": null,
      "drawingName": null
    }
  ]
}
```

**响应 200（FAILED）**：
```json
{
  "status": "FAILED",
  "pages": []
}
```

> `thumbnailUrl` 为 CDN 或对象存储预签名 URL，有效期 ≥ 30min（保障用户完成审阅）。

---

### 3.4 `POST /drawing/{drawingId}/version`

**功能**：创建新 DrawingVersion，触发内部审批流程

**权限**：JWT + `drawing:upload` + 用户为该图纸的上传人（或项目管理员）

**请求体**：
```json
{
  "versionFileId":      "file-uuid-001",
  "versionNote":        "Fixed dimension errors on page 2",
  "internalApproverId": "user-uuid-approver",
  "aiJobId":            "job-uuid-001",
  "skipAiResult":       false
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|:---:|------|
| `versionFileId` | string | ✅ | 上传文件接口返回的临时文件 ID |
| `versionNote` | string | ❌ | 版本说明，建议 ≤ 500 字符 |
| `internalApproverId` | string | ✅ | 内部审批人用户 ID |
| `aiJobId` | string | ✅ | AI 任务 ID（即使 skipAiResult=true 也需传入，用于关联记录） |
| `skipAiResult` | boolean | ✅ | `false`=正常路径；`true`=Proceed Anyway 降级 |

**后端处理流程**：

```
1. 校验 versionFileId 属于当前用户且未被使用（防重复提交）
2. 状态守卫：检查是否存在 PENDING_INTERNAL / PENDING_EXTERNAL 版本（AC-003F-011）
   → 若存在，返回 409 DRAWING_UNDER_REVIEW
3. 获取当前图纸最大版本号 maxN，新版本号 = maxN + 1
   → 数据库级唯一约束：(drawingId, versionNo) UNIQUE
4. 将 versionFileId 对应的临时文件移至正式路径：
   {drawingId}/versions/V{n}/{uuid}.pdf
5. 创建 DrawingVersion 记录：
   { drawingId, versionNo, fileId, versionNote, internalApproverId,
     approvalStatus='PENDING_INTERNAL', uploaderId, uploadedAt }
6. 更新 Drawing.currentVersionId = 新 DrawingVersion.id
7. 将 AIRecognitionJob.drawingVersionId = 新 DrawingVersion.id（关联绑定）
8. 发送内部审批通知（异步消息队列）：
   { type: 'INTERNAL_APPROVAL_REQUIRED', approverId, drawingVersionId }
9. 记录审计日志
10. 返回 { drawingVersionId, versionNo, approvalStatus }
```

**响应 201**：
```json
{
  "drawingVersionId": "version-uuid-001",
  "versionNo": "V3",
  "approvalStatus": "PENDING_INTERNAL"
}
```

**错误响应**：

| 状态码 | code | 说明 |
|--------|------|------|
| 400 | `INVALID_FILE_ID` | versionFileId 不存在或已被使用 |
| 400 | `MISSING_APPROVER` | internalApproverId 未传或无效 |
| 403 | `PERMISSION_DENIED` | 无上传权限 |
| 404 | `DRAWING_NOT_FOUND` | drawingId 不存在 |
| 409 | `DRAWING_UNDER_REVIEW` | 当前已有进行中版本（AC-003F-011） |

---

### 3.5 `GET /project/{projectId}/approvers`

**功能**：返回项目内有 `drawing:approve` 权限的用户列表

**权限**：JWT 鉴权，当前用户须属于该项目

**响应 200**：
```json
{
  "approvers": [
    { "id": "user-001", "name": "Alice Chen", "avatar": "https://..." },
    { "id": "user-002", "name": "Bob Wang",   "avatar": "https://..." }
  ]
}
```

---

## 4. AI 识别服务集成

### 4.1 集成方式

- **触发**：`POST /drawing/{drawingId}/version/file` 成功后，服务端向消息队列投递 AI 识别任务
- **回调**：AI 服务完成后通过内部事件（或 Webhook）更新 `AIRecognitionJob.status` 和 `AIRecognitionPage` 记录
- **超时处理**：`AIRecognitionJob` 设置超时字段，定时任务（每分钟）扫描超时任务并置为 `FAILED`

### 4.2 缩略图生成

- AI 服务完成后，同步生成各页低分辨率缩略图（建议分辨率 150×150px）
- 缩略图上传至对象存储，生成 `thumbnailUrl` 写入 `AIRecognitionPage` 记录
- `thumbnailUrl` 通过 CDN 或预签名 URL 提供，有效期 ≥ 30min

### 4.3 服务不可用降级

- AI 服务返回错误或超时（超出阈值）→ `AIRecognitionJob.status = FAILED`
- 前端 `Proceed Anyway` 路径允许跳过识别结果继续提交
- 后端 `submitNewVersion` 接受 `skipAiResult=true`，关联 FAILED 的 AIRecognitionJob 记录用于审计追溯

---

## 5. 业务规则

### 5.1 Drawing 主记录不可修改（AC-003F-001 / AC-003F-012）

提交新版本时，后端**严禁**更新 `Drawing` 主记录的以下字段：
- `drawingCode`（Drawing Code）
- `drawingName`（Drawing Name）
- `category`
- `description`

以上字段在新建图纸时（REQ-003E-pc）已确定，版本迭代仅创建新 `DrawingVersion`，主记录字段保持不变。

### 5.2 版本号自动递增（AC-003F-001）

```sql
-- 获取当前最大版本号（原始数字部分）
SELECT COALESCE(MAX(CAST(SUBSTRING(version_no, 2) AS UNSIGNED)), -1) + 1 AS next_n
FROM drawing_version WHERE drawing_id = :drawingId
-- 新版本号 = 'V' + next_n
```

数据库层唯一约束：`UNIQUE KEY uk_drawing_version (drawing_id, version_no)`

### 5.3 并发控制

- 使用乐观锁或数据库唯一约束防止并发创建相同版本号
- 建议：在 Transaction 内先执行状态守卫查询（加 `SELECT FOR UPDATE`），再创建 DrawingVersion

### 5.4 临时文件有效期

- `versionFileId` 对应的临时对象存储文件，有效期 **24 小时**
- 超期未提交的临时文件由定时清理任务删除
- 已提交（DrawingVersion 创建成功）的文件移至正式路径，永久保留

### 5.5 内部审批通知（AC-003F-013）

提交成功后，向消息队列发送事件：
```json
{
  "type": "INTERNAL_APPROVAL_REQUIRED",
  "approverId": "user-uuid-approver",
  "drawingId": "drawing-uuid-001",
  "drawingVersionId": "version-uuid-001",
  "drawingCode": "ARCH-001",
  "drawingName": "Ground Floor Plan",
  "versionNo": "V3",
  "uploaderId": "user-uuid-uploader",
  "projectId": "project-uuid-001"
}
```

消息消费方（Todo 服务）负责：
- 创建 Todo 任务：`"Internal Approval Required — ARCH-001 Ground Floor Plan V3"`
- （可选）发送 App Push 通知至审批人手机端

---

## 6. 安全

| 要点 | 规范 |
|------|------|
| 鉴权 | 所有接口需要有效 JWT；`drawing:upload` 权限校验 |
| 文件类型白名单 | 后端二次校验 MIME type = `application/pdf`（不信任前端 accept 属性） |
| AI 服务凭证 | AI 服务调用在服务端发起，凭证不出现在任何响应中 |
| 临时文件隔离 | 临时文件路径含用户 ID 或 UUID，不可被他人猜测访问 |
| 防重复提交 | `versionFileId` 与 `DrawingVersion` 一对一绑定，提交后标记已用 |
| 审计日志 | 记录：操作人、时间、drawingId、versionNo、fileKey、aiJobId、skipAiResult、操作结果 |

---

## 7. 性能与可观测性

### 7.1 性能指标

| 指标 | 目标值 |
|------|-------|
| `POST /version/file` 响应时间（P95，不含文件传输） | ≤ 3s |
| `GET /ai/job/{id}/status` 响应时间（P95） | ≤ 300ms |
| `POST /version` 响应时间（P95，不含文件传输） | ≤ 3s |
| AI 识别完成时间（P95，10 页以内，待与 AI 团队确认） | ≤ 15s |

### 7.2 监控告警

| 指标 | 告警阈值 |
|------|---------|
| 文件上传失败率 | > 5% |
| AI 识别失败率 | > 20% |
| `POST /version` 5xx 错误率 | > 1% |
| 临时文件积压（未被提交） | > 1000 个（可能存在前端异常） |

### 7.3 关键埋点（服务端）

| 事件 | 说明 |
|------|------|
| `version_file_uploaded` | 文件上传成功（记录文件大小、drawingId） |
| `ai_job_started` | AI 识别任务创建 |
| `ai_job_completed` | AI 识别完成（记录耗时、页数、识别成功页数） |
| `ai_job_failed` | AI 识别失败（记录原因） |
| `version_submitted` | 新版本提交成功（记录 skipAiResult） |
| `version_submit_rejected_409` | 状态冲突拒绝提交 |

---

## 8. 数据库变更

| 表 | 变更类型 | 说明 |
|----|---------|------|
| `drawing_version` | 新增字段 `ai_job_id`（VARCHAR，nullable） | 关联 AI 识别任务 |
| `drawing_version` | 新增字段 `version_note`（TEXT，nullable） | 版本说明 |
| `drawing_version` | 新增字段 `skip_ai_result`（BOOLEAN，default false） | 是否跳过 AI 识别结果 |
| `ai_recognition_job` | 新建表 | 见 §2.1 数据模型 |
| `ai_recognition_page` | 新建表 | 见 §2.1 数据模型 |
| `drawing_version` | 新增唯一约束 `uk_drawing_version (drawing_id, version_no)` | 防并发重复版本号 |

---

## 9. 迁移注意事项

- 存量 `DrawingVersion` 记录的 `ai_job_id` 为 NULL（历史版本），不影响查询
- `skip_ai_result` 默认 false，存量记录不受影响
- 新唯一约束在存量数据已满足时直接添加；如存量存在重复版本号，需先修复数据

---

## 10. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.5.0 | 2026-05-26 | agent | 基于 REQ-003F-pc@0.5.0 首次生成。涵盖 4 个接口设计（文件上传/AI 轮询/版本提交/审批人列表）、AI 识别服务集成（消息队列异步触发/回调/超时降级）、状态守卫（409 并发控制）、主记录不可修改规则、临时文件有效期、内部审批通知消息格式、数据库变更清单 |
