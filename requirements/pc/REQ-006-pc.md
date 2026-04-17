# PC 端 — 图纸二维码管理

> **端**: PC 管理端（桌面浏览器 Web）
> **共享需求**: [REQ-006-shared.md](../shared/REQ-006-shared.md)（QR 生成规则、公开状态 API）
> **依赖需求**: [REQ-003-pc.md](./REQ-003-pc.md)（版本历史页扩展）
> **对应 H5 功能**: [REQ-006-h5.md](../h5/REQ-006-h5.md)（公开状态页）
> **本文档仅包含**: PC 端 QR 查看、下载、重新生成相关界面与交互

## 基本信息

- **需求ID**: REQ-006-pc
- **需求标题**: PC 端图纸二维码管理
- **产品**: SMART SITE SYSTEM
- **平台**: PC 管理端（桌面浏览器，Vue 2 + Element UI）
- **优先级**: 高
- **状态**: 草稿
- **依赖**: REQ-003-pc（版本历史列表），REQ-006-shared

---

## 需求描述

### 背景

图纸版本审批通过后系统自动生成二维码并叠加到 PDF（见 [REQ-006-shared §3.1](../shared/REQ-006-shared.md)）。PC 端需要向 Drawing 团队提供：
- 在版本历史中展示 QR 生成状态与入口
- 支持下载单独的 QR 图片（用于贴纸打印等场景）
- 支持下载已叠加 QR 的 PDF
- 在 QR 自动生成失败时提供重试入口

### 用户故事

```
作为 Drawing 团队成员
我想要 在 PC 端查看每个已审批版本的 QR 码和下载链接
以便 在需要单独打印 QR 贴纸或补发带 QR 的 PDF 时快速获取
```

```
作为 Drawing 团队成员
我想要 在 QR 生成失败时点击重新生成
以便 即使自动化流程偶尔失败也能恢复，不影响图纸流通
```

---

## 功能需求

### 1. 版本历史抽屉 — QR 信息扩展

扩展 [REQ-003-pc](./REQ-003-pc.md) 版本历史列表，在每条 APPROVED 版本记录中增加 QR 相关展示与操作。

#### 1.1 列表列扩展

在原版本历史表格末尾新增 **QR** 列：

```
┌────────────────────────────────────────────────────────────────────────┐
│ Version │ Status    │ Uploaded    │ Approved   │ Confirmed │ QR       │
│─────────│───────────│─────────────│────────────│───────────│──────────│
│ V3      │ Approved  │ 2026-04-01  │ 2026-04-01 │ 5/12      │ [View]   │
│ V2      │ Approved  │ 2026-03-20  │ 2026-03-20 │ 12/12     │ [View]   │
│ V1      │ Rejected  │ 2026-03-10  │ —          │ —         │ —        │
└────────────────────────────────────────────────────────────────────────┘
```

| 状态 | QR 列显示 |
|------|----------|
| `approvalStatus = APPROVED` 且 `qrGeneratedTime` 不为空 | `[View]` 文字链接按钮 |
| `approvalStatus = APPROVED` 且 `qrGeneratedTime` 为空（生成中或生成失败） | `⚠️ Not generated`（橙色，hover 显示 "QR generation failed or in progress"） |
| `approvalStatus = PENDING` / `REJECTED` | `—` |

- 点击 `[View]` 打开 **QR 详情抽屉**（§1.2）
- 点击 `⚠️ Not generated`（仅 `drawing:upload` 权限可点）直接触发 `/drawing/version/qr/regenerate`

#### 1.2 QR 详情抽屉

从右侧滑出，宽度 480px：

```
┌──────────────────────────────────────┐
│  Drawing QR Code                   × │
├──────────────────────────────────────┤
│                                      │
│       ┌────────────────────┐         │
│       │                    │         │
│       │                    │         │
│       │    [QR IMAGE]      │         │
│       │    240 × 240       │         │
│       │                    │         │
│       │                    │         │
│       └────────────────────┘         │
│                                      │
│  Version        V3                   │
│  Generated      2026-04-01 14:35     │
│                                      │
│  Public URL                          │
│  ┌────────────────────────────────┐  │
│  │ https://site.xxx.com/public/   │  │
│  │ drawing-status/a1b2c3d4...     │  │
│  │                         [Copy] │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │    Download QR Image (PNG)     │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │    Download PDF with QR        │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │    Regenerate QR               │  │
│  └────────────────────────────────┘  │
│                                      │
└──────────────────────────────────────┘
```

| 元素 | 规格 |
|------|------|
| QR 图像 | 展示 `qrImageUrl`，渲染尺寸 240 × 240px，居中；加载失败显示占位图标 + "Failed to load QR" |
| Version | 版本号（如 V3），只读展示 |
| Generated | `qrGeneratedTime`，格式 `YYYY-MM-DD HH:mm` |
| Public URL | 完整公开状态页 URL（`publicStatusUrl`），单行省略显示；右侧 `[Copy]` 按钮点击复制到剪贴板，Toast 提示 "Copied" |
| Download QR Image | 主按钮，下载 `qrImageUrl` 原图（PNG）；文件命名 `{drawingCode}-{versionNo}-qr.png` |
| Download PDF with QR | 次按钮，下载 `pdfWithQrUrl`；文件命名 `{drawingCode}-{versionNo}-with-qr.pdf`。若原始文件非 PDF，按钮置灰，hover 显示 "Only PDF drawings support embedded QR" |
| Regenerate QR | 次按钮；仅 `drawing:upload` 权限的用户可见；点击前弹出二次确认 "Regenerate QR for this version? The existing QR code will be replaced." |

