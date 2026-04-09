# UI 说明文档 — PC 管理端工程图纸管理与审批

> **来源需求**: [REQ-003-pc](../../../requirements/pc/REQ-003-pc.md) + [REQ-003-shared](../../../requirements/shared/REQ-003-shared.md)
> **产品**: SMART SI#### 3.2.2 操作栏与搜索面板

**交互方式**：顶部操作栏左侧放置 **[🔍 Search]** 按钮，右侧放置 **[+ Upload Drawing]** 按钮，两者位于同一行。点击 [Search] 按钮后，在按钮正下方展开搜索面板；再次点击 [Search] 或点击面板内 [Cancel] 按钮，收起面板。

| 属性 | 规格 |
|------|------|
| Search 触发按钮 | el-button，icon="el-icon-search"，文字 "Search"，默认样式；激活（展开）时使用 type="primary" plain 高亮 |
| 搜索面板背景 | 白色 #FFFFFF，圆角 4px，阴影 0 2px 8px rgba(0,0,0,0.08) |
| 搜索面板内边距 | 16px 24px |
| Code 输入框 | el-input，宽度 220px，placeholder "Drawing code..." |
| Name 输入框 | el-input，宽度 220px，placeholder "Drawing name..." |
| Category 下拉 | el-select，宽度 160px，选项：All / Architectural / Structural / Mechanical / Electrical / Plumbing / Civil / Other |
| Status 下拉 | el-select，宽度 140px，选项：All / Active / Pending / Rejected |
| 字段布局 | 两行两列，第一行 Code + Name，第二行 Category + Status，操作按钮行右对齐 |
| 字段间距 | 列间距 16px，行间距 12px |
| Cancel 按钮 | el-button，"Cancel"；点击清空所有搜索条件并收起面板，列表恢复无筛选状态 |
| Search 按钮（面板内） | el-button type="primary"，"Search"；触发查询，面板保持展开 |**平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **设计令牌参考**: [UI-REQ-001-pc § 3.2](./UI-REQ-001-pc.md)
> **生成日期**: 2026-04-07

---

## 1. 设计目标

- 为 Drawing 团队提供结构化的图纸上传与版本管理工作台，表单清晰、上传流程顺畅
- 审批人员可在通知 / Todo 面板快速完成图纸审批，操作层级不超过两步
- 管理员可为每张图纸**指定需要查阅的 Site Engineer**，确保推送和可见范围精准，避免无关人员被打扰
- 管理员可按图纸、按版本查阅全量历史记录与 Site Engineer 确认情况（仅统计已分配人员）
- 版本状态（当前有效 / 待审批 / 已作废 / 已驳回）视觉区分度高
- 信息密度适合大屏展示，利用侧边抽屉减少页面跳转
- 视觉风格延续 REQ-001-pc 已有管理后台体系，沿用 Element UI 组件规范
- 支持多语言（中文 / English），默认英文界面

---

## 2. 页面清单与用户流程

### 2.1 页面清单

| 序号 | 视图 / 面板 | 类型 | 说明 |
|------|------------|------|------|
| 1 | 图纸列表页 | 独立路由页面 | 主视图，含筛选区 + 表格 + 操作入口 |
| 2 | 上传图纸弹窗（新建） | Modal Drawer | 新建图纸及第一版文件 |
| 3 | 上传新版本弹窗 | Modal Drawer | 已有图纸追加新版本 |
| 4 | 分配 Site Engineer 弹窗 | el-dialog | 为图纸指定需查阅的 Site Engineer |
| 5 | 审批 Todo 面板 | 顶部导航通知弹出层 | 待处理审批卡片，通过 / 驳回操作 |
| 6 | 版本历史抽屉 | 右侧 Drawer | 某图纸所有版本历史列表 |
| 7 | 查阅确认记录面板 | 右侧 Drawer | 某图纸最新版的确认 / 未确认人员列表（仅已分配人员） |

### 2.2 核心用户流程

```
Drawing Team 上传图纸：

  ┌─────────────────────┐
  │  图纸列表页          │
  │  [+ Upload Drawing] │ ← 右上角主按钮
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  上传图纸弹窗        │
  │  填写信息 + 上传文件 │
  │  选择审批人          │
  │  [Cancel] [Submit]  │
  └──────────┬──────────┘
             │ Submit
             ▼
  ┌─────────────────────┐
  │  表格新增一行        │
  │  状态：Pending       │
  │  审批人收到 Todo     │
  │  提示引导分配人员 → │ ← Toast 含 [Assign Now] 快捷入口
  └─────────────────────┘
```

```
管理员 分配 Site Engineer：

  图纸表格行  →  [Assign] 操作
                     │
                     ▼
          ┌──────────────────────────┐
          │  Assign Site Engineers   │
          │  ARCH-001 · 首层平面图    │
          ├──────────────────────────┤
          │  已分配 (2)   未分配 (8)  │ ← 左右双列
          │  Wang Wei  [Remove]      │
          │  Chen Ming [Remove]      │
          │  ─────────────────────   │
          │  Zhang San [Add]         │
          │  Liu Yang  [Add]         │
          │  ...                     │
          ├──────────────────────────┤
          │         [Cancel] [Save]  │
          └──────────────────────────┘
                     │ Save（全量覆盖提交）
                     ▼
          分配关系更新，对应 Site Engineer
          APP 端图纸列表立即生效
```

