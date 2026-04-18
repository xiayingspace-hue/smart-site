# UI 设计说明文档 — PC 管理端图纸两级审批流程

> **来源需求**: [REQ-007-pc](../../../requirements/pc/REQ-007-pc.md) + [REQ-007-shared](../../../requirements/shared/REQ-007-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **设计令牌参考**: [UI-REQ-001-pc § 3.2](./UI-REQ-001-pc.md)
> **依赖 UI 文档**: [UI-REQ-003-pc](./UI-REQ-003-pc.md)（图纸管理基础 UI）
> **生成日期**: 2026-04-17

---

## 1. 设计目标

- 在 REQ-003 已有图纸管理界面基础上，最小化变更地引入两级审批流程
- DC 外部审批 Todo 卡片信息充足，DC 无需在系统中来回查找，一卡完成所有操作
- 版本历史 4 阶段生命周期视图层次清晰，步骤条直观反映进度
- DC 配置页面简洁实用，操作低频但关键
- 5 种审批状态的颜色区分度高，橙（等待）/ 绿（通过）/ 红（驳回）三色体系一致
- 延续 REQ-001-pc / REQ-003-pc 已有视觉体系，沿用 Element UI 组件规范

---

## 2. 页面清单与用户流程

### 2.1 页面 / 组件清单

| 序号 | 视图 / 面板 | 类型 | 变更类型 | 说明 |
|------|------------|------|---------|------|
| 1 | 上传图纸弹窗 | Modal Dialog | **变更** | 审批人标签改为 "Internal Approver" |
| 2 | 内部审批 Todo 卡片 | 通知弹出层 | **变更** | 标题/文案/确认弹窗调整 |
| 3 | DC 外部审批 Todo 卡片 | 通知弹出层 | **新增** | 含下载原始文件 + 标记结果 |
| 4 | 外部审批标记 Dialog | Modal Dialog | **新增** | Approved / Rejected 分支表单 |
| 5 | 版本历史抽屉 | 右侧 Drawer | **重构** | 可展开 4 阶段卡片 + 步骤条 |
| 6 | 图纸列表状态标签 | 表格列 | **变更** | 3 态扩展为 5 态 |
| 7 | DC 配置页面 | 独立路由页面 | **新增** | DC 人员管理 |
| 8 | 添加 DC 弹窗 | Modal Dialog | **新增** | 搜索 + 添加 |

### 2.2 DC 外部审批核心流程

```
DC 收到通知（铃铛角标 +N）
       │ 点击铃铛
       ▼
┌─────────────────────────┐
│  Todo 面板               │
│  🌐 External Approval   │
│  Required × N            │
└──────────┬──────────────┘
           │
    ┌──────┴──────┐
    ↓              ↓
[📄 Download     [✅ Mark
 Original]        Result]
    │              │
    ▼              ▼
 浏览器下载    Mark External
 原始图纸文件  Approval Dialog
               │
        ┌──────┴──────┐
        ↓              ↓
   ○ Approved     ○ Rejected
        │              │
        ▼              ▼
   上传签字版      填写驳回原因
   上传凭证
   选日期
        │              │
        ▼              ▼
   [Confirm]       [Confirm]
   Loading 3-5s         │
        │              ▼
        ▼         Toast "Rejection
   Toast "Active   recorded"
   + QR generated" Todo 消失
   Todo 消失
```

---

## 3. 设计规范

### 3.1 上传弹窗变更

在 [UI-REQ-003-pc §3.3](./UI-REQ-003-pc.md) 基础上做以下变更：

| 属性 | 原值 | 新值 |
|------|------|------|
| Approver 标签 | `Approver *` | `Internal Approver *` |
| 标签字号 | 不变 | 14px，`--font-weight-medium`，`--color-text-primary` |
| 上传成功 Toast | "Drawing uploaded successfully. Pending approval." | "Drawing uploaded successfully. Pending internal approval." |

其他所有弹窗样式、字段、交互保持不变。

---

### 3.2 内部审批 Todo 卡片

在 [UI-REQ-003-pc §3.5](./UI-REQ-003-pc.md) 审批 Todo 卡片基础上变更：

| 属性 | 规格 |
|------|------|
| 图标 | 🔍（替代原审批图标） |
| 标题 | "Internal Approval Required"，16px，`--font-weight-semibold` |
| 卡片布局 | 与原审批 Todo 卡片一致 |
| Approve 确认 Dialog 标题 | "Confirm Internal Approval?" |
| Approve 确认 Dialog 正文 | "Once approved, this version will proceed to external approval by Document Controller." |
| 通过后 Toast | "Internal approval completed. DC has been notified for external approval." |

---

### 3.3 DC 外部审批 Todo 卡片（新增）

```
┌──────────────────────────────────────────────────────────────────┐
│ 🌐 External Approval Required                                   │  ← 标题行
│                                                                  │
│ ARCH-001  首层平面图  V3                                         │  ← 图纸信息
│ Uploaded by: 张三（Designer）  |  2026-04-01 10:00               │
│ Internal approved by: 王总工  |  2026-04-02 14:30               │  ← 新增行
│ Version Note: 修正轴网尺寸                                       │
│                                                                  │
│ [📄 Download Original]   [✅ Mark Result]                        │  ← 操作行
└──────────────────────────────────────────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| 卡片背景 | 白色 #FFFFFF，圆角 8px，阴影 0 2px 8px rgba(0,0,0,0.08) |
| 卡片内边距 | 16px 20px |
| 卡片间距 | 底部 12px |
| 标题图标 | 🌐，20px |
| 标题文字 | "External Approval Required"，16px，`--font-weight-semibold`，`--color-text-primary` |
| 图纸编号 + 名称 + 版本 | 14px，`--font-weight-medium`，`--color-text-primary` |
| "Uploaded by" 行 | 13px，`--color-text-secondary` (#909399) |
| "Internal approved by" 行 | 13px，`--color-text-secondary` (#909399)，新增行 |
| "Version Note" 行 | 13px，`--color-text-secondary`，斜体 |
| `[📄 Download Original]` | el-button size="small"，icon="el-icon-download"，文字 "Download Original" |
| `[✅ Mark Result]` | el-button size="small" type="primary"，文字 "Mark Result" |
| 按钮间距 | 12px |

---

### 3.4 外部审批标记 Dialog

#### 3.4.1 弹窗结构

| 属性 | 规格 |
|------|------|
| 弹窗宽度 | 560px |
| 标题 | "Mark External Approval Result"，16px，`--font-weight-semibold` |
| 副标题 | "{drawingCode}  {drawingName}  V{n}"，14px，`--color-text-secondary`，位于标题栏下方 |
| 分隔线 | 1px #EBEEF5，标题区与表单区之间 |
| 表单内边距 | 24px |
| 关闭按钮 | 右上角 ✕ |

#### 3.4.2 Result 选择

| 属性 | 规格 |
|------|------|
| 标签 | "Result *"，14px，`--font-weight-medium` |
| 组件 | el-radio-group，垂直排列 |
| 选项 | "Approved" / "Rejected"，14px |
| 默认 | 不选中 |
| 间距 | 标签与 Radio 间距 8px，Radio 项间距 12px |

#### 3.4.3 Approved 分支字段

以下字段在选择 Approved 后以 `v-show` 展开，带 300ms `transition`：

| 字段 | 组件 | 规格 |
|------|------|------|
| Signed Drawing File * | el-upload 拖拽上传区 | 高度 120px，虚线边框 #DCDFE6，圆角 6px；hover 边框变 #409EFF；已选文件后显示文件名 + 大小 + ✕ 移除 |
| Approval Evidence * | el-upload 拖拽上传区 | 同上，高度 100px |
| External Approval Date * | el-date-picker | 宽度 100%，format "yyyy-MM-dd"，`picker-options` 禁用未来日期 |
| Remarks | el-input type="textarea" | 3 行，maxlength 500，show-word-limit |

字段间距：每个字段块之间 16px。标签与输入组件间距 8px。

#### 3.4.4 Rejected 分支字段

| 字段 | 组件 | 规格 |
|------|------|------|
| Rejection Reason * | el-input type="textarea" | 4 行，maxlength 500，show-word-limit，placeholder "Please describe the rejection reason..." |

#### 3.4.5 底部按钮

| 属性 | 规格 |
|------|------|
| 布局 | 右对齐，间距 12px |
| Cancel | el-button，"Cancel" |
| Confirm | el-button type="primary"，"Confirm"；未选择 Result 时 disabled |
| Loading 态 | Confirm 按钮文案变为 "Processing..."，el-button :loading="true"；Dialog 内所有表单项 disabled |

---

### 3.5 版本历史抽屉（重构）

#### 3.5.1 抽屉容器

| 属性 | 规格 |
|------|------|
| 宽度 | 720px（原 480px → 扩展） |
| 标题 | "Version History — {drawingCode} {drawingName}"，16px，`--font-weight-semibold` |
| 背景 | #F5F7FA |
| 内边距 | 24px |

#### 3.5.2 主列表（版本行）

| 属性 | 规格 |
|------|------|
| 行高 | 48px |
| 行背景 | 白色 #FFFFFF，圆角 6px，底部间距 8px |
| 展开箭头 | ▶ / ▼，14px，`--color-text-secondary`，过渡 200ms 旋转 |
| Version 列 | 70px，14px，`--font-weight-semibold` |
| Status 列 | 160px，自定义标签组件（见 §3.5.3） |
| Designer 列 | 100px，13px |
| Confirmed 列 | 80px，13px，仅 APPROVED 显示 `x/y` |
| QR 列 | 60px，13px，APPROVED 时蓝色链接 `[View]` |

#### 3.5.3 状态标签

| 状态 | 文字 | 前缀图标 | 背景色 | 文字色 |
|------|------|---------|--------|--------|
| PENDING_INTERNAL | Pending Internal | ⏳ | #FDF6EC | #E6A23C |
| INTERNAL_APPROVED | Pending External | ⏳ | #FDF6EC | #E6A23C |
| INTERNAL_REJECTED | Int. Rejected | ❌ | #FEF0F0 | #F56C6C |
| APPROVED | Approved | ✅ | #F0F9EB | #67C23A |
| EXTERNAL_REJECTED | Ext. Rejected | ❌ | #FEF0F0 | #F56C6C |

标签规格：圆角 4px，内边距 2px 10px，12px，`--font-weight-medium`。

#### 3.5.4 展开行 — 4 阶段卡片

展开后背景 #F5F7FA，内边距 16px 20px。4 阶段卡片 2×2 网格排列：

| 属性 | 规格 |
|------|------|
| 网格 | 2 列 × 2 行，列间距 16px，行间距 12px |
| 卡片宽度 | 各占 50% - 间距 |
| 卡片背景 | 白色 #FFFFFF |
| 卡片圆角 | 8px |
| 卡片内边距 | 16px |
| 卡片阴影 | 0 1px 4px rgba(0,0,0,0.06) |
| 卡片标题 | ① Uploaded / ② Internal Approval / ③ External Approval / ④ Signed Version |
| 标题字号 | 13px，`--font-weight-semibold`，`--color-text-primary` |
| 内容文字 | 13px，行高 22px，`--color-text-secondary` |
| 状态图标 | ✅ 绿 / ❌ 红 / ⏳ 橙 / — 灰，16px |

**卡片内操作按钮**：

| 按钮 | 样式 |
|------|------|
| [Preview] | el-button size="mini" type="text"，蓝色链接样式 |
| [Download] | el-button size="mini" type="text"，蓝色链接样式 |
| [✅ Mark Result] | el-button size="mini" type="primary"，仅 DC 权限可见 |

**"Not reached" 态卡片**：

| 属性 | 规格 |
|------|------|
| 背景 | #FAFAFA |
| 文字 | "— Not reached"，13px，`--color-text-placeholder` (#C0C4CC) |
| 副文字 | "(Internal approval failed)" 等，12px，斜体 |

**"Waiting" 态卡片**：

| 属性 | 规格 |
|------|------|
| 背景 | #FFFFFF |
| 文字 | "⏳ Pending" / "Waiting for ..."，13px，`--color-text-secondary` |

#### 3.5.5 步骤条

| 属性 | 规格 |
|------|------|
| 位置 | 展开行底部，上方 12px 分隔（虚线 1px #DCDFE6） |
| 布局 | 水平居中，4 步等分 |
| 步骤文字 | 12px，`--font-weight-medium` |
| 步骤图标 | 📄 / 🔍 / 🌐 / ✍️，16px |
| 状态标记 | ✓ / ⏳ / ✕ / — / ...，跟随步骤文字后方 |
| 连接箭头 | →，`--color-text-placeholder`，12px |
| 完成色 | #67C23A |
| 进行中色 | #E6A23C |
| 失败色 | #F56C6C |
| 未到达色 | #C0C4CC |

---

### 3.6 图纸列表状态标签（扩展）

替代 [UI-REQ-003-pc §3.2.5](./UI-REQ-003-pc.md)：

| 状态 | 文字 | 背景色 | 文字色 | 边框色 |
|------|------|--------|--------|--------|
| Active | Active | #F0FAF0 | #2E7D32 | #A5D6A7 |
| Pending Internal | Pending Internal | #FDF6EC | #E6A23C | #FAECD8 |
| Pending External | Pending External | #FDF6EC | #E6A23C | #FAECD8 |
| Int. Rejected | Int. Rejected | #FEF0F0 | #F56C6C | #FDE2E2 |
| Ext. Rejected | Ext. Rejected | #FEF0F0 | #F56C6C | #FDE2E2 |

标签规格：圆角 4px，内边距 2px 8px，12px，`--font-weight-medium`。列宽从 110px → **140px**（容纳 "Pending Internal"）。

**Status 筛选下拉**（扩展 [UI-REQ-003-pc §3.2.2](./UI-REQ-003-pc.md)）：

宽度从 140px → **170px**，选项：All / Active / Pending Internal / Pending External / Int. Rejected / Ext. Rejected。

---

### 3.7 DC 配置页面（新增）

#### 3.7.1 入口

| 属性 | 规格 |
|------|------|
| 侧边导航菜单项 | Settings → "DC Configuration"，图标：⚙️ |
| 路由 | `/project/:projectId/settings/dc-config` |
| 面包屑 | Home / {项目名} / Settings / DC Configuration |
| 权限 | `v-permission="'drawing:dc-config'"` |

#### 3.7.2 页面布局

```
┌──────────────────────────────────────────────────────────────────┐
│ ⚙️ DC Configuration                                  [+ Add DC] │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Document Controllers for current project:                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Name           │ Configured By  │ Configured At │ Action  │  │
│  │─────────────────│────────────────│───────────────│─────────│  │
│  │  陈小明          │ Admin          │ 2026-04-01    │ [Remove]│  │
│  │  刘文静          │ Admin          │ 2026-04-01    │ [Remove]│  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ⓘ At least one DC is required for the drawing approval process. │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| 页面标题 | "DC Configuration"，20px，`--font-weight-semibold` |
| `[+ Add DC]` 按钮 | el-button type="primary" size="small" icon="el-icon-plus"，右上角 |
| 表格 | el-table，stripe，行高 48px |
| Name 列 | 200px，14px |
| Configured By 列 | 150px |
| Configured At 列 | 150px，格式 "Apr 1, 2026" |
| Action 列 | 100px |
| `[Remove]` | el-button size="mini" type="danger" plain |
| 提示文字 | ⓘ 信息提示，13px，`--color-text-secondary`，底部 16px margin-top |
| 空状态 | el-empty，描述 "No DC configured" |
| 移除后空警告 | el-alert type="warning"，"⚠️ No DC configured. Internal approvals cannot proceed to external approval." |

#### 3.7.3 添加 DC 弹窗

| 属性 | 规格 |
|------|------|
| 弹窗宽度 | 480px |
| 标题 | "Add Document Controller" |
| 搜索框 | el-input，icon="el-icon-search"，placeholder "Search by name..."，debounce 300ms |
| 用户列表 | 垂直列表，每行：用户名 + 右侧 `[Add]` 按钮 |
| 行高 | 40px |
| `[Add]` 按钮 | el-button size="mini" type="primary" plain |
| 添加成功 | 该行消失（已配置）+ 外层表格刷新 |
| `[Close]` 按钮 | 底部右对齐，el-button |

---

## 4. 设计约束

| 项 | 约束 |
|----|------|
| 颜色体系 | 橙（等待）#E6A23C / 绿（通过）#67C23A / 红（驳回）#F56C6C，与 Element UI 默认色一致 |
| 组件库 | Element UI 2.x，不引入外部组件 |
| 响应式 | 最小宽度 1280px，无移动端适配 |
| 国际化 | 所有文案支持 i18n，默认英文 |
| 动画 | 展开/收起 300ms ease-in-out；弹窗 fade 200ms |
| 无障碍 | 所有操作按钮含 aria-label；状态标签不仅依赖颜色（含图标前缀） |

---

## 5. 验收条件

- [ ] 上传弹窗审批人标签改为 "Internal Approver"
- [ ] 内部审批 Todo 卡片使用 🔍 图标 + "Internal Approval Required" 标题
- [ ] DC 外部审批 Todo 卡片使用 🌐 图标，包含 "Internal approved by" 信息行
- [ ] 外部审批标记 Dialog 宽度 560px，Approved / Rejected 分支条件显隐正确
- [ ] 版本历史抽屉宽度 720px，4 阶段卡片 2×2 网格清晰
- [ ] 步骤条颜色与状态匹配（绿/橙/红/灰）
- [ ] 5 种状态标签颜色区分明显
- [ ] DC 配置页面简洁，添加/移除流程顺畅
- [ ] 所有文案使用 i18n key
