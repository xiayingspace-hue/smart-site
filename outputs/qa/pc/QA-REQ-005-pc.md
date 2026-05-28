---
doc_type: qa_spec
source_req: REQ-005-pc (v0.3.5)
platform: PC 管理端
generated_at: 2026-05-28
---

# QA 测试规范：PC 端 Site Engineer 图纸查阅与局部更新查看

> 本文档依据 [REQ-005-pc.md v0.3.5](../../../requirements/pc/REQ-005-pc.md) 生成，覆盖 AC-005-001 ~ AC-005-012 的全部验收标准。

---

## 1. 测试目标

1. 验证 SE 图纸列表列结构正确（含 RFA 信息、无版本号）
2. 验证 Markups 列 `●{unread}/{total}` 格式与交互
3. 验证 Confirm Reading 按钮位于 Header 右侧
4. 验证 Markups Tab 卡片字段（元信息行、报审行、无 Affected Area）
5. 验证 Mark as Read 与 Confirm Reading 跨端同步
6. 验证站内通知的接收、跳转与权限控制

---

## 2. 前置条件

| 角色 | 账号 | 状态 |
|------|------|------|
| Site Engineer A | se_a@test.com | 已分配 DrawingVersion DV-001（含 3 条 ACTIVE Markup，其中 2 条未读） |
| Site Engineer B | se_b@test.com | 未被分配任何图纸 |
| Admin | admin@test.com | 有完整管理员权限 |

---

## 3. 测试场景与用例

### TC-005-PC-001：图纸列表仅显示已分配的提交记录（AC-005-001）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-001 |
| 前置 | SE A 已登录；DV-001 已分配给 SE A，DV-002 未分配 |
| 步骤 | 1. 点击侧边栏 "Drawings" 菜单 |
| 预期 | 1. 列表每行代表一次 DrawingVersion<br>2. 显示 DV-001，不显示 DV-002<br>3. 列头包含：Description、Category、RFA No.、Subject of RFA、Status、Markups、Confirmed、Last Updated<br>4. 列头**不包含**：Version / Drawing Code / Drawing Name<br>5. DV-001 的 RFA No. 已发起则显示具体编号，否则显示 `—` |

---

### TC-005-PC-002：空状态显示（AC-005-002）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-002 |
| 前置 | SE B 已登录，未被分配任何图纸 |
| 步骤 | 1. 点击侧边栏 "Drawings" 菜单 |
| 预期 | 1. 表格区域显示空状态图标<br>2. 主文字："No drawings assigned to you yet."<br>3. 副文字："Please contact your administrator for drawing access." |

---

### TC-005-PC-003：图纸在线查看 Header 信息（AC-005-003）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-003 |
| 前置 | SE A 已登录，DV-001 已分配 |
| 步骤 | 1. 点击列表中 DV-001 的行 |
| 预期 | 1. 图纸文件正常加载<br>2. Header 左侧显示 DV-001 的 Description 和 Category<br>3. Header **不显示**系统版本号（versionNo）<br>4. 支持鼠标滚轮缩放和拖拽平移 |

---

### TC-005-PC-004：Confirm Reading 按钮位于 Header（AC-005-004 + v0.3.4 改动）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-004 |
| 前置 | SE A 对 DV-001 尚未确认 |
| 步骤 | 1. 进入 DV-001 详情页<br>2. 观察页面布局（无需滚动） |
| 预期 | 1. [Confirm Reading] 按钮出现在 **Header 右侧**，页面加载后立即可见<br>2. 预览区底部**不显示**确认按钮<br>3. 点击按钮弹出确认对话框<br>4. 对话框副文字含 description 和 category |

---

### TC-005-PC-005：Confirm Reading 确认成功（AC-005-004）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-004 |
| 前置 | SE A 对 DV-001 尚未确认 |
| 步骤 | 1. 进入 DV-001 详情页<br>2. 点击 Header [Confirm Reading]<br>3. 在弹窗中点击 "Confirm" |
| 预期 | 1. Header 右侧按钮替换为 "✓ Confirmed on {date}"（绿色，不可点击）<br>2. 返回列表页，DV-001 行 Status 列显示 "✓ Read"<br>3. Confirmed 列填入确认日期 |

---

### TC-005-PC-006：Confirm Reading 写入 deviceInfo=PC（AC-005-005）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-005 |
| 前置 | SE A 对 DV-001 尚未确认 |
| 步骤 | 1. 在 PC 端完成 Confirm Reading<br>2. 查询后端 drawing_confirmation 表 |
| 预期 | 1. 对应记录的 `device_info` 字段值为 `'PC'`<br>2. `drawing_version_id` 指向 DV-001<br>3. `user_id` 为 SE A 的 ID |

