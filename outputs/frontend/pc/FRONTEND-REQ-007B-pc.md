---
doc_type: frontend_spec
req_id: REQ-007B-pc
version: 0.2.2
status: draft
generated_from: REQ-007B-pc.md@0.2.2
ui_spec_ref: UI-REQ-007B-pc.md
generated_at: 2026-05-26
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明 — PC 端 DC 外部审批 Todo 与标记 Dialog

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-007B-pc.md](../../../requirements/pc/REQ-007B-pc.md) @ v0.2.2
> **UI 设计参考**: [UI-REQ-007B-pc.md](../../ui/pc/UI-REQ-007B-pc.md)
> **依赖前端文档**: [FRONTEND-REQ-007A-pc.md](./FRONTEND-REQ-007A-pc.md)（内部审批，复用 DrawingApprovalDetailDrawer 基础结构）
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-05-26

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007B-pc @ v0.2.2 |
| UI 设计 | UI-REQ-007B-pc.md |
| 覆盖 Story | US-007B-001 / US-007B-002 |
| 覆盖 AC | AC-007B-001 ~ AC-007B-015 |

---

## 1. 功能概述

在 FRONTEND-REQ-007A-pc 基础上，本文档覆盖 DC 外部审批相关前端变更：

| 功能模块 | 变更类型 | 说明 |
|---------|---------|------|
| TodoPanel — 外部审批卡片（F-001） | **新增** | 图标 🌐、标题 "External Approval Required"、操作区仅保留 [Detail] 按钮 |
| ExternalApprovalDetailDrawer（F-002） | **新增** | 外部审批详情侧滑弹框：图纸提交信息展示 + 文件 inline 打开 + 强制下载 + Mark Result 入口 |
| MarkExternalApprovalDialog（F-003） | **新增** | Status of Approval 下拉（A–E）+ 报审三要素（始终必填）+ 条件显示字段 + loading 防重复 |

---

## 2. 技术栈

### 2.1 已有技术栈（继承自 FRONTEND-REQ-007A-pc）

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 |
| UI 库 | Element UI |
| HTTP 客户端 | Axios（封装在 `src/api/`） |
| 状态管理 | Vuex |
| 路由 | Vue Router |
| 构建 | Webpack / Vue CLI |

### 2.2 本需求无新增依赖

所有实现复用现有 Element UI 组件（`el-drawer`、`el-dialog`、`el-button`、`el-form`、`el-select`、`el-date-picker`、`el-upload`、`el-skeleton`）。

---

## 3. 文件与目录结构

在 FRONTEND-REQ-007A-pc 基础上新增以下文件：

```
src/
├── views/
│   └── drawing/
│       └── components/
│           ├── TodoPanel.vue                          # 变更：新增外部审批卡片（type=DRAWING_EXTERNAL_APPROVAL）
│           ├── ExternalApprovalDetailDrawer.vue       # 新增：外部审批详情侧滑弹框（F-002）
│           └── MarkExternalApprovalDialog.vue         # 新增：标记外部审批结果 Dialog（F-003）
├── api/
│   └── drawing.js                                     # 变更：新增 view/download/external-approve 接口
└── store/
    └── modules/
        └── todo.js                                    # 变更：新增 extDrawerVisible、currentExtTodoItem
```

---

## 4. 组件结构

### 4.1 组件树

```
<TodoPanel>
├── <ExternalApprovalTodoCard>           # 外部审批卡片（F-001），内联在 TodoPanel
│   └── [Detail] Button
├── <ExternalApprovalDetailDrawer>       # 详情侧滑弹框（F-002）
│   ├── Skeleton（加载态）
│   ├── FieldList（只读字段区）
│   │   └── OriginalFileLink            # 文件名可点击链接 + 辅助文案
│   └── ActionBar
│       ├── [Download Original File]    # 强制下载
│       └── [Mark Result]              # 触发 F-003
└── <MarkExternalApprovalDialog>        # 标记外部审批结果 Dialog（F-003）
```

### 4.2 组件说明

#### `TodoPanel.vue`（变更）

**关联 AC**: AC-007B-001

