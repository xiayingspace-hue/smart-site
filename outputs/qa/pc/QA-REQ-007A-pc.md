---
doc_type: qa_spec
req_id: REQ-007A-pc
req_title: "PC 端 — 内部审批 Todo 调整"
version: 0.2.0
status: draft
generated_from: REQ-007A-pc@0.2.0
generated_at: 2026-05-07
owner: ""
---

# QA 测试说明：PC 端 — 内部审批 Todo 调整

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - 与 REQ-007B/C/D-pc 联调时，本文档仅负责内部审批 Todo 侧；跨模块集成验收见各自文档。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007A-pc @ v0.2.0 |
| 依赖需求 | REQ-007-shared、REQ-003A-pc |
| 覆盖 Story | US-007A-001 |
| 覆盖 AC | AC-007A-001 ～ AC-007A-009、AC-007A-010、AC-007A-011 |
| 测试平台 | PC（Chrome 100+ / Edge 100+ / Safari 15+，1280px+） |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证 PC 端内部审批 Todo 卡片的展示、内部审批通过/驳回的完整交互流程、状态流转副作用（版本不生效、DC 收到通知、设计人员收到驳回通知），以及权限控制、前端校验、接口失败重试等边界行为的正确性。

---

## 2. 测试策略

### 2.1 测试范围

**必测**：
- Todo 卡片展示（图标、标题、字段、按钮）
- 内部审批通过——成功路径
- 内部审批通过——版本不触发生效（状态为 `PENDING_EXTERNAL`）
- 内部审批通过——DC 收到外部审批 Todo
- 内部审批驳回——Comment 必填校验
- 内部审批驳回——成功路径（设计人员收到通知）
- 驳回——旧版本保持 ACTIVE
- 对话框 loading 态防重复提交
- 接口失败后可重试
- **无 DC 配置时前端阻断（F-004 无 DC 警告弹窗，不打开确认对话框）**
- **无 DC 配置时后端兜底（POST /drawing/approve 返回 1003007012，版本保持 PENDING_INTERNAL）**

**不测（本期）**：
- DC 外部审批 Todo（REQ-007B-pc）
- 版本历史抽屉（REQ-007C-pc）
- DC 配置页面（REQ-007D-pc）
- APP 端内部审批 Todo

### 2.2 测试金字塔

| 层级 | 占比 | 说明 |
|-----|-----|------|
| 单元测试 | 60% | 前端校验逻辑（Comment 必填）、状态渲染 |
| 集成测试 | 25% | 前端 → 后端审批接口、审批后状态查询 |
| E2E 测试 | 15% | 完整通过/驳回流程，含 Toast + Todo 消失 |

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-007A-001 | 内部审批人在 PC Todo 看到 Internal Approval Required 卡片并完成通过 | US-007A-001 | P1 |
| SC-007A-002 | 内部审批人在 PC Todo 完成驳回，设计人员收到通知 | US-007A-001 | P1 |
| SC-007A-003 | 内部审批通过后 DC 收到外部审批 Todo | US-007A-001 | P1 |
| SC-007A-004 | 项目无 DC 时审批人点击 [Approve]，被前端拦截 | US-007A-001 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-007A-E01 | 驳回时 Comment 为空，前端阻止提交 | AC-007A-005 |
| SC-007A-E02 | 通过/驳回接口调用失败，对话框保留可重试 | AC-007A-009 |
| SC-007A-E03 | loading 期间重复点击 Confirm | AC-007A-008 |
| SC-007A-E04 | 前端预检查：项目无 DC 配置，点击 [Approve] 后弹出无 DC 警告弹窗 | AC-007A-010 |
| SC-007A-E05 | 后端兜底：绕过前端预检直接调用审批接口，后端返回 1003007012 | AC-007A-011 |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-007A-P01 | 无 `drawing:approve` 权限的用户调用审批接口，应返回 403 |
| SC-007A-P02 | 非指定内部审批人无法在 Todo 列表看到该任务 |

### 3.4 状态转换场景

**合法转换测试**：

| 场景 ID | From | Action | To | 测试要点 |
|--------|------|--------|-----|---------|
| SC-007A-T01 | PENDING | 审批人点击 Confirm（通过） | APPROVED（Todo 关闭） | DrawingVersion → `INTERNAL_APPROVED`；DC Todo 创建 |
| SC-007A-T02 | PENDING | 审批人驳回（含 Comment） | REJECTED（Todo 关闭） | DrawingVersion → `INTERNAL_REJECTED`；设计人员收到通知 |

**非法转换测试**：

