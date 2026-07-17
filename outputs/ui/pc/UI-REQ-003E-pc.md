---
doc_type: ui_spec
req_id: REQ-003E-pc
version: 0.2.0
status: draft
generated_from: REQ-003E-pc@0.4.2
generated_at: 2026-07-16
owner: ""
---

# UI 设计说明：PC 端 — 图纸上传 AI 自动识别页信息

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-003E-pc.md（主）、REQ-003A-pc.md（上传弹窗基础、图纸详情页）、glossary.md（术语）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc §3.2
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003E-pc @ v0.4.2 |
| 覆盖用户故事 | US-003E-001、US-003E-002、US-003E-003 |
| 覆盖 AC | AC-003E-001 ~ AC-003E-019 |
| 上次同步时间 | 2026-07-16 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

> **⚠️ 本文档存在阻塞项**：REQ-003E-pc OQ-002（多页各有不同 Drawing No 时，Drawing Code 输入框自动填入哪页的值）未关闭。§3.7 的自动填入行为**无法完整设计**，已在该节及 §12 显式标注，未编造方案。

---

## 1. 设计目标

### 1.1 核心目标

在 REQ-003A 定义的上传弹窗基础上，增加 **Submission Type 选择**与 **AI 识别结果列表**两个区域：让设计人员先按业务语义声明"我传的是图纸还是其它文件"，系统据此决定是否做 AI 识别；上传 Shop Drawing 时可在提交前直观核对 PDF 各页图纸信息，实现"零手动抄录"；上传 Others 时无识别等待、无需填写图纸编号，传完直接提交。

### 1.2 设计原则

1. **按业务语义分流，不暴露实现**：用户选的是"文件是什么"（Shop Drawing / Others），而非"要不要用 AI"。AI 是否运行由类型推导，不做成开关。
2. **状态可感知**：AI 识别过程的每个阶段（上传中 / 识别中 / 完成 / 失败）都有清晰的视觉反馈。
3. **只读核对，输入分离**：识别结果列表仅供核对，最终 Drawing Code / Name 在独立输入框中修改，不在列表内行内编辑。
4. **防误提交**：识别进行中时，Drawing Code / Name 输入框与 [Submit] 按钮均置灰。
5. **降级可恢复**：AI 识别失败时，橙色提示横幅明显提示，并提供 [Re-upload] 快速重试入口。
6. **Others 路径必须"轻"**：不显示识别区域、不显示 Loading、不要求填编号。任何为 Shop Drawing 设计的等待或核对成本都不得渗入 Others。
7. **单一产物**：无论识别页数多少、无论何种 Submission Type，弹窗始终只创建一条 Drawing 记录，视觉设计不应暗示批量创建。

---

## 2. 信息架构

### 2.1 入口

图纸管理列表页（Drawing Masterlist）右上角 → **[+ Upload Drawing]** 主按钮（仅设计人员可见）

### 2.2 页面层级

```
侧边栏
└── Drawing Management（一级菜单）
    └── Drawing Masterlist（二级菜单）→ 图纸列表页
        ├── Upload New Drawing 弹窗（Modal）
        │   ├── 形态 A：Shop Drawing（默认）
        │   │   ├── A-1 初始态：基本信息 + 类型 + 文件上传
        │   │   ├── A-2 AI 识别中（Loading）
        │   │   ├── A-3 AI 识别完成（含识别结果列表）
        │   │   └── A-4 AI 识别失败（降级提示 + 可编辑输入）
        │   └── 形态 B：Others
        │       ├── B-1 初始态：基本信息 + 类型 + 文件上传
        │       └── B-2 上传完成（无识别区、Code/Name 置灰为空）
        └── Drawing 详情页
            └── AI 识别结果区域（新增，仅 Shop Drawing + 识别成功的记录展示）
```

### 2.3 导航与入口

| 入口位置 | 链接到 | 触发场景 |
|---------|-------|---------|
| 图纸列表页右上角 [+ Upload Drawing] | Upload New Drawing 弹窗（默认形态 A） | 新建图纸 |
| 弹窗内 Submission Type 单选 | 形态 A ⇄ 形态 B 切换 | 用户改变提交类型（切换会清空已上传文件，见 §3.2） |
| 降级模式 [Re-upload] 按钮 | 重新触发文件选择 | AI 识别失败后重新选择文件 |
| 图纸列表行 → Drawing 详情页 | 详情页 AI 识别结果区域 | 回溯来源 PDF 的各页识别内容 |

---

## 3. 页面 / 组件清单

### 3.1 Upload New Drawing 弹窗 — 整体布局

**关联 Story**：US-003E-001、US-003E-002、US-003E-003
**关联 AC**：AC-003E-001 ~ AC-003E-011、AC-003E-014 ~ AC-003E-019

#### 布局 — 形态 A：Shop Drawing（默认，识别完成态）

```
┌─────────────────────────────────────────────────────────┐
│  Upload New Drawing                              [✕]    │
├─────────────────────────────────────────────────────────┤
│  Drawing Description（选填）                             │
│  [___________________________________________]          │
│                                                         │
│  Category *                                             │
│  [▼ Select Category                         ]           │
│                                                         │
│  Submission Type *                                      │
│  ( • ) Shop Drawing      (   ) Others                   │
│                                                         │
│  Drawing File *                                         │
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┐          │
│  │   拖拽或点击上传 PDF（≤ 50MB）             │          │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┘          │
│                                                         │
│  ┌───────────────────────────────────────────┐          │
│  │ [AI 识别结果列表区域]  详见 §3.5           │          │
│  └───────────────────────────────────────────┘          │
│                                                         │
│  Drawing Code *                  ← auto-filled by AI    │
│  [___________________________________________]          │
│                                                         │
│  Drawing Name *                  ← auto-filled by AI    │
│  [___________________________________________]          │
│                                                         │
│  Version Note（选填）                                    │
│  [___________________________________________]          │
│                                                         │
│  Internal Approver *                                    │
│  [▼ Select Approver                         ]           │
├─────────────────────────────────────────────────────────┤
│                              [Cancel]   [Submit ▶]      │
└─────────────────────────────────────────────────────────┘
```

#### 布局 — 形态 B：Others（上传完成态）

