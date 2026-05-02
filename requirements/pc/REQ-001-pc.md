# PC 管理端 — 登录页面

> **端**: PC 管理端（桌面浏览器 Web 管理后台）
> **共享需求**: [REQ-001-shared.md](../shared/REQ-001-shared.md)（业务规则、API 接口、会话管理）
> **本文档仅包含**: PC 端登录页面的布局、交互方式和样式
> **管理后台框架**: 见 [REQ-002-pc.md](./REQ-002-pc.md)（Header / Sidebar / Main Content）

## 基本信息

- **需求ID**: REQ-001-pc
- **需求标题**: PC 管理端登录页面
- **产品**: SMART SITE SYSTEM
- **品牌名称**: SMART CONSTRUCTION（PC 端品牌标识）
- **平台**: PC 管理端（桌面浏览器）
- **技术框架**: Vue 2 + Element UI
- **优先级**: 高
- **状态**: 草稿

## 技术说明

> PC 端技术栈详见 [background/tech-stack.md §1](../../background/tech-stack.md)
> 后端技术栈详见 [background/tech-stack.md §4](../../background/tech-stack.md)

## 需求描述

### 背景与目标

PC 管理端（Back Office）面向系统管理员、项目经理和管理人员，提供完整的后台管理功能。用户通过桌面浏览器登录后，可查看和管理项目信息、人员、设备、考勤等各项业务数据。本文档定义 PC 端的登录页面和登录后的管理后台页面框架（布局、导航、通用组件），具体业务页面内容在后续需求中细化。

### 用户故事

```
作为 项目经理 / 系统管理员
我想要 在桌面浏览器上登录 Smart Construction Back Office 管理后台
以便 我可以管理和查看项目数据、人员信息、设备状态等各项业务
```

## 功能需求

### 1. 登录页面

#### 1.1 页面整体布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                        (全屏城市夜景背景图)                                   │
│                                                                             │
│                    ┌─────────────────────────────┐                          │
│                    │                             │                          │
│                    │  SMART CONSTRUCTION         │                          │
│                    │  BACK OFFICE                │                          │
│                    │                             │                          │
│                    │  Account Login Page    文A   │                          │
│                    │                             │                          │
│                    │  MAINCON     SUBCON         │                          │
│                    │  ─────────                  │                          │
│                    │                             │                          │
│                    │  ┌─────────────────────┐    │                          │
│                    │  │ Please fill in account│   │                          │
│                    │  └─────────────────────┘    │                          │
│                    │                             │                          │
│                    │  ┌─────────────────────┐    │                          │
│                    │  │ Please fill in password 👁│  │                          │
│                    │  └─────────────────────┘    │                          │
│                    │                             │                          │
│                    │  ☑ Remember the password    │                          │
│                    │                             │                          │
│                    │  ┌─────────────────────┐    │                          │
│                    │  │       Login         │    │                          │
│                    │  └─────────────────────┘    │                          │
│                    │                             │                          │
│                    └─────────────────────────────┘                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 1.2 背景

- **背景图**: 全屏城市天际线夜景照片（新加坡城市夜景风格）
- **图片处理**: 轻微暗色滤镜叠加，使白色登录卡片更突出
- **填充方式**: `background-size: cover; background-position: center;`
- **备用背景色**: 深蓝灰色渐变（#1a2332 → #2c3e50），当背景图加载失败时使用

#### 1.3 登录卡片

| 属性 | 规格 |
|------|------|
| 位置 | 页面水平垂直居中 |
| 宽度 | 400-440px |
| 背景 | #FFFFFF |
| 圆角 | 16-20px |
| 阴影 | 0 8px 32px rgba(0,0,0,0.12) |
| 内边距 | 上 40px，左右 40px，下 36px |

#### 1.4 品牌标识区

| 属性 | 规格 |
|------|------|
| 主标题 | "SMART CONSTRUCTION"，20-22px，#212121，粗体（700），大写字母 |
| 副标题 | "BACK OFFICE"，14-16px，#333333，粗体（700），大写字母 |
| 主副标题间距 | 4px |
| 下边距 | 24px（与 Account Login Page 行） |

#### 1.5 Account Login Page 行

| 属性 | 规格 |
|------|------|
| 左侧文字 | "Account Login Page"，14px，#333333，粗体（600） |
| 右侧图标 | 语言切换图标（文A 样式），20px，#666666 |
| 布局 | 左右两端对齐（space-between） |
| 下边距 | 24px（与 Tab 栏） |
| 语言切换交互 | 点击图标弹出下拉菜单，可选 English / 中文（简体） |

#### 1.6 登录角色 Tab 切换

| 属性 | 规格 |
|------|------|
| Tab 数量 | 2 个：MAINCON / SUBCON |
| 布局 | 水平排列，左对齐 |
| Tab 间距 | 32px |
| 选中 Tab | 文字 #4A90D9（蓝色），14px，粗体（600），大写字母 |
| 选中下划线 | 2px 实线，#4A90D9，宽度等于文字宽度，底部紧贴 |
| 未选中 Tab | 文字 #999999，14px，常规（400），大写字母 |
| Tab 栏底部线 | 1px #E8E8E8，全宽 |
| 下边距 | 28px（与表单区域） |

