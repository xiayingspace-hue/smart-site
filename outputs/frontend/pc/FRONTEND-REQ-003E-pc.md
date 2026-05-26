---
doc_type: frontend_spec
req_id: REQ-003E-pc
version: 0.2.0
status: draft
generated_from: REQ-003E-pc.md@0.3.1
data_contract_ref: data-contract.md@0.1.0
ui_spec_ref: UI-REQ-003E-pc.md@0.1.0
generated_at: 2026-05-24
updated_at: 2026-05-25
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明：PC 端 — 图纸上传 AI 自动识别页信息

> RUNTIME LIBRARY：项目实现基于 **Element UI（Vue 2）**——所有实现必须使用 Element 组件或等价适配层。

> **本文档供前端开发工程师及其 agent 使用**。
>
> ⚠️ **重要约定**：
> - 所有 API 字段定义必须**引用** data-contract.md，本文档不重复定义。
> - 所有 UI 元素引用 UI-REQ-003E-pc.md。
> - 代码注释中必须标注覆盖的 AC ID（如 `// AC-003E-001`）。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003E-pc @ v0.3.1 |
| 数据契约 | data-contract.md @ v0.1.0 |
| UI 设计 | UI-REQ-003E-pc.md @ v0.1.0 |
| 覆盖 Story | US-003E-001、US-003E-002 |
| 覆盖 AC | AC-003E-001 ~ AC-003E-013 |

---

## 1. 功能概述

在 REQ-003A 新建图纸弹窗的基础上，增加 AI 识别功能：用户上传 PDF 文件后，前端自动触发识别流程，通过轮询获取识别状态，识别完成后渲染只读结果列表，并将 AI 建议值自动填入 Drawing Code / Drawing Name 输入框；识别失败时展示降级提示横幅，允许手动填写或重新上传。最终仍调用单条创建接口，与 REQ-003A 保持一致。

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

- 框架：Vue 2
- 语言：JavaScript（ES6+）
- UI 库：Element UI
- 样式：SCSS
- 构建：Webpack
- HTTP：Axios（项目封装层）
- 状态管理：Vuex（如有全局状态）

### 2.2 本需求新增依赖

| 包 | 版本 | 用途 | 评估 |
|---|------|------|------|
| 无新增 | — | 文件上传与轮询均用现有 Axios + Element UI 能力 | ✅ |

---

## 3. 路由设计

本需求不引入新路由，弹窗在 Drawing Masterlist 页（已有路由）内以 Modal 形式展示。

| 路径 | 组件 | 权限 | 关联 Story |
|-----|------|-----|-----------|
| `/drawing/masterlist`（已有） | `DrawingMasterlist` | `drawing:create` | US-003E-001 |

**URL state 同步**：无需将弹窗状态写入 URL。

---

## 4. 组件结构

### 4.1 组件树

```
<DrawingMasterlist>                     # 已有页面
└── <UploadDrawingDialog>               # 新建/改造：Upload New Drawing 弹窗
    ├── <el-dialog>                     # Element UI Dialog 容器
    ├── <el-form ref="uploadForm">      # 表单校验容器
    │   ├── <el-form-item> Description  # 选填文本
    │   ├── <el-form-item> Category     # 必填下拉
    │   ├── <DrawingFileUploader>       # 文件上传区（新组件）
    │   ├── <AiRecognitionPanel>        # AI 识别结果面板（新组件）
    │   │   ├── <AiLoadingState>        # 识别中 Loading 子视图
    │   │   ├── <AiResultList>          # 识别完成结果列表子视图（新组件）
    │   │   │   └── <ThumbnailPreview>  # 缩略图点击大图（新组件）
    │   │   └── <AiFailureBanner>       # 识别失败降级横幅子视图
    │   ├── <el-form-item> Drawing Code # 必填，AI 自动填入
    │   ├── <el-form-item> Drawing Name # 必填，AI 自动填入
    │   ├── <el-form-item> Version Note # 选填
    │   └── <el-form-item> Internal Approver # 必填下拉
    └── <DialogFooter>                  # Cancel / Submit 按钮
```

### 4.2 关键组件说明

#### `<UploadDrawingDialog>`

**关联 AC**：AC-003E-001 ~ AC-003E-013

**Props**：
```js
// Props
{
  visible: Boolean,       // 弹窗是否显示
  projectId: String,      // 当前项目 ID
}
// Emits
// 'close'   — 用户关闭弹窗
// 'success' — 创建成功，父组件刷新列表
```

