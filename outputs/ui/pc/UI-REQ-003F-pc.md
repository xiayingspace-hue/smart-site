---
doc_type: ui_spec
req_id: REQ-003F-pc
version: 0.1.0
status: draft
generated_from: REQ-003F-pc.md@0.5.0
generated_at: 2026-05-25
owner: ""
---

# UI 设计说明：PC 端 — 上传新版本

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-003F-pc.md（主）、REQ-003E-pc.md（AI 识别交互规范参考）、REQ-003A-pc.md（图纸列表页入口）
> - 输出引用：Figma 链接、交互原型链接
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003F-pc @ v0.5.0 |
| 覆盖用户故事 | US-003F-001 |
| 覆盖 AC | AC-003F-001 ~ AC-003F-013 |
| 上次同步时间 | 2026-05-25 |

> ⚠️ 当 requirement.md 版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

在已有图纸的上传新版本弹窗中，增加 AI 自动识别结果列表区域，让设计人员在提交前能够直观地核对新版本 PDF 各页图纸信息，确认上传了正确的文件，然后进入两级审批流程。

### 1.2 设计原则

1. **上下文明确**：弹窗标题区域清晰显示当前操作针对哪张图纸（Drawing Code / Drawing Name），避免误操作。
2. **主记录只读**：Drawing Code、Drawing Name、Category、Description 在版本迭代中不可修改，视觉上以只读灰化样式展示，消除编辑歧义。
3. **状态可感知**：AI 识别过程的每个阶段（上传中 / 识别中 / 完成 / 失败）都有清晰的视觉反馈。
4. **只读核对**：识别结果列表仅供核对 PDF 内容是否正确，不支持行内编辑（主记录字段已确定，无需填入）。
5. **防误提交**：AI 识别进行中时 [Submit] 按钮置灰；识别完成或用户选择 [Proceed Anyway] 后才解锁。
6. **降级可恢复**：AI 识别失败时，橙色提示横幅明显提示，提供 [Re-upload] 快速重试和 [Proceed Anyway] 跳过核对两条路径。

---

## 2. 信息架构

### 2.1 页面层级

```
侧边栏
└── Drawing Management（一级菜单）
    └── Drawing Masterlist（二级菜单）→ 图纸列表页
        └── Upload New Version 弹窗（Modal）
            ├── 阶段 A：基本只读信息 + 文件上传（初始状态）
            ├── 阶段 B：AI 识别中（Loading 状态）
            ├── 阶段 C：AI 识别完成（含识别结果列表，只读）
            └── 阶段 D：AI 识别失败（橙色降级提示，含 Re-upload / Proceed Anyway）
```

### 2.2 导航与入口

| 入口位置 | 触发场景 | 状态限制 |
|---------|---------|---------|
| 图纸列表行 Actions 列 [Upload New Version] 按钮 | 为已有图纸提交新版本 | 仅在 `ACTIVE` / `INTERNAL_REJECTED` / `EXTERNAL_REJECTED` 状态可点；其余状态置灰 |
| 降级模式 [Re-upload] 按钮 | AI 识别失败后重新选择文件 | 仅在识别失败状态显示 |

---

## 3. 页面 / 组件清单

### 3.1 Upload New Version 弹窗 — 整体布局

**关联 Story**：US-003F-001
**关联 AC**：AC-003F-001 ~ AC-003F-013

#### 布局（字段从上到下顺序）

```
┌─────────────────────────────────────────────────────────┐
│  Upload New Version                              [✕]     │
│  ARCH-001 — Ground Floor Plan                           │  ← 副标题（只读上下文，Drawing Code — Drawing Name）
├─────────────────────────────────────────────────────────┤
│  Current Version                                         │
│  V3  （只读，灰色）                                       │
│                                                          │
│  Category                                                │
│  Architectural  （只读，灰色）                            │
│                                                          │
│  Description                                             │
│  Ground floor layout for Block A  （只读，灰色）          │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│  Drawing File *（必填）                                   │
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┐           │
│  │   拖拽或点击上传 PDF（≤ 50MB）              │           │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┘           │
│                                                          │
│  ┌──────────────────────────────────────────┐           │
│  │ [AI 识别结果列表区域]（上传后出现）         │           │
│  │  详见 §3.3 / §3.4 / §3.5                 │           │
│  └──────────────────────────────────────────┘           │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│  Version Note（选填）                                     │
│  [___________________________________________]           │
│                                                          │
│  Internal Approver *（必填）                              │
│  [▼ Select Approver                         ]           │
│                                                          │
├─────────────────────────────────────────────────────────┤
│                              [Cancel]   [Submit ▶]      │
└─────────────────────────────────────────────────────────┘
```

