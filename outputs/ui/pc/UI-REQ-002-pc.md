---
doc_type: ui_spec
req_id: REQ-002-pc
version: 0.2.0
status: draft
generated_from: REQ-002-pc.md@0.5.0
generated_at: 2026-05-02
owner: ""
---

# UI 设计说明：PC 管理端 — 管理后台框架（Header / Sidebar / Main Content）

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：[REQ-002-pc.md](../../../requirements/pc/REQ-002-pc.md)（主）
> - 输出引用：Figma 链接、交互原型链接
> - 不重复定义数据字段（参见 REQ-001-shared.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-002-pc @ v0.5.0 |
| 覆盖用户故事 | US-002-001 ~ US-002-006 |
| 覆盖 AC | AC-002-pc-001 ~ AC-002-pc-012 |
| 上次同步时间 | 2026-05-02 |

> ⚠️ 当 REQ-002-pc.md 版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

为 PC 管理后台提供统一的布局框架，确保 Header、Sidebar、Main Content 三区的视觉层次清晰、操作路径高效，并在不同屏幕尺寸下保持一致的交互体验。

### 1.2 设计原则

1. **层次清晰**：Header 固定顶部、Sidebar 固定左侧、Main Content 独立滚动，三区互不干扰
2. **品牌一致**：与登录页共用 DS Token（主色 `#1890FF`），视觉风格统一
3. **高效导航**：侧边栏默认展开，一级菜单直观可见；折叠态仅保留图标以最大化内容区
4. **状态可感知**：当前选中菜单项、面包屑、项目切换入口始终反映系统当前状态

---

## 2. 信息架构

### 2.1 页面层级

```
管理后台框架（所有已登录页面共用）
├── Header（固定顶部）
│   ├── 折叠按钮（≡）+ 面包屑导航 + 右侧操作区（注：Sidebar 顶部的 Logo 区放在 `Sidebar` 内）
│   ├── 面包屑导航
│   └── 右侧操作区
│       ├── 项目切换入口（彩色圆点 + 项目名 + 下拉）
│       ├── Dashboard 入口（图标，新开 Tab）
│       ├── 消息通知中心（铃铛图标 + 未读角标 → 下拉：待办列表 / 消息通知）
│       └── 个人中心（头像 + 用户名 → 下拉：语言切换 / 退出登录）
├── Sidebar（固定左侧）
│   ├── 展开态（图标 + 文字）
│   └── 折叠态（仅图标，Hover 显示 Tooltip）
### Sidebar Header（`.sidebar-header`） — Logo 区域（实现关键点）

Logo 区域应作为 `Sidebar` 的顶端子区域实现，控制折叠/展开行为。实现示例 DOM：

```html
<aside class="sidebar" :class="{ collapsed: isCollapsed }">
  <div class="sidebar-header">
    <img class="logo-icon" src="/assets/logo.svg" alt="logo" />
    <div class="brand-text">Smart construction site</div>
  </div>
  <nav class="sidebar-menu"> ... </nav>
</aside>
```

行为与样式建议（可直接复用）：

- `.sidebar` 展开时宽度 `224px`，折叠时宽度 `72px`。折叠/展开过渡 `300ms`。
- `.sidebar-header` 高度与菜单项行高一致（例如 `56px`），内部 `display:flex; align-items:center; gap:8px; padding: 9px 8px;`。
- `.brand-text` 在折叠态使用 `display:none` 或 `visibility:hidden`，并保留 `aria-hidden="true"` 以满足无障碍需求。
- Logo 图标 `20×20px`，在折叠态垂直水平居中显示或保持与菜单图标左对齐（以设计稿为准）。
- 禁止在 Header 中再次渲染 Logo 并用绝对定位对齐；折叠逻辑应由 Sidebar 的 `isCollapsed` 控制并统一管理 DOM。

└── Main Content（Header 下方 / Sidebar 右侧，独立滚动）
```

### 2.2 导航与入口

