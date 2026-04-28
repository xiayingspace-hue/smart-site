# FRONTEND-REQ-012-pc.md

## CM"我的任务"PC 端前端开发说明

> 对应需求：REQ-012
> 平台：PC 端（Vue2 + Element UI）
> 关联 UI 文档：UI-REQ-012-pc.md

---

### 1. 功能概述

为 CM 角色实现专属的"我的任务（CM）"菜单页面，包含：
- 任务列表页（筛选 + 表格 + 操作列）
- 工序任务创建/编辑侧滑弹框（Drawer）
- 任务详情页（Tab 结构：任务信息 / 工序任务 / 问题上报）
- 问题汇报弹框（Dialog）

---

### 2. 技术栈

| 项目 | 版本/说明 |
|------|----------|
| 框架 | Vue 2.x |
| UI 组件库 | Element UI 2.x |
| 路由 | Vue Router |
| 状态管理 | Vuex（可选，与现有方案一致） |
| HTTP 请求 | Axios |

---

### 3. 页面与组件清单

| 组件/页面文件 | 说明 |
|--------------|------|
| `views/cm/MyTask.vue` | CM 我的任务列表页 |
| `views/cm/TaskDetail.vue` | 任务详情页（Tab 结构） |
| `components/cm/ProcessTaskDrawer.vue` | 工序任务创建/编辑侧滑弹框（复用） |
| `components/cm/IssueReportDialog.vue` | 问题汇报弹框 |

---

### 4. 路由配置

```js
// 新增路由（仅 CM 角色可访问）
{
  path: '/progress/cm/my-task',
  name: 'CMMyTask',
  component: () => import('@/views/cm/MyTask.vue'),
  meta: { roles: ['CM'] }
},
{
  path: '/progress/cm/task/:taskId',
  name: 'CMTaskDetail',
  component: () => import('@/views/cm/TaskDetail.vue'),
  meta: { roles: ['CM'] }
}
```

---

### 5. 功能实现细节

#### 5.1 任务列表页（MyTask.vue）

**筛选区：**
- 任务状态多选下拉（`el-select multiple`）
- WBS 输入搜索
- 计划时间范围选择（`el-date-picker type="daterange"`）
- 搜索 / 重置按钮
- 筛选条件变更后自动触发接口请求

**表格（`el-table`）：**
- 字段：Activity ID、Activity Name（可点击跳转详情）、WBS、计划时间、实际时间、Deviation、Status、优先级、进度、操作
- Deviation 超期（> 0）时单元格文字颜色显示红色
- 进度列展示 `el-progress`（`type="line"` `show-text`）
- 操作列：`el-button type="text"` "新建工序任务" | "汇报问题"
- 分页：`el-pagination`，默认每页 20 条

**事件处理：**
```js
// 点击新建工序任务
handleNewProcessTask(row) {
  this.currentTask = row
  this.drawerVisible = true
  this.drawerMode = 'create'
},
// 点击汇报问题
handleReportIssue(row) {
  this.currentTask = row
  this.issueDialogVisible = true
}
```

---

#### 5.2 工序任务创建/编辑侧滑弹框（ProcessTaskDrawer.vue）

**Props：**
```js
props: {
  visible: Boolean,
  mode: { type: String, default: 'create' }, // 'create' | 'edit'
  parentTask: Object,   // 关联上级任务信息
  editData: Object      // 编辑模式时传入的工序任务数据
}
```

**创建模式切换（引用模板 / 自定义）：**
```js
data() {
  return {
    createMode: 'template', // 'template' | 'custom'
    templateList: [],
    selectedTemplates: [],
    processTaskList: []   // 工序任务行列表（批量编辑）
  }
},
methods: {
  // 选择模板后生成工序任务行
  handleTemplateConfirm() {
    const rows = this.selectedTemplates.map(tpl => ({
      name: tpl.name,
      wbs: '',
      startDate: null,
      endDate: null,
      priority: 'Medium',
      weight: tpl.weight,   // 自动带出模板权重
      weightFromTemplate: true,
      assignedSE: null,
      remark: ''
    }))
    this.processTaskList.push(...rows)
  }
}
```

