# 前端开发说明文档 — PC 管理端图纸两级审批流程

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-007-pc.md](../../../requirements/pc/REQ-007-pc.md) + [REQ-007-shared.md](../../../requirements/shared/REQ-007-shared.md)
> **UI 设计参考**: [UI-REQ-007-pc.md](../../ui/pc/UI-REQ-007-pc.md)
> **依赖前端文档**: [FRONTEND-REQ-003-pc.md](./FRONTEND-REQ-003-pc.md)（图纸管理基础前端）
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-04-17

---

## 1. 功能概述

在 FRONTEND-REQ-003-pc 基础上扩展两级审批流程，涉及以下前端变更：

| 功能模块 | 变更类型 | 说明 |
|---------|---------|------|
| UploadDialog | **变更** | Approver 标签改为 "Internal Approver"，上传阻断条件更新 |
| TodoPanel — 内部审批 | **变更** | 标题/确认文案/通过后 Toast 调整 |
| TodoPanel — DC 外部审批 | **新增** | 外部审批 Todo 卡片 + Download Original + Mark Result |
| ExternalApprovalDialog | **新增** | Approved/Rejected 分支表单 + loading 等待 |
| HistoryDrawer | **重构** | 可展开行 + 4 阶段生命周期卡片 + 步骤条 |
| DrawingTable — StatusTag | **变更** | 3 态扩展为 5 态 |
| SearchPanel — Status 下拉 | **变更** | 选项扩展为 6 个 |
| DC 配置页面 | **新增** | 独立路由页面 + 添加 DC 弹窗 |

---

## 2. 技术栈与约定

沿用 [FRONTEND-REQ-003-pc §2](./FRONTEND-REQ-003-pc.md) 全部技术约定，新增：

| 项目 | 规范 |
|------|------|
| 新路由 | `/project/:projectId/settings/dc-config` — DC 配置页 |
| 新权限 | `drawing:external-approval`（DC 外部审批）、`drawing:dc-config`（DC 配置管理） |

---

## 3. 文件与目录结构

在 [FRONTEND-REQ-003-pc §3](./FRONTEND-REQ-003-pc.md) 基础上新增/变更：

```
src/
├── views/
│   └── drawing/
│       ├── components/
│       │   ├── UploadDialog.vue              # 变更：标签 + 阻断条件
│       │   ├── HistoryDrawer.vue             # 重构：展开行 + 4 阶段卡片
│       │   ├── VersionLifecycle.vue          # 新增：4 阶段卡片 + 步骤条
│       │   ├── ExternalApprovalDialog.vue    # 新增：外部审批标记 Dialog
│       │   └── StatusTag.vue                 # 变更：5 态支持
│       └── utils/
│           └── drawing-helper.js             # 变更：状态枚举扩展
│   └── settings/
│       └── dc-config/
│           ├── index.vue                     # DC 配置页面
│           └── AddDcDialog.vue               # 添加 DC 弹窗
├── api/
│   └── drawing.js                            # 新增外部审批 + DC 配置 API
└── layout/
    └── components/
        └── TodoPanel.vue                     # 变更：新增外部审批 Todo 类型
```

---

## 4. 页面与组件详细说明

### 4.1 状态枚举扩展 `drawing-helper.js`

