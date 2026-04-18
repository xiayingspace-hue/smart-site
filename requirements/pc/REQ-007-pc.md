# PC 管理端 — 图纸两级审批流程

> **端**: PC 管理端（桌面浏览器 Web 管理后台）
> **共享需求**: [REQ-007-shared.md](../shared/REQ-007-shared.md)（角色定义、完整审批流程、数据模型、API 接口）
> **依赖需求**: [REQ-003-pc.md](./REQ-003-pc.md)（图纸管理基础页面）
> **本文档仅包含**: PC 端两级审批相关的新增/变更交互，包括上传变更、内部审批 Todo 调整、DC 外部审批 Todo、版本历史 4 阶段展示、DC 配置页面

## 基本信息

- **需求ID**: REQ-007-pc
- **需求标题**: PC 端图纸两级审批流程
- **产品**: SMART SITE SYSTEM
- **平台**: PC 管理端（桌面浏览器，Vue 2 + Element UI）
- **优先级**: 高
- **状态**: 草稿
- **依赖**: REQ-003-pc、REQ-007-shared

---

## 需求描述

### 背景

REQ-003-pc 定义了单级审批的 PC 端交互。本需求将审批流程升级为**内部审批 + 外部审批**两级串行审批，涉及以下 PC 端变更：

1. **上传弹窗**：上传人角色从 "Drawing 团队" 明确为**设计人员**；审批人标签改为"内部审批人"
2. **内部审批 Todo**：与现有审批 Todo 基本一致，仅状态标签变更
3. **DC 外部审批 Todo**：全新交互，DC 从 Todo 下载原始文件、标记外部审批结果、上传签字版
4. **版本历史抽屉**：从简单列表升级为**可展开的 4 阶段生命周期视图**
5. **DC 配置页面**：新增项目级 DC 人员配置页
6. **图纸列表状态标签**：扩展为 5 种状态

### 用户故事

```
作为 设计人员
我想要 在上传图纸时指定内部审批人
以便 图纸先经过内部技术审核，再进入外部审批流程
```

```
作为 Document Controller (DC)
我想要 在 Todo 列表中看到等待外部审批的图纸，并直接下载原始文件
以便 快速将图纸提交到 Bentley，无需在系统中来回查找
```

```
作为 Document Controller (DC)
我想要 外部审批完成后，在一个弹窗中一次性上传签字版图纸、审批凭证并标记结果
以便 一次操作完成所有工作，版本立即生效并推送给 Site Engineer
```

```
作为 项目管理人员
我想要 在版本历史中看到每个版本的完整审批生命周期（上传 → 内部审批 → 外部审批 → 签字版）
以便 清晰了解图纸在各阶段的流转状态和责任人
```

```
作为 业务人员
我想要 在配置页面设置项目的 DC 人员
以便 内部审批通过后系统自动通知 DC，无需每次手动指定
```

---

## 功能需求

### 1. 上传弹窗变更（扩展 REQ-003-pc §3）

#### 1.1 变更点

在 REQ-003-pc §3.2 / §3.3 的上传弹窗基础上做以下调整：

| 项 | REQ-003（原） | REQ-007（新） |
|----|-------------|-------------|
| 操作角色 | Drawing 团队 | 设计人员 (Designer) |
| 审批人标签 | `Approver *` | `Internal Approver *`（内部审批人） |
| 审批人下拉范围 | 所有有 `drawing:approve` 权限的用户 | 不变 |
| 上传阻断条件 | 有版本 `PENDING` 时不可上传 | 有版本 `PENDING_INTERNAL` 或 `INTERNAL_APPROVED` 时不可上传 |

#### 1.2 上传弹窗（新建图纸）

