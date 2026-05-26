---
doc_type: frontend_spec
req_id: REQ-007C-pc
version: 0.2.5
status: draft
generated_from: REQ-007C-pc.md@0.2.5
ui_spec_ref: UI-REQ-007C-pc.md
generated_at: 2026-05-26
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明 — PC 端版本历史抽屉 4 阶段生命周期视图

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-007C-pc.md](../../../requirements/pc/REQ-007C-pc.md) @ v0.2.5
> **UI 设计参考**: [UI-REQ-007C-pc.md](../../ui/pc/UI-REQ-007C-pc.md)
> **依赖前端文档**:
> - [FRONTEND-REQ-007B-pc.md](./FRONTEND-REQ-007B-pc.md)（复用 `MarkExternalApprovalDialog` 组件）
> - [FRONTEND-REQ-003-pc.md](./FRONTEND-REQ-003-pc.md)（图纸列表基础前端，原 HistoryDrawer 入口）
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-05-26

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007C-pc @ v0.2.5 |
| UI 设计 | UI-REQ-007C-pc.md |
| 覆盖 Story | US-007C-001 |
| 覆盖 AC | AC-007C-001 ~ AC-007C-018 |

---

## 1. 功能概述

将图纸版本历史抽屉升级为可展开的 4 阶段生命周期视图，涉及以下前端变更：

| 功能模块 | 变更类型 | 说明 |
|---------|---------|------|
| VersionHistoryDrawer（F-001）| **重构** | 主列表列定义更新：新增 RFA No. 列、Actions 列，移除 Attachments 列；抽屉宽度 720px |
| VersionExpandRow（F-003）| **新增** | 展开行 4 阶段卡片（2×2 网格），5 种状态下渲染逻辑 |
| LifecycleStepBar（F-004）| **新增** | 底部步骤条，4 步颜色规范 |
| VersionLifecycleCard（F-005）| **新增** | 卡片内操作按钮：[Preview]、[Download]、[Mark Result]、Evidence [Download] |
| AttachmentsDrawer（F-006）| **新增** | 二级附件抽屉（480px），由主列表 Actions 列 [Details] 按钮触发；含 Part Print / Attachments 双 Tab |
| MarkExternalApprovalDialog | **复用** | 来自 FRONTEND-REQ-007B-pc，不重复实现 |

---

## 2. 技术栈

### 2.1 已有技术栈（继承自 FRONTEND-REQ-003-pc）

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 |
| UI 库 | Element UI |
| HTTP 客户端 | Axios（封装在 `src/api/`） |
| 状态管理 | Vuex |
| 路由 | Vue Router |
| 构建 | Webpack / Vue CLI |

### 2.2 本需求无新增依赖

所有实现复用现有 Element UI 组件（`el-drawer`、`el-table`、`el-table-column`、`el-steps`、`el-step`、`el-tabs`、`el-upload`、`el-dialog`、`el-button`、`el-skeleton`、`el-tag`、`el-tooltip`）。

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── drawing/
│       └── components/
│           ├── VersionHistoryDrawer.vue          # 重构：主列表 + 展开行容器（F-001）
│           ├── VersionExpandRow.vue              # 新增：4 阶段卡片展开行（F-003）
│           │   ├── UploadedCard.vue              # ① 上传卡片
│           │   ├── InternalApprovalCard.vue      # ② 内部审批卡片
│           │   ├── ExternalApprovalCard.vue      # ③ 外部审批卡片（含 Mark Result 入口）
│           │   └── SignedVersionCard.vue         # ④ 签字版卡片
│           ├── LifecycleStepBar.vue              # 新增：底部步骤条（F-004）
│           └── AttachmentsDrawer.vue             # 新增：附件二级抽屉（F-006）
│               ├── PartPrintTab.vue              # Part Print Tab（只读列表）
│               └── AttachmentsTab.vue            # Attachments Tab（上传/删除）
├── api/
│   └── drawing.js                               # 变更：新增版本历史详情、附件上传/删除接口
└── store/
    └── modules/
        └── versionHistory.js                    # 新增：展开行状态、附件抽屉状态
