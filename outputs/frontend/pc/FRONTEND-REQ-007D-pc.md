---
doc_type: frontend_spec
req_id: REQ-007D-pc
version: 0.1.0
status: draft
generated_from: REQ-007D-pc.md@0.1.0
ui_spec_ref: UI-REQ-007D-pc.md
generated_at: 2026-05-27
owner: ""
preferred_runtime_ui: "Element UI"
---

# 前端开发说明 — PC 端 DC 配置页面

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-007D-pc.md](../../../requirements/pc/REQ-007D-pc.md) @ v0.1.0
> **UI 设计参考**: [UI-REQ-007D-pc.md](../../ui/pc/UI-REQ-007D-pc.md)
> **依赖后端文档**: [BACKEND-REQ-007D.md](../../backend/BACKEND-REQ-007D.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-05-27

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007D-pc @ v0.1.0 |
| UI 设计 | UI-REQ-007D-pc.md |
| 覆盖 Story | US-007D-001 |
| 覆盖 AC | AC-007D-001 ~ AC-007D-010 |

---

## 1. 功能概述

本文档覆盖 DC 配置页面的全部前端实现：

| 功能模块 | 变更类型 | 说明 |
|---------|---------|------|
| DcConfigPage（F-001） | **新增** | DC 配置主页面：已配置 DC 表格 + 空状态警告横幅 |
| AddDcDialog（F-002） | **新增** | 添加 DC 弹窗：可搜索的 Available Users 列表 + 连续添加 |
| RemoveDcConfirmDialog（F-003） | **新增** | 移除 DC 二次确认弹窗 |
| 侧边栏菜单权限门控 | **变更** | `drawing:dc-config` 无权限时隐藏 DC Configuration 菜单项 |

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 |
| UI 库 | Element UI |
| HTTP 客户端 | Axios（封装在 `src/api/`） |
| 状态管理 | Vuex |
| 路由 | Vue Router |
| 构建 | Webpack / Vue CLI |

### 2.2 本需求无新增依赖

所有实现复用现有 Element UI 组件（`el-table`、`el-dialog`、`el-input`、`el-button`、`el-alert`）。

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── settings/
│       └── DcConfigPage.vue              # 新增：DC 配置主页面（F-001）
│           └── components/
│               ├── DcConfigTable.vue     # 新增：DC 列表表格子组件
│               ├── AddDcDialog.vue       # 新增：添加 DC 弹窗（F-002）
│               └── RemoveDcConfirmDialog.vue  # 新增：移除确认弹窗（F-003）
├── api/
│   └── dcConfig.js                       # 新增：DC 配置相关 API 封装
├── store/
│   └── modules/
│       └── dcConfig.js                   # 新增：DC 配置 Vuex module
└── router/
    └── settings.js                       # 变更：添加 /settings/dc-config 路由（权限守卫）
```

---

## 4. 路由配置

```js
// router/settings.js
{
  path: '/settings/dc-config',
  name: 'DcConfig',
  component: () => import('@/views/settings/DcConfigPage.vue'),
  meta: {
    permission: 'drawing:dc-config',   // AC-007D-001：无权限跳转 403
    title: 'DC Configuration'
  }
}
```

**权限门控规则**（AC-007D-001）：
- 侧边栏菜单渲染时检查 `hasPermission('drawing:dc-config')`，无权限则不渲染菜单项
- 路由守卫中校验权限，无权限时跳转至 `/403` 页面

---

## 5. API 封装

```js
// api/dcConfig.js
import request from '@/utils/request'

/**
 * 查询已配置 DC 列表
 * AC-007D-002
 */
export function getDcConfigList(projectId) {
  return request({
    url: `/project/${projectId}/dc-config`,
    method: 'get'
  })
}

/**
 * 查询可添加用户列表（有权限且未配置）
 * AC-007D-003
 */
export function getAvailableUsers(projectId, keyword = '') {
  return request({
    url: `/project/${projectId}/dc-config/available`,
    method: 'get',
    params: { keyword }
  })
}

/**
 * 添加 DC
 * AC-007D-004
 */
export function addDc(projectId, dcUserId) {
  return request({
    url: `/project/${projectId}/dc-config`,
    method: 'post',
    data: { dcUserId }
  })
}

/**
 * 移除 DC
 * AC-007D-007
 */
export function removeDc(projectId, dcUserId) {
  return request({
    url: `/project/${projectId}/dc-config/${dcUserId}`,
    method: 'delete'
  })
}
```

---

## 6. Vuex Store

```js
// store/modules/dcConfig.js
const state = {
  dcList: [],      // 已配置 DC 列表
  loading: false
}

const mutations = {
  SET_DC_LIST(state, list) { state.dcList = list },
  SET_LOADING(state, val) { state.loading = val },
  ADD_DC(state, item) { state.dcList.push(item) },
  REMOVE_DC(state, dcUserId) {
    state.dcList = state.dcList.filter(dc => dc.dcUserId !== dcUserId)
  }
}

const actions = {
  async fetchDcList({ commit }, projectId) {
    commit('SET_LOADING', true)
    try {
      const res = await getDcConfigList(projectId)
      commit('SET_DC_LIST', res.data || [])
    } finally {
      commit('SET_LOADING', false)
    }
  }
}
```

---

## 7. 组件实现规范

### 7.1 DcConfigPage.vue（F-001）

**职责**: DC 配置主页面，包含页面标题、操作按钮、DC 列表表格、空状态警告横幅。

**关键实现要点**:

```vue
<template>
  <div class="dc-config-page">
    <!-- 警告横幅：dcList 为空时显示 AC-007D-008 -->
    <el-alert
      v-if="dcList.length === 0 && !loading"
      type="warning"
      :closable="false"
      title="No DC configured. Internal approvals cannot proceed to external approval.
             Please add at least one Document Controller."
      show-icon
    />

    <div class="page-header">
      <span>DC Configuration</span>
      <el-button type="primary" @click="openAddDialog">+ Add DC</el-button>
    </div>

    <el-table :data="dcList" v-loading="loading">
      <el-table-column prop="dcUserName"      label="Name" />
      <el-table-column prop="configuredByName" label="Configured By" />
      <el-table-column prop="configuredAt"    label="Configured At" width="140" />
      <el-table-column label="Action" width="100">
        <template #default="{ row }">
          <el-button type="text" @click="onRemoveClick(row)">Remove</el-button>
        </template>
      </el-table-column>
    </el-table>

    <p v-if="dcList.length > 0" class="hint-text">
      ⓘ At least one DC is required for the drawing approval process.
    </p>

    <AddDcDialog ref="addDcDialog" @added="onDcAdded" />
    <RemoveDcConfirmDialog ref="removeDialog" @confirmed="onDcRemoved" />
  </div>
</template>
```

**生命周期**:
- `created`：调用 `fetchDcList(projectId)` 拉取列表

**事件处理**:

| 事件 | 行为 |
|-----|------|
| `openAddDialog` | 打开 AddDcDialog，传入当前 projectId |
| `onRemoveClick(row)` | 打开 RemoveDcConfirmDialog，传入 row.dcUserId、row.dcUserName |
| `onDcAdded(item)` | commit `ADD_DC(item)`，表格实时新增 |
| `onDcRemoved(dcUserId)` | commit `REMOVE_DC(dcUserId)`，表格实时移除 |

---

### 7.2 AddDcDialog.vue（F-002）

**职责**: 搜索并展示可添加用户，点击 [Add] 立即添加，弹窗保持打开支持连续添加。

**关键实现要点**:

```vue
<template>
  <el-dialog title="Add Document Controller" :visible.sync="visible" width="480px">

    <!-- 搜索框：实时过滤，不触发接口（AC-007D-009） -->
    <el-input
      v-model="keyword"
      placeholder="Search..."
      prefix-icon="el-icon-search"
      clearable
    />

    <!-- 可用用户列表 -->
    <div class="user-list" v-loading="listLoading">
      <div v-if="filteredUsers.length === 0" class="empty-text">
        All eligible users have been configured as DC.
      </div>
      <div v-for="user in filteredUsers" :key="user.userId" class="user-row">
        <span>{{ user.userName }}</span>
        <el-button
          size="small"
          type="primary"
          plain
          :loading="addingId === user.userId"
          @click="onAdd(user)"
        >Add</el-button>
      </div>
    </div>

    <template #footer>
      <el-button @click="close">Close</el-button>
    </template>
  </el-dialog>
</template>
```

**行为规范**:

1. **open()** 调用时，请求 `getAvailableUsers(projectId)`，加载 `availableUsers` 列表（AC-007D-003）
2. `keyword` 变化时，前端过滤 `availableUsers`（`userName.includes(keyword)`），**不重新请求接口**（AC-007D-009）
3. 点击 [Add]：
   - 该条目的 loading 设为 `true`（`addingId = user.userId`）
   - 调用 `addDc(projectId, user.userId)`
   - 成功：从 `availableUsers` 移除该用户（弹窗列表刷新），emit `added(newDcItem)`（AC-007D-004 / 005）
   - 失败：loading 恢复，Toast 报错
4. 弹窗**保持打开**，支持连续添加（AC-007D-005）
5. 点击 [Close] 关闭弹窗

---

### 7.3 RemoveDcConfirmDialog.vue（F-003）

**职责**: 移除 DC 前的二次确认弹窗，文案含被移除 DC 姓名。

```vue
<template>
  <el-dialog title="Remove DC" :visible.sync="visible" width="420px">
    <p>
      Remove <strong>{{ targetName }}</strong> from DC list?<br>
      They will no longer receive external approval tasks.
    </p>
    <template #footer>
      <el-button @click="cancel">Cancel</el-button>
      <el-button type="danger" :loading="removing" @click="confirm">Confirm</el-button>
    </template>
  </el-dialog>
</template>
```

**行为规范**:

1. **open(dcUserId, dcUserName)** 设置 `targetId` 和 `targetName`，显示弹窗（AC-007D-006）
2. 点击 [Confirm]：
   - `removing = true`
   - 调用 `removeDc(projectId, targetId)`
   - 成功：emit `confirmed(targetId)`，关闭弹窗（AC-007D-007）
   - 失败：`removing = false`，Toast 报错
3. 点击 [Cancel]：关闭弹窗，不提交

---

## 8. 验收标准映射

| AC ID | 实现要点 |
|-------|---------|
| AC-007D-001 | 路由守卫 + 侧边栏 `v-if="hasPermission('drawing:dc-config')"` |
| AC-007D-002 | DcConfigPage created 钩子调用 `fetchDcList`，表格显示 Name / Configured By / Configured At / [Remove] |
| AC-007D-003 | AddDcDialog open() 时调用 `getAvailableUsers`，仅显示有权限且未配置用户 |
| AC-007D-004 | 点击 [Add] 调用 `addDc`，成功后新用户从弹窗列表消失、主列表新增 |
| AC-007D-005 | AddDcDialog 不自动关闭，支持连续添加 |
| AC-007D-006 | RemoveDcConfirmDialog 文案包含 `dcUserName` |
| AC-007D-007 | [Confirm] 调用 `removeDc`，成功后主列表移除该条目 |
| AC-007D-008 | `dcList.length === 0` 时 `el-alert` 警告横幅显示 |
| AC-007D-009 | keyword 过滤在前端计算属性中完成，不触发额外接口请求 |
| AC-007D-010 | 集成验证（由后端 BACKEND-REQ-007D §5.1 覆盖，前端无直接对应实现） |

---

## 9. 错误处理规范

| 场景 | 前端行为 |
|-----|---------|
| 添加失败（目标用户无权限 1003007011） | Toast: "用户无 DC 权限" |
| 添加失败（已配置 1003007012） | Toast: "该用户已配置为 DC"，从弹窗列表移除该用户 |
| 移除失败（404） | Toast: "移除失败，请刷新后重试" |
| 列表加载失败 | 表格区域显示 "加载失败，请刷新页面" |

---

## 10. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-27 | agent | 从 REQ-007D-pc.md 初稿生成 |