**内部 data**：
```js
data() {
  return {
    form: {
      description: '',
      category: null,
      drawingCode: '',
      drawingName: '',
      versionNote: '',
      internalApproverId: null,
    },
    uploadedFileUrl: null,       // 上传成功后的文件 URL
    recognitionJobId: null,      // AIRecognitionJob ID
    recognitionStatus: null,     // 'PENDING' | 'PROCESSING' | 'DONE' | 'FAILED'
    recognitionPages: [],        // 识别结果页列表（来自 API）
    pollingTimer: null,          // 轮询定时器 ID
    isSubmitting: false,
  }
}
```

**职责**：
- 管理整体弹窗状态机（未上传 → 识别中 → 完成 / 失败 → 提交）
- 编排子组件间数据流
- 处理文件上传、轮询、提交等所有 API 调用

**不该做**：
- 不直接渲染识别结果行（委托给 `<AiResultList>`）
- 不包含文件选择逻辑（委托给 `<DrawingFileUploader>`）

---

#### `<DrawingFileUploader>`

**关联 AC**：AC-003E-010、AC-003E-011

**Props**：
```js
{
  disabled: Boolean,   // 识别进行中时禁用（防止重新选择）
}
```

**Emits**：
- `'file-selected'` — 用户选择了文件（payload: `File` 对象）
- `'upload-progress'` — 上传进度（payload: `{ percent: Number }`）
- `'upload-success'` — 上传完成（payload: `{ fileUrl: String }`）
- `'upload-error'` — 上传失败（payload: `{ message: String }`）

**职责**：
- 前端校验文件类型（仅 PDF）// AC-003E-010
- 前端校验文件大小（≤ 50MB）// AC-003E-011
- 调用文件上传接口，展示进度条

---

#### `<AiRecognitionPanel>`

**关联 AC**：AC-003E-001、AC-003E-002、AC-003E-004、AC-003E-006

**Props**：
```js
{
  status: String,    // 'IDLE' | 'PENDING' | 'PROCESSING' | 'DONE' | 'FAILED'
  pages: Array,      // 识别结果页列表
}
```

**职责**：
- 根据 `status` 切换渲染：`IDLE` 时不显示；`PENDING/PROCESSING` 时显示 Loading；`DONE` 时显示结果列表；`FAILED` 时显示降级横幅。

---

#### `<AiResultList>`

**关联 AC**：AC-003E-002、AC-003E-004

**Props**：
```js
{
  pages: Array,  // [{ pageNo, thumbnailUrl, drawingNo, drawingName, confidence }]
}
```

**职责**：
- 渲染只读结果表格
- 未识别字段显示 `—` + 橙色 ⚠ 图标
- 顶部汇总行（已识别 X 页 / 部分未识别橙色提示）

**不该做**：
- 不允许行内编辑（列表完全只读）
- 不影响 Drawing Code / Name 输入框（仅由父组件通过 `watch` 填入）

---

#### `<ThumbnailPreview>`

**关联 AC**：AC-003E-002

**Props**：
```js
{
  src: String,     // 缩略图 URL
  pageNo: Number,  // 用于 alt 文字
}
```

**职责**：
- 展示 60×60px 缩略图
- 点击后以 `el-dialog` 展示大图（full-size preview）

---

#### `<AiFailureBanner>`

**关联 AC**：AC-003E-006、AC-003E-007、AC-003E-008

**Props**：
```js
{
  // 无外部 Props；展示内容固定
}
```

**Emits**：
- `'reupload'` — 用户点击 [Re-upload] 按钮，父组件调用 `handleReupload()`

**职责**：
- 渲染橙色降级提示横幅：`"Unable to analyse the drawing automatically. Please enter the drawing information manually, or re-upload the file."`
- 提供 [Re-upload] 按钮，点击后 `$emit('reupload')`

**不该做**：
- 不直接操作 Drawing Code / Name 输入框（由父组件 `handleReupload()` 统一重置）

---

## 5. 状态管理

### 5.1 状态分层

| 状态类型 | 存放位置 | 例子 |
|---------|---------|------|
| 表单数据 + 识别状态 | `<UploadDrawingDialog>` 组件局部 data | `form`, `recognitionStatus`, `recognitionPages` |
| 图纸列表缓存 | 父组件（DrawingMasterlist）局部 | 创建成功后调用父组件刷新方法 |
| 全局 UI | 无 | — |

