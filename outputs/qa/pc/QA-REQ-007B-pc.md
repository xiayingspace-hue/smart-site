---
doc_type: qa_spec
req_id: REQ-007B-pc
req_title: "PC 端 — DC 外部审批 Todo 与标记 Dialog"
version: 0.2.2
status: draft
generated_from: REQ-007B-pc@0.2.2
generated_at: 2026-05-26
owner: ""
---

# QA 测试说明：PC 端 — DC 外部审批 Todo 与标记 Dialog

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - SE 通知相关验收依赖管理员完成 [Assign] 操作（REQ-003D-pc）；QR 生成逻辑见 REQ-006-shared。
> - v0.2.2 变更要点：Status of Approval 改为 A–E 下拉；报审三要素始终必填；新增 Others Reason（E 时必填）；签字版/凭证/日期仅 A/B/D 时必填；移除 Rejection Reason 字段；新增 F-002 Detail Drawer 测试。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007B-pc @ v0.2.2 |
| 依赖需求 | REQ-007-shared、REQ-007A-pc、REQ-003D-pc、REQ-006-shared |
| 覆盖 Story | US-007B-001、US-007B-002 |
| 覆盖 AC | AC-007B-001 ~ AC-007B-015 |
| 测试平台 | PC（Chrome 100+ / Edge 100+ / Safari 15+，1280px+） |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证 DC 外部审批 Todo 卡片"仅我名下"过滤、点击 [Detail] 打开详情侧滑弹框（字段展示、文件 inline 打开、强制下载）、Mark External Approval Result Dialog 的完整交互（Status A–E 五档路径）、报审三要素始终必填、条件字段按 Status 分组显隐与校验、版本生效与 QR 生成、多 DC 并发处理（一 DC 操作后其余 DC Todo 自动关闭）、权限控制及失败重试等行为的正确性。

---

## 2. 测试策略

### 2.1 测试范围

**必测**：
- External Approval Required Todo 卡片"仅我名下"过滤
- 点击 [Detail] 打开详情侧滑弹框，字段展示完整且不含 Drawing Code/Name
- 文件名 inline 打开（新标签页）
- [Download Original File] 强制下载（文件名格式、403 拦截）
- [Mark Result] Dialog 打开
- Status of Approval 下拉（A–E）选项正确
- 报审三要素（Ref No. / Subject / Description）始终必填校验
- Status E：Others Reason 必填校验
- Status A/B/D：签字版 / 凭证 / 日期必填校验及文件格式/大小限制
- Status C：仅报审三要素必填，无文件字段
- Status A/B/D 成功路径（版本生效 + QR 生成 + Toast）
- Status C 成功路径（退回 + 设计人员通知）
- Status E 成功路径
- QR 生成失败回滚
- 多 DC 并发：一 DC 操作完成后另一 DC Todo 自动关闭
- 权限控制（非 DC 不显示 [Mark Result]，接口 403）
- loading 态防重复提交

**不测（本期）**：
- 内部审批 Todo（REQ-007A-pc）
- 管理员 [Assign] SE 的具体操作（REQ-003D-pc）
- 版本历史抽屉（REQ-007C-pc）
- Bentley 平台本身（系统外）

### 2.2 测试金字塔

