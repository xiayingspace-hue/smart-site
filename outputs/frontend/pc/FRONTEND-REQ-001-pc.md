---
doc_type: frontend_spec
req_id: REQ-001-pc
version: 0.2.0
status: draft
generated_from: REQ-001-pc.md@0.2.0
ui_spec_ref: UI-REQ-001-pc.md@0.1.0
generated_at: 2026-05-03
owner: ""
---

# 前端开发说明：PC 管理端 — 登录页面

> **本文档供前端开发工程师及其 agent 使用**。
>
> ⚠️ **重要约定**：
> - 所有 UI 样式（颜色 / 间距 / 字体）引用 [UI-REQ-001-pc.md](../../ui/pc/UI-REQ-001-pc.md)，本文档不重复定义。
> - 所有 API 字段定义引用 [REQ-001-shared.md](../../../requirements/shared/REQ-001-shared.md)，本文档只列调用方式。
> - 代码注释中必须标注覆盖的 AC ID（如 `// AC-001-pc-001`）。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-001-pc @ v0.2.0 |
| UI 设计 | UI-REQ-001-pc @ v0.1.0 |
| 共享规则 | REQ-001-shared |
| 覆盖 Story | US-001、US-002、US-003 |
| 覆盖 AC | AC-001-pc-001 ~ AC-001-pc-008 |
| 上次同步时间 | 2026-05-03 |

---

## 1. 功能概述

实现 PC 管理端登录页面，包含：租户识别（URL 解析）、MAINCON / SUBCON Tab 切换、账号密码表单、记住密码、登录接口调用、语言切换。登录成功后跳转至管理后台框架（REQ-002-pc）。

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

- **框架**：Vue 2
- **语言**：JavaScript
- **UI 库**：Element UI
- **状态管理**：Vuex
- **路由**：Vue Router 3
- **HTTP**：Axios（含请求拦截器）
- **国际化**：vue-i18n
- **构建**：Webpack

### 2.2 本需求新增依赖

| 包 | 版本 | 用途 | 评估 |
|---|------|------|------|
| 无新增 | — | — | — |

---

## 3. 路由设计

| 路径 | 组件 | 权限 | 关联 Story |
|-----|------|-----|-----------|
| `/:tenantCode/login` | `LoginPage` | 公开（未登录可访问） | US-001、US-002 |

**URL state 同步**：
- `tenantCode` 从路由 params 读取，不写入 query string
- Tab 状态（MAINCON / SUBCON）不需要 URL 同步（页面刷新默认 MAINCON）
- 登录成功后跳转：`/:tenantCode/home`

---

## 4. 组件结构

### 4.1 组件树

```
<LoginPage>                        // 页面级，处理 tenantCode 解析
├── <InvalidTenantPage>            // URL 无效时渲染，替换整页
└── <LoginCard>                    // 登录卡片
    ├── <BrandSection>             // 品牌标识区（SMART CONSTRUCTION / BACK OFFICE）
    ├── <LanguageSwitcher>         // 语言切换行（Account Login Page + 文A 图标）
    ├── <LoginTabs>                // MAINCON / SUBCON Tab
    ├── <LoginForm>                // 表单容器
    │   ├── <el-input> 账号
    │   ├── <el-input type="password"> 密码（含眼睛图标）
    │   └── <RememberPassword>    // 记住密码 Checkbox
    └── <el-button type="primary"> // Login 按钮
```

### 4.2 关键组件说明

#### `<LoginPage>`

**关联 AC**：AC-001-pc-004

**职责**：
- 从 `$route.params.tenantCode` 读取租户标识
- tenantCode 为空时渲染 `<InvalidTenantPage>`，否则渲染 `<LoginCard>`
- 将 `tenantCode` 写入 Vuex（`auth/setTenantCode`）

**不该做**：不处理登录逻辑，只做路由与租户初始化。

---

#### `<LoginTabs>`

**关联 AC**：AC-001-pc-001、AC-001-pc-002

**Props**：
```js
props: {
  value: { type: String, default: 'MAINCON' } // 'MAINCON' | 'SUBCON'
}
// emits: input
```

**职责**：渲染 MAINCON / SUBCON 两个 Tab，默认选中 MAINCON，切换时 $emit('input')。

---

#### `<LoginForm>`

**关联 AC**：AC-001-pc-001 ~ AC-001-pc-003、AC-001-pc-005 ~ AC-001-pc-007

