---
doc_type: qa_spec
req_id: REQ-001-pc
version: 0.2.0
status: draft
generated_from: REQ-001-pc.md@0.2.0
frontend_spec_ref: FRONTEND-REQ-001-pc.md@0.2.0
generated_at: 2026-05-03
owner: ""
---

# QA 测试说明：PC 管理端 — 登录页面

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **测试范围边界**：本文档仅覆盖 REQ-001-pc（登录页面）。管理后台框架测试见 QA-REQ-002-pc.md。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-001-pc @ v0.2.0 |
| 前端说明 | FRONTEND-REQ-001-pc @ v0.2.0 |
| 共享规则 | REQ-001-shared |
| 覆盖 Story | US-001、US-002、US-003 |
| 覆盖 AC | AC-001-pc-001 ~ AC-001-pc-008 + REQ-001-shared §3 会话管理 |
| 上次同步时间 | 2026-05-03 |

---

## 1. 测试范围

| 类型 | 内容 |
|-----|-----|
| ✅ 纳入 | 登录页渲染、MAINCON/SUBCON 切换、表单校验、登录成功/失败流程、记住密码、无效租户、语言切换、Token 自动刷新、Refresh Token 过期处理、有效 Token 自动进入首页 |
| ❌ 排除 | 登录后页面（后台框架，属 REQ-002-pc）、接口内部逻辑（后端测试范围）、登出交互（属 REQ-002-pc，仅测试 storage 清除） |

---

## 2. 测试环境

| 项 | 值 |
|---|---|
| 环境 | dev |
| 浏览器 | Chrome latest（主）、Firefox latest（次）、Edge latest（次） |
| E2E 框架 | Cypress |
| 分辨率 | 1440×900、1920×1080 |
| 测试账号 | dev 环境 MAINCON/SUBCON 各一个有效账号 + 一个无效账号 |

---

## 3. 测试用例

### AC-001-pc-001：MAINCON 登录成功

| 字段 | 值 |
|-----|---|
| AC ID | AC-001-pc-001 |
| 优先级 | P0 |
| 类型 | E2E |

**前置条件**：访问 `/:tenantCode/login`，tenantCode 有效，Tab 为 MAINCON（默认）

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 输入有效账号 + 密码 | 输入框正常展示内容 |
| 2 | 点击 Login | 按钮进入 Loading 态，不可重复点击 |
| 3 | 接口返回 200 | 跳转至 `/:tenantCode/home` |
| 4 | 检查 localStorage | `token` 和 `tenantId` 已写入（默认记住密码=勾选） |

**失败条件**：未跳转、token 未写入、按钮可重复点击

---

### AC-001-pc-002：SUBCON 登录成功

| 字段 | 值 |
|-----|---|
| AC ID | AC-001-pc-002 |
| 优先级 | P0 |
| 类型 | E2E |

**前置条件**：同上，切换 Tab 至 SUBCON

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 点击 SUBCON Tab | Tab 切换，SUBCON 高亮 |
| 2 | 输入有效账号 + 密码 | — |
| 3 | 点击 Login | 调用 `/system/auth/subcontractor/login`（非 MAINCON 接口） |
| 4 | 接口返回 200 | 跳转至 `/:tenantCode/home` |

**验证点**：Network 面板确认请求 URL 为 subcontractor 接口

---

### AC-001-pc-003：前端表单必填校验

| 字段 | 值 |
|-----|---|
| AC ID | AC-001-pc-003 |
| 优先级 | P1 |
| 类型 | 功能 |

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 账号留空，密码留空，点击 Login | 账号、密码字段下均出现 "required" 提示，不发起请求 |
| 2 | 填写账号，密码留空，点击 Login | 仅密码字段出现 "required" 提示 |
| 3 | 不点 Login，仅在账号框 blur | 不出现任何校验提示（trigger 为 submit） |
| 4 | 填写账号和密码后点击 Login | 校验通过，发起请求 |

---

### AC-001-pc-004：无效租户 URL

