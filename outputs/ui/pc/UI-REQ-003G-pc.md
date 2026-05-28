---
doc_type: ui_spec
req_id: REQ-003G-pc
version: 0.1.0
status: draft
generated_from: REQ-003G-pc@0.2.0
generated_at: 2026-05-28
owner: ""
---

# UI 设计说明：PC 端 — 图纸管理区域配置与双模式 SE 分配

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-003G-pc.md（主）、REQ-003D-pc.md（Assign SE 弹框基础结构）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc §3.2
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003G-pc @ v0.2.0 |
| 覆盖用户故事 | US-003G-001、US-003G-002、US-003G-003、US-003G-004 |
| 覆盖 AC | AC-003G-001 ~ AC-003G-010b |
| 上次同步时间 | 2026-05-28 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

在图纸管理模块下新增区域配置能力，并在图纸 [Assign] 弹框中提供**单选（Individual）与按区域批量（By Area）**双模式 SE 选择，两种模式可在同一次操作中混合使用，大幅提升大规模图纸分配效率。

### 1.2 设计原则

1. **最小学习成本**：区域配置页风格与现有 DC Configuration 页（UI-REQ-007D-pc）保持一致，管理员无需重新学习交互模式。
2. **增量、不破坏原有流程**：[Assign] 弹框原有双栏结构和手动 Add/Remove 保持不变，按区域选择仅作为 Available 面板的可选快捷入口叠加其上。
3. **灵活混合**：批量 Add 完成后，管理员仍可自由增减单个 SE，最终结果以 [Save] 时 Assigned 列表为准。
4. **无区域时静默降级**：项目尚未配置区域时，[Select by Area] 按钮置灰而非隐藏，Tooltip 引导用户去配置，避免功能"消失"带来困惑。
5. **权限门禁**：区域配置入口及所有配置操作仅对具备 `drawing:area-config` 权限的项目管理员可见。

---

## 2. 信息架构

### 2.1 入口

图纸管理列表页（Drawing Management）顶部操作区 → **[⚙ Area Config]** 次要按钮（仅管理员可见）

### 2.2 页面层级

```
Drawing Management（图纸管理列表页）
├── [⚙ Area Config] 按钮 → Area Configuration 页面（§3.1）
│   ├── [+ Add Area] → Add New Area 弹窗（§3.2）
│   ├── [Edit] → Edit Area 弹窗（§3.3）
│   ├── [Delete] → Delete Area 确认 Dialog（§3.4）
│   └── [Manage SE] → Manage SEs 弹框（§3.5）
│
└── 图纸列表行 [Assign] → Assign Site Engineers 弹框（§3.6）
    ├── 单选模式：Available 列表逐条 [Add]（原有，REQ-003D-pc）
    └── 批量模式：[Select by Area ▾] → Select by Area 下拉面板（§3.7）
```

### 2.3 导航与入口

| 入口位置 | 链接到 | 触发条件 |
|---------|-------|---------|
| 图纸管理页顶部操作区 | Area Configuration 页（独立路由 `/drawing/area-config`） | 仅 `drawing:area-config` 权限可见 |
| Area Configuration 表格行 → [Manage SE] | Manage SEs 弹框 | 任意区域行 |
| 图纸列表行 → [Assign] | Assign Site Engineers 弹框 | 图纸状态为 ACTIVE |

---

## 3. 页面 / 组件清单

### 3.1 Area Configuration 页面（主列表）

**关联 Story**：US-003G-001
**关联 AC**：AC-003G-001、AC-003G-002、AC-003G-003、AC-003G-010b

#### 主布局

