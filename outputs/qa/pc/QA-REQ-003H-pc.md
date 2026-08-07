---
doc_type: qa_spec
req_id: REQ-003H-pc
version: 0.1.0
status: draft
generated_from: REQ-003H-pc@0.1.1
data_contract_ref: DATA-CONTRACT-REQ-003H-pc.md@0.1.0
generated_at: 2026-08-07
generator: qa-agent
owner: ""
---

# QA 测试说明：PC 端 — 免审批上传（已完成审批的图纸直接生效）

> **本文档定义测试场景、用例与 AC 覆盖矩阵**。字段与接口定义引用 [DATA-CONTRACT-REQ-003H-pc](../../shared/DATA-CONTRACT-REQ-003H-pc.md)，**不重复定义**。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003H-pc @ v0.1.1 |
| 数据契约 | DATA-CONTRACT-REQ-003H-pc.md @ v0.1.0 |
| UI / 前端 / 后端说明 | UI-REQ-003H-pc@0.1.0、FRONTEND-REQ-003H-pc@0.1.0、BACKEND-REQ-003H-pc@0.1.0 |
| 覆盖 AC | AC-003H-001 ~ AC-003H-013（13 条，全覆盖） |
| 上次同步时间 | 2026-08-07 |

---

## 1. 测试目标

验证免审批路径能在单事务内产出与正常流程等价的生效版本，且**该捷径不可被无权用户触达、不可被用于改版、不会留下半成品数据**。

本需求的测试重心不在"功能能用"，而在三类**否定性验证**：

1. 无权限用户在 UI 与接口两层都触达不到
2. 不产生任何审批记录、Todo、通知（易被"顺手复用既有流程代码"引入）
3. 失败时不留残留数据（QR 失败必须整体回滚）

以及一类**回归验证**：Standard 路径行为一字未变。

---

## 2. 测试策略

### 2.1 测试金字塔

| 层级 | 占比 | 负责方 |
|-----|:---:|-------|
| 单元 | 50% | 开发 |
| 集成 | 30% | 开发 |
| 契约 | 10% | QA |
| E2E | 10% | QA |

**调整说明**：相比默认（60/25/5/10），集成与契约占比上调。原因是本需求最关键的行为是**事务原子性**与**接口层权限拦截**——两者都无法在单元层验证。

### 2.2 分工

| 类型 | 负责 |
|-----|------|
| 单元 + 集成 | 开发 |
| 契约测试 | QA |
| E2E | QA |
| 权限矩阵验证 | QA（重点，见 §3.3） |

### 2.3 测试账号准备

| 账号 | 权限 | 用途 |
|-----|------|------|
| `qa_uploader_pre` | `drawing:upload` + `drawing:upload-approved` | 免审批路径主账号 |
| `qa_uploader_std` | 仅 `drawing:upload` | 验证区域不渲染、接口被拒 |
| `qa_admin` | `drawing:dc-config` + 权限管理 | 授予 / 回收权限，验证实时生效 |
| `qa_se` | `drawing:view`（无 `drawing:history`） | 验证 SE 视角字段过滤 |
| `qa_approver` | `drawing:approve` | 验证 Todo 未产生 |
| `qa_dc` | `drawing:external-approval` | 验证 DC Todo 未产生、Mark Result 不显示 |

---

## 3. 测试场景总览

### 3.1 主流程场景

**输入**：[REQ-003H-pc §2.1 主流程](../../../requirements/pc/REQ-003H-pc.md)

| 场景 ID | 场景 | 关联 AC |
|--------|------|--------|
| SC-001 | 有权限用户完成一次免审批上传（Shop Drawing），版本直接生效 | AC-003H-002、003、004、012 |
| SC-002 | 有权限用户完成一次免审批上传（Others 类型），版本直接生效 | AC-003H-004 |
| SC-003 | 有权限用户保持 Standard 路径提交，走完整两级审批 | AC-003H-009 |
| SC-004 | 免审批生效后管理员 [Assign] SE，SE 正常查看与扫码 | AC-003H-012 |

### 3.2 异常场景

**输入**：[REQ-003H-pc §2.3 异常流程](../../../requirements/pc/REQ-003H-pc.md)

| 场景 ID | 场景 | 关联 AC |
|--------|------|--------|
| SC-E01 | 无权限用户绕过前端调接口 | AC-003H-007 |
| SC-E02 | 对已有图纸使用免审批路径（携带 drawingId） | AC-003H-006 |
| SC-E03 | QR 服务不可用导致生成失败 | AC-003H-008 |
| SC-E04 | 图纸编号重复 | — |
| SC-E05 | 文件格式 / 大小不合规 | — |
| SC-E06 | 提交过程中权限被回收 | AC-003H-013 |

### 3.3 权限场景

**输入**：[REQ-003H-pc §5 权限矩阵](../../../requirements/pc/REQ-003H-pc.md)

对矩阵中每个 ❌ 单元格派生越权测试：

