# UI 说明文档 — APP 移动端工程图纸查阅确认与审批

> **来源需求**: [REQ-003-app](../../../requirements/app/REQ-003-app.md) + [REQ-003-shared](../../../requirements/shared/REQ-003-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: APP 移动端（iOS & Android，UNIAPP + Vue 2）
> **设计令牌参考**: [UI-REQ-001-app § 2](./UI-REQ-001-app.md)
> **生成日期**: 2026-04-07

---

## 1. 设计目标

- 确保 Site Engineer 在 APP 端始终看到经审批的最新版图纸，避免按旧版施工
- 图纸列表**只显示管理员分配给当前用户的图纸**，过滤无关内容，降低信息噪音
- 新版图纸审批通过后**仅向已分配的 Site Engineer** 发送 App Push，做到精准推送
- 提供流畅的在线预览体验，支持双指缩放，适配现场单手操作场景
- 查阅确认入口明显、流程简短，降低确认操作的认知成本
- 审批人员在 APP Todo 列表中也能便捷完成图纸审批，保持流程闭环
- 视觉风格与首页保持一致，延续 REQ-001-app 设计体系
- 支持多语言（中文 / English），默认英文界面

---

## 2. 页面清单与用户流程

### 2.1 页面清单

| 序号 | 页面 | 说明 |
|------|------|------|
| 1 | 图纸列表页 | 搜索 + 分类筛选 + 图纸卡片列表，显示查阅确认状态 |
| 2 | 图纸在线查看页 | 全屏预览区 + 底部版本信息栏 + 查阅确认按钮 |
| 3 | 审批 Todo（已有 Todo 列表页复用） | 待审批图纸卡片，支持通过 / 驳回操作 |

### 2.2 核心用户流程

```
Site Engineer 查阅流程：

  ┌─────────────────────┐
  │  收到 App Push 通知  │  ← 审批通过后系统推送
  │  "Drawing Updated"  │
  └──────────┬──────────┘
             │ 点击通知
             ▼
  ┌─────────────────────┐
  │  图纸列表页          │  ← 自动定位/高亮新版图纸
  │  卡片右侧显示        │
  │  [Confirm →]        │  ← 未确认状态
  └──────────┬──────────┘
             │ 点击卡片
             ▼
  ┌─────────────────────┐
  │  图纸在线查看页       │
  │  全屏预览图纸文件     │
  │  底部：[Confirm Reading] │
  └──────────┬──────────┘
             │ 点击按钮
             ▼
  ┌─────────────────────┐
  │  确认弹窗             │
  │  [Cancel] [Confirm] │
  └──────────┬──────────┘
             │ 确认
             ▼
  ┌─────────────────────┐
  │  按钮变为             │
  │  ✓ Confirmed on ... │  ← 绿色，不可再点击
  │  返回列表 → [✓ Read] │
  └─────────────────────┘
```

```
审批人员 Todo 流程：

  ┌───────────────────────┐
  │  首页 Todo 模块        │
  │  图纸审批角标 +N       │
  └──────────┬────────────┘
             │ 进入 Todo 列表
             ▼
  ┌───────────────────────┐     ┌────────────────────────┐
  │  图纸审批 Todo 卡片    │     │  点击 [View]           │
  │  ARCH-001 V2 待审批   │ ──▶ │  进入在线查看页（无确认）│
  │  [View] [Approve]     │     └────────────────────────┘
  │         [Reject]      │
  └──────────┬────────────┘
             │
     ┌───────┴───────┐
     ↓               ↓
 [Approve]       [Reject]
     │               │
     ▼               ▼
 确认弹窗        底部 Sheet
 二次确认        输入 Comment
     │               │（必填）
     ▼               ▼
 Todo 消失       Todo 消失
 Toast 成功      上传人收通知
```

---

## 3. 页面设计规范

### 3.1 图纸列表页

#### 3.1.1 页面结构

```
┌──────────────────────────────┐  ← 状态栏
│ ←   Drawing              🔔 │  ← 顶部导航栏，标题居中
├──────────────────────────────┤
│ 🔍  Search drawings...       │  ← 搜索框，圆角，灰色背景
├──────────────────────────────┤
│ [All ▼]  [Category ▼]       │  ← 筛选标签栏，横向滚动
├──────────────────────────────┤
│ ┌────────────────────────┐   │
│ │ ARCH-001               │   │  ← 图纸编号（主色蓝，粗体）
│ │ 首层平面图              │   │  ← 图纸名称（正文色，16px）
│ │ Architectural · V3     │   │  ← 分类 · 版本（次要色，13px）
│ │ Updated Apr 1, 2026    │   │  ← 更新时间（次要色，12px）
│ │                [✓ Read]│   │  ← 已确认（绿色标签）
│ └────────────────────────┘   │
│ ┌────────────────────────┐   │
│ │ STRU-001               │   │
│ │ 基础结构图              │   │
│ │ Structural · V2        │   │
│ │ Updated Mar 20, 2026   │   │
│ │         [Confirm →]    │   │  ← 未确认（蓝色，带箭头）
│ └────────────────────────┘   │
│            ...               │
└──────────────────────────────┘
```

#### 3.1.2 顶部导航栏

| 属性 | 规格 |
|------|------|
| 高度 | 44px（含状态栏）|
| 背景 | 白色 #FFFFFF |
| 标题 | "Drawing"，16px，`--font-weight-semibold`，居中 |
| 左侧 | ← 返回按钮，24px，`--color-text-secondary` |
| 右侧 | 🔔 通知图标，24px（可选，若统一由首页通知管理则不显示） |
| 底部边框 | 1px `--color-border-light`（#E8E8E8） |

#### 3.1.3 搜索框

| 属性 | 规格 |
|------|------|
| 高度 | 36px |
| 背景 | #F5F7FA |
| 圆角 | 18px（全圆角） |
| 内边距 | 水平 12px |
| 占位文字 | "Search drawings..."，`--color-text-placeholder`（#BDBDBD），13px |
| 图标 | 🔍 搜索图标，左侧，16px，#BDBDBD |
| 外边距 | 水平 16px，垂直 8px |
| 防抖 | 输入停止 300ms 后触发搜索 |

#### 3.1.4 筛选标签栏

| 属性 | 规格 |
|------|------|
| 高度 | 36px |
| 滚动 | 横向可滚动，超出自动隐藏 |
| 标签间距 | 8px |
| 默认标签 | "All"，灰色边框，白色背景 |
| 选中标签 | 主色 #4A90D9 背景，白色文字 |
| 标签圆角 | 16px |
| 标签内边距 | 水平 12px，垂直 6px |
| 标签文字 | 13px |

可用分类标签：All / Architectural / Structural / Mechanical / Electrical / Plumbing / Civil / Other

#### 3.1.5 图纸卡片

| 属性 | 规格 |
|------|------|
| 卡片背景 | 白色 #FFFFFF |
| 卡片圆角 | 8px |
| 卡片阴影 | 0 1px 4px rgba(0,0,0,0.06) |
| 外边距 | 水平 16px，底部 8px |
| 内边距 | 16px |
| 分隔线 | 无（通过阴影+间距区分） |

**卡片内容排布**：

| 区域 | 内容 | 样式 |
|------|------|------|
| 第 1 行 | 图纸编号 | 14px，`--font-weight-semibold`，`--color-primary`（#4A90D9） |
| 第 2 行 | 图纸名称 | 16px，`--font-weight-medium`，`--color-text-regular`（#333333） |
| 第 3 行 | 分类 · 版本号 | 13px，`--color-text-secondary`（#666666），· 为分隔符 |
| 第 4 行左 | "Updated {date}" | 12px，`--color-text-secondary` |
| 第 4 行右 | 确认状态标签 | 见下方状态定义 |

**确认状态标签**：

| 状态 | 文字 | 背景 | 文字色 | 圆角 |
|------|------|------|--------|------|
| 已确认 | ✓ Read | #E8F5E9（浅绿）| #2E7D32（深绿）| 12px |
| 未确认 | Confirm → | #E3F2FD（浅蓝）| #1565C0（深蓝）| 12px |

#### 3.1.6 空状态

| 场景 | 图标 | 文字 |
|------|------|------|
| 未被分配任何图纸 | 📄 图纸图标，48px，#BDBDBD | "No drawings assigned to you yet." |
| 搜索无结果 | 🔍 48px，#BDBDBD | "No results for "{keyword}"" |
| 网络错误 | ⚠ 48px，#BDBDBD | "Failed to load. Pull down to retry." |

---

### 3.2 图纸在线查看页

#### 3.2.1 页面结构

```
┌──────────────────────────────┐  ← 状态栏（沉浸式，深色/浅色自动适配）
│ ←  ARCH-001 · V3        [⋯] │  ← 顶部导航栏
│    首层平面图                 │  ← 副标题行
├──────────────────────────────┤
│                              │
│                              │
│                              │
│      ┌──────────────┐        │
│      │              │        │
│      │  图纸预览区域 │        │  ← 全屏可交互预览区
│      │              │        │
│      │  双指缩放      │        │
│      │  单指平移      │        │
│      └──────────────┘        │
│                              │
│                              │
├──────────────────────────────┤  ← 底部信息栏（白色背景）
│  V3 · Updated Apr 1, 2026   │  ← 版本信息（13px，次要色）
│                              │
│  ┌──────────────────────┐    │
│  │   ✓  Confirm Reading  │    │  ← 主按钮（蓝色）
│  └──────────────────────┘    │
│                              │  ← 安全区域底部留白
└──────────────────────────────┘
```

#### 3.2.2 顶部导航栏

| 属性 | 规格 |
|------|------|
| 高度 | 44px |
| 背景 | 白色 #FFFFFF（图纸预览区触顶时可半透明） |
| 左侧 | ← 返回，24px，`--color-text-secondary` |
| 主标题 | "{drawingCode} · {versionNo}"，14px，`--font-weight-semibold`，`--color-text-regular` |
| 副标题 | 图纸名称，12px，`--color-text-secondary`，导航栏标题正下方 |
| 右侧 | [⋯] 更多菜单图标，24px（可预留，当前暂无操作） |

#### 3.2.3 图纸预览区

| 属性 | 规格 |
|------|------|
| 区域背景 | #F5F5F5（浅灰，与图纸白色背景形成对比） |
| 高度 | `屏幕高度 - 导航栏高度 - 底部信息栏高度 - 安全区域` |
| 手势 | 双指缩放（pinch-to-zoom），最小 1×，最大 5× |
| 手势 | 单指平移（pan），放大后可拖动 |
| 双击 | 双击切换 1× / 2× |
| 加载状态 | 居中菊花 Loading 动画 + "Loading..." 文字 |
| 加载失败 | 居中显示 ⚠ 图标 + "Failed to load drawing. Tap to retry." |
| PDF 渲染 | WebView 内嵌 PDF.js 或系统原生 PDF 渲染 |
| 图片渲染 | UNIAPP image 组件，mode="aspectFit" |

#### 3.2.4 底部信息栏

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF |
| 顶部边框 | 1px `--color-border-light`（#E8E8E8） |
| 内边距 | 水平 16px，垂直 12px |
| 版本信息文字 | "{versionNo} · Updated {date}"，13px，`--color-text-secondary` |
| 按钮与版本信息间距 | 10px |

#### 3.2.5 Confirm Reading 按钮

**未确认状态**：

| 属性 | 规格 |
|------|------|
| 文字 | "Confirm Reading" |
| 样式 | 主按钮，蓝色 #4A90D9，白色文字，16px，`--font-weight-semibold` |
| 高度 | 48px |
| 宽度 | 100%（水平 16px 边距内） |
| 圆角 | 8px |
| Hover/按下 | `--color-primary-active`（#2A6FA8） |

**已确认状态**：

| 属性 | 规格 |
|------|------|
| 文字 | "✓ Confirmed on {Apr 1, 2026}" |
| 样式 | 绿色文字 #2E7D32，无背景（纯文字），16px |
| 图标 | ✓ 勾选图标，绿色，左侧 |
| 状态 | 不可点击，cursor: default |

#### 3.2.6 确认弹窗

| 属性 | 规格 |
|------|------|
| 类型 | Modal Popup，居中显示 |
| 背景遮罩 | rgba(0,0,0,0.4) |
| 弹窗背景 | 白色，圆角 12px，宽度 `屏幕宽度 - 64px` |
| 内边距 | 24px |
| 标题 | "Confirm Reading?"，16px，`--font-weight-semibold`，居中 |
| 正文 | "Please confirm that you have reviewed this drawing."，14px，`--color-text-secondary`，居中，上边距 8px |
| 分割线 | 1px `--color-border-light`，上边距 16px |
| 按钮区 | 横向排列，各占 50% |
| Cancel 按钮 | 灰色文字 #666666，无背景，14px |
| Confirm 按钮 | 主色 #4A90D9，白色文字，14px，`--font-weight-semibold` |

---

### 3.3 审批 Todo 卡片（Todo 列表页复用）

> 审批 Todo 复用现有的首页 Todo 列表模块，本节仅描述图纸审批类型的卡片样式和操作。

#### 3.3.1 Todo 卡片结构

```
┌──────────────────────────────────┐
│ 📋  Drawing Approval Required    │  ← 类型标题（14px，粗体）
│                                  │
│  ARCH-001  首层平面图  V2         │  ← 图纸编号 + 名称 + 版本
│  Uploaded by 李四                 │  ← 上传人
│  Apr 1, 2026  10:00              │  ← 上传时间
│                                  │
│  ┌──────┐  ┌─────────┐ ┌──────┐ │
│  │ View │  │ Approve │ │Reject│ │  ← 操作按钮行
│  └──────┘  └─────────┘ └──────┘ │
└──────────────────────────────────┘
```

#### 3.3.2 卡片样式

| 属性 | 规格 |
|------|------|
| 背景 | 白色 #FFFFFF |
| 圆角 | 8px |
| 阴影 | 0 1px 4px rgba(0,0,0,0.06) |
| 外边距 | 水平 16px，底部 8px |
| 内边距 | 16px |
| 类型图标 | 📋 14px，左对齐，`--color-primary` |
| 类型标题 | "Drawing Approval Required"，14px，`--font-weight-semibold`，`--color-text-regular` |
| 图纸信息 | 14px，`--color-text-regular`，上边距 8px |
| 元信息 | 12px，`--color-text-secondary`，上边距 4px |

#### 3.3.3 操作按钮行

| 按钮 | 样式 | 规格 |
|------|------|------|
| View | 次级边框按钮 | 灰色边框，白色背景，`--color-text-regular`，13px，高度 32px，圆角 6px |
| Approve | 主色按钮 | #4A90D9 背景，白色文字，13px，高度 32px，圆角 6px |
| Reject | 危险边框按钮 | 红色 #FF4D4F 边框，红色文字，13px，高度 32px，圆角 6px |
| 按钮间距 | 8px | |
| 上边距 | 12px（与上方元信息间距） | |

#### 3.3.4 驳回 Bottom Sheet

| 属性 | 规格 |
|------|------|
| 类型 | Bottom Sheet，从底部滑入 |
| 圆角 | 顶部圆角 16px |
| 背景 | 白色 #FFFFFF |
| 顶部横条 | 4px × 32px，圆角 2px，#D9D9D9，居中，顶部 8px |
| 标题 | "Reject Drawing"，16px，`--font-weight-semibold`，左对齐，内边距 16px |
| Comment 标签 | "Comment"，14px，`--font-weight-medium`，红色星号 * |
| 文本输入框 | 多行，最小高度 96px，圆角 6px，边框 1px #D9D9D9，内边距 10px 12px，14px |
| Placeholder | "Please enter rejection reason..."，`--color-text-placeholder` |
| 字数限制 | 最大 500 字，右下角显示 x/500 |
| Cancel 按钮 | 灰色边框，白色背景，全宽，高度 44px，圆角 8px，14px |
| Confirm 按钮 | 红色 #FF4D4F，白色文字，全宽，高度 44px，圆角 8px，14px，`--font-weight-semibold` |
| Confirm 禁用 | Comment 为空时，Confirm 按钮 opacity 0.4，不可点击 |
| 按钮间距 | 8px |
| 底部安全区 | iOS Home Indicator 区域留白 |

#### 3.3.5 审批通过确认弹窗

| 属性 | 规格 |
|------|------|
| 类型 | Modal Popup，居中 |
| 标题 | "Approve Drawing?" |
| 正文 | "This version will become active and previous versions will be deprecated." |
| Cancel | 灰色，`--color-text-secondary` |
| Confirm | 主色 #4A90D9，`--font-weight-semibold` |

---

## 4. 多语言文案

### 4.1 图纸列表页

| Key | English | 中文 |
|-----|---------|------|
| page_title | Drawing | 图纸 |
| search_placeholder | Search drawings... | 搜索图纸... |
| filter_all | All | 全部 |
| category_architectural | Architectural | 建筑 |
| category_structural | Structural | 结构 |
| category_mechanical | Mechanical | 机电 |
| category_electrical | Electrical | 电气 |
| category_plumbing | Plumbing | 给排水 |
| category_civil | Civil | 土建 |
| category_other | Other | 其他 |
| label_updated | Updated {date} | 更新于 {date} |
| status_read | ✓ Read | ✓ 已查阅 |
| status_confirm | Confirm → | 去确认 → |
| empty_not_assigned | No drawings assigned to you yet. | 暂无分配给您的图纸 |
| empty_no_results | No results for "{keyword}" | 未找到"{keyword}" |
| empty_load_failed | Failed to load. Pull down to retry. | 加载失败，下拉重试 |

### 4.2 图纸在线查看页

| Key | English | 中文 |
|-----|---------|------|
| version_info | {version} · Updated {date} | {version} · 更新于 {date} |
| btn_confirm_reading | Confirm Reading | 确认查阅 |
| btn_confirmed | ✓ Confirmed on {date} | ✓ 已于 {date} 确认查阅 |
| dialog_confirm_title | Confirm Reading? | 确认查阅？ |
| dialog_confirm_body | Please confirm that you have reviewed this drawing. | 请确认您已查阅本图纸。 |
| btn_cancel | Cancel | 取消 |
| btn_confirm | Confirm | 确认 |
| loading_text | Loading... | 加载中... |
| load_failed | Failed to load drawing. Tap to retry. | 图纸加载失败，点击重试。 |
| toast_confirm_success | Reading confirmed | 查阅确认成功 |

### 4.3 审批 Todo 卡片

| Key | English | 中文 |
|-----|---------|------|
| todo_type_title | Drawing Approval Required | 图纸审批待处理 |
| label_uploaded_by | Uploaded by {name} | 上传人：{name} |
| btn_view | View | 查看 |
| btn_approve | Approve | 通过 |
| btn_reject | Reject | 驳回 |
| approve_dialog_title | Approve Drawing? | 确认通过？ |
| approve_dialog_body | This version will become active and previous versions will be deprecated. | 该版本将成为当前有效版本，旧版本将自动作废。 |
| reject_sheet_title | Reject Drawing | 驳回图纸 |
| reject_comment_label | Comment | 驳回意见 |
| reject_comment_placeholder | Please enter rejection reason... | 请输入驳回原因... |
| toast_approved | Drawing approved successfully | 审批通过 |
| toast_rejected | Drawing rejected | 已驳回 |

---

## 5. 屏幕适配要求

| 设备类型 | 屏幕尺寸 | 适配要求 |
|---------|---------|---------|
| 标准手机 | 375 × 812pt（iPhone X 基准） | 基准设计尺寸 |
| 小屏手机 | 320 × 568pt（iPhone SE 1st） | 底部安全区无 Home Indicator，按钮高度不低于 44pt |
| 大屏手机 | 428 × 926pt（iPhone 14 Pro Max） | 布局自适应拉伸，卡片宽度跟随屏幕 |
| Android | 各尺寸 | 与 iOS 视觉一致，状态栏高度自适应 |
| 刘海屏 / 挖孔屏 | — | 顶部状态栏区域安全留白，底部 Home Indicator 留白 |
| 横屏 | — | 图纸在线查看页支持横屏，导航栏收起或缩小 |

---

## 6. 验收条件

### 6.1 图纸列表页

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 数据过滤 | 列表仅显示满足两个条件的图纸：① status=ACTIVE ② 已被管理员分配给当前用户；后端双重过滤，前端无法绕过 |
| 2 | 空状态—未分配 | 当前用户尚无分配图纸时，居中显示 📄 图标 + "No drawings assigned to you yet." |
| 3 | 确认状态 | 已确认图纸右侧显示绿色 ✓ Read，未确认显示蓝色 Confirm → |
| 4 | 搜索 | 输入停止 300ms 后触发搜索，清空输入后恢复列表 |
| 5 | 分类筛选 | 选中分类标签后列表正确过滤，全部标签恢复完整列表 |
| 6 | 下拉刷新 | 下拉触发刷新，转圈动画正确 |
| 7 | 上拉加载 | 滚动到底部触发加载更多，无更多数据时显示提示 |

### 6.2 在线查看与确认

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | PDF 预览 | PDF 文件可在 APP 内正确渲染，内容清晰 |
| 2 | 图片预览 | PNG/JPG 文件正确显示，不拉伸 |
| 3 | 双指缩放 | 支持 1× ~ 5× 缩放，手势流畅 |
| 4 | 平移 | 放大状态下支持单指平移 |
| 5 | 未确认按钮 | 未确认时显示蓝色 [Confirm Reading] 主按钮 |
| 6 | 确认弹窗 | 点击按钮弹出确认弹窗，Cancel/Confirm 功能正常 |
| 7 | 已确认状态 | 确认后按钮变为绿色 ✓ Confirmed on {date}，不可再点击 |
| 8 | 列表状态同步 | 返回列表后，该图纸卡片右侧状态即时更新为 ✓ Read |
| 9 | 加载失败处理 | 文件加载失败时显示错误提示，点击可重试 |
| 10 | 横屏支持 | 在线查看页横屏旋转后布局正常，预览区放大 |

### 6.3 审批 Todo

| # | 验收项 | 通过标准 |
|---|--------|---------|
| 1 | 角标计数 | 有待审批图纸时，Todo 入口显示正确的未处理数量 |
| 2 | View 跳转 | 点击 [View] 进入对应版本查看页，不显示 Confirm Reading 按钮 |
| 3 | Approve 流程 | 弹出确认弹窗 → 确认 → Todo 消失 → Toast 提示 |
| 4 | Reject 流程 | 弹出底部 Sheet → Comment 为空时 Confirm 禁用 → 提交 → Todo 消失 |
| 5 | 驳回通知 | 驳回成功后，图纸上传人在 PC 端收到站内消息 |
