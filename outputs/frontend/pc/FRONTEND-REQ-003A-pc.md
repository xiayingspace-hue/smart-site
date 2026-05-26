---
doc_type: frontend_spec
req_id: REQ-003A-pc
version: 0.4.6
status: draft
generated_from: REQ-003A-pc.md@0.4.6
ui_spec_ref: UI-REQ-003A-pc.md
generated_at: 2026-05-26
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明 — PC 端图纸管理列表页

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-003A-pc.md](../../../requirements/pc/REQ-003A-pc.md) @ v0.4.5
> **UI 设计参考**: [UI-REQ-003A-pc.md](../../ui/pc/UI-REQ-003A-pc.md)
> **依赖前端文档**:
> - [FRONTEND-REQ-003E-pc.md](./FRONTEND-REQ-003E-pc.md)（[+ Upload Drawing] 弹窗）
> - [FRONTEND-REQ-003F-pc.md](./FRONTEND-REQ-003F-pc.md)（[Upload New Version] 弹窗）
> - [FRONTEND-REQ-003C-pc.md](./FRONTEND-REQ-003C-pc.md)（[History] / [Confirms] 弹框）
> - [FRONTEND-REQ-003D-pc.md](./FRONTEND-REQ-003D-pc.md)（[Assign] 弹框）
> - [FRONTEND-REQ-004-pc.md](./FRONTEND-REQ-004-pc.md)（[Part Print] 弹窗）
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-05-26

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003A-pc @ v0.4.6 |
| UI 设计 | UI-REQ-003A-pc.md |
| 覆盖 Story | US-003A-LIST-001 |
| 覆盖 AC | AC-003A-005、AC-003A-005B、AC-003A-011、AC-003A-011B、AC-003A-011C、AC-003A-011D、AC-003A-012、AC-003A-013、AC-003A-014、AC-003A-015、AC-003A-016 |

---

## 1. 功能概述

图纸管理列表页（Drawing Masterlist）是 PC 管理端图纸操作的入口枢纽。本需求聚焦以下功能：

| 功能模块 | 说明 |
|---------|------|
| DrawingMasterlist（F-001）| 主列表页：表格展示、列定义、状态标签、排序 |
| FilterSearchPopover | Filter Search 浮层：字段、触发、条件计数 |
| ActionsColumn | 六个操作按钮：权限控制与状态限制 |

> 新建图纸上传弹窗见 FRONTEND-REQ-003E-pc；上传新版本弹窗见 FRONTEND-REQ-003F-pc；弹窗实现均在本页面通过按钮触发，不重复实现。

---

## 2. 技术栈

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 |
| UI 库 | Element UI |
| HTTP 客户端 | Axios（封装在 `src/api/`） |
| 状态管理 | Vuex |
| 路由 | Vue Router |
| 构建 | Webpack / Vue CLI |

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── drawing/
│       ├── DrawingMasterlist.vue             # 主列表页（路由入口）
│       └── components/
│           ├── FilterSearchPopover.vue       # Filter Search 浮层（含条件计数）
│           ├── DrawingStatusTag.vue          # 状态标签（5 态颜色复用组件）
│           └── ActionsCell.vue               # Actions 列按钮组（6 按钮权限封装）
├── api/
│   └── drawing.js                           # 图纸列表接口封装
└── store/
    └── modules/
        └── drawingList.js                   # 列表筛选状态、当前激活 Dialog 状态
