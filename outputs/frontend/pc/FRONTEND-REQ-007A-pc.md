---
doc_type: frontend_spec
req_id: REQ-007A-pc
version: 0.5.0
status: draft
generated_from: REQ-007A-pc.md@0.5.0
ui_spec_ref: UI-REQ-007A-pc.md
generated_at: 2026-05-25
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明 — PC 端内部审批 Todo、详情查看与原文件下载

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-007A-pc.md](../../../requirements/pc/REQ-007A-pc.md) @ v0.5.0
> **UI 设计参考**: [UI-REQ-007A-pc.md](../../ui/pc/UI-REQ-007A-pc.md)
> **依赖前端文档**: [FRONTEND-REQ-007-pc.md](./FRONTEND-REQ-007-pc.md)（图纸两级审批基础前端）
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-05-25

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007A-pc @ v0.5.0 |
| UI 设计 | UI-REQ-007A-pc.md |
| 覆盖 Story | US-007A-001 / 002 / 003 / 004 / 005 / 006 |
| 覆盖 AC | AC-007A-001 ~ AC-007A-023 |

---

## 1. 功能概述

在 FRONTEND-REQ-007-pc 基础上，本文档覆盖以下内部审批相关前端变更：

| 功能模块 | 变更类型 | 说明 |
|---------|---------|------|
| TodoPanel — 内部审批卡片（F-001） | **变更** | 图标 🔍、标题 "Internal Approval Required"、操作区仅保留 [Detail] 按钮 |
| Todo 列表过滤（F-002） | **规则确认** | 后端已按 assigneeId 过滤，前端无需额外控件 |
| DrawingApprovalDetailDrawer（F-003） | **新增** | 审批详情侧滑弹框：全量字段展示 + 文件名 inline 链接 |
| DownloadOriginalFile 按钮（F-004） | **新增** | 强制下载原始文件，带 loading 防重复点击 |
| ApproveConfirmDialog（F-005） | **新增** | 通过确认对话框，含 DC 已配置前置检查 |
| RejectDialog（F-006） | **新增** | 驳回对话框，Comment 必填校验 |
| NoDcWarningDialog（F-007） | **新增** | 项目无 DC 时警告弹窗，拦截 Approve 操作 |

---

## 2. 技术栈

### 2.1 已有技术栈（继承自 FRONTEND-REQ-003-pc / FRONTEND-REQ-007-pc）

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 |
| UI 库 | Element UI |
| HTTP 客户端 | Axios（封装在 `src/api/`） |
| 状态管理 | Vuex |
| 路由 | Vue Router |
| 构建 | Webpack / Vue CLI |

### 2.2 本需求无新增依赖

所有实现复用现有 Element UI 组件（`el-drawer`、`el-dialog`、`el-button`、`el-form`、`el-skeleton`）。

---

## 3. 文件与目录结构

在 FRONTEND-REQ-007-pc 基础上新增/变更以下文件：

```
src/
├── views/
│   └── drawing/
│       └── components/
│           ├── TodoPanel.vue                      # 变更：内部审批卡片图标/文案/按钮
│           ├── DrawingApprovalDetailDrawer.vue    # 新增：审批详情侧滑弹框（F-003）
│           ├── ApproveConfirmDialog.vue           # 新增：Approve 确认对话框（F-005）
│           ├── RejectDialog.vue                   # 新增：驳回对话框（F-006）
│           └── NoDcWarningDialog.vue              # 新增：无 DC 警告弹窗（F-007）
├── api/
│   └── drawing.js                                 # 变更：新增 view/download/dc-config 接口
└── store/
    └── modules/
        └── todo.js                                # 变更：新增 drawerVisible、currentTodoItem
```

---

## 4. 组件结构

### 4.1 组件树

```
<TodoPanel>                              # 待办列表面板
├── <InternalApprovalTodoCard>           # 内部审批卡片（F-001），内联在 TodoPanel
│   └── [Detail] Button
├── <DrawingApprovalDetailDrawer>        # 详情侧滑弹框（F-003）
│   ├── Skeleton（加载态）
│   ├── FieldList（只读字段区）
│   │   └── OriginalFileLink             # 文件名可点击链接 + 辅助文案
│   └── ActionBar
│       ├── [Download Original File]     # F-004
│       ├── [Approve]                    # 触发 DC 预检查 → F-007 or F-005
│       └── [Reject]                     # 触发 F-006
├── <NoDcWarningDialog>                  # F-007
├── <ApproveConfirmDialog>               # F-005
└── <RejectDialog>                       # F-006
```