```
┌─────────────────────────────────────────────────────────┐
│  Upload New Drawing                              [✕]    │
├─────────────────────────────────────────────────────────┤
│  Drawing Description（选填）                             │
│  [___________________________________________]          │
│                                                         │
│  Category *                                             │
│  [▼ Select Category                         ]           │
│                                                         │
│  Submission Type *                                      │
│  (   ) Shop Drawing      ( • ) Others                   │
│                                                         │
│  Drawing File *                                         │
│  ┌───────────────────────────────────────────┐          │
│  │  📄 method-statement.pdf  ✓        [替换]  │          │
│  └───────────────────────────────────────────┘          │
│                                                         │
│         ← 无 AI 识别结果列表区域（整块不渲染）            │
│                                                         │
│  Drawing Code                                           │
│  [▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒]  ← 置灰、空     │
│                                                         │
│  Drawing Name                                           │
│  [▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒]  ← 置灰、空     │
│                                                         │
│  Version Note（选填）                                    │
│  [___________________________________________]          │
│                                                         │
│  Internal Approver *                                    │
│  [▼ Select Approver                         ]           │
├─────────────────────────────────────────────────────────┤
│                              [Cancel]   [Submit ▶]      │
└─────────────────────────────────────────────────────────┘
```

> ASCII 图仅表达结构层级与元素并列关系，不表达间距数值与对齐方式——间距见 §4.2。

#### 字段展示

> 字段类型与约束引用 REQ-003E-pc §7.1 与 data-contract.md，此处不重复定义。

| 序号 | 字段 / 区域 | 必填 | Shop Drawing | Others |
|-----|-----------|:---:|-------------|--------|
| 1 | Drawing Description | ❌ | 文本输入框 | 同左 |
| 2 | Category | ✅ | 下拉单选 | 同左 |
| 3 | **Submission Type** | ✅ | 单选，默认选中 | 同左 |
| 4 | Drawing File | ✅ | 文件上传区，仅 PDF ≤ 50MB | 同左 |
| 5 | AI 识别结果列表 | — | 上传后出现（§3.5） | **整块不渲染** |
| 6 | Drawing Code | 条件 | ✅ 必填，AI 预填可编辑 | **空 + 置灰，不校验** |
| 7 | Drawing Name | 条件 | ✅ 必填，AI 预填可编辑 | **空 + 置灰，不校验** |
| 8 | Version Note | ❌ | 文本输入框 | 同左 |
| 9 | Internal Approver | ✅ | 下拉单选 | 同左 |

**弹窗规格**：
- 宽度：600px（固定）
- 最大高度：80vh，超出时整体纵向滚动
- 遮罩：背景 50% 不透明黑色遮罩，点击遮罩不关闭（防误操作）
- [✕] 按钮：右上角关闭，识别进行中时不禁用（允许取消；识别任务后台继续但结果不再展示）
- **形态 B 弹窗高度显著低于形态 A**（无识别列表区）；切换类型时弹窗高度变化应有过渡，见 §6

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 弹窗刚打开 | Submission Type 默认选中 Shop Drawing；文件上传区为虚线空态；Code/Name 空且可编辑 | 见 §9 |
| 加载 | 文件上传中 / AI 识别中（仅形态 A） | 局部：上传区进度条 → 识别区 Spinner。**不使用全屏遮罩**，用户可继续填 Version Note / Approver | `"Uploading... XX%"` / `"Analysing drawing pages…"` |
| 正常 | 形态 A 识别完成 / 形态 B 上传完成 | 形态 A 显示识别列表 + 预填值；形态 B 无列表、Code/Name 置灰 | — |
| 错误 | ① 接口失败：创建接口报错 → 弹窗保留 + 字段级错误<br>② 权限失败：非设计人员 → 入口按钮即隐藏，不会到达本弹窗<br>③ 网络断：上传中断 → Toast + 弹窗保留可重试 | ① Drawing Code 下方红字 + 边框红<br>③ 顶部 Toast，不关闭弹窗 | 见 §9 |
| 极端数据 | ① PDF 100 页 → 识别列表内滚动，容器 max-height 240px 不变<br>② Drawing Name 超长（>200 字符）→ 列表单元格单行截断 + `…` + Tooltip 悬浮显示全文<br>③ 文件名超长 → 中间省略（`abc…xyz.pdf`），保留扩展名<br>④ Category / Approver 选项 > 100 → 下拉内置搜索过滤 | — | — |

#### 权限可见性

> 依据 REQ-003E-pc §3 权限矩阵翻译。默认策略：**隐藏 > 禁用**。

| UI 元素 | 设计人员 | 内部审批人 | 项目管理人员 | Site Engineer |
|--------|:-------:|:---------:|:----------:|:------------:|
| [+ Upload Drawing] 入口按钮 | 显示 | 隐藏 | 隐藏 | 隐藏 |
| Upload New Drawing 弹窗（整体） | 显示 | 隐藏 | 隐藏 | 隐藏 |
| Submission Type 单选 | 显示 | — | — | — |
| AI 识别结果列表 | 显示 | — | — | — |
| Drawing Code / Name 输入框 | 显示 | — | — | — |
| [Submit] 按钮 | 显示 | — | — | — |

> 内部审批人 / 项目管理人员 / Site Engineer 对本需求所有操作均为 ❌，故整个弹窗入口隐藏，不存在"进入弹窗但按钮禁用"的中间态。

---

### 3.2 Submission Type 选择器（新增组件）

**关联 Story**：US-003E-003
**关联 AC**：AC-003E-014、AC-003E-018、AC-003E-019

#### 默认态

```
Submission Type *
( • ) Shop Drawing      (   ) Others
```

- 组件：`el-radio-group` + 2 个 `el-radio`
- **默认选中 `Shop Drawing`**（REQ-003E-pc §7.1）
- 水平排列，两项间距 32px
- 位置：**Category 之后、Drawing File 之前**（REQ-003E-pc §7.1 字段顺序 #3）

> **组件选型说明**：当前仅 2 个选项，`el-radio-group` 可一眼看全、零点击即知默认值，优于下拉。若 OQ-008（Others 拆分为更细类型）落地导致选项 ≥ 5，须改用 `el-select`，届时"默认选中"的表达方式需重新设计。见 §12。

#### 关键交互

| 操作 | 行为 |
|-----|------|
| 弹窗打开 | 默认选中 Shop Drawing；文件上传区**即刻可用**（不存在"未选类型"的禁用态） |
| 选中 Shop Drawing（从 Others 切换） | 显示 AI 识别结果列表占位；Drawing Code / Name 恢复为空**可编辑**；清空已上传文件 |
| 选中 Others（从 Shop Drawing 切换） | 隐藏 AI 识别结果列表整块；Drawing Code / Name 清空并**置灰**；清空已上传文件 |
| 已上传文件后切换类型 | **弹二次确认**：已上传文件将被清空 <!-- TODO: 需求 §6.1 仅规定"清空"，未规定是否二次确认。此处为 UI 提案，待 PM 确认，见 §12 --> |
| AI 识别中切换类型 | 同上；切换后原识别任务结果不再展示 |