```

---

## 4. 组件结构

### 4.1 组件树

```
<DrawingMasterlist>                          # 路由页面
├── <el-button> [Filter Search (n)]          # 触发 FilterSearchPopover
├── <FilterSearchPopover>                    # 筛选浮层
│   ├── Description 输入框（模糊搜索）
│   ├── Category 下拉（单选）
│   ├── Status 下拉（单选，6 项）
│   ├── [Search] 按钮
│   └── [Cancel] 按钮
├── <el-button> [+ Upload Drawing]           # 触发 UploadDrawingDialog（REQ-003E）
├── <el-table>                               # 主表格
│   ├── Description 列
│   ├── Category 列
│   ├── RFA No. 列
│   ├── Subject of RFA 列
│   ├── Current Version 列
│   ├── Status 列 → <DrawingStatusTag>
│   ├── Confirmed 列
│   ├── Total Markups 列
│   ├── Last Updated 列（可排序）
│   └── Actions 列 → <ActionsCell>
│       ├── [View]
│       ├── [History]           → HistoryDrawer（REQ-003C）
│       ├── [Confirms]          → ConfirmsDrawer（REQ-003C）
│       ├── [Assign]            → AssignDialog（REQ-003D，仅管理员）
│       ├── [Part Print]        → PartPrintDialog（REQ-004，仅设计人员 + ACTIVE）
│       └── [Upload New Version]→ UploadNewVersionDialog（REQ-003F，仅设计人员）
└── <el-pagination>                          # 分页
```

---

## 5. 组件详细说明

### 5.1 `DrawingMasterlist.vue`（主页面）

**关联 AC**: AC-003A-011 / AC-003A-016

**路由路径**: `/drawing/masterlist`

**导航菜单**: Drawing Management > Drawing Masterlist

**数据加载**:

```javascript
// 挂载时加载；筛选条件变更（点击 [Search]）时重新加载
async loadList(params = this.filterParams) {
  this.loading = true
  try {
    const res = await drawingApi.getDrawingList({
      ...params,
      page: this.currentPage,
      pageSize: this.pageSize,
      sortBy: 'lastUpdated',
      sortDir: 'desc',
    })
    this.tableData = res.data.list
    this.total = res.data.total
  } finally {
    this.loading = false
  }
}
```

**默认排序**: Last Updated 倒序（通过 API 参数 `sortBy=lastUpdated&sortDir=desc` 实现，表头同步设置 `:default-sort="{ prop: 'lastUpdated', order: 'descending' }"`）

**表格列定义**:

```vue
<el-table
  :data="tableData"
  :default-sort="{ prop: 'lastUpdated', order: 'descending' }"
  @sort-change="handleSortChange"
  v-loading="loading"
>
  <el-table-column prop="description"      label="Description"      min-width="200" show-overflow-tooltip />
  <el-table-column prop="category"         label="Category"         width="120" />
  <el-table-column label="RFA No."         width="140">
    <template slot-scope="{ row }">{{ row.rfaNo || '—' }}</template>
  </el-table-column>
  <el-table-column label="Subject of RFA"  min-width="160" show-overflow-tooltip>
    <template slot-scope="{ row }">{{ row.rfaSubject || '—' }}</template>
  </el-table-column>
  <el-table-column prop="currentVersion"   label="Current Version"  width="120" />
  <el-table-column label="Status"          width="180">
    <template slot-scope="{ row }">
      <DrawingStatusTag
        :status="row.approvalStatus"
        :external-approval-result="row.externalApprovalResult"
      />
    </template>
  </el-table-column>
  <el-table-column label="Confirmed"       width="100">
    <template slot-scope="{ row }">
      {{ row.approvalStatus === 'ACTIVE' ? `${row.confirmedCount}/${row.totalSeCount}` : '—' }}
    </template>
  </el-table-column>
  <el-table-column prop="totalMarkups"     label="Total Markups"    width="110" />
  <el-table-column prop="lastUpdated"      label="Last Updated"     width="160" sortable="custom" />
  <el-table-column label="Actions"         width="340" fixed="right">
    <template slot-scope="{ row }">
      <ActionsCell :row="row" @action="handleAction" />
    </template>
  </el-table-column>
</el-table>
```

**RFA 列显示规则**（AC-003A-011）：

`rfaNo` / `rfaSubject` 字段来自 REQ-007B-pc DC 在外部审批发起时写入的 `submissionRefNo` / `submissionSubject`。状态为 `PENDING_INTERNAL` 或 `INTERNAL_REJECTED` 时后端不返回该字段（或返回 null），前端统一展示 `—`。

---

### 5.2 `DrawingStatusTag.vue`（状态标签组件）

**关联 AC**: AC-003A-011B / AC-003A-011C / AC-003A-011D

**Props**:
```ts
interface DrawingStatusTagProps {
  status: 'PENDING_INTERNAL' | 'PENDING_EXTERNAL' | 'ACTIVE' | 'INTERNAL_REJECTED' | 'EXTERNAL_REJECTED'
  /** 外部审批结果代码（A/B/C/D/E），仅 ACTIVE 或 EXTERNAL_REJECTED 时传入；其余状态传 null/undefined */
  externalApprovalResult?: 'A' | 'B' | 'C' | 'D' | 'E' | null
}
```

**实现**:

```javascript
const STATUS_MAP = {
  PENDING_INTERNAL:  { label: 'Pending Internal', color: '#E6A23C', bg: '#fdf6ec' },
  PENDING_EXTERNAL:  { label: 'Pending External', color: '#E6A23C', bg: '#fdf6ec' },
  ACTIVE:            { label: 'Active',            color: '#67C23A', bg: '#f0f9eb' },
  INTERNAL_REJECTED: { label: 'Int. Rejected',     color: '#F56C6C', bg: '#fef0f0' },
  EXTERNAL_REJECTED: { label: 'Ext. Rejected',     color: '#F56C6C', bg: '#fef0f0' },
}

