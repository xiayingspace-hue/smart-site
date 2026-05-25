---
doc_type: ui_spec
req_id: REQ-003E-pc
version: 0.1.0
status: draft
generated_from: REQ-003E-pc.md@0.3.1
generated_at: 2026-05-24
owner: ""
---

# UI 设计说明：PC 端 — 图纸上传 AI 自动识别页信息

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-003E-pc.md（主）、REQ-003A-pc.md（上传弹窗基础）、glossary.md（术语）
> - 输出引用：Figma 链接、交互原型链接
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003E-pc @ v0.3.1 |
| 覆盖用户故事 | US-003E-001、US-003E-002 |
| 覆盖 AC | AC-003E-001 ~ AC-003E-013 |
| 上次同步时间 | 2026-05-24 |

> ⚠️ 当 requirement.md 版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

在 REQ-003A 定义的上传弹窗基础上，增加 AI 自动识别结果列表区域，让设计人员在提交前能够直观地核对 PDF 各页图纸信息，实现"零手动抄录"的上传体验。

### 1.2 设计原则

1. **状态可感知**：AI 识别过程的每个阶段（上传中 / 识别中 / 完成 / 失败）都有清晰的视觉反馈。
2. **只读核对，输入分离**：识别结果列表仅供核对，最终 Drawing Code / Name 在独立输入框中修改，不在列表内行内编辑。
3. **防误提交**：识别进行中时，Drawing Code / Name 输入框与 [Submit] 按钮均置灰。
4. **降级可恢复**：AI 识别失败时，橙色提示横幅明显提示，并提供 [Re-upload] 快速重试入口。
5. **单一产物**：无论识别页数多少，弹窗始终只创建一条 Drawing 记录，视觉设计不应暗示批量创建。

---

## 2. 信息架构

### 2.1 页面层级

```
侧边栏
└── Drawing Management（一级菜单）
    └── Drawing Masterlist（二级菜单）→ 图纸列表页
        └── Upload New Drawing 弹窗（Modal）
            ├── 阶段 A：基本信息 + 文件上传（初始状态）
            ├── 阶段 B：AI 识别中（Loading 状态）
            ├── 阶段 C：AI 识别完成（含识别结果列表）
            └── 阶段 D：AI 识别失败（降级提示 + 可编辑输入）
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发场景 |
|---------|-------|---------|
| 图纸列表页右上角 [+ Upload Drawing] | Upload New Drawing 弹窗 | 新建图纸，含 AI 识别流程 |
| 降级模式 [Re-upload] 按钮 | 重新触发文件选择 | AI 识别失败后重新选择文件 |

---

## 3. 页面 / 组件清单

### 3.1 Upload New Drawing 弹窗 — 整体布局

**关联 Story**：US-003E-001、US-003E-002
**关联 AC**：AC-003E-001 ~ AC-003E-013

#### 布局（字段从上到下顺序）

```
┌─────────────────────────────────────────────────────────┐
│  Upload New Drawing                              [✕]     │
├─────────────────────────────────────────────────────────┤
│  Drawing Description *（必填）                            │
│  [___________________________________________]           │
│                                                          │
│  Category *（必填）                                       │
│  [▼ Select Category                         ]           │
│                                                          │
│  Drawing File *（必填）                                   │
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┐           │
│  │   拖拽或点击上传 PDF（≤ 100MB）             │           │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┘           │
│                                                          │
│  ┌──────────────────────────────────────────┐           │
│  │ [AI 识别结果列表区域]（上传后出现）         │           │
│  │  详见 §3.3                                │           │
│  └──────────────────────────────────────────┘           │
│                                                          │
│  Drawing Code *（必填）               ← auto-filled by AI │
│  [___________________________________________]           │
│                                                          │
│  Drawing Name *（必填）               ← auto-filled by AI │
│  [___________________________________________]           │
│                                                          │
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
- 宽度：600px（固定）
- 最大高度：80vh，超出时整体纵向滚动
- 遮罩：背景 50% 不透明黑色遮罩，点击遮罩不关闭（防误操作）
- [✕] 按钮：右上角关闭，识别进行中时不禁用（允许取消）

---

### 3.2 文件上传区（Drawing File）

**关联 AC**：AC-003E-010、AC-003E-011

