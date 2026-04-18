# QA 测试说明文档 — REQ-007 PC 端图纸两级审批流程

> **来源需求**: REQ-007-shared + REQ-007-pc
> **平台**: PC 端（Chrome / Edge，1280px+）
> **生成日期**: 2026-04-17

---

## 1. 测试目标

验证 PC 端图纸两级审批（内部审批 + 外部审批）完整流程的正确性，包括：上传变更、内部审批、DC 外部审批标记、版本历史 4 阶段展示、DC 配置管理、状态流转、权限控制及通知机制。

---

## 2. 测试角色与账号

| 角色 | 测试账号 | 权限 |
|------|---------|------|
| 设计人员 (Designer) | `designer_user` | `drawing:upload` |
| 内部审批人 | `approver_user` | `drawing:approve` |
| Document Controller (DC) | `dc_user1`, `dc_user2` | `drawing:external-approval` |
| DC 配置管理员 | `admin_user` | `drawing:dc-config` |
| Site Engineer | `se_user1`, `se_user2` | `drawing:confirm`, `drawing:view` |
| 无权限用户 | `no_perm_user` | 无 drawing 相关权限 |

---

## 3. 测试场景与用例

### 场景 1：上传图纸变更

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-007-PC-01 | designer_user 登录，图纸 ARCH-001 已存在（Active） | 点击 Upload New Version，确认审批人标签显示 | 标签显示 "Internal Approver *" |
| TC-007-PC-02 | 同上 | 上传文件，选择 Internal Approver，点击 Submit | 上传成功，Toast 显示 "Drawing uploaded successfully. Pending internal approval." |
| TC-007-PC-03 | 图纸 ARCH-001 已有版本处于 PENDING_INTERNAL | designer_user 尝试点击 Upload New Version | 按钮置灰不可点击 |
| TC-007-PC-04 | 图纸 ARCH-001 已有版本处于 INTERNAL_APPROVED（待外部审批） | designer_user 尝试点击 Upload New Version | 按钮置灰不可点击 |
| TC-007-PC-05 | 图纸 ARCH-001 最新版本为 INTERNAL_REJECTED | designer_user 尝试点击 Upload New Version | 按钮可用，可正常上传新版本 |
| TC-007-PC-06 | 图纸 ARCH-001 最新版本为 EXTERNAL_REJECTED | designer_user 尝试点击 Upload New Version | 按钮可用，可正常上传新版本 |

### 场景 2：内部审批

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-007-PC-07 | approver_user 登录，有 PENDING_INTERNAL 版本 | 查看 Todo 列表 | Todo 卡片显示 🔍 图标 + "Internal Approval Required" |
| TC-007-PC-08 | 同上 | 点击 [Approve] | 弹出确认 Dialog："Once approved, this version will proceed to external approval by Document Controller." |
| TC-007-PC-09 | 同上 | 确认 Approve | 版本状态 → INTERNAL_APPROVED；图纸状态 → PENDING_EXTERNAL；Toast 显示 "Internal approval completed. DC has been notified for external approval." |
| TC-007-PC-10 | 同上 | 确认 Approve 后检查 | SE **未收到**通知；QR **未生成** |
| TC-007-PC-11 | 同上 | 确认 Approve 后检查 DC Todo 列表 | dc_user1 和 dc_user2 的 Todo 列表均出现 🌐 "External Approval Required" 任务 |
| TC-007-PC-12 | approver_user，有 PENDING_INTERNAL 版本 | 点击 [Reject]，填写驳回意见 "轴网标注不完整"，确认 | 版本状态 → INTERNAL_REJECTED；designer_user 收到站内消息 "Drawing Internally Rejected"，含驳回意见 |
| TC-007-PC-13 | approver_user，有 PENDING_INTERNAL 版本 | 点击 [Reject]，不填写驳回意见，点确认 | 提示 "Comment is required"，不允许提交 |
| TC-007-PC-14 | 非指定审批人登录 | 直接调用 `/drawing/approve` API | 返回 403 |

