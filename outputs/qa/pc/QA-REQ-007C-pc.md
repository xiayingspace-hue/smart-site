---
doc_type: qa_spec
req_id: REQ-007C-pc
req_title: "PC 端 — 版本历史抽屉 4 阶段生命周期视图"
version: 0.1.0
status: draft
generated_from: REQ-007C-pc@0.2.0
generated_at: 2026-05-07
owner: ""
---

# QA 测试说明：PC 端 — 版本历史抽屉 4 阶段生命周期视图

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - [Mark Result] Dialog 逻辑与 REQ-007B-pc 共用同一组件，本文档仅验证从版本历史入口触发的行为；Dialog 内部完整流程见 QA-REQ-007B-pc。
> - Attachments 功能（F-006）自 REQ-007C-pc v0.2.0 引入，相关 TC 见 TC 组 8。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007C-pc @ v0.2.0 |
| 依赖需求 | REQ-007-shared、REQ-007A-pc、REQ-007B-pc、REQ-006-shared |
| 覆盖 Story | US-007C-001 |
| 覆盖 AC | AC-007C-001 ～ AC-007C-016 |
| 测试平台 | PC（Chrome 100+ / Edge 100+ / Safari 15+，1280px+） |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证版本历史抽屉的主列表展示（宽度、列完整性、状态标签颜色）、5 种版本状态下 4 阶段卡片的正确渲染、底部步骤条颜色、卡片内操作按钮的显示条件（权限 + 状态）、签字版下载降级逻辑、Attachments 二级弹框（展示、上传、删除）等功能的正确性。

---

## 2. 测试策略

### 2.1 测试范围

**必测**：
- 抽屉尺寸与标题
- 主列表列完整性（Version、Status、Designer、Confirmed、QR、Attachments）
- 5 种状态下 4 阶段卡片展示（APPROVED / INTERNAL_APPROVED / PENDING_INTERNAL / INTERNAL_REJECTED / EXTERNAL_REJECTED）
- DC 用户的 [Mark Result] 按钮——仅在 `PENDING_EXTERNAL`（即 INTERNAL_APPROVED）状态且为项目 DC 时显示
- 非 DC 用户不显示 [Mark Result]
- 签字版 [Download] 优先返回 pdfWithQrUrl，降级返回 signedFileUrl
- 底部步骤条颜色与状态对应
- Attachments 二级弹框：触发方式、内容展示、上传（仅上传人）、删除（仅上传人，含二次确认）、计数同步

**不测（本期）**：
- [Mark Result] Dialog 内部完整流程（见 QA-REQ-007B-pc）
- QR 查看/下载详细逻辑（REQ-006）
- APP 端版本历史

### 2.2 测试金字塔

| 层级 | 占比 | 说明 |
|-----|-----|------|
| 单元测试 | 55% | 5 种状态 → 卡片渲染逻辑、步骤条颜色映射、文件 URL 降级逻辑 |
| 集成测试 | 30% | 版本历史 API 数据驱动渲染、[Mark Result] Dialog 触发 |
| E2E 测试 | 15% | 完整查看版本历史 + 展开 + 附件操作 |

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-007C-001 | 打开版本历史抽屉，查看主列表 | US-007C-001 | P1 |
| SC-007C-002 | 展开 APPROVED 版本，查看 4 阶段卡片完整内容 | US-007C-001 | P1 |
| SC-007C-003 | DC 在 INTERNAL_APPROVED 版本展开后点击 [Mark Result] | US-007C-001 | P1 |
| SC-007C-004 | 点击 Attachments 列，打开二级弹框，上传/删除附件 | US-007C-001 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-007C-E01 | 版本历史加载失败（网络异常） | — |
| SC-007C-E02 | pdfWithQrUrl 为 null，降级下载 signedFileUrl | AC-007C-010 |
| SC-007C-E03 | 附件上传超过 50MB | AC-007C-015 |
| SC-007C-E04 | 附件上传超过单次 5 个限制 | — |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-007C-P01 | 非项目 DC 查看 INTERNAL_APPROVED 版本展开卡片，不显示 [Mark Result] |
| SC-007C-P02 | Site Engineer 无法打开版本历史抽屉 |
| SC-007C-P03 | 非本版本上传人查看 Attachments 弹框，不显示 [+] 和 [Delete] |

