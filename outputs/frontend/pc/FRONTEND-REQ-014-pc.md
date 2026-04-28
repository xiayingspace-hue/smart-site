# 前端开发说明文档 — REQ-014 图纸管理用户反馈迭代（PC 端）

> **来源需求**: `requirements/pc/REQ-014-pc.md`
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue2 + Element UI）
> **生成日期**: 2026-04-28

---

## 1. 功能概述

本文档覆盖 REQ-003/004/005 第一版用户反馈的 PC 端前端改动，共 6 条反馈：

| 反馈编号 | 改动模块 | 改动类型 |
|----------|----------|----------|
| FB-001 | 图纸管理 — 上传新版本弹框 | 字段交互调整 |
| FB-002 | 图纸管理 — 版本详情弹框 | 新增附件区域 |
| FB-003 | 图纸管理 — 图纸列表操作列 | 入口位置调整 |
| FB-004 | Markup 历史弹框 | 新增列字段 |
| FB-005 | Markup 历史入口 | 权限收回（SE 隐藏） |
| FB-006 | 图纸列表 / 版本详情 | SE 视角字段显示控制 |

---

## 2. 涉及页面与组件

### 2.1 图纸管理页（Drawing Management）

- **路由**: 沿用 REQ-003 现有路由
- **涉及组件**:
  - `DrawingList.vue` — 图纸列表主页，操作列调整（FB-003、FB-005、FB-006）
  - `UploadVersionDialog.vue` — 上传新版本弹框（FB-001）
  - `VersionDetailDialog.vue` — 版本详情弹框（FB-002、FB-006）
  - `MarkupHistoryDialog.vue` — Markup 历史记录弹框（FB-003、FB-004、FB-005）

---

## 3. 各反馈改动详情

---

### 3.1 FB-001 上传新版本弹框 — Drawing Code 字段可编辑

**涉及组件**: `UploadVersionDialog.vue`

**改动说明**:
- `Drawing Code` 输入框从只读（`disabled`）改为可编辑（`readonly` 属性移除）
- 默认值回填当前图纸的 Drawing Code
- 添加实时唯一性校验（`blur` 事件触发，或提交时校验）：
  - 调用接口检查当前项目下是否已存在相同 Drawing Code
  - 若重复，输入框下方显示错误提示：`"该 Drawing Code 已存在，请重新输入"`，提交按钮 `disabled`
- 修改后的 Drawing Code 仅影响本次新版本，不修改历史版本数据

**字段状态**:

| 状态 | 行为 |
|------|------|
| 默认 | 回填当前 Drawing Code，可修改 |
| 输入值与其他图纸冲突 | 红色边框 + 错误提示，提交禁用 |
| 输入值合法 | 恢复正常样式，提交可用 |

---

### 3.2 FB-002 版本详情弹框 — 新增附件区域

**涉及组件**: `VersionDetailDialog.vue`（新增子区域）

**改动说明**:
- 在版本详情弹框 PDF 主文件信息下方，新增 `Attachments` 区域
- 区域结构：标题 `附件（Attachments）` + `[+ 上传附件]` 按钮 + 附件列表
- 附件列表字段：文件名、类型（文件后缀）、大小、上传时间、备注、操作

**附件列表渲染**:
```vue
<!-- 附件列表 columns -->
[{ label: '文件名', prop: 'name' },
 { label: '类型', prop: 'type' },
 { label: '大小', prop: 'size' },
 { label: '上传时间', prop: 'uploadedAt' },
 { label: '备注', prop: 'remark' },
 { label: '操作', slot: 'action' }]
```

**上传附件交互**:
1. 点击 `[+ 上传附件]` → 调起文件选择器（`el-upload`，不限文件类型，支持多选）
2. 弹出备注输入框（可选填）
3. 确认后调用上传接口，上传成功后刷新附件列表

**删除附件**:
- 二次确认 `el-dialog`：`"确认删除该附件？"`
- 确认后调用删除接口，移除列表项

**权限控制**:
- `[+ 上传附件]` 按钮：仅 PDF 上传人 / 有图纸管理权限的角色可见
- 操作列 `[删除]`：同上，SE 不显示
- 操作列 `[下载]`：所有有图纸访问权限角色可见（含 SE）

---

### 3.3 FB-003 图纸列表操作列 — 移除 "+Markup" 按钮，调整 Markup 历史弹框

**涉及组件**: `DrawingList.vue`、`MarkupHistoryDialog.vue`

**操作列改动**:
- 移除操作列中的 `+Markup` 按钮（对所有有权限角色）
- 保留"Markup 历史"图标按钮（非 SE 角色可见，参见 FB-005）

