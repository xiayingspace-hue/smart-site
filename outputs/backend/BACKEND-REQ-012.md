# BACKEND-REQ-012.md

## CM"我的任务"后端说明

> 对应需求：REQ-012 / REQ-013
> 影响模块：CM 任务管理、工序任务 CRUD、工序任务状态流转、任务进度计算、问题上报

---

### 1. 数据库设计

#### 1.1 `process_tasks` 表（新增/变更）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | 主键 |
| task_id | BIGINT FK | 关联上级任务 ID |
| name | VARCHAR(200) | 工序名称 |
| wbs | VARCHAR(100) | WBS 编码（可空） |
| plan_start | DATE | 计划开始时间 |
| plan_end | DATE | 计划结束时间 |
| actual_start_at | DATETIME | 实际开始时间（SE 点击 Start Task 时由系统记录，可空） |
| actual_end_at | DATETIME | 实际结束时间（SE 点击 Mark as Complete 时由系统记录，可空） |
| priority | ENUM('high','medium','low') | 优先级 |
| weight | TINYINT | 权重，1～10 |
| assigned_se_id | BIGINT FK | 分配的 SE 用户 ID |
| status | ENUM('not_started','in_progress','completed') | 状态；默认 not_started |
| is_locked | TINYINT(1) | 0=可编辑，1=锁定（status=completed 时自动置 1） |
| remark | TEXT | 备注（可空） |
| created_by | BIGINT | 创建人 |
| created_at | DATETIME | 创建时间 |
| updated_at | DATETIME | 更新时间 |

> **状态流转（不可逆）：**
> `not_started` → `in_progress`（Start Task）→ `completed`（Mark as Complete）
> 状态变为 `completed` 时自动将 `is_locked` 置 1，后续所有修改接口对 is_locked=1 的记录返回 403

---

### 2. 接口清单

#### 2.1 获取 CM 任务列表

```
GET /api/cm/tasks
权限：CM
```

查询参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| status | string | 任务状态筛选 |
| wbs | string | WBS 筛选 |
| startDate | date | 计划开始时间范围-开始 |
| endDate | date | 计划开始时间范围-结束 |
| keyword | string | Activity Name 模糊搜索 |
| page | int | 页码（默认 1） |
| pageSize | int | 每页条数（默认 20） |

响应：
```json
{
  "code": 0,
  "data": {
    "list": [
      {
        "id": "ACT-001",
        "activityName": "基础施工",
        "wbs": "1.1.1",
        "planStart": "2024-06-01",
        "planEnd": "2024-06-30",
        "deviation": 2,
        "status": "in_progress",
        "priority": "high",
        "progress": 60
      }
    ],
    "total": 100
  }
}
```

---

#### 2.2 获取工序任务列表（任务详情页 Tab2 使用）

```
GET /api/cm/tasks/:taskId/process-tasks
权限：CM
```

响应字段含：`id, name, weight, planStart, planEnd, actualStartAt, actualEndAt, priority, assignedSe, status`

---

#### 2.3 创建工序任务（批量）

```
POST /api/cm/tasks/:taskId/process-tasks
权限：CM
```

请求体：
```json
{
  "processTasks": [
    {
      "name": "土方开挖",
      "wbs": "1.1.1.1",
      "planStart": "2024-06-01",
      "planEnd": "2024-06-05",
      "priority": "high",
      "weight": 3,
      "assignedSeId": 1001,
      "remark": ""
    }
  ]
}
```

校验规则：
- `taskId` 对应任务的 status 不能为 `completed` 或 `cancelled`，否则返回 403
- `weight` 必须在 1～10 之间
- `planEnd` 不早于 `planStart`

---

#### 2.4 更新工序任务

```
PUT /api/process-tasks/:id
权限：CM
```

- 若 `is_locked = 1`（status = completed），返回 403，message: "该工序已完成并锁定，不可修改"

---

#### 2.5 Start Task（SE 端）

```
POST /api/se/process-tasks/:id/start
权限：SE（仅分配给自己的工序任务）
```

处理逻辑：
1. 校验 `status = not_started`，否则返回 400，message: "当前状态不允许执行此操作"
2. 更新：`status = in_progress`，`actual_start_at = NOW()`
3. 返回更新后的 `actualStart`

