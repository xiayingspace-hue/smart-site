# 工程图纸管理 — 跨端共享需求

> 本文档定义工程图纸管理功能的 **跨端共享** 部分：业务规则、数据模型、API 接口、审批流程、通知机制。
> 各端（APP / PC）的 UI 布局、交互和样式请参见各端独立需求文档。

## 基本信息

- **需求ID**: REQ-003
- **需求标### 2.5 查阅确认记录（DrawingReadConfirmation）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|--### 3.5 版本可见性规则

| 角色 | 可见版本 | 说明 |
|------|---------|------|
| Site Engineer（APP） | 仅当前有效版本（`isCurrent = true`）且**已被分配**该图纸 | 保证看到的始终是最新审批版，且只看到相关图纸 |
| Drawing 团队 / 管理员（PC） | 所有版本（含历史版本和待审批版本） | 完整版本历史管理 |
| 审批人 | 待审批版本 + 所有版本 | |

### 3.6 权限控制认ID | id | Long | 自动 | 主键，自增 |
| 图纸版本ID | drawingVersionId | Long | 自动 | 确认的是哪个版本 |
| 图纸ID | drawingId | Long | 自动 | 冗余字段 |
| 确认人ID | confirmerId | Long | 自动 | Site Engineer 用户 ID |
| 确认人姓名 | confirmerName | String | 自动 | 冗余字段 |
| 确认时间 | confirmTime | DateTime | 自动 | |
| 确认设备 | deviceInfo | String | 自动 | 确认时使用的设备/平台（APP/PC） |

### 2.6 图纸分类枚举（drawingCategory）传、审批、版本控制与查阅确认）
- **产品**: SMART SITE SYSTEM
- **适用端**: APP（查阅确认） / PC（上传、审批、通知接收）
- **优先级**: 高
- **状态**: 草稿

---

## 1. 需求描述

### 1.1 背景与目标

工地施工过程中，设计图纸的版本管理是保障施工质量和合规性的关键环节。当前痛点：

- **版本混乱**：现场工程师可能持有旧版图纸进行施工，造成返工和损失
- **流程不透明**：图纸更新后，相关人员无法及时获知
- **确认无据**：无法追溯哪些人员已阅读并确认最新版本
- **审批缺失**：图纸上传后未经审核即投入使用，存在质量风险

本功能目标：
1. 提供 PC 端图纸上传与版本管理能力，Drawing 团队可上传新版图纸并发起审批
2. 审批通过后，系统自动将新版本设为当前有效版本，旧版本自动作废
3. 审批失败时，系统通知相关设计人员在 PC 端重新更新图纸
4. Site Engineer 通过 APP 端接收推送通知，在线查看最新图纸并点击确认查阅
5. 全程留存操作记录，支持追溯审批历史和阅读确认情况

### 1.2 用户故事

```
作为 Site Engineer（现场工程师）
我想要 在任何时候看到的设计图纸都是最新的，并能在 APP 端收到新版推送通知
以便 避免按照旧的设计图纸施工，从而造成质量问题和经济损失
```

```
作为 Site Engineer（现场工程师）
我想要 只收到和我相关的图纸通知，只看到分配给我的图纸
以便 避免被大量无关图纸信息干扰，专注于与自身职责相关的内容
```

```
作为 Drawing 团队成员
我想要 在 PC 端上传新版图纸并发起审批
以便 确保图纸经过正式审核后才对外发布，保障设计质量
```

```
作为 审批人员
我想要 在 PC 端或 APP 端的 Todo 列表中查看待审批的图纸
以便 高效完成审批操作，不遗漏任何待处理事项
```

### 1.3 功能范围

