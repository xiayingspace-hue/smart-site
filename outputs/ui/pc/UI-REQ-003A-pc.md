---
doc_type: ui_spec
req_id: REQ-003A-pc
version: 0.1.0
status: draft
generated_from: REQ-003A-pc.md@0.1.0
generated_at: 2026-0#### Filter Search Popover

- 触发方式：点击顶部 [Filter Search] 按钮，以 **Popover（浮层）** 方式弹出，不推开表格
- Popover 位置：紧贴按钮下方左对齐展开
- 字段：Drawing Code（`el-input`）、Drawing Name（`el-input`）、Category（`el-select` 单选）、Status（`el-select` 单选）
- 底部按钮：[Cancel]（次要）、[Search]（主要）
  - [Search]：执行查询，Popover **关闭**，列表展示筛选结果，按钮文案更新为"Filter Search (n)"（n = 生效条件数）
  - [Cancel]：清空所有字段，Popover 关闭，列表恢复无筛选，按钮文案恢复为"Filter Search"
  - 点击 Popover 外部区域：Popover 关闭，已填字段**保留**但不执行查询，按钮计数不更新
- **按钮文案计数规则**：统计已填写（非空）的字段数量 n；n = 0 时显示"Filter Search"；n > 0 时显示"Filter Search (n)"er: ""
---

# UI 设计说明：PC 端 — 图纸上传与审批发起

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-003A-pc.md（主）、glossary.md（术语）
> - 输出引用：Figma 链接、交互原型链接
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003A-pc @ v0.1.0 |
| 覆盖用户故事 | US-003A-001 |
| 覆盖 AC | AC-003A-001 ~ AC-003A-008 |
| 上次同步时间 | 2026-05-04 |

> ⚠️ 当 requirement.md 版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

让 Drawing 团队成员在 PC 端高效完成图纸上传（新建 / 新版本）并发起审批，操作路径清晰、反馈即时、错误可恢复。

### 1.2 设计原则

1. **流程可见**：上传过程有进度反馈，用户随时知道当前状态。
2. **错误就地呈现**：校验失败信息紧贴对应字段，不打断操作流。
3. **防误触**：Pending 状态下 [Upload New Version] 按钮置灰，避免重复提交。
4. **完成后引导**：上传成功后 Snackbar 主动引导管理员完成 SE 分配，降低遗漏风险。
5. **最小化输入**：上传新版本时复用主记录字段，只要求填写变化部分。

---

## 2. 信息架构

### 2.1 页面层级

```
侧边栏
└── Drawing Management（一级菜单）
    └── Drawing Masterlist（二级菜单）→ 图纸列表页
        ├── [Filter Search] Popover（点击按钮触发）
        ├── 图纸列表表格
        │   └── Actions 列
        │       ├── [Upload New Version]（非 Pending 时可点）
        │       ├── [History]（见 REQ-003C-pc）
        │       └── [Assign SE]（见 REQ-003D-pc）
        ├── Upload New Drawing 侧滑面板（右侧滑出）
        └── Upload New Version 侧滑面板（右侧滑出）
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发场景 |
|---------|-------|---------|
| 侧边栏 Drawing Management → Drawing Masterlist | 图纸列表页 | 用户点击二级菜单 |
| 图纸列表页右上角 [+ Upload Drawing] | Upload New Drawing 侧滑面板 | 新建图纸 |
| 列表行 Actions 列 [Upload New Version] | Upload New Version 侧滑面板 | 为已有图纸上传新版本（Status ≠ PENDING_APPROVAL） |

---

## 3. 页面 / 组件清单

### 3.1 图纸列表页（Drawing Masterlist）

**关联 Story**：US-003A-001
**关联 AC**：AC-003A-001、AC-003A-005

#### 布局

```
┌─────────────────────────────────────────────────────────┐
│  顶部操作区                                               │
│  [⚙ Filter Search]  或  [⚙ Filter Search (2)]           │
│                                          [+ Upload Drawing]│
├─────────────────────────────────────────────────────────┤
│  表格                                                     │
│  Drawing Code ↕ | Name | Category | Version | Status    │
│                 | Confirmed | Last Updated ↕ | Actions   │
│                                    （Actions 示意）        │
│  ARCH-001  ...  V2  ACTIVE  3/5  2026-05-01  [↑][⏱][👤] │
└─────────────────────────────────────────────────────────┘

