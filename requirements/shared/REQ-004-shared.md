# 图纸局部更新（Markup）— 跨端共享需求

> 本文档定义图纸局部更新功能的 **跨端共享** 部分：业务规则、数据模型、API 接口、通知机制。
> 各端（APP / PC）的 UI 布局、交互和样式请参见各端独立需求文档。

## 基本信息

- **需求ID**: REQ-004
- **需求标题**: 图纸局部更新（Markup）
- **产品**: SMART SITE SYSTEM
- **适用端**: PC（发布局部更新） / APP（接收通知、查阅局部更新）
- **优先级**: 高
- **状态**: 草稿
- **依赖需求**: [REQ-003-shared.md](./REQ-003-shared.md)（图纸主版本管理）

---

## 1. 需求描述

### 1.1 背景与目标

在施工过程中，设计图纸的局部内容时常需要修订（如某区域标注更正、节点详图补充等），但这些修订尚未达到发布完整新版本的程度。当前痛点：

- **信息传递延迟**：局部修订无法及时通知现场人员，容易造成施工偏差
- **版本粒度过粗**：每次细微改动都发布新版本，导致版本历史膨胀、审批流程繁琐
- **通知范围不精准**：全员广播导致无关人员被打扰，相关人员反而可能遗漏

本功能目标：
1. 设计人员可在已有图纸版本上发布**局部更新（Markup）**，描述具体改动区域和内容，无需走完整版本审批流程
2. 局部更新**定向通知**已被分配该图纸的 Site Engineer，确保信息精准送达
3. 多次局部更新积累后，设计人员可**汇总为新版本**，触发正式版本审批流程
4. Site Engineer 可在 APP 端查看图纸的所有局部更新记录，掌握最新改动

### 1.2 用户故事

```
作为 图纸设计人员
我想要 在已有图纸版本上发布局部更新，描述具体的修改区域和内容
以便 现场工程师能及时了解图纸的局部变动，无需等待完整版本审批
```

```
作为 图纸设计人员
我想要 将多次局部更新汇总，一键提交为新的正式版本
以便 在局部改动积累到一定程度后，进入正式审批流程，形成新的有效版本
```

```
作为 Site Engineer
我想要 收到与我相关图纸的局部更新通知，并能在 APP 端查看更新内容
以便 及时了解施工图纸的最新改动，避免按照已有偏差的图纸施工
```

### 1.3 功能范围

| 功能 | PC 端 | APP 端 | 说明 |
|------|-------|--------|------|
| 发布局部更新 | ✅ | ❌ | 设计人员在 PC 端操作 |
| 查看图纸的局部更新列表 | ✅ | ✅ | 两端均可查看 |
| 查看局部更新详情 | ✅ | ✅ | |
| 删除局部更新 | ✅（仅创建人，未汇总状态） | ❌ | |
| 汇总局部更新为新版本 | ✅ | ❌ | 设计人员在 PC 端操作，触发 REQ-003 审批流程 |
| 接收局部更新推送通知 | ✅（站内消息） | ✅（App Push） | 仅已分配该图纸的 Site Engineer |
| 确认已阅读局部更新 | ❌ | ✅ | Site Engineer 在 APP 端确认 |

---

## 2. 数据模型

### 2.1 局部更新记录（DrawingMarkup）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| ID | id | Long | 自动 | 主键，自增 |
| 租户ID | tenantId | Long | 自动 | SaaS 多租户标识 |
| 项目ID | projectId | Long | ✅ | 所属项目 |
| 图纸ID | drawingId | Long | ✅ | 关联的图纸主记录（Drawing） |
| 关联版本ID | baseVersionId | Long | ✅ | 本次局部更新基于的图纸版本（DrawingVersion） |
| 关联版本号 | baseVersionNo | String | 自动 | 冗余字段，如 "V3" |
| 标题 | title | String(200) | ✅ | 局部更新标题，如 "A轴节点详图修正" |
| 更新说明 | description | String(2000) | ✅ | 具体改动内容描述 |
| 影响区域 | affectedArea | String(500) | ❌ | 受影响的区域或轴线，如 "A-C轴 / 3-5层" |
| 附件列表 | attachments | JSON | ❌ | 局部图片或补充文件的 OSS URL 列表（最多 10 个） |
| 状态 | status | String(枚举) | 自动 | 见 §2.3 |
| 汇总版本ID | mergedVersionId | Long | 自动 | 被汇总进的版本 ID，status=MERGED 时填充 |
| 创建人ID | creatorId | Long | 自动 | |
| 创建人姓名 | creatorName | String | 自动 | 冗余字段 |
| 创建时间 | createTime | DateTime | 自动 | |
| 更新时间 | updateTime | DateTime | 自动 | |

