# 图纸两级审批流程（内部审批 + 外部审批）— 跨端共享需求

> 本文档定义图纸两级审批流程的 **跨端共享** 部分：角色定义、完整审批流程、数据模型扩展、API 接口、通知机制。
> PC 端的 UI 布局、交互和样式请参见 [REQ-007-pc.md](../pc/REQ-007-pc.md)。
> 本需求基于 [REQ-003-shared.md](./REQ-003-shared.md) 进行扩展，将原有单级审批升级为内部 + 外部两级串行审批。

## 基本信息

- **需求ID**: REQ-007
- **需求标题**: 图纸两级审批流程（内部审批 + 外部审批）
- **产品**: SMART SITE SYSTEM
- **适用端**: PC（全部操作） / APP（内部审批 Todo）
- **优先级**: 高
- **状态**: 草稿
- **依赖需求**:
  - [REQ-003-shared.md](./REQ-003-shared.md)（图纸管理核心数据模型与审批基础）
  - [REQ-006-shared.md](./REQ-006-shared.md)（QR 生成机制，时机调整为同步）

---

## 1. 需求描述

### 1.1 背景与目标

当前 REQ-003 定义的图纸审批为单级审批（一个审批人通过即生效）。实际业务中，图纸需要经过**内部技术审核**和**外部方审批**两个阶段：

- **内部审批**：由能看懂图纸的技术人员（设计经理、总工程师等）审核图纸的技术内容和质量
- **外部审批**：由业主代表、监理、设计院等外部方在 **Bentley 平台**上审批，审批通过后产出带外部签字的正式图纸

当前痛点：
- 单级审批无法区分内部质量把关和外部合规审批
- Document Controller (DC) 需要在 Smart Site 和 Bentley 之间手动流转，缺乏系统化追踪
- Site Engineer 看到的应该是**外部签字版图纸**，而非原始上传版本
- 图纸的责任追溯不清晰（上传人 vs 审批人 vs 流转人）

本功能目标：
1. 将图纸审批升级为**内部审批 → 外部审批**两级串行流程
2. 明确角色分工：设计人员上传（责任人）→ 内部审批人审核技术内容 → DC 负责外部审批流转
3. DC 将外部审批通过的签字版图纸回传到 Smart Site，作为最终流通版本
4. Site Engineer 看到的始终是**带外部签字的正式图纸**
5. 版本正式生效时同步生成 QR 码并叠加到签字版 PDF，确保流通的图纸从一开始就带有 QR
6. 新增项目级 DC 配置页面，减少业务人员重复操作
7. 所有图纸均需经过外部审批，无"仅内部审批"选项

### 1.2 用户故事

```
作为 设计人员（Designer）
我想要 上传图纸并指定内部审批人
以便 图纸经过技术审核后再进入外部审批流程，出问题时责任明确可追溯
```

```
作为 内部审批人（设计经理 / 总工程师）
我想要 在 Todo 列表中审核图纸的技术内容并决定通过或驳回
以便 确保提交给外部方的图纸质量达标，避免外部审批来回反复
```

```
作为 Document Controller (DC)
我想要 收到内部审批通过的通知后，从 Todo 列表下载原始图纸提交到 Bentley
以便 高效完成外部审批流转，不需要在系统间来回查找文件
```

```
作为 Document Controller (DC)
我想要 外部审批通过后，在 Smart Site 上传签字版图纸和审批凭证，一次性完成标记
以便 版本立即生效并自动推送给 Site Engineer，无需重复操作
```

```
作为 Site Engineer
我想要 收到的图纸是经过外部签字的正式版本
以便 现场施工依据的是合规的正式图纸，与纸质流通版一致
```

```
作为 业务人员
我想要 在配置页面设置项目的 DC 人员，而不是每次上传都指定
以便 减少重复操作，DC 人员变动时只需修改一处配置
```

### 1.3 功能范围

