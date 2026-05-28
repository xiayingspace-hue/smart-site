---
doc_type: ui_spec
req_id: REQ-003A-pc
version: 0.5.0
status: draft
generated_from: REQ-003A-pc@0.5.0
generated_at: 2026-05-27
owner: ""
---

# UI 设计说明：PC 端 — 图纸管理列表页

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-003A-pc.md（主）、REQ-003-shared（业务规则与 API）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc § 3.2
> - 不重复定义数据字段（参见 data-contract.md）
> - 新建图纸上传 UI 见 UI-REQ-003E-pc；上传新版本 UI 见 UI-REQ-003F-pc

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003A-pc @ v0.5.0 |
| 覆盖用户故事 | US-003A-LIST-001、US-003A-LIST-002 |
| 覆盖 AC | AC-003A-005、AC-003A-005B、AC-003A-011、AC-003A-011B、AC-003A-011C、AC-003A-011D、AC-003A-012、AC-003A-013、AC-003A-014、AC-003A-015、AC-003A-016、AC-003A-017、AC-003A-018 |
| 上次同步时间 | 2026-05-27 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

让设计人员、内部审批人、项目管理员在统一的图纸列表视图中准确感知每张图纸的审批阶段，通过 Filter Search 快速定位目标图纸，并从 Actions 列的 6 个操作按钮（View / History / Confirms / Assign / Part Print / Upload New Version）直达各操作入口。

### 1.2 设计原则

1. **状态一目了然**：5 种审批状态以颜色标签区分，橙色=待审批、绿色=生效、红色=驳回；外部审批完成后在状态标签右侧附加结果代码 (A)–(E)，便于用户在列表层直接读取审批结论。
2. **非侵入式筛选**：Filter Search 以 Popover 浮层展开，不推动表格，不打断浏览体验；点击外部自动关闭并保留条件。
3. **操作前置守卫**：审批中状态时 [Upload New Version] 置灰并带 Tooltip 说明，防止误操作；[Assign] 仅对项目管理员可见；[Part Print] 仅对设计人员可见；[Export] 仅对项目管理员可见。
4. **列精简，行语义明确**：表格每一行代表**一次提交记录（DrawingVersion）**，对应设计人员上传的一个 PDF 文件（含多页图纸）；Drawing No / Drawing Name 为页级属性，**不在列表列中展示**，需进入详情页查看。列展示以 Description、Category、RFA No.、Subject of RFA、Version、Status、Confirmed、Total Markups、Last Updated、Actions 为准。
5. **跨文档职责清晰**：本页面仅承载列表展示与筛选；新建图纸弹窗见 UI-REQ-003E-pc，上传新版本弹窗见 UI-REQ-003F-pc。

---

## 2. 信息架构

### 2.1 页面层级

