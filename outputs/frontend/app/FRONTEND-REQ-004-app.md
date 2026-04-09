# 前端开发说明文档 — APP 端图纸标注查阅（Markups Tab）

> **来源需求**: [REQ-004-app.md](../../../requirements/app/REQ-004-app.md) + [REQ-004-shared.md](../../../requirements/shared/REQ-004-shared.md) + [REQ-005-shared.md](../../../requirements/shared/REQ-005-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: APP 移动端（UNIAPP + Vue 2，iOS & Android）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

本模块在图纸详情页（`detail.vue`）新增 **Markups Tab**，并扩展 App Push 通知路由，涵盖：

| 功能模块 | 说明 |
|---------|------|
| Markups Tab | 展示当前图纸 ACTIVE 状态的标注列表，含未读 [New] 角标 |
| Mark as Read | SE 查看标注后标记为已读（幂等接口） |
| 图纸列表未读角标 | 图纸卡片右上角橙色圆点（有未读 Markup 时显示）|
| Drawing Update Push | 标注发布时推送给已分配 SE，点击直接进入 Markups Tab |

---

## 2. 文件与目录结构

```
pages/
└── drawings/
    ├── list.vue                     # 图纸列表页（扩展：未读 Markup 角标）
    └── detail.vue                   # 图纸详情页（扩展：新增 Markups Tab）
components/
├── DrawingCard.vue                  # 图纸卡片（扩展：未读角标）
├── MarkupList.vue                   # Markups Tab 列表组件
├── MarkupCard.vue                   # 单条标注卡片
└── MarkupDetail.vue                 # 标注详情（从卡片点击进入）
api/
└── markup.js                        # Markup 相关 API 封装
```

---

## 3. 图纸详情页 Tabs 扩展 `detail.vue`

### 3.1 Tab Bar 结构

```vue
<template>
  <view class="drawing-detail">
    <uni-nav-bar :title="`${drawing.drawingCode} · ${drawing.currentVersionNo}`" left-icon="back" />

    <!-- Tab 导航 -->
    <uni-segmented-control
      :values="['Drawing Preview', `Markups (${markupCount})`]"
      :current="activeTab"
      @change="onTabChange"
    />

    <!-- Tab 内容 -->
    <view v-show="activeTab === 0" class="tab-content preview-tab">
      <DrawingPreviewWebView :file-url="drawing.currentVersionFileUrl" />
      <ConfirmReadingBar :drawing="drawing" @confirmed="onConfirmed" />
    </view>
    <view v-show="activeTab === 1" class="tab-content markup-tab">
      <MarkupList
        :drawing-id="drawingId"
        @markups-loaded="onMarkupsLoaded"
      />
    </view>
  </view>
</template>
```

### 3.2 页面初始化 — 支持 Push 跳转直接激活 Markups Tab

```javascript
onLoad(options) {
  this.drawingId = options.drawingId
  // fromPush=1 时直接激活 Markups Tab
  if (options.fromPush === '1') {
    this.activeTab = 1
  }
  this.loadDrawingDetail()
}
```

### 3.3 Markups Tab 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `markupList` | Array | Markup 列表（由子组件 `MarkupList` 管理）|
| `markupCount` | Number | ACTIVE Markup 总数，用于 Tab 标签显示 |
| `activeTab` | Number | 0=Drawing Preview，1=Markups |

---

## 4. Markup 列表组件 `MarkupList.vue`

### 4.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `drawingId` | String | 图纸 ID |

### 4.2 Emits

| 事件 | 说明 |
|------|------|
| `markups-loaded` | 传递 markups 数量 `{ count, unreadCount }` |

### 4.3 数据加载

```javascript
async loadMarkups() {
  this.loading = true
  try {
    const res = await getMarkupList({
      drawingId: this.drawingId,
      status: 'ACTIVE'
    })
    // 按 publishTime 降序
    this.markups = res.list.sort((a, b) => new Date(b.publishTime) - new Date(a.publishTime))
    this.$emit('markups-loaded', {
      count: this.markups.length,
      unreadCount: this.markups.filter(m => !m.isRead).length
    })
  } finally {
    this.loading = false
  }
}
```

### 4.4 列表渲染

```vue
<template>
  <view class="markup-list">
    <view v-if="loading" class="loading">
      <uni-load-more status="loading" />
    </view>
    <view v-else-if="markups.length === 0" class="empty">
      <text>No markups on this drawing.</text>
    </view>
    <scroll-view v-else scroll-y class="list-scroll">
      <MarkupCard
        v-for="m in markups"
        :key="m.id"
        :markup="m"
        @tap="openDetail(m)"
        @read="onMarkupRead(m)"
      />
    </scroll-view>
  </view>
</template>
```

---

## 5. Markup 卡片组件 `MarkupCard.vue`

### 5.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `markup` | Object | 标注数据 |

### 5.2 卡片结构

```vue
<template>
  <view class="markup-card" :class="{ unread: !markup.isRead }" @tap="$emit('tap')">
    <!-- 未读角标 -->
    <view v-if="!markup.isRead" class="new-badge">New</view>

    <view class="card-header">
      <text class="title">{{ markup.title }}</text>
      <text class="publisher">{{ markup.publisherName }}</text>
    </view>

    <text class="description">{{ markup.description }}</text>
    <text v-if="markup.affectedArea" class="affected-area">📍 {{ markup.affectedArea }}</text>

    <!-- 附件 -->
    <view v-if="markup.attachments && markup.attachments.length > 0" class="attachments">
      <view
        v-for="(att, i) in markup.attachments"
        :key="i"
        class="attachment-chip"
        @tap.stop="previewAttachment(att)"
      >
        <uni-icons type="paperclip" size="12" />
        <text>{{ att.fileName }}</text>
      </view>
    </view>

    <text class="publish-time">{{ formatDateTime(markup.publishTime) }}</text>
  </view>
</template>
```

### 5.3 附件预览

```javascript
previewAttachment(att) {
  if (['jpg', 'jpeg', 'png'].includes(att.fileType.toLowerCase())) {
    uni.previewImage({ urls: [att.fileUrl] })
  } else if (att.fileType.toLowerCase() === 'pdf') {
    uni.navigateTo({ url: `/pages/drawings/pdf-viewer?url=${encodeURIComponent(att.fileUrl)}` })
  }
}
```

---

## 6. Mark as Read 逻辑

### 6.1 触发时机

- **进入标注详情页**时自动调用（幂等，重复调用不报错）
- 也可在 `MarkupCard` tap 时触发（视产品需求，推荐在详情页触发）

### 6.2 实现

```javascript
// MarkupDetail.vue — onLoad
async onLoad(options) {
  this.markupId = options.markupId
  this.loadDetail()
  this.markAsRead()
}

async markAsRead() {
  try {
    await confirmMarkupRead({ markupId: this.markupId })
    // 通知父组件刷新未读状态
    uni.$emit('markupRead', { markupId: this.markupId })
  } catch (e) {
    // 幂等：失败不影响主流程
    console.warn('markAsRead failed', e)
  }
}
```

### 6.3 父组件监听刷新

```javascript
// MarkupList.vue — onLoad
uni.$on('markupRead', ({ markupId }) => {
  const m = this.markups.find(m => m.id === markupId)
  if (m) m.isRead = true
})
```

---

## 7. 图纸列表卡片扩展 — 未读 Markup 角标

`DrawingCard.vue` 中扩展未读角标（橙色圆点）：

```vue
<view class="card-badge-area">
  <!-- 未读 Markup 角标 -->
  <view v-if="drawing.unreadMarkupCount > 0" class="unread-dot" />
</view>
```

```scss
.unread-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background-color: #FF6B35;
  position: absolute;
  top: 8px;
  right: 8px;
}
```

`getSeDrawingList` 接口返回数据中应包含 `unreadMarkupCount` 字段；前端在 `markupRead` 事件触发后同步更新列表中对应图纸的 `unreadMarkupCount`。

---

## 8. App Push 通知扩展 — Drawing Update

### 8.1 Push Payload 格式

```json
{
  "type": "MARKUP_PUBLISHED",
  "drawingId": "xxx",
  "drawingCode": "DWG-001",
  "title": "Drawing Update — DWG-001",
  "body": "A new markup has been published for this drawing."
}
```

### 8.2 通知路由处理（扩展 `push-handler.js`）

```javascript
function handlePushNavigation(payload) {
  if (!payload) return
  if (payload.type === 'DRAWING_UPDATED' || payload.type === 'MARKUP_PUBLISHED') {
    uni.switchTab({ url: '/pages/home/index' })
    setTimeout(() => {
      uni.navigateTo({
        url: `/pages/drawings/detail?drawingId=${payload.drawingId}&fromPush=1`
      })
    }, 300)
  }
}
```

`fromPush=1` 参数使 `detail.vue` 在 `onLoad` 时自动激活 **Markups Tab**（见第 3.2 节）。

---

## 9. API 封装 `api/markup.js`

```javascript
import request from '@/utils/request'

// 获取图纸 Markup 列表
export const getMarkupList = (params) =>
  request({ url: '/drawing/markup/list', method: 'GET', data: params })

// 获取 Markup 详情
export const getMarkupDetail = (markupId) =>
  request({ url: '/drawing/markup/get', method: 'GET', data: { markupId } })

// 标记 Markup 已读
export const confirmMarkupRead = (data) =>
  request({ url: '/drawing/markup/confirm', method: 'POST', data })
```

---

## 10. 验收标准

### Markups Tab
- [ ] 图纸详情页新增 `Markups (n)` Tab，`n` 为 ACTIVE Markup 数量
- [ ] Tab 按 publishTime 降序展示 Markup 列表
- [ ] 未读 Markup 卡片左上角显示橙色 [New] 角标
- [ ] 已读 Markup 无角标，卡片背景色一致
- [ ] 无 Markup 时显示 "No markups on this drawing."
- [ ] 附件列表可点击预览（图片/PDF）

### Mark as Read
- [ ] 进入 Markup 详情页后自动调用 Mark as Read（`POST /drawing/markup/confirm`）
- [ ] 接口幂等：重复调用不报错（后端 409 时前端静默处理）
- [ ] 已读后卡片 [New] 角标消失
- [ ] 已读后图纸列表该图纸的橙色圆点随 `unreadMarkupCount` 减少而消失

### 图纸列表未读角标
- [ ] 有未读 Markup 的图纸卡片显示橙色圆点
- [ ] 全部 Markup 标记为已读后圆点消失
- [ ] 下拉刷新后未读状态实时同步

### App Push 通知
- [ ] 标注发布后已分配 SE 收到推送（"Drawing Update — {drawingCode}"）
- [ ] 点击推送 → 跳转至图纸详情页 **Markups Tab**（activeTab = 1）
- [ ] 冷启动时点击推送通知同样跳转到 Markups Tab
- [ ] 未分配给当前 SE 的图纸发布标注时，当前 SE 不收到推送