| 功能 | PC 端 | APP 端 | 说明 |
|------|-------|--------|------|
| 上传图纸（新版本） | ✅ | ❌ | Drawing 团队通过 PC 端上传 |
| 发起审批 | ✅（上传时自动发起） | ❌ | |
| 审批操作（通过 / 驳回） | ✅ | ✅（Todo 列表） | 审批人在两端均可操作 |
| 查看图纸列表 | ✅ | ✅ | 两端均可查看 |
| 在线查看图纸 | ✅ | ✅ | 在线预览（不下载） |
| 查阅确认 | ❌ | ✅ | 仅 Site Engineer 在 APP 端确认 |
| 查看版本历史 | ✅ | ✅（只读） | |
| 查看审批历史 | ✅ | ❌ | |
| 查看确认记录 | ✅ | ❌ | |
| 接收推送通知 | ✅（站内通知） | ✅（App Push） | 新版发布时推送**已被分配**的 Site Engineer |
| 接收审批失败通知 | ✅（站内通知） | ❌ | 通知设计人员 |
| 分配图纸给 Site Engineer | ✅ | ❌ | 管理员在 PC 端操作分配关系 |

---

## 2. 数据模型

### 2.1 图纸主记录（Drawing）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 图纸ID | id | Long | 自动 | 主键，自增 |
| 租户ID | tenantId | Long | 自动 | SaaS 多租户标识 |
| 项目ID | projectId | Long | ✅ | 所属项目 |
| 图纸编号 | drawingCode | String(50) | ✅ | 图纸唯一编号，同一项目内唯一，如 ARCH-001 |
| 图纸名称 | drawingName | String(200) | ✅ | 如 "首层平面图" |
| 图纸分类 | drawingCategory | String(枚举) | ✅ | 见 2.4 枚举 |
| 图纸描述 | description | String(500) | ❌ | 补充说明 |
| 当前版本号 | currentVersion | String | 自动 | 当前有效版本，如 "V3" |
| 当前版本ID | currentVersionId | Long | 自动 | 指向当前有效的版本记录 |
| 状态 | status | String(枚举) | 自动 | 见 2.5 枚举 |
| 创建人ID | creatorId | Long | 自动 | |
| 创建人姓名 | creatorName | String | 自动 | 冗余字段 |
| 创建时间 | createTime | DateTime | 自动 | |
| 更新时间 | updateTime | DateTime | 自动 | |

### 2.2 图纸版本记录（DrawingVersion）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 版本ID | id | Long | 自动 | 主键，自增 |
| 图纸ID | drawingId | Long | 自动 | 关联图纸主记录 |
| 版本号 | versionNo | String(20) | 自动 | 系统自动生成，V1、V2、V3… |
| 版本说明 | versionNote | String(500) | ❌ | 本次版本的修改说明 |
| 文件URL | fileUrl | String | 自动 | OSS 存储地址 |
| 文件名 | fileName | String | ✅ | 原始文件名，含扩展名 |
| 文件大小 | fileSize | Long | 自动 | 单位：字节 |
| 文件类型 | fileType | String | 自动 | PDF / DWG / DXF / PNG / JPG |
| 审批状态 | approvalStatus | String(枚举) | 自动 | 见 2.6 枚举 |
| 审批ID | approvalId | Long | 自动 | 关联审批流程记录 |
| 是否当前版本 | isCurrent | Boolean | 自动 | true = 当前有效版本 |
| 是否作废 | isDeprecated | Boolean | 自动 | true = 已被新版本替代 |
| 上传人ID | uploaderId | Long | 自动 | |
| 上传人姓名 | uploaderName | String | 自动 | 冗余字段 |
| 上传时间 | uploadTime | DateTime | 自动 | |
| 审批完成时间 | approvedTime | DateTime | 自动 | |