| 层级 | 占比 | 说明 |
|-----|-----|------|
| 单元测试 | 55% | 必填校验、文件类型/大小校验、日期选择器限制 |
| 集成测试 | 30% | 前端 → 后端外部审批接口、版本状态查询、其他 DC Todo 关闭 |
| E2E 测试 | 15% | 完整通过/驳回流程，含 QR 生成、Toast、Todo 消失 |

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-007B-001 | DC 从 Todo 打开 Detail Drawer，下载原始文件 | US-007B-001 | P1 |
| SC-007B-002 | DC 选择 Status A，完成外部审批通过全流程 | US-007B-002 | P1 |
| SC-007B-003 | DC 选择 Status B，完成带批注通过全流程 | US-007B-002 | P1 |
| SC-007B-004 | DC 选择 Status C（Revise And Resubmit），设计人员收到通知 | US-007B-002 | P1 |
| SC-007B-005 | DC 选择 Status D（For Record Purpose），完成全流程 | US-007B-002 | P2 |
| SC-007B-006 | DC 选择 Status E（Others），填写原因完成全流程 | US-007B-002 | P2 |
| SC-007B-007 | 一个 DC 操作后，同项目另一 DC Todo 自动关闭 | US-007B-001 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-007B-E01 | 报审三要素任一为空（任意 Status），点击 Confirm | AC-007B-006 / AC-007B-011 |
| SC-007B-E02 | Status A/B/D 时签字版/凭证/日期任一为空 | AC-007B-007 |
| SC-007B-E03 | Status E 时 Others Reason 为空 | AC-007B-007B |
| SC-007B-E04 | QR 生成失败，整体操作回滚 | AC-007B-010 |
| SC-007B-E05 | 签字版文件超过 50MB | — |
| SC-007B-E06 | 签字版文件格式不合法（如 .exe） | — |
| SC-007B-E07 | 凭证文件超过 20MB | — |
| SC-007B-E08 | External Approval Date 选择未来日期 | — |
| SC-007B-E09 | 接口调用失败（非 QR），Dialog 保留可重试 | AC-007B-015 |
| SC-007B-E10 | Drawer 详情加载失败，显示 Retry | — |
| SC-007B-E11 | 原始文件下载失败（fileUrl 过期） | — |
| SC-007B-E12 | Status 未选中直接点击 Confirm | AC-007B-006 |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-007B-P01 | 未配置为项目 DC 的用户调用 GET /drawing/version/{id}/download，返回 403 |
| SC-007B-P02 | 未配置为项目 DC 的用户调用 POST /drawing/external-approve，返回 403 |
| SC-007B-P03 | 无 `drawing:external-approval` 权限的用户进入 Todo 列表，不显示外部审批 Todo |
| SC-007B-P04 | 无权限用户的 Drawer（若可见），[Mark Result] 按钮不展示 |

### 3.4 状态转换场景

**合法转换测试**：

| 场景 ID | From | Action | To | 测试要点 |
|--------|------|--------|-----|---------|
| SC-007B-T01 | S-001 PENDING | DC 标记 Status A（含所有必填） | S-002 APPROVED | DrawingVersion → `APPROVED`；QR 生成；其余 DC Todo → S-004 |
| SC-007B-T02 | S-001 PENDING | DC 标记 Status B（含所有必填） | S-002 APPROVED | 同 T01 |
| SC-007B-T03 | S-001 PENDING | DC 标记 Status C（含报审三要素） | S-003 REJECTED | DrawingVersion → `EXTERNAL_REJECTED`；设计人员通知；其余 DC Todo → S-004 |
| SC-007B-T04 | S-001 PENDING | DC 标记 Status D（含所有必填） | S-002 APPROVED | 同 T01 |
| SC-007B-T05 | S-001 PENDING | DC 标记 Status E（含报审三要素 + Others Reason） | S-003 REJECTED | 同 T03 |
| SC-007B-T06 | S-001 PENDING | 其他 DC 先完成操作 | S-004 CLOSED_BY_OTHER | Todo 自动消失 |

**非法转换测试**：

| 场景 ID | From | 尝试 Action | 期望 |
|--------|------|------------|-----|
| SC-007B-TF01 | S-002/S-003/S-004（已关闭） | 再次调用 POST /drawing/external-approve | 403 / 业务错误码；前端不展示已关闭 Todo |
| SC-007B-TF02 | — | 非项目 DC 调用 POST /drawing/external-approve | 403 |

---

## 4. 测试用例详述

### TC 组 1：Todo 卡片"仅我名下"过滤与展示

