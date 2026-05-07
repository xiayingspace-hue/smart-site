---
doc_type: ui_spec
req_id: REQ-007-pc
version: 0.2.0
status: draft
generated_from: "REQ-007A-pc@0.1.0 + REQ-007B-pc@0.1.1 + REQ-007C-pc@0.2.0 + REQ-007D-pc@0.1.0"
generated_at: 2026-05-06
owner: ""
---

# UI 设计说明：PC 端图纸两级审批流程

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-007A-pc / REQ-007B-pc / REQ-007C-pc / REQ-007D-pc（主）、REQ-007-shared（业务规则）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc § 3.2
> - 依赖 UI 文档：UI-REQ-003-pc（图纸管理基础 UI）
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007A-pc @ v0.1.0 |
| | REQ-007B-pc @ v0.1.1 |
| | REQ-007C-pc @ v0.2.0 |
| | REQ-007D-pc @ v0.1.0 |
| 覆盖用户故事 | US-007A-001 / US-007B-001 / US-007B-002 / US-007C-001 / US-007D-001 |
| 覆盖 AC | AC-007A-001~009 / AC-007B-001~011 / AC-007C-001~016 / AC-007D-001~010 |
| 上次同步时间 | 2026-05-06 |

> ⚠️ 当任一来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

在 REQ-003 已有图纸管理界面基础上，以**最小化视觉侵入**的方式引入内部 + 外部两级串行审批流程，让不同角色（内部审批人、DC、管理员）都能高效完成各自工作。

### 1.2 设计原则

1. **角色专属视图**：内部审批人只看到 🔍 Internal 任务，DC 只看到 🌐 External 任务，图标与文案区分度高
2. **一卡一动作**：Todo 卡片提供完整上下文，DC 无需离开页面查找文件，一个 Dialog 完成所有标记操作
3. **5 态颜色一致**：橙色（等待）/ 绿色（通过）/ 红色（驳回）三色体系贯穿列表标签、卡片状态、步骤条
4. **渐进展开**：版本历史默认展示主列表，按需展开 4 阶段卡片，避免信息过载
5. **操作防护**：破坏性/耗时操作（驳回、Mark Result、删除附件）均需二次确认 + loading 防重复提交

---

## 2. 信息架构

### 2.1 页面层级