### 场景 3：DC 外部审批 — 通过

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-007-PC-15 | dc_user1 登录，有 INTERNAL_APPROVED 版本的外部审批 Todo | 查看 Todo 卡片 | 显示 🌐 图标、图纸信息、"Internal approved by" 行、[📄 Download Original] 和 [✅ Mark Result] 按钮 |
| TC-007-PC-16 | 同上 | 点击 [📄 Download Original] | 浏览器下载原始图纸文件，文件名为原始 fileName |
| TC-007-PC-17 | 同上 | 点击 [✅ Mark Result] | 打开 "Mark External Approval Result" Dialog，显示图纸编号/名称/版本 |
| TC-007-PC-18 | Dialog 打开，未选择 Result | 检查 [Confirm] 按钮 | 按钮 disabled |
| TC-007-PC-19 | 选择 Approved | 检查表单 | 显示：Signed Drawing File *、Approval Evidence *、External Approval Date *、Remarks |
| TC-007-PC-20 | 选择 Approved，不上传签字版 | 点击 Confirm | 提示 "Signed drawing file is required" |
| TC-007-PC-21 | 选择 Approved，不上传凭证 | 点击 Confirm | 提示 "Approval evidence is required" |
| TC-007-PC-22 | 选择 Approved，不选日期 | 点击 Confirm | 提示 "External approval date is required" |
| TC-007-PC-23 | 选择 Approved，日期选择器 | 尝试选择未来日期 | 未来日期不可选（禁用） |
| TC-007-PC-24 | 选择 Approved，上传签字版（60MB） | 上传 | 提示文件超过 50MB 限制 |
| TC-007-PC-25 | 选择 Approved，上传凭证（25MB） | 上传 | 提示文件超过 20MB 限制 |
| TC-007-PC-26 | 选择 Approved，填写完整必填信息 | 点击 Confirm | 按钮变为 "Processing..." loading 态，Dialog 内所有字段 disabled |
| TC-007-PC-27 | 同上，等待后端完成 | 等待 3-5 秒 | Dialog 关闭；Toast 显示 "External approval marked. Drawing is now active and QR code has been generated." |
| TC-007-PC-28 | 同上 | 检查图纸状态 | 版本 approvalStatus = APPROVED；图纸 status = ACTIVE；QR 已生成 |
| TC-007-PC-29 | 同上 | 检查 SE 通知 | se_user1, se_user2 收到 App Push + 站内通知 |
| TC-007-PC-30 | 同上 | 检查 dc_user2 的 Todo | dc_user2 的该外部审批 Todo 自动消失 |

### 场景 4：DC 外部审批 — 驳回

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-007-PC-31 | dc_user1 登录，外部审批 Dialog | 选择 Rejected | 显示 Rejection Reason *（必填文本域） |
| TC-007-PC-32 | 选择 Rejected，不填原因 | 点击 Confirm | 提示 "Rejection reason is required" |
| TC-007-PC-33 | 选择 Rejected，填写原因 "业主要求修改立面" | 点击 Confirm | Dialog 关闭；Toast 显示 "External rejection recorded. Designer has been notified." |
| TC-007-PC-34 | 同上 | 检查 | 版本状态 → EXTERNAL_REJECTED；designer_user 收到 "Drawing Externally Rejected" 站内消息，含驳回原因 |
| TC-007-PC-35 | 同上 | 检查 dc_user2 的 Todo | dc_user2 的该外部审批 Todo 自动消失 |

### 场景 5：DC 外部审批 — 异常

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-007-PC-36 | 模拟 QR 生成失败（如 OSS 不可用） | dc_user1 标记 Approved 并提交 | loading 恢复正常态；Toast 显示 "QR generation failed, please retry"；版本状态保持 INTERNAL_APPROVED |
| TC-007-PC-37 | dc_user1 和 dc_user2 同时点击 Mark Result 并提交 | 两人同时 Confirm Approved | 一人成功，另一人收到错误提示 "This version is not pending external approval"（乐观锁） |
| TC-007-PC-38 | no_perm_user 登录 | 直接调用 `/drawing/external-approve` API | 返回 403 |
| TC-007-PC-39 | 有 `drawing:external-approval` 权限但非项目 DC | 调用 `/drawing/external-approve` API | 返回错误码 1003007003 |

