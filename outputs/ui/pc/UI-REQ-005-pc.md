---# UI 说明文档 — PC 端 Site Engineer 图纸查阅与局部更新查看

doc_type: ui_spec

source_req: REQ-005-pc (v0.3.5)> **来源需求**: [REQ-005-pc](../../../requirements/pc/REQ-005-pc.md) + [REQ-005-shared](../../../requirements/shared/REQ-005-shared.md)

platform: PC 管理端> **产品**: SMART SITE SYSTEM

design_tokens: UI-REQ-001-pc.md> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）

generated_at: 2026-05-28> **设计令牌参考**: [UI-REQ-001-pc § 3.2](./UI-REQ-001-pc.md)

---> **依赖文档**: [UI-REQ-003-pc.md](./UI-REQ-003-pc.md)（图纸列表基础规范）、[UI-REQ-004-pc.md](./UI-REQ-004-pc.md)（Markups 组件规范）

> **生成日期**: 2026-04-08

# UI 说明文档：PC 端 Site Engineer 图纸查阅与局部更新查看

---

> 本文档依据 [REQ-005-pc.md v0.3.5](../../../requirements/pc/REQ-005-pc.md) 生成，规范 PC 端 Site Engineer 视角下的所有界面布局、组件样式与交互细节。

## 1. 设计目标

---

- 为 Site Engineer 提供与 APP 端功能等价的 PC 端图纸访问体验，降低跨设备切换成本

## 1. 设计目标- 页面布局充分利用大屏优势：图纸预览区域宽大，局部更新列表与预览并排可见

- 将 Site Engineer 的只读视图与管理员的管理视图完全隔离，避免权限干扰

1. 为 Site Engineer 提供清晰的图纸列表，每行代表一次提交记录（DrawingVersion）- Confirm Reading 与 Mark as Read 操作保持跨端视觉一致性，用户无需重新学习

2. 列表直接展示 RFA 信息与未读/总量 Markups 数量，减少进入详情的判断成本- 站内通知角标和下拉面板延续系统已有通知体系，视觉整合度高

3. Confirm Reading 按钮置于 Header 右侧，无需滚动即可完成确认- 支持多语言（中文 / English），默认英文界面

4. Markups 卡片对齐 REQ-004-pc 规范，提供明确的报审溯源信息

5. 整体风格与现有 Drawing Management 模块保持一致，使用 Element UI 2.x 组件体系---



---## 2. 页面清单与用户流程



## 2. 页面清单与用户流程### 2.1 新增视图



| 页面 | 路由 | 说明 || 序号 | 视图 | 类型 | 说明 |

|-----|------|------||------|------|------|------|

| 图纸列表页 | `/drawings` | Site Engineer 专属，仅展示已分配的 DrawingVersion || 1 | Site Engineer 图纸列表页 | 独立路由页面 | 仅展示分配给自己的 ACTIVE 图纸 |

| 图纸在线查看页 | `/drawings/:drawingVersionId` | 图纸预览 + Confirm Reading（Header 右侧）+ Markups Tab || 2 | 图纸详情页（Drawing Preview Tab） | 独立路由页面内 Tab | 在线预览图纸文件，执行 Confirm Reading |

| 3 | 图纸详情页（Markups Tab） | 独立路由页面内 Tab | 查看 ACTIVE 局部更新列表，执行 Mark as Read |

### 用户流程（ASCII）| 4 | PC 端站内通知下拉面板 | 顶部导航弹出层 | 展示最新通知，点击跳转到对应图纸 Markups Tab |



```### 2.2 核心用户流程

侧边栏 Drawings 菜单

       │```

       ▼Site Engineer 主动查看图纸：

  图纸列表页

  ┌──────────────────────────────────────┐  侧边栏菜单 [Drawings]

  │  Markups 列: ●{unread}/{total}       │       │

  │  Status 列: Confirm → / ✓ Read      │       ▼

  └──────────────────────────────────────┘  图纸列表页（仅我的图纸）

       │                      │  Status 列显示 "Confirm →" 或 "✓ Read"

  点击行 / Confirm →      点击 ●{n}/{total}  Markups 列显示橙色 ●n 未读角标

       │                      │       │ 点击任意行

       ▼                      ▼       ▼

  图纸在线查看页            图纸在线查看页  图纸详情页 — Drawing Preview Tab

  Drawing Preview Tab   自动激活 Markups Tab  在线查看图纸

```       │ 点击 [Confirm Reading]（未确认时）

       ▼

---  确认对话框 → [Confirm]

  状态切换为 "✓ Confirmed on Apr 8, 2026"

## 3. 组件设计规范  列表页 Status 列同步更新为 "✓ Read"

```

### 3.1 侧边栏菜单

```

| 属性 | 规格 |Site Engineer 通过通知查看局部更新：

|------|------|

| 菜单项 | "Drawings"，图标 `el-icon-document` |  顶部导航栏 🔔 铃铛（红色角标 +1）

| 与管理员菜单的区别 | Site Engineer 的 "Drawings" 与管理员的 "Drawing Management" 为独立菜单项，不可混用 |       │ 点击铃铛

| 双角色场景 | 若用户同时拥有管理员角色，两个菜单项均显示，标签文字区分 |       ▼

| 高亮规则 | 当前路由匹配 `/drawings`（SE 视图）时高亮 |  通知下拉面板

  "Drawing Update — ARCH-001"

---       │ 点击通知条目

       ▼

### 3.2 Site Engineer 图纸列表页  图纸详情页 — 自动激活 Markups Tab

  局部更新列表（未读条目显示 [New] 标签）

#### 3.2.1 页面布局       │ 点击 [Mark as Read]

       ▼

```  按钮变为 "✓ Read"（绿色）

┌──────────────────────────────────────────────────────────────────────────────────────────────────────┐  列表页 Markups 列计数 -1

│ 📄 Drawings                                                       [🔍 Search...]  [All Category ▼]   │```

├──────────────────────────────────────────────────────────────────────────────────────────────────────┤

│ Description            │ Category  │ RFA No.  │ Subject of RFA    │ Status     │ Markups    │ Confirmed │ Last Updated │```

│──────────────────────────────────────────────────────────────────────────────────────────────────────│Site Engineer 主动查看局部更新：

│ 首层平面图施工图纸      │ Arch.     │ RFA-001  │ 首层平面图报审     │ ✓ Read    │ ●2 / 5    │ Apr 8     │ 2026-04-01   │

│──────────────────────────────────────────────────────────────────────────────────────────────────────│  图纸列表页

│ 基础结构施工详图        │ Struct.   │ —        │ —                 │ Confirm →  │ 0 / 3     │ —         │ 2026-03-20   │  Markups 列 ●2（橙色链接）

│──────────────────────────────────────────────────────────────────────────────────────────────────────│       │ 点击 ●2

│ 通风管道平面图          │ Mech.     │ RFA-003  │ 机电图纸报审       │ ✓ Read    │ —         │ Apr 5     │ 2026-03-10   │       ▼

