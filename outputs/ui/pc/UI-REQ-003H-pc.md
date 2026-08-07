---
doc_type: ui_spec
req_id: REQ-003H-pc
version: 0.1.0
status: draft
generated_from: REQ-003H-pc@0.1.1
data_contract_ref: DATA-CONTRACT-REQ-003H-pc.md@0.1.0
generated_at: 2026-08-07
generator: ui-agent
owner: ""
---

# UI 设计说明：PC 端 — 免审批上传（已完成审批的图纸直接生效）

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> ⚠️ 本需求**不新增页面**，改动集中在一个弹窗的新增区域和两处已有页面的展示态。
> PM 已在 [REQ-003H-pc §8 页面简图](../../../requirements/pc/REQ-003H-pc.md) 提供布局，本文档**以简图为准**（[ui-rules §3.1](../../../rules/ui-rules.md)），不再从主流程推导页面。

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003H-pc @ v0.1.1 |
| 数据契约 | DATA-CONTRACT-REQ-003H-pc.md @ v0.1.0 |
| 页面布局来源 | REQ-003H-pc §8 页面简图（PM 已确认） |
| 覆盖 Story | US-003H-001、US-003H-002、US-003H-003、US-003H-004 |
| 覆盖 AC | AC-003H-001、002、003、004、008、010、011、012、013 |
| 上次同步时间 | 2026-08-07 |

---

## 1. 设计目标

### 1.1 核心目标

让持有专项权限的上传人在既有上传弹窗中多一个明确的路径选择，选中后一眼看懂"这张图会立刻生效、不再有人复核"；同时让管理人员在列表与版本历史中能一眼识别哪些版本是这样入库的。

### 1.2 设计原则

1. **默认路径不受任何影响**——无权限用户看到的弹窗与现状逐像素一致，有权限用户的默认选中项也是 Standard。这条捷径不能因为"顺手"而被误用
2. **风险必须可见且常驻**——选中 Pre-Approved 后警示文案常驻展示，不是 hover 提示、不是一次性 toast
3. **隐藏优于置灰**——无权限时 Approval Route 区域**整体不渲染**（[ui-rules §3.2.3](../../../rules/ui-rules.md) 默认策略），避免向无权用户暴露该功能存在
4. **"跳过"与"未到达"必须视觉可辨**——版本历史中 `⊘` 与 `—` 是两个不同语义，混用会让管理人员误以为版本还在等待审批
5. **对 SE 完全透明**——SE 视角看到的就是一份已生效的正式图纸，来源差异属管理侧信息，不下沉到现场

---

## 2. 信息架构

### 2.1 页面层级

```
图纸管理（既有模块）
├── 图纸管理列表页（REQ-003A-pc）
│   ├── Status 列 ················· ✏️ 本需求新增 Pre-approved 标识
│   ├── [+ Upload Drawing] → 新建图纸弹窗（REQ-003E-pc）
│   │   └── Approval Route 区域 ··· 🆕 本需求新增
│   └── Actions [History] → 版本历史抽屉（REQ-007C-pc）
│       └── 4 阶段卡片 + 步骤条 ··· ✏️ 本需求新增"跳过态"渲染分支
└── （系统设置）用户权限管理 ······· ✏️ 新增权限项 drawing:upload-approved（沿用既有权限管理 UI，无设计改动）
```

> 🆕 = 新增区域｜✏️ = 既有区域的新增展示分支

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发场景 |
|---------|-------|---------|
| 图纸列表页 [+ Upload Drawing] | 新建图纸弹窗（含 Approval Route 区域） | 上传人新建图纸；区域是否渲染取决于 `drawing:upload-approved` |
| 图纸列表 Actions 列 [History] | 版本历史抽屉 | 管理人员追溯某图纸各版本；免审批版本展开后呈现跳过态 |
| 图纸列表 Status 列 | 无跳转 | 列表浏览时直接识别免审批记录 |

> 本需求**不新增任何菜单入口、不新增路由**。

---

## 3. 页面/组件清单

### 3.1 新建图纸弹窗 — Approval Route 区域

