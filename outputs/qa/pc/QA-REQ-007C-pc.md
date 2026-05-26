---
doc_type: qa_spec
req_id: REQ-007C-pc
req_title: "PC 端 — 版本历史抽屉 4 阶段生命周期视图"
version: 0.2.5
status: draft
generated_from: REQ-007C-pc@0.2.5
generated_at: 2026-05-26
owner: ""
---

# QA 测试说明：PC 端 — 版本历史抽屉 4 阶段生命周期视图

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - [Mark Result] Dialog 逻辑与 REQ-007B-pc 共用同一组件，本文档仅验证从版本历史入口触发的行为；Dialog 内部完整流程见 QA-REQ-007B-pc。
> - Attachments 二级抽屉（F-006）由主列表 Actions 列 [Details] 按钮触发（自 v0.2.5 起，旧版 Attachments 列已移除）。
> - Part Print Tab 自 v0.2.3 引入，为只读展示，相关 TC 见 TC 组 9。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007C-pc @ v0.2.5 |
| 依赖需求 | REQ-007-shared、REQ-007A-pc、REQ-007B-pc、REQ-006-shared |
| 覆盖 Story | US-007C-001 |
| 覆盖 AC | AC-007C-001 ～ AC-007C-018 |
| 测试平台 | PC（Chrome 100+ / Edge 100+ / Safari 15+，1280px+） |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证版本历史抽屉的主列表展示（宽度、列完整性含 RFA No. 与 Actions 列）、5 种版本状态下 4 阶段卡片的正确渲染（含 ③ 卡片 Status A–E 展示）、底部步骤条颜色、卡片内操作按钮的显示条件（权限 + 状态）、签字版下载降级逻辑、Attachments 二级抽屉（由 [Details] 按钮触发，含 Part Print / Attachments 双 Tab，上传、删除、计数同步）等功能的正确性。

---

## 2. 测试策略

### 2.1 测试范围

**必测**：
- 抽屉尺寸与标题
- 主列表列完整性（Version、RFA No.、Status、Designer、Confirmed、QR、Actions 七列）
- RFA No. 字段显示规则（有值显示；无值显示 —）
- 5 种状态下 4 阶段卡片展示（APPROVED / INTERNAL_APPROVED / PENDING_INTERNAL / INTERNAL_REJECTED / EXTERNAL_REJECTED）
- ③ 外部审批卡片 Status A–E 展示逻辑（A/B/D → ✅ + Evidence [Download]；C/E → ❌ + 无 Evidence）
- DC 用户的 [Mark Result] 按钮——仅在 INTERNAL_APPROVED 状态且为项目 DC 时显示
- 非 DC 用户不显示 [Mark Result]
- 签字版 [Download] 优先返回 pdfWithQrUrl，降级返回 signedFileUrl
- 底部步骤条颜色与状态对应
- Attachments 二级抽屉：由 Actions 列 [Details] 按钮触发（与行展开/折叠无关）
- 附件抽屉顶部版本信息字段（Status、Ver、Description、Submission Ref No.、Submission Subject、Uploader、Upload Date、主文件行）
- Part Print Tab（只读，倒序，无操作按钮）
- Attachments Tab：上传（仅上传人）、删除（仅上传人，含二次确认）、计数同步

**不测（本期）**：
- [Mark Result] Dialog 内部完整流程（见 QA-REQ-007B-pc）
- QR 查看/下载详细逻辑（REQ-006）
- APP 端版本历史

### 2.2 测试金字塔

| 层级 | 占比 | 说明 |
|-----|-----|------|
| 单元测试 | 55% | 5 种状态 → 卡片渲染逻辑、步骤条颜色映射、文件 URL 降级逻辑、Status A–E 展示映射 |
| 集成测试 | 30% | 版本历史 API 数据驱动渲染、[Mark Result] Dialog 触发、附件 CRUD |
| E2E 测试 | 15% | 完整查看版本历史 + 展开 + 附件抽屉操作 |

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-007C-001 | 打开版本历史抽屉，查看主列表（含 RFA No. 与 Actions 列） | US-007C-001 | P1 |
| SC-007C-002 | 展开 APPROVED 版本，查看 4 阶段卡片完整内容（含 Status A/B/D 凭证展示） | US-007C-001 | P1 |
| SC-007C-003 | DC 在 INTERNAL_APPROVED 版本展开后点击 [Mark Result] | US-007C-001 | P1 |
| SC-007C-004 | 点击 Actions 列 [Details] 按钮，打开附件二级抽屉，查看 Part Print Tab 与 Attachments Tab | US-007C-001 | P1 |
| SC-007C-005 | 上传人在 Attachments Tab 上传/删除附件 | US-007C-001 | P1 |

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
| SC-007C-P03 | 非本版本上传人查看 Attachments Tab，不显示 [+] 和 [Delete] |

