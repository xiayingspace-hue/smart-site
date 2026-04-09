# QA 测试说明文档 — REQ-004 APP 端图纸标注（Markups Tab）与通知

> **来源需求**: REQ-004-shared + REQ-004-app + REQ-005-shared
> **平台**: APP 移动端（iOS & Android，UNIAPP）
> **生成日期**: 2026-04-08

---

## 1. 测试目标

验证 APP 端 Markups Tab 展示、Markup 已读确认、未读角标更新及 Drawing Update Push 通知的正确性。

---

## 2. 测试场景与用例

### 场景 1：Markups Tab 显示

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-004-APP-01 | SE 进入已分配图纸详情页 | 查看 Tab 导航栏 | 显示 [Drawing Preview] 和 [Markups (n)] 两个 Tab，n 为 ACTIVE Markup 数量 |
| TC-004-APP-02 | 图纸有 ≥1 个 ACTIVE Markup | 点击 [Markups] Tab | 显示 Markup 列表，按发布时间降序排列 |
| TC-004-APP-03 | 图纸无 Markup | 点击 [Markups] Tab | 显示 "No markups on this drawing." |
| TC-004-APP-04 | 有未读 Markup | 查看 Markup 列表 | 未读 Markup 卡片左上角有橙色 [New] 标签 |
| TC-004-APP-05 | 有已读 Markup | 查看 Markup 列表 | 该 Markup 卡片无 [New] 标签，背景色无高亮 |

### 场景 2：Markup 卡片内容

| ID | 测试步骤 | 预期结果 |
|----|---------|---------|
| TC-004-APP-06 | 查看 Markup 卡片 | 显示标题、描述、发布人、发布时间 |
| TC-004-APP-07 | Markup 含 affectedArea | 卡片显示 "📍 {affectedArea}" |
| TC-004-APP-08 | Markup 含附件 | 卡片显示附件 Chip 列表 |
| TC-004-APP-09 | 点击 PDF 附件 Chip | 跳转 PDF 预览页 |
| TC-004-APP-10 | 点击图片附件 Chip | 调用 `uni.previewImage` 全屏预览 |

### 场景 3：Mark as Read

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-004-APP-11 | 有未读 Markup | 点击该 Markup 卡片，进入详情页 | 自动调用 `POST /drawing/markup/confirm`（deviceInfo=APP）|
| TC-004-APP-12 | 同上 | 确认后返回 Markups Tab | 该 Markup [New] 标签消失 |
| TC-004-APP-13 | 图纸所有 Markup 均已读 | 返回图纸列表 | 该图纸卡片右上角橙色圆点消失 |
| TC-004-APP-14 | 重复调用 confirm API | — | 返回 200，无报错（幂等）|
| TC-004-APP-15 | 已读 Markup 再次进入详情 | — | 不重复触发 confirm（或幂等静默处理）|

### 场景 4：图纸列表未读角标

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-004-APP-16 | 有 ≥1 未读 Markup 的图纸 | 查看图纸列表 | 该图纸卡片右上角显示橙色圆点 |
| TC-004-APP-17 | 下拉刷新列表 | — | `unreadMarkupCount` 字段实时更新 |
| TC-004-APP-18 | 全部 Markup 已读后返回列表 | — | 橙色圆点消失 |

### 场景 5：Drawing Update Push 通知

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-004-APP-19 | SE 在前台，DC 发布新 Markup | — | SE 收到 Toast 提示 |
| TC-004-APP-20 | SE 在后台，DC 发布新 Markup | — | SE 收到系统推送通知，标题 "Drawing Update — {drawingCode}" |
| TC-004-APP-21 | 同上 | 点击推送通知 | 跳转到图纸详情页，**Markups Tab** 自动激活（activeTab=1）|
| TC-004-APP-22 | APP 未启动（冷启动），DC 发布 Markup | 点击推送通知 | APP 冷启动后跳转到正确图纸详情页的 Markups Tab |
| TC-004-APP-23 | SE 未分配该图纸 | DC 发布 Markup | SE 不收到推送 |

---

## 3. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-004-APP-01 | Tab 标签 | Markups Tab 数量 `(n)` 与 ACTIVE Markup 总数一致 |
| UI-004-APP-02 | [New] 角标 | 橙色背景，与白色字体对比清晰 |
| UI-004-APP-03 | 未读圆点 | 8px 橙色圆点，位于图纸卡片右上角 |
| UI-004-APP-04 | 附件 Chip | 显示文件名，超出宽度时省略号截断 |

---

## 4. 非功能测试

| 类型 | 测试点 |
|------|-------|
| 幂等 | Markup 已读确认重复调用 200 OK |
| 安全 | SE 只能确认分配给自己图纸的 Markup |
| 并发 | 多个 SE 同时标记同一 Markup 已读，无数据冲突 |
| 推送延迟 | Markup 发布后推送延迟 ≤ 10s |

---

## 5. 测试数据

| 数据 | 值 |
|------|---|
| 测试图纸 | DWG-001（已分配给 se_user1）|
| 测试 Markup | markup_id=10（ACTIVE，含 1 个 PDF 附件）|
| 未分配 SE | se_user2（不应收到推送）|

---

## 6. 验收标准

- [ ] Markups Tab 显示正确，数量与 ACTIVE 状态一致
- [ ] 进入详情自动标记已读，[New] 角标消失
- [ ] 图纸列表橙色圆点随未读数变化
- [ ] Push 通知精准推送（只推已分配 SE）
- [ ] 点击推送后 Markups Tab 自动激活（热启动 + 冷启动均通过）
- [ ] 幂等：重复 confirm 无报错