（点击 Filter Search 后浮出 Popover）
┌──────────────────────────────┐
│  Drawing Code  [___________] │
│  Drawing Name  [___________] │
│  Category      [▼ Select   ] │
│  Status        [▼ Select   ] │
│                [Cancel] [Search] │
└──────────────────────────────┘
```

#### 表格列定义

| 列名 | 内容 | 排序 | 备注 |
|-----|------|:---:|------|
| Drawing Code | 图纸编号文本 | ✅ | — |
| Name | 图纸名称 | ❌ | — |
| Category | 分类文本 | ❌ | — |
| Current Version | 版本号（如 V3） | ❌ | — |
| Status | `<DrawingStatusTag>` 颜色标签 | ❌ | 见 §4.1 颜色语义 |
| Confirmed | 已确认数/总分配数（如 3/5） | ❌ | Status = ACTIVE 时显示 x/y；其他状态显示 — |
| Last Updated | 日期时间 | ✅ 默认倒序 | — |
| Actions | 操作按钮组 | ❌ | 见下方 |

#### Actions 列按钮

- 三个操作均使用 **图标按钮**（`el-button` size="mini" circle + icon），不显示文字标签
- 每个按钮附带 `el-tooltip`，鼠标悬浮时显示操作名称（置灰时另有专属 Tooltip）
- 三个按钮水平排列，间距 4px

| 按钮 | 图标 | Element UI icon | 状态规则 | Tooltip（正常） |
|-----|------|----------------|---------|----------------|
| Upload New Version | 上传↑ | `el-icon-upload2` | Status = PENDING_APPROVAL → `disabled`（置灰） | "Upload New Version" / 置灰时："A version is pending approval. Cannot upload a new version." |
| History | 时钟 | `el-icon-time` | 始终可点 | "History" |
| Assign SE | 用户 | `el-icon-user` | 始终可点 | "Assign SE" |

#### 搜索面板

- 默认**收起**；点击 [🔍 Search] 按钮展开
- 展开时面板向下推开表格（非浮层覆盖）
- 字段：Drawing Code（`el-input`）、Drawing Name（`el-input`）、Category（`el-select` 单选）、Status（`el-select` 单选）
- 底部按钮：[Cancel]（次要）、[Search]（主要）
  - [Search]：执行查询，查询完成后面板**自动收起**，列表展示筛选结果
  - [Cancel]：清空所有筛选条件，收起面板，列表恢复无筛选

#### 关键交互

| 交互 | 触发 | 反馈 |
|-----|------|-----|
| 点击 [Filter Search] 打开 Popover | 点击顶部按钮 | Popover 浮出，按钮保持高亮状态 |
| 执行搜索（有条件） | 点击 Popover 内 [Search] | Loading 态 → Popover 关闭 → 列表更新 → 按钮显示"Filter Search (n)" |
| 执行搜索（无条件） | 点击 Popover 内 [Search] | 等同全量查询，Popover 关闭，按钮显示"Filter Search" |
| 取消搜索 | 点击 Popover 内 [Cancel] | 字段清空，Popover 关闭，列表恢复无筛选，按钮恢复"Filter Search" |
| 点击 Popover 外部 | 点击页面其他区域 | Popover 关闭，已填字段保留，不触发查询，按钮计数不变 |
| 排序 | 点击可排序列表头 | 列表重新排序，列头显示排序方向箭头 |
| 点击 [Upload New Version]（置灰） | 鼠标悬浮 | Tooltip 提示"当前版本正在审批中，不可上传新版本" |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 项目内无图纸，或搜索无结果 | 表格区居中展示空状态插图 + 说明文字；无图纸时显示上传引导按钮 | "暂无图纸" / "No drawings found. Try adjusting your search." |
| 加载 | 首次进入或执行搜索时 | 表格行 Skeleton 加载占位；[Search] 按钮 loading 态 | — |
| 正常 | 有数据 | 表格正常展示 | — |
| 错误 | 列表接口 5xx / 网络断开 | 表格区展示错误提示 + [重试] 按钮 | "加载失败，请重试" |
| 极端数据 | 单行图纸名称极长（> 50 字符） | 图纸名称单行截断加 Tooltip；Actions 三个图标按钮固定宽度，不受内容压缩 | — |

#### 权限可见性

| UI 元素 | Drawing 团队成员 | 审批人 | 项目管理人员 | Site Engineer |
|--------|:--------------:|:-----:|:----------:|:------------:|
| [+ Upload Drawing] | ✅ 显示 | ❌ 隐藏 | ❌ 隐藏 | — |
| [Upload New Version] | ✅ 显示（按状态 enabled/disabled） | ❌ 隐藏 | ❌ 隐藏 | — |
| [History] | ✅ 显示 | ✅ 显示 | ✅ 显示 | — |
| [Assign SE] | ❌ 隐藏 | ❌ 隐藏 | ✅ 显示 | — |

---

### 3.2 Upload New Drawing 侧滑面板

**关联 Story**：US-003A-001
**关联 AC**：AC-003A-001、AC-003A-002、AC-003A-003、AC-003A-004、AC-003A-007、AC-003A-008

#### 布局

> 以 `el-drawer` 从**页面右侧**滑入；面板宽度根据内容自适应（无固定值），最小宽度 720px，最大宽度 1180px；背景遮罩覆盖页面其余区域。

**表单栅格规则**：
- 默认每行 **2 列**（`el-row :gutter="24"` + `el-col :span="12"`）
- 以下字段内容较长，单独占满 **1 列（整行）**（`el-col :span="24"`）：
  - Description（textarea）
  - Drawing File 上传区
  - Version Note（textarea）
  - 进度条

```
┌────────────────────────────────────────────────────────┐
│  Upload New Drawing                                 [×] │
├────────────────────────────────────────────────────────┤
│  Drawing Code *          │  Drawing Name *             │
│  [____________________]  │  [____________________]     │
│                          │                             │
│  Category *              │  Approver *                 │
│  [▼ Select____________]  │  [▼ Select____________]     │
│                                                         │
│  Description                               （整行）      │
│  [__________________________________________________]   │
│                                                         │
│  Drawing File *                            （整行）      │
│  ┌──────────────────────────────────────────────────┐  │
│  │   拖拽文件至此，或 点击上传                          │  │
│  │   支持 PDF / DWG / DXF / PNG / JPG，≤ 50MB        │  │
│  └──────────────────────────────────────────────────┘  │
│  [进度条 0→100%]（上传中显示，整行）                      │
│                                                         │
│  Version Note                              （整行）      │
│  [__________________________________________________]   │
├────────────────────────────────────────────────────────┤
│                                    [Cancel]  [Submit]   │
└────────────────────────────────────────────────────────┘
```

#### 字段说明

| 字段 | 组件 | 必填 | 校验时机 | 错误提示位置 |
|-----|------|:---:|---------|------------|
| Drawing Code | `el-input` | ✅ | 提交时（前端非空 + 服务端唯一） | 字段正下方内联 |
| Drawing Name | `el-input` | ✅ | 提交时 | 字段正下方内联 |
| Category | `el-select` | ✅ | 提交时 | 字段正下方内联 |
| Description | `el-input` type=textarea | ❌ | — | — |
| Drawing File | 拖拽上传区（`el-upload`） | ✅ | 选文件即时校验 | 上传区下方内联 |
| Version Note | `el-input` type=textarea | ❌ | — | — |
| Approver | `el-select` | ✅ | 提交时 | 字段正下方内联 |

#### 关键交互

| 交互 | 触发 | 反馈 |
|-----|------|-----|
| 选择不支持格式的文件 | 文件选择 / 拖入 | 即时拒绝，上传区下方红色提示"不支持该格式，请上传 PDF/DWG/DXF/PNG/JPG" |
| 选择超过 50MB 的文件 | 文件选择 / 拖入 | 即时拒绝，提示"文件过大，最大支持 50MB" |
| 选择合法文件 | 文件选择 | 上传区展示文件名 + 大小，可替换 |
| 点击 [Submit]（有必填未填） | 点击 | 各必填字段下方展示内联错误，按钮保持可用 |
| 点击 [Submit]（校验通过） | 点击 | 进度条出现（0%），[Submit] 禁用，上传开始 |
| 上传中 | — | `el-progress` 从 0→100%，[Submit] 和 [Cancel] 均禁用 |
| 上传成功 | 接口返回成功 | 侧滑面板关闭，列表刷新，Snackbar 展示（见 §9 文案） |
| 上传失败 / Code 重复 | 接口返回错误 | 进度条消失；Code 重复时字段内联错误；其他错误全局 Toast；侧滑面板保留 |
| 点击 [Cancel] | 点击 | 侧滑面板关闭，表单重置，不保存任何内容 |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 |
|-----|---------|---------|
| 空（初始） | 侧滑面板刚打开 | 所有字段为空，进度条隐藏 |
| 加载（Approver/Category 选项获取中） | 选项接口未返回 | 对应 `el-select` 内 loading 态 |
| 正常 | 字段已填写 | 输入态 |
| 错误（校验失败） | 提交时字段不合法 | 字段边框红色 + 下方红色提示文字 |
| 上传中 | 文件上传进行中 | 进度条展示，所有操作按钮禁用 |

---

### 3.3 Upload New Version 侧滑面板

**关联 Story**：US-003A-001
**关联 AC**：AC-003A-001、AC-003A-003、AC-003A-004、AC-003A-005、AC-003A-006、AC-003A-007、AC-003A-009、AC-003A-010

#### 布局

> 同 §3.2，以 `el-drawer` 从**页面右侧**滑入；宽度自适应内容，最小 720px，最大 1180px。

**表单栅格规则**（同 §3.2）：
- 默认每行 **2 列**（`el-col :span="12"`）
- 以下字段占满 **整行**（`el-col :span="24"`）：
  - Description（只读文本，内容可能较长）
  - Drawing File 上传区
  - Version Note（textarea）
  - 进度条

```
┌────────────────────────────────────────────────────────┐
│  Upload New Version                                 [×] │
├────────────────────────────────────────────────────────┤
│  Current Version  V{n}（只读）   │                      │
│  ⓘ 提交后系统版本自动递增为 V{n+1}│                      │
│                                                         │
│  Drawing Code *              │  Approver *             │
│  [ARCH-001-R2_____________]  │  [▼ Select____________] │
│  ⓘ 可修改以与图纸文件版本号一致│                         │
│                                                         │
│  Drawing Name（只读）         │  Category（只读）        │
│  Architectural Plan          │  Structural             │
│                                                         │
│  Description（只读）                       （整行）      │
│  Ground floor layout — includes structural annotations  │
│                                                         │
│  Drawing File *                            （整行）      │
│  ┌──────────────────────────────────────────────────┐  │
│  │   拖拽文件至此，或 点击上传                          │  │
│  │   支持 PDF / DWG / DXF / PNG / JPG，≤ 50MB        │  │
│  └──────────────────────────────────────────────────┘  │
│  [进度条 0→100%]（上传中显示，整行）                      │
│                                                         │
│  Version Note                              （整行）      │
│  [__________________________________________________]   │
├────────────────────────────────────────────────────────┤
│                                    [Cancel]  [Submit]   │
└────────────────────────────────────────────────────────┘
```

#### 字段说明

| 字段 | 可编辑 | 说明 |
|-----|:-----:|------|
| Current Version | ❌ 只读 | 灰色文字；小字提示"系统版本独立递增，与 Drawing Code 无关" |
| Drawing Code | ✅ 可编辑（预填当前值） | 上传人可改为与图纸文件内版本号一致的编号；字段下方灰色小字提示"修改后将更新图纸主记录的 Code"；提交时唯一性由服务端校验，重复时字段下方内联报错 |
| Drawing Name | ❌ 只读 | 展示主记录值，灰色不可编辑，供上传人核对 |
| Category | ❌ 只读 | 展示主记录值，灰色不可编辑，供上传人核对 |
| Description | ❌ 只读 | 展示主记录值，灰色不可编辑，供上传人核对；无值时显示 — |
| Drawing File | ✅ 必填 | 同新建侧滑面板约束 |
| Version Note | ✅ 选填 | — |
| Approver | ✅ 必填 | 同新建侧滑面板约束 |

#### 状态（5 态）

与 §3.2 Upload New Drawing 侧滑面板一致，省略重复说明。

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色语义

| 用途 | Token | 默认值 | Element UI type |
|-----|-------|-------|----------------|
| Status: PENDING_APPROVAL | `--color-status-warning` | `#E6A23C` | `el-tag` type=`warning` |
| Status: ACTIVE | `--color-status-success` | `#67C23A` | `el-tag` type=`success` |
| Status: REJECTED | `--color-status-danger` | `#F56C6C` | `el-tag` type=`danger` |
| Status: DEPRECATED | `--color-status-info` | `#909399` | `el-tag` type=`info` |
| 主操作按钮（Upload / Submit） | `--color-primary` | `#409EFF` | `el-button` type=`primary` |
| 次要操作按钮（Cancel） | — | 默认 | `el-button`（默认） |
| 表单校验错误 | `--color-danger` | `#F56C6C` | Element UI 内置 |