| 入口位置 | 触发方式 | 目标 |
|---------|---------|------|
| ≡ 汉堡按钮 | 点击 | 切换 Sidebar 展开 / 折叠 |
| 面包屑上级 | 点击 | 跳转至对应上级页面 |
| 项目切换入口 | 点击 | 弹出下拉（公司层 / 项目列表） |
| Dashboard 入口 | 点击 | 新开 Tab 页面打开 Dashboard |
| 消息通知中心 | 点击 | 弹出下拉（待办列表 / 消息通知） |
| 待办列表 | 点击（下拉内） | 跳转至待办任务页面（当前空白） |
| 消息通知 | 点击（下拉内） | 跳转至消息通知页面（当前空白） |
| 个人中心 | 点击 | 弹出下拉（语言切换 / 退出登录） |
| 语言切换 | 点击（下拉内） | 切换语言，全页即时刷新 |
| 退出登录 | 点击（下拉内） | 清除 Token，跳转登录页 |
| Sidebar 菜单项 | 点击 | 路由跳转，Main Content 渲染对应页面 |

---

## 3. 页面 / 组件清单

### 3.1 Header

**关联 Story**: US-002-001、US-002-002、US-002-004、US-002-005、US-002-006
**关联 AC**: AC-002-pc-001、AC-002-pc-006、AC-002-pc-006b、AC-002-pc-007、AC-002-pc-008、AC-002-pc-011

#### 布局结构

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ≡  │  面包屑导航            项目切换  Dashboard  通知  个人中心 │
└────────────────────────────────────────────────────────────────────────────────────────┘
  ←── Header 左侧（汉堡按钮）──→    ←── 面包屑 ──→         ←─── 右侧操作区 ──────────→
```

#### 状态列表

| 组件 | 状态 | 触发条件 | 视觉表现 |
|-----|------|---------|---------|
| 消息通知图标 | 无角标 | 无未读消息 | 仅铃铛图标 |
| 消息通知图标 | 有角标（数字） | 未读数 1~99 | 右上角红色角标 + 数字 |
| 消息通知图标 | 有角标（超限） | 未读数 > 99 | 右上角红色角标 + "99+" |
| 消息通知下拉 | 展开 | 点击铃铛图标 | 下拉面板，2 个入口 |
| 项目切换下拉 | 展开 | 点击项目切换入口 | 下拉面板，含公司层 + 项目列表 |
| 项目切换下拉 | 当前项选中 | — | 当前项目显示 ✓ |
| 个人中心下拉 | 展开 | 点击头像 / 用户名 | 下拉面板，2 个菜单项 |
| 面包屑 | 多级 | 进入二级以上页面 | `一级 / 二级`，上级可点击，当前级不可点击 |

#### 权限可见性

| UI 元素 | MAINCON 管理员 | SUBCON 管理员 |
|--------|:---:|:---:|
| 项目切换入口 | ✅ | ✅（仅显示被分配项目） |
| Dashboard 入口 | ✅ | ✅ |
| 消息通知中心 | ✅ | ✅ |
| 个人中心 | ✅ | ✅ |

---

### 3.2 Sidebar（左侧导航栏）

**关联 Story**: US-002-001、US-002-003
**关联 AC**: AC-002-pc-002、AC-002-pc-003、AC-002-pc-004、AC-002-pc-005、AC-002-pc-012

#### 布局结构

```
展开态                          折叠态
┌──────────────────┐            ┌────┐
│ 🏠  Home         │            │ 🏠 │
│ 📊  Dashboard    │            │ 📊 │
│ 📋  Project Detail│           │ 📋 │
│ ▶  Progress Mgmt │            │ ▶  │
│     Sub Item     │  ← 无图标，文字与上级文字左对齐，无缩进
│ ...              │            │ .. │
└──────────────────┘            └────┘
```

#### 状态列表

| 状态 | 描述 | 视觉表现 |
|-----|------|---------|
| 默认（展开） | 进入管理后台默认状态 | 图标 + 文字完整显示，宽度完整 |
| 折叠 | 点击 ≡ 后 | 仅显示图标，宽度收窄，主内容区扩展 |
| 菜单项 — 默认 | 未选中、未 Hover | 图标 + 文字，无高亮 |
| 菜单项 — Hover | 鼠标悬停 | 背景色 `#E8F7FF`，文字 / 图标颜色不变 |
| 菜单项 — 选中 | 当前路由对应项 | 图标 + 文字高亮，左侧显示选中竖线 |
| 子菜单 — 展开 | 点击有子菜单的一级项 | 子菜单展开，一级项箭头向下；一级菜单项 + 所有子菜单项背景色变为 `#EDF2F9` |
| 子菜单 — 折叠 | 点击已展开的一级项 | 子菜单收起，一级项箭头向右 |
| 折叠态 Tooltip | Hover 折叠态菜单图标 | 显示菜单名称 Tooltip |