```
侧边栏
└── Drawing Management（一级菜单）
    └── Drawing Masterlist（二级菜单）→ 图纸列表页（本文档）
        ├── [Filter Search] Popover（点击按钮浮出）
        ├── [Export]           → 触发 Excel 文件下载（仅项目管理员可见）
        ├── [+ Upload Drawing] → 触发新建图纸弹窗（见 UI-REQ-003E-pc）
        ├── 图纸列表表格
        │   └── Actions 列（从左到右）
        │       ├── [View]             → 打开最新版本图纸 PDF 预览
        │       ├── [History]          → 提交历史弹框（见 UI-REQ-003C-pc）
        │       ├── [Confirms]         → SE 确认历史弹框
        │       ├── [Assign]           → 分配 SE 弹框（见 UI-REQ-003D-pc，仅管理员）
        │       ├── [Part Print]       → 图纸局部更新弹框（REQ-004-pc [+ Markup]，仅设计人员 & 状态 ACTIVE）
        │       └── [Upload New Version] → 上传新版本弹框（见 UI-REQ-003F-pc，仅设计人员）
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发角色 |
|---------|-------|---------|
| 侧边栏 Drawing Management → Drawing Masterlist | 图纸列表页 | 所有角色 |
| 图纸列表页右上角 [Export] | 触发 Excel 文件下载 | 项目管理员 |
| 图纸列表页右上角 [+ Upload Drawing] | 新建图纸弹窗（UI-REQ-003E-pc） | 设计人员 |
| 列表行 Actions 列 [View] | 最新版本图纸 PDF 预览 | 所有角色 |
| 列表行 Actions 列 [History] | 提交历史弹框（UI-REQ-003C-pc） | 所有角色 |
| 列表行 Actions 列 [Confirms] | SE 确认历史弹框 | 所有角色 |
| 列表行 Actions 列 [Assign] | SE 分配弹框（UI-REQ-003D-pc） | 项目管理员 |
| 列表行 Actions 列 [Part Print] | 图纸局部更新弹框（REQ-004-pc [+ Markup]，见 UI-REQ-004-pc） | 设计人员（仅状态 `ACTIVE` 时可见） |
| 列表行 Actions 列 [Upload New Version] | 上传新版本弹框（UI-REQ-003F-pc） | 设计人员（状态允许时） |

---

## 3. 页面 / 组件清单

### 3.1 图纸管理列表页（Drawing Masterlist）

**关联 Story**：US-003A-LIST-001、US-003A-LIST-002
**关联 AC**：AC-003A-005、AC-003A-005B、AC-003A-011、AC-003A-011B、AC-003A-012、AC-003A-013、AC-003A-014、AC-003A-015、AC-003A-016、AC-003A-017、AC-003A-018

> **列表行语义**：每一行 = 一次提交记录（**DrawingVersion**），对应一个上传的 PDF 文件。该 PDF 内各页图纸各自拥有 Drawing No 和 Drawing Name（页级属性），**不在列表列中展示**；可点击行进入详情页查看页级图纸信息。

#### 整体布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  顶部操作区                                                                   │
│  [⚙ Filter Search]  /  [⚙ Filter Search (2)]                                │
│                                        [↓ Export]  [+ Upload Drawing]        │
├─────────────────────────────────────────────────────────────────────────────┤
│  表格                                                                        │
│  Description | Category | RFA No. | Subject of RFA | Current Version        │
│  | Status | Confirmed | Total Markups | Last Updated ↕ | Actions             │
│                                                                              │
│  首层平面图说明  Architecture  RFA-001  首层审批  V3  [Active]  3/5  12       │
│  2026-05-01  [View][History][Confirms][Assign][Part Print][↑Upload]          │
└─────────────────────────────────────────────────────────────────────────────┘

（点击 Filter Search 后，紧贴按钮浮出 Popover）
┌──────────────────────────────────┐
│  Description   [_______________] │
│  Category      [▼ Select      ]  │
│  Status        [▼ Select      ]  │
│               [Cancel]  [Search] │
└──────────────────────────────────┘
```

**顶部操作区布局**：
- 左侧：[Filter Search] 按钮
- 右侧（从左到右）：[Export]（仅项目管理员可见）、[+ Upload Drawing]

**[Export] 按钮规格**：
- **可见性**：仅项目管理员角色渲染此按钮，其他角色完全隐藏（不占位）
- **默认态**：次要按钮（`el-button` plain），图标使用 `el-icon-download`，文案"Export"
- **Loading 态**：点击后按钮进入 loading 状态（`el-button` loading），不可重复点击，文案变为"Exporting…"
- **完成态**：文件下载触发后按钮恢复可点击默认态
- **失败态**：下载失败时按钮恢复默认态，同时页面右上角弹出 `el-message` error 类型，文案"Export failed, please try again"
- **导出范围**：当前 Filter Search 筛选条件下的全部记录（与当前列表所展示数据一致）

#### 表格列定义

| 列名 | 内容 | 排序 | 备注 |
|-----|------|:---:|------|
| Description | 图纸描述文本 | ❌ | 无内容时显示 `—` |
| Category | 图纸分类 | ❌ | — |
| RFA No. | 外部报审编号 | ❌ | 见 RFA No. 列显示规则 |
| Subject of RFA | 外部报审主题 | ❌ | 见 Subject of RFA 列显示规则 |
| Current Version | 当前版本号（如 V3） | ❌ | — |
| Status | `<DrawingStatusTag>` 颜色标签 | ❌ | 5 态，见 §4.1；外部审批完成后右侧附加灰色小字 (A)–(E)，见 §4.1 结果代码规则 |
| Confirmed | 已确认/总分配数（如 3/5） | ❌ | Status = `ACTIVE` 时显示 x/y；其他状态显示 `—` |
| Total Markups | 标注总数（数字） | ❌ | — |
| Last Updated | 最后更新日期时间 | ✅ 默认倒序 | — |
| Actions | 操作按钮组 | ❌ | 见下方 |

> ⚠️ **不展示** Drawing Code 列和 Drawing Name（Name）列。

**RFA No. 列显示规则**：
- 当前版本状态为 `PENDING_INTERNAL` 或 `INTERNAL_REJECTED` 时显示 `—`
- 外部审批已发起后显示具体报审编号（由 DC 在 REQ-007B-pc 流程写入）

