---
doc_type: frontend_spec
req_id: REQ-003G-pc
version: 0.1.0
status: draft
generated_from: REQ-003G-pc@0.3.0
data_contract_ref: data-contract-003G-pc.md@0.1.0
ui_spec_ref: ui-spec-003G-pc.md@0.1.0
generated_at: 2026-05-28
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明：PC 端 — 图纸管理区域配置与双模式 SE 分配（单选 + 按区域批量）

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **本文档供前端开发工程师及其 agent 使用**。
>
> ⚠️ **重要约定**：
> - 所有 API 字段定义必须**引用** data-contract.md，本文档不重复定义。
> - 所有 UI 元素引用 ui-spec.md。
> - 代码注释中必须标注覆盖的 AC ID（如 `// AC-003G-001`）。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003G-pc @ v0.2.0 |
| 数据契约 | data-contract-003G-pc.md @ v0.1.0 |
| UI 设计 | ui-spec-003G-pc.md @ v0.1.0 |
| 覆盖 Story | US-003G-001、US-003G-002、US-003G-003、US-003G-004 |
| 覆盖 AC | AC-003G-001 ～ AC-003G-010b |

---

## 1. 功能概述

本次前端交付两大功能模块：
1. **区域配置模块**（新页面）：在图纸管理页增加 `[Area Config]` 入口，提供区域 CRUD（F-001～F-004）和区域 SE 绑定管理弹框（F-005）。
2. **[Assign] 弹框双模式增强**（现有页面增量）：在 REQ-003D-pc 的图纸 Assign SE 弹框 Available 面板顶部新增 `[Select by Area ▾]` 按钮，实现按区域批量 Add SE（F-006），与原有单选模式并存可混用。

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

- 框架：Vue 3
- 语言：TypeScript
- 状态管理：Pinia
- UI 库：Element Plus（Element UI Vue 3 版本）
- 路由：Vue Router 4
- HTTP 客户端：Axios（封装统一请求层）
- 构建：Vite
- 测试：Vitest（单元）、Cypress（E2E）

### 2.2 本需求新增依赖

| 包 | 版本 | 用途 | 评估 |
|---|------|------|------|
| 无新增 | — | 全部在现有技术栈内实现 | — |

---

## 3. 路由设计

| 路径 | 组件 | 权限 | 关联 Story |
|-----|------|-----|-----------|
| `/projects/:projectId/drawings/area-config` | `AreaConfigPage` | `drawing:area-config` | US-003G-001、US-003G-002 |

> ⚠️ 若 PM 最终确认以 Drawer 替代独立页面（OQ-002），则无需新增路由，改由 `DrawingListPage` 内挂载 `<AreaConfigDrawer>`。

**路由守卫**：
- 进入 `/area-config` 时检查当前用户是否具有 `drawing:area-config` 权限；无权限跳转 403 页面。

**URL state 同步**：
- 区域列表搜索关键词同步到 query string：`?q=`，支持刷新保持。

---

## 4. 组件结构

### 4.1 组件树

```
<DrawingListPage>（现有，REQ-003A-pc）
└── <AreaConfigEntryButton>      // F-001：Area Config 入口按钮
    └── [路由跳转]

<AreaConfigPage>                  // F-002：区域配置列表页
├── <AreaConfigToolbar>
│   ├── <ElButton> [+ Add Area]
│   └── <ElInput> 搜索框
├── <AreaTable>                   // 区域列表表格
│   └── <AreaTableRow> × N
│       ├── [Manage SE] → <ManageSEsDialog>
│       ├── [Edit]      → <AreaFormDialog mode="edit">
│       └── [Delete]    → <DeleteAreaDialog>
├── <AreaFormDialog>              // F-003：新建/编辑弹窗
├── <DeleteAreaDialog>            // F-004：删除确认弹窗
└── <ManageSEsDialog>             // F-005：Manage SEs 弹框

<AssignSEDialog>（现有，REQ-003D-pc F-002 扩展）// F-006：双模式
└── <AvailablePanel>（扩展）
    ├── <SelectByAreaButton>      // [Select by Area ▾] 按钮
    └── <SelectByAreaDropdown>    // 区域多选下拉面板
        ├── <ElCheckbox> Select All
        ├── <ElCheckbox> × N（区域列表）
        └── <ElButton> [Add Selected Areas]
```

