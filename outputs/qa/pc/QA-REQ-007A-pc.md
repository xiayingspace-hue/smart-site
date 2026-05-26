---
doc_type: qa_spec
req_id: REQ-007A-pc
version: 0.5.0
status: draft
generated_from: REQ-007A-pc.md@0.5.0
generated_at: 2026-05-25
owner: ""
---

# QA 测试说明 — PC 端内部审批 Todo、详情查看与原文件下载

> **来源需求**: [REQ-007A-pc.md](../../../requirements/pc/REQ-007A-pc.md) @ v0.5.0
> **依赖文档**: [FRONTEND-REQ-007A-pc.md](../../frontend/pc/FRONTEND-REQ-007A-pc.md) / [BACKEND-REQ-007A.md](../../backend/BACKEND-REQ-007A.md)
> **产品**: SMART SITE SYSTEM | **平台**: PC 管理端
> **生成日期**: 2026-05-25

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007A-pc @ v0.5.0 |
| 覆盖 Story | US-007A-001 / 002 / 003 / 004 / 005 / 006 |
| 覆盖 AC | AC-007A-001 ~ AC-007A-023 |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证 PC 端内部审批人能够：
1. 在 Todo 列表中**仅看到自己名下**的内部审批待办，且卡片信息展示正确
2. 点击 [Detail] 打开侧滑弹框，展示与图纸上传时完全一致的全量信息
3. 通过 [Download Original File] 按钮将原始图纸文件下载到本地，且文件命名规则正确
4. 在详情弹框内完成**通过（Approve）**或**驳回（Reject）**操作，流程和副作用符合预期
5. 无 DC 配置时 Approve 操作被正确阻断；后端兜底校验有效
6. 非指派审批人无法越权下载文件或执行审批操作

---

## 2. 测试策略

### 2.1 测试金字塔

| 层级 | 占比 | 由谁写 | 框架 |
|-----|-----|-------|------|
| 单元测试 | 60% | 前端/后端开发 | Jest + Vue Test Utils / JUnit |
| 集成测试 | 25% | 开发 + QA | Jest + MSW / Testcontainers |
| 契约测试 | 5% | QA | schemathesis |
| E2E 测试 | 10% | QA | Playwright |

### 2.2 测试范围

**必测**：
- Todo 卡片展示（图标、标题、字段、按钮）
- 待办列表 assigneeId 过滤
- 详情侧滑弹框加载、字段展示、空值处理
- 文件名链接（inline 打开）及辅助文案
- 原文件下载（命名规则、loading、disabled 态、越权 403）
- Approve 流程（DC 预检查、确认对话框、loading、成功/失败路径）
- Reject 流程（Comment 必填校验、成功路径、旧版本不受影响）
- 并发操作（已处理任务的防御）
- 异常恢复（加载失败 Retry、接口失败可重试）

**不测（本期）**：
- DC 外部审批 Todo（由 REQ-007B-pc 覆盖）
- APP 端 Todo 文案变更
- 在线图纸预览（Markup 范畴）
- 已处理历史任务查看（由 REQ-007C-pc 覆盖）

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-001 | 内部审批人进入 Todo 列表，看到正确格式的内部审批卡片 | US-007A-001 | P0 |
| SC-002 | 列表仅展示指派给当前用户的待办记录 | US-007A-002 | P0 |
| SC-003 | 无待审批记录时展示空状态 | US-007A-002 | P1 |
| SC-004 | 点击 [Detail] 打开详情侧滑弹框，字段与上传时一致 | US-007A-003 | P0 |
| SC-005 | 选填字段（Description / Version Note）为空时显示 — | US-007A-003 | P1 |
| SC-006 | 文件名链接点击后在新标签页 inline 打开 | US-007A-003 | P1 |
| SC-007 | 文件名下方显示引导提示文案 | US-007A-003 | P1 |
| SC-008 | 点击 [Download Original File] 成功下载，文件命名正确 | US-007A-004 | P0 |
| SC-009 | 下载期间按钮 loading，完成后恢复 | US-007A-004 | P1 |
| SC-010 | 有 DC 配置时点击 Approve，弹出确认对话框并成功审批 | US-007A-005 | P0 |
| SC-011 | 无 DC 配置时点击 Approve，弹出无 DC 警告弹窗，不进入确认对话框 | US-007A-005 | P0 |
| SC-012 | Reject 填写 Comment 后成功驳回，设计人员收到通知 | US-007A-006 | P0 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-E01 | 详情弹框加载失败（5xx）→ 显示错误提示 + Retry | AC-007A-013 |
| SC-E02 | 文件下载失败（URL 过期）→ Toast 报错，按钮恢复 | AC-007A-007 |
| SC-E03 | fileUrl 为空 → 文件名显示 —，下载按钮 disabled + Tooltip | AC-007A-009 |
| SC-E04 | Approve 接口失败（5xx）→ loading 恢复，对话框保留，可重试 | AC-007A-019 |
| SC-E05 | Approve 后端兜底：无 DC（错误码 1003007012）→ Toast 提示 | AC-007A-020 |
| SC-E06 | Reject Comment 为空 → 前端阻止提交，字段标红 | AC-007A-021 |
| SC-E07 | 并发：Todo 已被处理 → 弹框内显示已完成，按钮隐藏 | 并发防御 |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-P01 | 非指派审批人 B 调用 /view（assigneeId=A）→ 403 |
| SC-P02 | 非指派审批人 B 调用 /download（assigneeId=A）→ 403 |
| SC-P03 | 无 drawing:approve 权限用户调用 POST /drawing/approve → 403 |
| SC-P04 | 未登录用户调用任何接口 → 401 |

