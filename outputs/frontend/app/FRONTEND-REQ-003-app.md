# 前端开发说明文档 — APP 端工程图纸查阅确认与审批

> **来源需求**: [REQ-003-app.md](../../../requirements/app/REQ-003-app.md) + [REQ-003-shared.md](../../../requirements/shared/REQ-003-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: APP 移动端（UNIAPP + Vue 2，iOS & Android）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

本模块实现 APP 端的图纸查阅确认与审批功能，涵盖：

| 功能模块 | 说明 |
|---------|------|
| 图纸列表页 | 仅展示分配给当前 SE 且 ACTIVE 的图纸，支持搜索/分类筛选，含确认状态标识 |
| 图纸在线查看页 | PDF/PNG/JPG 在线预览，双指缩放 + 单指平移 |
| Confirm Reading | Site Engineer 在 APP 端完成图纸查阅确认 |
| 审批 Todo | 审批人在 APP Todo 列表中通过/驳回待审批图纸 |
| App Push 通知 | 图纸审批通过后向已分配 SE 推送通知，点击跳转列表 |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | UNIAPP + Vue 2 |
| HTTP | `utils/request.js` |
| PDF 预览 | `<web-view>` 内嵌 PDF.js 或调用 `plus.io` 下载后使用系统预览 |
| 图片预览 | `uni.previewImage` |
| 推送通知 | UNIAPP Push（`uni.getPushClientId` + 后端 UniPush 集成） |
| 通知跳转 | `plus.runtime.arguments` 解析推送 payload（冷启动）；前台接收监听（热启动） |

---

## 3. 文件与目录结构

```
pages/
└── drawings/
    ├── list.vue                     # 图纸列表页
    ├── detail.vue                   # 图纸在线查看页（含 Confirm Reading）
    └── markup-tab.vue               # (内联组件，或独立页面) Markups Tab（REQ-004 扩展）
components/
├── DrawingCard.vue                  # 图纸列表卡片
├── ConfirmReadingBar.vue            # 底部确认查阅操作栏
├── TodoDrawingCard.vue              # 审批 Todo 卡片
└── DrawingPreviewWebView.vue        # PDF/图片预览组件
api/
└── drawing.js                       # 图纸相关 API 封装
utils/
└── push-handler.js                  # App Push 通知处理
```

---

## 4. 图纸列表页 `list.vue`

### 4.1 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `list` | Array | 图纸列表（累积追加） |
| `pageNo` | Number | 当前页，默认 1 |
| `pageSize` | Number | 每页 20 |
| `total` | Number | 总记录数 |
| `loading` | Boolean | 加载状态 |
| `refreshing` | Boolean | 下拉刷新 |
| `noMore` | Boolean | 是否全部加载完 |
| `keyword` | String | 搜索关键词 |
| `categoryFilter` | String | 分类筛选 |

### 4.2 数据加载

调用 `GET /drawing/se/page`（后端自动过滤：status=ACTIVE + 已分配给当前用户）。

```javascript
async fetchList(reset = false) {
  if (reset) { this.pageNo = 1; this.list = [] }
  if (this.loading || this.noMore) return
  this.loading = true
  try {
    const res = await getSeDrawingList({
      keyword: this.keyword,
      category: this.categoryFilter,
      pageNo: this.pageNo,
      pageSize: this.pageSize
    })
    this.list = [...this.list, ...res.list]
    this.total = res.total
    this.noMore = this.list.length >= this.total
    this.pageNo++
  } finally {
    this.loading = false
    this.refreshing = false
  }
}
```

### 4.3 图纸卡片 `DrawingCard.vue`

```vue
<template>
  <view class="drawing-card" @tap="$emit('tap', drawing)">
    <view class="card-left">
      <text class="code">{{ drawing.drawingCode }}</text>
      <text class="name">{{ drawing.drawingName }}</text>
      <text class="meta">{{ drawing.category }} · {{ drawing.currentVersionNo }}</text>
      <text class="updated">Updated {{ formatDate(drawing.updatedAt) }}</text>
    </view>
    <view class="card-right">
      <!-- 已确认 -->
      <view v-if="drawing.confirmed" class="status-read">
        <uni-icons type="checkmarkempty" size="14" color="#52C41A" />
        <text class="status-text read">Read</text>
      </view>
      <!-- 未确认 -->
      <view v-else class="status-confirm" @tap.stop="$emit('confirm', drawing)">
        <text class="status-text confirm">Confirm →</text>
      </view>
      <!-- Markup 角标 -->
      <text v-if="drawing.unreadMarkupCount > 0" class="markup-badge">●{{ drawing.unreadMarkupCount }}</text>
    </view>
  </view>
</template>
```

### 4.4 空状态

```vue
<view v-if="!loading && list.length === 0" class="empty-state">
  <image src="/static/images/empty-drawing.png" />
  <text>No drawings assigned to you yet.</text>
</view>
```

---

## 5. 图纸在线查看页 `detail.vue`

### 5.1 路由参数

| 参数 | 说明 |
|------|------|
| `drawingId` | 图纸 ID |
| `fromPush` | `'1'`（从推送通知跳转）可选 |

### 5.2 页面结构

```vue
<template>
  <view class="drawing-detail">
    <!-- 导航栏 -->
    <uni-nav-bar :title="`${drawing.drawingCode} · ${drawing.currentVersionNo}`" left-icon="back" />

    <!-- 预览区域 -->
    <DrawingPreviewWebView :file-url="drawing.currentVersionFileUrl" class="preview-area" />

    <!-- 底部信息栏 -->
    <view class="bottom-bar">
      <text class="version-info">{{ drawing.currentVersionNo }} · Updated {{ formatDate(drawing.updatedAt) }}</text>
      <ConfirmReadingBar :drawing="drawing" @confirmed="onConfirmed" />
    </view>
  </view>
</template>
```

### 5.3 PDF 预览策略

| 文件类型 | 预览方式 |
|---------|---------|
| PDF | `<web-view>` 内嵌 PDF.js（`/static/pdfjs/web/viewer.html?file=<url>`）或调用 `plus.io` 下载后 `plus.runtime.openFile` |
| PNG / JPG | `uni.previewImage({ urls: [url] })` |
| DWG（已转 PDF） | 同 PDF |

缩放/平移（图片模式）：
```vue
<image
  :src="fileUrl"
  :style="{ transform: `scale(${scale}) translate(${translateX}px, ${translateY}px)` }"
  @touchstart="onTouchStart"
  @touchmove="onTouchMove"
/>
```
使用双指检测计算 `scale`，单指拖拽更新 `translateX`/`translateY`。

---

## 6. Confirm Reading 组件 `ConfirmReadingBar.vue`

### 6.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `drawing` | Object | 图纸对象，含 `confirmed`、`confirmedAt`、`currentVersionId` |

### 6.2 Emits

| 事件 | 说明 |
|------|------|
| `confirmed` | 确认成功，传递确认时间 |

### 6.3 UI 状态

```vue
<template>
  <view class="confirm-bar">
    <!-- 未确认 -->
    <button
      v-if="!drawing.confirmed"
      class="btn-confirm"
      :loading="loading"
      @tap="handleConfirm"
    >
      ✓ Confirm Reading
    </button>
    <!-- 已确认 -->
    <text v-else class="confirmed-text">
      ✓ Confirmed on {{ formatDate(drawing.confirmedAt) }}
    </text>
  </view>
</template>
```

### 6.4 确认流程

```javascript
async handleConfirm() {
  const confirmed = await showConfirmDialog({
    title: '',
    content: 'Confirm that you have read this drawing?',
    confirmText: 'Confirm'
  })
  if (!confirmed) return
  this.loading = true
  try {
    await confirmDrawingRead({
      drawingVersionId: this.drawing.currentVersionId,
      deviceInfo: 'APP'
    })
    this.$emit('confirmed', new Date().toISOString())
    uni.showToast({ title: 'Confirmed!', icon: 'success' })
  } catch (e) {
    uni.showToast({ title: e.message || 'Failed', icon: 'none' })
  } finally {
    this.loading = false
  }
}
```

---

## 7. 审批 Todo 卡片 `TodoDrawingCard.vue`

### 7.1 数据来源

在 APP 首页 / Todo 列表中，后端返回 `type = 'DRAWING_APPROVAL'` 的 Todo 条目，前端渲染专属卡片。

### 7.2 卡片结构

```vue
<template>
  <view class="todo-card">
    <view class="card-header">
      <uni-icons type="document" size="18" color="#4A90D9" />
      <text class="todo-type">Drawing Approval</text>
    </view>
    <view class="drawing-info">
      <text class="code-name">{{ todo.drawingCode }} {{ todo.drawingName }}</text>
      <text class="version">{{ todo.versionNo }}</text>
      <text class="uploader">Uploaded by {{ todo.uploaderName }}</text>
      <text class="time">{{ formatDateTime(todo.createTime) }}</text>
    </view>
    <view class="action-row">
      <button class="btn-view" @tap="handleView">View</button>
      <button class="btn-approve" :loading="approveLoading" @tap="handleApprove">Approve</button>
      <button class="btn-reject" @tap="handleReject">Reject</button>
    </view>
  </view>
</template>
```

### 7.3 通过操作

```javascript
async handleApprove() {
  const confirmed = await showConfirmDialog({
    content: 'Approve this drawing?',
    confirmText: 'Approve'
  })
  if (!confirmed) return
  this.approveLoading = true
  try {
    await approveDrawing({ versionId: this.todo.versionId })
    uni.showToast({ title: 'Drawing approved successfully', icon: 'success' })
    this.$emit('done', this.todo.id)
  } finally {
    this.approveLoading = false
  }
}
```

### 7.4 驳回操作（Bottom Sheet）

```javascript
handleReject() {
  this.rejectSheetVisible = true
  this.rejectComment = ''
}
```

驳回 Bottom Sheet：
```vue
<uni-popup type="bottom" :show="rejectSheetVisible">
  <view class="reject-sheet">
    <text class="sheet-title">Reject Drawing</text>
    <text class="label">Comment (Required):</text>
    <textarea v-model="rejectComment" class="comment-input" maxlength="500" />
    <view class="sheet-actions">
      <button @tap="rejectSheetVisible = false">Cancel</button>
      <button
        :disabled="!rejectComment.trim()"
        :loading="rejectLoading"
        @tap="confirmReject"
      >Confirm</button>
    </view>
  </view>
</uni-popup>
```

```javascript
async confirmReject() {
  if (!this.rejectComment.trim()) return
  this.rejectLoading = true
  try {
    await rejectDrawing({
      versionId: this.todo.versionId,
      comment: this.rejectComment
    })
    this.rejectSheetVisible = false
    uni.showToast({ title: 'Drawing rejected', icon: 'none' })
    this.$emit('done', this.todo.id)
  } finally {
    this.rejectLoading = false
  }
}
```

---

## 8. App Push 通知处理 `utils/push-handler.js`

### 8.1 初始化（在 App.vue 中调用）

```javascript
// App.vue — onLaunch
import { initPush } from '@/utils/push-handler'
export default {
  onLaunch(options) {
    initPush()
    this.handleColdStartPush(options)
  }
}
```

### 8.2 通知处理逻辑

```javascript
export function initPush() {
  // 获取 Push Client ID，发送给后端绑定
  uni.getPushClientId({
    success: ({ cid }) => {
      store.dispatch('user/bindPushCid', cid)
    }
  })

  // 热启动：前台接收推送
  uni.onPushMessage(msg => {
    if (msg.type === 'receive') {
      // 展示系统通知
      uni.showToast({ title: msg.data?.title || 'New notification', icon: 'none' })
    } else if (msg.type === 'click') {
      handlePushNavigation(msg.data)
    }
  })
}

function handlePushNavigation(payload) {
  if (!payload) return
  if (payload.type === 'DRAWING_UPDATED') {
    uni.switchTab({ url: '/pages/home/index' })
    // 稍后导航到图纸列表
    setTimeout(() => {
      uni.navigateTo({ url: '/pages/drawings/list' })
    }, 300)
  }
}
```

### 8.3 冷启动跳转

```javascript
// App.vue — onLaunch options.path / options.query
handleColdStartPush(options) {
  const payload = parseStartupPayload(options)
  if (payload?.type === 'DRAWING_UPDATED') {
    // 等待首页加载完成后跳转
    setTimeout(() => handlePushNavigation(payload), 500)
  }
}
```

---

## 9. API 封装 `api/drawing.js`

```javascript
// SE 专用图纸列表
export const getSeDrawingList = (params) => request({ url: '/drawing/se/page', method: 'GET', data: params })

// SE 图纸详情
export const getSeDrawingDetail = (drawingId) => request({ url: '/drawing/se/get', method: 'GET', data: { drawingId } })

// 确认图纸查阅
export const confirmDrawingRead = (data) => request({ url: '/drawing/confirm', method: 'POST', data })

// 审批通过
export const approveDrawing = (data) => request({ url: '/drawing/version/approve', method: 'POST', data })

// 审批驳回
export const rejectDrawing = (data) => request({ url: '/drawing/version/reject', method: 'POST', data })

// 获取 Todo 列表（含图纸审批）
export const getTodoList = (params) => request({ url: '/todo/list', method: 'GET', data: params })
```

---

## 10. 验收标准

### 图纸列表
- [ ] 仅显示 ACTIVE 且已分配给当前 SE 的图纸
- [ ] 未分配任何图纸时显示空状态 "No drawings assigned to you yet."
- [ ] 每张图纸卡片显示编号、名称、分类、版本号、更新时间
- [ ] 已确认显示 `✓ Read`（绿色），未确认显示 `Confirm →`（蓝色）
- [ ] 下拉刷新和上拉加载正常工作
- [ ] 搜索和分类筛选正确过滤

### 在线查看
- [ ] PDF 文件在 APP 内在线预览
- [ ] 图片支持双指缩放和单指平移
- [ ] 顶部显示图纸编号、名称和版本号

### Confirm Reading
- [ ] 未确认时显示 [Confirm Reading] 主按钮
- [ ] 点击弹出确认对话框，确认后调用 API（携带 `deviceInfo: 'APP'`）
- [ ] 确认成功后按钮变为 `✓ Confirmed on {date}`
- [ ] 返回列表，该图纸状态更新为 `✓ Read`

### 审批 Todo
- [ ] 审批 Todo 卡片正确显示图纸信息
- [ ] [Approve] 点击确认后 Todo 消失，Toast 提示成功
- [ ] [Reject] 点击弹出 Bottom Sheet，Comment 为空时 [Confirm] 禁用
- [ ] [View] 进入图纸在线预览页，不显示 [Confirm Reading] 按钮

### App Push 通知
- [ ] 图纸审批通过后已分配 SE 收到 App Push
- [ ] 未分配 SE 不收到通知
- [ ] 点击通知跳转到图纸列表页
- [ ] 冷启动后通知跳转正常