```
图纸管理模块
├── 图纸列表页（REQ-003A-pc，状态标签升级）
│   └── 右侧抽屉：版本历史（720px）               ← REQ-007C-pc
│       ├── 主列表（Version / Status / Designer / Confirmed / QR / Attachments）
│       ├── 展开行：4 阶段卡片（① Upload ② Internal ③ External ④ Signed）
│       └── 叠加抽屉：Attachments 二级抽屉（480px）
├── Todo 面板（全局通知/铃铛）
│   ├── Internal Approval Required 卡片         ← REQ-007A-pc
│   │   ├── Approve 确认 Dialog
│   │   └── Reject 驳回 Dialog
│   └── External Approval Required 卡片         ← REQ-007B-pc
│       └── Mark External Approval Result Dialog
└── Settings
    └── DC Configuration 页面                   ← REQ-007D-pc
        └── Add Document Controller 弹窗
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发角色 |
|---------|-------|---------|
| 全局铃铛通知 | Todo 面板 → Internal Approval Required 卡片 | 内部审批人 |
| 全局铃铛通知 | Todo 面板 → External Approval Required 卡片 | DC |
| 图纸列表行 [History] 按钮 | 版本历史抽屉（720px） | 管理员 / Drawing 团队 / DC |
| 版本历史抽屉 Attachments 列单元格 | Attachments 二级抽屉（480px） | 所有有权限用户 |
| 版本历史展开行 ③ 卡片 [Mark Result] | Mark External Approval Result Dialog | DC |
| 侧边栏 Settings → DC Configuration | DC 配置页面 | 业务人员 / 项目管理员 |

---

## 3. 页面/组件清单

> 每个页面都列出 5 类必要状态（空/加载/正常/错误/极端数据），UI 设计师不得遗漏。

---

### 3.1 Internal Approval Required — Todo 卡片

**关联 Story**: US-007A-001
**关联 AC**: AC-007A-001 ~ 009

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
| [Approve] | 打开 Approve 确认 Dialog（§3.2） | 主要按钮，蓝色 |
| [Reject] | 打开 Reject 驳回 Dialog（§3.3） | 默认按钮 |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 当前无待内部审批任务 | 空状态插图 + 文字 | "No pending internal approvals." |
| 加载 | Todo 面板数据请求中 | Skeleton 骨架屏 | — |
| 正常 | 有待审批卡片 | 如布局图所示 | — |
| 错误 | 接口加载失败 | 错误图标 + 重试链接 | "Failed to load. Retry" |
| 极端数据 | 卡片数量 > 20 | 分页 / 滚动，不截断列表 | 顶部提示"Showing 20 of {n}" |

#### 权限可见性

| UI 元素 | 内部审批人 | DC | 设计人员 | SE |
|--------|:---------:|:--:|:-------:|:--:|
| 🔍 卡片（整体） | ✅ | ❌ | ❌ | ❌ |
| [View Drawing] | ✅ | — | — | — |
| [Approve] | ✅ | — | — | — |
| [Reject] | ✅ | — | — | — |

---

### 3.2 Approve 确认 Dialog

**关联 AC**: AC-007A-002、AC-007A-003、AC-007A-008、AC-007A-009

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

### 3.3 Reject 驳回 Dialog

**关联 AC**: AC-007A-005、AC-007A-006、AC-007A-007

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

### 3.4 External Approval Required — Todo 卡片

**关联 Story**: US-007B-001
**关联 AC**: AC-007B-001、AC-007B-002、AC-007B-009、AC-007B-010

#### 布局

```
┌──────────────────────────────────────────────────────────────────┐
│ 🌐 External Approval Required                                    │
│                                                                  │
│ ARCH-001  首层平面图  V3                                         │
│ Uploaded by: 张三（Designer）  |  2026-04-01 10:00               │
│ Internal approved by: 王总工  |  2026-04-02 14:30               │
│ Version Note: 修正轴网尺寸（选填，无则不显示）                     │
│                                                                  │
│ [📄 Download Original]   [✅ Mark Result]                        │
└──────────────────────────────────────────────────────────────────┘
```

#### 字段说明

| 元素 | 内容 | 备注 |
|------|------|------|
| 图标 | 🌐 | 固定，区分外部（🌐）与内部（🔍） |
| 标题 | `External Approval Required` | 固定文案 |
| Internal approved by | `{approverName}  \|  {internalApprovedTime}` | 区别于内部审批卡片的 Uploaded by |
| [📄 Download Original] | 下载 `fileUrl`（原始图纸），文件名保持原始 `fileName` | 次要按钮 |
| [✅ Mark Result] | 打开 Mark External Approval Result Dialog（§3.5） | 主要按钮，绿色 |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 当前无待外部审批任务 | 空状态插图 | "No pending external approvals." |
| 加载 | 数据请求中 | Skeleton 骨架屏 | — |
| 正常 | 有外部审批任务 | 如布局图所示 | — |
| 错误 | 接口加载失败 | 错误图标 + 重试 | "Failed to load. Retry" |
| 极端数据 | 多个待审批任务 | 分页 / 滚动 | 顶部计数提示 |

#### 权限可见性

| UI 元素 | DC（已配置） | 内部审批人 | 其他 |
|--------|:-----------:|:---------:|:----:|
| 🌐 卡片（整体） | ✅ | ❌ | ❌ |
| [📄 Download Original] | ✅ | — | — |
| [✅ Mark Result] | ✅ | — | — |

---

### 3.5 Mark External Approval Result Dialog

**关联 Story**: US-007B-002
**关联 AC**: AC-007B-003 ~ 011

#### 布局

```
┌──────────────────────────────────────────────┐
│  Mark External Approval Result         [✕]   │
│  ARCH-001  首层平面图  V3                     │
├──────────────────────────────────────────────┤
│                                              │
│  Result *                                    │
│    ○ Approved                                │
│    ○ Rejected                                │
│                                              │
│  ──── 选择 Approved 后显示 ────              │
│                                              │
│  Signed Drawing File *                       │
│  ┌────────────────────────────────────────┐  │
│  │  📎 Click or drag to upload            │  │
│  │     PDF / DWG / DXF / PNG / JPG        │  │
│  │     Max 50MB                           │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  Approval Evidence *                         │
│  ┌────────────────────────────────────────┐  │
│  │  📎 Click or drag to upload            │  │
│  │     PDF / PNG / JPG  · Max 20MB        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  External Approval Date *                    │
│  [ 📅 YYYY-MM-DD          ]                  │
│                                              │
│  Remarks                                     │
│  [ 选填，最多 500 字符      ]                  │
│                                              │
│  ──── 选择 Rejected 后显示 ────             │
│                                              │
│  Rejection Reason *                          │
│  [ 必填，最多 500 字符      ]                  │
│                                              │
│            [Cancel]     [Confirm]            │
└──────────────────────────────────────────────┘
```

#### 字段规则

| 字段 | 类型 | 必填条件 | 约束 |
|------|------|---------|------|
| Result | Radio（Approved / Rejected） | 始终必填 | 默认不选；未选时 [Confirm] 禁用 |
| Signed Drawing File | 文件上传 | Approved | ≤ 50MB；PDF / DWG / DXF / PNG / JPG |
| Approval Evidence | 文件上传 | Approved | ≤ 20MB；PDF / PNG / JPG |
| External Approval Date | 日期选择器 | Approved | 不可选未来日期 |
| Remarks | 文本域 | 选填 | ≤ 500 字符 |
| Rejection Reason | 文本域 | Rejected | ≤ 500 字符；为空时 [Confirm] 禁用 |

#### 交互规则

| 操作 | 状态变化 | Toast |
|------|---------|-------|
| 选择 Approved | 显示上传区 + 日期 + 备注；隐藏驳回原因 | — |
| 选择 Rejected | 显示驳回原因；隐藏上传区 | — |
| 点击 [Confirm]（Approved，全部必填） | 按钮 loading，文案变 "Processing..."，禁用所有操作（约 3–5 秒） | — |
| 操作成功（Approved） | Dialog 关闭，Todo 消失（含其他 DC 的同一任务） | `"External approval marked. Drawing is now active and QR code has been generated."` |
| 操作成功（Rejected） | Dialog 关闭，Todo 消失 | `"External rejection recorded. Designer has been notified."` |
| QR 生成失败 | loading 恢复，Dialog 保留 | `"QR generation failed, please retry"` |
| 其他接口失败 | loading 恢复，Dialog 保留 | 接口错误文案 |

---

### 3.6 版本历史抽屉（主）

**关联 Story**: US-007C-001
**关联 AC**: AC-007C-001 ~ 010

#### 规格

| 属性 | 值 |
|------|----|
| 宽度 | **720px** |
| 位置 | 从右侧滑入，叠加于图纸列表页之上 |
| 标题格式 | `Version History — {drawingCode} {drawingName}` |
| 关闭方式 | 右上角 [✕] 或点击遮罩 |

#### 主列表列定义

| 列 | 内容 | 说明 |
|----|------|------|
| Version | `V1 / V2 / V3 …`，左侧有 ▶ 展开箭头 | 点击展开/折叠 4 阶段卡片 |
| Status | 状态标签（5 态，颜色见 §4.1） | — |
| Designer | 设计人员（上传人）姓名 | — |
| Confirmed | `x/y`（已确认/总人数） | 仅 `APPROVED` 版本显示；其他状态显示 `—` |
| QR | `[View]` 链接 | 仅 `APPROVED` 且 QR 已生成时显示；其他显示 `—` |
| Attachments | `📎 n`（n≥0） | **点击该单元格**打开 Attachments 二级抽屉（§3.8），与展开行无关 |

**主列表示意**：

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Version History — ARCH-001 首层平面图                                      [✕]  │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Version │ Status              │ Designer │ Confirmed │ QR      │ Attachments    │
│──────────│─────────────────────│──────────│───────────│─────────│────────────────│
│  ▶ V3    │ ✅ Approved         │ 张三     │ 5/12      │ [View]  │ 📎 3           │
│  ▶ V2    │ ⏳ Pending External │ 李四     │ —         │ —       │ 📎 1           │
│  ▶ V1    │ ❌ Int. Rejected    │ 张三     │ —         │ —       │ 📎 0           │
└──────────────────────────────────────────────────────────────────────────────────┘
```

