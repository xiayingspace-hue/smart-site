> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

# FRONTEND-REQ-012-pc.md

## CM"我的任务"PC 端前端说明

> **对应需求：REQ-012
> 平台：PC 端（Vue2 + Element UI）
> 影响模块：CM 我的任务列表、工序任务侧滑弹框、任务详情页 Tab

---

### 1. 路由与页面结构

```
/cm/my-tasks                    → CM 我的任务列表页
/cm/my-tasks/:taskId            → 任务详情页（Tab 结构）
```

组件目录建议：
```
src/views/cm/
  MyTaskList.vue                 # 列表页
  MyTaskDetail.vue               # 详情页
  components/
    ProcessTaskDrawer.vue        # 工序任务创建/编辑侧滑弹框（复用）
    IssueReportDialog.vue        # 问题汇报弹框
    tabs/
      TaskInfoTab.vue            # Tab1：任务信息
      ProcessTaskTab.vue         # Tab2：工序任务列表
      IssueListTab.vue           # Tab3：问题上报列表
```

---

### 2. 列表页（MyTaskList.vue）

#### 2.1 数据获取

```js
// 接口：GET /api/cm/tasks
// 参数：{ status, wbs, startDate, endDate, keyword, page, pageSize }
fetchTasks() {
  this.loading = true
  getCmTaskList(this.queryParams).then(res => {
    this.tableData = res.data.list
    this.total = res.data.total
  }).finally(() => { this.loading = false })
}
```

#### 2.2 操作列按钮控制

```js
// 新建工序任务：任务已完成或已作废时禁用
isCreateProcessDisabled(row) {
  return ['completed', 'cancelled'].includes(row.status)
},
// 汇报问题：任务已作废时禁用
isReportIssueDisabled(row) {
  return row.status === 'cancelled'
}
```

模板示例：
```html
<el-button
  type="text"
  :disabled="isCreateProcessDisabled(scope.row)"
  @click="openProcessDrawer(scope.row)">
  新建工序任务
</el-button>
<el-button
  type="text"
  :disabled="isReportIssueDisabled(scope.row)"
  @click="openIssueDialog(scope.row)">
  汇报问题
</el-button>
```

---

### 3. 工序任务侧滑弹框（ProcessTaskDrawer.vue）

#### 3.1 Props

```js
props: {
  visible: Boolean,
  taskId: String,          // 关联上级任务 ID
  taskName: String,        // 关联上级任务名称（只读展示）
  editData: Object         // 编辑模式传入现有工序数据，新建时为 null
}
```

#### 3.2 创建方式切换

```js
data() {
  return {
    createMode: 'template',  // 'template' | 'custom'
    selectedTemplates: [],
    processTaskList: []      // 批量编辑列表
  }
}
```

- 选择模板后点击"确认引用"，将模板数据填充到 `processTaskList`，可继续编辑
- 自定义模式直接在 `processTaskList` 中增删行

#### 3.3 权重字段校验

```js
rules: {
  weight: [
    { required: true, message: '请输入权重', trigger: 'blur' },
    { type: 'number', min: 1, max: 10, message: '权重范围 1～10', trigger: 'blur' }
  ]
}
```

#### 3.4 提交

```js
handleSubmit() {
  this.$refs.form.validate(valid => {
    if (!valid) return
    const api = this.editData ? updateProcessTask : createProcessTasks
    api(this.taskId, this.processTaskList).then(() => {
      this.$message.success('保存成功')
      this.$emit('success')
      this.$emit('update:visible', false)
    })
  })
}
```

---

### 4. 任务详情页 Tab 2：工序任务（ProcessTaskTab.vue）

#### 4.1 表格列定义

```js
columns: [
  { prop: 'name',           label: '工序名称' },
  { prop: 'weight',         label: '权重' },
  { prop: 'planStart',      label: '计划开始' },
  { prop: 'planEnd',        label: '计划结束' },
  { prop: 'actualStart',    label: '实际开始' },   // 只读，系统记录
  { prop: 'actualEnd',      label: '实际结束' },   // 只读，系统记录
  { prop: 'priority',       label: '优先级' },
  { prop: 'assignedSe',     label: 'SE' },
  { prop: 'status',         label: '状态' },
  { label: '操作' }
]
```

#### 4.2 状态标签渲染

```js
statusTagType(status) {
  const map = {
    not_started: 'info',    // 灰色
    in_progress: '',         // 蓝色（Element UI 默认）
    completed: 'success'     // 绿色
  }
  return map[status] || 'info'
},
statusLabel(status) {
  const map = {
    not_started: 'Not Started',
    in_progress: 'In Progress',
    completed: 'Completed'
  }
  return map[status] || status
}
```

```html
<el-tag :type="statusTagType(row.status)" size="small">
  {{ statusLabel(row.status) }}
</el-tag>
```

#### 4.3 Completed 行置灰

```js
// el-table 的 row-class-name 属性
rowClassName({ row }) {
  return row.status === 'completed' ? 'row-completed' : ''
}
```

```css
/* 全局或组件 scoped 样式 */
.el-table .row-completed td {
  background-color: #F5F5F5 !important;
  color: #AAAAAA !important;
}
```

#### 4.4 编辑按钮禁用（仅 Completed 行禁用）

```html
<el-button
  type="text"
  :disabled="row.status === 'completed'"
  @click="openEditDrawer(row)">
  编辑
</el-button>
```

#### 4.5 进度展示

```js
// 计算属性：Completed 权重之和 / 总权重 × 100%
computed: {
  taskProgress() {
    const total = this.processTasks.reduce((s, t) => s + t.weight, 0)
    const done = this.processTasks
      .filter(t => t.status === 'completed')
      .reduce((s, t) => s + t.weight, 0)
    return total > 0 ? Math.round((done / total) * 100) : 0
  }
}
```

```html
<div class="progress-bar">
  任务进度：{{ taskProgress }}%
  <el-progress :percentage="taskProgress" :show-text="false" />
</div>
```

---

### 5. 问题汇报弹框（IssueReportDialog.vue）

```js
handleSubmit() {
  this.$refs.form.validate(valid => {
    if (!valid) return
    submitIssue({ taskId: this.taskId, ...this.form }).then(() => {
      this.$message.success('问题已上报')
      this.$emit('update:visible', false)
    })
  })
}
```

---

### 6. 验收条件

- [ ] 操作列"新建工序任务"在任务已完成/已作废时 disabled
- [ ] 侧滑弹框权重字段校验 1～10，必填
- [ ] 工序任务 Tab 状态列展示 Not Started / In Progress / Completed 三态标签
- [ ] 工序任务 Tab 展示实际开始/实际结束列（只读，无数据显示"—"）
- [ ] Completed 行背景 `#F5F5F5`，文字 `#AAAAAA`，编辑按钮 disabled
- [ ] 进度计算公式正确（Completed 权重 / 总权重 × 100%）
