# 后端开发说明文档 — REQ-001 多租户登录与会话管理

> **来源需求**: [REQ-001-shared.md](../../requirements/shared/REQ-001-shared.md) + [REQ-001-pc.md](../../requirements/pc/REQ-001-pc.md) + [REQ-001-app.md](../../requirements/app/REQ-001-app.md)
> **产品**: SMART SITE SYSTEM
> **服务模块**: `system-service`
> **生成日期**: 2026-04-08

---

## 1. 功能概述

实现 SaaS 多租户架构下的身份认证与会话管理，支持总包商 (MAINCON) 和分包商 (SUBCON) 两种用户类型分别登录：

| 功能 | 说明 |
|------|------|
| 多租户登录 | 根据 `tenantCode` 路由到对应租户，返回 JWT Token |
| 分包商登录 | 独立子路径 `/system/auth/subcontractor/login` |
| Token 刷新 | Access Token（1h）到期时使用 Refresh Token（30d）换新 |
| 获取权限信息 | 返回用户信息 + 角色 + 权限码 + 项目列表 |
| 登出 | 清除 Redis Session |

---

## 2. 业务逻辑

### 2.1 登录流程

```
1. 客户端携带 username / password / tenantCode 请求登录
2. system-service 查询 tenant 表，校验 tenantCode 是否存在且启用
3. 从对应租户 schema 查询用户（或通过 X-Tenant-Id 动态路由）
4. 校验密码（BCrypt 加密）
5. 生成 Access Token（JWT，含 userId、tenantId、roleIds，TTL=1h）
6. 生成 Refresh Token（随机 UUID，存入 Redis，Key=refresh:token:{token}，TTL=30d）
7. 返回 Token 对及用户基础信息
```

### 2.2 多租户识别

| 场景 | 租户来源 |
|------|---------|
| PC 端 | URL path 第一段（如 `/mcc/login` → tenantCode=`mcc`）前端从 URL 解析后放入请求体 |
| APP 端 | 登录表单中用户手动输入 tenantCode 字段 |
| 每次 API 请求 | 请求 Header `X-Tenant-Id`（登录成功后由 API Gateway 或前端自动附加）|

### 2.3 项目上下文

- 用户可关联多个项目，登录后返回 `projectInfos` 列表
- 前端选定项目后，后续所有请求 Header 携带 `Project-Id`
- 后端 Filter/Interceptor 从 Header 读取 `Project-Id` 注入线程上下文

### 2.4 Token 刷新

```
1. 前端检测 Access Token 即将过期或收到 401
2. 请求 POST /system/auth/refresh-token（携带 refreshToken）
3. 后端从 Redis 校验 refreshToken 是否存在且未过期
4. 重新签发新 Access Token，更新 Redis 中 refreshToken 的 TTL（滑动续期）
5. 返回新 Access Token
```

### 2.5 权限信息接口

`GET /system/auth/get-permission-info` 返回：
- 当前登录用户信息（userId, username, avatar, userType）
- 角色列表（roleId, roleName, roleCode）
- 权限码集合（如 `equipment:refueling:query`）
- 可访问项目列表（projectId, projectName, projectCode）

---

## 3. 数据模型

### 3.1 tenant 表

```sql
CREATE TABLE `system_tenant` (
  `id`          BIGINT PRIMARY KEY AUTO_INCREMENT,
  `tenant_code` VARCHAR(64) NOT NULL COMMENT '租户标识，唯一',
  `tenant_name` VARCHAR(100) NOT NULL COMMENT '租户名称',
  `status`      TINYINT(1) NOT NULL DEFAULT 1 COMMENT '0-禁用 1-启用',
  `created_at`  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY `uk_code` (`tenant_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='租户表';