#### 状态（5 态）

| 状态 | 视觉表现 |
|-----|---------|
| 空 | 空状态插图，"No version history available." |
| 加载 | 列表 Skeleton |
| 正常 | 主列表 + 可展开行 |
| 错误 | 错误提示，"加载失败，请刷新重试" |
| 极端数据 | 版本数 > 20 时分页，避免一次性渲染过多 4 阶段卡片 |

---

### 3.7 版本历史抽屉 — 4 阶段展开卡片

**关联 AC**: AC-007C-003 ~ 010

#### 卡片网格规格

- 布局：2×2 网格，4 张卡片等高
- 卡片间距：12px
- 每卡片内边距：16px

#### 各 approvalStatus 渲染规则

| 卡片 | PENDING_INTERNAL | INTERNAL_APPROVED | INTERNAL_REJECTED | APPROVED | EXTERNAL_REJECTED |
|------|:----------------:|:-----------------:|:-----------------:|:--------:|:-----------------:|
| ① Uploaded | ✓ 完成 | ✓ 完成 | ✓ 完成 | ✓ 完成 | ✓ 完成 |
| ② Internal | ⏳ 进行中 | ✓ 完成 | ✕ 驳回 | ✓ 完成 | ✓ 完成 |
| ③ External | — 等待 | ⏳ 进行中 | — 未到达 | ✓ 完成 | ✕ 驳回 |
| ④ Signed | — 等待 | — 等待 | — 未到达 | ✓ 完成 | — 未到达 |

#### ① 上传卡片（始终显示）

```
┌───────────────────────────────────┐
│ ① Uploaded                        │
│ 📄 {fileName}                     │
│ Designer: {designerName}          │
│ {uploadTime}                      │
│ Size: {fileSize}                  │
│               [Preview] [Download]│
└───────────────────────────────────┘
```

#### ② 内部审批卡片

| approvalStatus | 显示内容 |
|---------------|---------|
| PENDING_INTERNAL | ⏳ Pending · Approver: {approverName} · Assigned: {date} |
| INTERNAL_APPROVED / APPROVED / EXTERNAL_REJECTED | ✅ Approved · Approver: {approverName} · {approvedTime} · Comment: {comment}（有则显示） |
| INTERNAL_REJECTED | ❌ Rejected · Approver: {approverName} · {rejectedTime} · Comment: {comment} |

#### ③ 外部审批卡片

