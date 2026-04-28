# BACKEND-REQ-012.md

## CM 任务管理与工序任务后端开发说明

> 对应需求：REQ-012（CM 端）/ REQ-013（SE 端）
> 说明：REQ-012 与 REQ-013 共享同一套数据模型和核心接口，后端统一在此文档描述

---

### 1. 功能概述

- 为 CM 提供任务列表查询、工序任务创建/编辑接口
- 为 SE 提供工序任务查询、标记完成接口
- 实现基于权重的任务进度自动计算逻辑
- 问题汇报接口及消息通知触发

---

### 2. 数据模型

#### 2.1 工序任务表（process_tasks）

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | BIGINT | 主键，自增 |
| parent_task_id | BIGINT | 关联上级任务 ID（cm_tasks 表） |
| name | VARCHAR(200) | 工序任务名称 |
| wbs | VARCHAR(100) | WBS 编号，可为空 |
| template_id | BIGINT | 引用的工序模板 ID，自定义时为 NULL |
| weight | TINYINT | 权重，范围 1～10，必填 |
| start_date | DATE | 计划开始日期 |
| end_date | DATE | 计划结束日期 |
| priority | ENUM('High','Medium','Low') | 优先级 |
| assigned_se_id | BIGINT | 分配的 SE 用户 ID |
| status | ENUM('pending','completed') | 完成状态，默认 pending |
| is_locked | TINYINT(1) | 是否锁定，0=否，1=是（SE 标记完成后置 1） |
| remark | TEXT | 备注 |
| created_by | BIGINT | 创建人（CM 用户 ID） |
| created_at | DATETIME | 创建时间 |
| updated_at | DATETIME | 更新时间 |
| deleted_at | DATETIME | 软删除时间，NULL 表示未删除 |

**索引：**
- `idx_parent_task_id`：按上级任务查询工序列表
- `idx_assigned_se_id`：SE 查询自己的工序任务

---

#### 2.2 工序模板表（process_task_templates）

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | BIGINT | 主键 |
| name | VARCHAR(200) | 模板名称 |
| specialty | VARCHAR(100) | 专业分类 |
| process_type | VARCHAR(100) | 工序类型 |
| area | VARCHAR(100) | 区域 |
| default_weight | TINYINT | 默认权重，1～10 |
| created_at | DATETIME | 创建时间 |

---

#### 2.3 问题汇报表（issue_reports）

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | BIGINT | 主键 |
| task_id | BIGINT | 关联任务 ID |
| process_task_id | BIGINT | 关联工序任务 ID（SE 上报时填入，CM 上报时为 NULL） |
| reporter_id | BIGINT | 上报人用户 ID |
| reporter_role | ENUM('CM','SE') | 上报角色 |
| issue_type | VARCHAR(100) | 问题类型 |
| description | TEXT | 问题描述 |
| impact | TEXT | 影响范围 |
| suggestion | TEXT | 建议方案 |
| attachments | JSON | 附件 URL 数组 |
| status | ENUM('pending','processing','resolved') | 处理状态 |
| created_at | DATETIME | 上报时间 |
| updated_at | DATETIME | 更新时间 |

---

### 3. 进度计算逻辑

**计算公式：**

$$
任务进度 = \frac{\sum(\text{已完成工序的 weight})}{\sum(\text{全部工序的 weight})} \times 100\%
$$

**触发时机：**
- SE 调用标记完成接口后，服务端自动重新计算并更新上级任务的 `progress` 字段
- CM 修改工序任务权重后，同样触发重新计算

**实现伪代码：**
```
function recalculateTaskProgress(parentTaskId):
  tasks = SELECT * FROM process_tasks WHERE parent_task_id = parentTaskId AND deleted_at IS NULL
  totalWeight = SUM(tasks.weight)
  completedWeight = SUM(tasks.weight WHERE status = 'completed')
  progress = totalWeight > 0 ? ROUND(completedWeight / totalWeight * 100) : 0
  UPDATE cm_tasks SET progress = progress WHERE id = parentTaskId
```

**注意事项：**
- 权重为整型，进度四舍五入取整数百分比
- 若任务下无工序任务，进度为 0
- 进度字段由系统写入，不接受客户端直接传值

---

### 4. API 接口设计

---

#### 4.1 CM 任务列表

**GET** `/api/cm/tasks`

权限：仅 CM 角色

Query 参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| status | string | 任务状态筛选（可多选，逗号分隔） |
| wbs | string | WBS 模糊搜索 |
| startDateFrom | date | 计划开始时间起 |
| startDateTo | date | 计划开始时间止 |
| page | int | 页码，默认 1 |
| pageSize | int | 每页条数，默认 20 |

