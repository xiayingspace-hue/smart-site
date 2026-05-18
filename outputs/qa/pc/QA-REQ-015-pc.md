---
doc_type: qa_spec
req_id: REQ-015
version: 0.3.0
status: draft
generated_from: requirement.md@0.5.0
data_contract_ref: data-contract.md@0.1.0
generated_at: 2026-05-18
owner: ""
---

# QA 测试说明：BCA 月度人力数据提交（PC 端）

> ⚠️ 核心原则：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一，格式 `TC-015-{序号}`。
> - 接口测试字段定义引用 data-contract.md。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-015 @ v0.5.0 |
| 数据契约 | data-contract.md @ v0.1.0 |
| 覆盖 Story | US-001、US-002、US-003、US-004、US-005 |
| 覆盖 AC | AC-015-001 ~ AC-015-011 |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证 BCA 月度人力数据提交功能的完整性：包括手动/自动提交任务创建、批次分拆与发送状态管理、批次级/记录级重试、防重复提交、无数据保护、数据快照一致性，以及所有边界条件和权限控制。

---

## 2. 测试策略

### 2.1 测试金字塔

| 层级 | 占比 | 由谁写 | 框架 |
|-----|-----|-------|------|
| 单元测试 | 60% | 前端/后端开发 | Jest / 项目现有框架 |
| 集成测试 | 25% | 开发 + QA | Testcontainers + REST Assured |
| 契约测试 | 5% | QA | schemathesis |
| E2E 测试 | 10% | QA | Cypress |

### 2.2 测试范围

**必测**：
- 手动发起提交任务完整流程（拆批、并发发送、状态更新）
- 防重复提交校验（同月同项目，**任何状态**的已有任务均拦截）
- 无人员通行数据时的拦截
- 发起提交弹窗**不含项目选择**，仅选月份
- 任务列表筛选区**不含项目下拉**
- 批次级重试（失败 → 重试 → 成功/再次失败）
- 记录列表**无单条重试入口**（BCA 接口限制）
- Sending 状态下不可重试批次
- 数据快照一致性（源数据变更后快照不变）
- 自动提交：按配置触发 + 已有任务时跳过
- 权限隔离：Site Admin 只能操作本项目
- BCA 接口超时 / 返回错误的降级处理

**不测（本期）**：
- BCA 返回的统计报告内容（本期无该功能）
- App 端（不支持）
- 存量历史数据迁移（无需迁移）
- BCA 接口负载测试（依赖 BCA 测试环境，单独安排）

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-001 | Site Admin 手动发起提交，全部批次成功 | US-001、US-002 | P0 |
| SC-002 | Site Admin 手动发起提交，部分批次失败，重试失败批次后全部成功 | US-001、US-003 | P0 |
| SC-003 | Site Admin 手动发起提交，全部批次失败 | US-001、US-002 | P0 |
| SC-004 | 批次内存在失败记录，展开查看记录详情及失败原因，确认无单条重试按钮 | US-004 | P1 |
| SC-005 | 开启自动提交，系统在触发日自动创建任务并发送 | US-005 | P1 |
| SC-006 | 查看任务详情，批次列表展开记录列表 | US-002 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-E01 | 重复发起同月同项目提交（已有非 Failed 任务） | AC-015-002 |
| SC-E02 | 指定月份该项目无人员通行数据时发起提交 | AC-015-007 |
| SC-E03 | BCA 接口返回业务错误（非 2xx） | AC-015-004 |
| SC-E04 | BCA 接口调用超时（30s） | AC-015-004 |
| SC-E05 | 批次处于 Sending 状态时尝试重试 | AC-015-011 |
| SC-E06 | 已成功的批次尝试重试 | — |
| SC-E07 | 指定未来月份发起提交 | — |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-P01 | Site Admin 访问无权限的项目提交记录，应返回 403 |
| SC-P02 | Platform Admin 可查看所有项目的提交任务 |
| SC-P03 | 未登录用户访问提交任务列表，应返回 401 |
| SC-P04 | Site Admin 发起其他项目的提交，应返回 403 |

### 3.4 状态转换场景

**合法转换测试**：