```
┌──────────────────────────────────────┐
│ Upload New Drawing           [✕]     │
│                                      │
│ Drawing Code *  [________________]   │
│ Drawing Name *  [________________]   │
│ Category *      [________▼]          │
│ Description     [________________]   │
│                                      │
│ Drawing File *                       │
│ ┌──────────────────────────────────┐ │
│ │   📂  Drag file here or          │ │
│ │       Click to upload            │ │
│ │   PDF / DWG / DXF / PNG / JPG    │ │
│ │   Max 50MB                       │ │
│ └──────────────────────────────────┘ │
│                                      │
│ Version Note       [________________]│
│ Internal Approver * [________▼]      │
│                                      │
│          [Cancel]  [Submit]          │
└──────────────────────────────────────┘
```

#### 1.3 上传成功提示变更

原文案：`"Drawing uploaded successfully. Pending approval."`
新文案：`"Drawing uploaded successfully. Pending internal approval."`

---

### 2. 内部审批 Todo（扩展 REQ-003-pc §4）

#### 2.1 变更点

与 REQ-003-pc §4 的审批 Todo 基本一致，做以下调整：

| 项 | REQ-003（原） | REQ-007（新） |
|----|-------------|-------------|
| Todo 标题 | Drawing Approval Required | **Internal Approval Required** |
| Approve 确认文案 | "Once approved, this version will become the current active version..." | "Once approved, this version will proceed to external approval by DC." |
| 审批通过后行为 | 版本生效 + 推送 SE | 版本状态 → `INTERNAL_APPROVED`，通知 DC，**不推送 SE** |

#### 2.2 内部审批 Todo 卡片

```
┌─────────────────────────────────────────────────────┐
│ 🔍 Internal Approval Required                       │
│                                                     │
│ ARCH-001  首层平面图  V3                             │
│ Uploaded by: 张三（Designer）  |  2026-04-01 10:00  │
│ Version Note: 修正轴网尺寸                           │
│                                                     │
│ [View Drawing]   [Approve]   [Reject]               │
└─────────────────────────────────────────────────────┘
```

#### 2.3 Approve 确认对话框

```
┌──────────────────────────────────────────────┐
│  Confirm Internal Approval?                  │
│                                              │
│  Once approved, this version will proceed    │
│  to external approval by Document Controller.│
│                                              │
│  [Cancel]  [Confirm]                         │
└──────────────────────────────────────────────┘
```

#### 2.4 通过后 Toast

`"Internal approval completed. DC has been notified for external approval."`

---

### 3. DC 外部审批 Todo（新增）

#### 3.1 触发条件

内部审批通过后，系统自动在项目所有已配置 DC 的 Todo 列表中创建外部审批任务。

#### 3.2 外部审批 Todo 卡片

```
┌──────────────────────────────────────────────────────────────────┐
│ 🌐 External Approval Required                                   │
│                                                                  │
│ ARCH-001  首层平面图  V3                                         │
│ Uploaded by: 张三（Designer）  |  2026-04-01 10:00               │
│ Internal approved by: 王总工  |  2026-04-02 14:30               │
│ Version Note: 修正轴网尺寸                                       │
│                                                                  │
│ [📄 Download Original]   [✅ Mark Result]                        │
└──────────────────────────────────────────────────────────────────┘
```

| 元素 | 说明 |
|------|------|
| 图标 | 🌐（区别于内部审批的 🔍） |
| 标题 | `External Approval Required` |
| Uploaded by | 设计人员姓名 + 上传时间 |
| Internal approved by | 内部审批人姓名 + 审批时间 |
| Version Note | 版本修改说明 |
| `[📄 Download Original]` | 下载原始图纸文件（`fileUrl`），文件名为原始 `fileName`。DC 用此文件提交到 Bentley |
| `[✅ Mark Result]` | 打开外部审批标记 Dialog（§3.3） |

#### 3.3 外部审批标记 Dialog（Mark External Approval）

DC 点击 `[✅ Mark Result]` 后弹出 Dialog：