**状态（5 态）**：

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 初始未选择文件 | 虚线边框区域，居中图标 + 提示文字 | `"Click or drag PDF file here (max 100MB)"` |
| 上传中 | 文件已选择，正在上传 | 进度条（蓝色），百分比文字 | `"Uploading... XX%"` |
| 上传完成 / AI 识别中 | 文件上传成功，AI 处理中 | 文件名 + 绿色 ✓ 图标 + Loading Spinner | `"filename.pdf ✓"` + `"Analysing drawing pages…"` |
| 识别完成（DONE） | AI 返回成功结果 | 文件名 + 绿色 ✓ + 可替换小字链接 | `"filename.pdf ✓"` |
| 错误 | 非 PDF / 超 100MB / 上传失败 | 红色边框 + 红色错误文字 | 见下方错误文案 |

**错误文案**：
- 非 PDF：`"AI recognition only supports PDF format."`
- 超 100MB：`"File size exceeds the 100MB limit."`
- 上传失败（网络）：`"Upload failed. Please try again."`

---

### 3.3 AI 识别中状态（弹窗中部区域）

**关联 AC**：AC-003E-001、AC-003E-009

```
┌──────────────────────────────────────────────────────┐
│  ⟳  Analysing drawing pages…                         │
│     （Loading Spinner + 文案，居中）                   │
└──────────────────────────────────────────────────────┘
```

- 背景：浅灰色区域（`#F5F7FA`），圆角 4px
- Loading Spinner：Element UI `el-loading` 样式，蓝色主色
- 此阶段 Drawing Code / Drawing Name 输入框：背景灰化，`disabled` 态，placeholder 显示 `"Filling in after analysis…"`
- [Submit] 按钮：`disabled` 态，灰色

---

### 3.4 AI 识别结果列表区域（DONE 状态）

**关联 AC**：AC-003E-002、AC-003E-003、AC-003E-004、AC-003E-005、AC-003E-012

#### 顶部汇总行

```
┌──────────────────────────────────────────────────────┐
│  ✅ 5 pages detected. Drawing information has been   │
│     pre-filled below. Please review and confirm.     │
└──────────────────────────────────────────────────────┘
```

若有未识别到信息的页，在绿色提示下方追加橙色提示：
```
┌──────────────────────────────────────────────────────┐
│  ⚠  Some pages could not be fully analysed. Please  │
│     verify the drawing information below.            │
└──────────────────────────────────────────────────────┘
```

#### 识别结果列表表格

```
┌───────┬──────────┬───────────────────────┬──────────────────────┐
│ Page  │ Preview  │ Drawing No            │ Drawing Name         │
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
| Drawing No | ~200px | 左对齐 | 只读文本；未识别时显示 `—` + 橙色 ⚠ 图标 |
| Drawing Name | 剩余宽度 | 左对齐 | 只读文本；未识别时显示 `—` + 橙色 ⚠ 图标 |

**列表容器**：
- 最大高度：240px，超出时纵向滚动
- 边框：1px solid `#E4E7ED`，圆角 4px
- 表头背景：`#F5F7FA`，字色 `#909399`，字号 12px

**缩略图点击大图查看**：
- 触发：点击缩略图
- 展示：全屏遮罩 + 居中展示该页 PDF 预览图，点击遮罩或 [✕] 关闭

**未识别行样式**：
- Drawing No / Drawing Name 单元格：橙色文字 `#E6A23C` + ⚠ 图标

---

### 3.5 AI 识别失败降级区域（FAILED 状态）

**关联 AC**：AC-003E-006、AC-003E-007、AC-003E-008

```
┌──────────────────────────────────────────────────────────┐
│  ⚠  Unable to analyse the drawing automatically.        │
│     Please enter the drawing information manually,       │
│     or re-upload the file.           [Re-upload]        │
└──────────────────────────────────────────────────────────┘
```

- 背景：橙色浅底 `#FDF6EC`，左侧 3px 橙色 `#E6A23C` 竖线
- ⚠ 图标：橙色
- [Re-upload] 按钮：Secondary 按钮，右对齐
- 此阶段 Drawing Code / Drawing Name 输入框：恢复为空可编辑状态，placeholder 正常显示

---

### 3.6 Drawing Code / Drawing Name 输入框

**关联 AC**：AC-003E-002、AC-003E-003

**各阶段状态**：

| 阶段 | Drawing Code 状态 | Drawing Name 状态 |
|------|:----------------:|:----------------:|
| 未上传文件 | 空，可编辑 | 空，可编辑 |
| 上传中 / AI 识别中 | 置灰，不可编辑 | 置灰，不可编辑 |
| AI 识别完成（DONE） | 自动填入建议值，可编辑 | 自动填入建议值，可编辑 |
| AI 识别失败（FAILED） | 空，可编辑 | 空，可编辑 |

