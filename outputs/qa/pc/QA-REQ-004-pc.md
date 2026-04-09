# QA 测试说明文档 — REQ-004 PC 端图纸标注（Markup）管理

> **来源需求**: REQ-004-shared + REQ-004-pc
> **平台**: PC 端（Chrome / Edge，1280px+）
> **生成日期**: 2026-04-08

---

## 1. 测试目标

验证 DC 发布/删除 Markup、Markup 列表筛选、合并到新版本，以及 SE 已读确认功能的正确性与边界行为。

---

## 2. 测试场景与用例

### 场景 1：发布 Markup

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-004-PC-01 | DC 登录，有 ACTIVE 图纸 | 在图纸详情页点击 [Publish Markup]，填写 Title、Description，上传 1 个 PDF 附件，点击提交 | Markup 发布成功，出现在 Markups 列表中，状态 ACTIVE |
| TC-004-PC-02 | 同上 | Title 不填直接提交 | 提示 "Title is required" |
| TC-004-PC-03 | 同上 | Title 填写超过 200 字符 | 输入框达 200 字符后禁止继续输入，或提示超长 |
| TC-004-PC-04 | 同上 | Description 超过 2000 字符 | 同上，字符计数提示 |
| TC-004-PC-05 | 同上 | 上传第 11 个附件 | 提示 "Maximum 10 attachments" |
| TC-004-PC-06 | 同上 | 上传超过 20MB 的单个附件 | 提示 "File size must be ≤ 20MB" |

### 场景 2：Markup 列表

| ID | 测试步骤 | 预期结果 |
|----|---------|---------|
| TC-004-PC-07 | 打开图纸 Markups Drawer，切换 Tab 为 `Active` | 仅显示 status=ACTIVE 的 Markup |
| TC-004-PC-08 | 切换 Tab 为 `Merged` | 仅显示已合并的 Markup |
| TC-004-PC-09 | 切换 Tab 为 `All` | 显示所有 Markup（含 Deleted 和 Merged）|

### 场景 3：删除 Markup

| ID | 测试步骤 | 预期结果 |
|----|---------|---------|
| TC-004-PC-10 | 对 ACTIVE Markup 点击 [🗑 Delete]，确认删除 | Markup 从 Active 列表中消失，All 列表中仍可见（status=DELETED）|
| TC-004-PC-11 | 对 MERGED Markup 点击 [🗑 Delete] | 按钮禁用或提示 "Cannot delete merged markup"（错误码 1003004001）|
| TC-004-PC-12 | 点击删除后取消 | Markup 不删除 |

### 场景 4：合并到新版本

| ID | 测试步骤 | 预期结果 |
|----|---------|---------|
| TC-004-PC-13 | 勾选 ≥2 个 ACTIVE Markup，点击 [☑ Merge]，上传新版本文件，指定审批人，确认 | 新版本创建（PENDING_APPROVAL），选中 Markup 变为 MERGED |
| TC-004-PC-14 | 不勾选任何 Markup 直接点击 [Merge] | 提示 "Please select at least one markup" |
| TC-004-PC-15 | 合并时不上传新版本文件 | 提示 "New version file is required" |
| TC-004-PC-16 | 合并时不指定审批人 | 提示 "Approver is required" |
| TC-004-PC-17 | 合并成功后查看 Merged Tab | 刚合并的 Markup 出现在列表中，显示对应新版本号 |

### 场景 5：SE 已读确认（PC 端）

| ID | 测试步骤 | 预期结果 |
|----|---------|---------|
| TC-004-PC-18 | SE 在 PC 端打开 Markups Tab，点击某条 ACTIVE Markup | Markup 卡片 [New] 角标消失，调用 `/drawing/markup/confirm`（deviceInfo=PC）|
| TC-004-PC-19 | 重复点击已读 Markup | 接口返回 200，无报错（幂等）|

---

## 3. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-004-PC-01 | Markups Drawer 宽度 | 560px |
| UI-004-PC-02 | Markup 卡片 | 含标题、描述、影响区域（若有）、发布人、发布时间、附件列表 |
| UI-004-PC-03 | 未读 Markup 角标 | 橙色 [New] 标签 |
| UI-004-PC-04 | Merge 对话框 | 只读展示已选 Markup 列表，新版本文件和审批人为必填 |

---

## 4. 非功能测试

| 类型 | 测试点 |
|------|-------|
| 原子性 | 合并接口失败时新版本和 Markup 状态均回滚 |
| 幂等 | 重复确认已读不报错 |
| 安全 | 非 DC 角色无法发布/删除 Markup |

---

## 5. 验收标准

- [ ] 发布/删除 Happy Path 通过
- [ ] 字段长度校验（Title ≤200、Description ≤2000）生效
- [ ] 附件数量上限（10）和大小上限（20MB）生效
- [ ] 合并事务完整性验证通过
- [ ] SE 端已读状态正确同步