```

---

## 4. 组件结构

### 4.1 组件树

```
<VersionHistoryDrawer>                    # 主抽屉 720px（F-001）
├── el-table（主列表）
│   ├── ExpandColumn（▶ 展开列）
│   │   └── <VersionExpandRow>            # 展开行（F-003）
│   │       ├── <UploadedCard>            # ① 上传卡片
│   │       ├── <InternalApprovalCard>    # ② 内部审批卡片
│   │       ├── <ExternalApprovalCard>    # ③ 外部审批卡片
│   │       │   └── <MarkExternalApprovalDialog>  # 复用 FRONTEND-REQ-007B-pc
│   │       ├── <SignedVersionCard>       # ④ 签字版卡片
│   │       └── <LifecycleStepBar>        # 底部步骤条（F-004）
│   ├── RfaNoColumn（RFA No.）
│   ├── StatusColumn（Status 标签）
│   ├── DesignerColumn（Designer）
│   ├── ConfirmedColumn（Confirmed）
│   ├── QrColumn（QR [View]）
│   └── ActionsColumn（[Details] 按钮）
└── <AttachmentsDrawer>                   # 二级附件抽屉 480px（F-006），叠加在主抽屉之上
    ├── VersionInfoHeader                 # 顶部版本信息区（只读）
    ├── el-tabs
    │   ├── <PartPrintTab>                # Part Print Tab（只读）
    │   └── <AttachmentsTab>             # Attachments Tab（上传/删除）
    └── <DeleteConfirmDialog>            # 附件删除二次确认
```

---

## 5. 组件详细说明

### 5.1 `VersionHistoryDrawer.vue`（重构，F-001）

**关联 AC**: AC-007C-001 / AC-007C-002

**Props**:
```ts
interface VersionHistoryDrawerProps {
  visible: boolean        // v-model 控制显隐
  drawingId: string       // 当前图纸 ID
  drawingCode: string
  drawingName: string
}
```

**抽屉规格**:
```vue
<el-drawer
  :title="`Version History — ${drawingCode} ${drawingName}`"
  :visible.sync="visible"
  direction="rtl"
  size="720px"
  :wrapper-closable="false"
>
```

**主列表列定义**:

```vue
<!-- AC-007C-001: 展开列 -->
<el-table-column type="expand" width="40">
  <template slot-scope="{ row }">
    <VersionExpandRow :version="row" @mark-success="handleMarkSuccess" />
  </template>
</el-table-column>

<!-- Version 列 -->
<el-table-column prop="versionNo" label="Version" width="80" />

<!-- RFA No. 列（AC-007C-002：phase=EXTERNAL 的 submissionRefNo，无则显示 —）-->
<el-table-column label="RFA No." width="140">
  <template slot-scope="{ row }">
    {{ row.rfaNo || '—' }}
  </template>
</el-table-column>

<!-- Status 标签列 -->
<el-table-column label="Status" width="160">
  <template slot-scope="{ row }">
    <VersionStatusTag :status="row.approvalStatus" />
  </template>
</el-table-column>

<!-- Designer 列 -->
<el-table-column prop="uploaderName" label="Designer" width="100" />

<!-- Confirmed 列：仅 APPROVED 显示 x/y，其他显示 — -->
<el-table-column label="Confirmed" width="90">
  <template slot-scope="{ row }">
    {{ row.approvalStatus === 'APPROVED' ? `${row.confirmedCount}/${row.totalCount}` : '—' }}
  </template>
</el-table-column>

<!-- QR 列：仅 APPROVED 且 qrImageUrl 非空时显示 [View] -->
<el-table-column label="QR" width="70">
  <template slot-scope="{ row }">
    <a v-if="row.approvalStatus === 'APPROVED' && row.qrImageUrl"
       :href="row.qrImageUrl" target="_blank">View</a>
    <span v-else>—</span>
  </template>
</el-table-column>

<!-- Actions 列：[Details] 按钮触发 AttachmentsDrawer（AC-007C-011）-->
<el-table-column label="Actions" width="90">
  <template slot-scope="{ row }">
    <el-button size="mini" @click="openAttachmentsDrawer(row)">Details</el-button>
  </template>