| 场景 ID | From | Action | To | 测试要点 |
|--------|------|--------|-----|---------|
| SC-T01 | —（无任务） | 创建任务，批次并发发送（Sending） | —（Task 无中间态，等所有批次响应后写入终态） | 任务创建后批次立即 Sending；Task 终态取决于批次结果 |
| SC-T02 | 全部批次 Sending | 所有批次响应成功 | Task: Success | 任务终态，不可再重试 |
| SC-T03 | 部分批次 Sending | 批次响应返回，有成功有失败 | Task: Partial Failed | 任务有重试入口 |
| SC-T04 | 全部批次 Sending | 所有批次均返回失败 | Task: Failed | 任务有重试入口 |
| SC-T05 | Task: Partial Failed | 重试失败批次后全部成功 | Task: Success | 状态正确收敛 |
| SC-T06 | Task: Failed | 重试后全部成功 | Task: Success | — |

**非法转换测试**：

| 场景 ID | From | 尝试 Action | 期望 |
|--------|------|------------|-----|
| SC-T-F01 | Task: Success | 重试任意批次 | 拒绝，422（任务已全部成功） |
| SC-T-F02 | Batch: Sending | 手动重试该批次 | 拒绝，409（BCA_BATCH_SENDING） |

---

## 4. 测试用例（TC）详述

### TC-015-001

```yaml
tc_id: TC-015-001
covers_ac: [AC-015-001]
scenario: SC-001
priority: P0
test_level: e2e

preconditions:
  - 已登录 Site Admin 账号，有 Project A 权限
  - 2026-04 月份下 Project A 有 250 条人员通行记录
  - 该月该项目无任何提交任务

steps:
  given: 进入「BCA 数据提交」页面
  when: |
    1. 点击「+ 发起提交」按钮
    2. 选择月份 2026-04（弹窗无项目选择，固定当前项目）
    3. 点击「确认发起」
  then:
    - 弹窗关闭，任务列表出现新行，状态为「发送中」
    - 批次数量为 3（250 条 / 100 条每批 = 3 批）
    - 等待发送完成，任务状态变为「成功」
    - 成功批次数 = 3，失败批次数 = 0

cleanup:
  - 删除测试任务数据
```

---

### TC-015-002

```yaml
tc_id: TC-015-002
covers_ac: [AC-015-002]
scenario: SC-E01
priority: P0
test_level: integration

preconditions:
  - 已登录 Site Admin，有 Project A 权限
  - 2026-04 该项目已存在任意状态的任务（status=success / partial_failed / failed 均适用）

steps:
  given: 提交任务列表页
  when: |
    1. 点击「+ 发起提交」
    2. 选择 Project A，月份 2026-04
    3. 点击「确认发起」
  then:
    - 弹窗内显示 inline 错误：「该项目本月数据已提交或提交中，无需重复操作」
    - 「确认发起」按钮处于禁用状态
    - 弹窗不关闭
    - 接口返回 422，error_code=BCA_SUBMISSION_DUPLICATE

cleanup:
  - 无（不修改数据）
```

---

### TC-015-003

```yaml
tc_id: TC-015-003
covers_ac: [AC-015-007]
scenario: SC-E02
priority: P0
test_level: integration

preconditions:
  - 已登录 Site Admin，有 Project B 权限
  - 2026-04 月份下 Project B 无任何人员通行数据

steps:
  given: 提交任务列表页
  when: |
    1. 点击「+ 发起提交」
    2. 选择 Project B，月份 2026-04
    3. 点击「确认发起」
  then:
    - 弹窗内显示 inline 错误：「该月暂无人员通行数据，无需提交」
    - 「确认发起」按钮禁用
    - 接口返回 422，error_code=BCA_NO_ATTENDANCE_DATA
```

---

### TC-015-004

```yaml
tc_id: TC-015-004
covers_ac: [AC-015-003]
scenario: SC-001
priority: P0
test_level: integration

preconditions:
  - 已存在一个提交任务，其中批次 Batch #1 仍处于 Sending 状态（批次响应尚未全部返回）

steps:
  given: 批次发送 Worker 处理 Batch #1，BCA 接口返回成功
  when: Worker 收到 BCA 成功响应
  then:
    - batch #1 status → success
    - batch #1 内所有 batch_records status → success
    - 所有批次响应返回后，任务整体状态重新评估（若全部成功则 → success，否则 → partial_failed）

cleanup:
  - 无
```