#### 1.3 Regenerate QR 交互流程

1. 用户点击 `Regenerate QR` 按钮
2. 弹出 Element UI Dialog 二次确认
3. 确认后调用 `/drawing/version/qr/regenerate`（见 [REQ-006-shared §4.3](../shared/REQ-006-shared.md)）
4. 按钮进入 loading 态，禁用其他操作
5. 成功后：
   - Toast 提示 "QR regenerated successfully"
   - 刷新抽屉内 QR 图像与 `Generated` 时间
   - `publicToken` 不变，`Public URL` 保持不变
6. 失败后：
   - Toast 错误提示（展示后端返回的错误文案）
   - 按钮恢复可用

### 2. QR 生成状态指示（主列表）

在 [REQ-003-pc](./REQ-003-pc.md) 的 Drawings 主列表中，对每条图纸的**当前有效版本**增加 QR 生成状态微指示：

- 当前版本的 `qrGeneratedTime` 为空时，在版本号右侧显示 ⚠️ 小图标（12px）
- Hover 显示文案 "QR not generated for current version. Click to view."
- 点击跳转到该图纸的版本历史抽屉，定位到当前版本
- `qrGeneratedTime` 正常时无额外标识

### 3. 审批通过后的交互反馈

扩展 [REQ-003-pc](./REQ-003-pc.md) 中**审批 Todo 通过**操作的 Toast 提示：

- 原文案：`"Drawing approved successfully."`
- 新文案：`"Drawing approved. QR code is being generated..."`

> 用户无需等待 QR 生成完成即可继续其他操作；QR 在后台异步生成，完成后版本历史页会自动显示新生成的 QR。

### 4. 图纸在线预览 — 确认 QR 已叠加

扩展 [REQ-003-pc §3](./REQ-003-pc.md) 的图纸在线预览：
- 加载 PDF 时优先使用 `pdfWithQrUrl`，确保管理员看到的 PDF 与最终下载产物一致
- `pdfWithQrUrl` 不存在时降级为 `fileUrl`
- 预览区域无需额外标识，QR 自然出现在 PDF 右下角

---

## 验收标准

### 版本历史列表
- [ ] APPROVED 版本且 QR 已生成时，QR 列显示 `[View]` 链接
- [ ] APPROVED 但 QR 未生成时，QR 列显示 `⚠️ Not generated` 橙色提示
- [ ] PENDING / REJECTED 版本 QR 列显示 `—`
- [ ] 点击 `[View]` 正确打开 QR 详情抽屉

### QR 详情抽屉
- [ ] 抽屉正确展示 QR 图像、版本号、生成时间、公开 URL
- [ ] 点击 `[Copy]` 按钮成功复制 URL，Toast 提示 "Copied"
- [ ] `Download QR Image` 按钮下载 PNG 文件，文件名符合 `{drawingCode}-{versionNo}-qr.png`
- [ ] 原始文件为 PDF 时，`Download PDF with QR` 可用并下载带 QR 水印的 PDF
- [ ] 原始文件非 PDF（DWG/DXF/PNG/JPG）时，`Download PDF with QR` 置灰并显示 hover 提示
- [ ] 无 `drawing:upload` 权限的用户不可见 `Regenerate QR` 按钮

### Regenerate QR
- [ ] 点击 `Regenerate QR` 弹出二次确认 Dialog
- [ ] 确认后接口调用成功，抽屉内 QR 图像与生成时间更新
- [ ] 重新生成后 `Public URL` 保持不变（token 保留）
- [ ] 重新生成失败时按钮恢复并显示错误 Toast

### 主列表状态指示
- [ ] 当前版本 QR 未生成时，主列表版本号旁显示 ⚠️ 图标
- [ ] Hover ⚠️ 图标显示提示文案
- [ ] 点击 ⚠️ 图标跳转到对应图纸的版本历史并定位到当前版本

### 审批交互反馈
- [ ] 审批通过后 Toast 显示 QR 正在生成的提示文案

### 在线预览
- [ ] PDF 在线预览优先加载 `pdfWithQrUrl`
- [ ] `pdfWithQrUrl` 缺失时降级为 `fileUrl` 且预览正常

---

## 相关需求

| 文档 | 说明 |
|------|------|
| [REQ-006-shared.md](../shared/REQ-006-shared.md) | QR 生成规则、公开状态 API、数据模型扩展 |
| [REQ-006-h5.md](../h5/REQ-006-h5.md) | 扫码后的 H5 公开状态页 |
| [REQ-003-pc.md](./REQ-003-pc.md) | 图纸管理与版本历史基础页面 |
| [REQ-003-shared.md](../shared/REQ-003-shared.md) | 图纸共享业务规则与 API |
