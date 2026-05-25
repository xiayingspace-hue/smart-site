---
doc_type: ui_spec
req_id: REQ-007A-pc
version: 0.3.0
status: draft
generated_from: REQ-007A-pc@0.5.0
generated_at: 2026-05-25
owner: ""
---

# UI 设计说明：PC 端 — 内部审批 Todo、详情查看与原文件下载

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
> 本文档由原 UI-REQ-007A-pc（v0.2.0）与 UI-REQ-007E-pc（v0.1.0）合并而来，UI-REQ-007E-pc 已删除。
>
> - 输入：REQ-007A-pc.md（主，v0.5.0）、REQ-007-shared（业务规则）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc §3.2
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007A-pc @ v0.5.0 |
| 覆盖用户故事 | US-007A-001 ~ US-007A-006 |
| 覆盖 AC | AC-007A-001 ~ AC-007A-023 |
| 上次同步时间 | 2026-05-25 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

内部审批人在 Todo 列表中只看到**指定给自己**的图纸待审批任务；点击 [Detail] 打开详情侧滑弹框，获取与上传时完全一致的全量图纸信息，支持在线查看和本地下载原始文件，并在弹框内完成 Approve / Reject 操作，全程无需跳转页面。

### 1.2 设计原则

1. **角色专属视图**：🔍 Internal 卡片仅对当前被指派的内部审批人可见，与 🌐 External 卡片图标和文案保持高区分度。
2. **一入口、全上下文**：Todo 卡片只保留 [Detail] 一个入口，所有文件查阅与审批操作在侧滑弹框内完成，减少用户心智负担。
3. **信息完整性**：详情弹框展示字段与上传时填写的字段严格对应，不新增、不裁减（Drawing Code / Drawing Name 已在 Todo 卡片摘要中展示，弹框不重复）。
4. **双路访问文件**：文件名链接（inline，新标签页打开）与下载按钮（attachment，本地保存）满足不同使用场景，两者并存不互斥。
5. **操作防护**：Approve / Reject 均需二次确认 Dialog + loading 防重复提交；点击 [Approve] 前先做 DC 配置预检查，未配置时阻断并引导。

---

## 2. 信息架构

### 2.1 页面层级

