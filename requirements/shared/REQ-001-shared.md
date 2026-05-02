# 登录和首页功能 — 跨端共享需求

> 本文档定义登录和首页功能的 **跨端共享** 部分：业务规则、API 接口、数据模型、验证逻辑、会话管理。
> 各端（APP / H5 / PC）的 UI 布局、交互和样式请参见各端独立需求文档。

## 基本信息

- **需求ID**: REQ-001
- **需求标题**: 登录和首页功能
- **产品**: SMART SITE SYSTEM
- **适用端**: ALL（APP / H5 / PC 共享）
- **优先级**: 高
- **状态**: 草稿

## 1. 系统架构

### 1.1 SaaS 多租户模式
- **架构模式**: SaaS（Software as a Service）多租户架构
- **租户隔离**: 系统支持不同的租户，每个租户拥有独立的数据和业务空间
- **租户标识**: 
  - 在登录时需要用户输入租户名称（Tenant Name / Tenant ID）
  - 租户名称用于路由到正确的后端数据库或数据分区
  - 在所有 API 请求中需要传递租户标识（通常在 Header 中，如 `X-Tenant-Id`）
- **数据隔离**: 
  - 不同租户的数据完全隔离
  - 用户只能访问所属租户下的数据和资源
  - 后端数据库采用租户分表或租户分库策略
- **业务影响**:
  - 所有业务操作都必须基于租户上下文
  - 人员、设备、项目等信息都属于特定租户
  - 权限管理和数据访问都需要进行租户验证

### 1.2 后端技术栈

> 详见 [background/tech-stack.md §4](../../background/tech-stack.md)

## 2. 业务规则

### 2.1 登录业务规则

#### 登录字段定义

| 字段 | 对应后端概念 | 必填 | 验证规则 |
|------|-------------|------|----------|
| 企业账号（Company Account） | Tenant Name / Tenant ID（租户） | ✅ | 长度 >= 2 个字符 |
| 用户账号（User Account） | username | ✅ | 长度 >= 3 个字符 |
| 密码（Password） | password | ✅ | 长度 >= 6 个字符 |

> **各端租户标识获取方式差异**：
> - **APP 端**: 用户在登录表单中手动输入 "Company Account"（企业账号）字段
> - **PC 端**: 通过 URL 路径自动解析租户标识（如 `https://smartsite.mcc.sg/mcc/login` → `tenantCode = mcc`），登录表单无需企业账号字段

#### 验证规则（前端，跨端统一）

| 条件 | 错误提示（English） | 错误提示（中文） |
|------|---------------------|-----------------|
| 企业账号为空 | Please enter company account | 请输入企业账号 |
| 企业账号过短（< 2） | Company account must be at least 2 characters | 企业账号长度至少2位 |
| 用户账号为空 | Please enter user name | 请输入用户名 |
| 用户账号过短（< 3） | User name must be at least 3 characters | 用户名长度至少3位 |
| 密码为空 | Please enter password | 请输入密码 |
| 密码过短（< 6） | Password must be at least 6 characters | 密码长度至少6位 |
| 租户不存在 | Tenant does not exist or is disabled | 租户不存在或已禁用 |
| 账号或密码错误 | Incorrect account or password, please try again | 账号或密码错误，请重试 |

#### 账号密码登录流程（跨端通用）

```
1. 用户输入企业账号、用户账号、密码
2. 前端执行字段验证（非空 + 长度校验）
3. 验证不通过 → 显示错误提示，阻止提交
4. 验证通过 → 调用登录 API（POST /system/auth/login）
5. 显示 Loading 状态，禁止重复提交
6. 登录成功：
   - 本地保存 Access Token、Refresh Token、Tenant ID、Tenant Name
   - 调用获取用户权限信息 API
   - 导航到首页
7. 登录失败：
   - 显示后端返回的错误信息
   - 清空密码字段，保留企业账号和用户账号
```

### 2.2 首页业务规则

#### 功能模块列表（跨端统一功能入口）

以下功能模块在所有端中均可出现，但各端可根据场景裁剪：

| 模块 | 英文标签 | 所属分组 | 说明 |
|------|----------|----------|------|
| 项目信息 | Project | 项目卡片 | 项目基本信息查看 |
| 分包商管理 | Subcontractor | 项目卡片 | 分包商列表与管理 |
| 人员列表 | Person List | 项目卡片 | 工地人员信息 |
| 设备列表 | Equipment List | 项目卡片 | 工地设备信息 |
| 监控摄像头 | CCTV | 项目卡片 | 监控视频查看 |
| 用户反馈 | User Feedback | 项目卡片 | 用户反馈提交与查看 |
| 文档管理 | Document | 项目卡片 | 文档上传与查看 |
| 设备调遣 | Equipment departure | 项目卡片 | 设备出场记录 |
| 耗材管理 | Consumables | 项目卡片 | 耗材使用记录 |
| 车辆配送预约 | Vehicle Delivery Booking | 独立模块卡片 | 车辆配送预约管理 |