| 场景 ID | 场景 | 关联 AC |
|--------|------|--------|
| SC-P01 | 仅有 `drawing:upload` 的用户：区域不渲染 | AC-003H-001 |
| SC-P02 | 仅有 `drawing:upload-approved`（无 `drawing:upload`）：接口被拒 | AC-003H-007 |
| SC-P03 | 项目管理员（无上传权限）：无上传入口 | AC-003H-001 |
| SC-P04 | SE 视角：列表不返回也不显示 `approvalRoute` | AC-003H-011 |
| SC-P05 | 跨项目引用 fileUrl | — |
| SC-P06 | 未登录调用接口 | — |

### 3.4 状态转换场景

**输入**：[DATA-CONTRACT §3.1 状态机](../../shared/DATA-CONTRACT-REQ-003H-pc.md)

| 场景 ID | 场景 | 类型 |
|--------|------|------|
| SC-T01 | `null → APPROVED`（T-003H-001）合法转换 | 合法 |
| SC-T-F01 | 携带 drawingId 时禁止转换 | 非法 |
| SC-T-F02 | 缺权限时禁止转换 | 非法 |
| SC-T-F03 | 对免审批版本调 `/drawing/approve` | 非法 |
| SC-T-F04 | 对免审批版本调 `/drawing/external-approve` | 非法 |
| SC-T-F05 | 尝试修改已落库的 `approvalRoute` | 非法 |

> **状态机覆盖率**：T-003H-001 及 3 条 `forbidden_transitions` 全部覆盖，另补充 2 条终态约束验证。

### 3.5 否定性场景（本需求重点）

| 场景 ID | 场景 | 关联 AC |
|--------|------|--------|
| SC-N01 | 免审批后无 DrawingApproval 记录 | AC-003H-005 |
| SC-N02 | 免审批后无 Todo 产生（内部审批人 + DC 双视角） | AC-003H-005 |
| SC-N03 | 免审批后无站内消息、无 App Push | AC-003H-005 |
| SC-N04 | QR 失败后 DB 无任何残留记录 | AC-003H-008 |

---

## 4. 测试用例（TC）详述

### TC-003H-001-01：有权限用户可见 Approval Route 且默认 Standard

```yaml
tc_id: TC-003H-001-01
covers_ac: [AC-003H-002]
scenario: SC-001
priority: P1
test_level: e2e
preconditions:
  - 登录账号 qa_uploader_pre（持有 drawing:upload + drawing:upload-approved）
steps:
  given: 位于图纸管理列表页
  when: 点击 [+ Upload Drawing] 打开新建图纸弹窗
  then:
    - Internal Approver 选择器上方出现 Approval Route 单选区
    - 默认选中 "Standard Approval — internal then external"
    - Internal Approver 选择器显示且带必填星号
```

### TC-003H-001-02：无权限用户 DOM 中不存在该区域

```yaml
tc_id: TC-003H-001-02
covers_ac: [AC-003H-001]
scenario: SC-P01
priority: P0
test_level: e2e
preconditions:
  - 登录账号 qa_uploader_std（仅 drawing:upload）
steps:
  given: 位于图纸管理列表页
  when: 点击 [+ Upload Drawing] 并用 DevTools 检查 DOM
  then:
    - DOM 中不存在 .approval-route-field 元素（非 display:none，是不存在）
    - Internal Approver 正常显示且必填
    - 弹窗其余部分与上线前版本一致（无残留空白、无多余分隔线）
```

> ⚠️ **必须用 DevTools 验证，不能只看肉眼**。`v-show` 与 `v-if` 肉眼无差别，但前者会让无权用户发现该功能存在。

### TC-003H-001-03：项目管理员无上传入口

```yaml
tc_id: TC-003H-001-03
covers_ac: [AC-003H-001]
scenario: SC-P03
priority: P2
test_level: e2e
preconditions:
  - 登录账号 qa_admin（无 drawing:upload）
steps:
  given: 位于图纸管理列表页
  when: 查看顶部操作区
  then: [+ Upload Drawing] 按钮不显示，无从触达 Approval Route
```

### TC-003H-003-01：切换到 Pre-Approved 后豁免内部审批人

```yaml
tc_id: TC-003H-003-01
covers_ac: [AC-003H-003]
scenario: SC-001
priority: P1
test_level: e2e
preconditions:
  - 账号 qa_uploader_pre，弹窗已打开，已选定某内部审批人
steps:
  given: Approval Route 当前为 Standard 且已选审批人"王总工"
  when: 切换 Approval Route 为 Pre-Approved
  then:
    - Internal Approver 整行隐藏（含 label、星号、错误提示）
    - 警示文案出现："This drawing will take effect immediately without internal or external approval. This action is recorded."
    - 警示文案无关闭按钮，不可 dismiss
    - 此时点击 [Confirm] 不因缺少审批人被拦截
```

### TC-003H-003-02：切回 Standard 不回填历史值

```yaml
tc_id: TC-003H-003-02
covers_ac: [AC-003H-003]
scenario: SC-001
priority: P2
test_level: e2e
steps:
  given: 已从 Standard（选过"王总工"）切换到 Pre-Approved
  when: 再切回 Standard
  then:
    - Internal Approver 恢复显示且必填
    - 选择器为空，不回填"王总工"
    - 直接点 [Confirm] 触发必填校验拦截
```

### TC-003H-004-01：免审批提交后版本直接生效（Shop Drawing）