// 结果代码 Tooltip 文案映射（AC-003A-011C）
const APPROVAL_RESULT_LABEL = {
  A: 'A – Approved / No Exception Taken',
  B: 'B – Approved with comment, resubmission required',
  C: 'C – Revise And Resubmit',
  D: 'D – For Record Purpose',
  E: 'E – Others (please state reason)',
}
```

```vue
<template>
  <span class="drawing-status-tag-wrapper">
    <!-- 状态标签 -->
    <span
      class="drawing-status-tag"
      :style="{ color: config.color, background: config.bg, border: `1px solid ${config.color}` }"
      :aria-label="config.label"
    >
      {{ config.label }}
    </span>

    <!-- 外部审批结果代码（AC-003A-011C）：仅 externalApprovalResult 有值时展示 -->
    <el-tooltip
      v-if="externalApprovalResult"
      :content="resultLabel"
      placement="top"
    >
      <span class="drawing-status-result-code">
        ({{ externalApprovalResult }})
      </span>
    </el-tooltip>
  </span>
</template>

<style scoped>
.drawing-status-tag-wrapper {
  display: inline-flex;
  align-items: center;
  gap: 4px;
}
.drawing-status-result-code {
  font-size: 11px;
  color: #909399;  /* --text-color-secondary */
  cursor: default;
}
</style>
```

**computed**:

```javascript
computed: {
  config() { return STATUS_MAP[this.status] },
  resultLabel() {
    return this.externalApprovalResult
      ? APPROVAL_RESULT_LABEL[this.externalApprovalResult]
      : ''
  }
}
```

> **AC-003A-011D 说明（Status B）**：`externalApprovalResult = 'B'` 时，组件渲染 `Active (B)`；此时 `approvalStatus = 'ACTIVE'`，`ActionsCell` 中 `isUploadDisabled = false`，[Upload New Version] 按钮可点——设计人员看到 `Active (B)` 即可感知需要跟进上传新版本。
>
> 该组件为全局复用组件，供图纸列表、版本历史抽屉等多处使用；调用方须同时传入 `status` 与 `externalApprovalResult`（可为 null）。

---

### 5.3 `FilterSearchPopover.vue`（筛选浮层）

**关联 AC**: AC-003A-012 / AC-003A-013 / AC-003A-014 / AC-003A-015

**使用方式**:

```vue
<!-- DrawingMasterlist.vue 中 -->
<el-popover
  v-model="filterPopoverVisible"
  trigger="click"
  placement="bottom-start"
  :close-on-click-modal="false"
>
  <FilterSearchPopover
    :form="filterForm"
    :categories="categoryOptions"
    @search="onFilterSearch"
    @cancel="onFilterCancel"
  />
  <el-button slot="reference" plain>
    {{ filterButtonLabel }}
  </el-button>
