---
doc_type: frontend_spec
req_id: FRONTEND-REQ-002-pc
version: 0.1.0
status: draft
generated_from: outputs/ui/pc/UI-REQ-002-pc.md@0.2.0
generated_at: 2026-05-04
owner: ""
---

# 前端实现说明：PC 管理端框架（Header / Sidebar / Main Content）

目的：把 `outputs/ui/pc/UI-REQ-002-pc.md` 中的视觉/交互约定翻译成可执行的前端实现契约（Vue 2 + SCSS 为示例）。文档包含组件 contract、必需的 CSS 变量、示例 DOM、键盘/无障碍要点、以及常见实现注意事项，便于 agent 或开发者直接生成运行页面。

交付物清单（本文件应与代码一起被引用）
- `src/components/Layout/BaseLayout.vue` — 容器布局（Header + Sidebar + Content），管理 collapsed state
- `src/components/Header/AppHeader.vue` — 顶部操作区（汉堡、面包屑、项目切换、通知、个人中心）
- `src/components/Sidebar/AppSidebar.vue` — 侧边栏（支持展开/折叠、多级菜单、高亮）
- `src/components/Icon.vue` — 图标组件，基于 `assets/icons/icons.json` 的 manifest
- `src/styles/_tokens.scss` — CSS 变量与 design tokens

设计契约（小合同）
- Inputs: token 集（CSS 变量）、icons manifest 路径 `assets/icons/icons.json`、菜单数据（tree）以及 `isCollapsed` boolean
- Outputs: DOM + CSS 类（`.sidebar`, `.sidebar.collapsed`, `.menu-item`, `.menu-item.is-active` 等），并触发 events: `toggle-collapse`, `menu-select(route)`, `open-dropdown(name)`
- Error modes: missing icon → render placeholder icon and log warning; missing manifest entry → fallback to text label only; long text overflow handled by truncation
- Success criteria: visual result matches tokens (colors/sizes), keyboard navigable, `aria-current` set on active menu item, transitions smooth (300ms)

实现细节一览（要点先行）
- Sidebar 展开宽度 224px，折叠宽度 72px，过渡 300ms
- Header 高度 48px，左右 padding 16px
- Sidebar 菜单项高度 56px，icon 20×20，icon 与文字 gap 8px
- 选中态：使用背景色 `--Primary-Color`（#1890FF）并将文字/图标颜色设为 `--Vertical-Menu-White`（#FFFFFF）-- 不使用左侧竖线
- 折叠态：`brand-text` 隐藏，`logo-icon` 仍显示且垂直居中

一、CSS 变量（tokens） — `src/styles/_tokens.scss`

建议把以下变量放到 `_tokens.scss` 并在全局引入：

```scss
:root {
  --Primary-Color: #1890FF;
  --color-error: #FF4D4F;
  --Vertical-Menu-White: #FFFFFF;
  --text-color-primary-dark: #344050;
  --text-color-primary-light: #5E6E82;
  --background-color: #F1F1F4;
  --table-dividing-line: #E4E7ED;
  --border-color: #D8E2F0;

  --header-height: 48px;
  --sidebar-width-expanded: 224px;
  --sidebar-width-collapsed: 72px;
  --sidebar-item-height: 56px;
  --transition-fast: 100ms;
  --transition-medium: 300ms;
}
```

二、通用布局 DOM 示例（Layout）

文件：`src/components/Layout/BaseLayout.vue`（伪代码）：

```html
<template>
  <div class="app-layout" :class="{ 'sidebar-collapsed': isCollapsed }">
    <app-header
      :is-collapsed="isCollapsed"
      @toggle-collapse="$emit('toggle-collapse')"
      @open-dropdown="onOpenDropdown"
    />

    <aside class="sidebar" :class="{ collapsed: isCollapsed }">
      <div class="sidebar-header">
        <img class="logo-icon" src="/assets/logo.svg" alt="Smart construction site" />
        <div class="brand-text" aria-hidden="true">Smart construction site</div>
      </div>
      <app-sidebar
        :menus="menus"
        :is-collapsed="isCollapsed"
        @select="onMenuSelect"
      />
    </aside>

    <main class="main-content" :style="{ padding: '16px' }">
      <router-view />
    </main>
  </div>
</template>
```

