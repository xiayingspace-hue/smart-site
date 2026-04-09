# UI 设计规范 — PC 管理端登录和管理后台框架

> **来源需求**: [REQ-001-pc](../../../requirements/pc/REQ-001-pc.md) + [REQ-001-shared](../../../requirements/shared/REQ-001-shared.md)
> **产品**: SMART SITE SYSTEM
> **品牌标识**: SMART CONSTRUCTION · BACK OFFICE
> **平台**: PC 管理端（桌面浏览器，Vue 2 + Element UI）
> **生成日期**: 2026-03-24

---

## 1. 设计目标

- 为系统管理员和项目经理提供一个专业、可信赖的 Web 管理后台登录入口
- 管理后台框架（Header + Sidebar + Main Content）结构清晰、层次分明
- 视觉风格简洁商务，以蓝色为主品牌色，传递科技与可靠的产品形象
- 支持多语言（中文 / English），默认英文界面
- 适配 1280px 及以上分辨率，推荐 1920×1080px
- 基于 Element UI 组件库，确保一致的交互模式和视觉风格

---

## 2. 页面清单与用户流程

### 2.1 页面清单

| 序号 | 页面 | 说明 |
|------|------|------|
| 1 | 登录页 | 全屏背景 + 居中登录卡片（MAINCON / SUBCON Tab） |
| 2 | 无效 URL 错误页 | URL 中无有效 tenantCode 时显示 |
| 3 | 管理后台框架 | Header + Sidebar + Main Content 三栏布局 |
| 4 | Home 首页 | 默认落地页（当前空白占位） |

### 2.2 核心用户流程

```
登录流程：

 ┌─────────────────┐      URL 解析       ┌─────────────┐
 │  用户访问 URL    │ ──────────────────→ │ 解析 tenant │
 │ /mcc/login      │                     │ Code        │
 └─────────────────┘                     └──────┬──────┘
                                                │
                                    ┌───────────┴───────────┐
                                    ↓                       ↓
                             tenantCode 有效          tenantCode 无效
                                    │                       │
                                    ↓                       ↓
                            ┌──────────────┐       ┌──────────────┐
                            │  显示登录页   │       │ 显示错误页面  │
                            │ MAINCON/SUBCON│       │ Invalid URL  │
                            └──────┬───────┘       └──────────────┘
                                   │
                            输入账号+密码
                            点击 Login
                                   │
                            ┌──────┴──────┐
                            ↓             ↓
                       验证通过       验证失败
                            │             │
                            ↓             ↓
                     ┌────────────┐  ┌──────────┐
                     │ 调用登录API │  │ 红色边框  │
                     └─────┬──────┘  │ + required│
                           │         └──────────┘
                    ┌──────┴──────┐
                    ↓             ↓
               登录成功       登录失败
                    │             │
                    ↓             ↓
           ┌──────────────┐  ┌──────────────┐
           │ 保存 Token    │  │ 显示错误提示  │
           │ 跳转管理后台  │  └──────────────┘
           └──────────────┘
```

```
管理后台导航流程：

 ┌──────────────┐    点击菜单    ┌──────────────┐   URL 变化   ┌──────────────┐
 │  Sidebar     │ ────────────→ │ 路由切换      │ ──────────→ │ 面包屑更新    │
 │  一级菜单    │               │ 主内容区渲染  │             │ 菜单选中态    │
 └──────────────┘               └──────────────┘             └──────────────┘
```

---

## 3. 页面设计规范

### 3.1 登录页

