---
doc_type: backend_spec
req_id: REQ-003H-pc
version: 0.1.0
status: draft
generated_from: REQ-003H-pc@0.1.1
data_contract_ref: DATA-CONTRACT-REQ-003H-pc.md@0.1.0
generated_at: 2026-08-07
generator: backend-agent
owner: ""
---

# 后端开发说明：PC 端 — 免审批上传（已完成审批的图纸直接生效）

> **本文档定义后端实现要点**。字段与接口定义引用 [DATA-CONTRACT-REQ-003H-pc](../shared/DATA-CONTRACT-REQ-003H-pc.md)，**不重复定义**。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003H-pc @ v0.1.1 |
| 数据契约 | DATA-CONTRACT-REQ-003H-pc.md @ v0.1.0 |
| 覆盖 Story | US-003H-001、US-003H-003、US-003H-004 |
| 覆盖 AC | AC-003H-004 ~ 009、AC-003H-012、AC-003H-013 |
| 上次同步时间 | 2026-08-07 |

---

## 1. 功能概述

在既有创建图纸接口上增加一条审批路径分支：当请求携带 `approvalRoute = PRE_APPROVED` 且操作人持有专项权限时，跳过两级审批流程，在**单个事务内**完成"建记录 → 版本生效 → QR 生成叠加"，产出与正常流程完全等价的生效版本。

**本需求不新增接口、不新增表**，只新增：1 个字段、1 个权限项、2 个错误码、1 条状态转换分支。

**实现的核心风险在事务边界**——版本一旦生效即对 SE 可见，因此不允许出现"已生效但无 QR"或"记录已落库但审计缺失"的中间态。

---

## 2. 技术栈

### 2.1 沿用现有

| 项 | 规范 |
|---|------|
| 语言 / 框架 | Java + Spring Cloud（微服务） |
| 数据库 | MySQL |
| 缓存 | Redis |
| 对象存储 | OSS |
| 服务归属 | `mcc-staff`（路径段 `staff`，见 [backend-rules §8.4](../../rules/backend-rules.md)） |
| 响应包装 | `CommonResult<T>` |

### 2.2 新增依赖

| 依赖 | 用途 | 备注 |
|-----|------|------|
| **无新增外部依赖** | — | QR 服务、OSS、权限模块均为既有内部服务，本需求只是新增一处调用组合 |

> 需求 §12 列出的三项外部系统（权限管理模块、QR 服务、OSS）**全部为既有集成**，不需要新建客户端或配置。

---

## 3. 业务逻辑

### 3.1 创建图纸（免审批路径）— 覆盖 AC-003H-004、005、006、007、012、013

**入口**：`POST /admin-api/staff/drawing/create`（API-003H-001，契约见 [DATA-CONTRACT §4.2](../shared/DATA-CONTRACT-REQ-003H-pc.md)）

**路由分支**：本接口是**双路径共用**入口。`approvalRoute` 缺省或为 `STANDARD` 时走既有逻辑，**一行不改**；仅 `PRE_APPROVED` 进入下述新分支。

