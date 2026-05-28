---
doc_type: ui_spec
source_req: REQ-005-app (v0.1.0)
platform: APP 移动端（iOS & Android，UNIAPP）
design_tokens: UI-REQ-001-app.md
generated_at: 2026-05-28
---

# UI 说明文档：APP 端 Site Engineer 图纸查阅与局部更新查看

> 本文档依据 [REQ-005-app.md v0.1.0](../../../requirements/app/REQ-005-app.md) 生成，对齐 [UI-REQ-003-app.md](./UI-REQ-003-app.md)（图纸列表基础规范）与 [UI-REQ-004-app.md](./UI-REQ-004-app.md)（Markups 卡片规范）的设计体系。

---

## 1. 设计目标

1. 图纸列表卡片展示 Description / RFA 信息，替代旧版 Drawing Code / Name / Version
2. Markups 角标仅显示 unread 数，清晰传递"待处理"的紧迫感
3. Confirm Reading 操作保持在底部固定区域，单手可达
4. Markups 卡片提供报审溯源（Based on submissionNo · Page N），与 PC 端保持信息一致
5. 全流程支持 Push 通知冷热启动跳转，减少操作路径

---

## 2. 页面清单与用户流程

| 页面 | 路由 | 说明 |
|-----|------|------|
| 图纸列表页 | `pages/drawings/list` | SE 专属，仅展示已分配的 DrawingVersion |
| 图纸在线查看页 | `pages/drawings/detail` | 图纸预览 + Confirm Reading（底部固定）+ Markups Tab |

### 用户流程（ASCII）

```
首页 Drawings 入口
       │
       ▼
  图纸列表页
  ┌────────────────────────────────────┐
  │  卡片: Description · Category      │
  │        RFA No. · Subject of RFA   │
  │        状态: Confirm → / ✓ Read   │
  │        角标: ●{unread}            │
  └────────────────────────────────────┘
       │ 点击卡片
       ▼
  图纸在线查看页
  ┌──────────────────────────────────────────┐
  │  [Drawing Preview Tab]                   │
  │    图纸预览区（双指缩放/单指平移）         │
  │    底部: [Confirm Reading] / ✓ Confirmed │
  ├──────────────────────────────────────────┤
  │  [Markups Tab ●2]                        │
  │    Markup 卡片列表                        │
  │    卡片: 元信息行 + 报审行 + [Mark as Read]│
  └──────────────────────────────────────────┘

  App Push 通知
       │ 点击通知
       ▼
  图纸在线查看页 — 自动激活 Markups Tab
```

---

## 3. 组件设计规范

### 3.1 图纸列表页

#### 3.1.1 页面布局

```
┌──────────────────────────────────────┐
│ ← Drawings                           │
├──────────────────────────────────────┤
│ [🔍 Search description...]           │
│ [All ▼]                              │
├──────────────────────────────────────┤
│ ┌──────────────────────────────────┐ │
│ │ 首层平面图施工图纸            ●2 │ │
│ │ Architectural                    │ │
│ │ RFA-001 · 首层平面图报审         │ │
│ │ Updated 2026-04-01   [✓ Read]   │ │
│ └──────────────────────────────────┘ │
│ ┌──────────────────────────────────┐ │
│ │ 基础结构施工详图                  │ │
│ │ Structural                       │ │
│ │ —                                │ │
│ │ Updated 2026-03-20  [Confirm →]  │ │
│ └──────────────────────────────────┘ │
│ ┌──────────────────────────────────┐ │
│ │ 通风管道平面图                    │ │
│ │ Mechanical                       │ │
│ │ RFA-003 · 机电图纸报审           │ │
│ │ Updated 2026-03-10   [✓ Read]   │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘
```

#### 3.1.2 搜索与筛选栏

| 属性 | 规格 |
|------|------|
| 搜索框 | `uni-search-bar` 或自定义，placeholder "Search description..."；debounce 500ms |
| Category 筛选 | `uni-data-select`，选项：All / Architectural / Structural / Mechanical / Electrical / Plumbing / Civil / Other；宽度自适应 |
| 布局 | 搜索框占满宽度；筛选器紧接其下，单行 |

#### 3.1.3 DrawingVersion 卡片规格