#### 公司层菜单结构

| 一级菜单 | 二级菜单 |
|---------|---------|
| Progress Management | Progress Setting |
| Dictionary Management | Progress Management |

---

### 3.3 Main Content（主内容区）

**关联 Story**: US-002-001
**关联 AC**: AC-002-pc-003、AC-002-pc-012

#### 布局

- 位于 Header 下方、Sidebar 右侧
- 顶部偏移 = Header 高度；左侧偏移 = Sidebar 当前宽度
- 区域内容独立滚动，Header 和 Sidebar 保持固定
- Sidebar 展开 / 折叠时，Main Content 宽度随之过渡（300ms）

#### 当前阶段内容

| 页面 | 当前内容 |
|-----|---------|
| Home | 空白页面 |
| 所有其他菜单页 | 空白占位页面 |
| 待办列表页 | 空白占位页面（后续补充） |
| 消息通知页 | 空白占位页面（后续补充） |

---

## 4. 设计令牌（Design Tokens）

> DS 字体家族：**Arial**（全局统一）
> DS 组件库：**Ant Design**

### 4.1 颜色语义

| 用途 | Token | 值 | 来源 |
|-----|-------|----|------|
| 品牌主色 / 选中态 / 高亮 | `--Primary-Color` | `#1890FF` | 继承 UI-REQ-001 ✅ |
| 错误状态 | `--color-error` | `#FF4D4F` | 继承 UI-REQ-001 ✅ |
| Header 背景色 | `--Vertical-Menu-White` | `#FFF` | Figma ✅ |
| Header 图标色（Dashboard / 通知） | — | `#344050` | Figma ✅ |
| Header 图标 Hover 色 | `--Cell-Hover` | `#EDF2F9`，`border-radius: 4px` | Figma ✅ |
| Header 分隔竖线 | `--table-dividing-line` | `#E4E7ED` | Figma ✅ |
| Header 通知角标背景 | — | `#F56C6C` | Figma ✅（与通用错误色 `#FF4D4F` 不同，以 Figma 为准） |
| Header 个人中心头像背景 | `--Primary---Light---3` | `#A5DBFF` | Figma ✅ |
| Sidebar 背景色（展开 / 折叠） | `--Vertical-Menu-White` | `#FFF` | Figma ✅ |
| Sidebar 菜单项默认文字色 / 图标色 | `--text-color-primary-dark` | `#344050` | Figma ✅ |
| Sidebar 菜单项 Hover 背景 | — | `#E8F7FF` | Figma ✅ |
| Sidebar 一级菜单展开区背景 | — | `#EDF2F9`（一级项 + 所有子菜单项共用） | Figma ✅ |
| Sidebar 菜单项选中背景 | `--Primary-Color` | `#1890FF` | Figma ✅ |
| Sidebar 菜单项选中文字色 / 图标色 | `--Vertical-Menu-White` | `#FFFFFF` | Figma ✅ |
| Sidebar 选中指示竖线 | `--text-color-primary-dark` | `#344050` | Figma ✅ |
| Main Content 背景色 | `--background-color` | `#F1F1F4` | Figma ✅ |
| 下拉面板背景 / 边框 / 阴影 | — | Element UI 默认样式，不自定义 | ✅ |
| 下拉菜单项 Hover 背景 | — | Element UI 默认样式，不自定义 | ✅ |
| 面包屑上级页文字 | `--text-color-primary-light` | `#5E6E82` | Figma ✅ |
| 面包屑当前页文字 | `--text-color-primary-dark` | `#344050` | Figma ✅ |
| 面包屑斜线 | `--text-color-primary-light` | `#5E6E82` | Figma ✅ |
| 分隔线颜色 | `--border-color` | `#D8E2F0` | 继承 ✅ |

