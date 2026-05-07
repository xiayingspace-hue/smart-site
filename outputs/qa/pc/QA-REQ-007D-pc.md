---
doc_type: qa_spec
req_id: REQ-007D-pc
req_title: "PC 端 — DC 配置页面"
version: 0.1.0
status: draft
generated_from: REQ-007D-pc@0.1.0
generated_at: 2026-05-07
owner: ""
---

# QA 测试说明：PC 端 — DC 配置页面

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - DC 配置是 REQ-007A-pc 和 REQ-007B-pc 的前置依赖，集成测试需优先验证本模块。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007D-pc @ v0.1.0 |
| 依赖需求 | REQ-007-shared |
| 覆盖 Story | US-007D-001 |
| 覆盖 AC | AC-007D-001 ～ AC-007D-010 |
| 测试平台 | PC（Chrome 100+ / Edge 100+ / Safari 15+，1280px+） |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证 DC 配置页面的访问权限控制、DC 列表展示、添加 DC 弹窗（Available Users 过滤、搜索、连续添加）、移除 DC（二次确认、移除最后一个 DC 后的警告横幅）及集成验收（内部审批通过后配置 DC 收到任务）等功能的正确性。

---

## 2. 测试策略

### 2.1 测试范围

**必测**：
- 页面访问权限（`drawing:dc-config`）
- DC 列表展示（字段、列）
- 添加 DC—— Available Users 仅展示有权限且未配置的用户
- 添加 DC——成功路径，弹窗保持开启支持连续添加
- 搜索框实时过滤
- 移除 DC——二次确认文案含姓名
- 移除 DC——成功路径
- 移除最后一个 DC——警告横幅（不阻止移除）
- 集成验收：内部审批通过后已配置 DC 收到 External Approval Required Todo

**不测（本期）**：
- DC 执行外部审批操作（REQ-007B-pc）
- APP 端（无 DC 配置功能）
- 按图纸分类配置 DC（当前方案为项目级，暂不支持）

### 2.2 测试金字塔

| 层级 | 占比 | 说明 |
|-----|-----|------|
| 单元测试 | 50% | Available Users 过滤逻辑、搜索过滤、移除后空状态判断 |
| 集成测试 | 35% | 前端 → 后端 DC 配置接口、权限校验、内部审批通过后 Todo 推送 |
| E2E 测试 | 15% | 完整添加 + 移除 DC 流程，含警告横幅 |

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-007D-001 | 业务人员进入 DC Configuration 页，查看 DC 列表并添加 DC | US-007D-001 | P1 |
| SC-007D-002 | 业务人员移除 DC，移除最后一个后出现警告横幅 | US-007D-001 | P1 |
| SC-007D-003 | 内部审批通过后已配置 DC 收到外部审批 Todo（集成验收） | US-007D-001 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-007D-E01 | 添加 DC 时目标用户无 `drawing:external-approval` 权限，接口返回错误 | — |
| SC-007D-E02 | 添加 DC 接口调用失败，弹窗保留 | — |
| SC-007D-E03 | 移除 DC 接口调用失败，Toast 报错 | — |
| SC-007D-E04 | Available Users 列表为空（所有可用用户均已配置） | — |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-007D-P01 | 无 `drawing:dc-config` 权限的用户，侧边栏不显示 DC Configuration，直接访问 URL 跳转 403 |
| SC-007D-P02 | DC 自身无法访问 DC Configuration 页面 |
| SC-007D-P03 | 普通用户调用 DC 配置接口返回 403 |

---

## 4. 测试用例

### TC 组 1：页面访问权限

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007D-001 | AC-007D-001 | 用户不具备 `drawing:dc-config` 权限 | 查看侧边栏 Settings 菜单 | DC Configuration 菜单项不可见 |
| TC-007D-002 | AC-007D-001 | 同上 | 直接在浏览器地址栏输入 DC Configuration 页面 URL | 跳转至 403 错误页面 |
| TC-007D-003 | — | 用户具备 `drawing:dc-config` 权限（业务人员/项目管理员） | 点击侧边栏 Settings → DC Configuration | 成功进入 DC Configuration 页面 |