```javascript
// 版本审批状态（DrawingVersion.approvalStatus）
export const VERSION_STATUS = {
  PENDING_INTERNAL: 'PENDING_INTERNAL',
  INTERNAL_APPROVED: 'INTERNAL_APPROVED',
  INTERNAL_REJECTED: 'INTERNAL_REJECTED',
  APPROVED: 'APPROVED',
  EXTERNAL_REJECTED: 'EXTERNAL_REJECTED'
}

// 状态标签映射
export const VERSION_STATUS_MAP = {
  PENDING_INTERNAL:  { label: 'Pending Internal',  color: '#E6A23C', bgColor: '#FDF6EC', icon: '⏳' },
  INTERNAL_APPROVED: { label: 'Pending External',  color: '#E6A23C', bgColor: '#FDF6EC', icon: '⏳' },
  INTERNAL_REJECTED: { label: 'Int. Rejected',     color: '#F56C6C', bgColor: '#FEF0F0', icon: '❌' },
  APPROVED:          { label: 'Approved',           color: '#67C23A', bgColor: '#F0F9EB', icon: '✅' },
  EXTERNAL_REJECTED: { label: 'Ext. Rejected',     color: '#F56C6C', bgColor: '#FEF0F0', icon: '❌' }
}

// 图纸主记录状态（Drawing.status）— 用于列表筛选
export const DRAWING_STATUS = {
  PENDING_INTERNAL: 'PENDING_INTERNAL',
  PENDING_EXTERNAL: 'PENDING_EXTERNAL',
  ACTIVE: 'ACTIVE',
  INTERNAL_REJECTED: 'INTERNAL_REJECTED',
  EXTERNAL_REJECTED: 'EXTERNAL_REJECTED'
}

// 筛选下拉选项
export const STATUS_FILTER_OPTIONS = [
  { label: 'All', value: '' },
  { label: 'Active', value: 'ACTIVE' },
  { label: 'Pending Internal', value: 'PENDING_INTERNAL' },
  { label: 'Pending External', value: 'PENDING_EXTERNAL' },
  { label: 'Int. Rejected', value: 'INTERNAL_REJECTED' },
  { label: 'Ext. Rejected', value: 'EXTERNAL_REJECTED' }
]

// Upload New Version 按钮是否禁用
export function isUploadDisabled(drawingStatus) {
  return [DRAWING_STATUS.PENDING_INTERNAL, DRAWING_STATUS.PENDING_EXTERNAL].includes(drawingStatus)
}
```

---

### 4.2 上传弹窗变更 `UploadDialog.vue`

#### 4.2.1 变更点

| 变更 | 说明 |
|------|------|
| Approver 标签 | i18n key 从 `label_approver` 改为 `label_internal_approver`，显示 "Internal Approver *" |
| 上传成功 Toast | i18n key 从 `toast_upload_success` 改为 `toast_upload_success_internal`，文案："Drawing uploaded successfully. Pending internal approval." |
| Upload New Version 按钮条件 | 父组件中 `isUploadDisabled(row.status)` 控制禁用 |

---

### 4.3 TodoPanel 变更 `TodoPanel.vue`

#### 4.3.1 新增 Todo 类型

```javascript
// 现有 Todo 类型
const TODO_TYPE = {
  DRAWING_APPROVAL: 'DRAWING_APPROVAL',           // 原内部审批（改名）
  DRAWING_EXTERNAL_APPROVAL: 'DRAWING_EXTERNAL_APPROVAL'  // 新增：DC 外部审批
}
```

#### 4.3.2 内部审批卡片变更

| 变更 | 说明 |
|------|------|
| 图标 | 🔍 |
| 标题 | `this.$t('todo_internal_approval_required')` → "Internal Approval Required" |
| Approve 确认文案 | "Once approved, this version will proceed to external approval by Document Controller." |
| 通过后 Toast | "Internal approval completed. DC has been notified for external approval." |
| 通过后行为 | 仅更新列表，**不**做 SE 推送相关 UI 提示 |
| **[Approve] 点击前预检查** | 点击 [Approve] 后**先调用 `getDcConfigList()` 检查 DC 列表**；若为空，弹出无 DC 警告弹窗（见 §4.3.5），**不打开**确认对话框 |

#### 4.3.5 无 DC 警告弹窗（新增）

当审批人点击 [Approve] 时，前端预检查 DC 配置为空，使用 `this.$confirm`（warning 类型，无 Cancel 按钮）或独立 Dialog 展示：

```vue
<!-- TodoPanel.vue 内部使用 el-dialog 实现 -->
<el-dialog
  :visible.sync="noDcWarningVisible"
  title=""
  width="420px"
  :show-close="false"
  :close-on-click-modal="false"
  :close-on-press-escape="false"
>
  <div class="no-dc-warning">
    <i class="el-icon-warning-outline no-dc-warning__icon"></i>
    <h3>{{ $t('no_dc_warning_title') }}</h3>
    <p>{{ $t('no_dc_warning_body') }}</p>
  </div>
  <span slot="footer">
    <el-button type="primary" @click="noDcWarningVisible = false">
      {{ $t('btn_got_it') }}
    </el-button>
  </span>
</el-dialog>
```

**对应核心方法**：