```
Todo 面板（全局铃铛通知）
└── Internal Approval Required 卡片列表（仅我名下）
    └── [Detail] → 详情侧滑弹框（Detail Drawer，480px）
        ├── [Download Original File]（底部操作区）
        ├── [Approve] → 无 DC 警告弹窗（F-007）
        │           → Approve 确认 Dialog（F-005）
        └── [Reject]  → Reject 驳回 Dialog（F-006）
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发角色 |
|---------|-------|---------|
| 全局铃铛通知 | Todo 面板 → Internal Approval Required 卡片（仅我名下） | 内部审批人 |
| Todo 卡片 [Detail] 按钮 | 详情侧滑弹框（Detail Drawer） | 内部审批人 |

---

## 3. 页面 / 组件清单

### 3.1 Internal Approval Required — Todo 卡片（F-001）

**关联 Story**：US-007A-001、US-007A-002
**关联 AC**：AC-007A-001 ~ 003

#### 布局

```
┌─────────────────────────────────────────────────────┐
│ 🔍 Internal Approval Required                       │
│                                                     │
│ ARCH-001  首层平面图  V3                             │
│ Uploaded by: 张三（Designer）  |  2026-04-01 10:00  │
│ Version Note: 修正轴网尺寸（选填，无则整行隐藏）       │
│                                                     │
│ [Detail]                                            │
└─────────────────────────────────────────────────────┘
```

#### 字段说明

| 元素 | 内容 | 备注 |
|------|------|------|
| 图标 | 🔍 | 固定，区分内部审批（🔍）与外部审批（🌐） |
| 标题 | `Internal Approval Required` | 固定文案，加粗 |
| 图纸信息行 | `{drawingCode}  {drawingName}  {versionNo}` | 三项同行；`drawingCode` 加粗 |
| Uploaded by | `{designerName}（Designer）  \|  {uploadTime}` | 时间格式：YYYY-MM-DD HH:mm |
| Version Note | 版本修改说明 | 选填；值为空时**整行隐藏** |
| [Detail] | 打开详情侧滑弹框（§3.2） | 次要样式按钮（`el-button` 默认） |

> ⚠️ **与旧版区别**：卡片上不再保留 [View Drawing]、[Approve]、[Reject] 按钮，所有操作统一收敛至 Detail Drawer。

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 当前用户名下无待内部审批任务 | 空状态插图 + 文字 | `"No pending tasks"` |
| 加载 | Todo 面板数据请求中 | Skeleton 骨架屏 | — |
| 正常 | 有待审批卡片 | 如布局图所示 | — |
| 错误 | 接口加载失败 | 错误图标 + 重试链接 | `"Failed to load. Retry"` |
| 极端数据 | 卡片数量 > 20 | 分页 / 滚动，不截断 | 顶部提示 `"Showing 20 of {n}"` |

#### 权限可见性

| UI 元素 | 内部审批人（本人） | 内部审批人（他人） | DC | 设计人员 | SE |
|--------|:-----------------:|:----------------:|:--:|:-------:|:--:|
| 🔍 卡片（整体） | ✅ | ❌ | ❌ | ❌ | ❌ |
| [Detail] 按钮 | ✅ | ❌ | ❌ | ❌ | ❌ |

---

### 3.2 详情侧滑弹框（Detail Drawer，F-003）

**关联 Story**：US-007A-002、US-007A-003
**关联 AC**：AC-007A-004 ~ 006、AC-007A-013

#### 规格

| 属性 | 值 |
|------|----|
| 宽度 | **480px** |
| 位置 | 从页面右侧滑入 |
| 遮罩 | 半透明（背景列表不可操作） |
| 关闭方式 | 右上角 [✕] 或按 Esc；关闭后审批状态不变 |

#### 弹框头部

```
┌────────────────────────────────────────────────────────────┐
│  Drawing Approval Detail                              ×    │
│  Internal Approval Required                               │
└────────────────────────────────────────────────────────────┘
```

#### 内容区布局

```
┌────────────────────────────────────────────────────────────┐
│  Drawing Approval Detail                              ×    │
│  Internal Approval Required                               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Category            Architecture                         │
│  Description         包含外墙及核心筒轮廓线                 │
│                                                            │
│  Version             V3                                   │
│  Version Note        修正轴网尺寸，更新柱位标注              │
│                                                            │
│  Uploaded by         张三（Designer）                      │
│  Upload Time         2026-05-20 14:32                     │
│  Internal Approver   李四（当前用户）                       │
│                                                            │
│  Original File       📄 ARCH-001-V3-plan.pdf  ↗           │
│                      Click to open in browser.            │
│                      To save the file, use                │
│                      [Download Original File] below.      │
│                                                            │
├────────────────────────────────────────────────────────────┤
│  [Download Original File]  [Approve]  [Reject]            │
└────────────────────────────────────────────────────────────┘
```

#### 展示字段说明

| 字段名 | 数据来源 | 对应上传字段 | 说明 |
|--------|---------|------------|------|
| Category | Drawing.category | Category | 只读 |
| Description | Drawing.description | Description（选填） | 无内容时显示 `—` |
| Version | DrawingVersion.versionNo | 系统自动递增版本号 | 例：V3 |
| Version Note | DrawingVersion.versionNote | Version Note（选填） | 无内容时显示 `—` |
| Uploaded by | DrawingVersion.uploaderName | 上传人 | 姓名 + 角色标签，例：张三（Designer） |
| Upload Time | DrawingVersion.uploadTime | 上传时间 | 格式：`YYYY-MM-DD HH:mm`；Tooltip 显示完整时间戳 |
| Internal Approver | DrawingVersion.approverName | Internal Approver | 被指派的审批人姓名（应与当前登录用户一致） |
| Original File | DrawingVersion.fileUrl | 上传的图纸文件 | 文件名为可点击链接（↗ 新标签页 inline 打开）；下方有引导提示文案 |

> **设计决策**：Drawing Code 和 Drawing Name 已在 Todo 卡片摘要行展示，弹框**不重复**展示这两个字段（对应 AC-007A-005）。

#### Original File 字段状态

| 场景 | 文件名链接 | [Download Original File] 按钮 |
|------|:---------:|:-----------------------------:|
| `fileUrl` 存在 | 可点击链接（↗ 图标，新标签页打开），下方展示灰色引导提示文案 | 正常可点击 |
| `fileUrl` 为空（数据异常） | 显示 `—`，无链接样式，不可点击，引导提示文案隐藏 | 置灰禁用，hover 显示 Tooltip `"Original file unavailable."` |

#### 交互规则

| 操作 | 行为 |
|------|------|
| 弹框打开 | 显示 Skeleton 加载态，请求完成后渲染字段 |
| 点击文件名链接 | `target="_blank"` 新标签页 inline 打开；弹框保持不变 |
| 详情加载失败 | 弹框内显示 `"Failed to load drawing details."` + [Retry] 按钮 |
| 点击 [Retry] | 重新请求详情接口 |
| 点击 [✕] 或 Esc | 关闭弹框，审批状态不变 |
| 点击 [Approve] | 先执行 DC 配置预检查：无 DC → 弹出 F-007 警告弹窗；有 DC → 弹出 F-005 确认 Dialog |
| 点击 [Reject] | 弹出 F-006 驳回 Dialog（不依赖 DC 配置） |

---

### 3.3 [Download Original File] 按钮（F-004）

**关联 Story**：US-007A-004
**关联 AC**：AC-007A-007 ~ 012

#### 规格

| 属性 | 值 |
|------|----|
| 位置 | 详情侧滑弹框底部操作区最左侧 |
| 样式 | 次要按钮（`el-button` 默认样式） |
| 标签文案 | `Download Original File` |

#### 下载规则

- 点击后触发浏览器原生文件下载（HTTP `Content-Disposition: attachment`），强制保存到本地
- 文件命名规则：`{drawingCode}-V{versionNo}-original.{ext}`（示例：`ARCH-001-V3-original.pdf`）
- 支持文件类型：PDF / DWG / DXF / PNG / JPG

#### 按钮状态（4 态）

| 状态 | 触发条件 | 视觉表现 |
|-----|---------|---------|
| 正常 | `fileUrl` 存在 | 次要按钮，可点击 |
| loading | 点击后请求进行中 | 按钮 spinner，文案变 `"Downloading..."`，不可重复点击 |
| 恢复 | 浏览器接管下载后 | 恢复正常状态 |
| 禁用 | `fileUrl` 为空 | 置灰 disabled，hover Tooltip `"Original file unavailable."` |

#### 交互规则

| 操作 | Toast |
|------|-------|
| 下载成功（浏览器接管） | 无 Toast，静默完成 |
| 下载失败（接口 4xx / 5xx） | `"Download failed. Please try again."` |

> **接口参考**：
> - 文件名链接（inline）：`GET /drawing/version/{versionId}/view` → 302 预签名 URL，`Content-Disposition: inline`
> - 下载按钮（attachment）：`GET /drawing/version/{versionId}/download` → 302 预签名 URL，`Content-Disposition: attachment; filename=...`
> - 两接口须校验 `assigneeId = 当前登录用户`，否则返回 403

---

### 3.4 Approve 确认 Dialog（F-005）

**关联 Story**：US-007A-005
**关联 AC**：AC-007A-015 ~ 020

**触发前置**：前端 DC 配置预检查通过（项目已配置 DC）后方可弹出。

#### 布局

```
┌──────────────────────────────────────────────┐
│  Confirm Internal Approval?                  │
│                                              │
│  Once approved, this version will proceed    │
│  to external approval by Document Controller.│
│  The version will NOT become active until    │
│  external approval is completed.             │
│                                              │
│            [Cancel]     [Confirm]            │
└──────────────────────────────────────────────┘
```

#### 交互规则

| 操作 | 状态变化 | Toast |
|------|---------|-------|
| 点击 [Confirm] | 按钮进入 loading，Dialog 内所有操作禁用 | — |
| 操作成功 | Dialog 关闭 → Detail Drawer 关闭 → Todo 卡片消失 | `"Internal approval completed. DC has been notified for external approval."` |
| 操作失败（含后端错误码 1003007012） | loading 恢复，Dialog 保留 | 接口错误文案，可重试 |
| 点击 [Cancel] | Dialog 关闭，返回 Detail Drawer | — |

---

### 3.5 Reject 驳回 Dialog（F-006）

**关联 Story**：US-007A-006
**关联 AC**：AC-007A-021 ~ 023

#### 布局

```
┌──────────────────────────────────────────────┐
│  Reject Internal Approval                    │
│  ARCH-001  首层平面图  V3                     │
│                                              │
│  Comment *                                   │
│  ┌────────────────────────────────────────┐  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│  最多 500 字符                               │
│                                              │
│            [Cancel]     [Confirm]            │
└──────────────────────────────────────────────┘
```

#### 交互规则

| 场景 | 行为 |
|------|------|
| Comment 为空点击 [Confirm] | 前端拦截，字段标红，提示必填 |
| 点击 [Confirm]（Comment 已填） | 按钮进入 loading，Dialog 内所有操作禁用 |
| 操作成功 | Dialog 关闭 → Detail Drawer 关闭 → Todo 卡片消失 |
| 操作失败 | loading 恢复，Dialog 保留，Toast 报错 |
| 点击 [Cancel] | Dialog 关闭，返回 Detail Drawer |

| 操作 | Toast |
|------|-------|
| 驳回成功 | `"Internal approval rejected. Designer has been notified."` |
| 操作失败 | 接口错误文案，可重试 |

---

### 3.6 无 DC 警告弹窗（F-007）

**关联 Story**：US-007A-005
**关联 AC**：AC-007A-014

**触发条件**：点击 Detail Drawer 内 [Approve] 后，前端调用 `GET /project/dc-config` 返回空列表时弹出，**不进入** F-005 确认 Dialog。

#### 布局

```
┌──────────────────────────────────────────────┐
│                                              │
│         ⚠️  No DC Configured                 │
│                                              │
│  This project has no Document Controller    │
│  configured. Please contact your project    │
│  admin to add a DC before proceeding        │
│  with internal approval.                    │
│                                              │
│                          [Got it]            │
└──────────────────────────────────────────────┘
```

#### 规格

| 属性 | 值 |
|------|----|
| 宽度 | 420px |
| 图标 | ⚠️（warning，橙色，56px，居中） |
| 标题 | `"No DC Configured"`，居中，16px 加粗 |
| 正文 | 居中对齐，14px 正常字重 |
| 按钮 | `[Got it]` — 主要按钮，右对齐 |
| 关闭方式 | 仅 [Got it] 按钮；**禁止**背景点击关闭；**禁止** ESC 键关闭 |
| 关闭后效果 | 弹窗关闭，返回 Detail Drawer；审批状态不变；[Reject] 功能不受影响 |

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色语义

| 用途 | Token | 色值 |
|-----|-------|------|
| 审批状态：Pending Internal / Pending External | `--color-status-waiting` | `#E6A23C` |
| 审批状态：Internal Rejected | `--color-status-error` | `#F56C6C` |
| 审批状态：Approved | `--color-status-success` | `#67C23A` |
| 文件名链接颜色 | `--color-primary` | `#409EFF` |
| 引导提示文案颜色 | `--color-text-secondary` | `#909399` |
| 禁用按钮文字 | `--color-text-placeholder` | `#C0C4CC` |

