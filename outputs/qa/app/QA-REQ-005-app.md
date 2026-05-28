---
doc_type: qa_spec
runtime: UNIAPP + Vue 2（iOS & Android）
source_req: REQ-005-app (v0.1.0)
generated_at: 2026-05-28
---

# QA 测试说明文档：APP 端 Site Engineer 图纸查阅与局部更新查看

> 本文档依据 [REQ-005-app.md v0.1.0](../../../requirements/app/REQ-005-app.md) 验收标准生成。  
> 后端接口共享 [BACKEND-REQ-005.md](../../backend/BACKEND-REQ-005.md)，后端接口测试用例见该文档，本文档仅覆盖 APP 端表现。

---

## 1. 测试范围

| 编号 | 功能模块 | 来源 AC |
|------|----------|---------|
| TC-005-APP-001 | 图纸列表卡片字段 | AC-005-APP-001 |
| TC-005-APP-002 | Markups 未读角标 | AC-005-APP-002 |
| TC-005-APP-003 | 查看页顶部标题 | AC-005-APP-003 |
| TC-005-APP-004 | Confirm Reading 成功 | AC-005-APP-004 |
| TC-005-APP-005 | Confirm Reading deviceInfo=APP | AC-005-APP-005 |
| TC-005-APP-006 | 跨端同步（PC 确认 → APP 已读） | AC-005-APP-006 |
| TC-005-APP-007 | Markups Tab 仅显示 ACTIVE | AC-005-APP-007 |
| TC-005-APP-008 | Markups 卡片字段正确渲染 | AC-005-APP-008 |
| TC-005-APP-009 | Markups 卡片 appliedPageNo 为 null | AC-005-APP-008 |
| TC-005-APP-010 | Markups 卡片不含 Affected Area | AC-005-APP-008 |
| TC-005-APP-011 | Mark as Read 成功 + 角标更新 | AC-005-APP-009 |
| TC-005-APP-012 | Mark as Read 跨端同步 | AC-005-APP-010 |
| TC-005-APP-013 | Markup 发布 App Push 通知接收 | AC-005-APP-011 |
| TC-005-APP-014 | Markup 通知热启动跳转 | AC-005-APP-012 |
| TC-005-APP-015 | Markup 通知冷启动跳转 | AC-005-APP-012 |
| TC-005-APP-016 | 新版图纸发布 App Push 通知接收 | AC-005-APP-013 |
| TC-005-APP-017 | 新版图纸通知热启动跳转 | AC-005-APP-014 |
| TC-005-APP-018 | 新版图纸通知冷启动跳转 | AC-005-APP-014 |

---

## 2. 前置条件

| 条件 | 说明 |
|------|------|
| 账号 | 已登录 SE（Site Engineer）账号，且账号已分配至少一个 ACTIVE DrawingVersion |
| 图纸状态 | 至少一条 DrawingVersion 处于 `ACTIVE`，且含至少一条 ACTIVE DrawingMarkup |
| 设备 | iOS / Android 真机或模拟器，网络畅通 |
| Push | 已开启 App 通知权限；UniPush 客户端 ID 已注册 |

---

## 3. 测试用例

### TC-005-APP-001：图纸列表卡片字段

**用途**：验证卡片只渲染规范字段，不显示版本号。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 进入"图纸"列表页 | 列表加载成功 |
| 2 | 查看任意卡片内容 | 显示 Description（大标题）、Category、RFA 信息行 |
| 3 | RFA 信息行验证 | 若 rfaNo 有值：`{rfaNo} · {subjectOfRfa}`；若 rfaNo 为 null：显示 `—` |
| 4 | 底部状态区 | 未确认 → `Confirm →`（蓝色）；已确认 → `✓ Read`（绿色）|
| 5 | 检查卡片 | **不出现** versionNo、drawingCode、drawingName 任何字样 |

---

### TC-005-APP-002：Markups 未读角标

**用途**：验证角标只显示 unread 数量，unread=0 时不显示。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 进入列表页，找到有未读 Markup 的卡片 | 右上角显示橙色角标，数字为 unread 数 |
| 2 | unread=0 时 | 角标区域不出现，不显示 `0` |
| 3 | unread=100 时 | 角标显示 `99+` |
| 4 | 确认角标格式 | **不显示** `{unread}/{total}` 格式（与 PC 端不同） |

---

### TC-005-APP-003：查看页顶部标题

**用途**：验证查看页顶部导航栏只显示 Description + Category，不含版本号。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 点击任意图纸卡片，进入查看页 | 导航栏显示 Description（主标题）+ Category（副标题） |
| 2 | 检查标题区 | **不出现** versionNo 字段 |

---

### TC-005-APP-004：Confirm Reading 成功

