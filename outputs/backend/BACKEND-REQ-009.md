# BACKEND-REQ-009-pc 后端开发说明文档

## 1. 需求来源
- 对应需求文档：REQ-009-pc《项目层级字典与工序配置副本机制》

## 2. 数据模型

### 2.1 项目级工序模板主表 `project_process_template`
| 字段              | 类型     | 说明                          |
|-------------------|----------|-------------------------------|
| id                | bigint   | 主键                          |
| project_id        | bigint   | 所属项目                      |
| source_company_id | bigint   | 来源公司级工序ID（复制来源，可空）|
| specialty_id      | bigint   | 所属专业（项目字典）          |
| component_id      | bigint   | 所属构件类型（项目字典）      |
| name              | varchar  | 工序名称                      |
| weight            | decimal  | 权重（0-100）                 |
| std_duration      | int      | 标准工期（天）                |
| is_required       | tinyint  | 是否必选（0/1）               |
| sort_order        | int      | 排序                          |
| status            | tinyint  | 状态（0禁用/1启用）           |
| created_by        | bigint   | 创建人                        |
| created_at        | datetime | 创建时间                      |
| updated_at        | datetime | 更新时间                      |

### 2.2 复制操作日志表 `process_template_copy_log`
| 字段         | 类型     | 说明               |
|--------------|----------|--------------------|
| id           | bigint   | 主键               |
| project_id   | bigint   | 目标项目           |
| operated_by  | bigint   | 操作人             |
| operated_at  | datetime | 操作时间           |
| remark       | varchar  | 备注               |

## 3. 主要接口

### 3.1 获取项目专业Tab列表
- `GET /api/project/:projectId/process-template/specialties`
- 返回：项目字典专业列表

### 3.2 获取项目构件类型列表
- `GET /api/project/:projectId/process-template/components?specialtyId={id}`
- 返回：当前专业下构件类型列表

### 3.3 获取项目工序列表
- `GET /api/project/:projectId/process-template/processes?componentId={id}`
- 返回：工序列表，含引用状态

### 3.4 新增工序
- `POST /api/project/:projectId/process-template/processes`
- 请求体：name、weight、std_duration、is_required、sort_order、specialty_id、component_id
- 校验：同一构件下名称不可重复

### 3.5 编辑工序
- `PUT /api/project/:projectId/process-template/processes/:id`

### 3.6 删除工序
- `DELETE /api/project/:projectId/process-template/processes/:id`
- 业务校验：已用于实际任务或有进度历史，禁止删除，仅允许禁用

### 3.7 启用/禁用工序
- `PUT /api/project/:projectId/process-template/processes/:id/status`

### 3.8 从公司层级复制
- `POST /api/project/:projectId/process-template/copy-from-company`
- 逻辑：
  1. 校验操作人权限（项目管理员）
  2. 读取公司层级所有工序模板数据
  3. 事务内清除当前项目已有模板（或合并），批量写入项目级模板表
  4. 记录复制操作日志
  5. 返回成功/失败

## 4. 权限控制
- 项目管理员：所有接口可访问（含复制）
- Planning/PM：仅允许GET（查询）接口

## 5. 重点注意事项
- 复制操作为事务处理，保证原子性
- 复制操作需记录操作日志（操作人、时间）
- 项目级数据与公司级数据相互独立，修改互不影响
- 删除/禁用逻辑同REQ-008-pc

---
> 本文档配合REQ-009-pc需求使用。