### 4.2 间距与字体

<!--
  ⚠️ 填写规则（见 ui-rules.md §3.10）：
  含文字的组件行，必须在此表内联 font-size + line-height + font-weight + color，禁止仅依赖浏览器默认 16px。
-->

| 组件 | 属性 | 值 |
|-----|------|----|
| Todo 卡片 | padding | 16px 20px |
| Todo 卡片标题 | font-size | 14px |
| Todo 卡片标题 | font-weight | 600 |
| Todo 卡片标题 | line-height | 22px |
| Todo 卡片标题 | color | `#303133` |
| Todo 卡片图纸信息行 | font-size | 14px |
| Todo 卡片图纸信息行 | font-weight | 500 |
| Todo 卡片图纸信息行 | line-height | 22px |
| Todo 卡片元信息 | font-size | 13px |
| Todo 卡片元信息 | line-height | 20px |
| Todo 卡片元信息 | color | `#606266` |
| Detail Drawer | width | 480px |
| Detail Drawer 字段标签 | font-size | 13px |
| Detail Drawer 字段标签 | line-height | 20px |
| Detail Drawer 字段标签 | color | `#606266` |
| Detail Drawer 字段值 | font-size | 14px |
| Detail Drawer 字段值 | line-height | 22px |
| Detail Drawer 字段值 | color | `#303133` |
| 引导提示文案 | font-size | 12px |
| 引导提示文案 | line-height | 18px |
| 引导提示文案 | color | `#909399` |
| 底部操作区 | padding | 16px 20px |
| 底部操作区 | border-top | 1px solid `#EBEEF5` |
| Dialog | font-size | 14px |
| Dialog | line-height | 22px |
| Dialog | font-weight | 400 |
| Dialog 标题 | font-size | 16px |
| Dialog 标题 | font-weight | 600 |
| Dialog 标题 | line-height | 24px |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 完整布局；Detail Drawer 宽度 480px |
| Mobile | < 768px | 不支持（PC 专属） |