```yaml
tc_id: TC-003H-004-01
covers_ac: [AC-003H-004, AC-003H-012]
scenario: SC-001
priority: P0
test_level: integration
preconditions:
  - 账号 qa_uploader_pre
  - 已上传合法 PDF（≤50MB），AI 识别 DONE
test_data:
  submissionType: SHOP_DRAWING
  drawingCode: "ARCH-QA-001"（项目内唯一）
  approvalRoute: PRE_APPROVED
steps:
  given: 弹窗已填全部必填项，Approval Route = Pre-Approved
  when: 点击 [Confirm] 且后端处理成功
  then:
    - 响应 approvalStatus = "APPROVED"、approvalRoute = "PRE_APPROVED"、approvalId = null、versionNo = "V1"
    - DB：drawing_versions.approval_status = APPROVED
    - DB：drawing_versions.approval_route = PRE_APPROVED
    - DB：drawing_versions.is_current = 1、is_deprecated = 0
    - DB：drawing_versions.signed_file_url = file_url（同一 OSS 地址）
    - DB：drawing_versions.pdf_with_qr_url 非空
    - DB：drawing_versions.approved_time 非空
    - DB：drawings.status = ACTIVE、current_version = 'V1'
    - Toast："Drawing created and activated"
cleanup:
  - 删除测试图纸记录与 OSS 文件
```

### TC-003H-004-02：Others 类型免审批提交

```yaml
tc_id: TC-003H-004-02
covers_ac: [AC-003H-004]
scenario: SC-002
priority: P1
test_level: integration
test_data:
  submissionType: OTHERS
  drawingCode: null
  approvalRoute: PRE_APPROVED
steps:
  given: 选择 Others 类型上传 PDF（不触发 AI 识别）
  when: 选 Pre-Approved 并提交
  then:
    - 版本直接生效，字段断言同 TC-003H-004-01
    - drawing_code 与 drawing_name 落库为 NULL
    - page_count 有值（服务端解析）
    - 不因 drawingCode 为空而报错
```

### TC-003H-005-01：不产生任何审批记录

```yaml
tc_id: TC-003H-005-01
covers_ac: [AC-003H-005]
scenario: SC-N01
priority: P0
test_level: integration
steps:
  given: 已通过免审批路径成功创建图纸（TC-003H-004-01 的结果）
  when: 查询该 drawingVersionId 关联的 DrawingApproval
  then:
    - 记录数 = 0
    - 既无 phase = INTERNAL，也无 phase = EXTERNAL
```

### TC-003H-005-02：不产生任何 Todo（双视角验证）

```yaml
tc_id: TC-003H-005-02
covers_ac: [AC-003H-005]
scenario: SC-N02
priority: P0
test_level: e2e
preconditions:
  - 项目已配置至少一个 DC（qa_dc）
  - 存在内部审批人账号 qa_approver
steps:
  given: 已通过免审批路径成功创建图纸
  when:
    - 以 qa_approver 登录查看 Todo 列表
    - 以 qa_dc 登录查看 Todo 列表
  then:
    - 两个账号的 Todo 列表均不出现该图纸
    - /todo/list 响应中无该 drawingVersionId 相关条目
```

> **为什么双视角**：只查内部审批人不够。若实现时复用了 REQ-007 的通知代码，可能误创建 DC 的外部审批 Todo。

### TC-003H-005-03：不产生通知

```yaml
tc_id: TC-003H-005-03
covers_ac: [AC-003H-005]
scenario: SC-N03
priority: P1
test_level: integration
steps:
  given: 已通过免审批路径成功创建图纸
  when: 检查站内消息表与 App Push 发送日志
  then:
    - 内部审批人无站内消息
    - 项目 DC 无站内消息
    - 无 App Push 记录（新建图纸无 SE 分配关系，推送目标为空集）
```

### TC-003H-006-01：携带 drawingId 时被拒

```yaml
tc_id: TC-003H-006-01
covers_ac: [AC-003H-006]
scenario: SC-E02 / SC-T-F01
priority: P0
test_level: contract
preconditions:
  - 存在任意状态的既有图纸，drawingId = 201
steps:
  given: 构造请求 { drawingId: 201, approvalRoute: "PRE_APPROVED", ... }
  when: POST /admin-api/staff/drawing/create
  then:
    - HTTP 400，code = 1003003021
    - msg = "Pre-approved upload is only allowed when creating a new drawing"
    - DB 中未新增任何 drawing_versions 记录
    - 原图纸 201 的状态与版本数不变
```

### TC-003H-006-02：对各状态的既有图纸均被拒

```yaml
tc_id: TC-003H-006-02
covers_ac: [AC-003H-006]
scenario: SC-E02
priority: P1
test_level: contract
test_data:
  drawingId 分别取状态为 ACTIVE / PENDING_INTERNAL / INTERNAL_REJECTED / EXTERNAL_REJECTED 的图纸
steps:
  given: 对上述每个 drawingId 构造免审批请求
  when: 逐个调用创建接口
  then: 全部返回 1003003021，无一例外通过
```

### TC-003H-007-01：无权限用户绕过前端调接口