| 属性 | 规格 |
|------|------|
| 卡片背景 | 白色 #FFFFFF，圆角 8px，阴影 0 2px 8px rgba(0,0,0,0.06) |
| 内边距 | 14px 16px |
| 卡片间距 | 10px |
| Description | 16px，`font-weight: 600`，`--color-text-primary`；单行溢出省略 |
| Category | 14px，`--color-text-secondary`，Description 下方 2px |
| RFA 信息行 | `{rfaNo} · {subjectOfRfa}`，13px，`--color-text-secondary`；rfaNo 为 null 时整行显示 `—` |
| Updated | `Updated {date}`，12px，`--color-text-placeholder` |
| 状态标识 | 见 §3.1.4；位于卡片右下角 |
| Markups 角标 | 见 §3.1.5；位于卡片右上角 |

> **不展示的字段**：versionNo、Drawing Code、Drawing Name 均不在卡片中渲染。

#### 3.1.4 状态标识

| 状态 | 展示 | 位置 |
|------|------|------|
| 未确认 | `Confirm →`，蓝色文字（`--color-primary`），13px | 卡片右下角 |
| 已确认 | `✓ Read`，绿色（`--color-success`），13px | 卡片右下角 |

#### 3.1.5 Markups 角标（未读数）

| 状态 | 展示 |
|------|------|
| unread > 0 | 橙色圆形角标（`--color-warning` #F57F17），白色数字，最小宽度 18px，高度 18px；>99 显示 "99+"；绝对定位于卡片右上角，右 12px，上 12px |
| unread = 0 | 角标不显示 |

> **说明**：角标只展示未读（unread）数量，不展示 total。用户的关注点是"还有多少条没看"。

#### 3.1.6 空状态

```
┌──────────────────────────────────────┐
│                 📋                   │
│    No drawings assigned to you yet.  │
│    Please contact your administrator │
│    for drawing access.               │
└──────────────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| 图标 | 📋 48px，`--color-text-placeholder` |
| 主文字 | 14px，`--color-text-secondary` |
| 副文字 | 13px，`--color-text-placeholder` |

---

### 3.2 图纸在线查看页

#### 3.2.1 顶部导航栏

| 属性 | 规格 |
|------|------|
| 返回按钮 | ← 图标，点击返回列表 |
| 标题 | Description（16px，加粗，单行溢出省略） |
| 副标题（次行） | Category，13px，`--color-text-secondary` |
| 右侧 [⋯] | 更多操作（可选：在浏览器打开、分享） |
| 不显示 | 系统版本号（versionNo） |

#### 3.2.2 Tab 栏

| 属性 | 规格 |
|------|------|
| Tab 1 | "Drawing Preview" |
| Tab 2 | "Markups"；unread > 0 时 Tab 标题显示橙色气泡 `●{n}`；unread = 0 时仅显示 "Markups" |
| 样式 | 下划线式 Tab，激活时 `--color-primary` 下划线，字体加粗 |

#### 3.2.3 Drawing Preview Tab 布局

```
┌──────────────────────────────────────┐
│ ← 首层平面图施工图纸            [⋯] │
│   Architectural                      │
├──────────────────────────────────────┤
│  [Drawing Preview]  [Markups ●2]     │
├──────────────────────────────────────┤
│                                      │
│                                      │
│    （图纸在线预览区域）               │
│    双指缩放 / 单指平移               │
│                                      │
│                                      │
│                                      │
├──────────────────────────────────────┤
│  Updated 2026-04-01                  │
│  ┌────────────────────────────────┐  │
│  │  ✓  Confirm Reading            │  │  ← 未确认
│  └────────────────────────────────┘  │
│  ✓ Confirmed on Apr 8, 2026         │  ← 已确认（替换按钮）
└──────────────────────────────────────┘
```

#### 3.2.4 预览区规格

| 属性 | 规格 |
|------|------|
| 背景 | #F0F2F5 |
| 最小高度 | 屏幕高度 - 导航栏 - Tab 栏 - 底部操作栏 |
| 文件类型 | PDF → WebView 内嵌；PNG/JPG → `uni.previewImage`；DWG → 后端转 PDF |
| 缩放 | 双指缩放，范围 0.5× ～ 5× |
| 平移 | 单指平移，超出边界弹性回弹 |
| 加载中 | 居中 loading 动画 |
| 加载失败 | ⚠ 图标 + "Failed to load drawing." + [Retry] 按钮 |

#### 3.2.5 底部操作区（Confirm Reading）

| 状态 | 规格 |
|------|------|
| 未确认 | `[Confirm Reading]` 蓝色主按钮，圆角 8px，高度 48px，底部 SafeArea 内边距 |
| 已确认 | `✓ Confirmed on {date}`，绿色（`--color-success`），14px，居中对齐；不可点击 |

#### 3.2.6 Confirm Reading 确认 Popup

```
┌─────────────────────────────────────┐
│  Confirm Reading                    │
│                                     │
│  Confirm that you have read this    │
│  drawing?                           │
│  {description} · {category}         │
│                                     │
│  [Cancel]            [Confirm]      │
└─────────────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| 组件 | `uni.showModal` 或自定义 Popup |
| 副文字 | description · category（不含版本号） |
| [Confirm] | 蓝色，提交 loading 防重复点击 |
| 成功后 | 底部按钮替换为已确认文字；列表卡片状态实时更新 |