**关联 Story**：US-003H-001、US-003H-002、US-003H-003
**关联 AC**：AC-003H-001、AC-003H-002、AC-003H-003
**布局来源**：[REQ-003H-pc §8.1](../../../requirements/pc/REQ-003H-pc.md)
**宿主弹窗**：[REQ-003E-pc §3.1](../../../requirements/pc/REQ-003E-pc.md)（本文档只描述新增区域，弹窗其余部分不重复定义）

**布局**：

区域插入在 **Internal Approver 选择器上方**、AI 识别结果区下方，作为弹窗表单流的最后一段决策。

```
┌─ Upload Drawing ────────────────────────────────── ✕ ─┐
│   … Description / Category / Submission Type …         │
│   … 文件上传区 / AI 识别结果列表 …                      │
│   … Drawing Code / Drawing Name …                      │
│                                                        │
│  ┌── Approval Route ─────────────────────────────┐    │ ← 🆕 新增区域
│  │  ◉ Standard Approval — internal then external │    │   （无权限时整段不渲染）
│  │  ○ Pre-Approved — already approved outside…   │    │
│  └───────────────────────────────────────────────┘    │
│   Internal Approver *  [ 请选择审批人        ▾ ]       │
│                              [ Cancel ]  [ Confirm ]   │
└────────────────────────────────────────────────────────┘
```

**字段展示**（字段定义引用 [DATA-CONTRACT §4.2](../../shared/DATA-CONTRACT-REQ-003H-pc.md)）：

| UI 元素 | 绑定字段 | 说明 |
|--------|---------|------|
| Approval Route 单选组 | `approvalRoute` | 枚举见 [DATA-CONTRACT §2.1](../../shared/DATA-CONTRACT-REQ-003H-pc.md)；默认 `STANDARD` |
| Internal Approver 选择器 | `approverId` | 既有字段；`PRE_APPROVED` 时隐藏且解除必填 |

**关键交互**：

| 交互 | 触发 | 反馈 |
|-----|------|-----|
| 区域渲染判定 | 弹窗打开时读取当前用户权限集 | 持有 `drawing:upload-approved` → 渲染整个区域；否则 **DOM 中不存在**（非置灰、非隐藏样式） |
| 选中 Pre-Approved | 点击第二个单选项 / 键盘方向键 | ① Internal Approver 整行隐藏（含 label、必填星号、错误提示）；② 已选审批人值清空；③ 必填校验解除；④ 警示文案出现并常驻 |
| 切回 Standard | 点击第一个单选项 | Internal Approver 整行恢复显示，必填校验恢复；**不自动回填**此前选过的审批人（避免用户误以为已选） |
| 提交（Pre-Approved） | 点击 [Confirm] | 按钮进 loading，禁用弹窗全部交互；后端同步执行 3–5 秒 |
| 提交成功 | 后端返回 `code = 0` | 弹窗关闭；Toast「Drawing created and activated」；列表刷新并定位到新记录 |
| 提交失败（QR） | 后端返回 `1003007009` | 弹窗**保持打开**，已填内容全部保留；Toast「QR generation failed, please retry」；[Confirm] 恢复可点，用户可当场重试，**文件无需重传** |
| 提交失败（权限被回收） | 后端返回 `1003003020` | 弹窗保持打开，内容保留；Toast 提示无权限；用户可改选 Standard 重新提交 |

**状态（必填 5 态）**：

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 不适用 | 单选组恒有两个选项且必有一个选中，不存在空态 | — |
| 加载 | 点击 [Confirm] 后等待后端（3–5 秒） | [Confirm] 内嵌 spinner + 文案变为 Processing…；弹窗遮罩阻断全部交互，含 ✕ 与 Cancel | Processing… |
| 正常 | 弹窗打开且用户有权限 | 单选组常态，默认选中 Standard | 见 §9 文案表 |
| 错误 | 后端返回错误码 | Toast（非字段级错误，因本区域无输入框可挂错误）；弹窗保持打开，内容保留 | 见 §9 文案表 |
| 极端数据 | 选项文案在 en 语境下较长（`Pre-Approved — already approved outside the system`） | 单选项文案**允许换行**，不截断、不省略号——用户必须读完整句才知道自己在选什么；区域高度自适应 | 全文展示 |

**权限可见性**（对应需求 [§5 权限矩阵](../../../requirements/pc/REQ-003H-pc.md)）：