```
┌─────────────────────────────────────────────────────────────────────┐
│  Area Configuration                              [+ Add Area]       │
├─────────────────────────────────────────────────────────────────────┤
│                                  Search: [_____________________ 🔍] │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Area Name        │  SE Count  │  Created At   │  Actions    │   │
│  │───────────────────│────────────│───────────────│─────────────│   │
│  │  Zone A           │  5 SEs     │  2026-05-10   │ [Manage SE] │   │
│  │                   │            │               │ [Edit]      │   │
│  │                   │            │               │ [Delete]    │   │
│  │  Basement Level   │  3 SEs     │  2026-05-08   │ [Manage SE] │   │
│  │                   │            │               │ [Edit]      │   │
│  │                   │            │               │ [Delete]    │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

#### 表格列定义

| 列名 | 内容 | 对齐 | 宽度建议 |
|-----|------|------|---------|
| Area Name | 区域名称文本 | 左对齐 | flex-grow |
| SE Count | `{n} SEs`，n = 已绑定 SE 数量 | 居中 | 100px |
| Created At | `YYYY-MM-DD` | 居中 | 130px |
| Actions | [Manage SE]、[Edit]、[Delete] 三个操作，横向排列 | 右对齐 | 200px |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空 | 项目尚未配置任何区域 | 居中插图 + 提示文字 + [+ Add Area] 按钮 | `"No areas configured. Click [+ Add Area] to get started."` |
| 加载 | 列表请求中 | 表格 Skeleton（3 行占位） | — |
| 正常 | 存在 ≥ 1 个区域 | 如布局图所示 | — |
| 错误 | 列表接口失败 | 全页错误提示 + [Retry] 链接 | `"Failed to load area list."` |
| 极端数据 | 区域数量 > 20 | 表格内分页（每页 20 条）；搜索框可用 | — |

#### 权限可见性

| UI 元素 | 项目管理员 | 其他角色 |
|--------|:---------:|:-------:|
| 整个页面及路由 | ✅ | ❌（访问跳转 403） |
| [+ Add Area] | ✅ | — |
| [Edit] / [Delete] / [Manage SE] | ✅ | — |

---

### 3.2 Add New Area 弹窗

**关联 Story**：US-003G-001
**关联 AC**：AC-003G-002、AC-003G-003

#### 布局

```
┌──────────────────────────────────────────────┐
│  Add New Area                          [✕]   │
├──────────────────────────────────────────────┤
│                                              │
│  Area Name *                                 │
│  [_____________________________________]     │
│   e.g. Zone A, Basement                      │
│                                              │
│                                              │
│                       [Cancel]  [Save]       │
└──────────────────────────────────────────────┘
```

#### 表单字段

| 字段 | 类型 | 必填 | 约束 | Placeholder |
|-----|------|------|------|------------|
| Area Name | 文本输入框 | ✅ | 最大 50 字符；同项目内唯一 | `e.g. Zone A, Basement` |

#### 交互规则

| 操作 | 行为 |
|------|------|
| 点击 [Save] — 名称合法且不重复 | 接口调用 → 成功后弹窗关闭，列表刷新，Snackbar `"Area saved successfully."` |
| 点击 [Save] — 名称为空 | 输入框 inline 错误 `"Area name is required."` |
| 点击 [Save] — 名称与已有区域重复 | 输入框 inline 错误 `"Area name already exists."` |
| 点击 [Cancel] 或 [✕] | 弹窗关闭，不创建 |
| 输入超过 50 字符 | 超出字符不可继续输入，或截断并显示字符计数 `{n}/50` |

---

### 3.3 Edit Area 弹窗

**关联 Story**：US-003G-001

#### 布局

与 §3.2 Add New Area 弹窗结构相同，差异点：

- 标题改为 `Edit Area`
- Area Name 输入框回填当前区域名称
- 交互规则与 §3.2 一致（重名校验排除自身）

---

### 3.4 Delete Area 确认 Dialog

**关联 Story**：US-003G-001
**关联 AC**：AC-003G-004

#### 布局

```
┌──────────────────────────────────────────────────────────┐
│  Delete Area                                             │
│                                                          │
│  Are you sure you want to delete "{Area Name}"?          │
│  All SE bindings in this area will be removed.           │
│  This action cannot be undone.                           │
│                                                          │
│                          [Cancel]    [Delete]            │
└──────────────────────────────────────────────────────────┘
```

#### 交互规则

| 操作 | 行为 |
|------|------|
| [Delete]（Danger 样式） | loading 状态 → 接口调用 → 成功后 Dialog 关闭，列表移除该行，Snackbar `"Area deleted."` |
| [Cancel] 或点击蒙层 | Dialog 关闭，区域不删除 |
| 接口失败 | Toast 报错，Dialog 保留 |

---

### 3.5 Manage SEs 弹框（区域 SE 配置）

**关联 Story**：US-003G-002
**关联 AC**：AC-003G-005

#### 布局

```
┌─────────────────────────────────────────────────────────────────────┐
│  Manage SEs — Zone A                                         [✕]   │
├────────────────────────────┬────────────────────────────────────────┤
│  In Area (3)               │  Available (4)                         │
│  [Search engineer name 🔍] │  [Search engineer name 🔍]             │
│ ─────────────────────────  │ ────────────────────────────────────── │
│  🟦 Zhang Wei              │  🟩 Li Fang                            │
│     Site Engineer [Remove] │     Site Engineer              [Add]   │
│                            │                                        │
│  🟨 Wang Fang              │  🟦 Chen Jing                          │
│     Site Engineer [Remove] │     Site Engineer              [Add]   │
│                            │                                        │
│  🟩 Liu Yang               │  🟥 Sun Hao                            │
│     Site Engineer [Remove] │     Site Engineer              [Add]   │
│                            │                                        │
│                            │  🟪 Zhao Lei                           │
│                            │     Site Engineer              [Add]   │
├────────────────────────────┴────────────────────────────────────────┤
│                                            [Cancel]    [Save]       │
└─────────────────────────────────────────────────────────────────────┘
```

#### 两侧面板规范

| 面板 | 标题格式 | 空状态文案 |
|-----|---------|---------|
| 左侧 In Area | `In Area ({n})` | `"No SEs in this area."` |
| 右侧 Available | `Available ({n})` | `"All SEs have been added to this area."` |

#### SE 列表项结构（每行）

```
[头像色块]  [姓名]           [角色标签]       [操作按钮]
  🟦        Zhang Wei        Site Engineer    [Remove] / [Add]