**弹窗规格**：

| 属性 | 值 |
|-----|---|
| 宽度 | 600px（固定） |
| 最大高度 | 80vh，超出时整体纵向滚动 |
| 遮罩 | 背景 50% 不透明黑色，点击遮罩不关闭（防误操作） |
| [✕] 按钮 | 右上角，任何阶段均可点击（允许取消） |
| 副标题位置 | 主标题下方，字号 14px，字色 `#606266`，`{drawingCode} — {drawingName}` |

**只读信息区与可编辑区分隔**：
- 只读信息区（Current Version / Category / Description）与 Drawing File 上传区之间用细分隔线（`#E4E7ED`）分隔，视觉上区分"核对区"与"操作区"。

---

### 3.2 只读信息区（Current Version / Category / Description）

**关联 AC**：AC-003F-012

**样式规范**：
- 字段标签：字号 12px，字色 `#909399`，font-weight 400
- 字段值：字号 14px，字色 `#C0C4CC`（灰化，明确表示不可编辑），背景 `#F5F7FA`，padding 8px 12px，圆角 4px
- 无输入框边框，类似纯文本展示，不给用户编辑预期
- Description 无内容时显示 `—`（同色灰化）

**Current Version 展示**：
- 格式：`V{n}`（如 `V3`）
- 右侧可选附文字提示：`"Next version will be V{n+1} after submission"`，字号 12px，字色 `#909399`

---

### 3.3 文件上传区（Drawing File）

**关联 AC**：AC-003F-008、AC-003F-009

**状态（5 态）**：

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 初始未选择文件 | 虚线边框区域，居中图标 + 提示文字 | `"Click or drag PDF file here (max 50MB)"` |
| 上传中 | 文件已选择，正在上传 | 进度条（蓝色），百分比文字 | `"Uploading... XX%"` |
| 上传完成 / AI 识别中 | 文件上传成功，AI 处理中 | 文件名 + 绿色 ✓ 图标 + Loading Spinner | `"filename.pdf ✓"` + `"Analysing drawing pages…"` |
| 识别完成（DONE） | AI 返回成功结果 | 文件名 + 绿色 ✓，可点击更换 | `"filename.pdf ✓"` |
| 错误 | 非 PDF / 超 50MB / 上传失败 | 红色边框 + 红色错误文字 | 见下方错误文案 |

**错误文案**：
- 非 PDF：`"Only PDF format is supported for version upload."`
- 超 50MB：`"File size exceeds the 50MB limit."`
- 上传失败（网络）：`"Upload failed. Please try again."`

---

### 3.4 AI 识别中状态（弹窗中部区域）

**关联 AC**：AC-003F-002

```
┌──────────────────────────────────────────────────────┐
│  ⟳  Analysing drawing pages…                         │
│     （Loading Spinner + 文案，居中）                   │
└──────────────────────────────────────────────────────┘
```

- 背景：浅灰色区域（`#F5F7FA`），圆角 4px，高度 80px
- Loading Spinner：蓝色主色，旋转动画
- 文案：`"Analysing drawing pages…"`，字色 `#606266`，字号 14px
- 此阶段 [Submit] 按钮：`disabled` 态，灰色

---

### 3.5 AI 识别结果列表区域（DONE 状态，只读核对视图）

**关联 AC**：AC-003F-003、AC-003F-004

#### 顶部汇总行

识别正常（无缺失页）：
```
┌──────────────────────────────────────────────────────┐
│  ✅ 5 pages detected. Please review the drawing      │
│     information below before submitting.             │
└──────────────────────────────────────────────────────┘
```

识别完成但有页面信息缺失，在上方绿色提示下方追加橙色提示：
```
┌──────────────────────────────────────────────────────┐
│  ⚠  Some pages could not be fully analysed. Please  │
│     verify the drawing information below.            │
└──────────────────────────────────────────────────────┘
```

#### 识别结果列表表格

```
┌───────┬──────────┬───────────────────────┬──────────────────────┐
│ Page  │ Preview  │ Drawing Code          │ Drawing Name         │
├───────┼──────────┼───────────────────────┼──────────────────────┤
│ Page 1│ [img 60] │ ARCH-001              │ Ground Floor Plan    │
│ Page 2│ [img 60] │ ARCH-002              │ First Floor Plan     │
│ Page 3│ [img 60] │ ⚠ —                  │ ⚠ —                  │
│ Page 4│ [img 60] │ ARCH-004              │ Roof Plan            │
│ Page 5│ [img 60] │ STRUCT-001            │ Foundation Layout    │
└───────┴──────────┴───────────────────────┴──────────────────────┘
```