### 3.4 状态转换场景

**合法转换**:

| 场景 ID | from | action | to | 测试要点 |
|--------|------|--------|----|---------|
| SC-T01 | PENDING_INTERNAL | APPROVE（有 DC） | INTERNAL_APPROVED | Todo 关闭；DC 收到新 Todo；版本不生效 |
| SC-T02 | PENDING_INTERNAL | REJECT（comment 非空） | INTERNAL_REJECTED | Todo 关闭；设计人员收到通知；旧 ACTIVE 版本保持 |

**非法转换**:

| 场景 ID | from | 尝试操作 | 期望 |
|--------|------|---------|-----|
| SC-T-F01 | INTERNAL_APPROVED | APPROVE | 409 TASK_ALREADY_COMPLETED |
| SC-T-F02 | INTERNAL_APPROVED | REJECT | 409 TASK_ALREADY_COMPLETED |
| SC-T-F03 | PENDING_INTERNAL | APPROVE（非 assignee） | 403 |

---

## 4. 测试用例（TC）详述

### TC-007A-001-01 — Todo 卡片图标与标题



### TC-007A-002-01 — 待办列表仅展示自己名下记录



### TC-007A-003-01 — 空状态展示



### TC-007A-004-01 — 点击 [Detail] 打开详情侧滑弹框



### TC-007A-005-01 — 详情字段与上传时一致



### TC-007A-006-01 — 选填字段为空时显示占位符



### TC-007A-007-01 — 成功下载原始文件，文件命名规则正确



### TC-007A-008-01 — 下载期间按钮 loading 状态



### TC-007A-009-01 — fileUrl 为空时下载按钮 disabled



### TC-007A-010-01 — 非指派审批人无法下载文件（越权测试）



### TC-007A-011-01 — 文件名链接在新标签页 inline 打开



### TC-007A-012-01 — 文件名下方引导提示文案



### TC-007A-013-01 — 详情弹框加载失败时显示 Retry



### TC-007A-014-01 — 无 DC 配置时 Approve 弹出警告弹窗



### TC-007A-015-01 — 有 DC 配置时 Approve 弹出确认对话框



### TC-007A-016-01 — Approve 成功路径



### TC-007A-017-01 — 内部审批通过后版本不立即生效



### TC-007A-018-01 — Approve 后 DC 收到外部审批 Todo



### TC-007A-019-01 — Approve 接口失败时 loading 恢复，可重试



### TC-007A-020-01 — 后端兜底无 DC 配置错误码



### TC-007A-021-01 — Reject Comment 必填校验



### TC-007A-022-01 — Reject 成功路径



### TC-007A-023-01 — Reject 后旧版本保持有效



---

## 5. 边界值与等价类

### 5.1 Comment 字段（Reject 对话框）