---

### TC-005-PC-007：Confirm Reading 跨端同步（AC-005-006）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-006 |
| 前置 | SE A 已在 APP 端对 DV-001 完成 Confirm Reading |
| 步骤 | 1. SE A 在 PC 端打开 DV-001 详情页 |
| 预期 | 1. Header 右侧直接显示 "✓ Confirmed on {date}"<br>2. 不显示 [Confirm Reading] 按钮<br>3. 列表页 DV-001 行 Status 显示 "✓ Read" |

---

### TC-005-PC-008：Markups Tab 仅显示 ACTIVE 更新（AC-005-007）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-007 |
| 前置 | DV-001 有 2 条 ACTIVE Markup、1 条 MERGED Markup |
| 步骤 | 1. 进入 DV-001 详情页<br>2. 点击 Markups Tab |
| 预期 | 1. 仅显示 2 条 ACTIVE Markup<br>2. MERGED 状态的 Markup 不出现<br>3. 按 publishTime 降序排列（最新的在最上方） |

---

### TC-005-PC-009：Markups 卡片字段验证（v0.3.5 改动）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-007（卡片字段扩展） |
| 前置 | DV-001 有 1 条 ACTIVE Markup（submissionNo="SUB-2026-003"，appliedPageNo=3，creator="张三"） |
| 步骤 | 1. 进入 DV-001 Markups Tab<br>2. 检查卡片内容 |
| 预期 | 1. 卡片标题显示 markup.description<br>2. 元信息行显示 `{发布日期} · 张三`（格式：`Apr 8, 2026 · 张三`）<br>3. 报审行显示 `Based on SUB-2026-003 · Page 3`<br>4. **不显示** "Affected Area" 字段<br>5. 未读条目显示橙色 [New] 标签 |

---

### TC-005-PC-010：Markups 卡片报审行—无页码场景

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-007（边界场景） |
| 前置 | 某条 Markup 的 `appliedPageNo` 为 null，`submissionNo="SUB-2026-003"` |
| 步骤 | 1. 查看该 Markup 卡片的报审行 |
| 预期 | 1. 报审行显示 `Based on SUB-2026-003`<br>2. 不追加 ` · Page {n}` |

---

### TC-005-PC-011：Mark as Read 操作与列表角标更新（AC-005-008）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-008 |
| 前置 | DV-001 有 5 条 ACTIVE Markup，SE A 未读 2 条；列表页 Markups 列显示 `●2 / 5` |
| 步骤 | 1. 进入 DV-001 Markups Tab<br>2. 点击第一条未读 Markup 的 [Mark as Read]<br>3. 返回列表页观察 Markups 列<br>4. 再次进入 Markups Tab，点击第二条未读 [Mark as Read]<br>5. 返回列表页观察 Markups 列 |
| 预期 | 1. 第一次标记后：卡片按钮变为 "✓ Read"，[New] 标签消失<br>2. 列表页 Markups 列更新为 `●1 / 5`<br>3. 第二次标记后：列表页 Markups 列更新为 `0 / 5`（灰色，不可点击）<br>4. Markups Tab 标签气泡消失，仅显示 "Markups" |

---

### TC-005-PC-012：Mark as Read 跨端同步（AC-005-009）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-009 |
| 前置 | SE A 已在 APP 端对 Markup M-001 标记已读 |
| 步骤 | 1. SE A 在 PC 端刷新 DV-001 Markups Tab |
| 预期 | 1. M-001 卡片显示 "✓ Read"<br>2. 不显示 [Mark as Read] 按钮<br>3. 不显示 [New] 标签 |

---

### TC-005-PC-013：站内通知接收（AC-005-010）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-010 |
| 前置 | SE A 已分配 DV-001；Admin 准备发布一条新 Markup |
| 步骤 | 1. Admin 发布 DV-001 的新 Markup M-NEW<br>2. 等待最多 30s<br>3. SE A 在 PC 端观察铃铛角标 |
| 预期 | 1. SE A 铃铛角标数量 +1<br>2. 通知面板中出现对应通知条目（含 markup 标题和图纸描述） |

---

### TC-005-PC-014：未分配用户不收通知（AC-005-011）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-011 |
| 前置 | SE B 未被分配 DV-001；Admin 发布 DV-001 的新 Markup |
| 步骤 | 1. Admin 发布 Markup<br>2. 等待 30s<br>3. SE B 在 PC 端观察铃铛角标 |
| 预期 | 1. SE B 铃铛角标不变<br>2. SE B 通知面板中不出现该通知 |