```
审批人 处理审批：

  ┌─────────────────────────┐
  │  顶部导航栏 🔔 角标 +N  │
  └──────────┬──────────────┘
             │ 点击铃铛
             ▼
  ┌─────────────────────────┐
  │  审批 Todo 弹出面板      │
  │  图纸审批卡片 × N        │
  └──────────┬──────────────┘
             │
      ┌──────┴──────┐
      ↓             ↓
  [Approve]     [Reject]
      │             │
      ▼             ▼
  确认 Dialog    驳回意见 Dialog
      │             │（必填）
      ▼             ▼
  图纸 Active   图纸 Rejected
  旧版作废      上传人收站内消息
  Site Eng 推送
```

```
管理员 查看版本历史 / 确认记录：

  图纸表格行  →  [History] 操作  →  右侧版本历史抽屉
                                     │
                                     ▼
                                 版本列表（标 Current/Deprecated/Pending/Rejected）

  图纸表格行  →  [Confirms] 操作 →  右侧确认记录抽屉
                                     │
                                     ▼
                                 已确认 / 未确认 人员列表
```

---

## 3. 页面设计规范

### 3.1 入口与导航

| 属性 | 规格 |
|------|------|
| 侧边导航菜单项 | "Drawing Management"，图标：📋，位于项目管理相关模块组内 |
| 菜单展开 | 无子菜单，单级入口，点击直接跳转图纸列表页 |
| 路由 | `/project/:projectId/drawings` |
| 面包屑 | Home / {项目名} / Drawing Management |

---

### 3.2 图纸列表页

#### 3.2.1 页面整体布局

```
┌──────────────────────────────────────────────────────────────────────┐
│  顶部导航栏（Header，共用）                               🔔 [用户▼] │
├──────────────────────────────────────────────────────────────────────┤
│ 侧边 │  Drawing Management                                           │
│ 导航 ├────────────────────────────────────────────────────────────── │
│      │  [🔍 Search]                                [+ Upload Drawing]│
│      │  ↓ 点击 Search 展开，再次点击或点 Cancel 收起                   │
│      │  ┌ 搜索面板（折叠/展开）─────────────────────────────────────┐  │
│      │  │  Code:     [_______________]  Name:    [_______________] │  │
│      │  │  Category: [All ▼]            Status:  [All ▼]          │  │
│      │  │                                        [Cancel] [Search] │  │
│      │  └───────────────────────────────────────────────────────-─┘  │
│      │  ┌ 图纸表格 ─────────────────────────────────────────┐        │
│      │  │  # │ Code  │ Name   │ Category │ Ver │ Status │    │        │
│      │  │  Confirms │ Updated │         Actions          │   │        │
│      │  ├───────────────────────────────────────────────────┤        │
│      │  │  1 │ARCH-001│首层平面│Architectu│ V3  │ Active │   │        │
│      │  │    │       │        │          │     │        │   │        │
│      │  │    │ 8/10  │ Apr 1  │[View][History][Confirms][Assign]│  │
│      │  │    │       │        │          │[Upload V4]           │  │
│      │  ├───────────────────────────────────────────────────┤        │
│      │  │  ... ...                                          │        │
│      │  └──────────────────────────────────────────────────-┘        │
│      │  ┌ 分页 ────────────────────────────────────────────┐         │
│      │  │  Total 26 items   < 1  2  3 >   10/page ▼        │         │
│      │  └──────────────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────────────────┘
```

#### 3.2.2 筛选区

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF，圆角 4px，阴影 0 1px 4px rgba(0,0,0,0.06) |
| 内边距 | 16px 24px |
| Keyword 输入框 | el-input，宽度 220px，placeholder "Search by code or name..." |
| Category 下拉 | el-select，宽度 160px，选项：All / Architectural / Structural / Mechanical / Electrical / Plumbing / Civil / Other |
| Status 下拉 | el-select，宽度 140px，选项：All / Active / Pending / Rejected |
| Search 按钮 | el-button type="primary"，"Search" |
| Reset 按钮 | el-button，"Reset"，点击清空所有筛选条件并刷新 |
| 字段间距 | 16px |

#### 3.2.3 Upload Drawing 按钮

| 属性 | 规格 |
|------|------|
| 类型 | el-button type="primary" |
| 图标 | el-icon-plus |
| 文字 | "+ Upload Drawing" |
| 位置 | 操作栏右侧，与 [Search] 按钮同行，右对齐 |

#### 3.2.4 图纸表格

| 列名 | 字段 | 宽度 | 说明 |
|------|------|------|------|
| # | 行序号 | 60px | 自动序号 |
| Code | drawingCode | 120px | 主色 #4A90D9，可点击跳转查看 |
| Name | name | 180px | 超出省略，Tooltip 显示完整 |
| Category | category | 130px | 分类标签 |
| Ver | currentVersionNo | 70px | 当前版本号，如 V3 |
| Status | status | 110px | 状态标签（见 §3.2.5） |
| Confirms | confirmCount | 100px | "{confirmed}/{total}"，如 "8/10" |
| Updated | updatedAt | 130px | 日期格式 "Apr 1, 2026" |
| Actions | — | 260px | 操作按钮组（见 §3.2.6） |