**`MarkupHistoryDialog.vue` 改动**:
- 弹框右上角（或顶部操作区）新增 `[+ Add Markup]` 按钮
- 点击 `[+ Add Markup]` 触发原 `+Markup` 的上传流程（复用原有逻辑，调用 REQ-004 相关弹框/方法）

---

### 3.4 FB-004 Markup 历史列表 — 新增 "Drawing Code" 列

**涉及组件**: `MarkupHistoryDialog.vue`

**改动说明**:
- Markup 列表新增 `Drawing Code` 列，插入在"文件名"列之后
- 数据来源：接口返回字段 `drawing_code`（后端在创建 Markup 时自动关联，参见后端文档）
- 该字段只读，纯展示

**列表字段顺序（更新后）**:

| 列 | 字段 | 说明 |
|----|------|------|
| # | 序号 | - |
| 文件名 | `fileName` | Markup 文件名 |
| Drawing Code | `drawingCode` | 自动记录，只读 |
| 上传时间 | `uploadedAt` | - |
| 操作 | - | 查看、合并 |

---

### 3.5 FB-005 SE 角色完全隐藏 Markup 相关入口

**涉及组件**: `DrawingList.vue`

**改动说明**:
- 根据当前用户角色（`userRole === 'SE'`）控制以下元素的渲染（`v-if`，非 `v-show`）：
  - 操作列"Markup 历史"图标按钮：SE 不渲染
  - Markup 数量角标（如有）：SE 不渲染
- SE 无法通过任何 UI 入口触达 Markup 相关弹框

```vue
<!-- 操作列 Markup 历史按钮 -->
<el-button
  v-if="!isSE"
  icon="el-icon-document"
  @click="openMarkupHistory(row)"
/>
```

---

### 3.6 FB-006 SE 视角隐藏系统版本号（Ver）

**涉及组件**: `DrawingList.vue`、`VersionDetailDialog.vue`

**图纸列表 — 版本列**:
- 非 SE：显示 `Drawing Code` + `Ver`（系统版本号）
- SE：仅显示 `Drawing Code`，隐藏 `Ver` 列或合并显示

**版本详情弹框 — 信息区**:
- 非 SE：显示 `Drawing Code`、`Ver`、上传时间等
- SE：`Ver` 字段行不渲染（`v-if="!isSE"`），不留空白占位

```vue
<!-- 版本详情中系统版本号 -->
<el-form-item v-if="!isSE" label="系统版本（Ver）">
  {{ versionDetail.ver }}
</el-form-item>
```

---

## 4. 权限控制汇总

| 功能 | 上传人 / PM / CM | SE |
|------|------------------|----|
| 上传新版本时修改 Drawing Code | ✅ | ❌（只读） |
| 附件上传 / 删除 | ✅（有权限者） | ❌ |
| 附件下载 | ✅ | ✅（已分配版本） |
| Markup 历史入口 | ✅ | ❌ `v-if` 隐藏 |
| + Add Markup | ✅（有权限者） | ❌ |
| Markup 列表 Drawing Code 列 | ✅ | ❌（无入口） |
| 系统版本号（Ver）显示 | ✅ | ❌ `v-if` 隐藏 |

---

## 5. 接口依赖

| 方法 | 路径 | 说明 | 反馈编号 |
|------|------|------|----------|
| PUT | `/api/drawings/:drawingId/versions/:versionId` | 上传新版本（支持修改 Drawing Code） | FB-001 |
| GET | `/api/drawings/check-code?code=&projectId=` | Drawing Code 唯一性校验 | FB-001 |
| GET | `/api/drawings/:drawingId/versions/:versionId/attachments` | 获取版本附件列表 | FB-002 |
| POST | `/api/drawings/:drawingId/versions/:versionId/attachments` | 上传附件 | FB-002 |
| DELETE | `/api/drawings/:drawingId/versions/:versionId/attachments/:attachmentId` | 删除附件 | FB-002 |
| GET | `/api/drawings/:drawingId/attachments/:attachmentId/download` | 下载附件 | FB-002 |
| GET | `/api/drawings/:drawingId/markups` | 获取 Markup 列表（含 drawing_code 字段） | FB-004 |

---

## 6. 重点注意事项

- FB-001：Drawing Code 唯一性校验需防抖，避免频繁请求；提交时也需二次校验
- FB-002：附件上传使用 `el-upload` 的 `http-request` 自定义上传，支持上传进度显示
- FB-003：`+Markup` 按钮从操作列移除后，原有操作列宽度需重新评估，避免布局变化过大
- FB-005：使用 `v-if` 而非 `v-show`，确保 SE 视角 DOM 中不存在 Markup 相关元素
- FB-006：`isSE` 工具方法统一从全局 store 中获取当前用户角色，避免重复判断逻辑分散

---

> 本文档配合 `REQ-014-pc.md` 需求文档及 `UI-REQ-014-pc.md` UI 说明文档使用。
