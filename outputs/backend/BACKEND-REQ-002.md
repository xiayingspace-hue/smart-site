# 后端开发说明文档 — REQ-002 设备加油记录管理

> **来源需求**: [REQ-002-shared.md](../../requirements/shared/REQ-002-shared.md) + [REQ-002-pc.md](../../requirements/pc/REQ-002-pc.md) + [REQ-002-app.md](../../requirements/app/REQ-002-app.md)
> **产品**: SMART SITE SYSTEM
> **服务模块**: `equipment-service`
> **生成日期**: 2026-04-08

---

## 1. 功能概述

实现工地设备加油记录的全生命周期管理，支持 PC 端管理后台和 APP 端操作员录入：

| 功能 | 说明 |
|------|------|
| 加油记录 CRUD | 新增、修改、删除、查询（分页）|
| 图片上传 | 每条记录最多 5 张小票照片 |
| APP 24h 编辑窗口 | 操作员仅能修改 24h 内自己的记录 |
| 统计分析 | 按日/周/月/设备/分包商统计汇总 |
| 导出 Excel | 导出当前过滤条件下所有记录 |

---

## 2. 业务逻辑

### 2.1 权限规则

| 操作 | PC（管理员）| APP（操作员）|
|------|-----------|------------|
| 查看列表 | 全部记录 | 仅自己记录 |
| 创建 | ✅ | ✅ |
| 修改 | ✅ 任意记录 | ✅ 仅自己 + 24h 内 |
| 删除 | ✅ 任意记录 | ❌ |
| 统计/导出 | ✅ | ❌ |

权限码：`equipment:refueling:query`、`equipment:refueling:create`、`equipment:refueling:update`、`equipment:refueling:delete`、`equipment:refueling:export`、`equipment:refueling:statistics`

### 2.2 APP 24h 编辑规则

修改前端请求时后端校验：
```
createdAt + 24h > now  &&  createdBy == currentUserId
```
不满足则返回错误码 `1002003001`（超过编辑时限）。

### 2.3 图片上传

- 使用统一文件服务上传，返回 fileUrl
- receiptPhotos 存储为 JSON 数组（最多 5 个 URL）
- 前端在提交加油记录前先完成图片上传

### 2.4 统计逻辑

- `GET /equipment/refueling/statistics` 支持维度：`DAY`、`WEEK`、`MONTH`、`EQUIPMENT`、`SUBCONTRACTOR`
- 统计字段：总加油量（L）、总费用（元）、记录条数
- 支持 startDate / endDate 过滤
- 返回排名列表（Top N 设备 / 分包商）

### 2.5 导出

- 使用 EasyExcel 生成 `.xlsx`
- 导出全部符合当前过滤条件的记录（不分页）
- 文件名：`Refueling_Records_{projectCode}_{yyyyMMdd}.xlsx`
- 响应 `Content-Disposition: attachment; filename=...`

---

## 3. 数据模型

### 3.1 equipment_refueling_record 表

```sql
CREATE TABLE `equipment_refueling_record` (
  `id`                BIGINT PRIMARY KEY AUTO_INCREMENT,
  `tenant_id`         BIGINT NOT NULL,
  `project_id`        BIGINT NOT NULL,
  `equipment_id`      BIGINT NOT NULL COMMENT '设备 ID，关联 equipment 表',
  `equipment_name`    VARCHAR(200) COMMENT '冗余字段，设备名称',
  `subcontractor_id`  BIGINT COMMENT '分包商 ID',
  `subcontractor_name` VARCHAR(200) COMMENT '冗余字段',
  `fuel_type`         VARCHAR(50) NOT NULL COMMENT '燃料类型：DIESEL / GASOLINE / OTHER',
  `fuel_amount`       DECIMAL(10,2) NOT NULL COMMENT '加油量（升）',
  `unit_price`        DECIMAL(10,4) COMMENT '单价（元/L）',
  `total_cost`        DECIMAL(12,2) COMMENT '总费用（元）',
  `meter_reading`     DECIMAL(12,2) COMMENT '表盘读数',
  `refueling_method`  VARCHAR(50) COMMENT 'EXTERNAL_STATION / TANKER / OTHER',
  `refueling_time`    DATETIME NOT NULL COMMENT '加油时间',
  `receipt_photos`    JSON COMMENT '小票照片 URL 数组，最多 5 张',
  `remark`            TEXT,
  `created_by`        BIGINT NOT NULL COMMENT '创建人 userId',
  `created_by_name`   VARCHAR(64),
  `created_at`        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at`        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`           TINYINT(1) NOT NULL DEFAULT 0,
  KEY `idx_project_time` (`project_id`, `refueling_time`),
  KEY `idx_equipment` (`equipment_id`),
  KEY `idx_created_by` (`created_by`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='设备加油记录';
```