```pseudo
function createDrawing(req, actor):
  # ---------- 路径分流 ----------
  route = req.approvalRoute ?? STANDARD
  if route == STANDARD:
    return existingCreateFlow(req, actor)        # 既有逻辑，零改动（AC-003H-009）

  # ---------- 以下为 PRE_APPROVED 分支 ----------

  # 1. 权限（先于一切业务校验，避免通过报错探测资源）
  if not actor.hasPermission('drawing:upload'):
    throw BizException(1003003020)
  if not actor.hasPermission('drawing:upload-approved'):
    throw BizException(1003003020)              # 实时查询，不读登录态缓存（AC-003H-013）

  # 2. 路径适用性
  if req.drawingId != null:
    throw BizException(1003003021)              # 仅新建图纸（AC-003H-006）

  # 3. 文件归属与既有字段校验（完全沿用，无放宽）
  assertFileOwnedByActorInProject(req.fileUrl, actor, ctx.projectId)
  validateSubmissionTypeAndCodeName(req)        # SHOP_DRAWING 时 code 非空且项目内唯一 → 1003003019
  validatePageCount(req)

  # 4. 落库 + QR（同一事务，见 §5）
  [事务开始]
    drawing = Drawing.insert(
      tenantId    = ctx.tenantId,
      projectId   = ctx.projectId,
      submissionType = req.submissionType,
      drawingCode = req.drawingCode,            # OTHERS 时为 NULL
      drawingName = req.drawingName,
      drawingCategory = req.drawingCategory,
      description = req.description,
      status      = ACTIVE,                     # ← 不是 PENDING_APPROVAL
      creatorId   = actor.id
    )

    version = DrawingVersion.insert(
      drawingId      = drawing.id,
      versionNo      = 'V1',
      fileUrl        = req.fileUrl,
      signedFileUrl  = req.fileUrl,              # ← 上传件即最终流通版（AC-003H-012）
      fileName       = req.fileName,
      fileSize       = req.fileSize,
      fileType       = 'PDF',
      pageCount      = resolvePageCount(req),    # 沿用既有规则
      approvalStatus = APPROVED,                 # ← 创建即终态
      approvalRoute  = PRE_APPROVED,             # ← 新增字段
      approvalId     = null,                     # ← 无审批记录
      isCurrent      = true,
      isDeprecated   = false,
      uploaderId     = actor.id,
      uploadTime     = now(),
      approvedTime   = now()
    )

    drawing.currentVersion   = 'V1'
    drawing.currentVersionId = version.id
    drawing.update()

    qr = qrService.generateAndOverlay(version.id, version.signedFileUrl)
    if qr.failed:
      throw BizException(1003007009)             # → 整体回滚（AC-003H-008）
    version.qrImageUrl      = qr.imageUrl
    version.pdfWithQrUrl    = qr.pdfUrl
    version.qrGeneratedTime = now()
    version.update()

    auditLog.write(actor, 'DRAWING_PRE_APPROVED_CREATE', drawing.id, {
      fileName, submissionType, approvalRoute: PRE_APPROVED
    })
  [事务提交]

  # 5. 显式不做的事（AC-003H-005）
  #    - 不创建 DrawingApproval（INTERNAL / EXTERNAL 均不创建）
  #    - 不创建 Todo
  #    - 不通知内部审批人、不通知项目 DC
  #    - 不推送 SE（新建图纸无分配关系，推送目标为空集）

  return { drawingId, drawingVersionId, versionNo: 'V1',
           approvalId: null, approvalStatus: APPROVED, approvalRoute: PRE_APPROVED }
```

**6 项必填决策**（[backend-rules §3.1.1](../../rules/backend-rules.md)）：

| 决策项 | 结论 |
|-------|------|
| 事务边界 | 建记录 + 版本生效 + QR 生成 + 审计**全部原子**，见 §5 |
| 幂等性 | ❌ 非幂等，沿用既有接口语义。`SHOP_DRAWING` 由 `(project_id, drawing_code)` 唯一约束兜底；`OTHERS` 无此保护，见 §6 |
| 审计 | ✅ 强制。免审批是绕过质量管控的路径，审计不可缺失且不可删除，见 §11 |
| 并发控制 | 依赖 DB 唯一约束（`SHOP_DRAWING`）；无跨行协调需求，不加锁，见 §10 |
| 一致性模型 | **强一致**。版本生效即对 SE 可见，不接受最终一致 |
| 失败补偿 | 事务回滚，无补偿逻辑。QR 失败由用户当场重试（文件已在 OSS，无需重传） |

### 3.2 权限项注册 — 覆盖 AC-003H-007

| 项 | 实现 |
|---|------|
| 权限标识 | `drawing:upload-approved` |
| 注册方式 | 迁移脚本向权限表插入，见 §8.2 |
| 角色绑定 | **不绑定任何角色模板**。由管理员在既有用户权限管理界面为具体用户勾选 |
| 校验方式 | 复合校验：`drawing:upload` && `drawing:upload-approved`，缺一即 `1003003020` |
| 缓存策略 | **不缓存**。每次提交实时查询权限，见 §9 |