### 4.2 间距与栅格

| 组件 | 属性 | 值 | 来源 |
|-----|------|----|------|
| 侧滑面板宽度 | width | 自适应内容；min-width: 480px，max-width: 680px | 新建；`el-drawer` size 属性留空由内容撑开 |
| 侧滑面板内表单行间距 | margin-bottom | 20px | Element UI Form item 默认 |
| 文件上传区 | height | 120px | 新建 |
| 文件上传区 | border | 1px dashed `#DCDFE6` | 新建 |
| 文件上传区（hover） | border-color | `#409EFF` | 新建 |
| Filter Search 按钮容器 | display | flex | 新建 |
| Filter Search 按钮容器 | height | 32px | 新建 |
| Filter Search 按钮容器 | padding | 5px 8px | 新建 |
| Filter Search 按钮容器 | align-items | center | 新建 |
| Filter Search 按钮容器 | gap | 4px | 新建 |
| Filter Search 按钮容器 | border-radius | 2px | 新建 |
| Filter Search 按钮容器 | border | 1px solid `var(--border-color, #D8E2F0)` | 新建 |
| Filter Search 按钮容器 | background | `var(--Vertical-Menu-White, #FFF)` | 新建 |
| Filter Search Popover | width | 360px | 新建 |
| Filter Search Popover | padding | 16px | 新建 |
| Filter Search Popover 字段行 | display | flex，flex-direction: column，gap: 12px | 新建 |
| Filter Search Popover 按钮区 | justify-content | flex-end，gap: 8px | 新建 |
| 表格 Actions 列 | display | flex，gap: 4px，align-items: center | 新建 |
| DrawingStatusTag | font-size | 12px（⚠️ 必须显式设置） | 新建 |
| DrawingStatusTag | line-height | 20px | 新建 |
| DrawingStatusTag | font-weight | 400 | 新建 |
| 表格行 | font-size | 14px（⚠️ 必须显式设置） | 继承 Element UI Table |
| 上传提示文字 | font-size | 14px（⚠️ 必须显式设置） | 新建 |
| 上传提示文字 | color | `#909399` | 新建 |
| 只读版本号文字 | font-size | 14px（⚠️ 必须显式设置） | 新建 |
| 只读版本号文字 | color | `#909399` | 新建 |
| 版本提示小字 | font-size | 12px（⚠️ 必须显式设置） | 新建 |
| 版本提示小字 | color | `#C0C4CC` | 新建 |
| 进度条 | width | 100%，margin-top: 8px | Element UI Progress 默认 |