### 4.2 组件说明

#### `TodoPanel.vue`（变更）

**关联 AC**: AC-007A-001 / AC-007A-002 / AC-007A-003

**变更点**：
- 渲染 `type === 'DRAWING_INTERNAL_APPROVAL'` 的 Todo 卡片时：
  - 图标改为 🔍
  - 标题固定为 `"Internal Approval Required"`
  - 操作区只保留 `[Detail]` 按钮，移除 `[View Drawing]`、`[Approve]`、`[Reject]`
- 列表空状态（type 无记录时）显示 `"No pending tasks"`
- 点击 `[Detail]` 时 `emit('open-detail', todoItem)` 或直接写入 Vuex

```vue
<!-- AC-007A-001: 卡片图标与标题 -->
<template v-if="item.type === 'DRAWING_INTERNAL_APPROVAL'">
  <span class="todo-icon">🔍</span>
  <span class="todo-title">Internal Approval Required</span>
  <div class="todo-meta">
    {{ item.drawingCode }}  {{ item.drawingName }}  {{ item.versionNo }}
  </div>
  <div class="todo-meta">
    Uploaded by: {{ item.uploaderName }}（Designer） | {{ item.uploadTime | formatDateTime }}
  </div>
  <div v-if="item.versionNote" class="todo-meta">
    Version Note: {{ item.versionNote }}
  </div>
  <el-button size="small" @click="openDetail(item)">Detail</el-button>
</template>
```

**不该做**：
- 不在前端过滤 assigneeId（后端已过滤）
- 不在卡片上直接执行审批操作

---

#### `DrawingApprovalDetailDrawer.vue`（新增）

**关联 AC**: AC-007A-004 / AC-007A-005 / AC-007A-006 / AC-007A-011 / AC-007A-012 / AC-007A-013

**Props**:
```ts
interface DrawingApprovalDetailDrawerProps {
  visible: boolean           // 控制显隐
  todoItem: InternalApprovalTodo | null
}
```

**Emits**: `update:visible`, `approved`, `rejected`

**职责**：
1. `visible = true` 时，调用 `GET /drawing/version/{versionId}/detail` 加载详情（展示 Skeleton）
2. 加载成功后渲染只读字段列表（Category、Description、Version、Version Note、Uploaded by、Upload Time、Internal Approver、Original File）
3. Original File 文件名渲染为可点击链接（`target="_blank"`，调用 view 接口获取 inline 预签名 URL）
4. 文件名下方展示灰色辅助文案（AC-007A-012）
5. 若 `fileUrl` 为空，文件名显示 `—`（不可点击），辅助文案隐藏，下载按钮 disabled
6. 加载失败时展示错误提示 + [Retry] 按钮（AC-007A-013）
7. 按 `Esc` 或点击 `×` 关闭弹框

**关键实现细节**：

```vue
<el-drawer
  :visible.sync="visible"
  direction="rtl"
  size="480px"
  :before-close="handleClose"
  title="Drawing Approval Detail"
>
  <!-- 加载态 -->
  <el-skeleton v-if="loading" :rows="8" animated />

  <!-- 错误态 AC-007A-013 -->
  <div v-else-if="error" class="detail-error">
    <p>Failed to load drawing details.</p>
    <el-button size="small" @click="fetchDetail">Retry</el-button>
  </div>

  <!-- 内容区 -->
  <div v-else class="detail-content">
    <detail-field label="Category" :value="detail.category" />
    <detail-field label="Description" :value="detail.description || '—'" />
    <detail-field label="Version" :value="detail.versionNo" />
    <detail-field label="Version Note" :value="detail.versionNote || '—'" />
    <detail-field label="Uploaded by" :value="detail.uploaderName + '（Designer）'" />
    <detail-field label="Upload Time" :value="detail.uploadTime | formatDateTime" />
    <detail-field label="Internal Approver" :value="detail.approverName" />

    <!-- Original File  AC-007A-011 / AC-007A-012 -->
    <div class="detail-field">
      <span class="label">Original File</span>
      <span v-if="detail.fileUrl">
        <a :href="viewUrl" target="_blank" rel="noopener noreferrer">
          📄 {{ detail.fileName }}  ↗
        </a>
        <p class="file-hint">
          Click to open in browser. To save the file, use [Download Original File] below.
        </p>
      </span>
      <span v-else>—</span>
    </div>
  </div>

  <!-- 操作区 -->
  <template #footer>
    <div class="drawer-actions">
      <!-- F-004 -->
      <el-button
        :disabled="!detail.fileUrl"
        :loading="downloading"
        @click="handleDownload"
      >
        Download Original File
      </el-button>
      <!-- F-005 -->
      <el-button type="primary" @click="handleApproveClick">Approve</el-button>
      <!-- F-006 -->
      <el-button type="danger" plain @click="rejectDialogVisible = true">Reject</el-button>
    </div>
  </template>
</el-drawer>
```

