# 图纸二维码与公开状态页 — 跨端共享需求

> 本文档定义 REQ-006 的跨端共享部分：二维码生成、PDF 叠加、公开状态查询 API、数据模型扩展。
> 各端（PC / H5）的 UI 规范请参见各端独立需求文档。

## 基本信息

- **需求ID**: REQ-006
- **需求标题**: 图纸二维码与公开状态查询
- **产品**: SMART SITE SYSTEM
- **适用端**: PC（管理员查看/下载/重新生成 QR）/ H5（公开状态页）
- **优先级**: 高
- **状态**: 草稿
- **依赖需求**:
  - [REQ-003-shared.md](./REQ-003-shared.md)（图纸版本与审批流程）
  - [REQ-004-shared.md](./REQ-004-shared.md)（局部更新数量统计）

---

## 1. 需求描述

### 1.1 背景与目标

图纸印制或导出 PDF 后进入施工现场，现场人员（包括分包商、监理、业主访客等**非系统用户**）无法直接验证手中图纸是否为最新有效版本。当前痛点：

- 现场持有旧版本图纸施工，可能造成质量事故和返工
- 现场非系统用户无账号，无法登录核对版本
- 即使持有当前有效版本，也无法获知是否存在需要关注的活跃局部更新
- 人工贴码、盖章等方式存在遗漏或贴错的风险

本功能目标：
1. 每次图纸版本审批通过后，系统自动生成该版本专属二维码
2. 二维码自动叠加至 PDF 图纸文件，无需人工介入
3. 任何人扫描二维码即可打开 H5 公开状态页（**无需登录**）
4. 公开状态页明确告知：该版本是否仍为当前有效版本（Active / Deprecated）、是否存在活跃的局部更新
5. 不暴露图纸内容和附件，避免知识产权外泄

### 1.2 用户故事

```
作为 现场任何查看图纸的人员（包括分包商、监理、访客等非系统用户）
我想要 通过手机扫描图纸上的二维码
以便 立即判断手中图纸是否为当前有效版本，以及是否存在需要注意的局部更新
```

```
作为 Site Engineer
我想要 扫码后看到该版本是否有活跃的局部更新
以便 即使纸质图纸上没有标注，也能知道需要额外关注的改动
```

```
作为 Drawing 团队成员
我想要 图纸审批通过后系统自动在 PDF 上叠加二维码
以便 无需人工贴码或额外操作，减少流程风险与人工错误
```

### 1.3 功能范围

| 功能 | PC 端 | H5 端 | 说明 |
|------|-------|-------|------|
| 版本审批通过后自动生成 QR | ✅（后端触发） | — | 无 UI 操作 |
| QR 自动叠加到 PDF 文件 | ✅（后端处理） | — | 下载的 PDF 已带 QR |
| 查看 / 下载单独 QR 图片 | ✅ | — | Drawing 团队在版本详情中可查看 |
| 重新生成 QR（容错） | ✅ | — | 仅 `drawing:upload` 权限 |
| 扫码打开公开状态页 | — | ✅ | 无需登录 |
| 公开状态页展示版本状态 | — | ✅ | Active / Deprecated + Markup 数 |

---

## 2. 数据模型

### 2.1 DrawingVersion 字段扩展

在 [REQ-003-shared §2.2](./REQ-003-shared.md) 定义的 DrawingVersion 表中新增字段：

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 公开访问令牌 | publicToken | String(64) | 自动 | 随机生成的唯一令牌，作为公开状态页 URL 的路径参数，防止 ID 枚举攻击 |
| QR 图片 URL | qrImageUrl | String | 自动 | QR 二维码图片 OSS 地址（PNG 格式，尺寸 512×512） |
| 带 QR 的 PDF URL | pdfWithQrUrl | String | 自动 | 叠加 QR 后的 PDF 文件 OSS 地址；仅当原始文件为 PDF 时生成 |
| QR 生成时间 | qrGeneratedTime | DateTime | 自动 | QR 生成完成时间戳；为空表示未生成或生成失败 |

> **说明**：
> - `publicToken` 与 `drawingVersionId` 一一稳定绑定，对外仅以 token 暴露，不暴露 versionId
> - 非 PDF 原始文件（DWG / DXF / PNG / JPG）不进行 QR 叠加，但仍生成独立 QR 图片（`qrImageUrl`）供下载
> - 原始 `fileUrl`（REQ-003-shared §2.2）保留不动，便于审计与追溯

---

## 3. 业务规则

### 3.1 QR 生成时机