- 字段标签下方可选显示小字提示（AI 识别完成后）：`"Auto-filled from AI analysis. You may edit if needed."`，字色 `#909399`，字号 12px

---

### 3.7 [Submit] 按钮状态

| 条件 | 按钮状态 |
|-----|---------|
| AI 识别中（PENDING / PROCESSING） | `disabled`，灰色 |
| 必填项未填满 | `disabled`，灰色 |
| 所有必填项已填，AI 已完成或降级 | 可点击，主色蓝 |

---

## 4. 设计令牌（Design Tokens）

> 引用 Element UI 现有 Design System。本节列出新增或本需求内明确使用的 token。

### 4.1 颜色语义

| 用途 | Token / 值 | 备注 |
|-----|-----------|------|
| AI 识别中 Loading 背景 | `#F5F7FA` | Element UI `--color-fill-lighter` |
| 识别成功提示背景 | `#F0F9EB` | Element UI 成功浅底 |
| 识别成功文字 | `#67C23A` | Element UI `--color-success` |
| 识别失败 / 警告背景 | `#FDF6EC` | Element UI warning 浅底 |
| 识别失败 / 警告文字 | `#E6A23C` | Element UI `--color-warning` |
| 未识别单元格 ⚠ 文字 | `#E6A23C` | 同上 |
| 列表表头背景 | `#F5F7FA` | Element UI `--color-fill-lighter` |
| 列表边框 | `#E4E7ED` | Element UI `--border-color-lighter` |

### 4.2 间距与栅格

| 组件 | 属性 | 值 | 来源 |
|-----|------|----|------|
| 弹窗内区域块间距（各字段组） | margin-bottom | 20px | Element UI 表单规范 |
| 识别结果列表区域 | margin-top | 16px | 新增 |
| 识别结果列表区域 | max-height | 240px | REQ-003E §7.2 |
| 缩略图 | width × height | 60px × 60px | REQ-003E §7.2 |
| 降级提示横幅 | padding | 12px 16px | 与 Element UI Alert 对齐 |
| 提示小字（AI 自动填入说明） | font-size | 12px | — |
| 提示小字 | line-height | 18px | — |
| 提示小字 | color | `#909399` | Element UI `--color-text-placeholder` |
| 表头文字 | font-size | 12px | — |
| 表头文字 | font-weight | 500 | — |
| 表头文字 | color | `#909399` | Element UI `--color-text-placeholder` |
| 表格行文字 | font-size | 14px | Element UI 正文默认 |
| 表格行文字 | line-height | 22px | — |
| 表格行文字 | color | `#303133` | Element UI `--color-text-primary` |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 弹窗宽 600px，完整展示 |
| Tablet | 768~1280px | 弹窗宽 540px，缩略图列宽保持 80px |
| Mobile | < 768px | 不支持（PC 专属，见 REQ-003E §9.4） |

---

## 6. 可访问性（Accessibility）

**目标等级**：WCAG AA

| 要素 | 规范 |
|-----|------|
| 弹窗打开时焦点 | 自动移至弹窗第一个可交互元素（Drawing Description 输入框）|
| Tab 顺序 | 从上到下：Description → Category → 文件上传区 → Drawing Code → Drawing Name → Version Note → Internal Approver → Cancel → Submit |
| Loading 状态 | `aria-busy="true"` + `aria-label="Analysing drawing pages"` |
| 识别结果列表 | `role="table"`，表头 `<th scope="col">`，未识别单元格 `aria-label` 含 ⚠ 描述 |
| 缩略图 | `alt="Page N preview"`；点击大图触发 Dialog，`aria-modal="true"` |
| 错误提示 | `role="alert"` 实时播报 |

---

## 7. 动效规范

| 动效 | 时长 | 缓动 | 备注 |
|-----|------|------|------|
| 弹窗弹出 | 200ms | ease-out | Element UI Dialog 默认 |
| 识别结果列表出现 | 300ms | ease-in-out | `opacity: 0 → 1` + `height: 0 → auto` |
| 降级提示横幅出现 | 200ms | ease-out | `opacity: 0 → 1` |
| Loading Spinner | 持续旋转 | linear | — |

---

## 8. Figma / 原型链接

- Figma 设计稿：<!-- 填写上传弹窗（含 AI 识别中 / 识别结果列表 / 降级模式）Frame 链接 -->
- 交互原型：

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-24 | agent | 初稿，从 REQ-003E-pc@0.3.1 派生 |