</el-table-column>
```

> ⚠️ **注意**：主列表不再有 Attachments 列（已移除）。[Details] 按钮与版本行展开/折叠完全独立——两个操作互不影响。

**数据加载**:
```javascript
// watch(visible) → true 时加载
async loadVersionHistory() {
  this.loading = true
  try {
    const res = await drawingApi.getVersionHistory(this.drawingId)
    this.versions = res.data
  } catch (e) {
    this.loadError = true
  } finally {
    this.loading = false
  }
}
```

---

### 5.2 `VersionExpandRow.vue`（新增，F-003）

**关联 AC**: AC-007C-003 ~ AC-007C-010

**Props**:
```ts
interface VersionExpandRowProps {
  version: DrawingVersionDetail  // 含 DrawingApproval 列表
}
```

**布局**: 2×2 网格（`display: grid; grid-template-columns: 1fr 1fr; gap: 16px`），底部步骤条横跨全宽。

**卡片渲染逻辑**（核心状态映射）:

```javascript
// AC-007C-003~008: 根据 approvalStatus 决定各卡片状态
computed: {
  cardStates() {
    const s = this.version.approvalStatus
    return {
      uploaded:  { done: true },   // ① 始终完成
      internal: {
        done:    ['INTERNAL_APPROVED','APPROVED','EXTERNAL_REJECTED'].includes(s),
        pending: s === 'PENDING_INTERNAL',
        failed:  s === 'INTERNAL_REJECTED',
        notReached: false,
      },
      external: {
        done:    s === 'APPROVED',
        pending: s === 'INTERNAL_APPROVED',
        failed:  s === 'EXTERNAL_REJECTED',
        notReached: ['PENDING_INTERNAL','INTERNAL_REJECTED'].includes(s),
      },
      signed: {
        done:    s === 'APPROVED',
        notReached: s !== 'APPROVED',
      },
    }
  }
}
```

---

### 5.3 `ExternalApprovalCard.vue`（新增，③ 卡片）

**关联 AC**: AC-007C-003 / AC-007C-004 / AC-007C-005 / AC-007C-007

**Status of Approval 展示规则**（对应 REQ-007B-pc Status A–E）：

```javascript
// AC-007C-003/007: Status 标签 + 图标
const STATUS_DISPLAY = {
  A: { icon: '✅', label: 'A – Approved / No Exception Taken',              type: 'success' },
  B: { icon: '✅', label: 'B – Approved with comment, resubmission required', type: 'success' },
  D: { icon: '✅', label: 'D – For Record Purpose',                          type: 'success' },
  C: { icon: '❌', label: 'C – Revise And Resubmit',                        type: 'danger'  },
  E: { icon: '❌', label: (reason) => `E – Others: ${reason}`,              type: 'danger'  },
}
```

**Evidence [Download] 显示条件**（AC-007C-003）：仅当 `approvalStatus === 'APPROVED'` 时（即 Status A/B/D），`evidenceFileUrl` 非空才显示；Status C/E 无凭证文件，不显示。

**[Mark Result] 按钮权限控制**（AC-007C-004 / AC-007C-005）：

```vue
<!-- 仅 PENDING_EXTERNAL（即 approvalStatus=INTERNAL_APPROVED）+ DC 权限时显示 -->
<el-button
  v-if="version.approvalStatus === 'INTERNAL_APPROVED' && canMarkExternal"
  type="success"
  size="small"
  @click="openMarkDialog"
>
  ✅ Mark Result
</el-button>
```

```javascript
computed: {
  canMarkExternal() {
    // AC-007C-004/005
    return this.$store.getters.permissions.includes('drawing:external-approval')
  }
}
```

点击 [Mark Result] 后打开 `<MarkExternalApprovalDialog>`（直接复用 FRONTEND-REQ-007B-pc 的实现，不重复编写 Dialog 逻辑）。

---

### 5.4 `SignedVersionCard.vue`（新增，④ 卡片）

**关联 AC**: AC-007C-009 / AC-007C-010

**[Download] 降级逻辑**：

```javascript
// AC-007C-009: 优先 pdfWithQrUrl；AC-007C-010: 降级 signedFileUrl
computed: {
  downloadUrl() {
    return this.version.pdfWithQrUrl || this.version.signedFileUrl
  }
}
```

```vue
<el-button size="small" @click="downloadSigned">Download</el-button>

