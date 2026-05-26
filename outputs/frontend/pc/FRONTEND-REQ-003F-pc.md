---
doc_type: frontend_spec
req_id: REQ-003F-pc
version: 0.5.0
status: draft
generated_from: REQ-003F-pc.md@0.5.0
ui_spec_ref: UI-REQ-003F-pc.md
generated_at: 2026-05-26
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明 — PC 端上传新版本

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-003F-pc.md](../../../requirements/pc/REQ-003F-pc.md) @ v0.5.0
> **UI 设计参考**: [UI-REQ-003F-pc.md](../../ui/pc/UI-REQ-003F-pc.md)
> **关联前端文档**:
> - [FRONTEND-REQ-003A-pc.md](./FRONTEND-REQ-003A-pc.md)（入口列表页，[Upload New Version] 按钮在 ActionsCell 中）
> - [FRONTEND-REQ-003E-pc.md](./FRONTEND-REQ-003E-pc.md)（AI 识别结果列表交互规范参考）
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-05-26

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003F-pc @ v0.5.0 |
| UI 设计 | UI-REQ-003F-pc.md |
| 覆盖 Story | US-003F-001 |
| 覆盖 AC | AC-003F-001 ~ AC-003F-013 |

---

## 1. 功能概述

本需求实现"上传新版本"弹窗，从图纸列表 Actions 列 [Upload New Version] 按钮触发，全流程如下：

| 阶段 | 状态 | 说明 |
|-----|------|------|
| 1. 打开弹窗 | 展示只读信息 | 副标题 `{drawingCode} — {drawingName}`；Current Version / Category / Description 只读 |
| 2. 文件上传 | 上传 PDF | 仅 PDF，≤ 50MB；上传后自动触发 AI 识别 |
| 3. AI 识别中 | Loading | `"Analysing drawing pages…"`；[Submit] 置灰 |
| 4a. AI 识别 DONE | 展示识别结果列表 | 页码 / 缩略图 / Drawing Code / Drawing Name，只读核对 |
| 4b. AI 识别 FAILED | 降级提示 | 橙色横幅；[Re-upload] / [Proceed Anyway] |
| 5. 填写表单 | Version Note + Internal Approver | Internal Approver 必填 |
| 6. 提交 | 进度条 | 防重复提交；成功后弹窗关闭，列表刷新 |

---

## 2. 技术栈

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 |
| UI 库 | Element UI |
| HTTP 客户端 | Axios（封装在 `src/api/`） |
| 状态管理 | Vuex |
| 轮询 / 推送 | 前端短轮询（`setInterval` 每 2s 查询 AI 识别任务状态，最长 60s） |

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── drawing/
│       └── components/
│           └── UploadNewVersionDialog.vue     # 上传新版本弹窗（主组件）
│               ├── AiResultList.vue           # AI 识别结果列表（只读，复用自 REQ-003E）
│               ├── AiFailureBanner.vue        # AI 识别失败降级横幅
│               └── UploadProgressBar.vue      # 上传进度条
├── api/
│   └── drawing.js                            # 新增：上传新版本相关接口
└── store/
    └── modules/
        └── uploadNewVersion.js               # 弹窗状态机、AI 轮询逻辑
```

> `AiResultList.vue` 与 REQ-003E 中同名组件完全相同，提取为共享组件放在 `src/components/drawing/AiResultList.vue`，两处复用。

---

## 4. 组件结构

```
<UploadNewVersionDialog>
├── el-dialog（宽 600px）
│   ├── 副标题：{drawingCode} — {drawingName}
│   ├── 只读信息区
│   │   ├── Current Version（只读）
│   │   ├── Category（只读）
│   │   └── Description（只读，空则 —）
│   ├── Drawing File 上传区（el-upload，仅 PDF ≤ 50MB）
│   ├── [AI 识别中] Loading 区（v-if: phase==='recognizing'）
│   ├── <AiResultList>（v-if: phase==='done'，只读）
│   ├── <AiFailureBanner>（v-if: phase==='failed'）
│   │   ├── [Re-upload] 按钮
│   │   └── [Proceed Anyway] 按钮
│   ├── Version Note（el-input textarea，选填）
│   ├── Internal Approver（el-select，必填）
│   ├── <UploadProgressBar>（v-if: submitting）
│   └── 底部按钮：[Cancel] / [Submit]
```

---

## 5. 组件详细说明

### 5.1 `UploadNewVersionDialog.vue`（主组件）

**关联 AC**: AC-003F-001 ~ AC-003F-013

**Props**:
```ts
interface UploadNewVersionDialogProps {
  visible:       boolean
  drawingId:     string
  drawingCode:   string
  drawingName:   string
  currentVersion: string       // e.g. "V2"
  category:      string
  description:   string | null
}
```

**弹窗标题与副标题**（AC-003F-012）：
```vue
<el-dialog title="Upload New Version" :visible.sync="visible" width="600px" :close-on-click-modal="false">
  <p class="dialog-subtitle">{{ drawingCode }} — {{ drawingName }}</p>
  ...
