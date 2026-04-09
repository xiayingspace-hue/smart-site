# 后端开发说明文档 — REQ-003 图纸版本管理与查阅确认

> **来源需求**: [REQ-003-shared.md](../../requirements/shared/REQ-003-shared.md) + [REQ-003-app.md](../../requirements/app/REQ-003-app.md)
> **产品**: SMART SITE SYSTEM
> **服务模块**: `drawing-service`
> **生成日期**: 2026-04-08

---

## 1. 功能概述

实现工程图纸的版本管理、审批流程及 Site Engineer 查阅确认功能：

| 功能 | 说明 |
|------|------|
| 图纸 CRUD | 图纸基础信息维护 |
| 版本管理 | 每张图纸可有多个版本，当前 ACTIVE 版本唯一 |
| 审批流程 | DC 上传 → 分配审批人 → 审批通过/驳回 → 分配 SE |
| SE 分配 | 审批通过后指定负责的 Site Engineer 列表 |
| 查阅确认 | SE 确认已阅，记录 deviceInfo（PC/APP）|
| App Push | 审批通过后向已分配 SE 推送通知 |

---

## 2. 业务逻辑

### 2.1 图纸版本状态流转

```
DRAFT → PENDING_APPROVAL → ACTIVE
              └→ REJECTED → (重新上传) → PENDING_APPROVAL
```

| 状态 | 说明 |
|------|------|
| `DRAFT` | 已上传，尚未提交审批 |
| `PENDING_APPROVAL` | 已提交，等待审批人审批 |
| `ACTIVE` | 审批通过，当前有效版本 |
| `REJECTED` | 审批被驳回 |
| `ARCHIVED` | 被新版本取代，自动归档 |

### 2.2 审批通过后动作

```
1. 将当前版本状态置为 ACTIVE
2. 将同一图纸的其他版本置为 ARCHIVED
3. 向所有已分配 SE 发送 App Push（通过消息队列异步处理）
4. 创建 Notification 记录（type=DRAWING_UPDATED）
```

### 2.3 SE 查阅确认

- 接口幂等：`drawingVersionId + confirmerId` 联合唯一
- 记录 `deviceInfo`：`APP` 或 `PC`
- 确认后不可撤销
- 可多次调用（幂等，后续调用仅更新 `deviceInfo` 和 `confirmedAt`，或直接忽略）

### 2.4 SE 图纸列表过滤规则

```sql
WHERE status = 'ACTIVE'
  AND project_id = #{projectId}
  AND id IN (
    SELECT drawing_id FROM drawing_se_assignment
    WHERE se_user_id = #{currentUserId}
  )
```

---

## 3. 数据模型

### 3.1 drawing 表

```sql
CREATE TABLE `drawing` (
  `id`            BIGINT PRIMARY KEY AUTO_INCREMENT,
  `tenant_id`     BIGINT NOT NULL,
  `project_id`    BIGINT NOT NULL,
  `drawing_code`  VARCHAR(100) NOT NULL COMMENT '图纸编号',
  `drawing_name`  VARCHAR(200) NOT NULL COMMENT '图纸名称',
  `category`      VARCHAR(100) COMMENT '图纸分类',
  `created_at`    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at`    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`       TINYINT(1) NOT NULL DEFAULT 0,
  UNIQUE KEY `uk_code_project` (`project_id`, `drawing_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.2 drawing_version 表

