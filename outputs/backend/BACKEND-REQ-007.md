# 后端开发说明文档 — REQ-007 图纸两级审批流程

> **来源需求**: [REQ-007-shared.md](../../requirements/shared/REQ-007-shared.md) + [REQ-007-pc.md](../../requirements/pc/REQ-007-pc.md)
> **依赖后端文档**: [BACKEND-REQ-003.md](./BACKEND-REQ-003.md)（图纸管理基础后端）
> **产品**: SMART SITE SYSTEM
> **服务模块**: `drawing-service`
> **生成日期**: 2026-04-17

---

## 1. 功能概述

在 BACKEND-REQ-003 基础上，将图纸审批从单级升级为两级串行审批：

| 功能 | 说明 |
|------|------|
| 版本状态扩展 | 3 态 → 5 态（PENDING_INTERNAL / INTERNAL_APPROVED / INTERNAL_REJECTED / APPROVED / EXTERNAL_REJECTED） |
| 内部审批变更 | 通过后不生效，改为创建 DC 外部审批 Todo |
| 外部审批标记 | DC 上传签字版 + 凭证 → 同步执行版本生效 + QR 生成 + 推送 SE |
| DC 配置管理 | 项目级 DC 人员配置（CRUD） |
| 通知扩展 | 新增内部审批通知 DC、外部驳回通知设计人员 |

---

## 2. 业务逻辑

### 2.1 版本状态流转（替代 BACKEND-REQ-003 §2.1）

```
PENDING_INTERNAL → INTERNAL_APPROVED → APPROVED
       └→ INTERNAL_REJECTED        └→ EXTERNAL_REJECTED
```

| 状态 | 说明 |
|------|------|
| `PENDING_INTERNAL` | 设计人员上传后初始状态，等待内部审批 |
| `INTERNAL_APPROVED` | 内部审批通过，等待 DC 标记外部审批结果 |
| `INTERNAL_REJECTED` | 内部审批驳回 |
| `APPROVED` | 内部 + 外部均通过，版本正式生效 |
| `EXTERNAL_REJECTED` | DC 标记外部审批驳回 |

### 2.2 内部审批通过后动作（替代 BACKEND-REQ-003 §2.2）

```
0. 校验项目已配置至少一个 DC（查询 ProjectDcConfig 列表）
   → 若无 DC，返回错误码 1003007012，阻断操作，版本状态保持 PENDING_INTERNAL
1. 版本 approvalStatus → INTERNAL_APPROVED
2. 图纸主记录 status → PENDING_EXTERNAL
3. 创建 DrawingApproval 记录（phase = INTERNAL, status = APPROVED）
4. ❌ 不触发版本生效（isCurrent 不变）
5. ❌ 不推送 Site Engineer
6. ❌ 不生成 QR
7. ✅ 为每个已配置 DC 创建 Todo 任务（type = DRAWING_EXTERNAL_APPROVAL）
8. ✅ 向每个 DC 发送站内通知
```

### 2.3 内部审批驳回后动作

```
1. 版本 approvalStatus → INTERNAL_REJECTED
2. 图纸主记录 status → INTERNAL_REJECTED（若有旧 ACTIVE 版本则保持 ACTIVE）
3. 创建 DrawingApproval 记录（phase = INTERNAL, status = REJECTED）
4. 向设计人员发送站内消息（含驳回意见）
```

### 2.4 外部审批通过后动作（DC 标记）

**同步执行，在一个事务中完成（除推送外）**：

```
1. 校验版本 approvalStatus == INTERNAL_APPROVED
2. 校验当前用户有 drawing:external-approval 权限且为项目 DC
3. 上传签字版文件至 OSS → signedFileUrl, signedFileName, signedFileSize
4. 上传审批凭证至 OSS → evidenceFileUrl, evidenceFileName
5. 写入 externalApprovalDate, externalApprovalRemark
6. 记录 externalApproverId, externalApproverName
7. 创建 DrawingApproval 记录（phase = EXTERNAL, status = APPROVED）
8. 版本 approvalStatus → APPROVED, isCurrent = true, approvedTime = NOW()
9. 旧版本 isCurrent = false, isDeprecated = true
10. 图纸主记录 currentVersion / currentVersionId 更新, status = ACTIVE
11. QR 生成（基于 signedFileUrl）：
    - 生成 publicToken + QR 图片 → qrImageUrl
    - 若签字版为 PDF → 叠加 QR → pdfWithQrUrl
12. 关闭该版本所有 DC 的外部审批 Todo（含其他 DC）
13. 异步（MQ）推送已分配 SE（App Push + 站内消息）
```

