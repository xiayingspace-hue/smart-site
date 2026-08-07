---
doc_type: requirement
req_id: REQ-005-app
req_title: "APP 端 Site Engineer 图纸查阅与局部更新查看"
version: 0.1.1
status: draft
priority: P1
product: SMART SITE SYSTEM
owner: ""
created_at: 2026-05-28
updated_at: 2026-05-28

depends_on:
  - REQ-003A-pc
  - REQ-003-shared
  - REQ-004-shared
related_to:
  - REQ-005-pc
  - REQ-003-app
supersedes:
  - REQ-003-app
  - REQ-004-app
blocks: []

generate:
  data_contract: false
  ui_spec: true
  frontend_spec: true
  backend_spec: false
  qa_spec: true
---

# 需求文档：APP 端 Site Engineer 图纸查阅与局部更新查看

> **端**: APP 移动端（iOS & Android，UNIAPP + Vue 2）
> **对应 PC 功能**: [REQ-005-pc.md v0.3.5](./REQ-005-pc.md)
> **共享需求**: [REQ-003-shared.md](../shared/REQ-003-shared.md)、[REQ-004-shared.md](../shared/REQ-004-shared.md)
> **依赖需求**: [REQ-003A-pc.md](./REQ-003A-pc.md)（图纸提交记录数据模型）
> **本文档仅包含**: APP 端 Site Engineer 视角的图纸查阅、确认、局部更新查看功能，对齐 REQ-005-pc 的数据模型（DrawingVersion 为列表主体）

---

## 1. 背景与目标

### 1.1 业务背景

Site Engineer 主要在施工现场通过手机使用系统。本需求基于 REQ-005-pc 的数据模型重构（列表每行 = 一次提交记录 DrawingVersion），对齐 PC 端新增的 RFA 信息展示、Markups `●{unread}/{total}` 格式、Markups 卡片规格（报审号+页码行，无 Affected Area）。

### 1.2 业务目标

1. APP 端图纸列表与 PC 端使用相同数据模型（DrawingVersion），列表显示 Description、RFA No.、Subject of RFA
2. Markups 列/角标格式对齐 PC 端：`●{unread} / {total}` 含义清晰
3. Markups 卡片对齐 REQ-004-pc §7.3 规范：`{日期}·{创建人}` 元信息行、`Based on {submissionNo}·Page {n}` 报审行，不显示 Affected Area
4. Confirm Reading 确认后跨端同步（APP 确认后 PC 端显示已确认，反之亦然）

### 1.3 非目标（Out of Scope）

- APP 端上传或编辑图纸
- APP 端发布或管理局部更新（Markups）
- 局部更新的评论或回复功能

---

## 2. 用户与角色

| 角色 ID | 角色名 | 典型场景 |
|--------|-------|---------|
| ROLE-001 | Site Engineer | 在施工现场手机查阅最新图纸及局部更新 |
| ROLE-002 | 图纸管理员 | 上传图纸、分配图纸、发布局部更新 |

---

## 3. 角色与权限矩阵

| 操作 | Site Engineer | 图纸管理员 |
|-----|--------------|-------|
| 查看个人分配图纸列表 | ✅ | ❌ |
| 在线查看图纸文件 | ✅ | ✅ |
| 执行 Confirm Reading | ✅ | ❌ |
| 查看图纸的 Markups 列表 | ✅ | ✅ |
| 执行 Mark Markup as Read | ✅ | ❌ |
| 接收 App Push 通知 | ✅ | ❌ |

---

## 4. 核心实体（对齐 REQ-005-pc §4.1）

| 实体 ID | 实体名 | 描述 |
|--------|-------|------|
| ENT-001 | DrawingVersion | **列表每行所对应的实体**；一次 PDF 上传；description 为主标识；版本号不对 SE 展示 |
| ENT-002 | Drawing | 页级记录；不作为列表行展示 |
| ENT-003 | DrawingAssignment | DrawingVersion 与 SE 的分配关系 |
| ENT-004 | DrawingConfirmation | SE 对某次提交记录的查阅确认（drawingVersionId, userId, confirmedAt, deviceInfo） |
| ENT-005 | DrawingMarkup | 局部更新（drawingVersionId, title, description, submissionNo, appliedPageNo, publishTime, status） |
| ENT-006 | MarkupConfirmation | SE 对某条 Markup 的已读确认（markupId, userId） |

