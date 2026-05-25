# UI 说明文档 — PC 管理端图纸局部更新（Part Print）

> **来源需求**: [REQ-004-pc](../../../requirements/pc/REQ-004-pc.md) + [REQ-004-shared](../../../requirements/shared/REQ-004-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **设计令牌参考**: [UI-REQ-001-pc § 3.2](./UI-REQ-001-pc.md)
> **依赖文档**: [UI-REQ-003-pc.md](./UI-REQ-003-pc.md)（图纸列表页基础规范）
> **生成日期**: 2026-05-25

---

## 1. 设计目标

- 以最小改动将局部更新能力无缝嵌入已有的图纸列表页，不破坏现有工作流
- 发布入口收敛至 Part Print 抽屉内，避免 Actions 列过长
- AI 自动识别 Title Block 信息，减少手动填写负担，✨ 标识帮助用户快速核验
- 局部更新列表抽屉提供搜索与 Tab 筛选，快速定位目标记录
- 每条卡片展示所属报审号，用户可清晰知晓局部更新对应的版本基础
- 状态标签和操作按钮视觉区分度高，发布人权限控制明确
- 支持多语言（中文 / English），默认英文界面

---

## 2. 页面清单与用户流程

### 2.1 新增视图 / 面板（在 REQ-003 基础上扩展）

| 序号 | 视图 / 面板 | 类型 | 说明 |
|------|------------|------|------|
| 1 | 局部更新列表抽屉 | 右侧 el-drawer | 查看某图纸所有局部更新，支持 All/Active Tab 筛选与关键字搜索；设计人员可在此发布新 Part Print |
| 2 | 发布局部更新弹窗 | el-dialog | 上传主图纸文件，AI 自动识别 Title Block 字段，用户核验后发布 |

### 2.2 图纸列表页改动点

在 UI-REQ-003-pc §3.2.6 操作按钮组基础上，追加以下按钮：

| 按钮 | 位置 | 显示条件 |
|------|------|---------|
| Part Print | Assign 按钮之后 | 所有行（`drawing:view` 权限），任意图纸状态均显示 |

新增表格列 **Part Print**（可选，建议默认展示）：

| 列名 | 字段 | 宽度 | 说明 |
|------|------|------|------|
| Part Print | `activeMarkupCount` | 90px | 显示 ACTIVE 数量；0 时显示 `—`；数字为链接样式，点击打开局部更新抽屉 |

### 2.3 核心用户流程

```
设计人员 发布局部更新：

  图纸列表（approvalStatus = APPROVED_EXTERNAL）
       │ 点击 [Part Print]
       ▼
  局部更新列表抽屉
       │ 点击右上角 [+ Part Print]
       ▼
  发布局部更新弹窗（阶段一）
  上传 Part Print 主图纸文件
       │ 文件上传成功
       ▼
  AI 自动识别 Title Block → 字段回填（✨ 标识）
  用户核验并按需修改字段
  选择 Applied to Page（选填）
       │ [Publish]
       ▼
  弹窗关闭
  抽屉列表刷新，新卡片出现
  图纸列表 Part Print 列 +1
  Toast：已通知 Site Engineers
```

```
用户 查阅局部更新历史：

  图纸列表（任意图纸状态）
       │ 点击 [Part Print]
       ▼
  局部更新列表抽屉
  All / Active Tab 筛选
  搜索框输入关键字实时过滤
  查看卡片：标题 / 日期 / 创建人 / 报审号 / 页码 / 说明 / 附件
```

---

## 3. 组件设计规范

### 3.1 图纸列表页扩展

#### 3.1.1 操作列完整按钮组

```
┌─────────────────────────────────────────────────────────────────────────┐
│  [View] [History] [Confirms] [Assign] [Part Print] │ [Upload V{n}]      │
└─────────────────────────────────────────────────────────────────────────┘
```

| 按钮 | 样式 |
|------|------|
| Part Print | `el-button size="mini" icon="el-icon-document"`，文字 "Part Print"；若 `activeMarkupCount > 0`，图标右上角显示橙色计数角标 |

> 操作列按钮超出宽度时，折叠规则：优先保留 View 和 Upload V{n}；Part Print / Confirms / Assign / History 折入 `el-dropdown` 更多菜单。

#### 3.1.2 Part Print 列样式

| 状态 | 展示 |
|------|------|
| `activeMarkupCount = 0` | 灰色 `—` |
| `activeMarkupCount > 0` | 橙色数字链接（`--color-warning` #F57F17），hover 下划线，点击打开抽屉 |

---

### 3.2 局部更新列表抽屉

#### 3.2.1 抽屉结构

```
┌──────────────────────────────────────────────────────────┐
│ ←  Part Print — ARCH-001 · 首层平面图                     │  ← 标题行
│    Based on V3  ·  2 Active                              │  ← 副标题
├──────────────────────────────────────────────────────────┤
│  [All (5)]                        [+ Part Print]         │  ← Tab 筛选行
├──────────────────────────────────────────────────────────┤
│  🔍 Search by description or drawing no...               │  ← 搜索框
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  A轴节点详图修正                                    │  │
│  │  Apr 8, 2026  ·  张三                              │  │
│  │  Based on SUB-2026-003  ·  Page 3                  │  │
│  │  A轴与3轴交叉节点详图已更新，新增钢筋排布说明...    │  │
│  │  ...Show more                                      │  │
│  │  📎 node-detail.pdf                               │  │
│  │                                        [🗑 Delete] │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  C区消防管道路由修正                                │  │
│  │  Apr 7, 2026  ·  张三                              │  │
│  │  Based on SUB-2026-003  ·  Page 5                  │  │
│  │  消防主管道路由变更，详见附图...                    │  │
│  │  📎 route-update.png                              │  │
│  │                                        [🗑 Delete] │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

#### 3.2.2 抽屉属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-drawer`，direction="rtl" |
| 宽度 | 560px |
| 标题 | "Part Print — {drawingCode} · {drawingName}"，`--font-weight-semibold` |
| 副标题 | "Based on {versionNo} · {activeCount} Active"，13px，`--color-text-secondary` |
| Tab 筛选 | `el-tabs`，标签：All ({total}) / Active；All Tab 显示局部更新总条数；Active Tab 不显示数量角标 |
| [+ Part Print] 按钮 | Tab 行右侧，`el-button size="small" type="primary" plain`；仅设计人员（本图纸上传人）且当前版本 `approvalStatus = APPROVED_EXTERNAL` 时显示；点击打开 §3.3 发布弹窗 |
| 搜索框 | Tab 行下方，全宽 `el-input` with prefix 搜索图标，placeholder "Search by description or drawing no..."；前端实时过滤，匹配 Description 和 Part Print Drawing No.（大小写不敏感）；无匹配时显示空态"No results found" |
| 内容区背景 | #F5F7FA |
| 内边距 | 16px |

#### 3.2.3 局部更新卡片

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF，圆角 4px，边框 1px #EBEEF5，内边距 12px 16px，底部间距 8px |
| 标题 | 14px，`--font-weight-semibold`，`--color-text-regular` |
| 元信息行 | "{date} · {creatorName}"，12px，`--color-text-secondary`，标题下方 4px |
| 报审号 + 页码行 | 元信息行下方；格式 "Based on {submissionNo}"，若 `appliedPageNo` 不为 null 则追加 " · Page {n}"；12px，`--color-text-secondary` |
| 说明文字 | 13px，`--color-text-regular`，最多展示 3 行，超出显示 "...Show more" 文字链接（点击展开全文，再次点击收起） |
| 附件列表 | 📎 图标 + 文件名，12px，`--color-text-secondary`；文件名为链接，点击新标签页预览/下载 |
| [🗑 Delete] | 卡片右下角，`el-button size="mini" type="text"`，红色 `#C62828` 垃圾桶图标；仅 Part Print 创建人且状态为 ACTIVE 时可见；hover 背景 #FFF5F5 |
| 空态 | 搜索无结果时，内容区居中显示灰色图标 + 文字 "No results found" |

#### 3.2.4 删除确认弹窗

| 属性 | 规格 |
|------|------|
| 组件 | `el-dialog`，宽度 420px |
| 标题 | "Delete Part Print" |
| 内容 | "Are you sure you want to delete this Part Print? This action cannot be undone." |
| 取消按钮 | "Cancel"，`el-button`，关闭 Dialog，列表不变 |
| 删除按钮 | "Delete"，`el-button type="danger"`，点击后进入 loading 状态防重复提交 |
| 删除成功 | Dialog 关闭，卡片从列表移除，抽屉副标题 Active 计数 -1，图纸列表 Part Print 列 -1，Toast 提示 "Part Print deleted." |
| 删除失败 | Dialog 保持打开，按钮恢复可点击，顶部 Toast 错误提示，用户可重试 |

---

### 3.3 发布局部更新弹窗

#### 3.3.1 弹窗结构（阶段一 — 初始态，未上传图纸）

```
┌─────────────────────────────────────────────────────────┐
│  Publish Part Print — ARCH-001 · 首层平面图  V3      [✕] │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Part Print Drawing *                                   │
│  ┌─────────────────────────────────────────────────┐   │
│  │                                                 │   │
│  │    📎  Drag & Drop or  [Browse Files]           │   │
│  │    Supports: PDF / PNG / JPG · Max 50MB         │   │
│  │                                                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│  上传图纸后将自动识别 Title Block 信息                     │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                           [Cancel]  [Publish]           │
└─────────────────────────────────────────────────────────┘
```

#### 3.3.2 弹窗结构（阶段一 → 阶段二过渡 — AI 识别中）

```
┌─────────────────────────────────────────────────────────┐
│  Publish Part Print — ARCH-001 · 首层平面图  V3      [✕] │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Part Print Drawing *                                   │
│  ┌───────────────────────────────────────────────────┐  │
│  │  📄 ARCH-001-PP-0014.pdf (2.1 MB)           [✕]  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ✨ Identifying drawing info...  ████████░░  80%        │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                           [Cancel]  [Publish]           │
└─────────────────────────────────────────────────────────┘
```

#### 3.3.3 弹窗结构（阶段二 — AI 识别完成，字段回填，可编辑）

```
┌─────────────────────────────────────────────────────────┐
│  Publish Part Print — ARCH-001 · 首层平面图  V3      [✕] │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Part Print Drawing *                                   │
│  ┌───────────────────────────────────────────────────┐  │
│  │  📄 ARCH-001-PP-0014.pdf (2.1 MB)           [✕]  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ✨ Drawing info recognized — please verify and edit   │
│     if needed                                           │
│                                                         │
│  Dwg No. *                                              │
│  [CJY-P1-DW-SH-WFR-0117-ST-L1_0                    ✨] │
│                                                         │
│  Rev No.                                                │
│  [0                                                 ✨] │
│                                                         │
│  Part Print Drawing No. *                               │
│  [0014                                              ✨] │
│                                                         │
│  Description *                                          │
│  [Updated beam and wall layout                      ✨] │
│                                                  0/2000  │
│                                                         │
│  Issued Date                                            │
│  [10-Mar-2026                                       ✨] │
│                                                         │
│  Drawn By                                               │
│  [WANG WENHAO                                       ✨] │
│                                                         │
│  Approved By AES (C&S)                                  │
│  [Dicky                                             ✨] │
│                                                         │
│  Approved Date                                          │
│  [11/3/2026                                         ✨] │
│                                                         │
│  Applied to Page                                        │
│  [  Page 2 of 8           ▼                         ]  │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                           [Cancel]  [Publish]           │
└─────────────────────────────────────────────────────────┘
```

#### 3.3.4 弹窗属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-dialog` |
| 宽度 | 560px |
| 标题行 | "Publish Part Print — {drawingCode} · {drawingName}  {currentVersionNo}"，16px，`--font-weight-semibold` |
| 关闭 | ✕ / Cancel / 遮罩；有内容时弹出二次确认 |
| 内容区 | 高度超出时内部滚动 |

#### 3.3.5 表单字段规格

| 字段 | 必填 | 组件 | AI 识别 | 规格 |
|------|:---:|------|:-------:|------|
| Part Print Drawing | ✅ | 单文件上传区（`el-upload`） | — | PDF/PNG/JPG；≤ 50MB；仅 1 个；上传后显示文件名 + 大小 + [✕] 可替换 |
| Dwg No. | ✅ | `el-input` | ✅ | ≤ 200 字符；AI 回填时右侧显示 ✨ 图标 |
| Rev No. | — | `el-input` | ✅ | ≤ 50 字符 |
| Part Print Drawing No. | ✅ | `el-input` | ✅ | ≤ 50 字符 |
| Description | ✅ | `el-input` type="textarea" rows=3 | ✅ | ≤ 2000 字符；右下角字数计数 "x / 2000" |
| Issued Date | — | `el-input` | ✅ | ≤ 50 字符；原始文本格式，不强制日期 picker |
| Drawn By | — | `el-input` | ✅ | ≤ 200 字符 |
| Approved By AES (C&S) | — | `el-input` | ✅ | ≤ 200 字符；识别文字签名部分 |
| Approved Date | — | `el-input` | ✅ | ≤ 50 字符；原始文本格式 |
| Applied to Page | — | `el-select`（单选） | — | 选项 Page 1 ～ Page N，N = 基础版本 PDF 总页数；获取失败退化为 `el-input-number`（min=1）；默认空（未选）；选填不影响提交 |

#### 3.3.6 AI 识别交互规格

| 场景 | 视觉表现 |
|------|---------|
| 上传完成，识别中 | 文件行下方显示进度条 + 文字 "✨ Identifying drawing info..."，进度条蓝色 `--color-primary` |
| 识别成功 | 进度条消失，显示绿色提示条 "✨ Drawing info recognized — please verify and edit if needed"；回填字段右侧显示 ✨ 图标（`--color-warning` 金色）；字段背景 #FFFDE7 |
| 识别失败 / 超时（>15s） | 进度条消失，显示 `el-alert type="warning"` 警告条："AI recognition failed. Please fill in the fields manually."；字段全部留空，可手动填写 |
| 用户编辑 AI 回填字段 | 该字段 ✨ 图标消失，背景恢复默认，表示用户已覆盖 AI 值 |
| 用户点击 [✕] 替换文件 | 所有 AI 回填字段清空，背景恢复，重新触发上传 + 识别流程 |

#### 3.3.7 底部操作行

| 按钮 | 类型 | 规格 |
|------|------|------|
| Cancel | `el-button` | 灰色；有内容时弹出关闭确认 Dialog |
| Publish | `el-button type="primary"` | 蓝色；必填项（Part Print Drawing / Dwg No. / Part Print Drawing No. / Description）未填时 disabled；提交中 loading 防重复点击；AI 识别进行中不阻塞提交 |

#### 3.3.8 发布成功 Toast

| 属性 | 规格 |
|------|------|
| 组件 | `el-notification`，右上角，持续 4 秒 |
| 图标 | ✅ 绿色 |
| 标题 | "Part Print Published" |
| 内容 | "Assigned Site Engineers have been notified." |

---

## 4. 多语言文案

### 4.1 图纸列表页扩展

| Key | English | 中文 |
|-----|---------|------|
| action_part_print | Part Print | 局部更新 |
| col_part_print | Part Print | 局部更新 |

### 4.2 发布局部更新弹窗

| Key | English | 中文 |
|-----|---------|------|
| publish_dialog_title | Publish Part Print — {code} · {name}  {version} | 发布局部更新 — {code} · {name}  {version} |
| field_part_print_drawing | Part Print Drawing | 局部更新图纸 |
| upload_hint_part_print | Supports: PDF / PNG / JPG · Max 50MB | 支持 PDF / PNG / JPG · 最大 50MB |
| ai_identifying | ✨ Identifying drawing info... | ✨ 正在识别图纸信息... |
| ai_recognized | ✨ Drawing info recognized — please verify and edit if needed | ✨ 图纸信息已识别，请核验并按需修改 |
| ai_failed | AI recognition failed. Please fill in the fields manually. | AI 识别失败，请手动填写字段。 |
| field_dwg_no | Dwg No. | 图纸编号 |
| field_rev_no | Rev No. | 版本号 |
| field_part_print_drawing_no | Part Print Drawing No. | 局部更新图纸编号 |
| field_description | Description | 说明 |
| field_issued_date | Issued Date | 出图日期 |
| field_drawn_by | Drawn By | 绘图人 |
| field_approved_by_aes | Approved By AES (C&S) | AES 审批人 |
| field_approved_date | Approved Date | 审批日期 |
| field_applied_to_page | Applied to Page | 对应页码 |
| validation_required | {field} is required | {field} 为必填项 |
| validation_file_format | Only PDF, PNG, JPG files are allowed | 仅支持 PDF、PNG、JPG 格式 |
| validation_file_size | File size cannot exceed 50MB | 文件不能超过 50MB |
| btn_publish | Publish | 发布 |
| toast_part_print_published_title | Part Print Published | 局部更新已发布 |
| toast_part_print_published_body | Assigned Site Engineers have been notified. | 已通知相关 Site Engineer。 |

### 4.3 局部更新列表抽屉

| Key | English | 中文 |
|-----|---------|------|
| drawer_title | Part Print — {code} · {name} | 局部更新 — {code} · {name} |
| drawer_subtitle | Based on {version} · {active} Active | 基于 {version} · {active} 生效中 |
| tab_all | All ({n}) | 全部 ({n}) |
| tab_active | Active | 生效中 |
| search_placeholder | Search by description or drawing no... | 按说明或图纸编号搜索... |
| search_empty | No results found | 无匹配结果 |
| card_based_on | Based on {submissionNo} | 基于 {submissionNo} |
| card_page | · Page {n} | · 第 {n} 页 |
| label_show_more | ...Show more | ...展开全文 |
| label_show_less | Show less | 收起 |
| btn_add_part_print | + Part Print | + 局部更新 |
| btn_delete | Delete | 删除 |
| confirm_delete_title | Delete Part Print | 删除局部更新 |
| confirm_delete_body | Are you sure you want to delete this Part Print? This action cannot be undone. | 确定要删除该局部更新吗？此操作不可撤销。 |
| toast_part_print_deleted | Part Print deleted. | 局部更新已删除。 |

---

## 5. 响应式与分辨率适配

| 项目 | 规格 |
|------|------|
| 发布局部更新弹窗 | 固定宽度 560px，内容区高度超出时内部滚动 |
| 局部更新列表抽屉 | 固定宽度 560px，不随分辨率变化 |
| 操作列折叠 | 1280px 最小宽度下，Part Print 可折入更多菜单 |

---

## 6. 验收条件

### 6.1 图纸列表页扩展

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | Part Print 列显示 | ACTIVE 数量以橙色数字链接展示，0 时显示 `—` |
| 2 | [Part Print] 按钮 | 所有图纸行可见（任意状态），点击打开对应图纸的局部更新抽屉 |
| 3 | 权限控制 | 无 `drawing:view` 权限时，[Part Print] 按钮不可见 |

### 6.2 局部更新列表抽屉

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 数据加载 | 打开时正确展示所有 Part Print；All Tab 显示总数；Active Tab 不显示数量角标 |
| 2 | [+ Part Print] 按钮 | 仅设计人员（本图纸上传人）且 `approvalStatus = APPROVED_EXTERNAL` 时显示；其他角色或其他状态时不显示 |
| 3 | 搜索过滤 | 输入关键字后实时过滤，匹配 Description 和 Part Print Drawing No.（大小写不敏感）；无结果时显示空态 |
| 4 | 报审号展示 | 每张卡片正确显示 "Based on {submissionNo}"；有页码时追加 " · Page {n}" |
| 5 | 说明展开 | 超 3 行内容显示 "...Show more"，点击展开全文；再次点击收起 |
| 6 | 附件下载 | 点击附件文件名在新标签页预览 / 下载 |
| 7 | 删除按钮可见性 | 仅 Part Print 创建人可见 [🗑 Delete]；非创建人不可见 |
| 8 | 删除成功 | 点击删除 → 二次确认 Dialog → 确认后卡片消失，抽屉 Active 计数 -1，图纸列表 Part Print 列 -1，Toast 提示 |
| 9 | 删除失败 | 接口报错时 Dialog 保持打开，按钮恢复，Toast 错误提示 |

### 6.3 发布局部更新弹窗

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 弹窗标题 | 标题正确显示图纸编号、名称和当前版本号 |
| 2 | 文件上传后自动触发 AI 识别 | 上传完成后立即显示进度条与识别提示 |
| 3 | AI 识别成功 | 字段自动回填，右侧显示 ✨ 图标，字段背景高亮；字段可手动编辑，编辑后 ✨ 消失 |
| 4 | AI 识别失败 | 显示 warning 警告条，字段留空，不阻塞提交 |
| 5 | 替换文件重新识别 | 点击 [✕] 移除文件后，AI 回填字段清空，进度条重新出现 |
| 6 | 必填校验 | Part Print Drawing / Dwg No. / Part Print Drawing No. / Description 为空时 [Publish] disabled；提交后显示 inline 错误提示 |
| 7 | Applied to Page 选项范围 | el-select 选项为 Page 1 ～ Page N（N = 基础版本 PDF 总页数）；获取失败时退化为 el-input-number |
| 8 | 文件格式校验 | 非 PDF/PNG/JPG 被拦截，显示格式错误提示 |
| 9 | 文件大小校验 | 超 50MB 被拦截，显示大小超限提示 |
| 10 | 发布成功 | 弹窗关闭，抽屉列表刷新，图纸列表 Part Print 列 +1，Toast 正确显示 |
| 11 | 关闭确认 | 有内容时点击取消或遮罩弹出二次确认 |

---

## 7. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-004-pc.md](../../../requirements/pc/REQ-004-pc.md) | PC 端产品需求（单一事实源） |
| [REQ-004-shared.md](../../../requirements/shared/REQ-004-shared.md) | 跨端共享业务规则与 API |
| [UI-REQ-003-pc.md](./UI-REQ-003-pc.md) | 图纸列表页基础 UI 规范（本功能依赖） |
