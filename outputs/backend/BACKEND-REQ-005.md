---
doc_type: backend_spec
source_req: REQ-005-pc (v0.3.5) + REQ-005-app (v0.1.0)
service_module: drawing-service / notification-service
generated_at: 2026-05-28
---

# 后端说明文档：PC 端 Site Engineer 图纸查阅与局部更新查看

> 本文档依据 [REQ-005-pc.md v0.3.5](../../../requirements/pc/REQ-005-pc.md) 生成，描述 SE 视角下图纸查阅、确认与局部更新相关的后端接口、数据模型和业务逻辑。

---

## 1. 功能概述

| 功能模块 | 说明 |
|---------|------|
| SE 图纸列表接口 | 返回已分配给当前用户的 DrawingVersion 列表，含 RFA 信息、Markups 计数（总量+未读），不暴露系统版本号 |
| SE 图纸详情接口 | 返回单个 DrawingVersion 详情，含文件 URL、确认状态、Markups 计数 |
| Confirm Reading | 写入 DrawingConfirmation 记录，keyed on (drawingVersionId, userId)，跨端幂等 |
| Markup 列表接口 | 返回指定 DrawingVersion 的 ACTIVE Markup 列表，含已读状态（per user） |
| Mark as Read | 写入 MarkupConfirmation 记录，keyed on (markupId, userId)，幂等 |
| 站内通知 | Markup 发布时批量创建 InAppNotification；维护未读计数；支持标记已读 |

---

## 2. 业务逻辑

### 2.1 SE 图纸列表过滤规则

SE 专用接口必须同时满足以下两个条件：

1. `drawing_version.status = 'ACTIVE'`（仅审批通过的版本）
2. `drawing_assignment.user_id = currentUserId`（仅已分配给当前用户）

> 前端无法绕过后端过滤。`versionNo` 字段**不在响应中返回**，避免在 SE 视图中暴露内部版本信息。

### 2.2 Markups 计数规则

每个 DrawingVersion 返回两个 Markup 计数字段：

| 字段 | 说明 |
|------|------|
| `markupsTotal` | 该 DrawingVersion 下 `status = ACTIVE` 的 DrawingMarkup 总数 |
| `markupsUnread` | 同上，但排除当前用户已在 MarkupConfirmation 中有记录的 Markup |

```sql
-- markupsTotal
SELECT COUNT(*) FROM drawing_markup
WHERE drawing_version_id = #{drawingVersionId} AND status = 'ACTIVE'

-- markupsUnread（per user）
SELECT COUNT(*) FROM drawing_markup dm
WHERE dm.drawing_version_id = #{drawingVersionId}
  AND dm.status = 'ACTIVE'
  AND NOT EXISTS (
    SELECT 1 FROM markup_confirmation mc
    WHERE mc.markup_id = dm.id AND mc.user_id = #{userId}
  )
```

### 2.3 Confirm Reading 幂等规则

- `drawing_confirmation` 表对 `(drawing_version_id, user_id)` 建立唯一索引
- 重复提交时返回 200（幂等），不报错
- 跨端场景：PC 与 APP 的确认共享同一条记录，`device_info` 记录首次确认的来源

### 2.4 Markup 发布时通知触发逻辑

Admin 发布 DrawingMarkup 后，系统执行：

1. 查询 `drawing_assignment` 获取该 DrawingVersion 对应的所有已分配 SE 用户
2. 为每个 SE 用户插入一条 `in_app_notification` 记录（type=`MARKUP_PUBLISHED`）
3. 通知的 `target_route` 字段填写 `/drawings/{drawingVersionId}?tab=markups`
4. 批量插入使用 MQ 异步处理，失败重试 3 次

---

## 3. 数据模型

### 3.1 drawing_version（DrawingVersion，列表主体）