```

- **头像色块**：圆形，姓名首字母缩写，背景色随机固定（同一用户颜色不变）；尺寸 32×32px
- **角色标签**：灰色 Tag，文案 `Site Engineer`，字号 12px
- **[Add]** / **[Remove]**：文字链接式按钮（text type）；点击仅更新本地状态，不立即调用接口

#### 交互规则

| 操作 | 行为 |
|------|------|
| 点击 [Add] | 该 SE 从右侧消失，出现在左侧；两侧计数实时更新 |
| 点击 [Remove] | 该 SE 从左侧消失，出现在右侧；两侧计数实时更新 |
| 搜索框输入 | 实时过滤**当前面板**列表，不影响另一侧 |
| 点击 [Save] | 以当前 In Area 列表调用接口（覆盖保存）；成功后弹框关闭，Area Configuration 列表 SE Count 实时更新，Snackbar `"Area SE configuration saved."` |
| 接口失败 | Toast 报错，弹框保留，可重试 |
| 点击 [Cancel] 或 [✕] | 弹框关闭，本次修改不保存 |

#### 状态（5 态）

| 状态 | 左侧 In Area | 右侧 Available |
|-----|------------|--------------|
| 空（两侧均空） | `"No SEs in this area."` | `"No SEs available in this project."` |
| 加载 | 列表 Skeleton | 列表 Skeleton |
| 正常 | SE 列表 | SE 列表 |
| 全部在区域内 | SE 列表 | `"All SEs have been added to this area."` |
| 全部在可用 | `"No SEs in this area."` | SE 列表 |

---

### 3.6 Assign Site Engineers 弹框（双模式增强）

**关联 Story**：US-003G-003、US-003G-004
**关联 AC**：AC-003G-006 ~ AC-003G-010a

> ℹ️ 弹框整体结构（标题、双栏布局、[Save] / [Cancel]）沿用 REQ-003D-pc §F-002 的设计规范。本节仅描述新增的双模式增强部分。

#### 布局（增强后完整结构）

```
┌─────────────────────────────────────────────────────────────────────┐
│  Assign Site Engineers — Structural Drawing A-001          [✕]     │
├─────────────────────────────────┬───────────────────────────────────┤
│  Assigned (2)                   │  Available (5)                    │
│  [Search engineer name 🔍]      │  [Select by Area ▾]  ← 新增      │
│ ───────────────────────────     │  [Search engineer name 🔍]        │
│  🟦 Zhang Wei                   │ ─────────────────────────────     │
│     Site Engineer   [Remove]    │  🟩 Li Fang                       │
│                                 │     Site Engineer         [Add]   │
│  🟨 Wang Fang                   │                                   │
│     Site Engineer   [Remove]    │  🟦 Chen Jing                     │
│                                 │     Site Engineer         [Add]   │
│                                 │                                   │
│                                 │  🟥 Sun Hao                       │
│                                 │     Site Engineer         [Add]   │
│                                 │                                   │
│                                 │  🟪 Zhao Lei                      │
│                                 │     Site Engineer         [Add]   │
│                                 │                                   │
│                                 │  🟩 Liu Jian                      │
│                                 │     Site Engineer         [Add]   │
├─────────────────────────────────┴───────────────────────────────────┤
│                                              [Cancel]    [Save]     │
└─────────────────────────────────────────────────────────────────────┘
```

#### [Select by Area ▾] 按钮规范

| 属性 | 值 |
|-----|---|
| 样式 | Secondary Button（次要按钮），带下拉箭头图标 `▾` |
| 位置 | Available 面板搜索框**上方**，独占一行（宽度撑满面板） |
| 状态：有区域 | 可点击，展开下拉面板 |
| 状态：无区域 | 置灰（disabled），hover 显示 Tooltip `"No areas configured. Go to Area Config to set up."` |

---

### 3.7 Select by Area 下拉面板

**关联 Story**：US-003G-003
**关联 AC**：AC-003G-006、AC-003G-007、AC-003G-008

#### 布局

```
┌───────────────────────────────────────────────┐
│  ☑ Select All                                 │
│ ─────────────────────────────────────────── │
│  ☐  Zone A                          (5 SEs)   │
│  ☑  Basement Level                  (3 SEs)   │
│  ☐  Zone B                          (4 SEs)   │
│  ☐  Rooftop                         (2 SEs)   │
│ ─────────────────────────────────────────── │
│                  [Cancel]  [Add Selected Areas]│
└───────────────────────────────────────────────┘
```

#### 元素规范

| 元素 | 说明 |
|-----|------|
| Select All Checkbox | 列表顶部；全选/全不选联动所有区域 Checkbox；处于部分选中时显示 indeterminate 状态 |
| 区域行 | Checkbox + 区域名称（左对齐）+ SE 数量 `({n} SEs)`（右对齐，灰色辅助文字） |
| 分割线 | Select All 下方与底部按钮区上方各一条 |
| [Add Selected Areas] | Primary Button；未勾选任何区域时置灰 |
| [Cancel] | 文字链接式按钮；点击关闭下拉面板，不执行任何操作 |
| 空状态 | 无区域时显示 `"No areas configured."` 并隐藏 Checkbox 列表，[Add Selected Areas] 置灰 |

#### 批量 Add 交互规则

| 步骤 | 说明 |
|-----|------|
| 1. 点击 [Add Selected Areas] | 获取所有勾选区域内 SE 的并集 |
| 2. 幂等过滤 | 已在 Assigned 列表中的 SE 跳过，不重复添加（不报错，静默忽略） |
| 3. 批量加入 | 剩余 SE 全部加入 Assigned 列表；Assigned 计数增加，Available 计数减少 |
| 4. 下拉面板关闭 | 弹框恢复为双栏视图 |
| 5. 后续调整 | 管理员可继续通过单选模式（[Add] / [Remove]）自由增减，两种模式互不干扰 |
| 6. 统一提交 | 点击弹框底部 [Save] 后统一提交，与手动单选结果合并 |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 |
|-----|---------|---------|
| 空 | 项目无已配置区域 | 显示 `"No areas configured."`；[Add Selected Areas] 置灰 |
| 加载 | 下拉面板打开时获取区域列表 | 列表 Skeleton（3 行占位） |
| 正常 | 有区域，未勾选 | 如布局图，全部 Checkbox 未选中，[Add Selected Areas] 置灰 |
| 部分选中 | 已勾选 ≥ 1 个区域 | 对应行 Checkbox 选中，Select All 显示 indeterminate；[Add Selected Areas] 可点击 |
| 全选 | 点击 Select All | 所有行 Checkbox 选中，Select All 勾选 |

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色

| 用途 | Token | 色值（参考） |
|-----|-------|------------|
| SE 头像色块（随机色池） | `--color-avatar-{1~6}` | 沿用现有 Assign SE 弹框头像色规范 |
| [Delete] 按钮 | `--color-danger` | `#F56C6C` |
| 区域行 SE 数量辅助文字 | `--color-text-secondary` | `#909399` |
| 置灰按钮文字 | `--color-text-disabled` | `#C0C4CC` |