```yaml
tc_id: TC-007B-001
covers_ac: [AC-007B-001]
scenario: SC-007B-001
priority: P1
test_level: integration
preconditions:
  - 项目已配置 DC-A 和 DC-B
  - 内部审批已完成通过，两人各自收到一条外部审批 Todo
steps:
  given: DC-A 账号已登录
  when: DC-A 进入 PC Todo 列表
  then:
    - 列表中仅显示指派给 DC-A 的 External Approval Required 任务
    - 卡片显示 🌐 图标、标题"External Approval Required"
    - 包含图纸信息行：drawingCode、drawingName、versionNo
    - 包含 Uploaded by 姓名 + 上传时间
    - 包含 Internal approved by 姓名 + 通过时间
    - Version Note 存在时显示，不存在时该行不显示
    - 卡片右侧仅有 [Detail] 按钮
    - DC-B 的任务不出现在 DC-A 列表中
```

```yaml
tc_id: TC-007B-002
covers_ac: [AC-007B-001]
priority: P1
test_level: integration
steps:
  given: 非 DC 角色用户（如设计人员）已登录
  when: 进入 PC Todo 列表
  then:
    - 不显示任何 External Approval Required 任务
```

### TC 组 2：Detail Drawer 打开与字段展示

```yaml
tc_id: TC-007B-003
covers_ac: [AC-007B-002]
priority: P1
test_level: e2e
steps:
  given: DC-A 在 Todo 列表中看到一条 External Approval Required 记录
  when: 点击该记录右侧的 [Detail] 按钮
  then:
    - 右侧滑出 Drawer（宽约 480px）
    - Drawer 头部显示"Drawing Approval Detail"和"External Approval Required"标签
    - 背景列表有半透明遮罩，不可操作
```

```yaml
tc_id: TC-007B-004
covers_ac: [AC-007B-003]
priority: P1
test_level: integration
preconditions:
  - 设计人员上传时填写了 Category、Description、Version Note
  - 内部审批人已审批通过
steps:
  given: DC 打开该图纸待办的 Detail Drawer
  when: 等待 Drawer 加载完成
  then:
    - 展示 Category、Description、Version、Version Note
    - 展示 Uploaded by（姓名 + "（Designer）"）、Upload Time（YYYY-MM-DD HH:mm）
    - 展示 Internal Approver 姓名、Internal Approved Time
    - 展示 Original File（文件名 + 文件类型图标）
    - ⚠️ 不展示 Drawing Code 字段
    - ⚠️ 不展示 Drawing Name 字段
    - Description 为空时显示"—"；Version Note 为空时显示"—"
```

```yaml
tc_id: TC-007B-005
covers_ac: []
priority: P2
test_level: unit
steps:
  given: Drawer 加载时接口超时或返回错误
  when: 等待加载
  then:
    - Drawer 内显示"Failed to load drawing details."
    - 显示 [Retry] 按钮；点击 [Retry] 重新请求
```

### TC 组 3：Original File inline 打开与强制下载

```yaml
tc_id: TC-007B-006
covers_ac: []
priority: P2
test_level: integration
preconditions:
  - Drawer 已加载，fileUrl 非空
steps:
  given: DC 在 Drawer 中看到 Original File 文件名链接
  when: 点击文件名链接
  then:
    - 浏览器新标签页打开文件（Content-Disposition: inline）
    - 文件名下方辅助提示文案显示正确
```

```yaml
tc_id: TC-007B-007
covers_ac: [AC-007B-004]
priority: P1
test_level: integration
preconditions:
  - Drawer 已加载，fileUrl 非空
steps:
  given: DC 在 Drawer 底部 ActionBar
  when: 点击 [Download Original File]
  then:
    - 浏览器触发强制文件下载（Content-Disposition: attachment）
    - 下载文件名格式为"{drawingCode}-V{versionNo}-original.{ext}"
    - 内容与设计人员上传的原始文件一致
    - 下载期间按钮 loading，下载后恢复
```

```yaml
tc_id: TC-007B-008
covers_ac: [AC-007B-005]
priority: P1
test_level: integration
preconditions:
  - 用户未被配置为项目 DC
steps:
  given: 任意用户（非项目 DC）
  when: 直接调用 GET /drawing/version/{versionId}/download
  then:
    - 接口返回 403
    - 文件不下载
```