---

### 4.3 字体排版（侧滑面板）

#### 面板标题

| 属性 | 值 |
|------|----|
| color | `var(--text-color-primary-dark, #344050)` |
| font-family | Arial |
| font-size | 18px |
| font-weight | 700 |
| font-style | normal |
| line-height | 26px（144.444%） |
| 设计 Token 标注 | `500 Medium / H2-18-large` |

#### 表单项 Label

| 属性 | 值 |
|------|----|
| color | `var(--text-color-primary, #5E6E82)` |
| font-family | Arial |
| font-size | 14px |
| font-weight | 400 |
| font-style | normal |
| line-height | 22px（157.143%） |
| letter-spacing | -0.01px |
| 设计 Token 标注 | `400 Regular / B1-14-Base` |

#### 输入框有内容时的文字

| 属性 | 值 |
|------|----|
| color | `var(--text-color-primary-dark, #344050)` |
| font-family | "Helvetica Neue" |
| font-size | 14px |
| font-weight | 500 |
| font-style | normal |
| line-height | 22px（157.143%） |
| letter-spacing | -0.01px |

#### Filter Search 按钮文字

| 属性 | 值 |
|------|----|
| color | `var(--text-color-primary-dark, #344050)` |
| font-feature-settings | `'liga' off, 'clig' off` |
| font-family | "Helvetica Neue" |
| font-size | 12px |
| font-weight | 400 |
| font-style | normal |
| line-height | 24px（200%） |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 全功能展示，表格全列可见 |
| Tablet | 768~1280px | 部分列可隐藏（Category、Version 列可折叠）；侧滑面板宽度收窄至 min-width（480px）或 90vw 取小值 |
| Mobile | < 768px | **不支持**（PC 专属功能，移动端不渲染入口） |

