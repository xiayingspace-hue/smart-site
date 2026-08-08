---
doc_type: requirement
req_id: REQ-003A-pc
req_title: "PC 端 — 图纸管理列表页"
version: 0.5.1
status: draft
priority: P1
product: SMART SITE SYSTEM
owner: ""
created_at: 2026-05-04
updated_at: 2026-05-27

depends_on:
  - REQ-007-shared
  - REQ-003-shared
related_to:
  - REQ-003E-pc
  - REQ-003F-pc
  - REQ-003C-pc
  - REQ-003D-pc
  - REQ-004-pc
  - REQ-007A-pc
  - REQ-007B-pc
  - REQ-007C-pc
  - REQ-007D-pc
blocks: []

generate:
  data_contract: false
  ui_spec: true
  frontend_spec: true
  backend_spec: false
  qa_spec: true
---

# 需求文档：PC 端 — 图纸管理列表页

> **使用说明**：本文档是整个交付链路的**单一事实源**，聚焦于图纸管理列表页的展示、筛选与状态呈现逻辑。
> 新建图纸上传流程见 [REQ-003E-pc](REQ-003E-pc.md)；上传新版本流程见 [REQ-003F-pc](REQ-003F-pc.md)；完整审批业务规则见 [REQ-007-shared](../shared/REQ-007-shared.md)。

---

## 1. 背景与目标

### 1.1 业务背景

PC 管理端图纸列表页是所有图纸管理操作的入口枢纽：设计人员、图纸管理员、内部审批人在此查看项目图纸全貌，通过状态标签了解每张图纸所处的审批阶段，并从 Actions 列触达各操作入口（查看图纸、历史记录、SE 确认历史、分配 SE、局部更新、上传新版本）。

> **业务模型说明（REQ-003E v0.3.0+ 更新后）**：图纸列表中**每一行代表一次提交记录**（DrawingVersion），对应设计人员上传的**一个 PDF 文件**。一个 PDF 文件包含多页图纸，每一页图纸拥有独立的 Drawing No 和 Drawing Name——这些是**页级（图纸页）**属性，不在列表行中直接展示。提交记录本身以 Description、Category 等批次公共字段为主要标识，版本号体现该图纸包的迭代次数。

### 1.2 业务目标

让项目参与者在一个统一的列表视图中准确感知图纸状态、快速定位目标图纸并进入对应操作流程，无需跨页面来回切换。

### 1.3 非目标（Out of Scope）

- 新建图纸上传操作（由 REQ-003E-pc 覆盖）
- 上传新版本操作（由 REQ-003F-pc 覆盖）
- 内部审批操作（由 REQ-007A-pc 覆盖）
- DC 外部审批操作（由 REQ-007B-pc 覆盖）
- 版本历史 4 阶段视图（由 REQ-007C-pc 覆盖）
- DC 配置页面（由 REQ-007D-pc 覆盖）
- Site Engineer 分配（由 REQ-003D-pc 覆盖）
- APP 端操作（由 REQ-003-app 覆盖）

---

## 2. 用户与角色

### 2.1 角色定义

| 角色 ID | 角色名 | 描述 | 典型场景 |
|--------|-------|------|---------|
| ROLE-001 | 设计人员（Designer） | 图纸责任人 | 查看自己上传图纸的审批进度 |
| ROLE-002 | 内部审批人 | 设计经理/总工程师等技术人员 | 查看待审批图纸 |
| ROLE-003 | 图纸管理员 | 负责 SE 分配等管理操作 | 监控整体审批进度，发起 SE 分配 |

### 2.2 用户故事（User Stories）

#### US-003A-LIST-001：查看图纸列表与筛选

```
作为 项目参与者（设计人员 / 内部审批人 / 图纸管理员）
我想要 在图纸管理列表页通过图纸编号、分类、状态等条件筛选图纸
以便 快速定位目标图纸，了解其审批进度并进入对应操作入口
```

> **背景**：每行记录对应一次提交记录（DrawingVersion），即设计人员上传的一个 PDF 文件。PDF 内含多页图纸，每页各自拥有 Drawing No 和 Drawing Name（页级属性，不直接展示在列表行中）。提交记录以 Description、Category、版本号和审批状态为主要行级信息，各行独立走审批流程。

**优先级**：P1

---

#### US-003A-LIST-002：导出图纸信息至 Excel

