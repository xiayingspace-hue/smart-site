# 后端开发说明文档 — REQ-006 图纸二维码与公开状态查询

> **来源需求**: [REQ-006-shared.md](../../requirements/shared/REQ-006-shared.md) + [REQ-006-pc.md](../../requirements/pc/REQ-006-pc.md) + [REQ-006-h5.md](../../requirements/h5/REQ-006-h5.md)
> **产品**: SMART SITE SYSTEM
> **服务模块**: `drawing-service`
> **依赖**: BACKEND-REQ-003（图纸版本审批流程）、BACKEND-REQ-004（局部更新数据模型）
> **生成日期**: 2026-04-17

---

## 1. 功能概述

实现图纸版本审批通过后的二维码自动生成、PDF 叠加，以及面向非系统用户的公开状态查询接口。

| 功能 | 说明 |
|------|------|
| QR 自动生成 | 版本审批通过后异步生成 publicToken、QR 图片、带 QR 的 PDF |
| QR 叠加 PDF | 将 QR 图片叠加至 PDF 每一页右下角，另存为独立文件 |
| 公开状态 API | 无需登录，根据 publicToken 返回版本状态（ACTIVE / DEPRECATED） |
| QR 信息查询 | Drawing 团队登录后查看 QR 图片、下载链接、公开 URL |
| QR 重新生成 | 生成失败时手动重试，保留原 publicToken |
| PDF 下载优先级 | 下载/预览时优先返回 `pdfWithQrUrl`，降级为 `fileUrl` |

---

## 2. 数据模型

### 2.1 DrawingVersion 表扩展字段

在现有 `drawing_version` 表（REQ-003）中新增以下字段：

```sql
ALTER TABLE `drawing_version` ADD COLUMN `public_token` VARCHAR(64) DEFAULT NULL COMMENT '公开访问令牌，64位URL-safe Base64';
ALTER TABLE `drawing_version` ADD COLUMN `qr_image_url` VARCHAR(500) DEFAULT NULL COMMENT 'QR二维码图片OSS地址（PNG，512×512）';
ALTER TABLE `drawing_version` ADD COLUMN `pdf_with_qr_url` VARCHAR(500) DEFAULT NULL COMMENT '叠加QR后的PDF文件OSS地址';
ALTER TABLE `drawing_version` ADD COLUMN `qr_generated_time` DATETIME DEFAULT NULL COMMENT 'QR生成完成时间戳';

ALTER TABLE `drawing_version` ADD UNIQUE INDEX `uk_public_token` (`public_token`);
```

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 公开访问令牌 | public_token | VARCHAR(64) | 自动 | 密码学安全随机令牌，唯一索引，防枚举 |
| QR 图片 URL | qr_image_url | VARCHAR(500) | 自动 | QR 二维码 PNG 图片 OSS 地址（512×512） |
| 带 QR 的 PDF URL | pdf_with_qr_url | VARCHAR(500) | 自动 | 叠加 QR 后的 PDF OSS 地址；非 PDF 原始文件时为 NULL |
| QR 生成时间 | qr_generated_time | DATETIME | 自动 | 为 NULL 表示未生成或生成失败 |

---

## 3. 业务逻辑

### 3.1 QR 生成流程（审批通过后异步触发）

审批通过事务提交后，发送 MQ 消息触发 QR 生成：

```
审批通过事务完成
       │
       ▼
  发送 MQ 消息: { drawingVersionId, drawingId, tenantId }
       │
       ▼
  QR 生成 Consumer 消费消息：
       │
       ├── 1. 生成 publicToken（64 位 URL-safe Base64，SecureRandom）
       ├── 2. 构造公开状态页 URL：{H5_BASE_URL}/public/drawing-status/{publicToken}
       ├── 3. QR 编码 → PNG（512×512），上传 OSS → 得到 qrImageUrl
       ├── 4. 判断原始文件是否为 PDF：
       │      ├── 是 → 下载原始 PDF，叠加 QR 到每一页右下角（距底边/右边 15mm，QR 25mm×25mm），上传 OSS → 得到 pdfWithQrUrl
       │      └── 否 → pdfWithQrUrl = NULL
       ├── 5. 更新 drawing_version 记录：写入 public_token、qr_image_url、pdf_with_qr_url、qr_generated_time
       └── 6. 生成失败 → 记录错误日志，不阻断审批流程
```

### 3.2 QR 生成条件约束

| 条件 | 行为 |
|------|------|
| `approvalStatus = APPROVED` | 触发 QR 生成 |
| `approvalStatus = PENDING` | 不生成 QR |
| `approvalStatus = REJECTED` | 不生成 QR |
| 原始文件为 PDF | 生成 `qrImageUrl` + `pdfWithQrUrl` |
| 原始文件为 DWG / DXF / PNG / JPG | 仅生成 `qrImageUrl`，`pdfWithQrUrl` = NULL |

### 3.3 公开状态判定逻辑

