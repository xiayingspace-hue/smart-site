---
doc_type: requirement
req_id: REQ-003-app
req_title: "APP 端图纸审批 Todo"
version: 0.1.0
status: draft
priority: P1
product: SMART SITE SYSTEM
owner: ""
created_at: 2026-05-28
updated_at: 2026-05-28

depends_on:
  - REQ-003-shared
related_to:
  - REQ-003-pc
  - REQ-005-app
blocks: []

generate:
  data_contract: false
  ui_spec: true
  frontend_spec: true
  backend_spec: false
  qa_spec: true
---

# 需求文档：APP 端图纸审批 Todo

> **端**: APP 移动端（iOS & Android，UNIAPP + Vue 2）
> **主 Persona**: 审批人员（Approver）
> **共享需求**: [REQ-003-shared.md](../shared/REQ-003-shared.md)（审批流程、API 接口）
> **本文档仅包含**: APP 端 Approver 处理待审批图纸的 Todo 交互；SE 查阅图纸相关功能见 [REQ-005-app.md](./REQ-005-app.md)
> **提取自**: 原 REQ-003-app §4（审批 Todo）、§5（App Push 通知）Approver 相关部分

---

## 1. 背景与目标

### 1.1 业务背景

审批人员需要对 PC 端提交的图纸进行审批（通过/驳回），审批流程不应强依赖 PC 端。当审批人员在施工现场时，应能通过 APP 端 Todo 列表及时处理待审批任务，避免图纸发布流程阻塞。

### 1.2 业务目标

1. Approver 在 APP 端收到待审批图纸的 Todo 任务，可直接在 APP 端完成通过或驳回操作
2. 驳回时必须填写意见，确保决策可追溯
3. Approver 可在 APP 端在线预览待审批图纸，辅助审批决策

### 1.3 非目标（Out of Scope）

- APP 端上传或编辑图纸
- SE 查阅确认相关功能（见 REQ-005-app）
- Approver 的 App Push 通知（新图纸待审批通知由通用 Todo 系统处理）

---

## 2. 用户与角色

| 角色 ID | 角色名 | 典型场景 |
|--------|-------|---------|
| ROLE-003 | Approver（审批人员） | 不在 PC 端时，通过 APP 处理待审批图纸 |

---

## 3. 功能需求

### 3.1 F-001：Todo 入口

- APP 首页 **Todo** 列表模块，与其他 Todo 类型合并展示
- 有未处理的图纸审批 Todo 时，Todo 图标上显示红色数字角标

### 3.2 F-002：待审批图纸 Todo 卡片

**卡片布局**：

```
┌─────────────────────────────┐
│ 📋 Drawing Approval         │
│ ARCH-001  首层平面图  V2     │
│ Uploaded by 李四             │
│ 2026-04-01 10:00            │
│                             │
│ [View]  [Approve] [Reject]  │
└─────────────────────────────┘
```

**卡片字段**：

| 字段 | 说明 |
|------|------|
| 类型标识 | `📋 Drawing Approval`，固定文案 |
| 图纸信息 | Drawing Code、Drawing Name、版本号 |
| 上传人 | `Uploaded by {name}` |
| 上传时间 | `YYYY-MM-DD HH:mm` |
| 操作按钮 | [View]、[Approve]、[Reject] |

---

### 3.3 F-003：审批操作

#### 通过（Approve）

- 点击 **[Approve]** → 弹出确认弹窗：
  ```
  Approve this drawing?
  ARCH-001 首层平面图 V2
  [Cancel]  [Confirm]
  ```
- 点击 [Confirm] 后调用审批 API
- 成功后 Todo 卡片消失，显示 Toast "Drawing approved successfully"
- 失败时 Toast 提示错误，卡片保留

#### 驳回（Reject）

- 点击 **[Reject]** → 弹出底部弹出框（Bottom Sheet）输入驳回意见：
  ```
  ┌──────────────────────────┐
  │ Reject Drawing           │
  │                          │
  │ Comment (Required):      │
  │ ┌──────────────────────┐ │
  │ │                      │ │
  │ │                      │ │
  │ └──────────────────────┘ │
  │                          │
  │ [Cancel]  [Confirm]      │
  └──────────────────────────┘
  ```
