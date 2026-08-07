---
doc_type: data_contract
req_id: REQ-003H-pc
version: 0.1.0
status: draft
generated_from: REQ-003H-pc@0.1.1
generated_at: 2026-08-07
generator: data-contract-agent
owner: ""
---

# 数据契约文档：PC 端 — 免审批上传（已完成审批的图纸直接生效）

> **本文档是前后端联调与 QA 接口测试的单一事实源**。
>
> ⚠️ **本文档为增量契约**。REQ-003H 不新建实体、不新建接口，只在既有图纸契约上增加一个字段、一个枚举、一条状态转换和一个请求参数。
> 既有实体与接口的完整定义在 [REQ-003-shared §2 / §4](../../requirements/shared/REQ-003-shared.md) 与 [REQ-007-shared §4 / §7](../../requirements/shared/REQ-007-shared.md)，本文档**只引用不复制**。

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003H-pc @ v0.1.1 |
| 上游契约 | REQ-003-shared（图纸主契约）、REQ-007-shared（两级审批扩展） |
| 覆盖 Story | US-003H-001、US-003H-002、US-003H-003、US-003H-004 |
| 覆盖 AC | AC-003H-001 ~ AC-003H-013 |
| 上次同步时间 | 2026-08-07 |

---

## 1. 实体数据模型（Data Models）

> 对应需求 [§6 核心实体与数据生命周期](../../requirements/pc/REQ-003H-pc.md)。

### 1.1 DrawingVersion（增量）

**对应实体**：ENT-002

完整字段定义见 [REQ-003-shared §2.2](../../requirements/shared/REQ-003-shared.md) 与 [REQ-007-shared §4.1](../../requirements/shared/REQ-007-shared.md)。本需求**新增 1 个字段**：

| 字段名 | 类型 | 必填 | 默认值 | 约束 | 描述 |
|-------|------|-----|-------|------|------|
| approval_route | varchar(20) | ✅ | `STANDARD` | 枚举，见 §2.1 | 该版本到达当前状态所走的审批路径。落库后**永不变更**，任何接口不提供修改能力 |

**索引**：

- 无需新增独立索引。若后续 §12 回归监控需要按路径统计，可加 `(project_id, approval_route)` 复合索引，本期不加（数据量级见需求 §11，单项目免审批记录预期 < 150 条）

**存量数据**：全部回填 `STANDARD`，迁移方案见需求 §13。

**字段间约束（后端必须保证）**：

| 约束 | 说明 |
|------|------|
| `approval_route = PRE_APPROVED` ⇒ `approval_status = APPROVED` | 免审批版本创建即为终态，不存在中间态 |
| `approval_route = PRE_APPROVED` ⇒ `version_no = 'V1'` | 仅用于新建图纸首版 |
| `approval_route = PRE_APPROVED` ⇒ `signed_file_url = file_url` | 上传件即最终流通版 |
| `approval_route = PRE_APPROVED` ⇒ 关联 DrawingApproval 记录数 = 0 | 系统内未发生审批行为，不伪造审批记录 |

### 1.2 Drawing（无变更）

**对应实体**：ENT-001

本需求**不新增任何字段**。免审批创建的记录 `status` 直接为 `ACTIVE`（枚举值本身已存在，见 [REQ-003-shared §2.7](../../requirements/shared/REQ-003-shared.md)）。

### 1.3 DrawingApproval（无变更）

**对应实体**：ENT-003

本需求**不新增字段，也不产生记录**。见 §1.1 字段间约束第 4 条。

---

## 2. 枚举与字典（Enums）

### 2.1 ApprovalRoute（新增枚举）

| 值 | 显示名(zh-CN) | 显示名(en) | 描述 |
|---|--------------|-----------|------|
| STANDARD | 标准审批 | Standard Approval | 走内部审批 → 外部审批两级流程创建的版本。**默认值**，存量数据全部回填此值 |
| PRE_APPROVED | 免审批 | Pre-Approved | 上传时已在系统外完成全部审批，跳过两级审批直接生效 |

> **为什么是独立枚举而不是扩展 `approvalStatus`**：见需求 §6.2。版本状态仍是 `APPROVED`，改变的只是它**如何**到达该状态。下游任何"是否已生效"的判断**不得**改读 `approvalRoute`。

### 2.2 DrawingVersion.approvalStatus（引用，无变更）