### 3.3 版本历史与列表查询（读路径）— 覆盖 AC-003H-010、011

既有查询接口的响应 VO **新增 `approvalRoute` 字段**，供前端决定渲染分支：

| 接口 | 变更 |
|------|------|
| `/drawing/version/list`（版本历史） | `DrawingVersionRespVO` 增加 `approvalRoute` |
| `/drawing/get`（图纸详情） | 版本信息节点增加 `approvalRoute` |
| `/drawing/page`（列表分页） | `DrawingRespVO` 增加 `approvalRoute`（取当前版本的值） |

**SE 视角过滤**（AC-003H-011）：

- SE 调用 `/drawing/page` 时，响应中 `approvalRoute` 字段**由后端置空或不返回**——来源差异属管理侧信息，不下沉到现场
- 判定依据：调用方是否持有 `drawing:history` 权限；无该权限即视为 SE 视角
- 这是**接口层过滤**，不依赖前端隐藏（与 [REQ-014 FB-005/FB-006](../../requirements/pc/REQ-014-pc.md) 的处理方式一致）

---

## 4. 状态机实现

### 4.1 转换入口

沿用既有 `DrawingVersionStateMachine`，但本路径是**创建即终态**，不经过 `transition()`：

| 场景 | 实现方式 |
|-----|---------|
| `STANDARD` 路径 | 初始状态 `PENDING_INTERNAL`，后续变更走 `transition()` |
| `PRE_APPROVED` 路径 | **直接以 `APPROVED` 初始化**，不调用 `transition()`（无 from 状态可转换） |

> ⚠️ **不得**为实现本路径而放开 `transition()` 允许 `null → APPROVED`。那会给状态机开一个可被任意业务代码调用的后门。创建即终态由 insert 时赋值完成，状态机只管理**已存在记录**的变更。

### 4.2 后续变更的守卫

免审批版本进入 `APPROVED` 后，完全适用既有终态约束（[REQ-007-shared §5.3](../../requirements/shared/REQ-007-shared.md)）：

| 尝试的操作 | 结果 |
|-----------|------|
| 对该版本调 `/drawing/approve` | 拒绝（`APPROVED` 非 `PENDING_INTERNAL`） |
| 对该版本调 `/drawing/external-approve` | 拒绝（`approvalStatus != INTERNAL_APPROVED`），返回 `1003007002` |
| 修改 `approval_route` | **无任何接口提供该能力**，见 §8.3 |

### 4.3 与既有转换的关系

`APPROVED` 现有两条入边，二者**互不影响**：

| 入边 | 触发 | 是否创建 DrawingApproval |
|-----|------|:----------------------:|
| `INTERNAL_APPROVED → APPROVED` | DC 标记外部通过 | ✅ N 条 EXTERNAL |
| `null → APPROVED`（本需求） | 免审批上传 | ❌ 0 条 |

---

## 5. 事务边界

### 5.1 必须同一事务

| 步骤 | 理由 |
|-----|------|
| INSERT Drawing | 与版本互为存在前提 |
| INSERT DrawingVersion | 同上 |
| UPDATE Drawing.currentVersionId | 否则出现"图纸无当前版本"的脏数据 |
| QR 生成 + 回写 URL | **版本已生效即对 SE 可见**，不允许存在无 QR 的生效版本（AC-003H-008） |
| 审计日志写入 | 免审批操作的审计必须可信，不允许业务成功而审计丢失 |

### 5.2 QR 调用在事务内的取舍

[backend-rules §3.3.2](../../rules/backend-rules.md) 规定"业务写入 + 外部 API 调用"应当拆开事务。**本需求刻意违反该默认规则**，理由：

- QR 生成是版本"可流通"的构成要件，不是事后增强。缺 QR 的生效版本在业务上是**不合法状态**
- 该取舍与 [REQ-007 AC-007-008](../../requirements/shared/REQ-007-shared.md)（DC 外部审批通过时 QR 失败整体回滚）**保持一致**，两条产出生效版本的路径不应对同一失败给出不同处理
- 代价是事务持有时间较长（含一次外部调用），缓解措施见 §12.2

