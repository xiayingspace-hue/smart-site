---
doc_type: ui_spec
req_id: REQ-007B-pc
version: 0.2.2
status: draft
generated_from: REQ-007B-pc@0.2.2
generated_at: 2026-05-25
owner: ""
---

# UI 设计说明：PC 端 — 外部审批 Todo 处理

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-007B-pc.md（主）、REQ-007-shared（业务规则）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc § 3.2
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007B-pc @ v0.2.2 |
| 覆盖用户故事 | US-007B-001、US-007B-002 |
| 覆盖 AC | AC-007B-001 ~ AC-007B-015 |
| 上次同步时间 | 2026-05-25 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

让 DC（Document Controller）在 Todo 面板中高效处理外部审批任务：通过 Detail Drawer 查看图纸提交信息并下载原始文件，在 Mark Result Dialog 中按照与纸质报审表一致的 Status（A–E）提交审批结果及所需附件，一次操作完成外部审批全流程。

### 1.2 设计原则

1. **角色专属视图**：🌐 External 卡片仅对 DC 可见，与 🔍 Internal 卡片图标和文案明确区分。
2. **渐进式操作路径**：卡片 → Detail Drawer → Mark Result Dialog，层层递进，每个层级职责清晰。
3. **与纸质表单对齐**：Mark Result Dialog 的 Status of Approval（A–E）与 Bentley 纸质报审表一致，便于核查留档。
4. **报审信息完整留档**：Submission Ref No.、Subject、Description 始终必填，不因审批结果分支而遗漏。
5. **长操作防护**：Confirm 提交期间（约 3–5s）全面禁用 Dialog 内操作，文案变 "Processing..."。

---

## 2. 信息架构

### 2.1 页面层级

```
Todo 面板（全局通知/铃铛）
└── External Approval Required 卡片列表（F-001）
    └── 图纸待办详情侧滑弹框 Detail Drawer（F-002）
        └── Mark External Approval Result Dialog（F-003）
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发角色 |
|---------|-------|---------|
| 全局铃铛通知 | Todo 面板 → External Approval Required 卡片 | DC |
| F-001 卡片 [Detail] 按钮 | F-002 Detail Drawer | DC |
| F-002 Drawer 底部 [✅ Mark Result] 按钮 | F-003 Mark Result Dialog | DC |
| 版本历史抽屉 ③ 卡片 [Mark Result]（见 UI-REQ-007C-pc §3.7）| F-003 Mark External Approval Result Dialog | DC |

---

## 3. 页面 / 组件清单

### 3.1 F-001：External Approval Required — Todo 卡片

**关联 Story**：US-007B-001
**关联 AC**：AC-007B-001、AC-007B-002、AC-007B-013、AC-007B-014

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
│                                                  [Detail]        │
└──────────────────────────────────────────────────────────────────┘
```

#### 字段说明

| 元素 | 内容 | 备注 |
|------|------|------|
| 图标 | 🌐 | 固定，区分外部（🌐）与内部（🔍） |
| 标题 | `External Approval Required` | 固定文案 |
| 图纸信息行 | `{drawingCode}  {drawingName}  {versionNo}` | `drawingCode` 加粗 |
| Uploaded by | `{designerName}（Designer）  \|  {uploadTime}` | — |
| Internal approved by | `{approverName}  \|  {internalApprovedTime}` | 区别于内部审批卡片 |
| Version Note | 版本说明 | 空时整行隐藏 |
| [Detail] | 打开 F-002 图纸待办详情侧滑弹框 | 次要样式按钮，右下角对齐 |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 当前 DC 名下无待外部审批任务 | 空状态插图 | "No pending tasks" |
| 加载 | 数据请求中 | Skeleton 骨架屏 | — |
| 正常 | 有外部审批任务 | 如布局图所示 | — |
| 错误 | 接口加载失败 | 错误图标 + 重试 | "Failed to load. Retry" |
| 极端数据 | 多个待审批任务 | 分页 / 滚动 | 顶部计数提示 |

#### 权限可见性

| UI 元素 | DC（已配置） | 内部审批人 | 其他 |
|--------|:-----------:|:---------:|:----:|
| 🌐 卡片（整体） | ✅ | ❌ | ❌ |
| [Detail] | ✅ | — | — |

---

### 3.2 F-002：图纸待办详情侧滑弹框（Detail Drawer）

**关联 Story**：US-007B-001
**关联 AC**：AC-007B-002、AC-007B-003、AC-007B-004、AC-007B-005

**触发方式**：点击 F-001 卡片右侧 [Detail] 按钮，从页面右侧滑入（Drawer 宽度建议 480px）。

#### 布局

