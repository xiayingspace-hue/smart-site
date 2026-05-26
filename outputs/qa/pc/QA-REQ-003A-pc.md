---
doc_type: qa_spec
req_id: REQ-003A-pc
req_title: "PC 端 — 图纸管理列表页"
version: 0.4.6
status: draft
generated_from: REQ-003A-pc@0.4.6
generated_at: 2026-05-26
owner: ""
---

# QA 测试说明：PC 端 — 图纸管理列表页

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - 弹窗/抽屉内部流程（Upload Drawing、Upload New Version、History、Confirms、Assign、Part Print）由各自 QA 文档覆盖，本文档仅验证从列表页触发打开的行为（即按钮可点/置灰/隐藏）。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003A-pc @ v0.4.5 |
| 依赖需求 | REQ-003-shared、REQ-003E-pc、REQ-003F-pc、REQ-003C-pc、REQ-003D-pc、REQ-004-pc、REQ-007B-pc |
| 覆盖 Story | US-003A-LIST-001 |
| 覆盖 AC | AC-003A-005、AC-003A-005B、AC-003A-011、AC-003A-011B、AC-003A-011C、AC-003A-011D、AC-003A-012、AC-003A-013、AC-003A-014、AC-003A-015、AC-003A-016 |
| 测试平台 | PC（Chrome 100+ / Edge 100+ / Safari 15+，1280px+） |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证图纸管理列表页的表格列完整性与行含义、5 种状态标签颜色、**外部审批结果代码 (A)–(E) 附加显示**（AC-003A-011C）、**Status B 时设计人员可感知跟进动作**（AC-003A-011D）、Filter Search 浮层字段与条件计数、点击 Popover 外部不触发查询、Actions 列六按钮的权限控制与状态限制（特别是 [Part Print] 隐藏逻辑和 [Upload New Version] 置灰逻辑）、以及上传后列表即时更新等功能的正确性。

---

## 2. 测试策略

### 2.1 测试范围

**必测**：
- 表格列完整性：Description、Category、RFA No.、Subject of RFA、Current Version、Status、Confirmed、Total Markups、Last Updated、Actions 共 10 列
- 每行代表 DrawingVersion（一次 PDF 上传），列表中不展示 Drawing No / Drawing Name
- RFA No. / Subject of RFA 字段的显示规则（PENDING_INTERNAL / INTERNAL_REJECTED 时显示 `—`）
- Confirmed 列条件显示（仅 ACTIVE 时显示 x/y）
- 5 种状态标签颜色（橙/绿/红）
- **外部审批结果代码附加显示**：ACTIVE / EXTERNAL_REJECTED 时 Status 标签右侧附加灰色 (A)–(E)；PENDING_* / INTERNAL_REJECTED 时不附加；Tooltip 显示完整描述
- **Status B 感知**：显示 `Active (B)` 时 [Upload New Version] 可点
- Filter Search：3 字段、Status 6 选项、条件计数、点击外部不触发查询
- Actions 列 6 按钮：顺序、权限、状态限制
- [Part Print] 状态非 ACTIVE 时隐藏
- [Upload New Version] PENDING_* 时置灰 + Tooltip
- 上传成功后列表新增行
- 列表默认排序 Last Updated 倒序

**不测（本期）**：
- 弹窗/抽屉内部流程（各自 QA 文档覆盖）
- PDF 内页级 Drawing No / Drawing Name 展示（详情页覆盖）
- 分页逻辑（通用组件，单独测试）

### 2.2 测试金字塔

| 层级 | 占比 | 说明 |
|-----|-----|------|
| 单元测试 | 50% | DrawingStatusTag 颜色映射、filterButtonLabel computed、isUploadDisabled 逻辑 |
| 集成测试 | 35% | 列表 API 驱动渲染、筛选查询参数正确性、conditionCount 同步 |
| E2E 测试 | 15% | 完整筛选 → 查询 → 操作按钮触发流程 |

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 优先级 |
|--------|---------|-------|
| SC-003A-001 | 进入图纸管理列表页，查看默认列表 | P1 |
| SC-003A-002 | 使用 Filter Search 筛选，验证条件计数与查询 | P1 |
| SC-003A-003 | 查看不同状态图纸行的 Actions 按钮展示 | P1 |
| SC-003A-004 | 上传图纸成功后，列表即时出现新行 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-003A-E01 | 列表接口返回 500 | — |
| SC-003A-E02 | 点击 Popover 外部：条件保留，列表不刷新 | AC-003A-015 |

