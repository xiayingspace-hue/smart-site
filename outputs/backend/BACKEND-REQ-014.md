# 后端开发说明文档 — REQ-014 图纸管理用户反馈迭代

> **来源需求**: `requirements/pc/REQ-014-pc.md`
> **产品**: SMART SITE SYSTEM
> **生成日期**: 2026-04-28

---

## 1. 需求来源

REQ-014 整合 REQ-003（图纸版本管理）、REQ-004（Markup 管理）、REQ-005（图纸分发）第一版用户反馈，后端需：
- 允许 Drawing Code 在上传新版本时随版本更新（FB-001）
- 新增版本附件表及 CRUD 接口（FB-002）
- Markup 记录自动关联当时的 Drawing Code（FB-004）
- 对 SE 角色屏蔽 Markup 接口（FB-005）
- SE 请求图纸/版本数据时过滤 Ver 字段（FB-006）

---

## 2. 数据模型变更

### 2.1 `drawing_versions` 表 — Drawing Code 更新支持（FB-001）

| 字段 | 类型 | 说明 | 变更 |
|------|------|------|------|
| `drawing_code` | `VARCHAR(100)` | 图纸编号 | **新增可更新逻辑**（原只在首次创建图纸时写入） |

**说明**:
- `drawing_code` 字段本身已存在于 `drawings` 表（或 `drawing_versions` 表），本次变更允许在创建新版本（`POST /versions`）时传入新值覆盖当前版本的 Drawing Code
- 历史版本的 `drawing_code` 保持原值，不受影响
- 唯一性约束：同一 `project_id` 下，`drawing_code` + `version_status = active` 的组合不可重复（或按业务规则确定唯一性范围）

---

### 2.2 `drawing_attachments` 表 — 新建（FB-002）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | `BIGINT` PK | AUTO_INCREMENT | 主键 |
| `drawing_id` | `BIGINT` | NOT NULL, FK → `drawings.id` | 关联图纸 |
| `version_id` | `BIGINT` | NOT NULL, FK → `drawing_versions.id` | 关联版本 |
| `file_name` | `VARCHAR(255)` | NOT NULL | 原始文件名 |
| `file_type` | `VARCHAR(50)` | | 文件后缀/MIME |
| `file_size` | `BIGINT` | | 字节数 |
| `storage_key` | `VARCHAR(500)` | NOT NULL | OSS / S3 存储路径 |
| `remark` | `VARCHAR(500)` | | 备注 |
| `uploaded_by` | `BIGINT` | FK → `users.id` | 上传人 |
| `uploaded_at` | `DATETIME` | DEFAULT NOW() | 上传时间 |
| `is_deleted` | `TINYINT(1)` | DEFAULT 0 | 软删除标志 |

---

### 2.3 `markups` 表 — 新增 Drawing Code 字段（FB-004）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `drawing_code` | `VARCHAR(100)` | | **新增字段**，记录 Markup 创建时图纸的 Drawing Code；由后端在创建 Markup 时自动从当前版本读取并写入，只读 |

**迁移**:
- 新增字段，存量数据可通过关联查询回填（或留空，历史数据标注为 `NULL`）
- 后续所有新建 Markup 由服务层自动填充，不由前端传入

---

## 3. 接口变更

### 3.1 FB-001 — 上传新版本支持修改 Drawing Code

**接口**: `POST /api/drawings/:drawingId/versions`（REQ-003 已有接口，新增字段支持）

**Request Body 新增字段**:

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `drawingCode` | `String` | 否 | 本次版本的 Drawing Code；不传则沿用当前图纸的最新 Drawing Code |

**校验逻辑**:
1. 若 `drawingCode` 有传值：
   - 去除首尾空格
   - 查询同 `project_id` 下是否已存在相同 `drawing_code`（排除当前图纸自身）
   - 若存在 → 返回 `400 Bad Request`，`{ code: "DRAWING_CODE_CONFLICT", message: "该 Drawing Code 已存在" }`
   - 若不存在 → 将此值写入新版本记录
2. 若未传 `drawingCode`：沿用当前最新版本的 Drawing Code

---

### 3.2 FB-001 — Drawing Code 唯一性预校验接口（前端输入时调用）

**接口**: `GET /api/drawings/check-code`

**Query Params**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `code` | `String` | ✅ | 待校验的 Drawing Code |
| `projectId` | `Long` | ✅ | 项目 ID |
| `excludeDrawingId` | `Long` | 否 | 排除当前图纸自身（编辑场景） |

**Response**:
```json
{ "available": true }     // 可用
{ "available": false }    // 已被占用
```

---