---

## 6. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| Filter Search Popover 打开/关闭 | fade + 向下位移 4px | 150ms | ease-out / ease-in |
| Filter Search 按钮计数变化 | 文案切换（无动效，即时更新） | — | — |
| 侧滑面板打开 | 从右侧 slide-in（`el-drawer` 默认 RTL 方向） | 300ms | ease |
| 侧滑面板关闭 | 向右 slide-out | 200ms | ease |
| 进度条增长 | `el-progress` 平滑动画 | 随实际进度 | linear |
| Snackbar 出现 | 从右上角 slide-in | 300ms | ease-out |
| Snackbar 消失 | fade-out | 200ms | ease-in |
| [Upload New Version] 置灰过渡 | opacity 变化 | 150ms | ease |

---

## 7. 无障碍（A11y）

- **WCAG 等级**：AA
- **颜色对比度**：正文（14px）≥ 4.5:1；状态 Tag 文字与背景 ≥ 3:1
- **键盘导航**：
  - 侧滑面板内所有表单字段支持 Tab 键顺序导航
  - 文件上传区支持 Enter/Space 触发文件选择对话框
  - [Submit] / [Cancel] 支持 Enter 确认、Esc 关闭侧滑面板
- **焦点可见**：所有可交互元素 focus 态显示 2px solid `#409EFF` outline
- **屏幕阅读器**：
  - 上传进度变化通过 `aria-valuenow` 通知
  - 表单错误通过 `role="alert"` 通知
  - 置灰按钮使用 `aria-disabled="true"` + Tooltip 说明原因