### 3.3 权限场景

| 场景 ID | 角色 | 验证点 |
|--------|------|-------|
| SC-003A-P01 | 项目管理员 | [Assign] 可见；[Part Print] 不可见；[Upload New Version] 不可见 |
| SC-003A-P02 | 设计人员 | [Assign] 不可见；[Part Print] 仅 ACTIVE 时可见；[Upload New Version] 可见 |
| SC-003A-P03 | 内部审批人 | [Assign] 不可见；[Part Print] 不可见；[Upload New Version] 不可见 |

### 3.4 状态相关场景

| 场景 ID | approvalStatus | externalApprovalResult | 测试核心 |
|--------|---------------|----------------------|---------|
| SC-003A-ST01 | `ACTIVE` | A | Confirmed 显示 x/y；[Part Print] 可见；[Upload New Version] 可点；Status 显示 `Active (A)` |
| SC-003A-ST01B | `ACTIVE` | B | Status 显示 `Active (B)`；[Upload New Version] 可点（设计人员可跟进上传新版本） |
| SC-003A-ST01D | `ACTIVE` | D | Status 显示 `Active (D)` |
| SC-003A-ST02 | `PENDING_INTERNAL` | null | RFA No. = —；[Upload New Version] 置灰；[Part Print] 隐藏；Status 不附加代码 |
| SC-003A-ST03 | `PENDING_EXTERNAL` | null | RFA No. 有值；[Upload New Version] 置灰；[Part Print] 隐藏；Status 不附加代码 |
| SC-003A-ST04 | `INTERNAL_REJECTED` | null | RFA No. = —；[Part Print] 隐藏；Status 不附加代码 |
| SC-003A-ST05 | `EXTERNAL_REJECTED` | C | RFA No. 有值；[Part Print] 隐藏；Status 显示 `Ext. Rejected (C)` |
| SC-003A-ST05E | `EXTERNAL_REJECTED` | E | Status 显示 `Ext. Rejected (E)` |

---

## 4. 测试用例

### TC 组 1：列表基础展示

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-001 | AC-003A-011 | 项目中存在多条提交记录（含各种状态） | 进入 Drawing Management > Drawing Masterlist | 页面加载成功；表格显示 10 列：Description、Category、RFA No.、Subject of RFA、Current Version、Status、Confirmed、Total Markups、Last Updated、Actions；每行代表一次 PDF 上传（DrawingVersion）；列表中**不含** Drawing No 和 Drawing Name 列 |
| TC-003A-002 | AC-003A-011 | 同上 | 观察列表默认排序 | 列表按 Last Updated 倒序排列；Last Updated 列表头显示排序箭头 |
| TC-003A-003 | AC-003A-011 | 存在 ACTIVE 状态图纸行（confirmedCount=2，totalSeCount=5） | 查看该行 Confirmed 列 | 显示 `2/5` |
| TC-003A-004 | AC-003A-011 | 存在 PENDING_INTERNAL 状态图纸行 | 查看该行 Confirmed 列 | 显示 `—` |
| TC-003A-005 | AC-003A-011 | 存在 PENDING_EXTERNAL 状态图纸行（已有 RFA No. = "RFA-2024-001"，Subject = "Foundation Plan"） | 查看该行 RFA No. 和 Subject of RFA 列 | RFA No. 显示 `RFA-2024-001`；Subject of RFA 显示 `Foundation Plan` |
| TC-003A-006 | AC-003A-011 | 存在 PENDING_INTERNAL 状态图纸行（尚无 RFA No.） | 查看该行 RFA No. 和 Subject of RFA 列 | 两列均显示 `—` |
| TC-003A-007 | AC-003A-011 | 存在 INTERNAL_REJECTED 状态图纸行 | 查看该行 RFA No. 列 | 显示 `—` |