### 2.3 审批记录（DrawingApproval）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 审批ID | id | Long | 自动 | 主键，自增 |
| 图纸版本ID | drawingVersionId | Long | 自动 | 关联版本记录 |
| 图纸ID | drawingId | Long | 自动 | 冗余，便于查询 |
| 审批人ID | approverId | Long | ✅ | 指定的审批人 |
| 审批人姓名 | approverName | String | 自动 | 冗余字段 |
| 审批状态 | status | String(枚举) | 自动 | PENDING / APPROVED / REJECTED |
| 审批意见 | comment | String(500) | ❌ | 驳回时建议填写原因 |
| 审批时间 | approvalTime | DateTime | 自动 | 完成审批的时间 |
| 创建时间 | createTime | DateTime | 自动 | 审批任务创建时间 |

### 2.4 图纸分配记录（DrawingAssignment）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 分配ID | id | Long | 自动 | 主键，自增 |
| 租户ID | tenantId | Long | 自动 | SaaS 多租户标识 |
| 图纸ID | drawingId | Long | ✅ | 关联图纸主记录 |
| 项目ID | projectId | Long | ✅ | 所属项目 |
| 被分配用户ID | assigneeId | Long | ✅ | 被分配的 Site Engineer 用户 ID |
| 被分配用户姓名 | assigneeName | String | 自动 | 冗余字段 |
| 分配人ID | assignedById | Long | 自动 | 执行分配操作的管理员 ID |
| 分配人姓名 | assignedByName | String | 自动 | 冗余字段 |
| 分配时间 | assignedAt | DateTime | 自动 | |

> **唯一性约束**：`(drawingId, assigneeId)` 联合唯一，同一图纸同一人只允许有一条分配记录。

---

### 2.5 查阅确认记录（DrawingReadConfirmation）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 确认ID | id | Long | 自动 | 主键，自增 |
| 图纸版本ID | drawingVersionId | Long | 自动 | 确认的是哪个版本 |
| 图纸ID | drawingId | Long | 自动 | 冗余字段 |
| 确认人ID | confirmerId | Long | 自动 | Site Engineer 用户 ID |
| 确认人姓名 | confirmerName | String | 自动 | 冗余字段 |
| 确认时间 | confirmTime | DateTime | 自动 | |
| 确认设备 | deviceInfo | String | 自动 | 确认时使用的设备/平台（APP/PC） |

### 2.6 图纸分类枚举（drawingCategory）

| 值 | 英文显示 | 中文显示 |
|----|---------|---------|
| ARCHITECTURAL | Architectural | 建筑 |
| STRUCTURAL | Structural | 结构 |
| MECHANICAL | Mechanical | 机电 |
| ELECTRICAL | Electrical | 电气 |
| PLUMBING | Plumbing | 给排水 |
| CIVIL | Civil | 土建 |
| OTHER | Other | 其他 |

### 2.7 图纸主记录状态枚举（Drawing.status）

| 值 | 英文显示 | 中文显示 | 说明 |
|----|---------|---------|------|
| PENDING_APPROVAL | Pending Approval | 待审批 | 最新版本正在审批中 |
| ACTIVE | Active | 有效 | 最新版本已审批通过，为当前有效版本 |
| REJECTED | Rejected | 已驳回 | 最新版本审批被驳回，等待重新上传 |

### 2.8 版本审批状态枚举（DrawingVersion.approvalStatus）

| 值 | 英文显示 | 中文显示 |
|----|---------|---------|
| PENDING | Pending | 待审批 |
| APPROVED | Approved | 已通过 |
| REJECTED | Rejected | 已驳回 |

---

## 3. 业务规则

### 3.1 图纸上传与版本管理

1. Drawing 团队在 PC 端上传新版图纸，支持文件格式：PDF、DWG、DXF、PNG、JPG
2. 单个文件大小上限：**50MB**
3. 同一图纸编号每次上传自动生成新版本号（V1 → V2 → V3…），版本号不可手动修改
4. 上传时系统自动发起审批，无需手动操作
5. 若该图纸当前已有版本处于 `PENDING_APPROVAL` 状态，不允许再次上传，须等待审批完成
6. 新图纸（第一个版本）上传后，图纸主记录状态为 `PENDING_APPROVAL`
7. 审批通过前，新版本对 Site Engineer 不可见（不推送通知、不显示在图纸列表的当前版本中）