**权重字段校验：**
```js
// 表单校验规则
rules: {
  weight: [
    { required: true, message: '权重为必填项', trigger: 'blur' },
    { type: 'number', min: 1, max: 10, message: '权重范围为 1～10', trigger: 'blur' }
  ]
}
```

**保存逻辑：**
- 创建模式：调用 `POST /api/process-tasks/batch` 批量创建
- 编辑模式：调用 `PUT /api/process-tasks/:id` 更新单条

---

#### 5.3 任务详情页（TaskDetail.vue）

**Tab 组件：**
```html
<el-tabs v-model="activeTab">
  <el-tab-pane label="任务信息" name="info">
    <TaskInfoPanel :task="taskDetail" />
  </el-tab-pane>
  <el-tab-pane label="工序任务" name="process">
    <ProcessTaskPanel :taskId="taskId" />
  </el-tab-pane>
  <el-tab-pane label="问题上报" name="issues">
    <IssueListPanel :taskId="taskId" />
  </el-tab-pane>
</el-tabs>
```

**工序任务 Tab（ProcessTaskPanel）：**

```js
// 已完成行置灰
rowClassName({ row }) {
  return row.status === 'completed' ? 'row-completed-locked' : ''
}
```

```css
/* 样式 */
.row-completed-locked {
  background-color: #F5F5F5;
  color: #AAAAAA;
}
.row-completed-locked .el-button {
  color: #CCCCCC;
  cursor: not-allowed;
}
```

**进度计算展示（只读）：**
```js
computed: {
  taskProgress() {
    const total = this.processTasks.reduce((sum, t) => sum + t.weight, 0)
    const completed = this.processTasks
      .filter(t => t.status === 'completed')
      .reduce((sum, t) => sum + t.weight, 0)
    return total > 0 ? Math.round((completed / total) * 100) : 0
  }
}
```

---

#### 5.4 问题汇报弹框（IssueReportDialog.vue）

**Props：**
```js
props: {
  visible: Boolean,
  task: Object
}
```

**表单字段：** 问题类型（下拉必填）、问题描述（文本域必填）、影响范围（可选）、建议方案（可选）、附件（`el-upload`，可选）

**提交：** 调用 `POST /api/issue-reports`，成功后 `$emit('close')`

---

### 6. API 接口调用清单

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/cm/tasks` | GET | 获取 CM 任务列表（支持筛选/分页参数） |
| `/api/cm/tasks/:id` | GET | 获取任务详情 |
| `/api/process-task-templates` | GET | 获取工序模板列表（支持筛选） |
| `/api/process-tasks/batch` | POST | 批量创建工序任务 |
| `/api/process-tasks/:id` | PUT | 更新工序任务 |
| `/api/process-tasks?taskId=:id` | GET | 获取某任务下的工序任务列表 |
| `/api/issue-reports` | POST | 提交问题汇报 |
| `/api/issue-reports?taskId=:id` | GET | 获取某任务下的问题汇报列表 |

---

### 7. 权限控制

- 菜单"我的任务（CM）"仅对角色为 CM 的用户显示（路由 meta.roles 控制）
- 操作列按钮在前端根据 `userRole === 'CM'` 控制显示

---

### 8. 验收条件

- [ ] CM 登录后左侧菜单显示"我的任务（CM）"，其他角色不显示
- [ ] 任务列表页筛选、分页、排序正常，Deviation 超期红色展示
- [ ] 点击"新建工序任务"弹出侧滑弹框，引用模板自动带出权重，自定义时权重必填且范围 1～10
- [ ] 批量保存工序任务后列表自动刷新，任务进度重新计算展示
- [ ] 任务详情页 Tab 切换正常，工序任务 Tab 中已完成行置灰且编辑按钮 disabled
- [ ] 工序任务 Tab 底部进度公式计算正确（已完成权重/全部权重×100%）
- [ ] 问题汇报弹框提交后 Toast 提示，Dialog 关闭