**变更点**：在渲染 `type === 'DRAWING_EXTERNAL_APPROVAL'` 的 Todo 卡片时：
- 图标改为 🌐
- 标题固定为 `"External Approval Required"`
- 操作区只保留 `[Detail]` 按钮
- 列表无外部审批 Todo 时：展示空状态 `"No pending tasks"`
- 注意：后端已按 `assigneeId` 过滤，前端不增加额外筛选控件

```vue
<!-- AC-007B-001: 外部审批卡片 -->
<template v-if="item.type === 'DRAWING_EXTERNAL_APPROVAL'">
  <span class="todo-icon">🌐</span>
  <span class="todo-title">External Approval Required</span>
  <div class="todo-meta">
    {{ item.drawingCode }}  {{ item.drawingName }}  {{ item.versionNo }}
  </div>
  <div class="todo-meta">
    Uploaded by: {{ item.uploaderName }}（Designer） | {{ item.uploadTime | formatDateTime }}
  </div>
  <div class="todo-meta">
    Internal approved by: {{ item.internalApproverName }} | {{ item.internalApprovedTime | formatDateTime }}
  </div>
  <div v-if="item.versionNote" class="todo-meta">
    Version Note: {{ item.versionNote }}
  </div>
  <el-button size="small" @click="openExtDetail(item)">Detail</el-button>
</template>
```

**不该做**：
- 不在前端过滤 assigneeId（后端已过滤）
- 不在卡片上直接放 [Mark Result] 按钮（入口统一在 Drawer 底部）

---

#### `ExternalApprovalDetailDrawer.vue`（新增，F-002）

**关联 AC**: AC-007B-002 / AC-007B-003 / AC-007B-004 / AC-007B-005

**触发方式**：点击外部审批 Todo 卡片右侧 [Detail] 按钮，从页面右侧滑入（`el-drawer`，`:size="480px"`，`direction="rtl"`）。

**Props**：

```ts
interface ExternalApprovalDetailDrawerProps {
  visible: boolean         // v-model 控制显隐
  todoItem: ExternalTodoItem | null  // 当前 Todo（含 versionId、drawingCode 等）
}
```

**数据加载**：
- `watch(visible)` → `true` 时调用 `GET /drawing/version/{versionId}/detail` 加载弹框字段
- 加载中展示 `el-skeleton`（4 行），加载失败展示 `"Failed to load drawing details."` + `[Retry]` 按钮
- 点击 `×` 或按 `Esc` 关闭 Drawer，审批状态不变

**展示字段**（均只读，不可编辑）：

| 字段名 | 数据字段 | 空值处理 |
|--------|---------|---------|
| Category | `drawing.category` | 显示 `—` |
| Description | `drawing.description` | 显示 `—` |
| Version | `version.versionNo` | — |
| Version Note | `version.versionNote` | 显示 `—` |
| Uploaded by | `version.uploaderName` | 姓名后追加 `（Designer）` |
| Upload Time | `version.uploadTime` | 格式：`YYYY-MM-DD HH:mm`，Tooltip 显示完整时间戳 |
| Internal Approver | `version.approverName` | — |
| Internal Approved Time | `approval.approvedAt`（phase=INTERNAL） | 格式：`YYYY-MM-DD HH:mm` |
| Original File | `version.fileUrl` 对应文件名 | fileUrl 为空时：文件名区域显示 `—` |

> ⚠️ **不展示 Drawing Code 和 Drawing Name**（已在 Todo 卡片摘要行展示，避免冗余）。覆盖 AC-007B-003。

**Original File 交互**：
```vue
<!-- inline 打开：target="_blank" -->
<a :href="fileViewUrl" target="_blank" class="file-link">
  <i class="file-icon"></i> {{ fileName }}
</a>
<p class="file-hint">
  Click to open in browser. To save the file, use [Download Original File] below.
</p>
```

**底部 ActionBar**：

```vue
<div class="drawer-footer">
  <!-- AC-007B-004: 强制下载 -->
  <el-button
    :loading="downloading"
    :disabled="!todoItem.fileUrl"
    @click="handleDownload"
  >
    Download Original File
  </el-button>

  <!-- 打开 F-003 -->
  <el-button type="success" @click="openMarkDialog">
    ✅ Mark Result
  </el-button>
</div>
```