### 3.2 审批流程

1. 上传触发审批后，审批任务自动出现在指定审批人的 Todo 列表中（PC 端和 APP 端均可见）
2. 审批人可选择**通过**或**驳回**，驳回时必须填写审批意见（comment 为必填）
3. **审批通过**：
   - 新版本的 `isCurrent` 设为 `true`，`approvalStatus` 设为 `APPROVED`
   - 旧版本的 `isCurrent` 设为 `false`，`isDeprecated` 设为 `true`
   - 图纸主记录的 `currentVersion` 和 `currentVersionId` 更新为新版本
   - 图纸主记录状态更新为 `ACTIVE`
   - 系统向所有 Site Engineer 推送新版图纸通知
4. **审批驳回**：
   - 版本的 `approvalStatus` 设为 `REJECTED`
   - 图纸主记录状态更新为 `REJECTED`
   - 系统向图纸上传人（设计人员）发送站内消息通知，告知驳回原因
   - 旧版本（若存在）仍保持 `isCurrent = true`，为当前有效版本
5. 每个版本只有一个审批记录，不支持多级审批（当前阶段）

### 3.3 Site Engineer 查阅确认

1. Site Engineer 收到新版推送通知后，可进入 APP 图纸列表查看
2. 查看图纸后，需点击**"Confirm Reading"**按钮完成查阅确认
3. 确认操作仅允许对**当前有效版本**（`isCurrent = true`）执行
4. 每位 Site Engineer 对每个图纸版本只能确认一次，重复点击无效
5. 确认记录记入数据库，PC 端管理人员可查看所有确认情况
6. 未确认不影响图纸的正常查看，但系统保留未确认人员列表供管理人员跟进

### 3.4 图纸分配规则

1. 每张图纸创建后，默认**不分配**给任何 Site Engineer（即 Site Engineer 的图纸列表初始为空）
2. 管理员（拥有 `drawing:assign` 权限）在 PC 端为图纸指定一个或多个 Site Engineer
3. 分配关系与版本无关：只要图纸有分配关系，该 Site Engineer 始终能看到该图纸的当前有效版本
4. 管理员可随时新增或取消某图纸与某 Site Engineer 的分配关系
5. 取消分配后，该 Site Engineer 在 APP 端立即看不到该图纸，已有的查阅确认记录保留不删除
6. 同一图纸可分配给多名 Site Engineer；同一 Site Engineer 可被分配多张图纸
7. 审批通过后，推送通知**仅发送**给已被分配该图纸的 Site Engineer，未分配的 Site Engineer 不收到通知也看不到该图纸

### 3.5 版本可见性规则

| 角色 | 可见版本 | 说明 |
|------|---------|------|
| Site Engineer（APP） | 仅当前有效版本（`isCurrent = true`） | 保证看到的始终是最新审批版 |
| Drawing 团队 / 管理员（PC） | 所有版本（含历史版本和待审批版本） | 完整版本历史管理 |
| 审批人 | 待审批版本 + 所有版本 | |

### 3.6 权限控制

| 操作 | 所需权限 | 角色示例 |
|------|---------|---------|
| 查看图纸列表（当前版本） | `drawing:view` | Site Engineer、所有项目成员 |
| 上传图纸 / 新版本 | `drawing:upload` | Drawing 团队 |
| 审批图纸 | `drawing:approve` | 审批人员 |
| 查看版本历史 | `drawing:history` | 管理员、Drawing 团队 |
| 查看审批历史 | `drawing:approval:view` | 管理员 |
| 查看确认记录 | `drawing:confirm:view` | 管理员 |
| 查阅确认 | `drawing:confirm` | Site Engineer |
| 分配图纸给 Site Engineer | `drawing:assign` | 管理员 |

---

## 4. API 接口定义

