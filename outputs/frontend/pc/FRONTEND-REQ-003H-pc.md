---
doc_type: frontend_spec
req_id: REQ-003H-pc
version: 0.1.0
status: draft
generated_from: REQ-003H-pc@0.1.1
ui_spec_ref: UI-REQ-003H-pc.md@0.1.0
data_contract_ref: DATA-CONTRACT-REQ-003H-pc.md@0.1.0
generated_at: 2026-08-07
generator: frontend-agent
owner: ""
---

# 前端开发说明：PC 端 — 免审批上传（已完成审批的图纸直接生效）

> **本文档定义前端实现要点**。API 字段定义引用 [DATA-CONTRACT-REQ-003H-pc](../../shared/DATA-CONTRACT-REQ-003H-pc.md)，视觉与交互引用 [UI-REQ-003H-pc](../../ui/pc/UI-REQ-003H-pc.md)，**均不重复定义**。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003H-pc @ v0.1.1 |
| UI 说明 | UI-REQ-003H-pc.md @ v0.1.0 |
| 数据契约 | DATA-CONTRACT-REQ-003H-pc.md @ v0.1.0 |
| 覆盖 Story | US-003H-001、US-003H-002、US-003H-003、US-003H-004 |
| 覆盖 AC | AC-003H-001、002、003、004、006、008、009、010、011、012、013 |
| 上次同步时间 | 2026-08-07 |

---

## 1. 功能概述

在既有上传弹窗中增加一个受权限控制的单选区，并在版本历史与图纸列表中增加两处基于 `approvalRoute` 的渲染分支。**不新增页面、不新增路由、不新建组件**。

前端侧的三个关键点：**权限缺失时区域不进入 DOM**、**提交失败时弹窗不关闭且内容不丢**、**Standard 路径代码路径零改动**。

---

## 2. 技术栈

### 2.1 沿用现有

| 项 | 规范 |
|---|------|
| 框架 | Vue 2 + Element UI |
| 语言 | JavaScript（ES6+） |
| 样式 | SCSS |
| 构建 | Webpack |
| 包管理 | npm |

> 见 [background/tech-stack.md §1](../../../background/tech-stack.md)。

### 2.2 新增依赖

**无。** 本需求仅使用 Element UI 既有组件（`el-radio-group` / `el-alert` / `el-tooltip`）。

---

## 3. 路由设计

**本需求不新增任何路由**，也不改变既有路由的权限守卫。

| 涉及路由 | 变更 |
|---------|------|
| 图纸管理列表页 | 无路由变更；页面内 Status 列渲染逻辑扩展 |
| 上传弹窗 | 非独立路由（Modal，不加 query） |
| 版本历史抽屉 | 非独立路由（Drawer，不加 query） |

> 弹窗与抽屉均不写入 URL——沿用 [frontend-rules §3.1](../../../rules/frontend-rules.md)「Modal 内部表单数据不写入 URL」的约定。

---

## 4. 组件结构

### 4.1 组件树（变更部分）

```
DrawingListPage（既有）
├── DrawingTable（既有）
│   └── StatusCell（既有）
│       └── ✏️ 新增 Pre-approved 标识渲染分支
├── UploadDrawingDialog（既有，REQ-003E）
│   ├── … Description / Category / SubmissionType / 文件上传 / AI 识别 …
│   ├── 🆕 ApprovalRouteField（新增子组件）
│   └── InternalApproverSelect（既有）
│       └── ✏️ 新增 v-if 控制
└── VersionHistoryDrawer（既有，REQ-007C）
    └── VersionLifecycleCards（既有）
        └── ✏️ 新增 skipped 渲染分支
```

### 4.2 新建组件

#### `<ApprovalRouteField>`

**关联 AC**：AC-003H-001、AC-003H-002、AC-003H-003

**Props**：

```js
{
  value: String,      // v-model，'STANDARD' | 'PRE_APPROVED'
  visible: Boolean    // 由父级传入权限判定结果；false 时组件整体不渲染
}
```

**Events**：`input`（v-model 同步）

**职责**：

- 渲染 Approval Route 单选组与警示文案块
- 通过 `v-if="visible"` 控制自身**是否进入 DOM**

**不该做**：