表格属性：
- `el-table` stripe，行高 56px
- 边框：外框 1px #EBEEF5，行分隔线 1px #EBEEF5
- 表头背景 #F5F7FA，14px，`--font-weight-semibold`，`--color-text-secondary`
- 表格行 hover：`--table-row-hover`（#F5F7FA）

#### 3.2.5 状态标签

| 状态 | 文字 | 背景色 | 文字色 | 边框色 |
|------|------|--------|--------|--------|
| Active | Active | #F0FAF0 | #2E7D32 | #A5D6A7 |
| Pending | Pending | #FFFDE7 | #F57F17 | #FFE082 |
| Rejected | Rejected | #FFF5F5 | #C62828 | #FFAB91 |

标签规格：
- 圆角 4px，内边距 2px 8px，12px，`--font-weight-medium`
- 使用 `el-tag` 或自定义 span，避免使用 el-tag 内置色彩

#### 3.2.6 操作按钮组

| 按钮 | 显示条件 | 样式 |
|------|---------|------|
| View | 所有行 | el-button size="mini" icon="el-icon-view"，文字 "View" |
| History | 所有行 | el-button size="mini" icon="el-icon-time"，文字 "History" |
| Confirms | 所有行 | el-button size="mini" icon="el-icon-user"，文字 "Confirms"，旁边显示数量 |
| Assign | 所有行 | el-button size="mini" icon="el-icon-s-custom"，文字 "Assign" |
| Upload V{N+1} | status=Active | el-button size="mini" type="primary" icon="el-icon-upload2"，文字 "Upload V{nextVer}" |

> 操作列宽度不足时，以 el-dropdown 更多按钮折叠 History / Confirms / Assign 操作。

---

### 3.3 上传图纸弹窗（新建）

#### 3.3.1 弹窗结构

```
┌────────────────────────────────────────────────────┐
│  Upload Drawing                              ✕     │  ← 标题栏
├────────────────────────────────────────────────────┤
│                                                    │
│  Drawing Code *                                    │
│  [___________________________________]             │
│                                                    │
│  Drawing Name *                                    │
│  [___________________________________]             │
│                                                    │
│  Category *                                        │
│  [Architectural              ▼]                    │
│                                                    │
│  Description                                       │
│  [___________________________________]             │
│  [                                   ]             │
│                                                    │
│  Drawing File *                                    │
│  ┌───────────────────────────────────────────┐     │
│  │                                           │     │
│  │  📁  Drag & Drop or  [Browse Files]       │     │
│  │  Supports: PDF / PNG / JPG（Max 50MB）    │     │
│  │                                           │     │
│  └───────────────────────────────────────────┘     │
│                                                    │
│  Version Note                                      │
│  [___________________________________]             │
│                                                    │
│  Approver *                                        │
│  [Select approver...         ▼]                    │
│                                                    │
├────────────────────────────────────────────────────┤
│                        [Cancel]  [Submit]          │  ← 底部操作行
└────────────────────────────────────────────────────┘
```

#### 3.3.2 弹窗属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-dialog` |
| 宽度 | 560px |
| 顶部标题 | "Upload Drawing"，16px，`--font-weight-semibold` |
| 背景遮罩 | `modal-append-to-body`，`append-to-body` |
| 关闭 | 点击 ✕ 或遮罩关闭，关闭前如有内容弹出二次确认 |

#### 3.3.3 表单字段规格

| 字段 | 组件 | 校验规则 |
|------|------|---------|
| Drawing Code | el-input | 必填，唯一性（blur 触发接口校验），仅允许字母数字-_ |
| Drawing Name | el-input | 必填，最长 100 字符 |
| Category | el-select | 必填，选项见 §3.2.2 |
| Description | el-input type="textarea"，rows=3 | 可选，最长 500 字符 |
| Drawing File | 自定义上传区 | 必填，仅接受 PDF/PNG/JPG，文件≤50MB |
| Version Note | el-input | 可选，最长 200 字符，placeholder "Describe changes in this version..." |
| Approver | el-select filterable | 必填，从人员接口拉取可选审批人列表 |

#### 3.3.4 文件上传区

| 属性 | 规格 |
|------|------|
| 背景 | #F5F7FA，圆角 6px，边框 1px 虚线 #D9D9D9 |
| 高度 | 120px |
| 图标 | 📁 24px，`--color-text-secondary` |
| 主文字 | "Drag & Drop or"，14px，`--color-text-secondary` |
| 按钮 | "Browse Files"，文字链接样式，`--color-primary` |
| 次文字 | "Supports: PDF / PNG / JPG  ·  Max 50MB"，12px，`--color-text-secondary` |
| 拖拽进入 | 边框变为实线 `--color-primary`，背景 `--color-primary-light-9`（#ECF5FF） |
| 已选文件 | 显示文件名 + 大小 + 删除图标 × |
| 上传进度 | `el-progress` 条，显示百分比 |
| 错误提示 | 格式不符 / 超大时红色文字提示于上传区下方 |

#### 3.3.5 底部操作行

| 按钮 | 类型 | 规格 |
|------|------|------|
| Cancel | el-button | 灰色，点击关闭弹窗（有内容时二次确认） |
| Submit | el-button type="primary" | 蓝色，提交表单；提交中 loading 状态，禁止重复点击 |

---

