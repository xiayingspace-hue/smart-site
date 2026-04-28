# FRONTEND-REQ-013-app.md

## SE"My Task"APP 端前端说明

> 对应需求：REQ-013
> 平台：APP 端（UNIAPP + Vue2）
> 影响模块：My Task 列表页、工序任务详情页

---

### 1. 页面结构

```
pages/
  myTask/
    index.vue          # My Task 列表页
    detail.vue         # 工序任务详情页
```

路由（pages.json）：
```json
{ "path": "pages/myTask/index",  "style": { "navigationBarTitleText": "My Task" } },
{ "path": "pages/myTask/detail", "style": { "navigationBarTitleText": "工序任务详情" } }
```

---

### 2. My Task 列表页（index.vue）

#### 2.1 状态枚举

```js
const STATUS = {
  NOT_STARTED: 'not_started',
  IN_PROGRESS:  'in_progress',
  COMPLETED:    'completed'
}

const STATUS_LABEL = {
  not_started: 'Not Started',
  in_progress: 'In Progress',
  completed:   'Completed'
}

const STATUS_COLOR = {
  not_started: '#909399',
  in_progress: '#E6A23C',
  completed:   '#67C23A'
}
```

#### 2.2 顶部 Tab 筛选

```js
data() {
  return {
    tabs: [
      { label: 'All',          value: '' },
      { label: 'Not Started',  value: 'not_started' },
      { label: 'In Progress',  value: 'in_progress' },
      { label: 'Completed',    value: 'completed' }
    ],
    activeTab: '',
    taskList: [],
    keyword: ''
  }
},
methods: {
  onTabChange(value) {
    this.activeTab = value
    this.fetchTasks()
  },
  fetchTasks() {
    getMyTaskList({ status: this.activeTab, keyword: this.keyword })
      .then(res => { this.taskList = res.data.list })
  }
}
```

#### 2.3 卡片状态标签

```html
<view class="status-tag" :style="{ color: getStatusColor(item.status) }">
  {{ getStatusLabel(item.status) }}
</view>
```

```js
methods: {
  getStatusLabel(status) { return STATUS_LABEL[status] || status },
  getStatusColor(status) { return STATUS_COLOR[status] || '#909399' }
}
```

---

### 3. 工序任务详情页（detail.vue）

#### 3.1 数据结构

```js
data() {
  return {
    task: {
      id: '',
      name: '',
      parentTaskId: '',
      parentTaskName: '',
      planStart: '',
      planEnd: '',
      actualStart: '',    // 系统记录，只读
      actualEnd: '',      // 系统记录，只读
      priority: '',
      weight: null,
      status: 'not_started',
      remark: ''
    },
    showCompleteConfirm: false
  }
}
```

#### 3.2 实际时间展示

```html
<view class="field-row">
  <text class="label">实际开始：</text>
  <text class="value">{{ task.actualStart || '—' }}</text>
</view>
<view class="field-row">
  <text class="label">实际结束：</text>
  <text class="value">{{ task.actualEnd || '—' }}</text>
</view>
```

---

### 4. 操作区按钮（三态逻辑）

#### 4.1 按钮 disabled 计算属性

```js
computed: {
  isStartTaskDisabled() {
    return this.task.status !== 'not_started'
  },
  isMarkCompleteDisabled() {
    return this.task.status !== 'in_progress'
  }
}
```

#### 4.2 按钮模板

```html
<view class="action-area">
  <!-- Start Task：仅 not_started 可点 -->
  <button
    :disabled="isStartTaskDisabled"
    :class="['btn-primary', { 'btn-disabled': isStartTaskDisabled }]"
    @click="handleStartTask">
    Start Task
  </button>

  <!-- Mark as Complete：仅 in_progress 可点 -->
  <button
    :disabled="isMarkCompleteDisabled"
    :class="['btn-primary', { 'btn-disabled': isMarkCompleteDisabled }]"
    @click="handleMarkComplete">
    Mark as Complete
  </button>

  <!-- Report Issue：始终可点 -->
  <button class="btn-outline" @click="handleReportIssue">
    Report Issue
  </button>
</view>
```