```sql
CREATE TABLE drawing_version (
  id               BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id        BIGINT NOT NULL,
  description      VARCHAR(500) NOT NULL COMMENT '批次描述，SE 列表主标识',
  category         VARCHAR(100) NOT NULL COMMENT '图纸分类',
  rfa_no           VARCHAR(100) DEFAULT NULL COMMENT '外部审批报审编号',
  subject_of_rfa   VARCHAR(500) DEFAULT NULL COMMENT '外部审批报审主题',
  file_url         VARCHAR(1000) NOT NULL COMMENT '图纸文件 OSS URL',
  status           VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT 'PENDING/ACTIVE/ARCHIVED',
  version_no       VARCHAR(20) NOT NULL COMMENT '内部版本号（不对 SE 暴露）',
  created_by       BIGINT NOT NULL,
  created_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted          TINYINT(1) NOT NULL DEFAULT 0
);
```

### 3.2 drawing_assignment（分配关系）

```sql
CREATE TABLE drawing_assignment (
  id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id           BIGINT NOT NULL,
  drawing_version_id  BIGINT NOT NULL,
  user_id             BIGINT NOT NULL,
  assigned_at         DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uq_assignment (drawing_version_id, user_id)
);
```

### 3.3 drawing_confirmation（查阅确认）

```sql
CREATE TABLE drawing_confirmation (
  id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id           BIGINT NOT NULL,
  drawing_version_id  BIGINT NOT NULL COMMENT 'DrawingVersion ID（一次提交记录）',
  user_id             BIGINT NOT NULL,
  confirmed_at        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  device_info         VARCHAR(50) NOT NULL COMMENT 'PC 或 APP',
  UNIQUE KEY uq_confirmation (drawing_version_id, user_id)
);
```

### 3.4 drawing_markup（局部更新）

```sql
CREATE TABLE drawing_markup (
  id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id           BIGINT NOT NULL,
  drawing_version_id  BIGINT NOT NULL COMMENT '所属 DrawingVersion',
  description         VARCHAR(500) NOT NULL COMMENT 'Markup 标题',
  content             TEXT COMMENT '说明文字',
  affected_area       VARCHAR(500) DEFAULT NULL COMMENT '影响区域（管理员可见，SE 视图不返回）',
  submission_no       VARCHAR(100) DEFAULT NULL COMMENT '报审号',
  applied_page_no     INT DEFAULT NULL COMMENT '所在页码；null 表示未指定',
  publish_time        DATETIME NOT NULL,
  creator_id          BIGINT NOT NULL,
  creator_name        VARCHAR(100) NOT NULL COMMENT '冗余字段，避免 JOIN',
  status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' COMMENT 'ACTIVE/MERGED',
  created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 3.5 markup_confirmation（Markup 已读确认）

```sql
CREATE TABLE markup_confirmation (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id   BIGINT NOT NULL,
  markup_id   BIGINT NOT NULL,
  user_id     BIGINT NOT NULL,
  confirmed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  device_info VARCHAR(50) NOT NULL,
  UNIQUE KEY uq_markup_confirmation (markup_id, user_id)
);
```

### 3.6 in_app_notification（站内通知）

```sql
CREATE TABLE in_app_notification (
  id            BIGINT PRIMARY KEY AUTO_INCREMENT,
  tenant_id     BIGINT NOT NULL,
  recipient_id  BIGINT NOT NULL COMMENT '接收用户 ID（SE）',
  type          VARCHAR(50) NOT NULL COMMENT 'MARKUP_PUBLISHED',
  title         VARCHAR(500) NOT NULL,
  body          TEXT,
  entity_type   VARCHAR(50) COMMENT 'DRAWING_MARKUP',
  entity_id     BIGINT COMMENT 'DrawingMarkup ID',
  target_route  VARCHAR(500) COMMENT '跳转路由，如 /drawings/101?tab=markups',
  is_read       TINYINT(1) NOT NULL DEFAULT 0,
  read_at       DATETIME DEFAULT NULL,
  created_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_recipient (recipient_id, is_read),
  INDEX idx_created (created_at)
);
```

---

## 4. API 设计

### 4.1 接口列表

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/drawing/se/page` | SE 图纸列表（分页） |
| GET | `/drawing/se/get` | SE DrawingVersion 详情 |
| POST | `/drawing/confirm` | 提交 Confirm Reading |
| GET | `/drawing/markup/list` | SE 获取 Markup 列表 |
| POST | `/drawing/markup/confirm` | Mark as Read |
| GET | `/notification/list` | 通知列表（分页） |
| GET | `/notification/unread-count` | 未读通知数量 |
| PATCH | `/notification/{id}/read` | 标记单条通知已读 |
| PATCH | `/notification/read-all` | 全部标记已读 |