```
┌────────────────────────────────────────────────────────────┐
│  Drawing Approval Detail                              ×    │
│  External Approval Required                               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Category            Architecture                         │
│  Description         包含外墙及核心筒轮廓线                 │
│                                                            │
│  Version             V3                                   │
│  Version Note        修正轴网尺寸，更新柱位标注              │
│                                                            │
│  Uploaded by         张三（Designer）                      │
│  Upload Time         2026-04-01 10:00                     │
│  Internal Approver   王总工                               │
│  Internal Approved   2026-04-02 14:30                     │
│                                                            │
│  Original File       📄 ARCH-001-V3-plan.pdf  ↗           │
│                      Click to open in browser.            │
│                      To save the file, use                │
│                      [Download Original File] below.      │
│                                                            │
├────────────────────────────────────────────────────────────┤
│  [Download Original File]          [✅ Mark Result]        │
└────────────────────────────────────────────────────────────┘
```

#### 字段说明

> **设计决策**：Drawing Code 和 Drawing Name 已在 Todo 卡片摘要行展示，详情弹框不重复展示，以减少冗余信息。

| 字段 | 数据来源 | 备注 |
|------|---------|------|
| Category | Drawing.category | 只读，继承自图纸主记录 |
| Description | Drawing.description | 无内容时显示 `—` |
| Version | DrawingVersion.versionNo | 例：V3 |
| Version Note | DrawingVersion.versionNote | 无内容时显示 `—` |
| Uploaded by | DrawingVersion.uploaderName | 显示姓名 + 角色标签，例：张三（Designer） |
| Upload Time | DrawingVersion.uploadTime | 格式：YYYY-MM-DD HH:mm；Tooltip 显示完整时间戳 |
| Internal Approver | DrawingVersion.approverName | 内部审批人姓名 |
| Internal Approved Time | DrawingApproval.approvedAt（phase=INTERNAL） | 内部审批通过时间 |
| Original File | DrawingVersion.fileUrl | 文件名 + 文件类型图标；文件名为可点击链接（inline 打开）；下方展示下载引导提示 |

#### 交互规则

- 弹框打开时展示加载态（Skeleton），加载完成后渲染字段
- 点击 `×` 或按 `Esc` 关闭弹框，审批状态不变
- 弹框打开期间，背景列表不可操作（半透明遮罩）
- 所有字段均为**只读**，不提供编辑入口
- **文件名链接**：以 `target="_blank"` 在浏览器新标签页中打开（`Content-Disposition: inline`）；文件名下方展示灰色辅助提示："Click to open in browser. To save the file, use [Download Original File] below."
- **[Download Original File] 按钮**：触发浏览器强制下载（`Content-Disposition: attachment`），文件命名格式：`{drawingCode}-V{versionNo}-original.{ext}`；下载期间按钮 loading 防重复；`fileUrl` 为空时按钮置灰，Tooltip 提示 "Original file unavailable."
- **[✅ Mark Result] 按钮**：点击打开 F-003 Dialog，Drawer 保留在背景

#### 权限可见性

| UI 元素 | DC（已配置） | 其他 |
|--------|:-----------:|:----:|
| Drawer 整体（含字段） | ✅ | ❌ |
| [Download Original File] | ✅ | — |
| [✅ Mark Result] | ✅ | — |

---

### 3.3 F-003：Mark External Approval Result Dialog

**关联 Story**：US-007B-002
**关联 AC**：AC-007B-006、AC-007B-007、AC-007B-007B、AC-007B-008、AC-007B-010、AC-007B-011、AC-007B-012、AC-007B-015

#### 布局

```
┌──────────────────────────────────────────────┐
│  Mark External Approval Result         [✕]   │
│  ARCH-001  首层平面图  V3                     │
├──────────────────────────────────────────────┤
│                                              │
│  Status of Approval *                        │
│  ┌────────────────────────────────────────┐  │
│  │  Please select ▾                       │  │
│  └────────────────────────────────────────┘  │
│  选项：                                      │
│  A – Approved / No Exception Taken           │
│  B – Approved with comment,                  │
│      resubmission required                   │
│  C – Revise And Resubmit                     │
│  D – For Record Purpose                      │
│  E – Others (please state reason)            │
│                                              │
│  ─── Status = E 时显示 ────────────────────  │
│  Others Reason *                             │
│  ┌────────────────────────────────────────┐  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│  最多 500 字符                               │
│                                              │
│  ──────────────────────────────────────────  │
│                                              │
│  Submission Ref No. *                        │
│  ┌────────────────────────────────────────┐  │
│  │  请填写报审编号（如 Bentley 审批单号）   │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  Submission Subject *                        │
│  ┌────────────────────────────────────────┐  │
│  │  请填写报审主题                         │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  Submission Description *                    │
│  ┌────────────────────────────────────────┐  │
│  │                                        │  │
│  │  请填写报审说明                         │  │
│  └────────────────────────────────────────┘  │
│  最多 1000 字符                              │
│                                              │
│  ─── Status A / B / D 时显示 ──────────────  │
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
│  │     PDF / PNG / JPG · Max 20MB         │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  External Approval Date *                    │
│  ┌────────────────────────────────────────┐  │
│  │  📅 YYYY-MM-DD                         │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ──────────────────────────────────────────  │
│                                              │
│  Remarks                                     │
│  ┌────────────────────────────────────────┐  │
│  │                                        │  │
│  └────────────────────────────────────────┘  │
│  最多 500 字符                               │
│                                              │
│           [Cancel]     [Confirm]             │
└──────────────────────────────────────────────┘
```

