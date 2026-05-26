---
doc_type: qa_spec
req_id: REQ-003E-pc
version: 0.2.0
status: draft
generated_from: REQ-003E-pc.md@0.3.1
data_contract_ref: data-contract.md@0.1.0
generated_at: 2026-05-24
updated_at: 2026-05-25
owner: ""
---

# QA 测试说明：PC 端 — 图纸上传 AI 自动识别页信息

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一，格式：`TC-003E-XXX`。
> - 接口测试字段定义引用 data-contract.md。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003E-pc @ v0.3.1 |
| 数据契约 | data-contract.md @ v0.1.0 |
| 覆盖 Story | US-003E-001、US-003E-002 |
| 覆盖 AC | AC-003E-001 ~ AC-003E-013 |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证 PC 端"图纸上传 AI 自动识别页信息"功能：PDF 上传后 AI 识别流程（识别中状态、识别成功结果列表、Drawing Code/Name 自动填入）、AI 识别失败降级路径、最终仅创建一条 Drawing 记录并关联 AI 来源信息，以及文件格式/大小校验等边界条件。

---

## 2. 测试策略

### 2.1 测试金字塔

| 层级 | 占比 | 由谁写 | 框架 |
|-----|-----|-------|------|
| 单元测试 | 60% | 前端/后端开发 | JUnit 5（后端）/ Jest（前端）|
| 集成测试 | 25% | 开发 + QA | Testcontainers（后端）/ Cypress Component（前端）|
| 契约测试 | 5% | QA | schemathesis |
| E2E 测试 | 10% | QA | Cypress（PC 端）|

### 2.2 测试范围

**必测**：
- AI 识别成功完整主流程（上传 → 识别 → 填入 → 提交 → 创建）
- AI 识别失败降级流程（降级提示 / 手动填写 / Re-upload）
- 文件格式与大小校验
- 识别中状态的按钮/输入框禁用控制
- 识别结果中有未识别页的展示（⚠ 标记）
- Drawing 创建后 AI 来源信息持久关联（详情页可查）
- 提交后仅创建一条 Drawing 记录（非多条）
- 权限控制（非设计人员无法触发上传）

**不测（本期）**：
- 上传新版本时的 AI 识别（Out of Scope）
- APP 端（不支持上传）
- DWG / DXF 等非 PDF 格式的识别（Out of Scope）
- AI 识别引擎准确率本身（由 AI 团队负责验收）
- 审批流程变更（沿用 REQ-007-shared，不在本需求范围内）

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-001 | 上传合法 PDF → AI 识别成功 → 自动填入 → 提交 → 创建单条 Drawing | US-003E-001 | P0 |
| SC-002 | AI 识别完成，识别结果中部分页未识别到信息（⚠ 展示）| US-003E-001 | P1 |
| SC-003 | AI 识别成功，用户手动修改 AI 预填的 Drawing Code 后提交 | US-003E-002 | P1 |
| SC-004 | AI 识别失败 → 手动填写 Drawing Code/Name → 正常提交 | US-003E-001 | P0 |
| SC-005 | AI 识别失败 → 点击 Re-upload → 上传新文件 → 重新触发识别 | US-003E-001 | P1 |
| SC-006 | Drawing 创建成功后在详情页查看 AI 来源页面信息 | US-003E-001 | P1 |
| SC-007 | 降级手动上传的 Drawing，详情页不显示 AI 识别区域 | US-003E-001 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-E01 | 上传非 PDF 文件（.dwg）| AC-003E-010 |
| SC-E02 | 上传超过 50MB 的 PDF | AC-003E-011 |
| SC-E03 | AI 识别进行中点击 Submit（应被阻止）| AC-003E-009 |
| SC-E04 | Drawing Code 与项目内已有图纸重复，提交返回错误 | REQ-003A 沿用 |
| SC-E05 | 文件上传过程中网络中断 | REQ-003E §6.3 |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-P01 | 内部审批人尝试访问上传弹窗，应无 [+ Upload Drawing] 按钮 |
| SC-P02 | 非当前项目成员尝试查询其他项目的识别任务 Job，应返回 403/404 |

### 3.4 状态转换场景

**合法转换测试**：