#### 项目切换逻辑
- 用户可能关联多个项目
- 切换项目后，首页展示的功能模块和数据刷新为所选项目
- 选中的项目 ID 保存到本地存储
- 后续所有 API 请求在 Header 中传递 `Project-Id`

#### 首页数据加载流程（跨端通用）

```
1. 打开应用 → 检查本地是否有有效的 Token 和 Tenant ID
2. 无有效 Token → 跳转登录页
3. 有有效 Token → 调用 GET /system/auth/get-permission-info
4. 获取用户关联的项目列表（projectInfos）
5. 获取每个项目下用户有权限的功能模块
6. 渲染功能入口
7. 若用户无任何项目 → 显示空状态提示
```

### 2.3 多语言支持

- 支持语言：中文（简体）、English
- 默认语言：English
- 语言偏好保存到本地存储，下次打开应用时自动应用
- 切换语言后，所有文本即时刷新
- 各端均需实现多语言支持

## 3. 会话管理（跨端通用）

### 3.1 令牌和租户上下文存储

登录成功后，将后端返回的以下信息保存到本地安全存储：
- Access Token（访问令牌）
- Refresh Token（刷新令牌）
- Tenant ID（租户标识）
- Tenant Name（租户名称）

每次 API 请求都需要：
- 在 Authorization Header 中附加 token：`Authorization: Bearer <ACCESS_TOKEN>`
- 在请求 Header 中附加租户标识：`X-Tenant-Id: <TENANT_ID>` 或 `X-Tenant-Code: <TENANT_CODE>`
- 在请求 Header 中附加项目标识（如已选择）：`Project-Id: <PROJECT_ID>`
- 在请求 Header 中附加语言标识：`lang: en` 或 `lang: zh`

### 3.2 令牌过期处理

- API 返回 401（未授权），表示 Access Token 已过期
- 自动使用 Refresh Token 调用刷新接口获取新 Access Token
- Refresh Token 也过期 → 清除本地所有登录信息，跳转登录页
- 提示用户 "Login expired, please sign in again" / "登录已过期，请重新登录"

### 3.3 登出操作

点击"登出"后：
1. 调用后端登出 API（POST /system/auth/logout）
2. 清除本地所有存储：Access Token、Refresh Token、Tenant ID、Tenant Name、Project ID、语言缓存以外的临时数据
3. 导航到登录页

### 3.4 安全要求（跨端通用）

- 登录时使用 HTTPS 加密传输
- 密码字段不保存在日志或调试信息中
- 认证令牌使用 JWT（JSON Web Token）标准
  - Token 格式：`Authorization: Bearer <JWT_TOKEN>`
  - Token 包含用户标识、权限信息、过期时间等
  - 后端使用密钥对 Token 进行签名和验证
- 支持 Token 刷新机制：
  - 短期 Access Token（如 1 小时过期）用于 API 请求
  - 长期 Refresh Token（如 30 天过期）用于获取新的 Access Token

## 4. API 接口定义

> 以下 API 接口供所有端调用，接口定义统一。

### 4.1 账号密码登录

**接口位置**: AuthController.java

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/system/auth/login`
- 请求参数: `AuthLoginReqVO`

**请求格式**:
```json
{
  "username": "必填，4-16 位字母数字",
  "password": "必填，4-16 位",
  "captchaVerification": "可选，验证码（默认开启）",
  "socialType": "可选，社交类型",
  "socialCode": "可选，社交授权码",
  "socialState": "可选，state 参数"
}
```

**响应格式**: `CommonResult<AuthLoginRespVO>`
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "userId": 1024,
    "tenantId": "T001",
    "tenantName": "租户名称",
    "accessToken": "访问令牌",
    "refreshToken": "刷新令牌",
    "expiresTime": "2024-03-23 10:00:00",
    "lastLoginTime": "上次登录时间"
  }
}
```

**业务逻辑**:
- ✅ 验证租户是否存在且启用
- ✅ 校验验证码（如果开启）
- ✅ 验证账号密码（authenticate）
- ✅ 检查用户所属租户是否与登录租户匹配
- ✅ 检查用户类型是否匹配（PLATFORM）
- ✅ 检查用户状态是否启用
- ✅ 绑定社交用户（如果需要）
- ✅ 创建 Token 令牌（包含租户信息）
- ✅ 记录登录日志（包含租户信息）

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1002000000 | 账号或密码不正确 | Login failed, the account or password is incorrect |
| 1002000001 | 账号被禁用 | Login failed, accounts are disabled |
| 1002000002 | 租户不存在 | Tenant does not exist |
| 1002000003 | 租户已被禁用 | Tenant is disabled |
| 1002000004 | 验证码不正确 | The verification code is incorrect |
| 1002000005 | 用户不属于该租户 | User does not belong to this tenant |
| 1002000008 | 用户来源错误 | Login is not allowed due to the user source |

