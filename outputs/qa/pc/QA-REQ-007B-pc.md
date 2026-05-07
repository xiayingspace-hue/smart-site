---
doc_type: qa_spec
req_id: REQ-007B-pc
req_title: "PC 端 — DC 外部审批 Todo 与标记 Dialog"
version: 0.1.0
status: draft
generated_from: REQ-007B-pc@0.1.1
generated_at: 2026-05-07
owner: ""
---

# QA 测试说明：PC 端 — DC 外部审批 Todo 与标记 Dialog

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - SE 通知相关验收依赖管理员完成 [Assign] 操作（REQ-003D-pc）；QR 生成逻辑见 REQ-006-shared。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007B-pc @ v0.1.1 |
| 依赖需求 | REQ-007-shared、REQ-007A-pc、REQ-003D-pc |
| 覆盖 Story | US-007B-001、US-007B-002 |
| 覆盖 AC | AC-007B-001 ～ AC-007B-011 |
| 测试平台 | PC（Chrome 100+ / Edge 100+ / Safari 15+，1280px+） |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证 DC 外部审批 Todo 卡片的展示与触发时机、下载原始文件、Mark External Approval Result Dialog 的完整交互（通过/驳回两条路径）、版本生效与 QR 生成、多 DC 并发处理（一个 DC 操作完成后其他 DC Todo 自动关闭）、权限控制及失败重试等行为的正确性。

---

## 2. 测试策略

### 2.1 测试范围

**必测**：
- External Approval Required Todo 卡片展示与触发时机
- [Download Original] 文件下载
- [Mark Result] Dialog 打开
- Dialog 必填校验（Approved / Rejected 分支）
- 外部审批通过——成功路径（版本生效、QR 生成、其他 DC Todo 关闭）
- 外部审批通过——QR 生成失败回滚
- 外部审批驳回——成功路径（设计人员通知）
- 多 DC 并发：一 DC 操作完成后另一 DC Todo 自动关闭
- 非项目 DC / 无权限用户不展示 [Mark Result]
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
| SC-007B-001 | DC 从 Todo 下载原始文件并通过外部审批 | US-007B-001、US-007B-002 | P1 |
| SC-007B-002 | DC 从 Todo 驳回外部审批，设计人员收到通知 | US-007B-002 | P1 |
| SC-007B-003 | 一个 DC 操作后，同项目另一 DC Todo 自动关闭 | US-007B-001 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-007B-E01 | Approved 时任一必填字段（签字版/凭证/日期）为空 | AC-007B-003 |
| SC-007B-E02 | Rejected 时 Rejection Reason 为空 | AC-007B-007 |
| SC-007B-E03 | QR 生成失败，整体操作回滚 | AC-007B-006 |
| SC-007B-E04 | 文件上传超过大小限制 | — |
| SC-007B-E05 | 上传不支持的文件格式 | — |
| SC-007B-E06 | 接口调用失败（非 QR），Dialog 保留可重试 | AC-007B-011 |
| SC-007B-E07 | 未选择 Result 直接点击 Confirm | — |
| SC-007B-E08 | External Approval Date 选择未来日期 | — |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-007B-P01 | 未配置为项目 DC 的用户调用外部审批接口，返回 403 |
| SC-007B-P02 | 无 `drawing:external-approval` 权限的用户不显示 [Mark Result] |
| SC-007B-P03 | 非 DC 用户在 Todo 列表看不到 External Approval Required 任务 |

### 3.4 状态转换场景

**合法转换测试**：

| 场景 ID | From | Action | To | 测试要点 |
|--------|------|--------|-----|---------|
| SC-007B-T01 | S-001 PENDING | DC 标记通过（含必填字段） | S-002 APPROVED | DrawingVersion → `APPROVED`；QR 生成；其余 DC Todo → S-004 |
| SC-007B-T02 | S-001 PENDING | DC 标记驳回（含 Comment） | S-003 REJECTED | DrawingVersion → `EXTERNAL_REJECTED`；设计人员通知；其余 DC Todo → S-004 |
| SC-007B-T03 | S-001 PENDING | 其他 DC 先完成操作 | S-004 CLOSED_BY_OTHER | Todo 自动消失 |

**非法转换测试**：

| 场景 ID | From | 尝试 Action | 期望 |
|--------|------|------------|-----|
| SC-007B-TF01 | S-002/S-003/S-004（已关闭） | 再次调用外部审批接口 | 403 / 业务错误码，前端不展示已关闭 Todo |
| SC-007B-TF02 | — | 非项目 DC 调用 POST /drawing/external-approve | 403 |

---

## 4. 测试用例

