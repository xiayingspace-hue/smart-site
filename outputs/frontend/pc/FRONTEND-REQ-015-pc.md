---
doc_type: frontend_spec
req_id: REQ-015
version: 0.2.0
status: draft
generated_from: requirement.md@0.5.0
data_contract_ref: data-contract.md@0.1.0
ui_spec_ref: UI-REQ-015-pc.md@0.2.0
generated_at: 2026-05-18
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明：BCA 月度人力数据提交（PC 端）

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。
>
> ⚠️ 重要约定：
> - 所有 API 字段定义引用 data-contract.md，本文档不重复定义。
> - 所有 UI 元素引用 UI-REQ-015-pc.md。
> - 代码注释中必须标注覆盖的 AC ID（如 `// AC-015-001`）。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-015 @ v0.5.0 |
| 数据契约 | data-contract.md @ v0.1.0 |
| UI 设计 | UI-REQ-015-pc.md @ v0.2.0 |
| 覆盖 Story | US-001、US-002、US-003、US-004、US-005 |
| 覆盖 AC | AC-015-001 ~ AC-015-011 |

---

## 1. 功能概述

新增「BCA 数据提交」模块，包含：提交任务列表页（当前项目上下文，无项目筛选）、发起提交弹窗（仅选月份）、提交任务详情页（批次列表 + 行内展开记录列表）、自动提交配置区域。支持查看任务/批次/记录三层状态，任务有 Sending 批次时自动轮询刷新，失败批次支持批次级重试（**不支持单条记录重试**）。

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

- 框架：Vue 2
- 语言：JavaScript（或 TypeScript，视项目现状）
- UI 库：Element UI
- 状态管理：Vuex
- 路由：Vue Router
- HTTP 客户端：Axios（项目封装层）
- 构建：Webpack / Vue CLI

### 2.2 本需求新增依赖

| 包 | 版本 | 用途 | 评估 |
|---|------|------|------|
| 无新增强依赖 | — | 复用现有 Element UI 组件 | — |

---

## 3. 路由设计

| 路径 | 组件 | 权限 | 关联 Story |
|-----|------|-----|-----------|
| `/bca-submission` | `BcaSubmissionList` | Site Admin / Platform Admin | US-001、US-002 |
| `/bca-submission/:taskId` | `BcaSubmissionDetail` | Site Admin / Platform Admin | US-002、US-003、US-004 |

**URL state 同步**：
- 提交任务列表页的筛选项（month、status）同步写入 URL query string，支持刷新保持和链接分享；无需同步 projectId（页面固定当前项目上下文）
- 任务详情页的批次展开状态不写入 URL（属于临时交互态）

---

## 4. 组件结构

### 4.1 组件树

```
<BcaSubmissionList>                      # 提交任务列表页
├── <SubmissionFilterBar>                 # 筛选栏（月份/状态，无项目下拉）
├── <el-table>                           # 任务列表表格
│   └── <SubmissionStatusTag>            # 状态圆点 + 文字（复用组件）
├── <SubmissionCreateDialog>             # 发起提交弹窗
│   ├── <el-date-picker type="month">    # 月份选择（无项目选择）
│   └── <InlineAlert>                    # 校验错误提示（重复发起/无数据）

<BcaSubmissionDetail>                    # 任务详情页
├── <SubmissionInfoHeader>               # 顶部基本信息区
├── <BatchProgressBar>                   # 批次进度条（多段颜色）
├── <el-table>（批次列表，带 expand）
│   ├── <SubmissionStatusTag>            # 批次状态
│   ├── <RetryButton>                    # 批次重试按钮（支持 Loading 态）
│   └── <BatchRecordList>               # 行内展开记录列表（只读）
│       ├── <el-table>（记录列表）
│       │   └── <SubmissionStatusTag>    # 记录状态（无重试按钮）
│       └── <ErrorTooltip>              # 超长错误信息 Tooltip

<AutoSubmitConfig>                       # 自动提交配置（项目设置页内）
├── <el-switch>（自动提交开关）
├── <el-select>（触发日 1~7）
└── <el-time-picker>（触发时间）
```

### 4.2 关键组件说明

#### `<SubmissionStatusTag>`

**关联 AC**：AC-015-003、AC-015-004、AC-015-005、AC-015-006

**Props**：
```js
// status:
//   Task 层级: 'success' | 'partial_failed' | 'failed'
//   Batch 层级: 'sending' | 'success' | 'failed'
//   Record 层级: 'success' | 'failed'
// level: 'task' | 'batch' | 'record'（不同层级文案略有差异）
props: {
  status: { type: String, required: true },
  level: { type: String, default: 'batch' }
}
```