```javascript
data() {
  return {
    noDcWarningVisible: false,
    approveConfirmVisible: false,
    approvingItem: null
  }
},

methods: {
  async handleApproveClick(item) {
    // 前端预检查：查询 DC 配置
    try {
      const res = await getDcConfigList()
      if (!res.data.dcList || res.data.dcList.length === 0) {
        // 无 DC 配置，弹出警告弹窗，不进入确认对话框
        this.noDcWarningVisible = true
        return
      }
    } catch (e) {
      // 预检查接口失败，降级为后端兜底（仍打开确认对话框，由后端校验）
      console.warn('[DC pre-check] failed, fallback to backend validation', e)
    }
    // 有 DC 配置，打开确认对话框
    this.approvingItem = item
    this.approveConfirmVisible = true
  },

  async handleApproveConfirm() {
    this.approving = true
    try {
      await internalApprove({ drawingVersionId: this.approvingItem.relatedId })
      this.$message.success(this.$t('toast_internal_approved'))
      this.approveConfirmVisible = false
      this.$emit('refresh')
    } catch (err) {
      // 兜底：后端返回 1003007012 时提示无 DC
      const code = err.response?.data?.code
      if (code === 1003007012) {
        this.$message.error(this.$t('error_no_dc_on_approve'))
        this.approveConfirmVisible = false
        this.noDcWarningVisible = true
      } else {
        this.$message.error(err.response?.data?.msg || this.$t('error_unknown'))
      }
    } finally {
      this.approving = false
    }
  }
}
```

#### 4.3.3 外部审批卡片（新增）

```vue
<template>
  <div class="todo-card todo-card--external">
    <div class="todo-card__header">
      <span class="todo-card__icon">🌐</span>
      <span class="todo-card__title">{{ $t('todo_external_approval_required') }}</span>
    </div>
    <div class="todo-card__body">
      <p class="todo-card__drawing">{{ item.extra.drawingCode }}  {{ item.extra.drawingName }}  {{ item.extra.versionNo }}</p>
      <p class="todo-card__meta">Uploaded by: {{ item.extra.uploaderName }}  |  {{ formatTime(item.extra.uploadedTime) }}</p>
      <p class="todo-card__meta">Internal approved by: {{ item.extra.internalApproverName }}  |  {{ formatTime(item.extra.internalApprovedTime) }}</p>
      <p class="todo-card__note" v-if="item.extra.versionNote">Version Note: {{ item.extra.versionNote }}</p>
    </div>
    <div class="todo-card__actions">
      <el-button size="small" icon="el-icon-download" @click="handleDownload(item)">
        {{ $t('btn_download_original') }}
      </el-button>
      <el-button size="small" type="primary" @click="handleMarkResult(item)">
        {{ $t('btn_mark_result') }}
      </el-button>
    </div>
  </div>
</template>
```

#### 4.3.4 核心方法

```javascript
methods: {
  handleDownload(item) {
    // 直接触发浏览器下载
    const link = document.createElement('a')
    link.href = item.extra.fileUrl
    link.download = item.extra.fileName
    link.click()
  },

  handleMarkResult(item) {
    // 打开外部审批标记 Dialog
    this.externalApprovalVisible = true
    this.externalApprovalTarget = {
      drawingVersionId: item.relatedId,
      drawingCode: item.extra.drawingCode,
      drawingName: item.extra.drawingName,
      versionNo: item.extra.versionNo
    }
  }
}
```

---

### 4.4 外部审批标记 Dialog `ExternalApprovalDialog.vue`

#### 4.4.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 弹窗显示，`.sync` |
| `target` | Object | `{ drawingVersionId, drawingCode, drawingName, versionNo }` |

#### 4.4.2 Emits

| 事件 | 参数 | 说明 |
|------|------|------|
| `success` | `responseData` | 标记成功 |
| `update:visible` | Boolean | 关闭弹窗 |

#### 4.4.3 数据状态

```javascript
data() {
  return {
    form: {
      result: '',          // 'APPROVED' | 'REJECTED'
      signedFile: null,    // File object
      evidenceFile: null,  // File object
      externalApprovalDate: '',
      remark: '',
      comment: ''          // rejection reason
    },
    submitting: false
  }
}
```

#### 4.4.4 表单校验规则