| UI 元素 | 免审批上传人 | 设计人员 | 项目管理员 | 内部审批人 / DC / SE |
|--------|:-----------:|:-------:|:---------:|:-----------------:|
| Approval Route 区域整体 | 显示 | **不渲染** | **不渲染** | 不适用（无上传入口） |
| Standard 单选项 | 显示（默认选中） | 不渲染 | 不渲染 | — |
| Pre-Approved 单选项 | 显示 | 不渲染 | 不渲染 | — |
| 警示文案 | 选中 Pre-Approved 时显示 | 不渲染 | 不渲染 | — |
| Internal Approver 选择器 | Standard 时显示 / Pre-Approved 时隐藏 | 恒显示（必填） | 不适用 | — |

> ⚠️ **设计师注意**：无权限用户的弹窗**必须与本需求上线前完全一致**——不留空白占位、不留分隔线。这是 AC-003H-001 的验收点。

---

### 3.2 Approval Route 警示文案块

**关联 Story**：US-003H-001、US-003H-003
**关联 AC**：AC-003H-003
**布局来源**：[REQ-003H-pc §8.1 状态 B](../../../requirements/pc/REQ-003H-pc.md)

**布局**：

紧贴 Pre-Approved 单选项下方，与选项文字左对齐（缩进对齐到 radio 右侧文字起始位置），构成视觉从属关系。

```
│  ┌── Approval Route ─────────────────────────────┐
│  │  ○ Standard Approval — internal then external │
│  │  ◉ Pre-Approved — already approved outside…   │
│  │  ⚠ This drawing will take effect immediately  │  ← 常驻，不可关闭
│  │    without internal or external approval.     │
│  │    This action is recorded.                   │
│  └───────────────────────────────────────────────┘
```

**关键交互**：

| 交互 | 触发 | 反馈 |
|-----|------|-----|
| 出现 | 选中 Pre-Approved | 直接出现，**无淡入动效**——风险提示不做柔化处理 |
| 消失 | 切回 Standard | 直接消失 |
| 关闭 | **不提供** | 无关闭按钮，不可 dismiss |

**状态（必填 5 态）**：

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 不适用 | 文案为静态常量，不存在空态 | — |
| 加载 | 不适用 | 无数据依赖 | — |
| 正常 | Pre-Approved 选中 | warning 语义色文本 + ⚠ 图标；背景为 warning 浅底色块 | 见 §9 |
| 错误 | 不适用 | 无数据依赖 | — |
| 极端数据 | 中英文案长度差异 | 文本块自动换行，容器高度自适应；**不设最大高度、不滚动** | 全文展示 |

> **设计说明**：用 warning 而非 danger 语义色。这是一个**授权用户的合法操作**，不是错误——danger 会让日常使用该功能的资料员产生警报疲劳，反而削弱提示效力。

---

### 3.3 版本历史抽屉 — 免审批版本展开态

**关联 Story**：US-003H-004
**关联 AC**：AC-003H-010
**布局来源**：[REQ-003H-pc §8.2](../../../requirements/pc/REQ-003H-pc.md)
**宿主组件**：[REQ-007C-pc §7.3](../../../requirements/pc/REQ-007C-pc.md) 4 阶段生命周期视图（本文档只描述新增的渲染分支）

**布局**：

沿用既有 2×2 卡片网格 + 底部步骤条，**结构不变**，仅 ②③ 卡片与步骤条的渲染内容按 `approvalRoute` 分支。

```
▼ V1  ✅ Approved  (Pre-approved)                       [Collapse]
┌────────────────────────────────────────────────────────────────┐
│  ① Uploaded                     ② Internal Approval            │
│  ┌──────────────────────┐       ┌──────────────────────┐       │
│  │ 📄 arch001-v1.pdf    │       │ ⊘ Skipped —          │       │
│  │ Uploader: 张三       │       │   pre-approved upload│       │
│  │ 2026-08-06 10:00     │       │ Uploaded as already  │       │
│  │        [Preview]     │       │ approved by 张三     │       │
│  │        [Download]    │       │                      │       │
│  └──────────────────────┘       └──────────────────────┘       │
│  ③ External Approval            ④ Signed Version               │
│  ┌──────────────────────┐       ┌──────────────────────┐       │
│  │ ⊘ Skipped —          │       │ 📄 arch001-v1.pdf    │       │
│  │   pre-approved upload│       │ ⓘ Same file as       │       │
│  │ （不显示 Mark Result）│       │   uploaded           │       │
│  │                      │       │        [Preview]     │       │
│  │                      │       │        [Download]    │       │
│  └──────────────────────┘       └──────────────────────┘       │
│  📄 Uploaded ✓ → 🔍 Internal ⊘ → 🌐 External ⊘ → ✍️ Signed ✓  │
└────────────────────────────────────────────────────────────────┘
```