---

### TC-015-005

```yaml
tc_id: TC-015-005
covers_ac: [AC-015-004]
scenario: SC-E03
priority: P0
test_level: integration

preconditions:
  - 已存在一个提交任务，Batch #2 仍处于 Sending 状态
  - Mock BCA 接口返回 400 错误，error_code='INVALID_DATA'

steps:
  given: 批次发送 Worker 处理 Batch #2
  when: BCA 接口返回 400 错误
  then:
    - batch #2 status → failed
    - batch #2 error_message 包含 BCA 返回的错误描述
    - 任务整体状态重新评估（partial_failed 或 failed）
    - 批次列表页显示「重试」按钮
```

---

### TC-015-006

```yaml
tc_id: TC-015-006
covers_ac: [AC-015-004]
scenario: SC-E04
priority: P0
test_level: integration

preconditions:
  - 已存在一个提交任务，Batch #3 仍处于 Sending 状态
  - Mock BCA 接口响应超时（> 30s）

steps:
  given: 批次发送 Worker 处理 Batch #3
  when: BCA 接口超过 30s 无响应，触发超时
  then:
    - batch #3 status → failed
    - error_message 包含「接口超时」字样
    - 任务整体状态更新为 partial_failed 或 failed
```

---

### TC-015-007

```yaml
tc_id: TC-015-007
covers_ac: [AC-015-005]
scenario: SC-002
priority: P0
test_level: e2e

preconditions:
  - 已存在一个 partial_failed 任务，Batch #2 和 Batch #3 状态为 failed
  - Site Admin 已进入任务详情页

steps:
  given: 任务详情页，Batch #2 状态为 failed，显示「重试」按钮
  when: |
    1. 点击 Batch #2 的「重试」按钮
  then:
    - Batch #2 状态变为「发送中」，「重试」按钮变为 Loading / 置灰
    - 发送完成后（Mock BCA 返回成功）：
      Batch #2 状态 → 成功
      任务进度条绿色增加
      若所有批次均成功，任务状态 → 成功

cleanup:
  - 删除测试任务
```

---

### TC-015-008

```yaml
tc_id: TC-015-008
covers_ac: [AC-015-006]
scenario: SC-004
priority: P1
test_level: e2e

preconditions:
  - 已存在一个 partial_failed 任务，Batch #1 内有 3 条 record 状态为 failed
  - Site Admin 已进入任务详情页，展开 Batch #1

steps:
  given: Batch #1 展开，记录列表显示 3 条状态为 failed 的记录
  when: 查看记录列表每一行
  then:
    - 记录行无「重试」操作按钮（操作列不存在或仅显示 "—"）
    - 每条 failed 记录展示错误信息（BCA 返回的错误描述）
    - 页面有文字引导或 Tooltip 提示：如需补发，须通过批次「重试」按钮整批重发
    - 后端接口测试：POST .../records/:recordId/retry 返回 404（接口不存在）

cleanup:
  - 删除测试任务
```

---

### TC-015-009

```yaml
tc_id: TC-015-009
covers_ac: [AC-015-011]
scenario: SC-E05
priority: P0
test_level: e2e + integration

preconditions:
  - 已存在一个提交任务，Batch #1 当前状态为 sending（批次仍在等待 BCA 响应）

steps:
  given: 任务详情页，Batch #1 状态为「发送中」
  when: 用户尝试点击 Batch #1 的「重试」按钮
  then:
    - 前端：「重试」按钮处于禁用（置灰）状态，无法点击，hover 显示 Tooltip「发送中，请稍候」
    - 后端（接口测试）：POST .../batches/:batchId/retry 返回 409，error_code=BCA_BATCH_SENDING
```

---

### TC-015-010

