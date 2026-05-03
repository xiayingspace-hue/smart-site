---
doc_type: backend_spec
req_id: REQ-001
version: 0.1.0
status: draft
generated_from: REQ-001-shared.md@current
generated_at: 2026-05-03
owner: ""
---

# 后端开发说明：多租户登录与会话管理

> **本文档供后端开发工程师及其 agent 使用**。
>
> ⚠️ **重要约定**：
> - API 字段定义与数据模型完全引用 [REQ-001-shared.md](../../requirements/shared/REQ-001-shared.md)，本文档不重复定义字段。
> - 各端（PC/APP/H5）UI 交互差异由各端前端文档负责，本文档只关注后端实现。
> - 代码注释中必须标注覆盖的 AC ID。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-001-shared @ current |
| 适用端 | ALL（PC / APP / H5 共享同一套后端接口） |
| 覆盖 Story | 登录、会话管理、权限获取、登出 |
| 服务模块 | `system-service` |
| 上次同步时间 | 2026-05-03 |

> ⚠️ REQ-001-pc `generate.backend_spec: false`，本文档来源为 REQ-001-shared，不从 PC 端需求派生。

---

## 1. 功能概述

实现 SaaS 多租户架构下的身份认证与会话管理，包含：

| 功能 | 接口 | 说明 |
|-----|------|-----|
| MAINCON 用户登录 | `POST /system/auth/login` | 平台用户（总包商）登录，返回 JWT |
| SUBCON 用户登录 | `POST /system/auth/subcontractor/login` | 分包商用户登录，独立校验逻辑 |
| Token 刷新 | `POST /system/auth/refresh-token` | Access Token 过期后自动续期 |
| 获取用户权限信息 | `GET /system/auth/get-permission-info` | 返回用户信息 + 角色 + 权限码 + 项目列表 |
| 登出 | `POST /system/auth/logout` | 清除 Redis Session |

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

- **语言**：Java
- **框架**：Spring Boot（`system-service` 模块）
- **认证**：JWT（Access Token + Refresh Token 双 Token）
- **缓存**：Redis（Session / 幂等 / 限流）
- **数据库**：MySQL（租户分表/分库策略）
- **日志**：结构化 JSON 日志

### 2.2 本需求新增依赖

| 库 | 用途 | 评估 |
|---|------|------|
| 无新增 | — | — |

---

## 3. 业务逻辑

### 3.1 MAINCON 用户登录（`POST /system/auth/login`）

```
1. 接收 username / password / tenantCode（从 Header X-Tenant-Id 或请求体获取）
2. 校验租户是否存在且状态为启用
   └─ 失败 → 1002000002（租户不存在）/ 1002000003（已禁用）
3. 校验验证码（如果系统开启验证码）
   └─ 失败 → 1002000004
4. 通过 authenticate() 校验 username + password
   └─ 失败 → 1002000000（账号或密码不正确）
5. 检查用户所属租户 === 登录租户
   └─ 失败 → 1002000005
6. 检查用户类型 === PLATFORM
   └─ 失败 → 1002000008（用户来源错误）
7. 检查用户状态 === 启用
   └─ 失败 → 1002000001（账号被禁用）
8. 绑定社交账号（如 socialCode 存在）
9. 创建 Access Token（1h）+ Refresh Token（30d），写入 Redis
10. 记录登录日志（含租户信息、IP、User-Agent）
11. 返回 AuthLoginRespVO（userId / tenantId / tenantName / accessToken / refreshToken / expiresTime）
```

### 3.2 SUBCON 用户登录（`POST /system/auth/subcontractor/login`）

```
1. 接收同 AuthLoginReqVO 结构
2. 通过手机号查询分包商是否存在
   └─ 失败 → 1002029015（分包商信息不存在）
3. 检查用户类型 === SUBCONTRACTOR
   └─ 失败 → 1002000008
4. 走同 §3.1 步骤 2/4/7/9/10
5. 返回 AuthSubLoginRespVO
```

### 3.3 Token 刷新（`POST /system/auth/refresh-token`）

```
1. 接收 refreshToken（query 参数）
2. 从 Redis 查询 refreshToken 是否存在且未过期
   └─ 失败 → 1002000006（Token 已过期）→ 客户端需跳转登录页
3. 生成新的 Access Token，写入 Redis，旧 Access Token 作废
4. 返回新的 AuthLoginRespVO
```

### 3.4 获取用户权限信息（`GET /system/auth/get-permission-info`）

```
1. 从 Authorization Header 解析当前用户 ID + 租户 ID
2. 获取用户基本信息（nickname / avatar / mobile / signatureUrl）
3. 获取用户角色列表
4. 根据用户类型分支：
   ├─ PLATFORM 用户：获取角色关联的项目列表
   └─ SUBCONTRACTOR 用户：获取分包商关联的项目列表
5. 获取菜单权限码（按 Project-Id Header 过滤，如有）
6. 处理总工地特殊逻辑（PROJECT_ID_MAIN）
7. 返回 AuthPermissionInfoRespVO（user / roles / permissions / projectInfos / subcontractorId）
```