**不该做**：
- 不展示 Drawing Code 和 Drawing Name 字段（已在 Todo 卡片摘要展示）
- 不提供任何字段编辑入口

---

#### `ApproveConfirmDialog.vue`（新增）

**关联 AC**: AC-007A-014 / AC-007A-015 / AC-007A-016 / AC-007A-017 / AC-007A-019 / AC-007A-020

**Props**: `visible: boolean`, `versionId: string`

**触发逻辑**（在 DrawingApprovalDetailDrawer 中）：

```javascript
async handleApproveClick() {
  // AC-007A-014 / AC-007A-015：前端 DC 预检查
  const dcList = await getDcConfig(this.projectId)
  if (!dcList || dcList.length === 0) {
    this.noDcWarningVisible = true  // 弹出 F-007
    return
  }
  this.approveConfirmVisible = true  // 弹出 F-005
}
```

**对话框文案**：

| 元素 | 内容 |
|------|------|
| 标题 | `Confirm Internal Approval?` |
| 内容 | `Once approved, this version will proceed to external approval by Document Controller. The version will NOT become active until external approval is completed.` |
| 按钮 | `[Cancel]` `[Confirm]` |

**Confirm 点击后**：
1. `[Confirm]` 进入 loading，禁用所有按钮（AC-007A-019）
2. 调用 `POST /drawing/approve`，传 `{ versionId, phase: 'INTERNAL', action: 'APPROVE' }`
3. 成功：对话框关闭 → drawer 关闭 → emit `approved` → Todo 卡片从列表移除
4. Toast: `"Internal approval completed. DC has been notified for external approval."` (AC-007A-016)
5. 失败：loading 恢复，Toast 错误文案，对话框保留（AC-007A-019）
   - 错误码 `1003007012` 时 Toast: `"No DC configured. Please ask admin to add a DC first."` (AC-007A-020)

---

#### `RejectDialog.vue`（新增）

**关联 AC**: AC-007A-021 / AC-007A-022

**Props**: `visible: boolean`, `versionId: string`, `drawingCode: string`, `drawingName: string`, `versionNo: string`

**表单**：Comment 文本域（必填，最多 500 字符）

**校验规则**：

```javascript
rules: {
  comment: [
    { required: true, message: 'Comment is required', trigger: 'blur' },
    { max: 500, message: 'Max 500 characters', trigger: 'change' }
  ]
}
```

**Confirm 点击后**：
1. 前端校验 Comment 非空（AC-007A-021）
2. 调用 `POST /drawing/approve`，传 `{ versionId, phase: 'INTERNAL', action: 'REJECT', comment }`
3. 成功：对话框关闭 → drawer 关闭 → emit `rejected` → Todo 卡片移除
4. Toast: `"Internal approval rejected. Designer has been notified."` (AC-007A-022)
5. 失败：loading 恢复，Toast 错误，对话框保留

---

#### `NoDcWarningDialog.vue`（新增）

**关联 AC**: AC-007A-014

**Props**: `visible: boolean`

**说明**：仅有 `[Got it]` 按钮，点击关闭弹窗，返回详情弹框，不执行审批操作。弹窗不可通过背景点击关闭（`:close-on-click-modal="false"`）。

