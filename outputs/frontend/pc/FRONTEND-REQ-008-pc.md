# FRONTEND-REQ-008-pc 前端开发说明文档

## 1. 需求来源
- 对应需求文档：REQ-008-pc《公司层级工序模板配置》

## 2. 页面与路由
- 入口：切换至Company Insights视角 → 左侧导航 Project Progress > Process Settings
- 页面路由建议：`/company-insights/project-progress/process-settings`

## 3. 页面结构与组件

### 3.1 顶部专业Tab栏
- 组件：`el-tabs`（Element UI）
- 数据来源：公司字典接口，动态渲染Tab项
- 交互：点击Tab切换当前专业，联动左侧构件列表和右侧工序表格
- 限制：Tab仅展示，不支持新增、编辑、删除

### 3.2 左侧构件类型列表
- 组件：`el-menu` 或自定义列表
- 数据来源：根据当前选中专业请求构件类型字典
- 交互：点击构件类型，右侧工序表格刷新
- 限制：仅展示，不支持新增、编辑、删除

### 3.3 右侧工序表格
- 组件：`el-table`
- 表格字段：

| 字段       | 类型   | 说明                        |
|------------|--------|-----------------------------|
| 序号       | 自动   | 排序用                      |
| 工序名称   | 文本   | 必填                        |
| 权重       | 数字   | 必填，需校验总和提示        |
| 标准工期   | 数字   | 单位：天                    |
| 可选/必选  | 开关   | 默认必选                    |
| 操作       | 按钮   | 编辑、删除、禁用等          |

- 支持行内编辑或弹窗编辑
- 支持拖拽排序（`sortablejs`）
- 支持批量添加工序（弹窗表单或多行录入）
- 权重输入校验：输入非数字或超出范围时提示，总和不等于100%时给出提示（非阻断）

### 3.4 删除/禁用按钮逻辑
- 工序未被引用：显示"删除"按钮
- 工序已被项目引用但未生成实际任务：显示"删除"按钮（点击后弹出确认提示，说明会同步移除项目级引用）
- 工序已被实际任务使用或有进度填报：隐藏"删除"按钮，仅显示"禁用"按钮
- 已禁用工序：显示"启用"按钮，行样式灰化
- 所有删除/禁用操作需二次确认弹窗

## 4. 权限控制
- 仅企业管理员可新增、编辑、删除、启用/停用工序
- Planning/PM：仅可查看，操作按钮不可见或置灰

## 5. 接口依赖
- GET `/api/company/process-template/specialties`：获取专业Tab列表（字典）
- GET `/api/company/process-template/components?specialtyId=`：获取构件类型列表（字典）
- GET `/api/company/process-template/processes?componentId=`：获取工序列表
- POST `/api/company/process-template/processes`：新增工序
- PUT `/api/company/process-template/processes/:id`：编辑工序
- DELETE `/api/company/process-template/processes/:id`：删除工序
- PUT `/api/company/process-template/processes/:id/status`：启用/禁用工序
- POST `/api/company/process-template/processes/batch`：批量新增工序

## 6. 重点注意事项
- 专业Tab、构件类型、工序三级联动
- 工序权重总和校验与提示
- 删除/禁用按钮状态需根据后端返回的引用状态动态渲染
- UI风格与系统整体保持一致（Element UI + Vue2）

---
> 本文档配合REQ-008-pc需求使用。