---

## 6. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| Detail Drawer 滑入 | slide-in-right | 300ms | ease-out |
| Detail Drawer 滑出 | slide-out-right | 250ms | ease-in |
| Dialog 出现 | fade-in + scale（0.9→1） | 200ms | ease-out |
| 下载按钮 loading | spinner | — | — |

---

## 7. 无障碍（A11y）

- WCAG 等级：AA
- **键盘导航**：
  - Todo 列表 [Detail] 按钮可通过 Tab 聚焦，Enter 触发
  - Detail Drawer 可通过 Esc 关闭
  - Drawer 底部 [Download Original File]、[Approve]、[Reject] 均可通过 Tab 聚焦并 Enter 触发
  - Dialog 内字段支持 Tab 键导航；Esc 可关闭普通 Dialog（无 DC 警告弹窗 F-007 除外）
- **屏幕阅读器**：
  - 下载按钮须有 `aria-label="Download Original File"`
  - 文件图标须有 `alt` 文本（描述文件类型）
  - 禁用下载按钮须有 `aria-disabled="true"` 及 `title="Original file unavailable."`
  - Dialog 状态变更使用 `aria-live="polite"` 通知
- 颜色对比度：正文 ≥ 4.5:1

---

## 8. 复用与新建组件清单

| 组件 | 来源 | 备注 |
|-----|------|------|
| `el-drawer` | 现有 Element UI | 宽度 480px，右侧滑入 |
| `el-dialog` | 现有 Element UI | Approve 确认 / Reject 驳回 / 无 DC 警告 三个 Dialog |
| `el-button` | 现有 Element UI | [Detail]、[Download Original File]、[Approve]、[Reject] 等 |
| `el-skeleton` | 现有 Element UI | Drawer 内容区加载态 |
| `el-tooltip` | 现有 Element UI | 禁用下载按钮 hover 提示 |
| `InternalApprovalTodoCard` | 现有（改造） | 移除旧版 [View Drawing]、[Approve]、[Reject]，新增 [Detail] |
| `DrawingDetailDrawer` | **新建** | 详情侧滑弹框，包含字段展示 + 底部下载/审批按钮 |