**Subject of RFA 列显示规则**：
- 当前版本状态为 `PENDING_INTERNAL` 或 `INTERNAL_REJECTED` 时显示 `—`
- 外部审批已发起后显示具体报审主题（由 DC 在 REQ-007B-pc 流程写入）

#### Actions 列按钮

所有按钮均使用**图标按钮**（`el-button` size="mini" circle + icon），从左到右水平排列，间距 4px，悬浮时显示 `el-tooltip`。

| 顺序 | 按钮 | 图标 | Element UI icon | 点击行为 | 可见角色 | 状态规则 | Tooltip |
|:---:|------|------|----------------|---------|---------|---------|---------|
| 1 | **View** | 眼睛 | `el-icon-view` | 打开该提交记录最新版本图纸（PDF 预览） | 所有角色 | 始终可点 | "View" |
| 2 | **History** | 时钟 | `el-icon-time` | 显示提交历史弹框 | 所有角色 | 始终可点 | "History" |
| 3 | **Confirms** | 勾选列表 | `el-icon-finished` | 显示 SE 确认历史弹框 | 所有角色 | 始终可点 | "Confirms" |
| 4 | **Assign** | 用户 | `el-icon-user` | 显示分配 SE 弹框 | 仅项目管理员 | 始终可点 | "Assign SE" |
| 5 | **Part Print** | 打印/标注 | `el-icon-printer` | 显示图纸局部更新弹框（对应 REQ-004-pc [+ Markup] 发布局部更新弹窗） | 仅设计人员 | Status ≠ `ACTIVE` → **隐藏**（不展示按钮）；Status = `ACTIVE` → 可点 | "Part Print" |
| 6 | **Upload New Version** | 上传↑ | `el-icon-upload2` | 显示上传新版本弹框 | 仅设计人员 | Status = `PENDING_INTERNAL` 或 `PENDING_EXTERNAL` → `disabled` | "Upload New Version" / "A version is pending approval. Cannot upload now." |

#### Filter Search Popover

- **触发方式**：点击顶部 [Filter Search] 按钮，以 **Popover 浮层**方式弹出，**不推动表格**
- **Popover 位置**：紧贴按钮下方左对齐展开
- **Popover 宽度**：360px，内边距 16px
- **字段**：Description（`el-input` 文本）、Category（`el-select` 单选）、Status（`el-select` 单选）
  - ⚠️ **不包含** Drawing Code 和 Drawing Name 筛选项
- **Status 下拉选项（6 项）**：All / Active / Pending Internal / Pending External / Int. Rejected / Ext. Rejected
- **底部按钮**：[Cancel]（次要）、[Search]（主要）
  - [Search]：执行查询，Popover **关闭**，列表更新，按钮文案变为"Filter Search (n)"
  - [Cancel]：清空所有字段，Popover 关闭，列表恢复无筛选，按钮恢复"Filter Search"
  - **点击 Popover 外部**：Popover 关闭，已填字段**保留**，不执行查询，按钮计数不更新

**按钮文案计数规则**：统计已填写（非空）的字段数量 n；n = 0 → "Filter Search"；n > 0 → "Filter Search (n)"

#### 关键交互

| 交互 | 触发 | 反馈 |
|-----|------|-----|
| 点击 [+ Upload Drawing] | 点击顶部按钮 | 打开新建图纸弹窗（UI-REQ-003E-pc） |
| 点击 [Export]（默认态） | 项目管理员点击 Export 按钮 | 按钮进入 loading 态（"Exporting…"）→ 触发文件下载 → 按钮恢复默认态 |
| 点击 [Export]（失败） | 导出接口返回错误 | 按钮恢复默认态，右上角弹出 error toast："Export failed, please try again" |
| 点击 [Filter Search] | 点击顶部按钮 | Popover 浮出（fade + 向下位移 4px，150ms ease-out），按钮高亮 |
| 执行搜索（有条件） | 点击 Popover 内 [Search] | Loading 态 → Popover 关闭 → 列表更新 → 按钮显示"Filter Search (n)" |
| 执行搜索（无条件） | 点击 Popover 内 [Search] | 等同全量查询，Popover 关闭，按钮显示"Filter Search" |
| 取消搜索 | 点击 Popover 内 [Cancel] | 字段清空，Popover 关闭，列表恢复无筛选，按钮恢复"Filter Search" |
| 点击 Popover 外部 | 点击页面其他区域 | Popover 关闭，已填字段**保留**，不触发查询，按钮计数不变 |
| 排序 | 点击 Last Updated 列头 | 列表重排，列头显示排序方向箭头 |
| 点击 [View] | 点击 Actions 列 View 按钮 | 打开 PDF 预览弹框/新标签页 |
| 点击 [History] | 点击 Actions 列 History 按钮 | 打开提交历史弹框 |
| 点击 [Confirms] | 点击 Actions 列 Confirms 按钮 | 打开 SE 确认历史弹框 |
| 点击 [Assign] | 点击 Actions 列 Assign 按钮（仅管理员可见） | 打开分配 SE 弹框 |
| 点击 [Part Print] | 点击 Actions 列 Part Print 按钮（仅设计人员 & 状态 ACTIVE 时可见） | 打开图纸局部更新弹框（REQ-004-pc [+ Markup] 发布局部更新弹窗） |
| 点击 [Upload New Version]（可点） | 点击 Actions 列 Upload New Version 按钮（仅设计人员可见） | 打开上传新版本弹框 |
| 悬浮置灰 [Upload New Version] | 鼠标悬浮审批中行的 Upload New Version | Tooltip 显示"A version is pending approval. Cannot upload now." |