### 4.2 GET /drawing/se/page

SE 专用图纸列表。后端自动过滤：`status=ACTIVE` + `drawing_assignment.user_id=currentUserId`。

**Query Params**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `keyword` | String | Description 模糊搜索（可选） |
| `category` | String | 分类筛选（可选） |
| `pageNo` | Integer | 页码，默认 1 |
| `pageSize` | Integer | 每页条数，默认 20 |

**Response**:

```json
{
  "code": 0,
  "data": {
    "list": [
      {
        "id": 101,
        "description": "首层平面图施工图纸",
        "category": "Architectural",
        "rfaNo": "RFA-001",
        "subjectOfRfa": "首层平面图报审",
        "status": "ACTIVE",
        "confirmed": true,
        "confirmedAt": "2026-04-08T09:00:00",
        "markupsTotal": 5,
        "markupsUnread": 2,
        "updatedAt": "2026-04-01T08:00:00"
      }
    ],
    "total": 8,
    "pageNo": 1,
    "pageSize": 20
  }
}
```

> `versionNo` 字段**不在响应中返回**。

### 4.3 GET /drawing/se/get

**Query Params**: `drawingVersionId`（Long）

**Response**:

```json
{
  "code": 0,
  "data": {
    "id": 101,
    "description": "首层平面图施工图纸",
    "category": "Architectural",
    "rfaNo": "RFA-001",
    "subjectOfRfa": "首层平面图报审",
    "fileUrl": "https://cdn.example.com/drawings/101.pdf",
    "confirmed": false,
    "confirmedAt": null,
    "markupsTotal": 5,
    "markupsUnread": 2,
    "updatedAt": "2026-04-01T08:00:00"
  }
}
```

**后端校验**: 验证 DrawingAssignment 存在（当前用户 + drawingVersionId），否则返回 403。

### 4.4 POST /drawing/confirm

**Request Body**:

```json
{
  "drawingVersionId": 101,
  "deviceInfo": "PC"
}
```

**后端逻辑**:

1. 校验 DrawingAssignment 存在
2. 使用 `INSERT IGNORE` 或 `INSERT ... ON DUPLICATE KEY UPDATE` 写入 drawing_confirmation（幂等）
3. 返回 200 + 确认时间

**Response**:

```json
{
  "code": 0,
  "data": { "confirmedAt": "2026-05-28T10:00:00" }
}
```

### 4.5 GET /drawing/markup/list

**Query Params**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `drawingVersionId` | Long | DrawingVersion ID |
| `status` | String | 固定传 `ACTIVE` |

**Response**:

```json
{
  "code": 0,
  "data": {
    "list": [
      {
        "id": 201,
        "description": "A轴节点详图修正",
        "content": "A轴与3轴交叉节点详图已更新，新增钢筋排布说明。",
        "submissionNo": "SUB-2026-003",
        "appliedPageNo": 3,
        "publishTime": "2026-04-08T09:00:00",
        "creatorName": "张三",
        "status": "ACTIVE",
        "confirmed": false,
        "attachments": [
          { "name": "node-detail.pdf", "url": "https://cdn.example.com/attachments/node-detail.pdf" }
        ]
      }
    ]
  }
}
```