#### 状态（5 态）

| 状态 | 说明 |
|-----|------|
| 空 | 不适用——本组件恒有默认值（Shop Drawing），不存在未选态 |
| 加载 | 不适用——选项为前端静态枚举，不依赖接口 |
| 正常 | 两个单选项，其一选中 |
| 错误 | 不适用——恒有值，无法触发必填校验失败 |
| 极端数据 | 当前固定 2 项，无溢出风险。OQ-008 落地后需重新评估（见上方组件选型说明） |

#### 权限可见性

| UI 元素 | 设计人员 | 其他角色 |
|--------|:-------:|:-------:|
| Submission Type 单选 | 显示且可操作 | 隐藏（弹窗整体不可达） |

---

### 3.3 文件上传区（Drawing File）

**关联 AC**：AC-003E-010、AC-003E-011

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 初始未选择文件 | 虚线边框区域，居中上传图标 + 提示文字 | `"Click or drag PDF file here (max 50MB)"` |
| 加载 | 文件已选择，正在上传 | 进度条（蓝色）+ 百分比文字 | `"Uploading... XX%"` |
| 正常 | 上传成功 | 文件名 + 绿色 ✓ 图标 + [替换] 小字链接 | `"filename.pdf ✓"` |
| 错误 | 非 PDF / 超 50MB / 上传失败 / PDF 损坏 | 红色边框 + 下方红色错误文字 | 见 §9 错误文案 |
| 极端数据 | 文件名超长（> 40 字符） | 中间省略：`abcdefg…xyz.pdf`，保留扩展名；Tooltip 悬浮显示完整文件名 | — |

**格式与大小约束（两类提交完全相同）**：
- 仅接受 `.pdf`，`accept="application/pdf"`
- ≤ 50MB
- 前端拒绝后**文件上传区保持清空状态**（不保留非法文件名）

> ⚠️ **与旧版 UI 文档的差异**：旧版 v0.1.0 写的是 100MB，与需求 §7.1 的 50MB 不符，本次已修正。

---

### 3.4 AI 识别中状态（仅形态 A）

**关联 AC**：AC-003E-001、AC-003E-009

```
┌──────────────────────────────────────────────────────┐
│  ⟳  Analysing drawing pages…                         │
│     （Loading Spinner + 文案，水平垂直居中）           │
└──────────────────────────────────────────────────────┘
```

- 背景：`#F5F7FA`，圆角 4px，高度 88px
- Loading Spinner：Element UI `el-loading` 样式，蓝色主色
- 此阶段 Drawing Code / Drawing Name 输入框：`disabled` 态，placeholder 显示 `"Filling in after analysis…"`
- [Submit] 按钮：`disabled` 态
- **形态 B 永不进入本状态**——Others 无识别过程，上传完成即可提交

#### 状态（5 态）

| 状态 | 说明 |
|-----|------|
| 空 | 不适用——本区域仅在识别期间存在 |
| 加载 | 即本状态自身（Spinner + 文案） |
| 正常 | 识别完成后本区域被 §3.5 识别结果列表替换 |
| 错误 | 识别失败后本区域被 §3.6 降级横幅替换 |
| 极端数据 | 识别超时（超过 OQ-001 阈值）→ 视为失败，转 §3.6。<!-- TODO: OQ-001 超时阈值未定，Loading 最长持续时间无法确定，见 §12 --> |

---

### 3.5 AI 识别结果列表区域（形态 A — DONE 状态）

**关联 AC**：AC-003E-002、AC-003E-004
**适用范围**：**仅 Shop Drawing**。形态 B 整块不渲染。

#### 顶部汇总行

```
┌──────────────────────────────────────────────────────┐
│  ✅ 5 pages detected. Drawing information has been   │
│     pre-filled below. Please review and confirm.     │
└──────────────────────────────────────────────────────┘
```

若有未识别到信息的页，在绿色提示**下方追加**橙色提示（两条并存，非替换）：

```
┌──────────────────────────────────────────────────────┐
│  ⚠  Some pages could not be fully analysed. Please   │
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
│ Page 3│ [img 60] │ ⚠ —                   │ ⚠ —                  │
│ Page 4│ [img 60] │ ARCH-004              │ Roof Plan            │
│ Page 5│ [img 60] │ STRUCT-001            │ Foundation Layout    │
└───────┴──────────┴───────────────────────┴──────────────────────┘
```

**列规格**：

| 列 | 宽度 | 对齐 | 说明 |
|----|------|------|------|
| Page | 64px 固定 | 左对齐 | `Page N` 只读文本 |
| Preview | 80px 固定 | 居中 | 60×60px 缩略图，点击查看大图（Lightbox） |
| Drawing No | 200px 固定 | 左对齐 | 只读文本；未识别时 `—` + 橙色 ⚠ |
| Drawing Name | `flex: 1`（占满剩余） | 左对齐 | 只读文本；未识别时 `—` + 橙色 ⚠ |

> 列宽分配：前 3 列固定宽度，Drawing Name 列 `flex: 1` 吸收剩余空间；各单元格内容 `text-align: left`、`align-items: center`（垂直居中）。

**列表容器**：
- 最大高度：240px（REQ-003E-pc §7.2），超出时纵向滚动
- 边框：1px solid `#E4E7ED`，圆角 4px
- **只读**：无 Checkbox 列、无行内编辑、无行操作按钮——视觉上不得暗示可批量选择或批量创建

**缩略图点击大图（Lightbox）**：
- 触发：点击缩略图
- 展示：全屏遮罩 + 居中展示该页 PDF 预览图；点击遮罩或 [✕] 关闭
- 键盘：`Esc` 关闭

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 未上传文件（形态 A 初始态） | 整块不渲染，不占垂直空间 | — |
| 加载 | AI 识别中 | 被 §3.4 Loading 区替换 | `"Analysing drawing pages…"` |
| 正常 | 识别成功且所有页均识别到信息 | 绿色汇总行 + 表格 | `"X pages detected…"` |
| 错误 | ① 识别失败 → 被 §3.6 降级横幅替换<br>② 部分页未识别 → 追加橙色提示，对应单元格 `—` + ⚠ | 橙色文字 `#E6A23C` + ⚠ 图标 | `"Some pages could not be fully analysed."` |
| 极端数据 | ① 100 页 PDF → 容器内滚动，渲染 ≤ 1s（REQ-003E-pc §9.1）<br>② 超长 Drawing Name → 单行截断 + `…` + Tooltip 全文<br>③ 缩略图加载失败 → 占位灰块 + 文件图标，不阻塞整行渲染<br>④ 全部页均未识别 → 表格全为 `—`，输入框保持为空 | — | — |

