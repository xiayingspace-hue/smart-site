# FRONTEND-REQ-013-app.md

## SE"My Task"APP 端前端开发说明

> 对应需求：REQ-013
> 平台：APP 端（UNIAPP + Vue2）
> 关联 UI 文档：UI-REQ-013-app.md

---

### 1. 功能概述

为 SE 角色在 APP 端实现"My Task"工序任务管理功能，包含：
- 首页 Progress Management 模块角色动态入口
- 工序任务列表页（筛选 + 卡片列表）
- 工序任务详情页（只读信息 + 操作区）
- 标记完成确认弹框（含锁定提示）
- 问题上报表单页

---

### 2. 技术栈

| 项目 | 版本/说明 |
|------|----------|
| 框架 | UNIAPP + Vue 2.x |
| UI 组件库 | UNIAPP 内置组件 / uni-ui |
| 路由 | uni-app 页面路由 |
| HTTP 请求 | uni.request / 封装的 request 工具 |

---

### 3. 页面清单

| 页面文件 | 说明 |
|---------|------|
| `pages/my-task/index.vue` | My Task 工序任务列表页 |
| `pages/my-task/detail.vue` | 工序任务详情页 |
| `pages/my-task/issue-report.vue` | 问题上报表单页 |

---

### 4. 页面注册（pages.json）

```json
{
  "pages": [
    {
      "path": "pages/my-task/index",
      "style": { "navigationBarTitleText": "My Task" }
    },
    {
      "path": "pages/my-task/detail",
      "style": { "navigationBarTitleText": "工序任务详情" }
    },
    {
      "path": "pages/my-task/issue-report",
      "style": { "navigationBarTitleText": "问题上报" }
    }
  ]
}
```

---

### 5. 功能实现细节

#### 5.1 首页 Progress Management 模块

在首页组件中根据用户角色动态渲染入口：

```js
computed: {
  progressModuleItems() {
    const role = this.$store.getters.userRole
    if (role === 'SE') return [{ label: 'My Task', icon: 'task', path: '/pages/my-task/index' }]
    if (role === 'CM') return [{ label: 'CM Tasks', icon: 'cm-task', path: '/pages/cm/index' }]
    if (role === 'PM') return [{ label: 'PM Tasks', icon: 'pm-task', path: '/pages/pm/index' }]
    return []
  }
}
```

**角标（Badge）：**
- 读取未完成工序任务数量或未读消息数
- 调用接口：`GET /api/se/my-tasks/badge-count`，返回 `{ unfinished: 3, unread: 1 }`
- 有数量时在图标右上角显示红色数字角标，为 0 时隐藏

---

#### 5.2 工序任务列表页（index.vue）

**数据加载：**
```js
async loadTasks() {
  const { data } = await request.get('/api/se/my-tasks', {
    params: {
      status: this.filterStatus,    // '' | 'pending' | 'completed'
      priority: this.filterPriority,
      startDateFrom: this.dateRange[0],
      startDateTo: this.dateRange[1],
      page: this.page,
      pageSize: 20
    }
  })
  this.taskList = data.list
  this.total = data.total
}
```

**筛选区：**
- 状态 Tab：全部 / 未完成 / 已完成（切换时重新请求）
- 优先级下拉：High / Medium / Low / 全部
- 日期范围：`uni-datetime-picker`

**卡片渲染：**
```html
<view v-for="task in taskList" :key="task.id" class="task-card"
      @click="goToDetail(task.id)">
  <text class="task-name">{{ task.name }}</text>
  <text class="parent-task">上级：{{ task.parentTaskName }}</text>
  <text class="date">{{ task.startDate }} ~ {{ task.endDate }}</text>
  <view class="footer">
    <uni-tag :text="task.priority" :type="priorityType(task.priority)" />
    <uni-tag :text="task.status === 'completed' ? '已完成' : '未完成'"
             :type="task.status === 'completed' ? 'success' : 'default'" />
  </view>
</view>
```

**下拉刷新：**
```js
onPullDownRefresh() {
  this.page = 1
  this.loadTasks().finally(() => uni.stopPullDownRefresh())
}
```

---

#### 5.3 工序任务详情页（detail.vue）

**路由传参：**
```js
// 列表页跳转
uni.navigateTo({ url: `/pages/my-task/detail?taskId=${task.id}` })
// 详情页接收
onLoad(options) {
  this.taskId = options.taskId
  this.loadDetail()
}
```

**标记完成按钮控制：**
```html
<button :disabled="task.status === 'completed'"
        :class="{ 'btn-disabled': task.status === 'completed' }"
        @click="handleMarkComplete">
  {{ task.status === 'completed' ? '已完成' : '标记完成' }}
</button>
```

---

#### 5.4 标记完成确认逻辑

```js
handleMarkComplete() {
  uni.showModal({
    title: '确认标记完成？',
    content: '标记完成后将锁定该工序，如需修改请联系 CM，确认继续？',
    confirmText: '确认完成',
    cancelText: '取消',
    success: async (res) => {
      if (res.confirm) {
        try {
          await request.post(`/api/se/process-tasks/${this.taskId}/complete`)
          this.task.status = 'completed'
          uni.showToast({ title: '已标记完成', icon: 'success' })
        } catch (e) {
          uni.showToast({ title: '操作失败，请重试', icon: 'none' })
        }
      }
    }
  })
}
```

---

#### 5.5 问题上报表单（issue-report.vue）

**路由传参：**
```js
// 详情页点击"问题上报"
uni.navigateTo({ url: `/pages/my-task/issue-report?taskId=${this.taskId}&taskName=${this.task.name}` })
```

**表单字段：**
```js
data() {
  return {
    form: {
      taskId: '',
      issueType: '',
      description: '',
      impact: '',
      suggestion: '',
      attachments: []
    }
  }
}
```

**附件上传（`uni.chooseImage`）：**
```js
async handleChooseFile() {
  const { tempFilePaths } = await uni.chooseImage({ count: 5 })
  // 调用上传接口，获取 URL 后追加到 attachments
}
```

**提交：**
```js
async handleSubmit() {
  await request.post('/api/issue-reports', this.form)
  uni.showToast({ title: '上报成功，已通知 CM', icon: 'success' })
  setTimeout(() => uni.navigateBack(), 1500)
}
```

---

### 6. API 接口调用清单

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/se/my-tasks/badge-count` | GET | 获取未完成数/未读数（首页角标） |
| `/api/se/my-tasks` | GET | 获取 SE 工序任务列表（支持筛选/分页） |
| `/api/se/process-tasks/:id` | GET | 获取工序任务详情 |
| `/api/se/process-tasks/:id/complete` | POST | 标记工序任务完成 |
| `/api/issue-reports` | POST | 提交问题上报 |

---

### 7. 权限控制

- My Task 入口仅对 SE 角色显示（首页根据角色动态渲染）
- 标记完成、问题上报接口服务端同样需校验角色为 SE，且只能操作分配给自己的工序任务

---

### 8. 验收条件

- [ ] SE 登录后首页 Progress Management 显示 My Task 入口，CM/PM 不显示
- [ ] My Task 入口显示未完成数量角标
- [ ] 工序任务列表正确加载，卡片字段显示完整
- [ ] 状态 Tab 筛选切换正常，下拉刷新有效
- [ ] 点击卡片进入详情页，所有字段只读
- [ ] 未完成工序标记完成按钮可点，已完成置灰
- [ ] 点击标记完成弹出确认框，确认后调接口成功，按钮置灰，Toast 提示
- [ ] 问题上报表单提交后返回详情页，Toast 成功提示