```vue
<el-dialog
  title="⚠️ No DC Configured"
  :visible.sync="visible"
  :close-on-click-modal="false"
  width="440px"
>
  <p>
    This project has no Document Controller configured.
    You cannot proceed with internal approval until a DC is added.
  </p>
  <p>Please contact your project admin to configure a DC first.</p>
  <template #footer>
    <el-button type="primary" @click="$emit('update:visible', false)">Got it</el-button>
  </template>
</el-dialog>
```

---

## 5. 状态管理

### 5.1 状态分层

| 状态 | 存放位置 | 说明 |
|-----|---------|------|
| Todo 列表数据 | Vuex `todo.list` | 审批后 mutation 移除对应项 |
| 详情弹框显隐 | 组件局部 `drawerVisible` | 无需全局共享 |
| 当前操作的 Todo Item | 组件局部 `currentTodoItem` | 传给 Drawer |
| 详情加载数据 | 组件局部 `detail` | 每次打开重新请求，不缓存 |
| 各子对话框显隐 | 组件局部 | `approveConfirmVisible`、`rejectDialogVisible`、`noDcWarningVisible` |
| 下载 loading | 组件局部 `downloading` | 防重复点击 |

### 5.2 Todo 列表更新策略

审批操作（通过 / 驳回）成功后：

```javascript
// 从 Vuex todo.list 中移除已处理的 todoItem
this.$store.commit('todo/REMOVE_TODO', this.currentTodoItem.id)
```

---

## 6. API 接入

### 6.1 调用清单

| API | 说明 | 调用组件 | 方法 |
|-----|------|---------|------|
| `GET /todo/list?type=DRAWING_INTERNAL_APPROVAL` | 获取内部审批待办列表 | `TodoPanel` | `getTodoList()` |
| `GET /drawing/version/{versionId}/detail` | 获取图纸版本详情 | `DrawingApprovalDetailDrawer` | `getDrawingVersionDetail(versionId)` |
| `GET /drawing/version/{versionId}/view` | 获取 inline 预签名 URL（文件名链接） | `DrawingApprovalDetailDrawer` | `getDrawingViewUrl(versionId)` |
| `GET /drawing/version/{versionId}/download` | 触发强制下载（attachment 预签名 URL） | `DrawingApprovalDetailDrawer` | `downloadDrawingFile(versionId)` |
| `GET /project/{projectId}/dc-config` | 查询项目 DC 配置（Approve 前置检查） | `DrawingApprovalDetailDrawer` | `getDcConfig(projectId)` |
| `POST /drawing/approve` | 执行内部审批通过 / 驳回 | `ApproveConfirmDialog`, `RejectDialog` | `submitDrawingApproval(payload)` |

### 6.2 `drawing.js` 新增方法

```javascript
// 获取图纸版本详情（用于详情侧滑弹框）
export const getDrawingVersionDetail = (versionId) =>
  request.get(`/drawing/version/${versionId}/detail`)

// 获取 inline 预签名 URL（文件名链接点击时调用）
// 注意：接口返回 302 redirect，axios 默认跟随；或后端直接返回 { url } JSON
export const getDrawingViewUrl = (versionId) =>
  request.get(`/drawing/version/${versionId}/view`)

// 获取 attachment 预签名 URL（下载按钮点击时调用）
export const getDrawingDownloadUrl = (versionId) =>
  request.get(`/drawing/version/${versionId}/download`)

// 查询项目 DC 配置列表
export const getDcConfig = (projectId) =>
  request.get(`/project/${projectId}/dc-config`)

// 提交内部审批
export const submitDrawingApproval = (payload) =>
  request.post('/drawing/approve', payload)
// payload: { versionId, phase: 'INTERNAL', action: 'APPROVE' | 'REJECT', comment? }
```

### 6.3 文件下载实现

下载按钮不使用 `<a>` 标签直接下载（无法携带 Authorization header），通过接口获取带鉴权预签名 URL 后由浏览器触发下载：

```javascript
// AC-007A-007 / AC-007A-008
async handleDownload() {
  this.downloading = true
  try {
    const { data } = await getDrawingDownloadUrl(this.detail.versionId)
    // data.url 为带 Content-Disposition: attachment 的预签名 URL
    const link = document.createElement('a')
    link.href = data.url
    link.setAttribute('download', `${this.detail.drawingCode}-V${this.detail.versionNo}-original.${this.detail.fileExt}`)
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
  } catch (e) {
    this.$message.error('Download failed. Please try again.')
  } finally {
    this.downloading = false
  }
}
```