### 3.4 上传新版本弹窗

> 复用 §3.3 弹窗框架，以下描述差异点。

#### 3.4.1 差异说明

| 属性 | 差异 |
|------|------|
| 弹窗标题 | "Upload New Version — {drawingCode}"（含图纸编号） |
| Drawing Code | **只读展示**，不可修改 |
| Drawing Name | **只读展示**，不可修改 |
| Category | **只读展示**，不可修改 |
| Description | 可修改（展示当前描述，允许更新） |
| 版本提示 | 表单顶部添加蓝色信息块："Current version: {currentVersionNo}. After approval, this will become {nextVersionNo}." |
| Drawing File | 与新建相同，必填 |
| Version Note | 必填（新版本必须描述变更内容），placeholder "Describe what changed in this version..." |
| Approver | 与新建相同，必填 |

---

### 3.5 分配 Site Engineer 弹窗

#### 3.5.1 弹窗结构

```
┌─────────────────────────────────────────────────────────┐
│  Assign Site Engineers — ARCH-001 · 首层平面图      ✕   │  ← 标题栏
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────┐  ┌────────────────────────┐  │
│  │  Assigned (2)        │  │  Available (8)          │  │
│  ├──────────────────────┤  ├────────────────────────┤  │
│  │  👤 Wang Wei         │  │  👤 Zhang San           │  │
│  │     Site Engineer    │  │     Site Engineer [Add] │  │
│  │               [✕]   │  │                         │  │
│  │  👤 Chen Ming        │  │  👤 Liu Yang             │  │
│  │     Site Engineer    │  │     Site Engineer [Add] │  │
│  │               [✕]   │  │                         │  │
│  │                      │  │  👤 Sun Qi               │  │
│  │                      │  │     Site Engineer [Add] │  │
│  │                      │  │  ...                    │  │
│  └──────────────────────┘  └────────────────────────┘  │
│                                                         │
│  ⚠ Removing an assignee will immediately hide this     │
│    drawing from their APP list.                         │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                              [Cancel]  [Save]           │
└─────────────────────────────────────────────────────────┘
```

#### 3.5.2 弹窗属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-dialog` |
| 宽度 | 680px |
| 顶部标题 | "Assign Site Engineers — {drawingCode} · {drawingName}"，16px，`--font-weight-semibold` |
| 触发方式 | 点击图纸表格行 [Assign] 按钮 |
| 关闭 | ✕ / Cancel / 遮罩点击；有未保存变更时弹出二次确认 |

#### 3.5.3 双栏布局

| 栏 | 标题 | 内容 | 宽度 |
|----|------|------|------|
| 左栏 | "Assigned ({n})" | 已被分配该图纸的 Site Engineer 列表，每人右侧显示 [✕] 移除按钮 | 50% |
| 右栏 | "Available ({n})" | 项目中尚未分配该图纸的 Site Engineer 列表，每人右侧显示 [Add] 按钮 | 50% |

栏位属性：
- 背景 #F5F7FA，圆角 4px，内边距 12px
- 最大高度 360px，超出内部竖向滚动
- 两栏间距 16px

#### 3.5.4 人员卡片

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF，圆角 4px，内边距 8px 12px，底部间距 4px |
| 头像 | 姓名首字母圆形头像，28px，`--color-primary-light-8` 背景，`--color-primary` 文字，12px |
| 姓名 | 13px，`--font-weight-medium`，`--color-text-regular` |
| 角色 | "Site Engineer"，12px，`--color-text-secondary` |
| [✕] 移除按钮（左栏） | el-button size="mini" type="text"，红色 #C62828，图标 el-icon-close；hover 背景 #FFF5F5 |
| [Add] 添加按钮（右栏） | el-button size="mini" type="primary" plain，蓝色 #4A90D9 边框，小圆角 |

**即时反馈**（弹窗内暂存，点击 Save 才提交）：
- 点击 [Add]：该人员从右栏移入左栏，[Add] 按钮消失，左栏计数 +1，右栏计数 -1
- 点击 [✕]：该人员从左栏移回右栏，右栏计数 +1，左栏计数 -1

#### 3.5.5 警示提示区

| 属性 | 规格 |
|------|------|
| 背景 | #FFFDE7（浅黄），圆角 4px，内边距 10px 14px |
| 图标 | ⚠ #F57F17，左侧对齐 |
| 文字 | "Removing an assignee will immediately hide this drawing from their APP list."，13px，#795548 |
| 显示条件 | 当前操作移除了原有分配人员时显示（左栏有人员被移除但尚未保存） |

#### 3.5.6 底部操作行

| 按钮 | 类型 | 规格 |
|------|------|------|
| Cancel | el-button | 灰色，关闭弹窗（有变更时二次确认） |
| Save | el-button type="primary" | 蓝色，提交分配；loading 状态防重复点击 |

**Save 的特殊场景**：
- 左栏（已分配）为空时点击 Save，弹出二次确认："No Site Engineers assigned. This drawing will be hidden from all APP users. Continue?"
- 提交成功后 Toast："Assignment saved"，弹窗关闭，表格 Confirms 列数字刷新

#### 3.5.7 上传成功后的引导提示