> **规则偏离记录**：本节为对 backend-rules §3.3.2 的显式偏离，理由与风险如上，已在需求 §2.3 异常流程与 §10.4 性能中评估。

### 5.3 必须拆开

| 场景 | 处理 |
|-----|------|
| 文件上传至 OSS | 发生在前置接口 `/drawing/uploadFile`，与本事务无关 |
| AI 识别（SHOP_DRAWING） | 上传阶段异步完成，与审批路径无关 |

---

## 6. 幂等性要求

| API | 幂等 | 保护机制 | 缺口 |
|-----|:---:|---------|------|
| API-003H-001（`SHOP_DRAWING`） | ❌ | `(project_id, drawing_code)` 唯一约束 → 重复提交报 `1003003019` | 无 |
| API-003H-001（`OTHERS`） | ❌ | **无**。`drawing_code` 为 NULL，唯一约束不生效 | ⚠️ 重复点击会创建两条生效记录 |

**对 `OTHERS` 缺口的处理**：

沿用既有接口现状，本需求**不引入新的幂等机制**——该缺口在 `STANDARD` 路径同样存在，属既有接口的固有行为，不应在本需求中单独修补（会造成两条路径行为不一致）。

前端侧缓解：[Confirm] 点击后立即进入 loading 并禁用（见 [UI-REQ-003H-pc §3.1](../ui/pc/UI-REQ-003H-pc.md)）。

<!-- NOTE: OTHERS 类型的重复提交防护建议作为独立技术需求提出，覆盖两条路径，不在本需求范围内。 -->

---

## 7. 异步任务与事件

**本需求不产生任何异步任务，不发布任何领域事件。**

| 项 | 说明 |
|---|------|
| QR 生成 | **同步**，在事务内。不走 outbox——outbox 的前提是"业务已成功、事件可稍后送达"，而本场景 QR 失败必须回滚业务 |
| SE 推送 | 不触发。新建图纸默认无分配关系（[REQ-003-shared §3.4](../../requirements/shared/REQ-003-shared.md) 规则 1），推送目标为空集；后续管理员 [Assign] 时走 [REQ-003D-pc](../../requirements/pc/REQ-003D-pc.md) 既有机制 |
| Todo 创建 | 不触发（AC-003H-005） |
| 站内消息 | 不触发 |

> QA 需针对上述"不触发"设计**反向断言**，见 [QA-REQ-003H-pc §4](../qa/pc/QA-REQ-003H-pc.md)。

---

## 8. 数据库设计

### 8.1 表结构变更

```sql
-- drawing_versions 新增审批路径字段
ALTER TABLE drawing_versions
  ADD COLUMN approval_route VARCHAR(20) NOT NULL DEFAULT 'STANDARD'
  COMMENT '审批路径：STANDARD=两级审批 / PRE_APPROVED=免审批直接生效';
```

**索引**：不新增。数据量级见需求 §11（单项目免审批记录预期 < 150 条），全表扫描成本可忽略；若后续 §14 监控需按路径聚合统计，再评估 `(project_id, approval_route)` 复合索引。

**字段说明**：

| 项 | 值 | 理由 |
|---|---|------|
| 类型 | `VARCHAR(20)` | 存枚举值字符串，不存 int（[backend-rules §3.2.3](../../rules/backend-rules.md)） |
| 可空 | `NOT NULL` | 每个版本必然有一条路径 |
| 默认值 | `'STANDARD'` | 保证新旧代码并存期间行为一致——旧代码 insert 不带该字段时自动取默认值 |

### 8.2 权限数据初始化

```sql
-- 注册权限项，不绑定任何角色
INSERT INTO sys_permission (code, name, module, remark)
VALUES ('drawing:upload-approved', '免审批上传', 'drawing',
        '上传已在系统外完成审批的图纸，跳过两级审批直接生效。默认不授予任何角色，需按人单独开通');
```

### 8.3 不可变字段的强制

`approval_route` 落库后永不变更。实现要点：