├──────────────────────────────────────────────────────────────────────────────────────────────────────┤  图纸详情页 — 自动激活 Markups Tab

│ Total: 8                                                                          [< 1  2 >]          │```

└──────────────────────────────────────────────────────────────────────────────────────────────────────┘

```---



> **列表行含义**：每一行代表一次提交记录（DrawingVersion），即设计人员上传的一个 PDF 文件。系统版本号（Current Version）、Drawing Code、Drawing Name 均不在列表中展示。## 3. 组件设计规范



#### 3.2.2 页头工具栏### 3.1 侧边栏菜单



| 属性 | 规格 || 属性 | 规格 |

|------|------||------|------|

| 页面标题 | "Drawings"，20px，`--font-weight-semibold`，`--color-text-primary` || 菜单项 | "Drawings"，图标 `el-icon-document`（或同 Drawing Management 的图纸图标） |

| 搜索框 | `el-input`，宽度 220px，prefix 图标 `el-icon-search`，placeholder "Search by description..."；实时模糊搜索（debounce 300ms） || 与管理员菜单的区别 | Site Engineer 的 "Drawings" 菜单与管理员的 "Drawing Management" 是独立菜单项，不可混用 |

| Category 筛选 | `el-select`，宽度 160px，选项：All / Architectural / Structural / Mechanical / Electrical / Plumbing / Civil / Other || 双角色场景 | 若用户同时拥有管理员角色，两个菜单项均显示，标签文字区分（"Drawings" vs "Drawing Management"） |

| 两者布局 | 右对齐排布，Search 在左，Category 在右，间距 12px || 高亮规则 | 当前页路由匹配 `/drawings`（Site Engineer 视图）时高亮 |

| 无上传按钮 | Site Engineer 视图中不显示 [+ Upload Drawing] |

---

#### 3.2.3 表格列定义

### 3.2 Site Engineer 图纸列表页

| 列名 | 宽度 | 说明 |

|------|------|------|#### 3.2.1 页面布局

| Description | min-width 200px | 提交记录批次描述；14px，`--font-weight-semibold`，`--color-text-primary` |

| Category | 120px | 图纸分类，`--color-text-regular` |```

| RFA No. | 100px | 外部审批报审编号；未发起时显示 `—` |┌──────────────────────────────────────────────────────────────────────────────┐

| Subject of RFA | min-width 160px | 外部审批报审主题；未发起时显示 `—`，`--color-text-regular` |│ 📄 Drawings                                          [🔍 Search...]  [All ▼] │

| Status | 110px | 见 §3.2.4 |├──────────────────────────────────────────────────────────────────────────────┤

| Markups | 100px | 见 §3.2.5 |│ Drawing Code / Name     │ Category    │ Version │ Read Status │ Markups │ Confirmed │ Last Updated │

| Confirmed | 100px | 见 §3.2.6 |│──────────────────────────────────────────────────────────────────────────────│

| Last Updated | 120px | 格式 "yyyy-MM-dd"，`--color-text-secondary`，可排序 |│ ARCH-001                │ Architectural│ V3     │ ✓ Read      │  ●2     │ Apr 8     │ 2026-04-01   │

│ 首层平面图               │              │         │             │         │           │              │

> **不展示的字段**：系统版本号（Current Version）、Drawing Code、Drawing Name 均不作为列表列。│──────────────────────────────────────────────────────────────────────────────│

│ STRU-001                │ Structural   │ V2     │ Confirm →   │  —      │ —         │ 2026-03-20   │

#### 3.2.4 Status 列样式│ 基础结构图               │              │         │             │         │           │              │

│──────────────────────────────────────────────────────────────────────────────│

| 状态 | 展示 | 交互 |│ MEP-002                 │ Mechanical   │ V1     │ ✓ Read      │  —      │ Apr 5     │ 2026-03-10   │

|------|------|------|│ 通风管道平面图            │              │         │             │         │           │              │

| 未确认 | `Confirm →`，蓝色文字链接（`--color-primary`），13px | 点击 → 进入图纸详情页 Drawing Preview Tab |├──────────────────────────────────────────────────────────────────────────────┤

| 已确认 | `✓ Read`，绿色（`--color-success` #2E7D32），带 ✓ 图标，13px | 不可点击，纯状态展示 |│ Total: 8                                                    [< 1  2 >]       │

└──────────────────────────────────────────────────────────────────────────────┘

#### 3.2.5 Markups 列样式```



| 状态 | 展示 | 交互 |#### 3.2.2 页头工具栏

|------|------|------|

| 无 Markup（total = 0） | `—`，灰色 `--color-text-secondary` | 不可点击 || 属性 | 规格 |

| 全部已读（unread = 0，total > 0） | `0 / {total}`，灰色 `--color-text-secondary`，13px | 不可点击 ||------|------|

| 有未读（unread > 0） | `●{unread} / {total}`；`●{unread}` 部分橙色（`--color-warning` #F57F17），`/ {total}` 部分灰色；13px；hover 下划线 | 点击 → 进入详情页并自动激活 Markups Tab || 页面标题 | "Drawings"，20px，`--font-weight-semibold`，`--color-text-primary` |

| 搜索框 | `el-input`，宽度 220px，prefix 图标 `el-icon-search`，placeholder "Search by code or name..."；实时模糊搜索（debounce 300ms） |

> `{total}` = 该 DrawingVersion 下 ACTIVE Markup 总数；`{unread}` = 当前用户尚未标记已读的数量。| Category 筛选 | `el-select`，宽度 140px，选项：All / Architectural / Structural / Mechanical / Electrical / Plumbing / Civil / Other |

| 两者布局 | 右对齐排布，Search 在左，Category 在右，间距 12px |

#### 3.2.6 Confirmed 列样式| 无上传按钮 | Site Engineer 视图中不显示 [+ Upload Drawing] |



| 状态 | 展示 |#### 3.2.3 表格列定义

|------|------|

| 未确认 | `—`，灰色 `--color-text-secondary` || 列名 | 宽度 | 说明 |

| 已确认 | 确认日期（"Apr 8" 格式），`--color-text-secondary`，12px ||------|------|------|

| Drawing Code / Name | min-width 200px | 图纸编号（14px，`--font-weight-semibold`，`--color-text-primary`）+ 换行显示图纸名称（12px，`--color-text-secondary`） |

#### 3.2.7 行交互| Category | 130px | 分类文字，`--color-text-regular` |

| Version | 80px | 当前版本号（如 V3），居中 |

| 交互 | 行为 || Read Status | 110px | 见 §3.2.4 |

|------|------|| Markups | 90px | 见 §3.2.5 |

| 鼠标悬停行 | 背景 `#F5F7FA` || Confirmed | 100px | 见 §3.2.6 |

| 点击行任意区域（Status / Markups 列除外） | 进入图纸详情页，默认激活 Drawing Preview Tab || Last Updated | 120px | 格式 "yyyy-MM-dd"，`--color-text-secondary` |