### 4.2 关键组件说明

#### `<AreaConfigEntryButton>`

**关联 AC**：AC-003G-001

**Props**：
```ts
interface AreaConfigEntryButtonProps {
  projectId: string
}
```

**职责**：
- 根据用户权限 `drawing:area-config` 决定是否渲染；无权限时 `v-if="false"`，完全不渲染。
- 点击后路由跳转至区域配置页。

**不该做**：不处理权限校验逻辑本身（由 `usePermission` hook 提供）。

---

#### `<AreaConfigPage>`

**关联 AC**：AC-003G-002、AC-003G-003、AC-003G-010b

**职责**：
- 拉取当前项目区域列表，渲染 `<AreaTable>`。
- 管理 `<AreaFormDialog>`、`<DeleteAreaDialog>`、`<ManageSEsDialog>` 的显示状态。
- 处理增删改操作后刷新列表。

---

#### `<AreaFormDialog>`

**关联 AC**：AC-003G-002、AC-003G-003

**Props**：
```ts
interface AreaFormDialogProps {
  mode: 'create' | 'edit'
  area?: { areaId: string; areaName: string } // 编辑时传入
  visible: boolean
}
// Emits: 'saved' | 'cancel'
```

**职责**：
- 含 Area Name 输入框，最大 50 字符。
- 提交前校验名称非空；保存后接收 409 错误展示 inline 提示 `Area name already exists.`。

---

#### `<DeleteAreaDialog>`

**关联 AC**：AC-003G-004

**Props**：
```ts
interface DeleteAreaDialogProps {
  area: { areaId: string; areaName: string }
  visible: boolean
}
// Emits: 'deleted' | 'cancel'
```

**职责**：显示含区域名的确认文案，点击 [Delete] 调用删除接口后刷新列表。

---

#### `<ManageSEsDialog>`

**关联 AC**：AC-003G-005

**Props**：
```ts
interface ManageSEsDialogProps {
  area: { areaId: string; areaName: string }
  visible: boolean
}
// Emits: 'saved' | 'cancel'
```

**职责**：
- 双栏布局（In Area / Available），从接口拉取数据，本地暂存 Add/Remove 操作，点 [Save] 统一提交。
- 两侧各有独立搜索框，实时过滤本地已拉取的 SE 列表（不重新请求）。

**不该做**：不在每次 Add/Remove 点击时调用接口。

---

#### `<SelectByAreaButton>` + `<SelectByAreaDropdown>`

**关联 AC**：AC-003G-006、AC-003G-007、AC-003G-008、AC-003G-009、AC-003G-011、AC-003G-012、AC-003G-013

**Props**（`SelectByAreaButton`）：
```ts
interface SelectByAreaButtonProps {
  projectId: string
  disabled: boolean   // 无区域时置灰
  disabledTooltip: string
}
// Emits: 'areas-selected'（payload: string[] areaIds）
```

**职责**（下拉面板）：
- 拉取本项目区域列表（含每区域 SE 及其详情），渲染**树形 Checkbox 列表**：
  - 区域节点：支持三态 Checkbox（全选 / indeterminate / 未选）；点击展开/折叠子 SE 列表
  - SE 子节点：独立 Checkbox，可单独勾选/取消
  - 区域 Checkbox 联动规则：区域下所有 SE 已选 → 全选态；部分已选 → indeterminate；全未选 → 未选
- 顶部搜索框：同时过滤区域名和 SE 姓名，过滤时保持树形层级
- 顶部 `Select All` Checkbox：选中/取消所有区域的所有 SE
- 底部 [Add Selected] 按钮：显示当前已勾选 SE 总数（去重），如 `Add Selected (5)`；[Clear] 链接清空所有勾选
- 点击 [Add Selected]：收集所有已勾选 SE 的并集（无论来自整区域勾选还是单 SE 勾选），向父组件 `<AssignSEDialog>` 发出 SE ID 列表，由父组件执行与已 Assigned 列表的去重合并（幂等）

---

## 5. 状态管理