### 4.2 间距与字体

| 组件 | 属性 | 值 |
|-----|------|----|
| 页面标题 `Area Configuration` | font-size | 18px |
| 页面标题 | font-weight | 600 |
| 页面标题 | line-height | 26px |
| 页面标题 | color | `--color-text-primary`（`#303133`） |
| 表格行 | font-size | 14px |
| 表格行 | line-height | 22px |
| 表格行 | font-weight | 400 |
| 表格行 | color | `--color-text-regular`（`#606266`） |
| SE 角色标签 `Site Engineer` | font-size | 12px |
| SE 角色标签 | line-height | 20px |
| SE 角色标签 | color | `--color-text-secondary`（`#909399`） |
| 区域 SE 数量辅助文字 `({n} SEs)` | font-size | 13px |
| 区域 SE 数量辅助文字 | color | `--color-text-secondary`（`#909399`） |
| 弹框（Manage SEs / Assign） | width | 720px |
| 弹框内双栏 | 各占 50% 宽度，中间 1px 分割线 |
| 下拉面板（Select by Area） | width | 与 Available 面板等宽 |
| 下拉面板 | max-height | 280px，超出滚动 |
| SE 头像色块 | width × height | 32px × 32px，border-radius 50% |
| Add New Area / Edit Area 弹窗 | width | 420px |
| Delete Area Dialog | width | 420px |
| [+ Add Area] 按钮 | type | primary |
| [Manage SE] 按钮 | type | text（链接式） |
| [Edit] 按钮 | type | text（链接式） |
| [Delete] 按钮 | type | text danger（红色链接式） |
| [Select by Area ▾] 按钮 | type | default（次要），width 100% |
| [Add Selected Areas] 按钮 | type | primary |