- Update DTO / MyBatis update 语句中**不包含**该列
- 若使用动态 SQL 全字段更新，须显式将该列加入排除列表
- Code Review 检查项：任何 `UPDATE drawing_versions SET ... approval_route` 均为缺陷

### 8.4 迁移

| 项 | 内容 |
|---|------|
| 命名 | `V{时间戳}__add_approval_route_to_drawing_versions.sql` |
| 存量回填 | 由 `DEFAULT 'STANDARD'` 自动完成，**无需 UPDATE 语句** |
| 校验 | 执行后确认 `SELECT COUNT(*) FROM drawing_versions WHERE approval_route IS NULL` 为 0 |
| 回滚 | 字段保留不用即可（默认值下系统行为与现状一致），无需数据回滚 |
| 只前进不回退 | 遵循 [backend-rules §3.6.3](../../rules/backend-rules.md) |

---

## 9. 缓存策略

| 数据 | 是否缓存 | 理由 |
|-----|:-------:|------|
| `drawing:upload-approved` 权限判定 | ❌ **不缓存** | AC-003H-013 要求提交时实时校验——用户打开弹窗后权限可能被回收。缓存会让回收延迟生效，直接违反该 AC |
| 图纸列表 / 版本历史响应 | 沿用既有策略 | 新增 `approvalRoute` 字段不改变缓存粒度 |

> ⚠️ 若现有权限模块存在登录态权限集缓存，本接口**必须绕过缓存直查**，或确保权限变更时主动失效。这是 AC-003H-013 的实现要点，需与权限模块维护方确认。

---

## 10. 并发控制

| 场景 | 策略 |
|-----|------|
| 同一 `drawingCode` 并发创建 | DB 唯一约束 `(project_id, drawing_code)`，后到者报 `1003003019` |
| `OTHERS` 类型并发创建 | 无保护，见 §6 |
| 单条记录的后续变更 | 沿用既有乐观锁 / 状态机守卫 |

**不加锁的理由**：本路径只 INSERT 不 UPDATE 既有行，无跨行协调需求。按 [backend-rules §3.8](../../rules/backend-rules.md) 的偏好顺序，DB 唯一约束已是最强且最简单的手段。

---

## 11. 审计日志

### 11.1 强制审计

| 操作 | 审计 | 说明 |
|-----|:---:|------|
| 免审批创建图纸 | ✅ **强制且不可删除** | 需求 §10.2 明确要求 |
| 权限授予 / 回收 `drawing:upload-approved` | ✅ | 属用户权限管理模块既有能力，本需求不实现 |

### 11.2 审计字段

在既有 `audit_logs` 表结构上，本操作的 `payload` 必须包含：

```json
{
  "action": "DRAWING_PRE_APPROVED_CREATE",
  "resource_type": "DRAWING",
  "resource_id": 412,
  "payload": {
    "drawingVersionId": 908,
    "fileName": "basement-waterproofing.pdf",
    "drawingCode": "ARCH-088",
    "submissionType": "SHOP_DRAWING",
    "approvalRoute": "PRE_APPROVED",
    "projectId": 17
  }
}
```

### 11.3 写入时机

**同一事务内写入**（见 §5.1）。免审批操作的审计不允许"业务成功但审计丢失"。

---

## 12. 性能要求

### 12.1 指标

| 指标 | 目标值 | 来源 |
|-----|-------|------|
| 免审批提交同步操作（建记录 + 生效 + QR） | ≤ 5 秒（P95） | 需求 §10.1 |
| 权限判定 | 无额外 HTTP 往返 | 需求 §10.1 |

> 与 [REQ-007-shared §9.1](../../requirements/shared/REQ-007-shared.md) 的外部审批通过操作指标对齐——两者同为"生效 + QR"同步事务。

### 12.2 长事务的缓解

事务内含一次 QR 外部调用（见 §5.2 规则偏离），缓解措施：

| 措施 | 说明 |
|-----|------|
| QR 调用超时 | 设置明确超时（建议 ≤ 3 秒），超时即抛 `1003007009` 触发回滚，避免事务无限持有 |
| 不在事务内做文件 I/O | 文件已在 OSS，QR 服务直接基于 URL 处理 |
| 监控 | 事务持有时长进 APM，超阈值告警 |