| 场景 ID | From | Action | To | 测试要点 |
|--------|------|--------|-----|---------|
| SC-T01 | — | 上传 PDF | PENDING | 任务创建，前端显示 Loading |
| SC-T02 | PENDING | AI 开始处理 | PROCESSING | 状态更新，前端继续 Loading |
| SC-T03 | PROCESSING | AI 识别完成 | DONE | 结果列表展示，输入框填入 |
| SC-T04 | PROCESSING | AI 识别失败 | FAILED | 降级提示横幅展示 |
| SC-T05 | PENDING | 等待超时 | FAILED | 降级提示横幅展示 |

**非法转换测试**：

| 场景 ID | From | 尝试 Action | 期望 |
|--------|------|------------|-----|
| SC-T-F01 | DONE | 重新触发识别（不重新上传） | 拒绝，需重新上传文件 |
| SC-T-F02 | FAILED | 直接置为 DONE | 拒绝，422 |

---

## 4. 测试用例（TC）详述

### TC-003E-001-01

```yaml
tc_id: TC-003E-001-01
covers_ac: [AC-003E-001]
scenario: SC-001, SC-T01
priority: P0
test_level: e2e
preconditions:
  - 用户已登录，具有 drawing:create 权限
  - 处于 Drawing Masterlist 页面
  - 准备合法 PDF 文件（< 50MB，含多页图纸）

steps:
  given: 用户打开 Upload New Drawing 弹窗，填写 Category
  when: |
    1. 拖拽合法 PDF 文件到上传区
    2. 观察弹窗状态变化
  then:
    - 文件上传进度条正常显示
    - 上传完成后显示 Loading 动画和文案 "Analysing drawing pages…"
    - Drawing Code 输入框处于置灰不可编辑状态
    - Drawing Name 输入框处于置灰不可编辑状态
    - [Submit] 按钮处于置灰不可点状态

cleanup:
  - 关闭弹窗（不提交）
```

### TC-003E-002-01

```yaml
tc_id: TC-003E-002-01
covers_ac: [AC-003E-002]
scenario: SC-001, SC-T03
priority: P0
test_level: e2e
preconditions:
  - TC-003E-001-01 前置状态：文件已上传，AI 识别任务已创建
  - Mock / 等待 AI 识别任务状态变为 DONE（可在 staging 使用 mock AI 服务）

steps:
  given: AI 识别任务 status = DONE，包含 3 页识别结果
  when: |
    1. 前端轮询收到 DONE 状态
    2. 观察弹窗更新
  then:
    - 识别结果列表出现，展示 3 行
    - 每行包含：页码（Page 1/2/3）、缩略图、Drawing No、Drawing Name
    - 顶部显示 "3 pages detected. Drawing information has been pre-filled below. Please review and confirm."
    - Drawing Code 输入框自动填入 AI 建议值（非空）
    - Drawing Name 输入框自动填入 AI 建议值（非空）
    - 两个输入框恢复可编辑状态

cleanup:
  - 关闭弹窗
```

### TC-003E-003-01

```yaml
tc_id: TC-003E-003-01
covers_ac: [AC-003E-003]
scenario: SC-003
priority: P1
test_level: e2e
preconditions:
  - AI 识别完成，Drawing Code 输入框已自动填入 "ARCH-001"
  - 其他必填字段（Category、Drawing Name、Internal Approver）均已填写

steps:
  given: Drawing Code 输入框显示 "ARCH-001"
  when: |
    1. 点击 Drawing Code 输入框
    2. 清空内容，输入 "ARCH-001-REV1"
    3. 点击输入框外部区域（失焦）
  then:
    - Drawing Code 输入框显示 "ARCH-001-REV1"
    - 识别结果列表内容不变（只读，不联动）
    - [Submit] 按钮仍可点击

cleanup:
  - 关闭弹窗
```

### TC-003E-004-01

```yaml
tc_id: TC-003E-004-01
covers_ac: [AC-003E-004]
scenario: SC-002
priority: P1
test_level: e2e
preconditions:
  - AI 识别完成，共 4 页，其中 Page 3 的 Drawing No 和 Drawing Name 均未识别到（值为空）

steps:
  given: AI 识别结果包含未识别页
  when: 前端渲染识别结果列表
  then:
    - Page 3 的 Drawing No 单元格显示 "—" 并带橙色 ⚠ 图标
    - Page 3 的 Drawing Name 单元格显示 "—" 并带橙色 ⚠ 图标
    - 顶部追加橙色提示 "Some pages could not be fully analysed. Please verify the drawing information below."

cleanup:
  - 关闭弹窗
```

