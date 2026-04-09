# UI 设计规范 — PC 管理端设备加油管理

> **来源需求**: [REQ-002-pc](../../../requirements/pc/REQ-002-pc.md) + [REQ-002-shared](../../../requirements/shared/REQ-002-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（桌面浏览器，Vue 2 + Element UI）
> **生成日期**: 2026-03-24

---

## 1. 设计目标

- 为项目经理和设备管理员提供清晰、高效的加油数据管理界面
- 列表页支持多条件筛选、排序、批量操作，快速定位目标记录
- 统计页以图表直观呈现油耗趋势、设备排行和分包商排行，辅助成本决策
- 新增/编辑表单覆盖全部字段，兼顾信息完整性与录入效率
- 视觉风格与管理后台框架（UI-REQ-001-pc）一致，延续蓝色主品牌色体系
- 遵循 Element UI 组件风格，保持 UI 一致性
- 支持多语言（中文 / English），默认英文界面

---

## 2. 页面清单与用户流程

### 2.1 页面清单

| 序号 | 页面 | 说明 |
|------|------|------|
| 1 | 加油记录列表页 | 筛选区 + 统计摘要卡片 + 数据表格 + 分页 |
| 2 | 新增加油记录抽屉 | 右侧滑入表单（Drawer） |
| 3 | 编辑加油记录抽屉 | 复用新增，标题/按钮文字不同 |
| 4 | 加油记录详情弹窗 | 分组展示所有字段信息 |
| 5 | 删除确认弹窗 | 二次确认 |
| 6 | 油耗统计页 | 趋势图表 + 设备排行 + 分包商排行 + 明细表 |

### 2.2 核心用户流程

```
列表页流程：

 ┌──────────┐    设置筛选条件    ┌──────────┐    点击 Search    ┌──────────┐
 │ 进入列表页│ ──────────────→  │ 选择筛选项│ ──────────────→  │ 列表刷新  │
 │ 默认全部  │                  │          │                  │ 摘要更新  │
 └──────────┘                  └──────────┘                  └──────────┘
       │                                                          │
       │  点击 [+ New]                                             │  点击行
       ↓                                                          ↓
 ┌──────────────┐    填写表单    ┌──────────────┐    ┌──────────────────┐
 │ 新增抽屉弹出  │ ────────────→ │ 点击 Submit  │    │ 详情弹窗显示      │
 └──────────────┘               └──────┬───────┘    │ Edit / Delete    │
                                       │            └──────────────────┘
                                ┌──────┴──────┐
                                ↓             ↓
                           验证通过       验证失败
                                │             │
                                ↓             ↓
                         ┌──────────┐  ┌──────────┐
                         │ 提交成功  │  │ 红色边框  │
                         │ 关闭抽屉  │  │ 错误提示  │
                         │ 刷新列表  │  └──────────┘
                         └──────────┘
```

```
统计页流程：

 ┌──────────┐    设置筛选/分组    ┌──────────┐    数据加载    ┌──────────────┐
 │ 进入统计页│ ──────────────→   │ 筛选变更  │ ───────────→ │ 图表+表格刷新 │
 │ 默认3个月 │                   │ Group By  │              │ 卡片指标更新  │
 └──────────┘                   └──────────┘              └──────────────┘
```

```
导出流程：

 ┌──────────┐    点击 Export    ┌──────────────┐    下载完成    ┌──────────┐
 │ 列表页    │ ──────────────→ │ 生成 .xlsx    │ ───────────→ │ 浏览器下载│
 │ 筛选条件  │                 │ Loading 状态  │              │ 成功提示  │
 └──────────┘                 └──────────────┘              └──────────┘
```

---

## 3. 页面设计规范

### 3.1 入口与导航

| 属性 | 规格 |
|------|------|
| 入口路径 | Sidebar → Equipment Management → Refueling Management（二级菜单） |
| 面包屑 | Equipment Management / Refueling Management |
| Tab 切换 | 列表页顶部 Tab：**Records** \| **Statistics** |
| Tab 选中 | `--color-primary`（#4A90D9），`--font-weight-semibold`，2px 下划线 |
| Tab 未选中 | `--color-text-secondary`（#666666），`--font-weight-regular` |

> 注：Equipment Management 作为 Sidebar 一级菜单，点击展开后续新增的二级菜单。当前阶段二级菜单仅"Refueling Management"一项。

---

### 3.2 加油记录列表页（Records Tab）

#### 4.2.1 整体布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Refueling Management                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Records       Statistics                                           │   │
│  │  ═══════                                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────── 筛选卡片 ───────────────────────────┐   │
│  │  Equipment: [     ▼]   Subcontractor: [     ▼]   Fuel Type: [   ▼] │   │
│  │  Method:    [     ▼]   Date Range: [  📅] ~ [  📅]                 │   │
│  │  Creator:   [     ▼]   Supplier: [        ]   Cost: [   ] ~ [   ]  │   │
│  │                                                 [Search] [Reset]    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─── 摘要 ──┐ ┌─── 摘要 ──┐ ┌─── 摘要 ──┐ ┌─── 摘要 ──┐                │
│  │ ⛽ Total   │ │ 💰 Total  │ │ 📋 Record │ │ 📊 Avg    │                │
│  │ Fuel (L)  │ │ Cost      │ │ Count     │ │ Cost      │                │
│  │ 15,680.50 │ │ $29,044   │ │ 128       │ │ $226.91   │    [+ New] [📥]│
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘                │
│                                                                             │
│  ┌─────────────────────────── 数据表格 ────────────────────────────────┐   │
│  │ □ │ Equipment  │ Code  │ Subcon   │ Date     │ Type   │ Amount(L) │   │
│  │───┼────────────┼───────┼──────────┼──────────┼────────┼───────────│   │
│  │ □ │ Tower Crane│ TC-003│ ABC Cons.│ 2026-03-23│ Diesel│   150.00  │   │
│  │ □ │ Excavator  │ EX-001│ XYZ Eng. │ 2026-03-22│ Diesel│   200.00  │   │
│  │ □ │ Generator  │ GN-002│ ABC Cons.│ 2026-03-21│ Diesel│    80.00  │   │
│  │───┼────────────┼───────┼──────────┼──────────┼────────┼───────────│   │
│  │ (续: Price | Total Cost | Meter | Method | Supplier | Creator | ⚙ )│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                               < 1 2 3 ... 7 >  20▼ 条/页  │
│                                                                             │
│  ┌─────────────────── 批量操作浮动栏 (勾选时显示) ────────────────────┐    │
│  │  已选 3 条记录                          [Delete Selected (3)]      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 4.2.2 页面标题与操作栏

| 元素 | 规格 |
|------|------|
| 页面标题 | "Refueling Management"，`--font-size-lg`（16px），`--font-weight-semibold`，`--color-text-primary` |
| [+ New] 按钮 | Element UI `el-button`，type="primary"，size="medium"，文字 "+ New"，`--radius-md`（6-8px） |
| [📥 Export] 按钮 | Element UI `el-button`，type="default"，size="medium"，图标 + "Export" |
| 按钮间距 | `--spacing-sm`（8px） |
| 按钮位置 | 摘要卡片行右侧对齐，与卡片同行 |

#### 4.2.3 筛选区域

**整体属性**：

| 属性 | 规格 |
|------|------|
| 容器 | 白色卡片，`--radius-sm`（4-8px），`--shadow-content`，内边距 `--spacing-xxl`（24px） |
| 布局 | Flex wrap 多行排列 |
| 字段横向间距 | `--spacing-filter-gap`（16px） |
| 字段纵向间距 | `--spacing-filter-row-gap`（12px） |

**筛选字段**：

| 字段 | 组件 | 宽度 | 说明 |
|------|------|------|------|
| Equipment | `el-select`（可搜索） | 200px | 当前项目设备列表 |
| Subcontractor | `el-select`（可搜索） | 200px | 当前项目分包商列表 |
| Fuel Type | `el-select` | 140px | Diesel / Gasoline / Other |
| Method | `el-select` | 140px | Self-refueling / Tanker / Gas Station |
| Date Range | `el-date-picker`（daterange） | 260px | 日期范围选择器 |
| Creator | `el-select`（可搜索） | 160px | 当前项目用户列表 |
| Supplier | `el-input` | 160px | 文本输入，支持模糊搜索 |
| Cost Range | 两个 `el-input-number` + "~" | 100px + 100px | 最低 ~ 最高 |

**Label 样式**：

| 属性 | 规格 |
|------|------|
| 位置 | 字段上方 |
| 字体 | `--font-size-md`（14px），`--font-weight-regular`（400），`--color-text-secondary`（#666666） |
| 与输入框间距 | `--spacing-form-label`（8px） |

**Select 下拉组件**：

| 属性 | 规格 |
|------|------|
| 高度 | 36px（Element UI medium size） |
| 边框 | 1px `--color-border`（#D9D9D9） |
| 圆角 | `--radius-md`（6px） |
| 聚焦边框 | `--color-primary`（#4A90D9） |
| Placeholder | `--color-text-placeholder`（#BDBDBD） |
| 下拉菜单阴影 | `--shadow-dropdown` |
| 下拉项 Hover | `--color-hover-bg`（#F5F7FA） |
| 下拉项选中 | `--color-primary` 文字 + `--color-primary-bg` 背景 |

**操作按钮**：

| 按钮 | 规格 |
|------|------|
| [Search] | Element `el-button` type="primary"，size="medium"，高 36px |
| [Reset] | Element `el-button` type="default"，size="medium" |
| 按钮位置 | 筛选区域右下角 |
| 按钮间距 | `--spacing-sm`（8px） |

#### 4.2.4 统计摘要卡片

**整体属性**：

| 属性 | 规格 |
|------|------|
| 布局 | 4 张等宽卡片，Flex 排列 |
| 卡片间距 | `--spacing-summary-gap`（16px） |
| 卡片背景 | `--color-bg-component`（#FFFFFF） |
| 圆角 | `--radius-summary-card`（8px） |
| 阴影 | `--shadow-summary-card` |
| 内边距 | 16-20px |
| 与筛选区间距 | `--spacing-lg`（16px） |
| 与表格间距 | `--spacing-lg`（16px） |

**单张卡片布局**：

```
┌─────────────────────────────┐
│  ⛽                          │  ← 图标 24px，圆形背景 40px
│  Total Fuel (L)             │  ← 标题 12px, #999999
│  15,680.50                  │  ← 数值 20-22px, #333333, bold
└─────────────────────────────┘
```

| 元素 | 规格 |
|------|------|
| 图标 | 24px，放在 40px 圆形浅色背景内 |
| 图标背景色 | Total Fuel: #EBF5FF，Total Cost: #E8F5E9，Count: #FFF3E0，Avg: #F5F5F5 |
| 标题 | `--font-size-xs`（12px），`--color-text-disabled`（#999999） |
| 数值 | 20-22px，`--font-weight-bold`，`--color-text-regular`（#333333），千分位格式 |
| 数据更新 | 跟随筛选条件动态计算 |

#### 4.2.5 数据表格

**整体属性**：

| 属性 | 规格 |
|------|------|
| 组件 | `el-table` + `el-table-column` |
| 容器 | 白色卡片，`--radius-sm`（4-8px），`--shadow-content` |
| 表头背景 | #FAFAFA |
| 表头文字 | `--font-size-md`（14px），`--font-weight-medium`（500），`--color-text-regular`（#333333） |
| 表体文字 | `--font-size-md`（14px），`--font-weight-regular`（400），`--color-text-regular` |
| 行高 | 48-52px |
| 单元格内边距 | `--spacing-table-cell`（12px 16px） |
| 行分隔线 | 1px `--color-border-lighter`（#F0F0F0） |
| 斑马纹 | 奇数行 #FFFFFF，偶数行 #FAFAFA（可选，`stripe` 属性） |
| 行 Hover | 背景 `--color-hover-bg`（#F5F7FA） |
| 固定列 | 勾选列 + 操作列固定，中间列可横向滚动 |
| 空状态 | Element UI 默认空状态 + "No refueling records found" |

**列定义**：

| 列名 | 宽度 | 对齐 | 排序 | 特殊样式 |
|------|------|------|------|---------|
| 勾选框 | 50px | center | — | `el-table-column` type="selection" |
| Equipment Name | 150px | left | ✅ | 蓝色可点击链接 `--color-primary`（#4A90D9） |
| Equipment Code | 100px | left | ✅ | `--color-text-secondary`（#666666） |
| Subcontractor | 150px | left | ✅ | `--color-subcontractor`（#757575） |
| Refueling Date | 120px | left | ✅ 默认↓ | 格式 YYYY-MM-DD |
| Fuel Type | 80px | center | ✅ | 彩色标签（见下方） |
| Fuel Amount (L) | 100px | right | ✅ | 两位小数 |
| Unit Price | 80px | right | ✅ | 两位小数，带 $ |
| Total Cost | 100px | right | ✅ | 两位小数，带 $，`--font-weight-medium` |
| Meter Reading | 100px | left | — | 数值 + 单位，无值显示 "—" |
| Method | 100px | left | ✅ | |
| Supplier | 120px | left | — | 无值显示 "—" |
| Creator | 100px | left | ✅ | |
| Operations | 100px | center | — | 固定右侧，编辑+删除 |

**燃油类型标签**：

| 类型 | 背景色 | 文字色 | 尺寸 |
|------|--------|--------|------|
| Diesel | `--color-tag-diesel`（#2196F3） | #FFFFFF | 内边距 2px 8px，`--font-size-xs`（12px），`--radius-tag`（4px） |
| Gasoline | `--color-tag-gasoline`（#FF9800） | #FFFFFF | 同上 |
| Other | `--color-tag-other`（#9E9E9E） | #FFFFFF | 同上 |

**排序图标**：

| 状态 | 样式 |
|------|------|
| 无排序 | 上下双箭头 ↕，`--color-text-placeholder` |
| 升序 | ↑ 高亮 `--color-primary` |
| 降序 | ↓ 高亮 `--color-primary` |

**操作列**：

| 按钮 | 样式 |
|------|------|
| Edit | Element `el-button` type="text"，`--color-primary`（#4A90D9），图标 + "Edit" |
| Delete | Element `el-button` type="text"，`--color-error`（#FF4D4F），图标 + "Delete" |
| 按钮间距 | `--spacing-sm`（8px） |
| Hover | 下划线 |
| 权限控制 | 无权限时按钮不显示（`v-if`） |

#### 4.2.6 分页器

| 属性 | 规格 |
|------|------|
| 组件 | `el-pagination` |
| 位置 | 表格卡片底部右侧对齐 |
| 上边距 | `--spacing-lg`（16px） |
| 功能 | 上一页 / 下一页 / 页码跳转 / 每页条数切换 |
| 每页条数选项 | 10 / 20 / 50 / 100 |
| 默认 | 20 条/页 |
| 总记录数 | 左侧显示 "Total: N records" |

#### 4.2.7 批量操作浮动栏

| 属性 | 规格 |
|------|------|
| 触发 | 勾选 ≥ 1 条记录时从底部滑入 |
| 定位 | 页面底部固定悬浮，居中 |
| 宽度 | 内容自适应 + 左右 padding 24px |
| 高度 | 48-56px |
| 背景 | `--color-bg-component`（#FFFFFF） |
| 阴影 | `--shadow-float-bar`（0 -2px 8px rgba(0,0,0,0.06)） |
| 圆角 | `--radius-md`（6-8px）（上方圆角） |
| 左侧文字 | "已选 N 条记录"，`--font-size-md`，`--color-text-regular` |
| 右侧按钮 | "Delete Selected (N)"，Element `el-button` type="danger" |
| 出入动画 | `--transition-normal`（300ms），从底部 translateY 滑入 |
| 消失 | 取消所有勾选后滑出消失 |

---

### 3.3 新增 / 编辑抽屉面板

#### 4.3.1 整体属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-drawer`，direction="rtl" |
| 宽度 | 480-560px |
| 遮罩层 | `--color-overlay`（rgba(0,0,0,0.4)） |
| 出入动画 | `--transition-slow`（400ms ease），右侧滑入 |
| Header | 标题 + 关闭按钮 |
| Body | 可滚动表单区域 |
| Footer | 底部操作按钮（固定不随滚动） |
| z-index | 2000 |

#### 4.3.2 Header

| 属性 | 规格 |
|------|------|
| 高度 | 56px |
| 下边框 | 1px `--color-border-light`（#E8E8E8） |
| 标题文字（新增） | "New Refueling Record"，`--font-size-lg`（16px），`--font-weight-semibold`，`--color-text-primary` |
| 标题文字（编辑） | "Edit Refueling Record" |
| 关闭按钮 | ✕，20px，`--color-text-secondary`，Hover 色变 `--color-text-regular`，cursor: pointer |
| 内边距 | 0 24px |

#### 4.3.3 表单区域

**整体布局**：

| 属性 | 规格 |
|------|------|
| 布局 | 单列表单 |
| 内边距 | `--spacing-drawer-padding`（24px） |
| 字段间距 | `--spacing-form-field`（20px） |
| 标签位置 | 字段上方（top），Element `el-form` label-position="top" |
| 标签样式 | `--font-size-md`（14px），`--font-weight-medium`（500），`--color-text-regular`（#333333） |
| 必填标记 | 红色 * 号在 Label 前，`--color-error` |
| 可滚动 | `overflow-y: auto`，底部预留 Footer 高度 |

**表单字段**：

| 字段 | 必填 | 组件 | 高度 | 特殊说明 |
|------|------|------|------|---------|
| Equipment | * | `el-select`（可搜索） | 36px | 编辑时禁用（背景 `--color-field-readonly`） |
| Refueling Date | * | `el-date-picker` | 36px | 默认今天 |
| Refueling Time | | `el-time-picker` | 36px | HH:mm |
| Fuel Type | * | `el-radio-group` | 自适应 | 三项横排：Diesel / Gasoline / Other |
| Fuel Amount (L) | * | `el-input-number` | 36px | 两位小数，min=0.01 |
| Unit Price | | `el-input-number` | 36px | 两位小数 |
| Total Cost | | `el-input`（只读） | 36px | 背景 `--color-field-readonly`，自动 = Amount × Price |
| Meter Reading | | `el-input-number` + `el-select`（单位） | 36px | 单位 h / km |
| Refueling Method | * | `el-select` | 36px | Self-refueling / Tanker / Gas Station |
| Supplier | | `el-input` | 36px | 最多 100 字符 |
| Receipt Photos | | `el-upload` | 自适应 | 最多 5 张，格式 jpg/png，单张 ≤ 5MB |
| Remark | | `el-input` type="textarea" | 80px | 最多 500 字符，右下角字数统计 |

**单选按钮（Fuel Type）**：

| 属性 | 规格 |
|------|------|
| 组件 | `el-radio-group` + `el-radio` |
| 选中圆点 | `--color-radio-active`（#4A90D9） |
| 未选中圆点 | `--color-radio-inactive`（#BDBDBD） |
| 文字 | `--font-size-md`（14px），`--color-text-regular` |
| 选项间距 | 24px |

**照片上传区**：

| 属性 | 规格 |
|------|------|
| 组件 | `el-upload`，list-type="picture-card" |
| 上传框尺寸 | 80×80px |
| 边框 | 1px dashed `--color-upload-border`（#D9D9D9） |
| 背景 | `--color-upload-bg`（#FAFAFA） |
| 圆角 | `--radius-photo`（4px） |
| 加号图标 | 24px，`--color-text-placeholder` |
| 缩略图间距 | `--spacing-photo-gap`（8px） |
| 缩略图 Hover | 显示删除和预览图标遮罩 |
| 已满 5 张 | 隐藏上传框 |
| 上传进度 | 百分比条 `--color-primary` |
| 文件格式限制提示 | "支持 JPG/PNG，单张不超过 5MB" |

**字数统计（Remark）**：

| 属性 | 规格 |
|------|------|
| 位置 | Textarea 右下角 |
| 格式 | "N/500" |
| 正常色 | `--color-text-placeholder`（#BDBDBD） |
| 接近上限（≥450） | `--color-warning`（#FAAD14） |
| 达到上限 | `--color-error`（#FF4D4F） |

#### 4.3.4 Footer（操作按钮栏）

| 属性 | 规格 |
|------|------|
| 高度 | 64px |
| 上边框 | 1px `--color-border-light`（#E8E8E8） |
| 内边距 | 0 24px |
| 布局 | Flexbox，右对齐 |
| [Cancel] | Element `el-button` type="default"，size="medium" |
| [Submit] / [Save] | Element `el-button` type="primary"，size="medium" |
| 按钮间距 | `--spacing-md`（12px） |

**表单修改拦截**：

| 属性 | 规格 |
|------|------|
| 触发 | 点击 Cancel 或 ✕ 且表单有修改 |
| 弹窗 | `el-message-box` confirm，标题 "Unsaved Changes" |
| 正文 | "You have unsaved changes. Discard and close?" |
| 确认按钮 | "Discard"（type="primary"） |
| 取消按钮 | "Continue Editing"（type="default"） |

---

### 3.4 详情弹窗

#### 4.4.1 整体属性

| 属性 | 规格 |
|------|------|
| 组件 | `el-dialog`（模态弹窗） |
| 宽度 | 640-720px |
| 遮罩层 | `--color-overlay` |
| 圆角 | `--radius-md`（8px） |
| 阴影 | `--shadow-card` |
| z-index | 2000 |
| 出入动画 | `--transition-slow`，缩放淡入 |

#### 4.4.2 Header

| 属性 | 规格 |
|------|------|
| 标题 | "Refueling Record Detail"，`--font-size-lg`（16px），`--font-weight-semibold` |
| 关闭按钮 | ✕ |
| 下边框 | 1px `--color-border-light` |

#### 4.4.3 内容区域（分组卡片）

```
┌─────────────────────────────────────────────────┐
│  Refueling Record Detail                     ✕  │
├─────────────────────────────────────────────────┤
│                                                  │
│  Equipment Information                          │
│  ─────────────────────────────────              │
│  Equipment Name     Tower Crane #3              │
│  Equipment Code     TC-003                      │
│  Subcontractor      ABC Construction Pte Ltd    │
│                                                  │
│  Refueling Information                          │
│  ─────────────────────────────────              │
│  Refueling Date     2026-03-23                  │
│  Refueling Time     14:30                       │
│  Fuel Type          Diesel  (蓝色标签)           │
│  Fuel Amount (L)    150.00                      │
│  Unit Price         $1.85                       │
│  Total Cost         $277.50                     │
│  Meter Reading      3,520.5 h                   │
│  Refueling Method   Tanker                      │
│  Supplier           Shell Singapore             │
│                                                  │
│  Receipt Photos                                 │
│  ─────────────────────────────────              │
│  [📷 photo1] [📷 photo2]                        │
│                                                  │
│  Other Information                              │
│  ─────────────────────────────────              │
│  Remark             Normal refueling            │
│  Creator            John Smith                  │
│  Created Time       2026-03-23 14:35            │
│                                                  │
├─────────────────────────────────────────────────┤
│                        [Edit]  [Delete]  [Close]│
└─────────────────────────────────────────────────┘
```

**分组标题**：

| 属性 | 规格 |
|------|------|
| 文字 | `--font-size-md`（14px），`--font-weight-semibold`（600），`--color-text-primary`（#212121） |
| 下划线 | 1px `--color-border-lighter`（#F0F0F0），全宽 |
| 下边距 | `--spacing-lg`（16px） |
| 分组间距 | `--spacing-xxl`（24px） |

**字段行**：

| 属性 | 规格 |
|------|------|
| 布局 | 两列：Label（左）+ Value（右），或 Label 在上 Value 在下 |
| Label 宽度 | 160px 固定（右对齐或左对齐） |
| Label 颜色 | `--color-text-secondary`（#666666） |
| Value 颜色 | `--color-text-regular`（#333333） |
| 行高 | 36-40px |
| 无值 | 显示 "—"，`--color-text-placeholder` |

**照片缩略图**：

| 属性 | 规格 |
|------|------|
| 尺寸 | 80×80px |
| 圆角 | `--radius-photo`（4px） |
| 间距 | `--spacing-photo-gap`（8px） |
| 填充 | `object-fit: cover` |
| Hover | 放大镜图标遮罩，cursor: pointer |
| 点击 | 弹出全屏预览（Element `el-image-viewer`），支持左右翻页 |

#### 4.4.4 Footer

| 按钮 | 规格 |
|------|------|
| [Edit] | Element `el-button` type="primary"，size="medium" |
| [Delete] | Element `el-button` type="danger"，size="medium" |
| [Close] | Element `el-button` type="default"，size="medium" |
| 间距 | `--spacing-md`（12px） |
| 位置 | 右对齐 |
| 上边框 | 1px `--color-border-light` |
| 内边距 | 16px 24px |

---

### 3.5 删除确认弹窗

| 属性 | 规格 |
|------|------|
| 组件 | `el-message-box` confirm |
| 图标 | ⚠ 警告，`--color-warning`（#FAAD14） |
| 标题 | "Confirm Delete" |
| 正文（单条） | "Are you sure you want to delete this refueling record?" |
| 正文（批量） | "Are you sure you want to delete N selected refueling records?" |
| 确认按钮 | "Delete"，Element type="danger" |
| 取消按钮 | "Cancel"，Element type="default" |
| Loading | 确认按钮点击后 Loading 状态，禁止重复点击 |

---

### 3.6 油耗统计页（Statistics Tab）

#### 4.6.1 整体布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Refueling Management                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Records       Statistics                                           │   │
│  │                ══════════                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────── 筛选区 ─────────────────────────────────┐   │
│  │  Date Range: [  📅] ~ [  📅]   Equipment: [   ▼]   Type: [    ▼]  │   │
│  │  Subcontractor: [     ▼]                                            │   │
│  │  Group By: ○ Day  ● Month  ○ Week  ○ Equipment  ○ Subcontractor    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──── 汇总 ────┐  ┌──── 汇总 ────┐  ┌──── 汇总 ────┐                    │
│  │ ⛽ Total Fuel │  │ 💰 Total Cost│  │ 📋 Records   │                    │
│  │ 15,680.50 L  │  │ $29,044.93   │  │ 128          │                    │
│  └──────────────┘  └──────────────┘  └──────────────┘                    │
│                                                                             │
│  ┌────────────────────── 趋势图表 ─────────────────────────────────────┐   │
│  │  Oil Consumption Trend                              [柱状图] [折线图]│   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │    (ECharts 柱状图 / 折线图区域)                               │  │   │
│  │  │    X 轴: 时间/设备/分包商                                      │  │   │
│  │  │    Y 轴左: Fuel (L)     Y 轴右: Cost ($)                      │  │   │
│  │  │    ■ Fuel (L)  ■ Cost ($)                                     │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌────── 设备油耗排行 ──────┐  ┌────── 分包商油耗排行 ──────┐              │
│  │  Equipment Ranking       │  │  Subcontractor Ranking      │              │
│  │  Tower Crane  ████ 3200L│  │  ABC Constr. ██████ 5800L  │              │
│  │  Excavator    ███  2800L│  │  XYZ Eng.    ████  4200L   │              │
│  │  Generator    ██   2100L│  │  DEF Builder ███   2500L   │              │
│  │  ...                    │  │  ...                        │              │
│  └──────────────────────────┘  └──────────────────────────────┘              │
│                                                                             │
│  ┌────────────────────── 明细数据表格 ─────────────────────────────────┐   │
│  │ Period  │ Fuel(L)  │ Cost     │ Records │ Avg Fuel │ Avg Cost      │   │
│  │─────────┼──────────┼──────────┼─────────┼──────────┼───────────────│   │
│  │ 2026-03 │ 2,450.00 │ $4,532   │ 18      │ 136.1    │ $251.8        │   │
│  │ 2026-02 │ 2,180.00 │ $4,033   │ 16      │ 136.3    │ $252.1        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 4.6.2 筛选区

| 属性 | 规格 |
|------|------|
| 容器 | 白色卡片，同列表页筛选卡片 |
| 字段 | Date Range + Equipment + Fuel Type + Subcontractor |
| Group By | `el-radio-group`，5 个选项：Day / Week / Month / Equipment / Subcontractor |
| 默认 Group By | Month |
| 默认 Date Range | 最近 3 个月 |
| 筛选变更 | 自动触发数据刷新（无需 Search 按钮） |

#### 4.6.3 汇总指标卡片

| 属性 | 规格 |
|------|------|
| 数量 | 3 张（Total Fuel / Total Cost / Total Records） |
| 样式 | 同列表页摘要卡片 |
| 数据联动 | 根据筛选条件动态更新 |

#### 4.6.4 趋势图表

**图表容器**：

| 属性 | 规格 |
|------|------|
| 容器 | 白色卡片，`--radius-chart-card`（8px），`--shadow-chart-card` |
| 内边距 | `--spacing-xxl`（24px） |
| 图表高度 | 320-400px |
| 图表库 | ECharts |

**图表标题行**：

| 属性 | 规格 |
|------|------|
| 标题 | "Oil Consumption Trend"，`--font-size-lg`（16px），`--font-weight-semibold` |
| 右侧切换 | 柱状图 / 折线图 图标按钮组 |
| 图标选中 | `--color-primary`（#4A90D9） |
| 图标未选中 | `--color-text-placeholder`（#BDBDBD） |

**ECharts 配置**：

| 属性 | 规格 |
|------|------|
| X 轴 | 根据 Group By：日期字符串 / 设备名称 / 分包商名称 |
| Y 轴左 | Fuel Amount (L)，轴线 `--color-border-light` |
| Y 轴右 | Cost ($)，轴线 `--color-border-light` |
| 柱状图色 | `--color-stat-fuel`（#4A90D9） |
| 折线图色 | `--color-stat-cost`（#67C23A） |
| 网格线 | 虚线 `--color-border-lighter`（#F0F0F0） |
| Tooltip | 白色背景，`--shadow-dropdown`，显示日期 + Fuel + Cost |
| 图例 | 底部居中：■ Fuel (L)  ■ Cost ($) |
| 动画 | easing: 'cubicOut'，duration: 600 |

#### 4.6.5 排行图表（设备 + 分包商）

**布局**：

| 属性 | 规格 |
|------|------|
| 布局 | 两列等宽（50% : 50%），Flex 排列 |
| 间距 | `--spacing-chart-gap`（24px） |
| 容器 | 白色卡片，`--radius-chart-card`，`--shadow-chart-card` |
| 图表高度 | 280-320px |

**设备排行（Equipment Fuel Consumption Ranking）**：

| 属性 | 规格 |
|------|------|
| 图表类型 | 水平条形图（ECharts bar, horizontal） |
| 排序 | 加油量降序 |
| 显示 | Top 10，其余归入 "Others" |
| 条形颜色 | 第 1 名 `--color-rank-1`，第 2 `--color-rank-2`，第 3 `--color-rank-3`，其余 `--color-rank-other` |
| 标签 | 条形右侧显示数值 + 百分比，如 "3,200 L (20.4%)" |
| Y 轴 | 设备名称，`--font-size-sm`（13px） |
| Tooltip | 设备名 + 加油量 + 费用 + 记录数 |

**分包商排行（Subcontractor Fuel Consumption Ranking）**：

| 属性 | 规格 |
|------|------|
| 图表类型 | 水平条形图 |
| 排序 | 加油量降序 |
| 显示 | Top 10，其余归入 "Others" |
| 条形颜色 | 同设备排行色阶 |
| 标签 | 条形右侧显示数值 + 百分比 |
| Y 轴 | 分包商名称 |
| Tooltip | 分包商名 + 加油量 + 费用 + 关联设备数 |
| 点击交互 | 点击分包商条形可展开下属设备明细（可选） |

#### 4.6.6 明细数据表格

| 属性 | 规格 |
|------|------|
| 容器 | 白色卡片，同列表页表格容器 |
| 与排行图间距 | `--spacing-chart-gap`（24px） |
| 标题 | "Detail Data"，`--font-size-lg`，`--font-weight-semibold` |

**列定义**：

| 列名 | 宽度 | 对齐 | 排序 | 说明 |
|------|------|------|------|------|
| Period / Name | 160px | left | ✅ | 根据 Group By 显示日期/设备名/分包商名 |
| Fuel Amount (L) | 120px | right | ✅ | 千分位两位小数 |
| Total Cost | 120px | right | ✅ | 千分位两位小数，带 $ |
| Record Count | 100px | right | ✅ | |
| Avg Fuel (L) | 120px | right | ✅ | |
| Avg Cost | 120px | right | ✅ | |

| 属性 | 规格 |
|------|------|
| 表格样式 | 同列表页 `el-table` 规格 |
| 总计行 | 表格底部 Summary Row，`--font-weight-bold`，背景 #FAFAFA |
| 数据联动 | 与图表数据对应，筛选变更后同步刷新 |

---

### 3.7 数据导出交互

| 步骤 | 规格 |
|------|------|
| 1. 点击 [📥 Export] | 按钮进入 Loading 状态 |
| 2. 请求后端 | 传入当前筛选条件，不限分页 |
| 3. 小数据量（≤5000） | 同步下载，浏览器自动下载 .xlsx |
| 4. 大数据量（>5000） | 异步导出，`el-message` 提示 "Export is being processed, you will be notified when ready" |
| 5. 异步完成 | 通知角标更新，点击可下载 |
| 6. 导出成功 | `el-message` success "Export completed" |
| 7. 导出失败 | `el-message` error "Export failed, please try again" |

**导出文件名格式**: `Refueling_Records_{项目名}_{YYYYMMDD}.xlsx`

---

## 4. 多语言文案

### 4.1 列表页

| Key | English | 中文 |
|-----|---------|------|
| page_title | Refueling Management | 加油管理 |
| tab_records | Records | 记录 |
| tab_statistics | Statistics | 统计 |
| btn_new | + New | + 新增 |
| btn_export | Export | 导出 |
| btn_search | Search | 搜索 |
| btn_reset | Reset | 重置 |
| filter_equipment | Equipment | 设备 |
| filter_subcontractor | Subcontractor | 分包商 |
| filter_fuel_type | Fuel Type | 燃油类型 |
| filter_method | Method | 加油方式 |
| filter_date_range | Date Range | 日期范围 |
| filter_creator | Creator | 创建人 |
| filter_supplier | Supplier | 供应商 |
| filter_cost_range | Cost Range | 费用范围 |
| summary_total_fuel | Total Fuel (L) | 总加油量 (L) |
| summary_total_cost | Total Cost | 总费用 |
| summary_record_count | Record Count | 记录数 |
| summary_avg_cost | Avg Cost / Record | 平均费用/条 |
| col_equipment_name | Equipment Name | 设备名称 |
| col_equipment_code | Equipment Code | 设备编号 |
| col_subcontractor | Subcontractor | 所属分包商 |
| col_refueling_date | Refueling Date | 加油日期 |
| col_fuel_type | Fuel Type | 燃油类型 |
| col_fuel_amount | Fuel Amount (L) | 加油量 (L) |
| col_unit_price | Unit Price | 单价 |
| col_total_cost | Total Cost | 总费用 |
| col_meter_reading | Meter Reading | 表读数 |
| col_method | Method | 加油方式 |
| col_supplier | Supplier | 供应商 |
| col_creator | Creator | 创建人 |
| col_operations | Operations | 操作 |
| op_edit | Edit | 编辑 |
| op_delete | Delete | 删除 |
| pagination_total | Total: {n} records | 共 {n} 条记录 |
| batch_selected | {n} record(s) selected | 已选 {n} 条记录 |
| batch_delete | Delete Selected ({n}) | 删除所选 ({n}) |
| empty_data | No refueling records found | 暂无加油记录 |

### 4.2 新增/编辑抽屉

| Key | English | 中文 |
|-----|---------|------|
| drawer_title_new | New Refueling Record | 新增加油记录 |
| drawer_title_edit | Edit Refueling Record | 编辑加油记录 |
| field_equipment | Equipment | 设备 |
| field_refueling_date | Refueling Date | 加油日期 |
| field_refueling_time | Refueling Time | 加油时间 |
| field_fuel_type | Fuel Type | 燃油类型 |
| field_fuel_amount | Fuel Amount (L) | 加油量 (L) |
| field_unit_price | Unit Price | 单价 |
| field_total_cost | Total Cost | 总费用 |
| field_meter_reading | Meter Reading | 表读数 |
| field_meter_unit | Unit | 单位 |
| field_method | Refueling Method | 加油方式 |
| field_supplier | Supplier | 供应商 |
| field_photos | Receipt Photos | 收据照片 |
| field_remark | Remark | 备注 |
| fuel_diesel | Diesel | 柴油 |
| fuel_gasoline | Gasoline | 汽油 |
| fuel_other | Other | 其他 |
| method_self | Self-refueling | 自行加油 |
| method_tanker | Tanker | 油罐车 |
| method_station | Gas Station | 加油站 |
| meter_unit_h | h | 小时 |
| meter_unit_km | km | 公里 |
| btn_cancel | Cancel | 取消 |
| btn_submit | Submit | 提交 |
| btn_save | Save | 保存 |
| photo_hint | Support JPG/PNG, max 5MB per file | 支持 JPG/PNG，单张不超过 5MB |
| remark_count | {n}/500 | {n}/500 |
| error_required | This field is required | 此字段为必填 |
| auto_calculated | Auto calculated | 自动计算 |

### 4.3 详情弹窗

| Key | English | 中文 |
|-----|---------|------|
| detail_title | Refueling Record Detail | 加油记录详情 |
| group_equipment | Equipment Information | 设备信息 |
| group_refueling | Refueling Information | 加油信息 |
| group_photos | Receipt Photos | 收据照片 |
| group_other | Other Information | 其他信息 |
| field_created_time | Created Time | 创建时间 |
| btn_close | Close | 关闭 |
| no_value | — | — |

### 4.4 删除确认

| Key | English | 中文 |
|-----|---------|------|
| confirm_delete_title | Confirm Delete | 确认删除 |
| confirm_delete_single | Are you sure you want to delete this refueling record? | 确定要删除此条加油记录吗？ |
| confirm_delete_batch | Are you sure you want to delete {n} selected refueling records? | 确定要删除已选的 {n} 条加油记录吗？ |
| btn_confirm_delete | Delete | 删除 |
| btn_confirm_cancel | Cancel | 取消 |

### 4.5 统计页

| Key | English | 中文 |
|-----|---------|------|
| stat_title | Fuel Consumption Statistics | 油耗统计 |
| stat_group_by | Group By | 分组方式 |
| group_day | Day | 按天 |
| group_week | Week | 按周 |
| group_month | Month | 按月 |
| group_equipment | Equipment | 按设备 |
| group_subcontractor | Subcontractor | 按分包商 |
| chart_trend_title | Oil Consumption Trend | 油耗趋势 |
| chart_equip_rank | Equipment Fuel Consumption Ranking | 设备油耗排行 |
| chart_subcon_rank | Subcontractor Fuel Consumption Ranking | 分包商油耗排行 |
| chart_legend_fuel | Fuel (L) | 加油量 (L) |
| chart_legend_cost | Cost ($) | 费用 ($) |
| table_detail_title | Detail Data | 明细数据 |
| col_period | Period | 时段 |
| col_avg_fuel | Avg Fuel (L) | 平均加油量 (L) |
| col_avg_cost | Avg Cost | 平均费用 |
| chart_no_data | No data available | 暂无数据 |
| others | Others | 其他 |

### 4.6 导出

| Key | English | 中文 |
|-----|---------|------|
| export_processing | Export is being processed, you will be notified when ready | 正在生成导出文件，完成后将通知您 |
| export_success | Export completed | 导出完成 |
| export_failed | Export failed, please try again | 导出失败，请重试 |

### 4.7 表单拦截

| Key | English | 中文 |
|-----|---------|------|
| unsaved_title | Unsaved Changes | 未保存的修改 |
| unsaved_message | You have unsaved changes. Discard and close? | 您有未保存的修改。放弃并关闭吗？ |
| unsaved_discard | Discard | 放弃 |
| unsaved_continue | Continue Editing | 继续编辑 |

---

## 5. 响应式适配

| 屏幕宽度 | 筛选区域 | 摘要卡片 | 排行图表 | 表格 |
|---------|----------|---------|---------|------|
| ≥ 1920px | 3 列排列 | 4 列等宽 | 两列并排 | 所有列可见 |
| 1440-1919px | 3 列排列 | 4 列等宽 | 两列并排 | 部分列隐藏，横向滚动 |
| 1280-1439px | 2 列排列 | 2×2 排列 | 单列堆叠 | 固定首尾列，中间横向滚动 |
| < 1280px | 2 列排列 | 2×2 排列 | 单列堆叠 | 固定首尾列，中间横向滚动 |

**抽屉面板适配**：

| 屏幕宽度 | 抽屉宽度 |
|---------|---------|
| ≥ 1440px | 560px |
| 1280-1439px | 480px |
| < 1280px | 80% 屏宽 |

---

## 6. 验收标准

### 6.1 列表页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 页面加载 | 进入页面 < 2s，表格和摘要卡片正确渲染 |
| 2 | Tab 切换 | Records / Statistics 切换，下划线动画流畅 |
| 3 | 筛选功能 | 所有 8 个筛选字段正常工作，Search 触发查询，Reset 清空 |
| 4 | 摘要卡片 | 4 张卡片数据随筛选联动更新，千分位格式正确 |
| 5 | 表格渲染 | 14 列正确渲染，燃油类型彩色标签正确 |
| 6 | 列排序 | 支持排序的列点击标题切换升序/降序 |
| 7 | 分页器 | 页码切换、条数切换正常，总记录数显示正确 |
| 8 | 批量操作 | 勾选后浮动栏出现，取消勾选后消失，动画流畅 |
| 9 | 操作按钮 | Edit/Delete 根据权限显示，无权限时不可见 |
| 10 | 空状态 | 无数据时显示空状态提示 |
| 11 | 行交互 | 点击行或设备名称打开详情弹窗 |

### 6.2 新增/编辑抽屉

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 抽屉动画 | 右侧滑入 400ms，遮罩层正确 |
| 2 | 表单布局 | 单列排列，Label 在上，间距一致 |
| 3 | 必填校验 | 必填字段为空时红色边框 + 错误提示 |
| 4 | 自动计算 | Unit Price 变化后 Total Cost 自动更新 |
| 5 | 设备禁用 | 编辑模式下设备 Select 禁用，灰色背景 |
| 6 | 照片上传 | 最多 5 张，格式和大小限制正确，上传进度显示 |
| 7 | 字数统计 | Remark 区域字数实时统计，接近/达到上限变色 |
| 8 | 未保存拦截 | 有修改时关闭弹出确认对话框 |
| 9 | 提交成功 | 关闭抽屉、刷新列表、`el-message` success |

### 6.3 详情弹窗

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 信息完整 | 4 个分组正确展示所有字段 |
| 2 | 无值处理 | 可选字段无值时显示 "—" |
| 3 | 照片预览 | 缩略图可点击放大，支持左右翻页 |
| 4 | 操作按钮 | Edit / Delete / Close 功能正确 |

### 6.4 删除确认

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 单条删除 | 确认弹窗正确显示，Delete 按钮红色 |
| 2 | 批量删除 | 显示选中条数，确认后全部删除 |
| 3 | Loading | 确认按钮 Loading 状态，防重复点击 |
| 4 | 成功提示 | 删除后列表刷新 + success 提示 |

### 6.5 统计页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 图表渲染 | 趋势图表 + 排行图表 < 3s 渲染完成 |
| 2 | 筛选联动 | 修改筛选条件后图表和表格自动刷新 |
| 3 | Group By | 5 种分组切换后 X 轴和数据正确更新 |
| 4 | 趋势图 | 柱状/折线切换，Tooltip 正确，图例可点击切换 |
| 5 | 设备排行 | Top 10 降序排列，百分比正确 |
| 6 | 分包商排行 | Top 10 降序排列，百分比正确 |
| 7 | 明细表格 | 数据与图表一致，支持排序 |
| 8 | 空状态 | 无数据时图表区域显示空状态提示 |

### 6.6 导出

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 导出按钮 | 根据权限显示，Loading 状态正确 |
| 2 | 导出内容 | 包含当前筛选条件下所有记录（不限分页） |
| 3 | 导出字段 | 16 个字段完整，包含所属分包商 |
| 4 | 文件格式 | .xlsx，文件名格式正确 |
| 5 | 大数据量 | >5000 条异步导出不阻塞界面 |
| 6 | 成功/失败 | 对应 Message 提示正确 |