### TC 组 2：DC 列表展示

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007D-004 | AC-007D-002 | 项目已配置 2 个 DC | 业务人员进入 DC Configuration 页 | 列表显示 2 条记录，每条包含：Name（DC 用户姓名）、Configured By（配置操作人）、Configured At（格式 YYYY-MM-DD）、[Remove] 按钮 |
| TC-007D-005 | AC-007D-008 | 项目当前无 DC 配置 | 业务人员进入 DC Configuration 页 | 显示空状态警告横幅：包含"⚠️ No DC configured. Internal approvals cannot proceed to external approval."语义文案 |
| TC-007D-006 | — | 页面正常展示 | 检查页面右上角 | 显示 [+ Add DC] 按钮 |
| TC-007D-007 | — | 页面正常展示 | 检查页面底部提示 | 显示"At least one DC is required for the drawing approval process."提示信息 |

### TC 组 3：添加 DC

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007D-008 | — | 业务人员在 DC Configuration 页 | 点击 [+ Add DC] | 弹出 Add Document Controller 弹窗，包含 Search 输入框和 Available Users 列表 |
| TC-007D-009 | AC-007D-003 | 弹窗已打开；用户 A 有 `drawing:external-approval` 权限且未配置；用户 B 无该权限；用户 C 已配置为 DC | 查看 Available Users 列表 | 仅显示用户 A；用户 B 和 C 不出现 |
| TC-007D-010 | AC-007D-004 | 弹窗已打开，Available Users 列表有用户 A | 点击用户 A 的 [Add] | 接口调用成功：用户 A 从弹窗 Available Users 列表消失；主列表新增用户 A（含 Configured By 为当前操作人、Configured At 为今日） |
| TC-007D-011 | AC-007D-005 | 弹窗已打开，Available Users 有多个用户 | 连续点击多个用户的 [Add] | 每次添加成功后弹窗保持打开状态；可继续添加下一个用户 |
| TC-007D-012 | AC-007D-009 | 弹窗已打开，Available Users 列表有 3 个用户：王磊、赵强、孙丽 | 在 Search 框输入"王" | 列表实时过滤，仅显示"王磊" |
| TC-007D-013 | AC-007D-009 | Search 框已有内容 | 清空 Search 框 | Available Users 列表恢复显示所有可用用户 |
| TC-007D-014 | — | 弹窗已打开，所有有权限用户均已配置 | 查看 Available Users 列表 | 显示"All eligible users have been configured as DC."空状态文案 |
| TC-007D-015 | — | 添加 DC 接口返回失败（500） | 点击某用户 [Add] | Toast 提示错误；弹窗保持打开；该用户仍在 Available Users 列表中 |
| TC-007D-016 | — | 目标用户无 `drawing:external-approval` 权限（后端校验失败） | 点击该用户 [Add] | 接口返回错误码；Toast 提示用户无 DC 权限 |

### TC 组 4：移除 DC

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007D-017 | AC-007D-006 | DC 列表有用户"陈小明" | 点击"陈小明"的 [Remove] | 弹出二次确认对话框，文案包含"Remove 陈小明 from DC list?" |
| TC-007D-018 | AC-007D-006 | 同上，确认对话框已弹出 | 检查对话框文案 | 包含"They will no longer receive external approval tasks."语义 |
| TC-007D-019 | AC-007D-007 | 移除确认对话框已弹出 | 点击 [Confirm] | 接口调用成功：主列表中"陈小明"消失 |
| TC-007D-020 | AC-007D-007 | 移除确认对话框已弹出 | 点击 [Cancel] | 对话框关闭，主列表无变化 |
| TC-007D-021 | AC-007D-008 | 项目当前只有 1 个 DC | 完成移除该 DC 的操作（点击 [Remove] → [Confirm]） | 移除成功；主列表为空；页面显示警告横幅"⚠️ No DC configured. Internal approvals cannot proceed to external approval."；移除操作未被阻止 |
| TC-007D-022 | — | 移除接口返回失败（500） | 点击 [Confirm] | Toast 提示错误；主列表无变化 |
| TC-007D-023 | — | DC 有正在处理中的外部审批 Todo | 移除该 DC | 移除成功后，该 DC 已存在的外部审批 Todo 不受影响（不会自动关闭） |