// methods:
downloadSigned() {
  const url = this.version.pdfWithQrUrl || this.version.signedFileUrl
  if (!url) return
  triggerDownload(url)
}
```

---

### 5.5 `LifecycleStepBar.vue`（新增，F-004）

**关联 AC**: AC-007C-003 ~ AC-007C-008（步骤条颜色映射）

```vue
<div class="step-bar">
  <StepItem icon="📄" label="Uploaded"  :state="'done'" />
  <span class="arrow">→</span>
  <StepItem icon="🔍" label="Internal"  :state="internalState" />
  <span class="arrow">→</span>
  <StepItem icon="🌐" label="External"  :state="externalState" />
  <span class="arrow">→</span>
  <StepItem icon="✍️" label="Signed"    :state="signedState" />
</div>
```

**状态 → 颜色映射**:

```javascript
const STATE_COLOR = {
  done:       '#67C23A',  // 绿色 ✓
  pending:    '#E6A23C',  // 橙色 ⏳
  failed:     '#F56C6C',  // 红色 ✕
  notReached: '#C0C4CC',  // 灰色 —
}
```

---

### 5.6 `AttachmentsDrawer.vue`（新增，F-006）

**关联 AC**: AC-007C-011 ~ AC-007C-018

**触发方式**: 主列表 Actions 列 [Details] 按钮 → `openAttachmentsDrawer(row)` → Vuex commit → 二级抽屉打开。与版本行展开/折叠无关（AC-007C-011）。

**抽屉规格**:
```vue
<el-drawer
  :visible.sync="attachDrawerVisible"
  direction="rtl"
  size="480px"
  :append-to-body="true"   <!-- 叠加在版本历史抽屉之上 -->
  :title="`Version History — · ${currentVersion.drawingName}  ${currentVersion.versionNo}`"
>
```

> ⚠️ 标题不含 Drawing Code（AC-007C-012）。

**顶部版本信息区**（只读，AC-007C-012）：

| 字段 | 数据来源 | 空值处理 |
|------|---------|---------|
| Status | `version.approvalStatus` | 渲染 `<VersionStatusTag>` |
| Ver (System) | `version.versionNo` | — |
| Description | `drawing.description` | 显示 `—` |
| Submission Ref No. | `DrawingApproval.submissionRefNo`（phase=EXTERNAL） | 显示 `—` |
| Submission Subject | `DrawingApproval.submissionSubject`（phase=EXTERNAL） | 显示 `—` |
| Uploaded by | 头像 + `version.uploaderName` | — |
| Upload Date | `version.uploadTime` | 格式：`DD-MM-YYYY HH:mm:ss` |
| 主文件行 | 文件名 + `✅ Approved by {name} · {time}` + [Download] | 点击下载 `fileUrl` |

**Tab 导航**（AC-007C-017）：

```vue
<el-tabs v-model="activeTab" default-active="attachments">
  <el-tab-pane
    :label="`Part Print (${partPrintCount})`"
    name="partprint"
  >
    <PartPrintTab :version-id="currentVersion.id" />
  </el-tab-pane>
  <el-tab-pane
    :label="`Attachments (${attachmentCount})`"
    name="attachments"
  >
    <AttachmentsTab
      :version-id="currentVersion.id"
      :is-uploader="isCurrentUserUploader"
      @count-change="onAttachmentCountChange"
    />
  </el-tab-pane>
</el-tabs>
```

默认激活 `"attachments"` Tab（AC-007C-012）。

---

### 5.7 `PartPrintTab.vue`（新增）

**关联 AC**: AC-007C-017 / AC-007C-018

- 调用 `GET /drawing/version/{versionId}/part-prints` 加载数据，按发布时间倒序排列
- 纯**只读**展示，无发布/删除按钮（AC-007C-018）
- 无 Part Print 时显示空状态 `"No Data"`

**每条 Part Print 卡片**:

```vue
<div class="part-print-card">
  <div class="pp-title">{{ pp.title }}</div>
  <div class="pp-meta">
    {{ pp.publishedAt | formatDate('MMM D, YYYY') }} · {{ pp.creatorName }}
  </div>
  <div class="pp-submission">
    Based on {{ pp.submissionNo }}
    <span v-if="pp.appliedPageNo"> · Page {{ pp.appliedPageNo }}</span>
  </div>
  <!-- 说明文字：超 3 行折叠 + "…Show more" 展开 -->
  <CollapsibleText :text="pp.description" :max-lines="3" />
  <!-- 附件文件名，点击可下载 -->
  <a v-for="att in pp.attachments" :key="att.id"
     :href="att.url" target="_blank">
    📎 {{ att.filename }}
  </a>