</el-dialog>
```

#### 5.1.1 只读字段区（AC-003F-012）

```vue
<el-form label-position="top">
  <el-form-item label="Current Version">
    <span class="readonly-field">{{ currentVersion }}</span>
  </el-form-item>
  <el-form-item label="Category">
    <span class="readonly-field">{{ category }}</span>
  </el-form-item>
  <el-form-item label="Description">
    <span class="readonly-field">{{ description || '—' }}</span>
  </el-form-item>
</el-form>
```

所有只读字段以灰色样式渲染，无任何输入入口（AC-003F-012）。

#### 5.1.2 文件上传区（AC-003F-002 / AC-003F-008 / AC-003F-009）

```vue
<el-upload
  ref="uploader"
  action="#"
  :http-request="doUpload"
  :before-upload="validateFile"
  :show-file-list="true"
  :limit="1"
  accept=".pdf"
  drag
>
  <i class="el-icon-upload"></i>
  <div>Drop PDF here or <em>click to select</em></div>
  <div slot="tip" class="upload-tip">PDF only · Max 50MB</div>
</el-upload>
```

**文件校验**（AC-003F-008 / AC-003F-009）：
```javascript
validateFile(file) {
  // AC-003F-008: 仅 PDF
  if (file.type !== 'application/pdf' && !file.name.endsWith('.pdf')) {
    this.$message.error('Only PDF format is supported for version upload.')
    return false
  }
  // AC-003F-009: ≤ 50MB
  if (file.size > 50 * 1024 * 1024) {
    this.$message.error('File size must not exceed 50MB.')
    return false
  }
  return true
}
```

**上传成功 → 触发 AI 识别**（AC-003F-002）：
```javascript
async doUpload({ file }) {
  this.phase = 'uploading'
  const formData = new FormData()
  formData.append('file', file)
  formData.append('drawingId', this.drawingId)

  try {
    const res = await drawingApi.uploadVersionFile(formData, (progress) => {
      this.uploadProgress = progress
    })
    this.versionFileId = res.data.versionFileId
    this.aiJobId = res.data.aiJobId
    this.phase = 'recognizing'   // 触发 AI 识别中状态
    this.startAiPolling()
  } catch (e) {
    this.phase = 'idle'
    this.$message.error('Upload failed. Please try again.')
  }
}
```

#### 5.1.3 AI 识别轮询（AC-003F-002 / AC-003F-003 / AC-003F-005）

```javascript
startAiPolling() {
  let elapsed = 0
  const INTERVAL = 2000
  const TIMEOUT = 60000   // OQ-001: 与 AI 团队确认后更新

  this.pollingTimer = setInterval(async () => {
    elapsed += INTERVAL
    if (elapsed >= TIMEOUT) {
      clearInterval(this.pollingTimer)
      this.phase = 'failed'    // 超时视为 FAILED
      return
    }

    const res = await drawingApi.getAiJobStatus(this.aiJobId)
    if (res.data.status === 'DONE') {
      clearInterval(this.pollingTimer)
      this.aiPages = res.data.pages    // 识别结果页列表
      this.phase = 'done'
    } else if (res.data.status === 'FAILED') {
      clearInterval(this.pollingTimer)
      this.phase = 'failed'
    }
  }, INTERVAL)
},

beforeDestroy() {
  clearInterval(this.pollingTimer)   // 弹窗销毁时清理定时器
}
```

**phase 状态机**：

```
idle → uploading → recognizing → done
                             ↘ failed → idle（Re-upload）
                                     → proceedAnyway（→ 可提交）
```

#### 5.1.4 AI 识别中 Loading 区（AC-003F-002）

```vue
<div v-if="phase === 'recognizing'" class="ai-loading">
  <i class="el-icon-loading"></i>
  <span>Analysing drawing pages…</span>