```yaml
tc_id: TC-007B-009
covers_ac: []
priority: P2
test_level: unit
preconditions:
  - Drawer 已加载，fileUrl 为空
steps:
  given: DC 在 Drawer 底部
  when: 查看 [Download Original File] 按钮状态
  then:
    - 按钮置灰（disabled）
    - Hover 时 Tooltip 显示"Original file unavailable."
```

### TC 组 4：Mark Result Dialog 打开与 Status 下拉

```yaml
tc_id: TC-007B-010
covers_ac: [AC-007B-002]
priority: P1
test_level: e2e
preconditions:
  - Drawer 已打开
steps:
  given: DC 在 Drawer 底部 ActionBar
  when: 点击 [✅ Mark Result]
  then:
    - 弹出"Mark External Approval Result" Dialog
    - Dialog 头部显示图纸编号 + 图纸名称 + 版本号
    - Status of Approval 下拉默认显示"Please select"
    - Drawer 保留在背景
```

```yaml
tc_id: TC-007B-011
covers_ac: [AC-007B-006]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，Status of Approval 未选择
  when: 点击 [Confirm]
  then:
    - [Confirm] 按钮禁用（或前端校验提示"请选择审批结果"）
    - 表单不提交
```

```yaml
tc_id: TC-007B-012
covers_ac: []
priority: P1
test_level: unit
steps:
  given: Dialog 已打开
  when: 展开 Status of Approval 下拉
  then:
    - 显示 5 个选项（A / B / C / D / E）含完整描述文案
    - A：Approved / No Exception Taken
    - B：Approved with comment, resubmission required
    - C：Revise And Resubmit
    - D：For Record Purpose
    - E：Others (please state reason)
```

```yaml
tc_id: TC-007B-013
covers_ac: [AC-007B-007]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开
  when: 分别选择 Status A、B、D
  then:
    - 显示 Signed Drawing File（必填）
    - 显示 Approval Evidence（必填）
    - 显示 External Approval Date（必填）
    - 不显示 Others Reason
    - 报审三要素始终可见且必填
```

```yaml
tc_id: TC-007B-014
covers_ac: []
priority: P1
test_level: unit
steps:
  given: Dialog 已打开
  when: 选择 Status C
  then:
    - 不显示 Signed Drawing File / Approval Evidence / External Approval Date
    - 不显示 Others Reason
    - 报审三要素始终可见且必填
```

```yaml
tc_id: TC-007B-015
covers_ac: [AC-007B-007B]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开
  when: 选择 Status E
  then:
    - 显示 Others Reason 文本域（必填，最多 500 字符）
    - 不显示签字版/凭证/日期字段
    - 报审三要素始终可见且必填
```

### TC 组 5：报审三要素始终必填

```yaml
tc_id: TC-007B-016
covers_ac: [AC-007B-006]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status A
  when: Submission Ref No. 留空，点击 [Confirm]
  then:
    - 前端校验阻止提交
    - 字段标红并提示"必填"
```

```yaml
tc_id: TC-007B-017
covers_ac: [AC-007B-006, AC-007B-011]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status C（退回路径）
  when: Submission Subject 留空，点击 [Confirm]
  then:
    - 前端校验阻止提交（退回路径仍必填）
    - 字段标红并提示"必填"
```

```yaml
tc_id: TC-007B-018
covers_ac: [AC-007B-006]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status E
  when: Submission Description 留空，点击 [Confirm]
  then:
    - 前端校验阻止提交
    - 字段标红并提示"必填"
```

### TC 组 6：Status E — Others Reason 必填

```yaml
tc_id: TC-007B-019
covers_ac: [AC-007B-007B]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status E，填写报审三要素
  when: Others Reason 留空，点击 [Confirm]
  then:
    - 前端校验阻止提交
    - Others Reason 字段标红并提示"必填"
```

### TC 组 7：Status A/B/D — 文件与日期校验

```yaml
tc_id: TC-007B-020
covers_ac: [AC-007B-007]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status A，已填写报审三要素
  when: 不上传 Signed Drawing File，点击 [Confirm]
  then:
    - 前端校验阻止提交，字段标红
```

```yaml
tc_id: TC-007B-021
covers_ac: [AC-007B-007]
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status D，已填写三要素 + 两个文件
  when: 不填 External Approval Date，点击 [Confirm]
  then:
    - 前端校验阻止提交，日期字段标红
```