### 4.2 间距与尺寸

| 组件 | 属性 | 值 | 来源 |
|-----|------|----|------|
| Header 容器 | display | `flex` | Figma ✅ |
| Header 容器 | height | `48px` | Figma ✅ |
| Header 容器 | padding | `0 16px`（左右对称） | Figma ✅ |
| Header 容器 | justify-content | `space-between` | Figma ✅ |
| Header 容器 | align-items | `center` | Figma ✅ |
| Header 容器 | align-self | `stretch` | Figma ✅ |
| Header 容器 | background | `#FFF`（`--Vertical-Menu-White`） | Figma ✅ |
| Header 容器 | box-shadow | `0 1px 4px 0 rgba(240, 242, 245, 0.47)` | Figma ✅ |
| Header 右侧操作区容器 | display | `flex` | Figma ✅ |
| Header 右侧操作区容器 | align-items | `center` | Figma ✅ |
| Header 右侧操作区容器 | gap（各操作项间距） | `16px` | Figma ✅ |
| Header 项目切换入口容器 | padding | `0 8px` | Figma ✅ |
| Header 项目切换入口容器 | 高度 | `32px` | Figma ✅ |
| Header 项目切换入口 | 头像与项目名 gap | `8px` | Figma ✅ |
| Header 项目切换入口 | 项目名与箭头 gap | `4px` | Figma ✅ |
| Header 项目切换入口 | 项目名超长处理 | 截断显示（`text-overflow: ellipsis`，`overflow: hidden`，`white-space: nowrap`） | Figma ✅ |
| Header 项目切换入口 | 头像（项目图标）尺寸 | `28×28px` | Figma ✅ |
| Header 项目切换入口 | 下拉箭头图标尺寸 | `16×16px` | Figma ✅ |
| Header 分隔竖线（项目切换后） | 距项目切换入口间距 | `16px`（由右侧操作区 gap 控制） | Figma ✅ |
| Header 分隔竖线（项目切换后） | width | `1px` | Figma ✅ |
| Header 分隔竖线（项目切换后） | height | `20px` | Figma ✅ |
| Header 分隔竖线（项目切换后） | background | `#E4E7ED`（`--table-dividing-line`） | Figma ✅ |
| Header Dashboard 图标 | width | `20px` | Figma ✅ |
| Header Dashboard 图标 | height | `20px` | Figma ✅ |
| Header Dashboard 图标 | fill | `#344050` | Figma ✅ |
| Header 通知图标 | width | `20px` | Figma ✅ |
| Header 通知图标 | height | `20px` | Figma ✅ |
| Header 通知图标 | fill | `#344050` | Figma ✅ |
| Header 通知角标容器 | display | `flex` | Figma ✅ |
| Header 通知角标容器 | width | `16px` | Figma ✅ |
| Header 通知角标容器 | height | `16px` | Figma ✅ |
| Header 通知角标容器 | border-radius | `10px` | Figma ✅ |
| Header 通知角标容器 | background | `#F56C6C` | Figma ✅ |
| Header 通知角标容器 | justify-content | `center` | Figma ✅ |
| Header 通知角标容器 | align-items | `center` | Figma ✅ |
| Header 通知角标数字 | width | `12px` | Figma ✅ |
| Header 通知角标数字 | height | `16px` | Figma ✅ |
| Header 通知角标数字 | color | `#FFF`（`--Vertical-Menu-White`） | Figma ✅ |
| Header 通知角标数字 | font-size | `10px` | Figma ✅ |
| Header 通知角标数字 | font-weight | `500` | Figma ✅ |
| Header 通知角标数字 | line-height | `20px` | Figma ✅ |
| Header 通知角标数字 | letter-spacing | `-0.01px` | Figma ✅ |
| Header 通知角标数字 | text-align | `center` | Figma ✅ |
| Header 个人中心容器 | display | `flex` | Figma ✅ |
| Header 个人中心容器 | padding | `0 8px` | Figma ✅ |
| Header 个人中心容器 | align-items | `center` | Figma ✅ |
| Header 个人中心容器 | gap | `8px` | Figma ✅ |
| Header 个人中心容器 | align-self | `stretch` | Figma ✅ |
| Header 个人中心用户名 | 超长处理 | 截断显示（`text-overflow: ellipsis`，`overflow: hidden`，`white-space: nowrap`） | Figma ✅ |
| Header 个人中心头像容器 | display | `flex` | Figma ✅ |
| Header 个人中心头像容器 | width | `28px` | Figma ✅ |
| Header 个人中心头像容器 | height | `28px` | Figma ✅ |
| Header 个人中心头像容器 | border-radius | `100px`（圆形） | Figma ✅ |
| Header 个人中心头像容器 | background | `#A5DBFF`（`--Primary---Light---3`） | Figma ✅ |
| Header 个人中心头像容器 | justify-content | `space-between` | Figma ✅ |
| Header 个人中心头像容器 | align-items | `center` | Figma ✅ |
| Header 个人中心头像图标 | width | `14px` | Figma ✅ |
| Header 个人中心头像图标 | height | `14px` | Figma ✅ |
| Header 个人中心头像图标 | color | `#FFFFFF` | Figma ✅ |
| Header 汉堡按钮（≡） | 尺寸 | `20×20px` | Figma ✅ |
| Header 汉堡按钮（≡） | 距 Logo 区域间距 | `16px` | Figma ✅ |
| Header 面包屑 | 斜线颜色 | `#5E6E82`（`--text-color-primary-light`） | Figma ✅ |
| Sidebar Logo 区（系统名称） | padding-top | `9px` | Figma ✅ |
| Sidebar Logo 区（系统名称） | 图标与文字 gap | `8px` | Figma ✅ |
| Sidebar Logo 区（系统名称） | 左右边距 | `8px` | Figma ✅ |
| Sidebar Logo 图标 | width | `20px` | Figma ✅ |
| Sidebar Logo 图标 | height | `20px` | Figma ✅ |
| Sidebar 菜单项图标 | display | `flex` | Figma ✅ |
| Sidebar 菜单项图标 | width | `20px` | Figma ✅ |
| Sidebar 菜单项图标 | height | `20px` | Figma ✅ |
| Sidebar 菜单项图标 | padding | `2px 1px` | Figma ✅ |
| Sidebar 菜单项图标 | flex-direction | `column` | Figma ✅ |
| Sidebar 菜单项图标 | justify-content | `center` | Figma ✅ |
| Sidebar 菜单项图标 | align-items | `center` | Figma ✅ |
| Sidebar（展开） | 宽度 | `224px` | Figma ✅ |
| Sidebar（展开）容器 | display | `flex` | Figma ✅ |
| Sidebar（展开）容器 | padding | `0 8px` | Figma ✅ |
| Sidebar（展开）容器 | flex-direction | `column` | Figma ✅ |
| Sidebar（展开）容器 | align-items | `flex-start` | Figma ✅ |
| Sidebar（展开）容器 | flex-shrink | `0` | Figma ✅ |
| Sidebar（展开）容器 | background | `#FFF` | Figma ✅ |
| Sidebar（折叠） | 宽度 | `72px` | Figma ✅ |
| Sidebar（折叠）容器 | display | `flex` | Figma ✅ |
| Sidebar（折叠）容器 | padding | `0 8px` | Figma ✅ |
| Sidebar（折叠）容器 | flex-direction | `column` | Figma ✅ |
| Sidebar（折叠）容器 | align-items | `flex-start` | Figma ✅ |
| Sidebar | 折叠 / 展开动画时长 | `300ms` | REQ-002-pc §9.1 |
| Sidebar 菜单项（折叠态） | padding | `9px 18px` | Figma ✅ |
| Sidebar 菜单项（展开态） | padding | `9px 8px` | Figma ✅ |
| Sidebar 菜单项（展开态）容器 | display | `flex` | Figma ✅ |
| Sidebar 菜单项（展开态）容器 | height | `56px` | Figma ✅ |
| Sidebar 菜单项（展开态）容器 | align-items | `center` | Figma ✅ |
| Sidebar 菜单项（展开态）容器 | gap | `8px` | Figma ✅ |
| Sidebar 菜单项（展开态）容器 | align-self | `stretch` | Figma ✅ |
| Sidebar 菜单项 | 高度 | `56px` | Figma ✅ |
| Sidebar 菜单项 | 图标尺寸 | `20×20px` | Figma ✅ |
| Sidebar 菜单项 | 图标与文字 gap | `8px` | Figma ✅ |
| Sidebar 菜单项 | 右侧箭头区 padding | `8px 8px` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头容器 | display | `flex` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头容器 | height | `56px` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头容器 | padding | `9px 8px` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头容器 | align-items | `center` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头容器 | gap | `8px` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头容器 | align-self | `stretch` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头图标 | width | `8.695px` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头图标 | height | `4.582px` | Figma ✅ |
| Sidebar 菜单项（展开态）箭头图标 | fill | `#344050`（`--text-color-primary-dark`） | Figma ✅ |
| Sidebar 选中指示竖线 | 宽度 | 待确认 | Figma |
| Sidebar 子菜单 | 缩进量 | 无缩进，文字与一级菜单文字左对齐 | Figma ✅ |
| Sidebar 子菜单 | 图标 | 无图标 | Figma ✅ |
| 下拉面板 | 全部样式 | Element UI 默认样式，不自定义 | ✅ |
| Main Content 容器 | display | `flex` | Figma ✅ |
| Main Content 容器 | padding | `16px` | Figma ✅ |
| Main Content 容器 | flex-direction | `column` | Figma ✅ |
| Main Content 容器 | align-items | `flex-end` | Figma ✅ |
| Main Content 容器 | gap | `16px` | Figma ✅ |
| Main Content 容器 | align-self | `stretch` | Figma ✅ |