### 3.4 状态对应渲染场景

| 场景 ID | approvalStatus | 测试核心 |
|--------|---------------|---------|
| SC-007C-ST01 | `APPROVED` | 4 卡片全部完成，步骤条全绿；③ 卡片显示 Status A/B/D |
| SC-007C-ST02 | `INTERNAL_APPROVED` | ① ② 完成，③ 进行中（DC 可见 [Mark Result]），④ 等待 |
| SC-007C-ST03 | `PENDING_INTERNAL` | ① 完成，② 进行中，③ ④ 等待 |
| SC-007C-ST04 | `INTERNAL_REJECTED` | ① 完成，② 驳回（红），③ ④ Not reached |
| SC-007C-ST05 | `EXTERNAL_REJECTED` | ① ② 完成，③ 驳回显示 Status C 或 E 标签，④ Not reached |

---

## 4. 测试用例

### TC 组 1：抽屉主列表基础

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-001 | AC-007C-001 | 用户有权访问图纸列表 | 点击某图纸的 [History] 按钮 | 抽屉从右侧滑入，宽度 720px；标题格式 `Version History — {drawingCode} {drawingName}` |
| TC-007C-002 | AC-007C-002 | 抽屉已打开，图纸有多个版本（含 APPROVED 和其他状态，部分版本有 RFA No.） | 查看主列表 | 依次出现 Version（含 ▶）、RFA No.、Status、Designer、Confirmed、QR、Actions 七列；APPROVED 版本 Confirmed 显示 `x/y`，其他显示 `—`；APPROVED 且 QR 已生成时 QR 列显示 `[View]`，否则 `—`；有 RFA No. 的版本显示对应值，无则显示 `—`；Actions 列每行均显示 [Details] 按钮 |
| TC-007C-003 | AC-007C-002 | 同上，**不含** Attachments 列 | 检查列名 | 主列表不存在名为 "Attachments" 的列（已于 v0.2.5 移除） |

### TC 组 2：状态标签颜色

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-004 | AC-007C-002 | 主列表含各种状态版本 | 检查 Status 列标签 | `PENDING_INTERNAL` → ⏳ Pending Internal（橙）；`INTERNAL_APPROVED` → ⏳ Pending External（橙）；`INTERNAL_REJECTED` → ❌ Int. Rejected（红）；`APPROVED` → ✅ Approved（绿）；`EXTERNAL_REJECTED` → ❌ Ext. Rejected（红） |

### TC 组 3：APPROVED 版本展开（4 卡片全完成）

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-005 | AC-007C-003 | 版本 approvalStatus = APPROVED | 点击 ▶ 展开该版本 | ① 卡片：✅ 标识、原始文件名、Designer、上传时间、文件大小、[Preview]、[Download] |
| TC-007C-006 | AC-007C-003 | 同上 | 同上 | ② 卡片：✅ Approved、审批人姓名、审批时间、Comment（若有） |
| TC-007C-007 | AC-007C-003 | 同上，外部审批结果为 Status A 或 B 或 D | 同上 | ③ 卡片：✅ 图标、Status 标签文字（如 "A – Approved / No Exception Taken"）、DC 姓名、External Approval Date、Evidence [Download]（evidenceFileUrl 非空时）、Remark（若有） |
| TC-007C-008 | AC-007C-003 | 同上 | 同上 | ④ 卡片：签字版文件名、上传时间、文件大小、[Preview]、[Download] |
| TC-007C-009 | AC-007C-003 | 同上 | 同上 | 底部步骤条：4 步全为绿色 ✓ |
| TC-007C-010 | AC-007C-009 | APPROVED 版本，pdfWithQrUrl 存在 | 点击 ④ 卡片 [Download] | 下载的是 pdfWithQrUrl（带 QR 水印签字版 PDF） |
| TC-007C-011 | AC-007C-010 | APPROVED 版本，pdfWithQrUrl 为 null | 点击 ④ 卡片 [Download] | 下载的是 signedFileUrl（签字版原始文件） |