| 点击 `Confirm →` | 进入详情页 Drawing Preview Tab |

| 点击 `●{n} / {total}` | 进入详情页，直接激活 Markups Tab |#### 3.2.4 Read Status 列样式



#### 3.2.8 空状态| 状态 | 展示 | 交互 |

|------|------|------|

```| 未确认 | `Confirm →`，蓝色文字链接（`--color-primary`），13px | 点击 → 进入图纸详情页 Drawing Preview Tab（自动聚焦 Confirm Reading 按钮） |

┌──────────────────────────────────────────────────────────────┐| 已确认 | `✓ Read`，绿色（`--color-success` #2E7D32），带 ✓ 图标，13px | 不可点击，纯状态展示 |

│                                                              │

│                       📋                                     │#### 3.2.5 Markups 列样式

│              No drawings assigned to you yet.                │

│    Please contact your administrator for drawing access.     │| 状态 | 展示 | 交互 |

│                                                              │|------|------|------|

└──────────────────────────────────────────────────────────────┘| 无未读（n = 0） | `—`，灰色 `--color-text-secondary` | 不可点击 |

```| 有未读（n > 0） | `●{n}`，橙色 `--color-warning` #F57F17，13px，hover 下划线 | 点击 → 进入图纸详情页并自动激活 Markups Tab |



| 属性 | 规格 |#### 3.2.6 Confirmed 列样式

|------|------|

| 图标 | 📋 文档图标，48px，`--color-text-placeholder` || 状态 | 展示 |

| 主文字 | "No drawings assigned to you yet."，14px，`--color-text-secondary` ||------|------|

| 副文字 | "Please contact your administrator for drawing access."，12px，`--color-text-placeholder` || 未确认 | `—`，灰色 |

| 位置 | 表格区域垂直居中 || 已确认 | 确认日期（"Apr 8" 格式），`--color-text-secondary`，12px |



---#### 3.2.7 行交互



### 3.3 图纸详情页| 交互 | 行为 |

|------|------|

#### 3.3.1 页面结构| 鼠标悬停行 | 背景 `#F5F7FA` |

| 点击行任意区域（Read Status / Markups 列除外） | 进入图纸详情页，默认激活 Drawing Preview Tab |

```| 点击 Confirm → | 进入详情页 Drawing Preview Tab，聚焦 Confirm Reading 按钮 |

┌──────────────────────────────────────────────────────────────────────────────┐| 点击 ●{n} | 进入详情页，直接激活 Markups Tab |

│  Drawings  /  首层平面图施工图纸                                              │  ← 面包屑

├──────────────────────────────────────────────────────────────────────────────┤#### 3.2.8 空状态

│  首层平面图施工图纸                              [✓ Confirm Reading]          │

│  Architectural  ·  Updated 2026-04-01                                        │  ← Header```

├──────────────────────────────────────────────────────────────────────────────┤┌──────────────────────────────────────────────────────────────┐

│  [Drawing Preview]   [Markups  ●2]                                           │  ← Tab 栏│                                                              │

├──────────────────────────────────────────────────────────────────────────────┤│                     📋                                       │

│                                                                              ││             No drawings assigned to you yet.                 │

│                      （Tab 内容区）                                           ││    Please contact your administrator for drawing access.     │

│                                                                              ││                                                              │

└──────────────────────────────────────────────────────────────────────────────┘└──────────────────────────────────────────────────────────────┘

``````



已确认时 Header 右侧显示：`✓ Confirmed on Apr 8`（绿色文字，替换按钮）| 属性 | 规格 |

|------|------|

#### 3.3.2 面包屑| 图标 | 📋 文档图标，48px，`--color-text-placeholder` |

| 主文字 | "No drawings assigned to you yet."，14px，`--color-text-secondary` |

| 属性 | 规格 || 副文字 | "Please contact your administrator for drawing access."，12px，`--color-text-placeholder` |

|------|------|| 位置 | 表格区域垂直居中 |

| 组件 | `el-breadcrumb`，separator="/" |

| 层级 | Drawings（可点击，返回列表）→ {description}（不可点击） |---

| 字体 | 13px，`--color-text-secondary`；当前页文字 `--color-text-regular` |

### 3.3 图纸详情页

#### 3.3.3 Header 信息条

#### 3.3.1 页面结构

| 属性 | 规格 |

|------|------|```

| 背景 | 白色 #FFFFFF，下边框 1px `--color-border-base` |┌──────────────────────────────────────────────────────────────────────────────┐

| 内边距 | 16px 24px |│  Drawings  /  ARCH-001 首层平面图                                             │  ← 面包屑

| 布局 | Flex，两端对齐（左侧标题区 + 右侧按钮区） |├──────────────────────────────────────────────────────────────────────────────┤

| 左侧标题区 | Description（16px，`--font-weight-semibold`，`--color-text-primary`） |│  ARCH-001  首层平面图                                                          │

| 左侧元信息行 | `{Category} · Updated {date}`，12px，`--color-text-secondary`，标题下方 4px |│  Architectural  ·  V3  ·  🟢 Active  ·  Updated 2026-04-01                  │  ← 图纸信息条

| 右侧按钮区（未确认） | `[Confirm Reading]` 主按钮（`el-button type="primary"`），垂直居中 |├──────────────────────────────────────────────────────────────────────────────┤

| 右侧按钮区（已确认） | `✓ Confirmed on {date}`，`--color-success`（#2E7D32），14px，不可点击 |│  [Drawing Preview]   [Markups  ●2]                                           │  ← Tab 栏

| 不展示 | 系统版本号（Current Version）不在 Header 中显示 |├──────────────────────────────────────────────────────────────────────────────┤

│                                                                              │

#### 3.3.4 Tab 栏│  （Tab 内容区，见 §3.4 / §3.5）                                              │

│                                                                              │

| 属性 | 规格 |└──────────────────────────────────────────────────────────────────────────────┘

|------|------|```

| 组件 | `el-tabs type="card"` |

| Tab 1 | "Drawing Preview" |#### 3.3.2 面包屑

| Tab 2 | "Markups {badge}"；`{badge}` 为橙色气泡，unread > 0 时显示，= 0 时 Tab 标签仅显示 "Markups" |

| 默认激活 | 通常为 Drawing Preview Tab；通知跳转 / 点击 Markups 列角标时激活 Markups Tab || 属性 | 规格 |

|------|------|

---| 组件 | `el-breadcrumb`，separator="/" |

| 层级 | Drawings（可点击，返回列表）→ {drawingCode} {drawingName}（不可点击） |

### 3.4 Drawing Preview Tab| 字体 | 13px，`--color-text-secondary`；当前页文字 `--color-text-regular` |



#### 3.4.1 布局#### 3.3.3 图纸信息条