### 2.2 局部更新阅读确认（DrawingMarkupReadRecord）

| 字段 | 英文名 | 类型 | 必填 | 说明 |
|------|--------|------|------|------|
| ID | id | Long | 自动 | 主键，自增 |
| 局部更新ID | markupId | Long | ✅ | 关联 DrawingMarkup |
| 图纸ID | drawingId | Long | 自动 | 冗余字段 |
| 确认人ID | confirmerId | Long | ✅ | Site Engineer 用户 ID |
| 确认人姓名 | confirmerName | String | 自动 | 冗余字段 |
| 确认时间 | confirmTime | DateTime | 自动 | |

### 2.3 局部更新状态枚举（DrawingMarkup.status）

| 值 | 英文显示 | 中文显示 | 说明 |
|----|---------|---------|------|
| ACTIVE | Active | 生效中 | 已发布，正在使用中 |
| MERGED | Merged | 已汇总 | 已被汇总进正式新版本，不再单独生效 |

---

## 3. 业务规则

### 3.1 发布局部更新

1. 局部更新只能基于当前图纸的**有效版本**（`status = ACTIVE` 的图纸 + `isCurrent = true` 的版本）发布
2. 同一图纸可同时存在多条 `ACTIVE` 状态的局部更新
3. 局部更新**不经过审批流程**，发布后立即生效
4. 附件支持 PDF / PNG / JPG，单个文件 ≤ 20MB，最多 10 个附件
5. 发布后立即触发通知，向**已被分配该图纸**的 Site Engineer 发送 App Push 和站内消息
6. 局部更新发布后，创建人可在**未汇总（`ACTIVE`）** 状态下删除；一旦被汇总则不可删除

### 3.2 汇总局部更新为新版本

1. 设计人员在 PC 端选择一张图纸后，可将该图纸下所有 `ACTIVE` 状态的局部更新**一键汇总**，触发新版本上传
2. 汇总操作本质是触发 REQ-003 中的**上传新版本**流程（`/drawing/upload`），需要选择审批人并上传新版图纸文件
3. 汇总提交后，所选局部更新的 `status` 更新为 `MERGED`，`mergedVersionId` 记录新版本 ID
4. 汇总不强制包含全部 `ACTIVE` 局部更新，设计人员可勾选部分局部更新进行汇总
5. 未被勾选的局部更新保持 `ACTIVE` 状态，可在下次汇总时再处理
6. 新版本审批通过后，Site Engineer 通过 REQ-003 的通知机制收到正式版本更新推送

### 3.3 通知规则

| 触发事件 | 通知类型 | 接收人 | 说明 |
|---------|---------|--------|------|
| 局部更新发布 | App Push + 站内消息 | 已被分配该图纸的 Site Engineer | 仅通知相关人员 |
| 局部更新汇总并提交新版本 | Todo 任务 | 指定审批人 | 复用 REQ-003 审批通知机制 |

### 3.4 可见性规则

| 角色 | 可见内容 |
|------|---------|
| Site Engineer（APP） | 已被分配的图纸下，所有 `ACTIVE` 状态的局部更新 |
| Drawing 团队 / 管理员（PC） | 该图纸下所有局部更新（含 `ACTIVE` + `MERGED`） |

### 3.5 权限控制

| 操作 | 所需权限 |
|------|---------|
| 发布局部更新 | `drawing:markup:publish` |
| 删除局部更新 | `drawing:markup:delete`（且为创建人） |
| 汇总局部更新为新版本 | `drawing:upload`（复用上传权限） |
| 查看局部更新列表 | `drawing:view` |
| 确认已阅读局部更新（APP） | `drawing:confirm` |

---

## 4. API 接口定义

