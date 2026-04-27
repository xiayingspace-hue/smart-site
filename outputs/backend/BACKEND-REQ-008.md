# BACKEND-REQ-008-pc 后端开发说明文档

## 1. 需求来源
- 对应需求文档：REQ-008-pc《公司层级工序模板配置》

## 2. 数据模型

### 2.1 工序模板主表 `company_process_template`
| 字段         | 类型        | 说明               |
|--------------|-------------|--------------------|
| id           | bigint      | 主键               |
| specialty_id | bigint      | 所属专业（字典）   |
| component_id | bigint      | 所属构件类型（字典）|
| name         | varchar     | 工序名称           |
| weight       | decimal     | 权重（0-100）      |
| std_duration | int         | 标准工期（天）     |
| is_required  | tinyint     | 是否必选（0/1）    |
| sort_order   | int         | 排序               |
| status       | tinyint     | 状态（0禁用/1启用）|
| created_by   | bigint      | 创建人             |
| created_at   | datetime    | 创建时间           |
| updated_at   | datetime    | 更新时间           |

### 2.2 引用关系表 `project_process_template_ref`
| 字段                   | 类型     | 说明               |
|------------------------|----------|--------------------|
| id                     | bigint   | 主键               |
| company_process_id     | bigint   | 公司级工序ID       |
| project_id             | bigint   | 项目ID             |
| used_in_task           | tinyint  | 是否已用于实际任务 |
| has_progress_history   | tinyint  | 是否有进度历史     |

## 3. 主要接口

### 3.1 获取专业Tab列表
- `GET /api/company/process-template/specialties`
- 返回：字典专业列表（来自公司字典表）

### 3.2 获取构件类型列表
- `GET /api/company/process-template/components?specialtyId={id}`
- 返回：当前专业下的构件类型列表

### 3.3 获取工序列表
- `GET /api/company/process-template/processes?componentId={id}`
- 返回：工序列表，含引用状态（is_referenced、used_in_task、has_progress_history）

### 3.4 新增工序
- `POST /api/company/process-template/processes`
- 请求体：name、weight、std_duration、is_required、sort_order、specialty_id、component_id
- 校验：名称不可重复（同一构件下）

### 3.5 编辑工序
- `PUT /api/company/process-template/processes/:id`
- 请求体：可修改字段
- 校验：禁用状态工序不可编辑

### 3.6 删除工序
- `DELETE /api/company/process-template/processes/:id`
- 业务校验：
  - 若已被实际任务使用或有进度历史，返回错误，禁止删除
  - 若已同步到项目模板但未生成实际任务，允许删除，同步清理项目级引用，需事务处理
  - 若未被引用，直接删除

### 3.7 启用/禁用工序
- `PUT /api/company/process-template/processes/:id/status`
- 请求体：status（0/1）
- 禁用后，新项目/新任务不可引用

### 3.8 批量新增工序
- `POST /api/company/process-template/processes/batch`
- 请求体：工序数组
- 批量校验、批量插入，事务处理

## 4. 权限控制
- 企业管理员：所有接口可访问
- Planning/PM：仅允许GET（查询）接口

## 5. 重点注意事项
- 删除/禁用操作需严格校验引用关系，防止误删历史数据
- 所有写操作需记录操作日志（操作人、时间、变更内容）
- 接口返回需包含工序引用状态，供前端渲染按钮
- 批量操作需事务保证原子性

---
> 本文档配合REQ-008-pc需求使用。