```| 属性 | 规格 |

┌──────────────────────────────────────────────────────────────────────────────┐|------|------|

│  [Drawing Preview]   [Markups ●2]                                            │| 背景 | 白色 #FFFFFF，下边框 1px `--color-border-base` |

├──────────────────────────────────────────────────────────────────────────────┤| 内边距 | 16px 24px |

│                                                                 [⛶ Fullscreen]│| 图纸编号 | 16px，`--font-weight-semibold`，`--color-text-primary` |

│                                                                              │| 图纸名称 | 14px，`--color-text-regular`，编号右侧同行 |

│                          （在线预览区域）                                     │| 元信息行 | Category · Version · 状态 · Updated 日期，12px，`--color-text-secondary`，信息条第二行 |

│                       支持鼠标滚轮缩放 0.25x~5x                               │| 状态图标 | 🟢 Active：绿色圆点 + "Active" 文字 |

│                       支持鼠标拖拽平移                                        │

│                                                                              │#### 3.3.4 Tab 栏

└──────────────────────────────────────────────────────────────────────────────┘

```| 属性 | 规格 |

|------|------|

> Confirm Reading 按钮已移至 Header 右侧（§3.3.3），预览内容区不再显示确认按钮。| 组件 | `el-tabs type="card"` |

| Tab 1 | "Drawing Preview"，图标 `el-icon-document` |

#### 3.4.2 预览区规格| Tab 2 | "Markups {badge}"；`{badge}` 为橙色气泡（同 §3.5.2），未读 n>0 时显示，n=0 时 Tab 标签仅显示 "Markups" |

| 默认激活 | 通常为 Drawing Preview Tab；通知跳转 / 点击 ●{n} 时激活 Markups Tab |

| 属性 | 规格 |

|------|------|---

| 背景 | #F0F2F5（灰色画布感） |

| 最小高度 | calc(100vh - 200px) |### 3.4 Drawing Preview Tab

| 文件类型 | PDF → `<iframe>` 内嵌；PNG / JPG → `<img>`；DWG → 后端转换 PDF 后展示 |

| 缩放 | 鼠标滚轮（`transform: scale()`），范围 0.25× ～ 5× |#### 3.4.1 布局

| 平移 | 鼠标按住拖拽，超出边界自动限制 |

| 全屏 | 右上角 [⛶ Fullscreen]（`el-button size="mini" type="text"`），调用 `requestFullscreen()`；ESC 退出 |```

| 加载中 | 预览区居中显示 `el-loading` 动画 |┌──────────────────────────────────────────────────────────────────────────────┐

| 加载失败 | 居中显示错误图标 + "Failed to load drawing preview." + [Retry] 链接 |│  [Drawing Preview]   [Markups ●2]                                            │

├──────────────────────────────────────────────────────────────────────────────┤

#### 3.4.3 Confirm Reading 确认弹窗│                                                                              │

│                                                                 [⛶ Fullscreen]│

点击 Header 中 [Confirm Reading] 后弹出：│                                                                              │

│                        （在线预览区域）                                       │

| 属性 | 规格 |│                      支持鼠标滚轮缩放                                         │

|------|------|│                      支持鼠标拖拽平移                                         │

| 组件 | `this.$confirm()`，Element UI MessageBox |│                                                                              │

| 内容 | "Confirm that you have read this drawing?" |│                                                                              │

| 副文字 | "{description} · {category}" |├──────────────────────────────────────────────────────────────────────────────┤

| 取消 | "Cancel" |│  V3  ·  Updated 2026-04-01                                                   │

| 确认 | "Confirm"，`type="primary"` |│                                                                              │

| 成功后 | Header 右侧按钮替换为 "✓ Confirmed on {date}"（绿色，不可点击）；列表页 Status / Confirmed 列同步更新 |│  ┌───────────────────────────────┐                                           │

│  │  ✓  Confirm Reading           │     ← 未确认状态                          │

---│  └───────────────────────────────┘                                           │

│                                                                              │

### 3.5 Markups Tab│  ✓ Confirmed on Apr 8, 2026      ← 已确认状态（替换按钮）                    │

│                                                                              │

#### 3.5.1 整体布局└──────────────────────────────────────────────────────────────────────────────┘

```

```

┌──────────────────────────────────────────────────────────────────────────────┐#### 3.4.2 预览区

│  [Drawing Preview]   [Markups ●2]                                            │

├──────────────────────────────────────────────────────────────────────────────┤| 属性 | 规格 |

│                                                                              │|------|------|

│  ┌──────────────────────────────────────────────────────────────────────┐   │| 背景 | #F0F2F5（灰色画布感） |

│  │  [New]  A轴节点详图修正                   Apr 8, 2026 · 张三         │   │| 最小高度 | calc(100vh - 240px)，充分利用屏幕高度 |

│  │  Based on SUB-2026-003  · Page 3                                     │   │| 文件类型 | PDF → iframe 内嵌渲染；PNG / JPG → `<img>` 标签；DWG / DXF → 后端转换后以 PDF 或 PNG 预览 |

│  │  A轴与3轴交叉节点详图已更新，新增钢筋排布说明，请对照施工。            │   │| 缩放 | 鼠标滚轮缩放（`transform: scale()`），缩放范围 0.25× ～ 5× |

│  │  📎 node-detail.pdf                                                  │   │| 平移 | 鼠标按住拖拽，超出边界自动限制 |

│  │                                               [✓ Mark as Read]       │   │| 全屏 | 右上角 [⛶ Fullscreen] 按钮（`el-button size="mini" type="text"`），点击调用 `requestFullscreen()`；全屏内可按 ESC 退出 |

│  └──────────────────────────────────────────────────────────────────────┘   │| 加载中 | 预览区居中显示 `el-loading` 动画 |

│                                                                              │| 加载失败 | 居中显示错误图标 + "Failed to load drawing preview." + [Retry] 链接 |

│  ┌──────────────────────────────────────────────────────────────────────┐   │

│  │  C区消防管道路由修正                      Apr 7, 2026 · 张三         │   │#### 3.4.3 底部信息栏

│  │  Based on SUB-2026-003  · Page 5                                     │   │

│  │  消防主管道路由变更，详见附图...  ...Show more                         │   │| 属性 | 规格 |

│  │  📎 route-update.png                                                 │   │|------|------|

│  │                                                          ✓ Read       │   │| 背景 | 白色 #FFFFFF，顶部 1px `--color-border-base` |

│  └──────────────────────────────────────────────────────────────────────┘   │| 内边距 | 16px 24px |

│                                                                              │| 版本信息 | "{versionNo} · Updated {date}"，13px，`--color-text-secondary` |

└──────────────────────────────────────────────────────────────────────────────┘| Confirm Reading 按钮（未确认） | `el-button type="primary"` ✓ Confirm Reading；宽度自适应文字；高度 36px |

```| 已确认状态（替换按钮） | "✓ Confirmed on {date}"，`--color-success`（#2E7D32），14px，`--font-weight-medium`；不可点击，无 hover 样式 |



#### 3.5.2 Tab 标签未读气泡#### 3.4.4 Confirm Reading 确认弹窗



| 属性 | 规格 |点击 [Confirm Reading] 后弹出：

|------|------|