---

## 5. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| [Select by Area] 下拉展开 | 向下滑入 | 200ms | ease-out |
| [Select by Area] 下拉收起 | 向上滑出 | 150ms | ease-in |
| 批量 Add SE 后列表更新 | Assigned 列表新增行淡入 | 200ms | ease-in |
| 单条 [Add] / [Remove] | 列表项移入/移出（高度动画） | 150ms | ease-in-out |
| Snackbar 出现 | 从底部滑入 | 300ms | ease-out |

---

## 6. 无障碍（A11y）

- WCAG 等级：AA
- **键盘导航**：
  - Area Configuration 表格支持 Tab 键在行操作按钮间切换
  - Add New Area / Edit Area 弹窗：Tab 切换字段，Enter 触发 [Save]，Esc 关闭
  - Delete Area Dialog：Enter 触发 [Delete]，Esc 触发 [Cancel]
  - Manage SEs 弹框：Tab 在两侧列表间切换，Enter 触发 [Add] / [Remove]，Esc 关闭
  - Select by Area 下拉：Space 切换 Checkbox，Enter 触发 [Add Selected Areas]，Esc 关闭下拉
- **焦点可见**：所有交互元素有 `focus-visible` 样式（2px 蓝色轮廓）
- **ARIA**：
  - 下拉按钮 `aria-haspopup="listbox"` + `aria-expanded`
  - Checkbox 列表 `role="listbox"`，每行 `role="option"` + `aria-selected`
  - Snackbar 使用 `role="status"` + `aria-live="polite"`
  - 批量 Add 完成后，Assigned 面板变更通过 `aria-live="polite"` 播报新增数量

---

