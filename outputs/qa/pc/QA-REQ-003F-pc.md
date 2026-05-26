---
doc_type: qa_spec
req_id: REQ-003F-pc
req_title: "PC 端 — 上传新版本"
version: 0.5.0
status: draft
generated_from: REQ-003F-pc@0.5.0
generated_at: 2026-05-26
owner: ""
---

# QA 测试说明：PC 端 — 上传新版本

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - AI 识别服务集成需在 staging 环境使用 Mock AI 服务测试所有分支（DONE / FAILED / 超时）。
> - 列表页按钮置灰规则（AC-003F-011）与 REQ-003A-pc 共用，本文档侧重弹窗内部逻辑。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003F-pc @ v0.5.0 |
| 依赖需求 | REQ-003-shared、REQ-007-shared、REQ-003A-pc、REQ-003E-pc |
| 覆盖 Story | US-003F-001 |
| 覆盖 AC | AC-003F-001 ~ AC-003F-013 |
| 测试平台 | PC（Chrome 100+ / Edge 100+ / Safari 15+，1280px+） |
| 测试环境 | dev / staging（需 Mock AI 服务） |

---

## 1. 测试目标

验证"上传新版本"弹窗完整流程的正确性：弹窗只读字段展示、文件格式/大小校验、AI 识别中 Loading 状态与 [Submit] 置灰、AI 识别 DONE 后识别结果列表展示（含未识别页警告）、AI 识别 FAILED 降级路径（Re-upload / Proceed Anyway）、上传进度条防重复提交、提交成功后状态变更与 Todo 通知、以及列表页按钮在审批中状态时置灰行为。

---

## 2. 测试策略

### 2.1 测试范围

**必测**：
- 弹窗触发条件（允许/置灰状态）
- 弹窗副标题与只读字段（drawingCode/drawingName/currentVersion/category/description）
- PDF 文件类型校验（非 PDF 拒绝）
- 文件大小校验（> 50MB 拒绝）
- 上传后 AI 识别 Loading 状态，[Submit] 置灰
- AI 识别 DONE：识别结果列表展示（页码/缩略图/Drawing Code/Drawing Name）
- AI 识别 DONE：有未识别字段时 ⚠ 标记与橙色追加提示
- AI 识别 FAILED：降级横幅，[Re-upload] 清空重来，[Proceed Anyway] 解锁提交
- 成功提交后：弹窗关闭、列表刷新、Snackbar 提示、Drawing 主记录不变
- 内部审批人 Todo 即时出现
- 上传进度条 + 防重复提交
- 状态冲突（409）提示

**不测（本期）**：
- 内部审批操作本身（REQ-007A-pc）
- DC 外部审批操作（REQ-007B-pc）
- APP 端流程
- AI 识别引擎准确率（AI 团队负责）

### 2.2 Mock AI 服务策略

| Mock 场景 | 触发方式 | 说明 |
|---------|---------|------|
| AI 识别 DONE（全部识别） | 特定测试文件 `ai_success.pdf` | 返回 3 页，全部有 Drawing Code/Name |
| AI 识别 DONE（部分未识别） | `ai_partial.pdf` | 返回 2 页，第 2 页 drawingCode/drawingName 均为 null |
| AI 识别 FAILED | `ai_fail.pdf` | AI 服务立即返回 FAILED |
| AI 识别超时 | `ai_timeout.pdf` | AI 任务 60s 后仍为 PROCESSING，前端超时进入 failed |

### 2.3 测试金字塔

| 层级 | 占比 | 说明 |
|-----|-----|------|
| 单元测试 | 50% | `validateFile`、`submitDisabled` computed、`phase` 状态机转换、`reUpload()` 清理逻辑 |
| 集成测试 | 35% | 接口联调（文件上传/AI 轮询/版本提交/审批人列表）、409 状态冲突 |
| E2E 测试 | 15% | 正常路径提交 + Proceed Anyway 路径提交 |

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 优先级 |
|--------|---------|-------|
| SC-003F-001 | 正常路径：AI 识别 DONE，选择审批人，提交成功 | P1 |
| SC-003F-002 | 降级路径：AI 识别 FAILED，点击 Proceed Anyway，提交成功 | P1 |
| SC-003F-003 | 降级路径：AI 识别 FAILED，点击 Re-upload，重新识别成功后提交 | P1 |
| SC-003F-004 | AI 识别有部分未识别页，展示 ⚠ 提示，仍可提交 | P2 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-003F-E01 | 上传非 PDF 文件 | AC-003F-008 |
| SC-003F-E02 | 上传超过 50MB 的 PDF | AC-003F-009 |
| SC-003F-E03 | 上传过程中网络中断 | — |
| SC-003F-E04 | 并发场景：提交时图纸已被他人发起审批（409） | AC-003F-011 |
| SC-003F-E05 | AI 识别超时（60s 无响应） | AC-003F-005 |