| 气泡形式 | 橙色圆形，最小宽度 18px，高度 18px；超过 99 显示 "99+" || 属性 | 规格 |

| 颜色 | 背景 #F57F17，白色文字，11px ||------|------|

| 位置 | Tab 标签文字右侧，垂直居中 || 组件 | `this.$confirm()`，Element UI MessageBox |

| 未读 = 0 | 气泡不显示，Tab 标签仅显示 "Markups" || 内容 | "Confirm that you have read this drawing?" |

| 副文字 | "{drawingCode} · {drawingName} · {versionNo}" |

#### 3.5.3 局部更新卡片规格| 取消 | "Cancel" |

| 确认 | "Confirm"，`type="primary"` |

| 属性 | 规格 || 成功后 | 底部信息栏按钮替换为 "✓ Confirmed on {date}"；列表页该行 Read Status 更新为 "✓ Read"，Confirmed 列填入日期 |

|------|------|

| 卡片背景 | 白色 #FFFFFF，圆角 6px，边框 1px #EBEEF5，阴影 0 1px 4px rgba(0,0,0,0.06) |---

| 内边距 | 16px 20px |

| 卡片间距 | 12px |### 3.5 Markups Tab

| 内容区背景 | #F5F7FA，内边距 16px 24px |

| [New] 未读标签 | 橙色胶囊，背景 #FF6D00，白色文字 "New"，11px，内边距 2px 8px，圆角 10px；位于标题左侧；当前用户已标记已读后隐藏 |#### 3.5.1 整体布局

| 标题 | DrawingMarkup 的 description 字段；14px，`--font-weight-semibold`，`--color-text-primary` |

| 元信息行 | `{发布日期} · {创建人}`，12px，`#909399`；标题右侧右对齐，或标题下方第一行 |```

| 报审号+页码行 | 元信息行下方；格式：`Based on {submissionNo}`；若 `appliedPageNo` 不为 null 则追加 ` · Page {n}`；12px，`#909399` |┌──────────────────────────────────────────────────────────────────────────────┐

| 说明文字 | 13px，`--color-text-regular`，行高 1.6，最多 3 行，超出显示 "…Show more"（`--color-primary`） |│  [Drawing Preview]   [Markups ●2]                                            │

| 附件列表 | 📎 图标（12px）+ 文件名（`--color-primary`，hover 下划线）；点击在新标签页打开 |├──────────────────────────────────────────────────────────────────────────────┤

| 操作区 | 见 §3.5.4 / §3.5.5 |│                                                                              │

│  ┌──────────────────────────────────────────────────────────────────────┐   │

> **不展示的字段**：Affected Area（影响区域）不在 SE 视图卡片中显示。│  │  [New]  A轴节点详图修正                              Apr 8, 2026     │   │

│  │  Published by 张三                                                    │   │

#### 3.5.4 [Mark as Read] 按钮（未读状态）│  │  Affected Area: [A-C轴 / 3-5层]                                      │   │

│  │  A轴与3轴交叉节点详图已更新，新增钢筋排布说明，请对照施工。            │   │

| 属性 | 规格 |│  │  📎 node-detail.pdf                                                  │   │

|------|------|│  │                                               [✓ Mark as Read]       │   │

| 组件 | `el-button size="small" type="primary" plain` |│  └──────────────────────────────────────────────────────────────────────┘   │

| 文字 | "✓ Mark as Read" |│                                                                              │

| 位置 | 卡片右下角，右对齐 |│  ┌──────────────────────────────────────────────────────────────────────┐   │

| 点击 | loading 防重复；调用 `POST /drawing/markup/confirm`；成功后切换为已读样式 |│  │  C区消防管道路由修正                                 Apr 7, 2026     │   │

│  │  Published by 张三                                                    │   │

#### 3.5.5 已读状态│  │  Affected Area: [C区 / B1层]                                         │   │

│  │  消防主管道路由变更，详见附图...                                       │   │

| 属性 | 规格 |│  │  📎 route-update.png                                                 │   │

|------|------|│  │                                                          ✓ Read       │   │

| 展示 | "✓ Read"，`--color-success`（#2E7D32），13px，`--font-weight-medium` |│  └──────────────────────────────────────────────────────────────────────┘   │

| 位置 | 卡片右下角，右对齐 |│                                                                              │

| 可交互性 | 不可点击，无 hover 效果 |└──────────────────────────────────────────────────────────────────────────────┘

```

#### 3.5.6 空状态

#### 3.5.2 Tab 标签未读气泡

| 场景 | 展示 |

|------|------|| 属性 | 规格 |

| 无局部更新 | 📋 图标 + "No markups for this drawing yet." ||------|------|

| 加载失败 | ⚠ 图标 + "Failed to load. Please refresh." + [Refresh] 文字链接 || 气泡形式 | 橙色圆形，最小宽度 18px，高度 18px；超过 99 时显示 "99+" |

| 颜色 | 背景 #F57F17，白色文字，11px |

---| 位置 | Tab 标签文字右侧，垂直居中 |

| 未读 = 0 | 气泡不显示，Tab 标签仅显示 "Markups" |

### 3.6 PC 端站内通知

#### 3.5.3 局部更新卡片规格

#### 3.6.1 铃铛入口

| 属性 | 规格 |

```|------|------|

┌─────────────────────────────────────────────────────────────────────────┐| 背景 | 白色 #FFFFFF，圆角 6px，边框 1px #EBEEF5，阴影 0 1px 4px rgba(0,0,0,0.06) |

│  SMART SITE     [Drawing Management]  [Drawings]  ...    🔔  👤  [→出]  │| 内边距 | 16px 20px |

│                                                          ③              │| 卡片间距 | 12px |

└─────────────────────────────────────────────────────────────────────────┘| 整体背景 | 内容区背景 #F5F7FA，内边距 16px 24px |

```| [New] 未读标签 | 橙色胶囊，背景 #FF6D00，白色文字 "New"，11px，内边距 2px 8px，圆角 10px；位于标题左侧 |

| 标题 | 14px，`--font-weight-semibold`，`--color-text-primary` |

| 属性 | 规格 || 日期 | 右对齐，12px，`--color-text-secondary` |

|------|------|| 发布人 | "Published by {name}"，12px，`--color-text-secondary`，标题下方 4px |

| 图标 | `el-icon-bell`，20px，`--color-text-secondary` || 影响区域 Tag | `el-tag size="small"` 默认灰色边框样式，12px；无数据时不显示该行 |

| 未读角标 | 红色圆形，最小宽度 16px，高度 16px，白色文字，10px；>99 显示 "99+" || 说明文字 | 13px，`--color-text-regular`，行高 1.6，最多 3 行，超出显示 "...Show more" 文字链接（`--color-primary`） |

| 点击 | 展开通知下拉面板（`el-popover`），关闭时自动收起 || 附件列表 | 📎 图标（12px）+ 文件名（`--color-primary`，可点击，hover 下划线）；点击在新标签页打开 |

| [Mark as Read] 按钮 | 见 §3.5.4 |

#### 3.6.2 通知下拉面板| ✓ Read | 见 §3.5.5 |



```#### 3.5.4 [Mark as Read] 按钮（未确认状态）