</div>
```

---

### 5.8 `AttachmentsTab.vue`（新增）

**关联 AC**: AC-007C-013 ~ AC-007C-016

**[+] 上传按钮**（AC-007C-014）：仅本版本上传人（`isUploader = true`）可见

```vue
<div class="tab-header">
  <span>Attachments ({{ attachments.length }})</span>
  <!-- AC-007C-014: 仅上传人可见 -->
  <el-button v-if="isUploader" icon="el-icon-plus" circle size="mini"
             @click="triggerUpload" />
</div>

<el-upload
  v-if="isUploader"
  ref="uploader"
  action="#"
  :http-request="doUpload"
  :before-upload="validateAttachment"
  :multiple="true"
  :limit="5"
  :show-file-list="false"
  style="display:none"
/>
```

**文件校验**（AC-007C-015）：
```javascript
validateAttachment(file) {
  if (file.size > 50 * 1024 * 1024) {
    this.$message.error('文件不得超过 50MB')
    return false
  }
  return true
}
```

**上传成功处理**（AC-007C-015）：
```javascript
async doUpload({ file }) {
  const res = await drawingApi.uploadAttachment(this.versionId, file)
  this.attachments.push(res.data)      // 即时追加新行
  this.$emit('count-change', this.attachments.length)  // 触发 Tab 标题计数更新
  this.$message.success('Upload successful')
}
```

**Attachments 表格列**（AC-007C-013 / AC-007C-016）：

```vue
<el-table :data="attachments">
  <!-- 文件名点击可下载（AC-007C-013）-->
  <el-table-column label="Filename">
    <template slot-scope="{ row }">
      <a :href="row.url" target="_blank">{{ row.filename }}</a>
    </template>
  </el-table-column>
  <el-table-column prop="fileType" label="Type" width="70" />
  <el-table-column prop="sizeFormatted" label="Size" width="80" />
  <el-table-column prop="uploadDate" label="Uploaded" width="110" />
  <el-table-column prop="uploaderName" label="Uploaded by" width="100" />
  <el-table-column label="" width="140">
    <template slot-scope="{ row }">
      <!-- [Download] 所有人可见（AC-007C-013）-->
      <el-button size="mini" @click="downloadAttachment(row)">Download</el-button>
      <!-- [Delete] 仅本版本上传人可见（AC-007C-014/016）-->
      <el-button v-if="isUploader" size="mini" type="danger"
                 @click="confirmDelete(row)">Delete</el-button>
    </template>
  </el-table-column>
