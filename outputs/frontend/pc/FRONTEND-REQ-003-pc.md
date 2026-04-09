# 前端开发说明文档 — PC 管理端工程图纸管理与审批

> **来源需求**: [REQ-003-pc.md](../../../requirements/pc/REQ-003-pc.md) + [REQ-003-shared.md](../../../requirements/shared/REQ-003-shared.md)
> **UI 设计参考**: [UI-REQ-003-pc.md](../../ui/pc/UI-REQ-003-pc.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-04-07

---

## 1. 功能概述

本模块实现 PC 管理端的工程图纸管理功能，涵盖：

| 功能模块 | 说明 |
|---------|------|
| 图纸列表页 | 展示项目下所有图纸，支持按 Code / Name / Category / Status 搜索，含分页 |
| 上传图纸弹窗 | 新建图纸及上传第一版文件，支持拖拽上传、进度展示 |
| 上传新版本弹窗 | 为已有图纸追加新版本 |
| 分配 Site Engineer 弹窗 | 管理员为图纸指定可查阅的 Site Engineer（全量覆盖提交） |
| 审批 Todo 面板 | 顶部通知铃铛展开，显示待审批图纸卡片，支持通过 / 驳回操作 |
| 版本历史抽屉 | 查看某图纸所有历史版本及各版本状态 |
| 查阅确认记录抽屉 | 查看某图纸当前版本的 Site Engineer 已确认 / 未确认列表（仅统计已分配人员） |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 + Vue Router + Vuex |
| UI 组件库 | Element UI 2.x |
| HTTP | Axios，统一封装 `request.js`，自动附带 `Authorization` / `X-Tenant-Id` / `Project-Id` / `lang` Header |
| 文件上传 | `el-upload` 组件 + 自定义 `http-request`，使用分片或直传 OSS |
| 状态管理 | Vuex module，模块名 `drawing` |
| 路由 | `/project/:projectId/drawings` |
| 国际化 | Vue I18n，key 命名见第 7 节 |
| 权限指令 | `v-permission="'drawing:upload'"` 等，与现有权限体系保持一致 |
| 代码风格 | ESLint + Prettier，与项目现有配置一致 |

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── drawing/
│       ├── index.vue                  # 图纸列表页（主页面）
│       ├── components/
│       │   ├── SearchPanel.vue        # 可折叠搜索面板
│       │   ├── DrawingTable.vue       # 图纸列表表格
│       │   ├── UploadDialog.vue       # 上传图纸弹窗（新建 + 新版本复用）
│       │   ├── AssignDialog.vue       # 分配 Site Engineer 弹窗
│       │   ├── HistoryDrawer.vue      # 版本历史抽屉
│       │   └── ConfirmDrawer.vue      # 查阅确认记录抽屉
│       └── utils/
│           └── drawing-helper.js     # 状态枚举映射、格式化工具
├── store/
│   └── modules/
│       └── drawing.js                # Vuex module
├── api/
│   └── drawing.js                    # 所有图纸相关 API 封装
└── layout/
    └── components/
        └── TodoPanel.vue             # 顶部通知 / Todo 面板（复用 / 新增）
```

---

## 4. 页面与组件详细说明

### 4.1 图纸列表页 `index.vue`

#### 4.1.1 页面结构

```
<template>
  顶部操作栏（SearchToggleBtn + UploadBtn）
  SearchPanel（折叠/展开）
  DrawingTable（表格 + 分页）
  UploadDialog
  AssignDialog
  HistoryDrawer
  ConfirmDrawer
</template>
```

#### 4.1.2 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `searchVisible` | Boolean | 搜索面板展开/收起状态，默认 `false` |
| `searchForm` | Object | 搜索条件：`{ code, name, category, status }` |
| `activeSearchForm` | Object | 当前生效的搜索条件（点击 Search 后同步） |
| `tableData` | Array | 图纸列表数据 |
| `total` | Number | 总记录数 |
| `pageNo` | Number | 当前页码，默认 1 |
| `pageSize` | Number | 每页条数，默认 10 |
| `loading` | Boolean | 表格 loading |
| `uploadVisible` | Boolean | 上传弹窗显示 |
| `uploadMode` | String | `'new'`（新建）/ `'version'`（新版本） |
| `uploadTargetDrawing` | Object \| null | 上传新版本时，当前图纸对象 |
| `assignVisible` | Boolean | 分配弹窗显示 |
| `assignTargetDrawing` | Object \| null | 当前分配操作的图纸对象 |
| `historyVisible` | Boolean | 版本历史抽屉显示 |
| `historyTargetDrawing` | Object \| null | 查看历史的图纸对象 |
| `confirmVisible` | Boolean | 确认记录抽屉显示 |
| `confirmTargetDrawing` | Object \| null | 查看确认记录的图纸对象 |

#### 4.1.3 核心方法

| 方法 | 说明 |
|------|------|
| `fetchList()` | 调用 `/drawing/page` 接口，使用 `activeSearchForm` + 分页参数；默认按 `updatedAt` 降序 |
| `toggleSearch()` | 切换 `searchVisible`；若收起同时调用 `handleCancel()` |
| `handleSearch()` | 将 `searchForm` 同步到 `activeSearchForm`，重置页码为 1，调用 `fetchList()` |
| `handleCancel()` | 清空 `searchForm` 和 `activeSearchForm`，`searchVisible = false`，重置页码，调用 `fetchList()` |
| `handleUploadNew()` | `uploadMode = 'new'`，`uploadTargetDrawing = null`，`uploadVisible = true` |
| `handleUploadVersion(row)` | `uploadMode = 'version'`，`uploadTargetDrawing = row`，`uploadVisible = true` |
| `handleAssign(row)` | `assignTargetDrawing = row`，`assignVisible = true` |
| `handleHistory(row)` | `historyTargetDrawing = row`，`historyVisible = true` |
| `handleConfirm(row)` | `confirmTargetDrawing = row`，`confirmVisible = true` |
| `handleUploadSuccess()` | 关闭上传弹窗，调用 `fetchList()`，显示 Toast（含 Assign 快捷链接） |

---

### 4.2 搜索面板 `SearchPanel.vue`

#### 4.2.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 是否展开，父组件控制 |
| `value` | Object | 双向绑定搜索条件 `v-model` |

#### 4.2.2 Emits

| 事件 | 参数 | 说明 |
|------|------|------|
| `search` | — | 用户点击面板内 [Search] 按钮 |
| `cancel` | — | 用户点击面板内 [Cancel] 按钮 |

#### 4.2.3 布局说明

- 使用 `v-show` 控制展开/收起，保留表单状态
- 两行两列布局：第一行 Code（el-input） + Name（el-input），第二行 Category（el-select） + Status（el-select）
- 面板底部按钮行：[Cancel]（el-button）/ [Search]（el-button type="primary"），右对齐
- Category 选项从常量文件获取，Status 选项：All / Active / Pending / Rejected

---

### 4.3 图纸表格 `DrawingTable.vue`

#### 4.3.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `data` | Array | 表格数据 |
| `total` | Number | 总条数 |
| `loading` | Boolean | 加载状态 |
| `pageNo` | Number | 当前页 |
| `pageSize` | Number | 每页条数 |

#### 4.3.2 Emits

| 事件 | 参数 | 说明 |
|------|------|------|
| `page-change` | `{ pageNo, pageSize }` | 分页变化 |
| `upload-version` | `row` | 点击上传新版本 |
| `assign` | `row` | 点击分配 |
| `history` | `row` | 点击版本历史 |
| `confirms` | `row` | 点击确认记录 |

#### 4.3.3 列定义

| 列 | 字段 | 宽度 | 备注 |
|----|------|------|------|
| # | 序号 | 60px | `el-table-column type="index"` |
| Code | `drawingCode` | 120px | 主色链接样式，点击 `handleView(row)` 预览 |
| Name | `drawingName` | 180px | `show-overflow-tooltip` |
| Category | `drawingCategory` | 130px | 显示枚举映射后的中英文 |
| Ver | `currentVersion` | 70px | — |
| Status | `status` | 110px | 自定义 `StatusTag` 组件 |
| Confirms | `confirmedCount / totalSiteEngineers` | 100px | Active 时显示 `x/y`，其他状态显示 `—` |
| Updated | `updateTime` | 130px | 格式化为 `Apr 1, 2026` |
| Actions | — | 260px | 见 §4.3.4 |

#### 4.3.4 操作列按钮显示逻辑

```javascript
// View — 所有行显示
// History — 所有行显示
// Confirms — 所有行显示（链接打开确认记录抽屉）
// Assign — 所有行显示，需 v-permission="'drawing:assign'"
// Upload V{n+1} — 仅 status === 'ACTIVE' 时显示，需 v-permission="'drawing:upload'"
//   按钮文字动态拼接: `Upload V${nextVersion(row.currentVersion)}`

function nextVersion(versionStr) {
  // 'V3' => 'V4'
  const num = parseInt(versionStr.replace('V', ''), 10)
  return `V${num + 1}`
}
```

> Pending 状态下 [Upload New Version] 按钮不展示（因为 status !== 'ACTIVE'）。操作列宽度不足时，History / Confirms / Assign 三个次要操作折叠进 `el-dropdown` 更多按钮。

---

### 4.4 上传图纸弹窗 `UploadDialog.vue`

#### 4.4.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 弹窗显示 |
| `mode` | String | `'new'` \| `'version'` |
| `drawing` | Object \| null | `mode='version'` 时传入当前图纸对象 |

#### 4.4.2 Emits

| 事件 | 参数 | 说明 |
|------|------|------|
| `success` | `responseData` | 上传成功 |
| `close` | — | 弹窗关闭 |

#### 4.4.3 表单字段

| 字段 | 组件 | `mode='new'` | `mode='version'` | 校验 |
|------|------|:---:|:---:|------|
| Drawing Code | el-input | ✅ 可编辑 | 只读展示 | 必填；blur 时调 `/drawing/check-code` 唯一性校验 |
| Drawing Name | el-input | ✅ 可编辑 | 只读展示 | 必填，≤100 字符 |
| Category | el-select | ✅ 可编辑 | 只读展示 | 必填 |
| Description | el-input textarea | ✅ 可编辑 | 可编辑 | 可选，≤500 字符 |
| 版本提示 | 只读信息块 | ❌ | ✅ 显示当前版本和下一版本 | — |
| Drawing File | 自定义上传区 | ✅ | ✅ | 必填；格式：PDF/PNG/JPG；大小 ≤50MB |
| Version Note | el-input | ✅ 可选 | ✅ **必填** | `mode='version'` 时必填 |
| Approver | el-select filterable | ✅ | ✅ | 必填；从 `/member/list?permission=drawing:approve` 拉取 |

#### 4.4.4 文件上传区实现要点

```javascript
// 使用 el-upload，action="#"，auto-upload=false，手动触发
// before-upload 钩子做格式和大小前置校验

function beforeUpload(file) {
  const allowedTypes = ['application/pdf', 'image/png', 'image/jpeg']
  const maxSize = 50 * 1024 * 1024 // 50MB
  if (!allowedTypes.includes(file.type)) {
    this.$message.error(this.$t('validation_file_format'))
    return false
  }
  if (file.size > maxSize) {
    this.$message.error(this.$t('validation_file_size'))
    return false
  }
  return true
}

// submit 时构造 FormData，调用 uploadDrawing() API
// 使用 axios onUploadProgress 回调驱动 el-progress 百分比
```

#### 4.4.5 关闭弹窗确认

表单有任意字段填写或文件已选时，点击 ✕ / Cancel / 遮罩触发二次确认：

```javascript
// el-dialog 设置 :before-close="handleBeforeClose"
function handleBeforeClose(done) {
  if (this.isDirty) {
    this.$confirm(this.$t('confirm_discard_changes'), { ... })
      .then(() => { this.resetForm(); done() })
      .catch(() => {})
  } else {
    done()
  }
}
```

#### 4.4.6 提交成功后处理

```javascript
async handleSubmit() {
  await this.$refs.form.validate()
  this.submitting = true
  try {
    const formData = buildFormData(this.form, this.fileObj)
    const res = await uploadDrawing(formData)
    this.$emit('success', res.data)
    // 显示带 Assign 快捷入口的 Toast（el-notification，6s）
    this.$notify({
      title: this.$t('toast_upload_success'),
      message: h('span', [
        this.$t('toast_upload_assign_prefix'),
        h('a', { on: { click: () => this.$emit('open-assign', res.data.drawingId) } },
          this.$t('toast_upload_assign_link')),
        this.$t('toast_upload_assign_suffix')
      ]),
      duration: 6000
    })
  } finally {
    this.submitting = false
  }
}
```

---

### 4.5 分配 Site Engineer 弹窗 `AssignDialog.vue`

#### 4.5.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 弹窗显示 |
| `drawing` | Object | 当前图纸对象（含 `drawingId`, `drawingCode`, `drawingName`） |

#### 4.5.2 Emits

| 事件 | 参数 | 说明 |
|------|------|------|
| `saved` | — | 分配保存成功 |
| `close` | — | 弹窗关闭 |

#### 4.5.3 内部状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `assignedList` | Array | 已分配人员列表（弹窗内暂存，未提交） |
| `availableList` | Array | 未分配人员列表（弹窗内暂存） |
| `originalAssignedIds` | Array | 打开弹窗时的原始已分配 ID 列表，用于判断是否有移除操作 |
| `saving` | Boolean | 提交 loading |
| `hasRemoval` | Computed | `originalAssignedIds` 中有人员不在当前 `assignedList` 中 → 显示警示 |

#### 4.5.4 数据加载

```javascript
// watch visible，打开时调用 fetchAssignList(drawingId)
// GET /drawing/assign/list?drawingId=xxx
// 响应的 assignedList 和 unassignedList 分别存入本地状态
// 记录 originalAssignedIds = assignedList.map(i => i.assigneeId)
```

#### 4.5.5 Add / Remove 操作

```javascript
// 仅修改本地状态，不触发接口
function handleAdd(user) {
  this.availableList = this.availableList.filter(u => u.userId !== user.userId)
  this.assignedList.push({ assigneeId: user.userId, assigneeName: user.userName })
}

function handleRemove(user) {
  this.assignedList = this.assignedList.filter(u => u.assigneeId !== user.assigneeId)
  this.availableList.unshift({ userId: user.assigneeId, userName: user.assigneeName })
}
```

#### 4.5.6 Save 提交

```javascript
async handleSave() {
  // 如果 assignedList 为空，先弹二次确认
  if (this.assignedList.length === 0) {
    await this.$confirm(this.$t('assign_empty_confirm_body'), this.$t('assign_empty_confirm_title'))
  }
  this.saving = true
  try {
    await assignDrawing({
      drawingId: this.drawing.drawingId,
      assigneeIds: this.assignedList.map(u => u.assigneeId)
    })
    this.$message.success(this.$t('toast_assign_saved'))
    this.$emit('saved')
    this.handleClose()
  } finally {
    this.saving = false
  }
}
```

#### 4.5.7 关闭确认

有变更（`assignedList` 与 `originalAssignedIds` 不同）时，点击 Cancel / ✕ / 遮罩弹二次确认，确认后关闭并重置状态。

---

### 4.6 审批 Todo 面板 `TodoPanel.vue`

> 该组件位于顶部导航布局中，通过事件总线或 Vuex 驱动显示/隐藏。

#### 4.6.1 数据加载

```javascript
// 挂载或铃铛点击时调用 GET /todo/list?type=DRAWING_APPROVAL
// 同时拉取消息列表 GET /notification/list
// 铃铛角标 = pendingTodos.length + unreadMessages.length
```

#### 4.6.2 图纸审批 Todo 卡片操作

**Approve 流程**：

```javascript
async handleApprove(todo) {
  await this.$confirm(
    this.$t('approve_confirm_body'),
    this.$t('approve_confirm_title'),
    { confirmButtonText: this.$t('btn_approve'), type: 'warning' }
  )
  await approveDrawing({ approvalId: todo.relatedId, action: 'APPROVED', comment: '' })
  this.$message.success(this.$t('toast_approved'))
  this.fetchTodoList() // 刷新 todo 列表
  this.$emit('drawing-approved') // 通知列表页刷新
}
```

**Reject 流程**：

```javascript
// 点击 [Reject] 打开驳回意见 el-dialog（宽 400px）
// 输入框 el-input textarea，字数限制 500 字，右下角计数
// Comment 为空时 [Confirm Rejection] 按钮 disabled
async handleReject(todo) {
  await rejectDrawing({ approvalId: todo.relatedId, action: 'REJECTED', comment: this.rejectComment })
  this.$message.success(this.$t('toast_rejected'))
  this.rejectDialogVisible = false
  this.rejectComment = ''
  this.fetchTodoList()
  this.$emit('drawing-rejected')
}
```

---

### 4.7 版本历史抽屉 `HistoryDrawer.vue`

#### 4.7.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 抽屉显示 |
| `drawing` | Object | 图纸对象（含 `drawingId`, `drawingCode`, `drawingName`, `drawingCategory`） |

#### 4.7.2 数据加载

```javascript
// watch visible，打开时调用 GET /drawing/version/list?drawingId=xxx
// 响应数据按 versionNo 降序排列（V3 → V2 → V1）
```

#### 4.7.3 版本状态映射

```javascript
function getVersionStatus(version) {
  if (version.isCurrent) return 'current'
  if (version.approvalStatus === 'PENDING') return 'pending'
  if (version.approvalStatus === 'REJECTED') return 'rejected'
  if (version.isDeprecated) return 'deprecated'
  return 'deprecated'
}

const statusConfig = {
  current:    { label: 'Current',    color: '#2E7D32', bg: '#F0FAF0' },
  deprecated: { label: 'Deprecated', color: '#9E9E9E', bg: '#F5F5F5' },
  pending:    { label: 'Pending',    color: '#F57F17', bg: '#FFFDE7' },
  rejected:   { label: 'Rejected',   color: '#C62828', bg: '#FFF5F5' }
}
```

#### 4.7.4 View File

点击 [View] 使用 `window.open(version.fileUrl, '_blank')` 或在同页 PDF 预览弹窗中展示。

---

### 4.8 查阅确认记录抽屉 `ConfirmDrawer.vue`

#### 4.8.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 抽屉显示 |
| `drawing` | Object | 图纸对象（含 `drawingId`, `drawingCode`, `currentVersionId`, `currentVersion`） |

#### 4.8.2 数据加载

```javascript
// watch visible，打开时调用 GET /drawing/confirm/list?drawingVersionId=xxx
// 响应数据分入 confirmedList 和 unconfirmedList（Pending Tab）
// 副标题：`${confirmedCount} Confirmed / ${totalSiteEngineers} Assigned`
// 注意：totalSiteEngineers 仅为已分配该图纸的人数，非项目全员
```

#### 4.8.3 Tab 切换

```javascript
// el-tabs，两个 el-tab-pane
// Tab "Confirmed ({confirmedCount})" — 展示 confirmedList
// Tab "Pending ({unconfirmedCount})" — 展示 unconfirmedList，人员卡片右侧显示橙色 Pending 标签
```

---

## 5. API 封装（`api/drawing.js`）

```javascript
import request from '@/utils/request'

/** 上传图纸（新建 / 新版本） */
export function uploadDrawing(formData) {
  return request({ url: '/drawing/upload', method: 'post', data: formData,
    headers: { 'Content-Type': 'multipart/form-data' },
    onUploadProgress: formData._onProgress // 外部注入进度回调
  })
}

/** 校验图纸编号唯一性 */
export function checkDrawingCode(params) {
  return request({ url: '/drawing/check-code', method: 'get', params })
}

/** 获取图纸分页列表 */
export function getDrawingPage(params) {
  return request({ url: '/drawing/page', method: 'get', params })
}

/** 获取图纸版本历史 */
export function getVersionList(params) {
  return request({ url: '/drawing/version/list', method: 'get', params })
}

/** 审批图纸（通过 / 驳回） */
export function approveDrawing(data) {
  return request({ url: '/drawing/approve', method: 'post', data })
}

/** 获取查阅确认记录 */
export function getConfirmList(params) {
  return request({ url: '/drawing/confirm/list', method: 'get', params })
}

/** 获取图纸分配列表（已分配 + 未分配） */
export function getAssignList(params) {
  return request({ url: '/drawing/assign/list', method: 'get', params })
}

/** 保存图纸分配（全量覆盖） */
export function assignDrawing(data) {
  return request({ url: '/drawing/assign', method: 'post', data })
}

/** 获取可选审批人列表 */
export function getApproverList(params) {
  return request({ url: '/member/list', method: 'get', params })
}

/** 获取 Todo 列表 */
export function getTodoList(params) {
  return request({ url: '/todo/list', method: 'get', params })
}
```

---

## 6. Vuex Module（`store/modules/drawing.js`）

```javascript
// 轻量化设计，主要用于跨组件共享以下数据
const state = {
  todoPendingCount: 0,      // 待审批数量，驱动铃铛角标
  listNeedsRefresh: false   // 其他操作完成后通知列表刷新
}

const mutations = {
  SET_TODO_COUNT(state, count) { state.todoPendingCount = count },
  TRIGGER_REFRESH(state) { state.listNeedsRefresh = !state.listNeedsRefresh }
}

const actions = {
  async fetchTodoCount({ commit }) {
    const res = await getTodoList({ type: 'DRAWING_APPROVAL' })
    commit('SET_TODO_COUNT', res.data?.length ?? 0)
  }
}
```

> 图纸列表的搜索条件、分页状态等页面级数据建议保留在 `index.vue` 组件内，避免 Vuex 过度膨胀。

---

## 7. 权限控制

在模板中使用权限指令隐藏无权元素（后端接口同样做权限校验）：

```html
<!-- 上传入口 -->
<el-button v-permission="'drawing:upload'" @click="handleUploadNew">+ Upload Drawing</el-button>

<!-- 上传新版本按钮（表格操作列） -->
<el-button v-if="row.status === 'ACTIVE'" v-permission="'drawing:upload'"
  @click="handleUploadVersion(row)">Upload V{{ nextVersion(row.currentVersion) }}</el-button>

<!-- 分配入口 -->
<el-button v-permission="'drawing:assign'" @click="handleAssign(row)">Assign</el-button>
```

权限点汇总：

| 权限 Key | 控制的功能 |
|---------|---------|
| `drawing:view` | 图纸列表可见、View 按钮、History 按钮 |
| `drawing:upload` | Upload Drawing 按钮、Upload V{n} 按钮 |
| `drawing:approve` | Todo 面板审批操作（Approve / Reject） |
| `drawing:history` | History 按钮（版本历史抽屉） |
| `drawing:confirm:view` | Confirms 按钮（查阅确认记录抽屉） |
| `drawing:assign` | Assign 按钮（分配弹窗） |

---

## 8. 错误处理

| 场景 | 处理方式 |
|------|---------|
| 上传文件格式错误 | `before-upload` 钩子拦截，`this.$message.error(...)` 提示 |
| 上传文件超过 50MB | `before-upload` 钩子拦截，`this.$message.error(...)` 提示 |
| Drawing Code 已存在 | blur 后调接口，失败时表单项显示 inline 错误文字 |
| 该图纸已有版本待审批 | 上传接口返回 `1003003004`，捕获后弹 `this.$alert(...)` 提示 |
| 审批时 Comment 为空 | 前端 `disabled` 禁止提交，无需接口兜底 |
| 接口通用错误（4xx/5xx） | Axios 拦截器统一处理，`this.$message.error(res.msg)` |
| 网络超时 | Axios 超时设置 10s，超时后提示"Network error, please retry" |

---

## 9. 多语言 Key 索引

> 完整文案见 [UI-REQ-003-pc.md §4](../../ui/pc/UI-REQ-003-pc.md)。

| Key | 使用位置 |
|-----|---------|
| `page_title` | 页面标题 / 面包屑 |
| `btn_search_toggle` | 顶部 Search 切换按钮 |
| `filter_code_placeholder` | Code 搜索框 placeholder |
| `filter_name_placeholder` | Name 搜索框 placeholder |
| `btn_search` | 面板内 Search 按钮 |
| `btn_cancel` | 面板内 Cancel 按钮 |
| `btn_upload_drawing` | Upload Drawing 按钮 |
| `action_assign` | 表格 Assign 按钮 |
| `action_upload_version` | 表格 Upload V{n} 按钮 |
| `dialog_title_new` | 上传弹窗（新建）标题 |
| `dialog_title_version` | 上传弹窗（新版本）标题 |
| `validation_file_format` | 文件格式错误提示 |
| `validation_file_size` | 文件大小错误提示 |
| `toast_upload_success` | 上传成功 Toast |
| `toast_upload_assign_hint` | 上传成功后 Assign 引导 Toast |
| `approve_confirm_title` | 审批通过确认框标题 |
| `approve_confirm_body` | 审批通过确认框内容 |
| `reject_dialog_title` | 驳回弹窗标题 |
| `assign_dialog_title` | 分配弹窗标题 |
| `assign_warning` | 分配弹窗警示文案 |
| `assign_empty_confirm_title` | 空分配二次确认标题 |
| `assign_empty_confirm_body` | 空分配二次确认内容 |
| `toast_assign_saved` | 分配保存成功提示 |

---

## 10. 验收条件

### 10.1 图纸列表页

- [ ] 默认按 `updatedAt` 降序加载，分页正常翻页，每页条数可选
- [ ] 点击 [Search] 按钮，搜索面板展开；再次点击或点击面板内 [Cancel]，面板收起
- [ ] 面板内 [Cancel] 清空所有搜索条件，列表恢复无筛选状态
- [ ] Code / Name / Category / Status 可单独或组合筛选，结果正确
- [ ] Status 状态标签颜色：Active 绿 / Pending 黄 / Rejected 红
- [ ] `confirmedCount / totalSiteEngineers` 仅在 Active 状态显示，其他状态显示 `—`
- [ ] Pending 状态行不显示 [Upload V{n}] 按钮；Active 状态行显示且版本号正确递增
- [ ] 无 `drawing:upload` 权限时，Upload 入口不可见
- [ ] 无 `drawing:assign` 权限时，Assign 按钮不可见

### 10.2 上传图纸弹窗

- [ ] 新建模式：Code / Name / Category 可编辑；Code blur 时调接口唯一性校验
- [ ] 新版本模式：Code / Name / Category 只读；显示版本提示信息块
- [ ] 文件选择后显示文件名、大小；拖拽上传可正常触发
- [ ] 非 PDF/PNG/JPG 格式文件被拦截并提示错误
- [ ] 超 50MB 文件被拦截并提示错误
- [ ] 上传过程显示进度条；Submit 按钮 loading 防重复点击
- [ ] 提交成功后弹窗关闭，列表刷新，显示含 Assign 快捷入口的 Toast
- [ ] 有内容时点击取消/遮罩，弹出二次确认

### 10.3 分配 Site Engineer 弹窗

- [ ] 打开时加载已分配 / 未分配列表，双栏数量正确
- [ ] [Add] / [Remove] 即时更新两栏内容和计数，不触发接口
- [ ] 有原有人员被移除时，显示黄色警示提示文案
- [ ] 左栏为空时点击 Save，弹出二次确认，取消后不提交
- [ ] Save 成功后 Toast 提示、弹窗关闭，列表 Confirms 列数字刷新
- [ ] 有变更时点击 Cancel / ✕ / 遮罩，弹出关闭二次确认

### 10.4 审批 Todo 面板

- [ ] 铃铛角标正确展示待处理数量，审批完成后角标更新
- [ ] [Approve] 弹出确认框，确认后 Todo 消失，列表状态更新为 Active
- [ ] [Reject] 弹出驳回弹窗，Comment 为空时 [Confirm Rejection] 禁用
- [ ] 驳回成功后上传人收到站内消息

### 10.5 版本历史抽屉

- [ ] 版本按降序排列，最新版在顶部
- [ ] Current / Deprecated / Pending / Rejected 标签颜色正确
- [ ] Current 版本卡片使用主色边框高亮
- [ ] [View] 正确打开对应版本文件

### 10.6 查阅确认记录抽屉

- [ ] 副标题显示 `x Confirmed / y Assigned`，y 为已分配该图纸的人数
- [ ] Confirmed / Pending 两个 Tab 分别展示正确人员列表
- [ ] Pending Tab 中每人右侧显示橙色 Pending 标签
- [ ] 未被分配该图纸的 Site Engineer 不出现在列表中

---

## 11. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-003-pc.md](../../../requirements/pc/REQ-003-pc.md) | PC 端产品需求 |
| [REQ-003-shared.md](../../../requirements/shared/REQ-003-shared.md) | 跨端共享业务规则与 API 定义 |
| [UI-REQ-003-pc.md](../../ui/pc/UI-REQ-003-pc.md) | PC 端 UI 设计规范与组件规格 |