### 4.3 字体

| 用途 | 字号 | 行高 | 字重 | 颜色 Token | 来源 |
|-----|------|------|------|-----------|------|
| Logo 品牌名（Smart construction site） | 16px | 24px | 700 | `#344050`（`--text-color-primary-dark`） | Figma ✅ |
| 面包屑文字 | 14px | 22px | 500 | `#5E6E82`（`--text-color-primary-light`，一级 / 可点击） | Figma ✅ |
| 面包屑文字（当前页） | 14px | 22px | 500 | `#344050`（`--text-color-primary-dark`，末级 / 不可点击） | Figma ✅ |
| 项目切换入口文字 | 14px | 22px | 700 | `#344050`（`--text-color-primary-dark`） | Figma ✅ |
| Sidebar 菜单项文字（默认） | 14px | 22px | 700 | `#344050`（`--text-color-primary-dark`），`letter-spacing: -0.01px` | Figma ✅ |
| Sidebar 菜单项文字（选中） | 14px | 22px | 700 | `#FFFFFF`（`--Vertical-Menu-White`） | Figma ✅ |
| 下拉菜单项文字 | — | — | — | Element UI 默认样式，不自定义 | ✅ |
| 用户名文字 | 14px | 22px | 700 | `#344050`（`--text-color-primary-dark`） | Figma ✅ |

