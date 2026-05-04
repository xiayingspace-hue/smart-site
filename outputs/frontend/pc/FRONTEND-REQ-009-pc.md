# FRONTEND-REQ-009-pc 前端开发说明文档

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

## 1. 需求来源
- 对应需求文档：REQ-009-pc《项目层级字典与工序配置副本机制》

## 2. 页面与路由
- 入口：项目管理后台 → 左侧导航 Progress Management > Process Settings
- 页面路由建议：`/project/:projectId/progress/process-settings`

## 3. 页面结构与组件

### 3.1 Progress Management 二级菜单
- 左侧导航栏 Progress Management 下显示两个子菜单：
  - Dictionary Management（点击显示空白页，已实现，无需处理）
  - Process Settings（点击进入工序模板配置页）

### 3.2 Process Settings 页面
- 页面结构与公司层级 Process Settings 完全一致（见 FRONTEND-REQ-008-pc）：
  - 顶部专业Tab栏（动态渲染，来自项目字典）
  - 左侧构件类型列表
  - 右侧工序表格（支持新增、编辑、删除、禁用、排序、批量新增）
- 项目层级为独立副本，修改不影响公司层级和其他项目

### 3.3 "从公司层级复制"按钮
- 位置：页面顶部或右上角操作区
- 按钮文案建议："从公司层级复制"
- 交互：
  1. 点击弹出确认弹窗，说明复制内容（字典+工序模板），提示将覆盖当前项目配置
  2. 用户确认后，调用复制接口，前端显示加载状态
  3. 复制成功后，页面数据刷新，Toast 提示"复制成功"
  4. 复制失败时，Toast 提示错误原因
- 权限控制：仅项目管理员可见此按钮

## 4. 权限控制
- 项目管理员：可复制、新增、编辑、删除、启用/停用工序
- Planning/PM：仅可查看，操作按钮不可见或置灰

## 5. 接口依赖
- GET `/api/project/:projectId/process-template/specialties`：获取项目专业Tab列表
- GET `/api/project/:projectId/process-template/components?specialtyId=`：获取项目构件类型列表
- GET `/api/project/:projectId/process-template/processes?componentId=`：获取项目工序列表
- POST `/api/project/:projectId/process-template/processes`：新增工序
- PUT `/api/project/:projectId/process-template/processes/:id`：编辑工序
- DELETE `/api/project/:projectId/process-template/processes/:id`：删除工序
- PUT `/api/project/:projectId/process-template/processes/:id/status`：启用/禁用工序
- POST `/api/project/:projectId/process-template/copy-from-company`：从公司层级复制

## 6. 重点注意事项
- 项目层级页面结构与公司层级保持一致，可复用公司层级组件
- "从公司层级复制"操作需有明确确认弹窗，防止误操作
- 复制后页面数据需自动刷新
- 权限控制需区分项目管理员与其他角色

---
> 本文档配合REQ-009-pc需求使用。