> **QR 生成失败**：事务回滚，版本保持 `INTERNAL_APPROVED`，返回错误码 `1003007009`，DC 当场重试。

### 2.5 外部审批驳回后动作（DC 标记）

```
1. 版本 approvalStatus → EXTERNAL_REJECTED
2. 图纸主记录 status → EXTERNAL_REJECTED（若有旧 ACTIVE 版本则保持 ACTIVE）
3. 创建 DrawingApproval 记录（phase = EXTERNAL, status = REJECTED）
4. 向设计人员发送站内消息（含驳回原因）
5. 关闭该版本所有 DC 的外部审批 Todo
```

### 2.6 上传变更（vs BACKEND-REQ-003）

- 版本初始状态从 `PENDING_APPROVAL` → `PENDING_INTERNAL`
- 审批任务 phase = `INTERNAL`
- 新增校验：若图纸有版本处于 `PENDING_INTERNAL` 或 `INTERNAL_APPROVED`，拒绝上传（错误码 `1003007001`）

### 2.7 DC 配置规则

- 每个项目可配置多个 DC
- 全量覆盖模式：`POST /project/dc-config/update` 替换整个 DC 列表
- `dcUserIds` 不能为空（至少 1 人）
- 校验所有用户有 `drawing:external-approval` 权限
- **内部审批通过时若无可用 DC，返回错误码 1003007012，阻断审批通过操作**（版本状态保持 `PENDING_INTERNAL`，不产生任何副作用）

---

## 3. 数据模型

### 3.1 drawing_version 表扩展字段

在 [BACKEND-REQ-003 §3.2](./BACKEND-REQ-003.md) 基础上新增：

```sql
ALTER TABLE `drawing_version` ADD COLUMN `signed_file_url` VARCHAR(500) COMMENT '签字版文件 OSS 地址';
ALTER TABLE `drawing_version` ADD COLUMN `signed_file_name` VARCHAR(200) COMMENT '签字版文件名';
ALTER TABLE `drawing_version` ADD COLUMN `signed_file_size` BIGINT COMMENT '签字版文件大小（字节）';
ALTER TABLE `drawing_version` ADD COLUMN `signed_file_upload_time` DATETIME COMMENT '签字版上传时间';
ALTER TABLE `drawing_version` ADD COLUMN `evidence_file_url` VARCHAR(500) COMMENT '审批凭证文件 OSS 地址';
ALTER TABLE `drawing_version` ADD COLUMN `evidence_file_name` VARCHAR(200) COMMENT '审批凭证文件名';
ALTER TABLE `drawing_version` ADD COLUMN `external_approval_date` DATE COMMENT '外部实际审批日期';
ALTER TABLE `drawing_version` ADD COLUMN `external_approval_remark` VARCHAR(500) COMMENT '外部审批备注';
ALTER TABLE `drawing_version` ADD COLUMN `external_approver_id` BIGINT COMMENT 'DC 用户 ID';
ALTER TABLE `drawing_version` ADD COLUMN `external_approver_name` VARCHAR(100) COMMENT 'DC 用户姓名';
```

### 3.2 drawing_approval 表扩展字段

```sql
ALTER TABLE `drawing_approval` ADD COLUMN `phase` VARCHAR(20) NOT NULL DEFAULT 'INTERNAL' COMMENT '审批阶段：INTERNAL / EXTERNAL';
```

> 同一 `drawing_version_id` 下最多 2 条记录：INTERNAL + EXTERNAL。

### 3.3 新增表：project_dc_config

```sql
CREATE TABLE `project_dc_config` (
  `id`                BIGINT PRIMARY KEY AUTO_INCREMENT,
  `tenant_id`         BIGINT NOT NULL,
  `project_id`        BIGINT NOT NULL,
  `dc_user_id`        BIGINT NOT NULL,
  `dc_user_name`      VARCHAR(100) COMMENT '冗余：DC 用户姓名',
  `configured_by_id`  BIGINT NOT NULL,
  `configured_by_name` VARCHAR(100) COMMENT '冗余：配置人姓名',
  `configured_at`     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `deleted`           TINYINT(1) NOT NULL DEFAULT 0,
  UNIQUE KEY `uk_project_dc` (`project_id`, `dc_user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.4 status 枚举扩展

**DrawingVersion.approvalStatus**（替代原 status 字段）：