```yaml
tc_id: TC-015-010
covers_ac: [AC-015-008]
scenario: SC-005
priority: P1
test_level: integration

preconditions:
  - Project C 已开启自动提交，trigger_day=1，trigger_time=02:00
  - 2026-05-01 02:00 尚无该项目 2026-04 月的提交任务
  - 2026-04 有 150 条人员通行数据

steps:
  given: 系统时间推进到 2026-05-01 02:00
  when: Cron Job 触发自动提交逻辑
  then:
    - 系统自动为 Project C 创建 2026-04 月的提交任务，trigger_type='auto'
    - 任务列表中可见该任务，「触发方式」列显示「自动」Tag
    - 批次生成并开始发送

cleanup:
  - 删除测试任务
```

---

### TC-015-011

```yaml
tc_id: TC-015-011
covers_ac: [AC-015-009]
scenario: SC-005
priority: P1
test_level: integration

preconditions:
  - Project C 已开启自动提交，trigger_day=1
  - 2026-04 月份 Project C 已存在手动创建的任务（status=success）

steps:
  given: 系统时间推进到 2026-05-01 02:00
  when: Cron Job 触发自动提交逻辑
  then:
    - 系统识别到已有任务，跳过 Project C
    - 不新增任何提交任务记录
    - 原手动任务不受影响
```

---

### TC-015-012

```yaml
tc_id: TC-015-012
covers_ac: [AC-015-010]
scenario: SC-006
priority: P1
test_level: integration

preconditions:
  - 已存在一个提交任务（2026-04）
  - 任务创建后，源系统中修改了某条 2026-04 的人员通行数据（如 time_in 变更）

steps:
  given: 任务已创建，batch_records 中已有快照数据
  when: 查看该任务的批次记录列表，观察对应记录的 time_in 字段
  then:
    - 批次记录显示的 time_in 仍为任务创建时的快照值，不反映源数据的修改
    - 快照数据与源数据不同，但 batch_records 中保持原值

cleanup:
  - 还原源数据（或测试环境自动隔离）
```

---

### TC-015-013

```yaml
tc_id: TC-015-013
covers_ac: [AC-015-002]
scenario: SC-E01（全状态覆盖：success / partial_failed / failed 均拦截）
priority: P1
test_level: integration

preconditions:
  - 2026-04 Project A 已有一个任务（分别用 status=success、partial_failed、failed 各测一遍）

steps:
  given: 提交任务列表页
  when: |
    1. 点击「+ 发起提交」
    2. 选择月份 2026-04
    3. 点击「确认发起」
  then:
    - 无论已有任务状态如何，均显示 inline 错误：「本月已有提交记录，无需重复操作」
    - 「确认发起」按钮禁用
    - 接口返回 422，error_code=BCA_SUBMISSION_DUPLICATE
```

---

### TC-015-014

```yaml
tc_id: TC-015-014
covers_ac: [AC-015-001]
scenario: 边界：选择当月或未来月份
priority: P1
test_level: e2e

preconditions:
  - 当前日期为 2026-05-17

steps:
  given: 发起提交弹窗
  when: 月份选择器尝试选择 2026-05（当月）或 2026-06（未来）
  then:
    - 月份选择器禁止选择当月及未来月份（置灰不可选）
    - 最新可选月份为 2026-04（上个月）
```

---

### TC-015-015（权限测试）

```yaml
tc_id: TC-015-015
covers_ac: [AC-015-001]
scenario: SC-P01
priority: P0
test_level: integration

preconditions:
  - 已登录 Site Admin 账号，仅有 Project A 权限，无 Project B 权限

steps:
  given: —
  when: |
    1. 直接调用 GET /bca-submissions?projectId={Project_B_ID}
  then:
    - 接口返回 403，不返回 Project B 的任何数据
```

---

## 5. 边界值与等价类

### 5.1 批次大小边界

| 等价类 | 数据量 | 期望结果 |
|-------|-------|---------|
| 有效（整除） | 200 条，batch_size=100 | 生成 2 批，每批 100 条 |
| 有效（不整除） | 250 条，batch_size=100 | 生成 3 批：100、100、50 条 |
| 最小 | 1 条 | 生成 1 批，1 条记录 |
| 极大 | 10,000 条，batch_size=100 | 生成 100 批，任务创建 < 5s |