```
┌──────────────────────────────────────────────┐
│  Mark External Approval Result         [✕]   │
│  ARCH-001  首层平面图  V3                     │
├──────────────────────────────────────────────┤
│                                              │
│  Result *                                    │
│  ┌────────────────────────────────────────┐  │
│  │  ○ Approved                            │  │
│  │  ○ Rejected                            │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ─── 选择 Approved 后显示以下字段 ───         │
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
│  │     PDF / PNG / JPG                    │  │
│  │     Max 20MB                           │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  External Approval Date *                    │
│  ┌────────────────────────────────────────┐  │
│  │  📅 YYYY-MM-DD                         │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  Remarks                                     │
│  ┌────────────────────────────────────────┐  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ─── 选择 Rejected 后显示以下字段 ───         │
│                                              │
│  Rejection Reason *                          │
│  ┌────────────────────────────────────────┐  │
│  │                                        │  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│                                              │
│           [Cancel]     [Confirm]             │
└──────────────────────────────────────────────┘
```

#### 3.4 字段说明

| 字段 | 类型 | 必填条件 | 说明 |
|------|------|---------|------|
| Result | Radio（Approved / Rejected） | ✅ | 默认不选中，必须选择后才能点击 [Confirm] |
| Signed Drawing File | 文件上传 | Approved 时必填 | 签字版图纸，≤ 50MB，格式 PDF/DWG/DXF/PNG/JPG |
| Approval Evidence | 文件上传 | Approved 时必填 | 审批凭证（Bentley 截图/PDF），≤ 20MB，格式 PDF/PNG/JPG |
| External Approval Date | 日期选择器 | Approved 时必填 | 不可选未来日期 |
| Remarks | 文本域 | ❌ | 最多 500 字符 |
| Rejection Reason | 文本域 | Rejected 时必填 | 最多 500 字符 |

#### 3.5 Approved 交互流程

1. DC 选择 `Approved`，填写必填字段，点击 `[Confirm]`
2. 按钮进入 **loading 态**（文案变为 "Processing..."），禁用 Dialog 内所有操作
3. 后端同步执行：上传文件 → 版本生效 → QR 生成 → 推送 SE（约 3-5 秒）
4. **成功**：
   - Dialog 关闭
   - Toast 提示 `"External approval marked. Drawing is now active and QR code has been generated."`
   - Todo 任务消失（包括其他 DC 的同一任务）
5. **失败**：
   - loading 状态恢复
   - Toast 错误提示（显示后端返回的错误文案，如 "QR generation failed, please retry"）
   - DC 可修改内容后重试或关闭 Dialog

#### 3.6 Rejected 交互流程

1. DC 选择 `Rejected`，填写 Rejection Reason，点击 `[Confirm]`
2. 后端执行：版本状态 → `EXTERNAL_REJECTED`，通知设计人员
3. **成功**：
   - Dialog 关闭
   - Toast 提示 `"External rejection recorded. Designer has been notified."`
   - Todo 任务消失
4. **失败**：Toast 错误提示，DC 可重试

---

### 4. 版本历史抽屉升级（替代 REQ-003-pc §6）

#### 4.1 主列表

点击图纸行 `[History]` 按钮，从右侧滑入抽屉面板（宽度扩展至 **720px**，以容纳 4 阶段卡片）。

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Version History — ARCH-001 首层平面图                              [✕]  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Version  │ Status              │ Designer  │ Confirmed │ QR             │
│───────────│─────────────────────│───────────│───────────│────────────────│
│  ▶ V3     │ ✅ Approved         │ 张三      │ 5/12      │ [View]         │
│  ▶ V2     │ ⏳ Pending External │ 李四      │ —         │ —              │
│  ▶ V1     │ ❌ Int. Rejected    │ 张三      │ —         │ —              │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 4.2 主列表列定义

| 列 | 说明 |
|----|------|
| Version | 版本号（V1、V2、V3），左侧有 ▶ 展开箭头 |
| Status | 状态标签，颜色编码见 §4.3 |
| Designer | 设计人员（上传人）姓名 |
| Confirmed | 已确认/总人数；仅 APPROVED 状态显示，其他显示 `—` |
| QR | APPROVED 且 QR 已生成时显示 `[View]`，其他显示 `—`（复用 REQ-006 逻辑） |