**字段展示**（引用 [DATA-CONTRACT §1.1](../../shared/DATA-CONTRACT-REQ-003H-pc.md)）：

| 卡片 | 展示字段 | 数据来源 |
|-----|---------|---------|
| ① Uploaded | `fileName`、`uploaderName`、`uploadTime`、`fileSize` | 既有字段 |
| ② / ③ | 无数据字段，静态文案 + `uploaderName` | 渲染分支由 `approvalRoute` 决定 |
| ④ Signed Version | `fileName`、`uploadTime`、`fileSize` | `signedFileUrl` 与 `fileUrl` 同源，故复用 ① 的文件元信息 |

**关键交互**：

| 交互 | 触发 | 反馈 |
|-----|------|-----|
| 渲染分支判定 | 展开版本行 | `approvalRoute = PRE_APPROVED` → 跳过态；否则沿用 REQ-007C 既有 5 种分支 |
| ③ 卡片 [Mark Result] | — | **不渲染**。该版本已是终态，DC 无可执行动作（即便当前用户是已配置 DC） |
| ④ [Preview] | 点击 | 预览 `signedFileUrl` |
| ④ [Download] | 点击 | 优先 `pdfWithQrUrl`，缺失时降级 `signedFileUrl`（沿用 REQ-007C §7.5） |

**状态（必填 5 态）**：

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 不适用 | 免审批版本必然有文件，不存在空态 | — |
| 加载 | 抽屉打开、版本列表请求中 | 沿用 REQ-007C 既有骨架屏 | — |
| 正常 | 展开 `PRE_APPROVED` 版本 | 如上图；②③ 灰色 disabled 视觉层级 | 见 §9 |
| 错误 | 版本详情接口失败 | 沿用 REQ-007C 既有错误态 | — |
| 极端数据 | 上传人姓名超长 | ② 卡片 `approved by {name}` 单行截断 + tooltip 显示全名 | — |

**权限可见性**：

| UI 元素 | 管理员 / Drawing 团队 / 设计人员 / 内部审批人 / DC | Site Engineer |
|--------|:---------------------------------------------:|:------------:|
| 4 阶段卡片（含跳过态） | 显示 | **不适用**（SE 无 `drawing:history` 权限，看不到版本历史） |
| ③ 卡片 [Mark Result] | **隐藏**（该版本无此动作） | — |

---

### 3.4 图纸列表 Status 列 — Pre-approved 标识

**关联 Story**：US-003H-004
**关联 AC**：AC-003H-011
**布局来源**：[REQ-003H-pc §8.3](../../../requirements/pc/REQ-003H-pc.md)
**宿主页面**：[REQ-003A-pc §5.1](../../../requirements/pc/REQ-003A-pc.md) 图纸管理列表页

**布局**：

```
│ Description            │ Ver │ Status                  │ Actions      │
│ ───────────────────────┼─────┼─────────────────────────┼───────────── │
│ Level 3 slab rebar     │ V3  │ 🟢 Active (A)           │ [View] […]   │  ← 走两级审批
│ Basement waterproofing │ V1  │ 🟢 Active  Pre-approved │ [View] […]   │  ← 免审批入库
│ Roof truss layout      │ V2  │ ⏳ Pending External     │ [View] […]   │
```

**关键交互**：

| 交互 | 触发 | 反馈 |
|-----|------|-----|
| 标识渲染 | 列表渲染 | `approvalRoute = PRE_APPROVED` 且当前用户非 SE → 状态标签右侧附加灰色小字 `Pre-approved` |
| 与结果代码互斥 | — | 免审批版本无外部审批结果，**不附加** `(A)/(B)/(D)`；两者同一位置，只出现其一 |
| 悬停 | hover 标识 | Tooltip：`Uploaded as already approved. No internal or external approval in system.` |