5 态定义见 [REQ-007-shared §5.1](../../requirements/shared/REQ-007-shared.md)。本需求不新增状态值。

---

## 3. 状态机定义（State Machines）

> 对应需求 [§7 状态机](../../requirements/pc/REQ-003H-pc.md)。本需求为既有状态机**新增一条入边**，其余转换不变。

### 3.1 drawing_version_approval_status 状态机（增量）

```yaml
state_machine: drawing_version_approval_status
# 完整定义见 REQ-007-shared §5；此处仅声明本需求新增的转换

transitions:
  - id: T-003H-001
    from: null                      # 无前序状态：创建即终态
    to: APPROVED
    action: createPreApprovedDrawing
    guard:
      - actor_has_permission: ["drawing:upload", "drawing:upload-approved"]   # 两者必须同时持有
      - request_has_no_drawing_id: true                                        # 仅新建图纸
      - file_valid: "PDF && size <= 50MB"
      - drawing_code_unique_in_project: "when submissionType == SHOP_DRAWING"
    side_effects:
      - set: "approval_route = PRE_APPROVED"
      - set: "signed_file_url = file_url"
      - set: "is_current = true"
      - set: "is_deprecated = false"
      - set: "approved_time = now()"
      - set: "drawing.status = ACTIVE"
      - invoke: "qr.generateAndOverlay(signed_file_url)"    # 同事务，失败则整体回滚
      - audit: "记录操作人、时间、文件名、submissionType、approval_route"
    not_side_effects:                                        # 显式声明"不做什么"，QA 据此设计反向断言
      - "不创建 DrawingApproval 记录（phase=INTERNAL 与 phase=EXTERNAL 均不创建）"
      - "不创建任何 Todo 任务"
      - "不向内部审批人发送通知"
      - "不向项目 DC 发送通知"
      - "不主动推送 SE（新建图纸默认无分配关系，推送目标为空集）"
    ac_refs: [AC-003H-004, AC-003H-005]

forbidden_transitions:
  - from: null
    to: APPROVED
    when: "request contains drawingId"
    reason: "免审批路径仅适用于新建图纸首版；已有图纸改版必须走两级审批"
    error_code: 1003003021
    ac_refs: [AC-003H-006]

  - from: null
    to: APPROVED
    when: "actor lacks drawing:upload-approved"
    reason: "免审批为专项权限，默认不授予任何人"
    error_code: 1003003020
    ac_refs: [AC-003H-007, AC-003H-013]

  - from: [PENDING_INTERNAL, INTERNAL_APPROVED]
    to: APPROVED
    action: createPreApprovedDrawing
    reason: "不存在'将审批中的版本直接标记为已审批'的操作入口或接口"

  - from: APPROVED
    to: "*"
    reason: "APPROVED 为终态（沿用 REQ-007-shared §5.3）"

immutable_fields:
  - field: approval_route
    reason: "落库即为该版本的历史事实，任何接口不提供修改能力"
```

---

## 4. API 接口契约

### 4.1 接口清单（目录）

| API ID | 方法 | 路径 | 用途 | 关联 AC |
|--------|------|------|------|--------|
| API-003H-001 | POST | `/admin-api/staff/drawing/create` | 创建图纸 / 上传新版本（**既有接口，新增 1 个请求参数**） | AC-003H-004 ~ 008、AC-003H-013 |

> **本需求不新增接口**。文件上传（`/drawing/uploadFile`）与 AI 识别查询（`/drawing/recognition/queryJob`）完全沿用 [REQ-003-shared §4.1.1 / §4.1.2](../../requirements/shared/REQ-003-shared.md)，行为不受影响。
>
> **路径写法说明**：本文档按 [`backend-rules.md §8.1`](../../rules/backend-rules.md) 写完整路径（网关前缀 `admin-api` + 服务 `staff` + 模块 `drawing` + 动作 `create`）。REQ-003-shared 中记作 `/drawing/create` 的是同一个端点的简写形式。

---

### 4.2 API-003H-001：创建图纸 / 上传新版本（增量）

**用途**：在既有创建接口上增加审批路径选择，使持有专项权限的上传人可将已完成审批的图纸直接创建为生效版本。

**幂等性**：❌ 否（沿用既有接口语义；重复提交会因 `drawingCode` 唯一约束报 `1003003019`，`OTHERS` 类型无此保护）
**鉴权**：✅ 是（JWT）
**权限**：`drawing:upload`；当 `approvalRoute = PRE_APPROVED` 时**额外**要求 `drawing:upload-approved`

