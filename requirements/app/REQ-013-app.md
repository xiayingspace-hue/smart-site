# REQ-013 SE"我的任务"菜单与任务执行页面需求（APP端）

## 1. 需求背景

为提升现场工程师（SE）在移动端的工作效率和体验，需要为 SE 在 APP 端提供专属的"My Task"页面，便于其随时随地查看 CM 分配的工序任务、主动开始任务并执行标记完成操作。

## 2. 需求描述

作为现场工程师（SE），
我希望登录 APP 后，通过首页 Progress Management 模块下的 My Task 入口，可以：
- 查看 CM 分配给我的所有工序任务列表（仅显示与我相关的任务）。
- 查看工序任务详情，包括任务基本信息、计划时间、优先级等关键信息。
- 点击 **Start Task** 开始执行工序任务，系统自动记录实际开始时间。
- 点击 **Mark as Complete** 标记工序任务完成，系统自动记录实际结束时间。
- 对任务进展中的问题进行上报。

## 3. 页面结构与交互（APP端）

### 3.1 首页 Progress Management 模块

- APP 首页包含 Progress Management 模块，入口位于首页主要区域，图标及名称与 PC 端保持一致
- 不同角色在该模块下看到不同入口：
  - SE：显示 **My Task**
  - CM：显示 CM 专属入口（见 REQ-012）
  - PM：显示 PM 专属入口（见 REQ-011）
- 支持首页模块红点/数字角标提醒，提示有新工序任务分配或未读消息

### 3.2 My Task 工序任务列表页

- 入口：首页 Progress Management 模块 > **My Task**
- 页面顶部为筛选/搜索区，支持按任务状态（Not Started / In Progress / Completed）、计划时间、优先级等筛选
- 下方为工序任务卡片列表，每张卡片展示：
  - 工序任务名称
  - 所属上级任务名称
  - 计划开始/结束时间
  - 优先级
  - 任务状态（Not Started / In Progress / Completed）

### 3.3 工序任务详情页

点击任务卡片进入详情页，详情页包含：

**基本信息（只读）**
- 工序任务名称
- 所属上级任务
- WBS
- 计划开始/结束时间
- 实际开始时间（Start Task 后系统自动记录，未开始显示"—"）
- 实际结束时间（Mark as Complete 后系统自动记录，未完成显示"—"）
- 优先级
- 备注
- 当前状态（Not Started / In Progress / Completed）

**操作区**

页面操作按钮统一使用英文显示，状态流转为单向：**Not Started → In Progress → Completed**

| 按钮 | 可点击条件 | 置灰条件 |
|------|-----------|---------|
| **Start Task** | 状态为 Not Started | 状态为 In Progress 或 Completed |
| **Mark as Complete** | 状态为 In Progress | 状态为 Not Started 或 Completed |
| **Report Issue** | 任意状态均可点击 | — |

### 3.4 Start Task 交互

1. SE 点击 **Start Task** 按钮
2. 系统直接更新工序任务状态为 In Progress，**无需二次确认**
3. 系统自动记录**实际开始时间**（Actual Start）为当前时间
4. **Start Task** 按钮置灰，**Mark as Complete** 按钮变为可点击
5. 详情页"实际开始时间"字段实时更新展示

### 3.5 Mark as Complete 交互

1. SE 点击 **Mark as Complete** 按钮
2. 系统弹出确认提示：*"标记完成后将锁定该工序，如需修改请联系 CM，确认继续？"*
3. SE 确认后：
   - 工序任务状态变更为 Completed 并锁定
   - 系统自动记录**实际结束时间**（Actual End）为当前时间
   - SE 自身无法撤销（需 CM 介入，锁定解除流程待后续规划）
   - 系统自动重新计算上级任务的整体进度
4. SE 取消则保持原状态不变

### 3.6 Report Issue 表单

- SE 点击 **Report Issue** 按钮，弹出填写表单
- 填写字段：问题类型、问题描述、影响范围、建议、附件（可选）
- 提交后自动通知 CM

### 3.7 消息通知

- SE 收到以下消息提醒：
  - 新工序任务分配
  - 工序任务变更（时间、优先级等修改）
  - 工序任务作废
- 通知内容包含任务关键信息、变更内容、操作人、时间
- 消息中心统一查看，未读高亮

## 4. 权限说明

| 角色 | My Task 入口 | Start Task | Mark as Complete | Report Issue |
|------|-------------|------------|-----------------|--------------|
| SE | ✅ 可见 | ✅（仅自己的工序任务） | ✅（仅自己的工序任务） | ✅ |
| CM | ❌ 不显示 | ❌ | ❌ | ❌ |
| PM | ❌ 不显示 | ❌ | ❌ | ❌ |

## 5. 验收标准

- [ ] SE 登录 APP 后，首页 Progress Management 模块下可见 My Task 入口
- [ ] SE 可在 My Task 页面查看所有被分配的工序任务，支持筛选
- [ ] SE 可查看工序任务详情，包括基本信息、计划时间、实际时间、优先级、当前状态
- [ ] 详情页操作按钮以英文显示：Start Task / Mark as Complete / Report Issue
- [ ] SE 可对 Not Started 工序任务点击 Start Task，状态变为 In Progress，系统自动记录实际开始时间
- [ ] SE 可对 In Progress 工序任务点击 Mark as Complete，操作前弹出锁定确认提示
- [ ] 标记完成后状态变为 Completed，系统自动记录实际结束时间，工序锁定，上级任务进度自动更新
- [ ] SE 无法自行撤销已完成状态
- [ ] SE 可对任意状态工序任务点击 Report Issue 进行问题上报，提交后自动通知 CM
- [ ] SE 可收到工序任务相关消息通知
- [ ] CM/PM 不显示 My Task 入口

---

## 6. 对其他需求的影响

| 需求 | 影响说明 |
|------|----------|
| **REQ-001**（APP 首页） | 首页 Progress Management 模块需根据角色动态显示对应入口（SE 显示 My Task） |
| **REQ-012**（CM 端） | SE 标记完成后触发 REQ-012 中定义的进度计算规则，自动更新上级任务进度 |

---

> 备注：本需求为 SE 在 APP 端的专属工序任务查看与执行功能，配合 CM 端（REQ-012）实现全流程进度管理。