- ❌ 不自行查询权限——权限判定由父级弹窗完成并通过 prop 传入，避免组件内藏隐式依赖
- ❌ 不控制 `InternalApproverSelect` 的显隐——那是父级弹窗根据 `approvalRoute` 值决定的，子组件不反向操作兄弟组件
- ❌ 不发起任何 API 调用

**实现要点**：

```vue
<!-- 关键：v-if 而非 v-show。AC-003H-001 要求 DOM 中不存在 -->
<div v-if="visible" class="approval-route-field">
  <el-radio-group :value="value" @input="$emit('input', $event)">
    <el-radio label="STANDARD">…</el-radio>
    <el-radio label="PRE_APPROVED">…</el-radio>
  </el-radio-group>
  <el-alert
    v-if="value === 'PRE_APPROVED'"
    type="warning"
    :closable="false"
    :show-icon="true"
  >…</el-alert>
</div>
```

> ⚠️ **`v-if` 不可写成 `v-show`**。`v-show` 只是 `display:none`，元素仍在 DOM 中，无权限用户可通过 DevTools 看到该功能存在——直接违反 AC-003H-001，也违反 [ui-rules §3.2.3](../../../rules/ui-rules.md)「隐藏 > 禁用」的安全意图。

### 4.3 既有组件的改动

| 组件 | 改动 |
|-----|------|
| `UploadDrawingDialog` | 引入 `<ApprovalRouteField>`；表单模型增加 `approvalRoute`（默认 `'STANDARD'`）；`InternalApproverSelect` 加 `v-if`；提交前按路径组装 payload |
| `InternalApproverSelect` | 外层加 `v-if="form.approvalRoute === 'STANDARD'"`；切换时清空已选值 |
| `StatusCell` | 增加 `approvalRoute === 'PRE_APPROVED'` 分支，渲染 `Pre-approved` 灰字 + Tooltip |
| `VersionLifecycleCards` | ②③ 卡片与步骤条增加 `skipped` 分支 |

> ⚠️ `VersionLifecycleCards` 的改动依赖该组件是否支持外部扩展状态枚举，见 [UI-REQ-003H-pc OQ-UI-001](../../ui/pc/UI-REQ-003H-pc.md)。若组件内部状态写死为 5 种，需先改造组件本身。

---

## 5. 状态管理

**本需求不引入任何全局 store。** 全部状态归类如下（[frontend-rules §3.3](../../../rules/frontend-rules.md)）：

| 状态 | 归类 | 理由 |
|-----|------|------|
| `form.approvalRoute` | 组件内 `data` | 表单字段，生命周期与弹窗一致 |
| `canUsePreApproved`（权限判定结果） | 从既有权限集派生（computed） | 权限集已在登录态中，**不新增请求** |
| `submitting`（提交 loading） | 组件内 `data` | 仅当前弹窗使用 |
| 列表数据中的 `approvalRoute` | 随列表响应而来 | 服务端数据，沿用既有列表数据流 |
| 版本历史中的 `approvalRoute` | 随版本列表响应而来 | 同上 |

**权限判定的实现**：

```js
computed: {
  canUsePreApproved() {
    // 与后端一致：两个权限必须同时持有
    return this.$hasPermission('drawing:upload')
        && this.$hasPermission('drawing:upload-approved');
  }
}
```

> **前端权限仅控渲染，不构成授权**。后端会独立实时校验（见 [BACKEND-REQ-003H-pc §13.2](../../backend/BACKEND-REQ-003H-pc.md)）。因此**不需要**在前端做权限轮询或实时刷新——权限被回收的情况由提交时的 `1003003020` 兜底处理（AC-003H-013）。

**表单联动**：

```js
watch: {
  'form.approvalRoute'(next) {
    if (next === 'PRE_APPROVED') {
      this.form.approverId = null;              // 清空已选审批人
      this.clearValidate('approverId');         // 解除必填校验残留的错误提示
    }
    // 切回 STANDARD 时不回填历史值——避免用户误以为已选（见 UI §3.1）
  }
}
```

---

## 6. API 接入

### 6.1 涉及接口