### 5.1 状态分层

| 状态类型 | 存放位置 | 例子 |
|---------|---------|------|
| 服务端缓存 | Pinia（手动失效）或 VueQuery | 区域列表、区域 SE 列表、项目 SE 列表 |
| 全局 UI | 无（本需求无跨页状态） | — |
| 路由/筛选 | URL query string | 区域列表搜索词 `?q=` |
| 表单临时 | 组件内 `ref<>` | AreaFormDialog 表单值 |
| 弹框本地暂存 | 组件内 `ref<Set<string>>` | ManageSEsDialog 待 Add/Remove 的 SE ID 集合；AssignSEDialog Assigned 列表 |

### 5.2 缓存 Key 约定

```ts
export const drawingAreaKeys = {
  all: ['drawingArea'] as const,
  lists: (projectId: string) => [...drawingAreaKeys.all, 'list', projectId] as const,
  seList: (areaId: string) => [...drawingAreaKeys.all, 'se', areaId] as const,
}
```

### 5.3 缓存失效策略

| 操作 | 失效的 key |
|-----|-----------|
| 新建 / 编辑 / 删除区域 | `drawingAreaKeys.lists(projectId)` |
| 保存区域 SE 绑定 | `drawingAreaKeys.seList(areaId)`、`drawingAreaKeys.lists(projectId)`（SE Count 更新） |

---

## 6. API 接入

> 所有接口字段定义参见 `data-contract-003G-pc.md`，本节只列前端调用方式。

### 6.1 接口客户端生成

- **方式**：从 OpenAPI yaml 用 `openapi-typescript` 生成 TS 类型
- **位置**：`src/api/generated/`
- **更新**：data-contract 变更时，CI 自动重新生成，类型不一致则 build 失败

### 6.2 调用清单

| API ID | 功能 | 在哪些组件中调用 | 缓存策略 |
|--------|------|---------------|---------|
| API-003G-01 | 获取项目区域列表 | `<AreaConfigPage>`、`<SelectByAreaDropdown>` | 按需拉取，操作后手动失效 |
| API-003G-02 | 新建区域 | `<AreaFormDialog mode="create">` | 操作后失效区域列表缓存 |
| API-003G-03 | 编辑区域名称 | `<AreaFormDialog mode="edit">` | 操作后失效区域列表缓存 |
| API-003G-04 | 删除区域 | `<DeleteAreaDialog>` | 操作后失效区域列表缓存 |
| API-003G-05 | 获取区域 SE 列表（In Area） | `<ManageSEsDialog>` | 打开弹框时拉取 |
| API-003G-06 | 获取项目全部 SE 列表（Available） | `<ManageSEsDialog>` | 打开弹框时拉取（可复用项目成员缓存） |
| API-003G-07 | 保存区域 SE 绑定（批量 diff） | `<ManageSEsDialog>` | 操作后失效 seList 缓存 |
| API-003G-08 | 按区域获取 SE 列表（树形面板数据，含每区域 SE 详情） | `<SelectByAreaDropdown>` | 打开下拉时按需拉取 |

### 6.3 错误处理

```ts
// 全局错误映射（用户可见文案）
const errorMessages: Record<string, string> = {
  'AREA_NAME_DUPLICATE': 'Area name already exists.',
  'AREA_NOT_FOUND': 'Area not found.',
  'PERMISSION_DENIED': 'You do not have permission to perform this action.',
}
```

- **409 AREA_NAME_DUPLICATE**：`<AreaFormDialog>` 输入框下方 inline 错误提示。
- **403**：Toast 提示权限不足。
- **500 / 网络错误**：Toast 通用错误提示，弹框/页面保留（不关闭）。

---

## 7. 性能预算

| 指标 | 目标 | 测量 |
|-----|------|-----|
| 区域配置列表页 FCP | ≤ 1s | Lighthouse / Chrome DevTools |
| Manage SEs 弹框打开（含 SE 列表渲染） | ≤ 1s | 手动 |
| [Select by Area] 下拉区域列表渲染 | ≤ 500ms | 手动 |
| 批量 Add 区域 SE（本地操作，无接口） | 即时（< 16ms 一帧内） | 手动 |