- Comment 为必填，空白时 [Confirm] 按钮禁用
- 点击 [Confirm] 后调用驳回 API，传入 comment
- 成功后 Todo 卡片消失，显示 Toast "Drawing rejected"
- 失败时 Toast 提示错误，Bottom Sheet 保留

#### 查看图纸（View）

- 点击 **[View]** → 进入该版本的在线查看页
- 待审批版本的查看页**不显示** [Confirm Reading] 按钮（审批人查看不需要确认阅读）
- 在线预览能力复用 REQ-005-app §7.2 的预览规范（PDF.js / uni.previewImage）

---

## 4. 业务流程

1. Admin 在 PC 端提交新版图纸，状态流转为 `PENDING_APPROVAL`
2. 系统在 Approver 的 APP Todo 列表中生成一条 Drawing Approval 任务
3. Approver 在 APP 端看到红色角标，进入 Todo 列表
4. Approver 点击 [View] 在线预览图纸，确认内容后点击 [Approve] 或 [Reject]
5. 通过：图纸状态变为 `ACTIVE`，系统触发 SE App Push（见 REQ-005-app §F-005）
6. 驳回：图纸状态变为 `REJECTED`，上传人收到驳回通知

---

## 5. 验收标准

### AC-003B-APP-001：Todo 角标

```
Given  有 N 条未处理的图纸审批 Todo
When   Approver 进入 APP 首页
Then   Todo 图标显示红色数字角标 N
```

### AC-003B-APP-002：通过操作

```
Given  Approver 在 Todo 列表看到一条待审批图纸卡片
When   点击 [Approve] 并在弹窗中确认
Then   调用审批 API；Todo 卡片消失；显示 Toast "Drawing approved successfully"
```

### AC-003B-APP-003：驳回操作 — Comment 必填

```
Given  Approver 点击 [Reject]，Bottom Sheet 已弹出
When   Comment 输入框为空
Then   [Confirm] 按钮处于禁用状态，无法提交
```

### AC-003B-APP-004：驳回操作 — 成功

```
Given  Approver 在 Bottom Sheet 中输入了 Comment 并点击 [Confirm]
When   API 调用成功
Then   Todo 卡片消失；显示 Toast "Drawing rejected"
```

### AC-003B-APP-005：查看图纸不显示 Confirm Reading

```
Given  Approver 点击 [View] 进入图纸在线查看页
When   页面加载完成
Then   底部不显示 [Confirm Reading] 按钮
```

### AC-003B-APP-006：幂等保护

```
Given  Approver 快速重复点击 [Approve] 或 [Confirm]
When   第一次请求已发出
Then   后续点击被忽略（按钮 loading 状态），不重复调用 API
```

---

## 6. 非功能需求

| 指标 | 目标值 |
|-----|-------|
| 审批操作响应 | < 500ms |
| 图纸预览加载（< 10MB PDF） | < 5s（正常网络） |
| 兼容性 | iOS 13+，Android 7+，UNIAPP 最新稳定版 |

---

## 7. 依赖与外部系统

| 依赖 | 用途 | Owner |
|------|------|-------|
| REQ-003-shared API | 审批通过 / 驳回接口 | 后端团队 |
| REQ-005-app 在线预览 | 复用图纸文件预览能力 | 前端团队 |

---

## 8. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-28 | | 独立成文（原为 REQ-003-app §4、§5 Approver 部分，现以 REQ-003-app 命名） |

---

## 9. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-003-shared.md](../shared/REQ-003-shared.md) | 审批流程共享规则与 API |
| [REQ-003-pc.md](../pc/REQ-003-pc.md) | PC 端图纸上传与审批管理 |
| [REQ-005-app.md](./REQ-005-app.md) | APP 端 SE 图纸查阅与 Markups（含审批通过后 SE 侧 Push 通知） |