### TC 组 5：集成验收

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007D-024 | AC-007D-010 | 项目已配置 DC-A 和 DC-B | 内部审批人完成内部审批通过操作 | DC-A 和 DC-B 的 Todo 列表均出现"External Approval Required"任务 |
| TC-007D-025 | AC-007D-010 | 项目**未配置**任何 DC | 内部审批人完成内部审批通过操作 | 无 DC 收到任务（警告横幅已提示）；系统不报错，内部审批通过流程正常完成 |

### TC 组 6：幂等与唯一性

| TC ID | 关联 AC | 前置条件 | 步骤 | 预期结果 |
|-------|--------|---------|------|---------|
| TC-007D-026 | — | 用户 A 已配置为项目 DC | 尝试通过 API 重复添加用户 A（绕过前端） | 接口返回业务错误（唯一约束冲突）；主列表不新增重复记录 |

---

## 5. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-007D-001 | 页面标题 | "⚙️ DC Configuration" |
| UI-007D-002 | 主列表列顺序 | Name → Configured By → Configured At → Action |
| UI-007D-003 | 添加弹窗 Search 框 | 实时过滤（无需按回车），延迟 ≤ 300ms |
| UI-007D-004 | 警告横幅样式 | ⚠️ 图标 + 橙色/黄色警告色背景，文案包含"No DC configured" |
| UI-007D-005 | [Remove] 按钮样式 | 危险操作样式（红色或红色边框文字） |
| UI-007D-006 | Configured At 格式 | YYYY-MM-DD |

---

## 6. 非功能测试

| 类型 | 测试点 | 期望 |
|------|-------|------|
| 性能 | DC 列表加载 | ≤ 1s |
| 性能 | 添加/移除接口响应 P95 | ≤ 1s |
| 安全 | JWT + `drawing:dc-config` 权限校验 | 缺一返回 401/403 |
| 审计 | 添加/移除 DC 操作日志 | 记录操作人、时间、目标用户 ID |
| 可访问性 | 弹窗 Search 框及列表 Tab 键导航 | 可依次聚焦 |
| 国际化 | 切换中/英语言 | 页面文案、警告横幅、弹窗提示正确切换 |

---

## 7. 集成验收检查清单（上线前，必须早于或同步于 REQ-007A/B-pc）

- [ ] `drawing:dc-config` 权限已绑定到业务人员/项目管理员角色
- [ ] `drawing:external-approval` 权限已绑定到 DC 角色
- [ ] ProjectDcConfig 表已创建，`(projectId, dcUserId)` 联合唯一索引已添加
- [ ] 各项目 DC 人员初始化配置已完成
- [ ] 内部审批通过后正确向所有已配置 DC 推送外部审批 Todo
- [ ] 无 DC 时不影响内部审批通过本身，仅无人收到 Todo
- [ ] 添加/移除 DC 操作审计日志落库

---

## 8. 测试数据

| 数据 | 示例值 |
|------|-------|
| 业务人员/管理员账号 | `admin_01`（具备 `drawing:dc-config` 权限） |
| DC 候选用户 A | `dc_candidate_01`（有 `drawing:external-approval` 权限，未配置） |
| DC 候选用户 B | `dc_candidate_02`（无 `drawing:external-approval` 权限） |
| 已配置 DC | `dc_user_01`、`dc_user_02` |
| 无权限普通用户 | `normal_user` |
| 测试项目 | PROJECT-001（仅用于 DC 配置场景隔离） |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 REQ-007D-pc@0.1.0 生成初稿 |