**Props**：
```js
props: {
  tabType: { type: String, required: true },  // 'MAINCON' | 'SUBCON'
  tenantCode: { type: String, required: true }
}
```

**职责**：
- 管理账号、密码表单字段
- 前端必填校验（触发时机：点击 Login）
- 根据 `tabType` 调用对应登录接口
- 处理 Loading 态、禁止重复点击
- 登录成功：调用 `this.$store.dispatch('auth/saveToken')`，路由跳转
- 登录失败：`this.$message.error(msg)` Toast

---

#### `<LanguageSwitcher>`

**关联 AC**：AC-001-pc-008

**职责**：点击图标弹出 `el-dropdown`，切换语言调用 `this.$i18n.locale = lang`，语言偏好存 `localStorage`。

**实现约束（来自实测问题修正）**：

> ⚠️ 语言切换**必须使用 `el-dropdown` 下拉菜单**，禁止直接渲染中/英文按钮（否则无下拉交互）。

```vue
<!-- LanguageSwitcher.vue -->
<el-dropdown trigger="click" @command="handleLangChange">
  <span class="lang-trigger">
    <svg-icon icon-class="lang" />  <!-- 文A 图标 -->
  </span>
  <el-dropdown-menu slot="dropdown">
    <el-dropdown-item command="en">English</el-dropdown-item>
    <el-dropdown-item command="zh-CN">中文</el-dropdown-item>
  </el-dropdown-menu>
</el-dropdown>
```

```js
methods: {
  handleLangChange(lang) {
    this.$i18n.locale = lang
    localStorage.setItem('lang', lang)
  }
}
```

- `trigger="click"`：点击图标触发，不使用 hover
- 选项固定为 `English` / `中文` 两项，顺序不变
- 当前语言不需要高亮（设计稿无此要求）
- ⚠️ **默认态只渲染图标**：`<el-dropdown>` 的 trigger slot 中只放「文A」图标，**不得**在 trigger slot 或图标旁边显示当前语言名称文字（如 "English"）或备选语言文字（如 "中文"）——语言文字只出现在下拉菜单的 `el-dropdown-item` 中

---

#### `<RememberPassword>`

**关联 AC**：AC-001-pc-007

**职责**：默认选中（checked），值传递给 `<LoginForm>`，决定 Token 存储位置（localStorage / sessionStorage）。

---

## 5. 状态管理

### 5.1 状态分层

| 状态类型 | 存放位置 | 内容 |
|---------|---------|------|
| 全局认证 | Vuex `auth` module | `tenantCode`、`tenantId`、`token`、`tokenStorage` |
| 表单临时 | 组件 `data` | 账号、密码、记住密码开关 |
| 语言偏好 | `localStorage` + `vue-i18n` | `lang: 'zh-CN' / 'en'` |

### 5.2 Vuex auth module 관련 action

```js
// store/modules/auth.js

// AC-001-pc-001 / AC-001-pc-002 / AC-001-pc-007
// REQ-001-shared §3.1：登录成功后保存 accessToken + refreshToken + tenantId + tenantName
saveToken({ commit }, { accessToken, refreshToken, tenantId, tenantName, remember }) {
  commit('SET_TOKEN', accessToken)
  commit('SET_REFRESH_TOKEN', refreshToken)
  commit('SET_TENANT_ID', tenantId)
  const storage = remember ? localStorage : sessionStorage
  storage.setItem('token', accessToken)
  storage.setItem('refreshToken', refreshToken)
  storage.setItem('tenantId', tenantId)
  storage.setItem('tenantName', tenantName)
},

// REQ-001-shared §3.3：登出时清除所有本地登录信息
logout({ commit, dispatch }) {
  return request.post('/system/auth/logout').finally(() => {
    localStorage.removeItem('token')
    localStorage.removeItem('refreshToken')
    localStorage.removeItem('tenantId')
    localStorage.removeItem('tenantName')
    sessionStorage.removeItem('token')
    sessionStorage.removeItem('refreshToken')
    sessionStorage.removeItem('tenantId')
    commit('SET_TOKEN', '')
    commit('SET_REFRESH_TOKEN', '')
    commit('SET_TENANT_ID', '')
  })
},

clearToken({ commit }) {
  localStorage.removeItem('token')
  localStorage.removeItem('refreshToken')
  sessionStorage.removeItem('token')
  sessionStorage.removeItem('refreshToken')
  commit('SET_TOKEN', '')
  commit('SET_REFRESH_TOKEN', '')
}
```