---

## 8. 复用与新建组件清单

| 组件 | 来源 | 备注 |
|-----|------|------|
| `el-table` | Element UI 现有 | 表格主体 |
| `el-button` | Element UI 现有 | 所有操作按钮 |
| `el-drawer` | Element UI 现有 | 上传侧滑面板容器（替代 el-dialog），从右侧滑入，宽度自适应 |
| `el-form` / `el-form-item` | Element UI 现有 | 侧滑面板内表单 |
| `el-input` | Element UI 现有 | 文本输入 |
| `el-select` | Element UI 现有 | 下拉选择 |
| `el-upload` | Element UI 现有 | 文件上传区，封装为 `DrawingFileUploader` |
| `el-progress` | Element UI 现有 | 上传进度条 |
| `el-tag` | Element UI 现有 | 状态标签，封装为 `DrawingStatusTag` |
| `el-tooltip` | Element UI 现有 | 按钮置灰提示、长文本截断提示 |
| `DrawingStatusTag` | **新建** | 封装状态→颜色映射逻辑，复用于多处 |
| `DrawingFileUploader` | **新建** | 封装格式/大小校验 + 进度条展示，复用于两个上传弹窗 |
| `DrawingSearchPanel` | **新建** | Filter Search 触发的 Popover，含条件计数逻辑 |