### 3.3 FB-002 — 版本附件列表

**接口**: `GET /api/drawings/:drawingId/versions/:versionId/attachments`

**权限**: 已分配该版本的所有角色（含 SE）

**Response**:
```json
{
  "attachments": [
    {
      "id": 1,
      "fileName": "施工说明.pdf",
      "fileType": "pdf",
      "fileSize": 204800,
      "remark": "备注内容",
      "uploadedBy": "张三",
      "uploadedAt": "2026-03-10T08:00:00Z"
    }
  ]
}
```

---

### 3.4 FB-002 — 上传附件

**接口**: `POST /api/drawings/:drawingId/versions/:versionId/attachments`

**权限**: 图纸管理员 / 上传人（SE 无权）

**Request**: `multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | File | ✅ | 附件文件（不限类型） |
| `remark` | String | 否 | 备注 |

**处理逻辑**:
1. 文件上传至 OSS / S3，获取 `storageKey`
2. 写入 `drawing_attachments` 表
3. 返回新增附件记录

---

### 3.5 FB-002 — 删除附件

**接口**: `DELETE /api/drawings/:drawingId/versions/:versionId/attachments/:attachmentId`

**权限**: 图纸管理员 / 上传人（SE 无权）

**处理逻辑**: 软删除（`is_deleted = 1`），不删除 OSS 文件（或按业务决定）

---

### 3.6 FB-002 — 下载附件

**接口**: `GET /api/drawings/:drawingId/attachments/:attachmentId/download`

**权限**: 已分配版本的所有角色（含 SE）

**处理逻辑**: 返回预签名下载 URL（有效期 15 分钟）或直接 `302` 重定向

---

### 3.7 FB-004 — Markup 列表返回 Drawing Code 字段

**接口**: `GET /api/drawings/:drawingId/markups`（REQ-004 已有接口，新增字段）

**变更**: Response 中每条 Markup 记录新增 `drawingCode` 字段

```json
{
  "markups": [
    {
      "id": 10,
      "fileName": "markup-v2.pdf",
      "drawingCode": "DWG-001",  // ← 新增
      "uploadedAt": "2026-03-15T09:00:00Z"
    }
  ]
}
```

---

### 3.8 FB-004 — 创建 Markup 时自动写入 Drawing Code

**接口**: `POST /api/drawings/:drawingId/markups`（REQ-004 已有接口，服务层调整）

**变更**: 前端不传 `drawingCode`；服务层在保存 Markup 记录时，查询当前版本的 `drawing_code` 并自动写入 `markups.drawing_code`

---

## 4. 权限控制

### 4.1 SE 屏蔽 Markup 接口（FB-005）

在 **Markup 相关路由中间件** 新增 SE 角色拦截：

```
// 伪代码
if (currentUser.role === 'SE') {
  return res.status(403).json({ code: 'FORBIDDEN', message: 'SE 角色无权访问 Markup 功能' })
}
```

受影响接口（GET / POST / PUT / DELETE `/api/drawings/:id/markups*`）均需添加此校验。

---

### 4.2 SE 视角过滤 Ver 字段（FB-006）

**接口**: `GET /api/drawings/:drawingId/versions/:versionId`（及版本列表接口）

在响应序列化层，根据当前用户角色动态过滤字段：

| 角色 | `drawingCode` | `ver`（系统版本号） |
|------|---------------|---------------------|
| 非 SE | ✅ 返回 | ✅ 返回 |
| SE | ✅ 返回 | ❌ 不返回（字段从响应中移除） |

建议使用序列化层（如 `class-transformer` 的 `@Exclude` + 动态 groups，或 GraphQL resolver 层过滤），不在 SQL 层控制，以便日后维护。

---

## 5. 重点注意事项

- **FB-001 唯一性范围**：需与产品确认 Drawing Code 唯一性范围——是整个系统唯一、还是项目内唯一；当前文档按"项目内唯一"实现
- **FB-002 存储**：附件支持任意文件类型，OSS 需配置不做 MIME 校验限制；单文件大小上限建议与 PDF 保持一致（或单独配置）
- **FB-004 历史数据**：`markups.drawing_code` 历史存量数据可能为 `NULL`，前端需做空值处理（显示 `—`）
- **FB-005 防御性**：Markup 接口除中间件校验外，Service 层也应加角色判断，防止绕过中间件的内部调用
- **FB-006 字段过滤**：`ver` 过滤仅对 SE 角色生效；其他所有有权访问版本数据的角色均正常返回 `ver`

---

> 本文档配合 `REQ-014-pc.md` 需求文档及 `UI-REQ-014-pc.md` UI 说明文档使用。