### TC 组 1：Todo 卡片展示与触发时机

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007B-001 | AC-007B-001 | 内部审批已完成通过，项目已配置 DC-A 和 DC-B | DC-A 进入 PC Todo 列表 | 出现带 🌐 图标、标题"External Approval Required"的卡片；包含图纸信息、上传人、内部审批人及通过时间；卡片有 [📄 Download Original] 和 [✅ Mark Result] 按钮 |
| TC-007B-002 | AC-007B-001 | 同上 | DC-B 进入 PC Todo 列表 | DC-B 同样出现相同任务卡片 |
| TC-007B-003 | AC-007B-001 | 同上 | 非 DC 角色用户进入 PC Todo 列表 | 不显示 External Approval Required 任务 |
| TC-007B-004 | — | 卡片正常展示 | 检查卡片字段完整性 | 包含 drawingCode、drawingName、versionNo、Uploaded by（姓名 + 上传时间）、Internal approved by（内部审批人 + 通过时间）；Version Note 存在时显示，不存在时不显示该行 |

### TC 组 2：下载原始文件

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007B-005 | AC-007B-002 | DC 在 Todo 卡片中 | 点击 [📄 Download Original] | 浏览器触发文件下载；文件名与上传时原始文件名一致；不触发任何状态变更 |

### TC 组 3：Mark Result Dialog — Approved 路径

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007B-006 | — | DC 在 Todo 卡片中 | 点击 [✅ Mark Result] | 弹出"Mark External Approval Result" Dialog，显示图纸编号和版本号；Result 单选（默认未选中）；[Confirm] 按钮禁用 |
| TC-007B-007 | AC-007B-003 | Dialog 已打开，选择 Approved | 不填写任何 Approved 必填字段，直接点击 [Confirm] | 校验阻止：Signed Drawing File、Approval Evidence、External Approval Date 三个字段均标红并提示必填 |
| TC-007B-008 | AC-007B-003 | Dialog 已打开，选择 Approved | 只填写 Signed Drawing File，不填其余两个必填字段 | Approval Evidence 和 External Approval Date 标红提示必填 |
| TC-007B-009 | — | Dialog 已打开，选择 Approved | 上传超过 50MB 的 Signed Drawing File | 前端拦截，提示文件不得超过 50MB |
| TC-007B-010 | — | Dialog 已打开，选择 Approved | 上传不支持格式的 Signed Drawing File（如 .exe） | 前端拦截，提示仅支持 PDF/DWG/DXF/PNG/JPG |
| TC-007B-011 | — | Dialog 已打开，选择 Approved | 上传超过 20MB 的 Approval Evidence | 前端拦截，提示文件不得超过 20MB |
| TC-007B-012 | — | Dialog 已打开，选择 Approved | 尝试在 External Approval Date 选择明天的日期 | 日期选择器禁用未来日期，无法选中 |
| TC-007B-013 | AC-007B-004 | Dialog 已打开，选择 Approved，填写所有必填字段 | 点击 [Confirm]，等待 3-5 秒 | [Confirm] 显示"Processing..." loading；Dialog 内所有操作禁用；成功后：Dialog 关闭，Toast 提示"External approval marked. Drawing is now active and QR code has been generated."，Todo 卡片消失 |
| TC-007B-014 | AC-007B-004 | TC-007B-013 执行成功 | 查看图纸列表中该图纸的版本状态 | 版本状态变为 ACTIVE（`APPROVED`） |
| TC-007B-015 | AC-007B-005 | TC-007B-013 执行成功，管理员完成 [Assign] 操作 | 已分配的 SE 查看站内通知及 APP 图纸列表 | SE 收到站内通知，可在 APP 端查看签字版图纸 |
| TC-007B-016 | AC-007B-006 | Dialog 已打开，选择 Approved，填写所有必填字段，模拟 QR 生成服务异常 | 点击 [Confirm]，等待失败 | 版本状态不变（保持 `PENDING_EXTERNAL`）；Dialog 内 loading 恢复；Toast 提示"QR generation failed, please retry"；DC 可立即重试 |
| TC-007B-017 | — | Dialog Approved 路径 | 填写 Remarks 超过 500 字符 | 前端限制最多 500 字符（输入被截断或超限提示） |

### TC 组 4：Mark Result Dialog — Rejected 路径

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007B-018 | AC-007B-007 | Dialog 已打开，选择 Rejected | Rejection Reason 为空，点击 [Confirm] | 前端校验阻止提交，字段标红并提示必填 |
| TC-007B-019 | AC-007B-008 | Dialog 已打开，选择 Rejected，填写 Rejection Reason | 点击 [Confirm]，接口成功 | Dialog 关闭，Toast 提示"External rejection recorded. Designer has been notified."，Todo 消失 |
| TC-007B-020 | AC-007B-008 | TC-007B-019 执行成功 | 以设计人员账号查看站内消息 | 收到含驳回原因的站内消息 |
| TC-007B-021 | — | Dialog 已打开，选择 Rejected | 填写 Rejection Reason 超过 500 字符 | 前端限制最多 500 字符 |