```javascript
computed: {
  rules() {
    return {
      result: [{ required: true, message: this.$t('validation_result_required') }],
      signedFile: this.form.result === 'APPROVED'
        ? [{ required: true, message: this.$t('validation_signed_file_required') }]
        : [],
      evidenceFile: this.form.result === 'APPROVED'
        ? [{ required: true, message: this.$t('validation_evidence_required') }]
        : [],
      externalApprovalDate: this.form.result === 'APPROVED'
        ? [{ required: true, message: this.$t('validation_date_required') }]
        : [],
      comment: this.form.result === 'REJECTED'
        ? [{ required: true, message: this.$t('validation_comment_required') }]
        : []
    }
  }
}
```

#### 4.4.5 提交逻辑

```javascript
async handleSubmit() {
  await this.$refs.form.validate()
  this.submitting = true
  try {
    const formData = new FormData()
    formData.append('drawingVersionId', this.target.drawingVersionId)
    formData.append('action', this.form.result)

    if (this.form.result === 'APPROVED') {
      formData.append('signedFile', this.form.signedFile)
      formData.append('evidenceFile', this.form.evidenceFile)
      formData.append('externalApprovalDate', this.form.externalApprovalDate)
      if (this.form.remark) formData.append('remark', this.form.remark)
    } else {
      formData.append('comment', this.form.comment)
    }

    const res = await externalApprove(formData)
    this.$emit('success', res.data)
    this.$emit('update:visible', false)

    if (this.form.result === 'APPROVED') {
      this.$message.success(this.$t('toast_external_approved'))
      // "External approval marked. Drawing is now active and QR code has been generated."
    } else {
      this.$message.success(this.$t('toast_external_rejected'))
      // "External rejection recorded. Designer has been notified."
    }
  } catch (err) {
    // loading 恢复，Toast 显示后端错误文案
    this.$message.error(err.response?.data?.msg || this.$t('error_unknown'))
  } finally {
    this.submitting = false
  }
}
```

#### 4.4.6 日期限制

```javascript
// el-date-picker picker-options
pickerOptions: {
  disabledDate(time) {
    return time.getTime() > Date.now()
  }
}
```

#### 4.4.7 文件上传

```javascript
// signedFile: el-upload, auto-upload=false, limit=1
// 格式限制: PDF/DWG/DXF/PNG/JPG, ≤50MB
function beforeSignedUpload(file) {
  const allowed = ['application/pdf', 'image/png', 'image/jpeg',
    '.dwg', '.dxf'] // MIME + extension check
  const maxSize = 50 * 1024 * 1024
  // validate and set this.form.signedFile = file
  return false // prevent auto upload
}

// evidenceFile: el-upload, auto-upload=false, limit=1
// 格式限制: PDF/PNG/JPG, ≤20MB
function beforeEvidenceUpload(file) {
  const allowed = ['application/pdf', 'image/png', 'image/jpeg']
  const maxSize = 20 * 1024 * 1024
  return false
}
```

---

### 4.5 版本历史抽屉 `HistoryDrawer.vue`（重构）

#### 4.5.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 抽屉显示，`.sync` |
| `drawing` | Object | 当前图纸对象 `{ id, drawingCode, drawingName }` |

#### 4.5.2 数据状态

```javascript
data() {
  return {
    versions: [],     // 版本列表，含生命周期数据
    loading: false,
    expandedRows: []  // 当前展开的版本 ID 列表
  }
}
```

#### 4.5.3 版本列表数据结构（API 响应映射）

```javascript
// 每个 version 对象包含 4 阶段数据
{
  id: 301,
  versionNo: 'V3',
  approvalStatus: 'APPROVED',
  // ① Uploaded
  fileName: 'arch001-v3.pdf',
  fileUrl: 'https://...',
  fileSize: 2621440,
  uploaderName: '张三',
  uploadedAt: '2026-04-01T10:00:00',
  versionNote: '修正轴网尺寸',
  // ② Internal Approval
  internalApproval: {
    status: 'APPROVED',     // APPROVED | REJECTED | PENDING | null
    approverName: '王总工',
    approvedAt: '2026-04-02T14:30:00',
    comment: '尺寸已修正'
  },
  // ③ External Approval
  externalApproval: {
    status: 'APPROVED',     // APPROVED | REJECTED | PENDING | null
    approverName: 'DC 陈小明',
    externalApprovalDate: '2026-04-05',
    evidenceFileName: 'bentley.pdf',
    evidenceFileUrl: 'https://...',
    remark: '业主已签批'
  },
  // ④ Signed Version
  signedFile: {
    fileName: 'arch001-v3-signed.pdf',
    fileUrl: 'https://...',
    fileSize: 3250585,
    uploadedAt: '2026-04-05T16:00:00'
  },
  // Confirmed
  confirmedCount: 5,
  totalAssigned: 12,
  // QR
  qrImageUrl: 'https://...'
}
```

