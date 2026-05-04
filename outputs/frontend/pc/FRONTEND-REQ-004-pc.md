# 前端开发说明文档 — PC 管理端图纸局部更新（Markup）

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-004-pc.md](../../../requirements/pc/REQ-004-pc.md) + [REQ-004-shared.md](../../../requirements/shared/REQ-004-shared.md)
> **UI 设计参考**: [UI-REQ-004-pc.md](../../ui/pc/UI-REQ-004-pc.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

本模块在 REQ-003 图纸管理页面基础上扩展局部更新（Markup）功能，涵盖：

| 功能模块 | 说明 |
|---------|------|
| 图纸表格扩展 | 新增 Markups 列（Active 数量）+ 操作列新增 [Markups] / [+ Markup] 按钮 |
| 发布局部更新弹窗 | 表单弹窗：Title / Description / Affected Area / 多附件上传 |
| 局部更新列表抽屉 | 右侧抽屉展示图纸所有 Markup，All/Active/Merged Tab 分组 |
| Markup 卡片操作 | Active 卡片支持勾选（汇总）/ 删除（仅创建人） |
| 汇总为新版本弹窗 | 勾选多条 Active Markup → 上传新版本文件 + 选审批人 → 触发审批流 |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 + Vue Router + Vuex |
| UI 组件库 | Element UI 2.x |
| HTTP | Axios，统一封装 `request.js`，自动附带 `Authorization` / `X-Tenant-Id` / `Project-Id` / `lang` |
| 文件上传 | `el-upload` + 自定义 `http-request`，直传 OSS |
| 路由 | 本模块无独立路由，集成在 `/project/:projectId/drawings` 页面 |
| 权限指令 | `v-permission="'drawing:markup:publish'"` 等 |
| 国际化 | Vue I18n |

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── drawing/
│       ├── index.vue                          # 图纸列表页（在 REQ-003 基础上扩展）
│       └── components/
│           ├── DrawingTable.vue               # 图纸表格（新增 Markups 列 + 操作按钮）
│           ├── PublishMarkupDialog.vue        # 发布局部更新弹窗（新增）
│           ├── MarkupListDrawer.vue           # 局部更新列表抽屉（新增）
│           ├── MarkupCard.vue                 # 局部更新卡片（新增）
│           └── MergeMarkupDialog.vue          # 汇总为新版本弹窗（新增）
├── api/
│   └── drawing-markup.js                     # 局部更新相关 API 封装（新增）
└── utils/
    └── drawing-helper.js                     # 扩展：Markup 状态枚举映射
```

---

## 4. 图纸表格扩展 `DrawingTable.vue`

### 4.1 新增列

在 REQ-003 图纸表格 `Actions` 列前插入以下列：

| 列名 | 字段 | 宽度 | 说明 |
|------|------|------|------|
| Markups | `activeMarkupCount` | 90px | 显示 `ACTIVE` 状态 Markup 数量；0 时显示 `—`；点击触发打开局部更新抽屉 |

### 4.2 Actions 列扩展

在原有操作按钮（View / History / Confirms / Assign / Upload V{n}）基础上新增：

| 按钮 | 显示条件 | 权限 |
|------|---------|------|
| `[Markups]` | 始终显示 | `drawing:view` |
| `[+ Markup]` | `row.status === 'ACTIVE'` | `drawing:markup:publish` |

**完整 Actions 顺序**：
```
[View]  [History]  [Confirms]  [Assign]  [Markups]  [+ Markup]  [Upload V{n}]
```

> `[+ Markup]` 当 `status !== 'ACTIVE'` 时使用 `v-if` 隐藏（非置灰）。

### 4.3 Markups 列交互

```javascript
// DrawingTable.vue
handleMarkupsClick(row) {
  this.$emit('open-markups', row)
}
```

---

## 5. 图纸列表页 `index.vue` 扩展

### 5.1 新增数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `publishMarkupVisible` | Boolean | 发布局部更新弹窗显示 |
| `publishMarkupTarget` | Object \| null | 当前发布 Markup 的图纸对象 |
| `markupListVisible` | Boolean | 局部更新列表抽屉显示 |
| `markupListTarget` | Object \| null | 当前查看 Markup 列表的图纸对象 |

### 5.2 新增方法

| 方法 | 说明 |
|------|------|
| `handleOpenMarkups(row)` | `markupListTarget = row`，`markupListVisible = true` |
| `handlePublishMarkup(row)` | `publishMarkupTarget = row`，`publishMarkupVisible = true` |
| `handleMarkupPublished()` | 关闭发布弹窗，刷新当前图纸行的 `activeMarkupCount`（局部刷新或全量重加载） |
| `handleMarkupMerged()` | 关闭抽屉，全量刷新图纸列表（图纸状态变为 PENDING_APPROVAL） |

### 5.3 模板结构扩展

```vue
<template>
  <!-- 原有组件 -->
  <DrawingTable
    @open-markups="handleOpenMarkups"
    @publish-markup="handlePublishMarkup"
  />
  <!-- 新增组件 -->
  <PublishMarkupDialog
    :visible.sync="publishMarkupVisible"
    :drawing="publishMarkupTarget"
    @success="handleMarkupPublished"
  />
  <MarkupListDrawer
    :visible.sync="markupListVisible"
    :drawing="markupListTarget"
    @merged="handleMarkupMerged"
    @publish-markup="handlePublishMarkup"
  />
</template>
```

---

## 6. 发布局部更新弹窗 `PublishMarkupDialog.vue`

### 6.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 控制显示，`.sync` 修饰符 |
| `drawing` | Object | 图纸对象，含 `drawingCode`、`drawingName`、`currentVersionId`、`currentVersionNo` |

### 6.2 Emits

| 事件 | 说明 |
|------|------|
| `success` | 发布成功，父组件更新 activeMarkupCount |
| `update:visible` | 关闭弹窗 |

### 6.3 表单字段与校验

| 字段 | 组件 | 校验 |
|------|------|------|
| Title | `el-input` | 必填，≤ 200 字符 |
| Description | `el-input` type="textarea" rows=4 | 必填，≤ 2000 字符，右下角显示字数 |
| Affected Area | `el-input` | 可选，≤ 500 字符，placeholder: "e.g. A-C axis / Level 3-5" |
| Attachments | 自定义上传区 | 可选；格式 PDF/PNG/JPG；单文件 ≤ 20MB；最多 10 个 |

### 6.4 弹窗标题

```javascript
computed: {
  dialogTitle() {
    return `Publish Markup — ${this.drawing?.drawingCode} · ${this.drawing?.drawingName} · ${this.drawing?.currentVersionNo}`
  }
}
```

### 6.5 附件上传组件

```vue
<el-upload
  multiple
  :limit="10"
  :before-upload="validateAttachment"
  :http-request="customUpload"
  :file-list="attachmentList"
  :on-exceed="handleExceedLimit"
>
  <div class="upload-area">
    <i class="el-icon-paperclip" />
    <span>Drag & Drop or <em>Browse Files</em></span>
    <span class="hint">Supports: PDF / PNG / JPG · Max 20MB each</span>
  </div>
</el-upload>
```

附件校验：
```javascript
validateAttachment(file) {
  const allowedTypes = ['application/pdf', 'image/png', 'image/jpeg']
  if (!allowedTypes.includes(file.type)) {
    this.$message.error('Only PDF, PNG, JPG files are allowed')
    return false
  }
  if (file.size > 20 * 1024 * 1024) {
    this.$message.error('File size must not exceed 20MB')
    return false
  }
  return true
}
```

### 6.6 发布流程

```
用户点击 [Publish]
  → el-form.validate()
  → 验证失败 → 标红字段 + 停止
  → 验证通过 → publishLoading = true
  → 逐个上传附件（展示每个文件上传进度）
  → 所有附件上传完成后调用 POST /drawing/markup/publish
  → 成功:
      emit('success')
      关闭弹窗
      Toast: "Markup published. Assigned Site Engineers have been notified."
  → 失败: 显示错误信息，publishLoading = false
```

### 6.7 关闭确认

```javascript
handleClose() {
  const isDirty = this.form.title || this.form.description || this.attachmentList.length > 0
  if (isDirty) {
    this.$confirm(
      'Unsaved changes will be lost. Are you sure you want to close?',
      'Confirm',
      { type: 'warning' }
    ).then(() => this.$emit('update:visible', false))
  } else {
    this.$emit('update:visible', false)
  }
}
```

---

## 7. 局部更新列表抽屉 `MarkupListDrawer.vue`

### 7.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 控制显示，`.sync` |
| `drawing` | Object | 图纸对象 |

### 7.2 Emits

| 事件 | 说明 |
|------|------|
| `merged` | 汇总操作完成，父组件刷新图纸列表 |
| `publish-markup` | 用户点击抽屉内 [+ Markup] 按钮，传递图纸对象 |
| `update:visible` | 关闭抽屉 |

### 7.3 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `activeTab` | String | `'all'` / `'active'` / `'merged'` |
| `markupList` | Array | 当前 Tab 下的 Markup 列表 |
| `loading` | Boolean | 列表 loading |
| `selectedMarkupIds` | Array | 已勾选的 Markup ID（用于汇总） |
| `mergeDialogVisible` | Boolean | 汇总弹窗显示 |

### 7.4 El-Drawer 属性

```javascript
{
  direction: 'rtl',
  size: '560px',
  destroyOnClose: false,  // 保持勾选状态
  wrapperClosable: false  // 防止误触关闭（有勾选时）
}
```

### 7.5 Tab 切换与数据加载

```javascript
watch: {
  activeTab() { this.fetchMarkups() },
  visible(val) { if (val) this.fetchMarkups() }
},
methods: {
  fetchMarkups() {
    const statusFilter = {
      all: undefined,
      active: 'ACTIVE',
      merged: 'MERGED'
    }[this.activeTab]
    getMarkupList({ drawingId: this.drawing.id, status: statusFilter })
      .then(res => { this.markupList = res.list })
  }
}
```

### 7.6 底部汇总栏

```vue
<div v-if="selectedMarkupIds.length > 0" class="merge-bar">
  <span>{{ selectedMarkupIds.length }} markup(s) selected</span>
  <el-button type="primary" @click="mergeDialogVisible = true">
    Merge to New Version →
  </el-button>
</div>
```

---

## 8. 局部更新卡片 `MarkupCard.vue`

### 8.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `markup` | Object | Markup 数据 |
| `selected` | Boolean | 是否已被勾选（汇总） |
| `currentUserId` | Number | 当前登录用户 ID（用于判断是否为创建人） |

### 8.2 Emits

| 事件 | 参数 | 说明 |
|------|------|------|
| `toggle-select` | `markupId` | 勾选/取消勾选 |
| `delete` | `markupId` | 删除 Markup |
| `preview-attachment` | `{ url, name }` | 预览附件 |

### 8.3 卡片状态显示

| 状态 | 标签 | 颜色 | 显示操作 |
|------|------|------|---------|
| `ACTIVE` | `Active` | 绿色 `#52C41A` | [☑ Merge 复选框] + [🗑 Delete]（仅创建人） |
| `MERGED` | `Merged → V{n}` | 灰色 `#999999` | 无操作按钮 |

### 8.4 说明文字截断

```vue
<div
  :class="['description', { expanded: descExpanded }]"
>{{ markup.description }}</div>
<span
  v-if="isDescOverflow"
  class="show-more"
  @click="descExpanded = !descExpanded"
>{{ descExpanded ? 'Show less' : '...Show more' }}</span>
```

CSS：`.description { display: -webkit-box; -webkit-line-clamp: 3; overflow: hidden; }`  
`.description.expanded { -webkit-line-clamp: unset; }`

### 8.5 删除确认

```javascript
handleDelete() {
  this.$confirm(
    'Are you sure you want to delete this markup?',
    'Delete Markup',
    { type: 'warning', confirmButtonText: 'Delete', confirmButtonClass: 'el-button--danger' }
  ).then(() => {
    deleteMarkup(this.markup.id).then(() => {
      this.$emit('delete', this.markup.id)
      this.$message.success('Markup deleted')
    })
  })
}
```

---

## 9. 汇总为新版本弹窗 `MergeMarkupDialog.vue`

### 9.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 控制显示 |
| `drawing` | Object | 图纸对象 |
| `selectedMarkups` | Array | 已勾选的 Markup 对象列表 |

### 9.2 Emits

| 事件 | 说明 |
|------|------|
| `success` | 汇总成功，父组件刷新 |
| `update:visible` | 关闭弹窗 |

### 9.3 表单字段

| 字段 | 组件 | 规格 |
|------|------|------|
| Selected Markups | 只读列表 | 展示勾选的 Markup 标题 + 日期 |
| New Version File | `el-upload`（单文件） | 必填，PDF/PNG/JPG，≤ 50MB |
| 版本提示 | 蓝色 `el-alert` | 只读，"Current version: V{n}. After approval, this will become V{n+1}." |
| Version Note | `el-input` type="textarea" | 必填，预填 "Merged from {n} markups: {titles}"，≤ 500 字符 |
| Approver | `el-select` filterable | 必填，从 `/system/user/list-approver` 获取 |

### 9.4 Version Note 预填

```javascript
computed: {
  defaultVersionNote() {
    const titles = this.selectedMarkups.map(m => m.title).join('; ')
    return `Merged from ${this.selectedMarkups.length} markups: ${titles}`
  }
},
created() {
  this.form.versionNote = this.defaultVersionNote
}
```

### 9.5 提交流程

```
用户点击 [Submit for Approval]
  → 表单验证
  → 验证失败 → 停止
  → 上传新版本文件
  → 调用 POST /drawing/markup/merge
  → 成功:
      关闭弹窗，关闭局部更新抽屉
      Toast: "Submitted for approval. Version note includes {n} markups."
      emit('success')
  → 失败（已有版本待审批，错误码 DRAWING_PENDING_APPROVAL）:
      显示 el-alert: "A version of this drawing is already pending approval. Please wait for the review to complete."
```

---

## 10. API 封装 `api/drawing-markup.js`

```javascript
// 发布局部更新
export const publishMarkup = (data) =>
  request.post('/drawing/markup/publish', data)

// 获取局部更新列表
export const getMarkupList = (params) =>
  request.get('/drawing/markup/list', { params })

// 删除局部更新
export const deleteMarkup = (id) =>
  request.delete('/drawing/markup/delete', { params: { id } })

// 汇总局部更新为新版本
export const mergeMarkups = (data) =>
  request.post('/drawing/markup/merge', data)
```

---

## 11. 国际化（i18n）

```
markup.btn.publish         → "Publish Markup"
markup.btn.markups         → "Markups"
markup.btn.merge           → "Merge to New Version"
markup.status.active       → "Active"
markup.status.merged       → "Merged → V{version}"
markup.form.title          → "Title"
markup.form.description    → "Description"
markup.form.affectedArea   → "Affected Area"
markup.form.attachments    → "Attachments"
markup.toast.published     → "Markup published. Assigned Site Engineers have been notified."
markup.toast.merged        → "Submitted for approval. Version note includes {n} markups."
markup.alert.pendingExists → "A version of this drawing is already pending approval. Please wait for the review to complete."
```

---

## 12. 权限控制

| 操作 | 权限 Key |
|------|---------|
| 查看 Markups 列 / 打开抽屉 | `drawing:view` |
| 发布局部更新 | `drawing:markup:publish` |
| 删除局部更新 | 创建人本身（后端控制）+ 需 `ACTIVE` 状态 |
| 汇总为新版本 | `drawing:markup:publish`（复用，或新增专属权限） |

---

## 13. 验收标准

### 图纸表格扩展
- [ ] 图纸列表新增 Markups 列，显示 ACTIVE 数量；0 时显示 `—`
- [ ] Actions 列新增 [Markups] 和 [+ Markup] 按钮
- [ ] `[+ Markup]` 仅在 `status = ACTIVE` 时显示
- [ ] 无 `drawing:markup:publish` 权限时不显示 `[+ Markup]`

### 发布局部更新
- [ ] Title / Description 为空时阻止提交
- [ ] 附件格式非 PDF/PNG/JPG 时显示错误
- [ ] 附件超 20MB 时显示错误
- [ ] 附件数量超 10 个后禁止继续添加
- [ ] 所有附件上传后再调用发布接口
- [ ] 发布成功后表格 Markups 列 +1，Toast 正确显示
- [ ] 有内容时关闭弹窗弹出确认框

### 局部更新列表抽屉
- [ ] All / Active / Merged Tab 正确过滤显示
- [ ] Active 卡片显示 [☑ Merge] 和 [🗑 Delete]（仅创建人可见 Delete）
- [ ] Merged 卡片显示 "Merged → V{n}"，无操作按钮
- [ ] 勾选后底部汇总栏出现，取消全部勾选后隐藏
- [ ] 说明文字超 3 行时显示 "...Show more"，点击展开

### 删除局部更新
- [ ] 删除前弹出确认，确认后从列表移除
- [ ] 删除后表格 Markups 列数字 -1

### 汇总为新版本
- [ ] 弹窗正确展示已选 Markup 列表（只读）
- [ ] Version Note 预填正确
- [ ] 版本提示显示当前版本和下一版本号
- [ ] New Version File 和 Approver 为必填
- [ ] 汇总成功后图纸状态变为 PENDING_APPROVAL，Markups 列 Active 数量减少
- [ ] 若已有版本待审批则显示阻止提示