```
作为 图纸管理员
我想要 在图纸列表页点击导出按钮，将当前列表数据导出为 Excel 文件
以便 离线查阅所有图纸的关联关系与审批状态
```

**优先级**：P1

---

## 3. 角色与权限矩阵

| 操作 | 设计人员 | 内部审批人 | 图纸管理员 | Site Engineer |
|-----|:-------:|:---------:|:---------:|:-------------:|
| 查看图纸列表 | ✅ | ✅ | ✅ | — |
| 使用 Filter Search | ✅ | ✅ | ✅ | — |
| [Export]：导出图纸信息至 Excel | ❌ | ❌ | ✅ | — |
| [View]：查看最新版本图纸 | ✅ | ✅ | ✅ | — |
| [History]：查看提交历史 | ✅ | ✅ | ✅ | — |
| [Confirms]：查看 SE 确认历史 | ✅ | ✅ | ✅ | — |
| [Assign]：分配 SE | ❌ | ❌ | ✅ | — |
| [Part Print]：图纸局部更新（见 REQ-004-pc） | ✅（仅状态 `ACTIVE` 时） | ❌ | ❌ | ❌ |
| [Upload New Version]（状态允许时） | ✅ | ❌ | ❌ | ❌ |
| [Upload New Version] 置灰（状态不允许） | 置灰 | — | — | — |

---

## 4. 核心实体与状态定义

### 4.1 实体清单（参考）

| 实体 | 描述 | 关键属性 |
|------|------|---------|
| Drawing | 图纸页级记录；对应 PDF 中的一页，拥有唯一的 Drawing No 和 Drawing Name；**不直接作为列表行展示** | Drawing No、Drawing Name、所属 DrawingVersion |
| DrawingVersion | **列表每行所对应的实体**；对应设计人员的一次上传（一个 PDF 文件），V0 = 首版 | 版本号、PDF 文件、状态、报审编号、报审主题、Description、Category、包含图纸页数 |
| AIRecognitionJob | 一次 PDF 上传产生一个识别任务；识别完成后将各页图纸信息（Drawing No / Name）关联至该 DrawingVersion | JobID、状态（PROCESSING / DONE / FAILED）、关联 Drawing 页列表 |

> 每条提交记录（DrawingVersion）的 Description 与 Category 来自上传时填写的批次公共字段（见 REQ-003E-pc §7.1）；PDF 内各页图纸的 Drawing No 与 Drawing Name 为页级属性，可通过点击提交记录查看详情页获取。
>
> 完整实体定义与 API 见 [REQ-003-shared](../shared/REQ-003-shared.md)。

### 4.2 Drawing 状态枚举（列表展示用）

| 状态值 | 展示文案 | 颜色 |
|--------|---------|------|
| `PENDING_INTERNAL` | Pending Internal | 橙色 `#E6A23C` |
| `PENDING_EXTERNAL` | Pending External | 橙色 `#E6A23C` |
| `ACTIVE` | Active | 绿色 `#67C23A` |
| `INTERNAL_REJECTED` | Int. Rejected | 红色 `#F56C6C` |
| `EXTERNAL_REJECTED` | Ext. Rejected | 红色 `#F56C6C` |

**外部审批结果代码附加显示规则**：

外部审批完成后（Status 进入 `ACTIVE` 或 `EXTERNAL_REJECTED`），Status 列的状态标签右侧以灰色小字附加显示外部审批结果代码，格式为 `(A)` / `(B)` / `(C)` / `(D)` / `(E)`，与纸质报审表保持对应，便于用户在列表层直接感知审批结论。

| 外部审批结果 | 描述 | 对应状态值 | 列表 Status 展示示例 |
|------------|------|----------|-------------------|
| A | Approved / No Exception Taken | `ACTIVE` | 🟢 Active **(A)** |
| B | Approved with comment, resubmission required | `ACTIVE` | 🟢 Active **(B)** |
| C | Revise And Resubmit | `EXTERNAL_REJECTED` | 🔴 Ext. Rejected **(C)** |
| D | For Record Purpose | `ACTIVE` | 🟢 Active **(D)** |
| E | Others | `EXTERNAL_REJECTED` | 🔴 Ext. Rejected **(E)** |

> **设计说明**：
> - 5 个工作流状态值本身不变，代表图纸所处的流程节点；审批结果代码仅作为辅助信息附加展示，不新增独立状态值。
> - **Status B 需特别关注**：版本虽已生效（ACTIVE），但业主/审批方要求设计人员提交修改版，设计人员看到 `Active (B)` 应知晓需要跟进上传新版本。
> - 外部审批尚未发生时（`PENDING_INTERNAL`、`INTERNAL_REJECTED`、`PENDING_EXTERNAL`），Status 列不附加代码，仅展示工作流状态文案。