### TC-003E-005-01

```yaml
tc_id: TC-003E-005-01
covers_ac: [AC-003E-005]
scenario: SC-001
priority: P0
test_level: e2e
preconditions:
  - AI 识别完成（5 页），Drawing Code、Drawing Name 已自动填入
  - 所有必填字段已填写（Category、Internal Approver）

steps:
  given: 弹窗所有必填项已填写，AI 识别已完成
  when: 点击 [Submit] 按钮
  then:
    - 系统仅创建 1 条 Drawing 记录（状态 PENDING_INTERNAL）
    - 弹窗关闭
    - 图纸列表刷新出现 1 条新记录
    - Snackbar 显示 "Drawing uploaded successfully. Pending internal approval."
    - 内部审批人 Todo 列表出现 1 条（非多条）新任务

cleanup:
  - 删除测试创建的 Drawing 记录
```

### TC-003E-006-01

```yaml
tc_id: TC-003E-006-01
covers_ac: [AC-003E-006]
scenario: SC-004, SC-T04
priority: P0
test_level: e2e
preconditions:
  - 文件已上传，AI 识别任务 status 变为 FAILED（通过 mock 或触发超时）

steps:
  given: AI 识别任务 status = FAILED
  when: 前端收到失败通知（轮询）
  then:
    - 识别结果列表区域显示橙色提示横幅
    - 横幅文案包含 "Unable to analyse the drawing automatically."
    - 提供 [Re-upload] 按钮
    - Drawing Code 输入框恢复为空、可编辑
    - Drawing Name 输入框恢复为空、可编辑

cleanup:
  - 关闭弹窗
```

### TC-003E-007-01

```yaml
tc_id: TC-003E-007-01
covers_ac: [AC-003E-007]
scenario: SC-004
priority: P1
test_level: e2e
preconditions:
  - AI 识别失败，Drawing Code / Name 输入框恢复为空可编辑

steps:
  given: AI 识别失败，降级提示横幅已显示
  when: |
    1. 在 Drawing Code 输入框输入 "MANUAL-001"
    2. 在 Drawing Name 输入框输入 "Manual Drawing Name"
    3. 填写 Category、Internal Approver 必填字段
    4. 点击 [Submit]
  then:
    - 创建成功，弹窗关闭
    - 图纸列表出现 1 条新记录，状态 PENDING_INTERNAL
    - Snackbar 提示成功

cleanup:
  - 删除测试创建的 Drawing 记录
```

### TC-003E-008-01

```yaml
tc_id: TC-003E-008-01
covers_ac: [AC-003E-008]
scenario: SC-005
priority: P1
test_level: e2e
preconditions:
  - AI 识别失败，降级提示横幅已显示
  - 准备一份新的合法 PDF 文件

steps:
  given: AI 识别失败，弹窗显示降级提示
  when: |
    1. 点击 [Re-upload] 按钮
    2. 选择新的合法 PDF 文件
  then:
    - 原文件清空
    - 弹窗重新进入识别中状态（Loading 动画 + "Analysing drawing pages…"）
    - Drawing Code / Name 输入框重新置灰
    - 触发新一轮 AI 识别（新的 jobId 产生）

cleanup:
  - 关闭弹窗
```

### TC-003E-009-01

```yaml
tc_id: TC-003E-009-01
covers_ac: [AC-003E-009]
scenario: SC-E03, SC-T01, SC-T02
priority: P0
test_level: e2e
preconditions:
  - 文件已上传，AI 识别任务处于 PENDING 或 PROCESSING 状态

steps:
  given: AI 识别任务 status 为 PENDING 或 PROCESSING
  when: 用户查看弹窗
  then:
    - [Submit] 按钮处于置灰不可点状态
    - Drawing Code 输入框不可编辑（disabled）
    - Drawing Name 输入框不可编辑（disabled）

cleanup:
  - 等待识别完成或关闭弹窗
```