### 3.5 登出（`POST /system/auth/logout`）

```
1. 从 Authorization Header 提取 Access Token
2. 从 Redis 删除 Access Token 记录
3. 移除客户端 ID 绑定
4. 记录登出日志（who / when / IP）
5. 返回 200 OK
```

---

## 4. 状态机实现

本需求无业务实体状态机（登录是无状态操作，会话状态在 Redis 管理）。

**Token 状态**（Redis 管理，非数据库状态机）：

| Token 状态 | 说明 | 过期处理 |
|-----------|------|---------|
| ACTIVE | 写入 Redis，有效期内 | — |
| EXPIRED | TTL 到期，Redis 自动删除 | Access Token → 触发 refresh；Refresh Token → 跳转登录 |
| REVOKED | 手动删除（登出时） | 立即失效，返回 401 |

---

## 5. 事务边界

| 操作 | 事务边界 | 说明 |
|-----|---------|------|
| 登录（写 DB 日志 + 写 Redis Token） | ❌ 拆开 | Redis 写失败不回滚 DB 日志；Token 写入失败重试即可 |
| 社交账号绑定 + 登录日志 | ✅ 同一事务 | 绑定失败不记录日志 |
| 登出（删 Redis + 写登出日志） | ❌ 拆开 | 日志写失败不影响登出成功 |

---

## 6. 幂等性要求

| API | 幂等键 | 策略 |
|-----|-------|-----|
| `POST /system/auth/login` | username + tenantId + 时间窗口（5s） | Redis 防重，同一账号 5s 内重复请求返回同一 Token |
| `POST /system/auth/refresh-token` | refreshToken 本身 | refreshToken 只能使用一次（用后作废） |
| `POST /system/auth/logout` | accessToken | 已登出的 Token 再次请求返回 200（幂等成功） |

---

## 7. 异步任务与事件

| 任务 | 触发时机 | 实现方式 | 说明 |
|-----|---------|---------|------|
| 登录日志写入 | 登录成功后 | 异步写入 DB | 不阻塞登录响应 |
| 登出日志写入 | 登出成功后 | 异步写入 DB | 不阻塞登出响应 |
| Token 过期清理 | Redis TTL 自动触发 | Redis Keyspace Notification（可选） | 无需主动轮询 |

---

## 8. 数据库设计

### 8.1 核心表（已有，不新建）

| 表名 | 用途 |
|-----|------|
| `system_tenant` | 租户信息（tenantCode / status） |
| `system_user` | 用户信息（username / password_hash / tenant_id / type / status） |
| `system_role` | 角色表 |
| `system_user_role` | 用户-角色关联 |
| `system_login_log` | 登录日志 |
| `system_oauth2_access_token` | Access Token 记录（Redis 为主，DB 为辅） |
| `system_oauth2_refresh_token` | Refresh Token 记录 |

### 8.2 Redis Key 规范

| Key | TTL | 内容 |
|-----|-----|------|
| `oauth2:access_token:{token}` | 1h | userId / tenantId / roles / permissions |
| `oauth2:refresh_token:{token}` | 30d | userId / tenantId |
| `login:idem:{username}:{tenantId}` | 5s | 幂等控制 |

---

## 9. 缓存策略

| 数据 | 缓存层 | TTL | 失效时机 |
|-----|-------|-----|---------|
| Access Token | Redis | 1h | 登出 / TTL 到期 |
| Refresh Token | Redis | 30d | 使用一次后作废 / 登出 |
| 权限信息（get-permission-info） | Redis | 10min | 角色变更时主动清除 |

---

## 10. 并发控制

| 场景 | 策略 | 说明 |
|-----|------|------|
| 同账号并发登录 | Redis 幂等键（5s 窗口） | 防重复 Token 生成 |
| Token 刷新并发 | Refresh Token 单次使用（原子 DEL + 新建） | 防止多个 Access Token 同时生效 |
| 登出并发 | Redis DEL 幂等 | 重复登出安全返回 200 |

---

## 11. 审计日志

| 操作 | 必须审计 | 审计内容 |
|-----|---------|---------|
| 登录成功 | ✅ | who(userId) / when / IP / User-Agent / tenantId / 用户类型 |
| 登录失败 | ✅ | username / tenantId / 失败原因 / IP |
| Token 刷新 | ✅ | userId / tenantId / IP |
| 登出 | ✅ | userId / tenantId / IP |

**保留期限**：6 个月（依业务安全要求）

---

## 12. 性能要求