```yaml
tc_id: TC-003H-007-01
covers_ac: [AC-003H-007]
scenario: SC-E01 / SC-T-F02
priority: P0
test_level: contract
preconditions:
  - 账号 qa_uploader_std（仅 drawing:upload），取其有效 token
steps:
  given: 直接构造 { approvalRoute: "PRE_APPROVED", ... }（不经前端）
  when: POST /admin-api/staff/drawing/create
  then:
    - HTTP 403，code = 1003003020
    - msg = "You do not have permission to upload a pre-approved drawing"
    - DB 中未创建任何 Drawing / DrawingVersion 记录
    - 响应中不泄露任何资源存在性信息
```

### TC-003H-007-02：仅有 upload-approved 而无 upload 时被拒

```yaml
tc_id: TC-003H-007-02
covers_ac: [AC-003H-007]
scenario: SC-P02
priority: P1
test_level: contract
preconditions:
  - 构造仅持有 drawing:upload-approved 的账号
steps:
  given: 该账号调用免审批创建接口
  when: POST /admin-api/staff/drawing/create
  then: HTTP 403，code = 1003003020（两权限必须同时持有）
```

### TC-003H-008-01：QR 生成失败时整体回滚

```yaml
tc_id: TC-003H-008-01
covers_ac: [AC-003H-008]
scenario: SC-E03 / SC-N04
priority: P0
test_level: integration
preconditions:
  - 通过 mock / 故障注入使 QR 服务返回失败或超时
steps:
  given: 账号 qa_uploader_pre 填妥全部必填项，选 Pre-Approved
  when: 点击 [Confirm]，后端 QR 生成失败
  then:
    - 响应 code = 1003007009，msg = "QR generation failed, please retry"
    - DB：drawings 表无新增记录
    - DB：drawing_versions 表无新增记录
    - DB：audit_logs 无该操作成功记录
    - 图纸列表中不出现该记录
    - 前端：弹窗保持打开，全部已填内容保留，文件无需重传
    - 前端：[Confirm] 恢复可点
cleanup:
  - 恢复 QR 服务
```

### TC-003H-008-02：QR 恢复后当场重试成功

```yaml
tc_id: TC-003H-008-02
covers_ac: [AC-003H-008]
scenario: SC-E03
priority: P1
test_level: e2e
steps:
  given: TC-003H-008-01 执行后弹窗仍打开
  when: 恢复 QR 服务，直接再次点击 [Confirm]（不重新上传文件）
  then:
    - 提交成功，版本正常生效
    - 无需重新选择文件，说明 fileUrl 引用未被清除
```

### TC-003H-009-01：Standard 路径行为完全不变（回归）

```yaml
tc_id: TC-003H-009-01
covers_ac: [AC-003H-009]
scenario: SC-003
priority: P0
test_level: e2e
preconditions:
  - 账号 qa_uploader_pre（有权限，但保持 Standard）
steps:
  given: 弹窗中 Approval Route 保持默认 Standard，选定内部审批人
  when: 提交
  then:
    - 版本 approval_status = PENDING_INTERNAL、approval_route = STANDARD
    - 内部审批人 Todo 列表出现该任务
    - 后续内部审批 → DC 外部审批流程与 REQ-007 完全一致
```

### TC-003H-009-02：无权限用户请求体与上线前一致（回归）

```yaml
tc_id: TC-003H-009-02
covers_ac: [AC-003H-009]
scenario: SC-003
priority: P0
test_level: contract
preconditions:
  - 账号 qa_uploader_std
steps:
  given: 抓取该账号提交创建请求的原始 body
  when: 与上线前基线 body 逐字段比对
  then:
    - 请求体中不含 approvalRoute 字段
    - 其余字段与基线逐字段一致
    - 响应结构与基线一致
```

### TC-003H-010-01：版本历史中 ②③ 显示为跳过

```yaml
tc_id: TC-003H-010-01
covers_ac: [AC-003H-010]
scenario: SC-001
priority: P1
test_level: e2e
preconditions:
  - 存在 approvalRoute = PRE_APPROVED 的 APPROVED 版本
steps:
  given: 账号 qa_uploader_pre 打开该图纸的版本历史抽屉
  when: 展开该版本
  then:
    - ② Internal Approval 卡片显示 "⊘ Skipped — pre-approved upload"
    - ② 副文案显示 "Uploaded as already approved by {uploaderName}"
    - ③ External Approval 卡片显示 "⊘ Skipped — pre-approved upload"
    - ③ 卡片不显示 [Mark Result] 按钮
    - ④ Signed Version 显示文件并标注 "Same file as uploaded"
    - 步骤条为：📄 Uploaded ✓ → 🔍 Internal ⊘ → 🌐 External ⊘ → ✍️ Signed ✓
```

### TC-003H-010-02：DC 账号也看不到 Mark Result

```yaml
tc_id: TC-003H-010-02
covers_ac: [AC-003H-010]
scenario: SC-T-F04
priority: P1
test_level: e2e
preconditions:
  - 账号 qa_dc 为项目已配置 DC
steps:
  given: qa_dc 打开该免审批版本的版本历史并展开
  when: 查看 ③ External Approval 卡片
  then: [Mark Result] 按钮不显示（该版本已是终态，无可执行动作）
```

### TC-003H-010-03：跳过态与未到达态视觉可辨