---

## 6. API 接入

> 接口字段完整定义参见 [REQ-001-shared.md](../../../requirements/shared/REQ-001-shared.md)。

### 6.1 调用清单

| 接口 | 调用位置 | 触发时机 |
|-----|---------|---------|
| `POST /system/auth/login` | `<LoginForm>` | 点击 Login，MAINCON Tab |
| `POST /system/auth/subcontractor/login` | `<LoginForm>` | 点击 Login，SUBCON Tab |
| `GET /system/auth/get-permission-info` | 登录成功后（`auth/saveToken` action 内） | 获取用户信息 + 项目列表，写入 Vuex 后再跳转 |
| `POST /system/auth/refresh-token` | Axios 响应拦截器（自动触发） | 收到 401 时自动刷新 Token |
| `POST /system/auth/logout` | `auth/logout` action | 用户点击登出（REQ-002-pc 范围，此处仅定义 action） |

### 6.2 请求拦截器

```js
// utils/request.js
// REQ-001-shared §3.1：所有请求必须携带 Authorization / X-Tenant-Id / lang
service.interceptors.request.use(config => {
  const token = localStorage.getItem('token') || sessionStorage.getItem('token')
  const tenantId = localStorage.getItem('tenantId') || sessionStorage.getItem('tenantId')
  if (token) config.headers['Authorization'] = `Bearer ${token}`
  if (tenantId) config.headers['X-Tenant-Id'] = tenantId
  config.headers['lang'] = localStorage.getItem('lang') || 'en'
  // Project-Id 由各业务页面按需注入，登录页不传
  return config
})
```

### 6.3 响应拦截器 — 401 自动 Token 刷新

> ⚠️ 来自 REQ-001-shared §3.2，必须实现，否则用户会在 Access Token（1h）过期后被强制重新登录。

```js
// utils/request.js
// REQ-001-shared §3.2：401 → 自动用 refreshToken 换新 accessToken → 重发原请求
let isRefreshing = false
let pendingQueue = []

service.interceptors.response.use(
  res => res.data,
  async err => {
    const status = err.response?.status
    const originalRequest = err.config

    if (status === 401 && !originalRequest._retry) {
      const refreshToken = localStorage.getItem('refreshToken') || sessionStorage.getItem('refreshToken')

      if (!refreshToken) {
        // Refresh Token 不存在 → 直接跳转登录
        store.dispatch('auth/clearToken')
        router.push(`/${store.state.auth.tenantCode}/login`)
        return Promise.reject(err)
      }

      if (isRefreshing) {
        // 其他请求排队等待刷新完成
        return new Promise(resolve => {
          pendingQueue.push(newToken => {
            originalRequest.headers['Authorization'] = `Bearer ${newToken}`
            resolve(service(originalRequest))
          })
        })
      }

      isRefreshing = true
      originalRequest._retry = true

      try {
        const res = await service.post('/system/auth/refresh-token', null, {
          params: { refreshToken }
        })
        const newToken = res.data.accessToken
        // 更新存储
        const storage = localStorage.getItem('refreshToken') ? localStorage : sessionStorage
        storage.setItem('token', newToken)
        store.commit('auth/SET_TOKEN', newToken)
        // 唤醒排队请求
        pendingQueue.forEach(cb => cb(newToken))
        pendingQueue = []
        // 重发原请求
        originalRequest.headers['Authorization'] = `Bearer ${newToken}`
        return service(originalRequest)
      } catch (refreshErr) {
        // Refresh Token 过期 → 清除登录状态，跳转登录
        store.dispatch('auth/clearToken')
        router.push(`/${store.state.auth.tenantCode}/login`)
        Message.error('Login expired, please sign in again')
        return Promise.reject(refreshErr)
      } finally {
        isRefreshing = false
      }
    }

    // 非 401 错误
    const msg = err.response?.data?.message || '网络错误，请重试'
    Message.error(msg)
    return Promise.reject(err)
  }
)
```

### 6.4 登录成功后的完整流程