### TC 组 5：多 DC 并发处理

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007B-022 | AC-007B-009 | 项目配置了 DC-A 和 DC-B，两人 Todo 列表均有同一外部审批任务 | DC-A 完成标记通过操作 | DC-B 的 Todo 列表中该任务自动消失（状态 → S-004 CLOSED_BY_OTHER）；DC-B 刷新后不出现该任务 |
| TC-007B-023 | AC-007B-009 | 同上 | DC-A 完成标记驳回操作 | DC-B 的 Todo 中同一任务自动关闭 |

### TC 组 6：权限控制

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007B-024 | AC-007B-010 | 用户未被配置为项目 DC | 直接调用 POST /drawing/external-approve | 接口返回 403 |
| TC-007B-025 | AC-007B-010 | 用户未被配置为项目 DC，但进入 Todo 列表 | 查看 Todo 列表 | 不显示 External Approval Required 任务；即使能看到卡片，[Mark Result] 按钮不可见 |

### TC 组 7：Dialog 防重复提交

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007B-026 | AC-007B-011 | Dialog 已打开，点击 [Confirm] 后接口请求进行中 | 再次点击 [Confirm] 或其他按钮 | 按钮处于 loading 禁用态，不触发重复请求 |
| TC-007B-027 | AC-007B-011 | 模拟接口失败（非 QR 失败） | 点击 [Confirm] 后失败 | loading 恢复；Dialog 保留；Toast 显示错误文案；可再次点击 [Confirm] 重试 |

### TC 组 8：Result 未选中

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007B-028 | — | Dialog 已打开，Result 未选中 | 直接点击 [Confirm] | [Confirm] 按钮禁用（默认不可点击）或前端提示"请选择审批结果" |

---

## 5. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-007B-001 | 卡片图标 | 显示 🌐（区别于内部审批 🔍） |
| UI-007B-002 | [📄 Download Original] 样式 | 普通按钮 |
| UI-007B-003 | [✅ Mark Result] 样式 | 绿色主按钮 |
| UI-007B-004 | Dialog — Approved 选中后显示字段 | Signed Drawing File、Approval Evidence、External Approval Date（必填）+ Remarks（选填）；Rejected 字段隐藏 |
| UI-007B-005 | Dialog — Rejected 选中后显示字段 | Rejection Reason（必填）；Approved 字段隐藏 |
| UI-007B-006 | [Confirm] loading 文案 | 显示"Processing..." |
| UI-007B-007 | Toast（通过成功） | "External approval marked. Drawing is now active and QR code has been generated." |
| UI-007B-008 | Toast（驳回成功） | "External rejection recorded. Designer has been notified." |

---

## 6. 非功能测试

| 类型 | 测试点 | 期望 |
|------|-------|------|
| 性能 | 外部审批标记接口（含 QR 生成）P95 响应 | ≤ 8s |
| 性能 | 文件上传 50MB 完成时间 | ≤ 90s |
| 安全 | JWT + `drawing:external-approval` 权限 + 项目 DC 校验 | 三重校验，缺一返回 401/403 |
| 审计 | 标记操作日志 | 记录操作人、时间、上传文件信息 |
| 可访问性 | Dialog 内 Tab 键导航 | 所有字段可通过 Tab 依次聚焦 |
| 国际化 | 切换中/英语言 | 卡片、Dialog、Toast 正确切换 |

---

## 7. 集成验收检查清单（上线前）

- [ ] 外部审批通过后 DrawingVersion.approvalStatus = `APPROVED`，isCurrent = true
- [ ] QR 叠加到签字版 PDF 成功（pdfWithQrUrl 非空）
- [ ] 外部审批通过后其余 DC 的 Todo 自动关闭
- [ ] 管理员 [Assign] 操作后 SE 收到站内通知（REQ-003D-pc 集成）
- [ ] 外部审批驳回后设计人员收到含驳回原因的站内消息
- [ ] QR 生成失败时整体操作回滚，版本状态不变
- [ ] 非项目 DC 调用接口返回 403

---

## 8. 测试数据

| 数据 | 示例值 |
|------|-------|
| DC 账号（项目已配置） | `dc_user_01`、`dc_user_02` |
| 设计人员账号 | `designer_01` |
| 管理员账号 | `admin_01` |
| SE 账号 | `se_user_01` |
| 非 DC 用户 | `normal_user` |
| 测试图纸 | ARCH-001（首层平面图，V3，状态 `INTERNAL_APPROVED`） |
| 合法签字版文件 | arch001-v3-signed.pdf（< 50MB） |
| 合法凭证文件 | bentley-approval.pdf（< 20MB） |
| 超大签字版文件 | large-file.pdf（> 50MB） |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 REQ-007B-pc@0.1.1 生成初稿 |