---

## 5. 功能需求详述

### 5.1 功能 F-001：图纸管理列表页

**关联用户故事**：US-003A-LIST-001

**导航入口**：侧边栏菜单"Drawing Management" > "Drawing Masterlist"

**列表默认排序**：按 Last Updated 倒序

> **列表行含义**：表格每一行代表**一次提交记录**（DrawingVersion），即一个上传的 PDF 文件。PDF 内各页图纸（Drawing No / Drawing Name）为页级属性，**不在列表列中展示**，可通过详情页查看。

**表格列**：

| 列名 | 说明 |
|------|------|
| Description | 上传时填写的批次描述；列表主标识列 |
| Category | 图纸分类（上传时填写） |
| RFA No. | 外部审批报审编号（见下方显示规则） |
| Subject of RFA | 外部审批报审主题（见下方显示规则） |
| Current Version | 当前版本号 |
| Status | 颜色状态标签（5 态，见 §4.2） |
| Confirmed | 状态为 `ACTIVE` 时显示 x/y；其他状态显示 `—` |
| Total Markups | 标注总数 |
| Last Updated | 最后更新时间（可排序） |
| Actions | 操作按钮入口 |

**RFA No. 列显示规则**：
- 当前版本状态为 `PENDING_INTERNAL` 或 `INTERNAL_REJECTED` 时显示 `—`
- 外部审批已发起后显示具体报审编号（由 DC 在 REQ-007B-pc 流程中写入）

**Subject of RFA 列显示规则**：
- 当前版本状态为 `PENDING_INTERNAL` 或 `INTERNAL_REJECTED` 时显示 `—`
- 外部审批已发起后显示具体报审主题（由 DC 在 REQ-007B-pc 流程中写入）

**顶部操作区**：
- 左侧：[Filter Search] 按钮（见下方 Popover 规则）
- 右侧（从左到右）：[Export]（仅图纸管理员可见）、[+ Upload Drawing]（点击触发新建图纸弹窗，详见 REQ-003E-pc）

**[Export] 按钮规则**：
- 仅对**图纸管理员**角色显示，其他角色不展示此按钮
- 点击后触发 Excel 文件下载，导出**当前筛选条件下**的全部图纸数据
- 导出过程中按钮显示 loading 状态，完成后恢复可点击；若导出失败，以 Toast 提示"Export failed, please try again"

**Actions 列按钮规则**（从左到右排列）：

| 顺序 | 按钮 | 点击行为 | 权限 | 状态限制 |
|:---:|------|---------|------|---------|
| 1 | [View] | 查看该提交记录最新版本的图纸（PDF 预览） | 所有角色 | 始终可点 |
| 2 | [History] | 显示提交历史弹框（见 REQ-003C-pc） | 所有角色 | 始终可点 |
| 3 | [Confirms] | 显示 SE 确认历史弹框（见 REQ-003C-pc） | 所有角色 | 始终可点 |
| 4 | [Assign] | 显示分配 SE 的弹框（见 REQ-003D-pc） | 仅图纸管理员 | 始终可点 |
| 5 | [Part Print] | 显示图纸局部更新弹框（详见 [REQ-004-pc](REQ-004-pc.md)）；对应 REQ-004 中的 [+ Markup] 发布局部更新弹窗入口 | 仅设计人员 | 仅当状态为 `ACTIVE` 时可点（状态非 `ACTIVE` 时隐藏） |
| 6 | [Upload New Version] | 显示上传新版本弹框（见 REQ-003F-pc） | 仅设计人员 | 状态为 `PENDING_INTERNAL` 或 `PENDING_EXTERNAL` 时置灰，Tooltip 提示当前状态不允许上传 |

**Filter Search Popover 规则**：
- [Filter Search] 按钮文案根据生效条件数量动态变化：无条件时显示"Filter Search"；有 n 个条件时显示"Filter Search (n)"
- Popover 搜索字段：Description（文本，模糊搜索）、Category（下拉单选）、Status（下拉单选）
- Popover 底部：[Search] 触发查询并关闭 Popover；[Cancel] 清空所有条件并关闭 Popover
- 点击 Popover 外部时 Popover 关闭，已填写但未点击 [Search] 的条件**保留**在表单内，不执行查询