#### 字段规则

> **设计决策**：DC 直接选择与实际报审表一致的 Status of Approval（A–E），与纸质表单对齐；报审三要素（Ref No.、Subject、Description）**始终必填**，与审批结果无关。

| 字段 | 类型 | 必填条件 | 约束 |
|------|------|---------|------|
| Status of Approval | 下拉单选（A / B / C / D / E） | ✅ 始终 | 默认 "Please select"；未选中时 [Confirm] 禁用 |
| Others Reason | 文本域 | Status = E 时 ✅ | 最多 500 字符；仅 Status E 时显示 |
| Submission Ref No. | 文本输入 | ✅ 始终 | 最多 200 字符 |
| Submission Subject | 文本输入 | ✅ 始终 | 最多 200 字符 |
| Submission Description | 文本域 | ✅ 始终 | 最多 1000 字符 |
| Signed Drawing File | 文件上传 | Status A / B / D 时 ✅ | ≤ 50MB；PDF / DWG / DXF / PNG / JPG；Status A/B/D 时显示 |
| Approval Evidence | 文件上传 | Status A / B / D 时 ✅ | ≤ 20MB；PDF / PNG / JPG；Status A/B/D 时显示 |
| External Approval Date | 日期选择器 | Status A / B / D 时 ✅ | 不可选未来日期；Status A/B/D 时显示 |
| Remarks | 文本域 | ❌ 选填 | 最多 500 字符；始终显示 |

#### Status 分组与字段显示逻辑

| 分组 | Status | 额外显示字段 | 额外必填 |
|------|--------|------------|---------|
| 通过并上传文件 | A、B、D | Signed Drawing File、Approval Evidence、External Approval Date | 三者均必填 |
| 退回修改 | C | 无额外字段 | — |
| 其他 | E | Others Reason | Others Reason 必填 |

#### 交互规则

| 操作 | 状态变化 | Toast |
|------|---------|-------|
| 选择 Status A / B / D | 显示 Signed Drawing File、Approval Evidence、External Approval Date；隐藏 Others Reason | — |
| 选择 Status C | 隐藏上传区；隐藏 Others Reason | — |
| 选择 Status E | 显示 Others Reason；隐藏上传区 | — |
| 点击 [Confirm]（必填项未全部填写） | 前端校验阻止，空字段标红 + 提示必填 | — |
| 点击 [Confirm]（全部必填已填） | 按钮 loading，文案变 "Processing..."，禁用 Dialog 内所有操作（约 3–5 秒） | — |
| Status A / B / D — 操作成功 | Dialog 关闭，Todo 消失（含其他 DC 的同一任务） | `"External approval marked. Drawing is now active and QR code has been generated."` |
| Status C — 操作成功 | Dialog 关闭，Todo 消失，通知设计人员 | `"External rejection recorded. Designer has been notified."` |
| Status E — 操作成功 | Dialog 关闭，Todo 消失，通知设计人员（含 Others Reason 文本） | `"External rejection recorded. Designer has been notified."` |
| QR 生成失败（Status A / B / D） | loading 恢复，Dialog 保留 | `"QR generation failed, please retry"` |
| 其他接口失败 | loading 恢复，Dialog 保留 | 接口错误文案 |

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色语义

| 状态值 | 标签文案 | 颜色 Token | 色值 |
|--------|---------|-----------|------|
| INTERNAL_APPROVED | ⏳ Pending External | `--color-status-waiting` | `#E6A23C` |
| APPROVED | ✅ Approved | `--color-status-success` | `#67C23A` |
| EXTERNAL_REJECTED | ❌ Ext. Rejected | `--color-status-error` | `#F56C6C` |

### 4.2 间距与字体