---

## 9. 文案规范

| 场景 | 文案（en） |
|-----|----------|
| Todo 卡片标题 | Internal Approval Required |
| Todo 空状态 | No pending tasks |
| Todo 加载失败 | Failed to load. Retry |
| Drawer 标题 | Drawing Approval Detail |
| Drawer 副标题标签 | Internal Approval Required |
| 字段空值占位符 | — |
| 文件名引导提示 | Click to open in browser. To save the file, use [Download Original File] below. |
| 下载按钮文案 | Download Original File |
| 下载中按钮文案 | Downloading... |
| 下载失败 Toast | Download failed. Please try again. |
| 详情加载失败 | Failed to load drawing details. |
| 原始文件不可用 Tooltip | Original file unavailable. |
| Approve 成功 Toast | Internal approval completed. DC has been notified for external approval. |
| Reject 成功 Toast | Internal approval rejected. Designer has been notified. |
| 无 DC 警告弹窗标题 | No DC Configured |
| 任务已被处理（并发） | This task has already been completed. |

---

## 10. AC 覆盖检查表

| AC ID | 对应章节 / 组件 | 覆盖? |
|------|---------------|------|
| AC-007A-001 | §3.1 Todo 卡片标题与图标 | ✅ |
| AC-007A-002 | §3.1 待办列表仅展示自己名下记录 | ✅ |
| AC-007A-003 | §3.1 无任务时空状态文案 | ✅ |
| AC-007A-004 | §3.2 点击 [Detail] 打开 Drawer，头部标题与标签 | ✅ |
| AC-007A-005 | §3.2 展示字段与上传一致，不含 Drawing Code / Name | ✅ |
| AC-007A-006 | §3.2 选填字段为空时显示 `—` | ✅ |
| AC-007A-007 | §3.3 下载文件命名规则与内容一致 | ✅ |
| AC-007A-008 | §3.3 下载期间按钮 loading，防重复点击 | ✅ |
| AC-007A-009 | §3.3 fileUrl 为空时按钮禁用 + Tooltip；文件名不可点击 | ✅ |
| AC-007A-010 | §3.3 非指派人接口返回 403（后端逻辑，UI 无感知） | ✅ |
| AC-007A-011 | §3.2 文件名链接新标签页 inline 打开，弹框保持不变 | ✅ |
| AC-007A-012 | §3.2 文件名下方引导提示文案 | ✅ |
| AC-007A-013 | §3.2 详情加载失败 → 错误提示 + [Retry] | ✅ |
| AC-007A-014 | §3.6 无 DC 警告弹窗（F-007），不弹出确认 Dialog | ✅ |
| AC-007A-015 | §3.4 有 DC 时弹出 Approve 确认 Dialog，文案说明版本不立即生效 | ✅ |
| AC-007A-016 | §3.4 Approve 成功路径：Drawer 关闭、Todo 消失、Toast | ✅ |
| AC-007A-017 | §3.4 版本不生效（后端逻辑，UI 无感知） | ✅ |
| AC-007A-018 | DC 收到外部审批通知（后端逻辑，见 UI-REQ-007B-pc） | ✅ |
| AC-007A-019 | §3.4 Approve Dialog loading 态及失败可重试 | ✅ |
| AC-007A-020 | §3.4 后端兜底无 DC 错误码 1003007012，Toast 提示 | ✅ |
| AC-007A-021 | §3.5 Reject Comment 必填校验，字段标红 | ✅ |
| AC-007A-022 | §3.5 Reject 成功路径：Drawer 关闭、Todo 消失、Toast | ✅ |
| AC-007A-023 | §3.5 旧版本保持有效（后端逻辑，UI 无感知） | ✅ |

