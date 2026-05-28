---
doc_type: frontend_spec
runtime: Vue 2 + Element UI 2.x + Vue Router + Vuex
source_req: REQ-005-pc (v0.3.5)
ui_doc: UI-REQ-005-pc.md (v0.3.5)
generated_at: 2026-05-28
---

# 前端说明文档：PC 端 Site Engineer 图纸查阅与局部更新查看

> 本文档依据 [REQ-005-pc.md v0.3.5](../../../requirements/pc/REQ-005-pc.md) 与 [UI-REQ-005-pc.md v0.3.5](../../ui/pc/UI-REQ-005-pc.md) 生成，描述 Vue 2 / Element UI 实现细节。

---

## 1. 功能概述

| 功能 | 说明 |
|------|------|
| SE 图纸列表 | 仅展示 ACTIVE + 已分配给当前用户的 DrawingVersion 列表 |
| RFA 信息列 | 列表新增 RFA No. 与 Subject of RFA 列 |
| Markups 列 | 格式 `●{unread} / {total}`，点击进入 Markups Tab |
| 图纸在线查看 | PDF / PNG / JPG 在线预览，支持缩放与平移 |
| Confirm Reading | **位于 Header 右侧**，弹窗确认，写入 DrawingConfirmation（deviceInfo="PC"） |
| Markups Tab | 列表卡片显示 `{日期}·{创建人}` 元信息行 + `Based on {submissionNo}·Page {n}` 报审行；无 Affected Area |
| 站内通知 | 铃铛角标 + 下拉面板，点击跳转并激活 Markups Tab |

---

## 2. 技术栈与约定

| 项目 | 说明 |
|------|------|
| UI 框架 | Element UI 2.x（`el-table`、`el-tabs`、`el-button`、`el-dialog`、`el-popover` 等） |
| 路由 | Vue Router，路由守卫校验 `roles: ['SITE_ENGINEER']` |
| 状态 | Vuex（通知未读数）；页面级数据用组件内 data |
| 请求 | Axios 封装，路径前缀 `/api` |
| 国际化 | Vue I18n，中英双语 |
| 日期格式 | dayjs；列表日期 `YYYY-MM-DD`，卡片 / 确认文案 `MMM D, YYYY` |

---

## 3. 文件目录结构

```
src/
  views/
    se-drawings/
      index.vue              # SE 图纸列表页（/drawings）
      detail.vue             # 图纸在线查看页（/drawings/:drawingVersionId）
      components/
        SeDrawingTable.vue   # 图纸列表表格
        DrawingMeta.vue      # Header 信息条（含 Confirm Reading 按钮）
        DrawingPreview.vue   # 图纸在线预览区
        MarkupTabPanel.vue   # Markups Tab 面板
        MarkupCard.vue       # 单条 Markup 卡片
  layout/
    components/
      NotificationBell.vue   # 铃铛入口
      NotificationPanel.vue  # 通知下拉面板
  api/
    se-drawing.js            # SE 图纸相关接口
    notification.js          # 站内通知接口
```

---

## 4. 图纸列表页 `index.vue`

### 4.1 数据状态

```javascript
data() {
  return {
    tableData: [],       // DrawingVersion 列表
    total: 0,
    loading: false,
    searchKeyword: '',   // Description 模糊搜索
    categoryFilter: '',  // Category 下拉筛选
    pageNo: 1,
    pageSize: 20
  }
}
```

### 4.2 数据加载

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
| Description | `description` | min-width 200px | 批次描述，14px 加粗 |
| Category | `category` | 120px | 分类文字 |
| RFA No. | `rfaNo` | 100px | 外部报审编号；`—` 若为 null |
| Subject of RFA | `subjectOfRfa` | min-width 160px | 报审主题；`—` 若为 null |
| Status | `confirmed`（Boolean） | 110px | 见下方说明 |
| Markups | `markupsUnread` + `markupsTotal` | 100px | 见 §5.2 |
| Confirmed | `confirmedAt` | 100px | 格式 "Apr 8"，未确认显示 `—` |
| Last Updated | `updatedAt` | 120px | 格式 "YYYY-MM-DD" |

> **不包含的列**：`versionNo`（系统版本号）、`drawingCode`、`drawingName` 均不在 SE 列表中展示。

