# 后端开发说明文档 — REQ-004 图纸标注（Markup）管理

> **来源需求**: [REQ-004-shared.md](../../requirements/shared/REQ-004-shared.md) + [REQ-004-pc.md](../../requirements/pc/REQ-004-pc.md) + [REQ-004-app.md](../../requirements/app/REQ-004-app.md)
> **产品**: SMART SITE SYSTEM
> **服务模块**: `drawing-service`
> **生成日期**: 2026-04-08

---

## 1. 功能概述

实现 DC 对工程图纸发布标注（Markup）、SE 查看及确认已读，以及 DC 合并多条标注为新图纸版本的功能：

| 功能 | 说明 |
|------|------|
| 发布标注 | DC 在图纸上发布带标题/描述/影响区域/附件的 Markup |
| 标注列表 | 分 All / Active / Merged 三类查看 |
| 删除标注 | DC 可删除（已 Merge 的不可删除）|
| 合并到新版本 | 选择多个 ACTIVE Markup → 生成新图纸版本 |
| SE 确认已读 | SE 查看 Markup 后标记为已读（幂等）|
| Push 通知 | 发布 Markup 后异步推送给已分配 SE |

---

## 2. 业务逻辑

### 2.1 Markup 状态

| 状态 | 说明 |
|------|------|
| `ACTIVE` | 已发布，有效 |
| `MERGED` | 已被纳入新版本合并 |
| `DELETED` | 已被 DC 手动删除 |

### 2.2 发布 Markup 流程

```
1. DC 填写 Title（必填，≤200）、Description（必填，≤2000）、Affected Area（可选，≤500）
2. 上传附件（最多 10 个，单个 ≤20MB，支持 PDF/PNG/JPG）
3. 后端保存 drawing_markup 记录，status=ACTIVE
4. 异步（MQ）推送 Push 通知给所有已分配 SE
5. 写入 notification 表（type=MARKUP_PUBLISHED）
```

### 2.3 合并到新版本流程

```
1. DC 选择 ≥1 个 ACTIVE Markup
2. 指定审批人、上传新版本文件、填写版本说明
3. 后端：
   a. 创建新 drawing_version（status=PENDING_APPROVAL）
   b. 将选中 Markup 状态改为 MERGED（设置 merged_into_version_id）
   c. 审批流程继续（同 REQ-003）
```

### 2.4 SE Markup 已读确认

- 接口幂等：`markup_id + confirmer_id` 联合唯一
- 发布 Markup 时后端为所有已分配 SE 预创建 `drawing_markup_read_record` 记录（is_read=false）
- SE 调用 `POST /drawing/markup/confirm` 时更新 is_read=true + confirmed_at + device_info

---

## 3. 数据模型

### 3.1 drawing_markup 表

```sql
CREATE TABLE `drawing_markup` (
  `id`                      BIGINT PRIMARY KEY AUTO_INCREMENT,
  `drawing_id`              BIGINT NOT NULL,
  `drawing_version_id`      BIGINT NOT NULL COMMENT '发布时对应的图纸版本',
  `title`                   VARCHAR(200) NOT NULL,
  `description`             TEXT NOT NULL,
  `affected_area`           VARCHAR(500),
  `attachments`             JSON COMMENT '[{fileName, fileUrl, fileType, fileSize}]，最多 10 个',
  `status`                  VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
  `merged_into_version_id`  BIGINT COMMENT '合并进入的版本 ID',
  `published_by`            BIGINT NOT NULL,
  `published_by_name`       VARCHAR(64),
  `publish_time`            DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `deleted`                 TINYINT(1) NOT NULL DEFAULT 0,
  KEY `idx_drawing_id` (`drawing_id`),
  KEY `idx_version_id` (`drawing_version_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.2 drawing_markup_read_record 表