| 功能 | PC 端 | APP 端 | 说明 |
|------|-------|--------|------|
| 上传图纸，指定内部审批人 | ✅ | ❌ | 设计人员操作 |
| 内部审批（通过 / 驳回） | ✅ | ✅（Todo） | 内部审批人操作 |
| DC 外部审批流转（下载原始文件、标记结果、上传签字版） | ✅ | ❌ | DC 仅在 PC 端操作 |
| DC 配置页面 | ✅ | ❌ | 业务人员配置 |
| 版本历史展示（4 阶段生命周期） | ✅ | ❌ | 管理员 / Drawing 团队查看 |
| 查看签字版图纸 | ✅ | ✅ | Site Engineer 查看签字版 |

---

## 2. 完整审批流程

### 2.1 流程总览图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        图纸两级审批流程                                   │
└─────────────────────────────────────────────────────────────────────────┘

  ┌──────────┐
  │ 设计人员  │
  │ Designer │
  └────┬─────┘
       │ 上传原始图纸到 Smart Site
       │ 指定内部审批人
       │
       ▼
  ┌──────────────────────────────────────────────────┐
  │  Phase 1: 内部审批 (Internal Approval)             │
  │                                                    │
  │  审批人：设计经理 / 总工程师等（技术人员）             │
  │  审批内容：图纸技术质量                              │
  │  操作端：PC + APP（Todo 列表）                      │
  │                                                    │
  │  版本状态：PENDING_INTERNAL                         │
  └────────────────────┬───────────────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
        ┌──────────┐     ┌───────────┐
        │ ✅ 通过   │     │ ❌ 驳回    │
        └────┬─────┘     └─────┬─────┘
             │                 │
             │                 ▼
             │           ┌──────────────────────────────────┐
             │           │ 版本状态 → INTERNAL_REJECTED       │
             │           │ 通知设计人员：驳回原因              │
             │           │ 设计人员修改后重新上传新版本         │
             │           │ → 从 Phase 1 重新开始              │
             │           └──────────────────────────────────┘
             │
             ▼
  ┌──────────────────────────────────────────────────────────┐
  │ 版本状态 → PENDING_EXTERNAL                               │
  │ 系统自动通知项目 DC（从配置读取）                            │
  │ "图纸 XXX 内部审批已通过，请提交外部审批"                     │
  │ DC 的 Todo 列表自动创建外部审批任务                         │
  └────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
  ┌──────────────────────────────────────────────────┐
  │  Phase 2: 外部审批 (External Approval)             │
  │                                                    │
  │  流转人：Document Controller (DC)                   │
  │  审批方：业主代表 / 监理 / 设计院（在 Bentley 平台） │
  │  操作端：仅 PC 端                                   │
  │                                                    │
  │  DC 操作步骤：                                      │
  │  1. 从 Todo 列表下载原始图纸文件                     │
  │  2. 将图纸提交到 Bentley 平台发起外部审批            │
  │  3. （Bentley 上进行外部审批，Smart Site 无感知）    │
  │  4. 外部审批完成后，回到 Smart Site 标记结果          │
  │                                                    │
  │  版本状态：PENDING_EXTERNAL                         │
  └────────────────────┬───────────────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
  ┌─────────────────┐   ┌───────────────┐
  │ ✅ 标记外部通过   │   │ ❌ 标记外部驳回 │
  │                  │   └───────┬───────┘
  │ DC 必须提供：     │          │
  │ · 签字版图纸文件  │          ▼
  │  （必填）        │    ┌──────────────────────────────────┐
  │ · 审批凭证       │    │ 版本状态 → EXTERNAL_REJECTED       │
  │  （必填）        │    │ 通知设计人员：驳回原因              │
  │ · 外部审批日期    │    │ 设计人员修改后重新上传新版本         │
  │  （必填）        │    │ → 从 Phase 1 重新开始              │
  │ · 备注（选填）   │    └──────────────────────────────────┘
  └────────┬────────┘
           │
           ▼
  ┌──────────────────────────────────────────────────────────┐
  │ 同步执行（DC 点击确认后 loading 等待 3-5 秒）：            │
  │                                                          │
  │  1. 版本正式生效                                          │
  │     · isCurrent = true                                    │
  │     · 旧版本 isDeprecated = true                          │
  │     · 图纸主记录 status = ACTIVE                          │
  │                                                          │
  │  2. QR 生成（基于签字版 PDF）                              │
  │     · 生成 publicToken + QR 图片                          │
  │     · 将 QR 叠加到签字版 PDF → pdfWithQrUrl               │
  │                                                          │
  │  3. 推送通知给已分配的 Site Engineer                       │
  │     · App Push + 站内消息                                 │
  │                                                          │
  │  版本状态 → APPROVED                                      │
  └──────────────────────────────────────────────────────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────┐
  │ Site Engineer 收到通知                                │
  │ 查看的图纸 = 签字版 + QR 码                           │
  │ 与现场纸质流通版完全一致                               │
  └─────────────────────────────────────────────────────┘