### 5.2 触发日边界

| 等价类 | trigger_day | 期望 |
|-------|------------|-----|
| 有效最小值 | 1 | 接受，每月 1 日触发 |
| 有效最大值 | 7 | 接受，每月 7 日触发 |
| 无效（0） | 0 | 拒绝，400 |
| 无效（8） | 8 | 拒绝，400 |
| 无效（非整数） | 2.5 | 拒绝，400 |

### 5.3 submission_month 格式

| 等价类 | 输入 | 期望 |
|-------|-----|-----|
| 有效 | 2026-04 | 接受 |
| 无效（当月） | 2026-05（当前月） | 拒绝，400 |
| 无效（未来） | 2026-06 | 拒绝，400 |
| 无效格式 | 2026/04 | 拒绝，400 |
| 无效格式 | 202604 | 拒绝，400 |
| 无效月份值 | 2026-13 | 拒绝，400 |

### 5.4 批次列表分页（极端数据）

| 等价类 | 批次数 | 期望 |
|-------|-------|-----|
| 正常 | 10 批 | 完整展示，无分页 |
| 极端 | 200 批 | 分页展示，每页 50 条，性能 < 2s |
| 极端 | 300 批 | 分页展示，不崩溃 |

---

## 6. 接口测试（契约测试）

### 6.1 通用契约校验

用 schemathesis 基于 data-contract 自动生成校验：
- 请求 schema 合法 → 响应 schema 合法
- 必填字段缺失 → 400
- 字段类型错误 → 400
- 枚举值越界 → 400

### 6.2 各 API 专项接口测试

#### `POST /bca-submissions`（创建提交任务）

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-015-01 | 正常创建 | 合法 project_id + submission_month | 200，返回 task 对象 |
| TC-API-015-02 | 缺少 project_id | — | 400 |
| TC-API-015-03 | 缺少 submission_month | — | 400 |
| TC-API-015-04 | submission_month 格式错误 | "2026/04" | 400 |
| TC-API-015-05 | submission_month 为当月 | "2026-05" | 400 |
| TC-API-015-06 | submission_month 为未来月 | "2026-06" | 400 |
| TC-API-015-07 | 重复发起（已有非 failed 任务） | 同月同项目 | 422，BCA_SUBMISSION_DUPLICATE |
| TC-API-015-08 | 无数据 | 该月无通行记录 | 422，BCA_NO_ATTENDANCE_DATA |
| TC-API-015-09 | 未登录 | 无 token | 401 |
| TC-API-015-10 | 无项目权限 | 他人项目 id | 403 |

#### `POST /bca-submissions/:taskId/batches/:batchId/retry`（批次重试）

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-015-11 | 正常重试 failed 批次 | failed 状态 batch_id | 200 |
| TC-API-015-12 | 批次 sending 状态重试 | sending 状态 batch_id | 409，BCA_BATCH_SENDING |
| TC-API-015-13 | 批次 success 状态重试 | success 状态 batch_id | 422 |
| TC-API-015-14 | batch_id 不存在 | 随机 UUID | 404 |
| TC-API-015-15 | 无权限（他人项目） | — | 403 |
| TC-API-015-16 | 未登录 | 无 token | 401 |

#### `POST /bca-submissions/:taskId/batches/:batchId/records/:recordId/retry`（单条记录重试）

> **此接口不实现**（BCA 接口不支持单条重试）。

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-015-17 | 单条记录重试接口不存在 | 任意合法 record_id | 404（接口未实现） |

#### `PUT /projects/:projectId/bca-auto-submit-config`（保存自动提交配置）

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-015-22 | 正常开启自动提交 | enabled=true, trigger_day=1, trigger_time="02:00" | 200 |
| TC-API-015-23 | trigger_day 超出范围 | trigger_day=8 | 400 |
| TC-API-015-24 | trigger_day=0 | trigger_day=0 | 400 |
| TC-API-015-25 | 关闭自动提交 | enabled=false | 200 |
| TC-API-015-26 | 未登录 | 无 token | 401 |

---

## 7. 性能测试

### 7.1 测试目标

引用需求 §9.1 与 backend-spec §12。