### TC-003E-010-01

```yaml
tc_id: TC-003E-010-01
covers_ac: [AC-003E-010]
scenario: SC-E01
priority: P0
test_level: e2e
preconditions:
  - 用户打开 Upload New Drawing 弹窗

steps:
  given: 弹窗文件上传区处于空初始状态
  when: 用户尝试选择 .dwg 格式文件
  then:
    - 系统拒绝该文件
    - 提示文案包含 "AI recognition only supports PDF format."
    - 文件上传区保持清空状态，未显示任何文件

cleanup:
  - 无需清理
```

### TC-003E-011-01

```yaml
tc_id: TC-003E-011-01
covers_ac: [AC-003E-011]
scenario: SC-E02
priority: P0
test_level: e2e
preconditions:
  - 用户打开 Upload New Drawing 弹窗
  - 准备一个 > 50MB 的 PDF 文件（或修改文件大小以超限）

steps:
  given: 弹窗文件上传区处于空初始状态
  when: 用户选择大小 > 50MB 的 PDF 文件
  then:
    - 系统拒绝该文件
    - 提示文案包含文件超过大小限制（50MB）
    - 文件上传区保持清空状态

cleanup:
  - 无需清理
```

### TC-003E-012-01

```yaml
tc_id: TC-003E-012-01
covers_ac: [AC-003E-012]
scenario: SC-006
priority: P1
test_level: e2e
preconditions:
  - 已通过 AI 识别流程成功创建一条 Drawing 记录（jobId 已关联）
  - 导航至 Drawing 详情页

steps:
  given: Drawing 详情页已打开（该 Drawing 通过 AI 识别流程创建）
  when: 查看详情页
  then:
    - 页面中存在 "AI 识别结果" 区域
    - 显示本次识别的 PDF 总页数
    - 列表展示每页的页码、Drawing No、Drawing Name
    - 内容与上传时弹窗识别列表中显示的内容一致

cleanup:
  - 无需清理（只读操作）
```

### TC-003E-013-01

```yaml
tc_id: TC-003E-013-01
covers_ac: [AC-003E-013]
scenario: SC-007
priority: P1
test_level: e2e
preconditions:
  - AI 识别失败，设计人员选择手动输入后成功创建 Drawing 记录（ai_recognition_job_id = null）
  - 导航至该 Drawing 详情页

steps:
  given: Drawing 详情页已打开（该 Drawing 通过降级手动输入创建）
  when: 查看详情页
  then:
    - 不展示 "AI 识别结果" 区域
    - 或展示提示文字 "本次上传未使用 AI 识别"（取决于实现）
    - 其他字段（Drawing Code、Drawing Name、Category 等）正常展示，不受影响

cleanup:
  - 无需清理（只读操作）
```

---

## 5. 边界值与等价类

### 5.1 文件大小边界

| 等价类 | 边界值 | 期望 |
|-------|-------|-----|
| 有效 | 1KB（最小合法 PDF） | 接受，触发识别 |
| 有效 | 50MB（恰好等于限制） | 接受，触发识别 |
| 无效 | 50MB + 1B | 拒绝，提示超出大小限制 |
| 无效 | 0B | 拒绝，提示文件为空 |

### 5.2 文件类型边界

| 输入类型 | 期望 |
|---------|-----|
| `application/pdf`（合法 PDF） | 接受 |
| `.pdf` 扩展名但 MIME 为 `application/octet-stream` | 拒绝（后端服务端校验魔数）|
| `.dwg` | 拒绝 |
| `.png` / `.jpg` | 拒绝 |
| 修改扩展名为 `.pdf` 的 Word 文档 | 拒绝（魔数不匹配）|

### 5.3 识别结果页数边界

| 等价类 | 页数 | 期望 |
|-------|------|-----|
| 有效 | 1 页 | 正常识别，列表展示 1 行 |
| 有效 | 100 页（最大，OQ-003 待确认）| 正常识别，列表滚动展示 |
| 边界 | 0 页（PDF 无法解析）| status = FAILED，降级提示 |

### 5.4 Drawing Code 文本字段