```java
// 伪代码
PublicDrawingStatusRespVO getPublicStatus(String publicToken) {
    DrawingVersion version = findByPublicToken(publicToken);
    if (version == null || version.approvalStatus != APPROVED) {
        throw new BusinessException(1003006001, "Invalid QR code");
    }
    
    Drawing drawing = findDrawingById(version.drawingId);
    
    if (version.isCurrent && !version.isDeprecated) {
        // ACTIVE
        int markupCount = countActiveMarkups(drawing.id);
        return buildActiveResponse(drawing, version, markupCount);
    } else if (version.isDeprecated) {
        // DEPRECATED
        DrawingVersion latest = findCurrentVersion(drawing.id);
        return buildDeprecatedResponse(drawing, version, latest);
    } else {
        // 异常状态（PENDING/REJECTED 被意外访问）
        throw new BusinessException(1003006001, "Invalid QR code");
    }
}
```

### 3.4 PDF 下载优先级规则

Drawing 团队下载或预览 PDF 时（REQ-003 相关接口），应用以下优先级：

1. `pdfWithQrUrl` 存在且可访问 → 返回 `pdfWithQrUrl`
2. `pdfWithQrUrl` 为 NULL 或不可访问 → 降级返回 `fileUrl`

### 3.5 重新生成逻辑

- 保留原 `publicToken`（已流通图纸上的 QR 仍需有效）
- 重新生成 QR 图片和带 QR 的 PDF，覆盖原 `qrImageUrl` 和 `pdfWithQrUrl`
- 更新 `qrGeneratedTime`

---

## 4. API 设计

### 4.1 接口列表

- [x] `GET /public/drawing/status/{publicToken}` — 公开状态查询（**无需登录**）
- [x] `GET /drawing/version/qr` — 获取版本 QR 信息（需登录）
- [x] `POST /drawing/version/qr/regenerate` — 重新生成 QR（需登录）

### 4.2 GET /public/drawing/status/{publicToken}（公开接口）

**认证**: 无需 Authorization / X-Tenant-Id / Project-Id

**Path Params**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| publicToken | String | ✅ | 二维码中编码的 token |

**请求头**: 可传 `Accept-Language`（`zh-*` → 中文，其他 → 英文）

**Response — ACTIVE**:

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingCode": "ARCH-001",
    "drawingName": "首层平面图",
    "drawingCategory": "Structural",
    "versionNo": "V2",
    "approvedTime": "2026-04-10T14:30:00",
    "status": "ACTIVE",
    "activeMarkupCount": 3,
    "latestVersion": null
  }
}
```

**Response — DEPRECATED**:

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingCode": "ARCH-001",
    "drawingName": "首层平面图",
    "drawingCategory": "Structural",
    "versionNo": "V1",
    "approvedTime": "2026-03-20T16:00:00",
    "status": "DEPRECATED",
    "activeMarkupCount": null,
    "latestVersion": {
      "versionNo": "V3",
      "approvedTime": "2026-04-15T11:00:00"
    }
  }
}
```

**业务校验**:
- 根据 `publicToken` 查找 DrawingVersion（使用唯一索引）
- Token 不存在 / 版本状态异常 / 格式错误 → 统一返回 1003006001（防枚举）
- ACTIVE 状态：统计 `drawing_markup WHERE drawing_id = ? AND status = 'ACTIVE'`
- DEPRECATED 状态：查询 `drawing_version WHERE drawing_id = ? AND is_current = true`
- **禁止返回**：文件 URL、附件 URL、项目名称、人员姓名、tenantId 等敏感信息
- 所有时间字段返回 ISO 8601 格式

**限流规则**:
- 同一 IP：60 次 / 分钟
- 超出限流返回 HTTP 429

### 4.3 GET /drawing/version/qr（登录接口）

**Query Params**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| drawingVersionId | Long | ✅ | 图纸版本 ID |

**权限**: `drawing:view`

