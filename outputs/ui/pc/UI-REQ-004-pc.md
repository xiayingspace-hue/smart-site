# UI 说明文档 — PC 管理端图纸局部更新（Markup）

> **来源需求**: [REQ-004-pc](../../../requirements/pc/REQ-004-pc.md) + [REQ-004-shared](../../../requirements/shared/REQ-004-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **设计令牌参考**: [UI-REQ-001-pc § 3.2](./UI-REQ-001-pc.md)
> **依赖文档**: [UI-REQ-003-pc.md](./UI-REQ-003-pc.md)（图纸列表页基础规范）
> **生成日期**: 2026-04-08

---

## 1. 设计目标

- 以最小改动将局部更新能力无缝嵌入已有的图纸列表页，不破坏现有工作流
- 发布局部更新的操作路径短（≤ 2 步），表单简洁，避免与完整版本上传混淆
- 局部更新列表抽屉提供清晰的 Active / Merged 分层视图，并内置汇总操作入口
- 汇总为新版本的弹窗延续 REQ-003 上传弹窗的视觉规范，降低学习成本
- 状态标签和操作按钮视觉区分度高，发布人权限控制明确
- 支持多语言（中文 / English），默认英文界面

---

## 2. 页面清单与用户流程

### 2.1 新增视图 / 面板（在 REQ-003 基础上扩展）

| 序号 | 视图 / 面板 | 类型 | 说明 |
|------|------------|------|------|
| 1 | 发布局部更新弹窗 | el-dialog | 设计人员填写标题、说明、影响区域并上传附件 |
| 2 | 局部更新列表抽屉 | 右侧 el-drawer | 查看某图纸所有局部更新，支持 Active/Merged 筛选，勾选汇总 |
| 3 | 汇总为新版本弹窗 | el-dialog | 确认已选局部更新，上传汇总后图纸文件，选择审批人提交 |

### 2.2 图纸列表页改动点

在 UI-REQ-003-pc §3.2.6 操作按钮组基础上，追加以下按钮：

| 按钮 | 位置 | 显示条件 |
|------|------|---------|
| Markups | Assign 按钮之后 | 所有行（`drawing:view` 权限） |
| + Markup | Markups 按钮之后 | 仅 `status = ACTIVE` 行（`drawing:markup:publish` 权限） |

新增表格列 **Markups**（可选，建议默认展示）：

| 列名 | 字段 | 宽度 | 说明 |
|------|------|------|------|
| Markups | `activeMarkupCount` | 90px | 显示 ACTIVE 数量；0 时显示 `—`；数字为链接样式，点击打开局部更新抽屉 |

### 2.3 核心用户流程

```
设计人员 发布局部更新：

  图纸列表（status=Active）
       │ 点击 [+ Markup]
       ▼
  发布局部更新弹窗
  填写 Title / Description / Affected Area
  上传附件（可选）
       │ [Publish]
       ▼
  弹窗关闭
  Markups 列 +1
  Toast：已通知 Site Engineers
```

```
设计人员 汇总局部更新为新版本：

  图纸列表
       │ 点击 [Markups]
       ▼
  局部更新列表抽屉
  Active Tab → 勾选局部更新（☑ Merge）
       │ 底部汇总栏 → [Merge to New Version →]
       ▼
  汇总弹窗
  确认已选列表
  上传汇总后图纸文件
  填写 Version Note（自动预填）
  选择审批人
       │ [Submit for Approval]
       ▼
  弹窗 + 抽屉关闭
  图纸状态变为 Pending
  审批人收到 Todo
```

---

## 3. 组件设计规范

### 3.1 图纸列表页扩展

#### 3.1.1 操作列完整按钮组

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ [View] [History] [Confirms] [Assign] [Markups] [+ Markup] │ [Upload V{n}]   │
└──────────────────────────────────────────────────────────────────────────────┘
```

| 按钮 | 样式 |
|------|------|
| Markups | el-button size="mini" icon="el-icon-document"，文字 "Markups"；若 `activeMarkupCount > 0`，图标右上角显示橙色计数角标 |
| + Markup | el-button size="mini" type="warning" plain icon="el-icon-edit"，文字 "+ Markup"；仅 status=ACTIVE 时显示 |

> 操作列按钮超出宽度时，折叠规则：优先保留 View 和 Upload V{n}；Markups / + Markup / Confirms / Assign / History 折入 `el-dropdown` 更多菜单。

#### 3.1.2 Markups 列样式

| 状态 | 展示 |
|------|------|
| `activeMarkupCount = 0` | 灰色 `—` |
| `activeMarkupCount > 0` | 橙色数字链接（`--color-warning` #F57F17），hover 下划线，点击打开抽屉 |

---

### 3.2 发布局部更新弹窗

#### 3.2.1 弹窗结构

```
┌──────────────────────────────────────────────────────────┐
│  Publish Markup                                    ✕    │  ← 标题栏
│  ARCH-001 · 首层平面图 · V3                              │  ← 副标题（图纸信息，灰色）
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Title *                                                 │
│  [________________________________________________]      │
│                                                          │
│  Description *                                           │
│  [________________________________________________]      │
│  [                                                ]      │
│  [                                                ]      │
│                                            0 / 2000      │
│                                                          │
│  Affected Area                                           │
│  [________________________________________________]      │
│  e.g. "A-C axis / Level 3-5"                            │
│                                                          │
│  Attachments  (0 / 10)                                   │
│  ┌────────────────────────────────────────────────┐      │
│  │  📎  Drag & Drop or  [Browse Files]            │      │
│  │  PDF / PNG / JPG  ·  Max 20MB per file         │      │
│  └────────────────────────────────────────────────┘      │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                            [Cancel]  [Publish]           │
└──────────────────────────────────────────────────────────┘
```

#### 3.2.2 弹窗属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-dialog` |
| 宽度 | 560px |
| 标题行 | "Publish Markup"，16px，`--font-weight-semibold` |
| 副标题 | "{drawingCode} · {drawingName} · {currentVersionNo}"，13px，`--color-text-secondary`，标题行下方 4px 处 |
| 关闭 | ✕ / Cancel / 遮罩；有内容时弹出二次确认 |

#### 3.2.3 表单字段规格

| 字段 | 组件 | 规格 |
|------|------|------|
| Title | `el-input` | 必填；最大 200 字符；placeholder "Brief description of the change..." |
| Description | `el-input` type="textarea" rows=4 | 必填；最大 2000 字符；右下角灰色计数 "x / 2000"；placeholder "Describe what changed and why..." |
| Affected Area | `el-input` | 可选；最大 500 字符；placeholder "e.g. A-C axis / Level 3-5" |
| Attachments | 自定义多文件上传 | 可选；PDF/PNG/JPG；单文件 ≤20MB；最多 10 个；上传区右上角显示计数 "(x / 10)" |

#### 3.2.4 附件上传区

| 属性 | 规格 |
|------|------|
| 背景 | #F5F7FA，圆角 6px，边框 1px 虚线 #D9D9D9 |
| 高度 | 100px |
| 图标 | 📎 20px，`--color-text-secondary` |
| 主文字 | "Drag & Drop or"，`--color-text-secondary` |
| 按钮 | "Browse Files"，文字链接，`--color-primary` |
| 次文字 | "PDF / PNG / JPG · Max 20MB per file"，12px，`--color-text-secondary` |
| 已选文件列表 | 在上传区下方逐行展示：文件名 + 大小 + 进度条（上传中）/ ✓（完成）/ [✕]（可删除）|
| 达到上限 | 上传区显示禁用状态（背景 #FAFAFA，文字变灰），不可继续添加 |

#### 3.2.5 底部操作行

| 按钮 | 类型 | 规格 |
|------|------|------|
| Cancel | `el-button` | 灰色；有内容时弹关闭确认 |
| Publish | `el-button type="primary"` | 蓝色；必填项未填时 disabled；提交中 loading 防重复点击 |

#### 3.2.6 发布成功 Toast

| 属性 | 规格 |
|------|------|
| 组件 | `el-notification`，右上角，持续 4 秒 |
| 图标 | ✅ 绿色 |
| 标题 | "Markup Published" |
| 内容 | "Assigned Site Engineers have been notified." |

---

### 3.3 局部更新列表抽屉

#### 3.3.1 抽屉结构

```
┌──────────────────────────────────────────────────────────┐
│ ←  Markups — ARCH-001 · 首层平面图                [+ Markup]│
│    Based on V3  ·  2 Active  ·  1 Merged                │  ← 副标题
├──────────────────────────────────────────────────────────┤
│  [All (3)]  [Active (2)]  [Merged (1)]                   │  ← Tab 筛选
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  ☐  [Active]  A轴节点详图修正                       │  │
│  │     Apr 8, 2026  ·  张三                            │  │
│  │     [A-C轴 / 3-5层]                                │  │
│  │     A轴与3轴交叉节点详图已更新，新增钢筋排布...      │  │
│  │     ...Show more                                    │  │
│  │     📎 node-detail.pdf                             │  │
│  │                              [🗑]                   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  ☐  [Active]  C区消防管道路由修正                   │  │
│  │     Apr 7, 2026  ·  张三                            │  │
│  │     [C区 / B1层]                                   │  │
│  │     消防主管道路由变更，详见附图...                  │  │
│  │     📎 route-update.png                            │  │
│  │                              [🗑]                   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │     [Merged → V4]  外墙保温层厚度修正               │  │
│  │     Apr 5, 2026  ·  张三                            │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
├──────────────────────────────────────────────────────────┤
│  ☑ 2 selected            [Merge to New Version →]       │  ← 底部汇总栏（勾选后出现）
└──────────────────────────────────────────────────────────┘
```

#### 3.3.2 抽屉属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-drawer`，direction="rtl" |
| 宽度 | 560px |
| 标题 | "Markups — {drawingCode} · {drawingName}"，`--font-weight-semibold` |
| [+ Markup] 按钮 | 标题行右侧，`el-button size="small" type="primary" plain`，仅 status=ACTIVE 图纸时显示 |
| 副标题 | "Based on {versionNo} · {activeCount} Active · {mergedCount} Merged"，13px，`--color-text-secondary` |
| Tab | el-tabs，标签：All ({total}) / Active ({n}) / Merged ({n}) |
| 内容区背景 | #F5F7FA |
| 内边距 | 0 16px 80px（底部留 80px 给汇总栏） |

#### 3.3.3 局部更新卡片

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF，圆角 4px，边框 1px #EBEEF5，内边距 12px 16px，底部间距 8px |
| 复选框 | `el-checkbox`，仅 Active 卡片左侧显示；Merged 卡片无复选框，左侧缩进对齐 |
| 状态标签 | 见 §3.3.4 |
| 标题 | 14px，`--font-weight-semibold`，`--color-text-regular` |
| 元信息行 | "{date} · {creatorName}"，12px，`--color-text-secondary`，标题下方 4px |
| 影响区域 Tag | `el-tag size="mini"` 默认样式（灰色边框），文字 12px；无影响区域时不显示 |
| 说明文字 | 13px，`--color-text-regular`，最多展示 3 行，超出显示 "...Show more" 文字链接（展开全文） |
| 附件列表 | 📎 图标 + 文件名，12px，`--color-text-secondary`；文件名为链接，点击新标签预览 |
| [🗑 删除] | 卡片右下角，`el-button size="mini" type="text"`，红色 `#C62828` 垃圾桶图标；仅创建人可见；hover 背景 #FFF5F5 |
| Merged 卡片 | 无复选框，无删除按钮；整体透明度 0.75，状态标签样式见 §3.3.4 |

#### 3.3.4 状态标签样式

| 状态 | 文字 | 背景色 | 文字色 | 边框 |
|------|------|--------|--------|------|
| Active | Active | #F0FAF0 | #2E7D32 | #A5D6A7 |
| Merged → V{n} | Merged → V{n} | #F5F5F5 | #757575 | #E0E0E0 |

标签规格：圆角 4px，内边距 2px 8px，12px，`--font-weight-medium`

#### 3.3.5 底部汇总栏

| 属性 | 规格 |
|------|------|
| 显示条件 | 至少勾选 1 个 Active 局部更新时显示；全部取消勾选后隐藏 |
| 位置 | 抽屉底部固定（position: sticky bottom），高度 60px |
| 背景 | 白色 #FFFFFF，顶部边框 1px #EBEEF5，阴影 0 -2px 8px rgba(0,0,0,0.06) |
| 已选计数 | "☑ {n} selected"，13px，`--color-text-secondary`，左对齐 |
| [Merge to New Version →] | `el-button type="primary"`，右对齐；文字 "Merge to New Version →" |

#### 3.3.6 删除确认弹窗

| 属性 | 规格 |
|------|------|
| 组件 | `this.$confirm()`，Element UI MessageBox |
| 内容 | "This markup will be permanently deleted. This action cannot be undone." |
| 取消 | "Cancel" |
| 确认 | "Delete"，`type="danger"` |

---

### 3.4 汇总为新版本弹窗

#### 3.4.1 弹窗结构

```
┌────────────────────────────────────────────────────────────┐
│  Merge to New Version                                ✕    │
│  ARCH-001 · 首层平面图                                     │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Markups to Merge (2)                                      │
│  ┌──────────────────────────────────────────────────┐      │
│  │  ✓  A轴节点详图修正             Apr 8, 2026      │      │
│  │  ✓  C区消防管道路由修正         Apr 7, 2026      │      │
│  └──────────────────────────────────────────────────┘      │
│                                                            │
│  ℹ  Current version: V3. After approval: V4.              │
│                                                            │
│  New Version File *                                        │
│  ┌──────────────────────────────────────────────────┐      │
│  │  📁  Drag & Drop or  [Browse Files]              │      │
│  │  PDF / PNG / JPG  ·  Max 50MB                    │      │
│  └──────────────────────────────────────────────────┘      │
│                                                            │
│  Version Note                                              │
│  [Merged from 2 markups: A轴节点详图修正; C区消防管道...]   │
│                                                            │
│  Approver *                                                │
│  [Select approver...             ▼]                        │
│                                                            │
├────────────────────────────────────────────────────────────┤
│                       [Cancel]  [Submit for Approval]      │
└────────────────────────────────────────────────────────────┘
```

#### 3.4.2 弹窗属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-dialog` |
| 宽度 | 580px |
| 标题行 | "Merge to New Version"，16px，`--font-weight-semibold` |
| 副标题 | "{drawingCode} · {drawingName}"，13px，`--color-text-secondary` |

#### 3.4.3 字段规格

| 字段 | 组件 | 规格 |
|------|------|------|
| Markups to Merge | 只读列表区 | 浅灰背景 #F5F7FA，圆角 4px，内边距 8px 12px；每行：✓ 图标（绿色）+ 标题 + 右对齐日期；最大高度 160px，超出内部滚动 |
| 版本提示 | `el-alert type="info"` | "Current version: V{n}. After approval, this will become V{n+1}."；不可关闭；蓝色信息样式 |
| New Version File | 自定义上传区 | 与 REQ-003 上传弹窗规范一致；必填；PDF/PNG/JPG；≤50MB |
| Version Note | `el-input` | 自动预填 "Merged from {n} markups: {title1}; {title2}..."；可手动修改；最大 500 字符 |
| Approver | `el-select filterable` | 必填；从 `/member/list?permission=drawing:approve` 拉取 |

#### 3.4.4 底部操作行

| 按钮 | 类型 | 规格 |
|------|------|------|
| Cancel | `el-button` | 灰色，关闭弹窗 |
| Submit for Approval | `el-button type="primary"` | 蓝色；File / Approver 未填时 disabled；提交中 loading |

#### 3.4.5 提交成功 Toast

| 属性 | 规格 |
|------|------|
| 组件 | `el-notification`，右上角，持续 5 秒 |
| 标题 | "Submitted for Approval" |
| 内容 | "Version note includes {n} markups. The approver has been notified." |

#### 3.4.6 图纸已有版本待审批时的错误提示

| 属性 | 规格 |
|------|------|
| 组件 | 弹窗内顶部 `el-alert type="error"`，不可关闭 |
| 文字 | "A version of this drawing is already pending approval. Please wait for the review to complete." |
| 触发时机 | 点击 Submit 后接口返回错误码 `1003004011` 时显示 |

---

## 4. 多语言文案

### 4.1 图纸列表页扩展

| Key | English | 中文 |
|-----|---------|------|
| action_markups | Markups | 局部更新 |
| action_add_markup | + Markup | + 局部更新 |
| col_markups | Markups | 局部更新 |

### 4.2 发布局部更新弹窗

| Key | English | 中文 |
|-----|---------|------|
| publish_dialog_title | Publish Markup | 发布局部更新 |
| field_title | Title | 标题 |
| field_title_placeholder | Brief description of the change... | 简要描述本次改动... |
| field_description | Description | 更新说明 |
| field_description_placeholder | Describe what changed and why... | 详细描述改动内容和原因... |
| field_affected_area | Affected Area | 影响区域 |
| field_affected_area_placeholder | e.g. A-C axis / Level 3-5 | 如：A-C轴 / 3-5层 |
| field_attachments | Attachments | 附件 |
| upload_hint_markup | PDF / PNG / JPG · Max 20MB per file | PDF / PNG / JPG · 每个文件最大 20MB |
| validation_title_required | Title is required | 请填写标题 |
| validation_description_required | Description is required | 请填写更新说明 |
| validation_attachment_format | Only PDF, PNG, JPG files are allowed | 仅支持 PDF、PNG、JPG 格式 |
| validation_attachment_size | File size cannot exceed 20MB | 单个文件不能超过 20MB |
| validation_attachment_limit | Maximum 10 attachments allowed | 最多上传 10 个附件 |
| btn_publish | Publish | 发布 |
| toast_markup_published_title | Markup Published | 局部更新已发布 |
| toast_markup_published_body | Assigned Site Engineers have been notified. | 已通知相关 Site Engineer。 |

### 4.3 局部更新列表抽屉

| Key | English | 中文 |
|-----|---------|------|
| drawer_title_markups | Markups — {code} · {name} | 局部更新 — {code} · {name} |
| drawer_subtitle_markups | Based on {version} · {active} Active · {merged} Merged | 基于 {version} · {active} 生效中 · {merged} 已汇总 |
| tab_all | All ({n}) | 全部 ({n}) |
| tab_active | Active ({n}) | 生效中 ({n}) |
| tab_merged | Merged ({n}) | 已汇总 ({n}) |
| label_status_active | Active | 生效中 |
| label_status_merged | Merged → {version} | 已汇总 → {version} |
| label_show_more | ...Show more | ...展开全文 |
| label_show_less | Show less | 收起 |
| btn_delete_markup | Delete | 删除 |
| confirm_delete_title | Delete Markup | 删除局部更新 |
| confirm_delete_body | This markup will be permanently deleted. This action cannot be undone. | 该局部更新将被永久删除，此操作不可撤销。 |
| label_selected_count | {n} selected | 已选 {n} 条 |
| btn_merge_to_version | Merge to New Version → | 汇总为新版本 → |

### 4.4 汇总为新版本弹窗

| Key | English | 中文 |
|-----|---------|------|
| merge_dialog_title | Merge to New Version | 汇总为新版本 |
| merge_markups_label | Markups to Merge ({n}) | 待汇总的局部更新 ({n}) |
| merge_version_tip | Current version: {cur}. After approval, this will become {next}. | 当前版本：{cur}。审批通过后将升级为 {next}。 |
| field_new_version_file | New Version File | 新版图纸文件 |
| field_version_note | Version Note | 版本说明 |
| version_note_placeholder | Merged from {n} markups: {titles} | 汇总自 {n} 条局部更新：{titles} |
| field_approver | Approver | 审批人 |
| btn_submit_approval | Submit for Approval | 提交审批 |
| toast_merge_success_title | Submitted for Approval | 已提交审批 |
| toast_merge_success_body | Version note includes {n} markups. The approver has been notified. | 版本说明已包含 {n} 条局部更新，审批人已收到通知。 |
| error_pending_version | A version of this drawing is already pending approval. Please wait for the review to complete. | 该图纸已有版本正在审批中，请等待审批完成后再提交。 |

---

## 5. 响应式与分辨率适配

| 项目 | 规格 |
|------|------|
| 发布局部更新弹窗 | 固定宽度 560px，内容区高度超出时内部滚动 |
| 局部更新列表抽屉 | 固定宽度 560px，不随分辨率变化 |
| 汇总弹窗 | 固定宽度 580px，内容区高度超出时内部滚动 |
| 操作列折叠 | 1280px 最小宽度下，Markups / + Markup 折入更多菜单 |

---

## 6. 验收条件

### 6.1 图纸列表页扩展

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | Markups 列显示 | Active 状态数量以橙色数字链接展示，0 时显示 `—` |
| 2 | [Markups] 按钮 | 所有行可见，点击打开对应图纸的局部更新抽屉 |
| 3 | [+ Markup] 按钮 | 仅 status=ACTIVE 行显示，其他状态行不显示 |
| 4 | 权限控制 | 无 `drawing:markup:publish` 权限时，[+ Markup] 按钮不可见 |

### 6.2 发布局部更新弹窗

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 弹窗标题 | 副标题正确显示图纸编号、名称和当前版本号 |
| 2 | 必填校验 | Title / Description 为空时 [Publish] 禁用，提交后显示 inline 错误提示 |
| 3 | 字数限制 | Description 超 2000 字时阻止输入并提示；计数器实时更新 |
| 4 | 附件格式 | 非 PDF/PNG/JPG 文件被拦截，显示格式错误提示 |
| 5 | 附件大小 | 超 20MB 文件被拦截，显示大小错误提示 |
| 6 | 附件上限 | 已有 10 个附件时上传区进入禁用状态 |
| 7 | 发布成功 | 弹窗关闭，Markups 列 +1，Toast 正确显示 |
| 8 | 关闭确认 | 有内容时点击取消或遮罩弹出二次确认 |

### 6.3 局部更新列表抽屉

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 数据加载 | 打开时正确展示所有局部更新，All / Active / Merged Tab 数量准确 |
| 2 | 状态标签 | Active 绿色，Merged 灰色且显示目标版本号 |
| 3 | 说明展开 | 超 3 行内容显示 "...Show more"，点击展开全文 |
| 4 | 附件预览 | 点击附件文件名在新标签页打开预览 |
| 5 | 复选框 | 仅 Active 卡片有复选框，Merged 卡片无 |
| 6 | 底部汇总栏 | 有勾选时出现，全部取消后消失；数量计数准确 |
| 7 | 删除按钮可见性 | 仅卡片创建人可见删除按钮；非创建人不可见 |
| 8 | 删除流程 | 点击删除 → 二次确认 → 确认后卡片消失，Markups 列 -1 |

### 6.4 汇总为新版本弹窗

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 已选列表 | 只读展示正确的局部更新标题和日期 |
| 2 | 版本提示 | 正确显示当前版本号和下一版本号 |
| 3 | Version Note | 自动预填汇总摘要；内容可手动修改 |
| 4 | 必填校验 | 未选文件或未选审批人时 [Submit for Approval] 禁用 |
| 5 | 提交成功 | 弹窗和抽屉均关闭，图纸状态变为 Pending，Toast 正确显示 |
| 6 | 已有待审批版本 | 提交后接口报错，弹窗内顶部显示红色错误提示，弹窗不关闭 |

---

## 7. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-004-pc.md](../../../requirements/pc/REQ-004-pc.md) | PC 端产品需求 |
| [REQ-004-shared.md](../../../requirements/shared/REQ-004-shared.md) | 跨端共享业务规则与 API |
| [UI-REQ-003-pc.md](./UI-REQ-003-pc.md) | 图纸列表页基础 UI 规范（本功能依赖） |