### TC 组 2：状态标签颜色

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-008 | AC-003A-011B | 列表含 5 种状态各一行 | 检查 Status 列标签 | `PENDING_INTERNAL` → 橙色标签 "Pending Internal"；`PENDING_EXTERNAL` → 橙色标签 "Pending External"；`ACTIVE` → 绿色标签 "Active"；`INTERNAL_REJECTED` → 红色标签 "Int. Rejected"；`EXTERNAL_REJECTED` → 红色标签 "Ext. Rejected" |

### TC 组 12：外部审批结果代码附加显示

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-042 | AC-003A-011C | 图纸外部审批结果为 A（ACTIVE） | 查看 Status 列 | 显示 🟢 `Active` 加灰色小字 `(A)`；整体在同一行内联排列 |
| TC-003A-043 | AC-003A-011C | 图纸外部审批结果为 B（ACTIVE） | 查看 Status 列 | 显示 🟢 `Active` 加灰色小字 `(B)` |
| TC-003A-044 | AC-003A-011C | 图纸外部审批结果为 C（EXTERNAL_REJECTED） | 查看 Status 列 | 显示 🔴 `Ext. Rejected` 加灰色小字 `(C)` |
| TC-003A-045 | AC-003A-011C | 图纸外部审批结果为 D（ACTIVE） | 查看 Status 列 | 显示 🟢 `Active` 加灰色小字 `(D)` |
| TC-003A-046 | AC-003A-011C | 图纸外部审批结果为 E（EXTERNAL_REJECTED） | 查看 Status 列 | 显示 🔴 `Ext. Rejected` 加灰色小字 `(E)` |
| TC-003A-047 | AC-003A-011C | 图纸处于 PENDING_INTERNAL 状态（尚无外部审批） | 查看 Status 列 | 显示 `Pending Internal`，**不附加**任何结果代码 |
| TC-003A-048 | AC-003A-011C | 图纸处于 PENDING_EXTERNAL 状态（外部审批进行中） | 查看 Status 列 | 显示 `Pending External`，**不附加**任何结果代码 |
| TC-003A-049 | AC-003A-011C | 图纸处于 INTERNAL_REJECTED 状态 | 查看 Status 列 | 显示 `Int. Rejected`，**不附加**任何结果代码 |
| TC-003A-050 | AC-003A-011C | 图纸外部审批结果为 A（ACTIVE） | 鼠标悬停结果代码 `(A)` | Tooltip 显示 `"A – Approved / No Exception Taken"` |
| TC-003A-051 | AC-003A-011C | 图纸外部审批结果为 B（ACTIVE） | 鼠标悬停结果代码 `(B)` | Tooltip 显示 `"B – Approved with comment, resubmission required"` |
| TC-003A-052 | AC-003A-011C | 图纸外部审批结果为 C（EXTERNAL_REJECTED） | 鼠标悬停结果代码 `(C)` | Tooltip 显示 `"C – Revise And Resubmit"` |
| TC-003A-053 | AC-003A-011C | 图纸外部审批结果为 E（EXTERNAL_REJECTED） | 鼠标悬停结果代码 `(E)` | Tooltip 显示 `"E – Others (please state reason)"` |

### TC 组 13：Status B — 设计人员感知跟进动作

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-054 | AC-003A-011D | 图纸外部审批结果为 B，当前用户为**设计人员** | 查看 Status 列与 Actions 列 | Status 列显示 `Active (B)`（绿色标签 + 灰色代码）；[Upload New Version] 按钮**可见且可点**（不置灰） |
| TC-003A-055 | AC-003A-011D | 图纸外部审批结果为 B，当前用户为**设计人员** | 点击 [Upload New Version] | 上传新版本弹框**正常弹出**，可发起新版本上传 |
| TC-003A-056 | AC-003A-011D | 图纸外部审批结果为 A，当前用户为**设计人员** | 查看 Status 列与 Actions 列 | Status 列显示 `Active (A)`；[Upload New Version] 同样**可点**（ACTIVE 状态下可上传） |