| 输入 | 期望 |
|-----|-----|
| 含中文 | 接受，正确存储展示 |
| 含特殊符号（`/`, `-`, `_`）| 接受（遵循 REQ-003-shared 编码规则）|
| 超长文本（超过字段限制）| 拒绝，字段级错误提示 |
| SQL 注入字符串 | 安全转义，正确存储 |
| XSS 字符串 | 安全转义，展示时不执行 |

---

## 6. 接口测试（契约测试）

### 6.1 通用契约校验

使用 schemathesis 基于 data-contract.md OpenAPI 规范自动生成：
- 必填字段缺失 → 400
- 字段类型错误 → 400
- 未登录 → 401
- 无权限角色 → 403

### 6.2 针对各 API 的专项接口测试

#### API: `drawing/uploadFile`（文件上传 + 创建识别任务）

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003E-FILE-01 | 正常上传合法 PDF | 合法 PDF ≤ 50MB | 200，返回 `{ jobId, status: 'PENDING' }` |
| TC-API-003E-FILE-02 | 上传非 PDF 文件 | MIME = image/png | 400，`FILE_TYPE_NOT_SUPPORTED` |
| TC-API-003E-FILE-03 | 上传超 50MB 文件 | 文件 > 50MB | 400，`FILE_SIZE_EXCEEDED` |
| TC-API-003E-FILE-04 | 未登录 | 无 token | 401 |
| TC-API-003E-FILE-05 | 无 drawing:create 权限 | 低权限 token | 403 |

#### API: `drawing/recognition/queryJob`（查询识别任务状态）

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003E-JOB-01 | 查询 PENDING 状态 Job | 合法 jobId，status=PENDING | 200，`{ status: 'PENDING' }` |
| TC-API-003E-JOB-02 | 查询 DONE 状态 Job | 合法 jobId，status=DONE | 200，含 pages 数组 |
| TC-API-003E-JOB-03 | 查询 FAILED 状态 Job | 合法 jobId，status=FAILED | 200，含 failureReason |
| TC-API-003E-JOB-04 | 查询不存在的 Job | 随机 UUID | 404 |
| TC-API-003E-JOB-05 | 查询其他项目的 Job（越权）| 其他项目 jobId | 403 或 404 |

#### API: `drawing/create`（扩展，含 aiRecognitionJobId）

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003E-CREATE-01 | 正常创建（含 jobId）| 合法参数 + 有效 jobId | 200，Drawing 创建成功，`ai_recognition_job_id` 有值 |
| TC-API-003E-CREATE-02 | 降级创建（不含 jobId）| 合法参数，`aiRecognitionJobId=null` | 200，Drawing 创建成功，`ai_recognition_job_id` 为 null |
| TC-API-003E-CREATE-03 | Drawing Code 重复 | 项目内已存在该 Code | 409，`DRAWING_CODE_DUPLICATE` |
| TC-API-003E-CREATE-04 | jobId 识别中状态（PENDING/PROCESSING） | `aiRecognitionJobId` = 识别中的 jobId | 422，`AI_RECOGNITION_IN_PROGRESS` |
| TC-API-003E-CREATE-05 | 缺少必填字段 Drawing Code | `drawingCode = ""` | 400 |
| TC-API-003E-CREATE-06 | 缺少 Internal Approver | `internalApproverId = null` | 400 |
| TC-API-003E-CREATE-07 | jobId 识别失败（FAILED），用户手动填写提交 | 合法参数 + status=FAILED 的 jobId | 200，Drawing 创建成功，`ai_recognition_job_id` 有值 |

#### API: `drawing/getDetail`（扩展，含 aiRecognitionInfo）

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003E-DETAIL-01 | 查询 AI 识别成功创建的 Drawing | 合法 drawingId，job.status=DONE | 200，`aiRecognitionInfo` 非 null，含 totalPages 及 pages 数组 |
| TC-API-003E-DETAIL-02 | 查询降级手动输入创建的 Drawing（无 jobId）| 合法 drawingId，ai_recognition_job_id=null | 200，`aiRecognitionInfo=null` |
| TC-API-003E-DETAIL-03 | 查询 AI 识别失败后手动提交的 Drawing（有 jobId，FAILED）| 合法 drawingId，job.status=FAILED | 200，`aiRecognitionInfo=null`（无有效 pages）|

