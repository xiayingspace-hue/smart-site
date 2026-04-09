# 后端开发说明文档 — REQ-005 通知系统与跨设备同步

> **来源需求**: [REQ-005-shared.md](../../requirements/shared/REQ-005-shared.md) + [REQ-005-pc.md](../../requirements/pc/REQ-005-pc.md)
> **产品**: SMART SITE SYSTEM
> **服务模块**: `notification-service`（或集成在 `drawing-service` 内）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

实现 PC 端和 APP 端的站内通知及跨设备状态同步功能：

| 功能 | 说明 |
|------|------|
| 站内通知列表 | 获取当前用户的通知记录（PC + APP）|
| 标记单条已读 | 点击通知后标记为已读 |
| 全部已读 | 一键清除所有未读通知 |
| 未读数角标 | 轮询或 WebSocket 实时推送未读数 |
| 跨设备幂等 | 查阅确认和 Markup 已读记录 deviceInfo，跨设备不重复创建 |

---

## 2. 业务逻辑

### 2.1 通知生成时机

| 触发事件 | 接收人 | 通知类型 |
|---------|------|---------|
| 图纸版本审批通过 | 已分配 SE 列表 | `DRAWING_UPDATED` |
| Markup 发布 | 已分配 SE 列表 | `MARKUP_PUBLISHED` |
| 图纸版本被驳回 | 上传人（DC）| `DRAWING_REJECTED` |

通知由业务事件（MQ 消息）触发，notification-service 消费消息后写入 `notification` 表并发送 App Push。

### 2.2 通知路由

| 类型 | PC 点击跳转 | APP 点击跳转 |
|------|-----------|------------|
| `DRAWING_UPDATED` | 图纸详情页 → Markups Tab | 图纸详情页 → Markups Tab |
| `MARKUP_PUBLISHED` | 图纸详情页 → Markups Tab | 图纸详情页 → Markups Tab |
| `DRAWING_REJECTED` | Drawing Management 列表 | 无（PC 专属）|

### 2.3 跨设备幂等规则

| 操作 | 幂等键 |
|------|-------|
| 图纸查阅确认 | `drawing_version_id + confirmer_id` |
| Markup 已读确认 | `markup_id + confirmer_id` |
| 说明 | 同一用户无论在 PC 还是 APP 确认，后端使用 `INSERT IGNORE` 或 `ON DUPLICATE KEY UPDATE`，`device_info` 记录首次设备，不覆盖 |

### 2.4 未读数获取

PC 端通过轮询（30s/次）或 WebSocket 推送获取未读数。建议接口 `GET /notification/unread-count` 单独暴露，走缓存（Redis，TTL=60s）。

---

## 3. 数据模型

### 3.1 notification 表

```sql
CREATE TABLE `notification` (
  `id`             BIGINT PRIMARY KEY AUTO_INCREMENT,
  `tenant_id`      BIGINT NOT NULL,
  `recipient_id`   BIGINT NOT NULL COMMENT '接收人 userId',
  `type`           VARCHAR(50) NOT NULL COMMENT 'DRAWING_UPDATED / MARKUP_PUBLISHED / DRAWING_REJECTED',
  `title`          VARCHAR(200) NOT NULL,
  `body`           TEXT,
  `entity_type`    VARCHAR(50) COMMENT 'DRAWING / MARKUP',
  `entity_id`      BIGINT COMMENT '关联实体 ID',
  `target_route`   VARCHAR(500) COMMENT '前端路由（含查询参数）',
  `is_read`        TINYINT(1) NOT NULL DEFAULT 0,
  `read_at`        DATETIME,
  `created_at`     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  KEY `idx_recipient_read` (`recipient_id`, `is_read`),
  KEY `idx_recipient_created` (`recipient_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 4. API 设计

### 4.1 接口列表

- [x] `GET /notification/list` — 通知列表（分页）
- [x] `GET /notification/unread-count` — 未读数
- [x] `PATCH /notification/{id}/read` — 标记单条已读
- [x] `PATCH /notification/read-all` — 全部标记已读

### 4.2 接口规范

#### GET /notification/list

**Query Params**:

| 参数 | 说明 |
|------|------|
| `isRead` | `0`（未读）/ `1`（已读）/ 不传（全部）|
| `type` | 通知类型筛选（可选）|
| `pageNo` | 页码，默认 1 |
| `pageSize` | 每页条数，默认 20 |

**Response**:
```json
{
  "code": 0,
  "data": {
    "list": [
      {
        "id": 100,
        "type": "DRAWING_UPDATED",
        "title": "Drawing Update — DWG-001",
        "body": "Drawing DWG-001 has been approved and published.",
        "entityType": "DRAWING",
        "entityId": 1,
        "targetRoute": "/drawings/detail?drawingId=1&activeTab=markups",
        "isRead": false,
        "createdAt": "2026-04-08T10:00:00"
      }
    ],
    "total": 5,
    "unreadTotal": 3
  }
}
```

#### GET /notification/unread-count

**Response**: `{ "code": 0, "data": { "count": 3 } }`

#### PATCH /notification/{id}/read

**路径参数**: `id` — 通知 ID

**后端校验**: 该通知的 `recipient_id` 必须等于当前用户 ID（防止越权）

#### PATCH /notification/read-all

将当前用户所有 `is_read=0` 的通知批量更新为 `is_read=1`

```sql
UPDATE notification SET is_read=1, read_at=NOW()
WHERE recipient_id=#{userId} AND is_read=0 AND tenant_id=#{tenantId}
```

---

## 5. 错误码

| 错误码 | 说明 |
|--------|------|
| 1003005001 | 通知不存在 |
| 1003005002 | 无权操作该通知（recipient_id 不匹配）|
| 1003005003 | 系统错误（Push 服务异常）|

---

## 6. 非功能需求

| 要求 | 规范 |
|------|------|
| 推送可靠性 | App Push 通过 MQ 异步，失败重试 3 次，写入死信队列告警 |
| 性能 | 未读数接口走 Redis 缓存，TTL=60s；列表 P95 < 300ms |
| 数据保留 | 通知记录保留 180 天，超期自动清理（定时任务）|
| 越权防护 | 所有操作强制校验 `recipient_id == currentUserId` |

---

## 7. 验收条件

- [ ] 图纸审批通过后，已分配 SE 的 notification 表中创建 `DRAWING_UPDATED` 记录
- [ ] Markup 发布后，已分配 SE 的 notification 表中创建 `MARKUP_PUBLISHED` 记录
- [ ] `GET /notification/unread-count` 返回正确未读数
- [ ] 标记已读后 `is_read=1`，未读数减少
- [ ] `read-all` 接口将当前用户所有通知置为已读
- [ ] 越权标记他人通知已读时返回 1003005002
- [ ] 通知的 `targetRoute` 字段包含正确路由（含 drawingId / markupId）
- [ ] PC 端 🔔 角标数与 `unread-count` 接口一致
- [ ] 跨设备幂等：同一 SE 在 PC 和 APP 均确认图纸，`drawing_read_confirmation` 只有一条记录