| 等价类 | 测试值 | 期望 |
|-------|-------|-----|
| 有效 | 1 个字符 | 接受，提交成功 |
| 有效 | 500 个字符 | 接受，提交成功 |
| 无效 | 空字符串 | 拒绝，标红提示 |
| 无效 | 501 个字符 | 拒绝，长度超限提示 |
| 特殊 | 含中文 + 英文 + 符号 | 接受，正确存储和展示 |
| 安全 | SQL 注入字符串 | 安全转义，正确存储 |
| 安全 | XSS 字符串 | 安全转义，展示时不执行 |

### 5.2 预签名 URL 有效期

| 等价类 | 场景 | 期望 |
|-------|------|-----|
| URL 有效 | 5 分钟内点击下载 | 下载成功 |
| URL 过期 | 5 分钟后点击下载 | 下载失败，Toast 报错 |

---

## 6. 接口测试

### 6.1 

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-001-01 | 正常请求，有待办 | 有效 JWT + type=DRAWING_INTERNAL_APPROVAL | 200，仅返回 assigneeId=当前用户 |
| TC-API-001-02 | 无待办 | 无匹配待办 | 200，data=[] |
| TC-API-001-03 | 未登录 | 无 token | 401 |

### 6.2 

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-002-01 | 正常请求 | 有效 JWT + 本人 assignee | 200，字段完整 |
| TC-API-002-02 | 非 assignee | 非 assignee JWT | 403 |
| TC-API-002-03 | versionId 不存在 | 无效 id | 404 |

### 6.3 

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003-01 | 正常请求 | 本人 assignee + fileUrl 存在 | 200，fileName 符合命名规则 |
| TC-API-003-02 | 非 assignee 越权 | 非 assignee | 403 |
| TC-API-003-03 | fileUrl 为空 | fileUrl=null 版本 | 404 |
| TC-API-003-04 | 超出限流 | 超限请求 | 429 |

### 6.4 

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-006-01 | Approve 成功（有 DC） | action=APPROVE + 有 DC | 200，INTERNAL_APPROVED |
| TC-API-006-02 | Approve 失败（无 DC） | action=APPROVE + 无 DC | 422，1003007012 |
| TC-API-006-03 | Reject 成功 | action=REJECT + comment 非空 | 200，INTERNAL_REJECTED |
| TC-API-006-04 | Reject 失败（comment 空） | comment=null | 400 |
| TC-API-006-05 | 非 assignee 越权 | 非 assignee JWT | 403 |
| TC-API-006-06 | 重复操作 | Todo 已 APPROVED | 409 |
| TC-API-006-07 | 未登录 | 无 token | 401 |

---

## 7. 性能测试

| 指标 | 目标 | 工具 |
|-----|------|-----|
| Todo 列表加载 | P95 ≤ 1500ms | k6 |
| 详情弹框首屏渲染 | P95 ≤ 800ms | k6 + Performance API |
| 审批接口响应 | P95 ≤ 2000ms | k6 |
| 下载预签名 URL 生成 | P95 ≤ 500ms | k6 |

---

## 8. 安全测试

| TC ID | 描述 | 期望 |
|-------|-----|-----|
| TC-SEC-001 | 非 assignee 直接调用 download 接口（IDOR） | 403 |
| TC-SEC-002 | 预签名 URL 过期后访问 | OSS 拒绝访问 |
| TC-SEC-003 | Reject comment 含 XSS 脚本，在其他界面展示 | 不执行脚本，内容转义 |
| TC-SEC-004 | 枚举 versionId 批量请求 download | 非 assignee 全部 403；超限 429 |

---

## 9. 兼容性测试

| 维度 | 测试矩阵 |
|-----|---------|
| 浏览器 | Chrome 100+、Edge 100+、Safari 15+ |
| 文件下载 | 三端浏览器均能正确触发 attachment 下载 |
| 文件 inline 打开 | PDF 三端可 inline 展示；DWG/DXF 可能显示错误页（已知限制） |

---

## 10. AC 覆盖矩阵

