# 前端开发说明文档 — PC 管理端设备加油管理

> **来源需求**: [REQ-002-pc.md](../../../requirements/pc/REQ-002-pc.md) + [REQ-002-shared.md](../../../requirements/shared/REQ-002-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

本模块实现 PC 管理端的设备加油管理功能，涵盖：

| 功能模块 | 说明 |
|---------|------|
| 加油记录列表页 | 多条件筛选表格，含统计摘要卡片、排序、分页，支持权限控制的编辑/删除 |
| 统计摘要卡片 | 列表上方实时汇总：总加油量、总费用、记录数、人均费用 |
| 新增 / 编辑弹窗 | 右侧抽屉或 Dialog，完整表单校验，单价自动计算总费用 |
| 详情弹窗 | 所有字段 + 图片预览，可跳转编辑 / 删除 |
| 油耗统计页面 | 趋势图（ECharts）+ 设备/分包商排行条形图 + 明细表格 |
| 数据导出 | 按当前筛选条件导出 Excel，文件名含项目名 + 日期 |
| 批量删除 | 勾选多条记录批量删除（需权限） |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 + Vue Router + Vuex |
| UI 组件库 | Element UI 2.x |
| 图表库 | ECharts 5.x（通过 `vue-echarts` 封装） |
| HTTP | Axios，统一封装 `request.js`，自动附带 `Authorization` / `X-Tenant-Id` / `Project-Id` / `lang` |
| 状态管理 | Vuex module，模块名 `refueling` |
| 路由 | `/project/:projectId/equipment/refueling`（记录列表），`/project/:projectId/equipment/refueling/statistics`（统计页） |
| 权限指令 | `v-permission="'equipment:refueling:create'"` 等 |
| 国际化 | Vue I18n |
| 代码风格 | ESLint + Prettier，与项目现有配置一致 |

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── equipment/
│       └── refueling/
│           ├── index.vue                    # 加油记录列表页
│           ├── statistics.vue               # 油耗统计页
│           └── components/
│               ├── FilterPanel.vue          # 筛选面板
│               ├── SummaryCards.vue         # 统计摘要卡片
│               ├── RefuelingTable.vue        # 加油记录表格
│               ├── RefuelingFormDrawer.vue  # 新增/编辑抽屉
│               ├── RefuelingDetailDialog.vue # 详情弹窗
│               ├── FuelTrendChart.vue       # 趋势图
│               ├── EquipmentRankChart.vue   # 设备排行图
│               └── SubcontractorRankChart.vue # 分包商排行图
├── store/
│   └── modules/
│       └── refueling.js                    # Vuex module（可选，轻量场景可不用）
├── api/
│   └── refueling.js                        # 加油管理相关 API 封装
└── utils/
    └── refueling-helper.js                 # 枚举映射、格式化工具
```

---

## 4. 页面与组件详细说明

### 4.1 加油记录列表页 `index.vue`

#### 4.1.1 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `filterForm` | Object | 筛选表单（未提交）：`{ equipmentId, subcontractorId, fuelType, refuelingMethod, startDate, endDate, creatorId, supplier, minCost, maxCost, keyword }` |
| `activeFilter` | Object | 当前生效的筛选条件（点击 Search 后同步） |
| `tableData` | Array | 加油记录列表 |
| `total` | Number | 总记录数 |
| `pageNo` | Number | 当前页，默认 1 |
| `pageSize` | Number | 每页条数，默认 20 |
| `orderBy` | String | 排序字段，默认 `'refuelingDate'` |
| `orderDirection` | String | `'ASC'` / `'DESC'`，默认 `'DESC'` |
| `loading` | Boolean | 表格 loading |
| `summary` | Object | 统计摘要数据 |
| `selectedIds` | Array | 已勾选记录 ID |
| `formDrawerVisible` | Boolean | 新增/编辑抽屉显示 |
| `formMode` | String | `'create'` / `'edit'` |
| `editingRecord` | Object \| null | 当前编辑的记录 |
| `detailVisible` | Boolean | 详情弹窗显示 |
| `detailRecord` | Object \| null | 当前查看详情的记录 |

#### 4.1.2 核心方法

| 方法 | 说明 |
|------|------|
| `fetchList()` | 调用 `/equipment/refueling/page`，使用 `activeFilter` + 分页 + 排序参数；同时触发 `fetchSummary()` |
| `fetchSummary()` | 调用统计摘要接口（或从列表响应中提取），更新 `summary` |
| `handleSearch()` | 同步 `filterForm` → `activeFilter`，重置页码为 1，调用 `fetchList()` |
| `handleReset()` | 清空 `filterForm` 和 `activeFilter`，重置页码，调用 `fetchList()` |
| `handleSortChange({ prop, order })` | 更新 `orderBy` 和 `orderDirection`，调用 `fetchList()` |
| `handleNew()` | `formMode = 'create'`，`editingRecord = null`，`formDrawerVisible = true` |
| `handleEdit(row)` | `formMode = 'edit'`，`editingRecord = row`，`formDrawerVisible = true` |
| `handleDelete(id)` | 弹出确认框 → 调用删除 API → 刷新列表 |
| `handleBatchDelete()` | 检查 `selectedIds` 非空 → 弹确认框 → 批量删除 → 刷新列表 |
| `handleExport()` | 调用导出 API，以当前 `activeFilter` 为参数，触发文件下载 |
| `handleViewDetail(row)` | `detailRecord = row`，`detailVisible = true` |
| `handleFormSuccess()` | 关闭抽屉，调用 `fetchList()`，显示成功 Toast |

---

### 4.2 筛选面板 `FilterPanel.vue`

#### 4.2.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `value` | Object | 双向绑定筛选表单，`v-model` |

#### 4.2.2 Emits

| 事件 | 说明 |
|------|------|
| `search` | 用户点击 [Search] |
| `reset` | 用户点击 [Reset] |

#### 4.2.3 筛选控件说明

| 字段 | 控件类型 | 数据来源 |
|------|---------|---------|
| Equipment | `el-select`（支持搜索） | `/equipment/page`（当前项目） |
| Subcontractor | `el-select` | `/subcontractor/page`（当前项目） |
| Fuel Type | `el-select`，枚举 | 常量 `FUEL_TYPE_OPTIONS` |
| Method | `el-select`，枚举 | 常量 `REFUELING_METHOD_OPTIONS` |
| Date Range | `el-date-picker` type="daterange" | — |
| Creator | `el-select`（搜索） | `/system/user/page`（当前项目用户） |
| Supplier | `el-input` | — |
| Cost Range | 两个 `el-input-number` | — |

---

### 4.3 统计摘要卡片 `SummaryCards.vue`

#### 4.3.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `summary` | Object | `{ totalFuelAmount, totalCost, recordCount, avgCostPerRecord }` |
| `loading` | Boolean | 骨架屏 loading |

#### 4.3.2 卡片列表

| 卡片标题 | 字段 | 格式 |
|---------|------|------|
| Total Fuel (L) | `totalFuelAmount` | 千分位，2 位小数 |
| Total Cost | `totalCost` | `$` 前缀，千分位，2 位小数 |
| Record Count | `recordCount` | 整数 |
| Avg Cost / Record | `avgCostPerRecord` | `$` 前缀，千分位，2 位小数 |

---

### 4.4 加油记录表格 `RefuelingTable.vue`

#### 4.4.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `data` | Array | 表格数据 |
| `loading` | Boolean | 表格 loading |
| `total` | Number | 总记录数 |
| `pageNo` | Number | 当前页 |
| `pageSize` | Number | 每页条数 |

#### 4.4.2 Emits

| 事件 | 参数 | 说明 |
|------|------|------|
| `sort-change` | `{ prop, order }` | 列排序变更 |
| `page-change` | `{ pageNo, pageSize }` | 分页变更 |
| `selection-change` | `ids[]` | 勾选变更 |
| `edit` | `row` | 点击编辑 |
| `delete` | `id` | 点击删除 |
| `view-detail` | `row` | 点击行 / 设备名称 |

#### 4.4.3 列定义

| 列名 | 字段 | 宽度 | 可排序 | 备注 |
|------|------|------|--------|------|
| 勾选框 | — | 50px | — | `el-table-column type="selection"` |
| Equipment | `equipmentName` | 150px | ✅ | 点击链接触发 `view-detail` |
| Code | `equipmentCode` | 100px | ✅ | |
| Subcontractor | `subcontractorName` | 150px | ✅ | |
| Date | `refuelingDate` | 120px | ✅ | 默认排序列 |
| Fuel Type | `fuelType` | 90px | ✅ | 彩色 Tag（蓝/橙/灰） |
| Amount (L) | `fuelAmount` | 100px | ✅ | 右对齐，2 位小数 |
| Unit Price | `unitPrice` | 80px | ✅ | 右对齐；空显示 "—" |
| Total Cost | `totalCost` | 100px | ✅ | 右对齐；空显示 "—" |
| Meter | `meterReading` + `meterUnit` | 100px | — | 空显示 "—" |
| Method | `refuelingMethod` | 100px | ✅ | |
| Supplier | `supplier` | 120px | — | 空显示 "—" |
| Creator | `creatorName` | 100px | ✅ | |
| Actions | — | 100px | — | 编辑 + 删除，根据权限显示 |

---

### 4.5 新增 / 编辑抽屉 `RefuelingFormDrawer.vue`

#### 4.5.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 控制显示 |
| `mode` | String | `'create'` / `'edit'` |
| `record` | Object \| null | 编辑时的原始数据 |

#### 4.5.2 Emits

| 事件 | 说明 |
|------|------|
| `success` | 提交成功，父组件刷新列表 |
| `close` | 关闭抽屉 |

#### 4.5.3 表单字段

| 字段 | 控件 | 必填 | 备注 |
|------|------|------|------|
| Equipment | `el-select`（支持搜索） | ✅ | 选择后展示设备名称、编号、类型 |
| Refueling Date | `el-date-picker` | ✅ | `disabledDate`：禁用明天及以后 |
| Refueling Time | `el-time-picker` | ❌ | — |
| Fuel Type | `el-radio-group` | ✅ | Diesel / Gasoline / Other |
| Fuel Amount (L) | `el-input-number` | ✅ | `min=0.01, max=99999.99, precision=2` |
| Unit Price | `el-input-number` | ❌ | `min=0, precision=2`；变更时自动计算 totalCost |
| Total Cost | `el-input`（只读） | — | `= fuelAmount × unitPrice`，`precision=2` |
| Meter Reading | `el-input-number` | ❌ | + `el-select` 选择单位 KM / HOUR |
| Refueling Method | `el-select` | ✅ | 枚举 |
| Supplier | `el-input` | ❌ | — |
| Receipt Photos | `el-upload` | ❌ | 多图，最多 5 张，10MB/张，jpg/png/heic |
| Remark | `el-input` type="textarea" | ❌ | `maxlength=500`，显示字数 |

#### 4.5.4 自动计算总费用

```javascript
watch: {
  'form.fuelAmount'() { this.calcTotalCost() },
  'form.unitPrice'() { this.calcTotalCost() }
},
methods: {
  calcTotalCost() {
    if (this.form.fuelAmount && this.form.unitPrice) {
      this.form.totalCost = +(this.form.fuelAmount * this.form.unitPrice).toFixed(2)
    } else {
      this.form.totalCost = null
    }
  }
}
```

#### 4.5.5 关闭确认

如果表单有修改（通过对比初始值判断），关闭前弹出 `el-MessageBox.confirm`：
> "Unsaved changes will be lost. Are you sure you want to close?"

---

### 4.6 油耗统计页 `statistics.vue`

#### 4.6.1 组件结构

```
statistics.vue
  ├── FilterBar（Date Range / Equipment / Subcontractor / Fuel Type / Group By）
  ├── SummaryCards（3 张：Total Fuel / Total Cost / Total Records）
  ├── FuelTrendChart（ECharts 柱状图 + 折线图）
  ├── EquipmentRankChart（ECharts 水平条形图）
  └── SubcontractorRankChart（ECharts 水平条形图）
  └── DetailTable（明细数据表格）
```

#### 4.6.2 数据加载

筛选条件变更后调用 `/equipment/refueling/statistics`，一次性获取所有图表数据（`summary`、`details`、`equipmentRanking`、`subcontractorRanking`）。

#### 4.6.3 趋势图 `FuelTrendChart.vue`

- ECharts 混合图：柱状图（加油量 L，左 Y 轴）+ 折线图（费用 $，右 Y 轴）
- X 轴根据 `groupBy` 动态显示（日 / 周 / 月 / 设备名 / 分包商名）
- 鼠标悬浮 tooltip 显示加油量 + 费用

#### 4.6.4 排行图

- ECharts 水平条形图，`value_axis: x`
- 每条柱子右侧显示数值 + 百分比
- Top 10，其余合并为 "Others"

---

### 4.7 数据导出

```javascript
async handleExport() {
  this.exportLoading = true
  try {
    const res = await exportRefueling(this.activeFilter)
    // res 为 Blob
    const url = URL.createObjectURL(res)
    const a = document.createElement('a')
    a.href = url
    a.download = `Refueling_Records_${this.currentProjectName}_${formatDate(new Date())}.xlsx`
    a.click()
    URL.revokeObjectURL(url)
  } finally {
    this.exportLoading = false
  }
}
```

---

## 5. API 封装 `api/refueling.js`

```javascript
// 加油记录分页查询
export const getRefuelingPage = (params) =>
  request.get('/equipment/refueling/page', { params })

// 新增加油记录
export const createRefueling = (data) =>
  request.post('/equipment/refueling/create', data)

// 编辑加油记录
export const updateRefueling = (data) =>
  request.put('/equipment/refueling/update', data)

// 删除加油记录
export const deleteRefueling = (id) =>
  request.delete('/equipment/refueling/delete', { params: { id } })

// 获取加油记录详情
export const getRefuelingDetail = (id) =>
  request.get('/equipment/refueling/get', { params: { id } })

// 油耗统计
export const getRefuelingStatistics = (params) =>
  request.get('/equipment/refueling/statistics', { params })

// 导出加油记录
export const exportRefueling = (params) =>
  request.get('/equipment/refueling/export', { params, responseType: 'blob' })
```

---

## 6. 枚举常量 `utils/refueling-helper.js`

```javascript
export const FUEL_TYPE_OPTIONS = [
  { value: 'DIESEL', label: 'Diesel', labelZh: '柴油', tagType: 'primary' },
  { value: 'GASOLINE', label: 'Gasoline', labelZh: '汽油', tagType: 'warning' },
  { value: 'OTHER', label: 'Other', labelZh: '其他', tagType: 'info' }
]

export const REFUELING_METHOD_OPTIONS = [
  { value: 'TANKER', label: 'Tanker Truck', labelZh: '油罐车' },
  { value: 'STATION', label: 'Gas Station', labelZh: '加油站' },
  { value: 'BARREL', label: 'Barrel/Drum', labelZh: '桶装' },
  { value: 'OTHER', label: 'Other', labelZh: '其他' }
]

export const METER_UNIT_OPTIONS = [
  { value: 'KM', label: 'Kilometers', labelZh: '公里' },
  { value: 'HOUR', label: 'Hours', labelZh: '工时' }
]

export const GROUP_BY_OPTIONS = [
  { value: 'DAY', label: 'Day' },
  { value: 'WEEK', label: 'Week' },
  { value: 'MONTH', label: 'Month' },
  { value: 'EQUIPMENT', label: 'Equipment' },
  { value: 'SUBCONTRACTOR', label: 'Subcontractor' }
]

// 格式化加油量
export const formatFuelAmount = (val) =>
  val != null ? `${val.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })} L` : '—'

// 格式化费用
export const formatCost = (val) =>
  val != null ? `$${val.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}` : '—'
```

---

## 7. 国际化（i18n）

### 7.1 Key 命名

```
refueling.title            → "Refueling Management"
refueling.btn.new          → "New"
refueling.btn.export       → "Export"
refueling.btn.search       → "Search"
refueling.btn.reset        → "Reset"
refueling.col.equipment    → "Equipment"
refueling.col.code         → "Code"
refueling.col.subcontractor → "Subcontractor"
refueling.col.date         → "Date"
refueling.col.fuelType     → "Fuel Type"
refueling.col.amount       → "Amount (L)"
refueling.col.unitPrice    → "Unit Price"
refueling.col.totalCost    → "Total Cost"
refueling.col.method       → "Method"
refueling.col.creator      → "Creator"
refueling.summary.totalFuel   → "Total Fuel (L)"
refueling.summary.totalCost   → "Total Cost"
refueling.summary.recordCount → "Record Count"
refueling.summary.avgCost     → "Avg Cost / Record"
```

---

## 8. 权限控制

| 操作 | 权限 Key | 指令示例 |
|------|---------|---------|
| 查看列表入口 | `equipment:refueling:query` | 路由守卫控制 |
| 新增按钮 | `equipment:refueling:create` | `v-permission="'equipment:refueling:create'"` |
| 编辑按钮 | `equipment:refueling:update` | `v-permission="'equipment:refueling:update'"` |
| 删除按钮 | `equipment:refueling:delete` | `v-permission="'equipment:refueling:delete'"` |
| 导出按钮 | `equipment:refueling:export` | `v-permission="'equipment:refueling:export'"` |
| 统计页入口 | `equipment:refueling:statistics` | 路由守卫控制 |

---

## 9. 验收标准

### 列表页
- [ ] 列表默认按加油日期倒序展示，所有筛选条件正常生效
- [ ] 统计摘要卡片数值随筛选条件动态更新
- [ ] 表格列点击排序（升序 ↑ / 降序 ↓）正常
- [ ] 分页切换（每页 20/50/100 条）正常
- [ ] 批量勾选 → 出现批量删除操作栏，确认后执行删除
- [ ] 操作按钮根据权限显示/隐藏

### 新增 / 编辑
- [ ] 必填字段未填时阻止提交，显示对应错误提示
- [ ] 加油日期禁用未来日期
- [ ] 填写单价后自动计算总费用（精度 2 位小数）
- [ ] 未填单价时 Total Cost 显示空（不显示 0）
- [ ] 图片上传限制 ≤ 5 张，单张 ≤ 10MB
- [ ] 关闭抽屉时如有未保存修改弹出确认
- [ ] 提交成功后列表刷新，显示成功提示

### 统计页
- [ ] 默认筛选最近 3 个月数据
- [ ] 筛选条件变更后所有图表和表格自动刷新
- [ ] 趋势图（柱+折线）正确渲染，Tooltip 显示详细数值
- [ ] 设备/分包商排行正确排序，显示百分比
- [ ] Group By 切换正常响应

### 导出
- [ ] 导出按钮有权限才显示
- [ ] 导出文件名格式正确：`Refueling_Records_{项目名}_{日期}.xlsx`
- [ ] 导出内容包含所有筛选条件内的记录
- [ ] 导出包含所属分包商字段