| API ID | 用途 | 变更 |
|--------|------|------|
| API-003H-001 | 创建图纸 | 请求体新增可选字段 `approvalRoute` |
| `/drawing/page` | 图纸列表 | 响应新增 `approvalRoute`（只读消费） |
| `/drawing/version/list` | 版本历史 | 响应新增 `approvalRoute`（只读消费） |

字段定义见 [DATA-CONTRACT §4.2](../../shared/DATA-CONTRACT-REQ-003H-pc.md)，此处不重复。

### 6.2 请求组装

```js
buildCreatePayload() {
  const base = { /* … 既有字段，一行不改 … */ };

  // 无权限用户：不带该字段，请求体与上线前完全一致（AC-003H-009）
  if (!this.canUsePreApproved) return base;

  if (this.form.approvalRoute === 'PRE_APPROVED') {
    // approverId 不传——后端即使收到也会忽略，但前端不发无意义字段
    return { ...base, approvalRoute: 'PRE_APPROVED' };
  }
  return { ...base, approvalRoute: 'STANDARD', approverId: this.form.approverId };
}
```

> **为什么无权限时完全不传该字段**：而不是传 `'STANDARD'`。这样 Standard 路径的请求体与上线前逐字节一致，回归测试可直接做全量比对（AC-003H-009）。

### 6.3 错误码 → 用户提示映射

| 错误码 | HTTP | 处理 | 弹窗是否关闭 |
|--------|------|------|:-----------:|
| `1003003020` | 403 | Toast（文案见 UI §9）；提示用户可改选 Standard | **否** |
| `1003003021` | 400 | Toast（文案见 UI §9） | **否** |
| `1003007009` | 500 | Toast「QR generation failed, please retry」；[Confirm] 恢复可点 | **否** |
| `1003003019` | 400 | 挂到 Drawing Code 字段下方的字段级错误 | 否 |
| `1003003002/003/018` | 400 | 沿用既有处理 | 否 |
| 401 | 401 | 全局拦截跳登录 | — |

**统一原则**：**任何提交失败都不关闭弹窗、不清空已填内容、不清除已上传文件引用**。

理由：文件可能有 50MB，重传成本极高。QR 失败是可当场重试的瞬时故障（AC-003H-008），权限被回收后用户可改选 Standard 继续（AC-003H-013）——两种场景都要求表单状态完整保留。

```js
async handleConfirm() {
  this.submitting = true;
  try {
    const res = await createDrawing(this.buildCreatePayload());
    this.$message.success(this.$t('drawing.createdAndActivated'));
    this.$emit('success', res.data);
    this.close();                        // 仅成功时关闭
  } catch (err) {
    this.$message.error(err.msg);        // 失败：仅提示，不做任何状态清理
  } finally {
    this.submitting = false;             // 无论成败都恢复按钮，保证可重试
  }
}
```

### 6.4 成功后的分支处理

`approvalRoute = PRE_APPROVED` 的成功响应中 `approvalStatus` 为 `APPROVED`、`approvalId` 为 `null`（见 [DATA-CONTRACT §4.2](../../shared/DATA-CONTRACT-REQ-003H-pc.md)）。前端据此选择成功文案：

```js
const msg = res.data.approvalStatus === 'APPROVED'
  ? this.$t('drawing.createdAndActivated')   // 已创建并生效
  : this.$t('drawing.submittedForApproval'); // 已提交审批（既有文案）
```

> **不要**用 `form.approvalRoute` 判断——以服务端返回为准，避免前端状态与实际落库结果不一致时给出错误提示。

---

## 7. 性能预算

| 指标 | 目标 | 说明 |
|-----|------|------|
| 权限判定 | 0 次额外请求 | 从登录态权限集派生（§5） |
| 提交等待 | ≤ 5 秒（P95） | 后端同步事务，见 [BACKEND §12.1](../../backend/BACKEND-REQ-003H-pc.md) |
| 新增 bundle 体积 | < 2KB（gzip） | 仅一个小组件 + 文案 |
| 列表渲染 | 无额外开销 | `approvalRoute` 为随响应而来的普通字段，不触发额外计算 |

> 需求 §10.1 未指定更严格指标，其余沿用 [frontend-rules §3.5.1](../../../rules/frontend-rules.md) 默认值。