```js
// LoginForm.vue — handleLogin 方法
// REQ-001-shared §2.1 步骤 6：登录成功后先调用 get-permission-info，再跳转

async handleLogin() {
  // 1. 调用登录接口
  const loginRes = await loginApi({ username, password, tabType })
  const { accessToken, refreshToken, tenantId, tenantName } = loginRes.data

  // 2. 保存 Token（AC-001-pc-007）
  await this.$store.dispatch('auth/saveToken', {
    accessToken, refreshToken, tenantId, tenantName,
    remember: this.rememberPassword
  })

  // 3. 获取用户权限信息（写入 Vuex，后续页面使用）
  await this.$store.dispatch('auth/getPermissionInfo')

  // 4. 跳转首页（AC-001-pc-001 / AC-001-pc-002）
  this.$router.push(`/${this.tenantCode}/home`)
}
```

---

## 7. 表单校验

```js
// AC-001-pc-003
rules: {
  username: [{ required: true, message: 'required', trigger: 'submit' }],
  password: [{ required: true, message: 'required', trigger: 'submit' }]
}
```

- 触发时机：仅在点击 Login 时触发（`trigger: 'submit'`），不在 blur 触发
- 错误提示文案：`'required'`（中英文均同）
- 错误态：`el-form-item` 显示红色提示文字

**⚠️ 表单间距修正（来自实测问题）**：

`el-form-item` 默认保留 error 文字占位高度（`margin-bottom` 约 18~22px），与 UI Spec 的 `gap: 24px` 叠加后视觉间距过宽。必须通过以下方式消除 Element UI 默认的 error 占位：

```scss
// LoginForm 组件 scoped 样式
.login-form {
  // 使用 flex + gap 控制间距，由 UI Spec gap 24px 决定
  display: flex;
  flex-direction: column;
  gap: 24px;

  .el-form-item {
    margin-bottom: 0; // 覆盖 Element UI 默认 margin，由 flex gap 统一管理间距
  }

  // error 文字绝对定位，不占据文档流
  .el-form-item__error {
    position: absolute; // 不影响相邻 form-item 的位置
    padding-top: 2px;
  }
}
```

**禁止**使用 `el-form` 的 `label-position` + 默认 `margin-bottom` 来控制间距，否则与设计稿不符。

---

## 8. DS 与 Element UI 对应说明

> UI Spec 基于 Ant Design 描述，但项目使用 Element UI。以下为组件映射：

| UI Spec 组件 | Element UI 实现 | 差异处理 |
|------------|----------------|---------|
| `Input`（bordered） | `el-input` | 聚焦色通过覆盖 CSS 变量为 `#1890FF`；⚠️ **必须显式设置 `font-size: 14px`**，`el-input` 不会自动继承，否则使用浏览器默认 16px，placeholder 显示偏大 |
| `Input.Password` | `el-input` + `show-password` | 内置眼睛图标，可直接使用 |
| `Checkbox` | `el-checkbox` | 选中色通过 `--el-color-primary` 覆盖 |
| `Button type="primary"` | `el-button type="primary"` | 宽度需自定义 `240px` |
| Tab 下划线样式 | 自定义实现 | Element UI tabs 样式不完全匹配，需自定义下划线 |
| `Dropdown` | `el-dropdown` | 可直接使用 |
| `message` | `this.$message` | 可直接使用 |

---

## 9. 性能预算

| 指标 | 目标 | 说明 |
|-----|------|-----|
| 登录页首屏 FCP | ≤ 2s | 来自 REQ-001-pc §9.1 |
| 登录接口响应 P95 | ≤ 1s | 来自 REQ-001-pc §9.1 |
| 背景图加载 | 异步，不阻塞交互 | 先渲染卡片，背景图懒加载 |

---

## 10. 安全

- 密码输入框：`autocomplete="new-password"`（防浏览器自动填充）
- Token 不写入 URL
- 密码明文不持久化（提交后清空表单）

---

## 11. 国际化

- i18n 库：vue-i18n
- 文案来源：[UI-REQ-001-pc.md](../../ui/pc/UI-REQ-001-pc.md) §9
- 当前支持：`zh-CN`、`en`
- 默认语言：`en`
- 语言偏好存 `localStorage('lang')`

**关键文案 key**：

```js
// locales/en.js
module.exports = {
  login: {
    brandTitle: 'SMART CONSTRUCTION',
    brandSubtitle: 'BACK OFFICE',
    pageTitle: 'Account Login Page',
    tabMaincon: 'MAINCON',
    tabSubcon: 'SUBCON',
    accountPlaceholder: 'Please fill in account',
    passwordPlaceholder: 'Please fill in password',
    rememberPassword: 'Remember the password',
    loginBtn: 'Login',
    required: 'required',
    invalidUrl: 'Invalid access URL, please contact your administrator'
  }
}
```

---