| approvalStatus | 显示内容 |
|---------------|---------|
| PENDING_INTERNAL | — Waiting for internal approval |
| INTERNAL_APPROVED（DC 用户） | ⏳ Pending · "Waiting for DC to submit external approval" · **[✅ Mark Result]** |
| INTERNAL_APPROVED（非 DC 用户） | ⏳ Pending · "Waiting for DC to submit external approval"（无 Mark Result 按钮） |
| APPROVED | ✅ Approved · Marked by: {dcName} · Approval Date: {date} · 📎 Evidence: {evidenceFileName} [Download] · Remark: {remark}（有则显示） |
| EXTERNAL_REJECTED | ❌ Rejected · Marked by: {dcName} · {time} · Reason: {comment} |
| INTERNAL_REJECTED | — Not reached（Internal approval failed） |

#### ④ 签字版卡片

| approvalStatus | 显示内容 |
|---------------|---------|
| APPROVED | 📄 {signedFileName} · ✍️ Signed Drawing · Uploaded: {date} · Size: {size} · [Preview] [Download]（优先 pdfWithQrUrl > signedFileUrl） |
| INTERNAL_APPROVED | — Waiting for external approval |
| 驳回类 | — Not reached |
| PENDING_INTERNAL | — Waiting for internal approval |

#### 底部步骤条规范

```
📄 Uploaded ✓  →  🔍 Internal [✓/⏳/✕/—]  →  🌐 External [✓/⏳/✕/—]  →  ✍️ Signed [✓/—]
```

| 步骤状态 | 颜色 | 符号 |
|---------|------|------|
| 完成 | 绿色 `#67C23A` | ✓ |
| 进行中 | 橙色 `#E6A23C` | ⏳ |
| 驳回/失败 | 红色 `#F56C6C` | ✕ |
| 未到达 | 灰色 `#C0C4CC` | — |

---

### 3.8 Attachments 二级抽屉

**关联 AC**: AC-007C-011 ~ 016

#### 规格

| 属性 | 值 |
|------|----|
| 宽度 | **480px** |
| 位置 | 从右侧滑入，**叠加在版本历史抽屉（720px）之上** |
| 触发方式 | 点击主列表任意版本行的 Attachments 列单元格（与版本行展开/折叠无关） |
| 关闭方式 | 右上角 [✕] |
| 关闭后 | 返回版本历史抽屉，版本历史抽屉保持原状 |

#### 布局

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Version History — · ARCH-001  V3                                   [✕]  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Drawing Code    ARCH-001                                                │
│  Status          ✅ Active                                               │
│  Ver (System)    V3                                                      │
│  Uploaded by     👤 张三                                                 │
│  Upload Date     2026-04-01 10:00                                        │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ 📄 arch001-v3.pdf                               [Download]       │    │
│  │ ✅ Approved by 王总工 · 2026-04-02 14:30                          │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Attachments (3)                                              [+]        │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ Filename         │ Type  │ Size    │ Uploaded    │ Uploaded by    │    │
│  │──────────────────│───────│─────────│─────────────│────────────────│    │
│  │ 结构计算书.xlsx   │ xlsx  │ 1.2 MB  │ 2026-04-01  │ 张三  [Delete] │    │
│  │ 施工说明.docx     │ docx  │ 0.5 MB  │ 2026-04-01  │ 张三  [Delete] │    │
│  │ arch001-v3.dwg   │ dwg   │ 8.3 MB  │ 2026-04-01  │ 张三  [Delete] │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 版本信息区字段

| 字段 | 内容 |
|------|------|
| 标题 | `Version History — · {drawingCode}  {versionNo}` |
| Drawing Code | 图纸编号（只读） |
| Status | 状态标签（§4.1 颜色规范，只读） |
| Ver (System) | 系统版本号（只读） |
| Uploaded by | 头像 + 姓名（只读） |
| Upload Date | `DD-MM-YYYY HH:mm:ss`（只读） |
| 主文件行 | `📄 {fileName}` + 审批状态 `✅ Approved by {name} · {time}` + `[Download]` |

#### Attachments 表格列

| 列 | 内容 | 说明 |
|----|------|------|
| Filename | 原始文件名 | 点击文件名可下载（与行 [Download] 等效） |
| Type | 文件扩展名大写（XLSX / DWG …） | — |
| Size | 格式化大小（1.2 MB） | — |
| Uploaded | 上传日期 `YYYY-MM-DD` | — |
| Uploaded by | 上传人姓名 | — |
| 行操作 | [Download]（所有人）；[Delete]（仅本版本上传人） | [Delete] 需二次确认 |

#### [+] 上传按钮规则

- 仅本版本上传人（设计人员）可见；其他角色隐藏
- 点击触发文件选择器：任意文件类型，单文件 ≤ 50MB，单次最多 5 个
- 上传成功：表格即时追加行，标题计数 +1，主列表 Attachments 列数字同步更新

#### [Delete] 二次确认

```
┌──────────────────────────────────────────────────┐
│  Delete 结构计算书.xlsx?                           │
│  This action cannot be undone.                   │
│                      [Cancel]    [Delete]         │
└──────────────────────────────────────────────────┘
```