**状态（必填 5 态）**：

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 列表无数据 | 沿用 REQ-003A 既有空态 | — |
| 加载 | 列表请求中 | 沿用既有骨架屏 | — |
| 正常 | 有免审批记录 | `🟢 Active` + 灰色小字 `Pre-approved` | 见 §9 |
| 错误 | 列表接口失败 | 沿用既有错误态 | — |
| 极端数据 | 窄屏（1280px）下 Status 列拥挤 | 标识**不换行、不截断**；Status 列最小宽度增至 160px，必要时挤压 Description 列 | — |

**权限可见性**：

| UI 元素 | 管理员 / Drawing 团队 / 设计人员 / 内部审批人 / DC | Site Engineer |
|--------|:---------------------------------------------:|:------------:|
| `Pre-approved` 标识 | 显示 | **隐藏** |
| `🟢 Active` 状态标签本体 | 显示 | 显示（与正常图纸完全一致） |

> **设计说明**：SE 侧隐藏标识与 [REQ-014 FB-006](../../../requirements/pc/REQ-014-pc.md)（SE 视角隐藏系统版本号）方向一致——来源与版本机制属管理侧信息，不下沉到现场作业视图。

---

## 4. 设计令牌（Design Tokens）

> 引用现有 Design System，本节列出**新增或覆盖**的部分。按 [DEC-007](../../../background/key-decisions.md)，本文档只写语义名，不硬编码具体值。

### 4.1 颜色语义

| 用途 | Token | 说明 |
|-----|-------|------|
| 警示文案块 文字 | `warning` | 授权用户的合法高风险操作，非错误，故不用 `danger`（理由见 §3.2） |
| 警示文案块 背景 | `warning-light` | 浅底色块，与弹窗白底区分 |
| 警示文案块 图标 ⚠ | `warning` | 与文字同色 |
| `Pre-approved` 列表标识 | `text-secondary` | 灰色小字，辅助信息层级，不与状态标签争夺注意力 |
| ②③ 跳过态卡片 文字 | `text-disabled` | 表达"该阶段不会发生" |
| ②③ 跳过态卡片 图标 ⊘ | `text-disabled` | 同上 |
| 步骤条 跳过态 | `text-disabled` | 与"未到达"态同色但**图标不同**（⊘ vs —），区分靠图标而非颜色 |

> ⚠️ **不得仅靠颜色传递语义**（[ui-rules §3.8 A11y](../../../rules/ui-rules.md)）：跳过态与未到达态颜色相同，必须依靠 `⊘` / `—` 图标区分，且两者均配有文字说明。

### 4.2 间距与栅格

| 组件 | 属性 | 值 | 来源 |
|-----|------|----|------|
| Approval Route 区域 | 与上方 Drawing Name 字段的间距 | `spacing-lg` | 新增，与弹窗内其他字段组间距一致 |
| Approval Route 区域 | 与下方 Internal Approver 的间距 | `spacing-lg` | 新增 |
| Approval Route 单选组 | `flex-direction` | `column`（两选项纵向排列） | 新增 |
| Approval Route 单选组 | 选项间距 `gap` | `spacing-sm` | 新增 |
| Approval Route 单选组 | `align-items` | `flex-start`（文案可能换行，顶对齐） | 新增 |
| Approval Route 单选组 | `justify-content` | `flex-start` | 新增 |
| Approval Route 单选组 | 宽度 | `100%`（撑满弹窗表单区） | 新增 |
| 单选项文案 | `font-size` | `font-size-base`（⚠️ 必须显式设置，禁止继承浏览器默认） | 沿用 DS |
| 单选项文案 | `line-height` | `line-height-base` | 沿用 DS |
| 单选项文案 | `font-weight` | `font-weight-regular` | 沿用 DS |
| 单选项文案 | `color` | `text-primary` | 沿用 DS |
| 警示文案块 | 左缩进 | 对齐到 radio 右侧文字起始位置 | 新增，构成视觉从属 |
| 警示文案块 | 内边距 | `spacing-sm` | 新增 |
| 警示文案块 | `font-size` | `font-size-sm`（⚠️ 必须显式设置） | 沿用 DS |
| 警示文案块 | `line-height` | `line-height-base` | 沿用 DS |
| 警示文案块 | `font-weight` | `font-weight-regular` | 沿用 DS |
| `Pre-approved` 标识 | `font-size` | `font-size-sm`（⚠️ 必须显式设置） | 沿用 DS |
| `Pre-approved` 标识 | `font-weight` | `font-weight-regular` | 沿用 DS |
| `Pre-approved` 标识 | 与状态标签间距 | `spacing-xs` | 新增 |
| 列表 Status 列 | `min-width` | 160px（原宽度容纳不下"Active + Pre-approved"） | 覆盖 REQ-003A 既有值 |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 完整布局。上传弹窗固定宽度，Approval Route 单选项单行展示 |
| Tablet | 768~1280px | 不适用 |
| Mobile | < 768px | 不适用 |