</el-table>
```

**删除二次确认**（AC-007C-016）：

```javascript
confirmDelete(row) {
  this.$confirm(
    `Delete ${row.filename}? This action cannot be undone.`,
    'Confirm Delete',
    { confirmButtonText: 'Delete', cancelButtonText: 'Cancel', type: 'warning' }
  ).then(async () => {
    await drawingApi.deleteAttachment(this.versionId, row.id)
    const idx = this.attachments.findIndex(a => a.id === row.id)
    if (idx > -1) this.attachments.splice(idx, 1)
    this.$emit('count-change', this.attachments.length)
  }).catch(() => { /* Cancel: 无操作 */ })
}
```

---

## 6. 状态管理

### 6.1 Vuex — versionHistory.js（新增）

```javascript
state: {
  historyDrawerVisible: false,
  currentDrawing: null,           // { drawingId, drawingCode, drawingName }
  attachDrawerVisible: false,
  currentVersionForAttach: null,  // 触发附件抽屉的版本行数据
},
mutations: {
  OPEN_HISTORY(state, drawing)      { state.historyDrawerVisible = true;  state.currentDrawing = drawing },
  CLOSE_HISTORY(state)              { state.historyDrawerVisible = false; state.currentDrawing = null },
  OPEN_ATTACH_DRAWER(state, ver)    { state.attachDrawerVisible = true;   state.currentVersionForAttach = ver },
  CLOSE_ATTACH_DRAWER(state)        { state.attachDrawerVisible = false;  state.currentVersionForAttach = null },
},
```

### 6.2 缓存失效策略

| 操作 | 失效 key |
|------|---------|
| [Mark Result] 标记成功 | `drawing:versionHistory:{drawingId}` |
| 上传附件成功 | `drawing:attachments:{versionId}` |
| 删除附件成功 | `drawing:attachments:{versionId}` |

---

## 7. API 接入

### 7.1 调用清单

| API | 组件 | 说明 |
|-----|------|------|
| `GET /drawing/{drawingId}/version-history` | `VersionHistoryDrawer` | 主列表数据（含所有版本及 DrawingApproval 列表） |
| `GET /drawing/version/{versionId}/download` | 各卡片 [Download] | 302 → 预签名 URL（attachment，5 分钟有效） |
| `GET /drawing/version/{versionId}/view` | ① 卡片 [Preview] | 302 → 预签名 URL（inline，5 分钟有效） |
| `GET /drawing/version/{versionId}/part-prints` | `PartPrintTab` | 该版本关联的 Part Print 列表 |
| `GET /drawing/version/{versionId}/attachments` | `AttachmentsTab` | 附件列表 |
| `POST /drawing/version/{versionId}/attachments` | `AttachmentsTab` [+] | 上传附件 |
| `DELETE /drawing/version/{versionId}/attachments/{attachmentId}` | `AttachmentsTab` [Delete] | 删除附件 |
| `POST /drawing/external-approve` | `MarkExternalApprovalDialog`（复用） | 标记外部审批结果 |

### 7.2 接口封装（drawing.js 新增）

```javascript
// drawing.js 新增
export const drawingApi = {
  // ...已有方法...

  // 版本历史（含 DrawingApproval）
  getVersionHistory: (drawingId) =>
    axios.get(`/drawing/${drawingId}/version-history`),

  // Part Print 列表
  getVersionPartPrints: (versionId) =>
    axios.get(`/drawing/version/${versionId}/part-prints`),

  // 附件列表
  getVersionAttachments: (versionId) =>
    axios.get(`/drawing/version/${versionId}/attachments`),

  // 上传附件
  uploadAttachment: (versionId, file) => {
    const fd = new FormData()
    fd.append('file', file)
    return axios.post(`/drawing/version/${versionId}/attachments`, fd,
      { headers: { 'Content-Type': 'multipart/form-data' } })
  },

  // 删除附件
  deleteAttachment: (versionId, attachmentId) =>
    axios.delete(`/drawing/version/${versionId}/attachments/${attachmentId}`),
}
```

### 7.3 错误处理

| 场景 | 用户可见文案 | 处理位置 |
|------|-----------|---------|
| 版本历史加载失败 | `"加载失败，请刷新重试"` | `VersionHistoryDrawer` error state |
| 文件 URL 失效 (404) | `"文件不可用"` Toast | 各 [Preview]/[Download] 按钮 catch |
| 附件上传超 50MB | `"文件不得超过 50MB"` Toast | `validateAttachment` |
| 附件上传超 5 个 | `"单次最多上传 5 个文件"` Toast | el-upload `:limit` 的 `on-exceed` |
| 附件删除失败 | `"Delete failed, please retry"` Toast | `confirmDelete` catch |

---

## 8. 权限控制（前端层）

```javascript
// 全局 computed
computed: {
  canMarkExternal() {
    // AC-007C-004/005: drawing:external-approval 权限 + 项目已配置 DC
    return this.$store.getters.permissions.includes('drawing:external-approval')
  },
  isCurrentUserUploader() {
    // AC-007C-014: 仅本版本上传人可上传/删除附件
    return this.currentVersion?.uploaderUserId === this.$store.getters.userId
  }
}
```

> 前端仅做展示层隐藏；所有写操作后端再次校验权限。

---

## 9. 性能与可访问性

| 项 | 规范 |
|----|------|
| 抽屉加载骨架屏 | 超 300ms 展示 `el-skeleton`（4 行主列表骨架） |
| 展开/折叠动画 | `transition: max-height 0.2s ease`，目测 ≥ 60fps |
| 展开/折叠键盘操作 | ▶ 图标支持 `Enter`/`Space` 键（WCAG AA，AC-007C-001） |
| 国际化 | 所有文案使用 `$t()` 包裹，支持中英双语 |
| 浏览器兼容 | Chrome 100+、Edge 100+、Safari 15+ |

---

## 10. 埋点

| 事件 | 触发点 |
|------|-------|
| `version_history_open` | 打开版本历史抽屉 |
| `version_expand` | 展开版本行（携带 approvalStatus） |
| `version_download_original` | 下载原始文件 |
| `version_download_signed` | 下载签字版（携带 is_qr_version） |
| `version_download_evidence` | 下载审批凭证 |
| `version_mark_result_open` | 点击 [Mark Result]（从版本历史入口） |
| `attachments_drawer_open` | 点击 [Details] 打开附件抽屉 |
| `attachment_upload_success` | 附件上传成功 |
| `attachment_delete_success` | 附件删除成功 |

---

## 11. AC 覆盖矩阵

| AC ID | 描述简述 | 实现位置 |
|-------|---------|---------|
| AC-007C-001 | 抽屉宽度 720px，标题格式 | `VersionHistoryDrawer`（size="720px"，`:title`） |
| AC-007C-002 | 主列表列完整（Version/RFA No./Status/Designer/Confirmed/QR/Actions），RFA No. 显示规则，Confirmed/QR 条件展示 | `VersionHistoryDrawer` 列定义 |
| AC-007C-003 | APPROVED 版本 4 卡片全完成（含 Evidence [Download]，Status A/B/D 时） | `VersionExpandRow` + 各卡片 |
| AC-007C-004 | INTERNAL_APPROVED + DC 权限时显示 [Mark Result] | `ExternalApprovalCard`（v-if canMarkExternal） |
| AC-007C-005 | 非 DC 不显示 [Mark Result] | 同上（v-if 隐藏） |
| AC-007C-006 | INTERNAL_REJECTED：② 显示 Rejected + Comment，③④ Not reached | `InternalApprovalCard` + `ExternalApprovalCard` |
| AC-007C-007 | EXTERNAL_REJECTED：③ 显示 Status C/E，无 Evidence，④ Not reached | `ExternalApprovalCard`（STATUS_DISPLAY 映射） |
| AC-007C-008 | PENDING_INTERNAL：② Pending，③④ 等待 | `InternalApprovalCard` + `ExternalApprovalCard` |
| AC-007C-009 | 签字版 [Download] 优先 pdfWithQrUrl | `SignedVersionCard.downloadUrl` computed |
| AC-007C-010 | pdfWithQrUrl 为 null 时降级 signedFileUrl | 同上（`|| signedFileUrl`） |
| AC-007C-011 | [Details] 触发附件抽屉，与展开行无关 | `ActionsColumn` @click → `openAttachmentsDrawer` |
| AC-007C-012 | 附件抽屉 480px，标题含 drawingName 不含 drawingCode，顶部字段展示，默认 Attachments Tab | `AttachmentsDrawer` size/title/defaultTab |
| AC-007C-013 | 附件下载（点击文件名或 [Download]） | `AttachmentsTab` 文件名 `<a>` + [Download] 按钮 |
| AC-007C-014 | [+] 和 [Delete] 仅本版本上传人可见 | `AttachmentsTab`（v-if isUploader） |
| AC-007C-015 | 上传成功：表格即时新增，标题计数更新 | `doUpload()` + `$emit('count-change')` |
| AC-007C-016 | 删除二次确认，删除成功：行移除，计数减 1 | `confirmDelete()` 使用 `$confirm` |
| AC-007C-017 | Part Print Tab 只读列表，按发布时间倒序，格式字段完整 | `PartPrintTab.vue` |
| AC-007C-018 | Part Print Tab 无操作按钮（只读） | `PartPrintTab.vue`（无 v-if 操作按钮） |

---

## 12. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.2.5 | 2026-05-26 | agent | 基于 REQ-007C-pc@0.2.5 首次生成。涵盖全部功能：F-001 主列表（RFA No. 列 + Actions 列 + 移除 Attachments 列）、F-003 展开行 4 阶段卡片（5 种状态渲染逻辑 + Status A–E 映射）、F-004 步骤条、F-005 卡片操作按钮（pdfWithQrUrl 降级逻辑）、F-006 附件二级抽屉（480px + Part Print/Attachments 双 Tab + 上传/删除权限控制）、MarkExternalApprovalDialog 复用 |