#### 列表状态（6 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空（无图纸） | 项目内无图纸记录 | 空状态插图 + 文案 + [+ Upload Drawing] 引导按钮 | "No drawings yet." |
| 空（无搜索结果） | 搜索条件无匹配 | 空状态插图 + 文案，不显示上传按钮 | "No drawings found. Try adjusting your search." |
| 加载 | 首次进入或执行搜索 | 表格行 Skeleton 骨架屏；[Search] 按钮 loading 态 | — |
| 正常 | 有数据 | 表格正常展示 | — |
| 错误 | 列表接口 5xx / 网络断开 | 表格区居中展示错误图标 + [Retry] 按钮 | "Failed to load. Please retry." |
| 极端数据 | Description / Subject 文本极长 | 单行截断 + Tooltip 展示全文；Actions 按钮固定宽度不被压缩 | — |

#### 权限可见性

| UI 元素 | 设计人员 | 内部审批人 | 项目管理员 | Site Engineer |
|--------|:-------:|:---------:|:---------:|:-------------:|
| 列表整体 | ✅ | ✅ | ✅ | — |
| [Export] | ❌ 隐藏 | ❌ 隐藏 | ✅ | — |
| [+ Upload Drawing] | ✅ | ❌ | ❌ | — |
| [View] | ✅ | ✅ | ✅ | — |
| [History] | ✅ | ✅ | ✅ | — |
| [Confirms] | ✅ | ✅ | ✅ | — |
| [Assign] | ❌ 隐藏 | ❌ 隐藏 | ✅ | — |
| [Part Print] | ✅（仅 Status = `ACTIVE` 时展示，其他状态隐藏） | ❌ 隐藏 | ❌ 隐藏 | — |
| [Upload New Version]（状态允许） | ✅ 可点 | ❌ 隐藏 | ❌ 隐藏 | — |
| [Upload New Version]（审批中） | ✅ 置灰 | ❌ 隐藏 | ❌ 隐藏 | — |

---

## 4. 设计令牌（Design Tokens）

### 4.1 Status 颜色语义

| 状态值 | 展示文案 | 颜色 Token | 色值 | Element UI type |
|--------|---------|-----------|------|----------------|
| `PENDING_INTERNAL` | Pending Internal | `--color-status-warning` | `#E6A23C` | `el-tag` type=`warning` |
| `PENDING_EXTERNAL` | Pending External | `--color-status-warning` | `#E6A23C` | `el-tag` type=`warning` |
| `ACTIVE` | Active | `--color-status-success` | `#67C23A` | `el-tag` type=`success` |
| `INTERNAL_REJECTED` | Int. Rejected | `--color-status-danger` | `#F56C6C` | `el-tag` type=`danger` |
| `EXTERNAL_REJECTED` | Ext. Rejected | `--color-status-danger` | `#F56C6C` | `el-tag` type=`danger` |

**外部审批结果代码附加显示规则**（AC-003A-011C / AC-003A-011D）：

外部审批完成后（状态进入 `ACTIVE` 或 `EXTERNAL_REJECTED`），Status 标签右侧以**灰色小字**附加结果代码，格式为 `(A)` / `(B)` / `(C)` / `(D)` / `(E)`。

| 结果代码 | 描述 | 对应状态 | Status 列完整展示示例 |
|---------|------|---------|-------------------|
| A | Approved / No Exception Taken | `ACTIVE` | 🟢 `Active` `(A)` |
| B | Approved with comment, resubmission required | `ACTIVE` | 🟢 `Active` `(B)` |
| C | Revise And Resubmit | `EXTERNAL_REJECTED` | 🔴 `Ext. Rejected` `(C)` |
| D | For Record Purpose | `ACTIVE` | 🟢 `Active` `(D)` |
| E | Others | `EXTERNAL_REJECTED` | 🔴 `Ext. Rejected` `(E)` |