**下载逻辑**（覆盖 AC-007B-004 / AC-007B-005）：
```javascript
async handleDownload() {
  // AC-007B-004
  this.downloading = true
  try {
    const res = await drawingApi.getDownloadUrl(this.todoItem.versionId) // GET /drawing/version/{id}/download → 302 预签名 URL
    const link = document.createElement('a')
    link.href = res.data.url
    link.setAttribute('download', `${this.todoItem.drawingCode}-V${this.todoItem.versionNo}-original.${this.ext}`)
    document.body.appendChild(link)
    link.click()
    link.remove()
  } catch (err) {
    // AC-007B-005: 403 时提示权限不足；其余错误 Toast 提示
    this.$message.error(err.status === 403 ? 'Permission denied.' : 'Download failed. Please try again.')
  } finally {
    this.downloading = false
  }
}
```

**fileUrl 为空时**：[Download Original File] 置灰，Tooltip 提示 `"Original file unavailable."`

---

#### `MarkExternalApprovalDialog.vue`（新增，F-003）

**关联 AC**: AC-007B-006 ~ AC-007B-015

**触发方式**：Drawer 底部 [Mark Result] 按钮，使用 `el-dialog`，`width="560px"`，`close-on-click-modal="false"`，`close-on-press-escape="false"`（loading 中禁止关闭）。

**Header**：
```
Mark External Approval Result        [✕]
{drawingCode}  {drawingName}  {versionNo}
```

**表单结构与条件显隐**：

```vue
<el-form ref="markForm" :model="form" :rules="rules" label-position="top">

  <!-- 始终显示：Status of Approval -->
  <!-- AC-007B-006: 未选中时 Confirm 禁用 -->
  <el-form-item label="Status of Approval *" prop="statusOfApproval">
    <el-select v-model="form.statusOfApproval" placeholder="Please select">
      <el-option label="A – Approved / No Exception Taken"              value="A" />
      <el-option label="B – Approved with comment, resubmission required" value="B" />
      <el-option label="C – Revise And Resubmit"                        value="C" />
      <el-option label="D – For Record Purpose"                         value="D" />
      <el-option label="E – Others (please state reason)"               value="E" />
    </el-select>
  </el-form-item>

  <!-- Status = E 时显示：Others Reason（必填）-->
  <!-- AC-007B-007B -->
  <el-form-item
    v-if="form.statusOfApproval === 'E'"
    label="Others Reason *"
    prop="othersReason"
  >
    <el-input type="textarea" v-model="form.othersReason" :maxlength="500" show-word-limit />
  </el-form-item>

  <!-- 分隔线 -->
  <el-divider />

  <!-- 始终显示：报审三要素（始终必填）-->
  <!-- AC-007B-006 -->
  <el-form-item label="Submission Ref No. *" prop="submissionRefNo">
    <el-input v-model="form.submissionRefNo" :maxlength="200" />
  </el-form-item>

  <el-form-item label="Submission Subject *" prop="submissionSubject">
    <el-input v-model="form.submissionSubject" :maxlength="200" />
  </el-form-item>

  <el-form-item label="Submission Description *" prop="submissionDescription">
    <el-input type="textarea" v-model="form.submissionDescription" :maxlength="1000" show-word-limit />
  </el-form-item>

  <!-- Status A / B / D 时显示：签字版文件 + 凭证 + 审批日期（必填）-->
  <!-- AC-007B-007 -->
  <template v-if="['A','B','D'].includes(form.statusOfApproval)">
    <el-form-item label="Signed Drawing File *" prop="signedFile">
      <el-upload
        action="#"
        :http-request="uploadSignedFile"
        :before-upload="validateSignedFile"
        :limit="1"
        :file-list="form.signedFileList"
      >
        <el-button icon="el-icon-paperclip">Click or drag to upload</el-button>
        <div slot="tip" class="upload-tip">PDF / DWG / DXF / PNG / JPG · Max 50MB</div>
      </el-upload>
    </el-form-item>

    <el-form-item label="Approval Evidence *" prop="evidenceFile">
      <el-upload
        action="#"
        :http-request="uploadEvidenceFile"
        :before-upload="validateEvidenceFile"
        :limit="1"
        :file-list="form.evidenceFileList"
      >
        <el-button icon="el-icon-paperclip">Click or drag to upload</el-button>
        <div slot="tip" class="upload-tip">PDF / PNG / JPG · Max 20MB</div>
      </el-upload>
    </el-form-item>

    <el-form-item label="External Approval Date *" prop="externalApprovalDate">
      <!-- 不可选未来日期 -->
      <el-date-picker
        v-model="form.externalApprovalDate"
        type="date"
        placeholder="YYYY-MM-DD"
        :picker-options="{ disabledDate: d => d > Date.now() }"
        value-format="yyyy-MM-dd"
      />
    </el-form-item>
  </template>

  <!-- 始终显示：Remarks（选填）-->
  <el-form-item label="Remarks">
    <el-input type="textarea" v-model="form.remarks" :maxlength="500" show-word-limit />
  </el-form-item>

</el-form>
```