> 所有接口需在 Header 中传递 `Authorization`、`X-Tenant-Id`、`Project-Id`、`lang`。

### 4.1 上传图纸（新建或新版本）

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/upload`
- Content-Type: `multipart/form-data`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file | File | ✅ | 图纸文件，≤ 50MB，支持 PDF/DWG/DXF/PNG/JPG |
| drawingId | Long | ❌ | 已有图纸的 ID（新版本时填写；新图纸时不填） |
| drawingCode | String | 条件必填 | 新图纸时必填，新版本时忽略 |
| drawingName | String | 条件必填 | 新图纸时必填，新版本时可选（不填则保留原名称） |
| drawingCategory | String | 条件必填 | 新图纸时必填 |
| description | String | ❌ | 图纸描述（新图纸时填写） |
| versionNote | String | ❌ | 本版本修改说明 |
| approverId | Long | ✅ | 指定审批人 ID |

**响应格式**: `CommonResult<DrawingVersionRespVO>`
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingId": 201,
    "drawingVersionId": 305,
    "versionNo": "V2",
    "approvalId": 88,
    "approvalStatus": "PENDING"
  }
}
```

**业务逻辑**:
- ✅ 校验文件格式和大小
- ✅ 若 `drawingId` 不填，创建新图纸主记录
- ✅ 若 `drawingId` 填写，校验图纸是否存在且属于当前项目
- ✅ 检查该图纸是否有版本处于 `PENDING_APPROVAL`，有则拒绝上传
- ✅ 自动生成版本号（末尾版本号 + 1）
- ✅ 上传文件至 OSS，写入 fileUrl
- ✅ 创建审批记录，关联指定审批人，状态为 `PENDING`
- ✅ 向审批人的 Todo 列表推送审批任务

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003003001 | 图纸不存在 | Drawing not found |
| 1003003002 | 文件格式不支持 | File format not supported |
| 1003003003 | 文件超过大小限制 | File size exceeds 50MB limit |
| 1003003004 | 该图纸已有版本待审批 | A version of this drawing is pending approval |
| 1003003005 | 指定审批人不存在 | Approver not found |