| 组件 | 属性 | 值 |
|-----|------|----|
| Todo 卡片 | padding | 16px 20px |
| Detail Drawer | width | 480px |
| Detail Drawer 字段行 | row-gap | 12px |
| Dialog | font-size | 14px |
| Dialog | line-height | 22px |
| Dialog 标题 | font-size | 16px |
| Dialog 标题 | font-weight | 600 |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 完整布局 |
| Mobile | < 768px | 不支持（PC 专属） |

---

## 6. 无障碍（A11y）

- WCAG 等级：AA
- 键盘导航：Drawer / Dialog 内字段支持 Tab 键；下拉支持方向键切换选项；Esc 关闭 Drawer / Dialog
- 屏幕阅读器：`aria-live="polite"` 通知 Dialog 状态变更；Drawer 打开时焦点移入

---

## 7. 文案规范

| 场景 | 文案（en） |
|-----|----------|
| 外部审批 Todo 标题 | External Approval Required |
| 空状态文案 | No pending tasks |
| Drawer 标题 | Drawing Approval Detail |
| Drawer 副标题标签 | External Approval Required |
| 文件 inline 提示 | Click to open in browser. To save the file, use [Download Original File] below. |
| 文件不可用 Tooltip | Original file unavailable. |
| Status A/B/D 操作成功 Toast | External approval marked. Drawing is now active and QR code has been generated. |
| Status C/E 操作成功 Toast | External rejection recorded. Designer has been notified. |
| QR 生成失败 Toast | QR generation failed, please retry. |
| 操作失败通用 Toast | Operation failed, please retry. |

---

## 8. AC 覆盖检查表

| AC ID | 对应组件 | 描述 | 覆盖? |
|------|---------|------|------|
| AC-007B-001 | §3.1 卡片列表 | 仅展示当前 DC 自己名下待办 | ✅ |
| AC-007B-002 | §3.1 [Detail] 按钮 | 点击打开 Detail Drawer | ✅ |
| AC-007B-003 | §3.2 Drawer 字段展示 | 展示完整图纸提交信息，不含 Code/Name | ✅ |
| AC-007B-004 | §3.2 [Download Original File] | 触发强制下载，文件命名正确 | ✅ |
| AC-007B-005 | §3.2 权限校验 | 非项目 DC 无法下载（接口 403） | ✅ |
| AC-007B-006 | §3.3 报审三要素必填 | Ref No./Subject/Description 任一空时阻止提交 | ✅ |
| AC-007B-007 | §3.3 Status A/B/D 文件与日期必填 | 任一空时阻止提交 | ✅ |
| AC-007B-007B | §3.3 Status E Others Reason 必填 | 空时阻止提交 | ✅ |
| AC-007B-008 | §3.3 Status A/B/D 成功路径 | Toast + Todo 消失 + 版本生效 | ✅ |
| AC-007B-009 | §3.3 管理员分配后 SE 收到通知 | 版本生效后管理员 [Assign] 分配 SE | ✅ |
| AC-007B-010 | §3.3 QR 生成失败回滚 | loading 恢复，Toast 提示，可重试 | ✅ |
| AC-007B-011 | §3.3 Status C 退回必填校验 | 报审三要素空时阻止提交 | ✅ |
| AC-007B-012 | §3.3 Status C 成功路径 | Toast + Todo 消失 + 通知设计人员 | ✅ |
| AC-007B-013 | §3.1 / §3.2 其他 DC 任务自动关闭 | 任一 DC 完成操作后其他 DC 的 Todo 消失 | ✅ |
| AC-007B-014 | §3.1 / §3.2 权限可见性 | 非项目 DC 不显示卡片、Drawer、[Mark Result] | ✅ |
| AC-007B-015 | §3.3 loading 禁用防重复 | Confirm 后按钮 loading 禁用 | ✅ |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 UI-REQ-007-pc.md 拆分，覆盖 REQ-007B-pc 外部审批部分 |
| 0.1.1 | 2026-05-24 | agent | 按来源需求独立成文件 |
| 0.2.2 | 2026-05-25 | agent | 同步 REQ-007B-pc@0.2.2：① F-001 卡片移除 [Download Original] 和 [Mark Result] 按钮，仅保留 [Detail]；② 新增 §3.2 F-002 Detail Drawer 组件（字段展示 + Download + Mark Result）；③ §3.3 F-003 Dialog 重构：Approved/Rejected Radio 改为 Status of Approval A–E 下拉，新增 Others Reason（Status E 必填），报审三要素（Ref No./Subject/Description）改为始终必填，Signed Drawing File/Approval Evidence/External Approval Date 调整为 Status A/B/D 必填，移除 Rejection Reason 字段；④ 更新设计原则、信息架构、文案规范；⑤ AC 覆盖扩展至 AC-007B-001 ~ 015 |