| 属性 | 规格 |
|------|------|
| 时机 | 上传图纸弹窗（新建）提交成功后 |
| 样式 | Toast（el-notification），右上角弹出，持续 6 秒 |
| 内容 | "Drawing submitted for approval. Don't forget to [Assign Site Engineers →] to ensure the right people get notified." |
| [Assign] 链接 | 文字链接样式，点击直接打开该图纸的分配弹窗 |

---

### 3.6 审批 Todo 面板

#### 3.6.1 面板入口与结构

```
顶部导航栏：

┌─────────────────────────────────────────────────────────────────────┐
│  [🏠 Logo]   [项目名]                     🔔 [3]   [Avatar ▼]     │
└─────────────────────────────────────────────────────────────────────┘
                                            │
                                   点击铃铛图标弹出
                                            ▼
                    ┌──────────────────────────────────┐
                    │  Notifications              [✕]  │
                    ├──────────────────────────────────┤
                    │  [Todo (2)]  [Messages (1)]       │  ← Tab 切换
                    ├──────────────────────────────────┤
                    │  ┌────────────────────────────┐  │
                    │  │  Drawing Approval Required  │  │
                    │  │  ARCH-001 · V2 · 首层平面图  │  │
                    │  │  Uploaded by 李四            │  │
                    │  │  Apr 1, 2026  10:00         │  │
                    │  │  [View] [Approve] [Reject]  │  │
                    │  └────────────────────────────┘  │
                    │  ┌────────────────────────────┐  │
                    │  │  ...                        │  │
                    │  └────────────────────────────┘  │
                    └──────────────────────────────────┘
```

#### 3.6.2 面板属性

| 属性 | 规格 |
|------|------|
| 触发 | 点击顶部导航🔔铃铛图标 |
| 弹出位置 | 铃铛图标正下方，右对齐，`el-popover` 或自定义 dropdown |
| 宽度 | 380px |
| 最大高度 | 520px，超出内部滚动 |
| Tab 切换 | "Todo ({pendingCount})"  /  "Messages ({unreadCount})"，`el-tabs` |
| 标题行 | "Notifications"，14px，`--font-weight-semibold`；右侧关闭 ✕ |
| 角标 | 铃铛右上角，Todo + Messages 数之和，红色圆圈，最大 "99+" |

#### 3.6.3 图纸审批 Todo 卡片

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF |
| 圆角 | 4px |
| 边框 | 1px #EBEEF5 |
| 内边距 | 12px 16px |
| 卡片间距 | 8px |
| 类型标题 | "Drawing Approval Required"，13px，`--font-weight-semibold`，`--color-text-regular` |
| 图纸信息 | "{drawingCode} · {versionNo} · {name}"，13px，`--color-text-secondary`，上边距 4px |
| 元信息 | "Uploaded by {name}"，12px，`--color-text-secondary` |
| 时间 | 日期时间，12px，`--color-text-secondary` |
| 操作按钮间距 | 8px，上边距 10px |

#### 3.6.4 操作按钮

| 按钮 | 类型 | 规格 |
|------|------|------|
| View | el-button size="mini" | 默认样式，"View" |
| Approve | el-button size="mini" type="primary" | 蓝色，"Approve" |
| Reject | el-button size="mini" type="danger" plain | 红色边框，"Reject" |

#### 3.6.5 审批通过确认弹窗

| 属性 | 规格 |
|------|------|
| 组件 | `this.$confirm()` — Element UI MessageBox |
| 标题 | "Approve Drawing" |
| 内容 | "This version will become active and all previous versions will be deprecated. Assigned site engineers will be notified." |
| Cancel | "Cancel" |
| Confirm | "Approve"，type="primary" |

#### 3.6.6 驳回意见弹窗

| 属性 | 规格 |
|------|------|
| 组件 | `el-dialog`，宽度 400px |
| 标题 | "Reject Drawing" |
| Comment 标签 | "Rejection Comment *"，`--font-weight-medium`，红色 * |
| 文本输入框 | `el-input` type="textarea" rows=4，placeholder "Please enter rejection reason..." |
| 字数限制 | 最大 500 字，右下角 x/500 |
| Cancel 按钮 | `el-button`，"Cancel" |
| Confirm 按钮 | `el-button` type="danger"，"Reject"；Comment 为空时 disabled |

---

### 3.7 版本历史抽屉

#### 3.7.1 抽屉结构

```
屏幕右侧滑入：
┌──────────────────────────────────────────────────────┐
│ ← Drawing History — ARCH-001 · 首层平面图              │  ← 标题
│   Architectural                                       │  ← 分类副标题
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  V3  [Current]                               │   │  ← 当前有效版本（高亮行）
│  │  Approved by 王五 · Apr 1, 2026              │   │
│  │  Note: Updated beam dimensions               │   │
│  │  [View]                                      │   │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────┐   │
│  │  V2  [Deprecated]                            │   │
│  │  Approved by 王五 · Mar 20, 2026             │   │
│  │  Note: Added detail section                  │   │
│  │  [View]                                      │   │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────┐   │
│  │  V1  [Deprecated]                            │   │
│  │  Approved by 王五 · Mar 1, 2026              │   │
│  │  Note: Initial upload                        │   │
│  │  [View]                                      │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

#### 3.7.2 抽屉属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-drawer`，direction="rtl" |
| 宽度 | 480px |
| 标题 | "Drawing History — {drawingCode} · {name}"，`--font-weight-semibold` |
| 副标题 | 分类名称，14px，`--color-text-secondary` |
| 关闭方式 | ← 返回箭头 / 遮罩点击 |
| 内容区背景 | #F5F7FA |
| 内边距 | 16px 20px |