> 端支持范围取自 [background/tech-stack.md §1](../../../background/tech-stack.md)：PC 管理端最低分辨率 1280×720px，不支持移动端。本需求为 `pc/` 目录需求，**本期仅 Desktop**。
>
> 在最低分辨率 1280px 下需重点验证：列表 Status 列同时容纳 `🟢 Active` 与 `Pre-approved` 不折行（见 §3.4 极端数据态）。

---

## 6. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| 切换到 Pre-Approved，Internal Approver 隐藏 | 无动效，直接隐藏 | 0 | — |
| 警示文案出现 / 消失 | **无淡入淡出** | 0 | — |
| [Confirm] 进入 loading | spinner 常规旋转 | 沿用 DS | 沿用 DS |
| 提交成功后弹窗关闭 | 沿用 DS 弹窗关闭动效 | 沿用 DS | 沿用 DS |
| 版本历史行展开 | 沿用 REQ-007C 既有展开动效 | 沿用 | 沿用 |

> **为什么风险提示不做动效**：淡入会让警示文案显得"轻"，且在快速切换单选项时产生闪烁。风险信息应当是即时、确定、静止的。

---

## 7. 无障碍（A11y）

- **WCAG 等级**：AA（项目基线）
- **颜色对比度**：警示文案 warning 色于 warning-light 背景上 ≥ 4.5:1；`Pre-approved` 灰色小字于白底 ≥ 4.5:1；⊘ 跳过态 disabled 色 ≥ 3:1（非正文，按大字/图形标准）
- **键盘导航**（需求 §10.3 明确的基线外要求）：
  - Approval Route 单选组支持 `Tab` 聚焦进入，`↑` `↓` `←` `→` 在两选项间切换
  - 单选组作为一个 tab stop（radiogroup 语义），不是两个
  - 切换到 Pre-Approved 后 Internal Approver 移出 tab 序列（因已隐藏），焦点顺序自然跳至 [Cancel]
- **焦点可见**：单选项 focus 态使用 DS 标准 focus ring，不得移除
- **屏幕阅读器**：
  - 单选组容器 `role="radiogroup"` + `aria-label="Approval Route"`
  - 警示文案通过 `aria-describedby` 关联到 Pre-Approved 单选项（需求 §10.3 明确要求），使读屏在选中时朗读风险说明
  - 警示文案块 `role="alert"` **不加**——它是常驻说明而非动态告警，加 alert 会导致每次切换都打断朗读
  - `Pre-approved` 列表标识加 `aria-label="Pre-approved upload"`，避免读屏只读出截断的视觉文本
  - ⊘ 跳过态图标为装饰性，`aria-hidden="true"`，语义由相邻文字 `Skipped — pre-approved upload` 承载

---

## 8. 复用与新建组件清单

| 组件 | 来源 | 备注 |
|-----|------|------|
| 单选组（Approval Route） | 现有 DS | 标准 radio group，无需业务包装 |
| 警示文案块 | 现有 DS | 使用 DS 的 inline alert / notice 组件，warning 语义，去掉关闭按钮 |
| `Pre-approved` 标识 | 现有 DS | 复用列表中既有的灰色辅助文本样式，与 `(A)/(B)/(D)` 结果代码同一渲染位 |
| ⊘ 跳过态卡片 | **既有组件新增分支** | REQ-007C 4 阶段卡片组件内新增一种渲染分支，不新建组件 |
| 步骤条跳过态 | **既有组件新增分支** | REQ-007C 步骤条组件新增 `⊘ skipped` 态，与既有 `✓ / ⏳ / ✕ / —` 并列 |