#### 4.1.1 页面结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                        (全屏城市夜景背景图)                                   │
│                                                                             │
│                    ┌────────────── 400-440px ──────────────┐                │
│                    │                                       │                │
│                    │  SMART CONSTRUCTION                   │  ← 20-22px 粗体│
│                    │  BACK OFFICE                          │  ← 14-16px 粗体│
│                    │                           24px ↕      │                │
│                    │  Account Login Page           文A     │  ← 14px 600   │
│                    │                           24px ↕      │                │
│                    │  MAINCON     SUBCON                   │  ← Tab 切换   │
│                    │  ══════                               │  ← 2px 下划线 │
│                    │                           28px ↕      │                │
│                    │  ┌───────────────────────────────┐    │                │
│                    │  │ Please fill in account         │    │  ← 44-48px   │
│                    │  └───────────────────────────────┘    │                │
│                    │                           20px ↕      │                │
│                    │  ┌───────────────────────────────┐    │                │
│                    │  │ Please fill in password     👁 │    │  ← 44-48px   │
│                    │  └───────────────────────────────┘    │                │
│                    │                           16px ↕      │                │
│                    │  ☑ Remember the password              │  ← #4A90D9   │
│                    │                           24px ↕      │                │
│                    │  ┌───────────────────────────────┐    │                │
│                    │  │            Login               │    │  ← 44-48px   │
│                    │  └───────────────────────────────┘    │                │
│                    │                                       │                │
│                    └───────────────────────────────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 背景

| 属性 | 规格 |
|------|------|
| 背景图 | 城市天际线夜景（新加坡风格），全屏铺满 |
| 图片处理 | 轻微暗色滤镜叠加（CSS `filter` 或叠加半透明黑色层） |
| 填充方式 | `background-size: cover; background-position: center;` |
| 备用背景 | 深蓝灰色渐变 `linear-gradient(135deg, #1a2332 0%, #2c3e50 100%)` |
| 页面最小高度 | 100vh |

#### 4.1.3 登录卡片

| 属性 | 规格 |
|------|------|
| 定位 | 页面水平垂直居中（flex + center 或 absolute + transform） |
| 宽度 | 400-440px |
| 背景 | `--color-bg-component`（#FFFFFF） |
| 圆角 | `--radius-lg`（16-20px） |
| 阴影 | `--shadow-card`（0 8px 32px rgba(0,0,0,0.12)） |
| 内边距 | 上 40px，左右 40px，下 36px |

#### 4.1.4 品牌标识区

| 元素 | 规格 |
|------|------|
| 主标题 "SMART CONSTRUCTION" | `--font-size-xl`（20-22px），`--font-weight-bold`（700），`--color-text-primary`（#212121），大写 |
| 副标题 "BACK OFFICE" | `--font-size-subtitle`（14-16px），`--font-weight-bold`（700），`--color-text-regular`（#333333），大写 |
| 主副标题行间距 | `--spacing-xs`（4px） |
| 品牌区与下方间距 | `--spacing-xxl`（24px） |

#### 4.1.5 Account Login Page 行

| 元素 | 规格 |
|------|------|
| 左侧文字 | "Account Login Page"，`--font-size-md`（14px），`--font-weight-semibold`（600），`--color-text-regular` |
| 右侧图标 | 语言切换图标（文A），20px，`--color-text-secondary`（#666666） |
| 布局 | Flexbox，`justify-content: space-between` |
| 图标 hover | 色变 `--color-text-regular`（#333333），cursor: pointer |
| 下边距 | `--spacing-xxl`（24px） |

**语言切换下拉菜单**：

| 属性 | 规格 |
|------|------|
| 触发 | 点击语言图标 |
| 菜单宽度 | 120px |
| 菜单项 | "English" / "中文（简体）" |
| 菜单项高度 | 36px |
| 选中态 | 文字变 `--color-primary`（#4A90D9），右侧 ✓ 图标 |
| 圆角 | `--radius-sm`（4px） |
| 阴影 | `--shadow-dropdown` |

#### 4.1.6 角色 Tab 切换

| 属性 | 规格 |
|------|------|
| Tab 项 | "MAINCON" / "SUBCON"，大写字母 |
| Tab 间距 | 32px |
| 选中 Tab 文字 | `--color-primary`（#4A90D9），`--font-size-md`（14px），`--font-weight-semibold`（600） |
| 选中下划线 | 2px 实线 `--color-primary`，宽度 = 文字宽度，底部紧贴 |
| 未选中 Tab | `--color-text-disabled`（#999999），`--font-size-md`，`--font-weight-regular`（400） |
| Tab 栏底部线 | 1px `--color-border-light`（#E8E8E8），全宽 |
| 下划线切换动画 | `--transition-normal`（300ms ease-in-out） |
| 下边距 | 28px |