---

## 9. 文案规范

| 场景 | 文案（zh-CN） | 文案（en） |
|-----|-------------|----------|
| 列表页面标题 | 图纸清单 | Drawing Masterlist |
| 新建按钮 | + 上传图纸 | + Upload Drawing |
| 上传新版本按钮 | 上传新版本 | Upload New Version |
| 查看历史按钮 | 历史记录 | History |
| 分配 SE 按钮 | 分配 SE | Assign SE |
| Filter Search 按钮（无条件） | 筛选搜索 | Filter Search |
| Filter Search 按钮（有 n 个条件） | 筛选搜索 (n) | Filter Search (n) |
| 搜索 Popover - 执行按钮 | 搜索 | Search |
| 搜索 Popover - 取消按钮 | 取消 | Cancel |
| 列表空状态（无数据） | 暂无图纸 | No drawings yet. |
| 列表空状态（无搜索结果） | 未找到匹配的图纸，请调整搜索条件 | No drawings found. Try adjusting your search. |
| 列表加载失败 | 加载失败，请重试 | Failed to load. Please retry. |
| 置灰按钮 Tooltip | 当前版本正在审批中，不可上传新版本 | A version is pending approval. Cannot upload a new version. |
| Upload New Drawing 侧滑面板标题 | 上传新图纸 | Upload New Drawing |
| Upload New Version 侧滑面板标题 | 上传新版本 | Upload New Version |
| 当前版本只读提示 | 当前版本：V{n}，提交后新版本将为 V{n+1} | Current Version: V{n}. New version will be V{n+1}. |
| 文件上传区提示 | 拖拽文件至此，或 点击上传 | Drag file here, or click to upload |
| 文件上传区格式说明 | 支持 PDF / DWG / DXF / PNG / JPG，最大 50MB | Supports PDF / DWG / DXF / PNG / JPG, max 50MB |
| 文件格式错误 | 不支持该文件格式，请上传 PDF / DWG / DXF / PNG / JPG | Unsupported file type. Please upload PDF / DWG / DXF / PNG / JPG. |
| 文件大小超限 | 文件过大，最大支持 50MB | File too large. Max size is 50MB. |
| 弹窗 Submit 按钮 | 提交 | Submit |
| 弹窗 Cancel 按钮 | 取消 | Cancel |
| 上传成功 Snackbar（新建） | 图纸上传成功，等待审批。请记得为该图纸分配 Site Engineer，以便他们查阅。 | Drawing uploaded successfully. Pending approval. Remember to assign Site Engineers so they can view this drawing. |
| 上传成功 Snackbar（新版本） | 新版本上传成功，等待审批。 | New version uploaded successfully. Pending approval. |
| Drawing Code 重复错误 | 该 Drawing Code 已存在，请更换 | This Drawing Code already exists. Please use a different one. |
| 通用上传失败 Toast | 上传失败，请检查网络后重试 | Upload failed. Please check your network and retry. |