**loading 期间的交互阻断**：提交中必须禁用弹窗全部交互（含 ✕ 与 Cancel）。3–5 秒的等待里用户若关闭弹窗，会造成"以为取消了但实际已生效"的认知偏差。

---

## 8. 弱网与离线策略

**不适用。** 本需求属 `pc/` 目录（Desktop 管理端），按 [frontend-rules §3.6](../../../rules/frontend-rules.md) 填"不适用"。

唯一相关的是提交超时：沿用既有 30 秒请求超时。超时后提示用户重试，**不自动重试**——该操作非幂等（见 [BACKEND §6](../../backend/BACKEND-REQ-003H-pc.md)），自动重试可能创建重复记录。

---

## 9. 错误边界与降级

| 场景 | 处理 |
|-----|------|
| 权限集读取异常 | `canUsePreApproved` 取 `false` → 区域不渲染，降级为 Standard 单一路径。**安全侧降级**：宁可让有权限的人暂时用不了，不可让无权限的人看到 |
| `approvalRoute` 字段缺失（老接口 / 灰度未开） | 列表与版本历史按 `STANDARD` 渲染，不显示标识、不走跳过态分支 |
| `VersionLifecycleCards` 收到未知状态组合 | 沿用 REQ-007C 既有兜底分支，不因新增分支而崩溃 |
| 弹窗内 JS 异常 | 沿用既有弹窗级 ErrorBoundary；错误上报 |

---

## 10. 埋点设计

### 10.1 事件

| 事件名 | 触发 | 属性 |
|-------|------|------|
| `drawing.upload.route.switch` | 用户切换 Approval Route | `to`（STANDARD / PRE_APPROVED） |
| `drawing.preApproved.submit.start` | 选中 Pre-Approved 并点击 [Confirm] | `submissionType` |
| `drawing.preApproved.submit.success` | 提交成功 | `duration_ms`、`submissionType` |
| `drawing.preApproved.submit.fail` | 提交失败 | `error_code`、`duration_ms` |
| `drawing.preApproved.field.view` | Approval Route 区域首次渲染 | — |

### 10.2 反推自成功指标

对 [REQ-003H-pc §16](../../../requirements/pc/REQ-003H-pc.md) 每个成功指标反推埋点（[frontend-rules §3.8.2](../../../rules/frontend-rules.md)）：

| 成功指标 | 对应埋点 |
|---------|---------|
| 存量图纸补录耗时缩短 > 80% | `submit.start` + `submit.success` 的 `duration_ms` |
| 免审批占新建图纸比例稳态 < 5% | `preApproved.submit.success` ÷ 全部创建成功数（后端侧亦有统计，见 BACKEND §14.1） |
| 免审批图纸事后异议率 = 0 | 无前端埋点，靠人工复核 |
| 线下私发图纸减少 | 无埋点，定性访谈 |

> `field.view` 与 `route.switch` 用于观察"有权限的人是否真的在用这条路径"，可辅助判断权限授予是否合理。

---

## 11. 国际化

| 项 | 说明 |
|---|------|
| 语种 | 中英双语，默认 English（[tech-stack §5](../../../background/tech-stack.md)） |
| 新增 key | 见 [UI-REQ-003H-pc §9](../../ui/pc/UI-REQ-003H-pc.md) 文案表，共 13 条 |
| key 命名 | `drawing.approvalRoute.*` / `drawing.preApproved.*` |
| 注意 | 单选项与警示文案在 en 下较长，**样式不得设固定高度或 `text-overflow: ellipsis`**（见 UI §3.1 / §3.2 极端数据态） |

---

## 12. 测试要求

> 团队当前未强制单测，以下为建议项。

| 层级 | 建议程度 | 覆盖重点 |
|-----|---------|---------|
| 单元 | 建议 | `buildCreatePayload()` 三种分支（无权限 / STANDARD / PRE_APPROVED）；`canUsePreApproved` 复合权限判定 |
| 集成 | **强烈建议** | 提交失败后弹窗状态保留（表单值、文件引用、按钮可点）——这是最易在重构中被破坏的行为 |
| E2E | QA 负责 | 见 [QA-REQ-003H-pc](../../qa/pc/QA-REQ-003H-pc.md) |