### 3.3 权限场景

| 场景 ID | 角色 | 验证点 |
|--------|------|-------|
| SC-003F-P01 | 设计人员，状态 ACTIVE | [Upload New Version] 可点 |
| SC-003F-P02 | 设计人员，状态 PENDING_INTERNAL | [Upload New Version] 置灰 |
| SC-003F-P03 | 设计人员，状态 PENDING_EXTERNAL | [Upload New Version] 置灰 |
| SC-003F-P04 | 内部审批人 | [Upload New Version] 不可见 |
| SC-003F-P05 | 项目管理员 | [Upload New Version] 不可见 |

### 3.4 弹窗触发状态场景

| 场景 ID | approvalStatus | 期望 |
|--------|---------------|------|
| SC-003F-ST01 | `ACTIVE` | 按钮可点，弹窗正常打开 |
| SC-003F-ST02 | `INTERNAL_REJECTED` | 按钮可点，弹窗正常打开 |
| SC-003F-ST03 | `EXTERNAL_REJECTED` | 按钮可点，弹窗正常打开 |
| SC-003F-ST04 | `PENDING_INTERNAL` | 按钮置灰，弹窗不打开 |
| SC-003F-ST05 | `PENDING_EXTERNAL` | 按钮置灰，弹窗不打开 |

---

## 4. 测试用例

### TC 组 1：弹窗触发与只读信息展示

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-001 | AC-003F-011 | 设计人员登录，图纸状态 ACTIVE | 查看该行 Actions 列 | [Upload New Version] 按钮可见且**可点击** |
| TC-003F-002 | AC-003F-011 | 设计人员登录，图纸状态 PENDING_INTERNAL | 查看该行 Actions 列 | [Upload New Version] 按钮**置灰不可点**；鼠标悬停显示 Tooltip |
| TC-003F-003 | AC-003F-011 | 设计人员登录，图纸状态 PENDING_EXTERNAL | 查看该行 Actions 列 | [Upload New Version] 按钮**置灰不可点** |
| TC-003F-004 | AC-003F-012 | 设计人员点击 INTERNAL_REJECTED 图纸的 [Upload New Version] | 弹窗打开后查看内容 | 弹窗标题为 `"Upload New Version"`；副标题显示 `{drawingCode} — {drawingName}` |
| TC-003F-005 | AC-003F-012 | 同上 | 查看只读字段 | Current Version（Vn）、Category、Description 均以只读灰色样式展示，无编辑入口；Description 为空时显示 `—` |
| TC-003F-006 | AC-003F-012 | 同上 | 检查弹窗可编辑字段 | 可编辑字段仅为：Drawing File 上传区、Version Note 文本框、Internal Approver 下拉 |

### TC 组 2：文件选择校验

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-007 | AC-003F-008 | 弹窗已打开 | 尝试选择 `.dwg` 格式文件 | 文件被拒绝；显示提示 `"Only PDF format is supported for version upload."`；文件上传区清空 |
| TC-003F-008 | AC-003F-008 | 弹窗已打开 | 尝试选择 `.xlsx` 格式文件 | 同上 |
| TC-003F-009 | AC-003F-009 | 弹窗已打开 | 尝试选择大小为 60MB 的 PDF 文件 | 文件被拒绝；显示提示文件过大（最大 50MB）；文件上传区清空 |
| TC-003F-010 | AC-003F-009 | 弹窗已打开 | 选择大小恰好为 50MB 的 PDF 文件 | 文件**被接受**，开始上传（边界值测试） |
| TC-003F-011 | AC-003F-008 | 弹窗已打开 | 选择大小为 1MB 的合法 PDF 文件 | 文件被接受，开始上传并触发 AI 识别 |

### TC 组 3：AI 识别中状态

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-012 | AC-003F-002 | 弹窗已打开，选择合法 PDF | 文件上传完成后（上传进度 100%），查看弹窗状态 | 弹窗显示 Loading 动画 + 文案 `"Analysing drawing pages…"`；[Submit] 按钮**置灰不可点** |
| TC-003F-013 | AC-003F-002 | AI 识别进行中 | 尝试点击 [Submit] | 按钮无响应（disabled 状态） |