---

### 5. Start Task 处理（无确认弹框）

```js
async handleStartTask() {
  if (this.isStartTaskDisabled) return
  try {
    const res = await startProcessTask(this.task.id)
    // 更新本地状态
    this.task.status = 'in_progress'
    this.task.actualStart = res.data.actualStart  // 后端返回实际开始时间
    uni.showToast({ title: '任务已开始', icon: 'success' })
  } catch (e) {
    uni.showToast({ title: '操作失败，请重试', icon: 'none' })
  }
}
```

---

### 6. Mark as Complete 处理（有确认弹框）

```js
handleMarkComplete() {
  if (this.isMarkCompleteDisabled) return
  this.showCompleteConfirm = true
},

async confirmMarkComplete() {
  this.showCompleteConfirm = false
  try {
    const res = await completeProcessTask(this.task.id)
    this.task.status = 'completed'
    this.task.actualEnd = res.data.actualEnd  // 后端返回实际结束时间
    uni.showToast({ title: '已标记为完成', icon: 'success' })
  } catch (e) {
    uni.showToast({ title: '操作失败，请重试', icon: 'none' })
  }
}
```

确认弹框使用 uni-app 的 `uni.showModal` 或自定义组件：
```js
// 使用 uni.showModal（推荐，兼容性好）
handleMarkComplete() {
  if (this.isMarkCompleteDisabled) return
  uni.showModal({
    title: '确认完成',
    content: '确认将此工序任务标记为完成？完成后将无法修改。',
    success: async (res) => {
      if (res.confirm) {
        await this.confirmMarkComplete()
      }
    }
  })
}
```

---

### 7. Report Issue 处理

```js
handleReportIssue() {
  // 跳转至问题上报页或弹出底部 Sheet
  uni.navigateTo({
    url: `/pages/reportIssue/index?taskId=${this.task.id}`
  })
}
```

---

### 8. API 封装

```js
// src/api/seTask.js

// 获取 My Task 列表
export const getMyTaskList = (params) =>
  request.get('/api/se/process-tasks', { params })

// 获取工序任务详情
export const getProcessTaskDetail = (id) =>
  request.get(`/api/se/process-tasks/${id}`)

// Start Task
export const startProcessTask = (id) =>
  request.post(`/api/se/process-tasks/${id}/start`)

// Mark as Complete
export const completeProcessTask = (id) =>
  request.post(`/api/se/process-tasks/${id}/complete`)

// Report Issue
export const submitIssue = (data) =>
  request.post('/api/issues', data)
```

---

### 9. 按钮样式

```css
.btn-primary {
  background-color: #1890FF;
  color: #FFFFFF;
  border-radius: 4px;
  padding: 20rpx 40rpx;
}
.btn-disabled {
  background-color: #CCCCCC;
  color: #FFFFFF;
  pointer-events: none;
}
.btn-outline {
  background-color: transparent;
  color: #1890FF;
  border: 2rpx solid #1890FF;
  border-radius: 4px;
  padding: 20rpx 40rpx;
}
```

---

### 10. 验收条件

- [ ] 列表页 Tab 筛选：All / Not Started / In Progress / Completed 正确过滤
- [ ] 卡片状态标签颜色：灰 / 橙 / 绿 三态对应
- [ ] 详情页展示实际开始/结束时间（只读，未记录显示"—"）
- [ ] Start Task：仅 not_started 状态可点，点击后无弹框，直接变 in_progress，actualStart 更新
- [ ] Mark as Complete：仅 in_progress 状态可点，点击后弹 uni.showModal 确认，确认后变 completed，actualEnd 更新
- [ ] Completed 状态：Start Task + Mark as Complete 均 disabled
- [ ] Report Issue 在所有状态下可点击