### 3.4 状态对应渲染场景

| 场景 ID | approvalStatus | 测试核心 |
|--------|---------------|---------|
| SC-007C-ST01 | `APPROVED` | 4 卡片全部完成，步骤条全绿 |
| SC-007C-ST02 | `INTERNAL_APPROVED` | ① ② 完成，③ 进行中（DC 可见 [Mark Result]），④ 等待 |
| SC-007C-ST03 | `PENDING_INTERNAL` | ① 完成，② 进行中，③ ④ 等待 |
| SC-007C-ST04 | `INTERNAL_REJECTED` | ① 完成，② 驳回（红），③ ④ Not reached |
| SC-007C-ST05 | `EXTERNAL_REJECTED` | ① ② 完成，③ 驳回（红），④ Not reached |

---

## 4. 测试用例

### TC 组 1：抽屉主列表基础

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-001 | AC-007C-001 | 用户有权访问图纸列表 | 点击某图纸的 [History] 按钮 | 抽屉从右侧滑入，宽度 720px；标题格式"Version History — {drawingCode} {drawingName}" |
| TC-007C-002 | AC-007C-002 | 抽屉已打开，图纸有多个版本（含 APPROVED 和其他状态） | 查看主列表 | 包含 Version（左侧 ▶ 箭头）、Status、Designer、Confirmed、QR、Attachments 六列；APPROVED 版本的 Confirmed 显示"x/y"，其他状态显示"—"；APPROVED 且 QR 已生成时 QR 列显示"[View]"，否则显示"—" |
| TC-007C-003 | AC-007C-011 | 抽屉已打开 | 查看各版本 Attachments 列 | 有附件的版本显示"📎 n"；无附件显示"📎 0" |

### TC 组 2：状态标签颜色

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-004 | AC-007C-002 | 主列表含各种状态版本 | 检查 Status 列标签 | `PENDING_INTERNAL` → ⏳ Pending Internal（橙色）；`INTERNAL_APPROVED` → ⏳ Pending External（橙色）；`INTERNAL_REJECTED` → ❌ Int. Rejected（红色）；`APPROVED` → ✅ Approved（绿色）；`EXTERNAL_REJECTED` → ❌ Ext. Rejected（红色） |

### TC 组 3：APPROVED 版本展开（4 卡片全完成）

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-005 | AC-007C-003 | 版本 approvalStatus = APPROVED | 点击 ▶ 展开该版本 | ① 卡片显示：原始文件名、Designer、上传时间、文件大小、[Preview]、[Download] |
| TC-007C-006 | AC-007C-003 | 同上 | 同上 | ② 卡片显示：✅ Approved、审批人姓名、审批时间、Comment（若有） |
| TC-007C-007 | AC-007C-003 | 同上 | 同上 | ③ 卡片显示：✅ Approved、DC 姓名、External Approval Date、[Download]（凭证 evidenceFileUrl）、Remark（若有） |
| TC-007C-008 | AC-007C-003 | 同上 | 同上 | ④ 卡片显示：签字版文件名、上传时间、文件大小、[Preview]（signedFileUrl）、[Download] |
| TC-007C-009 | AC-007C-003 | 同上 | 同上 | 底部步骤条：4 步全为绿色 ✓ |
| TC-007C-010 | AC-007C-009 | APPROVED 版本，pdfWithQrUrl 存在 | 点击 ④ 卡片 [Download] | 下载的是 pdfWithQrUrl（带 QR 水印签字版 PDF） |
| TC-007C-011 | AC-007C-010 | APPROVED 版本，pdfWithQrUrl 为 null | 点击 ④ 卡片 [Download] | 下载的是 signedFileUrl（签字版原始文件） |