---

## 7. 性能测试

### 7.1 测试目标

| 指标 | 目标 | 工具 |
|-----|------|-----|
| AI 识别任务状态查询接口（P95）| ≤ 200ms | JMeter / k6 |
| Drawing 创建接口（P95）| ≤ 500ms | JMeter / k6 |
| 识别结果列表前端渲染（100 页）| ≤ 1s | Chrome DevTools |
| 文件上传（50MB，正常网络）| ≤ 60s | 实测 |

### 7.2 压测场景

| 场景 ID | 描述 | 持续 | 通过条件 |
|--------|------|-----|---------|
| PT-001 | 100 并发用户轮询识别任务状态 | 5min | P95 ≤ 200ms，错误率 < 0.1% |
| PT-002 | 50 并发 Drawing 创建请求 | 2min | P95 ≤ 500ms，无重复创建 |

---

## 8. 安全测试

### 8.1 OWASP Top 10 覆盖

| 项 | 测试方法 |
|---|---------|
| A01 越权访问 | 用 Token A 查询 Token B 项目的 jobId，预期 403/404 |
| A03 注入 | Drawing Code 字段输入 SQL 注入字符串，验证安全转义 |
| A04 文件上传 | 上传伪装为 PDF 的可执行文件，验证魔数校验拦截 |
| A05 安全配置 | 确认 AI 服务凭证未在前端网络请求中暴露 |
| A07 鉴权失效 | 无 Token / 过期 Token 访问所有新增接口，预期 401 |
| A09 日志监控 | 验证文件上传、Drawing 创建操作已记录审计日志 |

### 8.2 专项安全测试

| TC ID | 描述 | 期望 |
|-------|-----|-----|
| TC-SEC-003E-01 | 上传含恶意脚本内容的 PDF，检查是否被安全处理 | 文件存储，不执行任何脚本 |
| TC-SEC-003E-02 | 伪造 jobId 查询他人项目识别结果 | 403 或 404 |
| TC-SEC-003E-03 | 直接访问缩略图 OSS URL（未过期预签名）| 应可访问；过期后访问应返回 403 |

---

## 9. 兼容性测试

| 维度 | 测试矩阵 |
|-----|---------|
| 浏览器 | Chrome 100+、Edge 100+、Safari 15+（REQ-003E §9.4）|
| 移动端 | 不测试（PC 专属）|
| 屏幕分辨率 | 1280×720（最小）、1920×1080（推荐）|
| 网络 | 正常（50M 宽带）、弱网（模拟 3G）|

---

## 10. 测试数据

### 10.1 种子数据

| 数据集 | 用途 | 维护方式 |
|-------|------|---------|
| `test_drawing_valid_3pages.pdf` | 合法 3 页图纸 PDF，各页含 Drawing No/Name | fixtures/valid/ |
| `test_drawing_partial_5pages.pdf` | 5 页 PDF，其中 Page 3 图框信息缺失 | fixtures/valid/ |
| `test_file_over_50mb.pdf` | 超过 50MB 的 PDF | fixtures/invalid/ |
| `test_file_not_pdf.dwg` | 非 PDF 文件（.dwg）| fixtures/invalid/ |
| `test_file_fake_pdf.exe` | 扩展名改为 .pdf 的可执行文件 | fixtures/security/ |

### 10.2 测试 fixtures 目录

```
fixtures/
├── valid/
│   ├── test_drawing_valid_3pages.pdf
│   └── test_drawing_partial_5pages.pdf
├── invalid/
│   ├── test_file_over_50mb.pdf
│   └── test_file_not_pdf.dwg
└── security/
    └── test_file_fake_pdf.exe
```

---

## 11. AC 覆盖矩阵

