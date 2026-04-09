# 设备加油管理 — 跨端共享需求

> 本文档定义设备加油管理功能的 **跨端共享** 部分：业务规则、数据模型、API 接口、验证逻辑。
> 各端（APP / PC）的 UI 布局、交互和样式请参见各端独立需求文档。

## 基本信息

- **需求ID**: REQ-002
- **需求标题**: 设备加油管理
- **产品**: SMART SITE SYSTEM
- **适用端**: APP（数据录入） / PC（数据查看与管理）
- **优先级**: 中
- **状态**: 草稿

## 1. 需求描述

### 1.1 背景与目标

工地运行中，机械设备（塔吊、挖掘机、装载机、发电机等）的燃油消耗是重要的运营成本。目前设备加油记录多采用纸质台账或口头传达，容易出现以下问题：

- **信息滞后**：管理层无法及时掌握设备油耗情况
- **数据不准**：手工录入容易出现错误或遗漏
- **成本失控**：缺乏系统性统计，难以发现异常油耗
- **追溯困难**：纸质记录不易查找和归档

本功能目标：
1. 提供 APP 端加油信息录入能力，支持现场人员实时记录每次加油数据
2. 数据自动同步到 PC 管理端，管理员可查看、统计、导出加油记录
3. 建立设备油耗数据基础，为后续油耗分析和异常预警功能奠定数据基础

### 1.2 用户故事

```
作为 工地机械设备管理员
我想要 在 APP 端快速录入每次设备加油的信息，并在 PC 端查看汇总数据
以便 我能有效管理燃油支出、跟踪油量消耗趋势、及时发现异常油耗
```

### 1.3 功能范围

| 功能 | APP 端 | PC 端 | 说明 |
|------|--------|-------|------|
| 新增加油记录 | ✅ | ✅ | APP 为主要录入端，PC 端也可补录 |
| 编辑加油记录 | ✅（仅自己创建的，24h 内） | ✅（管理权限） | APP 端限制更严格 |
| 删除加油记录 | ❌ | ✅（管理权限） | APP 端不可删除，避免误操作 |
| 查看加油记录列表 | ✅（按设备/时间筛选） | ✅（高级筛选+表格） | 两端均可查看 |
| 查看加油记录详情 | ✅ | ✅ | |
| 油耗统计报表 | ❌ | ✅ | PC 端提供统计和图表 |
| 导出加油记录 | ❌ | ✅（Excel） | PC 端支持批量导出 |

## 2. 数据模型

### 2.1 加油记录（Refueling Record）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 记录ID | id | Long | 自动 | 主键，自增 |
| 租户ID | tenantId | Long | 自动 | SaaS 多租户标识 |
| 项目ID | projectId | Long | ✅ | 所属项目 |
| 设备ID | equipmentId | Long | ✅ | 关联的设备（从设备列表选择） |
| 设备名称 | equipmentName | String | 自动 | 冗余字段，跟随设备主数据 |
| 设备编号 | equipmentCode | String | 自动 | 冗余字段，跟随设备主数据 |
| 设备类型 | equipmentType | String | 自动 | 冗余字段，如 塔吊/挖掘机/发电机 |
| 设备图片 | equipmentImage | String | 自动 | 设备主数据中的图片 URL，用于列表页快速识别设备 |
| 分包商ID | subcontractorId | Long | 自动 | 设备所属分包商 ID，跟随设备主数据自动填充 |
| 分包商名称 | subcontractorName | String | 自动 | 冗余字段，跟随设备主数据 |
| 加油日期 | refuelingDate | Date | ✅ | 实际加油发生的日期 |
| 加油时间 | refuelingTime | Time | ❌ | 实际加油发生的时间（可选） |
| 燃油类型 | fuelType | String(枚举) | ✅ | DIESEL（柴油）/ GASOLINE（汽油）/ OTHER（其他） |
| 加油量(升) | fuelAmount | Decimal(10,2) | ✅ | 本次加油量，单位：升（L），> 0 |
| 单价 | unitPrice | Decimal(10,2) | ❌ | 每升单价，单位：当地货币 |
| 总费用 | totalCost | Decimal(10,2) | ❌ | 总费用 = 加油量 × 单价（若未填单价则留空） |
| 当前表读数 | meterReading | Decimal(10,2) | ❌ | 设备的里程表或工时表读数 |
| 表读数单位 | meterUnit | String(枚举) | ❌ | KM（公里）/ HOUR（工时） |
| 加油方式 | refuelingMethod | String(枚举) | ✅ | TANKER（油罐车）/ STATION（加油站）/ BARREL（桶装）/ OTHER（其他） |
| 供应商 | supplier | String | ❌ | 燃油供应商名称 |
| 凭证照片 | receiptPhotos | JSON(String[]) | ❌ | 加油凭证/发票照片 URL 数组，最多 5 张 |
| 备注 | remark | String(500) | ❌ | 补充说明 |
| 创建人ID | creatorId | Long | 自动 | 记录创建者 |
| 创建人姓名 | creatorName | String | 自动 | 冗余字段 |
| 创建时间 | createTime | DateTime | 自动 | 记录创建时间 |
| 更新时间 | updateTime | DateTime | 自动 | 最后更新时间 |