Response:
```json
{
  "code": 0,
  "data": {
    "list": [
      {
        "id": 1,
        "activityId": "ACT-001",
        "activityName": "基础施工",
        "wbs": "1.2.3",
        "plannedStart": "2025-06-01",
        "plannedEnd": "2025-06-30",
        "actualStart": null,
        "actualEnd": null,
        "deviation": 0,
        "status": "In Progress",
        "priority": "High",
        "progress": 80
      }
    ],
    "total": 100
  }
}
```

---

#### 4.2 CM 任务详情

**GET** `/api/cm/tasks/:id`

权限：仅 CM 角色，且 assigned_cm_id = 当前用户

---

#### 4.3 工序模板列表

**GET** `/api/process-task-templates`

权限：CM 角色

Query 参数：specialty、processType、area（可选筛选）

---

#### 4.4 批量创建工序任务

**POST** `/api/process-tasks/batch`

权限：CM 角色

Request Body:
```json
{
  "parentTaskId": 1,
  "tasks": [
    {
      "name": "土方开挖",
      "wbs": "1.2.3.1",
      "templateId": 10,
      "weight": 3,
      "startDate": "2025-06-01",
      "endDate": "2025-06-05",
      "priority": "High",
      "assignedSeId": 101,
      "remark": ""
    }
  ]
}
```

校验：
- weight 必须在 1～10 之间
- assignedSeId 必须是有效的 SE 用户
- endDate 不早于 startDate

成功后：
- 为每个 SE 发送"新工序任务分配"消息通知
- 触发上级任务进度重新计算

---

#### 4.5 更新工序任务

**PUT** `/api/process-tasks/:id`

权限：CM 角色，且工序任务 is_locked = 0

Request Body: 与创建接口字段一致（单条）

校验：
- 若 is_locked = 1（SE 已标记完成），返回 403，提示"该工序已锁定，无法编辑"

---

#### 4.6 SE 工序任务列表

**GET** `/api/se/my-tasks`

权限：SE 角色

Query 参数：status（pending/completed/空）、priority、startDateFrom、startDateTo、page、pageSize

Response: 仅返回 assigned_se_id = 当前用户的工序任务

---

#### 4.7 SE 工序任务详情

**GET** `/api/se/process-tasks/:id`

权限：SE 角色，且 assigned_se_id = 当前用户

---

#### 4.8 SE 标记工序任务完成

**POST** `/api/se/process-tasks/:id/complete`

权限：SE 角色，且 assigned_se_id = 当前用户

校验：
- 工序任务当前 status 为 pending
- is_locked = 0

操作：
1. 更新 status = 'completed'，is_locked = 1
2. 记录完成时间
3. 触发上级任务进度重新计算
4. 发送通知给 CM（"工序 xxx 已完成"）

Response:
```json
{ "code": 0, "message": "标记完成成功", "data": { "progress": 80 } }
```

---

#### 4.9 获取角标数量

**GET** `/api/se/my-tasks/badge-count`

权限：SE 角色

Response:
```json
{ "code": 0, "data": { "unfinished": 3, "unread": 1 } }
```

---

#### 4.10 提交问题汇报

**POST** `/api/issue-reports`

权限：CM 或 SE 角色

Request Body:
```json
{
  "taskId": 1,
  "processTaskId": null,
  "issueType": "进度延误",
  "description": "因天气原因...",
  "impact": "延误 3 天",
  "suggestion": "建议延长工期",
  "attachments": ["https://..."]
}
```

成功后：
- CM 汇报 → 发送通知给对应 PM
- SE 汇报 → 发送通知给对应 CM

---

#### 4.11 问题汇报列表

**GET** `/api/issue-reports`

Query 参数：taskId（必填）

权限：任务相关 CM / PM 可查看

---

### 5. 非功能需求

- **性能：** 任务列表接口响应时间 < 500ms；进度重新计算须在同一事务内完成
- **安全：** 所有接口需 JWT 认证；SE 只能操作自己的工序任务，CM 只能操作自己负责任务下的工序
- **数据一致性：** 标记完成与进度更新须在同一数据库事务中完成，避免进度不一致
- **扩展性：** process_tasks 表预留 `ext_json` JSON 字段，供后续迭代扩展

---

### 6. 验收条件

- [ ] CM 任务列表接口返回数据正确，支持所有筛选参数
- [ ] 批量创建工序任务时权重校验生效（1～10，必填）
- [ ] SE 标记完成后 is_locked = 1，进度自动重新计算并写入上级任务
- [ ] 权限校验：SE 只能看到/操作自己的工序任务，CM 只能编辑未锁定工序
- [ ] 消息通知在对应操作完成后正确发送
- [ ] 进度计算公式正确：已完成权重之和 / 全部权重之和 × 100%（四舍五入整数）