```yaml
tc_id: TC-003H-010-03
covers_ac: [AC-003H-010]
scenario: SC-001
priority: P2
test_level: e2e
steps:
  given: 同一抽屉中同时存在免审批版本与 PENDING_INTERNAL 版本
  when: 分别展开两个版本，对比 ③ External 卡片
  then:
    - 免审批版本显示 ⊘ Skipped
    - PENDING_INTERNAL 版本显示 — 或 Waiting 文案
    - 两者图标不同，不可仅靠颜色区分
```

### TC-003H-011-01：列表显示 Pre-approved 标识

```yaml
tc_id: TC-003H-011-01
covers_ac: [AC-003H-011]
scenario: SC-001
priority: P1
test_level: e2e
steps:
  given: 列表中同时存在免审批图纸与走完两级审批的图纸
  when: 账号 qa_uploader_pre 查看图纸列表 Status 列
  then:
    - 免审批记录显示 "🟢 Active" + 灰色小字 "Pre-approved"
    - 该记录不附加 (A)/(B)/(D) 外部审批结果代码
    - 正常审批记录显示 "🟢 Active (A)"，不显示 Pre-approved
    - 悬停标识显示 Tooltip
```

### TC-003H-011-02：SE 视角不显示也不返回该字段

```yaml
tc_id: TC-003H-011-02
covers_ac: [AC-003H-011]
scenario: SC-P04
priority: P0
test_level: contract
preconditions:
  - 账号 qa_se（无 drawing:history）
  - 该免审批图纸已通过 [Assign] 分配给 qa_se
steps:
  given: qa_se 调用 /drawing/page
  when: 检查响应 JSON
  then:
    - 响应中 approvalRoute 字段为 null 或不存在（后端过滤）
    - UI 上 Status 列仅显示 "🟢 Active"，无 Pre-approved 标识
    - 前端无法通过任何手段还原该信息
```

### TC-003H-012-01：SE 视角与正常图纸无差异

```yaml
tc_id: TC-003H-012-01
covers_ac: [AC-003H-012]
scenario: SC-004
priority: P1
test_level: e2e
preconditions:
  - 免审批图纸已生效，管理员已 [Assign] 给 qa_se
steps:
  given: qa_se 在 PC 端打开该图纸
  when: 查看、下载、扫描 QR
  then:
    - 可正常查看与下载
    - 下载返回 pdf_with_qr_url（带 QR 的 PDF）
    - 扫描 QR 可打开公开状态页（REQ-006）
    - 展示形态与走完两级审批的图纸完全一致
```

### TC-003H-013-01：提交时权限被回收

```yaml
tc_id: TC-003H-013-01
covers_ac: [AC-003H-013]
scenario: SC-E06
priority: P0
test_level: e2e
preconditions:
  - 账号 qa_uploader_pre 已打开弹窗并选中 Pre-Approved，已填全部必填项
steps:
  given: 弹窗保持打开状态
  when:
    - 由 qa_admin 回收 qa_uploader_pre 的 drawing:upload-approved 权限
    - qa_uploader_pre 点击 [Confirm]
  then:
    - 响应 HTTP 403，code = 1003003020
    - 弹窗保持打开，已填内容全部保留
    - Toast 提示无权限
    - 用户可切换到 Standard 路径、选定审批人后重新提交并成功
```

> **本用例是权限缓存的探针**。若后端缓存了登录态权限集，回收不会立即生效，本用例会失败——这正是它存在的目的。

### TC-003H-T-F05-01：approvalRoute 落库后不可修改

```yaml
tc_id: TC-003H-T-F05-01
covers_ac: [AC-003H-004]
scenario: SC-T-F05
priority: P1
test_level: contract
steps:
  given: 存在 approvalRoute = PRE_APPROVED 的版本
  when: 遍历所有涉及 drawing_versions 的更新接口，尝试传入 approvalRoute = STANDARD
  then:
    - 无任何接口接受该字段
    - DB 中该值保持 PRE_APPROVED 不变
```

---

## 5. 边界值与等价类

**输入**：[DATA-CONTRACT §8 数据约束](../../shared/DATA-CONTRACT-REQ-003H-pc.md)

### 5.1 approvalRoute 字段

| 等价类 | 输入 | 期望 |
|-------|------|-----|
| 有效 | `"STANDARD"` | 走既有流程 |
| 有效 | `"PRE_APPROVED"` | 走免审批流程（需权限） |
| 有效 | 字段缺省 | 取默认 `STANDARD`，行为同上线前 |
| 有效 | `null` | 同缺省，取默认值 |
| 无效 | `""` | 拒绝或按缺省处理（须明确，见 §15 遗留） |
| 无效 | `"pre_approved"`（小写） | 拒绝——枚举值大小写敏感 |
| 无效 | `"PRE-APPROVED"`（连字符） | 拒绝 |
| 无效 | `"UNKNOWN"` | 拒绝 |
| 无效 | 数字 `1` | 类型错误，400 |
| 无效 | 超长字符串（> 20 字符） | 拒绝，不得截断入库 |

### 5.2 文件（沿用既有约束，验证免审批路径无放宽）

| 等价类 | 输入 | 期望 |
|-------|------|-----|
| 有效 | PDF，1 字节 | 接受 |
| 有效 | PDF，恰好 50MB | 接受 |
| 无效 | PDF，50MB + 1 字节 | 拒绝 1003003003 |
| 无效 | DWG / PNG / JPG（新建场景） | 拒绝 1003003002 |
| 无效 | 损坏的 PDF（页数无法解析） | 拒绝 1003003015 |
| 无效 | 伪装扩展名（.pdf 但 magic number 非 PDF） | 拒绝 |