| 字段 | 值 |
|-----|---|
| AC ID | AC-001-pc-004 |
| 优先级 | P1 |
| 类型 | 功能 |

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 访问 `/login`（无 tenantCode） | 显示 InvalidTenantPage 整页错误，不显示登录卡片 |
| 2 | 访问 `//login` | 同上 |
| 3 | 访问 `/:validTenantCode/login` | 正常显示登录卡片 |

---

### AC-001-pc-005：登录失败错误提示

| 字段 | 值 |
|-----|---|
| AC ID | AC-001-pc-005 |
| 优先级 | P0 |
| 类型 | 功能 |

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 输入错误密码，点击 Login | 接口返回 4xx，Toast 弹出服务端 message 内容 |
| 2 | 断开网络，点击 Login | Toast 显示"网络错误，请重试" |
| 3 | Toast 显示后 | 按钮恢复可点状态 |

---

### AC-001-pc-006：密码显示/隐藏

| 字段 | 值 |
|-----|---|
| AC ID | AC-001-pc-006 |
| 优先级 | P2 |
| 类型 | UI |

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 在密码框中输入内容，默认状态 | 显示为黑点（隐藏） |
| 2 | 点击眼睛图标 | 密码明文可见，图标变为眼睛划线状 |
| 3 | 再次点击 | 恢复隐藏 |

---

### AC-001-pc-007：记住密码

| 字段 | 值 |
|-----|---|
| AC ID | AC-001-pc-007 |
| 优先级 | P1 |
| 类型 | 功能 |

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 默认进入页面 | "记住密码" Checkbox 已勾选 |
| 2 | 勾选状态下登录成功 | `localStorage` 中有 `token` |
| 3 | 取消勾选后登录成功 | `sessionStorage` 中有 `token`，`localStorage` 无 |
| 4 | 取消勾选登录，关闭 Tab 重开 | 需重新登录（session 已清） |

---

### AC-001-pc-008：语言切换

| 字段 | 值 |
|-----|---|
| AC ID | AC-001-pc-008 |
| 优先级 | P2 |
| 类型 | 功能 |

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 点击语言切换图标 | 出现下拉菜单（含语言选项） |
| 2 | 切换为中文 | 页面文案切换为中文，无刷新 |
| 3 | 刷新页面 | 语言仍为中文（localStorage 持久化） |
| 4 | 切回英文 | 页面文案切换为英文 |

---

### SESSION-001：有效 Token 自动进入首页（REQ-001-shared §2.2）

| 字段 | 值 |
|-----|---|
| 来源 | REQ-001-shared §2.2 首页数据加载流程 |
| 优先级 | P0 |
| 类型 | E2E |

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 登录成功（记住密码=勾选），关闭 Tab | localStorage 中有 token |
| 2 | 重新打开 `/:tenantCode/login` | 自动跳转至 `/:tenantCode/home`，不停留在登录页 |
| 3 | sessionStorage 中有 token（未记住），重开 Tab | 停留在登录页（session 已清） |

---

### SESSION-002：Access Token 自动刷新（REQ-001-shared §3.2）

| 字段 | 值 |
|-----|---|
| 来源 | REQ-001-shared §3.2 令牌过期处理 |
| 优先级 | P1 |
| 类型 | 集成（Mock） |

**前置条件**：已登录，Mock 服务器第一次返回 401，第二次返回正常响应。

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 触发任意 API 请求，Mock 返回 401 | 前端自动发起 `POST /system/auth/refresh-token` |
| 2 | refresh 成功 | 原请求自动重发，用户无感知（无 Toast、无跳转） |
| 3 | 期间多个请求同时遇到 401 | 只发出一次 refresh 请求，其他请求排队等待，不重复刷新 |
| 4 | localStorage 的 token 已更新为新 token | ✅ |

---

### SESSION-003：Refresh Token 过期（REQ-001-shared §3.2）

| 字段 | 值 |
|-----|---|
| 来源 | REQ-001-shared §3.2 令牌过期处理 |
| 优先级 | P0 |
| 类型 | 集成（Mock） |

