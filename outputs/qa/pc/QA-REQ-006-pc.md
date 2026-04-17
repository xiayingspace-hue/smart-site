# QA 测试说明文档 — REQ-006 PC 端图纸二维码管理

> **来源需求**: REQ-006-shared + REQ-006-pc
> **平台**: PC 端（Chrome / Edge，1280px+）
> **生成日期**: 2026-04-17

---

## 1. 测试目标

验证 PC 端图纸二维码管理功能的正确性，包括版本历史 QR 列展示、QR 详情抽屉、下载操作、重新生成、主列表状态指示以及审批通过后的 Toast 文案。

---

## 2. 测试场景与用例

### 场景 1：版本历史列表 — QR 列展示

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-PC-01 | 图纸有 APPROVED 版本且 QR 已生成 | 打开版本历史抽屉 | QR 列显示蓝色 `[View]` 文字链接 |
| TC-006-PC-02 | 图纸有 APPROVED 版本但 QR 未生成（`qrGeneratedTime` 为空） | 打开版本历史抽屉 | QR 列显示橙色 `⚠️ Not generated`，hover 提示 "QR generation failed or in progress" |
| TC-006-PC-03 | 图纸版本为 PENDING | 打开版本历史抽屉 | QR 列显示 `—` |
| TC-006-PC-04 | 图纸版本为 REJECTED | 打开版本历史抽屉 | QR 列显示 `—` |

### 场景 2：QR 详情抽屉

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-PC-05 | QR 已生成的 APPROVED 版本 | 点击 `[View]` | 右侧 480px 抽屉滑出，展示 QR 图像（240×240）、版本号、生成时间、公开 URL |
| TC-006-PC-06 | QR 详情抽屉已打开 | 点击 Public URL 右侧 `[Copy]` 按钮 | URL 复制到剪贴板，Toast 提示 "Copied" |
| TC-006-PC-07 | QR 详情抽屉已打开 | 点击 `[Download QR Image]` | 浏览器下载 PNG 文件，文件名为 `{drawingCode}-{versionNo}-qr.png` |
| TC-006-PC-08 | 原始文件为 PDF 的版本 | 点击 `[Download PDF with QR]` | 浏览器下载 PDF 文件，文件名为 `{drawingCode}-{versionNo}-with-qr.pdf`，PDF 每页右下角有 QR |
| TC-006-PC-09 | 原始文件为 DWG / DXF / PNG / JPG 的版本 | 查看 `[Download PDF with QR]` 按钮 | 按钮置灰不可点击，hover 提示 "Only PDF drawings support embedded QR" |
| TC-006-PC-10 | QR 图像加载失败 | 查看 QR 图像区域 | 显示占位图标 + "Failed to load QR" 文案 |

### 场景 3：重新生成 QR

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-PC-11 | 用户具有 `drawing:upload` 权限 | 在 QR 详情抽屉中查看 | `[Regenerate QR]` 按钮可见 |
| TC-006-PC-12 | 用户不具有 `drawing:upload` 权限 | 在 QR 详情抽屉中查看 | `[Regenerate QR]` 按钮不可见 |
| TC-006-PC-13 | 有权限用户 | 点击 `[Regenerate QR]` | 弹出二次确认 Dialog："Regenerate QR for this version? The existing QR code will be replaced." |
| TC-006-PC-14 | 二次确认 Dialog | 点击确认 | 按钮进入 loading 态，成功后 Toast "QR regenerated successfully"，QR 图像和生成时间刷新，Public URL 不变 |
| TC-006-PC-15 | 重新生成失败 | 确认后等待结果 | 按钮恢复可用，Toast 显示后端错误文案 |
| TC-006-PC-16 | QR 未生成的版本（⚠️ Not generated） | 有权限用户点击 `⚠️ Not generated` | 直接触发 regenerate 接口（无需进入 QR 详情抽屉） |
| TC-006-PC-17 | QR 未生成的版本（⚠️ Not generated） | 无权限用户点击 `⚠️ Not generated` | 无响应（不可点击） |

### 场景 4：主列表 QR 状态指示

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-PC-18 | 图纸当前版本的 `qrGeneratedTime` 为空 | 查看 Drawings 主列表 | 版本号右侧显示 ⚠️ 小图标（12px） |
| TC-006-PC-19 | 同上 | Hover ⚠️ 图标 | 显示 "QR not generated for current version. Click to view." |
| TC-006-PC-20 | 同上 | 点击 ⚠️ 图标 | 跳转到该图纸的版本历史抽屉，定位到当前版本 |
| TC-006-PC-21 | 图纸当前版本 QR 已正常生成 | 查看 Drawings 主列表 | 版本号旁无额外标识 |

### 场景 5：审批通过后交互反馈

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-PC-22 | 审批人审批一份图纸 | 点击 Approve 按钮 | Toast 显示 "Drawing approved. QR code is being generated..."（而非旧文案 "Drawing approved successfully."） |
| TC-006-PC-23 | 审批通过后等待异步生成完成 | 刷新版本历史 | 对应版本 QR 列从空变为 `[View]` |

### 场景 6：图纸在线预览

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-PC-24 | APPROVED 版本，原始文件为 PDF，QR 已生成 | 在线预览该版本图纸 | 加载 `pdfWithQrUrl`，PDF 右下角可见 QR 码 |
| TC-006-PC-25 | APPROVED 版本，`pdfWithQrUrl` 为空（非 PDF 或生成失败） | 在线预览该版本图纸 | 降级加载 `fileUrl`，预览正常 |

---

## 3. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-006-PC-01 | QR 列宽度 | 120px，居中对齐 |
| UI-006-PC-02 | `[View]` 链接样式 | Element UI text 类型按钮，蓝色 |
| UI-006-PC-03 | `⚠️ Not generated` 样式 | 橙色文字 |
| UI-006-PC-04 | QR 详情抽屉宽度 | 480px，从右侧滑出 |
| UI-006-PC-05 | QR 图像尺寸 | 240 × 240px，居中显示 |
| UI-006-PC-06 | Regenerate 按钮 | 仅 `drawing:upload` 权限用户可见 |

---

## 4. 非功能测试

| 类型 | 测试点 |
|------|-------|
| 权限 | 无 `drawing:upload` 权限用户调用 `/drawing/version/qr/regenerate` 返回 403 |
| 权限 | 无 `drawing:view` 权限用户调用 `/drawing/version/qr` 返回 403 |
| 安全 | 直接访问 `/drawing/version/qr?drawingVersionId=999`（不存在的版本）返回 1003006004 |
| 安全 | 未审批版本调用 regenerate 返回 1003006002 |

---

## 5. 验收标准

- [ ] APPROVED + QR 已生成的版本在版本历史 QR 列显示 `[View]`
- [ ] APPROVED + QR 未生成的版本显示 `⚠️ Not generated` 橙色提示
- [ ] QR 详情抽屉正确展示 QR 图像、生成时间、公开 URL
- [ ] Copy URL 功能正常，Toast 提示 "Copied"
- [ ] Download QR Image / Download PDF with QR 正确下载，文件名符合规范
- [ ] 非 PDF 文件的 Download PDF with QR 按钮置灰
- [ ] Regenerate QR 保留原 publicToken，刷新图像和时间
- [ ] 主列表 ⚠️ 图标在 QR 未生成时正确显示
- [ ] 审批通过后 Toast 文案包含 QR 生成提示
- [ ] PDF 在线预览优先加载 `pdfWithQrUrl`