**优化手段**：
- 区域列表一次性加载（单项目 ≤ 100 区域，无需分页）。
- Manage SEs 弹框 SE 列表使用虚拟列表（`el-virtual-list`）应对大量 SE（> 200）。
- [Select by Area] 区域列表随 Assign 弹框打开时懒加载（首次点击 [Select by Area ▾] 时请求）。

---

## 8. 弱网与离线策略

| 场景 | 策略 |
|-----|------|
| 请求超时（区域列表加载） | Axios timeout 10s，超时后 Toast 提示，显示重试按钮 |
| 保存区域 / SE 绑定中断网 | 操作失败 Toast 提示，弹框不关闭，用户可重试 |
| 完全离线 | 不支持离线操作；显示网络错误提示 |

---

## 9. 错误边界与降级

- `<AreaConfigPage>` 包裹 Vue ErrorBoundary（全局 `errorCaptured`），接口失败时展示空状态 + 重试按钮，不崩溃整页。
- `<ManageSEsDialog>` 和 `<AssignSEDialog>` 内部接口失败时 Toast 提示，弹框保留。
- 错误上报：通过全局 Sentry `captureException`。

---

## 10. 埋点设计

对应需求 §9.5 与 §15 成功指标。

| 事件名 | 触发时机 | 关键属性 | 关联指标 |
|-------|---------|---------|---------|
| `drawing_area_config_enter` | 进入区域配置页 | `projectId` | 功能使用率 |
| `drawing_area_created` | 新建区域成功 | `projectId`, `areaId` | — |
| `drawing_area_create_failed` | 新建区域失败 | `projectId`, `errorCode` | — |
| `drawing_area_deleted` | 删除区域成功 | `projectId`, `areaId` | — |
| `drawing_area_se_saved` | 区域 SE 绑定保存成功 | `projectId`, `areaId`, `seCount` | — |
| `drawing_area_se_save_failed` | 区域 SE 绑定保存失败 | `projectId`, `areaId`, `errorCode` | — |
| `drawing_assign_select_by_area_open` | 点击 [Select by Area ▾] 打开下拉 | `projectId`, `drawingId` | 按区域批量分配使用率 |
| `drawing_assign_batch_add_by_area` | 按区域批量 Add SE 完成 | `projectId`, `drawingId`, `selectedAreaCount`, `addedSECount` | 批量分配使用率、效率提升 |

---

## 11. 国际化

- i18n 库：vue-i18n
- 文案来源：ui-spec-003G-pc.md §9
- 当前支持：zh-CN、en-US（双语并行）
- 所有硬编码字符串均不允许直接写在模板中，必须走 `$t('key')` 引用

**新增 i18n Key 示例**：
```
drawing.areaConfig.title = "Area Configuration"
drawing.areaConfig.addArea = "+ Add Area"
drawing.areaConfig.searchPlaceholder = "Search area name"
drawing.areaConfig.emptyState = "No areas configured. Click [+ Add Area] to get started."
drawing.areaConfig.seCount = "{n} SEs"
drawing.areaForm.addTitle = "Add New Area"
drawing.areaForm.editTitle = "Edit Area"
drawing.areaForm.namePlaceholder = "e.g. Zone A, Basement"
drawing.areaForm.nameDuplicate = "Area name already exists."
drawing.areaForm.savedSuccess = "Area saved successfully."
drawing.deleteArea.title = "Delete Area"
drawing.deleteArea.content = 'Are you sure you want to delete "{name}"? All SE bindings in this area will be removed. This action cannot be undone.'
drawing.deleteArea.success = "Area deleted."
drawing.manageSE.title = "Manage SEs — {areaName}"
drawing.manageSE.inArea = "In Area ({n})"
drawing.manageSE.available = "Available ({n})"
drawing.manageSE.savedSuccess = "Area SE configuration saved."
drawing.manageSE.inAreaEmpty = "No SEs in this area."
drawing.manageSE.availableEmpty = "All SEs have been added to this area."
drawing.assign.selectByArea = "Select by Area"
drawing.assign.addSelected = "Add Selected ({n})"
drawing.assign.clearSelection = "Clear"
drawing.assign.selectAll = "Select All"
drawing.assign.noAreaDisabledTooltip = "No areas configured. Go to Area Config to set up."
drawing.assign.noAreas = "No areas configured."
drawing.assign.noResults = "No results found."
```