┌─────────────────────────────────────────────────┐

│  Notifications (3 unread)   [Mark all as read]  │| 属性 | 规格 |

├─────────────────────────────────────────────────┤|------|------|

│  ● 📄 Drawing Update — 首层平面图施工图纸        │| 组件 | `el-button size="small" type="success" plain` |

│    A轴节点详图修正. Please review...             │| 文字 | "✓ Mark as Read" |

│    5 minutes ago                                │| 位置 | 卡片右下角，右对齐 |

├─────────────────────────────────────────────────┤| 点击 | loading 防重复；调用 `POST /drawing/markup/confirm`；成功后切换为已确认样式 |

│  ● 📄 Drawing Update — 基础结构施工详图          │

│    外墙保温层厚度修正. Please review...           │#### 3.5.5 已确认状态

│    1 hour ago                                   │

├─────────────────────────────────────────────────┤| 属性 | 规格 |

│    📄 Drawing Update — 通风管道平面图            │  ← 已读|------|------|

│    通风管道路由变更. Please review...             │| 展示 | "✓ Read"，`--color-success`（#2E7D32），13px，`--font-weight-medium` |

│    Apr 5                                        │| 位置 | 卡片右下角，右对齐（与 [Mark as Read] 按钮相同位置） |

├─────────────────────────────────────────────────┤| 图标 | ✓ checkmark，内联于文字前 |

│                    View all notifications →     │| 可交互性 | 不可点击，无 hover 效果 |

└─────────────────────────────────────────────────┘

```#### 3.5.6 空状态



| 属性 | 规格 || 场景 | 展示 |

|------|------||------|------|

| 组件 | `el-popover`，placement="bottom-end"，宽度 360px || 无局部更新 | 📋 图标 + "No markups for this drawing." |

| 面板标题 | "Notifications ({n} unread)"，14px，`--font-weight-semibold` || 加载失败 | ⚠ 图标 + "Failed to load. Please refresh."  + [Refresh] 文字链接 |

| [Mark all as read] | 标题行右侧，`el-button size="mini" type="text"`；无未读时 disabled |

| 最大高度 | 480px，超出内部滚动 |---



#### 3.6.3 单条通知条目样式### 3.6 PC 端站内通知



| 属性 | 规格 |#### 3.6.1 铃铛入口

|------|------|

| 未读背景 | 白色 #FFFFFF；已读：#FAFAFA |```

| 未读圆点 | 左侧 6px 蓝色实心圆点，diameter 6px，垂直居中 |┌─────────────────────────────────────────────────────────────────────────┐

| 标题 | 14px，`--font-weight-semibold`，`--color-text-primary` |│  SMART SITE     [Drawing Management]  [Drawings]  ...    🔔  👤  [→出]  │

| 正文 | 12px，`--color-text-secondary`，最多 2 行，超出省略 |│                                                          ③              │  ← 顶部导航

| 时间 | 12px，`--color-text-placeholder`，右对齐；24h 内相对时间，超出显示日期 |└─────────────────────────────────────────────────────────────────────────┘

| hover | 背景 #F5F7FA |```

| 点击 | 标记已读 + 跳转图纸详情页激活 Markups Tab；面板关闭 |

| 属性 | 规格 |

---|------|------|

| 图标 | `el-icon-bell`，20px，`--color-text-secondary` |

## 4. 多语言文案| 未读角标 | 红色圆形，最小宽度 16px，高度 16px，白色文字，10px；1-99 显示数字，>99 显示 "99+" |

| 位置 | 顶部导航栏右侧用户信息区左侧 |

### 4.1 图纸列表页| 点击 | 展开通知下拉面板（`el-popover`），关闭时自动收起 |



| Key | English | 中文 |#### 3.6.2 通知下拉面板

|-----|---------|------|

| menu_drawings | Drawings | 我的图纸 |```

| search_placeholder | Search by description... | 搜索批次描述... |┌─────────────────────────────────────────────────┐

| col_description | Description | 批次描述 |│  Notifications (3 unread)   [Mark all as read]  │

| col_category | Category | 分类 |├─────────────────────────────────────────────────┤

| col_rfa_no | RFA No. | 报审编号 |│  ● 📄 Drawing Update — ARCH-001                 │

| col_subject_of_rfa | Subject of RFA | 报审主题 |│    A轴节点详图修正. Please review...             │

| col_status | Status | 状态 |│    5 minutes ago                                │

| col_markups | Markups | 局部更新 |├─────────────────────────────────────────────────┤

| col_confirmed | Confirmed | 确认时间 |│  ● 📄 Drawing Update — STRU-001                 │

| col_last_updated | Last Updated | 最后更新 |│    外墙保温层厚度修正. Please review...           │

| status_confirm | Confirm → | 待确认 → |│    1 hour ago                                   │

| status_read | ✓ Read | ✓ 已读 |├─────────────────────────────────────────────────┤

| empty_drawings_title | No drawings assigned to you yet. | 暂无分配给您的图纸 |│    📄 Drawing Update — MEP-002                  │  ← 已读（无 ● 圆点）

| empty_drawings_desc | Please contact your administrator for drawing access. | 请联系管理员获取图纸访问权限 |│    通风管道路由变更. Please review...             │

│    Apr 5                                        │

### 4.2 Confirm Reading├─────────────────────────────────────────────────┤

│                    View all notifications →     │

| Key | English | 中文 |└─────────────────────────────────────────────────┘

|-----|---------|------|```

| btn_confirm_reading | ✓ Confirm Reading | ✓ 确认查阅 |

| confirmed_on | ✓ Confirmed on {date} | ✓ 已于 {date} 确认 || 属性 | 规格 |

| confirm_dialog_msg | Confirm that you have read this drawing? | 确认您已阅读本图纸？ ||------|------|

| confirm_dialog_sub | {description} · {category} | {description} · {category} || 组件 | `el-popover`，placement="bottom-end"，宽度 360px |

| btn_confirm | Confirm | 确认 || 面板标题 | "Notifications ({n} unread)"，14px，`--font-weight-semibold`；无未读时显示 "Notifications" |

| btn_cancel | Cancel | 取消 || [Mark all as read] | 标题行右侧，`el-button size="mini" type="text"`，`--color-primary`；无未读时 disabled 置灰 |

| 最大高度 | 480px，超出内部滚动 |

### 4.3 Markups Tab| 底部链接 | "View all notifications →"，居中，12px，`--color-primary`；点击跳转通知中心页（若有）|



| Key | English | 中文 |#### 3.6.3 单条通知条目样式

|-----|---------|------|

| label_new | New | 新 || 属性 | 规格 |

| label_based_on | Based on {submissionNo} | 基于 {submissionNo} ||------|------|

| label_page | · Page {n} | · 第 {n} 页 || 背景 | 未读：白色 #FFFFFF；已读：#FAFAFA |