**Status 列渲染**：

```javascript
renderStatus(h, { row }) {
  if (row.confirmed) {
    return h('span', { class: 'status-read' }, '✓ Read')
  }
  return h('a', {
    class: 'status-confirm',
    on: { click: () => this.goToPreview(row) }
  }, 'Confirm →')
}
```

### 5.2 Markups 列渲染

```javascript
renderMarkups(h, { row }) {
  const { markupsTotal, markupsUnread } = row
  // 无 Markup
  if (!markupsTotal || markupsTotal === 0) {
    return h('span', { style: { color: '#909399' } }, '—')
  }
  // 有未读
  if (markupsUnread > 0) {
    return h('a', {
      class: 'markups-link',
      on: { click: () => this.goToMarkups(row) }
    }, [
      h('span', { style: { color: '#F57F17' } }, `●${markupsUnread}`),
      h('span', { style: { color: '#909399' } }, ` / ${markupsTotal}`)
    ])
  }
  // 全部已读
  return h('span', { style: { color: '#909399' } }, `0 / ${markupsTotal}`)
}
```

### 5.3 行点击

```javascript
handleRowClick(row, column) {
  // Status 列和 Markups 列有自己的点击处理，不触发行跳转
  const ignoredColumns = ['status', 'markups']
  if (ignoredColumns.includes(column.property)) return
  this.goToPreview(row)
},
goToPreview(row) {
  this.$router.push({ name: 'SeDrawingDetail', params: { drawingVersionId: row.id } })
},
goToMarkups(row) {
  this.$router.push({
    name: 'SeDrawingDetail',
    params: { drawingVersionId: row.id },
    query: { tab: 'markups' }
  })
}
```

---

## 6. 图纸在线查看页 `detail.vue`

### 6.1 路由参数

| 参数 | 来源 | 说明 |
|------|------|------|
| `drawingVersionId` | `$route.params.drawingVersionId` | DrawingVersion ID |
| `tab` | `$route.query.tab` | 初始激活 Tab：`'preview'`（默认）/ `'markups'` |

### 6.2 数据状态

```javascript
data() {
  return {
    drawingVersion: null,  // DrawingVersion 详情
    activeTab: 'preview',
    confirmLoading: false
  }
}
```

### 6.3 数据加载

```javascript
created() {
  this.activeTab = this.$route.query.tab || 'preview'
  this.fetchDetail()
},
methods: {
  fetchDetail() {
    getSeDrawingDetail(this.$route.params.drawingVersionId)
      .then(res => { this.drawingVersion = res })
  }
}
```

### 6.4 页面结构

```vue
<template>
  <div class="drawing-detail">
    <!-- 面包屑 -->
    <el-breadcrumb separator="/">
      <el-breadcrumb-item :to="{ name: 'SeDrawings' }">Drawings</el-breadcrumb-item>
      <el-breadcrumb-item>{{ drawingVersion && drawingVersion.description }}</el-breadcrumb-item>
    </el-breadcrumb>

    <!-- Header 信息条（含 Confirm Reading 按钮） -->
    <DrawingMeta
      v-if="drawingVersion"
      :drawing-version="drawingVersion"
      :confirm-loading="confirmLoading"
      @confirm="handleConfirm"
    />

    <!-- Tab -->
    <el-tabs v-model="activeTab" type="card">
      <el-tab-pane label="Drawing Preview" name="preview">
        <DrawingPreview
          v-if="drawingVersion"
          :file-url="drawingVersion.fileUrl"
        />
      </el-tab-pane>
      <el-tab-pane name="markups">
        <!-- Tab 标签含未读气泡 -->
        <span slot="label">
          Markups
          <el-badge
            v-if="drawingVersion && drawingVersion.markupsUnread > 0"
            :value="drawingVersion.markupsUnread"
            :max="99"
            type="warning"
          />
        </span>
        <MarkupTabPanel
          v-if="drawingVersion"
          :drawing-version-id="drawingVersion.id"
          @markup-read="onMarkupRead"
        />
      </el-tab-pane>
    </el-tabs>
  </div>
</template>
```

---

## 7. Header 信息条组件 `DrawingMeta.vue`

### 7.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `drawingVersion` | Object | DrawingVersion 详情对象 |
| `confirmLoading` | Boolean | 确认按钮 loading 状态 |