**Status 筛选下拉选项（6 项）**：All / Active / Pending Internal / Pending External / Int. Rejected / Ext. Rejected

---

## 6. 验收标准（Acceptance Criteria）

### AC-003A-005：Actions 列按钮排列与行为

```
Given  项目中存在提交记录
When   用户查看列表行的 Actions 列
Then   按钮从左到右顺序为：[View] [History] [Confirms] [Assign] [Part Print] [Upload New Version]；
       点击 [View]         → 打开该提交记录最新版本的图纸（PDF 预览）；
       点击 [History]      → 显示提交历史弹框；
       点击 [Confirms]     → 显示 SE 确认历史弹框；
       点击 [Assign]       → 显示分配 SE 的弹框（仅图纸管理员可见）；
       点击 [Part Print]   → 显示图纸局部更新弹框（REQ-004-pc [+ Markup]，仅状态为 ACTIVE 时可见）；
       点击 [Upload New Version] → 显示上传新版本弹框（仅设计人员可见）
```

### AC-003A-005B：审批中状态时 [Upload New Version] 置灰

```
Given  图纸当前状态为 PENDING_INTERNAL 或 PENDING_EXTERNAL
When   用户查看该行的 Actions 列
Then   [Upload New Version] 按钮处于置灰不可点状态，Tooltip 提示当前状态不允许上传
```

### AC-003A-011：图纸列表每行为一次提交记录，以 Description 为主标识列

```
Given  项目中存在提交记录（通过上传 PDF 文件创建）
When   用户进入图纸管理列表页
Then   表格列包含 Description、Category、RFA No.、Subject of RFA、Current Version、
       Status、Confirmed、Total Markups、Last Updated、Actions；
       每行代表一次提交记录（DrawingVersion），即一次 PDF 上传；
       列表中不展示 Drawing No 和 Drawing Name（此为 PDF 页级属性，需进入详情页查看）
```

### AC-003A-011B：Status 列显示 5 种颜色状态标签

```
Given  项目图纸处于不同审批阶段
When   用户查看图纸列表 Status 列
Then   PENDING_INTERNAL 显示橙色"Pending Internal"；
       PENDING_EXTERNAL 显示橙色"Pending External"；
       ACTIVE 显示绿色"Active"；
       INTERNAL_REJECTED 显示红色"Int. Rejected"；
       EXTERNAL_REJECTED 显示红色"Ext. Rejected"
```

### AC-003A-011C：外部审批完成后 Status 列附加显示审批结果代码

```
Given  DC 已在 REQ-007B 流程中完成外部审批标记，图纸状态进入 ACTIVE 或 EXTERNAL_REJECTED
When   用户查看图纸列表 Status 列
Then   状态标签右侧以灰色小字附加显示外部审批结果代码，格式为 (A) / (B) / (C) / (D) / (E)；
       具体示例：
         - 审批结果 A → 显示"Active (A)"（绿色）
         - 审批结果 B → 显示"Active (B)"（绿色）
         - 审批结果 C → 显示"Ext. Rejected (C)"（红色）
         - 审批结果 D → 显示"Active (D)"（绿色）
         - 审批结果 E → 显示"Ext. Rejected (E)"（红色）；
       外部审批尚未完成的状态（PENDING_INTERNAL / PENDING_EXTERNAL / INTERNAL_REJECTED）
       不显示结果代码
```

### AC-003A-011D：Status B 时设计人员可感知需要跟进上传新版本

```
Given  外部审批结果为 B（Approved with comment, resubmission required），图纸状态为 ACTIVE
When   设计人员查看图纸列表 Status 列
Then   Status 列显示"Active (B)"；
       [Upload New Version] 按钮在状态为 ACTIVE 时为可点击状态，
       设计人员可据此发起新版本上传以响应审批方的修改意见
```

### AC-003A-012：Filter Search Popover 字段

```
Given  用户打开 Filter Search Popover
When   查看搜索条件
Then   Popover 包含 Description（文本，模糊）、Category（下拉单选）、Status（下拉单选）共 3 个搜索字段；
       不包含 Drawing Code 或 Drawing Name（页级属性，不在列表层筛选）
```

### AC-003A-013：Status 筛选下拉包含 6 个选项

```
Given  用户打开 Filter Search Popover
When   点击 Status 筛选下拉
Then   选项包含：All / Active / Pending Internal / Pending External / Int. Rejected / Ext. Rejected
```