### TC 组 3：Filter Search 浮层字段

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-009 | AC-003A-012 | 图纸列表页已加载 | 点击 [Filter Search] 按钮 | 浮层弹出，包含且仅包含 3 个字段：Description（文本输入框，模糊搜索）、Category（下拉单选）、Status（下拉单选）；**不包含** Drawing Code 或 Drawing Name 字段 |
| TC-003A-010 | AC-003A-013 | Filter Search Popover 已打开 | 点击 Status 下拉 | 下拉列表包含且仅包含 6 项：All、Active、Pending Internal、Pending External、Int. Rejected、Ext. Rejected |
| TC-003A-011 | AC-003A-012 | Popover 已打开 | 检查底部操作按钮 | 底部有 [Cancel] 和 [Search] 两个按钮 |

### TC 组 4：Filter Search 条件计数与查询

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-012 | AC-003A-014 | 初始状态，无筛选条件 | 查看列表页顶部按钮文案 | 按钮显示 `Filter Search`（无括号计数） |
| TC-003A-013 | AC-003A-014 | Popover 已打开 | 填写 Description = "Foundation"，Status = "Active"，点击 [Search] | Popover 关闭；按钮文案变为 `Filter Search (2)`；列表按该条件刷新 |
| TC-003A-014 | AC-003A-014 | 已有 2 个筛选条件生效 | 打开 Popover，点击 [Cancel] | Popover 关闭；所有条件清空；按钮恢复为 `Filter Search`；列表显示全部数据 |
| TC-003A-015 | AC-003A-014 | 已有 2 个筛选条件生效，再次打开 Popover | 只修改 Category 字段（总条件数变为 3），点击 [Search] | 按钮文案变为 `Filter Search (3)` |

### TC 组 5：点击 Popover 外部不触发查询

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-016 | AC-003A-015 | 列表当前无筛选条件（显示全部数据） | 打开 Popover，填写 Description = "Bridge"，**不点击 [Search]**，直接点击 Popover 外部区域 | Popover 关闭；列表数据**不刷新**（仍显示全部数据）；按钮文案仍为 `Filter Search` |
| TC-003A-017 | AC-003A-015 | 同上，Popover 已关闭 | 再次打开 Popover | Popover 内 Description 字段仍显示 "Bridge"（填写内容保留） |

### TC 组 6：Actions 列按钮顺序与行为

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-018 | AC-003A-005 | 当前用户为**项目管理员**，图纸状态为 ACTIVE | 查看该行 Actions 列 | 从左到右顺序：[View]、[History]、[Confirms]、[Assign]；不显示 [Part Print] 和 [Upload New Version] |
| TC-003A-019 | AC-003A-005 | 当前用户为**设计人员**，图纸状态为 ACTIVE | 查看该行 Actions 列 | 从左到右顺序：[View]、[History]、[Confirms]、[Part Print]、[Upload New Version]；不显示 [Assign] |
| TC-003A-020 | AC-003A-005 | 当前用户为**内部审批人**，图纸状态为 PENDING_INTERNAL | 查看该行 Actions 列 | 显示：[View]、[History]、[Confirms]；不显示 [Assign]、[Part Print]、[Upload New Version] |
| TC-003A-021 | AC-003A-005 | 当前用户为设计人员，图纸状态为 ACTIVE | 点击 [View] | 打开该提交记录最新版本的 PDF 预览 |
| TC-003A-022 | AC-003A-005 | 当前用户为任意角色 | 点击 [History] | 弹出历史记录弹框（REQ-003C 对应组件） |
| TC-003A-023 | AC-003A-005 | 当前用户为任意角色 | 点击 [Confirms] | 弹出 SE 确认历史弹框（REQ-003C 对应组件） |
| TC-003A-024 | AC-003A-005 | 当前用户为项目管理员 | 点击 [Assign] | 弹出分配 SE 的弹框（REQ-003D） |
| TC-003A-025 | AC-003A-005 | 当前用户为设计人员，图纸状态为 ACTIVE | 点击 [Part Print] | 弹出图纸局部更新弹窗（REQ-004 [+ Markup]） |
| TC-003A-026 | AC-003A-005 | 当前用户为设计人员，图纸状态为 ACTIVE | 点击 [Upload New Version] | 弹出上传新版本弹框（REQ-003F） |