| 场景 ID | From | 尝试 Action | 期望 |
|--------|------|------------|-----|
| SC-007A-TF01 | APPROVED（Todo 已关闭） | 再次调用审批接口 | 403 / 业务错误码，前端不展示已关闭 Todo |
| SC-007A-TF02 | — | 无权限用户调用 POST /drawing/approve | 403 |

---

## 4. 测试用例

### TC 组 1：Todo 卡片展示

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007A-001 | AC-007A-001 | 设计人员已上传图纸，指定了内部审批人（当前登录用户） | 内部审批人进入 PC Todo 列表 | 出现带 🔍 图标、标题"Internal Approval Required"的卡片；卡片包含图纸编号、名称、版本号、上传人姓名（Designer）、上传时间；Version Note 存在时显示，不存在时不显示该行 |
| TC-007A-002 | AC-007A-001 | 同上，但当前登录用户不是指定审批人 | 该用户进入 PC Todo 列表 | 列表中不出现该任务卡片 |
| TC-007A-003 | AC-007A-001 | 卡片已正常展示 | 检查卡片按钮区 | 同时显示 [View Drawing]、[Approve]、[Reject] 三个按钮；[Approve] 为蓝色主按钮；[Reject] 为默认样式 |

### TC 组 2：View Drawing

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007A-004 | — | 卡片正常展示 | 点击 [View Drawing] | 在线预览原始图纸文件（PDF 打开预览或触发下载），不影响卡片状态 |

### TC 组 3：内部审批通过

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007A-005 | AC-007A-001、AC-007A-002 | 卡片正常展示 | 点击 [Approve] | 弹出确认对话框，标题"Confirm Internal Approval?"；对话框文案中明确说明"通过后进入外部审批阶段，版本不立即生效" |
| TC-007A-006 | AC-007A-002 | 确认对话框已打开 | 点击 [Confirm] | 按钮进入 loading；接口调用成功后：对话框关闭，Todo 卡片消失，Toast 提示"Internal approval completed. DC has been notified for external approval." |
| TC-007A-007 | AC-007A-003 | TC-007A-006 执行成功 | 任意角色查看图纸列表或版本历史 | 该版本状态为 `PENDING_EXTERNAL`（或 UI 文案"Pending External"），而非 ACTIVE；Site Engineer 未收到推送通知 |
| TC-007A-008 | AC-007A-004 | TC-007A-006 执行成功，项目已配置 DC | 已配置 DC 角色用户进入 PC Todo 列表 | DC 收到"External Approval Required"任务（依据 REQ-007B-pc） |
| TC-007A-009 | AC-007A-002 | 确认对话框已打开 | 点击 [Cancel] | 对话框关闭，卡片状态不变，未触发任何接口请求 |

### TC 组 4：内部审批驳回

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007A-010 | AC-007A-005 | 卡片正常展示 | 点击 [Reject]，打开驳回对话框 | 对话框显示图纸编号、名称、版本号；包含 Comment 文本域（必填标记 *）；最多 500 字符提示 |
| TC-007A-011 | AC-007A-005 | 驳回对话框已打开，Comment 为空 | 直接点击 [Confirm] | 前端校验阻止提交：Comment 字段标红，显示必填提示；接口未调用 |
| TC-007A-012 | AC-007A-006 | 驳回对话框已打开 | 填写 Comment 后点击 [Confirm]，接口成功 | 对话框关闭，Todo 卡片消失，Toast 提示"Internal approval rejected. Designer has been notified." |
| TC-007A-013 | AC-007A-006 | TC-007A-012 执行成功 | 以设计人员账号查看站内消息 | 收到含驳回原因的站内消息（消息中包含填写的 Comment 内容） |
| TC-007A-014 | AC-007A-007 | 图纸存在已 ACTIVE 的旧版本（V1），设计人员上传了新版本（V2）并进入内部审批 | V2 内部驳回成功 | V1 版本状态仍为 ACTIVE，图纸主记录 status 不变；V2 状态为 `INTERNAL_REJECTED` |
| TC-007A-015 | AC-007A-005 | 驳回对话框已打开 | 输入 501 字符 | 前端限制最多 500 字符（输入被截断或计数提示超限，Confirm 禁用） |

### TC 组 5：对话框交互细节

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007A-016 | AC-007A-008 | 通过/驳回对话框已打开，点击 [Confirm] | 接口请求进行中 | [Confirm] 按钮显示 loading 态；对话框内所有按钮、输入框禁用；无法重复触发请求 |
| TC-007A-017 | AC-007A-009 | 通过对话框已打开，模拟接口返回 500 | 点击 [Confirm]，等待失败响应 | loading 恢复为正常态；对话框保留（不关闭）；Toast 显示错误文案；可再次点击 [Confirm] 重试 |
| TC-007A-018 | AC-007A-009 | 驳回对话框已打开，模拟接口返回 500 | 点击 [Confirm]，等待失败响应 | 同上：loading 恢复，对话框保留，Toast 报错，可重试 |