**视觉规范**：
- 结果代码文字颜色：`var(--text-color-secondary, #909399)`（灰色），独立于状态标签颜色
- 结果代码字号：`11px`，与状态标签主文字（`12px`）区分
- 布局：结果代码与状态标签**内联排列**，间距 `4px`，整体保持在同一单元格内
- Tooltip：鼠标悬停结果代码时，展示完整描述文案，例如：`"B – Approved with comment, resubmission required"`
- 外部审批尚未完成的状态（`PENDING_INTERNAL` / `PENDING_EXTERNAL` / `INTERNAL_REJECTED`）**不附加**结果代码

### 4.2 间距与布局

| 组件 | 属性 | 值 |
|-----|------|----|
| 顶部操作区 | display | flex；justify-content: space-between；align-items: center |
| 顶部操作区右侧按钮组 | display | flex；gap: 8px；align-items: center |
| Filter Search 按钮容器 | height | 32px；padding: 5px 8px；border-radius: 2px |
| Filter Search 按钮容器 | border | 1px solid `var(--border-color, #D8E2F0)` |
| Filter Search 按钮容器 | background | `var(--Vertical-Menu-White, #FFF)` |
| Filter Search Popover | width | 360px；padding: 16px |
| Filter Search Popover 字段行 | gap | 12px（flex column） |
| Filter Search Popover 按钮区 | justify-content | flex-end；gap: 8px；margin-top: 16px |
| 表格 Actions 列 | display | flex；gap: 4px；align-items: center |
| DrawingStatusTag | font-size | 12px（⚠️ 必须显式设置）；line-height: 20px；font-weight: 400 |
| 表格行 | font-size | 14px（⚠️ 必须显式设置） |

### 4.3 字体排版

#### 页面标题

| 属性 | 值 |
|------|----|
| color | `var(--text-color-primary-dark, #344050)` |
| font-family | Arial；font-size: 18px；font-weight: 700；line-height: 26px |
| 设计 Token | `500 Medium / H2-18-large` |

#### 表格内容文字

| 属性 | 值 |
|------|----|
| color | `var(--text-color-primary-dark, #344050)` |
| font-family | "Helvetica Neue"；font-size: 14px；font-weight: 400；line-height: 22px；letter-spacing: -0.01px |

#### Filter Search 按钮文字

| 属性 | 值 |
|------|----|
| color | `var(--text-color-primary-dark, #344050)` |
| font-family | "Helvetica Neue"；font-size: 12px；font-weight: 400；line-height: 24px |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 全列可见，完整布局 |
| Tablet | 768–1280px | RFA No. / Subject of RFA / Total Markups 列可折叠隐藏 |
| Mobile | < 768px | 不支持（PC 专属） |

---

## 6. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| Filter Search Popover 打开 | fade + 向下位移 4px | 150ms | ease-out |
| Filter Search Popover 关闭 | fade | 100ms | ease-in |
| 按钮计数文案变化 | 即时更新，无动效 | — | — |
| 排序切换 | 列表行重排 | 200ms | ease |
| Snackbar 出现 | 从右上角 slide-in | 300ms | ease-out |
| Snackbar 消失 | fade-out | 200ms | ease-in |
| [Upload New Version] 置灰过渡 | opacity 变化 | 150ms | ease |
| 弹框（History / Confirms / Assign / Part Print / Upload New Version）打开 | fade + scale(0.95→1) | 200ms | ease-out |
| 弹框关闭 | fade | 150ms | ease-in |

---

## 7. 无障碍（A11y）

- **WCAG 等级**：AA
- **颜色对比度**：正文（14px）≥ 4.5:1；状态 Tag 文字与背景 ≥ 3:1
- **键盘导航**：
  - [Filter Search] 按钮支持 Enter/Space 打开 Popover；Esc 关闭 Popover
  - Popover 内字段支持 Tab 键顺序导航；[Search] / [Cancel] 支持 Enter 确认
- **焦点可见**：所有可交互元素 focus 态显示 2px solid `#409EFF` outline
- **屏幕阅读器**：
  - 置灰按钮使用 `aria-disabled="true"` + Tooltip 说明原因
  - 列表加载状态通过 `aria-live="polite"` 通知
  - Status Tag 使用 `aria-label` 声明完整状态文案

---

## 8. 复用与新建组件清单