```

### 2.2 异常流程：QR 生成失败

```
DC 点击确认（同步等待）
       │
       ├── 成功 → Toast "Approved successfully" → 推送 SE → 完成
       │
       └── 失败 → Toast 报错（如 "QR generation failed, please retry"）
                  DC 当场点击重试，无需离开页面
                  版本状态保持 PENDING_EXTERNAL，不会推送 SE
```

---

## 3. 角色与权限定义

### 3.1 角色变更对照（vs REQ-003）

| 角色 | REQ-003（原） | REQ-007（新） |
|------|-------------|-------------|
| 上传人 | Drawing 团队（模糊） | **设计人员 (Designer)**，图纸责任人 |
| 审批人 | 单一审批人 | **内部审批人**：技术审核（设计经理/总工） |
| DC | 未定义 | **Document Controller**：外部审批流转 |
| Site Engineer | 不变 | 不变，但看到的是签字版图纸 |

### 3.2 权限定义

| 操作 | 所需权限 | 角色示例 | 变更说明 |
|------|---------|---------|---------|
| 上传图纸 / 新版本 | `drawing:upload` | 设计人员 | 不变 |
| 内部审批（通过/驳回） | `drawing:approve` | 设计经理、总工程师 | 原 `drawing:approve` 现限定为内部审批 |
| 外部审批标记（通过/驳回） | `drawing:external-approval` | Document Controller | **新增权限** |
| DC 配置管理 | `drawing:dc-config` | 业务人员、项目管理员 | **新增权限** |
| 查看图纸列表 | `drawing:view` | Site Engineer 等 | 不变 |
| 查阅确认 | `drawing:confirm` | Site Engineer | 不变 |
| 查看版本历史 | `drawing:history` | 管理员、Drawing 团队 | 不变 |
| 分配图纸 | `drawing:assign` | 管理员 | 不变 |

---

## 4. 数据模型

### 4.1 DrawingVersion 表扩展字段

在 [REQ-003-shared §2.2](./REQ-003-shared.md) 已有字段基础上新增：

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 签字版文件URL | signedFileUrl | String | 条件必填 | DC 上传的外部签字版文件 OSS 地址；外部审批通过时必填 |
| 签字版文件名 | signedFileName | String | 条件必填 | 签字版原始文件名，含扩展名 |
| 签字版文件大小 | signedFileSize | Long | 自动 | 单位：字节 |
| 签字版上传时间 | signedFileUploadTime | DateTime | 自动 | DC 上传签字版的时间 |
| 审批凭证文件URL | evidenceFileUrl | String | 条件必填 | Bentley 审批截图/PDF 等凭证文件 OSS 地址；外部审批通过时必填 |
| 审批凭证文件名 | evidenceFileName | String | 条件必填 | 凭证文件名 |
| 外部审批日期 | externalApprovalDate | Date | 条件必填 | Bentley 上的实际审批通过日期；外部审批通过时必填 |
| 外部审批备注 | externalApprovalRemark | String(500) | ❌ | DC 填写的补充说明 |
| 外部审批操作人ID | externalApproverId | Long | 自动 | 执行外部审批标记的 DC 用户 ID |
| 外部审批操作人姓名 | externalApproverName | String | 自动 | 冗余字段 |

### 4.2 DrawingApproval 表扩展

在 [REQ-003-shared §2.3](./REQ-003-shared.md) 已有字段基础上新增：

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 审批阶段 | phase | String(枚举) | 自动 | `INTERNAL` / `EXTERNAL`；同一版本可有 2 条审批记录 |

> 同一个 `drawingVersionId` 下最多有 2 条 DrawingApproval 记录：一条 `phase = INTERNAL`，一条 `phase = EXTERNAL`。

### 4.3 新增：项目 DC 配置表（ProjectDcConfig）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| 配置ID | id | Long | 自动 | 主键，自增 |
| 租户ID | tenantId | Long | 自动 | SaaS 多租户标识 |
| 项目ID | projectId | Long | ✅ | 所属项目 |
| DC 用户ID | dcUserId | Long | ✅ | 被配置为 DC 的用户 ID |
| DC 用户姓名 | dcUserName | String | 自动 | 冗余字段 |
| 配置人ID | configuredById | Long | 自动 | 执行配置操作的用户 ID |
| 配置人姓名 | configuredByName | String | 自动 | 冗余字段 |
| 配置时间 | configuredAt | DateTime | 自动 | |

> **唯一性约束**：`(projectId, dcUserId)` 联合唯一，同一项目同一用户只能配置一次。

### 4.4 版本审批状态枚举扩展（DrawingVersion.approvalStatus）

替代 [REQ-003-shared §2.8](./REQ-003-shared.md) 的三态枚举：

| 值 | 英文显示 | 中文显示 | 说明 |
|----|---------|---------|------|
| PENDING_INTERNAL | Pending Internal | 待内部审批 | 设计人员上传后的初始状态 |
| INTERNAL_APPROVED | Pending External | 待外部审批 | 内部审批通过，等待 DC 流转外部审批 |
| INTERNAL_REJECTED | Int. Rejected | 内部驳回 | 内部审批人驳回 |
| APPROVED | Approved | 已通过 | 内部 + 外部均通过，版本正式生效 |
| EXTERNAL_REJECTED | Ext. Rejected | 外部驳回 | DC 标记外部审批驳回 |

> **注意**：`INTERNAL_APPROVED` 在用户界面上显示为 "Pending External"，因为从用户视角来看，该版本正在等待外部审批。

### 4.5 图纸主记录状态枚举扩展（Drawing.status）

替代 [REQ-003-shared §2.7](./REQ-003-shared.md) 的三态枚举：

| 值 | 英文显示 | 中文显示 | 说明 |
|----|---------|---------|------|
| PENDING_INTERNAL | Pending Internal | 待内部审批 | 最新版本正在内部审批中 |
| PENDING_EXTERNAL | Pending External | 待外部审批 | 最新版本内部已通过，等待外部审批 |
| ACTIVE | Active | 有效 | 最新版本已最终通过 |
| INTERNAL_REJECTED | Int. Rejected | 内部驳回 | 最新版本被内部驳回 |
| EXTERNAL_REJECTED | Ext. Rejected | 外部驳回 | 最新版本被外部驳回 |

---

## 5. 业务规则

### 5.1 上传变更（vs REQ-003 §3.1）

以下规则**替代** REQ-003 §3.1 中的审批相关逻辑：

1. **设计人员**上传图纸（非 DC），设计人员即图纸责任人
2. 上传时**必须指定内部审批人**（`approverId`），内部审批人需拥有 `drawing:approve` 权限
3. 上传后版本状态为 `PENDING_INTERNAL`（原 `PENDING`）
4. 若该图纸当前已有版本处于 `PENDING_INTERNAL` 或 `INTERNAL_APPROVED`（待外部审批）状态，不允许再次上传
5. 其余上传规则（文件格式、大小限制、版本号自增等）不变

### 5.2 Phase 1：内部审批规则

替代 REQ-003 §3.2 中的审批逻辑：

1. 上传后审批任务出现在内部审批人的 Todo 列表（PC + APP）
2. 审批人可选择**通过**或**驳回**，驳回时必须填写审批意见

#### 内部审批通过

- 版本 `approvalStatus` 更新为 `INTERNAL_APPROVED`
- 图纸主记录 `status` 更新为 `PENDING_EXTERNAL`
- 创建 `DrawingApproval` 记录：`phase = INTERNAL`、`status = APPROVED`
- **不触发**版本生效（`isCurrent` 不变）
- **不推送** Site Engineer
- **不生成** QR
- 系统向项目所有已配置的 DC 发送通知 + 创建 Todo 任务

#### 内部审批驳回

- 版本 `approvalStatus` 更新为 `INTERNAL_REJECTED`
- 图纸主记录 `status` 更新为 `INTERNAL_REJECTED`（若原来有有效版本则 `status` 保持 `ACTIVE`）
- 创建 `DrawingApproval` 记录：`phase = INTERNAL`、`status = REJECTED`
- 向设计人员（上传人）发送站内消息，包含驳回意见
- 旧版本（若存在）仍保持 `isCurrent = true`

### 5.3 Phase 2：外部审批规则

#### 外部审批通过（DC 标记）

DC 从 Todo 列表点击 `[Mark Result]`，选择"通过"并提交以下必填信息：

| 字段 | 必填 | 说明 |
|------|------|------|
| 签字版图纸文件 | ✅ | 外部审批通过后带签字的正式图纸文件 |
| 审批凭证 | ✅ | Bentley 审批截图 / PDF |
| 外部审批日期 | ✅ | Bentley 上的实际审批通过日期 |
| 备注 | ❌ | 补充说明 |

DC 点击确认后，系统**同步执行**以下操作（loading 等待 3-5 秒）：

1. 写入签字版文件信息（`signedFileUrl`、`signedFileName`、`signedFileSize`、`signedFileUploadTime`）
2. 写入审批凭证信息（`evidenceFileUrl`、`evidenceFileName`）
3. 写入外部审批日期和备注（`externalApprovalDate`、`externalApprovalRemark`）
4. 记录操作人（`externalApproverId`、`externalApproverName`）
5. 创建 `DrawingApproval` 记录：`phase = EXTERNAL`、`status = APPROVED`
6. 版本 `approvalStatus` 更新为 `APPROVED`、`isCurrent = true`、`approvedTime = NOW()`
7. 旧版本 `isCurrent = false`、`isDeprecated = true`
8. 图纸主记录 `currentVersion`、`currentVersionId` 更新，`status = ACTIVE`
9. **QR 生成**（基于签字版 PDF）：
   - 生成 `publicToken` + QR 图片 → `qrImageUrl`
   - 若签字版为 PDF，叠加 QR → `pdfWithQrUrl`
10. **推送通知**给已分配的 Site Engineer（App Push + 站内消息）

> **QR 生成失败**：整个操作回滚或保持 `PENDING_EXTERNAL` 状态，Toast 报错，DC 当场重试。不会出现"版本已生效但没有 QR"的情况。

#### 外部审批驳回（DC 标记）

1. DC 选择"驳回"，必须填写驳回原因
2. 版本 `approvalStatus` 更新为 `EXTERNAL_REJECTED`
3. 图纸主记录 `status` 更新为 `EXTERNAL_REJECTED`（若原来有有效版本则 `status` 保持 `ACTIVE`）
4. 创建 `DrawingApproval` 记录：`phase = EXTERNAL`、`status = REJECTED`
5. 向设计人员（上传人）发送站内消息，包含驳回原因
6. 旧版本（若存在）仍保持 `isCurrent = true`
7. 设计人员需修改后重新上传新版本，**从内部审批重新走**

### 5.4 文件使用优先级

| 场景 | 使用的文件 | 说明 |
|------|----------|------|
| 内部审批人审核 | `fileUrl`（原始文件） | 签字版还不存在 |
| DC 下载提交 Bentley | `fileUrl`（原始文件） | 从 Todo 列表下载 |
| Site Engineer 查看 | `signedFileUrl`（签字版） | 最终流通版 |
| Site Engineer 下载 | `pdfWithQrUrl` > `signedFileUrl` | 优先带 QR 的签字版 |
| 管理员 / Drawing 团队 | 两个文件都可查看 | 版本详情中区分展示 |
| QR 叠加 (REQ-006) | `signedFileUrl` | 叠加到签字版上 |
| Markup 标注 (REQ-004) | `signedFileUrl` | 基于签字版 |
| 审计追溯 | `fileUrl` + `signedFileUrl` | 原始文件始终保留 |

### 5.5 DC 配置规则

1. 每个项目可配置多个 DC（当前业务为 2 人）
2. 配置由拥有 `drawing:dc-config` 权限的业务人员在 PC 端操作
3. 内部审批通过后，系统通知**所有已配置的 DC**
4. 任一 DC 均可执行外部审批标记操作（先到先得）
5. 一个 DC 标记完成后，该版本的外部审批 Todo 对其他 DC 自动关闭
6. 项目至少配置 1 个 DC，否则内部审批通过后无法进入外部审批流程（系统给出提示）

### 5.6 版本可见性规则（更新 REQ-003 §3.5）

| 角色 | 可见版本 | 可见文件 |
|------|---------|---------|
| Site Engineer（APP / PC） | 仅最终通过版本（`approvalStatus = APPROVED` 且 `isCurrent = true`） | 签字版（`signedFileUrl`） |
| 内部审批人 | 待内部审批版本 + 所有版本 | 原始文件（`fileUrl`） |
| DC | 待外部审批版本 + 所有版本 | 原始文件 + 签字版 |
| Drawing 团队 / 管理员 | 所有版本 | 原始文件 + 签字版 |

---

## 6. API 接口定义

> 所有登录接口需在 Header 中传递 `Authorization`、`X-Tenant-Id`、`Project-Id`、`lang`。

### 6.1 上传图纸（变更，替代 REQ-003 §4.1）

接口不变（`POST /drawing/upload`），但业务逻辑调整：

- 上传人角色从 "Drawing 团队" 明确为**设计人员**
- 版本初始状态从 `PENDING` 改为 `PENDING_INTERNAL`
- 审批任务创建的 `phase = INTERNAL`
- 增加校验：若图纸有版本处于 `PENDING_INTERNAL` 或 `INTERNAL_APPROVED` 状态，拒绝上传

新增错误码：

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003007001 | 该图纸有版本正在审批流程中（内部或外部） | A version is currently in the approval process |

### 6.2 内部审批（变更，替代 REQ-003 §4.2）

接口不变（`POST /drawing/approve`），但业务逻辑调整：

- 仅处理 `phase = INTERNAL` 的审批
- **通过**后：版本状态 → `INTERNAL_APPROVED`，创建 DC 外部审批 Todo，通知 DC
- **驳回**后：版本状态 → `INTERNAL_REJECTED`，通知设计人员

### 6.3 外部审批标记（新增）

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/external-approve`
- Content-Type: `multipart/form-data`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| drawingVersionId | Long | ✅ | 图纸版本 ID |
| action | String | ✅ | `APPROVED`（通过）/ `REJECTED`（驳回） |
| signedFile | File | action=APPROVED 时必填 | 签字版图纸文件，≤ 50MB |
| evidenceFile | File | action=APPROVED 时必填 | 审批凭证文件，≤ 20MB，支持 PDF/PNG/JPG |
| externalApprovalDate | Date | action=APPROVED 时必填 | 外部实际审批日期，格式 YYYY-MM-DD |
| remark | String | ❌ | 备注，最多 500 字符 |
| comment | String | action=REJECTED 时必填 | 驳回原因 |