```java
public enum DrawingVersionApprovalStatus {
    PENDING_INTERNAL,      // 待内部审批
    INTERNAL_APPROVED,     // 内部通过，待外部审批
    INTERNAL_REJECTED,     // 内部驳回
    APPROVED,              // 最终通过
    EXTERNAL_REJECTED      // 外部驳回
}
```

**Drawing.status**：

```java
public enum DrawingStatus {
    PENDING_INTERNAL,      // 最新版本待内部审批
    PENDING_EXTERNAL,      // 最新版本待外部审批
    ACTIVE,                // 最新版本已最终通过
    INTERNAL_REJECTED,     // 最新版本内部驳回
    EXTERNAL_REJECTED      // 最新版本外部驳回
}
```

---

## 4. API 设计

### 4.1 接口列表

| 方法 | 端点 | 变更类型 | 说明 |
|------|------|---------|------|
| `POST /drawing/upload` | 上传图纸 | **变更** | 初始状态改为 PENDING_INTERNAL，新增阻断校验 |
| `POST /drawing/approve` | 内部审批 | **变更** | 通过后不生效，创建 DC Todo |
| `POST /drawing/external-approve` | 外部审批标记 | **新增** | multipart/form-data，同步执行 |
| `GET /project/dc-config/list` | DC 配置列表 | **新增** | 返回 dcList + availableUsers |
| `POST /project/dc-config/update` | DC 配置更新 | **新增** | 全量覆盖 |
| `GET /drawing/version/history` | 版本历史 | **变更** | 返回 4 阶段生命周期数据 |

### 4.2 POST /drawing/external-approve

**Content-Type**: `multipart/form-data`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| drawingVersionId | Long | ✅ | 版本 ID |
| action | String | ✅ | `APPROVED` / `REJECTED` |
| signedFile | File | action=APPROVED 时必填 | 签字版文件，≤ 50MB |
| evidenceFile | File | action=APPROVED 时必填 | 审批凭证，≤ 20MB |
| externalApprovalDate | Date | action=APPROVED 时必填 | YYYY-MM-DD |
| remark | String | ❌ | 最多 500 字符 |
| comment | String | action=REJECTED 时必填 | 驳回原因 |

**响应**: `CommonResult<DrawingVersionRespVO>`

**伪代码**:

```java
@PostMapping("/drawing/external-approve")
@RequirePermission("drawing:external-approval")
public CommonResult<DrawingVersionRespVO> externalApprove(ExternalApproveReqDTO req) {
    // 1. 校验当前用户为项目 DC
    ProjectDcConfig dc = dcConfigService.getByProjectAndUser(projectId, currentUserId);
    if (dc == null) throw new ServiceException(1003007003);

    // 2. 校验版本状态
    DrawingVersion version = versionService.getById(req.getDrawingVersionId());
    if (version.getApprovalStatus() != INTERNAL_APPROVED) throw new ServiceException(1003007002);

    if (req.getAction() == APPROVED) {
        // 3. 校验必填文件
        if (req.getSignedFile() == null) throw new ServiceException(1003007005);
        if (req.getEvidenceFile() == null) throw new ServiceException(1003007006);
        if (req.getExternalApprovalDate() == null) throw new ServiceException(1003007007);

        // 4. 同步事务
        return drawingService.processExternalApproval(version, req);
        // 内部：上传文件 → 写入字段 → 版本生效 → QR 生成 → 关闭 Todo → 异步推送 SE
    } else {
        // 5. 驳回
        if (StringUtils.isBlank(req.getComment())) throw new ServiceException(1003007008);
        return drawingService.processExternalRejection(version, req);
    }
}
```

### 4.3 GET /project/dc-config/list

**权限**: `drawing:dc-config`

**响应**:

```json
{
  "code": 0,
  "data": {
    "projectId": 100,
    "dcList": [
      {
        "id": 1,
        "dcUserId": 3001,
        "dcUserName": "陈小明",
        "configuredByName": "Admin",
        "configuredAt": "2026-04-01T10:00:00"
      }
    ],
    "availableUsers": [
      { "userId": 3003, "userName": "王磊" }
    ]
  }
}
```

### 4.4 POST /project/dc-config/update

**权限**: `drawing:dc-config`

**请求**:

```json
{ "dcUserIds": [3001, 3002] }
```

**逻辑**:
1. 校验 `dcUserIds` 非空（错误码 `1003007010`）
2. 校验所有用户有 `drawing:external-approval` 权限（错误码 `1003007011`）
3. 删除当前项目所有 DC 配置（软删除）
4. 插入新配置