**请求**：

```
POST /admin-api/staff/drawing/create
Content-Type: application/json
Headers:
  Authorization: Bearer {token}
  X-Tenant-Id: {tenantId}
  Project-Id: {projectId}
  lang: {zh-CN | en}
```

**新增请求参数**（其余参数见 [REQ-003-shared §4.1.3](../../requirements/shared/REQ-003-shared.md)，此处不复制）：

| 参数 | 类型 | 必填 | 默认值 | 约束 | 说明 |
|------|------|------|-------|------|------|
| approvalRoute | String | ❌ | `STANDARD` | 枚举，见 §2.1 | 传 `PRE_APPROVED` 时走免审批路径 |

**参数联动规则**：

| 条件 | `approverId`（内部审批人） | 说明 |
|------|------------------------|------|
| `approvalRoute` 缺省或 `STANDARD` | ✅ 必填 | 沿用既有校验 |
| `approvalRoute = PRE_APPROVED` | ❌ 非必填，**若传入则忽略** | 免审批路径不产生内部审批任务；忽略而非报错，避免前端切换路径后残留值导致提交失败 |

**响应：成功**（`approvalRoute = PRE_APPROVED`）

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingId": 412,
    "drawingVersionId": 908,
    "versionNo": "V1",
    "approvalId": null,
    "approvalStatus": "APPROVED",
    "approvalRoute": "PRE_APPROVED"
  }
}
```

> **与 Standard 路径响应的差异**：`approvalId` 为 `null`（未创建审批记录），`approvalStatus` 直接为 `APPROVED`（而非 `PENDING_INTERNAL`）。前端据此判断是否需要提示"已提交审批"或"已直接生效"。

**响应：错误**

| HTTP | 错误码 | 含义 | 触发条件 | 关联 AC |
|------|-------|------|---------|--------|
| 403 | 1003003020 | 无免审批上传权限 | `approvalRoute = PRE_APPROVED` 但操作人不持有 `drawing:upload-approved` | AC-003H-007、AC-003H-013 |
| 400 | 1003003021 | 免审批上传仅适用于新建图纸 | `approvalRoute = PRE_APPROVED` 且请求携带 `drawingId` | AC-003H-006 |
| 500 | 1003007009 | QR 生成失败，请重试 | QR 服务不可用或 OSS 异常；**整体事务回滚** | AC-003H-008 |
| 400 | 1003003019 | 图纸编号已存在 | 沿用既有校验（`SHOP_DRAWING` 且 `drawingCode` 项目内重复） | — |
| 400 | 1003003002 / 1003003003 | 文件格式 / 大小不合规 | 沿用既有校验 | — |
| 400 | 1003003018 | 提交类型缺失或无效 | 沿用既有校验 | — |

**英文提示文案**：

| 错误码 | message |
|--------|---------|
| 1003003020 | You do not have permission to upload a pre-approved drawing |
| 1003003021 | Pre-approved upload is only allowed when creating a new drawing |
| 1003007009 | QR generation failed, please retry |

**统一响应格式**：`CommonResult<T>`（沿用项目既有约定，`code = 0` 为成功；非 0 时 `msg` 为用户可见提示）

**业务校验顺序**（后端必须按此顺序执行，先权限后业务，避免通过报错信息探测资源）：

1. 鉴权：JWT 有效
2. 权限：持有 `drawing:upload`；若 `approvalRoute = PRE_APPROVED`，**额外**校验 `drawing:upload-approved`，否则 `1003003020`
3. 路径适用性：`approvalRoute = PRE_APPROVED` 时请求不得携带 `drawingId`，否则 `1003003021`
4. 项目归属：`fileUrl` 由当前用户在本项目内上传且未被其他记录引用
5. 其余字段校验：完全沿用既有规则，**无任何放宽**（文件、`submissionType`、`drawingCode` 唯一性、`pageCount`）
6. 落库 + QR 生成（同一事务，见 §5）

> ⚠️ **权限须在提交时实时校验**，不得依赖前端渲染状态或登录时缓存的权限集——用户打开弹窗后权限可能被回收（AC-003H-013）。

---

## 5. 事务与一致性

> 本节非模板固定章节，因本需求的核心风险即在事务边界，故显式列出。后端实现细节见 [BACKEND-REQ-003H-pc §5](../backend/BACKEND-REQ-003H-pc.md)。

**同一事务内必须原子完成**（`approvalRoute = PRE_APPROVED` 路径）：

| 顺序 | 操作 |
|:---:|------|
| 1 | INSERT Drawing（`status = ACTIVE`） |
| 2 | INSERT DrawingVersion（`V1`、`APPROVED`、`PRE_APPROVED`、`is_current = true`、`signed_file_url = file_url`、`approved_time = now()`） |
| 3 | QR 生成 + 叠加至 PDF，写回 `qr_image_url` / `pdf_with_qr_url` / `qr_generated_time` |
| 4 | 写审计日志 |

**任一步失败 → 整体回滚**，不允许出现以下中间态：

- ❌ Drawing 已创建但 DrawingVersion 未创建
- ❌ 版本已生效（`is_current = true`）但 `pdf_with_qr_url` 为空
- ❌ 记录已落库但审计日志缺失

> **与 REQ-007 外部审批通过的一致性**：两条路径都在同一事务内完成"生效 + QR"，QR 失败均整体回滚（见 [REQ-007-shared AC-007-008](../../requirements/shared/REQ-007-shared.md)），错误码复用 `1003007009`。不允许两条路径对同一失败给出不同处理。

---

## 6. 异步任务与事件

| 任务 ID | 触发时机 | 处理逻辑 | 失败重试 | 关联 AC |
|--------|---------|---------|---------|--------|
| — | — | **本需求不产生任何异步任务** | — | AC-003H-005 |

**显式说明**：

- QR 生成为**同步**执行（在事务内），不走队列——这是与 REQ-006 §3.1 "审批通过后触发"的实现差异，原因是本路径创建即生效，不存在可供异步补偿的中间态
- `submissionType = SHOP_DRAWING` 的 AI 识别任务仍为异步，但发生在上传阶段（`/drawing/uploadFile`），**与审批路径无关**，两条路径行为一致
- **不发布任何领域事件**：免审批生效不触发 SE 推送（新建图纸默认无分配关系，见 [REQ-003-shared §3.4](../../requirements/shared/REQ-003-shared.md) 规则 1）；后续管理员 [Assign] 时按 [REQ-003D-pc](../../requirements/pc/REQ-003D-pc.md) 既有机制处理

---

## 7. 鉴权与权限模型

### 7.1 鉴权

- 方式：JWT（Bearer Token），沿用现有 SSO 体系
- 请求 Header：`Authorization`、`X-Tenant-Id`、`Project-Id`、`lang`
- 本需求不改变鉴权机制

### 7.2 新增权限项

| 权限标识 | 归属模块 | 默认角色绑定 | 说明 |
|---------|---------|------------|------|
| `drawing:upload-approved` | 图纸管理 | **无**（不写入任何现有角色模板） | 免审批上传。必须与 `drawing:upload` 同时持有才生效；由管理员为具体用户单独勾选 |

### 7.3 权限校验点

| API ID | 校验类型 | 校验逻辑 |
|--------|---------|---------|
| API-003H-001（`approvalRoute` 缺省 / `STANDARD`） | 角色权限 | `hasPermission('drawing:upload')` |
| API-003H-001（`approvalRoute = PRE_APPROVED`） | 角色权限（复合） | `hasPermission('drawing:upload') && hasPermission('drawing:upload-approved')`，缺一返回 `1003003020` |
| API-003H-001（全部路径） | 资源归属 | `fileUrl` 归属当前 `Project-Id`，防跨项目引用 |

> **前端不可信**：前端根据权限控制 Approval Route 区域是否渲染，但后端**必须独立校验**，不接受前端传参作为授权依据（AC-003H-007）。

---

## 8. 数据约束总览

```yaml
constants:
  approval_route:
    default: STANDARD
    values: [STANDARD, PRE_APPROVED]
    immutable_after_insert: true
    column_type: varchar(20)
    nullable: false

  pre_approved_upload:
    applicable_scope: NEW_DRAWING_ONLY        # 携带 drawingId 即拒绝
    version_no: V1                            # 恒为首版
    creates_approval_record: false
    creates_todo: false
    sends_notification: false
    qr_generation: SYNCHRONOUS_IN_TRANSACTION

  file:                                       # 沿用 REQ-003-shared，无放宽
    max_size_mb: 50
    allowed_types_on_create: [PDF]

  error_codes:
    no_pre_approved_permission: 1003003020
    pre_approved_new_drawing_only: 1003003021
    qr_generation_failed: 1003007009          # 复用 REQ-007，不新增重复码

  monitoring:
    pre_approved_ratio_alert_threshold: 0.30  # 免审批占当月新建图纸比例超此值告警
    unauthorized_attempt_alert: 0             # 1003003020 触发次数非零即需人工确认