**表单校验规则**（`rules`）：

```javascript
computed: {
  rules() {
    const s = this.form.statusOfApproval
    return {
      statusOfApproval: [{ required: true, message: '请选择审批结果', trigger: 'change' }],
      othersReason: s === 'E'
        ? [{ required: true, message: '必填', trigger: 'blur' }]
        : [],
      submissionRefNo: [{ required: true, message: '必填', trigger: 'blur' }],
      submissionSubject: [{ required: true, message: '必填', trigger: 'blur' }],
      submissionDescription: [{ required: true, message: '必填', trigger: 'blur' }],
      signedFile: ['A','B','D'].includes(s)
        ? [{ required: true, message: '请上传签字版图纸', trigger: 'change' }]
        : [],
      evidenceFile: ['A','B','D'].includes(s)
        ? [{ required: true, message: '请上传审批凭证', trigger: 'change' }]
        : [],
      externalApprovalDate: ['A','B','D'].includes(s)
        ? [{ required: true, message: '请选择审批日期', trigger: 'change' }]
        : [],
    }
  }
}
```

**文件校验**：

```javascript
// 签字版文件：PDF / DWG / DXF / PNG / JPG，≤ 50MB
validateSignedFile(file) {
  const ALLOWED = ['application/pdf', 'image/png', 'image/jpeg',
                   '.dwg', '.dxf']  // MIME + 扩展名双重校验
  const MAX = 50 * 1024 * 1024
  if (file.size > MAX) {
    this.$message.error('文件不得超过 50MB')
    return false
  }
  // ... 格式校验
  return true
}

// 凭证文件：PDF / PNG / JPG，≤ 20MB
validateEvidenceFile(file) {
  const MAX = 20 * 1024 * 1024
  if (file.size > MAX) {
    this.$message.error('文件不得超过 20MB')
    return false
  }
  // ...
  return true
}
```

**提交逻辑**（覆盖 AC-007B-008 / AC-007B-010 ~ AC-007B-015）：

```javascript
async handleConfirm() {
  // AC-007B-015: 防重复提交 — loading 期间按钮禁用
  const valid = await this.$refs.markForm.validate().catch(() => false)
  if (!valid) return

  this.loading = true  // [Confirm] → "Processing..."，Dialog 内所有操作禁用

  try {
    // 文件上传（Status A/B/D 时）
    if (['A','B','D'].includes(this.form.statusOfApproval)) {
      // 文件已在 el-upload http-request 中预上传（OSS 预签名），此处传 key
    }

    await drawingApi.markExternalApproval({
      versionId: this.versionId,
      statusOfApproval: this.form.statusOfApproval,
      othersReason: this.form.othersReason,
      submissionRefNo: this.form.submissionRefNo,
      submissionSubject: this.form.submissionSubject,
      submissionDescription: this.form.submissionDescription,
      signedFileKey: this.form.signedFileKey,   // Status A/B/D
      evidenceFileKey: this.form.evidenceFileKey, // Status A/B/D
      externalApprovalDate: this.form.externalApprovalDate, // Status A/B/D
      remarks: this.form.remarks,
    })

    // AC-007B-008: Status A/B/D 成功
    if (['A','B','D'].includes(this.form.statusOfApproval)) {
      this.$message.success(
        'External approval marked. Drawing is now active and QR code has been generated.'
      )
    }
    // AC-007B-012: Status C 成功
    else if (this.form.statusOfApproval === 'C') {
      this.$message.success(
        'External rejection recorded. Designer has been notified.'
      )
    }
    // Status E 成功（同 C 路径）
    else {
      this.$message.success(
        'External rejection recorded. Designer has been notified.'
      )
    }

    this.$emit('success')  // Todo 卡片消失，Drawer 关闭
    this.handleClose()

  } catch (err) {
    // AC-007B-010: QR 生成失败（后端返回特定错误码）
    if (err.code === 'QR_GENERATION_FAILED') {
      this.$message.error('QR generation failed, please retry')
    } else {
      this.$message.error('Operation failed, please retry')
    }
    // loading 恢复，Dialog 保留，可重试
  } finally {
    this.loading = false
  }
}
```