#### 权限可见性

| UI 元素 | 设计人员 | 其他角色 |
|--------|:-------:|:-------:|
| AI 识别结果列表 | 显示（只读） | 隐藏（弹窗整体不可达） |

---

### 3.6 AI 识别失败降级区域（形态 A — FAILED 状态）

**关联 AC**：AC-003E-006、AC-003E-007、AC-003E-008
**适用范围**：**仅 Shop Drawing**。Others 不识别，不存在"降级"概念。

```
┌──────────────────────────────────────────────────────────┐
│  ⚠  Unable to analyse the drawing automatically.         │
│     Please enter the drawing information manually,        │
│     or re-upload the file.            [Re-upload]         │
└──────────────────────────────────────────────────────────┘
```

- 背景：橙色浅底 `#FDF6EC`，左侧 3px 橙色 `#E6A23C` 竖线
- ⚠ 图标：橙色
- [Re-upload] 按钮：Secondary 按钮，右对齐
- 此阶段 Drawing Code / Drawing Name 输入框：**恢复为空且可编辑**，placeholder 正常显示

#### 关键交互

| 操作 | 行为 |
|-----|------|
| 点击 [Re-upload] | 清空当前文件 → 重新唤起文件选择器 → 选中后触发新一轮识别（回到 §3.4） |
| 直接手动填写 Code / Name | 不清除降级横幅；填满必填项后 [Submit] 可点 |

#### 状态（5 态）

| 状态 | 说明 |
|-----|------|
| 空 | 不适用——本区域仅在识别失败后存在 |
| 加载 | 不适用——本区域为终态展示，[Re-upload] 后跳转至 §3.4 的加载态 |
| 正常 | 即本状态自身（橙色横幅 + [Re-upload]） |
| 错误 | 即本状态自身——本区域本身就是错误态的呈现 |
| 极端数据 | 连续多次识别失败 → 横幅文案不变、不累积堆叠，始终仅展示一条 |

---

### 3.7 Drawing Code / Drawing Name 输入框

**关联 AC**：AC-003E-002、AC-003E-003、AC-003E-007、AC-003E-014、AC-003E-015

#### 各阶段状态（按 Submission Type 分）

| Submission Type | 阶段 | Drawing Code | Drawing Name |
|----------------|------|-------------|-------------|
| Shop Drawing | 未上传文件 | 空，可编辑 | 空，可编辑 |
| Shop Drawing | 上传中 / AI 识别中 | 置灰，不可编辑 | 置灰，不可编辑 |
| Shop Drawing | 识别完成（DONE） | 自动填入建议值，可编辑 | 自动填入建议值，可编辑 |
| Shop Drawing | 识别失败（FAILED） | 空，可编辑 | 空，可编辑 |
| **Others** | **全阶段** | **空 + 置灰，不可编辑，提交时不校验** | **同左** |

#### Others 只读态视觉

- `disabled` 态：背景 `#F5F7FA`，边框 `#E4E7ED`，光标 `not-allowed`
- **内容为空**（非占位横线、非 `—`、非灰字提示）——需求 §7.1 明确"显示为空"
- 字段标签**保留显示**（不隐藏字段），但**移除必填星号 `*`**
- 标签下方小字说明：`"Not applicable for Others."`，字色 `#909399`，字号 12px

> **设计意图**：字段保留而非隐藏，是为了让用户理解"这两项存在但对 Others 不适用"，避免切换类型时表单结构剧烈跳动。

#### Shop Drawing 自动填入提示

- 识别完成后，字段标签下方显示小字：`"Auto-filled from AI analysis. You may edit if needed."`，字色 `#909399`，字号 12px

> 🚫 **阻塞：自动填入取值规则未定**
>
> <!-- TODO: REQ-003E-pc OQ-002 — PDF 内每页各有唯一 Drawing No/Name，但只创建一条 Drawing 记录，自动填入哪页的值未定。备选：① 第一页；② 置信度最高页；③ 不自动填入，由用户点击列表某行选择。 -->
>
> **对 UI 的影响**：若最终选择方案 ③，§3.5 识别结果列表**必须从只读改为可选中**（增加行选中态、hover 态、选中高亮），本文档 §3.5 与设计原则 3（只读核对）需整体重写。**在 OQ-002 关闭前，§3.5 与 §3.7 的设计稿不应开始绘制。**

#### 状态（5 态）

| 状态 | 说明 |
|-----|------|
| 空 | 形态 A 未上传 / 识别失败：空且可编辑。形态 B：空且置灰 |
| 加载 | 形态 A 识别中：置灰 + placeholder `"Filling in after analysis…"` |
| 正常 | 形态 A 识别完成：填入 AI 建议值，可编辑 |
| 错误 | 提交时 Drawing Code 重复 → 边框红 + 下方红字 `"Code already exists."`（仅形态 A 可能触发） |
| 极端数据 | AI 返回的 Drawing Code 超长（> 50 字符，超出 data-contract 的 String(50)）→ 截断至 50 并在字段下方橙色提示 <!-- TODO: 需求未规定 AI 返回值超长时的处理，此为 UI 提案，见 §12 --> |

#### 权限可见性

| UI 元素 | 设计人员 | 其他角色 |
|--------|:-------:|:-------:|
| Drawing Code / Name 输入框 | 显示（Others 下 disabled） | 隐藏（弹窗整体不可达） |

---

### 3.8 [Submit] 按钮状态

**关联 AC**：AC-003E-005、AC-003E-009、AC-003E-015

| Submission Type | 条件 | 按钮状态 |
|----------------|-----|---------|
| Shop Drawing | AI 识别中（PENDING / PROCESSING） | `disabled`，灰色 |
| Shop Drawing | 必填项未填满（Code / Name / Category / Approver） | `disabled`，灰色 |
| Shop Drawing | 必填项已满，AI 已完成或已降级 | 可点击，主色蓝 |
| **Others** | **文件未上传完成，或 Category / Approver 未填** | `disabled`，灰色 |
| **Others** | **文件上传完成 + Category + Approver 已填** | **可点击**（**不校验 Code / Name**） |