---

## 10. Figma 与原型链接

- Figma 设计稿：<!-- 填写图纸列表页 + 两个弹窗 Frame 链接 -->
- 交互原型：<!-- 填写可点击原型链接 -->
- 设计系统：<!-- 填写团队 Design System 链接 -->

---

## 11. AC 覆盖检查表

| AC ID | 对应页面/组件 | 对应状态/交互 | 覆盖? |
|------|------------|------------|------|
| AC-003A-001 | Upload New Drawing 弹窗 | 正常提交成功路径，Snackbar 提示，列表刷新 | ✅ |
| AC-003A-002 | Upload New Drawing 弹窗 | Drawing Code 字段下方内联错误（Code 重复） | ✅ |
| AC-003A-003 | DrawingFileUploader（上传区） | 选文件即时拒绝 + 错误提示（格式错误） | ✅ |
| AC-003A-004 | DrawingFileUploader（上传区） | 选文件即时拒绝 + 错误提示（大小超限） | ✅ |
| AC-003A-005 | 图纸列表 Actions 列 | [Upload New Version] 置灰态 + Tooltip | ✅ |
| AC-003A-006 | Upload New Drawing / Version 弹窗 | 提交成功副作用（描述于 Snackbar + 列表刷新） | ✅ |
| AC-003A-007 | DrawingFileUploader（进度条） | 上传中进度条展示 + [Submit] 禁用 | ✅ |
| AC-003A-008 | Upload New Drawing 弹窗 | Snackbar 文案含 SE 引导语 | ✅ |
| AC-003A-009 | Upload New Version 弹窗 | Drawing Code 字段预填可编辑，提交成功后列表 Code 更新，系统版本递增 | ✅ |
| AC-003A-010 | Upload New Version 弹窗 | Drawing Code 修改为已存在值时，字段下方内联报错，弹窗保留 | ✅ |

---

## 12. 待定问题（Open Questions）

| OQ ID | 问题 | 影响 UI 哪部分 |
|------|------|--------------|
| OQ-001（来自需求） | 文件大小上限未来是否放宽？ | 文件上传区说明文字、格式校验提示 |
| OQ-002（来自需求） | 是否支持批量上传多张图纸？ | 上传入口与弹窗结构（单文件 vs 多文件上传区） |
| OQ-UI-001 | 搜索面板是否需要"高级搜索"入口（更多筛选字段）？ | 搜索面板字段数 |
| OQ-UI-002 | Actions 列在图纸数量多时是否折叠为 [···] 下拉菜单？ | Actions 列布局（影响 REQ-003C/D 的按钮） |
| OQ-UI-003 | 弹窗宽度是否统一为 560px？ | 弹窗布局 |

---

## 13. 验收标准（本文档自身）

UI 设计交付物完成的判定：

- [ ] Figma 稿覆盖图纸列表页、Upload New Drawing 侧滑面板、Upload New Version 侧滑面板
- [ ] 每个页面/弹窗均有 5 态设计稿（空/加载/正常/错误/极端数据）
- [ ] §11 表中所有 AC 标记 ✅
- [ ] 移动端不适用（PC 专属，可忽略）
- [ ] DrawingStatusTag、DrawingFileUploader、DrawingSearchPanel 已列入 §8 新建清单并完成设计稿
- [ ] §12 所有 OQ 显式列出，未编造方案

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-04 | agent | 从 REQ-003A-pc.md@0.1.0 生成初稿 |