#### 4.5.4 展开/收起

```javascript
methods: {
  toggleExpand(versionId) {
    const idx = this.expandedRows.indexOf(versionId)
    if (idx > -1) {
      this.expandedRows.splice(idx, 1)
    } else {
      this.expandedRows.push(versionId)
    }
  },
  isExpanded(versionId) {
    return this.expandedRows.includes(versionId)
  }
}
```

#### 4.5.5 子组件 `VersionLifecycle.vue`

| Prop | 类型 | 说明 |
|------|------|------|
| `version` | Object | 版本数据（含 4 阶段） |

负责渲染 2×2 卡片网格 + 底部步骤条。

步骤条状态计算：

```javascript
computed: {
  steps() {
    const v = this.version
    return [
      { icon: '📄', label: 'Uploaded', status: 'done' }, // 始终 done
      {
        icon: '🔍', label: 'Internal',
        status: v.internalApproval?.status === 'APPROVED' ? 'done'
          : v.internalApproval?.status === 'REJECTED' ? 'failed'
          : v.internalApproval?.status === 'PENDING' ? 'active'
          : 'wait'
      },
      {
        icon: '🌐', label: 'External',
        status: v.externalApproval?.status === 'APPROVED' ? 'done'
          : v.externalApproval?.status === 'REJECTED' ? 'failed'
          : v.externalApproval?.status === 'PENDING' ? 'active'
          : 'wait'
      },
      {
        icon: '✍️', label: 'Signed',
        status: v.signedFile ? 'done' : 'wait'
      }
    ]
  }
}
```

#### 4.5.6 卡片内 Mark Result 按钮（版本历史内）

```javascript
// ③ External Approval 卡片中的 [✅ Mark Result]
// 仅 PENDING_EXTERNAL 状态 + 当前用户有 drawing:external-approval 权限 + 是项目 DC 时可见
// 点击打开与 TodoPanel 相同的 ExternalApprovalDialog
```

---

### 4.6 DC 配置页面 `settings/dc-config/index.vue`

#### 4.6.1 数据状态

```javascript
data() {
  return {
    dcList: [],          // 已配置 DC 列表
    loading: false,
    addDialogVisible: false
  }
}
```

#### 4.6.2 核心方法

```javascript
methods: {
  async fetchDcList() {
    this.loading = true
    const res = await getDcConfigList()
    this.dcList = res.data.dcList
    this.loading = false
  },

  async handleRemove(dc) {
    await this.$confirm(
      this.$t('confirm_remove_dc', { name: dc.dcUserName }),
      { type: 'warning' }
    )
    // 构建移除后的 dcUserIds
    const newIds = this.dcList
      .filter(d => d.dcUserId !== dc.dcUserId)
      .map(d => d.dcUserId)
    await updateDcConfig({ dcUserIds: newIds })
    this.$message.success(this.$t('toast_dc_removed'))
    this.fetchDcList()
    // 若列表为空，显示警告
    if (newIds.length === 0) {
      this.$message.warning(this.$t('warning_no_dc'))
    }
  },

  handleAddSuccess() {
    this.fetchDcList()
  }
}
```

#### 4.6.3 添加 DC 弹窗 `AddDcDialog.vue`

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | `.sync` |