| 组件 | 来源 | 备注 |
|-----|------|------|
| `el-table` | Element UI 现有 | 表格主体 |
| `el-button` | Element UI 现有 | 顶部操作按钮（含 [Export]、[+ Upload Drawing]）、Actions 列图标按钮 |
| `el-select` | Element UI 现有 | Popover 内 Category / Status 下拉 |
| `el-input` | Element UI 现有 | Popover 内 Description 文本输入 |
| `el-popover` | Element UI 现有 | Filter Search 浮层容器 |
| `el-dialog` | Element UI 现有 | History / Confirms / Assign / Part Print / Upload New Version 弹框容器 |
| `el-tag` | Element UI 现有 | 状态标签，封装为 `DrawingStatusTag` |
| `el-tooltip` | Element UI 现有 | Actions 按钮提示 + 长文本截断提示 |
| `DrawingStatusTag` | **新建** | 封装 5 态状态→颜色/文案映射，复用于列表及版本历史视图 |
| `DrawingSearchPopover` | **新建** | Filter Search Popover，含 Description / Category / Status 字段 + 计数逻辑 |
| `DrawingHistoryDialog` | **新建** | 提交历史弹框（History 按钮触发） |
| `DrawingConfirmsDialog` | **新建** | SE 确认历史弹框（Confirms 按钮触发） |
| `DrawingAssignDialog` | **新建** | 分配 SE 弹框（Assign 按钮触发，仅管理员） |
| `DrawingPartPrintDialog` | **新建** | 图纸局部更新弹框（Part Print 按钮触发）；对应 REQ-004-pc [+ Markup] 发布局部更新弹窗；详细字段与布局见 UI-REQ-004-pc |

---

## 9. 文案规范

| 场景 | 文案（en） |
|-----|----------|
| 页面标题 | Drawing Masterlist |
| 新建按钮 | + Upload Drawing |
| 导出按钮（默认态） | Export |
| 导出按钮（loading 态） | Exporting… |
| 导出失败提示 | Export failed, please try again |
| [View] 按钮 Tooltip | View |
| [History] 按钮 Tooltip | History |
| [Confirms] 按钮 Tooltip | Confirms |
| [Assign] 按钮 Tooltip | Assign SE |
| [Part Print] 按钮 Tooltip | Part Print |
| [Upload New Version] 按钮 Tooltip（正常） | Upload New Version |
| [Upload New Version] 按钮 Tooltip（置灰） | A version is pending approval. Cannot upload now. |
| Filter Search 按钮（无条件） | Filter Search |
| Filter Search 按钮（有 n 条件） | Filter Search (n) |
| Popover 搜索按钮 | Search |
| Popover 取消按钮 | Cancel |
| 列表空状态（无图纸） | No drawings yet. |
| 列表空状态（无搜索结果） | No drawings found. Try adjusting your search. |
| 列表加载失败 | Failed to load. Please retry. |
| RFA No. 未发起时占位 | — |
| Subject of RFA 未发起时占位 | — |
| Confirmed（非 ACTIVE 状态）占位 | — |
| Status 结果代码 Tooltip（结果 A） | A – Approved / No Exception Taken |
| Status 结果代码 Tooltip（结果 B） | B – Approved with comment, resubmission required |
| Status 结果代码 Tooltip（结果 C） | C – Revise And Resubmit |
| Status 结果代码 Tooltip（结果 D） | D – For Record Purpose |
| Status 结果代码 Tooltip（结果 E） | E – Others (please state reason) |

---

## 10. AC 覆盖检查表