---

### 3.3 Markups Tab

#### 3.3.1 整体布局

```
┌──────────────────────────────────────┐
│  [Drawing Preview]  [Markups ●2]     │
├──────────────────────────────────────┤
│  ┌────────────────────────────────┐  │
│  │  [New]  A轴节点详图修正        │  │
│  │  Apr 8, 2026 · 张三            │  │
│  │  Based on SUB-2026-003 · Page 3│  │
│  │  A轴与3轴交叉节点详图已更新，  │  │
│  │  新增钢筋排布说明...  Show more │  │
│  │  📎 node-detail.pdf            │  │
│  │           [✓ Mark as Read]     │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │  C区消防管道路由修正            │  │
│  │  Apr 7, 2026 · 张三            │  │
│  │  Based on SUB-2026-003 · Page 5│  │
│  │  消防主管道路由变更，详见附图   │  │
│  │  📎 route-update.png           │  │
│  │                     ✓ Read     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

#### 3.3.2 Markup 卡片规格

| 属性 | 规格 |
|------|------|
| 卡片背景 | 白色 #FFFFFF，圆角 8px，边框 1px #EBEEF5，阴影 0 1px 4px rgba(0,0,0,0.06) |
| 内边距 | 14px 16px |
| 卡片间距 | 10px |
| [New] 标签 | 橙色胶囊，背景 #FF6D00，白色文字 "New"，11px，内边距 2px 8px；标题左侧；已读后隐藏 |
| 标题 | description 字段；16px，`font-weight: 600`，`--color-text-primary` |
| 元信息行 | `{发布日期} · {创建人}`，13px，`#909399`；标题下方 4px |
| 报审号+页码行 | `Based on {submissionNo}`；若 `appliedPageNo` 不为 null 则追加 ` · Page {n}`；13px，`#909399`；元信息行下方 4px |
| 说明文字 | 14px，`--color-text-regular`，行高 1.6，最多 3 行，超出显示 "Show more" |
| 附件 | 📎 图标（13px）+ 文件名（`--color-primary`）；点击调用系统预览或浏览器打开 |
| [Mark as Read] | 见 §3.3.3 |

> **不展示的字段**：Affected Area（影响区域）不在 SE 视图卡片中显示。

#### 3.3.3 [Mark as Read] 按钮 / 已读状态

| 状态 | 展示 | 规格 |
|------|------|------|
| 未读 | `[✓ Mark as Read]` | 蓝色主按钮，`border-radius: 6px`，高度 36px，右下角对齐 |
| loading | 按钮显示 loading，禁止重复点击 | — |
| 已读 | `✓ Read` | 绿色（`--color-success`），13px，右下角对齐；不可点击 |

#### 3.3.4 空状态

| 场景 | 展示 |
|------|------|
| 无局部更新 | 📋 图标 + "No markups for this drawing yet." |
| 加载失败 | ⚠ 图标 + "Failed to load." + [Retry] 链接 |

---

### 3.4 App Push 通知样式

| 字段 | 内容 | 规格 |
|------|------|------|
| 标题 | Drawing Update — {description} | 系统通知标题样式 |
| 正文 | {markupTitle}. Please review the latest markup. | 系统通知正文样式 |
| 角标 | APP 图标右上角 +1 | 系统 Badge |
| 点击 | 跳转到图纸查看页 Markups Tab | 路由参数含 drawingVersionId + tab=markups |

---

## 4. 多语言文案