#### 4.3 状态标签颜色

| 状态 | 显示文本 | 颜色 |
|------|---------|------|
| `PENDING_INTERNAL` | ⏳ Pending Internal | 橙色 `#E6A23C` |
| `INTERNAL_APPROVED` | ⏳ Pending External | 橙色 `#E6A23C` |
| `INTERNAL_REJECTED` | ❌ Int. Rejected | 红色 `#F56C6C` |
| `APPROVED` | ✅ Approved | 绿色 `#67C23A` |
| `EXTERNAL_REJECTED` | ❌ Ext. Rejected | 红色 `#F56C6C` |

#### 4.4 展开行 — 4 阶段生命周期视图

点击 ▶ 展开某版本，显示该版本的完整生命周期：

**APPROVED 版本展开（完整 4 阶段）**：

```
▼ V3  ✅ Approved                                                [Collapse]
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  ① Uploaded                          ② Internal Approval                 │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ 📄 arch001-v3.pdf         │       │ ✅ Approved                │       │
│  │ Designer: 张三            │       │ Approver: 王总工           │       │
│  │ 2026-04-01 10:00          │       │ 2026-04-02 14:30           │       │
│  │ Size: 2.5 MB              │       │ Comment: "尺寸已修正"      │       │
│  │            [Preview]      │       │                            │       │
│  │            [Download]     │       │                            │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ③ External Approval                 ④ Signed Version                    │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ ✅ Approved                │       │ 📄 arch001-v3-signed.pdf  │       │
│  │ Marked by: DC 陈小明       │       │ ✍️ Signed Drawing         │       │
│  │ Approval Date: 2026-04-05 │       │ Uploaded: 2026-04-05      │       │
│  │ 📎 Evidence: bentley.pdf  │       │ Size: 3.1 MB              │       │
│  │           [Download]      │       │            [Preview]      │       │
│  │ Remark: 业主已签批         │       │            [Download]     │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│  📄 Uploaded ✓  →  🔍 Internal ✓  →  🌐 External ✓  →  ✍️ Signed ✓    │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**PENDING_EXTERNAL 版本展开（等待外部审批）**：

```
▼ V2  ⏳ Pending External                                        [Collapse]
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  ① Uploaded                          ② Internal Approval                 │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ 📄 arch001-v2.pdf         │       │ ✅ Approved                │       │
│  │ Designer: 李四            │       │ Approver: 王总工           │       │
│  │ 2026-03-15 09:00          │       │ 2026-03-16 11:00           │       │
│  │ Size: 2.1 MB              │       │                            │       │
│  │            [Preview]      │       │                            │       │
│  │            [Download]     │       │                            │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ③ External Approval                 ④ Signed Version                    │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ ⏳ Pending                 │       │                           │       │
│  │ Waiting for DC to submit  │       │  Waiting for external     │       │
│  │ external approval         │       │  approval                 │       │
│  │                            │       │                           │       │
│  │  [✅ Mark Result]          │       │                           │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│  📄 Uploaded ✓  →  🔍 Internal ✓  →  🌐 External ⏳  →  ✍️ Signed ...  │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

> ③ 卡片中的 `[✅ Mark Result]` 按钮仅对拥有 `drawing:external-approval` 权限且为项目已配置 DC 的用户可见。点击后打开与 §3.3 相同的 Dialog。

**INTERNAL_REJECTED 版本展开**：

