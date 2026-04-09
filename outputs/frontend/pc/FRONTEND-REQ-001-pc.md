# 前端开发说明文档 — PC 管理端登录与管理后台框架

> **来源需求**: [REQ-001-pc.md](../../../requirements/pc/REQ-001-pc.md) + [REQ-001-shared.md](../../../requirements/shared/REQ-001-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

本模块实现 PC 管理端的登录页面与登录后管理后台框架，涵盖：

| 功能模块 | 说明 |
|---------|------|
| 登录页面 | 全屏背景 + 居中登录卡片，支持 MAINCON / SUBCON 双角色登录，URL 自动解析租户 |
| 租户识别 | 从 URL 路径自动解析 `tenantCode`（如 `/mcc/login` → `mcc`） |
| 记住密码 | 勾选后本地存储账号，下次自动填充 |
| 管理后台框架 | 固定顶部导航栏 + 可折叠左侧菜单 + 主内容区域 |
| 项目切换器 | 顶部右侧，切换项目后全局刷新 `Project-Id` |
| 通知铃铛 | 显示 Todo/通知数量，点击打开通知面板 |
| 用户下拉菜单 | 个人中心 / 修改密码 / 退出登录 |
| 多语言切换 | English / 中文，语言偏好保存本地，即时刷新 |
| Token 刷新 | 401 自动刷新，Refresh Token 过期跳回登录页 |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 + Vue Router + Vuex |
| UI 组件库 | Element UI 2.x |
| HTTP | Axios，统一封装 `request.js`，自动附带 `Authorization` / `X-Tenant-Id` / `Project-Id` / `lang` Header |
| 国际化 | Vue I18n，key 命名见第 7 节 |
| 状态管理 | Vuex module：`user`（用户信息、Token）、`app`（全局布局状态） |
| 路由守卫 | 全局 `beforeEach`：无 Token → 跳转 `/:tenantCode/login`；有 Token → 放行 |
| 代码风格 | ESLint + Prettier，与项目现有配置一致 |
| 本地存储 key | `access_token`、`refresh_token`、`tenant_id`、`tenant_name`、`project_id`、`lang` |

---

## 3. 文件与目录结构

```
src/
├── views/
│   ├── login/
│   │   └── index.vue              # 登录页面
│   └── layout/
│       └── index.vue              # 管理后台框架（AppLayout）
├── layout/
│   └── components/
│       ├── AppHeader.vue          # 顶部导航栏
│       ├── AppSidebar.vue         # 左侧导航菜单
│       ├── ProjectSelector.vue    # 项目切换器
│       ├── NotificationBell.vue   # 通知铃铛 + 面板
│       └── UserDropdown.vue       # 用户下拉菜单
├── store/
│   └── modules/
│       ├── user.js                # 用户信息 / Token / 权限 Vuex module
│       └── app.js                 # 布局状态（sidebarCollapsed、lang、currentProject）
├── api/
│   └── auth.js                    # 登录 / 刷新 Token / 登出 / 权限信息 API 封装
├── router/
│   └── index.js                   # 路由配置 + 全局守卫
└── utils/
    ├── request.js                 # Axios 封装（含 Token 刷新拦截器）
    └── tenant.js                  # URL 租户解析工具
```

---

## 4. 页面与组件详细说明

### 4.1 登录页面 `login/index.vue`

#### 4.1.1 数据状态

| 变量 | 类型 | 初始值 | 说明 |
|------|------|--------|------|
| `activeTab` | String | `'MAINCON'` | 当前登录 Tab：`'MAINCON'` / `'SUBCON'` |
| `loginForm` | Object | `{ username: '', password: '' }` | 表单数据 |
| `rememberPassword` | Boolean | `true` | 记住密码 |
| `loading` | Boolean | `false` | 登录按钮 loading 状态 |
| `tenantCode` | String | `''` | 从 URL 解析出的租户标识 |
| `tenantError` | Boolean | `false` | URL 中无有效 tenantCode 时为 true |

#### 4.1.2 生命周期

```javascript
created() {
  // 1. 从 URL 路径解析 tenantCode
  this.tenantCode = parseTenantCode(this.$route.params.tenantCode)
  if (!this.tenantCode) {
    this.tenantError = true
    return
  }
  // 2. 若有记住的账号，自动填充 username
  const savedUsername = localStorage.getItem('remembered_username')
  if (savedUsername) {
    this.loginForm.username = savedUsername
  }
}
```

#### 4.1.3 核心方法

| 方法 | 说明 |
|------|------|
| `handleLogin()` | 触发 el-form 验证 → 调用对应登录 API → 保存 Token → 获取权限信息 → 跳转首页 |
| `switchTab(tab)` | 切换 MAINCON / SUBCON，重置表单 |
| `handleLoginSuccess(data)` | 保存 `access_token`、`refresh_token`、`tenant_id`、`tenant_name` 到 localStorage，提交到 Vuex |
| `fetchPermissionInfo()` | 调用 `/system/auth/get-permission-info`，获取项目列表和权限，存入 Vuex `user` module |
| `toggleLang()` | 弹出语言选择菜单，切换 Vue I18n locale，保存到 localStorage `lang` |

#### 4.1.4 登录流程

```
页面加载
  → parseTenantCode() from URL params
  → 无 tenantCode → 显示错误页"Invalid access URL..."，停止
  → 有 tenantCode → 正常渲染登录卡片
用户填写表单 → 点击 Login
  → el-form.validate()
  → 验证失败 → 输入框标红 + 显示 "required"，停止
  → 验证通过 → loading = true，禁用按钮
  → MAINCON: POST /system/auth/login（附带 tenantCode）
  → SUBCON: POST /system/auth/subcontractor/login（附带 tenantCode）
  → 成功: handleLoginSuccess() → fetchPermissionInfo() → $router.push('/')
  → 失败: loading = false，显示后端错误信息，清空密码字段
```

#### 4.1.5 表单验证规则

```javascript
loginRules: {
  username: [
    { required: true, message: 'required', trigger: 'submit' }
  ],
  password: [
    { required: true, message: 'required', trigger: 'submit' }
  ]
}
```

> 后端字段长度验证在 shared 文档中定义，前端仅做非空验证，错误提示显示在输入框下方红色文字 "required"。

#### 4.1.6 记住密码逻辑

```javascript
// 登录成功后
if (this.rememberPassword) {
  localStorage.setItem('remembered_username', this.loginForm.username)
} else {
  localStorage.removeItem('remembered_username')
}
```

---

### 4.2 管理后台框架 `AppLayout`

#### 4.2.1 整体结构

```vue
<template>
  <div class="app-layout">
    <AppHeader />
    <div class="app-body">
      <AppSidebar />
      <main class="main-content">
        <router-view />
      </main>
    </div>
  </div>
</template>
```

#### 4.2.2 布局尺寸

| 区域 | 规格 |
|------|------|
| Header 高度 | 56px，固定顶部（`position: fixed; top: 0`） |
| Sidebar 宽度（展开） | 220px |
| Sidebar 宽度（折叠） | 64px |
| Main 内边距 | 16px 或 24px |
| z-index: Header | 1000 |
| z-index: Sidebar | 900 |

---

### 4.3 顶部导航栏 `AppHeader.vue`

#### 4.3.1 Props / 数据

从 Vuex `app` module 读取：`sidebarCollapsed`、`currentProject`  
从 Vuex `user` module 读取：`userInfo`、`projectList`、`notificationCount`

#### 4.3.2 组件结构

```
AppHeader
  ├── Logo 区域（品牌图标 + "Smart construction site"，折叠时隐藏文字）
  ├── 侧边栏折叠按钮（≡ 图标）
  ├── 面包屑导航（动态，基于 $route.matched）
  └── 右侧工具区
      ├── ProjectSelector.vue
      ├── 数据统计图标（跳转 Dashboard）
      ├── NotificationBell.vue
      └── UserDropdown.vue
```

#### 4.3.3 折叠按钮

```javascript
toggleSidebar() {
  this.$store.commit('app/TOGGLE_SIDEBAR')
}
```
Sidebar 展开/折叠通过 CSS transition 实现宽度过渡动画（`transition: width 0.3s ease`）。

---

### 4.4 项目切换器 `ProjectSelector.vue`

#### 4.4.1 数据

```javascript
computed: {
  projectList() { return this.$store.state.user.projectInfos },
  currentProject() { return this.$store.state.app.currentProject }
}
```

#### 4.4.2 切换逻辑

```javascript
handleSelectProject(project) {
  // 1. 更新 Vuex 当前项目
  this.$store.commit('app/SET_CURRENT_PROJECT', project)
  // 2. 更新 localStorage
  localStorage.setItem('project_id', project.id)
  // 3. 触发全局事件，各页面监听并刷新数据
  this.$bus.$emit('project-changed', project.id)
  // 4. 关闭下拉
  this.dropdownVisible = false
}
```

> `request.js` 中的 Axios 请求拦截器自动从 Vuex / localStorage 读取 `Project-Id` 并附加到所有请求 Header。

---

### 4.5 通知铃铛 `NotificationBell.vue`

- 点击铃铛弹出通知面板（Popover / Drawer）
- 角标数量从 Vuex `user.todoCount` 读取
- 通知面板数据调用 `/system/notice/page`（或 Todo 相关 API）
- 超过 99 显示 "99+"

---

### 4.6 用户下拉菜单 `UserDropdown.vue`

| 菜单项 | 操作 |
|--------|------|
| 个人中心 | `$router.push('/profile')` |
| 修改密码 | 弹出修改密码 Dialog |
| 退出登录 | 调用 `POST /system/auth/logout` → 清除所有本地数据 → `$router.push('/:tenantCode/login')` |

**退出登录逻辑**:
```javascript
async handleLogout() {
  await this.$store.dispatch('user/logout')
  // store action 内部: 调用 API → 清除 localStorage → reset Vuex state
  this.$router.push(`/${this.tenantCode}/login`)
}
```

---

## 5. Vuex 状态管理

### 5.1 `store/modules/user.js`

| State | 类型 | 说明 |
|-------|------|------|
| `accessToken` | String | 当前有效的 Access Token |
| `refreshToken` | String | 刷新令牌 |
| `tenantId` | String | 租户 ID |
| `tenantName` | String | 租户名称 |
| `userInfo` | Object | 用户信息（id, nickname, avatar, mobile, signatureUrl） |
| `roles` | Array | 角色列表 |
| `permissions` | Array | 权限标识列表 |
| `projectInfos` | Array | 用户可访问的项目列表 |
| `todoCount` | Number | 待处理 Todo 数量 |

| Action | 说明 |
|--------|------|
| `login(loginReqVO)` | 调用登录 API，保存 Token 到 state + localStorage |
| `getPermissionInfo()` | 调用 `/system/auth/get-permission-info`，填充 userInfo / roles / permissions / projectInfos |
| `logout()` | 调用登出 API，清除所有状态和 localStorage |
| `refreshToken()` | 调用 `/system/auth/refresh-token`，更新 accessToken |

### 5.2 `store/modules/app.js`

| State | 类型 | 说明 |
|-------|------|------|
| `sidebarCollapsed` | Boolean | 侧边栏是否折叠，默认 false |
| `lang` | String | 当前语言，默认 `'en'` |
| `currentProject` | Object \| null | 当前选中项目 |

---

## 6. API 封装 `api/auth.js`

```javascript
// 账号密码登录（MAINCON）
export const loginByPassword = (data) =>
  request.post('/system/auth/login', data)

// 分包商登录（SUBCON）
export const loginSubcontractor = (data) =>
  request.post('/system/auth/subcontractor/login', data)

// 刷新 Token
export const refreshToken = (refreshToken) =>
  request.post('/system/auth/refresh-token', null, { params: { refreshToken } })

// 获取用户权限信息
export const getPermissionInfo = () =>
  request.get('/system/auth/get-permission-info')

// 登出
export const logout = () =>
  request.post('/system/auth/logout')
```

---

## 7. Axios 请求拦截器（`utils/request.js`）

### 7.1 请求拦截

```javascript
config.headers['Authorization'] = `Bearer ${store.state.user.accessToken}`
config.headers['X-Tenant-Id'] = store.state.user.tenantId
config.headers['Project-Id'] = store.state.app.currentProject?.id || localStorage.getItem('project_id')
config.headers['lang'] = store.state.app.lang || 'en'
```

### 7.2 响应拦截（Token 刷新）

```javascript
response.interceptors.response.use(
  res => res.data,
  async error => {
    if (error.response?.status === 401) {
      // 尝试刷新 Token
      try {
        await store.dispatch('user/refreshToken')
        return request(error.config) // 重试原请求
      } catch {
        // Refresh Token 也失效 → 跳转登录
        store.dispatch('user/logout')
        router.push(`/${store.state.user.tenantCode}/login`)
      }
    }
    return Promise.reject(error)
  }
)
```

---

## 8. 路由配置

```javascript
// router/index.js
const routes = [
  {
    path: '/:tenantCode/login',
    name: 'Login',
    component: () => import('@/views/login/index.vue'),
    meta: { public: true }
  },
  {
    path: '/',
    component: AppLayout,
    children: [
      { path: '', redirect: '/home' },
      { path: 'home', name: 'Home', component: () => import('@/views/home/index.vue') },
      // ... 其他业务页面路由
    ]
  }
]

// 全局路由守卫
router.beforeEach((to, from, next) => {
  const token = localStorage.getItem('access_token')
  if (to.meta.public) {
    next()
  } else if (!token) {
    const tenantCode = localStorage.getItem('tenant_code') || 'mcc'
    next(`/${tenantCode}/login`)
  } else {
    next()
  }
})
```

---

## 9. 国际化（i18n）

### 9.1 Key 命名规范

```
login.title              → "Account Login Page"
login.tab.maincon        → "MAINCON"
login.tab.subcon         → "SUBCON"
login.form.username      → "Please fill in account"
login.form.password      → "Please fill in password"
login.btn.submit         → "Login"
login.rememberPassword   → "Remember the password"
login.error.required     → "required"
login.error.invalidUrl   → "Invalid access URL, please contact your administrator"
nav.logout               → "Logout"
nav.profile              → "Profile"
nav.changePassword       → "Change Password"
```

---

## 10. 样式规范

### 10.1 登录页

| 元素 | 样式 |
|------|------|
| 背景 | 全屏城市夜景图，`background-size: cover`；加载失败回退 `#1a2332 → #2c3e50` |
| 登录卡片 | 宽 420px，白色背景，圆角 16px，阴影 `0 8px 32px rgba(0,0,0,0.12)`，水平垂直居中 |
| 输入框 | 高 44px，圆角 6px，边框 `1px #D9D9D9`，聚焦 `1px #4A90D9`，错误 `1px #FF4D4F` |
| 登录按钮 | 100% 宽，高 44px，背景 `#4A90D9`，hover `#357ABD` |
| Tab 选中 | `#4A90D9`，2px 下划线 |
| 错误提示 | `12px #FF4D4F`，显示在输入框下方 4px |

### 10.2 管理后台框架

| 元素 | 样式 |
|------|------|
| Header 背景 | `#FFFFFF`，底部边框 `1px #E8E8E8` |
| Sidebar 背景 | 深色主题（`#001428` 或当前项目主色，与 Element UI NavMenu 一致） |
| 菜单选中项 | 品牌蓝色高亮 `#4A90D9` |
| Sidebar 过渡 | `transition: width 0.3s ease` |

---

## 11. 验收标准

### 登录页
- [ ] URL 中无有效 `tenantCode` → 显示错误提示页面，无法进入登录
- [ ] URL 含 `tenantCode` → 正常显示登录卡片
- [ ] 点击 Login 时账号/密码为空 → 输入框红色 + 显示 "required"，阻止提交
- [ ] 调用登录 API 失败 → 显示后端返回错误信息，清空密码字段，按钮解除 loading
- [ ] 登录成功 → 保存 Token + Tenant 信息，跳转首页
- [ ] 记住密码勾选 → 下次打开自动填充账号
- [ ] MAINCON 与 SUBCON Tab 切换 → 表单重置，调用各自接口
- [ ] 语言切换 → 所有文本即时切换，偏好保存到 localStorage

### 管理后台框架
- [ ] 侧边栏折叠/展开按钮正常切换，带 CSS 过渡动画
- [ ] 面包屑导航随路由自动更新
- [ ] 项目切换器正确显示项目列表，切换后全局 `Project-Id` Header 更新
- [ ] 通知角标显示 Todo 数量，超过 99 显示 "99+"
- [ ] 用户下拉菜单 → 退出登录正确清除本地数据并跳回登录页

### Token 与会话
- [ ] API 返回 401 → 自动刷新 Token 后重试请求
- [ ] Refresh Token 过期 → 清除本地数据，跳转登录页，提示 "Login expired, please sign in again"
- [ ] 所有请求自动附带 `Authorization`、`X-Tenant-Id`、`Project-Id`、`lang` Header