---

## 11. 待定问题（Open Questions）

| OQ ID | 问题 | 影响 UI 哪部分 | 来源 |
|------|------|--------------|------|
| OQ-001 | 内部审批超时（如 3 天未处理）是否需要发送催办通知？若有催办通知，Todo 卡片是否需要展示"超时"状态样式？ | §3.1 卡片状态 | REQ-007A OQ-001 |
| OQ-002 | 预签名下载 URL 有效期（5 分钟是否合理，是否一次性使用）？ | §3.3 下载失败 Toast 时机 | REQ-007A OQ-002 |
| OQ-003 | 是否需要在 Drawer 中同时提供在线预览入口（区别于下载），还是仅保留下载？ | §3.2 内容区 / 底部操作区 | REQ-007A OQ-003 |

---

## 12. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.3.0 | 2026-05-25 | agent | 将 UI-REQ-007E-pc（v0.1.0）完整合并至本文档；新增 §3.2（Detail Drawer）、§3.3（Download 按钮）、§3.4（Approve Dialog）、§3.5（Reject Dialog）、§3.6（无 DC 警告弹窗）；Todo 卡片（§3.1）更新为仅保留 [Detail] 按钮；权限矩阵扩展；AC 覆盖表扩展至 AC-007A-001~023；更新设计令牌、文案规范、组件清单、A11y；UI-REQ-007E-pc.md 已删除 |
| 0.2.0 | 2026-05-24 | agent | 按来源需求独立成文件；补充 §3.2b 无 DC 警告弹窗完整规格 |
| 0.1.0 | 2026-05-07 | agent | 从 UI-REQ-007-pc.md 拆分，覆盖 REQ-007A-pc 内部审批部分 |


# UI 设计说明：PC 端 — 内部审批 Todo 处理

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-007A-pc.md（主）、REQ-007-shared（业务规则）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc § 3.2
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007A-pc @ v0.2.0 |
| 覆盖用户故事 | US-007A-001 |
| 覆盖 AC | AC-007A-001 ~ AC-007A-011 |
| 上次同步时间 | 2026-05-24 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

让内部审批人能在 Todo 面板中快速定位自己名下的图纸待审批任务，并在一张卡片内完成 Approve / Reject 全部操作，减少页面跳转。

### 1.2 设计原则

1. **角色专属视图**：🔍 Internal 卡片仅对内部审批人可见，与 🌐 External 卡片图标和文案区分度高。
2. **一卡一动作**：卡片提供完整上下文，无需离开 Todo 面板查找信息。
3. **操作防护**：Approve / Reject 均需二次确认 Dialog + loading 防重复提交。
4. **无 DC 预检查**：点击 [Approve] 时前端先检查项目是否已配置 DC，未配置时阻断并引导管理员处理。

---

## 2. 信息架构

### 2.1 页面层级