### 4.2 审批图纸版本

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/approve`

**请求格式**:
```json
{
  "approvalId": 88,
  "action": "APPROVED",
  "comment": ""
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| approvalId | Long | ✅ | 审批记录 ID |
| action | String | ✅ | APPROVED（通过）/ REJECTED（驳回） |
| comment | String | 条件必填 | 驳回时必填 |

**响应格式**: `CommonResult<Boolean>`

**业务逻辑（审批通过）**:
- ✅ 校验当前用户是该审批记录的审批人
- ✅ 校验审批状态为 `PENDING`（不允许重复审批）
- ✅ 更新当前版本 `approvalStatus = APPROVED`、`isCurrent = true`、`approvedTime`
- ✅ 将同图纸所有其他版本的 `isCurrent = false`、`isDeprecated = true`
- ✅ 更新图纸主记录 `currentVersion`、`currentVersionId`、`status = ACTIVE`
- ✅ 向所有拥有 `drawing:view` 权限的 Site Engineer 发送 App Push 通知
- ✅ 向所有拥有 `drawing:view` 权限的 Site Engineer 发送站内消息

**业务逻辑（审批驳回）**:
- ✅ 更新当前版本 `approvalStatus = REJECTED`
- ✅ 更新图纸主记录 `status = REJECTED`（若原来有有效版本则 status = ACTIVE）
- ✅ 向图纸上传人（uploaderName）发送站内消息通知，消息中包含驳回意见

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003003006 | 审批记录不存在 | Approval record not found |
| 1003003007 | 无权审批 | You are not the designated approver |
| 1003003008 | 审批已完成，不可重复操作 | This approval has already been completed |
| 1003003009 | 驳回时审批意见不能为空 | Comment is required when rejecting |

### 4.3 获取图纸列表（分页）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/drawing/page`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNo | Integer | ✅ | 页码，从 1 开始 |
| pageSize | Integer | ✅ | 每页条数 |
| keyword | String | ❌ | 图纸编号/名称模糊搜索 |
| drawingCategory | String | ❌ | 按分类筛选 |
| status | String | ❌ | 按图纸状态筛选（PC 端可用） |

**响应格式**: `CommonResult<PageResult<DrawingRespVO>>`

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 201,
        "drawingCode": "ARCH-001",
        "drawingName": "首层平面图",
        "drawingCategory": "ARCHITECTURAL",
        "currentVersion": "V3",
        "currentVersionId": 305,
        "status": "ACTIVE",
        "fileUrl": "https://oss.xxx.com/drawings/arch001-v3.pdf",
        "fileName": "arch001-v3.pdf",
        "fileSize": 2048000,
        "fileType": "PDF",
        "uploadTime": "2026-04-01 10:00:00",
        "uploaderName": "李四",
        "confirmedByMe": false,
        "confirmedCount": 5,
        "totalSiteEngineers": 12
      }
    ],
    "total": 45
  }
}
```

**说明**:
- Site Engineer 调用时，`status = ACTIVE` 为固定过滤条件（后端强制过滤，不依赖前端传参）
- Site Engineer 调用时，后端**同时过滤**仅返回已分配给当前用户的图纸（通过 DrawingAssignment 关联查询）
- `confirmedByMe`：当前登录用户是否已确认查阅当前版本
- `confirmedCount`：已确认查阅的 Site Engineer 人数（仅统计被分配该图纸的人员）
- `totalSiteEngineers`：被分配该图纸的 Site Engineer 总人数（非项目全体 Site Engineer）

### 4.4 获取图纸详情

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/drawing/get`
- 请求参数: `id`（图纸 ID，query 参数）

**响应格式**: `CommonResult<DrawingDetailRespVO>`（包含当前版本信息、版本历史列表、确认记录列表）

### 4.5 获取图纸版本历史

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/drawing/version/list`
- 请求参数: `drawingId`（图纸 ID，query 参数）

**响应格式**: `CommonResult<List<DrawingVersionRespVO>>`

```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "id": 305,
      "versionNo": "V3",
      "versionNote": "修正轴网尺寸",
      "fileName": "arch001-v3.pdf",
      "fileSize": 2048000,
      "approvalStatus": "APPROVED",
      "isCurrent": true,
      "isDeprecated": false,
      "uploaderName": "李四",
      "uploadTime": "2026-04-01 10:00:00",
      "approvedTime": "2026-04-01 14:30:00"
    },
    {
      "id": 287,
      "versionNo": "V2",
      "versionNote": "初版提交",
      "fileName": "arch001-v2.pdf",
      "fileSize": 1980000,
      "approvalStatus": "APPROVED",
      "isCurrent": false,
      "isDeprecated": true,
      "uploaderName": "李四",
      "uploadTime": "2026-03-20 09:00:00",
      "approvedTime": "2026-03-20 16:00:00"
    }
  ]
}
```

### 4.6 查阅确认

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/confirm`

**请求格式**:
```json
{
  "drawingVersionId": 305
}
```

**响应格式**: `CommonResult<Boolean>`

**业务逻辑**:
- ✅ 校验版本是否为当前有效版本（`isCurrent = true`）
- ✅ 检查当前用户是否已对该版本确认，若已确认则幂等返回成功
- ✅ 写入查阅确认记录，记录设备信息
- ✅ 返回成功

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003003010 | 版本不存在 | Drawing version not found |
| 1003003011 | 该版本已不是当前有效版本 | This version is no longer current |

### 4.7 获取 Todo 列表（待审批图纸）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/todo/list`
- 请求参数: `type=DRAWING_APPROVAL`（可选，不传则返回所有 Todo 类型）

**响应格式**: `CommonResult<List<TodoRespVO>>`