| label_show_more | ...Show more | ...展开全文 || 内边距 | 12px 16px |

| btn_mark_as_read | ✓ Mark as Read | ✓ 标记已读 || 分隔线 | 条目间 1px #F0F0F0 |

| label_read | ✓ Read | ✓ 已读 || 未读圆点 | 左侧 6px 蓝色实心圆点（`--color-primary`），diameter 6px，垂直居中 |

| empty_markups | No markups for this drawing yet. | 该图纸暂无局部更新 || 图标 | 📄 橙色文档图标，20px，condition type=DRAWING_MARKUP |

| 标题 | 14px，`--font-weight-semibold`，`--color-text-primary`；未读加粗更明显 |

### 4.4 站内通知| 正文 | 12px，`--color-text-secondary`，最多 2 行，超出省略 |

| 时间 | 12px，`--color-text-placeholder`，右对齐；24h 内显示相对时间（"5 minutes ago"），超出显示日期（"Apr 5"） |

| Key | English | 中文 || hover | 背景 #F5F7FA |

|-----|---------|------|| 点击 | 标记已读 + 跳转 targetRoute（图纸详情页，激活 Markups Tab）；面板关闭 |

| notification_title_unread | Notifications ({n} unread) | 通知（{n} 条未读） |

| btn_mark_all_read | Mark all as read | 全部标为已读 |---

| push_markup_title | Drawing Update — {description} | 图纸更新 — {description} |

| push_markup_body | {markupTitle}. Please review the latest markup. | {markupTitle}。请查阅最新局部更新。 |## 4. 多语言文案



---### 4.1 菜单与页面标题



## 5. 响应式与分辨率适配| Key | English | 中文 |

|-----|---------|------|

| 场景 | 规格 || menu_drawings | Drawings | 我的图纸 |

|------|------|| page_title_drawings | Drawings | 我的图纸 |

| 最小支持分辨率 | 1280px × 720px |

| 图纸列表表格 | 1280px 时 RFA No. / Subject of RFA / Confirmed 可折叠（列显示开关），优先保留 Description / Status / Markups / Last Updated |### 4.2 图纸列表页

| 图纸预览区域 | 高度随视口高度自适应，最小 400px |

| 通知下拉面板 | 固定 360px，不随分辨率缩放 || Key | English | 中文 |

|-----|---------|------|

---| search_placeholder | Search by code or name... | 搜索编号或名称... |

| col_drawing | Drawing Code / Name | 图纸编号 / 名称 |

## 6. 验收条件| col_category | Category | 分类 |

| col_version | Version | 版本 |

### 6.1 菜单与权限| col_read_status | Read Status | 阅读状态 |

| col_markups | Markups | 局部更新 |

| # | 验收项 | 通过标准 || col_confirmed | Confirmed | 确认时间 |

|---|--------|---------|| col_last_updated | Last Updated | 最后更新 |

| 1 | 菜单可见 | SE 登录后侧边栏显示 "Drawings" 菜单项 || status_confirm | Confirm → | 待确认 → |

| 2 | 权限隔离 | SE 无法访问 Drawing Management 路由 || status_read | ✓ Read | ✓ 已读 |

| 3 | 双角色兼容 | 同时拥有管理员角色时，两个菜单项均正确显示且功能独立 || empty_drawings_title | No drawings assigned to you yet. | 暂无分配给您的图纸 |

| empty_drawings_desc | Please contact your administrator for drawing access. | 请联系管理员获取图纸访问权限 |

### 6.2 图纸列表页

### 4.3 图纸详情页

| # | 验收项 | 通过标准 |

|---|--------|---------|| Key | English | 中文 |

| 1 | 列结构 | 显示 Description、Category、RFA No.、Subject of RFA、Status、Markups、Confirmed、Last Updated；**不显示**版本号、Drawing Code、Drawing Name ||-----|---------|------|

| 2 | 数据过滤 | 仅显示 ACTIVE 且已分配给当前用户的 DrawingVersion || breadcrumb_drawings | Drawings | 我的图纸 |

| 3 | RFA 显示 | 外部审批已发起时显示具体值；未发起时显示 `—` || tab_drawing_preview | Drawing Preview | 图纸预览 |

| 4 | Status 列 | 已确认行绿色 "✓ Read"；未确认行蓝色 "Confirm →" || tab_markups | Markups | 局部更新 |

| 5 | Markups 列 | unread>0 → `●{n}/{total}` 橙色前缀；全部已读 → `0/{total}` 灰色；无 Markup → `—` || btn_fullscreen | Fullscreen | 全屏查看 |

| 6 | 搜索筛选 | Description 模糊搜索生效，Category 筛选生效 || preview_loading_failed | Failed to load drawing preview. | 图纸加载失败 |

| 7 | 行/列点击 | 点击行进入 Drawing Preview Tab；点击 Markups 列角标进入 Markups Tab || btn_retry | Retry | 重试 |

| 8 | 空状态 | 无分配图纸时显示正确空状态文案 |

### 4.4 Confirm Reading

### 6.3 Header 与 Confirm Reading

| Key | English | 中文 |

| # | 验收项 | 通过标准 ||-----|---------|------|

|---|--------|---------|| btn_confirm_reading | ✓ Confirm Reading | ✓ 确认查阅 |

| 1 | Header 内容 | 显示 Description 和 Category，不显示版本号 || confirmed_on | ✓ Confirmed on {date} | ✓ 已于 {date} 确认 |

| 2 | 按钮位置 | [Confirm Reading] 在 Header 右侧，无需滚动可见 || confirm_dialog_msg | Confirm that you have read this drawing? | 确认您已阅读本图纸？ |

| 3 | 未确认状态 | Header 右侧显示蓝色主按钮 || btn_confirm | Confirm | 确认 |

| 4 | 已确认状态 | Header 右侧显示 "✓ Confirmed on {date}"，不可点击 || btn_cancel | Cancel | 取消 |

| 5 | 确认成功 | 列表页 Status / Confirmed 列同步更新 |

### 4.5 Markups Tab

### 6.4 Markups Tab 卡片

| Key | English | 中文 |

| # | 验收项 | 通过标准 ||-----|---------|------|

|---|--------|---------|| label_new | New | 新 |

| 1 | 元信息行 | 显示 `{日期} · {创建人}`，格式正确 || label_published_by | Published by {name} | 发布人：{name} |

| 2 | 报审行 | 显示 `Based on {submissionNo}`；`appliedPageNo` 不为 null 时追加 ` · Page {n}` || label_affected_area | Affected Area | 影响区域 |

| 3 | 无 Affected Area | 卡片中不显示影响区域字段 || label_show_more | ...Show more | ...展开全文 |

| 4 | [New] 标签 | 未读时显示，标记已读后隐藏 || label_show_less | Show less | 收起 |

| 5 | Mark as Read | 成功后按钮替换为 "✓ Read"；列表 Markups 列角标减一 || btn_mark_as_read | ✓ Mark as Read | ✓ 标记已读 |