**列规格**：

| 列 | 宽度 | 对齐 | 说明 |
|----|------|------|------|
| Page | 64px | 左对齐 | `Page N` 只读文本 |
| Preview | 80px | 居中 | 60×60px 缩略图，支持点击查看大图（Lightbox） |
| Drawing Code | ~200px | 左对齐 | 只读文本；未识别时显示 `—` + 橙色 ⚠ 图标 |
| Drawing Name | 剩余宽度 | 左对齐 | 只读文本；未识别时显示 `—` + 橙色 ⚠ 图标 |

> **与 REQ-003E 的关键区别**：REQ-003E 的识别结果列表中，AI 识别结果会自动填入弹窗下方的 Drawing Code / Name 输入框（因为是新建图纸，需要确认主记录字段值）。REQ-003F 中没有此填入行为——主记录字段已确定，列表仅供核对 PDF 内容是否正确，不存在"填入"动作。

**列表容器**：
- 最大高度：240px，超出时纵向滚动
- 边框：1px solid `#E4E7ED`，圆角 4px
- 表头背景：`#F5F7FA`，字色 `#909399`，字号 12px，font-weight 500

**缩略图点击大图查看**：
- 触发：点击缩略图
- 展示：全屏遮罩 + 居中展示该页 PDF 预览图，点击遮罩或 [✕] 关闭
- Dialog `aria-modal="true"`

**未识别行样式**：
- Drawing Code / Drawing Name 单元格：橙色文字 `#E6A23C` + ⚠ 图标，`aria-label` 含未识别描述

---

### 3.6 AI 识别失败降级区域（FAILED 状态）

**关联 AC**：AC-003F-005、AC-003F-006、AC-003F-007

```
┌──────────────────────────────────────────────────────────────┐
│  ⚠  Unable to analyse the drawing automatically.            │
│     Please re-upload the file or proceed to submit.         │
│                                    [Re-upload]  [Proceed Anyway] │
└──────────────────────────────────────────────────────────────┘
```

- 背景：橙色浅底 `#FDF6EC`，左侧 3px 橙色 `#E6A23C` 竖线，圆角 4px
- ⚠ 图标：橙色
- **[Re-upload]**：Secondary 按钮，点击清空当前文件，重新触发文件选择，开启新一轮 AI 识别
- **[Proceed Anyway]**：Primary 按钮（橙色 `#E6A23C` 变体，区别于正常蓝色 Primary），点击后：
  - 降级提示横幅保持显示（不消失）
  - [Submit] 按钮解锁，颜色恢复主色蓝，可正常提交
  - Version Note / Internal Approver 字段正常可填写

---

### 3.7 [Submit] 按钮状态

**关联 AC**：AC-003F-002、AC-003F-005、AC-003F-010

| 条件 | 按钮状态 | 颜色 |
|-----|---------|------|
| 未上传文件 | `disabled` | 灰色 |
| 上传中 / AI 识别中（PENDING / PROCESSING） | `disabled` | 灰色 |
| AI 识别完成（DONE），Internal Approver 未选 | `disabled` | 灰色 |
| AI 识别完成（DONE），所有必填项已填 | 可点击 | 主色蓝 |
| AI 识别失败（FAILED），用户未选择操作 | `disabled` | 灰色 |
| 用户点击 [Proceed Anyway]，所有必填项已填 | 可点击 | 主色蓝 |
| 提交进行中（文件上传中） | `disabled` + 加载态 | 灰色 + Spinner |

---

### 3.8 Actions 列按钮状态（图纸列表页）

**关联 AC**：AC-003F-011

| Drawing 状态 | [Upload New Version] 按钮 | Tooltip |
|------------|:------------------------:|---------|
| `ACTIVE` | 可点击 | — |
| `INTERNAL_REJECTED` | 可点击 | — |
| `EXTERNAL_REJECTED` | 可点击 | — |
| `PENDING_INTERNAL` | 置灰 | `"A version is currently under review."` |
| `PENDING_EXTERNAL` | 置灰 | `"A version is currently under review."` |

---

## 4. 设计令牌（Design Tokens）

> 引用 Element UI 现有 Design System。本节列出本需求明确使用的 token。

### 4.1 颜色语义