```
▼ V1  ❌ Int. Rejected                                           [Collapse]
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  ① Uploaded                          ② Internal Approval                 │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ 📄 arch001-v1.pdf         │       │ ❌ Rejected                │       │
│  │ Designer: 张三            │       │ Approver: 王总工           │       │
│  │ 2026-03-01 14:00          │       │ 2026-03-02 09:00           │       │
│  │ Size: 1.8 MB              │       │ Comment: "轴网标注不完整"  │       │
│  │            [Preview]      │       │                            │       │
│  │            [Download]     │       │                            │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ③ External Approval                 ④ Signed Version                    │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ — Not reached              │       │ — Not reached              │       │
│  │ (Internal approval failed) │       │                            │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│  📄 Uploaded ✓  →  🔍 Internal ✕  →  🌐 External —  →  ✍️ Signed —    │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**EXTERNAL_REJECTED 版本展开**：

```
▼ V2  ❌ Ext. Rejected                                           [Collapse]
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  ① Uploaded                          ② Internal Approval                 │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ 📄 arch001-v2.pdf         │       │ ✅ Approved                │       │
│  │ Designer: 张三            │       │ Approver: 王总工           │       │
│  │ 2026-03-15 09:00          │       │ 2026-03-16 11:00           │       │
│  │            [Preview]      │       │                            │       │
│  │            [Download]     │       │                            │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ③ External Approval                 ④ Signed Version                    │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ ❌ Rejected                │       │ — Not reached              │       │
│  │ Marked by: DC 陈小明       │       │ (External approval failed) │       │
│  │ 2026-03-20 15:00           │       │                            │       │
│  │ Reason: "业主要求修改立面"  │       │                            │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│  📄 Uploaded ✓  →  🔍 Internal ✓  →  🌐 External ✕  →  ✍️ Signed —    │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**PENDING_INTERNAL 版本展开**：

```
▼ V3  ⏳ Pending Internal                                        [Collapse]
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  ① Uploaded                          ② Internal Approval                 │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ 📄 arch001-v3.pdf         │       │ ⏳ Pending                 │       │
│  │ Designer: 张三            │       │ Approver: 王总工           │       │
│  │ 2026-04-01 10:00          │       │ Assigned: 2026-04-01       │       │
│  │ Size: 2.5 MB              │       │                            │       │
│  │            [Preview]      │       │                            │       │
│  │            [Download]     │       │                            │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ③ External Approval                 ④ Signed Version                    │
│  ┌───────────────────────────┐       ┌───────────────────────────┐       │
│  │ — Waiting for internal     │       │ — Waiting for internal     │       │
│  │   approval                 │       │   approval                 │       │
│  └───────────────────────────┘       └───────────────────────────┘       │
│                                                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│  📄 Uploaded ✓  →  🔍 Internal ⏳  →  🌐 External ...  →  ✍️ Signed ...│
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 4.5 步骤条规范

每个展开行底部显示 4 步步骤条，指示当前进度：

| 步骤 | 图标 | 完成 | 进行中 | 失败 | 未到达 |
|------|------|------|--------|------|--------|
| ① Uploaded | 📄 | ✓ 绿色 | — | — | — |
| ② Internal | 🔍 | ✓ 绿色 | ⏳ 橙色 | ✕ 红色 | — 灰色 |
| ③ External | 🌐 | ✓ 绿色 | ⏳ 橙色 | ✕ 红色 | — 灰色 |
| ④ Signed | ✍️ | ✓ 绿色 | — | — | — 灰色 |

步骤之间用 → 箭头连接。步骤始终上传完成（①恒为 ✓）。

#### 4.6 卡片内操作按钮

| 卡片 | 按钮 | 显示条件 | 说明 |
|------|------|---------|------|
| ① Uploaded | [Preview] | 始终 | 在线预览原始文件 |
| ① Uploaded | [Download] | 始终 | 下载原始文件 |
| ③ External | [✅ Mark Result] | `PENDING_EXTERNAL` 状态 + DC 权限 | 打开外部审批标记 Dialog |
| ③ External | [Download]（Evidence） | `APPROVED` / `EXTERNAL_REJECTED` 时 | 下载审批凭证 |
| ④ Signed | [Preview] | `APPROVED` 时 | 在线预览签字版 |
| ④ Signed | [Download] | `APPROVED` 时 | 下载签字版 |

---

### 5. 图纸列表状态标签扩展（变更 REQ-003-pc §2.3）

#### 5.1 Status 列颜色

| 状态 | 标签文案 | 颜色 | 图标 |
|------|---------|------|------|
| `PENDING_INTERNAL` | Pending Internal | 橙色 `#E6A23C` | 🟡 |
| `PENDING_EXTERNAL` | Pending External | 橙色 `#E6A23C` | 🟡 |
| `ACTIVE` | Active | 绿色 `#67C23A` | 🟢 |
| `INTERNAL_REJECTED` | Int. Rejected | 红色 `#F56C6C` | 🔴 |
| `EXTERNAL_REJECTED` | Ext. Rejected | 红色 `#F56C6C` | 🔴 |