### 场景 6：版本历史抽屉

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-007-PC-40 | ARCH-001 有 V1(INTERNAL_REJECTED) + V2(EXTERNAL_REJECTED) + V3(APPROVED) | 点击 [History] | 抽屉宽度 720px；显示 3 行版本 |
| TC-007-PC-41 | 同上 | 检查状态标签 | V3: ✅ Approved 绿色；V2: ❌ Ext. Rejected 红色；V1: ❌ Int. Rejected 红色 |
| TC-007-PC-42 | 同上 | 点击 ▶ 展开 V3 | 显示 4 阶段卡片（均完成）+ 步骤条 "📄 ✓ → 🔍 ✓ → 🌐 ✓ → ✍️ ✓" |
| TC-007-PC-43 | V3 展开 | 检查 ① Uploaded 卡片 | 显示文件名、设计人员、时间、大小、[Preview] [Download] |
| TC-007-PC-44 | V3 展开 | 检查 ② Internal Approval 卡片 | 显示 ✅ Approved、审批人、时间、Comment |
| TC-007-PC-45 | V3 展开 | 检查 ③ External Approval 卡片 | 显示 ✅ Approved、DC 姓名、审批日期、凭证文件名 [Download]、Remark |
| TC-007-PC-46 | V3 展开 | 检查 ④ Signed Version 卡片 | 显示签字版文件名、大小、上传时间、[Preview] [Download] |
| TC-007-PC-47 | V3 展开 | 点击 ① [Preview] | 在线预览原始文件 |
| TC-007-PC-48 | V3 展开 | 点击 ④ [Download] | 下载签字版文件 |
| TC-007-PC-49 | 同上 | 点击 ▶ 展开 V1（INTERNAL_REJECTED） | ①②卡片有数据；③④卡片显示 "— Not reached" |
| TC-007-PC-50 | 同上 | 展开 V1 步骤条 | "📄 ✓ → 🔍 ✕ → 🌐 — → ✍️ —" |
| TC-007-PC-51 | 同上 | 点击 ▶ 展开 V2（EXTERNAL_REJECTED） | ①②③有数据；④显示 "— Not reached"；③显示驳回原因 |
| TC-007-PC-52 | 有 PENDING_EXTERNAL 版本 | 展开该版本 | ③ 卡片显示 ⏳ Pending + [✅ Mark Result]（dc_user 可见）；④ 卡片显示 "Waiting for external approval" |
| TC-007-PC-53 | 同上，approver_user 登录（非 DC） | 展开该版本 | ③ 卡片**不显示** [✅ Mark Result] 按钮 |
| TC-007-PC-54 | 有 PENDING_INTERNAL 版本 | 展开该版本 | ② 显示 ⏳ Pending；③④显示 "— Waiting for internal approval" |

### 场景 7：图纸列表状态标签

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-007-PC-55 | 图纸列表有各状态图纸 | 检查 Status 列 | PENDING_INTERNAL: 橙色 "Pending Internal"；PENDING_EXTERNAL: 橙色 "Pending External"；ACTIVE: 绿色 "Active"；INTERNAL_REJECTED: 红色 "Int. Rejected"；EXTERNAL_REJECTED: 红色 "Ext. Rejected" |
| TC-007-PC-56 | 搜索面板 Status 下拉 | 展开选项 | 显示 6 个选项：All / Active / Pending Internal / Pending External / Int. Rejected / Ext. Rejected |
| TC-007-PC-57 | 选择 "Pending External" 筛选 | 点击 Search | 仅显示状态为 PENDING_EXTERNAL 的图纸 |

### 场景 8：DC 配置管理

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-007-PC-58 | admin_user 登录 | 导航到 Settings → DC Configuration | 页面正确加载，显示当前项目 DC 列表 |
| TC-007-PC-59 | 同上 | 点击 [+ Add DC] | 弹窗显示有 `drawing:external-approval` 权限且未配置的用户列表 |
| TC-007-PC-60 | Add DC 弹窗 | 搜索 "王" | 列表过滤，仅显示姓名含 "王" 的用户 |
| TC-007-PC-61 | 同上 | 点击 [Add] | 用户添加成功，弹窗中该用户消失，外层表格新增一行 |
| TC-007-PC-62 | DC 列表有 2 人 | 点击 dc_user1 的 [Remove] | 弹出确认 "Remove {name} from DC list?" |
| TC-007-PC-63 | 确认移除 | 检查 | dc_user1 从列表消失 |
| TC-007-PC-64 | DC 列表仅剩 1 人 | 移除最后一人 | 移除成功，显示 ⚠️ 警告 "No DC configured..." |
| TC-007-PC-65 | no_perm_user 登录 | 导航到 DC Configuration 路由 | 无权限，页面不可访问 |
| TC-007-PC-66 | DC 列表为空 | 内部审批通过 | 版本状态正常变为 INTERNAL_APPROVED，但无 DC Todo 创建（可能有系统警告） |

### 场景 9：完整流程 E2E