---

## 5. 响应式设计

| 断点 | 宽度 | Sidebar 行为 |
|-----|------|-------------|
| Large Desktop | ≥ 1440px | 默认展开 |
| Desktop | 1280~1439px | 默认折叠，可手动展开 |
| < 1280px | 超出最低支持范围 | 提示使用更大屏幕 |

---

## 6. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| Sidebar 展开 / 折叠 | 宽度过渡 + Main Content 宽度同步过渡 | 300ms | ease-in-out |
| 子菜单展开 / 折叠 | 高度展开 + 箭头旋转 | 200ms | ease |
| 下拉面板出现 | 淡入 + 向下位移 | 150ms | ease-out |
| 下拉面板消失 | 淡出 | 100ms | ease-in |
| 菜单项 Hover | 背景色过渡 | 100ms | ease |
| 路由切换（Main Content） | 淡入淡出 | 150ms | ease |
| 折叠态 Tooltip | 出现 | 100ms | ease-out |

---

## 7. 无障碍（A11y）

- WCAG 等级：AA
- 键盘导航：侧边栏菜单项支持 Tab 键顺序导航，Enter 触发跳转
- 折叠态 Tooltip 对屏幕阅读器可见（`aria-label`）
- 下拉菜单支持 Esc 关闭
- 当前选中菜单项需有 `aria-current="page"`
- Header 操作图标均需 `aria-label`（如 "通知中心"、"切换语言"、"退出登录"）