```yaml
tc_id: TC-007B-022
covers_ac: []
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status A
  when: 尝试在日期选择器选择未来日期
  then:
    - 未来日期置灰，无法选中
```

```yaml
tc_id: TC-007B-023
covers_ac: []
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status A
  when: 上传 Signed Drawing File 大小超过 50MB
  then:
    - 前端 before-upload 拦截，提示"文件不得超过 50MB"
```

```yaml
tc_id: TC-007B-024
covers_ac: []
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status A
  when: 上传格式不合法的文件（如 .exe）
  then:
    - 前端拦截，提示"仅支持 PDF/DWG/DXF/PNG/JPG 格式"
```

```yaml
tc_id: TC-007B-025
covers_ac: []
priority: P1
test_level: unit
steps:
  given: Dialog 已打开，选择 Status B
  when: 上传 Approval Evidence 超过 20MB
  then:
    - 前端拦截，提示"文件不得超过 20MB"
```

### TC 组 8：Status A/B/D 成功路径

```yaml
tc_id: TC-007B-026
covers_ac: [AC-007B-008]
priority: P1
test_level: e2e
preconditions:
  - 已选择 Status A，填写报审三要素 + 合法签字版 + 合法凭证 + 过去日期
steps:
  given: Dialog 完整填写
  when: 点击 [Confirm]，等待 3–5 秒
  then:
    - [Confirm] 变为"Processing..."，Dialog 内所有操作禁用
    - Dialog 关闭
    - Toast："External approval marked. Drawing is now active and QR code has been generated."
    - Todo 卡片消失
    - DrawingVersion.approvalStatus = APPROVED
    - DrawingApproval 包含 statusOfApproval=A、submissionRefNo、submissionSubject、submissionDescription
```

```yaml
tc_id: TC-007B-027
covers_ac: [AC-007B-009]
priority: P2
test_level: integration
preconditions:
  - TC-007B-026 执行成功，版本已 ACTIVE
steps:
  given: 外部审批通过，版本已生效
  when: 项目管理员在图纸列表完成 [Assign] 操作（REQ-003D-pc）
  then:
    - 被分配的 SE 收到站内通知
    - SE 在 APP 端可查看签字版图纸
```

### TC 组 9：QR 生成失败回滚

```yaml
tc_id: TC-007B-028
covers_ac: [AC-007B-010]
priority: P1
test_level: integration
preconditions:
  - Dialog 完整填写（Status A），模拟 QR 生成服务异常
steps:
  given: 点击 [Confirm] 后 QR 服务抛出 QR_GENERATION_FAILED
  when: 接口返回失败
  then:
    - 版本状态不变（保持 PENDING_EXTERNAL）
    - Dialog loading 恢复，表单数据保留
    - Toast："QR generation failed, please retry"
    - DC 可立即重试
```

### TC 组 10：Status C 成功路径

```yaml
tc_id: TC-007B-029
covers_ac: [AC-007B-011, AC-007B-012]
priority: P1
test_level: e2e
preconditions:
  - 选择 Status C，已填写报审三要素
steps:
  given: Status C 路径完整填写
  when: 点击 [Confirm]，接口成功
  then:
    - Dialog 关闭
    - Toast："External rejection recorded. Designer has been notified."
    - Todo 消失
    - DrawingVersion 状态 → EXTERNAL_REJECTED
    - 设计人员收到站内消息通知
```

### TC 组 11：Status E 成功路径

```yaml
tc_id: TC-007B-030
covers_ac: []
priority: P2
test_level: integration
preconditions:
  - 选择 Status E，填写报审三要素 + Others Reason
steps:
  given: Status E 完整填写
  when: 点击 [Confirm]，接口成功
  then:
    - Dialog 关闭，Toast 提示退回记录成功
    - Todo 消失
    - 设计人员通知内容包含 Others Reason 文本
```

### TC 组 12：多 DC 并发