---

## 12. 测试要求

| 层级 | 框架 | 覆盖范围 | 覆盖率目标 |
|-----|------|---------|-----------|
| 单元 | Vitest + Vue Test Utils | 组件逻辑、权限控制、本地状态（幂等合并、去重） | ≥ 80% |
| 集成 | Vitest + MSW（Mock Service Worker） | 接口调用、错误处理、弹框交互流程 | 核心流程 100% |
| E2E | Cypress | 区域 CRUD 主流程、按区域批量 Add SE 主流程、双模式混合使用 | P0 AC 全覆盖 |

> ⚠️ E2E 测试代码归 QA 维护，前端负责单元和集成。

---

## 13. AC 覆盖检查表

| AC ID | 实现位置（文件/组件） | 测试覆盖 | 状态 |
|------|------------------|---------|------|
| AC-003G-001 | `<AreaConfigEntryButton>` + `usePermission` | 单元 | TODO |
| AC-003G-002 | `<AreaFormDialog mode="create">` | 单元 + 集成 | TODO |
| AC-003G-003 | `<AreaFormDialog>` 409 错误处理 | 单元 + 集成 | TODO |
| AC-003G-004 | `<DeleteAreaDialog>` | 单元 + 集成 | TODO |
| AC-003G-005 | `<ManageSEsDialog>` | 单元 + 集成 | TODO |
| AC-003G-006 | `<SelectByAreaButton disabled>` + Tooltip | 单元 | TODO |
| AC-003G-007 | `<AssignSEDialog>` 整区域勾选批量合并逻辑 | 单元 | TODO |
| AC-003G-008 | `<AssignSEDialog>` 幂等去重逻辑 | 单元 | TODO |
| AC-003G-009 | `<AssignSEDialog>` 单选与批量混合 | 单元 + E2E | TODO |
| AC-003G-010a | `<AssignSEDialog>` 混合操作端到端 | E2E | TODO |
| AC-003G-010b | `<AreaConfigPage>` 空状态渲染 | 单元 | TODO |
| AC-003G-011 | `<SelectByAreaDropdown>` 单独勾选 SE 子节点 | 单元 | TODO |
| AC-003G-012 | `<SelectByAreaDropdown>` 整区域与单 SE 混合勾选 | 单元 + E2E | TODO |
| AC-003G-013 | `<SelectByAreaDropdown>` 区域行 indeterminate 态联动 | 单元 | TODO |

---

## 14. 与后端的联调约定

- **联调前置**：data-contract-003G-pc.md 已锁版
- **Mock 服务**：基于 data-contract 使用 MSW 自动生成，前端可独立开发
- **联调环境**：dev 环境
- **接口变更**：任何变更必须先改 data-contract，再通知前后端
- **重点联调项**：
  - API-003G-08（按区域批量获取 SE）的响应速度和数据去重处理
  - API-003G-07（保存区域 SE 绑定）的 diff 接口契约（传增量 or 全量）

---

## 15. 验收条件

前端开发完成的判定：

- [ ] 所有 AC 在 §13 表中标记完成
- [ ] 单元 + 集成测试通过，覆盖率达标
- [ ] E2E 测试由 QA 跑通（不阻塞前端提测）
- [ ] 区域配置列表页 Lighthouse 性能分数 ≥ 90
- [ ] 无 console 错误/警告
- [ ] 无障碍自动化检查无 critical 项（键盘导航、Tab/Enter/Esc、Checkbox 键盘多选）
- [ ] 已部署到 dev 环境，可联调
- [ ] 中英双语文案均已录入 i18n，无硬编码字符串

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-28 | agent | 初稿，覆盖 F-001～F-006 全部功能 |
| 0.2.0 | 2026-05-28 | agent | 同步 REQ-003G-pc v0.3.0：[Select by Area] 下拉面板升级为树形结构，支持整区域勾选与单 SE 勾选；新增 AC-003G-011/012/013 覆盖；更新 i18n Key；更新组件职责描述 |