---

## 5. 状态机（同 REQ-005-pc §5）

### 5.1 DrawingConfirmation

| From | To | 触发 | 副作用 |
|------|-----|------|-------|
| Unconfirmed | Confirmed | 点击 [Confirm Reading] 并在弹窗确认 | 写入 DrawingConfirmation（deviceInfo="APP"）；列表卡片状态更新为 ✓ Read |

### 5.2 MarkupConfirmation

| From | To | 触发 | 副作用 |
|------|-----|------|-------|
| Unread | Read | 点击 [Mark as Read] | 写入 MarkupConfirmation；卡片按钮替换为 ✓ Read；列表卡片 Markups 角标减一 |

---

## 6. 业务流程

### 6.1 主流程一：图纸查阅与确认（APP 端）

1. Site Engineer 进入 APP → 点击首页 Drawings 入口
2. 系统加载该用户已分配的 ACTIVE DrawingVersion 列表
3. SE 点击某条卡片 → 进入图纸在线查看页
4. 系统加载图纸文件（PDF/PNG/JPG），支持双指缩放与单指平移
5. SE 点击 **[Confirm Reading]** → 弹出确认弹窗
6. 用户确认 → 系统调用 `/drawing/confirm`，`deviceInfo = "APP"`
7. 按钮替换为 `✓ Confirmed on {date}`；返回列表该卡片状态更新

### 6.2 主流程二：局部更新查看与已读标记

1. SE 在图纸查看页切换到 Markups Tab（或从通知直接进入）
2. 系统加载该 DrawingVersion 的所有 `status = ACTIVE` Markup，按 publishTime 降序
3. SE 查看 Markup 卡片（标题、发布日期·创建人、报审号+页码、说明、附件）
4. 对未读 Markup 点击 **[Mark as Read]** → 调用 `/drawing/markup/confirm`
5. 成功后卡片变为已读；列表卡片 Markups 角标未读数减一

### 6.3 主流程三：App Push 通知

1. 图纸管理员发布 DrawingMarkup → 系统向已分配该图纸的 SE 推送 App Push 通知
2. SE 点击通知 → 进入对应图纸查看页，自动激活 Markups Tab

---

## 7. 功能需求详述

### 7.1 F-001：图纸列表页

**页面布局**:

```
┌──────────────────────────────────────┐
│ ← Drawings                           │
│                                      │
│ [🔍 Search description...]           │
│ [All ▼]                              │
│                                      │
│ ┌──────────────────────────────────┐ │
│ │ 首层平面图施工图纸            ●2 │ │  ← 橙色角标（未读 Markup 数）
│ │ Architectural                    │ │
│ │ RFA-001 · 首层平面图报审         │ │
│ │ Updated 2026-04-01   [✓ Read]   │ │
│ └──────────────────────────────────┘ │
│ ┌──────────────────────────────────┐ │
│ │ 基础结构施工详图                  │ │
│ │ Structural                       │ │
│ │ —                                │ │  ← RFA No. 为空时显示 —
│ │ Updated 2026-03-20  [Confirm →]  │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘
```

**卡片字段**:

| 字段 | 说明 |
|------|------|
| Description | 批次描述，主标识，16px，加粗 |
| Category | 图纸分类，14px，灰色 |
| RFA 信息行 | `{rfaNo} · {subjectOfRfa}`；rfaNo 为 null 时显示 `—` |
| Updated | 最后更新时间，12px，灰色 |
| 状态标识 | `[✓ Read]`（绿色）/ `[Confirm →]`（蓝色） |
| Markups 角标 | 右上角橙色数字角标（仅显示 unread 数量，unread=0 时隐藏） |

> **不展示的字段**：系统版本号（versionNo）、Drawing Code、Drawing Name 均不在卡片中展示。

**交互规则**:
- 下拉刷新，上拉加载更多（每次 20 条）
- 搜索框：Description 模糊搜索（debounce 500ms）
- Category 下拉筛选
- 点击卡片任意区域 → 进入图纸在线查看页（Drawing Preview Tab）
- 空状态："No drawings assigned to you yet."

---

### 7.2 F-002：图纸在线查看页

**触发方式**: 点击图纸列表卡片

**页面布局**:

```
┌──────────────────────────────────────┐
│ ← 首层平面图施工图纸            [⋯] │
│   Architectural                      │
├──────────────────────────────────────┤
│  [Drawing Preview]  [Markups (●2)]   │  ← Tab 栏
├──────────────────────────────────────┤
│                                      │
│                                      │
│    （图纸在线预览区域）               │
│    支持双指缩放                       │
│    支持单指平移                       │
│                                      │
│                                      │
├──────────────────────────────────────┤
│  Updated 2026-04-01                  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  ✓  Confirm Reading            │  │  ← 未确认状态
│  └────────────────────────────────┘  │
│                                      │
│  ✓ Confirmed on Apr 8, 2026         │  ← 已确认状态（替换按钮）
└──────────────────────────────────────┘
```

> **说明**：顶部标题显示 Description 和 Category，不显示系统版本号。

**在线预览规范**:
- PDF：WebView 内嵌 PDF.js 渲染
- PNG / JPG：`uni.previewImage` 支持双指缩放
- DWG：后端转换为 PDF 后展示
- 支持双指缩放、单指平移

**Confirm Reading 交互**:

| 状态 | 显示位置 | 显示内容 | 操作 |
|------|---------|---------|------|
| 未确认 | 底部固定区域 | `[Confirm Reading]` 蓝色主按钮 | 点击弹出确认 Popup |
| 已确认 | 底部固定区域 | `✓ Confirmed on {date}`，绿色文字 | 仅展示 |

**确认 Popup**:
```
┌─────────────────────────────────────┐
│  Confirm Reading                    │
│                                     │
│  Confirm that you have read this    │
│  drawing?                           │
│  {description} · {category}         │
│                                     │
│  [Cancel]        [Confirm]          │
└─────────────────────────────────────┘
```

**边界与约束**:
- `deviceInfo` 写入 `"APP"`
- 确认成功后列表卡片状态实时更新为 `✓ Read`
- 跨端幂等：PC 端已确认则 APP 显示已确认

---

### 7.3 F-003：Markups Tab — 局部更新列表

**触发方式**: 在图纸查看页切换 Markups Tab，或从 App Push 通知直接进入

**布局**:

```
┌──────────────────────────────────────┐
│  [Drawing Preview]  [Markups (●2)]   │
├──────────────────────────────────────┤
│  ┌────────────────────────────────┐  │
│  │  [New]  A轴节点详图修正        │  │
│  │  Apr 8, 2026 · 张三            │  │
│  │  Based on SUB-2026-003 · Page 3│  │
│  │  A轴与3轴交叉节点详图已更新，  │  │
│  │  新增钢筋排布说明...           │  │
│  │  📎 node-detail.pdf            │  │
│  │           [✓ Mark as Read]     │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │  C区消防管道路由修正            │  │
│  │  Apr 7, 2026 · 张三            │  │
│  │  Based on SUB-2026-003 · Page 5│  │
│  │  消防主管道路由变更，详见附图... │  │
│  │  📎 route-update.png           │  │
│  │                     ✓ Read     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**卡片规格**（对齐 REQ-004-pc §7.3）:

| 卡片属性 | 规格 |
|---------|------|
| [New] 标签 | 橙色胶囊，当前用户未标记已读时显示；标记已读后隐藏 |
| 标题 | DrawingMarkup 的 description 字段；16px，加粗 |
| 元信息行 | `{发布日期} · {创建人}`，13px，灰色 |
| 报审号+页码行 | `Based on {submissionNo}`；若 `appliedPageNo` 不为 null 则追加 ` · Page {n}`；13px，灰色 |
| 说明文字 | 最多 3 行，超出显示"...Show more"展开 |
| 附件 | 📎 文件名，点击调用系统预览或浏览器打开 |
| [Mark as Read] | 未读时显示蓝色主按钮；点击调用 `/drawing/markup/confirm`；成功后替换为 `✓ Read` |

> **不展示的字段**：Affected Area（影响区域）不在 SE 视图卡片中显示。

**列表规则**:
- 仅显示 `status = ACTIVE` 的 Markup
- 按 `publishTime` 降序排列
- 无 Markup 时显示："No markups for this drawing yet."

---

### 7.4 F-004：App Push 通知 — Markup 发布

**触发时机**: 图纸管理员发布 DrawingMarkup 时，系统向已分配该图纸的所有 SE 发送 App Push 通知

**通知格式**:

| 字段 | 内容 |
|------|------|
| 标题 | Drawing Update — {description} |
| 正文 | {markupTitle}. Please review the latest markup. |
| 角标 | APP 图标 +1 |
| 跳转 | 点击通知 → 进入对应图纸查看页，自动激活 Markups Tab |

**通知跳转处理**:
- 热启动：监听 `plus.push.addEventListener` 解析 payload，调用路由跳转
- 冷启动：在 `App.vue` onLaunch 中解析 `plus.runtime.arguments`，跳转目标页

---

### 7.5 F-005：App Push 通知 — 新版图纸发布

**触发时机**: 图纸审批通过（DrawingVersion 状态变为 `ACTIVE`）时，系统向已被分配该图纸的所有 SE 发送 App Push 通知

**通知格式**:

| 字段 | 内容 |
|------|------|
| 标题 | Drawing Updated |
| 正文 | {description} has a new version. Please confirm reading. |
| 角标 | APP 图标 +1 |
| 跳转 | 点击通知 → 进入图纸列表页，自动定位/高亮该 DrawingVersion |

**通知跳转处理**:
- 热启动：监听 `plus.push.addEventListener` 解析 payload，调用路由跳转
- 冷启动：在 `App.vue` onLaunch 中解析 `plus.runtime.arguments`，跳转图纸列表页并高亮目标条目

**通知权限说明**:
- 首次安装时引导用户开启通知权限
- 若用户关闭通知权限，不影响 APP 内正常使用；图纸列表仍可手动下拉刷新查看最新状态

---

## 8. 验收标准（Acceptance Criteria）

### AC-005-APP-001：图纸列表仅显示已分配的提交记录

```
Given  SE 已登录，图纸管理员已将 DrawingVersion A 分配给该用户，DrawingVersion B 未分配
When   用户进入 Drawings 页面
Then   列表每行代表一次 DrawingVersion；
       仅显示已分配的 A，不显示 B；
       卡片展示 Description、Category、RFA 信息行（rfaNo·subjectOfRfa）、Updated、确认状态；
       不显示系统版本号（versionNo）、Drawing Code、Drawing Name；
       RFA No. 和 Subject of RFA 在外部审批已发起后显示，否则显示 —
```

### AC-005-APP-002：Markups 角标显示未读数

```
Given  DrawingVersion A 有 5 条 ACTIVE Markup，当前用户未读 3 条
When   用户进入图纸列表页
Then   A 的卡片右上角显示橙色角标 "3"；
       全部已读后角标消失
```

### AC-005-APP-003：图纸在线查看 Header 不含版本号

```
Given  SE 点击 DrawingVersion A 的卡片
When   图纸查看页加载完成
Then   顶部导航栏标题显示 Description 和 Category，不显示系统版本号
```

### AC-005-APP-004：Confirm Reading 成功（APP 端）

```
Given  SE 在查看页看到 [Confirm Reading] 按钮
When   用户点击按钮并在 Popup 中确认
Then   按钮替换为 "✓ Confirmed on {date}"；返回列表后该卡片状态显示 "✓ Read"
```

### AC-005-APP-005：Confirm Reading 写入 deviceInfo=APP

```
Given  SE 在 APP 端完成 Confirm Reading
When   后端写入 DrawingConfirmation 记录
Then   记录的 device_info 字段值为 "APP"
```

### AC-005-APP-006：Confirm Reading 跨端同步

```
Given  SE 已在 PC 端对某张图纸完成 Confirm Reading
When   同一用户在 APP 端打开同一图纸
Then   查看页底部直接显示 "✓ Confirmed on {date}"，不显示 [Confirm Reading] 按钮
```

### AC-005-APP-007：Markups Tab 仅显示 ACTIVE 更新

```
Given  DrawingVersion A 有 2 条 ACTIVE Markup 和 1 条 MERGED Markup
When   SE 切换到 Markups Tab
Then   仅显示 2 条 ACTIVE Markup，按 publishTime 降序排列
```

### AC-005-APP-008：Markups 卡片字段验证

```
Given  某条 Markup submissionNo="SUB-2026-003"，appliedPageNo=3，creatorName="张三"
When   SE 查看该 Markup 卡片
Then   卡片显示元信息行 "{发布日期} · 张三"；
       显示报审行 "Based on SUB-2026-003 · Page 3"；
       不显示 Affected Area 字段