```javascript
data() {
  return {
    keyword: '',
    availableUsers: [],
    loading: false
  }
}

methods: {
  async fetchAvailable() {
    const res = await getDcConfigList()
    this.availableUsers = res.data.availableUsers
  },

  async handleAdd(user) {
    // 获取当前 dcList + 新用户
    const res = await getDcConfigList()
    const currentIds = res.data.dcList.map(d => d.dcUserId)
    currentIds.push(user.userId)
    await updateDcConfig({ dcUserIds: currentIds })
    this.$message.success(this.$t('toast_dc_added'))
    this.fetchAvailable()
    this.$emit('success')
  },

  filteredUsers() {
    if (!this.keyword) return this.availableUsers
    return this.availableUsers.filter(u =>
      u.userName.toLowerCase().includes(this.keyword.toLowerCase())
    )
  }
}
```

---

## 5. API 封装 `api/drawing.js`

在现有文件中新增：

```javascript
// === REQ-007 新增 API ===

/**
 * 外部审批标记
 * POST /drawing/external-approve
 * Content-Type: multipart/form-data
 */
export function externalApprove(formData) {
  return request({
    url: '/drawing/external-approve',
    method: 'post',
    data: formData,
    headers: { 'Content-Type': 'multipart/form-data' },
    timeout: 30000 // 同步等待 QR 生成，超时设长
  })
}

/**
 * 获取 DC 配置列表
 * GET /project/dc-config/list
 */
export function getDcConfigList() {
  return request({ url: '/project/dc-config/list', method: 'get' })
}

/**
 * 更新 DC 配置（全量覆盖）
 * POST /project/dc-config/update
 */
export function updateDcConfig(data) {
  return request({ url: '/project/dc-config/update', method: 'post', data })
}

/**
 * 获取版本历史（含 4 阶段生命周期数据）
 * GET /drawing/version/history?drawingId=xxx
 */
export function getVersionHistory(drawingId) {
  return request({ url: '/drawing/version/history', method: 'get', params: { drawingId } })
}
```

---

## 6. 状态管理 `store/modules/drawing.js`

在现有 Vuex module 中新增：

```javascript
// state 新增
externalApprovalVisible: false,
externalApprovalTarget: null,

// mutations 新增
SET_EXTERNAL_APPROVAL_VISIBLE(state, visible) {
  state.externalApprovalVisible = visible
},
SET_EXTERNAL_APPROVAL_TARGET(state, target) {
  state.externalApprovalTarget = target
},

// actions 新增
async externalApprove({ commit, dispatch }, formData) {
  const res = await externalApproveApi(formData)
  dispatch('fetchTodoList') // 刷新 Todo
  dispatch('fetchList')     // 刷新图纸列表
  return res
}
```

---

## 7. 路由配置

```javascript
// router/modules/project.js 新增
{
  path: 'settings/dc-config',
  name: 'DcConfig',
  component: () => import('@/views/settings/dc-config/index.vue'),
  meta: {
    title: 'DC Configuration',
    permission: 'drawing:dc-config',
    breadcrumb: ['Settings', 'DC Configuration']
  }
}
```

---

## 8. 国际化 i18n key

```javascript
// 新增 key（en.js）
{
  // 上传弹窗
  label_internal_approver: 'Internal Approver',
  toast_upload_success_internal: 'Drawing uploaded successfully. Pending internal approval.',

  // 内部审批 Todo
  todo_internal_approval_required: 'Internal Approval Required',
  confirm_internal_approval_title: 'Confirm Internal Approval?',
  confirm_internal_approval_body: 'Once approved, this version will proceed to external approval by Document Controller.',
  toast_internal_approved: 'Internal approval completed. DC has been notified for external approval.',

  // 外部审批 Todo
  todo_external_approval_required: 'External Approval Required',
  btn_download_original: 'Download Original',
  btn_mark_result: 'Mark Result',

  // 外部审批 Dialog
  dialog_mark_external_title: 'Mark External Approval Result',
  label_result: 'Result',
  label_signed_file: 'Signed Drawing File',
  label_evidence: 'Approval Evidence',
  label_external_date: 'External Approval Date',
  label_remarks: 'Remarks',
  label_rejection_reason: 'Rejection Reason',
  btn_processing: 'Processing...',
  toast_external_approved: 'External approval marked. Drawing is now active and QR code has been generated.',
  toast_external_rejected: 'External rejection recorded. Designer has been notified.',

  // 版本历史
  stage_uploaded: 'Uploaded',
  stage_internal: 'Internal Approval',
  stage_external: 'External Approval',
  stage_signed: 'Signed Version',
  label_not_reached: 'Not reached',

  // 无 DC 警告弹窗（F-004）
  no_dc_warning_title: 'No DC Configured',
  no_dc_warning_body: 'This project has no Document Controller configured. Please contact your project admin to add a DC before proceeding with internal approval.',
  btn_got_it: 'Got it',
  error_no_dc_on_approve: 'No DC configured. Please ask your admin to add a DC first.',

  // DC 配置
  page_dc_config: 'DC Configuration',
  dc_config_hint: 'At least one DC is required for the drawing approval process.',
  dialog_add_dc: 'Add Document Controller',
  confirm_remove_dc: 'Remove {name} from DC list? They will no longer receive external approval tasks.',
  toast_dc_added: 'DC added successfully.',
  toast_dc_removed: 'DC removed successfully.',
  warning_no_dc: 'No DC configured. Internal approvals cannot proceed to external approval.',

  // 状态筛选
  status_pending_internal: 'Pending Internal',
  status_pending_external: 'Pending External',
  status_int_rejected: 'Int. Rejected',
  status_ext_rejected: 'Ext. Rejected',

  // 校验
  validation_result_required: 'Please select a result',
  validation_signed_file_required: 'Signed drawing file is required',
  validation_evidence_required: 'Approval evidence is required',
  validation_date_required: 'External approval date is required',
  validation_comment_required: 'Rejection reason is required'
}
```