**前置条件**：已登录，Mock 服务器对 refresh-token 接口也返回 401/过期错误。

| # | 步骤 | 期望结果 |
|---|-----|---------|
| 1 | 触发 API 请求，收到 401 | 前端发起 refresh-token |
| 2 | refresh-token 接口也返回失败 | 清除 localStorage / sessionStorage 中所有 token |
| 3 | 跳转至登录页 | ✅ |
| 4 | Toast 显示 | "Login expired, please sign in again" |

---

## 4. 边界值测试

| 测试项 | 输入值 | 期望结果 |
|-------|-------|---------|
| 账号最大长度 | 256 字符 | 界面无崩溃，请求正常发出（服务端校验） |
| 密码特殊字符 | `"<script>alert(1)</script>"` | 正常作为字符串发送，不执行脚本 |
| 账号含空格 | `" admin "` | 不做前端 trim，原样传递（待 REQ 确认是否需 trim） |
| 并发点击 | 快速连点 Login 3 次 | 只发出 1 次请求（disabled 生效） |

---

## 5. 安全测试

| 测试项 | 方法 | 期望结果 |
|-------|-----|---------|
| 密码明文 XSS | 密码输入 `<img src=x onerror=alert(1)>` | 正常渲染为文本，不执行脚本 |
| Token 泄露 | 登录成功后查看 URL | URL 中无 token |
| 浏览器自动填充 | 打开登录页 | 密码框不被浏览器自动填入（autocomplete="new-password"） |
| 重放攻击（人工） | 复制登录请求重放 | 由后端处理（非前端范围） |

---

## 6. 回归测试矩阵

当以下文件变更时，需重新执行对应用例：

| 变更范围 | 必须回归的用例 |
|---------|-------------|
| `LoginForm.vue` | AC-001, AC-002, AC-003, AC-005 |
| `LoginPage.vue` | AC-004 |
| `store/modules/auth.js` | AC-001, AC-002, AC-007, SESSION-001, SESSION-002, SESSION-003 |
| `utils/request.js` | AC-005, SESSION-002, SESSION-003 |
| `LanguageSwitcher.vue` | AC-008 |
| i18n 文案 | AC-008 + 视觉回归（全页文案） |

---

## 7. 自动化测试计划

### 7.1 Cypress E2E 文件规划

```
cypress/
  e2e/
    login/
      maincon-login.cy.js       // AC-001
      subcon-login.cy.js        // AC-002
      form-validation.cy.js     // AC-003
      invalid-tenant.cy.js      // AC-004
      login-error.cy.js         // AC-005
      password-toggle.cy.js     // AC-006
      remember-password.cy.js   // AC-007
      language-switch.cy.js     // AC-008
    session/
      auto-enter-home.cy.js     // SESSION-001
      token-refresh.cy.js       // SESSION-002（需 Mock Server）
      refresh-token-expired.cy.js // SESSION-003（需 Mock Server）
```

### 7.2 CI 执行策略

| 触发时机 | 执行范围 |
|---------|---------|
| PR 合并到 dev | 全量 E2E（login/ 目录） |
| 仅文案 / 样式变更 | P2 用例可跳过，P0/P1 必跑 |

---

## 8. 已知风险 / 待确认项

| ID | 描述 | 状态 |
|----|-----|------|
| QA-OQ-001 | 账号是否需要前端 trim 空格？ | ❓ 待确认 |
| QA-OQ-002 | 无效 tenantCode 的具体判定逻辑（空字符串 vs URL格式错误 vs 未注册租户）？ | ❓ 待确认 |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.2.0 | 2026-05-03 | agent | 对齐 REQ-001-shared 会话管理：新增 SESSION-001（有效Token自动进入首页）、SESSION-002（Access Token自动刷新）、SESSION-003（Refresh Token过期处理）；扩展测试范围、回归矩阵、Cypress文件规划 |
| 0.1.0 | 2026-05-03 | agent | 从 REQ-001-pc.md v0.2.0 重写，升级为新模板格式；仅覆盖登录页 |