- 提交中：按钮进入 `loading` 态，文案 `"Submitting…"`，防重复点击
- 提交成功：弹窗关闭 → 列表刷新 → Snackbar
- **Snackbar 文案必须为单数**（`"Drawing uploaded successfully."`），不得出现 `"X drawings created"` 之类的复数/批量措辞——两类提交均只创建一条记录

---

### 3.9 Drawing 详情页 — AI 识别结果区域（新增）

**关联 Story**：US-003E-001
**关联 AC**：AC-003E-012、AC-003E-013、AC-003E-017

> **归属说明**：Drawing 详情页主体属于 REQ-003A-pc，本节仅定义 REQ-003E 在其上**新增**的"AI 识别结果"区域。

#### 布局

```
┌──────────────────────────────────────────────────────┐
│  AI Recognition Result                               │
│  5 pages detected                                    │
├───────┬───────────────────────┬──────────────────────┤
│ Page  │ Drawing No            │ Drawing Name         │
├───────┼───────────────────────┼──────────────────────┤
│ Page 1│ ARCH-001              │ Ground Floor Plan    │
│ Page 2│ ARCH-002              │ First Floor Plan     │
└───────┴───────────────────────┴──────────────────────┘
```

- 内容与上传弹窗 §3.5 识别列表一致（**不含缩略图列**——详情页可直接预览 PDF，缩略图冗余）
- 只读，无操作

#### 显隐规则

| Drawing 记录来源 | `ai_recognition_job_id` | 本区域 |
|----------------|:----------------------:|-------|
| Shop Drawing + AI 识别成功 | 有值 | **显示** |
| Shop Drawing + AI 失败降级手动填写 | 空 | **不显示** |
| Others | 空 | **不显示** |

> 判定依据统一为 `ai_recognition_job_id` 是否为空，**不依赖 `submission_type`**——降级手动上传的 Shop Drawing 同样无页信息，与 Others 表现一致。

#### 状态（5 态）

| 状态 | 说明 |
|-----|------|
| 空 | `ai_recognition_job_id` 为空 → 整块不渲染（不显示"暂无数据"占位，避免对 Others 造成"缺失感"） |
| 加载 | 详情页加载时随主体一同骨架屏（`el-skeleton`，3 行） |
| 正常 | 表格展示各页信息 |
| 错误 | 识别记录已被清理（OQ-004 保留期限到期）→ 显示 `"AI recognition record is no longer available."` <!-- TODO: OQ-004 保留期限未定，本态是否会发生取决于是否有清理策略，见 §12 --> |
| 极端数据 | 100 页 → 区域内最大高度 320px，超出纵向滚动 |

#### 权限可见性

| UI 元素 | 设计人员 | 内部审批人 | 项目管理人员 | Site Engineer |
|--------|:-------:|:---------:|:----------:|:------------:|
| AI 识别结果区域 | 显示 | <!-- TODO: 需求 §3 权限矩阵未定义详情页 AI 区域对各角色的可见性，见 §12 --> | 同左 | 同左 |

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
| **Others 下 Code/Name 禁用态背景** | `#F5F7FA` | Element UI disabled 底色 |
| 字段错误边框 / 文字 | `#F56C6C` | Element UI `--color-danger` |

> 状态色对照 ui-rules §3.3：识别中（处理中）→ 蓝 info；识别完成 → 绿 success；识别失败（可降级恢复）→ 橙 warning，**非红**。红色仅用于字段级校验失败（如 Code 重复）。

### 4.2 间距与字体

> 依 ui-rules §3.10.4：含文字组件行内联 `font-size` 等字体属性，不依赖跨表查询。

| 组件 | 属性 | 值 | 来源 |
|-----|------|----|------|
| 弹窗 | width | 600px（固定） | 新增 |
| 弹窗 | max-height | 80vh | 新增 |
| 弹窗内各字段组 | margin-bottom | 20px | Element UI 表单规范 |
| **Submission Type 单选组** | display | flex | 新增 |
| Submission Type 单选组 | width | 100%（`align-self: stretch`） | 新增 |
| Submission Type 单选组 | margin-bottom | 20px（同其它字段组） | 新增 |
| Submission Type 单选组 | flex-direction | row | 新增 |
| Submission Type 单选组 | justify-content | flex-start（两项左起排列，**非等分、非两端对齐**） | 新增 |
| Submission Type 单选组 | align-items | center | 新增 |
| Submission Type 单选组 | gap | 32px（两个单选项间距） | 新增 |
| Submission Type 单选项文字 | font-size | 14px（⚠️ 必须显式设置，禁止继承） | Element UI 正文 |
| Submission Type 单选项文字 | line-height | 22px | — |
| Submission Type 单选项文字 | font-weight | 400 | — |
| Submission Type 单选项文字 | color | `#303133`（`--color-text-primary`） | — |
| 识别结果列表区域 | margin-top | 16px | 新增 |
| 识别结果列表区域 | max-height | 240px | REQ-003E-pc §7.2 |
| 识别中 Loading 区 | height | 88px | 新增 |
| 识别中 Loading 区 | display | flex | 新增 |
| 识别中 Loading 区 | justify-content | center | 新增 |
| 识别中 Loading 区 | align-items | center | 新增 |
| 识别中 Loading 区 | flex-direction | row | 新增 |
| 识别中 Loading 区 | gap | 8px（Spinner 与文案间距） | 新增 |
| 识别中 Loading 区文案 | font-size | 14px（⚠️ 显式设置） | — |
| 识别中 Loading 区文案 | line-height | 22px | — |
| 识别中 Loading 区文案 | color | `#606266`（`--color-text-regular`） | — |
| 缩略图 | width × height | 60px × 60px | REQ-003E-pc §7.2 |
| 降级提示横幅 | padding | 12px 16px | 与 Element UI Alert 对齐 |
| 降级提示横幅文字 | font-size | 14px（⚠️ 显式设置） | — |
| 降级提示横幅文字 | line-height | 22px | — |
| 降级提示横幅文字 | color | `#E6A23C` | — |
| 表头文字 | font-size | 12px（⚠️ 显式设置） | — |
| 表头文字 | line-height | 18px | — |
| 表头文字 | font-weight | 500 | — |
| 表头文字 | color | `#909399`（`--color-text-placeholder`） | — |
| 表格行文字 | font-size | 14px（⚠️ 显式设置） | Element UI 正文默认 |
| 表格行文字 | line-height | 22px | — |
| 表格行文字 | font-weight | 400 | — |
| 表格行文字 | color | `#303133`（`--color-text-primary`） | — |
| 字段标签 | font-size | 14px（⚠️ 显式设置） | — |
| 字段标签 | line-height | 22px | — |
| 字段标签 | font-weight | 400 | — |
| 字段标签 | color | `#606266`（`--color-text-regular`） | — |
| 辅助小字（AI 自动填入说明 / Not applicable for Others） | font-size | 12px（⚠️ 显式设置） | — |
| 辅助小字 | line-height | 18px | — |
| 辅助小字 | color | `#909399`（`--color-text-placeholder`） | — |
| 字段级错误文字 | font-size | 12px（⚠️ 显式设置） | — |
| 字段级错误文字 | line-height | 18px | — |
| 字段级错误文字 | color | `#F56C6C`（`--color-danger`） | — |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 弹窗宽 600px，完整展示 |
| Tablet | 768~1280px | 弹窗宽 540px；识别列表 Drawing No 列压缩至 160px，Drawing Name 仍 `flex: 1` |
| Mobile | < 768px | **不支持**（PC 专属，见 REQ-003E-pc §9.4） |

