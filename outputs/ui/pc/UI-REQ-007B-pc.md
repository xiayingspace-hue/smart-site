---
doc_type: ui_spec
req_id: REQ-007B-pc
version: 0.1.1
status: draft
generated_from: REQ-007B-pc@0.1.1
generated_at: 2026-05-24
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
| 来源需求 | REQ-007B-pc @ v0.1.1 |
| 覆盖用户故事 | US-007B-001、US-007B-002 |
| 覆盖 AC | AC-007B-001 ~ AC-007B-011 |
| 上次同步时间 | 2026-05-24 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

让 DC（Document Controller）在 Todo 面板中高效处理外部审批任务：下载原始文件进行离线审阅，并通过 Mark Result Dialog 提交 Approved / Rejected 结果及所需附件。

### 1.2 设计原则

1. **角色专属视图**：🌐 External 卡片仅对 DC 可见，与 🔍 Internal 卡片图标和文案明确区分。
2. **一卡一动作**：卡片内直接提供下载和标记结果入口，无需跳转。
3. **双分支表单**：Mark Result Dialog 根据 Approved / Rejected 动态切换必填字段，避免表单冗余。
4. **长操作防护**：Confirm 提交期间（约 3–5s）全面禁用 Dialog 内操作，文案变 "Processing..."。

---

## 2. 信息架构

### 2.1 页面层级

```
Todo 面板（全局通知/铃铛）
└── External Approval Required 卡片列表
    └── Mark External Approval Result Dialog
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发角色 |
|---------|-------|---------|
| 全局铃铛通知 | Todo 面板 → External Approval Required 卡片 | DC |
| 版本历史抽屉 ③ 卡片 [Mark Result]（见 UI-REQ-007C-pc §3.7）| Mark External Approval Result Dialog | DC |

---

## 3. 页面 / 组件清单

### 3.1 External Approval Required — Todo 卡片

**关联 Story**：US-007B-001
**关联 AC**：AC-007B-001、AC-007B-002、AC-007B-009、AC-007B-010

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
| 图纸信息行 | `{drawingCode}  {drawingName}  {versionNo}` | `code` 加粗 |
| Uploaded by | `{designerName}  \|  {uploadTime}` | — |
| Internal approved by | `{approverName}  \|  {internalApprovedTime}` | 区别于内部审批卡片 |
| Version Note | 版本说明 | 空时整行隐藏 |
| [📄 Download Original] | 下载 `fileUrl`（原始图纸），文件名保持原始 `fileName` | 次要按钮 |
| [✅ Mark Result] | 打开 Mark External Approval Result Dialog（§3.2） | 主要按钮，绿色 |

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

### 3.2 Mark External Approval Result Dialog

**关联 Story**：US-007B-002
**关联 AC**：AC-007B-003 ~ 011

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
| 点击 [Confirm]（Approved，全部必填已填） | 按钮 loading，文案变 "Processing..."，禁用所有操作（约 3–5 秒） | — |
| 操作成功（Approved） | Dialog 关闭，Todo 消失（含其他 DC 的同一任务） | `"External approval marked. Drawing is now active and QR code has been generated."` |
| 操作成功（Rejected） | Dialog 关闭，Todo 消失 | `"External rejection recorded. Designer has been notified."` |
| QR 生成失败 | loading 恢复，Dialog 保留 | `"QR generation failed, please retry"` |
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
- 键盘导航：Dialog 内字段支持 Tab 键；Radio 支持方向键切换；Esc 关闭 Dialog
- 屏幕阅读器：`aria-live="polite"` 通知 Dialog 状态变更

---

## 7. 文案规范

| 场景 | 文案（en） |
|-----|----------|
| 外部审批 Todo 标题 | External Approval Required |
| 外部审批通过成功 Toast | External approval marked. Drawing is now active and QR code has been generated. |
| 外部审批驳回成功 Toast | External rejection recorded. Designer has been notified. |
| QR 生成失败 Toast | QR generation failed, please retry. |

---

## 8. AC 覆盖检查表

| AC ID | 对应组件 | 覆盖? |
|------|---------|------|
| AC-007B-001 | §3.1 卡片出现时机与内容 | ✅ |
| AC-007B-002 | §3.1 [Download Original] 按钮 | ✅ |
| AC-007B-003 | §3.2 Approved 必填校验 | ✅ |
| AC-007B-004 | §3.2 Approved 成功路径 | ✅ |
| AC-007B-005 | §3.2 Approved 成功 Toast | ✅ |
| AC-007B-006 | §3.2 QR 失败回滚 | ✅ |
| AC-007B-007 | §3.2 Rejected Reason 必填 | ✅ |
| AC-007B-008 | §3.2 Rejected 成功路径 | ✅ |
| AC-007B-009 | §3.1 其他 DC 的任务自动消失 | ✅ |
| AC-007B-010 | §3.1 权限可见性，非 DC 不显示 | ✅ |
| AC-007B-011 | §3.2 loading 禁用防重复 | ✅ |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 UI-REQ-007-pc.md 拆分，覆盖 REQ-007B-pc 外部审批部分 |
| 0.1.1 | 2026-05-24 | agent | 按来源需求独立成文件 |