### 2.2 燃油类型枚举

| 值 | 英文显示 | 中文显示 |
|----|---------|---------|
| DIESEL | Diesel | 柴油 |
| GASOLINE | Gasoline | 汽油 |
| OTHER | Other | 其他 |

### 2.3 加油方式枚举

| 值 | 英文显示 | 中文显示 |
|----|---------|---------|
| TANKER | Tanker Truck | 油罐车 |
| STATION | Gas Station | 加油站 |
| BARREL | Barrel/Drum | 桶装 |
| OTHER | Other | 其他 |

### 2.4 表读数单位枚举

| 值 | 英文显示 | 中文显示 |
|----|---------|---------|
| KM | Kilometers | 公里 |
| HOUR | Hours | 工时 |

## 3. 业务规则

### 3.1 新增加油记录

1. 用户必须先选择当前项目（通过首页项目切换或加油记录页面的项目上下文）
2. 设备从当前项目的设备列表中选取（调用设备列表 API 获取）
3. 加油日期不能晚于当前日期（不允许录入未来的加油记录）
4. 加油量必须 > 0，最大不超过 99999.99 升
5. 若填写了单价，自动计算总费用 = 加油量 × 单价，精度为小数点后两位，四舍五入
6. 若未填写单价，总费用字段留空（不显示 0）
7. 凭证照片最多 5 张，单张大小 ≤ 10MB，支持 JPG / PNG / HEIC 格式
8. 提交成功后数据实时同步到 PC 管理端（通过后端 API，无需额外同步机制）
9. 创建人、创建时间由系统自动填充，不可手动修改

### 3.2 编辑加油记录

**APP 端**：
- 仅允许编辑自己创建的记录
- 仅允许在创建后 24 小时内编辑
- 超过 24 小时的记录显示为只读
- 编辑后更新 `updateTime`

**PC 端**：
- 具有管理权限的用户可编辑所有记录
- 无时间限制
- 编辑后更新 `updateTime`，记录操作日志

### 3.3 删除加油记录

- APP 端不提供删除功能
- PC 端具有管理权限的用户可删除记录
- 删除为逻辑删除（标记 `deleted = 1`），不物理删除
- 删除前需二次确认

### 3.4 列表查询

**通用筛选条件**（APP 端和 PC 端均支持）：
- 设备名称 / 设备编号（模糊搜索）
- 加油日期范围（起始 ~ 结束）
- 燃油类型

**PC 端额外筛选条件**：
- 加油方式
- 创建人
- 费用范围
- 供应商
- 分包商

**排序规则**：
- 默认按加油日期倒序（最近的在前）
- PC 端支持按列点击排序

**分页**：
- APP 端：滚动加载，每页 20 条
- PC 端：分页器，每页 20/50/100 条

### 3.5 权限控制