### AC-003A-014：Filter Search 按钮文案动态更新

```
Given  用户在 Filter Search Popover 中填写了 Description 和 Status 共 2 个筛选条件并点击 [Search]
When   Popover 关闭后查看按钮文案
Then   按钮显示 "Filter Search (2)"；清空所有条件后按钮恢复为 "Filter Search"
```

### AC-003A-015：点击 Popover 外部不触发查询
```
Given  用户在 Filter Search Popover 中填写了条件但未点击 [Search]
When   点击 Popover 外部区域关闭 Popover
Then   Popover 关闭，填写的条件保留在表单内，列表数据不刷新
```

### AC-003A-017：[Export] 按钮仅对图纸管理员可见

```
Given  用户登录系统并进入图纸管理列表页
When   用户角色为设计人员或内部审批人
Then   顶部操作区不显示 [Export] 按钮；
       当用户角色为图纸管理员时，[Export] 按钮显示在 [+ Upload Drawing] 左侧
```

### AC-003A-018：[Export] 按钮触发文件下载

```
Given  图纸管理员在图纸管理列表页（可含筛选条件）
When   点击 [Export] 按钮
Then   系统触发 Excel 文件下载，文件内容为当前筛选结果下的全部记录；
       导出过程中按钮显示 loading 状态；
       若导出失败，页面顶部显示 Toast 提示"Export failed, please try again"
```

### AC-003A-016：上传 PDF 后列表出现对应提交记录

```
Given  设计人员通过 REQ-003E 上传了一个包含 3 页图纸的 PDF 文件
When   上传成功后用户进入图纸管理列表页
Then   列表新增 1 行，对应该 PDF 的提交记录（DrawingVersion V0）；
       该行展示上传时填写的 Description 和 Category；
       状态为 "Pending Internal"；
       PDF 内 3 页图纸各自的 Drawing No 和 Drawing Name 不在列表行中展示，需进入详情页查看
```

---

### 7.1 性能

| 指标 | 目标值 | 测量方式 |
|-----|-------|---------|
| 图纸列表首屏加载 | ≤ 2s（100 条数据以内） | Lighthouse / 手动 |

### 7.2 安全

- 鉴权方式：JWT
- 审计：列表查询操作记录操作人与时间

### 7.3 可访问性

- WCAG 等级：AA
- 键盘可达：Filter Search Popover 支持 Tab 键导航；Esc 关闭 Popover
- 屏幕阅读器：是

### 7.4 兼容性

- 浏览器：Chrome 100+、Edge 100+、Safari 15+
- 移动端：不支持（PC 专属）
- 国际化：中英双语

---

## 8. 数据量级

| 维度 | 当前预期 | 1 年后 | 3 年后 |
|-----|---------|-------|-------|
| 单项目图纸数量 | ≤ 500 条 | ≤ 2000 条 | ≤ 5000 条 |

---

## 9. 依赖与外部系统

| 依赖 | 用途 | 集成方式 |
|-----|------|---------|
| REQ-003E-pc | 新建图纸上传入口（[+ Upload Drawing] 触发） | 文档引用 |
| REQ-003F-pc | 上传新版本入口（Actions 列按钮触发） | 文档引用 |
| REQ-003-shared | 接口定义、业务规则 | 文档引用 |
| REQ-007B-pc | RFA No. / Subject of RFA 列数据来源 | 文档引用 |

---

## 10. 灰度与发布策略

- 灰度方式：按项目灰度
- 灰度比例：1 个试点项目 → 全量
- 回滚预案：关闭功能开关，数据无需回滚