### TC 组 4：③ 卡片 Status C / E 展示

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-012 | AC-007C-007 | 版本 approvalStatus = EXTERNAL_REJECTED，外部审批 Status = C | 点击 ▶ 展开 | ③ 卡片：❌ 图标、`"C – Revise And Resubmit"` 文字、DC 姓名、驳回时间、Remarks（若有）；**无** Evidence [Download] 按钮；④ 卡片 Not reached；步骤条：① ② 绿，③ 红，④ 灰 |
| TC-007C-013 | AC-007C-007 | 版本 approvalStatus = EXTERNAL_REJECTED，外部审批 Status = E，othersReason = "See Attachment" | 点击 ▶ 展开 | ③ 卡片：❌ 图标、`"E – Others: See Attachment"` 文字；**无** Evidence [Download] 按钮；④ 卡片 Not reached |

### TC 组 5：INTERNAL_APPROVED 版本展开（等待外部审批）

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-014 | AC-007C-004 | 版本 approvalStatus = INTERNAL_APPROVED，当前用户为项目 DC | 点击 ▶ 展开 | ③ 卡片：⏳ Pending 文字 + [✅ Mark Result] 按钮；④ 卡片：等待提示；步骤条：① ② 绿，③ 橙 ⏳，④ 灰 |
| TC-007C-015 | AC-007C-004 | 同上，当前用户为项目 DC | 点击 [Mark Result] | 弹出与 REQ-007B-pc §7.2 相同的 Dialog（验证打开即可，内部流程见 QA-REQ-007B-pc） |
| TC-007C-016 | AC-007C-005 | 版本 approvalStatus = INTERNAL_APPROVED，当前用户**不是**项目 DC | 点击 ▶ 展开 | ③ 卡片：⏳ Pending 文字，**不显示** [Mark Result] 按钮 |

### TC 组 6：PENDING_INTERNAL 版本展开

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-017 | AC-007C-008 | 版本 approvalStatus = PENDING_INTERNAL | 点击 ▶ 展开 | ② 卡片：⏳ Pending 及指定内部审批人；③ ④ 卡片：等待提示；步骤条：① 绿，② 橙 ⏳，③ ④ 灰 |

### TC 组 7：INTERNAL_REJECTED 版本展开

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-018 | AC-007C-006 | 版本 approvalStatus = INTERNAL_REJECTED | 点击 ▶ 展开 | ② 卡片：❌ Rejected、审批人、驳回时间、驳回 Comment；③ ④ 卡片：— Not reached；步骤条：① 绿，② 红 ✕，③ ④ 灰 |

### TC 组 8：[Details] 按钮与附件抽屉触发

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-019 | AC-007C-011 | 主列表已加载，某版本行**未展开** | 点击该行 Actions 列 [Details] 按钮 | 附件抽屉从右侧滑入（叠加在版本历史抽屉之上）；该版本行**展开状态不变**（仍为折叠） |
| TC-007C-020 | AC-007C-011 | 主列表已加载，某版本行**已展开** | 点击该行 Actions 列 [Details] 按钮 | 附件抽屉正常打开；该版本行**保持展开状态**，不自动折叠 |
| TC-007C-021 | AC-007C-012 | [Details] 附件抽屉已打开（版本有 Description 和 Submission 信息） | 查看抽屉顶部 | 宽度 480px；标题格式 `Version History — · {drawingName} {versionNo}`（**不含** Drawing Code）；顶部只读字段：Status 标签、Ver (System)、Description、Submission Ref No.、Submission Subject、Uploaded by（头像 + 姓名）、Upload Date（DD-MM-YYYY HH:mm:ss）、主文件行（文件名 + ✅ Approved by {name} · {time} + [Download]） |
| TC-007C-022 | AC-007C-012 | [Details] 附件抽屉已打开（版本无 Description 和 Submission 信息） | 查看顶部相应字段 | Description、Submission Ref No.、Submission Subject 字段显示 `—` |
| TC-007C-023 | AC-007C-012 | 附件抽屉已打开 | 查看默认激活 Tab | 默认激活 `Attachments` Tab（非 Part Print Tab） |