响应：
```json
{
  "code": 0,
  "data": {
    "id": 2001,
    "status": "in_progress",
    "actualStart": "2024-06-02 09:30:00"
  }
}
```

---

#### 2.6 Mark as Complete（SE 端）

```
POST /api/se/process-tasks/:id/complete
权限：SE（仅分配给自己的工序任务）
```

处理逻辑：
1. 校验 `status = in_progress`，否则返回 400，message: "请先开始任务"
2. 更新：`status = completed`，`actual_end_at = NOW()`，`is_locked = 1`
3. 触发上级任务进度重新计算（异步或同步均可）
4. 返回更新后的 `actualEnd`

响应：
```json
{
  "code": 0,
  "data": {
    "id": 2001,
    "status": "completed",
    "actualEnd": "2024-06-05 17:00:00"
  }
}
```

---

#### 2.7 SE 获取工序任务列表（My Task）

```
GET /api/se/process-tasks
权限：SE（仅返回分配给当前 SE 的工序任务）
```

查询参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| status | string | not_started / in_progress / completed，不传返回全部 |
| keyword | string | 工序名称模糊搜索 |
| page | int | 页码 |
| pageSize | int | 每页条数 |

---

#### 2.8 SE 获取工序任务详情

```
GET /api/se/process-tasks/:id
权限：SE（仅限分配给自己的）
```

响应含：`id, name, parentTaskId, parentTaskName, planStart, planEnd, actualStartAt, actualEndAt, priority, weight, status, remark`

---

#### 2.9 汇报问题

```
POST /api/issues
权限：CM / SE
```

请求体：
```json
{
  "taskId": "ACT-001",
  "issueType": "safety",
  "description": "...",
  "affectedScope": "...",
  "suggestion": "...",
  "attachments": []
}
```

---

### 3. 任务进度计算

**公式：**

$$\text{进度} = \frac{\sum_{i \in \text{completed}} w_i}{\sum_{i \in \text{all}} w_i} \times 100\%$$

**触发时机：**
- SE 调用 `/complete` 接口成功后，重新计算上级任务进度并更新 `tasks.progress` 字段

**SQL 示例：**
```sql
SELECT
  ROUND(
    SUM(CASE WHEN status = 'completed' THEN weight ELSE 0 END) * 100.0
    / SUM(weight),
    0
  ) AS progress
FROM process_tasks
WHERE task_id = :taskId
  AND deleted_at IS NULL
```

---

### 4. 权限矩阵

| 接口 | PM | CM | SE |
|------|----|----|-----|
| GET /api/cm/tasks | ✅（只读） | ✅ | — |
| GET /api/cm/tasks/:id/process-tasks | ✅ | ✅ | — |
| POST /api/cm/tasks/:id/process-tasks | — | ✅ | — |
| PUT /api/process-tasks/:id | — | ✅ | — |
| POST /api/se/process-tasks/:id/start | — | — | ✅ |
| POST /api/se/process-tasks/:id/complete | — | — | ✅ |
| GET /api/se/process-tasks | — | — | ✅ |
| GET /api/se/process-tasks/:id | — | — | ✅ |
| POST /api/issues | — | ✅ | ✅ |

---

### 5. 错误码约定

| Code | HTTP | 说明 |
|------|------|------|
| 400 | 400 | 参数校验失败 / 状态不允许操作 |
| 403 | 403 | 无权限 / 记录已锁定 |
| 404 | 404 | 资源不存在 |
| 0 | 200 | 成功 |

---

### 6. 验收条件

- [ ] `process_tasks` 表含 `actual_start_at`、`actual_end_at` 字段
- [ ] `status` ENUM 为 `not_started` / `in_progress` / `completed`
- [ ] `/start` 接口：校验 not_started，写入 actual_start_at
- [ ] `/complete` 接口：校验 in_progress，写入 actual_end_at，is_locked=1
- [ ] PUT 接口：is_locked=1 时返回 403
- [ ] 进度计算仅统计 status=completed 的工序权重
- [ ] 创建工序任务时校验上级任务状态不为 completed/cancelled