### 5.2 识别状态流转

```
IDLE
  → [用户选择文件并上传成功] → PENDING
  → [轮询返回 PROCESSING]   → PROCESSING
  → [轮询返回 DONE]         → DONE   // AC-003E-002
  → [轮询返回 FAILED 或超时] → FAILED // AC-003E-006
```

### 5.3 轮询策略

```js
// AC-003E-001: 上传完成后启动轮询
startPolling(jobId) {
  this.pollingTimer = setInterval(async () => {
    const result = await api.getRecognitionJobStatus(jobId)
    this.recognitionStatus = result.status
    if (result.status === 'DONE') {
      this.recognitionPages = result.pages
      this.autofillDrawingInfo(result.pages) // AC-003E-002
      this.stopPolling()
    } else if (result.status === 'FAILED') {
      this.stopPolling() // AC-003E-006
    }
  }, 2000) // 每 2s 轮询一次
},

stopPolling() {
  clearInterval(this.pollingTimer)
  this.pollingTimer = null
},

// 组件销毁时清理
beforeDestroy() {
  this.stopPolling()
}
```

> 可选优化：若后端支持 WebSocket，可替换轮询为 WebSocket 推送（接口约定见 data-contract.md）。

---

## 6. API 接入

> 所有接口字段定义参见 `data-contract.md`，本节只列前端调用方式。

### 6.1 调用清单

| API ID | 接口描述 | 在哪些组件中调用 | 缓存策略 |
|--------|---------|---------------|---------|
| API-FILE-UPLOAD | 上传 PDF 文件，触发识别任务创建 | `<DrawingFileUploader>` | 不缓存 |
| API-JOB-STATUS | 轮询识别任务状态及结果 | `<UploadDrawingDialog>` | 不缓存（实时轮询）|
| API-DRAWING-CREATE | 创建 Drawing 记录（复用 REQ-003A 接口）| `<UploadDrawingDialog>` | 创建成功后失效列表缓存 |
| API-APPROVER-LIST | 获取可选内部审批人列表 | `<UploadDrawingDialog>` | 短期缓存（5min）|

### 6.2 错误处理

```js
// 全局错误映射（用户可见文案）
const errorMessages = {
  'FILE_TYPE_NOT_SUPPORTED': 'AI recognition only supports PDF format.',
  'FILE_SIZE_EXCEEDED': 'File size exceeds the 50MB limit.',
  'RECOGNITION_FAILED': 'Unable to analyse the drawing automatically.',
  'DRAWING_CODE_DUPLICATE': 'Drawing Code already exists.',
  'NETWORK_ERROR': 'Network error. Please check your connection and try again.',
}
```

---

## 7. 关键交互逻辑

### 7.1 文件校验（前端）

```js
// AC-003E-010, AC-003E-011
validateFile(file) {
  if (file.type !== 'application/pdf') {
    this.$message.error('AI recognition only supports PDF format.')
    return false
  }
  if (file.size > 50 * 1024 * 1024) {
    this.$message.error('File size exceeds the 50MB limit.')
    return false
  }
  return true
}
```

### 7.2 AI 自动填入 Drawing Code / Name

```js
// AC-003E-002: 识别完成后自动填入
autofillDrawingInfo(pages) {
  // TODO(OQ-002): 当前临时策略取第一页，待 PM 与业务确认后更新
  const firstPage = pages[0]
  if (firstPage) {
    this.form.drawingCode = firstPage.drawingNo || ''
    this.form.drawingName = firstPage.drawingName || ''
  }
}
```

> ⚠️ OQ-002 未解：多页不同 Drawing No/Name 时的填入规则待 PM 确认，当前使用第一页作为临时策略。

### 7.3 Drawing Code / Name 输入框禁用控制

```js
// AC-003E-009
computed: {
  isInputDisabled() {
    return ['PENDING', 'PROCESSING'].includes(this.recognitionStatus)
  },
  isSubmitDisabled() {
    return this.isInputDisabled || this.isSubmitting
  }
}
```

### 7.4 Re-upload 逻辑

```js
// AC-003E-008
// 由 <AiFailureBanner> emit 'reupload' 事件触发
handleReupload() {
  this.stopPolling()
  this.uploadedFileUrl = null
  this.recognitionJobId = null
  this.recognitionStatus = 'IDLE'
  this.recognitionPages = []
  this.form.drawingCode = ''
  this.form.drawingName = ''
  this.$refs.fileUploader.reset() // 触发文件选择框重置，重新进入上传待机状态
}
```