**Tab 说明**：
- **MAINCON（总承包商）**: 对应 `/system/auth/login` 接口（标准登录）
- **SUBCON（分包商）**: 对应 `/system/auth/subcontractor/login` 接口（分包商登录）
- 两个 Tab 共享相同的表单布局，仅调用不同的后端接口

#### 1.7 登录表单

**租户（企业账号）识别方式**：

PC 端的企业账号（租户标识）**通过 URL 路径参数自动获取**，用户无需手动输入。

- **URL 格式**: `https://smartsite.mcc.sg/{tenantCode}/login`
- **示例**: `https://smartsite.mcc.sg/mcc/login` → 租户标识为 `mcc`
- **前端逻辑**:
  1. 页面加载时从 URL 路径中解析 `tenantCode`（如 `/mcc/login` → `mcc`）
  2. 将 `tenantCode` 保存到前端状态（Vuex / 内存变量）
  3. 登录请求时，自动将 `tenantCode` 作为请求参数或 Header（`X-Tenant-Code: mcc`）传递给后端
  4. 登录成功后，后端返回的 `tenantId` 保存到本地存储，后续 API 请求携带 `X-Tenant-Id`
- **异常处理**:
  - URL 中无 `tenantCode` → 显示错误页面 "Invalid access URL, please contact your administrator"
  - `tenantCode` 对应的租户不存在或已禁用 → 登录时后端返回错误，前端显示提示

> **与 APP 端的差异**: APP 端通过表单字段 "Company Account" 手动输入企业账号；PC 端通过 URL 路径自动识别，登录表单仅需账号 + 密码两个字段。

**MAINCON Tab 下的表单字段**：

| 字段 | Placeholder | 类型 | 说明 |
|------|-------------|------|------|
| 账号 | "Please fill in account" | text | 用户账号（username） |
| 密码 | "Please fill in password" | password | 用户密码 |

**SUBCON Tab 下的表单字段**：

| 字段 | Placeholder | 类型 | 说明 |
|------|-------------|------|------|
| 账号 | "Please fill in account" | text | 分包商账号 |
| 密码 | "Please fill in password" | password | 分包商密码 |

**输入框样式**：

| 属性 | 规格 |
|------|------|
| 宽度 | 100%（卡片内宽） |
| 高度 | 44-48px |
| 背景 | #FFFFFF |
| 边框 | 1px #D9D9D9（默认）/ 1px #4A90D9（聚焦）/ 1px #FF4D4F（错误） |
| 圆角 | 6-8px |
| 内边距 | 水平 12px |
| 文字 | 14px，#333333 |
| Placeholder | 14px，#BDBDBD |
| 字段间距 | 20px（账号与密码之间） |

**密码输入框附加**：
- 右侧显示 **密码可见/隐藏切换图标**（👁 眼睛图标）
- 默认隐藏密码（掩码显示 ••••••）
- 点击眼睛图标切换明文/掩码显示
- 图标样式：20px，#BDBDBD（默认）/ #666666（hover）

**错误状态**：

| 属性 | 规格 |
|------|------|
| 边框 | 1px #FF4D4F（红色） |
| 错误提示 | 输入框下方 4px，红色文字 "required"，12px，#FF4D4F |
| 触发条件 | 点击 Login 时字段为空 |

#### 1.8 记住密码

| 属性 | 规格 |
|------|------|
| 位置 | 密码输入框下方，间距 16px |
| 布局 | 水平排列，Checkbox + 文字 |
| Checkbox | 16×16px，选中时背景 #4A90D9，白色勾选图标 |
| 文字 | "Remember the password"，14px，#4A90D9 |
| Checkbox 与文字间距 | 8px |
| 默认状态 | 选中（checked） |
| 功能 | 选中时本地存储用户名，下次自动填充；密码以安全方式存储或仅记住账号 |

#### 1.9 登录按钮

| 属性 | 规格 |
|------|------|
| 文字 | "Login"，白色，16px，粗体 |
| 背景 | #4A90D9（蓝色）/ hover 加深为 #357ABD |
| 宽度 | 100%（卡片内宽） |
| 高度 | 44-48px |
| 圆角 | 6-8px |
| 上边距 | 24px（与 Remember the password） |
| cursor | pointer |

**登录按钮状态**：

| 状态 | 样式 |
|------|------|
| 默认 | 背景 #4A90D9，白色文字 |
| Hover | 背景 #357ABD（加深） |
| 按下 | 背景 #2A6FA8（更深） |
| Loading | 背景 #4A90D9，白色旋转加载图标，禁止点击 |
| 禁用 | 背景 #4A90D9 透明度 0.6，cursor: not-allowed |

#### 1.10 登录流程

