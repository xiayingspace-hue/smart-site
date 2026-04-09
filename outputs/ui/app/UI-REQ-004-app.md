# UI 说明文档 — APP 移动端图纸局部更新（Markup）

> **来源需求**: [REQ-004-app](../../../requirements/app/REQ-004-app.md) + [REQ-004-shared](../../../requirements/shared/REQ-004-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: APP 移动端（iOS & Android，UNIAPP + Vue 2）
> **设计令牌参考**: [UI-REQ-001-app § 2](./UI-REQ-001-app.md)
> **依赖文档**: [UI-REQ-003-app.md](./UI-REQ-003-app.md)（图纸列表页基础规范）
> **生成日期**: 2026-04-08

---

## 1. 设计目标

- 局部更新以「增量补丁」的视觉语言呈现，与完整版本在视觉层级上明确区分
- 未读局部更新通过橙色角标和 `[New]` 标签主动提醒，确保不遗漏
- Mark as Read 操作一步完成，符合现场单手操作习惯
- 附件查看流畅，支持在线预览，无需下载
- 视觉风格延续 REQ-003-app 图纸模块体系，保持统一
- 支持多语言（中文 / English），默认英文界面

---

## 2. 页面清单与用户流程

### 2.1 新增 / 调整的页面

| 序号 | 页面 | 类型 | 说明 |
|------|------|------|------|
| 1 | 图纸列表页（调整） | 既有页面扩展 | 图纸卡片新增局部更新角标 |
| 2 | 图纸详情页 Markups Tab（新增） | Tab 页 | 展示图纸当前所有 ACTIVE 局部更新列表 |
| 3 | 局部更新详情页（新增） | 独立页面 | 完整展示单条局部更新内容，含 Mark as Read 操作 |

### 2.2 核心用户流程

```
收到 App Push 通知：

  手机通知栏
  "Drawing Update — ARCH-001"
  "A轴节点详图修正. Please review..."
         │ 点击通知
         ▼
  图纸详情页（ARCH-001）
  自动切换到 [Markups (1)] Tab
         │ 点击局部更新卡片
         ▼
  局部更新详情页
  查看说明、影响区域、附件
         │ 点击 [Mark as Read]
         ▼
  按钮变为 ✓ You've confirmed reading
  返回 Markups Tab → 卡片 [New] 标签消失
```

```
主动查看局部更新：

  图纸列表页
  图纸卡片右上角橙色角标
         │ 点击图纸卡片
         ▼
  图纸详情页 → Tab 切换到 [Markups]
  局部更新列表（按发布时间降序）
         │
  点击卡片 → 局部更新详情页
```

---

## 3. 页面设计规范

### 3.1 图纸列表页（扩展）

#### 3.1.1 图纸卡片未读角标

在 UI-REQ-003-app 图纸卡片基础上，新增局部更新未读角标：

```
┌─────────────────────────────────────────────┐
│  ARCH-001  首层平面图                    ●  │  ← 橙色小圆点（右上角）
│  Architectural  V3  🟢 Active               │
│  Confirms: 8 / 10                           │
│                               [Confirm →]   │
└─────────────────────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| 角标形式 | 实心圆点，直径 8px，颜色 `--color-warning`（#F57F17） |
| 位置 | 卡片右上角，绝对定位，距右 12px，距上 12px |
| 显示条件 | 该图纸有至少 1 条未被当前用户确认的 ACTIVE 局部更新 |
| 消失条件 | 当前用户确认该图纸所有 ACTIVE 局部更新后消失 |

---

### 3.2 图纸详情页（扩展）

#### 3.2.1 新增 Markups Tab

在 UI-REQ-003-app 图纸详情页的 Tab 栏中新增 Markups Tab：

```
┌─────────────────────────────────────────────┐
│  ARCH-001  首层平面图                        │
│  Architectural  ·  V3  ·  🟢 Active         │
├─────────────────────────────────────────────┤
│  [Info]      [Markups  ●2]                  │  ← Tab 切换
├─────────────────────────────────────────────┤
```

| 属性 | 规格 |
|------|------|
| Tab 标签文字 | "Markups"；有未读时在文字右侧显示橙色计数气泡（如 `●2`） |
| 未读气泡 | 橙色圆形，最小宽度 16px，高度 16px，白色文字，10px；未读数 = 0 时隐藏气泡 |
| 激活跳转 | 点击通知跳转时，详情页自动激活 Markups Tab |

#### 3.2.2 Markups Tab — 局部更新列表

```
┌─────────────────────────────────────────────┐
│  ┌───────────────────────────────────────┐  │
│  │  [New]  A轴节点详图修正               │  │  ← 未读标记
│  │  Apr 8, 2026  ·  张三                 │  │
│  │  [A-C轴 / 3-5层]                      │  │
│  │  A轴与3轴交叉节点详图已更新，新增      │  │
│  │  钢筋排布说明，请对照施工。            │  │
│  │  📎 node-detail.pdf                  │  │
│  │                   [✓ Mark as Read]    │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │  C区消防管道路由修正                   │  │
│  │  Apr 7, 2026  ·  张三                 │  │
│  │  [C区 / B1层]                         │  │
│  │  消防主管道路由变更，详见附图...       │  │
│  │  📎 route-update.png                 │  │
│  │                          ✓ Read      │  │  ← 已确认
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

#### 3.2.3 局部更新卡片规格

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF，圆角 8px，阴影 0 1px 4px rgba(0,0,0,0.06) |
| 内边距 | 12px 16px |
| 卡片间距 | 8px |
| [New] 未读标签 | 橙色胶囊，背景 #FF6D00，文字 "New"，白色 10px，内边距 2px 6px；位于标题左侧 |
| 标题 | 15px，`--font-weight-semibold`，`--color-text-regular` |
| 元信息行 | "{date} · {creatorName}"，12px，`--color-text-secondary`，标题下方 4px |
| 影响区域 Tag | 圆角 4px，背景 #F5F5F5，文字色 #616161，12px，内边距 2px 8px |
| 说明文字 | 13px，`--color-text-regular`，最多 3 行，超出省略显示（点击卡片进入详情查看完整内容） |
| 附件列表 | 📎 图标（12px，`--color-text-secondary`）+ 文件名，12px，`--color-primary`，点击跳转预览 |
| [Mark as Read] 按钮 | 见 §3.2.4 |
| ✓ Read | 见 §3.2.5 |
| 点击整张卡片 | 跳转局部更新详情页 |

#### 3.2.4 [Mark as Read] 按钮（卡片内，未确认状态）

| 属性 | 规格 |
|------|------|
| 类型 | 文字按钮 + 图标 |
| 图标 | ✓ checkmark，14px |
| 文字 | "Mark as Read"，13px，`--color-primary` |
| 位置 | 卡片右下角，右对齐 |
| 点击效果 | loading 防重复；成功后即时切换为已确认样式 |

#### 3.2.5 已确认状态（卡片内）

| 属性 | 规格 |
|------|------|
| 显示内容 | "✓ Read"，13px，颜色 `--color-success`（#2E7D32） |
| 位置 | 卡片右下角，右对齐，替换 [Mark as Read] 按钮 |
| 卡片整体 | 无其他视觉变化（不降低透明度，保持可读性） |

---

### 3.3 局部更新详情页

#### 3.3.1 页面结构

```
┌─────────────────────────────────────────────┐
│  ←  A轴节点详图修正                          │  ← 顶部导航栏（返回 + 标题）
├─────────────────────────────────────────────┤
│  ARCH-001  首层平面图  ·  V3                 │  ← 图纸信息行
│  Published by 张三  ·  Apr 8, 2026          │  ← 发布人和时间
├─────────────────────────────────────────────┤
│                                             │
│  Affected Area                              │  ← section 标题
│  ┌─────────────────────────────────────┐   │
│  │  A-C轴 / 3-5层                      │   │  ← 灰色区块
│  └─────────────────────────────────────┘   │
│                                             │
│  Description                               │
│  A轴与3轴交叉节点详图已更新，新增钢筋       │
│  排布说明，请对照最新节点详图施工，避免      │
│  按旧图操作造成返工。                        │
│                                             │
│  Attachments (1)                            │
│  ┌─────────────────────────────────────┐   │
│  │  📄 node-detail.pdf          512KB │   │
│  │                     [Preview / Open]│   │
│  └─────────────────────────────────────┘   │
│                                             │
└─────────────────────────────────────────────┘
┌─────────────────────────────────────────────┐
│         [✓ Mark as Read]                    │  ← 底部固定操作栏（未确认）
└─────────────────────────────────────────────┘
```

#### 3.3.2 顶部导航栏

| 属性 | 规格 |
|------|------|
| 左侧 | ← 返回图标，点击返回 Markups Tab |
| 标题 | 局部更新 Title，截断超出部分（最多 1 行） |
| 背景 | 白色，底部 1px 分隔线 `--color-border-light` |

#### 3.3.3 图纸信息区

| 属性 | 规格 |
|------|------|
| 图纸信息行 | "{drawingCode} {drawingName} · {versionNo}"，14px，`--font-weight-medium`，`--color-text-regular` |
| 发布信息行 | "Published by {creatorName} · {date}"，12px，`--color-text-secondary` |
| 背景 | #F5F7FA，内边距 12px 16px |

#### 3.3.4 内容区

| 属性 | 规格 |
|------|------|
| 内边距 | 16px |
| 区块间距 | 20px |
| Section 标题 | 13px，`--font-weight-semibold`，`--color-text-secondary`，大写（Affected Area / Description / Attachments） |
| Affected Area 区块 | 背景 #F5F7FA，圆角 6px，内边距 10px 14px，13px，`--color-text-regular` |
| Description 文字 | 14px，`--color-text-regular`，行高 1.6，全文展开（无省略） |
| 附件卡片 | 白色背景，圆角 6px，边框 1px #EBEEF5，内边距 12px 16px；左侧文件图标（PDF 红色 / PNG+JPG 蓝色）+ 文件名 + 文件大小；右侧 [Preview / Open] 文字链接 |

#### 3.3.5 底部操作栏

**未确认状态**：

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF，顶部 1px 分隔线，高度 64px，`safe-area-inset-bottom` |
| 按钮 | 全宽 `[✓ Mark as Read]`；背景 `--color-primary`，白色文字，圆角 8px，高度 44px |
| 点击状态 | loading 动画，按钮禁用防重复点击 |

**已确认状态**：

| 属性 | 规格 |
|------|------|
| 内容 | "✓ You've confirmed reading"，居中显示 |
| 文字色 | `--color-success`（#2E7D32），14px |
| 图标 | ✓ checkmark，16px，绿色 |
| 按钮不可点击 | 无 active 状态，视觉上为静态文字展示 |

---

## 4. 多语言文案

### 4.1 图纸列表页 / 详情页 Tab

| Key | English | 中文 |
|-----|---------|------|
| tab_markups | Markups | 局部更新 |
| tab_markups_count | Markups ({n}) | 局部更新 ({n}) |

### 4.2 Markups Tab — 局部更新卡片

| Key | English | 中文 |
|-----|---------|------|
| label_new | New | 新 |
| label_affected_area | Affected Area | 影响区域 |
| btn_mark_as_read | Mark as Read | 标记已读 |
| label_read | ✓ Read | ✓ 已读 |

### 4.3 局部更新详情页

| Key | English | 中文 |
|-----|---------|------|
| detail_published_by | Published by {name} · {date} | 发布人：{name} · {date} |
| section_affected_area | Affected Area | 影响区域 |
| section_description | Description | 更新说明 |
| section_attachments | Attachments ({n}) | 附件 ({n}) |
| attachment_preview | Preview / Open | 预览 / 打开 |
| btn_mark_as_read_full | ✓ Mark as Read | ✓ 标记已读 |
| label_confirmed_reading | ✓ You've confirmed reading | ✓ 您已确认阅读 |

### 4.4 通知文案

| Key | English | 中文 |
|-----|---------|------|
| push_markup_title | Drawing Update — {drawingCode} | 图纸更新 — {drawingCode} |
| push_markup_body | {markupTitle}. Please review the latest markup for {drawingCode} {drawingName}. | {markupTitle}。请查阅 {drawingCode} {drawingName} 的最新局部更新。 |

---

## 5. 空状态

| 场景 | 图示 | 文字 |
|------|------|------|
| Markups Tab 无局部更新 | 📋 空文件图标 | "No markups yet." |
| 网络加载失败 | ⚠ 错误图标 | "Failed to load. Pull down to retry." |

---

## 6. 验收条件

### 6.1 图纸列表页角标

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 未读角标显示 | 有未确认 ACTIVE 局部更新的图纸卡片右上角显示橙色圆点 |
| 2 | 角标消失 | 当前用户确认所有 ACTIVE 局部更新后，橙色圆点消失 |

### 6.2 Markups Tab

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | Tab 显示 | 图纸详情页正确出现 Markups Tab |
| 2 | 未读气泡 | Tab 标签上橙色未读气泡数量准确，全部确认后气泡消失 |
| 3 | 通知跳转 | 点击 App Push 通知跳转到正确图纸详情页且自动激活 Markups Tab |
| 4 | 列表排序 | 局部更新按发布时间降序排列，最新在上 |
| 5 | 仅显示 ACTIVE | 已汇总（Merged）的局部更新不出现在列表中 |
| 6 | [New] 标签 | 未确认的卡片显示橙色 [New] 标签；确认后标签消失 |

### 6.3 Mark as Read

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 卡片内操作 | 点击卡片内 [Mark as Read]，成功后按钮变为 "✓ Read" |
| 2 | 详情页操作 | 点击详情页底部 [Mark as Read]，成功后按钮变为 "✓ You've confirmed reading" |
| 3 | 状态持久化 | 重新进入页面后已确认状态正确保留，不回退为未确认 |
| 4 | 幂等处理 | 重复点击确认，后端幂等返回成功，前端不出现异常 |
| 5 | Loading 防重复 | 点击后按钮立即进入 loading 状态，防止重复提交 |

### 6.4 局部更新详情页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 内容完整 | 正确展示标题、影响区域、说明、发布人、发布时间、附件列表 |
| 2 | 无影响区域 | Affected Area 区块在数据为空时不显示 |
| 3 | 附件预览 | 点击 [Preview / Open] 可正常打开或预览附件文件 |

---

## 7. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-004-app.md](../../../requirements/app/REQ-004-app.md) | APP 端产品需求 |
| [REQ-004-shared.md](../../../requirements/shared/REQ-004-shared.md) | 跨端共享业务规则与 API |
| [UI-REQ-003-app.md](./UI-REQ-003-app.md) | 图纸列表页基础 UI 规范（本功能依赖） |