### 6.4 错误处理

| 错误码 | 场景 | 用户提示 |
|-------|------|---------|
| `1003007012` | 项目无 DC 配置（后端兜底） | `"No DC configured. Please ask admin to add a DC first."` |
| `403` | 非指派审批人越权访问 | `"You don't have permission to perform this action."` |
| `404` | 图纸版本不存在 | `"Drawing version not found."` |
| `409` | Todo 已被处理（并发） | `"This task has already been completed."` — 按钮隐藏 |
| `5xx` | 服务端错误 | `"Server error. Please try again."` |

---

## 7. 性能预算

| 指标 | 目标 | 备注 |
|-----|------|------|
| 详情弹框首屏渲染 | P95 ≤ 800ms（局域网） | 请求 + 渲染 |
| DC 配置查询 | P95 ≤ 500ms | Approve 前置检查 |
| 下载预签名 URL 生成 | P95 ≤ 500ms | 后端接口 |

**优化手段**：
- 详情接口不做客户端缓存（保证数据实时性，避免拿到已处理任务的旧快照）
- DC 配置查询结果可在同一次弹框生命周期内缓存（避免重复点击时多次请求）

---

## 8. 弱网与离线策略

| 场景 | 策略 |
|-----|------|
| 详情接口超时 | 弹框内显示 "Failed to load drawing details." + [Retry] 按钮 |
| 审批接口请求中断 | loading 恢复，Toast 报错，对话框保留，可重试 |
| 下载接口失败 | Toast: "Download failed. Please try again."，按钮恢复可点击 |
| 文件名链接打开失败 | 新标签页浏览器错误页；用户可改用 [Download Original File] |

---

## 9. 埋点设计

| 事件名 | 触发时机 | 关键属性 |
|-------|---------|---------|
| `todo_detail_opened` | 点击 [Detail] 打开弹框成功 | `drawingVersionId`, `drawingCode` |
| `drawing_original_download_triggered` | 点击 [Download Original File] | `drawingVersionId`, `fileExt` |
| `drawing_original_download_failed` | 下载接口返回错误 | `drawingVersionId`, `errorCode` |
| `internal_approval_approved` | Approve 确认成功 | `drawingVersionId` |
| `internal_approval_rejected` | Reject 确认成功 | `drawingVersionId` |

---

## 10. 国际化

- i18n 库：vue-i18n
- 界面文案以英文为主，与现有系统一致
- 关键文案：

| Key | 英文值 |
|-----|-------|
| `todo.internalApproval.title` | `Internal Approval Required` |
| `todo.internalApproval.drawer.title` | `Drawing Approval Detail` |
| `todo.internalApproval.file.hint` | `Click to open in browser. To save the file, use [Download Original File] below.` |
| `todo.internalApproval.file.unavailable` | `Original file unavailable.` |
| `todo.internalApproval.approve.toast` | `Internal approval completed. DC has been notified for external approval.` |
| `todo.internalApproval.reject.toast` | `Internal approval rejected. Designer has been notified.` |
| `todo.internalApproval.noDc.title` | `No DC Configured` |
| `todo.internalApproval.download.failed` | `Download failed. Please try again.` |
| `todo.internalApproval.detail.failed` | `Failed to load drawing details.` |

---

## 11. 可访问性（WCAG AA）

- 下载按钮添加 `aria-label="Download original drawing file"`
- 文件图标添加 `alt` 文本（`alt="file"`）
- 弹框内所有字段支持 Tab 键导航
- `Esc` 键关闭侧滑弹框
- `[Download Original File]` 可通过 Tab 聚焦并按 Enter 触发
- disabled 状态按钮的 Tooltip 通过 `el-tooltip` 实现，`aria-disabled="true"`

---

## 12. 测试要求

| 层级 | 框架 | 范围 | 覆盖率目标 |
|-----|------|------|-----------|
| 单元 | Jest + Vue Test Utils | 各组件 props/emits/逻辑分支 | ≥ 80% |
| 集成 | Jest + MSW | API 调用 + 状态更新流程 | 主流程全覆盖 |
| E2E | Playwright（QA 维护） | — | — |