<!-- TODO: QR 服务的具体超时阈值需与该服务维护方确认；需求未指定，不擅自定死。此处 3 秒为基于 §12.1 总预算 5 秒的建议值，待评审。 -->

---

## 13. 安全要求

### 13.1 鉴权

JWT（Bearer Token），沿用现有 SSO 体系，本需求不改变鉴权机制。

### 13.2 权限校验

| API | 校验逻辑 | 失败响应 |
|-----|---------|---------|
| API-003H-001（`STANDARD`） | `hasPermission('drawing:upload')` | 403 |
| API-003H-001（`PRE_APPROVED`） | `hasPermission('drawing:upload')` **且** `hasPermission('drawing:upload-approved')` | 403 / `1003003020` |
| 全部路径 | `fileUrl` 归属当前 `Project-Id` | 403 |

**关键约束**：

1. **不信任前端传参**。前端根据权限控制区域渲染，但 `approvalRoute = PRE_APPROVED` 到达后端时必须独立校验，不接受任何形式的前端断言（AC-003H-007）
2. **实时校验**，不读缓存（见 §9）
3. **先权限后业务**。权限失败时不得泄露资源是否存在

### 13.3 越权防护

| 风险 | 防护 |
|-----|------|
| 跨项目引用 `fileUrl` | 校验文件归属当前 `Project-Id` 且由当前用户上传 |
| 绕过前端直接调接口 | §13.2 后端独立校验 |
| 通过修改 `approval_route` 伪造来源 | 无接口提供修改能力（§8.3） |

### 13.4 数据加密

传输 TLS 1.2+；OSS 文件存储加密。沿用现有配置。

---

## 14. 可观测性

### 14.1 埋点与指标

| 指标 | 类型 | 维度 | 用途 |
|-----|------|------|------|
| `drawing.preApproved.create.success` | 计数器 | 操作人、项目 | 需求 §10.4 关键埋点 |
| `drawing.preApproved.create.fail` | 计数器 | error_code、项目 | 失败分布 |
| `drawing.preApproved.ratio` | 比率 | 项目、月 | 免审批占当月新建图纸比例 |
| `drawing.preApproved.txn.duration` | 直方图 | — | 事务持有时长（§12.2） |
| `drawing.preApproved.qr.fail` | 计数器 | — | QR 失败率 |
| `permission.uploadApproved.holders` | Gauge | 项目 | 持有该权限的用户数 |

### 14.2 告警规则

| 条件 | 级别 | 通知 | 来源 |
|-----|------|------|------|
| 免审批占当月新建图纸比例 > 30% | 业务告警 | PM | 需求 §10.4 |
| `1003003020` 触发次数 > 0 | 安全告警 | 管理员 + 后端 TL | 需求 §10.4：非零即需人工确认，可能是越权尝试 |
| `drawing:upload-approved` 持有者数量变化 | 通知 | 管理员复核 | 需求 §10.4 |
| QR 失败率 > 1% | 技术告警 | 后端 TL | 与 REQ-007 §14 对齐 |
| 事务持有时长 P95 > 5 秒 | 技术告警 | 后端 TL | §12.1 |

---

## 15. AC 覆盖检查表

| AC ID | 实现位置 | 覆盖 |
|------|---------|:---:|
| AC-003H-001 | 前端渲染控制，后端无实现 | N/A |
| AC-003H-002 | 前端渲染控制，后端无实现 | N/A |
| AC-003H-003 | 前端校验；后端侧对应"`approverId` 传入则忽略"（§3.1 伪代码未取用该字段） | ✅ |
| AC-003H-004 | §3.1 事务块（Drawing.status=ACTIVE、版本 APPROVED/PRE_APPROVED/isCurrent、signedFileUrl、QR） | ✅ |
| AC-003H-005 | §3.1 步骤 5「显式不做的事」、§7 异步任务（全为不触发） | ✅ |
| AC-003H-006 | §3.1 步骤 2（`drawingId != null` → 1003003021） | ✅ |
| AC-003H-007 | §3.1 步骤 1、§13.2 后端独立校验 | ✅ |
| AC-003H-008 | §3.1 QR 失败抛异常触发回滚、§5.1 事务边界 | ✅ |
| AC-003H-009 | §3.1 路径分流（STANDARD 走既有逻辑零改动） | ✅ |
| AC-003H-010 | §3.3 版本历史响应新增 `approvalRoute` 供前端渲染分支 | ✅ |
| AC-003H-011 | §3.3 列表响应新增 `approvalRoute` + SE 视角接口层过滤 | ✅ |
| AC-003H-012 | §3.1 `signedFileUrl = fileUrl` + QR 生成写入 `pdfWithQrUrl` | ✅ |
| AC-003H-013 | §3.1 步骤 1 实时校验、§9 不缓存权限 | ✅ |