| label_read | ✓ Read | ✓ 已读 |

### 6.5 站内通知| empty_markups | No markups for this drawing. | 该图纸暂无局部更新 |



| # | 验收项 | 通过标准 |### 4.6 站内通知

|---|--------|---------|

| 1 | 通知接收 | Markup 发布后，已分配 SE 铃铛角标 +1 || Key | English | 中文 |

| 2 | 未分配用户 | 未分配该图纸的 SE 不收到通知 ||-----|---------|------|

| 3 | 点击跳转 | 跳转到对应图纸详情页并激活 Markups Tab || notification_title | Notifications | 通知 |

| 4 | Mark all as read | 全部置已读，角标清零 || notification_title_unread | Notifications ({n} unread) | 通知（{n} 条未读） |

| btn_mark_all_read | Mark all as read | 全部标为已读 |

---| link_view_all | View all notifications → | 查看全部通知 → |

| push_markup_title | Drawing Update — {drawingCode} | 图纸更新 — {drawingCode} |

## 7. 相关文档| push_markup_body | {markupTitle}. Please review the latest markup for {drawingCode} {drawingName}. | {markupTitle}。请查阅 {drawingCode} {drawingName} 的最新局部更新。 |

| time_just_now | Just now | 刚刚 |

| 文档 | 说明 || time_minutes_ago | {n} minutes ago | {n} 分钟前 |

|------|------|| time_hours_ago | {n} hours ago | {n} 小时前 |

| [REQ-005-pc.md](../../../requirements/pc/REQ-005-pc.md) | PC 端产品需求（v0.3.5） |

| [REQ-003A-pc.md](../../../requirements/pc/REQ-003A-pc.md) | 图纸管理列表页基础规范（列表行含义与数据模型） |---

| [UI-REQ-004-pc.md](./UI-REQ-004-pc.md) | 局部更新 UI 规范（Markups 卡片组件参考） |

| [FRONTEND-REQ-005-pc.md](../../frontend/pc/FRONTEND-REQ-005-pc.md) | 前端实现规范 |## 5. 响应式与分辨率适配

| [QA-REQ-005-pc.md](../../qa/pc/QA-REQ-005-pc.md) | QA 测试规范 |

| 场景 | 规格 |
|------|------|
| 最小支持分辨率 | 1280px × 720px |
| 图纸列表表格 | 1280px 时 Markups / Confirmed 列可隐藏（列显示开关），优先保留 Drawing Code / Name / Read Status / Last Updated |
| 图纸预览区域 | 高度随视口高度自适应，最小 400px |
| 通知下拉面板 | 固定 360px 宽度，不随分辨率缩放 |
| Markups Tab 卡片 | 全宽自适应，内边距 16px 24px |

---

## 6. 验收条件

### 6.1 菜单与权限

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 菜单可见 | Site Engineer 登录后侧边栏显示 "Drawings" 菜单项 |
| 2 | 权限隔离 | Site Engineer 无法访问 Drawing Management 路由（非本角色菜单） |
| 3 | 双角色兼容 | 同时拥有管理员角色时，两个菜单项均正确显示且功能独立 |

### 6.2 图纸列表页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 数据过滤 | 列表仅显示 status=ACTIVE 且已分配给当前用户的图纸 |
| 2 | 未分配场景 | 无图纸时正确显示空状态图标和文案 |
| 3 | Read Status | 已确认行显示 "✓ Read"（绿），未确认行显示 "Confirm →"（蓝色链接） |
| 4 | Markups 列 | n>0 时橙色 ●n 链接正确显示；n=0 时显示 `—` |
| 5 | 搜索筛选 | Code / Name 模糊搜索实时生效，Category 下拉筛选正常工作 |
| 6 | 行点击 | 点击行（非状态列）进入详情页 Drawing Preview Tab |
| 7 | Confirm → 跳转 | 点击后进入详情页 Drawing Preview Tab |
| 8 | ●n 跳转 | 点击后进入详情页并自动激活 Markups Tab |

### 6.3 Drawing Preview Tab

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 预览加载 | 图纸文件正常加载，PDF 内嵌，图片直接展示 |
| 2 | 缩放平移 | 鼠标滚轮缩放（0.25×–5×），拖拽平移正常工作 |
| 3 | 全屏 | [Fullscreen] 按钮触发全屏，ESC 可退出 |
| 4 | Confirm Reading | 未确认时显示蓝色主按钮 |
| 5 | 确认弹窗 | 点击按钮弹出确认对话框，包含图纸信息副文字 |
| 6 | 确认成功 | 按钮替换为 "✓ Confirmed on {date}"；列表页 Read Status / Confirmed 列同步更新 |
| 7 | 已确认状态 | 重新进入页面仍显示 "✓ Confirmed on {date}"，不回退 |
| 8 | 跨端同步 | 在 APP 端确认后，PC 端刷新同样显示已确认状态 |

### 6.4 Markups Tab

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 仅 ACTIVE | 列表中不出现 MERGED 状态的局部更新 |
| 2 | 排序 | 按 publishTime 降序排列，最新的在上方 |
| 3 | [New] 标签 | 未确认条目显示橙色 [New] 标签；确认后消失 |
| 4 | Mark as Read | 点击后 loading 防重复；成功后按钮变为 "✓ Read" |
| 5 | 状态持久化 | 刷新或重新进入页面，已确认状态正确保留 |
| 6 | 跨端同步 | 在 APP 端已确认的条目，PC 端刷新后同样显示 "✓ Read" |
| 7 | Tab 气泡 | 未读数量准确；全部确认后气泡消失，Tab 标签恢复 "Markups" |
| 8 | 列表 Markups 列同步 | 全部确认后，列表页该行 Markups 列数字减为 0，显示 `—` |

### 6.5 站内通知

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 通知接收 | 局部更新发布后，已分配该图纸的 Site Engineer 铃铛角标 +1 |
| 2 | 权限控制 | 未分配该图纸的用户不收到通知 |
| 3 | 面板内容 | 通知标题、正文、时间格式正确显示 |
| 4 | 点击跳转 | 点击通知条目，跳转到对应图纸详情页并激活 Markups Tab |
| 5 | 已读标记 | 点击条目后该通知标记已读，角标数量减少 |
| 6 | Mark all as read | 点击后所有通知标记已读，角标清零，按钮变为 disabled |
| 7 | 未读高亮 | 未读条目有左侧蓝色圆点和较深背景区分，已读条目无圆点 |

---

## 7. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-005-pc.md](../../../requirements/pc/REQ-005-pc.md) | PC 端产品需求 |
| [REQ-005-shared.md](../../../requirements/shared/REQ-005-shared.md) | 跨端共享规则（通知模型、跨端同步、API） |
| [UI-REQ-003-pc.md](./UI-REQ-003-pc.md) | 图纸列表基础 UI 规范（组件与样式参考） |
| [UI-REQ-004-pc.md](./UI-REQ-004-pc.md) | 局部更新 UI 规范（Markups 卡片组件参考） |
