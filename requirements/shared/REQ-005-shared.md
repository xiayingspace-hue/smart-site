# Site Engineer PC 端图纸访问能力 — 跨端共享需求

> 本文档定义 REQ-005 的跨端共享部分：PC 端站内通知扩展、跨端确认记录同步规则。
> PC 端 Site Engineer 专属交互请参见 [REQ-005-pc.md](../pc/REQ-005-pc.md)。

## 基本信息

- **需求ID**: REQ-005
- **需求标题**: Site Engineer PC 端图纸查阅与局部更新查看
- **产品**: SMART SITE SYSTEM
- **适用端**: PC（Site Engineer 只读视图，站内通知）
- **优先级**: 高
- **状态**: 草稿
- **依赖需求**:
  - [REQ-003-shared.md](./REQ-003-shared.md)（图纸管理核心数据模型与 API）
  - [REQ-004-shared.md](./REQ-004-shared.md)（局部更新数据模型与 API）

---

## 1. 需求描述

### 1.1 背景与目标

REQ-003 / REQ-004 将 Site Engineer 的图纸访问和局部更新查阅限定在 APP 端。但 Site Engineer 在办公室或会议室使用 PC 时，同样需要查看图纸和局部更新。本需求将 APP 端的 Site Engineer 能力等效扩展到 PC 端。

主要扩展点：
1. **PC 端 Site Engineer 图纸列表**：数据过滤规则与 APP 端完全一致
2. **图纸查阅确认（Confirm Reading）跨端同步**：在 PC 或 APP 任一端确认，视为同一条记录，双端均更新
3. **局部更新 Mark as Read 跨端同步**：同上，跨端共享确认状态
4. **PC 端站内通知（In-App Notification）**：局部更新发布时，同步推送 PC 站内通知，与 App Push 并行发送

### 1.2 用户故事

```
作为 Site Engineer
我想要 在 PC 端看到和移动端一样的图纸管理功能
以便 在使用电脑的时候也能查看设计图和收到图纸更新通知
```

---

## 2. 数据模型扩展

### 2.1 DrawingReadConfirmation — deviceInfo 字段扩展

原 REQ-003-shared 中的 `deviceInfo` 字段新增枚举值：

| 值 | 说明 |
|----|------|
| `APP` | 通过 APP 移动端确认 |
| `PC` | 通过 PC 浏览器确认 |

跨端确认幂等规则：
- 同一用户对同一图纸版本，无论在哪个端确认，均记录为**同一条确认记录**（`drawingVersionId + confirmerId` 唯一）
- 已在 APP 确认，PC 端查看时显示"已确认"；已在 PC 确认，APP 端查看时显示"已确认"
- `deviceInfo` 记录**首次确认**所用的设备端

### 2.2 DrawingMarkupReadRecord — deviceInfo 字段新增

在 REQ-004-shared 的 `DrawingMarkupReadRecord` 模型中新增 `deviceInfo` 字段：

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 确认设备 | deviceInfo | String | 自动 | 首次确认所用设备端（`APP` / `PC`） |

跨端幂等规则与 §2.1 一致（`markupId + confirmerId` 唯一）。

---

## 3. PC 端站内通知机制

### 3.1 触发时机

与 App Push 触发时机完全一致（REQ-004-shared §4 通知机制），即：
- 管理员在 PC 端**发布局部更新**（POST `/drawing/markup/publish`）后
- 系统向所有已被分配该图纸的 Site Engineer 同时发送：
  1. App Push（原有，REQ-004-shared 定义）
  2. **PC 站内通知**（本需求新增）

### 3.2 通知数据模型

**Notification**（站内通知）：

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 主键 | id | Long | 自动 | |
| 接收人ID | recipientId | Long | 是 | Site Engineer 用户 ID |
| 通知类型 | type | Enum | 是 | `DRAWING_MARKUP`（本需求）; 后续可扩展 |
| 标题 | title | String | 是 | 如 "Drawing Update — ARCH-001" |
| 内容 | body | String | 是 | 正文文案 |
| 关联实体类型 | entityType | String | 是 | `DrawingMarkup` |
| 关联实体ID | entityId | Long | 是 | markupId |
| 跳转路由 | targetRoute | String | 是 | PC 前端路由，如 `/drawings/{drawingId}?tab=markups` |
| 是否已读 | isRead | Boolean | 是 | 默认 false |
| 已读时间 | readTime | DateTime | 否 | |
| 创建时间 | createdAt | DateTime | 自动 | |

### 3.3 API 接口

#### GET `/notification/list`

获取当前登录用户的站内通知列表。

**Query Params**：

| 参数 | 类型 | 说明 |
|------|------|------|
| page | Integer | 页码，默认 1 |
| pageSize | Integer | 每页条数，默认 20 |
| isRead | Boolean | 可选，筛选已读/未读 |

**Response**：

```json
{
  "code": 200,
  "data": {
    "total": 5,
    "unreadCount": 2,
    "list": [
      {
        "id": 1001,
        "type": "DRAWING_MARKUP",
        "title": "Drawing Update — ARCH-001",
        "body": "A轴节点详图修正. Please review the latest markup for ARCH-001 首层平面图.",
        "targetRoute": "/drawings/101?tab=markups",
        "isRead": false,
        "createdAt": "2026-04-08T09:00:00Z"
      }
    ]
  }
}
```

#### PATCH `/notification/{id}/read`

将单条通知标记为已读。

**Response**：`{ "code": 200 }`

#### PATCH `/notification/read-all`

将当前用户所有未读通知标记为已读。

**Response**：`{ "code": 200 }`

---

## 4. 权限控制

| 操作 | 允许角色 |
|------|---------|
| 查看 Site Engineer 专属图纸列表 | Site Engineer |
| Confirm Reading（图纸查阅确认） | Site Engineer |
| 查看 Markups Tab | Site Engineer |
| Mark as Read（局部更新确认） | Site Engineer |
| 接收 PC 站内通知 | Site Engineer |
| 获取通知列表 / 标记已读 | 已登录用户（各角色均可，返回各自的通知） |

---

## 5. 错误码

| 错误码 | 说明 |
|--------|------|
| 1003005001 | 通知不存在 |
| 1003005002 | 无权操作他人通知 |
| 1003005003 | 图纸未分配给当前用户 |

---

## 相关需求

| 文档 | 说明 |
|------|------|
| [REQ-003-shared.md](./REQ-003-shared.md) | 图纸核心数据模型与 API（本需求扩展 deviceInfo） |
| [REQ-004-shared.md](./REQ-004-shared.md) | 局部更新数据模型与 API（本需求扩展 deviceInfo） |
| [REQ-005-pc.md](../pc/REQ-005-pc.md) | PC 端 Site Engineer 专属交互规范 |