**覆盖率**：11 / 13，2 条纯前端渲染 AC 标 N/A（由 [FRONTEND-REQ-003H-pc §13](../frontend/pc/FRONTEND-REQ-003H-pc.md) 覆盖）。

---

## 16. 测试要求

> 团队当前未强制要求单元测试，以下为建议项。

| 层级 | 建议程度 | 覆盖重点 |
|-----|---------|---------|
| 单元 | 建议 | 路径分流判定、权限复合校验、`drawingId` 存在性判定 |
| 集成 | **强烈建议** | 事务回滚（QR 失败后 Drawing / DrawingVersion 均无残留）——这是本需求最关键且最易出错的行为 |
| 契约 | QA 负责 | §4.2 全部错误码，见 [QA-REQ-003H-pc §6](../qa/pc/QA-REQ-003H-pc.md) |
| 回归 | **必须** | `approvalRoute` 缺省时响应与上线前完全一致（AC-003H-009） |

---

## 17. 部署与回滚

### 17.1 上线前

- [ ] 执行迁移脚本：`drawing_versions` 增加 `approval_route`（§8.1）
- [ ] 执行权限注册脚本，确认**未绑定任何角色模板**（§8.2）
- [ ] 确认 QR 服务可用性与超时配置
- [ ] 确认审计日志通道能正确接收 `DRAWING_PRE_APPROVED_CREATE`
- [ ] Feature Flag 默认关闭

### 17.2 上线后

- [ ] 校验 `approval_route` 无空值
- [ ] 端到端：免审批上传一张图纸，确认版本直接 ACTIVE、`pdfWithQrUrl` 有值、无 Todo 产生
- [ ] 回归：`STANDARD` 路径完整走一遍两级审批，确认行为无变化
- [ ] 越权：无权限账号直接调接口，确认返回 403 / `1003003020`

### 17.3 回滚

| 项 | 方案 |
|---|------|
| 代码 | Feature Flag 关闭免审批分支，接口对 `PRE_APPROVED` 一律返回 `1003003020` |
| 数据 | **不回滚**。已通过该路径生效的版本保持有效——它们是合法的已生效图纸 |
| 字段 | 保留不用，默认值 `STANDARD` 下行为与现状一致 |

---

## 18. 验收条件

- [ ] `approvalRoute` 缺省时接口行为与上线前逐字段一致
- [ ] 免审批路径在单事务内完成建记录 + 生效 + QR + 审计
- [ ] QR 失败后 DB 中无任何残留记录
- [ ] 无 `drawing:upload-approved` 的请求一律 403，且不泄露资源存在性
- [ ] 携带 `drawingId` 的免审批请求返回 `1003003021`
- [ ] 免审批版本关联的 DrawingApproval 记录数为 0，Todo 数为 0
- [ ] 权限校验为实时查询，回收后立即生效
- [ ] `approval_route` 无任何更新路径
- [ ] SE 调用列表接口时响应不含 `approvalRoute`
- [ ] §14 全部指标与告警已接入

---

## 19. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-08-07 | backend-agent | 初稿：定义免审批路径分支的业务逻辑、事务边界（含对 backend-rules §3.3.2 的显式偏离记录）、`approval_route` 字段与不可变约束、权限项注册与实时校验、审计与告警规则 |