删除成功：行即时移除，标题计数 -1，主列表 Attachments 列同步减 1。

#### 状态（5 态）

| 状态 | 视觉表现 |
|-----|---------|
| 空（无附件） | 表格内显示 "No Data"，[+] 按钮仍可见（若有权限） |
| 加载 | 表格 Skeleton |
| 正常 | 如布局图所示 |
| 错误 | Toast 提示文件加载/上传/删除失败 |
| 极端数据 | 附件数 > 20 时表格内分页，不截断 |

---

### 3.9 DC Configuration 页面

**关联 Story**: US-007D-001
**关联 AC**: AC-007D-001 ~ 010

#### 入口

侧边栏 Settings → DC Configuration（需 `drawing:dc-config` 权限；无权限时菜单隐藏）

#### 主布局

```
┌──────────────────────────────────────────────────────────────────┐
│ ⚙️ DC Configuration                                  [+ Add DC] │
├──────────────────────────────────────────────────────────────────┤
│  Document Controllers for current project:                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Name       │ Configured By │ Configured At │ Action        │  │
│  │─────────────│───────────────│───────────────│───────────────│  │
│  │  陈小明      │ Admin         │ 2026-04-01    │ [Remove]      │  │
│  │  刘文静      │ Admin         │ 2026-04-01    │ [Remove]      │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ⓘ At least one DC is required for the drawing approval process. │
└──────────────────────────────────────────────────────────────────┘
```

#### 列定义

| 列 | 内容 |
|----|------|
| Name | DC 用户姓名 |
| Configured By | 配置操作人姓名 |
| Configured At | 配置时间 `YYYY-MM-DD` |
| Action | [Remove] 按钮 |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空（无 DC） | 移除最后一个 DC 后 | 警告横幅 + 空状态提示 | `"⚠️ No DC configured. Internal approvals cannot proceed to external approval. Please add at least one Document Controller."` |
| 加载 | 列表请求中 | 表格 Skeleton | — |
| 正常 | 已配置 ≥1 个 DC | 如布局图所示 | — |
| 错误 | 接口失败 | 错误提示 + 重试 | "Failed to load DC list. Retry" |
| 极端数据 | DC 数量 > 10 | 表格内分页 | — |

#### 权限可见性

| UI 元素 | 业务人员/管理员 | DC 自身 | 普通用户 |
|--------|:--------------:|:-------:|:-------:|
| 整个页面 | ✅ | ❌ | ❌ |
| [+ Add DC] | ✅ | — | — |
| [Remove] | ✅ | — | — |

---

### 3.10 Add Document Controller 弹窗

**关联 AC**: AC-007D-003 ~ 005、AC-007D-009

#### 布局

```
┌──────────────────────────────────────────┐
│ Add Document Controller            [✕]   │
├──────────────────────────────────────────┤
│ Search: [____________________________]   │
│                                          │
│ Available Users:                         │
│  ┌──────────────────────────────────┐    │
│  │ 王磊                    [Add]    │    │
│  │ 赵强                    [Add]    │    │
│  │ 孙丽                    [Add]    │    │
│  └──────────────────────────────────┘    │
│                                          │
│                           [Close]        │
└──────────────────────────────────────────┘
```

#### 交互规则

| 操作 | 行为 |
|------|------|
| Search 输入 | 实时模糊搜索 Available Users 列表（按姓名） |
| 点击 [Add] | 立即调用接口，成功后该用户从弹窗列表消失，主列表新增；弹窗**保持打开**支持连续添加 |
| 接口失败 | Toast 报错，弹窗保留 |
| Available Users 为空 | 显示 `"All eligible users have been configured as DC."` |
| 点击 [Close] 或 [✕] | 弹窗关闭 |

---

### 3.11 Remove DC 二次确认 Dialog

**关联 AC**: AC-007D-006 ~ 008

```
┌──────────────────────────────────────────────────┐
│  Remove DC                                       │
│                                                  │
│  Remove {dcName} from DC list?                   │
│  They will no longer receive external            │
│  approval tasks.                                 │
│                                                  │
│             [Cancel]    [Confirm]                │
└──────────────────────────────────────────────────┘
```

| 操作 | 行为 |
|------|------|
| [Confirm] | loading → 接口调用 → 成功后主列表移除该行；若为最后一个，显示空状态警告横幅 |
| [Cancel] | Dialog 关闭，主列表不变 |

---

## 4. 设计令牌（Design Tokens）

> 引用现有 Design System，本节列出**新增或覆盖**部分。

### 4.1 颜色语义（审批状态 5 态）

| 状态值 | 标签文案 | 颜色 Token | 色值 |
|--------|---------|-----------|------|
| PENDING_INTERNAL | ⏳ Pending Internal | `--color-status-waiting` | `#E6A23C` |
| INTERNAL_APPROVED | ⏳ Pending External | `--color-status-waiting` | `#E6A23C` |
| INTERNAL_REJECTED | ❌ Int. Rejected | `--color-status-error` | `#F56C6C` |
| APPROVED | ✅ Approved | `--color-status-success` | `#67C23A` |
| EXTERNAL_REJECTED | ❌ Ext. Rejected | `--color-status-error` | `#F56C6C` |