| AC ID | 描述简述 | 覆盖的 TC | 状态 |
|------|---------|----------|------|
| AC-007A-001 | 卡片图标/标题/字段/按钮 | TC-007A-001-01 | TODO |
| AC-007A-002 | 列表仅展示 assigneeId=当前用户 | TC-007A-002-01, TC-API-001-01 | TODO |
| AC-007A-003 | 无待办时显示空状态 | TC-007A-003-01 | TODO |
| AC-007A-004 | 点击 [Detail] 打开 Drawer | TC-007A-004-01 | TODO |
| AC-007A-005 | 弹框字段与上传时完全一致 | TC-007A-005-01, TC-API-002-01 | TODO |
| AC-007A-006 | 选填字段为空时显示 — | TC-007A-006-01 | TODO |
| AC-007A-007 | 下载文件命名规则 | TC-007A-007-01, TC-API-003-01 | TODO |
| AC-007A-008 | 下载期间 loading 防重复 | TC-007A-008-01 | TODO |
| AC-007A-009 | fileUrl 为空时 disabled + Tooltip | TC-007A-009-01, TC-API-003-03 | TODO |
| AC-007A-010 | 非 assignee 403 | TC-007A-010-01, TC-API-003-02 | TODO |
| AC-007A-011 | 文件名链接新标签页 inline 打开 | TC-007A-011-01 | TODO |
| AC-007A-012 | 文件名下方辅助提示文案 | TC-007A-012-01 | TODO |
| AC-007A-013 | 加载失败显示错误 + Retry | TC-007A-013-01 | TODO |
| AC-007A-014 | 无 DC 时弹出警告弹窗 | TC-007A-014-01 | TODO |
| AC-007A-015 | 有 DC 时弹出确认对话框 | TC-007A-015-01 | TODO |
| AC-007A-016 | Approve 成功 Toast + UI 更新 | TC-007A-016-01, TC-API-006-01 | TODO |
| AC-007A-017 | 通过后版本不立即生效 | TC-007A-017-01 | TODO |
| AC-007A-018 | DC 收到外部审批 Todo | TC-007A-018-01 | TODO |
| AC-007A-019 | Approve 失败 loading 恢复可重试 | TC-007A-019-01 | TODO |
| AC-007A-020 | 后端兜底 1003007012 | TC-007A-020-01, TC-API-006-02 | TODO |
| AC-007A-021 | Reject comment 必填校验 | TC-007A-021-01, TC-API-006-04 | TODO |
| AC-007A-022 | Reject 成功路径 | TC-007A-022-01, TC-API-006-03 | TODO |
| AC-007A-023 | Reject 旧版本保持 ACTIVE | TC-007A-023-01 | TODO |

---

## 11. 回归测试范围

| 受影响功能 | 影响原因 | 回归用例 |
|----------|---------|---------|
| Todo 列表（REQ-003B-pc 旧单级审批） | 内部审批卡片操作区变更 | 验证旧 Todo 卡片样式不受影响 |
| 图纸上传流程（REQ-003A-pc） | fileUrl 字段由上传流程写入 | 上传后 /detail 接口返回正确 fileUrl |
| 外部审批 Todo（REQ-007B-pc） | 内部审批通过后触发创建 | DC 用户收到正确的 External Approval Required Todo |
| 图纸状态标签（REQ-007-pc） | INTERNAL_APPROVED 状态展示 | 版本标签显示 Pending External |

---

## 12. 验收条件

- [ ] 所有 P0 AC 100% 覆盖
- [ ] P0 用例全部通过
- [ ] P1 用例 ≥ 95% 通过
- [ ] 越权下载（TC-007A-010-01）测试通过
- [ ] 性能指标在 staging 环境达标
- [ ] 安全扫描无 high 项
- [ ] 三端浏览器兼容性矩阵全绿
- [ ] AC 覆盖矩阵 §10 全部 ✅

---

## 13. 已知问题与遗留

| 问题 ID | 描述 | 严重度 | 处理方案 |
|--------|------|-------|---------|
| KNOWN-001 | DWG/DXF 文件无法在浏览器 inline 预览 | 低 | 已知限制，用户可改用下载按钮；OQ-003 跟进 |
| KNOWN-002 | 预签名 URL 有效期（5 分钟）待 PM 确认 | 中 | OQ-002 开放中，暂以 5 分钟测试 |

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.5.0 | 2026-05-25 | agent | 基于 REQ-007A-pc v0.5.0 全量重新生成；覆盖 AC-007A-001~023 全部 23 条；新增 TC-007A-007~023 覆盖 F-003/004/005/006/007；补充接口测试、安全测试、性能压测、回归范围 |
| 0.2.0 | 2026-05-07 | agent | 从 REQ-007A-pc@0.2.0 生成，覆盖 AC-007A-001~011 |
| 0.1.0 | 2026-05-07 | agent | 从 REQ-007A-pc@0.1.0 生成初稿 |