---

## 6. 微交互与动效

| 动效 | 时长 | 缓动 | 备注 |
|-----|------|------|------|
| 弹窗弹出 | 200ms | ease-out | Element UI Dialog 默认 |
| **切换 Submission Type → 识别列表区显隐** | 200ms | ease-in-out | `opacity` + `height` 过渡；避免弹窗高度瞬间跳变 |
| **切换 Submission Type → Code/Name 置灰** | 150ms | ease-out | `background-color` 过渡 |
| 识别结果列表出现 | 300ms | ease-in-out | `opacity: 0 → 1` + `height: 0 → auto` |
| 降级提示横幅出现 | 200ms | ease-out | `opacity: 0 → 1` |
| Loading Spinner | 持续旋转 | linear | — |
| 缩略图 Lightbox 展开 | 200ms | ease-out | `opacity` + `scale: 0.95 → 1` |

---

## 7. 无障碍（A11y）

**目标等级**：WCAG AA

| 要素 | 规范 |
|-----|------|
| 弹窗打开时焦点 | 自动移至第一个可交互元素（Drawing Description 输入框） |
| Tab 顺序 | Description → Category → **Submission Type** → 文件上传区 → Drawing Code → Drawing Name → Version Note → Internal Approver → Cancel → Submit |
| **Submission Type 单选** | `role="radiogroup"` + `aria-label="Submission Type"`；方向键切换选项（REQ-003E-pc §9.3）；空格键选中 |
| **Others 下 Code/Name 禁用态** | `aria-disabled="true"` + `aria-describedby` 指向 `"Not applicable for Others."` 小字，**不可仅靠灰色传达不可用** |
| Loading 状态 | `aria-busy="true"` + `aria-label="Analysing drawing pages"` |
| 识别结果列表 | `role="table"`，表头 `<th scope="col">`；未识别单元格 `aria-label="Not recognised"`，**不可仅靠橙色 ⚠ 传达** |
| 缩略图 | `alt="Page N preview"`；Lightbox `aria-modal="true"`，`Esc` 关闭 |
| 错误提示 | `role="alert"` + `aria-live="assertive"` 实时播报 |
| 识别完成通知 | `aria-live="polite"` 播报 `"X pages detected"` |
| 颜色对比度 | 正文 ≥ 4.5:1；橙色 `#E6A23C` 在 `#FDF6EC` 底上需校验 <!-- TODO: 该组合对比度约 2.4:1，低于 AA。需 UI 确认是否加深橙色文字或改用深橙 `#B88230`，见 §12 --> |

---

## 8. 复用与新建组件清单

| 组件 | 来源 | 备注 |
|-----|------|------|
| Modal Dialog（`el-dialog`） | 现有 DS（Element UI） | 复用 REQ-003A 上传弹窗结构 |
| 表单输入框（`el-input`） | 现有 DS | Description / Code / Name / Version Note |
| 下拉单选（`el-select`） | 现有 DS | Category / Internal Approver |
| **单选组（`el-radio-group` + `el-radio`）** | 现有 DS | **Submission Type 新增使用**；若 OQ-008 落地致选项 ≥ 5 需改 `el-select` |
| 文件上传（`el-upload`） | 现有 DS | 限制 `accept="application/pdf"`、≤ 50MB |
| 进度条（`el-progress`） | 现有 DS | 文件上传进度 |
| Loading（`el-loading`） | 现有 DS | AI 识别中 Spinner |
| 提示横幅（`el-alert`） | 现有 DS | 识别成功绿条 / 部分未识别橙条 / 降级失败橙条 |
| 数据表格（`el-table`） | 现有 DS | 识别结果列表；**禁用 selection 列**（只读核对） |
| Tooltip（`el-tooltip`） | 现有 DS | 超长文本截断悬浮全文 |
| Skeleton（`el-skeleton`） | 现有 DS | 详情页 AI 区域加载态 |
| Snackbar（`el-message`） | 现有 DS | 提交成功提示 |
| **缩略图 Lightbox** | **新建** | 60×60 缩略图 → 全屏预览。建议组件名 `PdfPageLightbox`；可评估复用 `el-image` 的 `preview-src-list` 能力 |
| **AI 识别结果只读表格** | **新建（跨页复用）** | 上传弹窗 §3.5 与详情页 §3.9 共用，差异仅在是否含缩略图列。建议组件名 `AiRecognitionTable`，以 `showThumbnail` prop 区分 |

---

## 9. 文案规范

> 业务术语须来自 glossary.md §1。**本需求引入的 3 个术语尚未入库**，见下方标注与 §12。

<!-- TODO: 术语 "Shop Drawing"、"Others"、"Submission Type" 未在 glossary.md 定义（全仓零命中）。下方中文译法为 UI 提案，须经 PM 确认并回写 glossary 后方可定稿 -->