| 操作 | 所需权限 | 说明 |
|------|---------|------|
| 查看加油记录 | `equipment:refueling:query` | 所有有设备模块权限的用户 |
| 新增加油记录 | `equipment:refueling:create` | 设备管理员、工地管理员 |
| 编辑加油记录 | `equipment:refueling:update` | APP：自己的 + 24h 内；PC：管理权限 |
| 删除加油记录 | `equipment:refueling:delete` | 仅 PC 端管理权限 |
| 导出加油记录 | `equipment:refueling:export` | 仅 PC 端管理权限 |
| 查看油耗统计 | `equipment:refueling:statistics` | 设备管理员、项目经理 |

### 3.6 字段验证规则（跨端统一）

| 字段 | 验证规则 | 错误提示（English） | 错误提示（中文） |
|------|---------|---------------------|-----------------|
| 设备 | 必填 | Please select equipment | 请选择设备 |
| 加油日期 | 必填 | Please select refueling date | 请选择加油日期 |
| 加油日期 | ≤ 今天 | Refueling date cannot be in the future | 加油日期不能晚于今天 |
| 燃油类型 | 必填 | Please select fuel type | 请选择燃油类型 |
| 加油量 | 必填 | Please enter fuel amount | 请输入加油量 |
| 加油量 | > 0 | Fuel amount must be greater than 0 | 加油量必须大于 0 |
| 加油量 | ≤ 99999.99 | Fuel amount exceeds maximum limit | 加油量超出最大限制 |
| 单价 | ≥ 0（若填写） | Unit price cannot be negative | 单价不能为负数 |
| 加油方式 | 必填 | Please select refueling method | 请选择加油方式 |
| 凭证照片 | ≤ 5 张 | Maximum 5 photos allowed | 最多上传 5 张照片 |
| 凭证照片 | 单张 ≤ 10MB | Photo size exceeds 10MB limit | 照片大小超过 10MB 限制 |
| 备注 | ≤ 500 字 | Remark cannot exceed 500 characters | 备注不能超过 500 字 |

## 4. API 接口定义

> 以下 API 接口供所有端调用，接口定义统一。
> 所有接口需在 Header 中传递 `Authorization`、`X-Tenant-Id`、`Project-Id`、`lang`。