| 用途 | Token / 值 | 备注 |
|-----|-----------|------|
| 只读字段值字色 | `#C0C4CC` | Element UI `--color-text-secondary`（偏浅灰，区分可编辑） |
| 只读字段背景 | `#F5F7FA` | Element UI `--color-fill-lighter` |
| AI 识别中 Loading 背景 | `#F5F7FA` | 同上 |
| 识别成功提示背景 | `#F0F9EB` | Element UI 成功浅底 |
| 识别成功文字 / 图标 | `#67C23A` | Element UI `--color-success` |
| 识别失败 / 警告背景 | `#FDF6EC` | Element UI warning 浅底 |
| 识别失败 / 警告文字 | `#E6A23C` | Element UI `--color-warning` |
| 未识别单元格 ⚠ 文字 | `#E6A23C` | 同上 |
| 列表表头背景 | `#F5F7FA` | Element UI `--color-fill-lighter` |
| 列表边框 | `#E4E7ED` | Element UI `--border-color-lighter` |
| 副标题文字（drawingCode — drawingName） | `#606266` | Element UI `--color-text-regular` |
| 分隔线 | `#E4E7ED` | Element UI `--border-color-lighter` |

### 4.2 间距与尺寸

| 组件 | 属性 | 值 |
|-----|------|---|
| 弹窗宽度 | width | 600px |
| 弹窗最大高度 | max-height | 80vh |
| 副标题 | font-size | 14px |
| 只读字段区块间距 | margin-bottom | 16px |
| 只读字段值区域 | padding | 8px 12px |
| 只读区与操作区分隔线 | margin | 20px 0 |
| 识别结果列表最大高度 | max-height | 240px |
| 缩略图 | width × height | 60px × 60px |
| 降级提示横幅 | padding | 12px 16px |
| 降级提示左竖线 | border-left | 3px solid `#E6A23C` |
| 表头文字 | font-size | 12px |
| 表头文字 | font-weight | 500 |
| 表格行文字 | font-size | 14px |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 弹窗宽 600px，完整展示 |
| Tablet | 768~1280px | 弹窗宽 540px，缩略图列宽保持 80px |
| Mobile | < 768px | 不支持（PC 专属，见 REQ-003F §8.4） |

---

## 6. 可访问性（Accessibility）

**目标等级**：WCAG AA

| 要素 | 规范 |
|-----|------|
| 弹窗打开时焦点 | 自动移至弹窗第一个可交互元素（Drawing File 上传区）|
| 只读字段 | `aria-readonly="true"`，屏幕阅读器可读取字段值 |
| Tab 顺序 | 文件上传区 → Version Note → Internal Approver → Cancel → Submit |
| Loading 状态 | `aria-busy="true"` + `aria-label="Analysing drawing pages"` |
| 识别结果列表 | `role="table"`，表头 `<th scope="col">`；整体标注 `aria-label="AI recognition results, read-only"` |
| 未识别单元格 | `aria-label="Not recognised"` + ⚠ 描述 |
| 缩略图 | `alt="Page N preview"`；点击大图触发 Dialog，`aria-modal="true"` |
| 降级提示横幅 | `role="alert"` 实时播报 |
| [Proceed Anyway] 按钮 | 无障碍标签：`"Proceed to submit without AI verification"` |
| 置灰的 [Upload New Version] 按钮 | `aria-disabled="true"` + Tooltip 文案 |

---

## 7. 动效规范

| 动效 | 时长 | 缓动 | 备注 |
|-----|------|------|------|
| 弹窗弹出 | 200ms | ease-out | Element UI Dialog 默认 |
| AI 识别结果列表出现 | 300ms | ease-in-out | `opacity: 0 → 1` + `height: 0 → auto` |
| 降级提示横幅出现 | 200ms | ease-out | `opacity: 0 → 1` |
| Loading Spinner | 持续旋转 | linear | — |
| [Proceed Anyway] 点击后 Submit 按钮解锁 | 150ms | ease-out | 灰色 → 主色蓝过渡 |

---

## 8. Figma / 原型链接

- Figma 设计稿：<!-- 填写 Upload New Version 弹窗（含阶段 A/B/C/D）Frame 链接 -->
- 交互原型：

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-25 | agent | 初稿，从 REQ-003F-pc@0.5.0 派生；完整定义 Upload New Version 弹窗四阶段（初始 / AI 识别中 / 识别完成只读列表 / 识别失败降级）、只读信息区样式、[Proceed Anyway] 降级按钮、Actions 列按钮状态、Design Tokens、可访问性规范 |