### 7.5 弹窗关闭时资源清理

```js
// 弹窗关闭时必须停止轮询，防止内存泄漏 / 后台静默重置表单
handleClose() {
  this.stopPolling()
  this.$refs.uploadForm && this.$refs.uploadForm.resetFields()
  this.uploadedFileUrl = null
  this.recognitionJobId = null
  this.recognitionStatus = null
  this.recognitionPages = []
  this.isSubmitting = false
  this.$emit('close')
}
```

### 7.6 表单提交

```js
// AC-003E-005: 校验通过后调用单条创建接口
async handleSubmit() {
  const valid = await this.$refs.uploadForm.validate()
  if (!valid) return
  this.isSubmitting = true
  try {
    await api.createDrawing({
      ...this.form,
      fileUrl: this.uploadedFileUrl,
      aiRecognitionJobId: this.recognitionJobId || null, // AC-003E-012 / AC-003E-013
    })
    this.$message.success('Drawing uploaded successfully. Pending internal approval.')
    this.$emit('success')
    this.handleClose()
  } catch (err) {
    this.handleApiError(err) // 显示行内错误或 Toast
  } finally {
    this.isSubmitting = false
  }
}
```

---

## 8. 性能预算

| 指标 | 目标 | 测量 |
|-----|------|-----|
| 识别结果列表渲染（≤ 100 页） | ≤ 1s | Chrome DevTools Performance |
| 弹窗打开到可交互 | ≤ 200ms | — |

**优化手段**：
- 缩略图懒加载（`v-lazy` 或 `IntersectionObserver`）：列表滚动时按需加载缩略图
- 轮询只在弹窗打开且 `status` 为 PENDING / PROCESSING 时运行，弹窗关闭时立即清除

---

## 9. 弱网与离线策略

| 场景 | 策略 |
|-----|------|
| 文件上传超时 | Axios timeout 60s；超时后显示错误 Toast，文件上传区恢复初始状态，可重试 |
| 轮询中断网 | 连续 3 次轮询请求失败后，停止轮询，将 `recognitionStatus` 置为 `'FAILED'`，展示降级提示 |
| 提交时断网 | 显示 `"Network error. Please check your connection and try again."` Toast，弹窗保留 |

---

## 10. 错误边界与降级

- `<AiRecognitionPanel>` 内渲染错误由 Vue `errorCaptured` 钩子捕获，降级显示降级提示横幅，上报 Sentry
- 缩略图加载失败：显示灰色占位图（图纸图标）

---

## 11. AC 覆盖检查表

| AC ID | 实现位置 | 状态 |
|------|---------|------|
| AC-003E-001 | `<DrawingFileUploader>` 上传成功后触发轮询；`isInputDisabled` 计算属性 | TODO |
| AC-003E-002 | `autofillDrawingInfo()`；`<AiResultList>` 渲染 | TODO |
| AC-003E-003 | `form.drawingCode` 双向绑定，输入框 DONE 后可编辑 | TODO |
| AC-003E-004 | `<AiResultList>` 未识别行渲染 ⚠ 橙色标记 | TODO |
| AC-003E-005 | `handleSubmit()` 单条创建 | TODO |
| AC-003E-006 | 轮询 FAILED 分支，`<AiFailureBanner>` 展示 | TODO |
| AC-003E-007 | FAILED 后 `form.drawingCode/Name` 恢复可编辑，正常走提交流程 | TODO |
| AC-003E-008 | `handleReupload()` | TODO |
| AC-003E-009 | `isInputDisabled` / `isSubmitDisabled` 计算属性 | TODO |
| AC-003E-010 | `validateFile()` 类型校验 | TODO |
| AC-003E-011 | `validateFile()` 大小校验 | TODO |
| AC-003E-012 | `handleSubmit()` 传入 `aiRecognitionJobId` | TODO |
| AC-003E-013 | 降级路径 `recognitionJobId` 为 `null`，创建接口不传 job id | TODO |

---

## 12. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-24 | agent | 初稿，从 REQ-003E-pc@0.3.1 派生 |
| 0.2.0 | 2026-05-25 | agent | 补充 `<AiFailureBanner>` Props/Emits 定义及事件流；补充弹窗关闭时资源清理逻辑 `handleClose()`；`handleReupload()` 注明事件来源；小节编号修正 |