## 12. 弱网与异常策略

| 场景 | 策略 |
|-----|------|
| 登录接口超时（> 10s） | Axios timeout 10s，超时后 Toast 提示，按钮恢复可点 |
| 网络断开 | catch 后 Toast 提示"网络错误，请重试" |
| 重复点击 Login | `:loading="isLoading"` + `:disabled="isLoading"` 禁止重复提交 |
| Access Token 过期（401） | 响应拦截器自动用 refreshToken 换新 token，无感重试原请求（见 §6.3） |
| Refresh Token 过期 | 跳转登录页，Toast 提示"Login expired, please sign in again"，清除所有本地存储 |

---

## 13. 测试要求

| 层级 | 框架 | 覆盖范围 |
|-----|------|---------|
| 单元 | Jest | Vuex auth module、tenantCode 解析逻辑、表单校验规则 |
| 集成 | Jest + Vue Test Utils | `<LoginForm>` 完整提交流程（Mock Axios） |
| E2E | Cypress（QA 维护） | 完整登录流程，见 QA Spec |

---

## 14. AC 覆盖检查表

| AC ID | 实现位置 | 状态 |
|------|---------|------|
| AC-001-pc-001 | `<LoginForm>` MAINCON 分支 → `auth/saveToken` → `auth/getPermissionInfo` → router.push | TODO |
| AC-001-pc-002 | `<LoginForm>` SUBCON 分支（调用 `subcontractor/login`） | TODO |
| AC-001-pc-003 | `<LoginForm>` rules（trigger: submit，必填） | TODO |
| AC-001-pc-004 | `<LoginPage>` tenantCode 校验 → `<InvalidTenantPage>` | TODO |
| AC-001-pc-005 | Axios 响应拦截器 → `Message.error`（非 401 错误） | TODO |
| AC-001-pc-006 | `el-input` `show-password` prop | TODO |
| AC-001-pc-007 | `<RememberPassword>` → `auth/saveToken(remember)` → localStorage / sessionStorage | TODO |
| AC-001-pc-008 | `<LanguageSwitcher>` → `this.$i18n.locale` | TODO |
| REQ-001-shared §3.2 | Axios 401 响应拦截器 → 自动 refresh → 无感重试 | TODO |
| REQ-001-shared §3.2 | Refresh Token 过期 → clearToken → 跳转登录页 | TODO |
| REQ-001-shared §3.1 | `auth/saveToken` 保存 accessToken + refreshToken + tenantId + tenantName | TODO |
| REQ-001-shared §3.3 | `auth/logout` 调用后端 API + 清除全部本地存储 | TODO |

---

## 15. 与后端的联调约定

- **联调前置**：REQ-001-shared 接口定义已确认
- **Mock**：基于接口约定本地 Mock，前端可独立开发
- **联调环境**：dev 环境
- **接口变更**：任何字段变更需先通知前端

---

## 16. 验收条件

- [ ] 所有 AC 在 §14 表中标记完成
- [ ] 单元 + 集成测试通过
- [ ] E2E 由 QA 跑通（不阻塞前端提测）
- [ ] 无 console 错误 / 警告
- [ ] 已部署到 dev 环境可联调
- [ ] 语言切换在中英文下均正常渲染
- [ ] 无效 tenantCode 显示错误整页（不显示登录卡片）

---

## 17. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.2.0 | 2026-05-03 | agent | 补全 REQ-001-shared 会话管理要求：saveToken 增加 refreshToken/tenantName；新增 401 自动刷新拦截器（§6.3）；新增登录后 get-permission-info 调用（§6.4）；补充 logout action；AC 覆盖表扩展 shared 层条目 |
| 0.1.3 | 2026-05-03 | agent | DS映射表补充 el-input 必须显式设置 font-size:14px，禁止继承浏览器默认 16px |
| 0.1.2 | 2026-05-03 | agent | 补充 LanguageSwitcher 默认态约束：trigger slot 只放图标，禁止在图标旁显示当前/备选语言文字 |
| 0.1.1 | 2026-05-03 | agent | 补充两条实测问题修正：LanguageSwitcher 必须用 el-dropdown；LoginForm 使用 flex gap + margin-bottom:0 消除 el-form-item error 占位 |
| 0.1.0 | 2026-05-03 | agent | 从 REQ-001-pc.md v0.2.0 重写，升级为新模板格式；仅覆盖登录页，后台框架拆至 FRONTEND-REQ-002-pc.md |