> ⚠️ **关键**：这组用例必须在 `approvalRoute = PRE_APPROVED` 下**全部重跑一遍**。免审批路径最容易被引入的缺陷就是"为了简化而跳过了文件校验"。

### 5.3 drawingCode（Shop Drawing）

| 等价类 | 输入 | 期望 |
|-------|------|-----|
| 有效 | 项目内唯一 | 接受 |
| 无效 | 项目内已存在（同为 PRE_APPROVED） | 拒绝 1003003019 |
| 无效 | 项目内已存在（既有 STANDARD 记录） | 拒绝 1003003019 |
| 有效 | 其他项目已存在同编号 | 接受（唯一性为项目内） |
| 有效 | Others 类型时为 NULL | 接受，且多条 NULL 不冲突 |

### 5.4 文本字段 fuzzing

对 `description` / `drawingName` 施加标准集：emoji、多语言、特殊符号、SQL 注入串、XSS 串、超长文本、空字符串、仅空格。期望与 Standard 路径**完全一致**（沿用既有校验）。

---

## 6. 接口测试（契约测试）

**输入**：[DATA-CONTRACT §4.2](../../shared/DATA-CONTRACT-REQ-003H-pc.md)

### 6.1 API-003H-001 标准用例集

| TC | 描述 | 期望 |
|----|------|-----|
| TC-API-003H-01 | 正常请求（PRE_APPROVED，有权限） | code = 0 |
| TC-API-003H-02 | 缺必填字段（fileUrl） | 400 |
| TC-API-003H-03 | approvalRoute 类型错误 | 400 |
| TC-API-003H-04 | approvalRoute 超长 | 400 |
| TC-API-003H-05 | 未登录 | 401 |
| TC-API-003H-06 | 无 drawing:upload-approved | 403 / 1003003020 |
| TC-API-003H-07 | 携带不存在的 drawingId | 1003003021（路径校验先于存在性校验） |
| TC-API-003H-08 | drawingCode 冲突 | 1003003019 |
| TC-API-003H-09 | 速率限制 | 沿用既有策略 |
| TC-API-003H-10 | 幂等性 | ⚠️ 本接口非幂等，见 §15 |

### 6.2 错误码全覆盖校验

[DATA-CONTRACT §4.2](../../shared/DATA-CONTRACT-REQ-003H-pc.md) 错误码表中每个码必须有对应 TC：

| 错误码 | 覆盖 TC |
|--------|--------|
| 1003003020 | TC-003H-007-01、TC-003H-007-02、TC-003H-013-01 |
| 1003003021 | TC-003H-006-01、TC-003H-006-02 |
| 1003007009 | TC-003H-008-01 |
| 1003003019 | §5.3 边界值 |
| 1003003002 / 003 / 015 / 018 | §5.2 边界值 |

### 6.3 响应结构断言

```yaml
# PRE_APPROVED 成功响应必须满足
assert:
  - data.approvalStatus == "APPROVED"
  - data.approvalRoute == "PRE_APPROVED"
  - data.approvalId == null          # 关键：不得为任何数字
  - data.versionNo == "V1"
```

### 6.4 读接口的字段新增

| 接口 | 断言 |
|------|------|
| `/drawing/page` | 非 SE 角色响应含 `approvalRoute`；SE 角色不含 |
| `/drawing/version/list` | 响应含 `approvalRoute` |
| `/drawing/get` | 版本节点含 `approvalRoute` |

---

## 7. 性能测试

**输入**：[REQ-003H-pc §10.1](../../../requirements/pc/REQ-003H-pc.md) + [BACKEND §12](../../backend/BACKEND-REQ-003H-pc.md)

| 场景 ID | 场景 | 通过条件 |
|--------|------|---------|
| PT-001 | 免审批提交（含 QR 生成）单次耗时 | P95 ≤ 5 秒 |
| PT-002 | 50MB PDF 的免审批提交 | P95 ≤ 5 秒，成功率 > 99% |
| PT-003 | 10 并发免审批提交（不同 drawingCode） | 全部成功，P95 ≤ 5 秒，无死锁 |
| PT-004 | 事务持有时长 | P95 ≤ 5 秒；监控无长事务告警 |

**重点关注**：BACKEND §5.2 记录了"QR 调用在事务内"的规则偏离。PT-003 与 PT-004 用于验证该取舍在并发下不引发连接池耗尽或锁等待。

---

## 8. 安全测试

**强制覆盖 OWASP Top 10 相关项**：

| 项 | 测试 |
|---|------|
| A01 越权 | TC-003H-007-01/02；SC-P05 跨项目引用 fileUrl；SC-P06 未登录 |
| A01 越权 | 修改 JWT 中权限声明后调用接口，验证服务端不信任 token 内声明的权限而是实时查库 |
| A03 注入 | `description` / `drawingName` 的 SQL / NoSQL 注入串（§5.4） |
| A03 注入 | `approvalRoute` 传入 SQL 片段 |
| A07 鉴权 | TC-003H-013-01 权限回收实时生效 |
| A09 日志监控 | 验证每次免审批创建都产生不可删除的审计记录；验证 1003003020 触发时产生安全告警 |
| 文件安全 | magic number 校验、路径穿越、病毒文件（沿用既有，在免审批路径下重跑） |