| 场景 | 文案（zh-CN） | 文案（en） |
|-----|------------|----------|
| 弹窗标题 | 上传新图纸 | Upload New Drawing |
| Drawing Description 标签 | 图纸描述 | Drawing Description |
| Category 标签 | 图纸分类 | Category |
| **Submission Type 标签** | 提交类型 <!-- TODO: 待 glossary 确认 --> | Submission Type |
| **Submission Type 选项 1** | 深化图 <!-- TODO: 待 glossary 确认译法 --> | Shop Drawing |
| **Submission Type 选项 2** | 其它 <!-- TODO: 待 glossary 确认译法 --> | Others |
| Drawing File 标签 | 图纸文件 | Drawing File |
| 文件上传区空态 | 点击或拖拽上传 PDF 文件（最大 50MB） | Click or drag PDF file here (max 50MB) |
| 文件上传中 | 上传中… {n}% | Uploading... {n}% |
| AI 识别中 | 正在识别图纸页信息… | Analysing drawing pages… |
| Code/Name 识别中 placeholder | 识别完成后自动填入… | Filling in after analysis… |
| 识别成功汇总 | 共识别到 {n} 页。图纸信息已自动填入，请核对确认。 | {n} pages detected. Drawing information has been pre-filled below. Please review and confirm. |
| 部分页未识别 | 部分页面未能完整识别，请核对下方图纸信息。 | Some pages could not be fully analysed. Please verify the drawing information below. |
| 未识别单元格 | — | — |
| 识别失败降级横幅 | 无法自动识别该图纸。请手动填写图纸信息，或重新上传文件。 | Unable to analyse the drawing automatically. Please enter the drawing information manually, or re-upload the file. |
| [Re-upload] 按钮 | 重新上传 | Re-upload |
| AI 自动填入说明小字 | 由 AI 识别自动填入，可修改。 | Auto-filled from AI analysis. You may edit if needed. |
| **Others 下 Code/Name 说明小字** | 其它类型文件不适用。 | Not applicable for Others. |
| 非 PDF 错误 | 仅支持 PDF 格式。 | Only PDF format is supported. |
| 超 50MB 错误 | 文件大小超出 50MB 限制。 | File size exceeds the 50MB limit. |
| 上传失败（网络） | 上传失败，请重试。 | Upload failed. Please try again. |
| PDF 损坏（Others 页数解析失败） | 无法读取该 PDF，请检查文件。 | Unable to read this PDF. Please check the file. |
| Drawing Code 重复 | 该图纸编号已存在。 | Code already exists. |
| Drawing Code 为空 | 图纸编号不能为空。 | Drawing Code is required. |
| Drawing Name 为空 | 图纸名称不能为空。 | Drawing Name is required. |
| 切换类型二次确认 | 切换提交类型将清空已上传的文件，是否继续？ | Changing submission type will clear the uploaded file. Continue? |
| 提交中按钮 | 提交中… | Submitting… |
| 提交成功 Snackbar | 图纸上传成功，等待内部审批。 | Drawing uploaded successfully. Pending internal approval. |
| 详情页 AI 区域标题 | AI 识别结果 | AI Recognition Result |
| 详情页 AI 区域页数 | 共 {n} 页 | {n} pages detected |
| 详情页 AI 记录已清理 | AI 识别记录已不可用。 | AI recognition record is no longer available. |

> 文案为 UI 临时提案，**待文案 / UX 写作 review**。

---

## 10. Figma 与原型链接

- Upload New Drawing 弹窗 — 形态 A（Shop Drawing，5 态）：<!-- 填写 Figma Frame 链接 -->
- Upload New Drawing 弹窗 — 形态 B（Others，5 态）：<!-- 填写 Figma Frame 链接 -->
- Submission Type 切换交互（含二次确认）：<!-- 填写 Figma Frame 链接 -->
- AI 识别结果列表（正常 / 部分未识别 / 极端数据）：<!-- 填写 Figma Frame 链接 -->
- AI 识别失败降级横幅：<!-- 填写 Figma Frame 链接 -->
- 缩略图 Lightbox：<!-- 填写 Figma Frame 链接 -->
- Drawing 详情页 AI 识别结果区域：<!-- 填写 Figma Frame 链接 -->
- 交互原型：<!-- 填写链接 -->

---

## 11. AC 覆盖检查表

| AC ID | 对应章节 | 对应状态 / 交互 | 覆盖? |
|------|---------|---------------|------|
| AC-003E-001 | §3.4 + §3.7 | Shop Drawing 上传完成 → Loading 态 + Code/Name 置灰 + Submit 置灰 | ✅ |
| AC-003E-002 | §3.5 + §3.7 | DONE → 列表展示 + 汇总文案 + 输入框预填并恢复可编辑 | ⚠️ 部分 |
| AC-003E-003 | §3.7 | 编辑已预填的 Code → 列表只读不联动 | ✅ |
| AC-003E-004 | §3.5 错误态 | 部分页未识别 → `—` + ⚠ + 追加橙色提示 | ✅ |
| AC-003E-005 | §3.8 | 提交成功 → Snackbar 单数文案；§3.5 列表无 Checkbox，视觉不暗示批量 | ✅ |
| AC-003E-006 | §3.6 | FAILED → 橙色横幅 + Code/Name 恢复可编辑 + [Re-upload] | ✅ |
| AC-003E-007 | §3.6 + §3.7 | 降级后手动填写 → Submit 可点 | ✅ |
| AC-003E-008 | §3.6 关键交互 | [Re-upload] → 清空文件 → 回到 §3.4 加载态 | ✅ |
| AC-003E-009 | §3.8 | PENDING / PROCESSING → Submit 置灰 + 输入框不可编辑 | ✅ |
| AC-003E-010 | §3.3 错误态 | 选择 .dwg → 拒绝 + `"Only PDF format is supported."` + 上传区保持清空 | ✅ |
| AC-003E-011 | §3.3 错误态 | > 50MB → 拒绝 + 超限提示 + 上传区保持清空 | ✅ |
| AC-003E-012 | §3.9 显隐规则 | `ai_recognition_job_id` 有值 → 详情页展示 AI 识别结果区域 | ✅ |
| AC-003E-013 | §3.9 显隐规则 + 空态 | 降级手动上传 → `ai_recognition_job_id` 为空 → 整块不渲染 | ✅ |
| AC-003E-014 | §3.2 + §3.5 空态 + §3.7 | 选 Others → 无 Loading、无识别列表、Code/Name 空且置灰、Submit 可用 | ✅ |
| AC-003E-015 | §3.8 Others 行 | Others 提交不校验 Code/Name → 创建成功 → Snackbar 单数 | ✅ |
| AC-003E-016 | （后端规则，无 UI 体现） | `page_count` 由系统记录且**明确不在页面展示**——本 AC 的 UI 语义即"不渲染任何页码"，已在 §3.1 形态 B 布局与 §3.5 空态体现 | N/A（部分） |
| AC-003E-017 | §3.9 显隐规则 + 空态 | Others → `ai_recognition_job_id` 为空 → 详情页无 AI 区域 | ✅ |
| AC-003E-018 | §3.2 关键交互 | 切换类型 → 清空文件 + 识别列表消失 + Code/Name 清空置灰 | ✅ |
| AC-003E-019 | §3.2 默认态 + §3.3 空态 | 弹窗打开 → 默认选中 Shop Drawing + 上传区即刻可用 + Code/Name 空可编辑 | ✅ |