### 4.1 新增加油记录

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/equipment/refueling/create`

**请求格式**:
```json
{
  "equipmentId": 1001,
  "refuelingDate": "2026-03-23",
  "refuelingTime": "14:30",
  "fuelType": "DIESEL",
  "fuelAmount": 150.00,
  "unitPrice": 1.85,
  "totalCost": 277.50,
  "meterReading": 3520.5,
  "meterUnit": "HOUR",
  "refuelingMethod": "TANKER",
  "supplier": "Shell Singapore",
  "receiptPhotos": ["https://oss.xxx.com/photo1.jpg"],
  "remark": "常规加油"
}
```

**响应格式**: `CommonResult<Long>`
```json
{
  "code": 0,
  "msg": "success",
  "data": 10086
}
```
- `data` 为新创建的记录 ID

**业务逻辑**:
- ✅ 校验设备是否存在且属于当前项目
- ✅ 校验加油日期不超过当前日期
- ✅ 校验字段合法性（加油量 > 0、单价 ≥ 0 等）
- ✅ 若填写了单价，后端重新计算 totalCost（不信任前端计算结果）
- ✅ 自动填充设备冗余字段（equipmentName、equipmentCode、equipmentType、equipmentImage）
- ✅ 自动填充设备所属分包商信息（subcontractorId、subcontractorName）
- ✅ 自动填充创建人信息（creatorId、creatorName）
- ✅ 处理照片上传（若为 Base64 则转存至 OSS）

### 4.2 编辑加油记录

**请求信息**:
- HTTP 方法: `PUT`
- 端点: `/equipment/refueling/update`

**请求格式**:
```json
{
  "id": 10086,
  "equipmentId": 1001,
  "refuelingDate": "2026-03-23",
  "refuelingTime": "14:30",
  "fuelType": "DIESEL",
  "fuelAmount": 160.00,
  "unitPrice": 1.85,
  "totalCost": 296.00,
  "meterReading": 3520.5,
  "meterUnit": "HOUR",
  "refuelingMethod": "TANKER",
  "supplier": "Shell Singapore",
  "receiptPhotos": ["https://oss.xxx.com/photo1.jpg"],
  "remark": "更正加油量"
}
```

**响应格式**: `CommonResult<Boolean>`

**业务逻辑**:
- ✅ 校验记录是否存在
- ✅ APP 端：校验是否为创建人本人操作
- ✅ APP 端：校验是否在创建后 24 小时内
- ✅ 与新增相同的字段校验
- ✅ 后端重新计算 totalCost
- ✅ 更新 updateTime

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003002001 | 加油记录不存在 | Refueling record not found |
| 1003002002 | 无权编辑此记录 | You do not have permission to edit this record |
| 1003002003 | 已超过可编辑时间 | Editing period has expired (24 hours) |

### 4.3 删除加油记录

**请求信息**:
- HTTP 方法: `DELETE`
- 端点: `/equipment/refueling/delete`
- 请求参数: `id`（query 参数，必填）

**响应格式**: `CommonResult<Boolean>`

**业务逻辑**:
- ✅ 校验记录是否存在
- ✅ 校验操作者是否有删除权限
- ✅ 逻辑删除（标记 deleted = 1）
- ✅ 记录操作日志

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003002001 | 加油记录不存在 | Refueling record not found |
| 1003002004 | 无权删除此记录 | You do not have permission to delete this record |

### 4.4 获取加油记录列表（分页）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/equipment/refueling/page`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNo | Integer | ✅ | 页码，从 1 开始 |
| pageSize | Integer | ✅ | 每页条数（20/50/100） |
| equipmentId | Long | ❌ | 按设备筛选 |
| keyword | String | ❌ | 设备名称/编号模糊搜索 |
| fuelType | String | ❌ | 燃油类型 |
| refuelingMethod | String | ❌ | 加油方式 |
| startDate | Date | ❌ | 加油日期起始 |
| endDate | Date | ❌ | 加油日期截止 |
| creatorId | Long | ❌ | 创建人（PC 端使用） |
| supplier | String | ❌ | 供应商模糊搜索（PC 端使用） |
| subcontractorId | Long | ❌ | 按分包商筛选（PC 端使用） |
| minCost | Decimal | ❌ | 最低费用（PC 端使用） |
| maxCost | Decimal | ❌ | 最高费用（PC 端使用） |
| orderBy | String | ❌ | 排序字段，默认 refuelingDate |
| orderDirection | String | ❌ | ASC / DESC，默认 DESC |

**响应格式**: `CommonResult<PageResult<RefuelingRecordRespVO>>`
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 10086,
        "equipmentId": 1001,
        "equipmentName": "Tower Crane #3",
        "equipmentCode": "TC-003",
        "equipmentType": "Tower Crane",
        "equipmentImage": "https://oss.xxx.com/equipment/tc003.jpg",
        "subcontractorId": 501,
        "subcontractorName": "ABC Construction Pte Ltd",
        "refuelingDate": "2026-03-23",
        "refuelingTime": "14:30",
        "fuelType": "DIESEL",
        "fuelAmount": 150.00,
        "unitPrice": 1.85,
        "totalCost": 277.50,
        "meterReading": 3520.5,
        "meterUnit": "HOUR",
        "refuelingMethod": "TANKER",
        "supplier": "Shell Singapore",
        "receiptPhotos": ["https://oss.xxx.com/photo1.jpg"],
        "remark": "常规加油",
        "creatorId": 1024,
        "creatorName": "张三",
        "createTime": "2026-03-23 14:35:00",
        "updateTime": "2026-03-23 14:35:00",
        "editable": true
      }
    ],
    "total": 128
  }
}
```

**说明**：
- `editable` 字段由后端计算：APP 端根据创建人 + 24h 判断，PC 端根据权限判断
- 数据自动按当前项目 `Project-Id` 过滤

### 4.5 获取加油记录详情

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/equipment/refueling/get`
- 请求参数: `id`（query 参数，必填）

**响应格式**: `CommonResult<RefuelingRecordRespVO>`（同列表中的单条数据结构）