> 所有接口需在 Header 中传递 `Authorization`、`X-Tenant-Id`、`Project-Id`、`lang`。

### 4.1 发布局部更新

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/markup/publish`
- Content-Type: `multipart/form-data`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| drawingId | Long | ✅ | 图纸 ID |
| title | String | ✅ | 局部更新标题，≤200 字符 |
| description | String | ✅ | 改动内容说明，≤2000 字符 |
| affectedArea | String | ❌ | 受影响区域描述，≤500 字符 |
| files | File[] | ❌ | 附件文件列表，最多 10 个，每个 ≤20MB，支持 PDF/PNG/JPG |

**响应格式**: `CommonResult<DrawingMarkupRespVO>`
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "markupId": 501,
    "drawingId": 201,
    "baseVersionNo": "V3",
    "title": "A轴节点详图修正",
    "status": "ACTIVE",
    "createTime": "2026-04-08 10:00:00"
  }
}
```

**业务逻辑**:
- ✅ 校验图纸存在且属于当前项目
- ✅ 校验图纸主记录 `status = ACTIVE`（不允许在待审批或已驳回的图纸上发布局部更新）
- ✅ 上传附件至 OSS，写入 `attachments` JSON 列表
- ✅ 记录 `baseVersionId` = 当前图纸的 `currentVersionId`
- ✅ 向已被分配该图纸的 Site Engineer 发送 App Push 和站内消息

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003004001 | 图纸不存在 | Drawing not found |
| 1003004002 | 图纸当前状态不允许发布局部更新 | Cannot publish markup on a drawing that is not active |
| 1003004003 | 附件数量超出限制 | Maximum 10 attachments allowed |
| 1003004004 | 附件格式不支持 | Attachment format not supported |
| 1003004005 | 附件超过大小限制 | Attachment size exceeds 20MB limit |

### 4.2 获取图纸的局部更新列表

**请求信息**:
- HTTP 方法: `GET`
- 端点: `/drawing/markup/list`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| drawingId | Long | ✅ | 图纸 ID |
| status | String | ❌ | 筛选状态（ACTIVE / MERGED），不传则返回全部 |

**响应格式**: `CommonResult<List<DrawingMarkupRespVO>>`
```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "markupId": 501,
      "drawingId": 201,
      "baseVersionNo": "V3",
      "title": "A轴节点详图修正",
      "description": "A轴与3轴交叉节点详图已更新，新增钢筋排布说明",
      "affectedArea": "A轴 / 3-5层",
      "attachments": [
        { "fileName": "node-detail.pdf", "fileUrl": "https://oss.xxx.com/...", "fileSize": 512000 }
      ],
      "status": "ACTIVE",
      "creatorName": "张三",
      "createTime": "2026-04-08 10:00:00"
    }
  ]
}
```

**说明**:
- Site Engineer（APP 端）调用时，后端仅返回 `status = ACTIVE` 的局部更新
- PC 端管理员可通过 `status` 参数筛选

### 4.3 删除局部更新

**请求信息**:
- HTTP 方法: `DELETE`
- 端点: `/drawing/markup/delete`
- 请求参数: `markupId`（query 参数，必填）

**业务逻辑**:
- ✅ 校验当前用户为该局部更新的创建人
- ✅ 校验 `status = ACTIVE`（已汇总的局部更新不可删除）
- ✅ 删除关联附件（OSS 文件保留，仅删除记录）

**响应格式**: `CommonResult<Boolean>`

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003004006 | 局部更新不存在 | Markup not found |
| 1003004007 | 无权删除 | You can only delete your own markups |
| 1003004008 | 已汇总的局部更新不可删除 | Cannot delete a merged markup |