**职责**：
- 根据 status 渲染对应颜色圆点 + 文字标签
- Sending 状态下圆点带 CSS 闪烁动画（`opacity` 关键帧）

**不该做**：
- 不持有任何业务状态，不发请求，纯展示组件

---

#### `<BatchProgressBar>`

**关联 AC**：AC-015-003、AC-015-004

**Props**：
```js
props: {
  total:   { type: Number, required: true },
  success: { type: Number, required: true },
  failed:  { type: Number, required: true },
  sending: { type: Number, default: 0 }
}
```

**职责**：
- 渲染多段颜色进度条（成功绿 / 失败红 / 发送中蓝 / 待发送灰）
- 右侧展示 `{success} / {total} 批次成功` 文字及「重试全部失败批次」按钮

**不该做**：
- 不直接触发重试请求，通过 `$emit('retry-all-failed')` 向父组件传递

---

#### `<RetryButton>`

**关联 AC**：AC-015-005、AC-015-006、AC-015-011

**Props**：
```js
props: {
  disabled: { type: Boolean, default: false },   // Sending 状态时置灰
  loading:  { type: Boolean, default: false },   // 重试中 Loading
  tooltipText: { type: String, default: '' }     // 置灰时 Tooltip 文案
}
```

**职责**：
- 点击时 `$emit('retry')`，由父组件执行请求
- disabled=true 时展示 Tooltip（「发送中，请稍候」）

---

#### `<SubmissionCreateDialog>`

**关联 AC**：AC-015-001、AC-015-002、AC-015-007

**职责**：
- 持有弹窗内表单状态（month）；projectId 从当前路由或全局项目上下文获取，无需用户选择
- 点击「确认发起」时调用校验接口 → 校验通过后调用创建接口
- 处理两类前置校验错误（重复/无数据），以 inline 方式展示，不关闭弹窗
- 成功后 `$emit('created', newTask)`，父组件刷新列表并高亮新行

---

#### `<BatchRecordList>`

**关联 AC**：AC-015-006、AC-015-010

**Props**：
```js
props: {
  batchId: { type: String, required: true }
}
```

**职责**：
- 展开时懒加载该批次的记录列表（仅在首次展开时请求）
- 记录列表最大高度 400px，超出内部滚动
- 记录列表为**只读展示**，不提供单条重试操作（BCA 接口限制）
- 失败记录展示错误描述，引导用户通过批次级「重试」按钮整批重发

**不该做**：
- 不提供单条记录重试按钮
- 不自动轮询，记录状态仅在批次重试后由父组件触发刷新

---

## 5. 状态管理

### 5.1 状态分层

| 状态类型 | 存放位置 | 例子 |
|---------|---------|------|
| 任务列表数据、详情数据 | Vuex store（`bca` module） | `bca/taskList`、`bca/taskDetail` |
| 筛选条件 | URL query string | `?month=2026-04&status=failed` |
| 弹窗开关 | 组件局部 data | `dialogVisible` |
| 批次展开状态 | 组件局部 data | `expandedBatchIds: Set<string>` |
| 记录列表数据（懒加载） | 组件局部 data（`BatchRecordList`） | `records: []` |
| 重试 Loading 状态 | 组件局部 data | `retryingBatchIds: Set<string>` |

### 5.2 Vuex Store 结构（`bca` module）

```js
// store/modules/bca.js
state: {
  taskList: [],          // 提交任务列表
  taskListTotal: 0,
  taskDetail: null,      // 当前详情页任务
  batchList: [],         // 当前详情页批次列表
  listLoading: false,
  detailLoading: false,
  pollingTimer: null,    // 轮询定时器 ID
}
```

### 5.3 缓存失效策略

| 操作 | 失效 / 刷新的数据 |
|-----|----------------|
| 发起提交成功 | 刷新 `taskList` |
| 批次重试成功 | 更新 `batchList` 中对应批次；重新评估 `taskDetail.status`；刷新该批次的记录列表 |
| 轮询触发 | 刷新 `taskDetail` + `batchList` |

---

## 6. API 接入

> 所有接口字段定义参见 `data-contract.md`，本节只列前端调用方式。

### 6.1 调用清单