```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "id": 88,
      "type": "DRAWING_APPROVAL",
      "title": "Drawing Approval Required",
      "description": "ARCH-001 首层平面图 V2 is pending your approval",
      "relatedId": 305,
      "relatedType": "DRAWING_VERSION",
      "status": "PENDING",
      "createTime": "2026-04-01 10:00:00"
    }
  ]
}
```

### 4.8 获取查阅确认记录列表（PC 端管理使用）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/drawing/confirm/list`
- 请求参数: `drawingVersionId`（query 参数，必填）

**响应格式**: `CommonResult<DrawingConfirmSummaryRespVO>`

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingVersionId": 305,
    "versionNo": "V3",
    "totalSiteEngineers": 12,
    "confirmedCount": 5,
    "unconfirmedCount": 7,
    "confirmedList": [
      {
        "confirmerId": 2001,
        "confirmerName": "Wang Wei",
        "confirmTime": "2026-04-01 16:20:00"
      }
    ],
    "unconfirmedList": [
      {
        "userId": 2002,
        "userName": "Zhang San"
      }
    ]
  }
}
```

> **注意**：`totalSiteEngineers`、`confirmedList`、`unconfirmedList` 均仅统计已被分配该图纸的 Site Engineer，而非项目全体 Site Engineer。

### 4.9 分配图纸给 Site Engineer

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/assign`

**请求格式**:
```json
{
  "drawingId": 201,
  "assigneeIds": [2001, 2002, 2003]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| drawingId | Long | ✅ | 图纸 ID |
| assigneeIds | List\<Long\> | ✅ | 要分配的 Site Engineer 用户 ID 列表 |

**业务逻辑**:
- ✅ 校验操作人拥有 `drawing:assign` 权限
- ✅ 校验图纸存在且属于当前项目
- ✅ 校验所有 `assigneeIds` 均为有效用户且拥有 `drawing:view` 权限
- ✅ **全量覆盖模式**：将该图纸的分配关系替换为 `assigneeIds` 列表（删除不在列表中的旧分配，新增列表中不存在的分配）
- ✅ 若 `assigneeIds` 为空数组，则清空该图纸的所有分配关系

**响应格式**: `CommonResult<Boolean>`

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003003012 | 图纸不存在 | Drawing not found |
| 1003003013 | 包含无效的用户 ID | One or more assignee IDs are invalid |
| 1003003014 | 无分配权限 | You do not have permission to assign this drawing |

### 4.10 获取图纸的当前分配列表（PC 端分配管理使用）

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/drawing/assign/list`
- 请求参数: `drawingId`（query 参数，必填）