三、Sidebar 组件契约与实现示例

文件：`src/components/Sidebar/AppSidebar.vue` — props & events

- props:
  - menus: Array<MenuItem> （MenuItem = { id, label, icon, route, children? }）
  - isCollapsed: Boolean
- events:
  - select(menuItem)

DOM 结构：

```html
<nav class="sidebar-menu" role="navigation" aria-label="主导航">
  <ul>
    <li v-for="item in menus" :key="item.id" :class="['menu-item', { 'has-children': item.children, 'is-active': isActive(item) }]" tabindex="0" @click="select(item)" @keydown.enter="select(item)">
      <span class="menu-item-icon"><icon :name="item.icon" /></span>
      <span class="menu-item-text" v-if="!isCollapsed">{{ item.label }}</span>
      <span class="menu-item-arrow" v-if="item.children && !isCollapsed">▾</span>
      <!-- children rendering omitted for brevity -->
    </li>
  </ul>
</nav>
```

SCSS（摘选）：

```scss
.sidebar {
  width: var(--sidebar-width-expanded);
  transition: width var(--transition-medium) ease-in-out;
  &.collapsed {
    width: var(--sidebar-width-collapsed);
  }
}

.sidebar-header {
  height: var(--sidebar-item-height);
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 9px 8px;
}

.sidebar-menu .menu-item {
  display: flex;
  align-items: center;
  height: var(--sidebar-item-height);
  padding: 9px 8px;
  gap: 8px;
  color: var(--text-color-primary-dark);
  cursor: pointer;
  transition: background var(--transition-fast) ease;
}

.sidebar-menu .menu-item:hover {
  background: #E8F7FF; // hover token
}

.sidebar-menu .menu-item.is-active {
  background: var(--Primary-Color);
  color: var(--Vertical-Menu-White);
}

.sidebar.collapsed .menu-item .menu-item-text {
  display: none;
}
```

注意：`is-active` 的设置应由路由匹配或组件内部判断（例如使用 `this.$route.path.startsWith(item.route)`）；active 项必须在 DOM 上设置 `aria-current="page"`。

四、Header 组件契约与实现示例

文件：`src/components/Header/AppHeader.vue`

- props:
  - isCollapsed: Boolean
- events:
  - toggle-collapse
  - open-dropdown(name)

重要元素：汉堡按钮、面包屑、项目切换、Dashboard、通知（Badge）、个人中心（Avatar + 下拉）

DOM 片段：

```html
<header class="app-header" role="banner">
  <button class="hamburger" @click="$emit('toggle-collapse')" aria-label="折叠侧边栏 / 展开侧边栏">
    <icon name="menu" />
  </button>

  <nav class="breadcrumb" aria-label="Breadcrumb"> ... </nav>

  <div class="header-actions">
    <div class="project-switch" role="button" tabindex="0" @click="$emit('open-dropdown','project')">...</div>
    <button class="dashboard" aria-label="Dashboard"> <icon name="dashboard"/> </button>
    <div class="notifications"> <badge :count="count"> <icon name="bell"/> </badge> </div>
    <dropdown class="profile"> <avatar/> </dropdown>
  </div>
</header>
```

SCSS（摘选）：

```scss
.app-header {
  height: var(--header-height);
  padding: 0 16px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  box-shadow: 0 1px 4px 0 rgba(240, 242, 245, 0.47);
}

.header-actions { display:flex; gap:16px; align-items:center; }

.hamburger { width:20px; height:20px; }
.notifications .badge { background: #F56C6C; color: var(--Vertical-Menu-White); }
```

五、Icon 组件（简短实现建议）

目的：统一 icon 使用；从 `assets/icons/icons.json` 读取映射并内联或使用 sprite。