---

## 4. API 设计

### 4.1 接口列表

- [x] `POST /equipment/refueling/create` — 新增记录
- [x] `PUT /equipment/refueling/update` — 修改记录
- [x] `DELETE /equipment/refueling/delete` — 删除记录
- [x] `GET /equipment/refueling/page` — 分页查询
- [x] `GET /equipment/refueling/get` — 获取单条
- [x] `GET /equipment/refueling/statistics` — 统计数据
- [x] `GET /equipment/refueling/export` — 导出 Excel

### 4.2 接口规范

#### POST /equipment/refueling/create

**Headers**: `Authorization`, `X-Tenant-Id`, `Project-Id`

**Request Body**:
```json
{
  "equipmentId": 101,
  "subcontractorId": 20,
  "fuelType": "DIESEL",
  "fuelAmount": 120.5,
  "unitPrice": 7.86,
  "totalCost": 947.23,
  "meterReading": 23456.7,
  "refuelingMethod": "EXTERNAL_STATION",
  "refuelingTime": "2026-04-08T10:30:00",
  "receiptPhotos": ["https://oss.example.com/r1.jpg"],
  "remark": ""
}
```

#### GET /equipment/refueling/page

**Query Params**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `pageNo` | int | 页码（默认 1）|
| `pageSize` | int | 每页条数（默认 20，最大 100）|
| `keyword` | string | 设备名称/编号模糊搜索 |
| `fuelType` | string | 燃料类型筛选 |
| `equipmentId` | long | 设备 ID 精确 |
| `subcontractorId` | long | 分包商 ID |
| `startDate` | date | 加油时间起始 |
| `endDate` | date | 加油时间结束 |
| `onlyMine` | boolean | `true` 时仅返回当前用户记录（APP 使用）|

**Response**:
```json
{
  "code": 0,
  "data": {
    "list": [ /* RefuelingRecordVO */ ],
    "total": 200,
    "pageNo": 1,
    "pageSize": 20
  }
}
```

#### GET /equipment/refueling/statistics

**Query Params**: `startDate`, `endDate`, `groupBy`（DAY/WEEK/MONTH/EQUIPMENT/SUBCONTRACTOR）, `topN`（默认 10）

**Response**:
```json
{
  "code": 0,
  "data": {
    "totalAmount": 5620.5,
    "totalCost": 44218.93,
    "recordCount": 47,
    "trend": [
      { "label": "2026-04-01", "amount": 320.0, "cost": 2515.2 }
    ],
    "topEquipments": [
      { "equipmentId": 101, "equipmentName": "挖掘机A", "amount": 1200.0 }
    ],
    "topSubcontractors": [
      { "subcontractorId": 20, "subcontractorName": "XX分包", "cost": 8000.0 }
    ]
  }
}
```

#### GET /equipment/refueling/export

**Query Params**: 与 `/page` 相同（不含分页参数）
**Response**: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

**导出列**: 加油时间、设备编号、设备名称、分包商、燃料类型、加油量(L)、单价(元/L)、总费用(元)、表盘读数、加油方式、录入人、备注

---

## 5. 非功能需求

| 要求 | 规范 |
|------|------|
| 查询性能 | 列表 P95 < 500ms（project_id + time 联合索引）|
| 导出性能 | 10 万条以内导出时间 < 30s（EasyExcel 流式写入）|
| 数据隔离 | 所有查询强制过滤 `tenant_id` + `project_id` |
| 软删除 | 使用 `deleted` 字段，禁止物理删除 |
| 幂等创建 | 客户端可附带 `clientId` 字段防重复提交（5 分钟内去重）|

---

## 6. 验收条件

- [ ] 新增记录，receiptPhotos 超过 5 张时返回 400 校验错误
- [ ] APP 操作员修改他人记录返回 403
- [ ] APP 操作员修改 24h 前自己的记录返回 1002003001
- [ ] 统计接口按 DAY 分组时返回日期趋势数据
- [ ] 导出接口生成文件名含项目代码和日期，正确编码
- [ ] `onlyMine=true` 仅返回当前用户记录
- [ ] 所有列表/统计接口严格隔离 tenant_id + project_id
- [ ] 软删除后列表中不再出现该记录