| ID | 测试步骤 | 预期结果 |
|----|---------|---------|
| TC-007-PC-67 | ① designer_user 上传新图纸，指定 approver_user 为 Internal Approver | 图纸创建，版本 V1 状态 PENDING_INTERNAL |
| TC-007-PC-68 | ② approver_user 在 Todo 中 Approve | 版本 → INTERNAL_APPROVED，dc_user1/dc_user2 收到 Todo |
| TC-007-PC-69 | ③ dc_user1 下载原始文件 | 文件下载成功 |
| TC-007-PC-70 | ④ dc_user1 点击 Mark Result → Approved → 上传签字版 + 凭证 + 选日期 → Confirm | Loading 3-5s → 成功；版本 → APPROVED；QR 已生成；SE 收到推送 |
| TC-007-PC-71 | ⑤ se_user1 在 APP/PC 查看图纸 | 看到的是签字版文件（signedFileUrl） |
| TC-007-PC-72 | ⑥ 检查版本历史 V1 展开 | 4 阶段均完成，步骤条全绿 |

### 场景 10：驳回后重新上传 E2E

| ID | 测试步骤 | 预期结果 |
|----|---------|---------|
| TC-007-PC-73 | ① 上传 V1 → 内部审批通过 → DC 标记外部驳回 | V1 状态 EXTERNAL_REJECTED，designer 收到通知 |
| TC-007-PC-74 | ② designer_user 上传 V2 | V2 状态 PENDING_INTERNAL |
| TC-007-PC-75 | ③ approver_user 内部驳回 V2 | V2 状态 INTERNAL_REJECTED，designer 收到通知 |
| TC-007-PC-76 | ④ designer_user 上传 V3 → 内部通过 → DC 外部通过 | V3 状态 APPROVED，V1 isDeprecated |
| TC-007-PC-77 | ⑤ 查看版本历史 | V1: Ext. Rejected；V2: Int. Rejected；V3: Approved |

---

## 4. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-007-PC-01 | 上传弹窗 | 审批人标签显示 "Internal Approver *" |
| UI-007-PC-02 | 内部审批 Todo 卡片 | 🔍 图标 + "Internal Approval Required" |
| UI-007-PC-03 | DC 外部审批 Todo 卡片 | 🌐 图标 + "External Approval Required" + "Internal approved by" 行 |
| UI-007-PC-04 | 外部审批 Dialog | 宽度 560px，Approved/Rejected 分支显隐正确 |
| UI-007-PC-05 | 外部审批 Dialog loading | Confirm 按钮变 "Processing..."，表单 disabled |
| UI-007-PC-06 | 版本历史抽屉 | 宽度 720px |
| UI-007-PC-07 | 版本历史展开 | 4 阶段卡片 2×2 布局 + 底部步骤条 |
| UI-007-PC-08 | 步骤条颜色 | 完成=绿，进行中=橙，失败=红，未到达=灰 |
| UI-007-PC-09 | 状态标签 | 5 种状态颜色正确（橙/绿/红） |
| UI-007-PC-10 | DC 配置页 | 表格 + [+ Add DC] + [Remove] + 底部提示文字 |
| UI-007-PC-11 | Status 筛选下拉 | 6 个选项，宽度 170px |

---

## 5. 非功能测试

| 类型 | 测试点 | 预期结果 |
|------|-------|---------|
| 并发 | 两个 DC 同时标记同一版本 Approved | 乐观锁：一个成功，另一个提示错误 |
| 超时 | 外部审批通过后 QR 生成超过 10s | 前端 30s timeout 内等待；若后端超时，Toast 报错 |
| 安全 | 非 DC 用户直接调用 `/drawing/external-approve` | 返回 403 / 错误码 1003007003 |
| 安全 | 非 dc-config 权限用户访问 DC 配置页 | 页面不可访问（路由守卫） |
| 幂等 | DC 配置重复添加同一用户 | 唯一键冲突静默处理 |
| 兼容性 | Chrome 最新版 + Edge 最新版 | 所有功能正常 |

---

## 6. 测试数据

| 数据 | 值 |
|------|---|
| 测试项目 | Project-A（已配置 dc_user1, dc_user2 为 DC） |
| 测试图纸 | ARCH-001（结构类，已有 V1 ACTIVE 版本） |
| 签字版测试文件 | `test-signed.pdf`（5MB，PDF 格式） |
| 凭证测试文件 | `test-evidence.png`（2MB，PNG 格式） |
| 超大文件 | `large-file.pdf`（60MB，用于超限测试） |