#### 4.1.7 输入框

| 属性 | 规格 |
|------|------|
| 宽度 | 100%（卡片内宽） |
| 高度 | 44-48px |
| 背景 | `--color-bg-component`（#FFFFFF） |
| 边框（默认） | 1px `--color-border`（#D9D9D9） |
| 边框（聚焦） | 1px `--color-primary`（#4A90D9） |
| 边框（错误） | 1px `--color-error`（#FF4D4F） |
| 圆角 | `--radius-md`（6-8px） |
| 内边距 | 水平 12px |
| 文字 | `--font-size-md`（14px），`--color-text-regular`（#333333） |
| Placeholder | `--font-size-md`，`--color-text-placeholder`（#BDBDBD） |
| 字段间距 | `--spacing-xl`（20px） |
| 聚焦过渡 | `--transition-fast`（200ms） |

**密码输入框**：

| 属性 | 规格 |
|------|------|
| 右侧图标 | 👁 眼睛图标（显示/隐藏密码切换），20px |
| 图标色（默认） | `--color-text-placeholder`（#BDBDBD） |
| 图标色（hover） | `--color-text-secondary`（#666666） |
| 默认状态 | 掩码显示（type="password"） |
| cursor | pointer |

**错误状态**：

| 属性 | 规格 |
|------|------|
| 边框 | 1px `--color-error`（#FF4D4F） |
| 错误提示文字 | "required"，`--font-size-xs`（12px），`--color-error`（#FF4D4F） |
| 提示位置 | 输入框下方 `--spacing-xs`（4px） |
| 触发 | 点击 Login 时字段为空 |

#### 4.1.8 Remember the password

| 属性 | 规格 |
|------|------|
| 位置 | 密码框下方 `--spacing-lg`（16px） |
| Checkbox 尺寸 | 16×16px |
| 选中态 | 背景 `--color-primary`（#4A90D9），白色 ✓ 图标 |
| 未选中态 | 边框 1px `--color-border`（#D9D9D9），白色背景 |
| 文字 | "Remember the password"，`--font-size-md`（14px），`--color-primary`（#4A90D9） |
| Checkbox 与文字间距 | `--spacing-sm`（8px） |
| 默认 | 选中（checked） |

#### 4.1.9 Login 按钮

| 属性 | 规格 |
|------|------|
| 文字 | "Login"，白色 #FFFFFF，`--font-size-lg`（16px），`--font-weight-bold` |
| 宽度 | 100%（卡片内宽） |
| 高度 | 44-48px |
| 圆角 | `--radius-md`（6-8px） |
| 上边距 | `--spacing-xxl`（24px） |
| cursor | pointer |

| 状态 | 背景 | 文字 | 说明 |
|------|------|------|------|
| 默认 | `--color-primary`（#4A90D9） | 白色 "Login" | |
| Hover | `--color-primary-hover`（#357ABD） | 白色 | |
| 按下 | `--color-primary-active`（#2A6FA8） | 白色 | |
| Loading | `--color-primary`（#4A90D9） | 白色旋转加载图标 | 禁止点击 |
| 禁用 | `--color-primary` opacity 0.6 | 白色 | cursor: not-allowed |

#### 4.1.10 无效 URL 错误页

| 属性 | 规格 |
|------|------|
| 背景 | 同登录页背景 |
| 卡片 | 同登录卡片样式 |
| 图标 | ⚠ 警告图标，48px，`--color-warning`（#FAAD14） |
| 主文字 | "Invalid access URL"，`--font-size-lg`（16px），`--font-weight-semibold`，`--color-text-primary` |
| 副文字 | "Please contact your administrator"，`--font-size-md`（14px），`--color-text-secondary` |
| 主副间距 | `--spacing-sm`（8px） |
| 图标与文字间距 | `--spacing-lg`（16px） |

---

### 3.2 管理后台框架