**响应格式**: `CommonResult<DrawingVersionRespVO>`

**业务逻辑（通过）**:
- ✅ 校验当前用户拥有 `drawing:external-approval` 权限
- ✅ 校验用户为项目已配置的 DC
- ✅ 校验版本 `approvalStatus = INTERNAL_APPROVED`
- ✅ 上传签字版文件至 OSS
- ✅ 上传审批凭证至 OSS
- ✅ 写入所有外部审批相关字段
- ✅ 创建 `DrawingApproval` 记录（`phase = EXTERNAL`、`status = APPROVED`）
- ✅ 同步执行版本生效 + QR 生成 + 推送通知（见 §5.3）
- ✅ 关闭其他 DC 的外部审批 Todo
- ✅ 成功后返回更新后的版本信息

**业务逻辑（驳回）**:
- ✅ 校验同上
- ✅ 版本状态 → `EXTERNAL_REJECTED`
- ✅ 创建 `DrawingApproval` 记录（`phase = EXTERNAL`、`status = REJECTED`）
- ✅ 通知设计人员
- ✅ 关闭其他 DC 的外部审批 Todo

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003007002 | 版本未处于待外部审批状态 | This version is not pending external approval |
| 1003007003 | 无外部审批权限或非项目 DC | You are not a configured DC for this project |
| 1003007004 | 签字版文件格式不支持 | Signed file format not supported |
| 1003007005 | 通过时签字版图纸文件必填 | Signed drawing file is required when approving |
| 1003007006 | 通过时审批凭证必填 | Approval evidence is required when approving |
| 1003007007 | 通过时外部审批日期必填 | External approval date is required when approving |
| 1003007008 | 驳回时原因不能为空 | Comment is required when rejecting |
| 1003007009 | QR 生成失败，请重试 | QR generation failed, please retry |