</div>
```

#### 5.1.5 [Submit] 按钮禁用逻辑（AC-003F-002 / AC-003F-003 / AC-003F-005 / AC-003F-010）

```javascript
computed: {
  submitDisabled() {
    if (this.submitting) return true                         // 提交中
    if (['idle','uploading','recognizing'].includes(this.phase)) return true  // 文件未上传完或识别中
    if (this.phase === 'failed' && !this.proceedAnyway) return true          // 识别失败且未确认继续
    if (!this.internalApproverId) return true                // 必填项未填
    return false
  }
}
```

#### 5.1.6 `AiFailureBanner.vue` 降级横幅（AC-003F-005 / AC-003F-006 / AC-003F-007）

```vue
<!-- phase === 'failed' 时展示 -->
<div class="ai-failure-banner">
  <el-alert type="warning" :closable="false">
    Unable to analyse the drawing automatically.
    Please re-upload the file or proceed to submit.
  </el-alert>
  <el-button @click="reUpload">Re-upload</el-button>          <!-- AC-003F-006 -->
  <el-button @click="proceedAnyway = true">Proceed Anyway</el-button>  <!-- AC-003F-007 -->
</div>
```

**[Re-upload] 逻辑**（AC-003F-006）：
```javascript
reUpload() {
  this.phase = 'idle'
  this.aiPages = []
  this.aiJobId = null
  this.versionFileId = null
  this.proceedAnyway = false
  this.$refs.uploader.clearFiles()   // 清空 el-upload 已选文件
}
```

#### 5.1.7 表单剩余字段与提交（AC-003F-001 / AC-003F-010 / AC-003F-013）

```vue
<el-form-item label="Version Note">
  <el-input v-model="form.versionNote" type="textarea" :rows="3"
            placeholder="Describe the changes in this version… (optional)" />
</el-form-item>

<el-form-item label="Internal Approver" required>
  <el-select v-model="form.internalApproverId" placeholder="Select approver" filterable>
    <el-option v-for="u in approverList" :key="u.id" :label="u.name" :value="u.id" />
  </el-select>
  <div v-if="showApproverError" class="field-error">Please select an internal approver.</div>
</el-form-item>
```

**提交逻辑**（AC-003F-001 / AC-003F-010）：
```javascript
async handleSubmit() {
  // 前端校验
  if (!this.form.internalApproverId) {
    this.showApproverError = true
    return
  }

  this.submitting = true
  this.uploadProgress = 0

  try {
    await drawingApi.submitNewVersion({
      drawingId:          this.drawingId,
      versionFileId:      this.versionFileId,
      versionNote:        this.form.versionNote,
      internalApproverId: this.form.internalApproverId,
      aiJobId:            this.aiJobId,
      skipAiResult:       this.proceedAnyway,
    })
    this.$emit('success')
    this.$emit('update:visible', false)
    this.$message.success('New version uploaded. Pending internal approval.')
    // 通知列表页刷新（AC-003F-001）
    this.$emit('refresh-list')
  } catch (err) {
    if (err.response?.status === 409) {
      this.$message.error('This drawing is now under review. Please refresh and try again.')
    } else {
      this.$message.error('Upload failed. Please try again.')
    }
  } finally {
    this.submitting = false
  }
}
```

---

### 5.2 `AiResultList.vue`（AI 识别结果只读列表）

**关联 AC**: AC-003F-003 / AC-003F-004

**与 REQ-003E 共用组件**，此处仅说明差异：

- 本场景为只读核对，列表**不支持行内编辑**（REQ-003E 中可能允许修改，本流程不允许）
- 顶部汇总文案：`"X pages detected. Please review the drawing information below before submitting."`
- 有未识别字段时追加橙色提示（AC-003F-004）

**Props**:
```ts
interface AiResultListProps {
  pages: AiPage[]   // AI 识别结果页列表
  readonly: true    // 始终 true（上传新版本场景）
}

interface AiPage {
  pageNo:       number
  thumbnailUrl: string
  drawingCode:  string | null
  drawingName:  string | null
}
```

**列表渲染**（AC-003F-003 / AC-003F-004）：

```vue
<div class="ai-result-list">
  <!-- 顶部汇总 -->
  <p class="summary">
    {{ pages.length }} pages detected. Please review the drawing information below before submitting.
  </p>
  <el-alert v-if="hasUnrecognisedPages" type="warning" :closable="false">
    Some pages could not be fully analysed. Please verify the drawing information below.
  </el-alert>

  <!-- 滚动容器，最大高度 240px -->
  <div class="result-scroll" style="max-height:240px; overflow-y:auto">
    <el-table :data="pages" size="small" :show-header="true">
      <el-table-column label="Page" width="60">
        <template slot-scope="{ row }">Page {{ row.pageNo }}</template>
      </el-table-column>
      <el-table-column label="Preview" width="80">
        <template slot-scope="{ row }">
          <img :src="row.thumbnailUrl" style="width:60px;height:60px;object-fit:contain;cursor:pointer"
               @click="previewThumbnail(row.thumbnailUrl)" />
        </template>
      </el-table-column>
      <el-table-column label="Drawing Code" min-width="140">
        <template slot-scope="{ row }">
          <!-- AC-003F-004: 未识别到时显示 — 并橙色 ⚠ -->
          <span v-if="row.drawingCode">{{ row.drawingCode }}</span>
          <span v-else class="unrecognised">⚠ —</span>
        </template>
      </el-table-column>
      <el-table-column label="Drawing Name" min-width="160">
        <template slot-scope="{ row }">
          <span v-if="row.drawingName">{{ row.drawingName }}</span>
          <span v-else class="unrecognised">⚠ —</span>
        </template>
      </el-table-column>
    </el-table>
  </div>