| Key | English | 中文 |
|-----|---------|------|
| page_drawings | Drawings | 我的图纸 |
| search_placeholder | Search description... | 搜索批次描述... |
| card_rfa_empty | — | — |
| card_status_confirm | Confirm → | 待确认 → |
| card_status_read | ✓ Read | ✓ 已读 |
| empty_drawings | No drawings assigned to you yet. | 暂无分配给您的图纸 |
| tab_preview | Drawing Preview | 图纸预览 |
| tab_markups | Markups | 局部更新 |
| btn_confirm_reading | ✓ Confirm Reading | ✓ 确认查阅 |
| confirmed_on | ✓ Confirmed on {date} | ✓ 已于 {date} 确认 |
| confirm_popup_title | Confirm Reading | 确认查阅 |
| confirm_popup_msg | Confirm that you have read this drawing? | 确认您已阅读本图纸？ |
| btn_confirm | Confirm | 确认 |
| btn_cancel | Cancel | 取消 |
| markup_new | New | 新 |
| markup_based_on | Based on {submissionNo} | 基于 {submissionNo} |
| markup_page | · Page {n} | · 第 {n} 页 |
| markup_show_more | Show more | 展开全文 |
| btn_mark_as_read | ✓ Mark as Read | ✓ 标记已读 |
| label_read | ✓ Read | ✓ 已读 |
| empty_markups | No markups for this drawing yet. | 该图纸暂无局部更新 |
| push_title | Drawing Update — {description} | 图纸更新 — {description} |
| push_body | {markupTitle}. Please review the latest markup. | {markupTitle}。请查阅最新局部更新。 |

---

## 5. 验收条件

### 5.1 图纸列表页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 卡片字段 | 显示 Description、Category、RFA 信息行（或 —）、Updated、状态标识；**不显示**版本号 |
| 2 | 角标 | unread > 0 → 橙色数字角标；unread = 0 → 无角标 |
| 3 | 数据过滤 | 仅显示 ACTIVE + 已分配给当前用户的 DrawingVersion |
| 4 | 下拉刷新 / 上拉加载 | 正常工作 |
| 5 | 搜索筛选 | Description 模糊搜索、Category 筛选生效 |
| 6 | 空状态 | 无分配图纸时显示正确文案 |

### 5.2 图纸查看页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 标题 | 显示 Description + Category，不含版本号 |
| 2 | 预览 | 图纸文件正常加载，支持双指缩放和单指平移 |
| 3 | Confirm Reading | 未确认时显示底部蓝色按钮；已确认时显示绿色文字 |
| 4 | 确认 Popup | 副文字含 description 和 category，不含版本号 |
| 5 | 确认成功 | 底部替换为已确认文字；列表卡片状态同步更新 |
| 6 | 跨端同步 | PC 端确认后 APP 端刷新显示已确认；反之亦然 |

### 5.3 Markups Tab

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 卡片元信息行 | 显示 `{日期} · {创建人}` |
| 2 | 卡片报审行 | 显示 `Based on {submissionNo}`；appliedPageNo 不为 null 时追加 ` · Page {n}` |
| 3 | 无 Affected Area | 卡片中不显示影响区域字段 |
| 4 | [New] 标签 | 未读时显示；Mark as Read 后消失 |
| 5 | Mark as Read | 成功后按钮替换为 "✓ Read"；角标减一 |
| 6 | 跨端同步 | PC 端标记已读后，APP 端刷新显示 "✓ Read" |

### 5.4 App Push 通知

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 推送接收 | Markup 发布后，已分配 SE 收到通知，APP 角标 +1 |
| 2 | 未分配用户 | 未分配该图纸的 SE 不收到通知 |
| 3 | 热启动跳转 | 点击通知跳转到图纸查看页 Markups Tab |
| 4 | 冷启动跳转 | 冷启动后正确解析 payload 并跳转 |

---

## 6. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-005-app.md](../../../requirements/app/REQ-005-app.md) | APP 端产品需求（v0.1.0） |
| [UI-REQ-003-app.md](./UI-REQ-003-app.md) | 图纸列表基础 UI 规范 |
| [UI-REQ-004-app.md](./UI-REQ-004-app.md) | Markups UI 规范（卡片组件参考） |
| [UI-REQ-005-pc.md](../pc/UI-REQ-005-pc.md) | PC 端对等 UI 规范 |