#### 4.2.1 整体布局

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              Header（56-64px，fixed top）                              │
│  ┌─────────────┬───┬──────────────────────┬──────────────────────────────────────┐   │
│  │ 🏗 Logo+Name│ ≡ │  面包屑              │  🟠 Project ▼   📊  🔔⁹⁹⁺  👤 Name │   │
│  └─────────────┴───┴──────────────────────┴──────────────────────────────────────┘   │
├───────────────────────┬──────────────────────────────────────────────────────────────┤
│                       │                                                              │
│   Sidebar             │   Main Content                                               │
│   220-240px           │   flex: 1                                                    │
│   (折叠时 64px)        │   背景 #F5F7FA                                               │
│                       │   内边距 16-24px                                              │
│   fixed left          │                                                              │
│   100vh - Header      │   ┌──────────────────────────────────────────────────────┐   │
│                       │   │                                                      │   │
│   ┌───────────────┐   │   │           页面内容（空白占位 / 功能页面）                │   │
│   │ 📺 Home    ◀  │   │   │                                                      │   │
│   │ 🕐 Dashboard  │   │   │                                                      │   │
│   │ ⚙ Project..   │   │   │                                                      │   │
│   │ 📋 Progress.. │   │   │                                                      │   │
│   │ 🤝 Subcon..   │   │   └──────────────────────────────────────────────────────┘   │
│   │ 👥 Personnel..│   │                                                              │
│   │ 📅 Attend..   │   │                                                              │
│   │ 🔧 Equip..    │   │                                                              │
│   │ 🏗 Tower..    │   │                                                              │
│   │ 📦 Material.. │   │                                                              │
│   │ 📊 Progress.. │   │                                                              │
│   │ 🕐 Visit..    │   │                                                              │
│   └───────────────┘   │                                                              │
│                       │                                                              │
└───────────────────────┴──────────────────────────────────────────────────────────────┘
```

#### 4.2.2 Header 顶部导航栏

**整体属性**：

| 属性 | 规格 |
|------|------|
| 高度 | 56-64px |
| 背景 | `--color-bg-component`（#FFFFFF） |
| 底部边框 | 1px `--color-border-light`（#E8E8E8） |
| 阴影 | `--shadow-header` |
| 内边距 | 左右 16-24px |
| 定位 | `position: fixed; top: 0; left: 0; right: 0;` |
| z-index | 1000 |
| 布局 | Flexbox，`align-items: center;` |

**Logo 区域**：

| 属性 | 规格 |
|------|------|
| 宽度 | 与 Sidebar 等宽（展开 220-240px，折叠 64px） |
| Logo 图标 | 建筑/工地图标，高度 28-32px |
| 品牌名称 | "Smart construction site"，14-16px，`--color-text-regular`（#333333），`--font-weight-bold` |
| 间距 | Logo 与文字 `--spacing-sm`（8-12px） |
| 折叠时 | 仅显示 Logo 图标，过渡动画 `--transition-normal` |

**侧边栏折叠按钮**：

| 属性 | 规格 |
|------|------|
| 图标 | ≡ 汉堡菜单图标，20px，`--color-text-secondary`（#666666） |
| Hover | 色变 `--color-text-regular`（#333333） |
| cursor | pointer |
| 功能 | 点击切换 Sidebar 展开/折叠 |

**面包屑导航**：

| 属性 | 规格 |
|------|------|
| 位置 | 折叠按钮右侧 `--spacing-lg`（16px） |
| 当前页文字 | `--font-size-md`（14px），`--color-text-regular`（#333333） |
| 上级页文字 | `--font-size-md`，`--color-text-disabled`（#999999） |
| 上级页 hover | 色变 `--color-primary`（#4A90D9），cursor: pointer |
| 分隔符 | " / "，`--color-text-placeholder`（#CCCCCC） |
| 动态更新 | 根据当前路由自动生成 |

**项目选择器**：

| 属性 | 规格 |
|------|------|
| 彩色圆点 | 8px 直径，项目标识色，border-radius: 50% |
| 项目名称 | `--font-size-md`（14px），`--color-text-regular`，最大宽度 200px，超出 `text-overflow: ellipsis` |
| 下拉箭头 | ▼，10px，`--color-text-disabled` |
| Hover 背景 | `--color-hover-bg`（#F5F7FA），`--radius-sm`（4px），内边距 6px 12px |
| 下拉菜单宽度 | 260px |
| 下拉菜单最大高度 | 400px，可滚动 |
| 菜单项 | 彩色圆点 + 项目名称，高度 40px，内边距 0 16px |
| 当前选中 | 右侧显示 ✓ 图标，`--color-primary` |
| 菜单项 hover | 背景 `--color-hover-bg` |
| 阴影 | `--shadow-dropdown` |

**通知铃铛**：

| 属性 | 规格 |
|------|------|
| 图标 | 🔔 铃铛，20px，`--color-text-secondary`（#666666） |
| Hover | 色变 `--color-text-regular` |
| 角标位置 | 铃铛右上角偏移（top: -4px, right: -8px） |
| 角标背景 | `--color-badge`（#FF4D4F） |
| 角标文字 | 白色，`--font-size-xs`（10-11px），`--font-weight-bold` |
| 角标形状 | border-radius: 8px，最小宽度 16px，高度 16px，内边距 0 4px |
| 超过 99 | 显示 "99+" |
| 无未读 | 不显示角标 |

**数据统计图标**：

| 属性 | 规格 |
|------|------|
| 图标 | 📊 柱状图图标，20px，`--color-text-secondary` |
| Hover | 色变 `--color-text-regular` |
| cursor | pointer |

**用户信息**：

| 属性 | 规格 |
|------|------|
| 头像 | 32px 圆形（border-radius: 50%），`object-fit: cover` |
| 用户名 | `--font-size-md`（14px），`--color-text-regular`，头像右侧 `--spacing-sm`（8px） |
| Hover 背景 | `--color-hover-bg`，`--radius-sm` |
| cursor | pointer |

**用户下拉菜单**：

| 属性 | 规格 |
|------|------|
| 宽度 | 160px |
| 圆角 | `--radius-sm`（4px） |
| 阴影 | `--shadow-dropdown` |
| 菜单项高度 | 40px |
| 菜单项图标 | 16px，`--color-text-secondary`，右间距 `--spacing-sm`（8px） |
| 菜单项文字 | `--font-size-md`，`--color-text-regular` |
| 菜单项 hover | 背景 `--color-hover-bg` |
| 分隔线 | "退出登录" 上方 1px `--color-border-lighter` |
| 退出登录文字色 | `--color-error`（#FF4D4F） |

**右侧工具区各组件间距**: `--spacing-lg` ~ `--spacing-xxl`（16-24px）

#### 4.2.3 Sidebar 侧边栏

**整体属性**：

| 属性 | 规格 |
|------|------|
| 宽度（展开） | 220-240px |
| 宽度（折叠） | 64px |
| 高度 | `calc(100vh - Header高度)` |
| 背景 | `--color-bg-component`（#FFFFFF） |
| 右侧边框 | 1px `--color-border-light`（#E8E8E8） |
| 定位 | `position: fixed; left: 0; top: Header高度;` |
| z-index | 900 |
| 溢出 | `overflow-y: auto;`（自定义滚动条或隐藏默认滚动条） |
| 宽度过渡 | `--transition-normal`（300ms ease-in-out） |

**一级菜单项**：

| 属性 | 规格 |
|------|------|
| 高度 | 48-52px |
| 左内边距 | `--spacing-xl`（20-24px） |
| 图标 | 功能图标，18-20px |
| 图标色（默认） | `--color-text-secondary`（#666666） |
| 图标色（选中） | `--color-primary`（#4A90D9） |
| 文字 | `--font-size-md`（14px） |
| 文字色（默认） | `--color-text-regular`（#333333） |
| 文字色（选中） | `--color-primary`（#4A90D9） |
| 图标与文字间距 | `--spacing-md`（12px） |
| Hover 背景 | `--color-hover-bg`（#F5F7FA） |
| 选中指示器 | 左侧 4px 宽 `--color-primary` 色条 |
| hover/选中过渡 | `--transition-fast`（200ms） |

**折叠状态**：

| 属性 | 规格 |
|------|------|
| 图标 | 居中显示，隐藏文字 |
| Tooltip | Hover 时右侧弹出 Tooltip 显示菜单名称 |
| Tooltip 样式 | 深灰背景 #333333，白色文字，`--radius-sm`，内边距 4px 8px，`--font-size-xs` |

**二级菜单项（预定义，当前暂无）**：

| 属性 | 规格 |
|------|------|
| 高度 | 44-48px |
| 左内边距 | 48-52px |
| 文字 | `--font-size-md`，`--color-text-secondary`（默认） |
| 选中态 | 左侧 4px `--color-primary` 色条，文字变 `--color-primary`，背景 `--color-primary-bg`（#EBF5FF） |
| Hover 背景 | `--color-hover-bg` |
| 展开动画 | 父菜单箭头旋转 180°，子列表高度动画 `--transition-normal` |

**菜单列表**：

| 序号 | 菜单名称 | 图标描述 |
|------|---------|---------|
| 1 | Home | 显示器图标 |
| 2 | Dashboard | 时钟图标 |
| 3 | Project Detail | 齿轮图标 |
| 4 | Progress Management | 剪贴板图标 |
| 5 | Subcon Management | 握手图标 |
| 6 | Personnel Management | 人员图标 |
| 7 | Attendance Management | 日历图标 |
| 8 | Equipment Management | 扳手图标 |
| 9 | Tower crane management | 塔吊图标 |
| 10 | Material Management | 包裹图标 |
| 11 | Progress Management | 图表图标 |
| 12 | Visit Record | 时钟图标 |

> 菜单根据用户权限动态渲染，当前阶段仅一级菜单。

#### 4.2.4 Main Content 主内容区

**整体属性**：

| 属性 | 规格 |
|------|------|
| 左侧偏移 | 等于 Sidebar 宽度（展开 220-240px，折叠 64px） |
| 顶部偏移 | 等于 Header 高度（56-64px） |
| 背景 | `--color-bg-page`（#F5F7FA） |
| 内边距 | `--spacing-lg` ~ `--spacing-xxl`（16-24px） |
| 最小高度 | `calc(100vh - Header高度)` |
| 宽度过渡 | `--transition-normal`（Sidebar 折叠时自适应） |

**内容卡片（通用容器）**：

| 属性 | 规格 |
|------|------|
| 背景 | `--color-bg-component`（#FFFFFF） |
| 圆角 | `--radius-sm`（4-8px） |
| 阴影 | `--shadow-content`（0 1px 4px rgba(0,0,0,0.04)） |
| 内边距 | `--spacing-xxl`（24px） |
| 卡片间距 | `--spacing-lg`（16px） |

**页面内 Tab 栏（通用组件）**：

| 属性 | 规格 |
|------|------|
| 位置 | 内容卡片顶部 |
| Tab 文字 | `--font-size-md`（14-15px） |
| 选中 Tab | `--color-primary`（#4A90D9），`--font-weight-semibold` |
| 未选中 Tab | `--color-text-secondary`（#666666） |
| 选中下划线 | 2-3px `--color-primary`，宽度 = 文字宽度 |
| Tab 间距 | 32-40px |
| 底部分隔线 | 1px `--color-border-light`（#E8E8E8） |
| 左内边距 | `--spacing-xxl`（24px） |
| 上下内边距 | `--spacing-lg`（16px） |
| 下划线动画 | `--transition-normal` |

**Home 首页（空白占位）**：

| 属性 | 规格 |
|------|------|
| 内容 | 当前为空白，后续根据业务需求补充 |
| 可选占位 | 居中显示"欢迎使用"文字或品牌 Logo 水印（可选） |

---

## 4. 多语言文案

### 4.1 登录页

| Key | English | 中文 |
|-----|---------|------|
| brand_title | SMART CONSTRUCTION | SMART CONSTRUCTION |
| brand_subtitle | BACK OFFICE | BACK OFFICE |
| login_page_title | Account Login Page | 账号登录 |
| tab_maincon | MAINCON | 总承包商 |
| tab_subcon | SUBCON | 分包商 |
| placeholder_account | Please fill in account | 请输入账号 |
| placeholder_password | Please fill in password | 请输入密码 |
| remember_password | Remember the password | 记住密码 |
| btn_login | Login | 登录 |
| error_required | required | 必填 |
| error_invalid_url | Invalid access URL | 访问链接无效 |
| error_contact_admin | Please contact your administrator | 请联系管理员 |

### 4.2 管理后台框架

| Key | English | 中文 |
|-----|---------|------|
| brand_name | Smart construction site | 智慧工地 |
| menu_home | Home | 首页 |
| menu_dashboard | Dashboard | 数据看板 |
| menu_project_detail | Project Detail | 项目详情 |
| menu_progress | Progress Management | 进度管理 |
| menu_subcon | Subcon Management | 分包商管理 |
| menu_personnel | Personnel Management | 人员管理 |
| menu_attendance | Attendance Management | 考勤管理 |
| menu_equipment | Equipment Management | 设备管理 |
| menu_tower_crane | Tower crane management | 塔吊管理 |
| menu_material | Material Management | 材料管理 |
| menu_visit_record | Visit Record | 访客记录 |
| user_profile | Profile | 个人中心 |
| user_change_password | Change Password | 修改密码 |
| user_logout | Logout | 退出登录 |
| lang_english | English | English |
| lang_chinese | 中文（简体） | 中文（简体） |

---

## 5. 响应式适配

| 屏幕宽度 | Sidebar | Main Content | 说明 |
|---------|---------|-------------|------|
| ≥ 1920px | 展开 240px | `calc(100% - 240px)` | 完整布局，推荐 |
| 1440-1919px | 展开 220px | `calc(100% - 220px)` | 完整布局 |
| 1280-1439px | 默认折叠 64px | `calc(100% - 64px)` | 可手动展开 |
| < 1280px | 强制折叠 64px | `calc(100% - 64px)` | 提示用户使用更大屏幕（可选） |

---

## 6. 验收标准

### 6.1 登录页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 背景图 | 全屏城市夜景背景，cover 填充，无拉伸变形 |
| 2 | 登录卡片 | 白色居中，圆角 16-20px，阴影正确 |
| 3 | 品牌标识 | "SMART CONSTRUCTION" + "BACK OFFICE" 显示正确 |
| 4 | 语言切换 | 文A 图标点击弹出下拉，切换后文本即时更新 |
| 5 | Tab 切换 | MAINCON/SUBCON 切换，下划线动画流畅 |
| 6 | 输入框 | 有边框式（bordered），聚焦蓝色边框，Placeholder 正确 |
| 7 | 密码切换 | 眼睛图标点击切换明文/掩码 |
| 8 | 错误态 | 空字段红色边框 + "required" 提示 |
| 9 | Remember | Checkbox 默认选中，蓝色填充 |
| 10 | Login 按钮 | 蓝色全宽，hover 加深，Loading 状态 |
| 11 | Tenant URL | 页面加载解析 tenantCode，无效 URL 显示错误页 |
| 12 | 登录成功 | 跳转管理后台，Token 保存正确 |

### 6.2 管理后台框架

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | Header 固定 | 不随页面滚动，z-index 正确 |
| 2 | Logo 区域 | Logo + 品牌文字正确显示，折叠时仅显示 Logo |
| 3 | 折叠按钮 | 点击切换 Sidebar 展开/折叠，动画 300ms |
| 4 | 面包屑 | 根据路由自动生成，上级可点击 |
| 5 | 项目选择器 | 下拉菜单正确，切换刷新数据 |
| 6 | 通知角标 | 有未读时显示红色数字，超 99 显示 "99+" |
| 7 | 用户菜单 | 下拉包含个人中心、修改密码、退出登录 |
| 8 | Sidebar 展开 | 宽度 220-240px，菜单图标 + 文字 + 选中指示器 |
| 9 | Sidebar 折叠 | 宽度 64px，仅图标，Tooltip 正确 |
| 10 | 菜单权限 | 根据用户权限动态渲染 |
| 11 | Main Content | 正确偏移，背景色 #F5F7FA，Sidebar 切换时过渡平滑 |
| 12 | Home 默认 | 登录后默认进入 Home 空白页，菜单选中态正确 |
| 13 | Tab 栏 | 选中蓝色下划线，切换动画流畅 |
| 14 | 响应式 | 1280px 以下 Sidebar 自动折叠 |