## 7. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop | ≥ 1280px | 完整布局，如上述所有设计稿 |
| Tablet | 768~1280px | 弹框宽度自适应（min-width: 600px） |
| Mobile | < 768px | 不支持（PC 专属管理功能） |

---

## 8. 复用与新建组件清单

| 组件 | 来源 | 备注 |
|-----|------|------|
| 数据表格（el-table） | 现有 DS（Element UI） | 沿用现有表格样式规范 |
| Modal Dialog（el-dialog） | 现有 DS（Element UI） | Add / Edit / Delete 弹窗均复用 |
| 双栏 SE 选择弹框结构 | 现有（REQ-003D-pc Assign SE 弹框） | Manage SEs 和 Assign 弹框均复用此结构 |
| 搜索输入框（el-input + 搜索图标） | 现有 DS | — |
| SE 头像色块 | 现有（Assign SE 弹框已有） | 复用相同色块逻辑 |
| Snackbar / Toast | 现有 DS（el-message） | — |
| Checkbox 列表 | 现有 DS（el-checkbox-group） | Select by Area 下拉内使用 |
| **[Select by Area] 下拉面板** | **新建** | Checkbox 列表 + 底部操作按钮的组合，需新建组件 `AreaSelectDropdown` |
| Skeleton（el-skeleton） | 现有 DS | 列表加载态使用 |
| Tooltip（el-tooltip） | 现有 DS | [Select by Area] 置灰时使用 |

---

## 9. 文案规范

| 场景 | 文案（zh-CN） | 文案（en） |
|-----|------------|----------|
| 页面标题 | 区域配置 | Area Configuration |
| 搜索框 placeholder | 搜索区域名称 | Search area name |
| 空状态 | 暂无区域配置，点击 [+ 添加区域] 开始 | No areas configured. Click [+ Add Area] to get started. |
| 表格列：SE 数量 | {n} 人 | {n} SEs |
| 新建弹窗标题 | 新建区域 | Add New Area |
| 编辑弹窗标题 | 编辑区域 | Edit Area |
| Area Name 输入 placeholder | 例：A区、地下室 | e.g. Zone A, Basement |
| 保存成功 Snackbar | 区域保存成功 | Area saved successfully. |
| 区域名重复 inline 错误 | 区域名称已存在 | Area name already exists. |
| 区域名为空 inline 错误 | 区域名称不能为空 | Area name is required. |
| 删除确认标题 | 删除区域 | Delete Area |
| 删除确认内容 | 确定删除"{区域名}"？该区域内所有 SE 绑定将被移除，此操作不可撤销。 | Are you sure you want to delete "{Area Name}"? All SE bindings in this area will be removed. This action cannot be undone. |
| 删除成功 Snackbar | 区域已删除 | Area deleted. |
| Manage SEs 弹框标题 | 管理 SE — {区域名} | Manage SEs — {Area Name} |
| In Area 面板标题 | 已在区域 ({n}) | In Area ({n}) |
| Available 面板标题 | 可选 ({n}) | Available ({n}) |
| In Area 空状态 | 该区域暂无 SE | No SEs in this area. |
| Available 空状态（全部已加入） | 所有 SE 已加入该区域 | All SEs have been added to this area. |
| Available 空状态（项目无 SE） | 该项目暂无 SE 成员 | No SEs available in this project. |
| Manage SEs 保存成功 | 区域 SE 配置已保存 | Area SE configuration saved. |
| [Select by Area] 按钮文字 | 按区域选择 ▾ | Select by Area ▾ |
| [Select by Area] 置灰 Tooltip | 暂无区域配置，请先前往区域配置页设置 | No areas configured. Go to Area Config to set up. |
| Select by Area 下拉空状态 | 暂无区域配置 | No areas configured. |
| [Add Selected Areas] 按钮文字 | 添加所选区域的 SE | Add Selected Areas |
| [Area Config] 入口按钮文字 | 区域配置 | Area Config |

---

## 10. Figma 与原型链接