</div>
```

```javascript
computed: {
  hasUnrecognisedPages() {
    return this.pages.some(p => !p.drawingCode || !p.drawingName)
  }
}
```

---

### 5.3 `UploadProgressBar.vue`（AC-003F-010）

```vue
<!-- submitting === true 时显示 -->
<div class="upload-progress">
  <el-progress :percentage="uploadProgress" status="active" />
  <span>Uploading… {{ uploadProgress }}%</span>
</div>
```

进度值来自 Axios `onUploadProgress` 回调：
```javascript
drawingApi.uploadVersionFile(formData, (e) => {
  this.uploadProgress = Math.round((e.loaded / e.total) * 100)
})
```

---

## 6. 状态管理

### 6.1 弹窗内部状态（组件 data，不入 Vuex）

```javascript
data() {
  return {
    phase: 'idle',           // idle | uploading | recognizing | done | failed
    proceedAnyway: false,    // 用户选择"Proceed Anyway"
    versionFileId: null,     // 服务端返回的临时文件 ID
    aiJobId: null,           // AI 识别任务 ID
    aiPages: [],             // AI 识别结果页列表
    uploadProgress: 0,       // 0~100
    submitting: false,
    pollingTimer: null,
    showApproverError: false,
    form: {
      versionNote: '',
      internalApproverId: null,
    }
  }
}
```

### 6.2 弹窗关闭时清理

```javascript
watch: {
  visible(val) {
    if (!val) this.resetDialog()
  }
},
methods: {
  resetDialog() {
    clearInterval(this.pollingTimer)
    this.phase = 'idle'
    this.proceedAnyway = false
    this.versionFileId = null
    this.aiJobId = null
    this.aiPages = []
    this.uploadProgress = 0
    this.submitting = false
    this.form = { versionNote: '', internalApproverId: null }
    this.$refs.uploader?.clearFiles()
  }
}
```

---

## 7. API 接入

### 7.1 接口清单

| API | 场景 | 说明 |
|-----|------|------|
| `POST /drawing/{drawingId}/version/file` | 文件上传 | multipart/form-data，返回 `versionFileId` 和 `aiJobId` |
| `GET /ai/job/{aiJobId}/status` | AI 识别状态轮询 | 返回 `{ status: 'PROCESSING'\|'DONE'\|'FAILED', pages: [...] }` |
| `POST /drawing/{drawingId}/version` | 提交新版本 | 创建 DrawingVersion，触发内部审批 Todo |
| `GET /project/{projectId}/approvers` | 获取内部审批人列表 | 返回有 `drawing:approve` 权限的用户列表 |

### 7.2 接口封装

```javascript
// drawing.js 新增
export const drawingApi = {
  // 上传文件（含进度回调）
  uploadVersionFile: (formData, onProgress) =>
    axios.post(`/drawing/${formData.get('drawingId')}/version/file`, formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
      onUploadProgress: (e) => onProgress && onProgress(Math.round(e.loaded / e.total * 100)),
    }),

  // AI 识别状态轮询
  getAiJobStatus: (aiJobId) =>
    axios.get(`/ai/job/${aiJobId}/status`),

  // 提交新版本
  submitNewVersion: (payload) =>
    axios.post(`/drawing/${payload.drawingId}/version`, payload),

  // 内部审批人列表
  getApprovers: (projectId) =>
    axios.get(`/project/${projectId}/approvers`),
}
```

### 7.3 错误处理

| HTTP 状态 / 场景 | 用户可见文案 | 处理位置 |
|----------------|-----------|---------|
| 上传文件失败（网络中断） | `"Upload failed. Please try again."` | `doUpload` catch |
| AI 识别超时（60s） | 进入 failed 状态，展示 AiFailureBanner | `startAiPolling` timeout |
| 提交 409（状态冲突） | `"This drawing is now under review. Please refresh and try again."` | `handleSubmit` catch |
| 提交 500 | `"Upload failed. Please try again."` | `handleSubmit` catch |
| 文件非 PDF | `"Only PDF format is supported for version upload."` | `validateFile` |
| 文件超 50MB | `"File size must not exceed 50MB."` | `validateFile` |

---

## 8. 埋点

| 事件 | 触发点 |
|------|-------|
| `upload_new_version_open` | 点击 [Upload New Version]（从列表页） |
| `upload_new_version_file_selected` | 选择合法文件 |
| `upload_new_version_ai_start` | 文件上传成功，开始 AI 识别 |
| `upload_new_version_ai_done` | AI 识别 DONE |
| `upload_new_version_ai_failed` | AI 识别 FAILED |
| `upload_new_version_proceed_anyway` | 点击 [Proceed Anyway] |
| `upload_new_version_submit_success` | 提交成功 |
| `upload_new_version_submit_fail` | 提交失败（携带 error_type） |

---

## 9. 性能与可访问性

| 项 | 规范 |
|----|------|
| 识别结果列表渲染（100 页以内） | ≤ 1s |
| AI 轮询间隔 | 2s，最长 60s 后超时进入 failed |
| 弹窗键盘导航 | Tab 键在所有输入项间切换；Esc 关闭弹窗 |
| 国际化 | 所有文案使用 `$t()` 包裹 |
| 浏览器兼容 | Chrome 100+、Edge 100+、Safari 15+ |

---

## 10. AC 覆盖矩阵

| AC ID | 描述简述 | 实现位置 |
|-------|---------|---------|
| AC-003F-001 | 成功路径：弹窗关闭，列表刷新，状态变 PENDING_INTERNAL，Snackbar 提示，主记录不变 | `handleSubmit` 成功后 `$emit('refresh-list')` + `$message.success` |
| AC-003F-002 | 文件上传完成后显示 Loading + "Analysing drawing pages…"；[Submit] 置灰 | `phase==='recognizing'` Loading 区 + `submitDisabled` computed |
| AC-003F-003 | AI DONE 后展示识别结果列表，顶部汇总文案，[Submit] 解锁 | `AiResultList`（`phase==='done'`）+ `submitDisabled` |
| AC-003F-004 | 有未识别字段时显示 ⚠ — 和橙色追加提示 | `AiResultList` `hasUnrecognisedPages` computed + `el-alert` |
| AC-003F-005 | AI FAILED 时显示橙色横幅，提供 [Re-upload] / [Proceed Anyway]，[Submit] 保持置灰 | `AiFailureBanner`（`phase==='failed'`）+ `submitDisabled` |
| AC-003F-006 | [Re-upload] 清空文件，重新触发文件选择 | `reUpload()` → `phase='idle'` + `$refs.uploader.clearFiles()` |
| AC-003F-007 | [Proceed Anyway] 后 [Submit] 解锁，提交逻辑与正常路径一致 | `proceedAnyway=true` → `submitDisabled` 解锁 |
| AC-003F-008 | 非 PDF 文件被拒绝，提示文案 | `validateFile` type/extension 检查 |
| AC-003F-009 | > 50MB 文件被拒绝，提示文案 | `validateFile` size 检查 |
| AC-003F-010 | 提交后显示进度条，[Submit] 禁用，完成后消失 | `submitting=true` + `UploadProgressBar` |
| AC-003F-011 | PENDING_INTERNAL / PENDING_EXTERNAL 时 [Upload New Version] 置灰 | `ActionsCell.vue`（见 FRONTEND-REQ-003A-pc §5.4） |
| AC-003F-012 | 弹窗副标题 `drawingCode — drawingName`；只读字段灰色无编辑入口；可编辑仅 Drawing File / Version Note / Internal Approver | `dialog-subtitle` + 只读 `span` 渲染 |
| AC-003F-013 | 提交成功后内部审批人 Todo 出现任务（后端触发，前端无需额外处理） | `submitNewVersion` API 调用后后端异步推送 |

---

## 11. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.5.0 | 2026-05-26 | agent | 基于 REQ-003F-pc@0.5.0 首次生成。涵盖 AI 识别完整流程（上传→轮询→DONE/FAILED）、只读核对列表（共用 AiResultList.vue）、降级横幅（Re-upload/Proceed Anyway）、表单校验（PDF/50MB/必填）、进度条防重复提交、9 项埋点、AC-003F-001~013 全覆盖 |