### 4.5 GET /drawing/version/history（变更）

新增响应字段：

```json
{
  "code": 0,
  "data": {
    "versions": [
      {
        "id": 301,
        "versionNo": "V3",
        "approvalStatus": "APPROVED",
        "fileName": "arch001-v3.pdf",
        "fileUrl": "https://...",
        "fileSize": 2621440,
        "uploaderName": "张三",
        "uploadedAt": "2026-04-01T10:00:00",
        "versionNote": "修正轴网尺寸",
        "internalApproval": {
          "status": "APPROVED",
          "approverName": "王总工",
          "approvedAt": "2026-04-02T14:30:00",
          "comment": "尺寸已修正"
        },
        "externalApproval": {
          "status": "APPROVED",
          "approverName": "DC 陈小明",
          "externalApprovalDate": "2026-04-05",
          "evidenceFileName": "bentley.pdf",
          "evidenceFileUrl": "https://...",
          "remark": "业主已签批"
        },
        "signedFile": {
          "fileName": "arch001-v3-signed.pdf",
          "fileUrl": "https://...",
          "fileSize": 3250585,
          "uploadedAt": "2026-04-05T16:00:00"
        },
        "confirmedCount": 5,
        "totalAssigned": 12,
        "qrImageUrl": "https://..."
      }
    ]
  }
}
```

### 4.6 错误码汇总

| 错误码 | 说明 |
|--------|------|
| 1003007001 | 该图纸有版本正在审批流程中（内部或外部） |
| 1003007002 | 版本未处于待外部审批状态 |
| 1003007003 | 无外部审批权限或非项目 DC |
| 1003007004 | 签字版文件格式不支持 |
| 1003007005 | 通过时签字版图纸文件必填 |
| 1003007006 | 通过时审批凭证必填 |
| 1003007007 | 通过时外部审批日期必填 |
| 1003007008 | 驳回时原因不能为空 |
| 1003007009 | QR 生成失败，请重试 |
| 1003007010 | DC 列表不能为空 |
| 1003007011 | 包含无效的用户 ID 或用户无 DC 权限 |
| 1003007012 | 项目未配置 DC，内部审批通过被阻断 |

---

## 5. 非功能需求

| 要求 | 规范 |
|------|------|
| 事务 | 外部审批通过（步骤 3-12）在一个数据库事务中，QR 生成失败则回滚 |
| 文件存储 | 签字版 + 凭证存储于 OSS，数据库仅存 URL |
| QR 生成 | 同步执行（vs REQ-006 原异步），超时 ≤ 10s |
| SE 推送 | 异步 MQ，推送失败不影响审批结果 |
| 数据隔离 | 所有查询强制过滤 `tenant_id` + `project_id` |
| DC Todo 关闭 | 一个 DC 标记后，同项目同版本的其他 DC Todo 状态 → CLOSED |
| 并发安全 | 外部审批标记使用乐观锁（`approvalStatus = INTERNAL_APPROVED` 作为 WHERE 条件），防止多 DC 同时操作 |
| 幂等 | DC 配置 `uk_project_dc` 唯一键保证幂等 |

---

## 6. 验收条件

- [ ] 上传图纸版本初始状态为 `PENDING_INTERNAL`
- [ ] 图纸有版本处于 `PENDING_INTERNAL` / `INTERNAL_APPROVED` 时上传被拒（错误码 1003007001）
- [ ] 内部审批通过后版本状态 → `INTERNAL_APPROVED`，不生效、不推送 SE、不生成 QR
- [ ] **项目无 DC 时调用内部审批通过接口返回 1003007012，版本状态保持 `PENDING_INTERNAL`**
- [ ] 内部审批通过后为项目所有 DC 创建 Todo + 发送通知
- [ ] 内部审批驳回后通知设计人员
- [ ] 外部审批通过后同步执行：版本生效 + QR 生成 + 推送 SE
- [ ] QR 生成失败时事务回滚，返回错误码 1003007009
- [ ] 外部审批驳回后通知设计人员
- [ ] 一个 DC 标记后其他 DC 的 Todo 自动关闭
- [ ] 多 DC 同时操作：乐观锁保证只有一个成功
- [ ] DC 配置全量覆盖，不能为空
- [ ] `GET /drawing/version/history` 返回完整 4 阶段生命周期数据
- [ ] 非项目 DC 调用外部审批接口返回 403（错误码 1003007003）