</el-popover>
```

**filterButtonLabel 计算**（AC-003A-014）：

```javascript
computed: {
  activeFilterCount() {
    // 统计已生效（已点击 [Search] 的）条件数量
    return [
      this.appliedFilter.description,
      this.appliedFilter.category,
      this.appliedFilter.status,
    ].filter(v => v && v !== 'ALL').length
  },
  filterButtonLabel() {
    return this.activeFilterCount > 0
      ? `Filter Search (${this.activeFilterCount})`
      : 'Filter Search'
  }
}
```

> **关键区分**：`filterForm`（Popover 内表单，实时编辑）与 `appliedFilter`（已点 [Search] 生效的条件）是两个独立对象——点击 Popover 外部时仅关闭浮层，`filterForm` 保留用户填写内容，但 `appliedFilter` 不更新，列表数据不刷新（AC-003A-015）。

**Popover 字段**（AC-003A-012）：

```vue
<el-form :model="form" label-position="top" size="small">

  <!-- Description（文本，模糊搜索）-->
  <el-form-item label="Description">
    <el-input v-model="form.description" placeholder="Search by description..." clearable />
  </el-form-item>

  <!-- Category（下拉单选）-->
  <el-form-item label="Category">
    <el-select v-model="form.category" placeholder="All" clearable>
      <el-option v-for="c in categories" :key="c" :label="c" :value="c" />
    </el-select>
  </el-form-item>

  <!-- Status（下拉单选，6 项，AC-003A-013）-->
  <el-form-item label="Status">
    <el-select v-model="form.status" placeholder="All">
      <el-option label="All"              value="ALL" />
      <el-option label="Active"           value="ACTIVE" />
      <el-option label="Pending Internal" value="PENDING_INTERNAL" />
      <el-option label="Pending External" value="PENDING_EXTERNAL" />
      <el-option label="Int. Rejected"    value="INTERNAL_REJECTED" />
      <el-option label="Ext. Rejected"    value="EXTERNAL_REJECTED" />
    </el-select>
  </el-form-item>

  <div class="popover-footer">
    <el-button size="small" @click="$emit('cancel')">Cancel</el-button>
    <el-button size="small" type="primary" @click="$emit('search', form)">Search</el-button>
  </div>

</el-form>
```

**[Search] 点击行为**（AC-003A-014 / AC-003A-015）：

```javascript
// DrawingMasterlist.vue
onFilterSearch(form) {
  this.appliedFilter = { ...form }        // 更新已生效条件
  this.filterPopoverVisible = false       // 关闭 Popover
  this.currentPage = 1
  this.loadList(this.appliedFilter)       // 触发查询
},
onFilterCancel() {
  this.filterForm = { description: '', category: '', status: 'ALL' }  // 清空表单
  this.appliedFilter = { description: '', category: '', status: 'ALL' }
  this.filterPopoverVisible = false
  this.currentPage = 1
  this.loadList({})                       // 清空条件，重新查询
}
```

**Popover 外部点击关闭**（AC-003A-015）：

使用 `el-popover` 的原生 `trigger="click"` + `v-model`：用户点击触发按钮外的区域时，`el-popover` 自动将 `v-model` 置为 false 关闭浮层，但不触发任何 `@search` 事件——`appliedFilter` 不变，列表不刷新，`filterForm` 因为是 Popover 内组件的绑定数据，状态保留。

---

### 5.4 `ActionsCell.vue`（Actions 列按钮组）

**关联 AC**: AC-003A-005 / AC-003A-005B

**Props**:
```ts
interface ActionsCellProps {
  row: DrawingVersionRow    // 当前行数据，含 approvalStatus、uploaderUserId 等
}
```

**六按钮渲染**（AC-003A-005）：

```vue
<template>
  <div class="actions-cell">

    <!-- 1. [View] 所有角色，始终可点 -->
    <el-button size="mini" @click="emit('action', { type: 'view', row })">View</el-button>

    <!-- 2. [History] 所有角色，始终可点（触发 REQ-003C 历史弹框）-->
    <el-button size="mini" @click="emit('action', { type: 'history', row })">History</el-button>

    <!-- 3. [Confirms] 所有角色，始终可点（触发 REQ-003C 确认历史弹框）-->
    <el-button size="mini" @click="emit('action', { type: 'confirms', row })">Confirms</el-button>

    <!-- 4. [Assign] 仅项目管理员（AC-003A-005）-->
    <el-button
      v-if="isAdmin"
      size="mini"
      @click="emit('action', { type: 'assign', row })"
    >Assign</el-button>

    <!-- 5. [Part Print] 仅设计人员，且状态为 ACTIVE（AC-003A-005 / AC-003A-005B）-->
    <el-button
      v-if="isDesigner && row.approvalStatus === 'ACTIVE'"
      size="mini"
      @click="emit('action', { type: 'partPrint', row })"
    >Part Print</el-button>

    <!-- 6. [Upload New Version] 仅设计人员（AC-003A-005 / AC-003A-005B）-->
    <el-tooltip
      v-if="isDesigner"
      :disabled="!isUploadDisabled"
      content="Cannot upload while drawing is under review"
      placement="top"
    >
      <span><!-- span 包裹，让 disabled 按钮也能触发 tooltip -->
        <el-button
          size="mini"
          :disabled="isUploadDisabled"
          @click="emit('action', { type: 'uploadNewVersion', row })"
        >Upload New Version</el-button>
      </span>
    </el-tooltip>

  </div>