```
Todo 面板（全局通知/铃铛）
└── Internal Approval Required 卡片列表
    ├── Approve 确认 Dialog
    ├── 无 DC 警告弹窗（F-004）
    └── Reject 驳回 Dialog
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发角色 |
|---------|-------|---------|
| 全局铃铛通知 | Todo 面板 → Internal Approval Required 卡片 | 内部审批人 |

---

## 3. 页面 / 组件清单

### 3.1 Internal Approval Required — Todo 卡片

**关联 Story**：US-007A-001
**关联 AC**：AC-007A-001 ~ 009

#### 布局

```
┌─────────────────────────────────────────────────────┐
│ 🔍 Internal Approval Required                       │
│                                                     │
│ ARCH-001  首层平面图  V3                             │
│ Uploaded by: 张三（Designer）  |  2026-04-01 10:00  │
│ Version Note: 修正轴网尺寸（选填，无则不显示）         │
│                                                     │
│ [View Drawing]   [Approve]   [Reject]               │
└─────────────────────────────────────────────────────┘
```

#### 字段说明

| 元素 | 内容 | 备注 |
|------|------|------|
| 图标 | 🔍 | 固定，区分内部审批与外部审批（🌐） |
| 标题 | `Internal Approval Required` | 固定文案 |
| 图纸信息行 | `{drawingCode}  {drawingName}  {versionNo}` | 三项同行，`code` 加粗 |
| Uploaded by | `{designerName}（Designer）  \|  {uploadTime}` | 时间格式 YYYY-MM-DD HH:mm |
| Version Note | 版本修改说明 | 选填字段；值为空时整行隐藏 |
| [View Drawing] | 打开 PDF 预览 / 文件下载（`fileUrl`） | 次要按钮 |
| [Approve] | 前端预检查 DC；有 DC → Approve 确认 Dialog（§3.2）；无 DC → 无 DC 警告弹窗（§3.2b） | 主要按钮，蓝色 |
| [Reject] | 打开 Reject 驳回 Dialog（§3.3） | 默认按钮 |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 当前无待内部审批任务 | 空状态插图 + 文字 | "No pending internal approvals." |
| 加载 | Todo 面板数据请求中 | Skeleton 骨架屏 | — |
| 正常 | 有待审批卡片 | 如布局图所示 | — |
| 错误 | 接口加载失败 | 错误图标 + 重试链接 | "Failed to load. Retry" |
| 极端数据 | 卡片数量 > 20 | 分页 / 滚动，不截断 | 顶部提示"Showing 20 of {n}" |

#### 权限可见性

| UI 元素 | 内部审批人 | DC | 设计人员 | SE |
|--------|:---------:|:--:|:-------:|:--:|
| 🔍 卡片（整体） | ✅ | ❌ | ❌ | ❌ |
| [View Drawing] | ✅ | — | — | — |
| [Approve] | ✅ | — | — | — |
| [Reject] | ✅ | — | — | — |

---

### 3.2 Approve 确认 Dialog

**关联 AC**：AC-007A-002、AC-007A-003、AC-007A-008、AC-007A-009

#### 布局

```
┌──────────────────────────────────────────────┐
│  Confirm Internal Approval?                  │
│                                              │
│  Once approved, this version will proceed   │
│  to external approval by Document           │
│  Controller. The version will NOT become    │
│  active until external approval is          │
│  completed.                                 │
│                                              │
│            [Cancel]     [Confirm]            │
└──────────────────────────────────────────────┘
```

#### 交互规则

| 操作 | 状态变化 | Toast |
|------|---------|-------|
| 点击 [Confirm] | 按钮进入 loading，Dialog 内所有操作禁用 | — |
| 操作成功 | Dialog 关闭，Todo 卡片消失 | `"Internal approval completed. DC has been notified for external approval."` |
| 操作失败 | loading 恢复，Dialog 保留 | 接口错误文案，可重试 |
| 点击 [Cancel] | Dialog 关闭 | — |

---

### 3.2b 无 DC 警告弹窗（F-004）

**关联 AC**：AC-007A-010

**触发条件**：点击 [Approve] 后，前端预检查发现项目未配置任何 DC。

#### 布局

```
┌──────────────────────────────────────────────┐
│                                              │
│         ⚠️  No DC Configured                 │
│                                              │
│  This project has no Document Controller    │
│  configured. Please contact your project    │
│  admin to add a DC before proceeding        │
│  with internal approval.                    │
│                                              │
│                           [Got it]           │
└──────────────────────────────────────────────┘
```

#### 规格

| 属性 | 值 |
|------|----|
| 宽度 | 420px |
| 图标 | ⚠️（warning，橙色），居中，56px |
| 标题 | "No DC Configured"，居中，16px 加粗 |
| 正文 | 两行文案，居中对齐，14px 正常 |
| 按钮 | [Got it] — 主要按钮，右对齐 |
| 关闭方式 | 仅通过 [Got it] 按钮关闭；**禁止**背景点击关闭；**禁止** ESC 键关闭 |
| 关闭后效果 | 弹窗关闭，卡片状态不变，[Reject] 功能不受影响 |

---

### 3.3 Reject 驳回 Dialog

**关联 AC**：AC-007A-005、AC-007A-006、AC-007A-007

#### 布局

```
┌──────────────────────────────────────────────┐
│  Reject Internal Approval                    │
│  ARCH-001  首层平面图  V3                     │
│                                              │
│  Comment *                                   │
│  ┌────────────────────────────────────────┐  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│  最多 500 字符                               │
│                                              │
│            [Cancel]     [Confirm]            │
└──────────────────────────────────────────────┘
```

#### 交互规则

| 场景 | 行为 |
|------|------|
| Comment 为空点击 [Confirm] | 前端拦截，字段标红，提示必填 |
| 操作成功 | Dialog 关闭，Todo 消失，Toast `"Internal approval rejected. Designer has been notified."` |
| 操作失败 | loading 恢复，Dialog 保留，Toast 报错 |

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色语义（审批状态）

| 状态值 | 标签文案 | 颜色 Token | 色值 |
|--------|---------|-----------|------|
| PENDING_INTERNAL | ⏳ Pending Internal | `--color-status-waiting` | `#E6A23C` |
| INTERNAL_APPROVED | ⏳ Pending External | `--color-status-waiting` | `#E6A23C` |
| INTERNAL_REJECTED | ❌ Int. Rejected | `--color-status-error` | `#F56C6C` |
| APPROVED | ✅ Approved | `--color-status-success` | `#67C23A` |