1. 仅在图纸版本**审批通过后**触发 QR 生成（见 [REQ-003-shared §3.2 审批通过逻辑](./REQ-003-shared.md)）
2. 审批驳回（`REJECTED`）或待审批（`PENDING`）的版本**不生成 QR**，确保流通的图纸均为经过审批的合法版本
3. 生成流程（审批通过事务完成后异步执行）：
   - 生成密码学安全的随机 `publicToken`（64 位 URL-safe Base64）
   - 构造公开状态页 URL：`{H5_BASE_URL}/public/drawing-status/{publicToken}`
   - 使用 QR 生成库将 URL 编码为 PNG 图片，写入 OSS，得到 `qrImageUrl`
   - 若原始文件为 PDF，将 QR 图片叠加至 PDF **每一页的右下角**（距离底边和右边 15mm，QR 尺寸 25mm × 25mm），另存为 `pdfWithQrUrl`
   - 写入 `qrGeneratedTime`
4. 生成失败时记录错误日志，不阻断审批流程；支持手动重试（见 §4.3）

### 3.2 QR 指向逻辑

- 每个审批通过的版本独立一个 QR，`publicToken` 与 `drawingVersionId` 稳定绑定
- 扫码用户打开的始终是**该版本**的状态，无论当前图纸的最新版本为何
- 版本状态由扫码时实时查询决定：

| 版本字段状态 | 公开状态 | 附加信息 |
|------|---------|---------|
| `isCurrent = true` 且 `isDeprecated = false` | ACTIVE | 返回该图纸 ACTIVE Markup 数量 |
| `isDeprecated = true` | DEPRECATED | 返回当前最新版本摘要（版本号 + 审批时间） |

### 3.3 下载与预览资源优先级

- Drawing 团队下载或预览 PDF 时，**优先返回 `pdfWithQrUrl`**（已带 QR 水印）
- `pdfWithQrUrl` 不存在（非 PDF 原始文件 或 生成失败未重试）时，降级使用 `fileUrl`
- QR 图片单独下载，返回 `qrImageUrl`

---

## 4. API 接口定义

### 4.1 获取公开状态（公开接口，无需登录）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/public/drawing/status/{publicToken}`
- **无需 Authorization / X-Tenant-Id / Project-Id**；仅可传 `Accept-Language` 以决定返回文案语言

**Path Params**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| publicToken | String | ✅ | 二维码中编码的 token |

**响应格式**: `CommonResult<PublicDrawingStatusRespVO>`

**ACTIVE 版本响应**:

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingCode": "ARCH-001",
    "drawingName": "首层平面图",
    "drawingCategory": "Structural",
    "versionNo": "V2",
    "approvedTime": "2026-04-10 14:30:00",
    "status": "ACTIVE",
    "activeMarkupCount": 3,
    "latestVersion": null
  }
}
```

**DEPRECATED 版本响应**:

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingCode": "ARCH-001",
    "drawingName": "首层平面图",
    "drawingCategory": "Structural",
    "versionNo": "V1",
    "approvedTime": "2026-03-20 16:00:00",
    "status": "DEPRECATED",
    "activeMarkupCount": null,
    "latestVersion": {
      "versionNo": "V3",
      "approvedTime": "2026-04-15 11:00:00"
    }
  }
}
```

**响应字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| drawingCode | String | 图纸编号 |
| drawingName | String | 图纸名称 |
| drawingCategory | String | 图纸分类（英文显示值，如 Structural） |
| versionNo | String | 被扫描版本的版本号 |
| approvedTime | DateTime | 被扫描版本的审批通过时间 |
| status | String | `ACTIVE` / `DEPRECATED` |
| activeMarkupCount | Integer \| null | 仅 ACTIVE 时返回：该图纸的活跃 Markup 数量；DEPRECATED 时为 null |
| latestVersion | Object \| null | 仅 DEPRECATED 时返回：当前最新有效版本摘要；ACTIVE 时为 null |
| latestVersion.versionNo | String | 最新版本号 |
| latestVersion.approvedTime | DateTime | 最新版本审批通过时间 |

**业务逻辑**:
- ✅ 根据 `publicToken` 查找对应 DrawingVersion
- ✅ 联查 Drawing 主记录获取 drawingCode / drawingName / drawingCategory
- ✅ 状态判定：
  - `isCurrent = true AND isDeprecated = false` → `ACTIVE`
  - `isDeprecated = true` → `DEPRECATED`
  - 其他（例如 `PENDING` / `REJECTED` 被异常访问）→ 视为无效 token，返回 1003006001
- ✅ ACTIVE 状态下，统计该图纸的 ACTIVE Markup 数量：`DrawingMarkup WHERE drawingId = {drawingId} AND status = ACTIVE`
- ✅ DEPRECATED 状态下，查询该图纸 `isCurrent = true` 的版本并返回摘要
- ✅ **不返回** 图纸文件 URL、附件、项目名称、人员姓名、tenantId 等敏感信息
- ✅ 所有时间字段返回 ISO 8601 格式，前端自行格式化

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003006001 | 二维码无效或已失效 | Invalid QR code |

> 为防止枚举攻击，token 不存在、版本状态异常、token 格式错误等情况**统一返回 1003006001**，不向外区分具体原因。

**限流规则**:
- 同一 IP 访问该接口：60 次 / 分钟
- 超出限流返回 HTTP 429