### TC 组 9：Part Print Tab（只读）

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-024 | AC-007C-017 | 附件抽屉已打开，该版本有 Part Print 记录 | 点击 `Part Print` Tab | 列表按发布时间倒序排列；每条卡片显示：Title、发布日期（MMM D, YYYY）、创建人、Based on 版本号（及 Page No. 若有）、Description（超 3 行可展开）、附件文件名（可点击下载） |
| TC-007C-025 | AC-007C-018 | 同上，已切换到 Part Print Tab | 检查 Tab 内是否有操作按钮 | 无发布、编辑、删除等任何操作按钮（纯只读） |
| TC-007C-026 | AC-007C-017 | 附件抽屉已打开，该版本**无** Part Print 记录 | 点击 `Part Print` Tab | 显示空状态"No Data" |
| TC-007C-027 | AC-007C-017 | Part Print Tab，标签页标题显示计数 | 查看 Tab 标题 | 格式 `Part Print (n)`，n 与列表条数一致 |

### TC 组 10：Attachments Tab 展示与权限

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-028 | AC-007C-013 | 附件抽屉已打开，Attachments Tab 默认激活，有附件，用户**不是**上传人 | 查看 Attachments Tab | 表格含附件列（文件名为链接/[Download]、Type、Size、Uploaded、Uploaded by）；[Download] 可点击下载；标题显示 `Attachments (n)`；**不显示** [+] 按钮，**不显示** [Delete] 按钮 |
| TC-007C-029 | AC-007C-013 | 同上，点击文件名链接或 [Download] | 下载对应附件 | 浏览器下载对应文件，文件名与上传时一致 |
| TC-007C-030 | AC-007C-014 | 附件抽屉已打开，当前用户是本版本上传人 | 查看 Attachments Tab | 显示 [+] 按钮；各附件行显示 [Delete] 按钮 |

### TC 组 11：Attachments Tab 上传

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-031 | AC-007C-015 | Attachments Tab 已打开，用户为上传人，Tab 当前显示 `Attachments (2)` | 点击 [+]，选择合法文件（< 50MB）并确认 | 上传成功：表格即时新增该行；Tab 标题变为 `Attachments (3)` |
| TC-007C-032 | AC-007C-015 | 同上 | 尝试上传超过 50MB 的文件 | Toast 提示文件过大；表格无新增行 |
| TC-007C-033 | — | 同上 | 尝试一次选择 6 个文件 | 前端提示"单次最多上传 5 个文件"；超出的文件不上传 |

### TC 组 12：Attachments Tab 删除

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-034 | AC-007C-016 | Attachments Tab，用户为上传人，有附件行 | 点击某附件行 [Delete] | 弹出确认 Dialog：`"Delete {fileName}? This action cannot be undone."` → [Cancel] / [Delete] |
| TC-007C-035 | AC-007C-016 | 删除确认 Dialog 已弹出 | 点击 [Delete] 确认 | 该附件行从表格移除；Tab 标题计数减 1 |
| TC-007C-036 | AC-007C-016 | 删除确认 Dialog 已弹出 | 点击 [Cancel] 取消 | Dialog 关闭，附件列表无变化 |

### TC 组 13：权限控制

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-037 | — | 当前用户为 Site Engineer | 尝试点击图纸列表的 [History] 按钮 | 按钮不可见；直接访问版本历史接口返回 403 |

### TC 组 14：异常处理

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007C-038 | — | 模拟版本历史接口返回 500 | 用户点击 [History] 打开抽屉 | 抽屉内显示错误状态，提示"加载失败，请刷新重试" |
| TC-007C-039 | — | 附件上传接口超时 | 上传文件中途断网 | 显示上传失败 Toast；表格无新增行 |

---