**本需求专项安全用例**：

```yaml
tc_id: TC-003H-SEC-01
covers_ac: [AC-003H-007]
priority: P0
test_level: contract
steps:
  given: 无权限账号，尝试通过各种方式伪造授权
  when:
    - 请求头注入伪造的权限声明
    - 篡改 JWT payload 中的权限列表（不重签名）
    - 篡改 JWT 并用弱密钥重签名
  then: 全部返回 403 / 1003003020，无一绕过
```

---

## 9. 兼容性测试

取自 [background/tech-stack.md §1](../../../background/tech-stack.md)（PC 管理端）：

| 项 | 范围 |
|---|------|
| 浏览器 | Chrome 80+、Firefox 78+、Safari 14+、Edge 80+ |
| 最低分辨率 | 1280×720px |
| 推荐分辨率 | 1920×1080px |
| 移动端 | 不适用（PC 专属） |
| 国际化 | 中 / 英双语 |

**重点用例**：

| TC | 描述 | 期望 |
|----|------|-----|
| TC-COMPAT-01 | 1280×720 下查看列表 Status 列 | `🟢 Active` + `Pre-approved` 不折行、不截断 |
| TC-COMPAT-02 | 英文语境下打开弹窗 | 单选项长文案完整展示（允许换行），不出现省略号 |
| TC-COMPAT-03 | 英文语境下警示文案 | 三行完整展示，容器高度自适应，不滚动 |
| TC-COMPAT-04 | 各浏览器下键盘操作单选组 | Tab 聚焦 + 方向键切换均正常 |

---

## 10. 测试数据

### 10.1 种子数据

| 数据 | 数量 | 说明 |
|-----|:---:|------|
| 测试账号 | 6 | 见 §2.3 |
| 既有图纸（各状态） | 4 | ACTIVE / PENDING_INTERNAL / INTERNAL_REJECTED / EXTERNAL_REJECTED，用于 TC-003H-006-02 |
| 存量 STANDARD 版本 | ≥ 2 | 验证迁移后 approval_route 回填正确 |
| 合法 PDF | 3 | 1 字节 / 常规 / 恰好 50MB |
| 异常文件 | 4 | 50MB+1B、DWG、损坏 PDF、伪装扩展名 |

### 10.2 fixtures 结构

```
fixtures/req-003h/
├── valid/
│   ├── shop-drawing-normal.pdf
│   ├── shop-drawing-50mb.pdf
│   ├── shop-drawing-1byte.pdf
│   └── others-normal.pdf
├── invalid/
│   ├── oversized-50mb-plus.pdf
│   ├── corrupted.pdf
│   ├── wrong-format.dwg
│   └── fake-extension.pdf
└── security/
    ├── xss-payloads.txt
    ├── sql-injection.txt
    └── path-traversal.txt
```

### 10.3 迁移验证数据

上线后须验证：`SELECT COUNT(*) FROM drawing_versions WHERE approval_route IS NULL` 返回 0，且全部存量记录为 `STANDARD`。

---

## 11. AC 覆盖矩阵

| AC ID | 描述简述 | 覆盖的 TC | 状态 |
|------|---------|----------|:---:|
| AC-003H-001 | 无权限用户看不到 Approval Route | TC-003H-001-02、TC-003H-001-03 | ✅ |
| AC-003H-002 | 有权限可见且默认 Standard | TC-003H-001-01 | ✅ |
| AC-003H-003 | 切换后豁免内部审批人 | TC-003H-003-01、TC-003H-003-02 | ✅ |
| AC-003H-004 | 提交后版本直接生效 | TC-003H-004-01、TC-003H-004-02、TC-003H-T-F05-01 | ✅ |
| AC-003H-005 | 不产生审批记录与 Todo | TC-003H-005-01、TC-003H-005-02、TC-003H-005-03 | ✅ |
| AC-003H-006 | 不可用于上传新版本 | TC-003H-006-01、TC-003H-006-02 | ✅ |
| AC-003H-007 | 绕过前端调接口被拦截 | TC-003H-007-01、TC-003H-007-02、TC-003H-SEC-01 | ✅ |
| AC-003H-008 | QR 失败整体回滚 | TC-003H-008-01、TC-003H-008-02 | ✅ |
| AC-003H-009 | Standard 路径不受影响 | TC-003H-009-01、TC-003H-009-02 | ✅ |
| AC-003H-010 | 版本历史 ②③ 显示跳过 | TC-003H-010-01、TC-003H-010-02、TC-003H-010-03 | ✅ |
| AC-003H-011 | 列表显示 Pre-approved | TC-003H-011-01、TC-003H-011-02 | ✅ |
| AC-003H-012 | SE 视角与正常图纸无差异 | TC-003H-012-01、TC-003H-004-01 | ✅ |
| AC-003H-013 | 提交时权限实时校验 | TC-003H-013-01 | ✅ |

**覆盖率**：13 / 13 = **100%**，共 24 条 TC。

**双向校验**：