### TC 组 7：[Part Print] 状态控制

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-027 | AC-003A-005 | 当前用户为设计人员，图纸状态为 ACTIVE | 查看 Actions 列 | [Part Print] 按钮**可见且可点** |
| TC-003A-028 | AC-003A-005 | 当前用户为设计人员，图纸状态为 PENDING_INTERNAL | 查看 Actions 列 | [Part Print] 按钮**不显示**（隐藏，非置灰） |
| TC-003A-029 | AC-003A-005 | 当前用户为设计人员，图纸状态为 PENDING_EXTERNAL | 查看 Actions 列 | [Part Print] 按钮**不显示** |
| TC-003A-030 | AC-003A-005 | 当前用户为设计人员，图纸状态为 INTERNAL_REJECTED | 查看 Actions 列 | [Part Print] 按钮**不显示** |
| TC-003A-031 | AC-003A-005 | 当前用户为设计人员，图纸状态为 EXTERNAL_REJECTED | 查看 Actions 列 | [Part Print] 按钮**不显示** |

### TC 组 8：[Upload New Version] 置灰规则

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-032 | AC-003A-005B | 当前用户为设计人员，图纸状态为 PENDING_INTERNAL | 查看 Actions 列 [Upload New Version] 按钮 | 按钮**可见但置灰**（disabled）；鼠标悬停显示 Tooltip："Cannot upload while drawing is under review"（或同等提示文案） |
| TC-003A-033 | AC-003A-005B | 当前用户为设计人员，图纸状态为 PENDING_EXTERNAL | 查看 Actions 列 [Upload New Version] 按钮 | 按钮**可见但置灰**；鼠标悬停显示 Tooltip |
| TC-003A-034 | AC-003A-005B | 当前用户为设计人员，图纸状态为 PENDING_INTERNAL | 点击置灰的 [Upload New Version] | 弹窗**不打开**，无响应 |
| TC-003A-035 | AC-003A-005B | 当前用户为设计人员，图纸状态为 ACTIVE | 查看 Actions 列 [Upload New Version] 按钮 | 按钮**可见且可点**（不置灰） |
| TC-003A-036 | AC-003A-005B | 当前用户为设计人员，图纸状态为 INTERNAL_REJECTED | 查看 Actions 列 [Upload New Version] 按钮 | 按钮**可见且可点** |

### TC 组 9：上传后列表更新

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-037 | AC-003A-016 | 设计人员在列表页点击 [+ Upload Drawing]，上传包含 3 页图纸的 PDF 文件 | 上传完成后查看图纸列表 | 列表**新增 1 行**（DrawingVersion V0）；该行 Description 和 Category 为上传时填写的值；Status 显示 "Pending Internal"（橙色）；列表中**不展示** Drawing No 或 Drawing Name |
| TC-003A-038 | AC-003A-016 | 同上，新行已出现 | 查看新行 RFA No. 和 Subject of RFA | 两列均显示 `—`（首版尚未进入外部审批） |

### TC 组 10：权限控制

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-039 | AC-003A-005 | 当前用户为**项目管理员** | 查看任意行 Actions 列 | [Assign] 可见；[Part Print] 不可见；[Upload New Version] 不可见 |
| TC-003A-040 | AC-003A-005 | 当前用户为**内部审批人** | 查看任意行 Actions 列 | 只显示 [View]、[History]、[Confirms]；其他按钮均不可见 |

### TC 组 11：异常处理

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003A-041 | — | 模拟 `/drawing/list` 接口返回 500 | 进入图纸管理列表页 | 列表区域显示错误状态，提示"加载失败，请刷新重试"；不显示空表格 |

---

## 5. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-003A-001 | 列表 10 列顺序 | Description / Category / RFA No. / Subject of RFA / Current Version / Status / Confirmed / Total Markups / Last Updated / Actions |
| UI-003A-002 | 状态标签颜色 | 橙色（Pending）/ 绿色（Active）/ 红色（Rejected）颜色值符合 §4.2 规范 |
| UI-003A-003 | Actions 按钮顺序（设计人员 + ACTIVE） | View / History / Confirms / Part Print / Upload New Version |
| UI-003A-004 | [Upload New Version] 置灰样式 | 按钮视觉上明显为不可点状态；Tooltip 在鼠标悬停 300ms 后出现 |
| UI-003A-005 | Filter Search Popover 位置 | 在 [Filter Search] 按钮下方对齐弹出 |
| UI-003A-006 | 列表骨架屏 | 加载超 300ms 时显示骨架占位行 |
| UI-003A-007 | Status 结果代码视觉 | 结果代码 `(A)`–`(E)` 以灰色小字内联显示在状态标签右侧，间距 4px；字号小于状态标签主文字 |
| UI-003A-008 | 结果代码 Tooltip | 鼠标悬停结果代码时弹出完整描述文案（如 `"B – Approved with comment, resubmission required"`） |