</template>
```

**权限 & 状态 computed**（AC-003A-005B）：

```javascript
computed: {
  isAdmin()    { return this.$store.getters.roles.includes('PROJECT_ADMIN') },
  isDesigner() { return this.$store.getters.roles.includes('DESIGNER') },
  // AC-003A-005B: PENDING_INTERNAL 或 PENDING_EXTERNAL 时置灰
  isUploadDisabled() {
    return ['PENDING_INTERNAL', 'PENDING_EXTERNAL'].includes(this.row.approvalStatus)
  },
}
```

> ⚠️ **[Part Print] 与状态的关系（AC-003A-005）**：状态非 `ACTIVE` 时按钮**隐藏**（不是置灰）；[Upload New Version] 则是可见但置灰——两者处理方式不同。

**Action 事件处理（DrawingMasterlist.vue）**：

```javascript
handleAction({ type, row }) {
  switch (type) {
    case 'view':             this.openViewDialog(row);           break
    case 'history':          this.openHistoryDrawer(row);        break   // REQ-003C
    case 'confirms':         this.openConfirmsDrawer(row);       break   // REQ-003C
    case 'assign':           this.openAssignDialog(row);         break   // REQ-003D
    case 'partPrint':        this.openPartPrintDialog(row);      break   // REQ-004
    case 'uploadNewVersion': this.openUploadNewVersionDialog(row); break // REQ-003F
  }
}
```

---

## 6. 状态管理

### 6.1 Vuex — drawingList.js

```javascript
state: {
  filterParams: { description: '', category: '', status: 'ALL' },
  appliedFilterCount: 0,
  // 各弹窗/抽屉显隐状态
  viewDialogVisible: false,
  historyDrawerVisible: false,
  confirmsDrawerVisible: false,
  assignDialogVisible: false,
  partPrintDialogVisible: false,
  uploadNewVersionDialogVisible: false,
  currentRow: null,
},
```

---

## 7. API 接入

### 7.1 接口清单

| API | 组件 | 说明 |
|-----|------|------|
| `GET /drawing/list` | `DrawingMasterlist` | 图纸列表（含分页、排序、筛选） |
| `GET /drawing/categories` | `FilterSearchPopover` | Category 下拉选项 |

### 7.2 接口封装

```javascript
// drawing.js
export const drawingApi = {
  getDrawingList: (params) => axios.get('/drawing/list', { params }),
  getCategories:  ()       => axios.get('/drawing/categories'),
}
```

### 7.3 请求参数

```typescript
interface DrawingListParams {
  description?: string          // 模糊匹配
  category?:    string
  status?:      string          // 'ALL' 时不传或忽略
  page:         number          // 1-based
  pageSize:     number          // default 20
  sortBy:       'lastUpdated'   // 当前仅支持 lastUpdated
  sortDir:      'asc' | 'desc'
}
```

### 7.4 响应数据结构（列表行）

```typescript
interface DrawingVersionRow {
  id:              string
  description:     string
  category:        string
  rfaNo:           string | null       // 外部审批报审编号，无则 null
  rfaSubject:      string | null       // 外部审批报审主题，无则 null
  currentVersion:  string              // e.g. "V2"
  approvalStatus:  'PENDING_INTERNAL' | 'PENDING_EXTERNAL' | 'ACTIVE'
                   | 'INTERNAL_REJECTED' | 'EXTERNAL_REJECTED'
  /** 外部审批结果代码（A/B/C/D/E）；仅 approvalStatus 为 ACTIVE 或 EXTERNAL_REJECTED 时有值，其余为 null */
  externalApprovalResult: 'A' | 'B' | 'C' | 'D' | 'E' | null
  confirmedCount:  number
  totalSeCount:    number
  totalMarkups:    number
  lastUpdated:     string              // ISO 8601
  uploaderUserId:  string
  fileUrl:         string              // 最新版本 PDF 预签名 URL
}
```

### 7.5 错误处理

| 场景 | 用户可见文案 | 处理位置 |
|------|-----------|---------|
| 列表加载失败（500） | `"加载失败，请刷新重试"` | `DrawingMasterlist` error state |
| Categories 加载失败 | 下拉显示空，不影响主流程 | `FilterSearchPopover` silent fail |

---

## 8. 性能与可访问性

| 项 | 规范 |
|----|------|
| 首屏加载（100 条以内） | ≤ 2s（含 API 响应） |
| 骨架屏 | 超 300ms 时展示 `el-skeleton`（10 行占位） |
| Filter Search Popover | 支持 Tab 键导航；Esc 关闭 Popover |
| 状态标签 | `aria-label` 输出完整状态文案，供屏幕阅读器 |
| 国际化 | 所有文案使用 `$t()` 包裹，支持中英双语 |
| 浏览器兼容 | Chrome 100+、Edge 100+、Safari 15+ |

---

## 9. AC 覆盖矩阵

| AC ID | 描述简述 | 实现位置 |
|-------|---------|---------|
| AC-003A-005 | Actions 6 按钮排列顺序与点击行为（View/History/Confirms/Assign/Part Print/Upload New Version） | `ActionsCell.vue` 按钮顺序与 `handleAction` |
| AC-003A-005B | PENDING_* 状态时 [Upload New Version] 置灰 + Tooltip | `ActionsCell` `isUploadDisabled` + `el-tooltip` |
| AC-003A-011 | 表格 10 列定义，每行 = DrawingVersion，不展示 Drawing No/Name | `DrawingMasterlist` `el-table` 列定义 |
| AC-003A-011B | 5 种状态颜色标签 | `DrawingStatusTag` `STATUS_MAP` |
| AC-003A-011C | 外部审批完成后 Status 标签右侧附加灰色 (A)–(E) 结果代码；外部审批未完成时不显示 | `DrawingStatusTag` `externalApprovalResult` prop + `<el-tooltip>` 结果代码 span；`DrawingVersionRow.externalApprovalResult` 字段 |
| AC-003A-011D | Status B 时显示 `Active (B)`；[Upload New Version] 在 ACTIVE 状态可点 | `DrawingStatusTag`（Status B → `Active (B)`）；`ActionsCell.isUploadDisabled`（`ACTIVE` 时为 false） |
| AC-003A-012 | Filter Search 3 个字段（Description/Category/Status），不含 Drawing Code/Name | `FilterSearchPopover` 表单字段 |
| AC-003A-013 | Status 下拉 6 个选项（All/Active/Pending Internal/Pending External/Int. Rejected/Ext. Rejected） | `FilterSearchPopover` `el-select` options |
| AC-003A-014 | 按钮文案动态显示条件计数 `Filter Search (n)` | `DrawingMasterlist` `filterButtonLabel` computed |
| AC-003A-015 | 点击 Popover 外部：关闭浮层，条件保留，列表不刷新 | `el-popover` v-model 机制 + `appliedFilter` / `filterForm` 分离 |
| AC-003A-016 | 上传 PDF 后列表新增 1 行（V0，Pending Internal），不展示页级属性 | 上传成功回调触发 `loadList()`；列定义不含 Drawing No/Name |

---

## 10. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.4.6 | 2026-05-26 | agent | 同步 REQ-003A-pc@0.4.6：① `DrawingStatusTag` 新增 `externalApprovalResult` prop（'A'–'E' \| null），渲染灰色 `(A)`–`(E)` 结果代码 + el-tooltip 完整描述；② `DrawingMasterlist` Status 列传入 `externalApprovalResult` 字段；③ `DrawingVersionRow` 接口新增 `externalApprovalResult` 字段；④ AC 覆盖矩阵新增 AC-003A-011C / 011D | UI、QA |
| 0.4.5 | 2026-05-26 | agent | 基于 REQ-003A-pc@0.4.5 首次生成。涵盖主列表 10 列（含 RFA No./Subject of RFA）、5 态状态标签组件、FilterSearchPopover（3 字段 + 条件计数 + 外部点击不触发查询逻辑）、ActionsCell 6 按钮（权限控制 + [Part Print] 隐藏逻辑 + [Upload New Version] 置灰 Tooltip）、API 封装、AC 覆盖矩阵 |