#### 3.7.3 版本卡片

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF |
| 圆角 | 4px |
| 边框 | 1px #EBEEF5；Current 版本使用主色 #4A90D9 边框 |
| 内边距 | 12px 16px |
| 卡片间距 | 8px |
| 版本号 | "{versionNo}"，14px，`--font-weight-semibold` |
| 操作人 | "Approved by {name} · {date}" or "Rejected by {name} · {date}"，13px，`--color-text-secondary` |
| 版本说明 | "Note: {versionNote}"，13px，`--color-text-regular`，灰色引用块样式，左侧 3px `--color-border-light` 竖线 |
| View 按钮 | el-button size="mini"，右对齐 |

#### 3.7.4 版本状态标签

| 状态 | 文字 | 背景 | 文字色 |
|------|------|------|--------|
| Current | Current | #F0FAF0 | #2E7D32 |
| Deprecated | Deprecated | #F5F5F5 | #9E9E9E |
| Pending | Pending | #FFFDE7 | #F57F17 |
| Rejected | Rejected | #FFF5F5 | #C62828 |

---

### 3.8 查阅确认记录抽屉

#### 3.8.1 抽屉结构

```
屏幕右侧滑入：
┌──────────────────────────────────────────────────────┐
│ ← Confirmation Records — ARCH-001 · V3               │  ← 标题
│   8 Confirmed / 10 Assigned                          │  ← 摘要副标题（仅统计已分配人员）
├──────────────────────────────────────────────────────┤
│  [Confirmed (8)]   [Pending (2)]                     │  ← Tab 切换
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  👤 张三  Site Engineer                      │   │
│  │  Confirmed at Apr 2, 2026  09:30             │   │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────┐   │
│  │  👤 李四  Site Engineer                      │   │
│  │  Confirmed at Apr 2, 2026  14:05             │   │
│  └──────────────────────────────────────────────┘   │
│            ... ...                                   │
└──────────────────────────────────────────────────────┘
```

> "Pending" Tab 显示**已分配但尚未确认**的人员列表（仅显示姓名 + 角色，无确认时间）。未被分配的 Site Engineer 不出现在本抽屉中。

#### 3.8.2 抽屉属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-drawer`，direction="rtl" |
| 宽度 | 420px |
| 标题 | "Confirmation Records — {drawingCode} · {versionNo}" |
| 副标题 | "{confirmed} Confirmed / {total} Assigned"，14px，`--color-text-secondary` |
| Tab 标签 | "Confirmed ({count})"  /  "Pending ({count})" |

#### 3.8.3 已确认记录卡片

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF，圆角 4px，边框 1px #EBEEF5，内边距 12px 16px，底部间距 8px |
| 头像 | 👤 或姓名首字母圆形头像，32px，`--color-primary-light-8` 背景，`--color-primary` 文字 |
| 姓名 | 14px，`--font-weight-medium`，`--color-text-regular` |
| 角色 | 12px，`--color-text-secondary`，姓名右侧或姓名下方 |
| 确认时间 | "Confirmed at {date} {time}"，12px，`--color-text-secondary` |

#### 3.8.4 未确认记录卡片

| 属性 | 规格 |
|------|------|
| 样式 | 与已确认卡片相同 |
| 差异 | 不显示确认时间，右侧显示橙色标签 "Pending"（#FFFDE7 背景，#F57F17 文字） |

---

## 4. 多语言文案

### 4.1 图纸列表页

| Key | English | 中文 |
|-----|---------|------|
| page_title | Drawing Management | 图纸管理 |
| btn_search_toggle | Search | 搜索 |
| filter_code_placeholder | Drawing code... | 搜索图纸编号... |
| filter_name_placeholder | Drawing name... | 搜索图纸名称... |
| filter_category_all | All Categories | 全部分类 |
| filter_status_all | All Status | 全部状态 |
| btn_search | Search | 搜索 |
| btn_cancel | Cancel | 取消 |
| btn_upload_drawing | + Upload Drawing | + 上传图纸 |
| col_code | Code | 图纸编号 |
| col_name | Name | 图纸名称 |
| col_category | Category | 分类 |
| col_version | Ver | 版本 |
| col_status | Status | 状态 |
| col_confirms | Confirms | 确认情况 |
| col_updated | Updated | 更新时间 |
| col_actions | Actions | 操作 |
| action_view | View | 查看 |
| action_history | History | 历史版本 |
| action_confirms | Confirms | 确认记录 |
| action_assign | Assign | 分配人员 |
| action_upload_version | Upload V{n} | 上传 V{n} |
| status_active | Active | 已生效 |
| status_pending | Pending | 待审批 |
| status_rejected | Rejected | 已驳回 |

### 4.2 上传图纸弹窗

