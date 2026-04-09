# QA 测试说明文档 — REQ-003 PC 端图纸版本管理与查阅确认

> **来源需求**: REQ-003-shared + REQ-003-pc（Drawing Management for DC）
> **平台**: PC 端（Chrome / Edge，1280px+）
> **生成日期**: 2026-04-08

---

## 1. 测试目标

验证 PC 端图纸版本上传、审批流程、SE 分配、查阅确认及相关权限控制的正确性。

---

## 2. 测试场景与用例

### 场景 1：上传图纸新版本

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-003-PC-01 | DC 角色，图纸 DWG-001 已存在 | 进入 Drawing Management，点击 DWG-001 → [Upload New Version]，上传 PDF 文件，填写版本说明，点击提交 | 版本创建成功，状态为 PENDING_APPROVAL，版本号自动递增 |
| TC-003-PC-02 | 同上 | 上传非 PDF/PNG/JPG 格式文件 | 提示 "Unsupported file format" |
| TC-003-PC-03 | 同上 | 不选择文件直接提交 | 表单校验提示 "Please upload a file" |

### 场景 2：审批流程

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-003-PC-04 | 审批人已登录，有 PENDING_APPROVAL 版本 | 在 Todo 列表点击 [View]，查看图纸后点击 [Approve] | 版本状态变为 ACTIVE；原有 ACTIVE 版本变为 ARCHIVED；Toast 提示成功 |
| TC-003-PC-05 | 同上 | 点击 [Reject]，填写驳回意见后确认 | 版本状态变为 REJECTED；DC 收到驳回通知 |
| TC-003-PC-06 | 非该版本指定审批人 | 直接调用 approve API | 返回 403 |
| TC-003-PC-07 | 图纸已有 ACTIVE 版本 | 上传新版本并审批通过 | 新版本变 ACTIVE，旧版本变 ARCHIVED |

### 场景 3：SE 分配

| ID | 测试步骤 | 预期结果 |
|----|---------|---------|
| TC-003-PC-08 | DC 在已通过图纸的详情页，点击 [Assign SE]，选择 SE 列表并保存 | 分配成功，被分配 SE 在 APP 端图纸列表中看到该图纸 |
| TC-003-PC-09 | 重复分配同一 SE | 后端幂等处理，不报错，不重复插入 |

### 场景 4：查阅确认（PC 端 SE 视图）

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-003-PC-10 | SE 登录，有已分配 ACTIVE 图纸 | 进入 Drawings 菜单，看到图纸列表，Status 列显示 "Confirm →" | 图纸列表正确显示 |
| TC-003-PC-11 | 同上 | 点击 "Confirm →"，弹框确认后点 Confirm | 状态变为 "✓ Read"，调用 `/drawing/confirm`（deviceInfo=PC）|
| TC-003-PC-12 | 图纸已确认 | 刷新页面 | 仍显示 "✓ Read"，确认时间正确 |
| TC-003-PC-13 | SE 未被分配该图纸 | 该图纸不在 SE 图纸列表中 | 不可见 |

---

## 3. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-003-PC-01 | 图纸列表列 | 显示图纸编号、名称、分类、版本号、状态、Markups 数、已确认数、最后更新 |
| UI-003-PC-02 | 查阅确认状态 | ✓ Read（绿色）/ Confirm →（蓝色可点击）|
| UI-003-PC-03 | 在线查看页 | 包含 [Drawing Preview] 和 [Markups(n)] 两个 Tab |

---

## 4. 非功能测试

| 类型 | 测试点 |
|------|-------|
| 幂等 | 重复调用 `/drawing/confirm` 不报错 |
| 并发 | 多个 SE 同时确认同一图纸，无数据冲突 |
| 安全 | SE 无法访问未分配的图纸（直接访问详情 URL）|

---

## 5. 测试数据

| 数据 | 值 |
|------|---|
| DC 账号 | `dc_user` |
| 审批人账号 | `approver_user` |
| SE 账号 | `se_user1`, `se_user2` |
| 测试图纸 | DWG-001（结构类，已有 V1.0 ACTIVE 版本）|

---

## 6. 验收标准

- [ ] 审批通过后版本状态流转正确（ACTIVE/ARCHIVED）
- [ ] SE 图纸列表只显示 ACTIVE + 已分配图纸
- [ ] 确认查阅后状态持久化，刷新不丢失
- [ ] deviceInfo 字段记录为 "PC"
- [ ] 驳回后 DC 收到通知