---

## 6. 非功能测试

| 类型 | 测试点 | 期望 |
|------|-------|------|
| 性能 | 列表首屏加载（100 条以内） | ≤ 2s（含 API 响应） |
| 安全 | JWT 鉴权 | 无 Token 时重定向登录页 |
| 可访问性 | Filter Search Popover | 支持 Tab 键导航字段；Esc 关闭 Popover |
| 可访问性 | 状态标签 | aria-label 输出完整状态文案 |
| 国际化 | 切换中/英语言 | 列名、状态标签、按钮、Tooltip 正确切换 |

---

## 7. 集成验收检查清单（上线前）

- [ ] `/drawing/list` 接口返回 10 列所需全部字段（含 rfaNo / rfaSubject，PENDING_INTERNAL 和 INTERNAL_REJECTED 状态下为 null）
- [ ] **`/drawing/list` 响应包含 `externalApprovalResult` 字段**：ACTIVE / EXTERNAL_REJECTED 时返回 'A'–'E'；其余状态返回 null
- [ ] Confirmed 列：`confirmedCount` / `totalSeCount` 仅在 ACTIVE 状态返回有效值，其他状态返回 null 或 0
- [ ] Filter Search 查询参数正确传递至后端（description / category / status / sortBy / sortDir）
- [ ] Status 筛选为 "All"（或不传）时返回全部状态数据
- [ ] Description 字段模糊搜索有效（不区分大小写）
- [ ] 上传图纸成功后，列表接口刷新可见新行（V0，状态 PENDING_INTERNAL，`externalApprovalResult = null`）
- [ ] [Part Print] / [Upload New Version] 权限后端再次校验（前端隐藏/置灰仅为 UX 层）

---

## 8. 测试数据

| 数据 | 说明 |
|------|-----|
| 项目管理员账号 | `admin_01` |
| 设计人员账号 | `designer_01` |
| 内部审批人账号 | `approver_01` |
| 图纸数据集 | 项目 PROJ-001 含 5 种状态各一条 DrawingVersion；ACTIVE 行 confirmedCount=2，totalSeCount=5，**externalApprovalResult='A'**；另需补充 externalApprovalResult='B' 的 ACTIVE 行一条；PENDING_EXTERNAL 行有 RFA No. = "RFA-2024-001" 和 Subject = "Foundation Plan"，externalApprovalResult=null；EXTERNAL_REJECTED 行 externalApprovalResult='C' |
| 上传测试文件 | 3 页图纸 PDF（< 50MB） |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.4.6 | 2026-05-26 | agent | 同步 REQ-003A-pc@0.4.6：① 覆盖 AC 新增 011C / 011D；② 测试目标、必测范围、状态场景表更新（新增结果代码列）；③ 新增 TC 组 12（外部审批结果代码附加显示，TC-003A-042–053，覆盖 5 种代码展示 + 3 种不展示 + 5 条 Tooltip）；④ 新增 TC 组 13（Status B 设计人员跟进，TC-003A-054–056）；⑤ 界面验收新增 UI-003A-007 / UI-003A-008；⑥ 集成检查清单新增 externalApprovalResult 字段验证项；⑦ 测试数据补充 externalApprovalResult 字段 |
| 0.4.5 | 2026-05-26 | agent | 基于 REQ-003A-pc@0.4.5 首次生成。涵盖 AC-003A-005/005B/011/011B/012/013/014/015/016 全部 9 个 AC；TC 组 1（列表基础展示含 RFA 规则）、TC 组 2（状态标签）、TC 组 3–5（Filter Search）、TC 组 6（Actions 6 按钮）、TC 组 7（Part Print 隐藏）、TC 组 8（Upload New Version 置灰）、TC 组 9（上传后列表更新）、TC 组 10–11（权限 + 异常）|