**响应格式**: `CommonResult<DrawingAssignmentRespVO>`

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "drawingId": 201,
    "drawingCode": "ARCH-001",
    "assignedList": [
      {
        "assigneeId": 2001,
        "assigneeName": "Wang Wei",
        "assignedAt": "2026-04-01 10:00:00",
        "assignedByName": "Admin"
      }
    ],
    "unassignedList": [
      {
        "userId": 2002,
        "userName": "Zhang San"
      }
    ]
  }
}
```

> `unassignedList` 返回项目中拥有 `drawing:view` 权限但尚未被分配该图纸的 Site Engineer 列表，用于在 PC 端分配弹窗中展示可选人员。

---

## 5. 通知机制

### 5.1 通知触发场景

| 触发事件 | 通知类型 | 接收人 | 通知内容 |
|---------|---------|--------|---------|
| 上传图纸，发起审批 | Todo 任务 | 指定审批人 | "图纸 {drawingCode} {drawingName} V{n} 等待您的审批" |
| 审批通过 | App Push + 站内消息 | **已被分配该图纸**的 Site Engineer | "图纸 {drawingCode} {drawingName} 已更新至 V{n}，请查阅" |
| 审批驳回 | 站内消息 | 图纸上传人（设计人员） | "图纸 {drawingCode} {drawingName} V{n} 审批未通过：{comment}" |

### 5.2 App Push 通知格式（审批通过后推送 Site Engineer）

| 字段 | 内容 |
|------|------|
| 标题 | Drawing Updated |
| 正文 | {drawingCode} {drawingName} has been updated to V{n}. Please confirm reading. |
| 跳转 | APP 图纸列表页，定位到对应图纸 |

> 仅向已被分配该图纸的 Site Engineer 发送推送，未分配的用户不接收通知。

---

## 6. 验收标准（跨端通用）

### 上传与版本管理
- [ ] 支持 PDF、DWG、DXF、PNG、JPG 格式上传，其他格式拒绝并提示
- [ ] 单文件大小超过 50MB 时提示错误
- [ ] 同图纸已有版本待审批时，不允许再次上传并提示
- [ ] 上传成功后版本号自动递增（V1 → V2 → V3）
- [ ] 上传后审批任务自动出现在审批人的 Todo 列表中

### 审批流程
- [ ] 审批人在 PC 端和 APP 端 Todo 列表均可看到待审批任务
- [ ] 审批通过后：新版本 `isCurrent = true`，旧版本 `isDeprecated = true`
- [ ] 审批通过后：所有 Site Engineer 收到 App Push 通知
- [ ] 审批驳回时：comment 字段为必填，不填时无法提交
- [ ] 审批驳回后：上传人收到站内消息通知，内容包含驳回意见
- [ ] 审批驳回后：旧版本（若存在）仍为有效当前版本

### Site Engineer 查阅确认
- [ ] Site Engineer 在图纸列表中只能看到 `status = ACTIVE` 且**已被分配给自己**的图纸
- [ ] Site Engineer 看到的版本始终是当前有效版本（`isCurrent = true`）
- [ ] 点击"Confirm Reading"后，确认记录写入数据库
- [ ] 同一用户对同一版本重复点击确认，系统幂等处理
- [ ] PC 端管理员可查看每张图纸的确认人数和未确认人员列表（仅统计被分配的人员）

### 图纸分配
- [ ] 管理员可在 PC 端为图纸指定一个或多个 Site Engineer
- [ ] 分配操作使用全量覆盖：提交后以最新列表为准，自动新增/移除分配关系
- [ ] `assigneeIds` 为空数组时，清空该图纸所有分配关系
- [ ] 取消分配后，对应 Site Engineer 的 APP 图纸列表立即不再显示该图纸
- [ ] 分配或取消分配不影响已有的查阅确认历史记录

### 通知
- [ ] 审批通过后，**已被分配该图纸**的 Site Engineer 收到 App Push 通知，点击可跳转到对应图纸
- [ ] 未被分配该图纸的 Site Engineer 不收到推送通知
- [ ] 审批驳回后，上传人在 PC 端收到站内消息通知

### 权限
- [ ] 无 `drawing:upload` 权限的用户看不到上传入口
- [ ] 无 `drawing:approve` 权限的用户无法进行审批操作
- [ ] 无 `drawing:confirm` 权限的用户不显示"Confirm Reading"按钮
- [ ] 无 `drawing:assign` 权限的用户看不到分配入口
- [ ] Site Engineer 无法查看待审批版本和历史版本（仅当前有效版本可见）
- [ ] Site Engineer 无法看到未分配给自己的图纸（后端过滤，前端无法绕过）

---

## 7. 各端需求文档索引

| 端 | 需求文档 | 说明 |
|----|---------|------|
| PC 管理端 | [REQ-003-pc.md](../pc/REQ-003-pc.md) | 图纸上传、审批 Todo、版本历史、确认记录查看 |
| APP 移动端 | [REQ-003-app.md](../app/REQ-003-app.md) | 推送通知、图纸列表、在线查看、查阅确认、审批 Todo |
| H5 移动端 | 不涉及 | 本功能不涉及 H5 端 |