| AC ID | 对应组件 | 描述 | 覆盖? |
|------|---------|------|------|
| AC-003A-005 | §3.1 Actions 列按钮 | 6 个按钮从左到右排列：View / History / Confirms / Assign / Part Print / Upload New Version；各按钮点击行为与弹框正确对应；[Part Print] 仅状态 `ACTIVE` 时可见，触发 REQ-004-pc [+ Markup] 局部更新弹窗 | ✅ |
| AC-003A-005B | §3.1 Actions 列 [Upload New Version] | PENDING_INTERNAL / PENDING_EXTERNAL 时置灰 + Tooltip | ✅ |
| AC-003A-011 | §3.1 表格列定义 | 展示 Description / Category / RFA No. 等列，不展示 Drawing Code / Name | ✅ |
| AC-003A-011B | §3.1 Status 颜色标签 + §4.1 | 5 态颜色正确对应 | ✅ |
| AC-003A-011C | §3.1 Status 列说明 + §4.1 结果代码规则 | 外部审批完成后 Status 标签右侧附加灰色 (A)–(E) 结果代码；外部审批未完成时不显示代码 | ✅ |
| AC-003A-011D | §4.1 结果代码规则 + §3.1 Actions | Status B 时显示 `Active (B)`；[Upload New Version] 在 ACTIVE 状态可点，设计人员可据此发起新版本上传 | ✅ |
| AC-003A-012 | §3.1 Filter Search Popover 字段 | 包含 Description / Category / Status；不包含 Drawing Code / Name | ✅ |
| AC-003A-013 | §3.1 Status 下拉选项 | 6 项：All / Active / Pending Internal / Pending External / Int. Rejected / Ext. Rejected | ✅ |
| AC-003A-014 | §3.1 Filter Search 按钮文案 | 有 n 条件时显示"Filter Search (n)"；清空后恢复"Filter Search" | ✅ |
| AC-003A-015 | §3.1 点击 Popover 外部 | Popover 关闭，条件保留，不触发查询 | ✅ |
| AC-003A-016 | §3.1 列表行语义说明 + 表格列定义 | 上传 1 个 PDF → 列表新增 1 行（DrawingVersion V0）；页级 Drawing No / Name 不在列展示 | ✅ |
| AC-003A-017 | §3.1 权限可见性 + [Export] 按钮规格 | [Export] 仅对项目管理员渲染，其他角色隐藏 | ✅ |
| AC-003A-018 | §3.1 [Export] 按钮规格 + 关键交互 | 点击触发文件下载；loading 态；失败时 error toast | ✅ |

---

## 11. 待定问题（Open Questions）

| OQ ID | 问题 | 影响 UI 哪部分 |
|------|------|--------------|
| OQ-UI-001 | Actions 列 6 个按钮宽度较大，是否在窄屏/更多列时折叠为 [···] 下拉菜单？ | Actions 列布局 |
| OQ-UI-002 | Description / Subject of RFA 列是否需要固定最大宽度？ | 表格列宽约束 |
| OQ-UI-003 | 是否需要列的显示/隐藏自定义功能（Column Picker）？ | 顶部操作区 |
| OQ-UI-004 | [Confirms] 弹框展示内容的字段与布局待确认（关联 SE 确认详情需求）。 | DrawingConfirmsDialog |
| OQ-UI-005 | ~~[Part Print] 局部更新弹框的具体字段与业务流程待关联需求文档补充。~~ **已关闭**：已关联 REQ-004-pc，弹框字段详见 UI-REQ-004-pc。 | DrawingPartPrintDialog |

---

## 12. 设计交付验收清单

- [ ] [Export] 按钮设计稿（默认态 / loading 态，仅项目管理员视角可见）
- [ ] [Export] 失败 error toast 设计稿
- [ ] Figma 稿覆盖图纸管理列表页全部列（含 Description / RFA No. / Subject of RFA / Total Markups）
- [ ] Filter Search Popover 设计稿（收起态 / 展开空态 / 已填写态 / 按钮计数态）
- [ ] 列表 6 态设计稿（空-无图纸 / 空-无结果 / 加载 / 正常 / 错误 / 极端数据）
- [ ] DrawingStatusTag 所有 5 种状态色块设计稿
- [ ] DrawingStatusTag 结果代码附加展示设计稿（Active (A) / Active (B) / Active (D) / Ext. Rejected (C) / Ext. Rejected (E)），包含结果代码 Tooltip 气泡
- [ ] Actions 列 6 个按钮正常态设计稿（含各按钮 Tooltip 气泡）
- [ ] Actions 列 [Upload New Version] 置灰态 + Tooltip 设计稿
- [ ] Actions 列 [Part Print] 按钮状态设计稿（ACTIVE 时展示 vs 非 ACTIVE 时隐藏）
- [ ] DrawingPartPrintDialog 弹框设计稿（参考 UI-REQ-004-pc）
- [ ] §10 所有 AC 标记 ✅
- [ ] 移动端不适用（PC 专属，可忽略）

---

## 13. Figma / 原型链接