```yaml
tc_id: TC-007B-031
covers_ac: [AC-007B-013]
priority: P1
test_level: integration
preconditions:
  - 项目配置了 DC-A 和 DC-B，两人 Todo 列表均有同一外部审批任务
steps:
  given: DC-A 完成外部审批标记（任意 Status）
  when: DC-B 刷新 Todo 列表
  then:
    - DC-B 的 Todo 中该任务自动消失（状态 → S-004 CLOSED_BY_OTHER）
```

### TC 组 13：权限控制

```yaml
tc_id: TC-007B-032
covers_ac: [AC-007B-014]
priority: P1
test_level: integration
steps:
  given: 用户未被配置为项目 DC
  when: 直接调用 POST /drawing/external-approve
  then:
    - 接口返回 403
```

```yaml
tc_id: TC-007B-033
covers_ac: [AC-007B-014]
priority: P1
test_level: e2e
steps:
  given: 无 drawing:external-approval 权限用户进入 Todo 列表
  when: 查看页面
  then:
    - 不显示 External Approval Required 任务
    - Drawer 的 [Mark Result] 按钮不可见
```

### TC 组 14：防重复提交

```yaml
tc_id: TC-007B-034
covers_ac: [AC-007B-015]
priority: P1
test_level: unit
preconditions:
  - Dialog 完整填写，点击 [Confirm] 后接口请求进行中
steps:
  given: loading 状态中
  when: 再次点击 [Confirm]
  then:
    - 按钮处于 loading 禁用态，不触发重复请求
```

```yaml
tc_id: TC-007B-035
covers_ac: [AC-007B-015]
priority: P1
test_level: integration
preconditions:
  - 模拟接口返回 500
steps:
  given: 点击 [Confirm] 后接口失败
  when: 错误返回
  then:
    - loading 恢复，[Confirm] 可再次点击
    - Dialog 保留，表单数据不清空
    - Toast："Operation failed, please retry"
```

---

## 5. 边界值与等价类

### 5.1 文本字段边界

| 字段 | 最大限制 | 临界值 | 超限值 | 期望 |
|------|---------|-------|-------|------|
| Submission Ref No. | 200 字符 | 200 | 201 | 200 接受 / 201 截断或拒绝 |
| Submission Subject | 200 字符 | 200 | 201 | 同上 |
| Submission Description | 1000 字符 | 1000 | 1001 | 同上 |
| Others Reason | 500 字符 | 500 | 501 | 同上 |
| Remarks | 500 字符 | 500 | 501 | 同上 |

### 5.2 文本字段 fuzzing

| 输入 | 期望 |
|-----|------|
| 含 emoji | 接受，正确存储和展示 |
| 含中文 | 接受 |
| XSS `<script>alert(1)</script>` | 安全转义，不执行 |
| SQL 注入 `'; DROP TABLE` | 安全处理，正确存储 |

### 5.3 文件边界

| 文件 | 格式 | 大小边界 | 期望 |
|------|------|---------|------|
| Signed Drawing File | PDF | 50MB（临界） | 接受 |
| Signed Drawing File | PDF | 50MB + 1B | 拒绝 |
| Signed Drawing File | .exe | 任意 | 拒绝（格式不合法） |
| Approval Evidence | JPG | 20MB（临界） | 接受 |
| Approval Evidence | JPG | 20MB + 1B | 拒绝 |

### 5.4 日期边界

| 输入 | 期望 |
|-----|------|
| 今天 | 接受 |
| 过去任意日期 | 接受 |
| 明天 | 禁止选择（picker 置灰） |

---

## 6. 接口测试

### 6.1 GET /drawing/version/{versionId}/download

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-007B-DL01 | 正常请求（项目 DC） | 合法 versionId + DC token | 302 → attachment 预签名 URL |
| TC-API-007B-DL02 | 非项目 DC | 非 DC token | 403 |
| TC-API-007B-DL03 | 无 token | — | 401 |

### 6.2 GET /drawing/version/{versionId}/view

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-007B-V01 | 正常请求（项目 DC） | 合法 versionId + DC token | 302 → inline 预签名 URL |
| TC-API-007B-V02 | 非项目 DC | 非 DC token | 403 |