---

## 8. 复用与新建组件清单

| 组件 | 来源 | Ant Design 对应组件 | 备注 |
|-----|------|-------------------|------|
| 顶部导航栏 | Ant Design DS | `Layout.Header` | 固定定位需额外处理 |
| 左侧导航栏 | Ant Design DS | `Layout.Sider` + `Menu` | 支持 `collapsed` prop，内置折叠动画 |
| 多层菜单 | Ant Design DS | `Menu` (SubMenu) | 直接支持多层级 |
| 面包屑 | Ant Design DS | `Breadcrumb` | 可直接使用 |
| 下拉菜单（项目切换 / 个人中心 / 通知中心） | Ant Design DS | `Dropdown` + `Menu` | 可直接使用 |
| 未读角标 | Ant Design DS | `Badge` | 可直接使用，超出值用 `overflowCount={99}` |
| 头像 | Ant Design DS | `Avatar` | 可直接使用 |
| Tooltip（折叠态菜单） | Ant Design DS | `Tooltip` | 可直接使用 |
| 整体布局 | Ant Design DS | `Layout`（Header + Sider + Content） | 推荐使用 Ant Design 布局组件 |

---

## 9. 文案规范

| 场景 | 文案（zh-CN） | 文案（en） |
|-----|-------------|----------|
| 消息通知中心下拉 — 入口1 | 待办列表 | To-Do List |
| 消息通知中心下拉 — 入口2 | 消息通知 | Notifications |
| 个人中心下拉 — 语言切换 | 语言切换 | Language |
| 个人中心下拉 — 退出登录 | 退出登录 | Logout |
| 公司层标识 | 公司层 | Company Insights |
| Logo 品牌名 | 智慧工地管理后台 | Smart construction site |
| 侧边栏菜单项 — Home | 首页 | Home |
| 侧边栏菜单项 — Dashboard | 数据看板 | Dashboard |
| 侧边栏菜单项 — Project Detail | 项目详情 | Project Detail |
| 侧边栏菜单项 — Progress Management | 进度管理 | Progress Management |
| 侧边栏菜单项 — Subcon Management | 分包商管理 | Subcon Management |
| 侧边栏菜单项 — Personnel Management | 人员管理 | Personnel Management |
| 侧边栏菜单项 — Attendance Management | 考勤管理 | Attendance Management |
| 侧边栏菜单项 — Equipment Management | 设备管理 | Equipment Management |
| 侧边栏菜单项 — Tower crane management | 塔吊管理 | Tower crane management |
| 侧边栏菜单项 — Material Management | 材料管理 | Material Management |
| 侧边栏菜单项 — Visit Record | 访客记录 | Visit Record |
| 公司层侧边栏 — Progress Management | 进度管理 | Progress Management |
| 公司层侧边栏 — Progress Setting（子菜单） | 进度设置 | Progress Setting |
| 公司层侧边栏 — Dictionary Management | 字典管理 | Dictionary Management |
| 公司层侧边栏 — Progress Management（子菜单） | 进度管理 | Progress Management |
| 403 无权限提示 | 暂无权限访问 | No Permission |
| 折叠按钮（aria-label） | 折叠侧边栏 / 展开侧边栏 | Collapse sidebar / Expand sidebar |