### 7.2 模板结构

```vue
<template>
  <div class="drawing-meta-header">
    <!-- 左侧标题区 -->
    <div class="meta-left">
      <h2 class="meta-title">{{ drawingVersion.description }}</h2>
      <p class="meta-sub">
        {{ drawingVersion.category }} · Updated {{ formatDate(drawingVersion.updatedAt) }}
      </p>
    </div>
    <!-- 右侧按钮区 -->
    <div class="meta-right">
      <!-- 未确认 -->
      <el-button
        v-if="!drawingVersion.confirmed"
        type="primary"
        :loading="confirmLoading"
        @click="$emit('confirm')"
      >
        ✓ Confirm Reading
      </el-button>
      <!-- 已确认 -->
      <span v-else class="confirmed-text">
        ✓ Confirmed on {{ formatDate(drawingVersion.confirmedAt) }}
      </span>
    </div>
  </div>
</template>
```

> **说明**：系统版本号（`versionNo`）不在 Header 中渲染。

---

## 8. Confirm Reading 操作（`detail.vue`）

### 8.1 确认流程

```javascript
async handleConfirm() {
  try {
    await this.$confirm(
      'Confirm that you have read this drawing?',
      'Confirm Reading',
      {
        type: 'info',
        confirmButtonText: 'Confirm',
        cancelButtonText: 'Cancel',
        message: `${this.drawingVersion.description} · ${this.drawingVersion.category}`
      }
    )
  } catch {
    return  // 用户取消
  }
  this.confirmLoading = true
  try {
    await confirmDrawingRead({
      drawingVersionId: this.drawingVersion.id,
      deviceInfo: 'PC'
    })
    this.drawingVersion.confirmed = true
    this.drawingVersion.confirmedAt = new Date().toISOString()
    this.$message.success('Reading confirmed successfully')
  } catch {
    this.$message.error('Confirm failed, please try again')
  } finally {
    this.confirmLoading = false
  }
}
```

> `deviceInfo` 固定传 `'PC'`。跨端幂等：同一用户对同一 DrawingVersion 唯一，已在 APP 确认则 PC 显示已确认。

---

## 9. 图纸预览组件 `DrawingPreview.vue`

### 9.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `fileUrl` | String | 图纸文件 OSS URL |

### 9.2 预览策略

| 文件类型 | 预览方式 |
|---------|---------|
| `.pdf` | `<iframe :src="fileUrl" />` 或 pdf.js Viewer |
| `.png` / `.jpg` / `.jpeg` | `<img :src="fileUrl" />` |
| `.dwg`（转换后 PDF） | 同 PDF |

### 9.3 缩放 & 平移

```javascript
handleWheel(e) {
  e.preventDefault()
  const delta = e.deltaY > 0 ? 0.9 : 1.1
  this.scale = Math.min(Math.max(this.scale * delta, 0.25), 5)
},
handleFullscreen() {
  this.$el.requestFullscreen()
}
```

---

## 10. Markups Tab 面板 `MarkupTabPanel.vue`

### 10.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `drawingVersionId` | Number | DrawingVersion ID |

### 10.2 数据加载

```javascript
created() { this.fetchMarkups() },
methods: {
  fetchMarkups() {
    // 仅加载 ACTIVE 状态，按 publishTime 降序
    getSeMarkupList({ drawingVersionId: this.drawingVersionId, status: 'ACTIVE' })
      .then(res => { this.markups = res.list })
  }
}
```

> 接口参数为 `drawingVersionId`（非 `drawingId`），对应 DrawingMarkup 的 FK。

### 10.3 Markup 卡片组件 `MarkupCard.vue`

Props:

| Prop | 类型 | 说明 |
|------|------|------|
| `markup` | Object | DrawingMarkup 数据 |

卡片显示字段：

| 元素 | 字段 | 说明 |
|------|------|------|
| [New] 标签 | `markup.confirmed === false` | 橙色胶囊；已读后隐藏 |
| 标题 | `markup.description` | 14px 加粗 |
| 元信息行 | `markup.publishTime` + `markup.creatorName` | `{日期} · {创建人}`，12px，#909399 |
| 报审号+页码行 | `markup.submissionNo` + `markup.appliedPageNo` | `Based on {submissionNo}`；若 `appliedPageNo` 不为 null 追加 ` · Page {n}`；12px，#909399 |
| 说明文字 | `markup.content` | 最多 3 行，"…Show more" 展开 |
| 附件列表 | `markup.attachments[]` | 📎 文件名，点击新 Tab 打开 |
| 操作区 | `markup.confirmed` | 未读 → `[Mark as Read]`；已读 → `✓ Read` |