### 4.2 间距与字体

| 组件 | 属性 | 值 |
|-----|------|----|
| Todo 卡片 | padding | 16px 20px |
| Dialog | font-size | 14px |
| Dialog | line-height | 22px |
| Dialog | font-weight | 400 |
| Dialog 标题 | font-size | 16px |
| Dialog 标题 | font-weight | 600 |
| Todo 卡片标题 | font-size | 14px |
| Todo 卡片标题 | font-weight | 600 |
| Todo 卡片标题 | color | `#303133` |
| Todo 卡片图纸信息行 | font-size | 14px |
| Todo 卡片图纸信息行 | font-weight | 500 |
| Todo 卡片元信息 | font-size | 13px |
| Todo 卡片元信息 | color | `#606266` |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 完整布局 |
| Mobile | < 768px | 不支持（PC 专属） |

---

## 6. 无障碍（A11y）

- WCAG 等级：AA
- 键盘导航：Dialog 内所有字段支持 Tab 键；Esc 可关闭普通 Dialog（无 DC 警告弹窗除外）
- 屏幕阅读器：Dialog 状态变更使用 `aria-live="polite"` 通知

---

## 7. 文案规范

| 场景 | 文案（en） |
|-----|----------|
| 内部审批 Todo 标题 | Internal Approval Required |
| Approve 成功 Toast | Internal approval completed. DC has been notified for external approval. |
| Reject 成功 Toast | Internal approval rejected. Designer has been notified. |
| 无 DC 警告弹窗标题 | No DC Configured |

---

## 8. AC 覆盖检查表

| AC ID | 对应组件 | 覆盖? |
|------|---------|------|
| AC-007A-001 | §3.1 Todo 卡片 | ✅ |
| AC-007A-002 | §3.2 Approve Dialog 成功路径 | ✅ |
| AC-007A-003 | §3.2 版本不生效说明文案 | ✅ |
| AC-007A-004 | DC 收到外部任务（后端逻辑，见 UI-REQ-007B-pc） | ✅ |
| AC-007A-005 | §3.3 Comment 必填校验 | ✅ |
| AC-007A-006 | §3.3 Reject 成功路径 | ✅ |
| AC-007A-007 | §3.3 旧版本保持有效（后端逻辑，UI 无感知） | ✅ |
| AC-007A-008 | §3.2 / §3.3 loading 防重复提交 | ✅ |
| AC-007A-009 | §3.2 / §3.3 失败可重试 | ✅ |
| AC-007A-010 | §3.2b 无 DC 警告弹窗 | ✅ |
| AC-007A-011 | §3.2b 关闭方式限制 | ✅ |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 UI-REQ-007-pc.md 拆分，覆盖 REQ-007A-pc 内部审批部分 |
| 0.2.0 | 2026-05-24 | agent | 按来源需求独立成文件，补充 §3.2b 无 DC 警告弹窗完整规格 |