**用途**：验证底部 Confirm Reading 完整交互流程。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 进入未确认图纸查看页 | 底部显示 `✓ Confirm Reading` 按钮（蓝色） |
| 2 | 点击按钮 | 弹出确认 Popup，副文字含 description 和 category |
| 3 | 点击 [Confirm] | Popup 关闭，按钮 Loading，请求发出 |
| 4 | 请求成功 | 底部区域变为 `✓ Confirmed on {date}`（绿色） |
| 5 | 返回列表页刷新 | 对应卡片底部显示 `✓ Read`（绿色） |
| 6 | 点击 [Cancel] | Popup 关闭，无任何变化 |
| 7 | 再次进入已确认图纸 | 底部**不显示** Confirm Reading 按钮 |

---

### TC-005-APP-005：Confirm Reading deviceInfo=APP

**用途**：验证确认请求的 deviceInfo 字段。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 在 APP 端点击 Confirm Reading 确认 | 请求 Body `deviceInfo` 字段值为 `"APP"` |
| 2 | 查询 `drawing_confirmation` 数据库记录 | `device_info = 'APP'` |

---

### TC-005-APP-006：跨端同步（PC 确认 → APP 已读）

**用途**：验证 PC 端确认后 APP 端正确显示已读状态。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | SE 在 PC 端对某 DrawingVersion 执行 Confirm Reading | PC 端显示已确认 |
| 2 | 同一 SE 在 APP 端拉取该图纸（下拉刷新或重新进入） | 列表卡片显示 `✓ Read`；进入查看页底部显示已确认文字；**不显示** Confirm Reading 按钮 |

---

### TC-005-APP-007：Markups 卡片字段正确渲染（appliedPageNo 有值）

**用途**：验证 Markups 卡片的元信息行和报审行格式。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 进入查看页，切换到 Markups Tab | 列表正常显示 |
| 2 | 查看已有 Markup 卡片 | 显示 Description（标题，16px 加粗） |
| 3 | 元信息行 | `{publishTime(MMM D, YYYY)} · {creatorName}`，灰色，13px |
| 4 | 报审行（submissionNo + appliedPageNo 均有值） | `Based on {submissionNo} · Page {n}` |
| 5 | 说明内容 | 显示 content，超过 3 行有 Show more 展开链接 |

---

### TC-005-APP-008：Markups 卡片 appliedPageNo 为 null

**用途**：验证报审行在 appliedPageNo 为 null 时不输出 Page 部分。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 找到 `appliedPageNo = null` 的 Markup 卡片 | 报审行显示 `Based on {submissionNo}`，后面**无** `· Page ...` |

---

### TC-005-APP-009：Markups 卡片不含 Affected Area

**用途**：验证 SE 视图不渲染 Affected Area。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 查看 Markups 卡片所有内容 | 页面中**不出现** "Affected Area"、"受影响区域" 任何字样 |

---

### TC-005-APP-010：Mark as Read 成功 + 角标更新

**用途**：验证 Mark as Read 完整交互。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 在 Markups Tab，找到未读（有 [New] 标签）的卡片 | 卡片右下角显示 `✓ Mark as Read` 按钮 |
| 2 | 点击 `✓ Mark as Read` | 按钮 Loading，请求发出 |
| 3 | 请求成功 | 卡片操作区变为 `✓ Read`；`[New]` 标签消失 |
| 4 | Tab 上角标 | unread 数 -1；若降至 0 则角标消失 |
| 5 | Toast 提示 | 显示 `✓ Marked as read` |
| 6 | 再次查看已读卡片 | 不显示 Mark as Read 按钮 |

---

### TC-005-APP-011：Mark as Read 跨端同步

**用途**：验证 APP 端标记后 PC 端同步更新。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | SE 在 APP 端对某 Markup 执行 Mark as Read | APP 端显示 `✓ Read` |
| 2 | 同一 SE 在 PC 端刷新该图纸 Markups Tab | 对应 Markup 显示已读；PC 端 Markups 列角标减一 |

---

### TC-005-APP-012：App Push 通知接收（Markup 发布）

**用途**：验证 Markup 发布后推送通知只达到已分配的 SE。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | PM/RFA 发布一条 DrawingMarkup，目标图纸已分配给 SE-A | SE-A 的 APP 收到通知，标题含 Description，内容含 Markup description |
| 2 | 未被分配该图纸的 SE-B | SE-B **不收到**该通知 |
| 3 | 通知 APP 角标 | APP 角标数 +1 |

---

### TC-005-APP-013：Markup 通知热启动跳转

**用途**：验证 APP 在前台/后台时点击 Markup 通知能正确跳转。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | APP 在前台时收到通知 | 弹出 Modal 提示（title + content） |
| 2 | 点击 [View] | 跳转到对应图纸查看页，**默认激活 Markups Tab** |
| 3 | APP 在后台（进程存在）收到通知 | 系统通知弹出 |
| 4 | 点击通知 | 跳转到对应图纸查看页，默认激活 Markups Tab |
| 5 | 跳转后 URL 参数 | `drawingVersionId` 和 `tab=markups` 均正确 |