### TC 组 4：AI 识别 DONE — 识别结果列表

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-014 | AC-003F-003 | 使用 `ai_success.pdf`（3 页，全部识别） | 等待 AI 识别完成 | Loading 消失；显示识别结果列表，顶部文案 `"3 pages detected. Please review the drawing information below before submitting."`；列表含 3 行，每行显示页码、缩略图、Drawing Code、Drawing Name |
| TC-003F-015 | AC-003F-003 | 同上 | 查看 [Submit] 按钮状态 | [Submit] 按钮**解锁可点**（前提：Internal Approver 已选） |
| TC-003F-016 | AC-003F-003 | 识别结果列表已展示 | 点击某页缩略图 | 弹出大图预览 |
| TC-003F-017 | AC-003F-004 | 使用 `ai_partial.pdf`（2 页，第 2 页无法识别） | 等待 AI 识别完成 | 第 2 页的 Drawing Code 和 Drawing Name 单元格显示 `⚠ —`（橙色标记）；顶部追加橙色提示 `"Some pages could not be fully analysed. Please verify the drawing information below."` |
| TC-003F-018 | AC-003F-004 | 同上 | 检查 [Submit] 状态 | [Submit] 按钮**仍可点击**（部分识别失败不阻止提交） |

### TC 组 5：AI 识别 FAILED — 降级处理

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-019 | AC-003F-005 | 使用 `ai_fail.pdf` | 等待 AI 识别结果 | 识别结果区域显示橙色提示横幅 `"Unable to analyse the drawing automatically. Please re-upload the file or proceed to submit."`；[Re-upload] 和 [Proceed Anyway] 两个按钮可见；[Submit] 按钮**保持置灰** |
| TC-003F-020 | AC-003F-005 | AI 识别超时（使用 `ai_timeout.pdf`，等待 60s） | 等待超时 | 前端超时进入失败状态，展示与 AI 识别 FAILED 相同的降级横幅 |
| TC-003F-021 | AC-003F-006 | AI 识别 FAILED 降级横幅已展示 | 点击 [Re-upload] | 当前文件清空（文件上传区重置）；降级横幅消失；弹窗回到初始等待上传文件状态 |
| TC-003F-022 | AC-003F-006 | 点击 Re-upload 后重新选择合法 PDF | 使用 `ai_success.pdf` 重新上传 | 触发新一轮 AI 识别；Loading 重新出现；识别完成后展示正常结果列表 |
| TC-003F-023 | AC-003F-007 | AI 识别 FAILED 降级横幅已展示 | 点击 [Proceed Anyway] | [Submit] 按钮**解锁**（前提：Internal Approver 已选）；降级横幅保留但状态转为"继续" |
| TC-003F-024 | AC-003F-007 | 已点击 Proceed Anyway，已选 Internal Approver | 点击 [Submit] | 提交正常执行，流程与正常路径一致（见 TC 组 7） |

### TC 组 6：表单校验

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-025 | AC-003F-001 | AI 识别完成（DONE），未选择 Internal Approver | 点击 [Submit] | 提交被阻止；Internal Approver 字段显示错误提示 `"Please select an internal approver."` |
| TC-003F-026 | AC-003F-001 | AI 识别完成（DONE），Internal Approver 已选，Version Note 为空 | 点击 [Submit] | 提交**正常执行**（Version Note 为选填） |

### TC 组 7：提交成功路径

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-027 | AC-003F-001 | AI 识别 DONE，Internal Approver 已选，点击 [Submit] | 等待提交完成 | 弹窗关闭；图纸列表刷新；目标图纸状态变为 `Pending Internal`（橙色标签）；顶部 Snackbar 显示 `"New version uploaded. Pending internal approval."` |
| TC-003F-028 | AC-003F-001 | 同上，提交成功后 | 在图纸列表查看该行 | Current Version 列自动加 1（例：V2 → V3）；Drawing 主记录的 Drawing Code、Drawing Name、Category、Description **保持不变** |
| TC-003F-029 | AC-003F-013 | 提交成功，切换至指定内部审批人账号 | 查看 Todo 列表 | Todo 列表立即出现新任务 `"Internal Approval Required"`，包含正确的图纸编号、名称和新版本号 |

### TC 组 8：上传进度条与防重复提交

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-030 | AC-003F-010 | AI 识别完成，Internal Approver 已选 | 点击 [Submit]，上传期间立即查看弹窗 | 弹窗内展示进度条（0% → 100%）；[Submit] 按钮**禁用**（防止重复点击） |
| TC-003F-031 | AC-003F-010 | 上传期间 | 尝试再次点击 [Submit] | 按钮无响应（disabled） |
| TC-003F-032 | AC-003F-010 | 上传完成（成功） | 查看进度条 | 进度条消失；弹窗随即关闭 |

### TC 组 9：异常与错误处理

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-003F-033 | — | AI 识别完成，Internal Approver 已选 | 提交过程中模拟网络中断 | Toast 提示 `"Upload failed. Please try again."`；弹窗保留当前状态；用户可重新点击 [Submit] |
| TC-003F-034 | AC-003F-011 | 另一测试账号同时对同一图纸发起审批，使状态变为 PENDING_INTERNAL | 此时当前用户点击 [Submit] | 返回 409；提示 `"This drawing is now under review. Please refresh and try again."`；弹窗保留 |