### 4.6 油耗统计（PC 端使用）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/equipment/refueling/statistics`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| startDate | Date | ✅ | 统计起始日期 |
| endDate | Date | ✅ | 统计截止日期 |
| equipmentId | Long | ❌ | 按设备筛选 |
| subcontractorId | Long | ❌ | 按分包商筛选 |
| fuelType | String | ❌ | 按燃油类型筛选 |
| groupBy | String | ✅ | 分组维度：DAY / WEEK / MONTH / EQUIPMENT / SUBCONTRACTOR |

**响应格式**: `CommonResult<RefuelingStatisticsRespVO>`
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "summary": {
      "totalFuelAmount": 15680.50,
      "totalCost": 29044.93,
      "recordCount": 128,
      "avgFuelPerRecord": 122.50,
      "avgCostPerRecord": 226.91
    },
    "details": [
      {
        "groupKey": "2026-03",
        "groupLabel": "March 2026",
        "fuelAmount": 2450.00,
        "totalCost": 4532.50,
        "recordCount": 18
      }
    ],
    "equipmentRanking": [
      {
        "equipmentId": 1001,
        "equipmentName": "Tower Crane #3",
        "equipmentCode": "TC-003",
        "subcontractorName": "ABC Construction Pte Ltd",
        "totalFuelAmount": 3200.00,
        "totalCost": 5920.00,
        "percentage": 20.4
      }
    ],
    "subcontractorRanking": [
      {
        "subcontractorId": 501,
        "subcontractorName": "ABC Construction Pte Ltd",
        "totalFuelAmount": 5800.00,
        "totalCost": 10730.00,
        "equipmentCount": 4,
        "percentage": 37.0
      }
    ]
  }
}
```

### 4.7 导出加油记录（PC 端使用）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/equipment/refueling/export`
- 请求参数: 同 4.4 列表查询参数（不含分页参数）

**响应格式**: 文件流（`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`）

**导出字段**:
设备编号、设备名称、设备类型、所属分包商、加油日期、加油时间、燃油类型、加油量(L)、单价、总费用、表读数、加油方式、供应商、备注、创建人、创建时间

## 5. 验收标准（跨端通用）

### 数据录入
- [ ] 必填字段为空时，前端阻止提交并提示对应错误信息
- [ ] 加油量必须 > 0，超出范围时提示错误
- [ ] 加油日期不能选择未来日期
- [ ] 填写单价后自动计算并显示总费用
- [ ] 未填单价时总费用不显示
- [ ] 照片上传不超过 5 张，超出提示
- [ ] 照片单张不超过 10MB，超出提示
- [ ] 提交成功后，另一端能实时查看到新增记录

### 数据编辑
- [ ] APP 端仅能编辑自己创建的且在 24h 内的记录
- [ ] APP 端超过 24h 的记录编辑按钮不可见或置灰
- [ ] PC 端有权限的用户可编辑所有记录
- [ ] 编辑后 updateTime 更新

### 数据删除
- [ ] APP 端不显示删除按钮
- [ ] PC 端删除前显示二次确认弹窗
- [ ] 删除后列表刷新，记录消失

### 列表查询
- [ ] 默认按加油日期倒序排列
- [ ] 筛选条件正确过滤结果
- [ ] 分页正常工作
- [ ] 无数据时显示空状态

### 权限控制
- [ ] 无 `equipment:refueling:query` 权限的用户看不到加油管理入口
- [ ] 无 `equipment:refueling:create` 权限的用户看不到新增按钮
- [ ] 无 `equipment:refueling:delete` 权限的用户看不到删除按钮
- [ ] 无 `equipment:refueling:export` 权限的用户看不到导出按钮

### 多语言
- [ ] 所有页面文本支持中英文切换
- [ ] 枚举值显示根据当前语言切换

## 6. 各端需求文档索引

| 端 | 需求文档 | 说明 |
|----|---------|------|
| APP 移动端 | [REQ-002-app.md](../app/REQ-002-app.md) | 加油录入为主，UNIAPP + Vue 2 |
| PC 管理端 | [REQ-002-pc.md](../pc/REQ-002-pc.md) | 数据管理、统计、导出 |
| H5 移动端 | 不涉及 | 本功能不涉及 H5 端 |