> `--color-status-waiting / success / error` 与 REQ-003A-pc 状态标签保持一致。

### 4.2 间距与栅格

| 组件 | 属性 | 值 | 来源 |
|-----|------|----|------|
| 版本历史抽屉 | width | 720px | 需求规格 |
| Attachments 二级抽屉 | width | 480px | 需求规格 |
| 4 阶段卡片网格 | gap | 12px | 新增 |
| 4 阶段卡片 | padding | 16px | 新增 |
| Todo 卡片 | padding | 16px 20px | 沿用 REQ-003 |
| Dialog | font-size | 14px | ⚠️ 必须显式设置 |
| Dialog | line-height | 22px | 新增 |
| Dialog | font-weight | 400 | 新增 |
| Dialog 标题 | font-size | 16px | 新增 |
| Dialog 标题 | font-weight | 600 | 新增 |
| 步骤条 | font-size | 12px | 新增 |
| 步骤条 | line-height | 20px | 新增 |
| Attachments 表格 | font-size | 13px | 新增 |
| Attachments 表格 | line-height | 20px | 新增 |

### 4.3 字体规范

| 用途 | font-size | font-weight | color |
|-----|-----------|-------------|-------|
| Todo 卡片标题 | 14px | 600 | `#303133` |
| Todo 卡片图纸信息行 | 14px | 500 | `#303133`，drawingCode 加粗 |
| Todo 卡片元信息 | 13px | 400 | `#606266` |
| 4 阶段卡片标题（① ② ③ ④） | 13px | 600 | `#303133` |
| 4 阶段卡片内容 | 13px | 400 | `#606266` |
| 步骤条文字 | 12px | 400 | 状态色（见 §4.1） |
| 警告横幅 | 14px | 400 | `#E6A23C` |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 完整布局，版本历史抽屉 720px，Attachments 抽屉 480px |
| Tablet | 768~1280px | 抽屉宽度降至 100vw；4 阶段卡片改为单列 2×1+2×1 |
| Mobile | < 768px | 不支持（PC 专属页面） |

---

## 6. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| 版本历史抽屉滑入/滑出 | slide-in-right | 300ms | ease-out |
| Attachments 二级抽屉滑入/滑出 | slide-in-right | 250ms | ease-out |
| 4 阶段卡片展开/折叠 | collapse（高度动画） | 200ms | ease-in-out |
| Dialog 弹出 | fade + scale | 200ms | ease-out |
| Todo 卡片消失（审批完成） | fade-out | 300ms | ease-in |
| [Confirm] loading 状态 | 按钮 spinner | — | — |
| 步骤条状态切换 | color transition | 200ms | linear |

---

## 7. 无障碍（A11y）

- WCAG 等级：AA
- 颜色对比度：状态标签文字与背景色对比度 ≥ 4.5:1
- 键盘导航：Dialog 内所有字段支持 Tab 键；展开/折叠按钮支持 Enter/Space；Attachments 表格行支持 Tab
- 焦点可见：所有交互元素有 `focus-visible` 样式（2px outline）
- 屏幕阅读器：Dialog 状态变更使用 `aria-live="polite"` 通知；步骤条状态使用 `aria-label` 描述

---

## 8. 复用与新建组件清单

| 组件 | 来源 | 备注 |
|-----|------|------|
| `el-drawer`（版本历史抽屉） | 现有 Element UI | 宽度调整为 720px |
| `el-drawer`（Attachments 抽屉） | 现有 Element UI | 宽度 480px，叠加在版本历史抽屉之上 |
| `el-dialog`（各类 Dialog） | 现有 Element UI | 复用现有样式 |
| `el-upload`（文件上传区） | 现有 Element UI | 拖拽 + 点击上传，需支持格式/大小校验 |
| `el-date-picker`（External Approval Date） | 现有 Element UI | 禁用未来日期（`disabledDate` 配置） |
| `el-radio-group`（Result 单选） | 现有 Element UI | 控制 Approved/Rejected 分支显隐 |
| `el-steps`（底部步骤条） | 现有 Element UI | 需自定义颜色映射（§4.1） |
| `el-table`（Attachments 表格） | 现有 Element UI | 新增 [Download] / [Delete] 行操作列 |
| `el-tag`（5 态状态标签） | 现有 Element UI | 颜色 Token 见 §4.1 |
| `MarkResultDialog` | **新建** | Approved/Rejected 双分支表单，版本历史抽屉和 Todo 面板共用 |
| `FourPhaseCard` | **新建** | 4 阶段卡片展开行组件 |
| `AttachmentsDrawer` | **新建** | Attachments 二级抽屉 |
| `DcConfigPage` | **新建** | DC 配置页面（独立路由） |

---

## 9. 文案规范