> **不渲染的字段**：`affectedArea`（影响区域）在 SE 视图卡片中不显示。

### 10.4 Mark as Read 操作

```javascript
async handleMarkAsRead(markup) {
  this.$set(markup, 'confirmLoading', true)
  try {
    await confirmMarkupRead({ markupId: markup.id, deviceInfo: 'PC' })
    markup.confirmed = true
    // 通知父组件更新列表页 Markups 列计数
    this.$emit('markup-read')
  } catch {
    this.$message.error('Failed to mark as read, please try again')
  } finally {
    this.$set(markup, 'confirmLoading', false)
  }
}
```

---

## 11. PC 站内通知

### 11.1 NotificationPanel.vue 结构

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
        <span>Notifications <template v-if="unreadCount">({{ unreadCount }} unread)</template></span>
        <el-button size="mini" type="text" :disabled="unreadCount === 0" @click="markAllRead">
          Mark all as read
        </el-button>
      </div>
      <NotificationItem
        v-for="item in notifications"
        :key="item.id"
        :notification="item"
        @click.native="handleNotificationClick(item)"
      />
    </div>
  </el-popover>
</template>
```

### 11.2 未读数量轮询（Vuex user module）

```javascript
actions: {
  startNotificationPolling({ dispatch }) {
    dispatch('fetchUnreadCount')
    setInterval(() => dispatch('fetchUnreadCount'), 30000) // 30s 轮询
  },
  async fetchUnreadCount({ commit }) {
    const res = await getNotificationUnreadCount()
    commit('SET_NOTIFICATION_UNREAD', res.count)
  }
}
```

### 11.3 通知点击跳转

```javascript
handleNotificationClick(notification) {
  markNotificationRead(notification.id).then(() => {
    this.$store.dispatch('user/fetchUnreadCount')
  })
  if (notification.targetRoute) {
    this.$router.push(notification.targetRoute)
    // targetRoute 示例: "/drawings/101?tab=markups"
  }
  this.popoverVisible = false
}
```

---

## 12. API 封装

### `api/se-drawing.js`

```javascript
// SE 专用图纸列表（后端自动过滤 ACTIVE + 已分配给当前用户）
export const getSeDrawingList = (params) =>
  request.get('/drawing/se/page', { params })
// 响应字段包含: id, description, category, rfaNo, subjectOfRfa,
//               confirmed, confirmedAt, markupsTotal, markupsUnread, updatedAt

// SE DrawingVersion 详情
export const getSeDrawingDetail = (drawingVersionId) =>
  request.get('/drawing/se/get', { params: { drawingVersionId } })
// 响应字段包含: id, description, category, fileUrl, confirmed, confirmedAt,
//               markupsUnread, markupsTotal, updatedAt

// 确认图纸已读
export const confirmDrawingRead = (data) =>
  request.post('/drawing/confirm', data)
// data: { drawingVersionId, deviceInfo: 'PC' }

// SE 获取 DrawingVersion 的 Markup 列表（仅 ACTIVE）
export const getSeMarkupList = (params) =>
  request.get('/drawing/markup/list', { params })
// params: { drawingVersionId, status: 'ACTIVE' }
// 响应字段: id, description, publishTime, creatorName, submissionNo,
//           appliedPageNo, content, attachments[], confirmed

// 确认 Markup 已读（Mark as Read）
export const confirmMarkupRead = (data) =>
  request.post('/drawing/markup/confirm', data)
// data: { markupId, deviceInfo: 'PC' }
```

### `api/notification.js`

```javascript
export const getNotificationList = (params) =>
  request.get('/notification/list', { params })

export const getNotificationUnreadCount = () =>
  request.get('/notification/unread-count')

export const markNotificationRead = (id) =>
  request.patch(`/notification/${id}/read`)

export const markAllNotificationsRead = () =>
  request.patch('/notification/read-all')