**[Confirm] 按钮状态**：
```vue
<el-button
  type="primary"
  :loading="loading"
  :disabled="!form.statusOfApproval || loading"
  @click="handleConfirm"
>
  {{ loading ? 'Processing...' : 'Confirm' }}
</el-button>
```

---

## 5. 状态管理

### 5.1 Vuex — todo.js 变更

```javascript
// 新增 state
extDrawerVisible: false,
currentExtTodoItem: null,

// 新增 mutations
SET_EXT_DRAWER_VISIBLE(state, val) { state.extDrawerVisible = val },
SET_CURRENT_EXT_TODO(state, item)  { state.currentExtTodoItem = item },

// 新增 actions
openExtDetail({ commit }, todoItem) {
  commit('SET_CURRENT_EXT_TODO', todoItem)
  commit('SET_EXT_DRAWER_VISIBLE', true)
},
closeExtDetail({ commit }) {
  commit('SET_EXT_DRAWER_VISIBLE', false)
  commit('SET_CURRENT_EXT_TODO', null)
},
```

### 5.2 缓存失效策略

| 操作 | 失效 key |
|------|---------|
| 标记外部审批结果成功 | `todo:external-list`、`drawing:version:{versionId}` |

---

## 6. API 接入

### 6.1 调用清单

| API | 组件 | 说明 |
|-----|------|------|
| `GET /drawing/version/{versionId}/detail` | `ExternalApprovalDetailDrawer` | 加载 Drawer 内详情字段 |
| `GET /drawing/version/{versionId}/view` | OriginalFileLink（href 赋值） | 302 → 预签名 URL（inline 打开），有效期 5 分钟 |
| `GET /drawing/version/{versionId}/download` | [Download Original File] | 302 → 预签名 URL（attachment 下载），有效期 5 分钟 |
| `POST /drawing/external-approve` | `MarkExternalApprovalDialog` | 提交外部审批标记结果 |

### 6.2 接口调用封装（drawing.js 新增）

```javascript
// drawing.js
export const drawingApi = {
  // 获取版本详情（Drawer 展示）
  getVersionDetail: (versionId) =>
    axios.get(`/drawing/version/${versionId}/detail`),

  // 获取 inline 查看 URL（重定向到预签名 URL）
  getViewUrl: (versionId) =>
    axios.get(`/drawing/version/${versionId}/view`, { maxRedirects: 0 }),

  // 获取强制下载 URL
  getDownloadUrl: (versionId) =>
    axios.get(`/drawing/version/${versionId}/download`, { maxRedirects: 0 }),

  // 提交外部审批标记
  markExternalApproval: (data) =>
    axios.post('/drawing/external-approve', data),
}
```

### 6.3 错误处理

| 错误码 | 用户可见文案 | 处理位置 |
|--------|-----------|---------|
| 403 | `"Permission denied."` | 全局拦截器 + 组件内兜底 |
| 404 | `"Drawing version not found."` | Drawer 内错误状态 |
| QR_GENERATION_FAILED | `"QR generation failed, please retry"` | `MarkExternalApprovalDialog` catch |
| 网络超时 / 500 | `"Operation failed, please retry"` | `MarkExternalApprovalDialog` catch |

---

## 7. 权限控制（前端层）

```javascript
// 路由守卫 / 组件内：无 drawing:external-approval 权限时不渲染外部审批卡片和 Drawer
// AC-007B-014
computed: {
  canMarkExternal() {
    return this.$store.getters.permissions.includes('drawing:external-approval')
  }
}
```

```vue
<!-- 只有 DC 可见 -->
<template v-if="canMarkExternal">
  <el-button type="success" @click="openMarkDialog">✅ Mark Result</el-button>
</template>
```