- Figma 设计稿：<!-- 填写图纸列表 Frame 链接 -->
- 交互原型：<!-- 填写可点击原型链接 -->

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-04 | agent | 从 REQ-003A-pc.md@0.1.0 生成初稿（含列表页 + Upload New Drawing 侧滑面板 + Upload New Version 侧滑面板） |
| 0.4.3 | 2026-05-25 | agent | 同步 REQ-003A-pc@0.4.3：① 明确列表行语义——每行 = DrawingVersion（一次 PDF 上传），Drawing No / Drawing Name 为页级属性不在列展示；② §1.2 设计原则第 4 条更新；③ §3.1 新增列表行语义说明块；④ AC 覆盖表新增 AC-003A-016 |
| 0.4.4 | 2026-05-25 | agent | 同步 REQ-003A-pc@0.4.4：① Actions 列扩展为 6 个按钮，从左到右：View / History / Confirms / Assign / Part Print / Upload New Version；② 更新信息架构（§2.1 页面层级、§2.2 导航入口）；③ 重写 §3.1 Actions 列按钮表格（含顺序、图标、行为、可见角色、状态规则、Tooltip）；④ 扩充关键交互表（新增 6 个按钮点击交互行为）；⑤ 权限可见性表扩展为 9 行；⑥ 新增 el-dialog 现有组件 + DrawingHistoryDialog / DrawingConfirmsDialog / DrawingAssignDialog / DrawingPartPrintDialog 新建组件；⑦ 文案规范重整，按钮分别列出；⑧ AC 覆盖表新增 AC-003A-005B；⑨ Open Questions 新增 OQ-UI-004 / OQ-UI-005；⑩ 设计交付验收清单更新 Actions 列验收项 |
| 0.5.0 | 2026-05-27 | agent | 同步 REQ-003A-pc@0.5.0：① 顶部操作区新增 [Export] 按钮（次要按钮，icon `el-icon-download`，位于 [+ Upload Drawing] 左侧），仅项目管理员可见；② 更新顶部操作区 ASCII 布局图；③ 新增 [Export] 按钮规格说明（默认态/loading 态/失败态）；④ 新增右侧按钮组 flex 容器间距规则（gap: 8px）；⑤ §2.1 页面层级新增 [Export] 节点；⑥ §2.2 导航表新增 Export 行；⑦ §1.2 设计原则第 3 条补充 Export 可见性说明；⑧ 权限可见性表新增 [Export] 行；⑨ 关键交互表新增 Export 点击与失败两条；⑩ 文案规范新增 Export 相关 3 条；⑪ AC 覆盖表新增 AC-003A-017 / AC-003A-018；⑫ 溯源块更新版本与 AC 列表；⑬ 验收清单新增 Export 设计稿项 |
| 0.4.6 | 2026-05-26 | agent | 同步 REQ-003A-pc@0.4.6：① §4.1 新增外部审批结果代码附加显示规则（A–E 代码映射表、视觉规范：灰色 `#909399` 11px 小字、间距 4px、Tooltip 完整描述）；② §1.2 设计原则第 1 条更新；③ §3.1 Status 列备注补充结果代码说明；④ §9 文案规范新增 5 条结果代码 Tooltip 文案；⑤ §10 AC 覆盖表新增 AC-003A-011C / 011D；⑥ §12 设计交付验收清单新增结果代码设计稿项 | Frontend、QA |
| 0.4.5 | 2026-05-25 | agent | 同步 REQ-003A-pc@0.4.5：① [Part Print] 关联 REQ-004-pc——点击触发 REQ-004 [+ Markup] 发布局部更新弹窗；② 状态限制更新：非 `ACTIVE` 时**隐藏**按钮（而非始终可见）；③ 更新 §2.1 页面层级、§2.2 导航入口、§3.1 Actions 按钮表（第 5 行）、关键交互、权限可见性；④ §8 DrawingPartPrintDialog 补充引用 REQ-004-pc 与 UI-REQ-004-pc；⑤ AC-003A-005 补充 Part Print 状态限制说明；⑥ OQ-UI-005 标记已关闭；⑦ 验收清单新增 Part Print 状态设计稿与弹框设计稿 |① 移除 §3.2 Upload New Drawing / §3.3 Upload New Version（分别迁至 UI-REQ-003E-pc / UI-REQ-003F-pc）；② 表格列更新：移除 Drawing Code / Name，新增 Description、RFA No.、Subject of RFA、Total Markups；③ Filter Search 从推开面板改为 Popover，筛选字段改为 Description / Category / Status（移除 Drawing Code / Name）；④ Status 枚举更新为 5 态（PENDING_INTERNAL / PENDING_EXTERNAL / ACTIVE / INTERNAL_REJECTED / EXTERNAL_REJECTED），颜色语义表更新；⑤ Status 筛选下拉更新为 6 项；⑥ [Upload New Version] 置灰条件更新为 PENDING_INTERNAL 或 PENDING_EXTERNAL；⑦ AC 覆盖表更新为 003A-005 / 011 / 011B / 012 / 013 / 014 / 015；⑧ 新建组件清单新增 DrawingSearchPopover，替换原 DrawingSearchPanel |