```

---

## 13. 路由配置

```javascript
{
  path: 'drawings',
  name: 'SeDrawings',
  component: () => import('@/views/se-drawings/index.vue'),
  meta: { title: 'Drawings', roles: ['SITE_ENGINEER'] }
},
{
  path: 'drawings/:drawingVersionId',
  name: 'SeDrawingDetail',
  component: () => import('@/views/se-drawings/detail.vue'),
  meta: { title: 'Drawing Detail', roles: ['SITE_ENGINEER'] }
}
```

> 路由守卫检查 `roles`，非 Site Engineer 角色访问 `/drawings` 时重定向到 `/403`。

---

## 14. 国际化（i18n）

```
seDrawing.title                  → "Drawings" / "我的图纸"
seDrawing.col.description        → "Description" / "批次描述"
seDrawing.col.category           → "Category" / "分类"
seDrawing.col.rfaNo              → "RFA No." / "报审编号"
seDrawing.col.subjectOfRfa       → "Subject of RFA" / "报审主题"
seDrawing.col.status             → "Status" / "状态"
seDrawing.col.markups            → "Markups" / "局部更新"
seDrawing.col.confirmed          → "Confirmed" / "确认时间"
seDrawing.col.lastUpdated        → "Last Updated" / "最后更新"
seDrawing.status.read            → "✓ Read" / "✓ 已读"
seDrawing.status.confirm         → "Confirm →" / "待确认 →"
seDrawing.empty                  → "No drawings assigned to you yet."
seDrawing.btn.confirmReading     → "✓ Confirm Reading" / "✓ 确认查阅"
seDrawing.confirmedText          → "✓ Confirmed on {date}" / "✓ 已于 {date} 确认"
seDrawing.tab.preview            → "Drawing Preview" / "图纸预览"
seDrawing.tab.markups            → "Markups" / "局部更新"
markup.btn.markAsRead            → "✓ Mark as Read" / "✓ 标记已读"
markup.status.read               → "✓ Read" / "✓ 已读"
markup.badge.new                 → "New" / "新"
markup.basedOn                   → "Based on {submissionNo}"
markup.page                      → "· Page {n}"
notification.markAllRead         → "Mark all as read" / "全部标为已读"
```

---

## 15. 验收标准

### Site Engineer 图纸列表

- [ ] 只显示 `status = ACTIVE` 且已分配给当前 SE 的 DrawingVersion
- [ ] 列表包含：Description、Category、RFA No.、Subject of RFA、Status、Markups、Confirmed、Last Updated
- [ ] **不显示** 系统版本号（versionNo）、Drawing Code、Drawing Name
- [ ] Description 模糊搜索和 Category 筛选正常生效
- [ ] RFA No. / Subject of RFA：有值时显示具体值，无值时显示 `—`
- [ ] Status 列已确认显示 `✓ Read`（绿），未确认显示 `Confirm →`（蓝色可点击）
- [ ] Markups 列：`unread>0` → `●{unread}/{total}` 橙色可点击；全部已读 → `0/{total}` 灰色；无 Markup → `—`

### Confirm Reading（Header 位置）

- [ ] [Confirm Reading] 按钮在 Header 右侧，无需滚动可见
- [ ] 点击弹出确认对话框，副文字含 description 和 category
- [ ] 确认后 `deviceInfo: 'PC'`，Header 替换为已确认文字
- [ ] 在 APP 端确认后，PC 端刷新显示"已确认"（跨端同步）

### Markups Tab 卡片

- [ ] 卡片显示 `{日期} · {创建人}` 元信息行
- [ ] 卡片显示 `Based on {submissionNo}` 报审行；`appliedPageNo` 不为 null 时追加 ` · Page {n}`
- [ ] 卡片**不显示** Affected Area 字段
- [ ] 未读条目显示 [New] 标签和 [Mark as Read] 按钮
- [ ] Mark as Read 成功后替换为 `✓ Read`；列表 Markups 列角标减一

### 站内通知

- [ ] Markup 发布后，铃铛角标 +1（30s 内轮询更新）
- [ ] 点击通知跳转到图纸详情页并激活 Markups Tab
- [ ] 未被分配图纸的 SE 不收到通知
- [ ] Mark all as read 功能将所有通知置为已读，角标清零