| API | 调用组件 | 触发时机 | 缓存/轮询策略 |
|-----|---------|---------|-------------|
| `GET /bca-submissions` | `BcaSubmissionList` | 页面加载、筛选变更 | 无缓存，每次请求 |
| `POST /bca-submissions` | `SubmissionCreateDialog` | 点击「确认发起」 | 无缓存 |
| `GET /bca-submissions/:taskId` | `BcaSubmissionDetail` | 页面加载、轮询 | 有 Sending 批次时轮询，间隔 10s |
| `GET /bca-submissions/:taskId/batches` | `BcaSubmissionDetail` | 页面加载、轮询 | 同上 |
| `POST /bca-submissions/:taskId/batches/:batchId/retry` | `BcaSubmissionDetail` | 点击批次「重试」 | 无缓存 |
| `GET /bca-submissions/:taskId/batches/:batchId/records` | `BatchRecordList` | 批次行首次展开 | 懒加载，本地缓存（组件级）；批次重试后失效 |
| `GET /projects/:projectId/bca-auto-submit-config` | `AutoSubmitConfig` | 配置区加载 | 无缓存 |
| `PUT /projects/:projectId/bca-auto-submit-config` | `AutoSubmitConfig` | 点击「保存配置」 | 无缓存 |

### 6.2 轮询实现规范

```js
// BcaSubmissionDetail.vue
// AC-015-003 AC-015-004
startPolling() {
  this.pollingTimer = setInterval(async () => {
    await this.$store.dispatch('bca/fetchTaskDetail', this.taskId)
    await this.$store.dispatch('bca/fetchBatchList', this.taskId)
    // 所有批次均已落地（无 Sending），且任务处于终态时停止轮询
    const hasSendingBatch = this.batchList.some(b => b.status === 'sending')
    const terminalTaskStatuses = ['success', 'partial_failed', 'failed']
    if (!hasSendingBatch && terminalTaskStatuses.includes(this.taskDetail?.status)) {
      this.stopPolling()
    }
  }, 10000)
},
stopPolling() {
  if (this.pollingTimer) {
    clearInterval(this.pollingTimer)
    this.pollingTimer = null
  }
},
beforeDestroy() {
  this.stopPolling() // 离开页面必须清除，防内存泄漏
}
```

### 6.3 错误处理

```js
// 全局错误映射（用户可见文案）
const BCA_ERROR_MESSAGES = {
  'BCA_SUBMISSION_DUPLICATE':     '本月已有提交记录，无需重复操作',
  'BCA_NO_ATTENDANCE_DATA':       '该月暂无人员通行数据，无需提交',
  'BCA_BATCH_SENDING':            '当前批次发送中，请稍候再试',
  'NETWORK_TIMEOUT':              '请求超时，请检查网络后重试',
}
// 未命中映射的错误使用后端返回的 message，或通用兜底文案
```

---

## 7. 性能预算

| 指标 | 目标 | 测量 |
|-----|------|-----|
| 任务列表页 FCP | < 1.5s | Lighthouse |
| 批次详情页 FCP | < 1.5s | Lighthouse |
| 批次列表（100 条）渲染 | < 300ms | Chrome DevTools |
| 记录列表懒加载响应 | < 500ms（含网络） | Network panel |

**优化手段**：
- 批次列表不使用虚拟滚动（200 条以内 DOM 可接受），超 200 条启用分页
- 记录列表懒加载（首次展开时请求），已加载则直接复用组件内缓存，不重复请求
- 轮询使用 `setInterval`，离开页面时 `beforeDestroy` 清除，防止内存泄漏

---

## 8. 弱网与离线策略

| 场景 | 策略 |
|-----|------|
| 轮询请求超时/失败 | 静默忽略，继续下一次轮询，不弹错误提示（避免频繁打扰） |
| 发起提交/重试请求超时 | Toast 提示「操作失败，请检查网络后重试」，按钮恢复可点击 |
| 页面加载失败 | 表格区展示错误占位图 + 「刷新重试」按钮 |

---

## 9. 错误边界与降级

- `BcaSubmissionDetail` 和 `BatchRecordList` 需捕获渲染异常，降级为错误占位提示
- 轮询期间接口异常不中断轮询，仅记录 console.warn，不打断用户操作

---

## 10. 埋点设计

| 事件名 | 触发时机 | 关键属性 | 关联指标 |
|-------|---------|---------|---------|
| `bca_submission_create` | 发起提交成功 | `project_id`、`month`、`trigger_type: 'manual'` | 月度提交次数 |
| `bca_submission_batch_retry` | 点击批次重试 | `task_id`、`batch_id`、`batch_status_before` | 重试频率 |
| `bca_submission_view_detail` | 进入详情页 | `task_id`、`task_status` | 页面访问深度 |
| `bca_submission_expand_batch` | 展开批次查看记录 | `task_id`、`batch_id`、`batch_status` | 查看记录行为 |

---

## 11. 国际化

- i18n 库：vue-i18n（与现有系统一致）
- 文案来源：UI-REQ-015-pc.md §9（全量中英文案已列出）
- 当前支持：zh-CN、en
- 新增 i18n key 命名空间：`bca_submission.*`