> **AC-003E-002 标记为"部分覆盖"**：列表展示、汇总文案、输入框恢复可编辑均已设计；但"自动填入 AI 建议值"取哪一页的值受 OQ-002 阻塞，无法完整设计。见 §3.7 与 §12。

---

## 12. 待定问题（Open Questions）

> 引用自 REQ-003E-pc §16 中影响 UI 的项，并附本文档生成过程中自发现的问题。

### 12.1 来自需求 §16

| OQ ID | 问题 | 影响 UI 哪部分 |
|------|------|--------------|
| **OQ-002** | 🚫 **阻塞**：多页各有不同 Drawing No/Name 时，Drawing Code 输入框自动填入哪页的值？ | §3.7 自动填入行为无法设计。**若选方案 ③（用户点列表某行选择），§3.5 必须从只读表格改为可选中表格**（新增行 hover / 选中态 / 单选控件），设计原则 3 需重写。**OQ-002 关闭前 §3.5 / §3.7 设计稿不应开工。** |
| OQ-001 | AI 识别超时阈值？ | §3.4 Loading 最长持续时间；超时后转 §3.6 的时机 |
| OQ-003 | 单文件最大页数？超出如何处理？ | §3.5 极端数据态（当前按 100 页设计滚动容器） |
| OQ-004 | AIRecognitionJob 保留多久？是否定期清理？ | §3.9 错误态"记录已不可用"是否真实存在——若永不清理，该态可删 |
| OQ-008 | Others 后期拆分为哪些细分类型？ | §3.2 组件选型：选项 ≥ 5 时 `el-radio-group` 须改 `el-select`，默认值表达方式随之改变 |

### 12.2 本文档自发现

| # | 问题 | 影响 UI 哪部分 | 建议 |
|---|------|--------------|------|
| U-01 | **术语未入 glossary**："Shop Drawing"、"Others"、"Submission Type" 全仓零命中，无标准中文译法 | §9 全部中文文案 | PM 确认译法后回写 glossary.md §1；本文档中文列现为提案 |
| U-02 | **已上传文件后切换 Submission Type 是否二次确认**？需求 §6.1 仅规定"清空"，未规定是否提示 | §3.2 关键交互；§9 二次确认文案 | 建议二次确认——清空已上传的 50MB 文件代价高，误触不可撤销 |
| U-03 | **橙色对比度可能不达 AA**：`#E6A23C` 文字在 `#FDF6EC` 底上约 2.4:1，低于 AA 要求的 4.5:1 | §3.5 未识别单元格、§3.6 降级横幅、§7 A11y | 建议文字改用深橙 `#B88230`（约 4.6:1），或保持橙底但文字用 `#303133` + 橙色图标 |
| U-04 | **AI 返回的 Drawing Code 超长如何处理**？data-contract 限 String(50)，需求未规定 AI 返回超长值的行为 | §3.7 极端数据态 | 建议截断至 50 + 字段下方橙色提示，提醒用户核对 |
| U-05 | **详情页 AI 识别结果区域对各角色的可见性未定义**：需求 §3 权限矩阵只覆盖上传弹窗，未涉及详情页 | §3.9 权限可见性矩阵（现为 TODO） | 需 PM 补充；建议与 Drawing 详情页其余字段的可见性保持一致 |
| U-06 | **Others 的 Code/Name 保留字段还是隐藏字段**？需求说"显示为空"，本文档理解为"保留字段 + 置灰 + 内容空" | §3.7 Others 只读态；§3.1 形态 B 布局 | 已按"保留 + 置灰"设计（避免切换时表单结构跳动）；若 PM 本意是隐藏，§3.1 形态 B 与 §3.7 需改 |

---

## 13. 验收标准（本文档自身）

UI 设计交付物完成的判定：

- [ ] Figma 稿覆盖 §3 所有页面/组件（共 9 个：弹窗整体形态 A/B、Submission Type 选择器、文件上传区、识别中状态、识别结果列表、降级区域、Code/Name 输入框、Submit 按钮、详情页 AI 区域）
- [ ] 弹窗**两种形态各有 5 态设计稿**（形态 A：空/加载/正常/错误/极端；形态 B：空/加载/正常/错误/极端）
- [ ] Submission Type 切换的过渡动效有交互走查原型（含弹窗高度变化、二次确认）
- [ ] §11 AC 覆盖检查表所有项标记 ✅ 或显式 N/A —— **当前 AC-003E-002 为 ⚠️ 部分，须待 OQ-002 关闭后补齐**
- [ ] §12.1 OQ-002 已关闭，§3.5 / §3.7 据结论定稿
- [ ] §12.2 U-01 术语已回写 glossary.md，§9 中文文案定稿并经 UX 写作 review
- [ ] §12.2 U-03 橙色对比度已实测并达 WCAG AA
- [ ] 新建组件（`PdfPageLightbox`、`AiRecognitionTable`）已在 §8 记录并与 DS 负责人对齐
- [ ] §12 所有 OQ 已显式列出，未自行编造解决方案

> ⚠️ **本文档不具备开工条件**：OQ-002 未关闭，§3.5 与 §3.7 存在结构性分歧（只读表格 vs 可选中表格）。建议先关闭 OQ-002 再启动设计稿绘制，否则返工面覆盖识别列表与输入框两大核心区域。

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-24 | agent | 初稿，从 REQ-003E-pc@0.3.1 派生 |
| 0.2.0 | 2026-07-16 | agent | 同步至 REQ-003E-pc@0.4.2：**新增 Submission Type 选择器（§3.2）**、弹窗双形态布局（§3.1）、Others 路径下 Code/Name 空且置灰（§3.7）、Drawing 详情页 AI 识别结果区域（§3.9，覆盖此前遗漏的 AC-012/013）；AC 覆盖由 13 条扩至 19 条。**补齐 ui-rules 强制章节**：此前缺失的 §8 复用与新建组件清单、§9 文案规范、§11 AC 覆盖检查表、§12 待定问题、§13 验收标准，并为每个组件补全 5 态与权限可见性矩阵。**修正两处硬错误**：文件大小上限 100MB → 50MB（需求为 50MB）、Drawing Description 由"必填"改为"选填"（需求为选填）。新增自发现问题 U-01 ~ U-06 |
