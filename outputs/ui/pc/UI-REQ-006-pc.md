# UI 说明文档 — PC 端图纸二维码管理

> **来源需求**: [REQ-006-pc](../../../requirements/pc/REQ-006-pc.md) + [REQ-006-shared](../../../requirements/shared/REQ-006-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **设计令牌参考**: [UI-REQ-001-pc § 3.2](./UI-REQ-001-pc.md)
> **依赖文档**: [UI-REQ-003-pc.md](./UI-REQ-003-pc.md)（图纸管理与版本历史基础规范）
> **生成日期**: 2026-04-17

---

## 1. 设计目标

- 在 Drawing 团队现有工作流中**无感嵌入 QR 管理**：审批通过即自动生成，管理员无需额外点击
- 将 QR 信息集中在**版本历史抽屉**中展示，避免占用主列表宽度
- 通过 QR 详情抽屉将"QR 图像、下载操作、公开 URL、重新生成"等信息/操作整合在一个右侧面板内
- 对 QR 生成失败场景提供**显性错误提示**（⚠️ 图标 + hover 文案 + 快速重试入口）
- 保持与 REQ-003 图纸管理视觉一致性，所有组件复用 Element UI 基础样式

---

## 2. 页面清单与用户流程

### 2.1 扩展与新增视图

| 序号 | 视图 | 类型 | 说明 |
|------|------|------|------|
| 1 | 版本历史列表 — QR 列扩展 | 现有视图扩展 | 在 REQ-003-pc 的版本历史抽屉表格中新增 QR 列 |
| 2 | QR 详情抽屉 | 新增右侧抽屉 | 展示 QR 图像、生成时间、公开 URL、下载与重新生成操作 |
| 3 | Drawings 主列表 — QR 状态指示 | 现有视图扩展 | 当前版本 QR 未生成时在版本号旁显示 ⚠️ 图标 |
| 4 | 审批通过 Toast 文案调整 | 现有交互扩展 | 审批通过提示改为 "Drawing approved. QR code is being generated..." |

### 2.2 核心用户流程

```
Drawing 团队查看并下载 QR：

  Drawing Management 列表
       │ 点击图纸行 "Versions" 按钮
       ▼
  版本历史抽屉
  APPROVED 版本 QR 列显示 [View]
       │ 点击 [View]
       ▼
  QR 详情抽屉（右侧）
  展示 QR 图像、生成时间、公开 URL
       │ 点击 [Download QR Image] / [Download PDF with QR]
       ▼
  浏览器下载对应文件
```

```
Drawing 团队重新生成失败的 QR：

  Drawing Management 列表
  当前版本号旁显示 ⚠️ 图标
       │ Hover 显示 "QR not generated..."
       │ 点击 ⚠️ 或进入版本历史
       ▼
  版本历史抽屉
  QR 列显示 [⚠️ Not generated]
       │ 点击 [⚠️ Not generated]（需 drawing:upload 权限）
       ▼
  二次确认对话框 "Regenerate QR for this version?"
       │ 确认
       ▼
  加载态 → 成功 Toast → QR 列自动刷新为 [View]
```

```
审批通过后自动生成 QR（后台异步）：

  审批 Todo → 点击 Approve
       │
       ▼
  Toast 提示 "Drawing approved. QR code is being generated..."
  用户可继续其他操作
       │（后台异步）
       ▼
  版本历史中对应版本 QR 列显示 [View]（刷新后生效）
```

---

## 3. 组件设计规范

### 3.1 版本历史列表 — QR 列扩展

#### 3.1.1 扩展后的表格布局

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  Version History — ARCH-001 首层平面图                                     ×    │
├────────────────────────────────────────────────────────────────────────────────┤
│  Version │ Status    │ Uploaded    │ Approved   │ Confirmed │ QR               │
│──────────│───────────│─────────────│────────────│───────────│──────────────────│
│  V3      │ Approved  │ 2026-04-01  │ 2026-04-01 │ 5/12      │ [View]           │
│  V2      │ Approved  │ 2026-03-20  │ 2026-03-20 │ 12/12     │ [View]           │
│  V1      │ Approved  │ 2026-03-10  │ 2026-03-12 │ 12/12     │ ⚠️ Not generated │
│  —       │ Rejected  │ 2026-03-05  │ —          │ —         │ —                │
└────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 QR 列样式与交互

| 状态 | 展示 | 颜色 | 交互 |
|------|------|------|------|
| 已生成（`approvalStatus = APPROVED` 且 `qrGeneratedTime` 不为空） | `[View]` 文字链接 | `--color-primary`（蓝色），13px，hover 下划线 | 点击 → 打开 QR 详情抽屉 |
| 未生成（APPROVED 但 `qrGeneratedTime` 为空） | `⚠️ Not generated` | 橙色 `--color-warning` #F57F17，13px | Hover 显示 "QR generation failed or in progress. Click to regenerate."；点击触发重新生成（仅 `drawing:upload` 权限可点） |
| 非 APPROVED（PENDING / REJECTED） | `—` | `--color-text-secondary` 灰色 | 不可点击 |

> 列宽 120px，居中对齐。

---

### 3.2 QR 详情抽屉

#### 3.2.1 抽屉整体布局

```
┌────────────────────────────────────────┐
│  Drawing QR Code                     × │
├────────────────────────────────────────┤
│                                        │
│     ┌───────────────────────────┐      │
│     │                           │      │
│     │                           │      │
│     │    [QR IMAGE 240×240]     │      │
│     │                           │      │
│     │                           │      │
│     └───────────────────────────┘      │
│                                        │
│  Version         V3                    │
│  Generated       2026-04-01 14:35      │
│                                        │
│  Public URL                            │
│  ┌──────────────────────────────────┐  │
│  │ https://site.xxx.com/public/     │  │
│  │ drawing-status/a1b2c3d4e5f6...   │  │
│  │                          [Copy]  │  │
│  └──────────────────────────────────┘  │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │     Download QR Image (PNG)      │  │  ← 主按钮
│  └──────────────────────────────────┘  │
│  ┌──────────────────────────────────┐  │
│  │     Download PDF with QR         │  │  ← 次按钮
│  └──────────────────────────────────┘  │
│  ┌──────────────────────────────────┐  │
│  │     Regenerate QR                │  │  ← 次按钮（权限控制）
│  └──────────────────────────────────┘  │
│                                        │
└────────────────────────────────────────┘
```

#### 3.2.2 抽屉容器

| 属性 | 规格 |
|------|------|
| 组件 | `el-drawer`，direction="rtl"，size="480px" |
| 标题 | "Drawing QR Code"，16px，`--font-weight-semibold`，`--color-text-primary` |
| 关闭按钮 | 右上角 `×`，hover 背景 #F0F2F5 |
| 内边距 | 24px |

#### 3.2.3 QR 图像区

| 属性 | 规格 |
|------|------|
| 容器 | 居中对齐，上下外边距 8px |
| 图像尺寸 | 240 × 240 px，实际分辨率 512 × 512（保证打印清晰度） |
| 边框 | 1px `--color-border-base` #DCDFE6，圆角 4px |
| 加载状态 | 加载时显示骨架占位；加载失败显示占位图标（48px）+ "Failed to load QR" 灰色文字 |

#### 3.2.4 元信息区

| 字段 | 展示 | 样式 |
|------|------|------|
| Version | 版本号，如 "V3" | 左侧 Label 13px `--color-text-secondary`；右侧 Value 14px `--color-text-primary` |
| Generated | `qrGeneratedTime`，格式 "YYYY-MM-DD HH:mm" | 同上 |

- Label - Value 两列布局，Label 固定 88px 宽，12px 垂直间距

#### 3.2.5 Public URL 只读区

| 属性 | 规格 |
|------|------|
| 容器 | 灰色底 #F5F7FA，圆角 4px，内边距 12px |
| URL 文本 | `publicStatusUrl`，13px 等宽字体 `Monaco, Consolas, monospace`；最多 2 行，超出省略 |
| Copy 按钮 | 右下角对齐，`el-button size="mini" type="text"` "Copy"，`--color-primary` |
| 复制成功 | Element Toast "Copied to clipboard" |

#### 3.2.6 操作按钮

| 按钮 | 组件 | 样式 | 禁用规则 |
|------|------|------|----------|
| Download QR Image (PNG) | `el-button type="primary"` 全宽 | 主按钮，高度 40px | `qrImageUrl` 为空时置灰 |
| Download PDF with QR | `el-button type="default"` 全宽 | 次按钮，高度 40px | `pdfWithQrUrl` 为空时置灰，hover tooltip "Only PDF drawings support embedded QR" |
| Regenerate QR | `el-button type="default"` 全宽 | 次按钮，高度 40px；左侧前置 `el-icon-refresh` 图标 | 仅 `drawing:upload` 权限用户可见；loading 时禁用 |

#### 3.2.7 Regenerate 二次确认

点击 [Regenerate QR] 后弹出：

| 属性 | 规格 |
|------|------|
| 组件 | `this.$confirm()`，Element UI MessageBox |
| 标题 | "Regenerate QR" |
| 内容 | "Regenerate QR for version {versionNo}? The existing QR image and PDF will be replaced. The public URL stays unchanged." |
| 取消 | "Cancel" |
| 确认 | "Regenerate"，`type="primary"` |
| 成功后 | Toast "QR regenerated successfully"，抽屉内 QR 图像与 Generated 时间刷新 |
| 失败后 | Toast 错误提示（后端错误文案），按钮恢复可点击 |

---

### 3.3 Drawings 主列表 — QR 未生成状态指示

扩展 [UI-REQ-003-pc](./UI-REQ-003-pc.md) 的主列表：

#### 3.3.1 Version 列扩展

```
┌─────────────────────────────────────────────────┐
│  Version                                        │
│  V3                      ← 正常（QR 已生成）    │
│  V3 ⚠️                   ← QR 未生成（橙色）    │
└─────────────────────────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| ⚠️ 图标 | 12 × 12 px，橙色 `--color-warning` #F57F17 |
| 位置 | 版本号文字右侧，间距 4px |
| Hover tooltip | "QR not generated for current version. Click to view."；tooltip 位置 bottom |
| 点击 | 打开对应图纸的版本历史抽屉并滚动定位到当前版本行 |
| 无显示条件 | 当前版本 `qrGeneratedTime` 不为空时不显示图标 |

---

### 3.4 审批通过 Toast 文案

扩展 [UI-REQ-003-pc](./UI-REQ-003-pc.md) 中审批 Todo 操作反馈：

| 操作 | 原文案 | 新文案 |
|------|--------|--------|
| 审批通过（Approve） | "Drawing approved successfully." | "Drawing approved. QR code is being generated..." |
| 审批驳回（Reject） | 保持原文案 | 保持原文案 |

| 属性 | 规格 |
|------|------|
| 组件 | `this.$message({ type: 'success' })` |
| 持续时间 | 3000ms |

---

### 3.5 图纸在线预览 — 优先加载带 QR 的 PDF

扩展 [UI-REQ-003-pc § 图纸预览](./UI-REQ-003-pc.md)：

| 场景 | 文件来源 |
|------|---------|
| `pdfWithQrUrl` 不为空 | 优先加载 `pdfWithQrUrl` |
| `pdfWithQrUrl` 为空（非 PDF 或生成失败） | 降级加载原 `fileUrl` |

预览界面无额外 UI 变化，QR 自然出现在 PDF 右下角（距离底边和右边 15mm，尺寸 25mm × 25mm）。

---

## 4. 交互细节

### 4.1 QR 列与主列表 ⚠️ 图标的联动

| 动作 | 联动效果 |
|------|----------|
| 在 QR 详情抽屉中 [Regenerate QR] 成功 | 关闭成功 Toast；抽屉内 QR 图像与时间刷新；版本历史列表 QR 列同步刷新；若为当前版本，主列表 ⚠️ 图标消失 |
| 审批通过后 ~10 秒 | 后端异步完成 QR 生成；此时版本历史 QR 列显示 `[View]`（需刷新或重新打开抽屉） |
| 审批通过后 QR 生成失败 | 版本历史 QR 列显示 `⚠️ Not generated`；主列表版本号旁显示 ⚠️ 图标 |

### 4.2 权限视觉表现

| 权限 | QR 列表现 | QR 详情抽屉 |
|------|-----------|-------------|
| `drawing:view`（基础查看） | 可见 `[View]`；点击可打开抽屉 | 抽屉仅显示 QR 图像、元信息、Public URL、Download 按钮；**不显示** Regenerate QR 按钮 |
| `drawing:upload`（Drawing 团队） | 可见 `[View]` + `⚠️ Not generated` 可点击 | 抽屉完整显示所有按钮 |

### 4.3 响应式

| 断点 | 规则 |
|------|------|
| ≥ 1280px | 抽屉宽度 480px |
| ≥ 1440px | 抽屉宽度保持 480px（不跟随屏幕宽度增加） |
| < 1280px | 不做兼容支持（PC 端最小宽度约定） |

---

## 5. 设计令牌引用

| 令牌 | 值 | 使用位置 |
|------|----|---------|
| `--color-primary` | #409EFF | [View] 链接、[Copy] 按钮、主按钮、Public URL 复制 |
| `--color-warning` | #F57F17 | ⚠️ 图标、`⚠️ Not generated` 文字 |
| `--color-text-primary` | #303133 | 抽屉标题、元信息 Value |
| `--color-text-regular` | #606266 | 普通文字 |
| `--color-text-secondary` | #909399 | 元信息 Label、`—` 占位 |
| `--color-border-base` | #DCDFE6 | QR 图像边框 |
| `--color-border-light` | #EBEEF5 | 分隔线 |
| `--bg-page` | #F5F7FA | Public URL 只读区背景 |
| `--font-weight-semibold` | 600 | 标题文字 |

---

## 6. 国际化对照表

| Key | 英文 | 中文 |
|-----|------|------|
| qr.column.title | QR | 二维码 |
| qr.column.view | View | 查看 |
| qr.column.notGenerated | ⚠️ Not generated | ⚠️ 未生成 |
| qr.drawer.title | Drawing QR Code | 图纸二维码 |
| qr.drawer.version | Version | 版本 |
| qr.drawer.generated | Generated | 生成时间 |
| qr.drawer.publicUrl | Public URL | 公开链接 |
| qr.drawer.copy | Copy | 复制 |
| qr.drawer.copied | Copied to clipboard | 已复制到剪贴板 |
| qr.drawer.downloadImage | Download QR Image (PNG) | 下载二维码图片（PNG） |
| qr.drawer.downloadPdf | Download PDF with QR | 下载含二维码的 PDF |
| qr.drawer.regenerate | Regenerate QR | 重新生成二维码 |
| qr.drawer.pdfDisabledHint | Only PDF drawings support embedded QR | 仅 PDF 图纸支持内嵌二维码 |
| qr.regenerate.confirmTitle | Regenerate QR | 重新生成二维码 |
| qr.regenerate.confirmBody | Regenerate QR for version {versionNo}? The existing QR image and PDF will be replaced. The public URL stays unchanged. | 为版本 {versionNo} 重新生成二维码？将替换现有的二维码图片和 PDF，但公开链接保持不变。 |
| qr.regenerate.success | QR regenerated successfully | 二维码已重新生成 |
| qr.mainList.warningTooltip | QR not generated for current version. Click to view. | 当前版本的二维码未生成，点击查看 |
| approval.toast.approvedWithQr | Drawing approved. QR code is being generated... | 图纸已审批通过，二维码正在生成... |

---

## 7. 验收标准

### 版本历史 QR 列
- [ ] APPROVED 且 QR 已生成版本显示 `[View]` 蓝色链接
- [ ] APPROVED 但 QR 未生成显示 `⚠️ Not generated` 橙色文字
- [ ] PENDING / REJECTED 版本显示 `—` 灰色占位
- [ ] QR 列宽度 120px，居中对齐

### QR 详情抽屉
- [ ] 点击 `[View]` 从右侧滑出宽度 480px 抽屉
- [ ] 抽屉内 QR 图像渲染为 240 × 240 px，居中
- [ ] Version / Generated 字段显示正确数据
- [ ] Public URL 使用等宽字体，最多两行显示，可点击 [Copy] 复制
- [ ] [Download QR Image] 按钮为主样式，全宽
- [ ] [Download PDF with QR] 按钮为次样式；原始文件非 PDF 时置灰并显示 tooltip
- [ ] [Regenerate QR] 按钮仅对 `drawing:upload` 权限用户可见
- [ ] 点击 [Regenerate QR] 弹出二次确认 MessageBox

### 主列表状态指示
- [ ] 当前版本 QR 未生成时版本号旁显示 ⚠️ 橙色图标
- [ ] Hover ⚠️ 图标显示 tooltip 文案
- [ ] 点击 ⚠️ 打开对应图纸的版本历史抽屉
- [ ] QR 正常生成时版本号旁无额外图标

### 审批交互反馈
- [ ] 审批通过 Toast 文案为 "Drawing approved. QR code is being generated..."
- [ ] Toast 持续 3000ms 后自动消失

### 在线预览
- [ ] PDF 预览优先加载 `pdfWithQrUrl`；不存在时降级 `fileUrl`
- [ ] QR 出现在 PDF 右下角，不遮挡图纸主要内容

### 无障碍
- [ ] 所有按钮具备 aria-label 描述
- [ ] 抽屉关闭可通过 ESC 键触发
- [ ] Focus 状态具备可见的聚焦环（2px `--color-primary` 外描边）

---

## 8. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-006-shared.md](../../../requirements/shared/REQ-006-shared.md) | 二维码生成规则、公开状态 API |
| [REQ-006-pc.md](../../../requirements/pc/REQ-006-pc.md) | PC 端 QR 管理功能需求 |
| [UI-REQ-003-pc.md](./UI-REQ-003-pc.md) | 图纸管理基础 UI 规范（版本历史抽屉、主列表） |
| [UI-REQ-006-h5.md](../h5/UI-REQ-006-h5.md) | H5 端扫码后公开状态页 UI 规范 |