---

### TC-005-PC-015：通知跳转至 Markups Tab（AC-005-012）

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-012 |
| 前置 | SE A 通知面板中有一条 DV-001 Markup 通知（未读） |
| 步骤 | 1. 点击该通知条目 |
| 预期 | 1. 跳转到 DV-001 详情页<br>2. Markups Tab 自动激活（不是 Drawing Preview Tab）<br>3. 该通知标记为已读（圆点消失，背景变浅）<br>4. 铃铛角标数量减一<br>5. 通知面板自动关闭 |

---

### TC-005-PC-016：搜索与筛选

| 项 | 内容 |
|----|------|
| 关联 AC | AC-005-001（补充场景） |
| 前置 | SE A 有多条已分配 DrawingVersion |
| 步骤 | 1. 在搜索框输入 "首层"（300ms 后触发搜索）<br>2. 在 Category 筛选选择 "Structural" |
| 预期 | 1. 列表过滤为 Description 中包含"首层"的记录<br>2. 再次过滤为 Category=Structural 的记录<br>3. 清空搜索框后恢复完整列表 |

---

### TC-005-PC-017：Confirm Reading 取消操作

| 项 | 内容 |
|----|------|
| 前置 | SE A 对 DV-001 尚未确认 |
| 步骤 | 1. 进入 DV-001 详情页<br>2. 点击 Header [Confirm Reading]<br>3. 在弹窗中点击 "Cancel" |
| 预期 | 1. 弹窗关闭，Header 仍显示 [Confirm Reading] 按钮<br>2. 状态不变，未写入 DrawingConfirmation 记录 |

---

### TC-005-PC-018：Confirm Reading API 失败容错

| 项 | 内容 |
|----|------|
| 前置 | SE A 对 DV-001 尚未确认；模拟接口返回 500 |
| 步骤 | 1. 点击 [Confirm Reading] 并确认<br>2. 等待接口响应 |
| 预期 | 1. Toast 提示 "Confirm failed, please try again"<br>2. Header 仍显示 [Confirm Reading] 按钮，状态不变 |

---

## 4. 界面验收

| # | 验收项 | 通过标准 |
|---|--------|---------|
| UI-001 | 列不含版本号 | 列表中无 Version / versionNo 列 |
| UI-002 | Markups 列格式 | unread>0 → 橙色 `●{n}` 前缀；全部已读 → 灰色 `0/{total}`；无 Markup → `—` |
| UI-003 | Confirm Reading 位置 | 按钮在 Header 右侧，与标题同行，无需滚动 |
| UI-004 | 卡片元信息行 | 格式为 `{日期} · {创建人}` |
| UI-005 | 卡片报审行 | 格式为 `Based on {submissionNo} · Page {n}`（或无 Page 部分） |
| UI-006 | 无 Affected Area | 卡片中不出现"Affected Area"或"影响区域"字样 |
| UI-007 | [New] 标签 | 未读时橙色胶囊；已读后消失 |
| UI-008 | Tab 气泡 | 未读 > 0 时显示橙色气泡；全部已读后消失 |

---

## 5. 非功能测试

| # | 测试项 | 通过标准 |
|---|--------|---------|
| NF-001 | 列表首屏加载 | ≤ 1.5s（100 条以内，正常网络） |
| NF-002 | 图纸文件加载 | < 10MB PDF ≤ 3s（正常网络） |
| NF-003 | Confirm / Mark as Read 响应 | ≤ 500ms P95 |
| NF-004 | 越权访问 | SE A 无法通过 URL 直接访问未分配给自己的 DrawingVersion 详情页（后端返回 403） |
| NF-005 | 跨端幂等 | 同一用户对同一 DrawingVersion 执行两次 Confirm Reading，后端 drawing_confirmation 只有一条记录 |
| NF-006 | 通知推送延迟 | Markup 发布后 ≤ 30s 铃铛角标更新（轮询机制） |

---

## 6. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-005-pc.md](../../../requirements/pc/REQ-005-pc.md) | PC 端产品需求（v0.3.5） |
| [UI-REQ-005-pc.md](../../ui/pc/UI-REQ-005-pc.md) | UI 设计规范 |
| [FRONTEND-REQ-005-pc.md](../../frontend/pc/FRONTEND-REQ-005-pc.md) | 前端实现规范 |
| [BACKEND-REQ-005.md](../../backend/BACKEND-REQ-005.md) | 后端接口规范 |