> **本需求不新建任何组件**。两处"新增分支"需要 REQ-007C 对应组件开放新的状态入参，属既有组件的能力扩展，需与该组件维护方确认。
>
> <!-- NOTE: README 声称 outputs/ui/shared/UI-COMPONENT-REGISTRY.md 是"生成新文档前必查"，但该文件及整个 outputs/ui/shared/ 目录当前不存在，本节的"现有 DS"判断依据的是既有 UI 文档中的实际用法，无注册表可对照。 -->

---

## 9. 文案规范

| 场景 | 文案(zh-CN) | 文案(en) |
|-----|-------------|----------|
| 区域标题 | 审批路径 | Approval Route |
| 选项 1 | 标准审批 — 先内部后外部 | Standard Approval — internal then external |
| 选项 2 | 免审批 — 已在系统外完成审批 | Pre-Approved — already approved outside the system |
| 警示文案 | 该图纸将立即生效，不经过内部与外部审批。此操作已被记录。 | This drawing will take effect immediately without internal or external approval. This action is recorded. |
| 提交成功 Toast | 图纸已创建并生效 | Drawing created and activated |
| QR 失败 Toast | 二维码生成失败，请重试 | QR generation failed, please retry |
| 无权限 Toast | 您没有免审批上传权限 | You do not have permission to upload a pre-approved drawing |
| 非新建图纸 Toast | 免审批上传仅适用于新建图纸 | Pre-approved upload is only allowed when creating a new drawing |
| ②③ 卡片主文案 | 已跳过 — 免审批上传 | Skipped — pre-approved upload |
| ② 卡片副文案 | 由 {uploaderName} 以已审批状态上传 | Uploaded as already approved by {uploaderName} |
| ④ 卡片标注 | 与上传文件为同一份 | Same file as uploaded |
| 列表标识 | 免审批 | Pre-approved |
| 列表标识 Tooltip | 以已审批状态上传，系统内无内部与外部审批记录 | Uploaded as already approved. No internal or external approval in system. |

> 国际化：中英双语，沿用现有体系（[background/tech-stack.md §5](../../../background/tech-stack.md)，默认语言 English）。
>
> **术语一致性**：`Pre-approved` / `Pre-Approved` 两种写法有明确分工——单选项标签用 `Pre-Approved`（与枚举值 `PRE_APPROVED` 对应，作为路径专名），列表与卡片中的状态描述用 `Pre-approved`（普通形容词用法）。<!-- NOTE: glossary.md 当前为空模板，本项目尚无术语表可对照；建议将 Pre-Approved / Standard Approval 登记为业务术语。 -->

---

## 10. Figma 与原型链接

- Figma 设计稿：<!-- TODO: 待设计输出后补充。需覆盖：上传弹窗 Approval Route 两态（Standard / Pre-Approved）、无权限用户弹窗（验证与现状一致）、版本历史跳过态展开、列表 Pre-approved 标识（1280px 窄屏） -->
- 交互原型：<!-- TODO: 待设计输出后补充 -->
- 设计系统：PC 端设计令牌（`design-tokens-pc.json`，见 [DEC-004](../../../background/key-decisions.md)）

---

## 11. AC 覆盖检查表

