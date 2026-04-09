# 前端开发说明文档 — APP 端登录与首页功能

> **来源需求**: [REQ-001-app.md](../../../requirements/app/REQ-001-app.md) + [REQ-001-shared.md](../../../requirements/shared/REQ-001-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: APP 移动端（UNIAPP + Vue 2，iOS & Android）
> **生成日期**: 2026-04-08

---

## 1. 功能概述

本模块实现 APP 端的登录页面与登录后首页（Work Space），涵盖：

| 功能模块 | 说明 |
|---------|------|
| 登录页面 | 上半品牌展示区 + 下半白色表单卡片，支持账号密码登录和扫码登录 |
| 企业账号（租户）识别 | 用户手动输入 Company Account（区别于 PC 端 URL 解析） |
| 语言切换 | 左上角语言按钮，中文/English，偏好保存本地 |
| 首页（Work Space） | 安全标语 Banner + 按项目/模块分组的功能入口卡片 |
| 项目切换面板 | 汉堡菜单展开，左侧滑入项目列表，支持手势关闭 |
| 底部 Tab 导航 | Work Space / Notification / Profile 三 Tab 固定底部 |
| Token 刷新 | 401 自动刷新，Refresh Token 过期跳回登录 |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | UNIAPP + Vue 2 |
| 编程语言 | JavaScript（ES6+） |
| 样式预处理 | SCSS |
| HTTP | 基于 `uni.request` 封装的 `request.js`，自动附带 `Authorization` / `X-Tenant-Id` / `Project-Id` / `lang` Header |
| 本地存储 | `uni.setStorageSync` / `uni.getStorageSync`（非敏感） |
| 全局状态 | Vuex（UNIAPP 支持），module：`user`、`app` |
| 国际化 | `vue-i18n`，key 命名见第 7 节 |
| 图片选择 | `uni.chooseImage` |
| 摄像头（扫码） | `uni.scanCode` |
| 安全区域 | `uni.getSystemInfoSync().safeAreaInsets` 处理底部安全区域 |

---

## 3. 文件与目录结构

```
├── pages/
│   ├── login/
│   │   └── index.vue              # 登录页面
│   └── home/
│       └── index.vue              # 首页（Work Space）
├── components/
│   ├── ProjectDrawer.vue          # 项目选择侧边面板
│   ├── FunctionCard.vue           # 功能入口卡片（含图标网格）
│   └── SafetyBanner.vue           # 安全标语 Banner
├── store/
│   └── modules/
│       ├── user.js                # Token / 用户信息 / 权限
│       └── app.js                 # 当前项目 / 语言设置
├── api/
│   └── auth.js                    # 登录 / 刷新 Token / 登出 / 权限信息 API
├── utils/
│   ├── request.js                 # uni.request 封装
│   └── storage.js                 # 本地存储封装（key 常量）
└── i18n/
    ├── en.json
    └── zh.json
```

---

## 4. 登录页 `pages/login/index.vue`

### 4.1 数据状态

| 变量 | 类型 | 初始值 | 说明 |
|------|------|--------|------|
| `form` | Object | `{ companyAccount: '', username: '', password: '' }` | 登录表单 |
| `errors` | Object | `{}` | 字段错误信息 |
| `loading` | Boolean | `false` | Sign In 按钮 loading |

### 4.2 核心方法

| 方法 | 说明 |
|------|------|
| `handleSignIn()` | 表单本地校验 → 调用登录 API → 保存 Token → 获取权限信息 → 跳转首页 |
| `handleScanCode()` | 调用 `uni.scanCode` 扫码 → 发送扫码令牌到后端验证 |
| `toggleLang()` | 弹出语言选择弹窗，切换 i18n locale，存入 `uni.setStorageSync('lang', ...)` |
| `validateForm()` | 本地字段校验（非空 + 长度），更新 `errors`，返回 `Boolean` |

### 4.3 表单校验

```javascript
validateForm() {
  this.errors = {}
  const { companyAccount, username, password } = this.form
  if (!companyAccount) this.errors.companyAccount = this.$t('login.error.companyRequired')
  else if (companyAccount.length < 2) this.errors.companyAccount = this.$t('login.error.companyTooShort')
  if (!username) this.errors.username = this.$t('login.error.usernameRequired')
  else if (username.length < 3) this.errors.username = this.$t('login.error.usernameTooShort')
  if (!password) this.errors.password = this.$t('login.error.passwordRequired')
  else if (password.length < 6) this.errors.password = this.$t('login.error.passwordTooShort')
  return Object.keys(this.errors).length === 0
}
```

### 4.4 登录流程

```
用户点击 Sign In
  → validateForm()
  → 校验失败 → 字段下方红色提示，停止
  → 校验通过 → loading = true，禁用按钮
  → POST /system/auth/login（附带 companyAccount 作为 tenantCode / X-Tenant-Code）
  → 成功:
      保存 access_token, refresh_token, tenant_id, tenant_name 到 storage
      调用 GET /system/auth/get-permission-info
      获取 projectInfos，存入 Vuex user.projectInfos
      uni.switchTab({ url: '/pages/home/index' })
  → 失败: loading = false，显示后端错误，清空密码字段
```

### 4.5 扫码登录流程

```javascript
async handleScanCode() {
  try {
    const { result } = await uni.scanCode({ onlyFromCamera: true })
    this.loading = true
    const res = await scanLogin(result)
    await this.handleLoginSuccess(res)
  } catch (e) {
    if (e?.errMsg !== 'scanCode:fail cancel') {
      uni.showToast({ title: this.$t('login.error.qrInvalid'), icon: 'none' })
    }
  } finally {
    this.loading = false
  }
}
```

### 4.6 页面结构（模板）

```vue
<template>
  <view class="login-page">
    <!-- 上半：品牌展示区 -->
    <view class="brand-section">
      <view class="top-bar">
        <view class="lang-btn" @tap="toggleLang">文A</view>
        <text class="brand-label">Smart Site</text>
      </view>
      <view class="brand-logo">
        <text class="logo-main">NOVASYNC</text>
        <text class="logo-sub">S M A R T  S I T E</text>
      </view>
      <image class="brand-illustration" src="/static/images/city-3d.png" />
    </view>

    <!-- 下半：表单卡片 -->
    <view class="form-card">
      <!-- Company Account -->
      <view class="field">
        <text class="label">Company Account</text>
        <input class="input" v-model="form.companyAccount" :placeholder="$t('login.placeholder.company')" />
        <text v-if="errors.companyAccount" class="error">{{ errors.companyAccount }}</text>
      </view>
      <!-- User Account -->
      <view class="field">
        <text class="label">User Account</text>
        <input class="input" v-model="form.username" :placeholder="$t('login.placeholder.username')" />
        <text v-if="errors.username" class="error">{{ errors.username }}</text>
      </view>
      <!-- Password -->
      <view class="field">
        <text class="label">Password</text>
        <input class="input" v-model="form.password" password :placeholder="$t('login.placeholder.password')" />
        <text v-if="errors.password" class="error">{{ errors.password }}</text>
      </view>

      <!-- 操作按钮 -->
      <view class="action-row">
        <button class="btn-signin" :loading="loading" @tap="handleSignIn">Sign In</button>
        <button class="btn-scan" @tap="handleScanCode"><uni-icons type="scan" size="24" /></button>
      </view>

      <!-- 协议文本 -->
      <view class="agreement">
        <text>By clicking 'Login' you agree to our </text>
        <text class="link">Terms of Conditions</text>
        <text> and </text>
        <text class="link">Privacy Policy</text>
      </view>
    </view>
  </view>
</template>
```

---

## 5. 首页 `pages/home/index.vue`（Work Space）

### 5.1 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `projectDrawerVisible` | Boolean | 项目选择面板是否展开 |
| `functionModules` | Array | 当前项目下的功能模块卡片列表 |
| `loading` | Boolean | 首页数据加载状态 |

### 5.2 首页数据加载

```javascript
onShow() {
  this.loadHomeData()
},
methods: {
  async loadHomeData() {
    this.loading = true
    try {
      const res = await getPermissionInfo()
      this.$store.commit('user/SET_PERMISSION_INFO', res)
      this.buildFunctionModules(res.projectInfos, res.permissions)
    } finally {
      this.loading = false
    }
  },
  buildFunctionModules(projectInfos, permissions) {
    // 根据权限过滤，生成功能模块卡片结构
    this.functionModules = projectInfos.map(project => ({
      title: project.shortName || project.fullName,
      projectId: project.id,
      items: this.filterFunctionItems(project, permissions)
    }))
  }
}
```

### 5.3 项目切换

```javascript
handleProjectSelect(project) {
  this.$store.commit('app/SET_CURRENT_PROJECT', project)
  uni.setStorageSync('project_id', project.id)
  this.projectDrawerVisible = false
  this.loadHomeData()
}
```

### 5.4 功能入口配置

```javascript
// utils/function-items.js
export const FUNCTION_ITEMS = [
  { key: 'project',       label: 'Project',             permission: 'project:info:query',    icon: 'project',    route: '/pages/project/index' },
  { key: 'subcontractor', label: 'Subcontractor',        permission: 'subcontractor:query',   icon: 'subcontractor', route: '/pages/subcontractor/index' },
  { key: 'personList',    label: 'Person List',          permission: 'personnel:query',       icon: 'person',     route: '/pages/personnel/index' },
  { key: 'equipmentList', label: 'Equipment List',       permission: 'equipment:query',       icon: 'equipment',  route: '/pages/equipment/index' },
  { key: 'refueling',     label: 'Refueling',            permission: 'equipment:refueling:query', icon: 'refueling', route: '/pages/equipment/refueling/index' },
  { key: 'drawings',      label: 'Drawings',             permission: 'drawing:view',          icon: 'drawing',    route: '/pages/drawings/index' },
  // ... 其他功能
]
```

---

## 6. 项目选择面板 `ProjectDrawer.vue`

### 6.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 是否展开 |
| `projectList` | Array | 项目列表 |
| `currentProjectId` | Number \| null | 当前选中项目 ID |

### 6.2 Emits

| 事件 | 参数 | 说明 |
|------|------|------|
| `select` | `project` | 用户选择项目 |
| `close` | — | 关闭面板 |

### 6.3 动画

```javascript
// 使用 uni.createAnimation 或 CSS transform
watch: {
  visible(val) {
    this.translateX = val ? '0' : '-100%'
  }
}
```

CSS：`transition: transform 0.3s ease-in-out;`

---

## 7. Axios（uni.request）封装 `utils/request.js`

```javascript
// 请求拦截：自动附加 Header
const headers = {
  'Authorization': `Bearer ${uni.getStorageSync('access_token')}`,
  'X-Tenant-Id': uni.getStorageSync('tenant_id'),
  'Project-Id': uni.getStorageSync('project_id') || '',
  'lang': uni.getStorageSync('lang') || 'en'
}

// 响应拦截：401 刷新 Token
if (res.statusCode === 401) {
  try {
    await refreshTokenApi()
    return request(options) // 重试
  } catch {
    uni.clearStorageSync()
    uni.navigateTo({ url: '/pages/login/index' })
  }
}
```

---

## 8. Vuex 状态管理

### `store/modules/user.js`

| State | 说明 |
|-------|------|
| `accessToken` | Access Token |
| `refreshToken` | Refresh Token |
| `tenantId` | 租户 ID |
| `userInfo` | 用户信息 |
| `permissions` | 权限列表 |
| `projectInfos` | 用户可访问的项目列表 |

### `store/modules/app.js`

| State | 说明 |
|-------|------|
| `currentProject` | 当前选中项目对象 |
| `lang` | 当前语言（`'en'` / `'zh'`） |

---

## 9. 国际化（i18n）

```
login.placeholder.company  → "Please enter company account"
login.placeholder.username → "Please enter user name"
login.placeholder.password → "Please enter password"
login.btn.signIn           → "Sign In"
login.error.companyRequired  → "Please enter company account"
login.error.companyTooShort  → "Company account must be at least 2 characters"
login.error.usernameRequired → "Please enter user name"
login.error.usernameTooShort → "User name must be at least 3 characters"
login.error.passwordRequired → "Please enter password"
login.error.passwordTooShort → "Password must be at least 6 characters"
login.error.qrInvalid      → "QR code is invalid or expired, please try again"
home.title                 → "Smart Site"
home.empty                 → "No project assigned. Please contact your administrator."
home.selectProject         → "Select Project"
```

---

## 10. 验收标准

### 登录页
- [ ] 上半品牌区正确显示 NOVASYNC logo、SMART SITE 文字和城市 3D 插画
- [ ] 下半表单卡片正确显示 3 个字段（Company Account / User Account / Password）
- [ ] 点击 Sign In 时必填校验触发，错误提示显示在对应字段下方
- [ ] 企业账号 < 2 字符 / 用户名 < 3 字符 / 密码 < 6 字符显示对应长度错误
- [ ] 登录成功后跳转首页，本地保存 Token 和租户信息
- [ ] 登录失败后显示错误信息，清空密码字段
- [ ] 扫码登录按钮打开摄像头，扫码失效时提示
- [ ] 语言切换后所有文本即时更新，偏好持久保存

### 首页
- [ ] 有有效 Token 时直接进入首页（无需重新登录）
- [ ] 无有效 Token 时跳转登录页
- [ ] 调用权限信息 API 后根据项目和权限动态渲染功能卡片
- [ ] 无项目权限时显示空状态提示
- [ ] 汉堡菜单点击后从左侧滑入项目选择面板，带遮罩
- [ ] 点击项目后切换并刷新首页内容，Project-Id 正确更新
- [ ] 点击遮罩 / 左滑 / 返回按钮关闭面板不切换项目
- [ ] 底部 Tab 正确切换（Work Space / Notification / Profile）
- [ ] 选中 Tab 显示青绿色

### Token 与会话
- [ ] 401 时自动刷新 Token 并重试原请求
- [ ] Refresh Token 过期时清除本地存储，跳转登录页
- [ ] 所有请求自动附带 Authorization / X-Tenant-Id / Project-Id / lang