> `affectedArea` 字段在 SE 接口响应中**不返回**（仅管理员接口返回）。

### 4.6 POST /drawing/markup/confirm

**Request Body**:

```json
{
  "markupId": 201,
  "deviceInfo": "PC"
}
```

**后端逻辑**:

1. 校验 Markup 所属 DrawingVersion 已分配给当前用户
2. 幂等写入 markup_confirmation（`INSERT IGNORE`）
3. 写入后 drawing_version 的 markupsUnread 计数自动更新（查询时实时计算或缓存失效）

### 4.7 站内通知接口

| 接口 | 说明 |
|------|------|
| `GET /notification/list` | 分页，支持 `isRead` 筛选；响应含 `unreadTotal` |
| `GET /notification/unread-count` | 返回 `{ count: N }` |
| `PATCH /notification/{id}/read` | 校验 `recipient_id == currentUserId`（防越权） |
| `PATCH /notification/read-all` | 批量 UPDATE `is_read=1` where `recipient_id=#{userId}` |

---

## 5. 错误码

| 错误码 | 说明 |
|--------|------|
| 1003005001 | DrawingVersion 不存在 |
| 1003005002 | 当前用户未被分配该图纸（DrawingAssignment 不存在） |
| 1003005003 | Markup 不存在或状态非 ACTIVE |
| 1003005004 | 通知不存在或无权操作 |
| 1003005005 | 确认操作重复提交（幂等场景返回 200，此码预留用于严格校验） |

---

## 6. 非功能需求

| 要求 | 规范 |
|------|------|
| SE 列表性能 | 单用户 ≤ 200 条图纸时 P95 < 500ms；markupsUnread 使用子查询或 Redis 缓存（TTL=60s） |
| 幂等 | `drawing_confirmation` 与 `markup_confirmation` 均有唯一索引保证幂等 |
| 越权防护 | 所有 SE 接口强制校验 DrawingAssignment；通知接口校验 recipient_id |
| 通知推送可靠性 | MQ 异步推送，失败重试 3 次，写入死信队列告警 |
| 数据保留 | in_app_notification 保留 90 天，超期自动归档 |
| 审计 | drawing_confirmation 与 markup_confirmation 均记录 device_info + created_at |

---

## 7. 验收条件

- [ ] `GET /drawing/se/page` 响应中不包含 `versionNo` 字段
- [ ] 响应中包含 `rfaNo`、`subjectOfRfa`（无值时为 null）
- [ ] `markupsTotal` = 该 DrawingVersion 下 ACTIVE Markup 数量
- [ ] `markupsUnread` = 该 DrawingVersion 下当前用户未确认的 ACTIVE Markup 数量
- [ ] `GET /drawing/markup/list` 响应中不包含 `affectedArea` 字段
- [ ] Markup 列表按 `publish_time DESC` 排序
- [ ] 响应中包含 `submissionNo`、`appliedPageNo`（nullable）、`creatorName`
- [ ] `POST /drawing/confirm` 幂等：重复提交返回 200，不报错，数据库只有一条记录
- [ ] `drawing_confirmation.device_info` = `"APP"` 时（APP 端提交）正确写入；`"PC"` 时同理
- [ ] **F-004**：Markup 发布后，已分配 SE 的 in_app_notification 正确创建，`payload.type = MARKUP_PUBLISHED`，`target_route` 包含正确 drawingVersionId 和 `tab=markups`
- [ ] **F-005**：DrawingVersion 状态变为 `ACTIVE` 后，系统向已分配该图纸的所有 SE 发送 App Push 通知，`payload.type = DRAWING_PUBLISHED`，通知正文含 Description；未分配的 SE 不收到推送
- [ ] 未分配该图纸的 SE 不会收到任何通知（F-004 / F-005 均不收到）
- [ ] `PATCH /notification/{id}/read` 越权操作返回 1003005004