---

## 5. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-003F-001 | 弹窗宽度 | 600px |
| UI-003F-002 | 弹窗副标题位置 | 标题 `"Upload New Version"` 下方，格式 `{drawingCode} — {drawingName}` |
| UI-003F-003 | 只读字段样式 | 灰色文本，无输入框边框，视觉上明显不可编辑 |
| UI-003F-004 | AI 识别 Loading | Loading 图标居中 + 文案 "Analysing drawing pages…"，视觉清晰 |
| UI-003F-005 | 识别结果列表高度 | 超过 240px 时出现纵向滚动条 |
| UI-003F-006 | 缩略图尺寸 | 约 60×60px，对象适应（object-fit: contain） |
| UI-003F-007 | ⚠ 未识别标记颜色 | 橙色，与正常识别结果有明显视觉区分 |
| UI-003F-008 | 降级横幅颜色 | 橙色 warning 样式 |
| UI-003F-009 | [Submit] 置灰样式 | 视觉上明显不可点；cursor: not-allowed |
| UI-003F-010 | Snackbar 提示 | 提交成功后出现在页面顶部或底部，3s 后自动消失 |

---

## 6. 非功能测试

| 类型 | 测试点 | 期望 |
|------|-------|------|
| 性能 | 识别结果列表渲染（100 页以内） | ≤ 1s |
| 性能 | `POST /version/file` 响应（不含传输） | ≤ 3s |
| 安全 | 非设计人员尝试调用 `POST /drawing/{id}/version/file` | 返回 403 |
| 安全 | 上传 `.exe` 改后缀为 `.pdf` | 后端 MIME type 校验拦截 |
| 可访问性 | 弹窗 Tab 键导航 | 所有输入项可通过 Tab 到达；Esc 关闭弹窗 |
| 国际化 | 切换中/英语言 | 弹窗标题、字段标签、提示文案正确切换 |

---

## 7. 集成验收检查清单（上线前）

- [ ] `POST /drawing/{drawingId}/version/file` 返回 `versionFileId` 和 `aiJobId`
- [ ] `GET /ai/job/{aiJobId}/status` 正确返回 PROCESSING / DONE / FAILED 及 `pages` 列表
- [ ] `thumbnailUrl` 为有效 URL，有效期 ≥ 30min
- [ ] `POST /drawing/{drawingId}/version` 在有进行中版本时返回 409
- [ ] 提交成功后图纸状态变为 `PENDING_INTERNAL`，版本号正确递增
- [ ] Drawing 主记录（drawingCode / drawingName / category / description）在提交后**不变**
- [ ] 指定内部审批人 Todo 在提交后即时出现（延迟 ≤ 5s）
- [ ] `versionFileId` 已使用后再次提交同一 ID 返回 400 INVALID_FILE_ID

---

## 8. 测试数据

| 数据 | 说明 |
|------|-----|
| 设计人员账号 | `designer_01` |
| 内部审批人账号 | `approver_01`（具有 `drawing:approve` 权限）|
| 图纸 ARCH-001 | 状态 ACTIVE，currentVersion=V2，category="Architecture"，description="Foundation Plan" |
| 图纸 ARCH-002 | 状态 INTERNAL_REJECTED，currentVersion=V1 |
| 图纸 ARCH-003 | 状态 PENDING_INTERNAL（用于置灰测试） |
| 图纸 ARCH-004 | 状态 PENDING_EXTERNAL（用于置灰测试） |
| `ai_success.pdf` | 3 页，全部含 Drawing Code / Drawing Name |
| `ai_partial.pdf` | 2 页，第 2 页 Drawing Code / Name 均为 null |
| `ai_fail.pdf` | 触发 Mock AI 立即返回 FAILED |
| `ai_timeout.pdf` | 触发 Mock AI 60s 内保持 PROCESSING |
| `large_60mb.pdf` | 60MB PDF，用于大小校验测试 |
| `test.dwg` | DWG 格式文件，用于类型校验测试 |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.5.0 | 2026-05-26 | agent | 基于 REQ-003F-pc@0.5.0 首次生成。涵盖 AC-003F-001~013 全部 13 个 AC；TC 组 1（弹窗触发与只读字段）、TC 组 2（文件校验）、TC 组 3（AI 识别 Loading）、TC 组 4（AI DONE 识别结果含部分未识别）、TC 组 5（AI FAILED 降级：Re-upload/Proceed Anyway/超时）、TC 组 6（表单校验）、TC 组 7（提交成功路径含主记录不变验证）、TC 组 8（进度条防重复提交）、TC 组 9（异常 409 并发冲突）；Mock AI 服务策略说明；集成验收清单 8 条 |