```js
// i18n key 示例（与 ui-spec 文案对应）
'bca_submission.page_title':              'BCA 数据提交 / BCA Data Submission',
'bca_submission.create_btn':              '+ 发起提交 / + New Submission',
'bca_submission.duplicate_error':         '本月已有提交记录，无需重复操作',
'bca_submission.no_data_error':           '该月暂无人员通行数据，无需提交',
'bca_submission.status.task.success':     '成功 / Success',
'bca_submission.status.task.partial_failed': '部分失败 / Partial Failed',
'bca_submission.status.task.failed':      '失败 / Failed',
'bca_submission.status.batch.sending':    '发送中 / Sending',
'bca_submission.status.batch.success':    '成功 / Success',
'bca_submission.status.batch.failed':     '失败 / Failed',
// ... 其余见 ui-spec §9
```

---

## 12. 测试要求

| 层级 | 框架 | 覆盖范围 | 覆盖率目标 |
|-----|------|---------|-----------|
| 单元 | Jest + Vue Test Utils | `SubmissionStatusTag`、`BatchProgressBar`、`RetryButton` 纯展示逻辑；状态枚举映射 | ≥ 80% |
| 集成 | Jest + Vue Test Utils + Mock API | `SubmissionCreateDialog` 表单校验流程；`BcaSubmissionDetail` 轮询启停逻辑；重试按钮 Loading 态 | 主流程 100% |
| E2E | QA 维护（Cypress） | 见 qa-spec | — |

---

## 13. AC 覆盖检查表

| AC ID | 实现位置（文件/组件） | 测试覆盖 | 状态 |
|------|------------------|---------|------|
| AC-015-001 | `SubmissionCreateDialog` → `POST /bca-submissions` | 集成测试 | TODO |
| AC-015-002 | `SubmissionCreateDialog` inline 错误提示 + 按钮禁用 | 集成测试 | TODO |
| AC-015-003 | `BcaSubmissionDetail` 轮询 + `SubmissionStatusTag` | 集成测试 | TODO |
| AC-015-004 | `BcaSubmissionDetail` 轮询 + 错误信息列渲染 | 集成测试 | TODO |
| AC-015-005 | `RetryButton` → `POST .../retry` + 状态更新 | 集成测试 | TODO |
| AC-015-006 | `BatchRecordList` 无重试按钮；失败记录展示错误信息 + 说明 | 集成测试 | TODO |
| AC-015-007 | `SubmissionCreateDialog` 无数据校验 inline 提示 | 单元测试 | TODO |
| AC-015-008 | `AutoSubmitConfig` 保存配置 → 显示「自动」Tag | 集成测试 | TODO |
| AC-015-009 | 自动任务创建后不重复出现（后端保证，前端无需额外处理） | — | TODO |
| AC-015-010 | `BatchRecordList` 展示快照数据（来自 API 快照字段） | 集成测试 | TODO |
| AC-015-011 | `RetryButton` disabled 态 + Tooltip（Sending 状态） | 单元测试 | TODO |

---

## 14. 与后端的联调约定

- **联调前置**：data-contract.md 已锁版，接口 Schema 确定
- **Mock 服务**：基于 data-contract 使用 Mock.js / MSW 搭建 Mock，前端可独立开发
- **联调环境**：dev 环境
- **轮询接口**：后端需保证批次列表接口 P95 < 500ms，避免轮询堆积
- **接口变更**：任何字段变更必须先更新 data-contract，再通知前后端

---

## 15. 验收条件

- [ ] 所有 AC 在 §13 表中标记完成
- [ ] 单元 + 集成测试通过，覆盖率达标
- [ ] E2E 测试由 QA 跑通（不阻塞前端提测）
- [ ] 无 console 错误/警告
- [ ] 页面切换/离开时轮询定时器正确清除（无内存泄漏）
- [ ] i18n 中英文案完整，无遗漏 key
- [ ] 已部署到 dev 环境，可联调

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-17 | | 初稿，覆盖任务列表、发起弹窗、批次详情、记录列表、自动提交配置、轮询机制 |
| 0.2.0 | 2026-05-18 | | 同步 REQ-015 v0.2.0~v0.5.0：1) 弹窗移除项目选择，筛选区移除 projectId；2) SubmissionStatusTag 移除 pending/in_progress；3) 移除单条记录重试 API 及 RetryButton；4) BatchRecordList 改为只读；5) 轮询条件改为检查 Sending 批次；6) 防重校验文案更新；7) i18n 精简状态 key；8) 埋点移除单条重试事件 |