| AC ID | 对应页面/组件 | 对应状态/交互 | 覆盖 |
|------|------------|------------|:---:|
| AC-003H-001 | §3.1 Approval Route 区域 | 权限可见性表：无权限时 DOM 不渲染 | ✅ |
| AC-003H-002 | §3.1 Approval Route 区域 | 关键交互：区域渲染判定；默认选中 Standard | ✅ |
| AC-003H-003 | §3.1 + §3.2 警示文案块 | 关键交互：选中 Pre-Approved 后隐藏审批人、解除必填、警示常驻 | ✅ |
| AC-003H-004 | §3.1 | 关键交互：提交成功 → Toast + 列表刷新 | ✅ |
| AC-003H-005 | — | 后端行为，无 UI 体现 | N/A |
| AC-003H-006 | §3.1 | 关键交互：提交失败（1003003021）→ Toast | ✅ |
| AC-003H-007 | — | 接口层拦截，前端仅呈现 Toast（已在 §3.1 覆盖） | N/A |
| AC-003H-008 | §3.1 | 关键交互：QR 失败 → 弹窗保持打开、内容保留、可重试 | ✅ |
| AC-003H-009 | §3.1 | 权限可见性表 + Standard 路径交互不变 | ✅ |
| AC-003H-010 | §3.3 版本历史跳过态 | 布局 + 渲染分支判定 + 步骤条 ⊘ | ✅ |
| AC-003H-011 | §3.4 列表 Status 列 | 标识渲染 + 与结果代码互斥 + SE 隐藏 | ✅ |
| AC-003H-012 | §3.3 ④ Signed Version 卡片 | [Download] 优先 `pdfWithQrUrl` | ✅ |
| AC-003H-013 | §3.1 | 关键交互：提交失败（权限被回收）→ 弹窗保持打开、内容保留、可改选 Standard | ✅ |

**覆盖率**：11 / 13 有 UI 体现，2 条标 N/A（纯后端行为）。

---

## 12. 待定问题（Open Questions）

> 引用自 [REQ-003H-pc §17](../../../requirements/pc/REQ-003H-pc.md) 中影响 UI 的项，以及本文档生成过程中发现的新问题。

| OQ ID | 问题 | 影响 UI 哪部分 |
|------|------|--------------|
| OQ-001 | 是否提供选填的审批凭证补录入口 | 若采纳，§3.3 版本历史 ②③ 卡片需从"纯跳过态"改为"可补录态"，并新增补录弹窗。<!-- TODO: 等待 OQ-001 解决 --> |
| OQ-002 | 列表导出 Excel 是否包含 `approvalRoute` 列 | 若采纳，导出列配置需增加该列；不影响页面渲染。<!-- TODO: 等待 OQ-002 解决 --> |
| OQ-004 | `drawing:upload-approved` 授予是否需审批流 | 若采纳，权限管理页需新增审批流 UI。本文档假设管理员单方勾选（沿用既有权限管理界面，无设计改动）。<!-- TODO: 等待 OQ-004 解决 --> |
| OQ-UI-001 | **本文档新发现**：REQ-007C 的 4 阶段卡片与步骤条组件是否支持外部传入新的状态分支？若组件内部状态枚举写死为 5 种，新增 ⊘ 跳过态需改造组件本身 | §3.3 全节实现方式；需与 REQ-007C 组件维护方确认 |
| OQ-UI-002 | **本文档新发现**：1280px 最低分辨率下 Status 列加宽至 160px 后，Description 列被挤压是否可接受？还是应让 `Pre-approved` 标识改为图标 + Tooltip 形式 | §3.4 极端数据态、§4.2 列宽令牌 |

---

## 13. 验收标准（本文档自身）

UI 设计交付物完成的判定：

- [ ] Figma 稿覆盖 §3 全部 4 个页面/组件
- [ ] 每个页面/组件均有 5 态设计稿（不适用的态显式标注"不适用"及理由）
- [ ] **无权限用户的上传弹窗单独出稿**，与上线前版本逐像素比对，验证无残留空白（AC-003H-001）
- [ ] 1280px 窄屏下列表 Status 列出稿，验证标识不折行（AC-003H-011）
- [ ] 版本历史跳过态与既有 5 种分支并排出稿，验证 `⊘` 与 `—` 视觉可辨
- [ ] 每个 AC 都能在 §11 表中标记 ✅ 或有明确 N/A 理由
- [ ] 与现有 DS 的差异已列入 §8 并经评审
- [ ] 所有 OQ 在 §12 显式列出，未编造方案

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-08-07 | ui-agent | 初稿：基于 REQ-003H-pc §8 页面简图生成。定义 Approval Route 区域、警示文案块、版本历史跳过态、列表 Pre-approved 标识共 4 个页面/组件；新增 OQ-UI-001（REQ-007C 组件是否支持新状态分支）、OQ-UI-002（窄屏列宽取舍） |