| 指标 | 目标 | 工具 |
|-----|------|-----|
| 批次列表接口 P95 | < 500ms | JMeter / k6 |
| 创建任务（10,000 条数据） | < 5s | k6 |
| 任务列表接口 P95 | < 500ms | k6 |

### 7.2 压测场景

| 场景 ID | 描述 | 持续 | 通过条件 |
|--------|------|-----|---------|
| PT-001 | 并发 50 用户轮询任务详情接口 | 5 分钟 | P95 < 500ms，错误率 < 0.1% |
| PT-002 | 并发创建 5 个大任务（各 10,000 条） | 单次 | 每个任务创建 < 5s，无死锁 |

---

## 8. 安全测试

### 8.1 OWASP Top 10 覆盖

| 项 | 测试方法 |
|---|---------|
| A01 越权访问 | Site Admin 用 token 访问他人 project 的提交任务和批次数据 |
| A02 加密失效 | 检查 BCA API Key 是否明文返回给前端（不应出现） |
| A03 注入 | submission_month、project_id 等字段 SQL 注入 fuzz |
| A05 安全配置错误 | 检查 BCA API Key 是否在响应中暴露 |
| A07 鉴权失效 | 过期 token、伪造 token 访问接口 |
| A09 日志监控 | 验证发起提交、重试操作均写入 audit_logs |

### 8.2 项目专项安全测试

| TC ID | 描述 | 期望 |
|-------|-----|-----|
| TC-SEC-015-01 | Site Admin A 尝试查看 Site Admin B 所属项目的批次记录 | 403，数据隔离 |
| TC-SEC-015-02 | 检查 GET /bca-submissions 响应体，不含 BCA API Key 字段 | 无 API Key 泄露 |
| TC-SEC-015-03 | submission_month 传入 SQL 注入字符串 | 400，安全转义 |

---

## 9. 兼容性测试

| 维度 | 测试矩阵 |
|-----|---------|
| 浏览器 | Chrome 100+、Edge 100+、Safari 15+（仅 PC） |
| 移动端 | 不测试（本需求仅支持 PC） |
| 屏幕分辨率 | 1280×800、1920×1080（批次列表列宽是否正常） |
| 网络 | 正常网络（E2E）；弱网（throttle 3G，验证 Loading 态和超时处理） |

---

## 10. 测试数据

### 10.1 种子数据

| 数据集 | 用途 | 维护方式 |
|-------|------|---------|
| `project_with_attendance_250` | 有 250 条通行记录的项目（2026-04） | SQL fixture |
| `project_with_no_attendance` | 无通行数据的项目（2026-04） | SQL fixture |
| `task_partial_failed` | 已有部分失败批次的任务 | SQL fixture |
| `task_all_failed` | 全部批次失败的任务 | SQL fixture |
| `task_success` | 全部批次成功的任务 | SQL fixture |
| `auto_submit_config_enabled` | 已开启自动提交的项目配置 | SQL fixture |

### 10.2 Mock BCA 接口

| Mock 场景 | 行为 | 用于 TC |
|----------|------|--------|
| `bca_mock_success` | 所有请求返回 200 成功 | TC-015-001、TC-015-007、TC-015-008 |
| `bca_mock_error_400` | 返回 400 业务错误 | TC-015-005 |
| `bca_mock_timeout` | 30s 后无响应 | TC-015-006 |
| `bca_mock_partial` | 批次内部分记录失败 | TC-015-004 |

### 10.3 测试 fixtures 结构

```
fixtures/
├── valid/
│   ├── attendance_250_records.sql       # 250 条通行数据
│   ├── task_partial_failed.sql          # 部分失败任务
│   └── auto_submit_config_enabled.sql
├── invalid/
│   ├── no_attendance_data.sql           # 无通行数据
│   └── duplicate_month.sql             # 已有任务（同月同项目）
└── security/
    ├── sql_injection_month.txt          # submission_month 注入样本
    └── cross_project_token.json        # 越权 token 测试数据
```

---

## 11. AC 覆盖矩阵

