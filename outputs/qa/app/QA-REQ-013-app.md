# QA-REQ-013-app.md

## SE"My Task"APP 端测试用例

> 对应需求：REQ-013
> 平台：APP 端
> 测试范围：My Task 列表筛选、状态标签、详情页字段（实际时间）、三态按钮操作流程

---

### 1. 测试用例

---

#### 模块一：My Task 列表页

| ID | 用例名称 | 前置条件 | 操作步骤 | 预期结果 |
|----|---------|---------|---------|---------|
| TC-013-APP-001 | Tab All 显示全部任务 | SE 有多个不同状态的工序任务 | 点击 All Tab | 列表显示所有分配给当前 SE 的工序任务 |
| TC-013-APP-002 | Tab Not Started 筛选 | SE 有 not_started 工序任务 | 点击 Not Started Tab | 仅显示 status=not_started 的任务 |
| TC-013-APP-003 | Tab In Progress 筛选 | SE 有 in_progress 工序任务 | 点击 In Progress Tab | 仅显示 status=in_progress 的任务 |
| TC-013-APP-004 | Tab Completed 筛选 | SE 有 completed 工序任务 | 点击 Completed Tab | 仅显示 status=completed 的任务 |
| TC-013-APP-005 | 状态标签颜色：Not Started | 工序任务 not_started | 查看卡片状态标签 | 显示灰色"Not Started" |
| TC-013-APP-006 | 状态标签颜色：In Progress | 工序任务 in_progress | 查看卡片状态标签 | 显示橙色"In Progress" |
| TC-013-APP-007 | 状态标签颜色：Completed | 工序任务 completed | 查看卡片状态标签 | 显示绿色"Completed" |

---

#### 模块二：详情页基本信息

| ID | 用例名称 | 前置条件 | 操作步骤 | 预期结果 |
|----|---------|---------|---------|---------|
| TC-013-APP-010 | 实际开始时间：未开始显示"—" | 工序任务 status=not_started | 进入详情页 | 实际开始字段显示"—" |
| TC-013-APP-011 | 实际结束时间：未完成显示"—" | 工序任务 status≠completed | 进入详情页 | 实际结束字段显示"—" |
| TC-013-APP-012 | 实际开始时间：已开始显示时间 | SE 已执行 Start Task | 进入详情页 | 实际开始字段显示系统记录时间，不可编辑 |
| TC-013-APP-013 | 实际结束时间：已完成显示时间 | SE 已执行 Mark as Complete | 进入详情页 | 实际结束字段显示系统记录时间，不可编辑 |

---

#### 模块三：Start Task 流程

| ID | 用例名称 | 前置条件 | 操作步骤 | 预期结果 |
|----|---------|---------|---------|---------|
| TC-013-APP-020 | Not Started 状态 Start Task 按钮可点 | 工序任务 status=not_started | 进入详情页，查看操作区 | Start Task 按钮正常显示，可点击（主色） |
| TC-013-APP-021 | 点击 Start Task 无确认弹框 | 工序任务 status=not_started | 点击 Start Task | 无任何弹框，直接执行 |
| TC-013-APP-022 | Start Task 后状态变 In Progress | 工序任务 status=not_started | 点击 Start Task → 等待成功 | 状态标签变为橙色"In Progress" |
| TC-013-APP-023 | Start Task 后实际开始时间更新 | 工序任务 status=not_started | 点击 Start Task 后查看详情 | 实际开始字段显示当前时间 |
| TC-013-APP-024 | Start Task 后按钮状态变化 | 工序任务 status=not_started | 点击 Start Task 后查看操作区 | Start Task 变 disabled，Mark as Complete 变可点 |
| TC-013-APP-025 | In Progress 状态 Start Task 按钮 disabled | 工序任务 status=in_progress | 进入详情页 | Start Task 按钮置灰，不可点击 |
| TC-013-APP-026 | Completed 状态 Start Task 按钮 disabled | 工序任务 status=completed | 进入详情页 | Start Task 按钮置灰，不可点击 |

---

#### 模块四：Mark as Complete 流程

| ID | 用例名称 | 前置条件 | 操作步骤 | 预期结果 |
|----|---------|---------|---------|---------|
| TC-013-APP-030 | Not Started 状态 Mark as Complete disabled | 工序任务 status=not_started | 进入详情页 | Mark as Complete 按钮置灰 |
| TC-013-APP-031 | In Progress 状态 Mark as Complete 可点 | 工序任务 status=in_progress | 进入详情页 | Mark as Complete 按钮正常，可点击 |
| TC-013-APP-032 | 点击 Mark as Complete 弹出确认弹框 | 工序任务 status=in_progress | 点击 Mark as Complete | 弹出确认弹框，标题"确认完成"，含取消/确认按钮 |
| TC-013-APP-033 | 取消确认弹框不变更状态 | 确认弹框已弹出 | 点击"取消" | 弹框关闭，状态仍为 in_progress |
| TC-013-APP-034 | 确认后状态变 Completed | 确认弹框已弹出 | 点击"确认" | 状态标签变为绿色"Completed" |
| TC-013-APP-035 | 确认后实际结束时间更新 | 确认完成后 | 查看详情页 | 实际结束字段显示当前时间 |
| TC-013-APP-036 | 完成后两个主操作按钮均 disabled | 工序任务 status=completed | 进入详情页 | Start Task 和 Mark as Complete 均置灰 |
| TC-013-APP-037 | Completed 状态 Mark as Complete disabled | 工序任务 status=completed | 进入详情页 | Mark as Complete 按钮置灰 |

---

#### 模块五：Report Issue

| ID | 用例名称 | 前置条件 | 操作步骤 | 预期结果 |
|----|---------|---------|---------|---------|
| TC-013-APP-040 | Not Started 状态 Report Issue 可点 | 工序任务 not_started | 进入详情页 | Report Issue 按钮可点击 |
| TC-013-APP-041 | In Progress 状态 Report Issue 可点 | 工序任务 in_progress | 进入详情页 | Report Issue 按钮可点击 |
| TC-013-APP-042 | Completed 状态 Report Issue 可点 | 工序任务 completed | 进入详情页 | Report Issue 按钮可点击 |
| TC-013-APP-043 | 问题上报提交成功 | 表单填写完整 | 点击提交 | 弹框关闭，提示上报成功，页面不跳转 |
| TC-013-APP-044 | 问题上报必填项校验 | 弹框打开 | 不填问题描述直接提交 | 校验提示，不允许提交 |

---

#### 模块六：异常场景

| ID | 用例名称 | 前置条件 | 操作步骤 | 预期结果 |
|----|---------|---------|---------|---------|
| TC-013-APP-050 | 网络异常时 Start Task 失败提示 | 模拟网络断开 | 点击 Start Task | 提示"操作失败，请重试"，状态不变 |
| TC-013-APP-051 | 网络异常时 Mark as Complete 失败提示 | 模拟网络断开 | 确认 Mark as Complete | 提示"操作失败，请重试"，状态不变 |
| TC-013-APP-052 | 非本人工序任务无法操作 | 当前登录为 SE-A，工序分配给 SE-B | 通过 URL 直接访问 SE-B 的工序详情 | 返回无权限提示 |