| AC ID | 描述简述 | 覆盖的 TC | 状态 |
|------|---------|----------|------|
| AC-003E-001 | 上传后 Loading，输入框置灰 | TC-003E-001-01 | TODO |
| AC-003E-002 | 识别完成展示列表并自动填入 | TC-003E-002-01 | TODO |
| AC-003E-003 | 用户可覆盖 AI 填入的 Drawing Code | TC-003E-003-01 | TODO |
| AC-003E-004 | 部分未识别页显示 ⚠ 橙色标记 | TC-003E-004-01 | TODO |
| AC-003E-005 | 提交后仅创建 1 条 Drawing 记录 | TC-003E-005-01 | TODO |
| AC-003E-006 | AI 失败展示降级提示横幅 + Re-upload | TC-003E-006-01 | TODO |
| AC-003E-007 | 降级手动填写后可正常提交 | TC-003E-007-01 | TODO |
| AC-003E-008 | Re-upload 清空文件并重新触发识别 | TC-003E-008-01 | TODO |
| AC-003E-009 | 识别中不可提交，输入框不可编辑 | TC-003E-009-01 | TODO |
| AC-003E-010 | 非 PDF 文件被拒绝 | TC-003E-010-01、TC-API-003E-FILE-02 | TODO |
| AC-003E-011 | 超过 50MB 的文件被拒绝 | TC-003E-011-01、TC-API-003E-FILE-03 | TODO |
| AC-003E-012 | Drawing 详情展示 AI 来源页面信息 | TC-003E-012-01、TC-API-003E-CREATE-01 | TODO |
| AC-003E-013 | 降级上传 Drawing 详情不展示 AI 信息 | TC-003E-013-01、TC-API-003E-CREATE-02 | TODO |

---

## 12. 回归测试范围

| 受影响功能 | 影响原因 | 回归用例 |
|----------|---------|---------|
| REQ-003A 新建图纸弹窗（手动上传路径）| Upload Drawing 弹窗为 REQ-003A 的增强，原手动路径需回归 | REQ-003A 全部 AC 回归测试 |
| Drawing 详情页 | 新增 AI 识别信息区域，需回归原有字段展示 | Drawing 详情页基础字段展示用例 |
| 内部审批人 Todo 推送 | Drawing 创建逻辑调整，需确认 Todo 仍正常推送 | REQ-007-shared Todo 推送相关用例 |

---

## 13. 测试环境

| 环境 | 用途 | 数据 | AI 服务 |
|-----|------|-----|---------|
| local | 开发自测 | mock | Mock AI 服务（固定返回 DONE / FAILED 结果）|
| dev | 集成、E2E | 共享种子 | Mock AI 服务 |
| staging | 验收、压测 | 接近生产 | 真实 AI 服务（联调验证）|
| prod | 上线 | 真实数据 | 真实 AI 服务 |

> ⚠️ Mock AI 服务必须支持模拟 DONE（含各页识别结果）和 FAILED 两种场景，以覆盖所有测试分支。

---

## 14. 验收条件

QA 测试完成的判定：

- [ ] 所有 P0 AC（001、002、005、006、009、010、011）100% 覆盖
- [ ] P0 用例全通过
- [ ] P1 用例 ≥ 95% 通过
- [ ] 性能指标达标（§7 目标）
- [ ] 安全扫描无 high 项
- [ ] 兼容性矩阵（Chrome / Edge / Safari）全绿
- [ ] REQ-003A 回归用例全绿
- [ ] AC 覆盖矩阵 §11 全部 ✅

---

## 15. 已知问题与遗留

| 问题 ID | 描述 | 严重度 | 处理方案 |
|--------|------|-------|---------|
| OQ-001 | AI 识别超时阈值未确认（建议 30s），影响 FAILED 分支测试数据准备 | Medium | 等待 PM + AI 团队确认后更新 TC |
| OQ-002 | 多页不同 Drawing No/Name 时自动填入规则未确认，当前临时取第一页 | High | PM 确认后更新 TC-003E-002-01 验证逻辑 |
| OQ-003 | 单文件最大支持页数未确认，超出时行为（截断/报错）未定义 | Medium | 等待 PM + 后端确认后补充边界测试用例 |

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-24 | agent | 初稿，从 REQ-003E-pc@0.3.1 派生 |
| 0.2.0 | 2026-05-25 | agent | §6.2 `drawing/create` 用例调整（TC-CREATE-04 错误码更正为 422 + `AI_RECOGNITION_IN_PROGRESS`，TC-CREATE-07 整理 FAILED job 场景）；新增 §6.2 `drawing/getDetail` 接口测试三条（DETAIL-01/02/03），覆盖 aiRecognitionInfo 三种返回场景 |