```sql
CREATE TABLE `drawing_version` (
  `id`            BIGINT PRIMARY KEY AUTO_INCREMENT,
  `drawing_id`    BIGINT NOT NULL,
  `version_no`    VARCHAR(50) NOT NULL COMMENT '版本号，如 V1.0',
  `file_url`      VARCHAR(500) NOT NULL,
  `file_name`     VARCHAR(200) NOT NULL,
  `status`        VARCHAR(30) NOT NULL DEFAULT 'DRAFT',
  `approver_id`   BIGINT COMMENT '审批人 userId',
  `approved_at`   DATETIME,
  `reject_comment` TEXT COMMENT '驳回意见',
  `uploaded_by`   BIGINT NOT NULL,
  `uploaded_at`   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at`    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  KEY `idx_drawing_id` (`drawing_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.3 drawing_se_assignment 表

```sql
CREATE TABLE `drawing_se_assignment` (
  `id`          BIGINT PRIMARY KEY AUTO_INCREMENT,
  `drawing_id`  BIGINT NOT NULL,
  `version_id`  BIGINT NOT NULL,
  `se_user_id`  BIGINT NOT NULL,
  `assigned_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY `uk_version_se` (`version_id`, `se_user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.4 drawing_read_confirmation 表

```sql
CREATE TABLE `drawing_read_confirmation` (
  `id`                BIGINT PRIMARY KEY AUTO_INCREMENT,
  `drawing_version_id` BIGINT NOT NULL,
  `confirmer_id`      BIGINT NOT NULL,
  `confirmed_at`      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `device_info`       VARCHAR(20) COMMENT 'APP / PC',
  UNIQUE KEY `uk_version_confirmer` (`drawing_version_id`, `confirmer_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 4. API 设计

### 4.1 接口列表

- [x] `GET /drawing/page` — 图纸列表（DC/管理员）
- [x] `GET /drawing/se/page` — SE 专用图纸列表（已过滤 ACTIVE + 已分配）
- [x] `GET /drawing/get` — 图纸详情
- [x] `POST /drawing/create` — 新建图纸
- [x] `POST /drawing/version/upload` — 上传新版本
- [x] `POST /drawing/version/submit` — 提交审批
- [x] `POST /drawing/version/approve` — 审批通过
- [x] `POST /drawing/version/reject` — 审批驳回
- [x] `POST /drawing/se/assign` — 分配 SE
- [x] `POST /drawing/confirm` — 查阅确认（SE 用）

### 4.2 接口规范

#### GET /drawing/se/page

**Query Params**: `keyword`（drawingCode/drawingName 模糊）, `category`, `pageNo`, `pageSize`

**Response**（DrawingSeVO）:
```json
{
  "code": 0,
  "data": {
    "list": [
      {
        "id": 1,
        "drawingCode": "DWG-001",
        "drawingName": "基础平面图",
        "category": "结构",
        "currentVersionId": 5,
        "currentVersionNo": "V2.0",
        "currentVersionFileUrl": "https://oss.example.com/v2.pdf",
        "status": "ACTIVE",
        "confirmed": true,
        "confirmedAt": "2026-04-07T09:00:00",
        "unreadMarkupCount": 2,
        "updatedAt": "2026-04-06T15:00:00"
      }
    ],
    "total": 10
  }
}
```

#### POST /drawing/version/approve

**Request Body**:
```json
{ "versionId": 5 }
```

**后端操作**:
1. 校验当前用户是否为该版本指定的审批人
2. 更新 `drawing_version.status = ACTIVE`
3. 将其他版本置为 ARCHIVED
4. 异步（MQ）向已分配 SE 推送 Push 通知 + 写入 notification 表

#### POST /drawing/confirm

**Request Body**:
```json
{
  "drawingVersionId": 5,
  "deviceInfo": "APP"
}
```

**逻辑**: 执行 `INSERT IGNORE INTO drawing_read_confirmation ...`（幂等）

---

## 5. 非功能需求

| 要求 | 规范 |
|------|------|
| 审批推送 | 通过 RabbitMQ 异步处理，Push 失败不影响审批结果 |
| 文件存储 | 图纸文件存储于 OSS，接口仅存储 URL |
| 数据隔离 | 所有查询强制过滤 `tenant_id` + `project_id` |
| 幂等 | `drawing_read_confirmation` 唯一键保证确认幂等 |
| 并发 | 多 SE 同时确认同一图纸，唯一键冲突静默处理（catch Duplicate Entry）|

---

## 6. 验收条件

- [ ] SE 图纸列表仅返回 ACTIVE + 已分配给当前用户的图纸
- [ ] 审批通过后，当前版本变 ACTIVE，历史版本变 ARCHIVED
- [ ] 审批通过后异步向已分配 SE 推送通知
- [ ] 驳回后可重新上传新版本并再次提交审批
- [ ] 查阅确认接口幂等，重复调用不报错
- [ ] `device_info` 字段正确记录 APP / PC
- [ ] 未分配给该 SE 的图纸不在 SE 列表中显示
- [ ] SE 图纸列表中 `unreadMarkupCount` 字段正确统计