### 4.2 分包商登录

**接口位置**: AuthController.java

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/system/auth/subcontractor/login`
- 请求参数: 同上（`AuthLoginReqVO`）

**响应格式**: `CommonResult<AuthSubLoginRespVO>`

**特殊业务逻辑**:
- 先查询分包商是否存在（通过手机号）
- 用户类型必须是 SUBCONTRACTOR
- 错误码：1002029015 - 分包商信息不存在

### 4.3 Token 刷新

**接口位置**: AuthController.java

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/system/auth/refresh-token`
- 请求参数: `refreshToken`（query 参数，必填）

**响应格式**: `CommonResult<AuthLoginRespVO>`（新的访问令牌）

**错误码**:
- 1002000006 - Token 已经过期

### 4.4 获取用户权限信息

**接口位置**: AuthController.java

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/system/auth/get-permission-info`
- 请求参数:
  - `Project-Id`（header, 可选）
  - `lang`（header, 必填）- 语言切换

**响应格式**: `CommonResult<AuthPermissionInfoRespVO>`
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "user": {
      "id": 1024,
      "nickname": "用户昵称",
      "avatar": "头像 URL",
      "mobile": "手机号",
      "signatureUrl": "签名链接"
    },
    "roles": ["ROLE_ADMIN"],
    "permissions": ["system:user:query"],
    "projectInfos": [
      {
        "id": 290,
        "shortName": "RP101",
        "fullName": "雨林公园",
        "isChecked": true,
        "qcdmLogo": "logo URL"
      }
    ],
    "subcontractorId": "分包商 ID（如果是分包商用户）"
  }
}
```

**业务逻辑**:
- 获取当前登录用户信息
- 获取用户角色列表
- 根据用户类型获取项目权限：
  - 平台用户：获取角色关联的项目
  - 分包商用户：获取分包商关联的项目
- 获取菜单权限（按项目过滤）
- 支持总工地特殊处理（PROJECT_ID_MAIN）

### 4.5 登出

**接口位置**: AuthController.java

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/system/auth/logout`
- 请求参数: 从 Header 获取 Authorization token

**业务逻辑**:
- 删除访问令牌
- 移除客户端 ID
- 记录登出日志

## 5. 验收标准（跨端通用）

### 登录业务逻辑
- [ ] 企业账号、用户账号、密码为空时，前端阻止提交并提示
- [ ] 字段长度不满足要求时，前端阻止提交并提示
- [ ] 调用登录 API 成功后，本地保存 Token 和租户信息
- [ ] 调用登录 API 失败后，显示后端返回的错误信息
- [ ] 登录失败后清空密码字段，保留企业账号和用户账号
- [ ] 登录过程中 Loading 状态禁止重复提交

### 首页业务逻辑
- [ ] 登录成功后调用权限信息 API 获取项目列表和功能模块
- [ ] 根据用户权限动态展示功能模块（无权限的不显示）
- [ ] 无任何项目权限时显示空状态提示
- [ ] 切换项目后首页内容刷新为所选项目
- [ ] 选中的项目 ID 保存到本地，后续 API 请求正确传递 `Project-Id`

### 会话管理
- [ ] 打开应用时，有效 Token 自动进入首页，无效 Token 跳转登录
- [ ] API 返回 401 时自动尝试刷新 Token
- [ ] Refresh Token 过期时跳转登录页
- [ ] 所有 API 请求正确传递 `Authorization`、`X-Tenant-Id`、`Project-Id`、`lang` Header
- [ ] 登出时清除所有本地登录信息并返回登录页

### 多语言
- [ ] 支持中文 / English 切换
- [ ] 语言偏好保存到本地，下次打开自动应用
- [ ] 切换语言后所有文本即时刷新

## 6. 各端需求文档索引

| 端 | 需求文档 | 说明 |
|----|---------|------|
| APP 移动端 | [REQ-001-app.md](../app/REQ-001-app.md) | UNIAPP + Vue 2，iOS & Android |
| H5 移动端 | REQ-001-h5.md（待编写） | 浏览器/微信内嵌 H5 |
| PC 管理端 | [REQ-001-pc.md](../pc/REQ-001-pc.md) | 桌面浏览器 Web 管理后台 |