### TC 组 4：INTERNAL_APPROVED 版本展开（等待外部审批）

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-012 | AC-007C-004 | 版本 approvalStatus = INTERNAL_APPROVED，当前用户为项目 DC | 点击 ▶ 展开 | ③ 卡片显示"⏳ Pending"，包含 [✅ Mark Result] 按钮；④ 卡片显示"— Waiting for external approval"；底部步骤条：① ② 绿色，③ 橙色 ⏳，④ 灰色 — |
| TC-007C-013 | AC-007C-004 | 同上，当前用户为项目 DC | 点击 ③ 卡片的 [Mark Result] | 弹出与 REQ-007B-pc §7.2 相同的 Dialog（验证打开即可，Dialog 内流程见 QA-REQ-007B-pc） |
| TC-007C-014 | AC-007C-005 | 版本 approvalStatus = INTERNAL_APPROVED，当前用户**不是**项目 DC | 点击 ▶ 展开 | ③ 卡片显示"⏳ Pending"状态文字，但不显示 [✅ Mark Result] 按钮 |

### TC 组 5：PENDING_INTERNAL 版本展开

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-015 | AC-007C-008 | 版本 approvalStatus = PENDING_INTERNAL | 点击 ▶ 展开 | ② 卡片显示"⏳ Pending"及指定内部审批人；③ ④ 卡片显示等待内部审批的提示；底部步骤条：① 绿色，② 橙色 ⏳，③ ④ 灰色 — |

### TC 组 6：INTERNAL_REJECTED 版本展开

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-016 | AC-007C-006 | 版本 approvalStatus = INTERNAL_REJECTED | 点击 ▶ 展开 | ② 卡片显示：❌ Rejected、审批人、驳回时间、驳回 Comment；③ ④ 卡片显示"— Not reached"；底部步骤条：① 绿色，② 红色 ✕，③ ④ 灰色 — |

### TC 组 7：EXTERNAL_REJECTED 版本展开

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-017 | AC-007C-007 | 版本 approvalStatus = EXTERNAL_REJECTED | 点击 ▶ 展开 | ② 卡片显示：✅ Approved（内部）；③ 卡片显示：❌ Rejected、DC 姓名、驳回时间、驳回原因；④ 卡片显示"— Not reached"；底部步骤条：① ② 绿色，③ 红色 ✕，④ 灰色 — |

### TC 组 8：Attachments 二级弹框

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-018 | AC-007C-012 | 抽屉已打开，某版本行 Attachments 列显示"📎 3" | 点击该 Attachments 列单元格（无论版本行是否展开） | 从右侧滑入宽度 480px 的弹框，叠加在版本历史抽屉之上；标题格式"Version History — · {drawingCode} {versionNo}"；顶部显示版本信息（Drawing Code、Status、Ver、Uploaded by、Upload Date）及主文件行（含 [Download]）；Attachments 区块标题"Attachments (3)"，表格含 3 条记录 |
| TC-007C-019 | AC-007C-012 | 同上，无附件版本 Attachments 列显示"📎 0" | 点击该 Attachments 列单元格 | 弹框打开，Attachments 区块表格内显示"No Data" |
| TC-007C-020 | AC-007C-012 | 弹框已打开 | 关闭弹框（点击 [✕]） | 返回版本历史抽屉；原版本行展开/折叠状态不受影响 |
| TC-007C-021 | AC-007C-013 | 弹框已打开，有附件，用户为管理员（非上传人） | 点击附件行 [Download] 或文件名 | 浏览器下载对应附件；文件名与原始上传文件名一致 |
| TC-007C-022 | AC-007C-014 | 弹框已打开，当前用户**不是**本版本上传人 | 查看 Attachments 区块 | [+] 按钮不显示；所有附件行不显示 [Delete] 按钮 |
| TC-007C-023 | AC-007C-014 | 弹框已打开，当前用户是本版本上传人（设计人员） | 查看 Attachments 区块 | [+] 按钮可见；各附件行显示 [Delete] 按钮 |
| TC-007C-024 | AC-007C-015 | 弹框已打开，用户是本版本上传人 | 点击 [+]，选择合法文件（< 50MB）并确认上传 | 上传成功：附件表格即时新增该行；标题计数加 1；主列表 Attachments 列数字同步加 1 |
| TC-007C-025 | AC-007C-015 | 同上 | 尝试上传超过 50MB 的文件 | Toast 提示文件过大；表格不新增行 |
| TC-007C-026 | — | 同上 | 尝试一次上传 6 个文件 | 前端提示单次最多 5 个文件 |
| TC-007C-027 | AC-007C-016 | 弹框已打开，用户是本版本上传人，有附件行 | 点击某附件行 [Delete] | 弹出确认 Dialog：`"Delete {fileName}? This action cannot be undone."` → [Cancel] / [Delete] |
| TC-007C-028 | AC-007C-016 | 删除确认 Dialog 已弹出 | 点击 [Delete] 确认 | 该附件行从表格移除；标题计数减 1；主列表 Attachments 列数字同步减 1（减至 0 显示"📎 0"） |
| TC-007C-029 | AC-007C-016 | 删除确认 Dialog 已弹出 | 点击 [Cancel] 取消 | 弹框关闭，附件表格无变化 |