**必须覆盖的回归断言**：无权限用户的弹窗 DOM 结构与请求体与上线前完全一致（AC-003H-001、AC-003H-009）。

---

## 13. AC 覆盖检查表

| AC ID | 实现位置 | 覆盖 |
|------|---------|:---:|
| AC-003H-001 | §4.2 `<ApprovalRouteField>` 的 `v-if="visible"`；§5 `canUsePreApproved` | ✅ |
| AC-003H-002 | §5 表单模型默认 `'STANDARD'`；§4.2 组件渲染 | ✅ |
| AC-003H-003 | §5 `watch` 联动清空 + 解除校验；§4.2 警示文案 `v-if` | ✅ |
| AC-003H-004 | §6.4 成功分支文案；§6.3 成功后关闭并刷新列表 | ✅ |
| AC-003H-005 | 后端行为，前端无实现 | N/A |
| AC-003H-006 | §6.3 错误码 `1003003021` → Toast，弹窗不关 | ✅ |
| AC-003H-007 | 后端拦截；前端仅呈现 Toast（§6.3） | N/A |
| AC-003H-008 | §6.3 `1003007009` 处理 + `finally` 恢复按钮，表单与文件引用不清理 | ✅ |
| AC-003H-009 | §6.2 无权限时不传 `approvalRoute`，请求体逐字节一致 | ✅ |
| AC-003H-010 | §4.3 `VersionLifecycleCards` skipped 分支 | ✅ |
| AC-003H-011 | §4.3 `StatusCell` Pre-approved 分支；SE 侧因后端不返回该字段而自然不渲染（§9） | ✅ |
| AC-003H-012 | 下载按钮沿用 REQ-007C 既有 `pdfWithQrUrl > signedFileUrl` 逻辑，无需改动 | ✅ |
| AC-003H-013 | §6.3 `1003003020` 处理，弹窗保持打开、内容保留、可改选 Standard | ✅ |

**覆盖率**：11 / 13，2 条纯后端 AC 标 N/A。

---

## 14. 与后端的联调约定

| 项 | 约定 |
|---|------|
| `approvalRoute` 缺省 | 后端取默认值 `STANDARD`，行为与上线前一致 |
| `approverId` 在 PRE_APPROVED 下传入 | 后端**忽略**而非报错（见 [DATA-CONTRACT §4.2](../../shared/DATA-CONTRACT-REQ-003H-pc.md) 参数联动）。前端不传，但该约定保证切换路径后残留值不会导致提交失败 |
| 成功响应判定 | 前端以 `data.approvalStatus` 判断文案分支，不以自身表单状态判断 |
| SE 视角字段过滤 | 由**后端**在响应中剔除 `approvalRoute`（BACKEND §3.3），前端不做角色判断 |
| QR 失败语义 | `1003007009` 表示**整体回滚、无任何记录残留**，前端可安全提示"重试"而不必担心产生半成品数据 |
| 灰度期兼容 | Feature Flag 关闭时后端对 `PRE_APPROVED` 返回 `1003003020`，前端按无权限处理即可，无需额外分支 |

---

## 15. 验收条件

- [ ] 无 `drawing:upload-approved` 权限时，Approval Route 区域**不在 DOM 中**（DevTools 验证）
- [ ] 有权限时默认选中 Standard，Internal Approver 正常必填
- [ ] 切换到 Pre-Approved：审批人隐藏、值清空、校验解除、警示文案出现
- [ ] 切回 Standard：审批人恢复且**不回填**历史值
- [ ] 提交中弹窗全部交互被阻断（含 ✕ 与 Cancel）
- [ ] 三类失败（1003003020 / 1003003021 / 1003007009）均不关闭弹窗、不清空表单、按钮恢复可点
- [ ] 无权限用户的创建请求体与上线前逐字段一致
- [ ] 列表与版本历史在 `approvalRoute` 缺失时按 STANDARD 正常渲染
- [ ] 新增文案中英双语完整，长文案不截断
- [ ] §10 埋点全部接入

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-08-07 | frontend-agent | 初稿：新建 `<ApprovalRouteField>` 组件（`v-if` 控制入 DOM）；定义表单联动、请求组装三分支、错误码映射与"失败不关弹窗"原则、成功文案以服务端响应为准、埋点与联调约定 |