- 每个 AC → 至少 1 条 TC ✅
- 每条 TC → 至少关联 1 个有效 AC ✅

**自动校验伪代码**：

```pseudo
for ac in requirement.acs:
  tcs = qa_spec.find_tcs_covering(ac.id)
  assert len(tcs) >= 1, f"AC {ac.id} 未被覆盖"
for tc in qa_spec.tcs:
  for ac_id in tc.covers_ac:
    assert ac_id in requirement.ac_ids, f"TC {tc.id} 引用了不存在的 AC {ac_id}"
```

---

## 12. 回归测试范围

**输入**：[REQ-003H-pc §12 外部系统依赖](../../../requirements/pc/REQ-003H-pc.md) + front matter `depends_on` / `related_to`

| 受影响模块 | 回归理由 | 回归重点 |
|-----------|---------|---------|
| **REQ-003E-pc**（上传弹窗） | 弹窗新增区域，共用表单与提交逻辑 | Standard 路径全流程；AI 识别流程不受影响；Others 路径不受影响 |
| **REQ-007-shared**（两级审批） | 共用创建接口与状态机 | 内部审批 → 外部审批完整链路；状态机其余转换不受影响 |
| **REQ-007A-pc**（内部审批 Todo） | 免审批不应产生 Todo | Todo 列表在免审批操作后无新增 |
| **REQ-007B-pc**（DC 外部审批） | 同上 | DC Todo 无新增；Mark Result 对免审批版本不显示 |
| **REQ-007C-pc**（版本历史抽屉） | 新增渲染分支 | 既有 5 种状态分支渲染不受影响 |
| **REQ-003A-pc**（图纸列表） | Status 列新增标识、列宽变化 | 既有状态标签与 (A)/(B)/(D) 代码显示正常；导出功能不受影响 |
| **REQ-006-shared**（QR） | 新增一个 QR 生成调用点 | 既有"外部审批通过后生成 QR"路径不受影响 |
| **REQ-003D-pc**（SE 分配） | 免审批图纸也需可分配 | [Assign] 对免审批图纸正常工作，分配后 SE 收到通知 |
| **用户权限模块** | 新增权限项 | 既有权限项不受影响；新权限可正常授予 / 回收 |
| **数据库迁移** | 新增列 | 存量记录 approval_route 全部为 STANDARD，无空值 |

---

## 13. 测试环境

| 项 | 要求 |
|---|------|
| 环境 | 测试环境（可注入 QR 服务故障） |
| 数据库 | 需支持事务回滚验证，可直接查表断言 |
| QR 服务 | 需可 mock 或人为制造故障（TC-003H-008-01 必需） |
| 权限模块 | 需可实时授予 / 回收（TC-003H-013-01 必需） |
| OSS | 测试桶，可验证文件与 QR 叠加结果 |
| 抓包 | 需能抓取并比对请求体（TC-003H-009-02 必需） |

---

## 14. 验收条件

- [ ] §11 AC 覆盖矩阵全部 ✅，无 ❌ 或 🟡
- [ ] 全部 P0 用例通过（共 9 条：001-02、004-01、005-01、005-02、006-01、007-01、008-01、009-01、009-02、011-02、013-01、SEC-01）
- [ ] §6.2 错误码全覆盖
- [ ] §12 回归范围全部执行且无新增缺陷
- [ ] §5.2 文件边界用例在免审批路径下全部重跑通过
- [ ] PT-001 ~ PT-004 达标
- [ ] 迁移验证：存量数据 approval_route 无空值

---

## 15. 已知问题与遗留

| 项 | 说明 | 处置 |
|---|------|------|
| 接口非幂等（Others 类型） | `OTHERS` 无 `drawingCode` 唯一约束，重复提交会创建两条生效记录。该缺口在 Standard 路径同样存在，属既有接口固有行为（见 [BACKEND §6](../../backend/BACKEND-REQ-003H-pc.md)） | 不在本需求修复。建议作为独立技术需求提出，覆盖两条路径 |
| `approvalRoute = ""` 的行为未定义 | 需求与契约均未明确空字符串应拒绝还是按缺省处理 | <!-- TODO: 需 PM / 后端 TL 确认。测试执行前须明确，否则 §5.1 该行用例无法判定 --> |
| QR 服务超时阈值未定 | [BACKEND §12.2](../../backend/BACKEND-REQ-003H-pc.md) 建议 3 秒但标注待评审 | <!-- TODO: 阈值确定后补充对应超时用例 --> |
| OQ-001 审计缺口 | 免审批版本无凭证可查，测试无法验证"该图纸确实审批过" | 属需求已知取舍，测试范围内仅验证 approvalRoute + 上传人 + 时间三者留痕完整 |
| REQ-007C 组件扩展性未确认 | [UI OQ-UI-001](../../ui/pc/UI-REQ-003H-pc.md)：4 阶段卡片组件是否支持新状态分支 | 若需改造组件，TC-003H-010-* 的执行时间需相应后移 |

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-08-07 | qa-agent | 初稿：24 条 TC 覆盖全部 13 条 AC（100%）；重点设计否定性场景（无审批记录 / 无 Todo / 无通知 / 无残留数据）与 Standard 路径回归；补充权限缓存探针用例、事务回滚验证、文件边界在免审批路径下的重跑要求 |