| 接口 | P95 延迟目标 | 说明 |
|-----|------------|------|
| `POST /system/auth/login` | ≤ 1s | 来自 REQ-001-shared §5 |
| `GET /system/auth/get-permission-info` | ≤ 500ms | 权限信息有 Redis 缓存 |
| `POST /system/auth/refresh-token` | ≤ 300ms | 纯 Redis 操作 |

**关键优化**：
- `get-permission-info` 权限数据缓存 Redis（TTL 10min），角色变更时主动失效
- 登录日志异步写入，不阻塞响应

---

## 13. 安全要求

### 13.1 输入校验

- 所有外部输入走 DTO 校验（`@Valid` + JSR-303）
- SQL 使用 MyBatis 参数化查询，禁止字符串拼接
- username / password 长度校验见 REQ-001-shared §2.1

### 13.2 鉴权

- 除登录/刷新接口外，所有接口强制校验 `Authorization: Bearer <token>`
- 中间件统一解析 Token，注入 SecurityContext
- 接口层不直接读 Header，通过 `@AuthenticationPrincipal` 获取当前用户

### 13.3 密码安全

- 密码存储：BCrypt Hash，禁止明文
- 密码字段不写入任何日志
- 登录失败不区分"账号不存在"与"密码错误"（防枚举）——统一返回 1002000000

### 13.4 限流

- 登录接口：同 IP 每分钟最多 20 次（Redis 滑动窗口）
- 超限返回 429，提示"Too many login attempts, please try again later"

---

## 14. Request Headers 约定

所有接口（含登录后接口）需正确处理以下 Header：

| Header | 必填 | 说明 |
|--------|-----|------|
| `Authorization` | 登录后必填 | `Bearer <accessToken>` |
| `X-Tenant-Id` | 必填 | 租户标识 |
| `Project-Id` | 可选 | 当前选中项目 ID（`get-permission-info` 使用） |
| `lang` | 必填 | `en` 或 `zh`，控制错误信息语言 |

---

## 15. 错误码清单

| 错误码 | 说明 | HTTP 状态 |
|-------|------|---------|
| 1002000000 | 账号或密码不正确 | 400 |
| 1002000001 | 账号被禁用 | 400 |
| 1002000002 | 租户不存在 | 400 |
| 1002000003 | 租户已被禁用 | 400 |
| 1002000004 | 验证码不正确 | 400 |
| 1002000005 | 用户不属于该租户 | 400 |
| 1002000006 | Token 已过期 | 401 |
| 1002000008 | 用户来源错误 | 400 |
| 1002029015 | 分包商信息不存在 | 400 |

---

## 16. AC 覆盖检查表

> REQ-001-pc `generate.backend_spec: false`，AC 来源为 REQ-001-shared §5。

| 验收条件 | 实现位置 | 状态 |
|---------|---------|------|
| 租户不存在/禁用时返回对应错误码 | `AuthServiceImpl.login()` | TODO |
| 账号密码错误返回 1002000000 | `AuthServiceImpl.login()` | TODO |
| 用户类型不匹配返回 1002000008 | `AuthServiceImpl.login()` | TODO |
| 登录成功返回 accessToken + refreshToken | `AuthServiceImpl.login()` | TODO |
| Token 刷新：旧 refreshToken 作废，返回新 accessToken | `AuthServiceImpl.refreshToken()` | TODO |
| `get-permission-info` 返回 projectInfos（含权限过滤） | `AuthServiceImpl.getPermissionInfo()` | TODO |
| 登出后 Token 立即失效（再次请求返回 401） | `AuthServiceImpl.logout()` | TODO |
| 所有接口 `lang` Header 控制错误消息语言 | 统一消息国际化中间件 | TODO |
| 登录/登出操作写入审计日志 | `LoginLogServiceImpl` | TODO |
| 同 IP 超过限流阈值返回 429 | Redis 限流中间件 | TODO |

---

## 17. 测试要求

| 层级 | 框架 | 覆盖范围 |
|-----|------|---------|
| 单元 | JUnit 5 + Mockito | 所有业务分支（正常 + 全部错误码）、Token 生命周期 |
| 集成 | Testcontainers（MySQL + Redis） | 登录全流程、Token 刷新、并发幂等 |
| 契约 | schemathesis | 与前端约定的请求/响应 schema |

---

## 18. 验收条件

- [ ] §16 所有条目标记完成
- [ ] 单元 + 集成测试通过
- [ ] 契约测试与前端 agent 通过
- [ ] 登录接口压测 P95 ≤ 1s
- [ ] 安全扫描无 high 项（密码不出现在日志）
- [ ] 已部署到 dev，PC / APP 端可联调

---

## 19. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-03 | agent | 按新模板格式从 REQ-001-shared 重写；来源统一为 shared 层，删除旧版对 pc/app 端的直接引用 |