**重点测试场景**：
- fileUrl 为空时下载按钮 disabled、文件名显示 `—`（AC-007A-009）
- DC 配置为空时弹出 NoDcWarningDialog（AC-007A-014）
- Comment 为空时 Reject 无法提交（AC-007A-021）
- 并发：Todo 已处理时详情弹框显示已完成提示（AC-007A 相关异常）

---

## 13. AC 覆盖检查表

| AC ID | 实现位置 | 状态 |
|------|---------|------|
| AC-007A-001 | `TodoPanel.vue` — 内部审批卡片图标/标题 | TODO |
| AC-007A-002 | `TodoPanel.vue` — 后端已按 assigneeId 过滤，前端无额外操作 | TODO |
| AC-007A-003 | `TodoPanel.vue` — 空状态文案 "No pending tasks" | TODO |
| AC-007A-004 | `DrawingApprovalDetailDrawer.vue` — 点击 [Detail] 打开 | TODO |
| AC-007A-005 | `DrawingApprovalDetailDrawer.vue` — 字段完整展示 | TODO |
| AC-007A-006 | `DrawingApprovalDetailDrawer.vue` — 选填字段为空显示 `—` | TODO |
| AC-007A-007 | `handleDownload()` — 文件命名规则 | TODO |
| AC-007A-008 | `handleDownload()` — loading 防重复点击 | TODO |
| AC-007A-009 | `DrawingApprovalDetailDrawer.vue` — fileUrl 为空时 disabled + Tooltip | TODO |
| AC-007A-010 | `BACKEND-REQ-007A.md` — 后端 403 校验，前端无需额外处理 | TODO |
| AC-007A-011 | `DrawingApprovalDetailDrawer.vue` — 文件名链接 target="_blank" | TODO |
| AC-007A-012 | `DrawingApprovalDetailDrawer.vue` — 文件名下方辅助文案 | TODO |
| AC-007A-013 | `DrawingApprovalDetailDrawer.vue` — 加载失败 + Retry | TODO |
| AC-007A-014 | `handleApproveClick()` — DC 预检查 → NoDcWarningDialog | TODO |
| AC-007A-015 | `handleApproveClick()` — 有 DC 时弹出 ApproveConfirmDialog | TODO |
| AC-007A-016 | `ApproveConfirmDialog.vue` — 成功 Toast | TODO |
| AC-007A-017 | 前端不感知版本状态，由后端保证；可通过图纸列表验证 | TODO |
| AC-007A-018 | 由后端 + FRONTEND-REQ-007B-pc 覆盖 | TODO |
| AC-007A-019 | `ApproveConfirmDialog.vue` — loading + 失败可重试 | TODO |
| AC-007A-020 | `ApproveConfirmDialog.vue` — 错误码 1003007012 处理 | TODO |
| AC-007A-021 | `RejectDialog.vue` — Comment 必填前端校验 | TODO |
| AC-007A-022 | `RejectDialog.vue` — 成功 Toast | TODO |
| AC-007A-023 | 由后端保证，前端不感知旧版本状态 | TODO |

---

## 14. 与后端的联调约定

- **联调前置**：`BACKEND-REQ-007A.md` 接口签名确认
- **Mock 服务**：基于接口文档 §6.1 生成 MSW handlers，前端可独立开发
- **重点对齐**：
  - `GET /drawing/version/{versionId}/download` 响应方式（302 redirect vs JSON `{url}`），需与后端对齐，影响下载实现方式
  - `GET /drawing/version/{versionId}/view` 同上
  - `POST /drawing/approve` payload 结构与错误码
  - `GET /project/{projectId}/dc-config` 返回结构（空列表 vs `[]`）

---

## 15. 验收条件

- [ ] 所有 AC 在 §13 表中标记完成
- [ ] 单元 + 集成测试通过，覆盖率达标
- [ ] E2E 测试由 QA 跑通（不阻塞前端提测）
- [ ] 无 console 错误 / 警告
- [ ] 无障碍自动化检查无 critical 项
- [ ] 已部署到 dev 环境，可联调

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.5.0 | 2026-05-25 | agent | 基于 REQ-007A-pc v0.5.0（合并 REQ-007E-pc 后）重新生成；新增 DrawingApprovalDetailDrawer（F-003）、DownloadOriginalFile（F-004）、ApproveConfirmDialog（F-005）、RejectDialog（F-006）、NoDcWarningDialog（F-007）完整实现说明 |