## 5. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-007C-001 | 版本历史抽屉宽度 | 720px |
| UI-007C-002 | 展开/折叠动画 | 流畅（目测 ≥ 60fps） |
| UI-007C-003 | 步骤条图标 | 📄 Uploaded → 🔍 Internal → 🌐 External → ✍️ Signed |
| UI-007C-004 | 步骤条颜色规范 | 完成：绿色；进行中：橙色；失败：红色；未到达：灰色 |
| UI-007C-005 | 4 阶段卡片布局 | 2×2 网格排列 |
| UI-007C-006 | 附件二级抽屉宽度 | 480px，叠加在版本历史抽屉之上 |
| UI-007C-007 | 状态标签颜色 | 与 F-002 规范一致（橙/红/绿） |
| UI-007C-008 | ③ 卡片 Status A/B/D 标识 | ✅ 图标 + 对应 Status 标签文字 |
| UI-007C-009 | ③ 卡片 Status C/E 标识 | ❌ 图标 + 对应 Status 标签文字；无 Evidence [Download] |

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

- [ ] 版本历史接口返回 DrawingApproval 列表（含 phase / rfaNo / submissionRefNo / submissionSubject 字段）
- [ ] 5 种状态版本均能正确渲染 4 阶段卡片
- [ ] ③ 卡片 Status A/B/D 展示 Evidence [Download]；Status C/E 无 Evidence [Download]
- [ ] [Mark Result] 按钮仅在 INTERNAL_APPROVED 且用户为项目 DC 时可见
- [ ] pdfWithQrUrl 优先，降级 signedFileUrl 逻辑正常
- [ ] [Details] 按钮触发附件抽屉，与版本行展开/折叠无关
- [ ] 附件抽屉顶部字段：Description / Submission Ref No. / Submission Subject 有值显示，无值显示 `—`
- [ ] Part Print Tab 列表只读，无操作按钮，按发布时间倒序
- [ ] Attachments Tab 计数与主列表 Tab 标签同步
- [ ] 附件上传/删除操作仅对本版本上传人开放（前端隐藏 + 后端 403 双重校验）

---

## 8. 测试数据

| 数据 | 说明 |
|------|-----|
| 管理员账号 | `admin_01`（可查看全版本） |
| DC 账号（已配置） | `dc_user_01` |
| 设计人员账号 | `designer_01`（版本上传人） |
| 内部审批人账号 | `internal_approver` |
| SE 账号 | `se_user_01`（应无法访问版本历史） |
| 图纸 ARCH-001 | 包含 5 种状态各一版本：V1 INTERNAL_REJECTED、V2 EXTERNAL_REJECTED（Status C）、V3 EXTERNAL_REJECTED（Status E，othersReason 非空）、V4 PENDING_INTERNAL、V5 INTERNAL_APPROVED（有 RFA No.）、V6 APPROVED（Status A，pdfWithQrUrl 有值）、V7 APPROVED（Status B，pdfWithQrUrl 为 null）|
| 附件测试文件 | 合法 xlsx（1MB）、过大 pdf（60MB）、合法 dwg（5MB） |
| Part Print 数据 | ARCH-001 V6 绑定 ≥ 2 条 Part Print 记录 |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 REQ-007C-pc@0.2.0 生成初稿（含 F-006 Attachments 弹框基础 TC） |
| 0.2.5 | 2026-05-26 | agent | 全量更新至 REQ-007C-pc@0.2.5。主要变更：① 溯源块 AC 范围扩至 AC-007C-018；② §2.1 测试范围新增 RFA No. 列、Actions 列（移除 Attachments 列）、③ 卡片 Status A–E 展示、附件抽屉顶部新字段（Description/Submission fields）、Part Print Tab；③ TC 组 1–2 更新列定义（TC-007C-002/003）；④ 新增 TC 组 4（Status C/E 展示）；⑤ TC 组 5 重命名（原 TC 组 4），更新 [Mark Result] TC 编号；⑥ 重构 TC 组 8（附件抽屉）：触发方式改为 Actions 列 [Details] 按钮，新增顶部信息字段 TC、默认 Tab TC；⑦ 新增 TC 组 9（Part Print Tab，覆盖 AC-007C-017/018）；⑧ TC 组 10–12 对应原 Attachments TC（计数/权限/上传/删除），更新编号；⑨ 集成验收清单新增 5 条；⑩ 测试数据新增 Status C/E 版本与 Part Print 数据 |
