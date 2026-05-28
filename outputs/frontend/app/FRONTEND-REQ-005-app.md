---
doc_type: frontend_spec
runtime: UNIAPP + Vue 2（iOS & Android）
source_req: REQ-005-app (v0.1.0)
ui_doc: UI-REQ-005-app.md (v0.1.0)
generated_at: 2026-05-28
---

# 前端说明文档：APP 端 Site Engineer 图纸查阅与局部更新查看

> 本文档依据 [REQ-005-app.md v0.1.0](../../../requirements/app/REQ-005-app.md) 与 [UI-REQ-005-app.md](../../ui/app/UI-REQ-005-app.md) 生成，描述 UNIAPP + Vue 2 实现细节。

---

## 1. 功能概述

| 功能 | 说明 |
|------|------|
| 图纸列表 | 卡片显示 Description / Category / RFA 信息行；角标只显示 unread 数；不显示版本号 |
| 图纸在线查看 | PDF/PNG/JPG 预览，双指缩放+单指平移；底部固定 Confirm Reading |
| Confirm Reading | 底部 Popup 确认，写入 DrawingConfirmation（deviceInfo="APP"），跨端幂等 |
| Markups Tab | 卡片含元信息行（`{日期}·{创建人}`）+ 报审行（`Based on {submissionNo}·Page {n}`）；无 Affected Area |
| Mark as Read | 调用 `/drawing/markup/confirm`，卡片实时更新；跨端同步 |
| App Push — Markup 发布 | 热启动/冷启动跳转到图纸查看页 Markups Tab（F-004） |
| App Push — 新版图纸发布 | 热启动/冷启动跳转到图纸列表页并高亮目标条目（F-005） |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | UNIAPP + Vue 2 |
| HTTP | `utils/request.js`（Axios 封装） |
| PDF 预览 | `<web-view>` 内嵌 PDF.js；或调用 `plus.runtime.openFile` |
| 图片预览 | `uni.previewImage` |
| Push 通知 | UniPush（`uni.getPushClientId`）；`plus.push.addEventListener`（热启动）；`plus.runtime.arguments`（冷启动） |
| 日期格式 | `dayjs`；通知/卡片 `MMM D, YYYY`；列表 `YYYY-MM-DD` |
| 国际化 | Vue I18n，中英双语 |

---

## 3. 文件目录结构

```
pages/
  drawings/
    list.vue                  # 图纸列表页
    detail.vue                # 图纸在线查看页（含 Confirm Reading + Markups Tab）
components/
  drawings/
    DrawingVersionCard.vue    # DrawingVersion 列表卡片
    DrawingPreview.vue        # 图纸预览区（WebView / 图片）
    MarkupCard.vue            # 单条 Markup 卡片
    MarkupTabPanel.vue        # Markups Tab 面板
api/
  se-drawing.js               # SE 图纸相关接口
utils/
  push-handler.js             # App Push 通知处理（冷/热启动跳转）
```

---

## 4. 图纸列表页 `list.vue`

### 4.1 数据状态

```javascript
data() {
  return {
    list: [],           // DrawingVersion 列表（累积追加）
    pageNo: 1,
    pageSize: 20,
    total: 0,
    loading: false,
    refreshing: false,
    noMore: false,
    keyword: '',        // Description 模糊搜索
    categoryFilter: ''  // Category 筛选
  }
}
```

### 4.2 数据加载

```javascript
async fetchList(reset = false) {
  if (reset) { this.pageNo = 1; this.list = []; this.noMore = false }
  if (this.loading || this.noMore) return
  this.loading = true
  try {
    const res = await getSeDrawingList({
      keyword: this.keyword,
      category: this.categoryFilter,
      pageNo: this.pageNo,
      pageSize: this.pageSize
    })
    // 响应字段: id, description, category, rfaNo, subjectOfRfa,
    //           confirmed, confirmedAt, markupsUnread, updatedAt
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

---

## 5. DrawingVersion 卡片组件 `DrawingVersionCard.vue`

### 5.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `item` | Object | DrawingVersion 数据 |

### 5.2 模板结构

```vue
<template>
  <view class="drawing-card" @click="$emit('click')">
    <!-- 右上角 Markups 未读角标 -->
    <view v-if="item.markupsUnread > 0" class="markups-badge">
      {{ item.markupsUnread > 99 ? '99+' : item.markupsUnread }}
    </view>

    <!-- 主信息区 -->
    <view class="card-body">
      <text class="card-description">{{ item.description }}</text>
      <text class="card-category">{{ item.category }}</text>

      <!-- RFA 信息行：rfaNo 有值则显示 rfaNo · subjectOfRfa，否则显示 — -->
      <text class="card-rfa">
        {{ item.rfaNo ? `${item.rfaNo} · ${item.subjectOfRfa}` : '—' }}
      </text>

      <!-- 底部：Updated + 状态 -->
      <view class="card-footer">
        <text class="card-updated">Updated {{ formatDate(item.updatedAt) }}</text>
        <text :class="item.confirmed ? 'status-read' : 'status-confirm'">
          {{ item.confirmed ? '✓ Read' : 'Confirm →' }}
        </text>
      </view>
    </view>
  </view>