```

### 3.2 system_user 表（多租户数据隔离）

```sql
CREATE TABLE `system_user` (
  `id`          BIGINT PRIMARY KEY AUTO_INCREMENT,
  `tenant_id`   BIGINT NOT NULL COMMENT '租户 ID',
  `username`    VARCHAR(64) NOT NULL,
  `password`    VARCHAR(128) NOT NULL COMMENT 'BCrypt 加密',
  `user_type`   VARCHAR(20) NOT NULL COMMENT 'MAINCON / SUBCON',
  `real_name`   VARCHAR(64),
  `avatar`      VARCHAR(500),
  `status`      TINYINT(1) NOT NULL DEFAULT 1,
  `created_at`  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY `uk_tenant_username` (`tenant_id`, `username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.3 Redis Key 规范

| Key | TTL | 说明 |
|-----|-----|------|
| `auth:refresh:{token}` | 30d | Refresh Token → userId, tenantId |
| `auth:permission:{userId}` | 30min | 权限信息缓存（可选） |

---

## 4. API 设计

### 4.1 接口列表

- [x] `POST /system/auth/login` — 总包商登录
- [x] `POST /system/auth/subcontractor/login` — 分包商登录
- [x] `GET /system/auth/get-permission-info` — 获取权限信息
- [x] `POST /system/auth/refresh-token` — 刷新 Token
- [x] `POST /system/auth/logout` — 登出

### 4.2 接口规范

#### POST /system/auth/login

**Request Body**:
```json
{
  "username": "string",
  "password": "string",
  "tenantCode": "string"
}
```

**Response**:
```json
{
  "code": 0,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "accessTokenExpireIn": 3600,
    "userId": 1001,
    "username": "admin",
    "userType": "MAINCON",
    "tenantId": 1
  }
}
```

**错误码**:
| Code | 说明 |
|------|------|
| 1001001001 | 用户名或密码错误 |
| 1001001002 | 租户不存在或已禁用 |
| 1001001003 | 账号已被禁用 |

#### POST /system/auth/subcontractor/login

Request/Response 结构同上，但只允许 `userType=SUBCON` 的账号登录。

#### GET /system/auth/get-permission-info

**Headers**: `Authorization: Bearer {accessToken}`

**Response**:
```json
{
  "code": 0,
  "data": {
    "user": {
      "userId": 1001,
      "username": "engineer1",
      "realName": "张三",
      "avatar": "https://...",
      "userType": "MAINCON"
    },
    "roles": [{ "roleId": 1, "roleName": "Site Engineer", "roleCode": "SITE_ENGINEER" }],
    "permissions": ["drawing:view", "equipment:refueling:query"],
    "projectInfos": [
      { "projectId": 10, "projectName": "G104 Project", "projectCode": "G104" }
    ]
  }
}
```

#### POST /system/auth/refresh-token

**Request Body**:
```json
{ "refreshToken": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" }
```

**Response**:
```json
{
  "code": 0,
  "data": {
    "accessToken": "new_token_here",
    "accessTokenExpireIn": 3600
  }
}
```

**错误码**:
| Code | 说明 |
|------|------|
| 1001002001 | Refresh Token 无效或已过期 |

#### POST /system/auth/logout

**Headers**: `Authorization: Bearer {accessToken}`
- 后端从 JWT 中解析 userId，删除 Redis 中对应 refreshToken

---

## 5. 安全与非功能需求

| 要求 | 规范 |
|------|------|
| 密码加密 | BCrypt（strength=12）|
| JWT 签名算法 | HS256，密钥从配置中心读取 |
| 防爆破 | 同一账号 5 分钟内失败 5 次锁定 10 分钟（Redis 计数）|
| HTTPS | 所有接口必须通过 HTTPS（网关层处理）|
| 多租户隔离 | 数据库层通过 `tenant_id` 字段隔离 |
| 响应时间 | 登录接口 P99 < 500ms |

---

## 6. 验收条件

- [ ] 使用正确 tenantCode + 用户名 + 密码可成功登录，返回 JWT
- [ ] 错误密码返回 1001001001 错误码
- [ ] 租户 Code 不存在返回 1001001002 错误码
- [ ] Access Token 1h 后过期，使用 Refresh Token 可换新 Token
- [ ] Refresh Token 30d 后过期，需重新登录
- [ ] get-permission-info 返回用户角色、权限码、项目列表
- [ ] 登出后 Refresh Token 在 Redis 中被清除
- [ ] SUBCON 账号无法通过 `/login` 接口登录（须使用 `/subcontractor/login`）
- [ ] 5 分钟内失败 5 次后账号被临时锁定