| Key | English | 中文 |
|-----|---------|------|
| dialog_title_new | Upload Drawing | 上传图纸 |
| dialog_title_version | Upload New Version — {code} | 上传新版本 — {code} |
| field_code | Drawing Code | 图纸编号 |
| field_name | Drawing Name | 图纸名称 |
| field_category | Category | 分类 |
| field_description | Description | 图纸说明 |
| field_file | Drawing File | 图纸文件 |
| upload_hint | Drag & Drop or | 拖拽或 |
| upload_browse | Browse Files | 选择文件 |
| upload_format_hint | Supports: PDF / PNG / JPG · Max 50MB | 支持：PDF / PNG / JPG · 最大 50MB |
| field_version_note | Version Note | 版本说明 |
| field_approver | Approver | 审批人 |
| version_info_tip | Current version: {cur}. After approval, this will become {next}. | 当前版本：{cur}。审批通过后将升级为 {next}。 |
| btn_cancel | Cancel | 取消 |
| btn_submit | Submit | 提交 |
| validation_code_required | Drawing code is required | 请输入图纸编号 |
| validation_code_duplicate | This drawing code already exists | 图纸编号已存在 |
| validation_file_required | Please upload a drawing file | 请上传图纸文件 |
| validation_file_format | Only PDF, PNG, JPG files are allowed | 仅支持 PDF、PNG、JPG 格式 |
| validation_file_size | File size cannot exceed 50MB | 文件大小不能超过 50MB |
| toast_upload_success | Drawing submitted for approval | 图纸已提交审批 |

### 4.3 审批 Todo 面板

| Key | English | 中文 |
|-----|---------|------|
| panel_title | Notifications | 通知 |
| tab_todo | Todo ({n}) | 待处理 ({n}) |
| tab_messages | Messages ({n}) | 消息 ({n}) |
| todo_card_title | Drawing Approval Required | 图纸审批待处理 |
| label_uploaded_by | Uploaded by {name} | 上传人：{name} |
| btn_view | View | 查看 |
| btn_approve | Approve | 通过 |
| btn_reject | Reject | 驳回 |
| approve_confirm_title | Approve Drawing | 确认通过 |
| approve_confirm_body | This version will become active and all previous versions will be deprecated. Site engineers will be notified. | 该版本将成为当前有效版本，旧版本将自动作废，Site Engineer 将收到推送通知。 |
| reject_dialog_title | Reject Drawing | 驳回图纸 |
| reject_comment_label | Rejection Comment | 驳回意见 |
| reject_comment_placeholder | Please enter rejection reason... | 请输入驳回原因... |
| toast_approved | Drawing approved | 审批通过 |
| toast_rejected | Drawing rejected | 已驳回 |

### 4.4 版本历史抽屉

| Key | English | 中文 |
|-----|---------|------|
| drawer_title_history | Drawing History — {code} · {name} | 版本历史 — {code} · {name} |
| version_label_current | Current | 当前版本 |
| version_label_deprecated | Deprecated | 已作废 |
| version_label_pending | Pending | 待审批 |
| version_label_rejected | Rejected | 已驳回 |
| label_approved_by | Approved by {name} · {date} | 审批人：{name} · {date} |
| label_rejected_by | Rejected by {name} · {date} | 驳回人：{name} · {date} |

### 4.5 查阅确认记录抽屉

| Key | English | 中文 |
|-----|---------|------|
| drawer_title_confirms | Confirmation Records — {code} · {version} | 查阅确认记录 — {code} · {version} |
| subtitle_confirms | {confirmed} Confirmed / {total} Assigned | {confirmed} 已确认 / {total} 已分配 |
| tab_confirmed | Confirmed ({n}) | 已确认 ({n}) |
| tab_pending | Pending ({n}) | 未确认 ({n}) |
| label_confirmed_at | Confirmed at {datetime} | 确认于 {datetime} |
| label_pending_status | Pending | 未确认 |

### 4.6 分配 Site Engineer 弹窗

| Key | English | 中文 |
|-----|---------|------|
| assign_dialog_title | Assign Site Engineers — {code} · {name} | 分配人员 — {code} · {name} |
| assign_col_assigned | Assigned ({n}) | 已分配 ({n}) |
| assign_col_available | Available ({n}) | 未分配 ({n}) |
| assign_btn_add | Add | 添加 |
| assign_btn_remove | Remove | 移除 |
| assign_warning | Removing an assignee will immediately hide this drawing from their APP list. | 移除分配后，该人员的 APP 图纸列表将立即隐藏此图纸。 |
| assign_empty_confirm_title | No assignees | 确认清空分配 |
| assign_empty_confirm_body | No Site Engineers assigned. This drawing will be hidden from all APP users. Continue? | 未分配任何人员，该图纸将对所有 APP 用户不可见。确认保存？ |
| toast_assign_saved | Assignment saved | 分配已保存 |
| toast_upload_assign_hint | Drawing submitted for approval. Don't forget to [Assign Site Engineers →] to ensure the right people get notified. | 图纸已提交审批。别忘了 [分配人员 →]，确保相关人员收到通知。 |

---

## 5. 响应式与分辨率适配

| 分辨率 | 适配策略 |
|--------|---------|
| 1280px（最小） | 表格展示全部列；若宽度不足，操作列折叠为更多按钮 |
| 1440px（标准） | 基准设计尺寸，所有元素完整展示 |
| 1920px（宽屏） | 内容区最大宽度 1600px，居中显示，两侧留白 |
| 4K 屏 | 字号随基准放大，图标高清显示（使用 SVG 或 2× 图标） |
| Drawer | 版本历史抽屉 480px / 确认记录抽屉 420px，不随分辨率变化 |
| Modal | 弹窗宽度固定 560px，居中，超出高度时内部滚动 |