</template>
```

> **不渲染的字段**：`versionNo`、`drawingCode`、`drawingName` 均不在卡片中输出。

### 5.3 样式

```css
.drawing-card {
  position: relative;
  background: #fff;
  border-radius: 8px;
  padding: 28rpx 32rpx;
  margin-bottom: 20rpx;
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
}
.markups-badge {
  position: absolute;
  top: 24rpx;
  right: 24rpx;
  min-width: 36rpx;
  height: 36rpx;
  background: #F57F17;
  color: #fff;
  font-size: 22rpx;
  border-radius: 18rpx;
  text-align: center;
  line-height: 36rpx;
  padding: 0 8rpx;
}
.status-confirm { color: #409EFF; }
.status-read    { color: #2E7D32; }
```

---

## 6. 图纸在线查看页 `detail.vue`

### 6.1 页面参数

| 参数 | 来源 | 说明 |
|------|------|------|
| `drawingVersionId` | `$route.params` 或 `onLoad(options)` | DrawingVersion ID |
| `tab` | `$route.query` 或 `onLoad(options)` | 初始激活 Tab：`'preview'`（默认）/ `'markups'` |

### 6.2 数据状态

```javascript
data() {
  return {
    drawingVersion: null,
    activeTab: 'preview',
    confirmLoading: false,
    showConfirmPopup: false
  }
}
```

### 6.3 数据加载

```javascript
onLoad(options) {
  this.activeTab = options.tab || 'preview'
  this.fetchDetail(options.drawingVersionId)
},
methods: {
  fetchDetail(id) {
    getSeDrawingDetail(id).then(res => { this.drawingVersion = res })
  }
}
```

### 6.4 页面结构

```vue
<template>
  <view class="detail-page">
    <!-- 顶部导航（自定义导航栏） -->
    <view class="nav-bar">
      <uni-icons type="left" @click="$router.back()" />
      <view class="nav-title">
        <text class="title-main">{{ drawingVersion && drawingVersion.description }}</text>
        <text class="title-sub">{{ drawingVersion && drawingVersion.category }}</text>
      </view>
    </view>

    <!-- Tab 栏 -->
    <view class="tab-bar">
      <view :class="['tab-item', activeTab==='preview'?'active':'']" @click="activeTab='preview'">
        Drawing Preview
      </view>
      <view :class="['tab-item', activeTab==='markups'?'active':'']" @click="activeTab='markups'">
        Markups
        <text v-if="drawingVersion && drawingVersion.markupsUnread > 0" class="tab-badge">
          ●{{ drawingVersion.markupsUnread }}
        </text>
      </view>
    </view>

    <!-- Tab 内容 -->
    <view v-show="activeTab==='preview'">
      <DrawingPreview
        v-if="drawingVersion"
        :file-url="drawingVersion.fileUrl"
      />
    </view>
    <view v-show="activeTab==='markups'">
      <MarkupTabPanel
        v-if="drawingVersion"
        :drawing-version-id="drawingVersion.id"
        @markup-read="onMarkupRead"
      />
    </view>

    <!-- 底部 Confirm Reading（仅 preview tab 显示） -->
    <view v-if="activeTab==='preview' && drawingVersion" class="confirm-bar">
      <button
        v-if="!drawingVersion.confirmed"
        class="btn-primary"
        :loading="confirmLoading"
        @click="showConfirmPopup = true"
      >✓ Confirm Reading</button>
      <text v-else class="confirmed-text">
        ✓ Confirmed on {{ formatDate(drawingVersion.confirmedAt) }}
      </text>
    </view>

    <!-- Confirm Popup -->
    <uni-popup ref="confirmPopup" type="dialog">
      <uni-popup-dialog
        title="Confirm Reading"
        :content="`Confirm that you have read this drawing?\n${drawingVersion && drawingVersion.description} · ${drawingVersion && drawingVersion.category}`"
        confirm-text="Confirm"
        cancel-text="Cancel"
        @confirm="handleConfirm"
        @close="showConfirmPopup = false"
      />
    </uni-popup>
  </view>
</template>
```

---

## 7. Confirm Reading 操作

```javascript
async handleConfirm() {
  this.showConfirmPopup = false
  this.confirmLoading = true
  try {
    await confirmDrawingRead({
      drawingVersionId: this.drawingVersion.id,
      deviceInfo: 'APP'
    })
    this.drawingVersion.confirmed = true
    this.drawingVersion.confirmedAt = new Date().toISOString()
    uni.showToast({ title: 'Confirmed!', icon: 'success' })
  } catch {
    uni.showToast({ title: 'Confirm failed, please try again', icon: 'none' })
  } finally {
    this.confirmLoading = false
  }
}
```

> `deviceInfo` 固定传 `'APP'`。跨端幂等：PC 端已确认则 APP 显示已确认。

---

## 8. 图纸预览组件 `DrawingPreview.vue`

```vue
<template>
  <view class="preview-container">
    <!-- PDF -->
    <web-view
      v-if="isPdf"
      :src="pdfViewerUrl"
      class="pdf-webview"
    />
    <!-- 图片 -->
    <image
      v-else
      :src="fileUrl"
      mode="widthFix"
      @tap="handleImagePreview"
    />
    <!-- 加载失败 -->
    <view v-if="loadFailed" class="error-state">
      <text>⚠ Failed to load drawing.</text>
      <text class="retry-link" @tap="reload">Retry</text>
    </view>
  </view>
</template>
```

```javascript
computed: {
  isPdf() { return this.fileUrl && this.fileUrl.endsWith('.pdf') },
  pdfViewerUrl() {
    return `/hybrid/html/pdf-viewer.html?file=${encodeURIComponent(this.fileUrl)}`
  }
},
methods: {
  handleImagePreview() {
    uni.previewImage({ urls: [this.fileUrl], current: this.fileUrl })
  }
}
```

---

## 9. Markups Tab 面板 `MarkupTabPanel.vue`

### 9.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `drawingVersionId` | Number | DrawingVersion ID |

### 9.2 数据加载

```javascript
created() { this.fetchMarkups() },
methods: {
  fetchMarkups() {
    // 仅加载 ACTIVE，按 publishTime DESC
    getSeMarkupList({ drawingVersionId: this.drawingVersionId, status: 'ACTIVE' })
      .then(res => { this.markups = res.list })
  }
}
```

---

## 10. Markup 卡片组件 `MarkupCard.vue`

### 10.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `markup` | Object | DrawingMarkup 数据 |

### 10.2 渲染字段

| 元素 | 字段 | 说明 |
|------|------|------|
| [New] 标签 | `!markup.confirmed` | 橙色胶囊；已读后隐藏 |
| 标题 | `markup.description` | 16px，加粗 |
| 元信息行 | `markup.publishTime` + `markup.creatorName` | `{日期} · {creatorName}`，13px，#909399 |
| 报审号+页码行 | `markup.submissionNo` + `markup.appliedPageNo` | `Based on {submissionNo}`；appliedPageNo 不为 null 时追加 ` · Page {n}` |
| 说明 | `markup.content` | 最多 3 行，"Show more" 展开 |
| 附件 | `markup.attachments[]` | 📎 文件名，点击打开 |
| 操作 | `markup.confirmed` | 未读 → `[Mark as Read]`；已读 → `✓ Read` |

> **不渲染的字段**：`affectedArea` 在 SE 卡片中不输出。

### 10.3 模板示例

```vue
<template>
  <view class="markup-card">
    <!-- 标题行 -->
    <view class="card-title-row">
      <view v-if="!markup.confirmed" class="badge-new">New</view>
      <text class="card-title">{{ markup.description }}</text>
    </view>

    <!-- 元信息行 -->
    <text class="card-meta">
      {{ formatDate(markup.publishTime) }} · {{ markup.creatorName }}
    </text>

    <!-- 报审号+页码行 -->
    <text v-if="markup.submissionNo" class="card-ref">
      Based on {{ markup.submissionNo }}{{ markup.appliedPageNo != null ? ` · Page ${markup.appliedPageNo}` : '' }}
    </text>

    <!-- 说明文字 -->
    <text class="card-content" :class="{collapsed: !expanded}">{{ markup.content }}</text>
    <text v-if="shouldShowMore" class="show-more" @tap="expanded = !expanded">
      {{ expanded ? 'Show less' : 'Show more' }}
    </text>

    <!-- 附件 -->
    <view v-for="att in markup.attachments" :key="att.url" class="attachment"
          @tap="openAttachment(att.url)">
      <text>📎 {{ att.name }}</text>
    </view>

    <!-- 操作区 -->
    <view class="card-action">
      <button
        v-if="!markup.confirmed"
        class="btn-mark-read"
        :loading="markup.confirmLoading"
        @click="$emit('mark-read', markup)"
      >✓ Mark as Read</button>
      <text v-else class="read-text">✓ Read</text>
    </view>
  </view>
</template>
```

### 10.4 Mark as Read（在 MarkupTabPanel 中处理）

```javascript
async handleMarkRead(markup) {
  this.$set(markup, 'confirmLoading', true)
  try {
    await confirmMarkupRead({ markupId: markup.id, deviceInfo: 'APP' })
    markup.confirmed = true
    this.$emit('markup-read')
    uni.showToast({ title: '✓ Marked as read', icon: 'success' })
  } catch {
    uni.showToast({ title: 'Failed, please try again', icon: 'none' })
  } finally {
    this.$set(markup, 'confirmLoading', false)
  }
}
```

---

## 11. App Push 通知处理 `push-handler.js`

```javascript
// 热启动：APP 在前台时收到推送
export function initPushListener(router) {
  plus.push.addEventListener('receive', (msg) => {
    const payload = JSON.parse(msg.payload || '{}')
    uni.showModal({
      title: msg.title,
      content: msg.content,
      confirmText: 'View',
      success({ confirm }) {
        if (confirm) navigateByPayload(router, payload)
      }
    })
  })

  // 用户点击通知（APP 在后台）
  plus.push.addEventListener('click', (msg) => {
    const payload = JSON.parse(msg.payload || '{}')
    navigateByPayload(router, payload)
  })
}

// 冷启动（在 App.vue onLaunch 中调用）
export function handleColdStart() {
  const args = plus.runtime.arguments
  if (!args) return
  try {
    const payload = JSON.parse(args)
    navigateByPayload(null, payload)
  } catch {}
}

/**
 * 根据 payload.type 路由跳转
 * MARKUP_PUBLISHED  → 图纸查看页 Markups Tab（F-004）
 * DRAWING_PUBLISHED → 图纸列表页并高亮目标条目（F-005）
 */
function navigateByPayload(router, payload) {
  if (payload.type === 'MARKUP_PUBLISHED') {
    uni.navigateTo({
      url: `/pages/drawings/detail?drawingVersionId=${payload.drawingVersionId}&tab=markups`
    })
  } else if (payload.type === 'DRAWING_PUBLISHED') {
    uni.navigateTo({
      url: `/pages/drawings/list?highlightId=${payload.drawingVersionId}`
    })
  }
}
```

---

## 12. API 封装 `api/se-drawing.js`

```javascript
// SE DrawingVersion 列表（后端自动过滤 ACTIVE + 已分配）
export const getSeDrawingList = (params) =>
  request.get('/drawing/se/page', { params })
// 响应字段: id, description, category, rfaNo, subjectOfRfa,
//           confirmed, confirmedAt, markupsUnread, updatedAt

// SE DrawingVersion 详情
export const getSeDrawingDetail = (drawingVersionId) =>
  request.get('/drawing/se/get', { params: { drawingVersionId } })
// 响应字段: id, description, category, fileUrl,
//           confirmed, confirmedAt, markupsUnread, updatedAt

// 确认图纸已读
export const confirmDrawingRead = (data) =>
  request.post('/drawing/confirm', data)
// data: { drawingVersionId, deviceInfo: 'APP' }

// SE 获取 DrawingVersion 的 Markup 列表（仅 ACTIVE）
export const getSeMarkupList = (params) =>
  request.get('/drawing/markup/list', { params })
// params: { drawingVersionId, status: 'ACTIVE' }
// 响应字段: id, description, publishTime, creatorName,
//           submissionNo, appliedPageNo, content, attachments[], confirmed

// Markup 标记已读
export const confirmMarkupRead = (data) =>
  request.post('/drawing/markup/confirm', data)
// data: { markupId, deviceInfo: 'APP' }
```

---

## 13. 路由与页面配置（pages.json）

```json
{
  "pages": [
    {
      "path": "pages/drawings/list",
      "style": {
        "navigationBarTitleText": "Drawings",
        "enablePullDownRefresh": true
      }
    },
    {
      "path": "pages/drawings/detail",
      "style": {
        "navigationBarTitleText": "Drawing Detail",
        "navigationStyle": "custom"
      }
    }
  ]
}
```

> `detail` 页使用自定义导航栏（`navigationStyle: "custom"`），以便在标题区同时显示 Description 和 Category，并隐藏版本号。

---

## 14. 国际化（i18n）

```
seDrawing.title                 → "Drawings" / "我的图纸"
seDrawing.col.description       → "Description" / "批次描述"
seDrawing.col.category          → "Category" / "分类"
seDrawing.col.rfaNo             → "RFA No." / "报审编号"
seDrawing.col.subjectOfRfa      → "Subject of RFA" / "报审主题"
seDrawing.status.read           → "✓ Read" / "✓ 已读"
seDrawing.status.confirm        → "Confirm →" / "待确认 →"
seDrawing.empty                 → "No drawings assigned to you yet."
seDrawing.btn.confirmReading    → "✓ Confirm Reading" / "✓ 确认查阅"
seDrawing.confirmedText         → "✓ Confirmed on {date}"
seDrawing.tab.preview           → "Drawing Preview"
seDrawing.tab.markups           → "Markups"
markup.btn.markAsRead           → "✓ Mark as Read" / "✓ 标记已读"
markup.status.read              → "✓ Read" / "✓ 已读"
markup.badge.new                → "New" / "新"
markup.basedOn                  → "Based on {submissionNo}"
markup.page                     → "· Page {n}"
push.markup.title               → "Drawing Update — {description}"
push.markup.body                → "{markupTitle}. Please review the latest markup."
push.drawing.title              → "Drawing Updated"
push.drawing.body               → "{description} has a new version. Please confirm reading."
```

---

## 15. 验收标准

### 图纸列表

- [ ] 卡片显示 Description、Category、RFA 信息行（rfaNo/subjectOfRfa，无值时 `—`）
- [ ] **不显示** versionNo、drawingCode、drawingName
- [ ] Markups 角标只显示 unread 数；unread=0 时无角标
- [ ] 仅展示 ACTIVE + 已分配给当前用户的 DrawingVersion
- [ ] Description 模糊搜索 + Category 筛选正常工作
- [ ] 下拉刷新 / 上拉加载更多正常工作

### 图纸查看页

- [ ] 顶部标题显示 Description + Category，不含版本号
- [ ] PDF 正常加载，图片支持双指缩放
- [ ] 底部 [Confirm Reading] 按钮，未确认时显示，已确认时显示绿色文字
- [ ] Confirm Popup 副文字含 description 和 category
- [ ] 确认后 `deviceInfo: 'APP'` 写入后端；底部替换为已确认文字
- [ ] PC 端确认后 APP 端刷新显示已确认（跨端同步）

### Markups Tab

- [ ] 卡片显示 `{日期} · {创建人}` 元信息行
- [ ] 卡片显示 `Based on {submissionNo}`；appliedPageNo 不为 null 时追加 ` · Page {n}`
- [ ] 卡片**不显示** Affected Area 字段
- [ ] Mark as Read 成功后卡片变为已读；角标减一
- [ ] PC 端标记已读后 APP 端刷新显示已读（跨端同步）

### App Push 通知

- [ ] Markup 发布后，已分配 SE 收到通知，APP 角标 +1
- [ ] 未分配图纸的 SE 不收到通知
- [ ] 热启动：点击 Markup 通知跳转到图纸查看页 Markups Tab
- [ ] 冷启动：解析 MARKUP_PUBLISHED payload 正确并跳转到目标页
- [ ] 热启动：点击新版图纸通知跳转到图纸列表页，目标条目高亮
- [ ] 冷启动：解析 DRAWING_PUBLISHED payload 正确并跳转图纸列表页