---

### TC-005-APP-014：Markup 通知冷启动跳转

**用途**：验证 APP 进程被杀死时通过 Markup 通知冷启动能正确跳转。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 完全退出 APP | 进程不存在 |
| 2 | 点击系统通知栏的推送通知 | APP 启动（冷启动） |
| 3 | 启动完成后 | 自动跳转到对应图纸查看页，**默认激活 Markups Tab** |
| 4 | payload 解析 | `drawingVersionId` 正确；`type = MARKUP_PUBLISHED`；无异常 |
| 5 | 异常 payload | 若 payload 解析失败，APP 正常启动到首页，不崩溃 |

---

### TC-005-APP-015：Markups Tab 仅显示 ACTIVE 局部更新

**用途**：验证 Markups Tab 过滤规则（对应 AC-005-APP-007）。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | DrawingVersion 有 2 条 ACTIVE Markup 和 1 条 MERGED Markup | 进入 Markups Tab |
| 2 | 查看列表 | 仅显示 2 条 ACTIVE，不显示 MERGED 条目 |
| 3 | 排序 | 按 publishTime 降序，最新在上 |

---

### TC-005-APP-016：新版图纸发布 App Push 通知接收（F-005）

**用途**：验证 DrawingVersion 审批通过后，已分配 SE 收到图纸发布通知。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 管理员在 PC 端审批通过 DrawingVersion A，SE-A 已被分配该图纸 | SE-A 的 APP 收到通知，标题为 "Drawing Updated"，正文含 Description |
| 2 | 未被分配该图纸的 SE-B | SE-B **不收到**该通知 |
| 3 | 通知 APP 角标 | APP 角标数 +1 |
| 4 | 通知正文 | 包含 Description；**不包含**系统版本号（versionNo） |

---

### TC-005-APP-017：新版图纸通知热启动跳转（F-005）

**用途**：验证 APP 在前台/后台时点击新版图纸通知能正确跳转到列表页。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | APP 在前台时收到新版图纸通知 | 弹出 Modal 提示（title + content） |
| 2 | 点击 [View] | 跳转到图纸列表页，目标 DrawingVersion 被定位/高亮 |
| 3 | APP 在后台收到通知，点击通知 | 跳转到图纸列表页，目标条目高亮 |
| 4 | 跳转后 URL 参数 | `highlightId = drawingVersionId` 正确；**无** `tab=markups` 参数 |

---

### TC-005-APP-018：新版图纸通知冷启动跳转（F-005）

**用途**：验证 APP 进程被杀死时通过新版图纸通知冷启动能正确跳转。

| # | 操作 | 期望结果 |
|---|------|----------|
| 1 | 完全退出 APP | 进程不存在 |
| 2 | 点击系统通知栏的新版图纸通知 | APP 启动（冷启动） |
| 3 | 启动完成后 | 自动跳转到图纸列表页，目标 DrawingVersion 被高亮 |
| 4 | payload 解析 | `drawingVersionId` 正确；`type = DRAWING_PUBLISHED`；无异常 |
| 5 | 异常 payload | 若 payload 解析失败，APP 正常启动到首页，不崩溃 |

---

## 4. 边界与异常场景

| 场景 | 期望结果 |
|------|----------|
| 图纸文件加载失败（PDF/图片） | 显示错误占位符 + `Retry` 链接 |
| Confirm Reading 接口超时 | Toast 提示"Confirm failed, please try again"；按钮恢复可点击 |
| Mark as Read 接口超时 | Toast 提示"Failed, please try again"；按钮恢复可点击 |
| 重复点击 Confirm Reading | 按钮 Loading 期间不可再次触发；接口幂等返回成功 |
| 网络断开时点击 Confirm Reading | 接口报错，Toast 提示；图纸状态不变 |
| 无 Markup 数据 | Markups Tab 显示空状态说明文字 |
| 图纸列表为空 | 显示 `No drawings assigned to you yet.`（双语） |

---

## 5. 测试数据构造

| 数据项 | 用途 |
|--------|------|
| DrawingVersion（rfaNo=null） | TC-005-APP-001 RFA `—` 分支 |
| DrawingVersion（markupsUnread=0） | TC-005-APP-002 无角标场景 |
| DrawingVersion（markupsUnread=100） | TC-005-APP-002 `99+` 场景 |
| DrawingVersion（PC 已确认） | TC-005-APP-006 跨端同步 |
| DrawingMarkup（appliedPageNo=null） | TC-005-APP-009 无 Page 行 |
| DrawingVersion（含 ACTIVE + MERGED Markup） | TC-005-APP-015 过滤验证 |
| 两个 SE 账号（其中一个未分配） | TC-005-APP-013、TC-005-APP-016 Push 精准推送 |
| 已审批通过的 DrawingVersion | TC-005-APP-016～018 新版图纸 Push 验证 |