- Area Configuration 页面：<!-- 填写 Figma Frame 链接 -->
- Add New Area / Edit Area 弹窗：<!-- 填写 Figma Frame 链接 -->
- Delete Area Dialog：<!-- 填写 Figma Frame 链接 -->
- Manage SEs 弹框：<!-- 填写 Figma Frame 链接 -->
- Assign SE 弹框（双模式增强）：<!-- 填写 Figma Frame 链接 -->
- Select by Area 下拉面板（各状态）：<!-- 填写 Figma Frame 链接 -->

---

## 11. AC 覆盖检查表

| AC ID | 对应章节 | 覆盖? |
|------|---------|------|
| AC-003G-001 | §3.1 权限可见性：[Area Config] 入口及整个页面仅管理员可见 | ✅ |
| AC-003G-002 | §3.2 新建区域成功路径 → Snackbar + 列表刷新 | ✅ |
| AC-003G-003 | §3.2 名称重复 → inline 错误提示 | ✅ |
| AC-003G-004 | §3.4 删除区域 → 确认 Dialog + 列表移除 | ✅ |
| AC-003G-005 | §3.5 Manage SEs 弹框 Add SE → SE Count 更新 + Snackbar | ✅ |
| AC-003G-006 | §3.6 / §3.7 无区域时 [Select by Area] 置灰 + Tooltip | ✅ |
| AC-003G-007 | §3.7 批量 Add 多区域 SE 正常路径（并集加入 Assigned） | ✅ |
| AC-003G-008 | §3.7 批量 Add 幂等性（已在 Assigned 的 SE 不重复添加） | ✅ |
| AC-003G-009 | §3.6 批量 Add 后仍可手动 Add / Remove 单个 SE | ✅ |
| AC-003G-010a | §3.6 + §3.7 双模式混合使用端到端（批量 Add → 手动 Remove → 手动 Add → Save） | ✅ |
| AC-003G-010b | §3.1 区域配置页空状态（无区域时展示空状态 + [+ Add Area] 可用） | ✅ |

---

## 12. 待定问题（Open Questions）

> 引用自 REQ-003G-pc §16 中影响 UI 的项。

| OQ ID | 问题 | 影响 UI 哪部分 |
|------|------|--------------|
| OQ-001 | SE 被加入区域时是否发送站内通知？ | 若是，Manage SEs [Save] 成功后 Snackbar 文案需提及"通知已发送" |
| OQ-002 | Area Config 入口是独立页面还是 Drawer？ | §2.1 入口形式、§3.1 整体布局（Drawer 需调整宽度和关闭方式） |
| OQ-003 | 区域名称是否需要中英双名称字段？ | §3.2 / §3.3 表单字段数量，影响弹窗高度 |
| OQ-004 | 批量 Add 后，该批次 SE 在 Available 面板是否仍保留可见？ | §3.7 批量 Add 规则第 3 步，影响 Available 计数实时展示逻辑 |
| OQ-005 | 删除区域是否需要校验"区域 SE 有未完成图纸分配"？ | §3.4 Dialog 内容文案，可能需要增加警告说明 |
| OQ-006 | Assigned 面板是否需要区分"来自批量"和"手动添加"的 SE（如不同 Tag 标识）？ | §3.6 Assigned 面板 SE 列表项结构，需新增来源标识 Tag 或 icon |

---

## 13. 验收标准（本文档自身）

UI 设计交付物完成的判定：

- [ ] Figma 稿覆盖 §3 所有页面/组件（共 7 个：Area Configuration 页、Add 弹窗、Edit 弹窗、Delete Dialog、Manage SEs 弹框、Assign SE 弹框增强、Select by Area 下拉面板）
- [ ] 每个页面/弹框均有 5 态设计稿（空/加载/正常/错误/极端数据）
- [ ] §11 AC 覆盖检查表所有项标记 ✅
- [ ] Select by Area 下拉面板的 5 种状态（空/加载/正常/部分选中/全选）均有设计稿
- [ ] 双模式混合操作流程（批量 Add → 手动调整 → Save）已有交互走查原型
- [ ] §12 所有 OQ 已显式列出，未自行编造解决方案
- [ ] 与现有 DS 的差异（新建 `AreaSelectDropdown` 组件）已记录在 §8

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-28 | agent | 初稿，基于 REQ-003G-pc v0.2.0 生成，覆盖全部 7 个页面/组件、11 条 AC |