```sql
CREATE TABLE `drawing_markup_read_record` (
  `id`           BIGINT PRIMARY KEY AUTO_INCREMENT,
  `markup_id`    BIGINT NOT NULL,
  `confirmer_id` BIGINT NOT NULL COMMENT 'SE userId',
  `is_read`      TINYINT(1) NOT NULL DEFAULT 0,
  `confirmed_at` DATETIME,
  `device_info`  VARCHAR(20) COMMENT 'APP / PC',
  UNIQUE KEY `uk_markup_confirmer` (`markup_id`, `confirmer_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 4. API 设计

### 4.1 接口列表

- [x] `POST /drawing/markup/publish` — 发布标注
- [x] `GET /drawing/markup/list` — 标注列表
- [x] `GET /drawing/markup/get` — 标注详情
- [x] `DELETE /drawing/markup/delete` — 删除标注
- [x] `POST /drawing/markup/merge` — 合并到新版本
- [x] `POST /drawing/markup/confirm` — SE 确认已读

### 4.2 接口规范

#### POST /drawing/markup/publish

**Request Body**:
```json
{
  "drawingId": 1,
  "drawingVersionId": 5,
  "title": "结构修改说明",
  "description": "详细描述...",
  "affectedArea": "A区 3-5轴",
  "attachments": [
    { "fileName": "change-notice.pdf", "fileUrl": "https://...", "fileType": "pdf", "fileSize": 1048576 }
  ]
}
```

**Response**: `{ "code": 0, "data": { "markupId": 10 } }`

#### GET /drawing/markup/list

**Query Params**:

| 参数 | 说明 |
|------|------|
| `drawingId` | 图纸 ID（必填）|
| `status` | `ACTIVE` / `MERGED` / `ALL` |
| `pageNo` | 页码 |
| `pageSize` | 每页条数 |

**Response** 中每条 Markup 包含 `isRead`（对当前 SE 是否已读）字段。

#### POST /drawing/markup/merge

**Request Body**:
```json
{
  "drawingId": 1,
  "markupIds": [10, 11, 12],
  "newVersionFileUrl": "https://oss.example.com/v3.pdf",
  "newVersionFileName": "DWG-001-V3.pdf",
  "versionNote": "根据标注 #10 #11 #12 修订",
  "approverId": 200
}
```

**后端校验**:
- `markupIds` 全部为 ACTIVE 状态
- `markupIds` 属于同一 `drawingId`

#### DELETE /drawing/markup/delete

**Query Params**: `markupId`

**后端校验**:
- Markup 状态必须为 ACTIVE（MERGED 不可删除，返回 1003004001）
- 仅 DC 角色可操作

#### POST /drawing/markup/confirm

**Request Body**:
```json
{
  "markupId": 10,
  "deviceInfo": "APP"
}
```

**逻辑**: 更新 `drawing_markup_read_record`，`INSERT IGNORE` 或 `ON DUPLICATE KEY UPDATE`

---

## 5. 非功能需求

| 要求 | 规范 |
|------|------|
| 附件存储 | 文件上传至 OSS，接口仅存 URL + 元数据 |
| 推送异步 | MQ 异步推送，失败重试 3 次 |
| 已读幂等 | 唯一键 `(markup_id, confirmer_id)` 保证幂等 |
| 合并原子性 | 创建新版本与标注状态更新在同一事务中 |
| 附件限制 | 超过 10 个附件返回 400；单文件超 20MB 在上传接口校验 |

---

## 6. 验收条件

- [ ] 发布 Markup 后，已分配 SE 收到 App Push
- [ ] Markup 列表 `ACTIVE` 过滤正确
- [ ] `isRead` 字段对当前 SE 返回正确
- [ ] 确认已读后 `isRead=true`，重复调用不报错
- [ ] 合并成功后选中 Markup 状态变为 `MERGED`
- [ ] 尝试删除 `MERGED` 状态 Markup 返回 1003004001
- [ ] 合并事务失败时，新版本和 Markup 状态均回滚
- [ ] 附件数量超 10 时返回 400