### TC 组 9：权限控制

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-030 | — | 当前用户为 Site Engineer | 尝试点击图纸列表的 [History] 按钮 | 按钮不可见；直接访问版本历史接口返回 403 |

### TC 组 10：异常处理

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-031 | — | 模拟版本历史接口返回 500 | 用户点击 [History] 打开抽屉 | 抽屉内显示错误状态，提示"加载失败，请刷新重试" |

---

## 5. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-007C-001 | 抽屉宽度 | 720px |
| UI-007C-002 | 展开/折叠动画 | 流畅（目测 ≥ 60fps） |
| UI-007C-003 | 步骤条图标 | 📄 Uploaded → 🔍 Internal → 🌐 External → ✍️ Signed |
| UI-007C-004 | 步骤条颜色规范 | 完成：绿色；进行中：橙色；失败：红色；未到达：灰色 |
| UI-007C-005 | 4 阶段卡片布局 | 2×2 网格排列 |
| UI-007C-006 | 二级弹框宽度 | 480px，叠加在版本历史抽屉之上 |
| UI-007C-007 | 状态标签颜色 | 与 §7.2 规范一致（橙/红/绿） |

---

## 6. 非功能测试

| 类型 | 测试点 | 期望 |
|------|-------|------|
| 性能 | 版本历史抽屉加载（10 版本内） | ≤ 1.5s |
| 性能 | 展开/折叠响应 | 即时，无明显卡顿 |
| 安全 | fileUrl / signedFileUrl / evidenceFileUrl | 均通过预签名 URL 访问，不直接暴露 OSS 路径 |
| 可访问性 | 展开/折叠 | 支持 Enter/Space 键操作 |
| 国际化 | 切换中/英语言 | 各状态标签、按钮、提示文案正确切换 |

---

## 7. 集成验收检查清单（上线前）

- [ ] 版本历史接口返回 DrawingApproval 列表（含 phase 字段）
- [ ] 5 种状态版本均能正确渲染 4 阶段卡片
- [ ] [Mark Result] 按钮仅在 `INTERNAL_APPROVED` 且用户为项目 DC 时可见
- [ ] pdfWithQrUrl 优先，降级 signedFileUrl 逻辑正常
- [ ] Attachments 二级弹框计数与主列表同步
- [ ] 附件上传/删除操作仅对本版本上传人开放
- [ ] 存量无签字版的 ACTIVE 版本 ④ 卡片降级展示（Legacy 注记，若 PM 确认）

---

## 8. 测试数据

| 数据 | 说明 |
|------|-----|
| 管理员账号 | `admin_01`（可查看全版本） |
| DC 账号（已配置） | `dc_user_01` |
| 设计人员账号 | `designer_01`（版本上传人） |
| 内部审批人账号 | `internal_approver` |
| SE 账号 | `se_user_01`（应无法访问版本历史） |
| 图纸 ARCH-001 | 包含 5 种状态各一个版本（V1 INTERNAL_REJECTED，V2 EXTERNAL_REJECTED，V3 PENDING_INTERNAL，V4 INTERNAL_APPROVED，V5 APPROVED） |
| 附件测试文件 | 合法 xlsx（1MB）、过大 pdf（60MB）、合法 dwg（5MB） |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 REQ-007C-pc@0.2.0 生成初稿（含 Attachments F-006 相关 TC） |