| AC ID | 描述简述 | 覆盖的 TC | 状态 |
|------|---------|----------|------|
| AC-015-001 | 手动发起提交，创建任务并分批发送 | TC-015-001、TC-015-013、TC-015-014、TC-API-015-01~06 | TODO |
| AC-015-002 | 防重复发起（已有非 failed 任务） | TC-015-002、TC-API-015-07 | TODO |
| AC-015-003 | 批次成功后状态更新，联动任务状态 | TC-015-004、TC-015-007 | TODO |
| AC-015-004 | 批次失败后状态更新，记录错误信息 | TC-015-005、TC-015-006 | TODO |
| AC-015-005 | 批次重试，状态 Sending → 完成后更新 | TC-015-007、TC-API-015-11~16 | TODO |
| AC-015-006 | 记录列表无单条重试入口；后端接口返回 404 | TC-015-008、TC-API-015-17 | TODO |
| AC-015-007 | 无数据时阻止创建任务 | TC-015-003、TC-API-015-08 | TODO |
| AC-015-008 | 自动提交在触发日创建任务 | TC-015-010 | TODO |
| AC-015-009 | 自动提交跳过已有任务 | TC-015-011 | TODO |
| AC-015-010 | 数据快照与源数据变更隔离 | TC-015-012 | TODO |
| AC-015-011 | Sending 状态下不可重试（置灰+409） | TC-015-009、TC-API-015-12 | TODO |

---

## 12. 回归测试范围

| 受影响功能 | 影响原因 | 回归用例 |
|----------|---------|---------|
| 项目设置页 | 新增「BCA 自动提交」配置区域，需验证不影响其他设置项 | 项目设置页基础功能冒烟用例 |
| 人员通行数据读取 | 新增快照读取逻辑，需验证源数据展示不受影响 | 人员通行记录列表展示用例 |

---

## 13. 测试环境

| 环境 | 用途 | 数据 | BCA 接口 |
|-----|------|-----|---------|
| local | 开发自测 | mock fixtures | Mock Server |
| dev | 集成、E2E | 共享种子数据 | Mock Server（可切换 BCA Sandbox） |
| staging | 验收、压测 | 接近生产量级数据 | BCA Sandbox（如 BCA 提供） |
| prod | 上线 | 真实数据 | BCA 生产接口 |

---

## 14. 验收条件

- [ ] 所有 P0 AC 100% 覆盖（AC-015-001~007、011）
- [ ] P0 用例全部通过（TC-015-001 ~ TC-015-009、相关 API TC）
- [ ] P1 用例 ≥ 95% 通过（TC-015-010 ~ TC-015-015）
- [ ] 压测：PT-001 P95 < 500ms，PT-002 创建任务 < 5s
- [ ] 安全测试：TC-SEC-015-01 ~ 03 全部通过，无 API Key 泄露
- [ ] 兼容性：Chrome / Edge / Safari 主流程无异常
- [ ] 回归用例全绿
- [ ] AC 覆盖矩阵 §11 全部 ✅

---

## 15. 已知问题与遗留

| 问题 ID | 描述 | 严重度 | 处理方案 |
|--------|------|-------|---------|
| OQ-001 | BCA 接口单批次最大记录数未确认，可能影响 batch_size 边界测试 | 中 | 等待 OQ-001 确认后补充边界 TC |
| OQ-002 | BCA 接口幂等性未确认，影响批次重试是否重发已成功记录的测试策略 | 高 | 等待 OQ-002 确认后更新 TC-015-007 测试预期 |

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-17 | | 初稿，覆盖 11 条 AC、15 条 TC、API 契约测试、性能/安全/兼容性测试 |
| 0.2.0 | 2026-05-18 | | 同步需求 v0.4.0：AC-015-006 改为不可单条重试；删除单条重试相关 TC；TC-API-015-17 改为验证 404 |
| 0.3.0 | 2026-05-18 | | 同步需求 v0.3.0/v0.5.0：1) 移除所有 "in_progress" Task 状态引用（Task 无中间态，直接落地终态）；2) 删除非法转换 SC-T-F03（Batch 无 Pending 状态）；3) 修正 AC-015-002 覆盖说明（任意状态任务均拦截重复发起） |