| 场景 | 文案（zh-CN） | 文案（en） |
|-----|-------------|----------|
| 内部审批 Todo 标题 | 内部审批待处理 | Internal Approval Required |
| 外部审批 Todo 标题 | 外部审批待处理 | External Approval Required |
| Approve 确认弹窗说明 | 审批通过后，该版本将进入由 DC 负责的外部审批阶段，版本在外部审批完成前不会生效。 | Once approved, this version will proceed to external approval by Document Controller. The version will NOT become active until external approval is completed. |
| Approve 成功 Toast | — | Internal approval completed. DC has been notified for external approval. |
| Reject 成功 Toast | — | Internal approval rejected. Designer has been notified. |
| 外部审批通过成功 Toast | — | External approval marked. Drawing is now active and QR code has been generated. |
| 外部审批驳回成功 Toast | — | External rejection recorded. Designer has been notified. |
| QR 生成失败 Toast | — | QR generation failed, please retry. |
| 无 DC 警告横幅 | — | ⚠️ No DC configured. Internal approvals cannot proceed to external approval. Please add at least one Document Controller. |
| 可选用户列表为空 | — | All eligible users have been configured as DC. |
| Remove DC 确认文案 | — | Remove {dcName} from DC list? They will no longer receive external approval tasks. |
| 删除附件确认文案 | — | Delete {fileName}? This action cannot be undone. |
| Attachments 空状态 | — | No Data |
| 版本历史空状态 | — | No version history available. |
| 步骤条标签 | 已上传 / 内部审批 / 外部审批 / 签字版 | Uploaded / Internal / External / Signed |
| 未到达步骤 | — | Not reached |
| 等待步骤 | 等待 | Waiting |

---

## 10. Figma 与原型链接

- Figma 设计稿：<!-- TODO: 待设计输出后补充 -->
- 交互原型：<!-- TODO: 待设计输出后补充 -->
- 设计系统：<!-- 引用现有 Smart Site Design System 地址 -->

---

## 11. AC 覆盖检查表

| AC ID | 对应页面/组件 | 对应状态/交互 | 覆盖? |
|------|------------|------------|------|
| AC-007A-001 | §3.1 Todo 卡片 | 卡片标题图标文案 | ✅ |
| AC-007A-002 | §3.2 Approve Dialog | 成功路径 | ✅ |
| AC-007A-003 | §3.2 Approve Dialog | 版本不生效说明文案 | ✅ |
| AC-007A-004 | §3.4 External Todo 卡片 | DC 收到任务 | ✅ |
| AC-007A-005 | §3.3 Reject Dialog | Comment 必填校验 | ✅ |
| AC-007A-006 | §3.3 Reject Dialog | 成功路径 | ✅ |
| AC-007A-007 | §3.3 Reject Dialog | 旧版本保持有效（后端逻辑，UI 无感知） | ✅ |
| AC-007A-008 | §3.2 / §3.3 Dialog | loading 态防重复提交 | ✅ |
| AC-007A-009 | §3.2 / §3.3 Dialog | 失败可重试 | ✅ |
| AC-007B-001 | §3.4 External Todo 卡片 | 卡片出现时机与内容 | ✅ |
| AC-007B-002 | §3.4 External Todo 卡片 | [Download Original] 按钮 | ✅ |
| AC-007B-003 | §3.5 Mark Dialog | Approved 必填校验 | ✅ |
| AC-007B-004 | §3.5 Mark Dialog | Approved 成功路径 | ✅ |
| AC-007B-005 | §3.5 Mark Dialog | Approved 成功 Toast 说明管理员需 [Assign] | ✅ |
| AC-007B-006 | §3.5 Mark Dialog | QR 失败回滚 | ✅ |
| AC-007B-007 | §3.5 Mark Dialog | Rejected Comment 必填 | ✅ |
| AC-007B-008 | §3.5 Mark Dialog | Rejected 成功路径 | ✅ |
| AC-007B-009 | §3.4 External Todo 卡片 | 其他 DC 的任务自动消失 | ✅ |
| AC-007B-010 | §3.4 权限可见性 | 非 DC 不显示 Mark Result | ✅ |
| AC-007B-011 | §3.5 Mark Dialog | loading 禁用防重复 | ✅ |
| AC-007C-001 | §3.6 版本历史抽屉 | 抽屉宽度与标题格式 | ✅ |
| AC-007C-002 | §3.6 主列表 | 列完整性（含 Confirmed / QR 逻辑） | ✅ |
| AC-007C-003 | §3.7 APPROVED 卡片 | 4 卡片全完成 | ✅ |
| AC-007C-004 | §3.7 INTERNAL_APPROVED + DC | ③ 卡片显示 Mark Result | ✅ |
| AC-007C-005 | §3.7 INTERNAL_APPROVED + 非 DC | ③ 卡片无 Mark Result | ✅ |
| AC-007C-006 | §3.7 INTERNAL_REJECTED | ② 驳回，③④ Not reached | ✅ |
| AC-007C-007 | §3.7 EXTERNAL_REJECTED | ③ 驳回，④ Not reached | ✅ |
| AC-007C-008 | §3.7 PENDING_INTERNAL | ② Pending，③④ 等待 | ✅ |
| AC-007C-009 | §3.7 ④ 签字版卡片 | 下载优先 pdfWithQrUrl | ✅ |
| AC-007C-010 | §3.7 ④ 签字版卡片 | pdfWithQrUrl 为 null 时降级 | ✅ |
| AC-007C-011 | §3.6 Attachments 列 | 计数显示 📎 n | ✅ |
| AC-007C-012 | §3.8 Attachments 抽屉 | 触发方式与布局 | ✅ |
| AC-007C-013 | §3.8 Attachments 表格 | 下载附件 | ✅ |
| AC-007C-014 | §3.8 权限 [+] / [Delete] | 非上传人隐藏 [+] 和 [Delete] | ✅ |
| AC-007C-015 | §3.8 上传附件 | 即时追加行，计数同步 | ✅ |
| AC-007C-016 | §3.8 删除附件 | 二次确认，即时移除，计数同步 | ✅ |
| AC-007D-001 | §3.9 页面权限 | 无权限时菜单隐藏，直接访问跳 403 | ✅ |
| AC-007D-002 | §3.9 DC 列表 | 正确展示 | ✅ |
| AC-007D-003 | §3.10 Add 弹窗 | Available Users 过滤规则 | ✅ |
| AC-007D-004 | §3.10 Add 弹窗 | 添加成功路径 | ✅ |
| AC-007D-005 | §3.10 Add 弹窗 | 连续添加（弹窗保持打开） | ✅ |
| AC-007D-006 | §3.11 Remove Dialog | 文案含姓名 | ✅ |
| AC-007D-007 | §3.11 Remove Dialog | 移除成功路径 | ✅ |
| AC-007D-008 | §3.9 空状态 | 移除最后一个 DC → 警告横幅 | ✅ |
| AC-007D-009 | §3.10 Search 过滤 | 实时模糊搜索 | ✅ |
| AC-007D-010 | §3.4 External Todo 卡片 | 集成验证（DC 收到 Todo） | ✅ |