### 6.3 POST /drawing/external-approve

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-007B-E01 | Status A 正常请求 | 全字段 + DC token | 200，版本 → APPROVED，QR 生成 |
| TC-API-007B-E02 | Status C 正常请求 | statusOfApproval=C + 报审三要素 | 200，版本 → EXTERNAL_REJECTED |
| TC-API-007B-E03 | Status E 正常请求 | statusOfApproval=E + 三要素 + othersReason | 200 |
| TC-API-007B-E04 | submissionRefNo 缺失 | 不含 submissionRefNo | 400 |
| TC-API-007B-E05 | submissionSubject 缺失 | 不含 submissionSubject | 400 |
| TC-API-007B-E06 | submissionDescription 缺失 | — | 400 |
| TC-API-007B-E07 | Status A 时 signedFileKey 缺失 | statusOfApproval=A，无 signedFileKey | 400 |
| TC-API-007B-E08 | Status E 时 othersReason 缺失 | statusOfApproval=E，无 othersReason | 400 |
| TC-API-007B-E09 | 未登录 | 无 token | 401 |
| TC-API-007B-E10 | 非项目 DC | 非 DC token | 403 |
| TC-API-007B-E11 | Todo 已关闭 | 已处理的 versionId | 403 / 业务错误码 |
| TC-API-007B-E12 | submissionRefNo 超长（201 字符） | 201 字符 | 400 |

---

## 7. 性能测试

| 指标 | 目标 | 工具 |
|-----|------|-----|
| 外部审批标记接口（含 QR 生成）P95 | ≤ 8s | JMeter / k6 |
| 50MB 文件上传完成 | ≤ 90s | 实测 |

---

## 8. 安全测试

| TC ID | 描述 | 期望 |
|-------|-----|-----|
| SEC-007B-001 | 伪造 JWT 调用外部审批接口 | 401 |
| SEC-007B-002 | 无 `drawing:external-approval` 权限调用接口 | 403 |
| SEC-007B-003 | 已关闭 Todo 重复提交 | 403 / 业务错误码 |
| SEC-007B-004 | 文件上传 path traversal（文件名含 `../`） | 服务端过滤，正常存储 |
| SEC-007B-005 | 关键操作审计日志 | 记录操作人、时间、上传文件信息 |

---

## 9. 兼容性测试

| 维度 | 测试矩阵 |
|-----|---------|
| 浏览器 | Chrome 100+、Edge 100+、Safari 15+ |
| 移动端 | 不支持（PC 专属，无需测试） |
| 分辨率 | 1280px、1440px、1920px |
| 网络 | 正常 + 弱网（验证 loading 和超时处理） |

---

## 10. 测试数据

| 数据 | 示例值 |
|------|-------|
| DC 账号（项目已配置） | `dc_user_01`（DC-A）、`dc_user_02`（DC-B） |
| 设计人员账号 | `designer_01` |
| 项目管理员账号 | `admin_01` |
| SE 账号 | `se_user_01` |
| 非 DC 用户 | `normal_user` |
| 测试图纸 | ARCH-001（首层平面图，V3，状态 `INTERNAL_APPROVED`） |
| 合法签字版文件 | `arch001-v3-signed.pdf`（< 50MB） |
| 合法凭证文件 | `bentley-evidence.pdf`（< 20MB） |
| 超大签字版文件 | `large-signed.pdf`（> 50MB） |
| 超大凭证文件 | `large-evidence.pdf`（> 20MB） |
| 非法格式文件 | `malware.exe` |
| 合法外部审批日期 | `2026-05-26`（今天）或更早 |
| 非法外部审批日期 | `2026-05-27`（明天） |
| Submission Ref No. | `BENTLEY-2026-00123` |

---

## 11. AC 覆盖矩阵