#### 5.2 Status 筛选下拉扩展

原有 3 个选项扩展为 6 个：

| 选项 | 说明 |
|------|------|
| All | 全部 |
| Active | 已通过 |
| Pending Internal | 待内部审批 |
| Pending External | 待外部审批 |
| Int. Rejected | 内部驳回 |
| Ext. Rejected | 外部驳回 |

#### 5.3 Upload New Version 按钮条件变更

| REQ-003（原） | REQ-007（新） |
|-------------|-------------|
| Status ≠ PENDING_APPROVAL 时可用 | Status 不为 `PENDING_INTERNAL` 且不为 `PENDING_EXTERNAL` 时可用 |

---

### 6. DC 配置页面（新增）

#### 6.1 入口

- **侧边栏菜单**: Settings → DC Configuration（或 Project Settings 下的子页面）
- 权限：`drawing:dc-config`

#### 6.2 页面布局

```
┌──────────────────────────────────────────────────────────────────┐
│ ⚙️ DC Configuration                                  [+ Add DC] │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Document Controllers for current project:                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Name           │ Configured By  │ Configured At │ Action  │  │
│  │─────────────────│────────────────│───────────────│─────────│  │
│  │  陈小明          │ Admin          │ 2026-04-01    │ [Remove]│  │
│  │  刘文静          │ Admin          │ 2026-04-01    │ [Remove]│  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ⓘ At least one DC is required for the drawing approval process. │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

#### 6.3 添加 DC

点击 `[+ Add DC]` 弹出选择弹窗：

```
┌──────────────────────────────────────┐
│ Add Document Controller        [✕]   │
├──────────────────────────────────────┤
│ Search: [__________________________] │
│                                      │
│ Available Users:                     │
│  ○ 王磊                      [Add]  │
│  ○ 赵强                      [Add]  │
│  ○ 孙丽                      [Add]  │
│                                      │
│                         [Close]      │
└──────────────────────────────────────┘
```

- 列表仅显示拥有 `drawing:external-approval` 权限且尚未配置为 DC 的用户
- 点击 `[Add]` 立即调用接口添加，成功后列表刷新
- 支持搜索过滤

#### 6.4 移除 DC

- 点击 `[Remove]` 弹出二次确认：`"Remove {name} from DC list? They will no longer receive external approval tasks."`
- 确认后调用接口移除
- 若移除后 DC 列表为空，显示警告：`"⚠️ No DC configured. Internal approvals cannot proceed to external approval."`
- **不阻止移除**（允许暂时为空），但在内部审批通过时系统会提示"无可用 DC"

---

### 7. 站内消息通知扩展（变更 REQ-003-pc §8）

| 触发事件 | 接收人 | 通知标题 | 通知内容 |
|---------|--------|---------|---------|
| 内部审批驳回 | 设计人员 | Drawing Internally Rejected | Your drawing {drawingCode} {drawingName} V{n} was rejected during internal review. Comment: {comment} |
| 外部审批驳回 | 设计人员 | Drawing Externally Rejected | Your drawing {drawingCode} {drawingName} V{n} was rejected during external review. Reason: {comment} |
| 内部审批通过 | 项目所有 DC | External Approval Required | Drawing {drawingCode} {drawingName} V{n} has passed internal review. Please submit for external approval. |

---

## 验收标准

### 上传弹窗
- [ ] 审批人标签显示为 "Internal Approver"
- [ ] 图纸有版本处于 `PENDING_INTERNAL` 或 `PENDING_EXTERNAL` 时，Upload New Version 按钮置灰
- [ ] 上传成功 Toast 显示 "Pending internal approval"

### 内部审批 Todo
- [ ] Todo 标题显示 "Internal Approval Required"（带 🔍 图标）
- [ ] Approve 确认文案提示"将进入外部审批"而非"版本将生效"
- [ ] 内部审批通过后**不推送 SE**，版本状态变为 `INTERNAL_APPROVED`
- [ ] 通过后 Toast 提示 DC 已收到通知

### DC 外部审批 Todo
- [ ] 内部审批通过后，所有已配置 DC 的 Todo 列表出现外部审批任务
- [ ] Todo 卡片显示 🌐 图标和 "External Approval Required" 标题
- [ ] `[📄 Download Original]` 可下载原始图纸文件
- [ ] 点击 `[✅ Mark Result]` 打开标记 Dialog
- [ ] Approved 时签字版文件、审批凭证、外部审批日期为必填
- [ ] Rejected 时 Rejection Reason 为必填
- [ ] Approved 确认后 loading 等待，成功后 Toast 提示版本生效 + QR 已生成
- [ ] Approved 失败时（如 QR 生成失败）loading 恢复，DC 可重试
- [ ] 一个 DC 标记完成后，其他 DC 的同一 Todo 任务自动消失

### 版本历史抽屉
- [ ] 抽屉宽度 720px
- [ ] 主列表显示 Version、Status、Designer、Confirmed、QR 列
- [ ] 状态标签颜色正确（5 种状态）
- [ ] 点击 ▶ 展开显示 4 阶段卡片 + 底部步骤条
- [ ] APPROVED 版本展开：4 个卡片均为完成状态，原始文件和签字版可分别 Preview/Download
- [ ] PENDING_EXTERNAL 版本展开：③ 卡片显示 `[✅ Mark Result]`（DC 权限用户可见）
- [ ] INTERNAL_REJECTED 版本展开：③④ 卡片显示 "Not reached"
- [ ] EXTERNAL_REJECTED 版本展开：③ 卡片显示驳回原因，④ 卡片显示 "Not reached"
- [ ] PENDING_INTERNAL 版本展开：②③④ 卡片显示等待状态
- [ ] 步骤条各步状态图标/颜色与对应审批阶段一致

### 图纸列表
- [ ] Status 列正确显示 5 种状态标签和颜色
- [ ] Status 筛选下拉包含 6 个选项（含 All）
- [ ] `PENDING_INTERNAL` 和 `PENDING_EXTERNAL` 状态时 Upload New Version 按钮置灰

### DC 配置页
- [ ] 页面正确展示当前项目已配置的 DC 列表
- [ ] `[+ Add DC]` 弹窗仅显示有 `drawing:external-approval` 权限且未配置的用户
- [ ] 添加/移除 DC 后列表实时刷新
- [ ] 移除最后一个 DC 时显示警告提示（但不阻止）

### 站内通知
- [ ] 内部驳回后设计人员收到 "Drawing Internally Rejected" 通知
- [ ] 外部驳回后设计人员收到 "Drawing Externally Rejected" 通知
- [ ] 内部通过后所有 DC 收到 "External Approval Required" 通知

---

## 相关需求

| 文档 | 说明 |
|------|------|
| [REQ-007-shared.md](../shared/REQ-007-shared.md) | 跨端共享：角色定义、完整审批流程、数据模型、API |
| [REQ-003-pc.md](./REQ-003-pc.md) | 图纸管理基础页面（被本需求扩展） |
| [REQ-003-shared.md](../shared/REQ-003-shared.md) | 图纸管理核心数据模型（被本需求扩展） |
| [REQ-006-shared.md](../shared/REQ-006-shared.md) | QR 生成机制（时机调整为外部审批通过时同步生成） |
