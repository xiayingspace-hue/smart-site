# 前端开发说明文档 — PC 管理端 Site Engineer 图纸查阅与局部更新查看

> **来源需求**: [REQ-005-pc.md](../../../requirements/pc/REQ-005-pc.md) + [REQ-005-shared.md](../../../requirements/shared/REQ-005-shared.md)
> **UI 设计参考**: [UI-REQ-005-pc.md](../../ui/pc/UI-REQ-005-pc.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

本模块实现 Site Engineer 在 PC 端的图纸查阅能力，涵盖：

| 功能模块 | 说明 |
|---------|------|
| Site Engineer 图纸列表 | 仅展示分配给当前 SE 且 ACTIVE 的图纸，可搜索/筛选 Category |
| 图纸在线查看页 | PDF/PNG/JPG 在线预览，支持缩放、平移、全屏 |
| Confirm Reading | SE 在 PC 端完成图纸查阅确认，跨端同步 |
| Markups Tab | 查看图纸 ACTIVE 局部更新列表，支持 Mark as Read |
| PC 站内通知 | 铃铛角标 + 通知下拉面板，点击跳转至图纸 Markups Tab |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 + Vue Router + Vuex |
| UI 组件库 | Element UI 2.x |
| HTTP | Axios，统一封装 `request.js` |
| 文件预览 | PDF 使用 `pdf.js`（或 `<iframe>` 内嵌），PNG/JPG 直接 `<img>` |
| 路由 | `/drawings`（SE 图纸列表），`/drawings/:drawingId`（图纸查看页） |
| 权限 | 路由守卫控制 Site Engineer 角色才可访问 `/drawings` |
| 国际化 | Vue I18n |

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── se-drawings/
│       ├── index.vue                         # Site Engineer 图纸列表页
│       ├── detail.vue                        # 图纸在线查看页
│       └── components/
│           ├── SeDrawingTable.vue            # SE 图纸表格
│           ├── DrawingPreview.vue            # 图纸在线预览组件
│           └── MarkupTabPanel.vue            # Markups Tab 面板
├── layout/
│   └── components/
│       └── NotificationBell.vue              # 铃铛（扩展，支持通知列表）
│       └── NotificationPanel.vue            # 通知下拉面板（新增）
├── api/
│   ├── se-drawing.js                         # SE 图纸相关 API
│   └── notification.js                       # 站内通知 API
└── utils/
    └── drawing-helper.js                     # 扩展：SE 确认状态映射
```

---

## 4. Site Engineer 图纸列表页 `index.vue`

### 4.1 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `searchKeyword` | String | Code / Name 搜索关键词，防抖 300ms |
| `categoryFilter` | String | Category 下拉筛选 |
| `tableData` | Array | 图纸列表数据 |
| `total` | Number | 总记录数 |
| `pageNo` | Number | 当前页，默认 1 |
| `pageSize` | Number | 每页 20 |
| `loading` | Boolean | 表格 loading |

### 4.2 数据加载

调用 API：`GET /drawing/se/page`（SE 专用接口，后端自动过滤：status=ACTIVE + 已分配给当前用户）。

```javascript
methods: {
  async fetchList() {
    this.loading = true
    try {
      const res = await getSeDrawingList({
        keyword: this.searchKeyword,
        category: this.categoryFilter,
        pageNo: this.pageNo,
        pageSize: this.pageSize
      })
      this.tableData = res.list
      this.total = res.total
    } finally {
      this.loading = false
    }
  }
}
```

### 4.3 空状态

```vue
<el-empty
  v-if="!loading && tableData.length === 0"
  description="No drawings assigned to you yet."
  :image="drawingEmptyImage"
/>
```

---

## 5. SE 图纸表格 `SeDrawingTable.vue`

### 5.1 列定义

| 列名 | 字段 | 宽度 | 说明 |
|------|------|------|------|
| Drawing Code / Name | `drawingCode` + `drawingName` | 200px | Code 加粗，Name 作为次级文字 |
| Category | `category` | 100px | 彩色 Tag |
| Version | `currentVersionNo` | 80px | 如 "V3" |
| Status | `confirmStatus` | 120px | 见下方说明 |
| Markups | `unreadMarkupCount` | 90px | `●{n}` 橙色（n>0）；`—`（n=0） |
| Confirmed | `confirmedAt` | 100px | 格式 "Apr 8"，未确认显示 `—` |
| Last Updated | `updatedAt` | 120px | 格式 "YYYY-MM-DD" |

**Status 列渲染**：
```javascript
// 已确认
if (row.confirmed) {
  return '<span class="status-read">✓ Read</span>'
}
// 未确认
return '<a class="status-confirm" @click="goToDetail(row)">Confirm →</a>'
```

### 5.2 行点击

整行点击 → `$router.push({ name: 'SeDrawingDetail', params: { drawingId: row.id } })`

### 5.3 Markups 列点击

```javascript
handleMarkupsClick(row) {
  // 进入图纸查看页并自动激活 Markups Tab
  this.$router.push({
    name: 'SeDrawingDetail',
    params: { drawingId: row.id },
    query: { tab: 'markups' }
  })
}
```

---

## 6. 图纸在线查看页 `detail.vue`

### 6.1 路由参数

| 参数 | 来源 | 说明 |
|------|------|------|
| `drawingId` | `$route.params.drawingId` | 图纸 ID |
| `tab` | `$route.query.tab` | 初始激活 Tab：`'preview'`（默认）/ `'markups'` |

### 6.2 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `drawing` | Object | 图纸详情 |
| `activeTab` | String | `'preview'` / `'markups'`，初始值来自 query |
| `confirmLoading` | Boolean | Confirm Reading 按钮 loading |

### 6.3 数据加载

```javascript
created() {
  this.activeTab = this.$route.query.tab || 'preview'
  this.fetchDrawingDetail()
},
methods: {
  fetchDrawingDetail() {
    getSeDrawingDetail(this.$route.params.drawingId)
      .then(res => { this.drawing = res })
  }
}
```

### 6.4 页面结构

```vue
<template>
  <div class="drawing-detail">
    <!-- 面包屑 -->
    <el-breadcrumb>
      <el-breadcrumb-item :to="{ name: 'SeDrawings' }">Drawings</el-breadcrumb-item>
      <el-breadcrumb-item>{{ drawing.drawingCode }} {{ drawing.drawingName }}</el-breadcrumb-item>
    </el-breadcrumb>

    <!-- 图纸元信息 -->
    <DrawingMeta :drawing="drawing" />

    <!-- Tab -->
    <el-tabs v-model="activeTab">
      <el-tab-pane label="Drawing Preview" name="preview">
        <DrawingPreview :fileUrl="drawing.currentVersionFileUrl" />
        <ConfirmReadingBar :drawing="drawing" @confirmed="onConfirmed" />
      </el-tab-pane>
      <el-tab-pane :label="`Markups (${drawing.activeMarkupCount})`" name="markups">
        <MarkupTabPanel :drawing-id="drawing.id" />
      </el-tab-pane>
    </el-tabs>
  </div>
</template>
```

---

## 7. 图纸预览组件 `DrawingPreview.vue`

### 7.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `fileUrl` | String | 图纸文件 OSS URL |

### 7.2 预览策略

| 文件类型 | 预览方式 |
|---------|---------|
| `.pdf` | `<iframe :src="fileUrl" />` 或 pdf.js Viewer |
| `.png` / `.jpg` / `.jpeg` | `<img :src="fileUrl" />` |
| `.dwg`（转换后 PDF） | 同 PDF |

### 7.3 缩放 & 平移

- PDF：依赖 pdf.js 内置 zoom 控件
- 图片：使用 `wheel` 事件监听 + CSS `transform: scale()` + 鼠标拖拽平移
- 提供 [全屏查看] 按钮，调用 `element.requestFullscreen()`

```javascript
handleWheel(e) {
  e.preventDefault()
  const delta = e.deltaY > 0 ? 0.9 : 1.1
  this.scale = Math.min(Math.max(this.scale * delta, 0.1), 10)
}
```

---

## 8. Confirm Reading 操作

### 8.1 显示逻辑

```vue
<!-- 未确认 -->
<el-button
  v-if="!drawing.confirmed"
  type="primary"
  :loading="confirmLoading"
  @click="handleConfirm"
>
  Confirm Reading
</el-button>

<!-- 已确认 -->
<div v-else class="confirmed-text">
  ✓ Confirmed on {{ formatDate(drawing.confirmedAt) }}
</div>
```

### 8.2 确认流程

```javascript
async handleConfirm() {
  await this.$confirm(
    'Confirm that you have read and understood this drawing?',
    'Confirm Reading',
    { type: 'info', confirmButtonText: 'Confirm' }
  )
  this.confirmLoading = true
  try {
    await confirmDrawingRead({
      drawingVersionId: this.drawing.currentVersionId,
      deviceInfo: 'PC'
    })
    // 更新本地状态
    this.drawing.confirmed = true
    this.drawing.confirmedAt = new Date().toISOString()
    // 更新列表页中的 Status 列
    this.$message.success('Reading confirmed successfully')
  } finally {
    this.confirmLoading = false
  }
}
```

> `deviceInfo` 固定传 `'PC'`，后端用于记录确认来源。跨端幂等：同一用户对同一图纸版本唯一，已在 APP 确认则 PC 显示"已确认"。

---

## 9. Markups Tab 面板 `MarkupTabPanel.vue`

### 9.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `drawingId` | Number | 图纸 ID |

### 9.2 数据加载

```javascript
created() { this.fetchMarkups() },
methods: {
  fetchMarkups() {
    // 仅加载 ACTIVE 状态
    getMarkupList({ drawingId: this.drawingId, status: 'ACTIVE' })
      .then(res => { this.markups = res.list })
  }
}
```

### 9.3 Markup 卡片（SE 视角）

与 REQ-004 的 `MarkupCard` 不同，SE 视图**无勾选/删除**操作，仅显示 `[Mark as Read]` / `✓ Read`：

| 元素 | 显示 |
|------|------|
| `[New]` 标签 | 橙色，当前用户未确认时显示 |
| 内容区 | 标题、发布人、日期、影响区域、说明文字（最多 3 行）、附件 |
| `[Mark as Read]` | 当前用户未确认时显示 |
| `✓ Read` | 已确认时显示（绿色，不可点击） |

### 9.4 Mark as Read 操作

```javascript
async handleMarkAsRead(markup) {
  markup.confirmLoading = true
  try {
    await confirmMarkupRead({ markupId: markup.id, deviceInfo: 'PC' })
    markup.confirmed = true
    // 更新图纸列表中的 unreadMarkupCount
    this.$emit('markup-read')
  } finally {
    markup.confirmLoading = false
  }
}
```

### 9.5 附件查看

点击附件文件名 → `window.open(url, '_blank')` 在新 Tab 打开。

---

## 10. PC 站内通知

### 10.1 NotificationPanel.vue 结构

```vue
<template>
  <el-popover trigger="click" placement="bottom-end" width="360">
    <template #reference>
      <el-badge :value="unreadCount" :max="99" :hidden="unreadCount === 0">
        <i class="el-icon-bell notification-bell" />
      </el-badge>
    </template>

    <div class="notification-panel">
      <div class="panel-header">
        <span>Notifications</span>
        <a @click="markAllRead">Mark all as read</a>
      </div>
      <div class="notification-list">
        <NotificationItem
          v-for="item in notifications"
          :key="item.id"
          :notification="item"
          @click="handleNotificationClick(item)"
        />
      </div>
    </div>
  </el-popover>
</template>
```

### 10.2 未读数量轮询

```javascript
// 在 Vuex user module 中
actions: {
  startNotificationPolling({ commit, dispatch }) {
    dispatch('fetchUnreadCount')
    setInterval(() => dispatch('fetchUnreadCount'), 30000) // 30s 轮询
  },
  async fetchUnreadCount({ commit }) {
    const res = await getNotificationList({ pageSize: 1 })
    commit('SET_TODO_COUNT', res.unreadCount)
  }
}
```

### 10.3 通知点击跳转

```javascript
handleNotificationClick(notification) {
  // 标记已读
  markNotificationRead(notification.id).then(() => {
    this.$store.dispatch('user/fetchUnreadCount')
  })
  // 跳转
  if (notification.targetRoute) {
    this.$router.push(notification.targetRoute)
    // targetRoute 示例: "/drawings/101?tab=markups"
  }
  // 关闭 Popover
  this.popoverVisible = false
}
```

### 10.4 Mark All as Read

```javascript
markAllRead() {
  markAllNotificationsRead().then(() => {
    this.notifications.forEach(n => { n.isRead = true })
    this.$store.commit('user/SET_TODO_COUNT', 0)
  })
}
```

---

## 11. API 封装

### `api/se-drawing.js`

```javascript
// SE 专用图纸列表（自动过滤：ACTIVE + 已分配给当前用户）
export const getSeDrawingList = (params) =>
  request.get('/drawing/se/page', { params })

// SE 图纸详情
export const getSeDrawingDetail = (drawingId) =>
  request.get('/drawing/se/get', { params: { drawingId } })

// 确认图纸已读
export const confirmDrawingRead = (data) =>
  request.post('/drawing/confirm', data)
  // data: { drawingVersionId, deviceInfo: 'PC' }

// SE 获取图纸的 Markup 列表（仅 ACTIVE）
export const getSeMarkupList = (params) =>
  request.get('/drawing/markup/list', { params })
  // params: { drawingId, status: 'ACTIVE' }

// 确认 Markup 已读（Mark as Read）
export const confirmMarkupRead = (data) =>
  request.post('/drawing/markup/confirm', data)
  // data: { markupId, deviceInfo: 'PC' }
```

### `api/notification.js`

```javascript
// 获取通知列表
export const getNotificationList = (params) =>
  request.get('/notification/list', { params })

// 标记单条已读
export const markNotificationRead = (id) =>
  request.patch(`/notification/${id}/read`)

// 全部标记已读
export const markAllNotificationsRead = () =>
  request.patch('/notification/read-all')
```

---

## 12. 路由配置

```javascript
// 在管理后台框架路由 children 中添加
{
  path: 'drawings',
  name: 'SeDrawings',
  component: () => import('@/views/se-drawings/index.vue'),
  meta: { title: 'Drawings', roles: ['SITE_ENGINEER'] }
},
{
  path: 'drawings/:drawingId',
  name: 'SeDrawingDetail',
  component: () => import('@/views/se-drawings/detail.vue'),
  meta: { title: 'Drawing Detail', roles: ['SITE_ENGINEER'] }
}
```

> 路由守卫检查 `roles`，非 Site Engineer 角色访问 `/drawings` 时重定向到 `/403`。

---

## 13. 国际化（i18n）

```
seDrawing.title              → "Drawings"
seDrawing.status.read        → "✓ Read"
seDrawing.status.confirm     → "Confirm →"
seDrawing.col.category       → "Category"
seDrawing.col.version        → "Version"
seDrawing.col.status         → "Status"
seDrawing.col.markups        → "Markups"
seDrawing.col.confirmed      → "Confirmed"
seDrawing.col.lastUpdated    → "Last Updated"
seDrawing.empty              → "No drawings assigned to you yet."
seDrawing.btn.confirmReading → "Confirm Reading"
seDrawing.confirmedText      → "✓ Confirmed on {date}"
seDrawing.tab.preview        → "Drawing Preview"
seDrawing.tab.markups        → "Markups ({count})"
markup.btn.markAsRead        → "Mark as Read"
markup.status.read           → "✓ Read"
markup.badge.new             → "New"
notification.markAllRead     → "Mark all as read"
```

---

## 14. 验收标准

### Site Engineer 图纸列表
- [ ] 只显示 `status = ACTIVE` 且已分配给当前 SE 的图纸
- [ ] 搜索（Code / Name）和 Category 筛选正常生效
- [ ] Status 列已确认显示 `✓ Read`，未确认显示 `Confirm →`（蓝色可点击）
- [ ] Markups 列：有 ACTIVE 未读时显示 `●{n}` 橙色；无未读时显示 `—`
- [ ] 无分配图纸时显示空状态文案

### 图纸在线查看
- [ ] 点击行 / `Confirm →` 链接进入图纸查看页
- [ ] 图纸文件正常加载（PDF 内嵌 / 图片显示）
- [ ] 支持鼠标滚轮缩放和拖拽平移
- [ ] [全屏查看] 按钮正常工作

### Confirm Reading
- [ ] 未确认时显示 [Confirm Reading] 按钮
- [ ] 点击弹出确认对话框，确认后调用 API（携带 `deviceInfo: 'PC'`）
- [ ] 确认成功后按钮替换为 `✓ Confirmed on {date}`
- [ ] 在 APP 端确认后，PC 端刷新显示"已确认"（跨端同步）
- [ ] 返回列表页，Status 列更新为 `✓ Read`

### Markups Tab
- [ ] Markups Tab 仅显示 ACTIVE 状态的局部更新
- [ ] 按发布时间降序排列
- [ ] 未读条目显示 `[New]` 标签和 `[Mark as Read]` 按钮
- [ ] 点击 `[Mark as Read]` 成功后替换为 `✓ Read`
- [ ] 已在 APP 确认的条目，PC 端刷新后显示 `✓ Read`（跨端同步）
- [ ] 点击 Markups 列 `●{n}` 角标 → 进入图纸查看页并激活 Markups Tab

### PC 站内通知
- [ ] 有未读通知时铃铛显示红色角标（最多 "99+"）
- [ ] 点击铃铛打开通知面板，显示最近通知列表
- [ ] 点击通知条目跳转到对应图纸页并激活 Markups Tab，通知标记已读
- [ ] 角标数量在标记已读后正确减少
- [ ] "Mark all as read" 功能将所有通知标为已读，角标清零
- [ ] 未被分配该图纸的用户不收到该图纸的通知