> 注意：前端仅做展示层隐藏，最终权限校验由后端完成（403）。

---

## 8. 弱网与异常策略

| 场景 | 策略 |
|-----|------|
| Drawer 加载超时 | 展示 `"Failed to load drawing details. [Retry]"`，点击 Retry 重新请求 |
| 原始文件下载失败 | Toast `"Download failed. Please try again."`，按钮恢复可点击 |
| QR 生成失败 | Dialog loading 恢复，Toast 提示，保留表单，DC 可原地重试（覆盖 AC-007B-010） |
| 文件上传失败 | Dialog loading 恢复，Toast 报错，表单数据不清空，DC 可重试 |
| loading 中断网 | `finally` 块确保 loading = false，Dialog 不卡死 |

---

## 9. 性能与可访问性

| 项 | 规范 |
|----|------|
| 浏览器 | Chrome 100+、Edge 100+、Safari 15+ |
| Drawer 骨架屏 | 加载超 300ms 展示 `el-skeleton` |
| Dialog Tab 导航 | 所有表单字段支持 Tab 键导航（WCAG AA） |
| 国际化 | 所有文案使用 `$t()` 包裹，支持中英双语 |

---

## 10. 埋点

| 事件 | 触发点 |
|------|-------|
| `ext_approval_detail_open` | 点击 [Detail] 按钮 |
| `ext_approval_download` | 点击 [Download Original File] |
| `ext_approval_mark_dialog_open` | 点击 [Mark Result] |
| `ext_approval_mark_success` | 标记成功（携带 statusOfApproval） |
| `ext_approval_mark_fail` | 标记失败（携带错误码） |
| `ext_approval_qr_fail` | QR 生成失败 |

---

## 11. AC 覆盖矩阵

| AC ID | 描述简述 | 实现位置 |
|-------|---------|---------|
| AC-007B-001 | Todo 列表仅展示当前 DC 自己名下待办 | `TodoPanel.vue`（后端已过滤，前端不额外控件） |
| AC-007B-002 | 点击 Detail 打开 Drawer | `TodoPanel.vue` → `ExternalApprovalDetailDrawer.vue` |
| AC-007B-003 | Drawer 展示完整字段，不含 Drawing Code/Name | `ExternalApprovalDetailDrawer.vue` FieldList |
| AC-007B-004 | 下载原始文件，文件名格式正确 | `handleDownload()` |
| AC-007B-005 | 非项目 DC 无法下载（403） | 后端校验，前端错误处理 |
| AC-007B-006 | 报审三要素始终必填 | `rules.submissionRefNo/Subject/Description` |
| AC-007B-007 | Status A/B/D 时签字版/凭证/日期必填 | `rules.signedFile/evidenceFile/externalApprovalDate`（条件校验） |
| AC-007B-007B | Status E 时 Others Reason 必填 | `rules.othersReason`（条件校验） |
| AC-007B-008 | Status A/B/D 成功路径 | `handleConfirm()` → success Toast + emit |
| AC-007B-009 | 管理员 Assign 后 SE 收到通知 | 前端无感知（后端 + REQ-003D-pc） |
| AC-007B-010 | QR 生成失败回滚 | `catch(QR_GENERATION_FAILED)` |
| AC-007B-011 | Status C 退回必填校验 | 同 AC-007B-006（报审三要素始终必填） |
| AC-007B-012 | Status C 成功路径 | `handleConfirm()` → success Toast |
| AC-007B-013 | 其他 DC Todo 自动关闭 | 后端处理，前端在 `emit('success')` 后刷新 Todo 列表 |
| AC-007B-014 | 非项目 DC 不显示 [Mark Result] | `v-if="canMarkExternal"` + 后端 403 |
| AC-007B-015 | Dialog loading 防重复提交 | `:disabled="loading"` + `finally { loading=false }` |

---

## 12. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.2.2 | 2026-05-26 | agent | 基于 REQ-007B-pc@0.2.2 首次生成：新增 F-001 外部审批卡片、F-002 详情 Drawer（含 inline 打开 + 强制下载）、F-003 MarkExternalApprovalDialog（Status A–E 下拉 + 报审三要素始终必填 + 条件字段显隐） |