行为：
- props: `name`, `size`, `aria`
- 如果 manifest 标记 `isPlaceholder`，在渲染元素上加 `data-placeholder="true"` 并使用 `aria-label`。

实现要点：内联 SVG 更易染色与无障碍控制，若使用 sprite，确保 build step 生成 `assets/dist/icons/sprite.svg` 并在页面头部 inline 引入以避免跨-origin referrer 问题。

六、交互 & 无障碍

- 键盘：
  - Sidebar 菜单项需可通过 Tab / ArrowUp / ArrowDown 导航，Enter 激活，Esc 关闭子菜单。
  - 折叠态 Tooltip 需有 `role="tooltip"` 并与图标 `aria-describedby` 关联。
- ARIA：
  - 当前菜单项加 `aria-current="page"`。
  - 所有图标装饰元素加 `aria-hidden="true"`，交互图标需提供 `aria-label`。
- 对比度：
  - 选中态（白字 + 主色）必须满足 WCAG AA（若不可，调整颜色或加入 outline）。

七、菜单数据 shape（示例）

```json
[
  { "id": "home", "label": "首页", "icon": "home", "route": "/home" },
  { "id": "dashboard", "label": "数据看板", "icon": "dashboard", "route": "/dashboard" },
  { "id": "progress", "label": "进度管理", "icon": "progress", "route": "/progress", "children": [ { "id":"progress-setting","label":"进度设置","route":"/progress/setting" } ] }
]
```

八、折叠/展开逻辑（实现 contract）

- 控制点：父组件 `BaseLayout` 持有 `isCollapsed`（Boolean），并通过 prop 传给 `AppSidebar` 与 `AppHeader`。
- 切换动作：`AppHeader` 触发 `toggle-collapse` 事件；`BaseLayout` 切换 state 并在必要时把 state 同步到 localStorage（以便刷新后保持折叠偏好）。
- 动画：宽度使用 CSS transition，内容区域使用 margin/transform 过渡；不依赖 JS 动画以保证流畅和可回退。

九、测试与验收（smoke tests）

- 单元测试：
  - Sidebar 渲染：在展开/折叠两种状态下快照测试（menu items, icons, tooltip behavior）
  - Active 状态：路由变更后，正确 menu item 加 `is-active` 与 `aria-current`。
- 集成（端到端）：
  - 验证点击汉堡按钮后，Sidebar 宽度从 224px 变为 72px（或 class 切换），Main Content 宽度相应变化。
  - 通知下拉打开后，badge 数字展示与超限 `99+` 行为。

十、构建与 assets（建议）

- icons manifest：`assets/icons/icons.json`（agent 可读取）
- 占位资源：`assets/placeholders/` 用于生成原型（占位 svg），并在 manifest 中用 `isPlaceholder` 标记
- Build step：在 CI 中执行 `npm run build:assets`（svgo + sprite）并把产物部署到 CDN；开发环境可引用 `assets/dist/`。

十一、兼容与降级

- 在不支持 CSS variable 的环境（很少见）使用 SASS variables 生成兼容样式。
- 在不得已使用 icon font 的旧项目，提供 `assets/dist/icons/iconfont.woff2` 与 CSS 类替代内联 SVG（同时在 manifest 标注 type: "font"）。

十二、变更记录

- 本文为从 `UI-REQ-002-pc.md` 自动生成的前端实现草案。实现或视图与实际产品上线前，请保证与设计稿（Figma）完成一次一致性校验。

---

如需，我可以：
- A. 在仓库中生成上述组件样板文件（`BaseLayout.vue`、`AppHeader.vue`、`AppSidebar.vue`、`Icon.vue`）并提交一个示例页面供 agent 生成页面时直接引用；或
- B. 仅把此 spec 提交（我已添加到 `outputs/frontend/pc/FRONTEND-REQ-002-pc.md`），等你确认后我再生成代码样板。

请选择 A 或 B（或提供修改点），我会继续执行下一步。
