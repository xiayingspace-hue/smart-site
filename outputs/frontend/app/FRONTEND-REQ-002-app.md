# 前端开发说明文档 — APP 端设备加油记录

> **来源需求**: [REQ-002-app.md](../../../requirements/app/REQ-002-app.md) + [REQ-002-shared.md](../../../requirements/shared/REQ-002-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: APP 移动端（UNIAPP + Vue 2，iOS & Android）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

本模块实现 APP 端的设备加油记录功能，涵盖：

| 功能模块 | 说明 |
|---------|------|
| 加油记录列表页 | 卡片列表，支持搜索 + 底部筛选面板，滚动加载更多 |
| 新增加油记录页 | 完整表单，设备选择器 / 日期时间选择器 / 单价自动计算总费用 / 照片上传 |
| 编辑加油记录页 | 仅允许本人创建且 24h 内，复用新增表单 |
| 加油记录详情页 | 分组卡片展示所有字段，照片全屏预览 |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | UNIAPP + Vue 2 |
| HTTP | `utils/request.js`（uni.request 封装），自动附带认证 Header |
| 图片选择 | `uni.chooseImage`，支持拍照 + 相册 |
| 图片上传 | `uni.uploadFile`，上传到 OSS |
| 日期选择 | `<picker mode="date">` |
| 时间选择 | `<picker mode="time">` |
| 数字键盘 | `<input type="digit">` |
| 滚动加载 | `<scroll-view>` + `onReachBottom` 生命周期 |

---

## 3. 文件与目录结构

```
pages/
└── equipment/
    └── refueling/
        ├── list.vue                 # 加油记录列表页
        ├── form.vue                 # 新增/编辑页（通过 mode 参数区分）
        └── detail.vue               # 详情页
components/
├── EquipmentSelector.vue            # 设备选择弹窗
├── FuelTypeTag.vue                  # 燃油类型彩色标签
└── RefuelingFilterPanel.vue        # 筛选面板（底部弹出）
api/
└── refueling.js                     # 加油记录 API 封装
utils/
└── refueling-helper.js             # 枚举常量、格式化
```

---

## 4. 加油记录列表页 `list.vue`

### 4.1 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `list` | Array | 加油记录列表（累积追加，不替换） |
| `pageNo` | Number | 当前页，默认 1 |
| `pageSize` | Number | 每页 20 |
| `total` | Number | 总记录数 |
| `loading` | Boolean | 列表加载 |
| `refreshing` | Boolean | 下拉刷新状态 |
| `noMore` | Boolean | 是否已加载全部（`list.length >= total`） |
| `keyword` | String | 搜索关键词 |
| `filterVisible` | Boolean | 筛选面板显示 |
| `filter` | Object | 当前激活筛选条件 `{ fuelType, startDate, endDate }` |
| `filterCount` | Number | 激活筛选条件数量（用于角标） |

### 4.2 核心方法

| 方法 | 说明 |
|------|------|
| `fetchList(reset = false)` | 若 reset=true 则重置 pageNo=1 清空 list；调用 `/equipment/refueling/page` |
| `handleSearch()` | 300ms 防抖后调用 `fetchList(true)` |
| `handleReachBottom()` | 若 `!noMore && !loading`，pageNo++，调用 `fetchList()` |
| `handleRefresh()` | 下拉刷新，调用 `fetchList(true)`，完成后 `refreshing = false` |
| `handleFilterApply(filterObj)` | 更新 filter，更新 filterCount，`filterVisible = false`，`fetchList(true)` |
| `handleFilterReset()` | 清空 filter，filterCount=0，`fetchList(true)` |
| `goToDetail(item)` | `uni.navigateTo({ url: '/pages/equipment/refueling/detail?id=' + item.id })` |

### 4.3 卡片渲染

每条记录渲染为 **上下两区域** 卡片：

**上部区域（设备信息）**：
```vue
<view class="card-top">
  <image class="equipment-img" :src="item.equipmentImage || defaultEquipmentIcon" />
  <view class="equipment-info">
    <text class="name">{{ item.equipmentName }}</text>
    <text class="code">{{ item.equipmentCode }}</text>
    <text class="type">{{ item.equipmentType }}</text>
    <text class="subcon">{{ item.subcontractorName }}</text>
  </view>
</view>
```

**下部区域（加油信息）**：
```vue
<view class="card-bottom">
  <view class="row">
    <text>📅 {{ item.refuelingDate }}</text>
    <FuelTypeTag :type="item.fuelType" />
  </view>
  <view class="row">
    <text>🛢️ {{ item.fuelAmount }} L</text>
    <text v-if="item.totalCost">💰 ${{ item.totalCost }}</text>
  </view>
</view>
```

### 4.4 搜索防抖

```javascript
watch: {
  keyword() {
    clearTimeout(this.searchTimer)
    this.searchTimer = setTimeout(() => this.fetchList(true), 300)
  }
}
```

### 4.5 筛选角标

```vue
<!-- 筛选图标 -->
<view class="filter-btn" @tap="filterVisible = true">
  <uni-icons type="tune" size="20" />
  <text v-if="filterCount > 0" class="filter-badge">{{ filterCount }}</text>
</view>
```

---

## 5. 新增/编辑页 `form.vue`

### 5.1 路由参数

| 参数 | 说明 |
|------|------|
| `mode` | `'create'`（新增）/ `'edit'`（编辑） |
| `id` | 编辑时的记录 ID |

### 5.2 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `form` | Object | 表单数据 |
| `errors` | Object | 字段错误信息 |
| `submitting` | Boolean | 提交 loading |
| `photoList` | Array | 已上传照片列表（`{ url, name, status }`） |
| `equipmentOptions` | Array | 设备选项列表 |
| `equipmentSelectorVisible` | Boolean | 设备选择弹窗 |

### 5.3 表单字段与组件

| 字段 | 组件 | 必填 | 备注 |
|------|------|------|------|
| Equipment | 自定义选择器 `EquipmentSelector.vue` | ✅ | 选中后自动填充分包商信息 |
| Refueling Date | `<picker mode="date">` | ✅ | `max="{{ today }}"`，默认今天 |
| Refueling Time | `<picker mode="time">` | ❌ | — |
| Fuel Type | `<radio-group>` | ✅ | Diesel / Gasoline / Other |
| Fuel Amount (L) | `<input type="digit">` | ✅ | 2 位小数 |
| Unit Price | `<input type="digit">` | ❌ | 变更时自动计算 totalCost |
| Total Cost | `<input disabled>` | — | 只读，灰底 |
| Meter Reading | `<input type="digit">` + `<picker>` | ❌ | 单位 KM / HOUR |
| Refueling Method | `<radio-group>` | ✅ | 4 个选项 |
| Supplier | `<input>` | ❌ | — |
| Receipt Photos | 自定义照片上传区 | ❌ | 最多 5 张，10MB 单张 |
| Remark | `<textarea>` | ❌ | maxlength=500，右下角字数 |

### 5.4 总费用自动计算

```javascript
watch: {
  'form.fuelAmount'() { this.calcTotalCost() },
  'form.unitPrice'() { this.calcTotalCost() }
},
methods: {
  calcTotalCost() {
    const { fuelAmount, unitPrice } = this.form
    if (fuelAmount && unitPrice) {
      this.form.totalCost = +(fuelAmount * unitPrice).toFixed(2)
    } else {
      this.form.totalCost = null
    }
  }
}
```

### 5.5 照片上传

```javascript
async handleChoosePhoto() {
  if (this.photoList.length >= 5) {
    uni.showToast({ title: 'Maximum 5 photos allowed', icon: 'none' })
    return
  }
  const { tempFilePaths } = await uni.chooseImage({
    count: 5 - this.photoList.length,
    sizeType: ['compressed']
  })
  for (const path of tempFilePaths) {
    await this.uploadPhoto(path)
  }
},
async uploadPhoto(path) {
  // 检查文件大小 < 10MB（UNIAPP 文件信息通过 uni.getFileInfo 获取）
  const photo = { url: path, status: 'uploading', ossUrl: null }
  this.photoList.push(photo)
  try {
    const { data } = await uni.uploadFile({
      url: API_BASE + '/infra/file/upload',
      filePath: path,
      name: 'file'
    })
    photo.ossUrl = JSON.parse(data).data
    photo.status = 'done'
  } catch {
    photo.status = 'error'
  }
}
```

### 5.6 表单校验

```javascript
validateForm() {
  this.errors = {}
  if (!this.form.equipmentId) this.errors.equipment = 'Please select equipment'
  if (!this.form.refuelingDate) this.errors.refuelingDate = 'Please select refueling date'
  if (!this.form.fuelType) this.errors.fuelType = 'Please select fuel type'
  if (!this.form.fuelAmount) this.errors.fuelAmount = 'Please enter fuel amount'
  else if (this.form.fuelAmount <= 0) this.errors.fuelAmount = 'Fuel amount must be greater than 0'
  else if (this.form.fuelAmount > 99999.99) this.errors.fuelAmount = 'Fuel amount exceeds maximum limit'
  if (!this.form.refuelingMethod) this.errors.refuelingMethod = 'Please select refueling method'
  return Object.keys(this.errors).length === 0
}
```

### 5.7 提交流程

```
用户点击 Submit
  → validateForm()
  → 校验失败 → 滚动到第一个错误字段，停止
  → 校验通过 → submitting = true
  → 新增: POST /equipment/refueling/create
  → 编辑: PUT /equipment/refueling/update
  → 成功:
      Toast: "Record created/updated successfully"
      uni.navigateBack()
  → 失败: 显示错误 Toast，submitting = false
```

### 5.8 编辑权限检查（进入页面时）

```javascript
created() {
  if (this.mode === 'edit') {
    this.fetchRecordDetail().then(record => {
      const isOwner = record.creatorId === this.currentUserId
      const withinTime = Date.now() - new Date(record.createTime).getTime() < 24 * 3600 * 1000
      if (!isOwner || !withinTime) {
        uni.showToast({ title: 'Editing period has expired', icon: 'none' })
        uni.navigateBack()
      }
    })
  }
}
```

---

## 6. 详情页 `detail.vue`

### 6.1 分组卡片展示

| 分组 | 字段 |
|------|------|
| 设备信息 | equipmentName, equipmentCode, equipmentType, subcontractorName |
| 加油信息 | refuelingDate, refuelingTime, fuelType, fuelAmount, unitPrice, totalCost, meterReading+meterUnit, refuelingMethod, supplier |
| 凭证照片 | receiptPhotos（缩略图，点击全屏预览） |
| 备注 | remark |
| 记录信息 | creatorName, createTime, updateTime |

### 6.2 编辑入口

导航栏右侧显示 ✏️ 编辑图标，当 `record.editable === true` 时显示。

### 6.3 照片预览

```javascript
previewPhoto(index) {
  uni.previewImage({
    current: index,
    urls: this.record.receiptPhotos
  })
}
```

---

## 7. API 封装 `api/refueling.js`

```javascript
// 加油记录分页查询
export const getRefuelingPage = (params) => request({ url: '/equipment/refueling/page', method: 'GET', data: params })

// 新增加油记录
export const createRefueling = (data) => request({ url: '/equipment/refueling/create', method: 'POST', data })

// 编辑加油记录
export const updateRefueling = (data) => request({ url: '/equipment/refueling/update', method: 'PUT', data })

// 获取加油记录详情
export const getRefuelingDetail = (id) => request({ url: '/equipment/refueling/get', method: 'GET', data: { id } })
```

---

## 8. 验收标准

### 列表页
- [ ] 默认按加油日期倒序展示加油记录卡片
- [ ] 卡片正确显示设备名称、编号、类型、分包商名称、日期、燃油类型标签、加油量、费用
- [ ] 无费用时不显示费用行
- [ ] 搜索框输入防抖 300ms 后触发搜索
- [ ] 筛选面板弹出，条件应用后列表刷新，漏斗图标显示角标
- [ ] 上拉加载更多，全部加载后不再触发
- [ ] 下拉刷新正常工作
- [ ] 空列表显示空状态插画 + "No refueling records yet"
- [ ] 点击卡片进入详情页

### 新增/编辑
- [ ] 必填字段为空时阻止提交，错误提示显示在字段下方
- [ ] 加油日期禁止选择未来日期，默认今天
- [ ] 填写单价后自动计算总费用，Total Cost 只读
- [ ] 未填单价时 Total Cost 显示 "—"
- [ ] 设备选择后自动显示设备编号和分包商
- [ ] 照片上传限制 ≤ 5 张，超出提示
- [ ] 照片上传显示进度状态
- [ ] 提交成功后返回列表页，Toast 提示
- [ ] 编辑时，非本人或超 24h 的记录立即返回并提示

### 详情页
- [ ] 所有字段分组正确展示，空字段不显示
- [ ] 照片缩略图点击后全屏预览，支持左右滑动切换
- [ ] `editable = true` 时导航栏显示编辑按钮