### 4.2 获取版本 QR 信息（登录接口，Drawing 团队使用）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/drawing/version/qr`
- Query Params: `drawingVersionId`（必填）
- 需登录，需携带 `Authorization` / `X-Tenant-Id` / `Project-Id`

**响应格式**: `CommonResult<DrawingQrRespVO>`

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingVersionId": 305,
    "versionNo": "V3",
    "qrImageUrl": "https://oss.xxx.com/qr/305.png",
    "pdfWithQrUrl": "https://oss.xxx.com/drawings/arch001-v3-qr.pdf",
    "qrGeneratedTime": "2026-04-01 14:35:00",
    "publicStatusUrl": "https://site.xxx.com/public/drawing-status/a1b2c3d4..."
  }
}
```

| 字段 | 说明 |
|------|------|
| qrImageUrl | QR 图片 PNG 下载地址 |
| pdfWithQrUrl | 带 QR 水印的 PDF 下载地址；若原始文件非 PDF，返回 null |
| qrGeneratedTime | QR 生成时间；未生成时为 null |
| publicStatusUrl | 扫码后用户打开的完整 URL，供 PC 端展示和复制 |

**权限**: `drawing:view`

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003006004 | 版本不存在 | Drawing version not found |
| 1003006005 | 版本未审批通过，暂无 QR | QR not available for this version |

### 4.3 重新生成 QR（登录接口，容错用途）

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/version/qr/regenerate`

**请求格式**:
```json
{ "drawingVersionId": 305 }
```

**响应格式**: `CommonResult<DrawingQrRespVO>`（结构同 §4.2）

**业务逻辑**:
- ✅ 校验版本已审批通过（`approvalStatus = APPROVED`）
- ✅ 保留原 `publicToken`（已印制流通中的图纸扫码仍需有效）
- ✅ 重新生成 QR 图片与带 QR 的 PDF，覆盖原 `qrImageUrl` 和 `pdfWithQrUrl`
- ✅ 更新 `qrGeneratedTime`

**权限**: `drawing:upload`（Drawing 团队）

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003006002 | 版本未审批通过，无法生成 QR | Cannot generate QR for unapproved version |
| 1003006003 | QR 生成失败 | QR generation failed |

---

## 5. 安全与合规

- 公开状态 API 不返回图纸文件、附件、内部 ID、项目名称、人员姓名
- `publicToken` 使用密码学安全的随机数生成（64 位 URL-safe Base64）
- H5 公开页不提供任何登录入口与注册引导
- `publicToken` 一经生成即长期有效，不提供显式失效机制（与图纸印制寿命保持一致）
- 重新生成 QR 时保留原 token，避免已印制图纸上的 QR 失效

---

## 6. 验收标准（跨端通用）

### QR 生成
- [ ] 版本审批通过后，DrawingVersion 的 `publicToken` / `qrImageUrl` / `qrGeneratedTime` 自动填充
- [ ] 原始文件为 PDF 时，`pdfWithQrUrl` 自动生成并可下载
- [ ] 非 PDF 文件（DWG / DXF / PNG / JPG）不生成 `pdfWithQrUrl`，但 `qrImageUrl` 仍生成
- [ ] QR 编码的 URL 指向 `/public/drawing-status/{publicToken}`
- [ ] 审批驳回（`REJECTED`）或待审批（`PENDING`）的版本不生成 QR

### 公开状态 API
- [ ] `/public/drawing/status/{publicToken}` 无需登录即可访问
- [ ] 响应不包含图纸文件 URL、附件 URL、人员姓名等敏感字段
- [ ] ACTIVE 版本正确返回 `activeMarkupCount`（统计值仅包含 ACTIVE 状态的 Markup）
- [ ] DEPRECATED 版本正确返回 `latestVersion` 摘要（版本号 + 审批时间）
- [ ] 无效 / 不存在 / 格式错误的 token 统一返回 1003006001
- [ ] 限流规则生效（60 次 / 分钟 / IP）

### QR 下载与重新生成
- [ ] `/drawing/version/qr` 接口正确返回 QR 信息
- [ ] `/drawing/version/qr/regenerate` 保留原 `publicToken`，仅刷新图片与 PDF
- [ ] 无 `drawing:upload` 权限的用户调用 regenerate 返回 403

### 资源优先级
- [ ] 图纸下载接口优先返回 `pdfWithQrUrl`，不存在时降级为 `fileUrl`
- [ ] 图纸在线预览（REQ-003 §4）同样优先加载 `pdfWithQrUrl`

---

## 7. 各端需求文档索引

| 端 | 需求文档 | 说明 |
|----|---------|------|
| PC 管理端 | [REQ-006-pc.md](../pc/REQ-006-pc.md) | QR 查看、下载、重新生成、状态指示 |
| H5 公开端 | [REQ-006-h5.md](../h5/REQ-006-h5.md) | 扫码后的公开状态页 |
| APP 移动端 | 不涉及 | 本功能不涉及 APP 端 |