### 4.4 汇总局部更新为新版本

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/markup/merge`
- Content-Type: `multipart/form-data`

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| drawingId | Long | ✅ | 图纸 ID |
| markupIds | List\<Long\> | ✅ | 要汇总的局部更新 ID 列表（至少 1 条） |
| file | File | ✅ | 汇总后的新版图纸文件 |
| versionNote | String | ❌ | 版本说明（默认自动生成："Merged from {n} markups"） |
| approverId | Long | ✅ | 指定审批人 ID |

**业务逻辑**:
- ✅ 校验所有 `markupIds` 属于该图纸且 `status = ACTIVE`
- ✅ 调用 REQ-003 上传新版本逻辑（内部复用 `/drawing/upload` 流程）
- ✅ 上传成功后，将所选局部更新的 `status` 更新为 `MERGED`，`mergedVersionId` 记录新版本 ID
- ✅ 向审批人的 Todo 列表推送审批任务

**响应格式**: `CommonResult<DrawingVersionRespVO>`（与 REQ-003 上传版本响应格式一致）

**错误响应码**:

| 错误码 | 说明 | 英文提示 |
|--------|------|---------|
| 1003004009 | 局部更新列表中包含无效项 | One or more markup IDs are invalid |
| 1003004010 | 局部更新已被汇总 | One or more markups have already been merged |
| 1003004011 | 该图纸已有版本待审批 | A version of this drawing is pending approval（复用 REQ-003 错误码 1003003004） |

### 4.5 确认已阅读局部更新（APP 端）

**请求信息**:
- HTTP 方法: `POST`
- 端点: `/drawing/markup/confirm`

**请求格式**:
```json
{ "markupId": 501 }
```

**业务逻辑**:
- ✅ 校验当前用户已被分配该图纸
- ✅ 幂等处理：同一用户对同一局部更新重复确认，返回成功
- ✅ 写入 `DrawingMarkupReadRecord`

**响应格式**: `CommonResult<Boolean>`

---

## 5. 通知消息格式

### 5.1 局部更新发布 — App Push（推送 Site Engineer）

| 字段 | 内容 |
|------|------|
| 标题 | Drawing Update — {drawingCode} |
| 正文 | {title}. Please review the latest markup for {drawingCode} {drawingName}. |
| 跳转 | APP 图纸详情页 → 局部更新 Tab，定位到该条局部更新 |

### 5.2 局部更新发布 — 站内消息（PC）

| 字段 | 内容 |
|------|------|
| 标题 | Drawing Markup Published |
| 内容 | A new markup "{title}" has been published for {drawingCode} {drawingName}. |

---

## 6. 验收标准

### 局部更新发布
- [ ] 仅允许在 `status = ACTIVE` 的图纸上发布局部更新
- [ ] 标题和更新说明为必填，为空时阻止提交
- [ ] 附件格式非 PDF/PNG/JPG 时提示错误并阻止上传
- [ ] 单个附件超过 20MB 时提示错误
- [ ] 附件数量超过 10 个时阻止继续添加
- [ ] 发布成功后，已被分配该图纸的 Site Engineer 收到 App Push 和站内消息
- [ ] 未被分配该图纸的 Site Engineer 不收到通知

### 汇总为新版本
- [ ] 只能汇总 `status = ACTIVE` 的局部更新
- [ ] 提交汇总时必须上传新版图纸文件并选择审批人
- [ ] 汇总成功后所选局部更新状态变为 `MERGED`，未选中的保持 `ACTIVE`
- [ ] 若该图纸当前有版本处于 `PENDING_APPROVAL`，汇总操作被阻止并提示
- [ ] 汇总后审批流程与 REQ-003 完全一致

### 查看局部更新
- [ ] PC 端可查看某图纸下所有状态（ACTIVE + MERGED）的局部更新列表
- [ ] APP 端仅展示 `ACTIVE` 状态的局部更新，`MERGED` 的不展示
- [ ] 未被分配该图纸的 Site Engineer 无法看到该图纸的任何局部更新

### 删除局部更新
- [ ] 只有创建人可删除，非创建人无删除入口
- [ ] `MERGED` 状态的局部更新无法删除
- [ ] 删除后局部更新从列表中消失

### 通知
- [ ] 局部更新推送仅发送给**已被分配**该图纸的 Site Engineer
- [ ] App Push 点击后正确跳转到对应图纸的局部更新详情

---

## 7. 各端需求文档索引

| 端 | 需求文档 | 说明 |
|----|---------|------|
| PC 管理端 | [REQ-004-pc.md](../pc/REQ-004-pc.md) | 发布局部更新、汇总为新版本 |
| APP 移动端 | [REQ-004-app.md](../app/REQ-004-app.md) | 接收通知、查看局部更新、确认已阅 |