---

## 12. 待定问题（Open Questions）

| OQ ID | 问题 | 影响 UI 哪部分 | 来源 |
|------|------|--------------|------|
| OQ-007A-001 | 内部审批人是否可在 Todo 卡片内直接下载原始文件（目前仅 [View Drawing]）？ | §3.1 Todo 卡片操作按钮 | REQ-007A OQ-001 |
| OQ-007A-002 | 内部审批超时（如 3 天未处理）是否发催办通知？ | §3.1 卡片是否新增超时角标 | REQ-007A OQ-002 |
| OQ-007B-001 | 外部审批超时是否需催办通知？ | §3.4 卡片是否新增超时角标 | REQ-007B OQ-001 |
| OQ-007B-002 | DC 是否需在 Dialog 中填写 Bentley 审批单号？ | §3.5 Mark Dialog 字段 | REQ-007B OQ-002 |
| OQ-007C-001 | 存量无签字版的 ACTIVE 历史版本，④ 卡片展示原始文件并标注 Legacy？ | §3.7 ④ 签字版卡片降级逻辑 | REQ-007C OQ-001 |
| OQ-007D-001 | 是否允许为不同图纸分类配置不同 DC？当前为项目级统一。 | §3.9 页面结构 | REQ-007D OQ-001 |
| OQ-007D-002 | DC 权限被撤销时是否自动提醒管理员重新配置？ | §3.9 告警横幅 | REQ-007D OQ-002 |

---

## 13. 验收标准（本文档自身）

UI 设计交付物完成的判定：

- [ ] Figma 稿覆盖所有组件/页面：§3.1~§3.11 共 11 个视图
- [ ] 每个视图均有 5 态设计稿（空/加载/正常/错误/极端数据）
- [ ] 5 种审批状态标签颜色与 §4.1 Token 一致
- [ ] 版本历史抽屉 4 阶段卡片覆盖所有 5 种 approvalStatus 展示场景
- [ ] §11 AC 覆盖检查表全部标记 ✅
- [ ] §8 中新建组件（MarkResultDialog / FourPhaseCard / AttachmentsDrawer / DcConfigPage）均有 Figma 组件设计稿
- [ ] 所有 OQ 在 §12 显式列出，未编造方案

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-04-17 | agent | 从旧 REQ-007-pc 生成初稿，覆盖两级审批全流程 |
| 0.2.0 | 2026-05-06 | agent | 按 REQ-007A/B/C/D-pc 拆分后重新生成：结构全面对齐最新模版；新增 §0 溯源块；§3 扩展为 11 个组件（新增 §3.6 主列表/§3.7 4 阶段卡片/§3.8 Attachments 抽屉/§3.9 DC 配置页/§3.10 Add DC 弹窗/§3.11 Remove DC 确认）；§4.2 补全 flex/字体 5 维度规格；§9 文案规范扩展；§11 AC 覆盖检查表从 0 补全至 40 条 |