**Response**:

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingVersionId": 305,
    "versionNo": "V3",
    "qrImageUrl": "https://oss.xxx.com/qr/305.png",
    "pdfWithQrUrl": "https://oss.xxx.com/drawings/arch001-v3-qr.pdf",
    "qrGeneratedTime": "2026-04-01T14:35:00",
    "publicStatusUrl": "https://site.xxx.com/public/drawing-status/a1b2c3d4..."
  }
}
```

| 字段 | 说明 |
|------|------|
| qrImageUrl | QR 图片 PNG 下载地址 |
| pdfWithQrUrl | 带 QR 水印的 PDF 下载地址；原始文件非 PDF 时返回 null |
| qrGeneratedTime | QR 生成时间；未生成时为 null |
| publicStatusUrl | 扫码后的完整 URL，供 PC 端展示和复制 |

### 4.4 POST /drawing/version/qr/regenerate（登录接口）

**权限**: `drawing:upload`

**Request Body**:

```json
{ "drawingVersionId": 305 }
```

**业务逻辑**:
- 校验版本 `approvalStatus = APPROVED`，否则返回 1003006002
- **保留原 publicToken**
- 重新生成 QR 图片 + 带 QR 的 PDF，覆盖原 OSS 文件
- 更新 `qrGeneratedTime`
- 生成失败返回 1003006003

**Response**: 结构同 §4.3

---

## 5. 错误码

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003006001 | 二维码无效或已失效（公开接口统一错误） | Invalid QR code |
| 1003006002 | 版本未审批通过，无法生成 QR | Cannot generate QR for unapproved version |
| 1003006003 | QR 生成失败（OSS / PDF 处理异常） | QR generation failed |
| 1003006004 | 版本不存在 | Drawing version not found |
| 1003006005 | 版本未审批通过，暂无 QR 信息 | QR not available for this version |

> **安全要求**：公开接口对 token 不存在、格式错误、状态异常等情况**统一返回 1003006001**，不向外区分具体原因。

---

## 6. 技术实现要点

### 6.1 publicToken 生成

```java
// 使用 SecureRandom 生成 48 字节（编码后约 64 字符 URL-safe Base64）
byte[] bytes = new byte[48];
new SecureRandom().nextBytes(bytes);
String token = Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
```

### 6.2 QR 图片生成

- 推荐使用 `com.google.zxing` 库
- 输出格式：PNG，512 × 512px
- 编码内容：完整公开状态页 URL
- 生成后上传至 OSS，记录 URL

### 6.3 PDF QR 叠加

- 推荐使用 `Apache PDFBox` 或 `iText`
- 从 OSS 下载原始 PDF → 在每一页右下角叠加 QR → 另存上传 OSS
- QR 位置：距底边 15mm，距右边 15mm；QR 尺寸 25mm × 25mm
- 原始 `fileUrl` 保留不动，叠加后的文件写入 `pdfWithQrUrl`

### 6.4 异步处理

- 审批通过事务完成后发送 RabbitMQ 消息
- Consumer 消费后执行 QR 生成全流程
- 失败重试策略：最多重试 3 次，间隔 5s / 30s / 120s
- 超过重试次数进入死信队列，记录错误日志，等待手动 Regenerate

### 6.5 限流实现

- 公开接口 `/public/drawing/status/{publicToken}` 使用 Redis 滑动窗口限流
- Key: `rate_limit:public_drawing_status:{ip}`
- 限制：60 次 / 分钟
- 超出返回 HTTP 429 Too Many Requests

---

## 7. 非功能需求

| 要求 | 规范 |
|------|------|
| 安全 | 公开接口不返回文件 URL、附件、内部 ID、项目名称、人员姓名；publicToken 使用密码学安全随机数 |
| 性能 | 公开状态查询 P95 < 200ms；QR 异步生成不影响审批操作响应时间 |
| 可靠性 | QR 生成失败不阻断审批流程；MQ 消息失败重试 3 次 + 死信队列 |
| 数据一致性 | publicToken 一经生成即长期有效，regenerate 时保留原 token |
| 限流 | 公开接口每 IP 60 次/分钟，防止滥用和枚举攻击 |
| 存储 | QR 图片和带 QR 的 PDF 存储在 OSS，与原始文件独立管理 |

---

## 8. 验收条件

### QR 自动生成
- [ ] 版本审批通过后，`drawing_version` 的 `public_token`、`qr_image_url`、`qr_generated_time` 自动填充
- [ ] 原始文件为 PDF 时，`pdf_with_qr_url` 自动生成且可下载
- [ ] 非 PDF 文件（DWG / DXF / PNG / JPG）不生成 `pdf_with_qr_url`，但 `qr_image_url` 仍生成
- [ ] QR 编码的 URL 指向正确的公开状态页地址
- [ ] PENDING / REJECTED 版本不触发 QR 生成
- [ ] QR 生成失败不阻断审批事务

### 公开状态 API
- [ ] `/public/drawing/status/{publicToken}` 无需登录即可正常访问
- [ ] 响应不包含文件 URL、附件 URL、人员姓名等敏感字段
- [ ] ACTIVE 版本正确返回 `activeMarkupCount`
- [ ] DEPRECATED 版本正确返回 `latestVersion` 摘要
- [ ] 无效 / 不存在 / 格式错误的 token 统一返回 1003006001
- [ ] 限流规则生效（60 次 / 分钟 / IP），超出返回 HTTP 429

### QR 查询与重新生成
- [ ] `/drawing/version/qr` 接口正确返回 QR 信息，权限 `drawing:view`
- [ ] `/drawing/version/qr/regenerate` 保留原 `publicToken`，仅刷新图片与 PDF
- [ ] 无 `drawing:upload` 权限的用户调用 regenerate 返回 403
- [ ] 未审批版本调用 regenerate 返回 1003006002

### PDF 下载优先级
- [ ] 图纸下载接口优先返回 `pdfWithQrUrl`，不存在时降级为 `fileUrl`
- [ ] 图纸在线预览同样优先加载 `pdfWithQrUrl`