## 12. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 | 影响下游文档 |
|-----|------|-------|---------|------------|
| 0.1.0 | 2026-05-04 | agent | 从 REQ-003-pc 按 US-003A-001 拆分初稿（含上传、列表、新版本） | 全部 |
| 0.2.0 | 2026-05-05 | agent | 两级审批升级：状态枚举扩展为 5 态、列表筛选扩展为 6 项 | Frontend、Backend、QA |
| 0.3.0 | 2026-05-23 | agent | 根据 REQ-003E-pc 更新新建图纸流程 | Frontend、Backend、QA |
| 0.3.1 | 2026-05-23 | agent | 列表列移除 Drawing Code / Name，改为 Description；Filter Search 更新 | Frontend、QA |
| 0.4.0 | 2026-05-25 | agent | 文件拆分重构：本文件聚焦"图纸管理列表页"；新建图纸上传移至 REQ-003E-pc；上传新版本移至 REQ-003F-pc；调整 related_to、generate 标记、AC 编号、依赖与外部系统 | Frontend、QA |
| 0.4.1 | 2026-05-26 | agent | 根据 REQ-003E v0.3.x 批量创建模型更新列表逻辑：每行 = 一张独立图纸（来自 PDF 某页）；表格列新增 Drawing Code 与 Drawing Name 为主标识；Description 降为公共批次字段；Filter Search 新增 Drawing Code / Drawing Name 搜索项；更新 §1.1 业务背景说明、§4.1 实体清单、US-003A-LIST-001、AC-003A-011、AC-003A-012、AC-003A-014，新增 AC-003A-016 | Frontend、QA |
| 0.4.2 | 2026-05-25 | agent | 修正列表行含义：每行代表一次**提交记录**（DrawingVersion），而非一个独立 Drawing 实体；同一张图纸的多个版本各占独立一行；更新 §1.1、§4.1 实体清单（突出 DrawingVersion 为列表主体）、§5.1 列表行含义说明、US-003A-LIST-001 背景、AC-003A-011、AC-003A-016 | Frontend、QA |
| 0.4.3 | 2026-05-25 | agent | 修正列表行与图纸页的关系：每行（DrawingVersion）= 一个 PDF 文件；Drawing No / Drawing Name 为 PDF 页级属性，不在列表列中展示；移除表格列 Drawing Code、Drawing Name；Filter Search 字段由 5 项缩减为 3 项（Description、Category、Status）；更新 §1.1、§4.1、§5.1 表格列与行含义说明、US-003A-LIST-001 背景、AC-003A-011、AC-003A-012、AC-003A-014、AC-003A-016 | Frontend、QA |
| 0.4.4 | 2026-05-25 | agent | 明确 Actions 列 6 个按钮的排列顺序与点击行为：从左到右为 View / History / Confirms / Assign / Part Print / Upload New Version；扩展权限矩阵（§3）、重写 §5.1 Actions 列按钮规则为表格形式；AC-003A-005 拆分为 AC-003A-005（按钮排列与行为）+ AC-003A-005B（置灰规则） | Frontend、QA |
| 0.5.0 | 2026-05-27 | agent | 新增导出功能：顶部操作区在 [+ Upload Drawing] 左侧新增 [Export] 按钮（仅图纸管理员可见），支持将当前筛选结果导出为 Excel；新增 US-003A-LIST-002；§3 权限矩阵新增导出行；§5.1 顶部操作区补充 [Export] 按钮规则；新增 AC-003A-017、AC-003A-018 | Frontend、QA |
| 0.4.6 | 2026-05-26 | agent | 补充外部审批结果代码附加显示规则：Status 列在 ACTIVE / EXTERNAL_REJECTED 时以灰色小字附加 (A)–(E) 结果代码；§4.2 新增结果代码映射表与设计说明；新增 AC-003A-011C（结果代码展示）、AC-003A-011D（Status B 时设计人员感知跟进动作） | Frontend、QA |
| 0.4.5 | 2026-05-25 | agent | 关联 REQ-004-pc：[Part Print] 按钮对应 REQ-004-pc 局部更新需求（[+ Markup] 发布局部更新弹窗入口）；新增 related_to REQ-004-pc；更新 §3 权限矩阵（Part Print 仅状态 ACTIVE 时可见）、§5.1 Actions 表（Part Print 行补充引用与状态限制）、AC-003A-005 行为说明 | Frontend、QA |
| 0.5.1 | 2026-08-08 | XIA YING | 按 glossary.md §2 统一角色名称：项目管理员 / 项目管理人员 / 业务人员 / 管理员 → 图纸管理员；Drawing 团队（成员）→ 设计人员；审批人 → 内部审批人；普通业务人员 → 普通用户 | 全部 |

---

## 13. 备注

- 本文档从原 REQ-003A-pc v0.3.1（图纸上传与审批发起）拆分重构，原文档同时承担列表、上传新建、上传新版本三项职责，拆分后职责收窄。
- 新建图纸上传：见 [REQ-003E-pc](REQ-003E-pc.md)
- 上传新版本：见 [REQ-003F-pc](REQ-003F-pc.md)
- 图纸管理其他用户故事：REQ-003C-pc（查看历史/确认）、REQ-003D-pc（分配 SE）