---

## 9. 验收条件

### 上传弹窗
- [ ] Approver 标签显示 "Internal Approver *"
- [ ] 图纸状态为 `PENDING_INTERNAL` 或 `PENDING_EXTERNAL` 时 Upload New Version 按钮 disabled
- [ ] 上传成功 Toast 显示 "Pending internal approval"

### 内部审批 Todo
- [ ] 卡片图标 🔍 + 标题 "Internal Approval Required"
- [ ] Approve 确认弹窗文案正确
- [ ] 通过后 Toast 提示 DC 已通知
- [ ] 通过后刷新 Todo 列表 + 图纸列表
- [ ] **点击 [Approve] 时若项目无 DC 配置，弹出无 DC 警告弹窗，不打开确认对话框**
- [ ] **无 DC 警告弹窗仅有 [Got it] 按钮，点击关闭后卡片状态不变**
- [ ] **无 DC 警告弹窗不可通过背景点击或 ESC 键关闭**
- [ ] **无 DC 配置时 [Reject] 流程不受影响**

### DC 外部审批 Todo
- [ ] 内部审批通过后外部审批 Todo 出现在 DC 的 Todo 列表
- [ ] `[Download Original]` 触发浏览器下载原始文件
- [ ] `[Mark Result]` 打开 ExternalApprovalDialog

### 外部审批标记 Dialog
- [ ] 未选择 Result 时 Confirm 按钮 disabled
- [ ] 选择 Approved 后：签字版文件、凭证、日期必填；日期不可选未来
- [ ] 选择 Rejected 后：Rejection Reason 必填
- [ ] Approved 提交：loading 态 → 成功后 Dialog 关闭 + Toast + Todo 刷新
- [ ] Approved 失败：loading 恢复 + 错误 Toast
- [ ] Rejected 提交：Dialog 关闭 + Toast
- [ ] `externalApprove` API timeout 设为 30s

### 版本历史抽屉
- [ ] 抽屉宽度 720px
- [ ] 主列表 5 种状态标签颜色正确
- [ ] 点击 ▶ 展开 4 阶段卡片 + 步骤条
- [ ] PENDING_EXTERNAL 版本：③ 卡片显示 `[Mark Result]`（DC 权限可见）
- [ ] APPROVED 版本：④ 卡片可 Preview / Download 签字版
- [ ] 步骤条颜色与状态一致

### DC 配置页面
- [ ] 路由 `/project/:projectId/settings/dc-config` 正确加载
- [ ] 权限 `drawing:dc-config` 控制页面可见性
- [ ] `[+ Add DC]` 弹窗仅显示有权限且未配置的用户
- [ ] 添加/移除后列表实时刷新
- [ ] 移除最后一个 DC 时 warning Toast

### 筛选
- [ ] Status 下拉包含 6 个选项
- [ ] 各选项筛选结果正确