```

---

## 9. 契约校验机制

- **前端**：`approvalRoute` 作为可选字段加入创建接口的请求类型；缺省时不传，由后端取默认值
- **后端**：`approval_route` 列非空且有默认值，保证新旧代码并存期间行为一致（迁移见需求 §13）
- **契约测试**：QA 需覆盖 §4.2 全部错误码（见 [QA-REQ-003H-pc §6](../qa/pc/QA-REQ-003H-pc.md)）
- **回归保护**：`approvalRoute` 缺省时的响应必须与本需求上线前**完全一致**（AC-003H-009），契约测试须包含该断言

---

## 10. 未决项（Open Questions）

> 引用自 [REQ-003H-pc §17](../../requirements/pc/REQ-003H-pc.md)，仅列影响契约的项。

| OQ ID | 问题 | 对契约的影响 |
|------|------|------------|
| OQ-001 | 是否提供选填的审批凭证补录入口 | 若采纳，需在 DrawingVersion 增加凭证相关字段并新增补录接口。<!-- TODO: 等待 OQ-001 解决 --> |
| OQ-002 | 图纸列表导出是否包含 `approvalRoute` 列 | 若采纳，导出接口响应需增加该列。<!-- TODO: 等待 OQ-002 解决 --> |
| OQ-003 | 是否需要批量导入能力 | 若采纳，需新增批量接口与异步任务定义。<!-- TODO: 等待 OQ-003 解决 --> |

---

## 11. AC 覆盖检查表

| AC ID | 契约体现位置 | 覆盖 |
|------|------------|:---:|
| AC-003H-001 | 无契约体现（纯前端渲染控制），由 §7.2 权限项支撑 | N/A |
| AC-003H-002 | 无契约体现（纯前端渲染控制） | N/A |
| AC-003H-003 | §4.2 参数联动规则（`approverId` 在 PRE_APPROVED 时非必填） | ✅ |
| AC-003H-004 | §3.1 T-003H-001 side_effects、§4.2 成功响应、§5 事务 | ✅ |
| AC-003H-005 | §3.1 not_side_effects、§6 异步任务（显式为空） | ✅ |
| AC-003H-006 | §3.1 forbidden_transitions（含 drawingId）、§4.2 错误码 1003003021 | ✅ |
| AC-003H-007 | §3.1 forbidden_transitions（缺权限）、§4.2 错误码 1003003020、§7.3 | ✅ |
| AC-003H-008 | §4.2 错误码 1003007009、§5 事务回滚约束 | ✅ |
| AC-003H-009 | §4.2 参数默认值 `STANDARD`、§9 回归保护断言 | ✅ |
| AC-003H-010 | 无契约体现（版本历史展示逻辑，读 `approval_route` 渲染） | N/A |
| AC-003H-011 | 无契约体现（列表展示逻辑，读 `approval_route` 渲染） | N/A |
| AC-003H-012 | §1.1 字段间约束第 3 条（`signed_file_url = file_url`）+ QR 生成 | ✅ |
| AC-003H-013 | §4.2 业务校验顺序（提交时实时校验权限） | ✅ |

> 标 N/A 的 4 条为纯 UI 表现类 AC，由 [UI-REQ-003H-pc §11](../ui/pc/UI-REQ-003H-pc.md) 与 [FRONTEND-REQ-003H-pc §13](../frontend/pc/FRONTEND-REQ-003H-pc.md) 覆盖，不属于数据契约职责。

---

## 12. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 | Breaking? |
|-----|------|-------|---------|-----------|
| 0.1.0 | 2026-08-07 | data-contract-agent | 初稿：新增 `approval_route` 字段与 `ApprovalRoute` 枚举；新增状态转换 T-003H-001 及 3 条禁止转换；`/drawing/create` 新增可选参数 `approvalRoute`；新增权限项 `drawing:upload-approved`；新增错误码 1003003020 / 1003003021 | 否（新增字段有默认值，新增参数可选，缺省行为与现状一致） |