### 6.4 DC 配置 — 获取列表

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/project/dc-config/list`

**权限**: `drawing:dc-config`

**响应格式**: `CommonResult<ProjectDcConfigRespVO>`

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "projectId": 100,
    "dcList": [
      {
        "id": 1,
        "dcUserId": 3001,
        "dcUserName": "陈小明",
        "configuredByName": "Admin",
        "configuredAt": "2026-04-01T10:00:00"
      },
      {
        "id": 2,
        "dcUserId": 3002,
        "dcUserName": "刘文静",
        "configuredByName": "Admin",
        "configuredAt": "2026-04-01T10:00:00"
      }
    ],
    "availableUsers": [
      {
        "userId": 3003,
        "userName": "王磊"
      }
    ]
  }
}
```

> `availableUsers`：项目中拥有 `drawing:external-approval` 权限但尚未被配置为 DC 的用户。

### 6.5 DC 配置 — 更新

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/project/dc-config/update`

**请求格式**:
```json
{
  "dcUserIds": [3001, 3002]
}
```

**业务逻辑**:
- ✅ 校验操作人拥有 `drawing:dc-config` 权限
- ✅ 校验所有 `dcUserIds` 均为有效用户且拥有 `drawing:external-approval` 权限
- ✅ **全量覆盖模式**：替换当前项目的 DC 配置
- ✅ `dcUserIds` 不能为空数组（项目至少需要 1 个 DC）

**权限**: `drawing:dc-config`

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003007010 | DC 列表不能为空 | At least one DC is required |
| 1003007011 | 包含无效的用户 ID 或用户无 DC 权限 | One or more user IDs are invalid or lack DC permission |

### 6.6 获取外部审批 Todo（扩展 REQ-003 §4.7）

在现有 Todo 列表接口 `GET /todo/list` 中新增 Todo 类型：

```json
{
  "id": 120,
  "type": "DRAWING_EXTERNAL_APPROVAL",
  "title": "External Approval Required",
  "description": "ARCH-001 首层平面图 V3 has passed internal review. Please submit for external approval.",
  "relatedId": 305,
  "relatedType": "DRAWING_VERSION",
  "status": "PENDING",
  "createTime": "2026-04-02T14:30:00",
  "extra": {
    "drawingId": 201,
    "drawingCode": "ARCH-001",
    "drawingName": "首层平面图",
    "versionNo": "V3",
    "fileUrl": "https://oss.xxx.com/drawings/arch001-v3.pdf",
    "fileName": "arch001-v3.pdf",
    "internalApproverName": "王总工",
    "internalApprovedTime": "2026-04-02T14:30:00"
  }
}
```

> `extra.fileUrl` 和 `extra.fileName` 用于 DC 直接从 Todo 列表下载原始文件。

---

## 7. 通知机制

### 7.1 通知触发场景（替代 REQ-003 §5.1）

| 触发事件 | 通知类型 | 接收人 | 通知内容 |
|---------|---------|--------|---------|
| 上传图纸，发起内部审批 | Todo 任务 | 指定的内部审批人 | "图纸 {drawingCode} {drawingName} V{n} 等待您的内部审批" |
| 内部审批通过 | Todo 任务 + 站内通知 | 项目所有已配置 DC | "图纸 {drawingCode} {drawingName} V{n} 内部审批已通过，请提交外部审批" |
| 内部审批驳回 | 站内消息 | 设计人员（上传人） | "图纸 {drawingCode} V{n} 内部审批未通过：{comment}" |
| 外部审批通过（DC 标记） | App Push + 站内消息 | 已分配的 Site Engineer | "图纸 {drawingCode} {drawingName} 已更新至 V{n}，请查阅" |
| 外部审批驳回（DC 标记） | 站内消息 | 设计人员（上传人） | "图纸 {drawingCode} V{n} 外部审批未通过：{comment}" |

### 7.2 App Push 格式（外部审批通过后推送 SE）

与 REQ-003 §5.2 一致，仅在**外部审批通过后**才推送，内部审批通过不推送 SE。

---

## 8. 对现有需求的影响

| 需求 | 影响说明 |
|------|---------|
| **REQ-003** | 审批状态枚举扩展为 5 态；上传人角色明确为设计人员；审批流程从单级改为两级串行；文件分为原始文件 + 签字版 |
| **REQ-004 (Markup)** | Markup 标注基于签字版 `signedFileUrl`；无审批流程变更 |
| **REQ-005 (SE PC端)** | SE 在 PC 端看到的也是签字版；无其他变更 |
| **REQ-006 (QR)** | QR 生成时机从"审批通过后异步"改为"外部审批通过时同步"；QR 叠加基于签字版 |
| **BACKEND-REQ-003** | 需适配新的审批状态、新 API、数据模型扩展 |

---

## 9. 验收标准（跨端通用）

### 上传与内部审批
- [ ] 设计人员上传图纸时必须指定内部审批人
- [ ] 上传后版本状态为 `PENDING_INTERNAL`
- [ ] 图纸有版本处于 `PENDING_INTERNAL` 或 `INTERNAL_APPROVED` 状态时，不允许再次上传
- [ ] 内部审批人在 PC 和 APP Todo 列表均可看到待审批任务
- [ ] 内部审批驳回时 comment 必填
- [ ] 内部审批通过后，版本状态变为 `INTERNAL_APPROVED`，**不推送** SE、**不生成** QR

### DC 外部审批流转
- [ ] 内部审批通过后，项目所有已配置 DC 收到通知并在 Todo 列表看到任务
- [ ] DC 可从 Todo 列表直接下载原始图纸文件
- [ ] DC 标记外部通过时：签字版文件、审批凭证、外部审批日期为必填
- [ ] DC 标记外部通过后：版本生效 + QR 生成 + 推送 SE **同步完成**
- [ ] QR 生成失败时 Toast 报错，DC 当场重试，版本不会在无 QR 的情况下生效
- [ ] DC 标记外部驳回时 comment 必填
- [ ] 外部驳回后通知设计人员，设计人员需重新上传新版本从内部审批重走
- [ ] 一个 DC 标记完成后，其他 DC 的 Todo 自动关闭

### 文件使用
- [ ] Site Engineer 查看/下载的是签字版文件（`signedFileUrl`）
- [ ] QR 叠加在签字版 PDF 上（`signedFileUrl` → `pdfWithQrUrl`）
- [ ] 管理员在版本详情中可同时查看原始文件和签字版
- [ ] 原始文件始终保留不删除

### DC 配置
- [ ] 业务人员可在 PC 端配置项目的 DC 人员
- [ ] DC 列表不能为空（至少 1 人）
- [ ] DC 配置为全量覆盖模式

### 状态流转
- [ ] 版本审批状态正确流转：`PENDING_INTERNAL` → `INTERNAL_APPROVED` → `APPROVED`
- [ ] 内部驳回：`PENDING_INTERNAL` → `INTERNAL_REJECTED`
- [ ] 外部驳回：`INTERNAL_APPROVED` → `EXTERNAL_REJECTED`
- [ ] 驳回后旧版本（若存在）仍保持有效

### 权限
- [ ] 无 `drawing:upload` 权限的用户看不到上传入口
- [ ] 无 `drawing:approve` 权限的用户无法执行内部审批
- [ ] 无 `drawing:external-approval` 权限的用户无法执行外部审批标记
- [ ] 无 `drawing:dc-config` 权限的用户看不到 DC 配置页面
- [ ] 非项目已配置 DC 的用户调用外部审批接口返回 403

---

## 10. 各端需求文档索引

| 端 | 需求文档 | 说明 |
|----|---------|------|
| PC 管理端 | [REQ-007-pc.md](../pc/REQ-007-pc.md) | DC Todo 列表、外部审批 Dialog、版本历史 4 阶段展示、DC 配置页 |
| APP 移动端 | 沿用 REQ-003-app | 内部审批 Todo 与现有审批 Todo 操作一致，仅状态标签变更 |
| H5 移动端 | 不涉及 | |