```

### AC-005-APP-009：Mark as Read 操作（APP 端）

```
Given  Markup 卡片显示 [New] 标签和 [Mark as Read] 按钮
When   用户点击 [Mark as Read]
Then   按钮替换为 "✓ Read"；[New] 标签消失；
       列表卡片 Markups 角标未读数减一；刷新后状态持久保留
```

### AC-005-APP-010：Mark as Read 跨端同步

```
Given  SE 已在 PC 端对某条 Markup 标记已读
When   同一用户在 APP 端刷新 Markups Tab
Then   该条 Markup 显示 "✓ Read"，不显示 [Mark as Read] 按钮
```

### AC-005-APP-011：App Push 通知接收

```
Given  图纸管理员发布了 DrawingVersion A 的一条 Markup，SE 已被分配 A
When   图纸管理员点击发布
Then   SE 的 APP 图标角标 +1，通知栏出现对应通知
```

### AC-005-APP-012：通知跳转至 Markups Tab

```
Given  SE 的通知栏有一条图纸 Markup 通知
When   用户点击该通知
Then   跳转到对应图纸查看页，Markups Tab 自动激活
```

### AC-005-APP-013：新版图纸发布 App Push 通知接收

```
Given  DrawingVersion A 审批通过（状态变为 ACTIVE），SE 已被分配 A
When   审批通过事件触发
Then   SE 的 APP 图标角标 +1，通知栏出现对应通知；
       通知正文包含 Description；
       未被分配 A 的 SE 不收到通知
```

### AC-005-APP-014：新版图纸发布通知跳转

```
Given  SE 的通知栏有一条新版图纸发布通知
When   用户点击该通知
Then   跳转到图纸列表页，并自动定位/高亮对应 DrawingVersion；
       冷启动场景下跳转同样正常
```

---

## 9. 非功能需求

| 指标 | 目标值 |
|-----|-------|
| 图纸列表首屏加载 | < 2s（50 条以内） |
| 图纸文件加载（< 10MB PDF） | < 5s（正常网络） |
| Confirm / Mark as Read 响应 | < 500ms |
| App Push 推送延迟 | < 10s（从发布到通知到达） |
| 兼容性 | iOS 13+，Android 7+，UNIAPP 最新稳定版 |

---

## 10. 依赖与外部系统

| 依赖 | 用途 | Owner |
|------|------|-------|
| REQ-003-shared API | 图纸列表、详情、Confirm Reading | 后端团队 |
| REQ-004-shared API | Markup 列表、Mark as Read | 后端团队 |
| UNIAPP Push（UniPush） | App Push 通知推送 | 后端团队 |
| 文件存储服务 | 图纸文件 URL 生成（预签名） | 后端团队 |

---

## 11. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-28 | | 初稿；对齐 REQ-005-pc v0.3.5 数据模型：DrawingVersion 为列表主体，卡片展示 Description/RFA No./Subject of RFA，Markups 角标仅显示 unread，Markups 卡片对齐 REQ-004-pc §7.3（元信息行+报审行，无 Affected Area） |
| 0.1.1 | 2026-08-08 | XIA YING | 按 glossary.md §2 统一角色名称：项目管理员 / 项目管理人员 / 业务人员 / 管理员 → 图纸管理员；Drawing 团队（成员）→ 设计人员；审批人 → 内部审批人；普通业务人员 → 普通用户 |

---

## 12. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-005-pc.md](../pc/REQ-005-pc.md) | PC 端对等功能（v0.3.5，数据模型来源） |
| [REQ-003-app.md](./REQ-003-app.md) | APP 端 Approver 审批 Todo |
| [REQ-003-shared.md](../shared/REQ-003-shared.md) | 图纸共享业务规则与 API |
| [REQ-004-shared.md](../shared/REQ-004-shared.md) | 局部更新共享业务规则与 API |
| [REQ-003A-pc.md](../pc/REQ-003A-pc.md) | 图纸管理列表页基础规范（列表行含义与数据模型） |
| ~~旧版 REQ-003-app / REQ-004-app~~ | ⚠️ 已废弃，由本文档 + 新版 REQ-003-app 替代 |
| ~~REQ-004-app.md~~ | ⚠️ 已废弃，由本文档替代 |