### TC 组 6：权限控制

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007A-019 | — | 用户不具备 `drawing:approve` 权限 | 直接调用 POST /drawing/approve | 接口返回 403；前端对该用户不展示 [Approve]/[Reject] 按钮 |
| TC-007A-020 | — | 审批人尝试对同一版本重复审批（Todo 已关闭） | 重新调用审批接口 | 接口返回业务错误（已处理），前端 Todo 列表不显示已关闭任务 |

### TC 组 7：无 DC 配置防护

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007A-021 | AC-007A-010 | 项目未配置任何 DC；审批人看到内部审批 Todo 卡片 | 点击 [Approve] | **不**弹出确认对话框；弹出无 DC 警告弹窗，标题"No DC Configured"；弹窗正文含"no Document Controller configured"语义；仅有 [Got it] 按钮 |
| TC-007A-022 | AC-007A-010 | 同上（项目无 DC） | 在无 DC 警告弹窗中点击 [Got it] | 弹窗关闭；卡片状态不变（仍可操作）；**[Reject] 按钮功能正常，不受无 DC 状态影响** |
| TC-007A-023 | AC-007A-011 | 项目无 DC；绕过前端（如 Postman/devtools）直接调用 POST /drawing/approve | 发送请求 | 接口返回错误码 `1003007012`，msg 含"No DC configured"；DrawingVersion 状态保持 `PENDING_INTERNAL`，未发生状态变更 |

---

## 5. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-007A-001 | 卡片图标 | 显示 🔍（区别于外部审批 🌐） |
| UI-007A-002 | 确认对话框文案 | 含"version will NOT become active until external approval is completed"语义 |
| UI-007A-003 | 驳回对话框字段 | Comment 文本域含 placeholder，字符限制 500，超限有提示 |
| UI-007A-004 | [Approve] 按钮样式 | 蓝色主按钮 |
| UI-007A-005 | Toast 文案（通过） | "Internal approval completed. DC has been notified for external approval." |
| UI-007A-006 | Toast 文案（驳回） | "Internal approval rejected. Designer has been notified." |

---

## 6. 非功能测试

| 类型 | 测试点 | 期望 |
|------|-------|------|
| 性能 | Todo 列表加载时间 | ≤ 1.5s（P95） |
| 性能 | 审批接口响应时间 | ≤ 2s（P95） |
| 安全 | JWT + `drawing:approve` 权限双重校验 | 缺少任一返回 401/403 |
| 审计 | 审批通过/驳回操作日志 | 记录操作人、时间、DrawingVersion ID |
| 可访问性 | 对话框内 Tab 键导航 | 可依次聚焦到所有交互元素 |
| 国际化 | 切换中/英语言 | 卡片标题、对话框、Toast 均正确切换 |

---

## 7. 集成验收检查清单（上线前）

- [ ] 内部审批通过后 DrawingVersion.approvalStatus = `INTERNAL_APPROVED`（非 `APPROVED`）
- [ ] 内部审批通过后所有已配置 DC 均收到 External Approval Required Todo
- [ ] 内部审批驳回后设计人员收到含驳回原因的站内消息
- [ ] 驳回不影响旧版本 ACTIVE 状态
- [ ] 审批操作审计日志落库
- [ ] 无 `drawing:approve` 权限时接口返回 403
- [ ] **项目无 DC 时点击 [Approve] 弹出无 DC 警告弹窗，[Reject] 流程不受影响**
- [ ] **绕过前端直接调用 POST /drawing/approve 在无 DC 时返回 1003007012**

---

## 8. 测试数据

| 数据 | 示例值 |
|------|-------|
| 内部审批人账号 | `internal_approver` |
| 设计人员账号 | `designer_01` |
| 已配置 DC 账号 | `dc_user_01` |
| 无权限账号 | `normal_user` |
| 测试图纸 | ARCH-001（首层平面图，V3，状态 PENDING_INTERNAL） |
| 含旧 ACTIVE 版本的图纸 | ARCH-002（V1 ACTIVE，V2 PENDING_INTERNAL） |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 REQ-007A-pc@0.1.0 生成初稿 |
| 0.2.0 | 2026-05-07 | agent | 同步 REQ-007A-pc@0.2.0：新增 SC-007A-004/E04/E05，TC 组7（TC-007A-021~023），覆盖 AC-007A-010/011 |