---

## 6. 验收条件

### 6.1 图纸列表页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 表格展示 | 所有列正确渲染，状态标签颜色符合规范 |
| 2 | Search 面板展开/收起 | 点击顶部 [Search] 按钮展开搜索面板，再次点击或点击面板内 [Cancel] 收起 |
| 3 | 筛选功能 | Code / Name / Category / Status 筛选单独或组合均生效 |
| 4 | Cancel | 点击面板内 [Cancel] 后所有搜索条件清空，面板收起，表格恢复无筛选完整列表 |
| 5 | Upload 按钮 | 点击弹出上传图纸弹窗 |
| 6 | Upload Version | 仅 Active 行显示 "Upload V{n}" 按钮，点击弹出上传新版本弹窗 |
| 7 | History 入口 | 点击 History 按钮打开右侧版本历史抽屉 |
| 8 | Confirms 入口 | 点击 Confirms 按钮打开右侧确认记录抽屉 |
| 9 | Assign 入口 | 点击 Assign 按钮打开分配 Site Engineer 弹窗 |
| 10 | 分页 | 分页控件正常翻页，每页条数可选 |

### 6.2 上传图纸弹窗

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 必填校验 | Code / Name / Category / File / Approver 为空时提示报错，Submit 不可点击 |
| 2 | Code 唯一性 | Code blur 后调接口校验，重复时显示错误提示 |
| 3 | 文件类型 | 上传非 PDF/PNG/JPG 文件时报格式错误 |
| 4 | 文件大小 | 超过 50MB 时报大小错误 |
| 5 | 拖拽上传 | 拖拽文件到上传区可正常触发上传 |
| 6 | 进度显示 | 上传过程中显示进度条 |
| 7 | 提交成功 | 提交后弹窗关闭，表格刷新，新增一行状态 Pending |
| 8 | 关闭确认 | 已填写内容后点击取消/遮罩，弹出二次确认 |

### 6.3 上传新版本弹窗

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 只读字段 | Code / Name / Category 只读，不可编辑 |
| 2 | 版本提示 | 信息栏正确显示当前版本和下一版本号 |
| 3 | 版本说明必填 | 新版本 Version Note 必填校验 |

### 6.4 审批 Todo 面板

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 角标计数 | 铃铛角标正确显示待处理数量，处理后更新 |
| 2 | Approve 流程 | 点击 Approve → 弹确认框 → 确认 → Todo 消失 → Toast 成功 |
| 3 | Reject 流程 | 点击 Reject → 弹驳回弹窗 → Comment 空时 Reject 按钮禁用 → 填写后提交 → Todo 消失 |
| 4 | Approve 联动 | 审批通过后，图纸列表页该行状态变为 Active，旧版作废 |
| 5 | Reject 通知 | 驳回成功后，上传人的 Messages Tab 收到站内消息 |

### 6.5 版本历史抽屉

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 版本排序 | 版本列表按版本号降序排列，最新版本在顶 |
| 2 | 状态标签 | Current / Deprecated / Pending / Rejected 标签颜色正确 |
| 3 | Current 高亮 | 当前有效版本卡片使用主色边框高亮 |
| 4 | View 操作 | 点击 View 正确在新标签或弹窗中打开对应版本图纸文件 |

### 6.6 查阅确认记录抽屉

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 数量展示 | 副标题显示"{confirmed} Confirmed / {total} Assigned"，仅统计已分配该图纸的人员，不含未分配人员 |
| 2 | Tab 切换 | Confirmed / Pending 两个 Tab 各自展示正确人员列表 |
| 3 | 已确认时间 | 已确认 Tab 中每人显示正确的确认时间 |
| 4 | Pending 标签 | 未确认 Tab 中每人右侧显示橙色 "Pending" 标签 |
| 5 | 范围正确性 | 未被分配该图纸的 Site Engineer 不出现在本抽屉中 |

### 6.7 分配 Site Engineer 弹窗

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 弹窗打开 | 点击 [Assign] 按钮，弹窗打开，左栏展示已分配人员，右栏展示可添加人员 |
| 2 | 双栏数量准确 | 已分配 + 可添加的人数之和等于项目中拥有 `drawing:view` 权限的 Site Engineer 总数 |
| 3 | 即时反馈 | 点击 [Add] / [✕] 后人员在两栏间即时移动，计数同步更新 |
| 4 | 警示提示 | 有已分配人员被移除（但尚未 Save）时，底部显示黄色警示文字 |
| 5 | 空分配二次确认 | 左栏清空后点击 Save，弹出二次确认对话框，取消后不提交 |
| 6 | 提交成功 | Save 成功后 Toast 提示 "Assignment saved"，弹窗关闭，表格 Confirms 列数字刷新 |
| 7 | 取消分配生效 | 被移除的 Site Engineer 在 APP 端图纸列表立即不再看到该图纸 |
| 8 | 上传引导提示 | 新建图纸上传成功后，通知中含 [Assign Site Engineers →] 快捷链接，点击直接打开对应分配弹窗 |
| 9 | 关闭确认 | 有未保存变更时点击取消或遮罩，弹出二次确认 |