**MAINCON 登录**：
1. 页面加载 → 从 URL 路径解析 `tenantCode`（如 `/mcc/login` → `mcc`）
2. 若 URL 中无有效 `tenantCode` → 显示错误页面，阻止登录
3. 用户选择 MAINCON Tab（默认选中）
4. 输入账号（username）和密码
5. 点击 "Login"
6. 前端验证：账号和密码不能为空
7. 验证不通过 → 对应输入框标红 + 显示 "required"
8. 验证通过 → 调用 `POST /system/auth/login`，请求中附带 `tenantCode`
9. 登录成功 → 保存 Token 和 Tenant ID，跳转管理后台首页
10. 登录失败 → 显示错误提示

**SUBCON 登录**：
1. 页面加载 → 同上，从 URL 路径解析 `tenantCode`
2. 用户切换到 SUBCON Tab
3. 输入账号和密码
4. 点击 "Login"
5. 前端验证：账号和密码不能为空
6. 验证通过 → 调用 `POST /system/auth/subcontractor/login`，请求中附带 `tenantCode`
7. 登录成功 → 保存 Token 和 Tenant ID，跳转管理后台首页
8. 登录失败 → 显示错误提示

---

## API 接口定义

> 详见 [REQ-001-shared.md](../shared/REQ-001-shared.md) — 第 4 节「API 接口定义」
>
> 登录相关接口：
> - MAINCON 登录: `POST /system/auth/login`
> - SUBCON 登录: `POST /system/auth/subcontractor/login`

---

## 样式要求汇总

| 属性 | 值 |
|------|-----|
| 背景图 | 城市天际线夜景（新加坡风格），暗色叠加 |
| 登录卡片 | 白色 #FFFFFF，圆角 16-20px，阴影 |
| 主品牌色 | #4A90D9（蓝色，用于选中状态、按钮、链接） |
| 错误色 | #FF4D4F（红色，用于校验失败提示） |
| 输入框 | 有边框式（bordered），圆角 6-8px |
| 按钮 | 蓝色背景 #4A90D9，圆角 6-8px |

---

## 验收标准

- [ ] 页面全屏显示城市夜景背景图
- [ ] 页面中央显示白色圆角登录卡片
- [ ] 卡片顶部显示 "SMART CONSTRUCTION" + "BACK OFFICE" 品牌标识
- [ ] 显示 "Account Login Page" 文字和右侧语言切换图标（文A）
- [ ] 显示 MAINCON / SUBCON 两个 Tab 切换
- [ ] 默认选中 MAINCON Tab，蓝色文字 + 蓝色下划线
- [ ] 切换到 SUBCON Tab 时样式正确切换
- [ ] 显示账号输入框，Placeholder "Please fill in account"
- [ ] 显示密码输入框，Placeholder "Please fill in password"
- [ ] 密码输入框右侧显示眼睛图标，点击可切换明文/掩码
- [ ] 输入框为有边框样式（bordered），非下划线样式
- [ ] 显示 "Remember the password" 复选框，默认选中
- [ ] 显示 "Login" 按钮，蓝色背景全宽
- [ ] 账号或密码为空时点击 Login，输入框边框变红 + 显示 "required"
- [ ] 页面加载时从 URL 路径正确解析 tenantCode（如 `/mcc/login` → `mcc`）
- [ ] URL 中无有效 tenantCode 时显示错误页面
- [ ] MAINCON 登录请求中正确附带 tenantCode，调用 `/system/auth/login`
- [ ] SUBCON 登录请求中正确附带 tenantCode，调用 `/system/auth/subcontractor/login`
- [ ] 登录成功后保存 Token 和 Tenant ID 并跳转管理后台首页
- [ ] 登录失败显示错误提示
- [ ] 语言切换功能正常

---

## 相关需求

- [REQ-001-shared.md](../shared/REQ-001-shared.md)：登录和首页跨端共享需求（业务规则、API、会话管理）
- [REQ-002-pc.md](./REQ-002-pc.md)：PC 端管理后台框架（Header / Sidebar / Main Content）
- [REQ-001-app.md](../app/REQ-001-app.md)：APP 移动端登录和首页

## 备注

- PC 端品牌标识为 "SMART CONSTRUCTION" + "BACK OFFICE"，与 APP 端的 "NOVASYNC" + "SMART SITE" 品牌标识不同
- PC 端企业账号（租户标识）通过 URL 路径自动识别（如 `https://smartsite.mcc.sg/mcc/login`），无需用户手动输入；APP 端通过表单字段 "Company Account" 手动输入
- PC 端登录表单仅需账号 + 密码两个字段，APP 端需企业账号 + 用户账号 + 密码三个字段
- PC 端登录按钮文案为 "Login"，APP 端为 "Sign In"
- PC 端输入框为有边框样式（bordered），APP 端为下划线样式（underline）
- PC 端主色调为 #4A90D9（蓝色），APP 端为 #6892FF（蓝色），两端色调有差异
- PC 端新增 "Remember the password" 功能，APP 端无此功能
- PC 端新增 MAINCON / SUBCON Tab 切换，APP 端仅有标准登录 + 扫码登录