| AC ID | 描述简述 | 覆盖的 TC | 状态 |
|-------|---------|----------|------|
| AC-007B-001 | Todo 仅展示当前 DC 名下待办 | TC-007B-001、TC-007B-002 | TODO |
| AC-007B-002 | 点击 Detail 打开 Drawer；[Mark Result] 打开 Dialog | TC-007B-003、TC-007B-010 | TODO |
| AC-007B-003 | Drawer 展示完整字段，不含 Drawing Code/Name | TC-007B-004 | TODO |
| AC-007B-004 | 下载原始文件，文件名格式正确 | TC-007B-007 | TODO |
| AC-007B-005 | 非项目 DC 调用下载接口返回 403 | TC-007B-008、TC-API-007B-DL02 | TODO |
| AC-007B-006 | 报审三要素始终必填 | TC-007B-016 ~ TC-007B-018、TC-007B-011、TC-API-007B-E04~E06 | TODO |
| AC-007B-007 | Status A/B/D 时签字版/凭证/日期必填 | TC-007B-020 ~ TC-007B-021、TC-API-007B-E07 | TODO |
| AC-007B-007B | Status E 时 Others Reason 必填 | TC-007B-019、TC-API-007B-E08 | TODO |
| AC-007B-008 | Status A/B/D 成功路径（版本生效 + QR + Toast） | TC-007B-026 | TODO |
| AC-007B-009 | 管理员 Assign 后 SE 收到通知 | TC-007B-027 | TODO |
| AC-007B-010 | QR 生成失败回滚 | TC-007B-028 | TODO |
| AC-007B-011 | Status C 退回必填校验（报审三要素） | TC-007B-017 | TODO |
| AC-007B-012 | Status C 成功路径 | TC-007B-029 | TODO |
| AC-007B-013 | 其他 DC Todo 自动关闭 | TC-007B-031 | TODO |
| AC-007B-014 | 非项目 DC 不显示 [Mark Result]，接口 403 | TC-007B-032、TC-007B-033 | TODO |
| AC-007B-015 | Dialog loading 防重复提交 | TC-007B-034、TC-007B-035 | TODO |

---

## 12. 回归测试范围

| 受影响功能 | 影响原因 | 回归用例 |
|----------|---------|---------|
| 内部审批 Todo（REQ-007A-pc） | 同 TodoPanel 组件，外部审批卡片渲染可能影响内部审批卡片 | 内部审批 Todo 卡片展示、Approve/Reject 操作正常 |
| 图纸列表状态标签 | 版本状态含 EXTERNAL_REJECTED，StatusTag 需正确展示 | 各状态标签颜色/文案正确 |
| QR 生成（REQ-006-shared） | 外部审批通过时触发 QR 生成 | QR 叠加到签字版 PDF，pdfWithQrUrl 非空 |

---

## 13. 集成验收检查清单（上线前）

- [ ] 外部审批通过后 DrawingVersion.approvalStatus = `APPROVED`，isCurrent = true
- [ ] DrawingApproval 记录包含 statusOfApproval、submissionRefNo、submissionSubject、submissionDescription
- [ ] QR 叠加到签字版 PDF 成功（pdfWithQrUrl 非空）
- [ ] 外部审批通过后其余 DC 的 Todo 自动关闭（状态 → CLOSED_BY_OTHER）
- [ ] 外部审批驳回（Status C/E）后设计人员收到含报审说明的站内消息
- [ ] QR 生成失败时整体操作回滚，版本状态保持 PENDING_EXTERNAL
- [ ] 非项目 DC 调用 view/download/external-approve 接口均返回 403
- [ ] Status E 通知内容包含 Others Reason 文本

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 REQ-007B-pc@0.1.1 生成初稿 |
| 0.2.2 | 2026-05-26 | agent | 基于 REQ-007B-pc@0.2.2 全面重写：1）新增 F-002 Detail Drawer 相关 TC（TC-007B-003~009，覆盖 AC-007B-003/004/005）；2）Mark Result Dialog 改为 Status A–E 下拉，重构 TC 组 4–11；3）报审三要素由"Approved 时必填"改为"始终必填"（AC-007B-006/011）；4）新增 Others Reason 测试（AC-007B-007B）；5）移除旧版 Approved/Rejected Radio 相关用例；6）接口测试补充 Status C/E 路径及新字段校验；7）AC 覆盖矩阵更新至 AC-007B-015 |