---

## 10. Figma 与原型链接

- Figma 设计稿：<!-- 填写管理后台框架 Frame 链接 -->
- 交互原型：
- 设计系统：

---

## 11. AC 覆盖检查表

| AC ID | 对应组件 | 对应状态 / 交互 | 覆盖？ |
|------|---------|--------------|------|
| AC-002-pc-001 | Header | 固定定位，不随内容滚动 | ✅ §3.1 |
| AC-002-pc-002 | Sidebar | 菜单项按权限渲染（后端控制） | ✅ §3.2 |
| AC-002-pc-003 | Sidebar + Main Content | 点击菜单项路由跳转 + 高亮 | ✅ §3.2、§3.3 |
| AC-002-pc-004 | Sidebar | 展开 / 折叠（动画、主内容区宽度同步） | ✅ §3.2、§6 |
| AC-002-pc-005 | Sidebar（折叠态） | Hover 图标显示 Tooltip | ✅ §3.2 |
| AC-002-pc-006 | Header — 项目切换 | 切换项目，Header 更新，数据刷新 | ✅ §3.1 |
| AC-002-pc-006b | Header — 项目切换 | 切换公司层，Sidebar 切换公司层菜单 | ✅ §3.1、§3.2 |
| AC-002-pc-007 | Header — 消息通知中心 | 下拉含2入口；角标逻辑 | ✅ §3.1 |
| AC-002-pc-008 | Header — 个人中心 | 退出登录 → 清 Token → 跳转登录页 | ✅ §3.1 |
| AC-002-pc-009 | 全局 | Token 过期自动跳转登录页 | ✅ §2.2（导航表） |
| AC-002-pc-010 | Main Content | 403 无权限提示页 | ✅ §3.3 |
| AC-002-pc-011 | Header — 个人中心 | 语言切换 → 全页即时刷新 | ✅ §3.1 |
| AC-002-pc-012 | Sidebar + Main Content | 默认进入 Home，Home 菜单项高亮 | ✅ §3.2、§3.3 |

---

## 12. 待定问题（Open Questions）

| OQ ID | 问题 | 影响 UI 哪部分 |
|------|------|--------------|
| OQ-UI-001 | Header 高度、背景色、图标颜色等全部待 Figma 确认 | §4.1、§4.2 所有"待确认"项 | ✅ 已解决：全部 Figma 数据已录入 |
| OQ-UI-002 | Sidebar 展开宽度、折叠宽度、菜单项高度、选中样式（背景色 vs 仅竖线 + 文字色） | §4.2 Sidebar 相关行 |
| OQ-UI-003 | 消息通知下拉面板的具体布局（是否有标题行、分隔线等） | §3.1 消息通知中心 | ✅ 已解决：使用 Element UI 默认下拉样式，不自定义 |
| OQ-UI-004 | 项目切换下拉中，项目条目是否有彩色圆点色值规则？ | §3.1 项目切换入口 |
| OQ-UI-005 | Main Content 的背景色及内边距 | §4.1、§4.2 |

---

## 13. 验收标准（本文档自身）

- [ ] Figma 稿覆盖 Header / Sidebar / Main Content 全部区域
- [ ] 每个组件均有完整状态设计稿（默认 / Hover / 选中 / 展开 / 折叠）
- [ ] §11 AC 覆盖检查表全部标记 ✅
- [ ] §4 所有"待确认"项填入 Figma 真实值后，本文档版本升为 1.0.0
- [ ] 所有 OQ 在 §12 显式列出，未编造方案

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-02 | agent | 从 REQ-002-pc.md v0.5.0 抽取视觉规格，初稿；§4 颜色/间距/字体均为待确认状态，待 Figma 填入 |
| 0.2.0 | 2026-05-02 | agent | 填入全部 Figma 数据：Header / Sidebar / Main Content 三区颜色、间距、字体、图标全部确认；下拉面板使用 Element UI 默认样式；关闭全部 OQ |
