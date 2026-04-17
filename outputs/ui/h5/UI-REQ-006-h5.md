# UI 说明文档 — H5 图纸二维码公开状态页

> **来源需求**: [REQ-006-h5](../../../requirements/h5/REQ-006-h5.md) + [REQ-006-shared](../../../requirements/shared/REQ-006-shared.md)
> **产品**: SMART SITE SYSTEM
> **平台**: 移动端 H5（任何浏览器，无需登录；桌面浏览器兼容访问）
> **生成日期**: 2026-04-17

---

## 1. 设计目标

- 为**非系统用户**提供极简直接的扫码体验：打开页面 1 秒内看到核心状态结论
- 用强烈的视觉语言区分 **Active** / **Deprecated** 状态（绿色 / 红色徽章）
- 对存在局部更新的 Active 版本，**不让用户遗漏**（橙色警示）
- 对 Deprecated 版本，**劝阻继续使用**并引导获取最新版本（红色警示）
- 严格遵循**最小信息披露**原则：不展示图纸文件、附件、人员、项目等敏感信息
- 移动端优先设计，同时兼容 PC 浏览器（某人复制 URL 粘贴到桌面浏览器时仍然可用）

---

## 2. 页面清单与用户流程

### 2.1 页面

| 序号 | 页面 | 路由 | 说明 |
|------|------|------|------|
| 1 | Active 状态页 | `/public/drawing-status/{publicToken}` | 版本仍有效，可能附带局部更新提醒 |
| 2 | Deprecated 状态页 | 同上 | 版本已作废，引导用户关注最新版本 |
| 3 | 无效 QR 错误页 | 同上 | token 无效 / 不存在时展示 |
| 4 | 加载/错误态 | 同上 | 请求中、超时、网络错误的中间态 |

### 2.2 核心用户流程

```
现场用户扫描图纸二维码（无账号）：

  手机相机扫码
       │
       ▼
  浏览器打开 H5 URL
       │ 请求 GET /public/drawing/status/{publicToken}
       ▼
  加载骨架屏（< 1s）
       │
       ├─ ACTIVE 响应
       │    ▼
       │  绿色徽章 "ACTIVE"
       │  （若 activeMarkupCount > 0）
       │   显示 "· N Active Markups" 橙色提示
       │  图纸信息 + 底部说明文案
       │
       ├─ DEPRECATED 响应
       │    ▼
       │  红色徽章 "DEPRECATED"
       │  已扫版本信息 + 最新版本摘要
       │  红色警示 "Do NOT use..."
       │
       └─ 1003006001 错误
            ▼
          错误页 "QR Code Not Valid"
```

---

## 3. 组件设计规范

### 3.1 全局容器

| 属性 | 规格 |
|------|------|
| 背景 | `--bg-app` #F5F7FA（浅灰，衬托白色卡片） |
| 内容最大宽度 | 480px；>= 480px 时容器居中，两侧留白 |
| 外边距 | 页面顶部 16px，左右 16px，底部 24px |
| 基础字体 | 系统字体栈：`-apple-system, BlinkMacSystemFont, 'PingFang SC', 'Helvetica Neue', sans-serif` |
| 基础字号 | 15px |
| 行高 | 1.5 |
| 文字颜色 | `--color-text-primary` #303133 |

### 3.2 页头

| 属性 | 规格 |
|------|------|
| 文案 | "Drawing Status" / "图纸状态" |
| 字号 | 18px |
| 字重 | 600 `--font-weight-semibold` |
| 对齐 | 居中 |
| 外边距 | top 16px，bottom 24px |
| 颜色 | `--color-text-primary` |

---

### 3.3 ACTIVE 状态页

#### 3.3.1 页面布局

```
┌──────────────────────────────────┐
│         Drawing Status           │
├──────────────────────────────────┤
│  ┌────────────────────────────┐  │
│  │  🟢  ACTIVE                │  │   ← 状态徽章
│  │  · 3 Active Markups        │  │   ← markup > 0 时附加
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │  Drawing Code   ARCH-001   │  │
│  │  Drawing Name   首层平面图  │  │
│  │  Category       Structural │  │
│  │  Version        V2         │  │
│  │  Approved At    2026-04-10 │  │
│  └────────────────────────────┘  │
│                                  │
│  This drawing version is         │   ← 默认说明文案
│  currently in use.               │
│                                  │
│  ┌────────────────────────────┐  │   ← markup > 0 时显示
│  │ ⚠️ There are 3 active      │  │
│  │ markups. Please consult    │  │
│  │ the design team...         │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

#### 3.3.2 状态徽章（Active）

| 属性 | 规格 |
|------|------|
| 容器 | 白底卡片 #FFFFFF，圆角 12px，阴影 `0 1px 4px rgba(0,0,0,0.06)` |
| 主背景（徽章内部） | `#E8F5E9`（浅绿） |
| 主文字颜色 | `#2E7D32`（深绿） |
| 内边距 | 16px 20px |
| 图标 | 🟢（Emoji）或圆形绿色点 `●`，直径 10px，颜色 `#2E7D32` |
| 主文字 | "ACTIVE" 全大写，16px，字重 600，图标右侧间距 8px |
| Markup 附加 | 换行显示 "· {n} Active Markups"；橙色 `#EF6C00`；14px；图标前置间距 8px；count = 0 时此行不显示 |

#### 3.3.3 图纸信息卡

```
┌────────────────────────────┐
│  Drawing Code    ARCH-001  │
│  Drawing Name    首层平面图 │
│  Category        Structural│
│  Version         V2        │
│  Approved At     2026-04-10│
└────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| 容器 | 白底 #FFFFFF，圆角 12px，内边距 16px 20px，顶部外边距 12px |
| 布局 | Flexbox 每行 Label/Value 两列，`justify-content: space-between` |
| 行间距 | 12px |
| Label | 14px，`--color-text-secondary` #909399 |
| Value | 15px，`--color-text-primary` #303133，右对齐，`word-break: break-all`（图纸名称可能较长） |
| Category 显示 | 按用户语言显示（英文 Architectural / 中文 建筑），遵循 [REQ-003-shared § 2.6](../../../requirements/shared/REQ-003-shared.md) |
| Approved At 格式 | "YYYY-MM-DD"（仅日期，不显示时间） |

#### 3.3.4 底部说明文案

**默认文案（无 Markup）**:

| 属性 | 规格 |
|------|------|
| 容器 | 无底框；外边距 top 16px |
| 文字 | "This drawing version is currently in use." / "此图纸版本为当前有效版本。" |
| 字号 | 14px |
| 字重 | 400 |
| 颜色 | `--color-text-regular` #606266 |
| 对齐 | 左对齐 |

**Markup 警示文案（count > 0）**:

| 属性 | 规格 |
|------|------|
| 容器 | 橙色底 `#FFF3E0`，圆角 8px，左侧 3px 橙色竖条 `#EF6C00`，内边距 12px 14px，顶部外边距 12px |
| 图标 | ⚠️ Emoji，前置于文字 |
| 文字 | "There are {n} active markups. Please consult the design team for the latest changes before proceeding with construction." / "该图纸存在 {n} 条活跃局部更新。请在施工前向设计团队确认最新改动。" |
| 字号 | 14px |
| 颜色 | `#EF6C00`（橙色主色） |

---

### 3.4 DEPRECATED 状态页

#### 3.4.1 页面布局

```
┌──────────────────────────────────┐
│         Drawing Status           │
├──────────────────────────────────┤
│  ┌────────────────────────────┐  │
│  │  🔴  DEPRECATED            │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │  Drawing Code   ARCH-001   │  │
│  │  Drawing Name   首层平面图  │  │
│  │  Category       Structural │  │
│  │  Version        V1         │  │
│  │  Approved At    2026-03-20 │  │
│  └────────────────────────────┘  │
│                                  │
│  ──── Superseded by ────         │
│                                  │
│  ┌────────────────────────────┐  │
│  │  Latest Version   V3       │  │
│  │  Approved At      2026-04-15│ │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │ ⚠️ This version has been   │  │
│  │ superseded. Do NOT use...  │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

#### 3.4.2 状态徽章（Deprecated）

| 属性 | 规格 |
|------|------|
| 容器 | 同 Active 白底卡片 |
| 主背景 | `#FFEBEE`（浅红） |
| 主文字颜色 | `#C62828`（深红） |
| 图标 | 🔴 或红色圆点 |
| 主文字 | "DEPRECATED" 全大写，16px，字重 600 |
| 附加信息 | **不显示**任何 Markup 数量或其他附加信息 |

#### 3.4.3 已扫版本信息卡

与 §3.3.3 规格一致，显示被扫描版本（而非最新版本）的 Code / Name / Category / Version / Approved At。

#### 3.4.4 "Superseded by" 分隔

| 属性 | 规格 |
|------|------|
| 布局 | 水平线 + 居中文字 + 水平线 |
| 文字 | "Superseded by" / "已被替代为" |
| 字号 | 13px |
| 颜色 | `--color-text-secondary` #909399 |
| 水平线 | 1px 细线，颜色 `--color-border-light` #EBEEF5，左右各 Flex: 1 |
| 外边距 | top 20px, bottom 16px |

#### 3.4.5 最新版本信息卡

| 属性 | 规格 |
|------|------|
| 容器 | 白底 #FFFFFF，圆角 12px，内边距 16px 20px |
| 字段 | 仅两行：`Latest Version`（版本号）+ `Approved At`（审批日期） |
| Label/Value 样式 | 同 §3.3.3 |

#### 3.4.6 底部警示文案

| 属性 | 规格 |
|------|------|
| 容器 | 红色底 `#FFEBEE`，圆角 8px，左侧 3px 红色竖条 `#C62828`，内边距 12px 14px，顶部外边距 16px |
| 图标 | ⚠️ Emoji |
| 文字 | "This version has been superseded. Do NOT use this drawing for construction. Please obtain the latest version from your project team." / "此版本已作废。请勿按此图纸施工，请向项目团队索取最新版本。" |
| 字号 | 14px |
| 字重 | 500（比 Active 警示略重，强调严重性） |
| 颜色 | `#C62828` |

---

### 3.5 无效 QR 错误页

#### 3.5.1 页面布局

```
┌──────────────────────────────────┐
│                                  │
│              ❌                  │
│                                  │
│       QR Code Not Valid          │
│                                  │
│  The QR code on this drawing is  │
│  invalid or has expired. Please  │
│  contact your project            │
│  administrator.                  │
│                                  │
└──────────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| 容器 | 白底 #FFFFFF，圆角 12px，内边距 32px 20px |
| 图标 | ❌ Emoji，居中，字号 48px，外边距 bottom 16px |
| 主标题 | "QR Code Not Valid" / "二维码无效"，18px，字重 600，`--color-text-primary`，居中 |
| 说明文字 | 14px，`--color-text-regular`，居中，行高 1.6，最大宽度 320px |
| 无重试按钮 | token 无效通常为永久性问题，不提供 [Retry] |
| 顶部距离 | 距页头 40px |

---

### 3.6 加载与中间态

#### 3.6.1 加载骨架屏

- 页面打开即渲染骨架：空徽章卡 + 5 行 Label/Value 占位
- 骨架颜色：`#F0F2F5`（浅灰），shimmer 动画（linear-gradient 左右扫描）
- 加载时间 < 300ms 时可跳过骨架直接渲染数据

#### 3.6.2 网络错误页

```
┌──────────────────────────────────┐
│                                  │
│              ⚠️                  │
│                                  │
│     Failed to Load Status        │
│                                  │
│  Please check your connection    │
│  and try again.                  │
│                                  │
│       ┌──────────────┐           │
│       │    Retry     │           │
│       └──────────────┘           │
│                                  │
└──────────────────────────────────┘
```

| 属性 | 规格 |
|------|------|
| 图标 | ⚠️ Emoji，48px |
| 主标题 | "Failed to Load Status" / "加载失败" |
| 说明 | "Please check your connection and try again." / "请检查网络后重试" |
| Retry 按钮 | 主按钮样式，背景 `--color-primary` #409EFF，白色文字，圆角 8px，内边距 10px 24px，16px 字号 |
| 触发 | 请求超时（>10s）/ 网络失败时展示 |

---

## 4. 设计令牌

### 4.1 颜色

| 令牌 | 值 | 用途 |
|------|-----|------|
| `--color-primary` | #409EFF | Retry 按钮 |
| `--color-text-primary` | #303133 | 主标题、Value |
| `--color-text-regular` | #606266 | 正文 |
| `--color-text-secondary` | #909399 | Label、"Superseded by" |
| `--color-border-light` | #EBEEF5 | 分隔线 |
| `--bg-app` | #F5F7FA | 页面整体背景 |
| `--status-active-bg` | #E8F5E9 | Active 徽章背景 |
| `--status-active-fg` | #2E7D32 | Active 徽章文字 |
| `--status-deprecated-bg` | #FFEBEE | Deprecated 徽章背景 |
| `--status-deprecated-fg` | #C62828 | Deprecated 徽章文字、严重警示 |
| `--status-warning-bg` | #FFF3E0 | Markup 警示背景 |
| `--status-warning-fg` | #EF6C00 | Markup 警示文字 |

### 4.2 尺寸

| 令牌 | 值 | 用途 |
|------|-----|------|
| `--radius-card` | 12px | 卡片圆角 |
| `--radius-alert` | 8px | 警示块圆角 |
| `--spacing-card-gap` | 12px | 卡片间距 |
| `--shadow-card` | 0 1px 4px rgba(0,0,0,0.06) | 卡片阴影 |

---

## 5. 响应式与适配

| 场景 | 规则 |
|------|------|
| 移动端 ≤ 480px | 容器占满宽度，左右 16px 内边距 |
| 移动端 > 480px 且 < 768px | 容器最大 480px，居中，两侧自适应 |
| 桌面端 ≥ 768px | 同上，容器 480px 居中，页面整体不变形 |
| 横屏 | 布局不调整，内容自然滚动 |
| iOS 安全区 | top / bottom 使用 `env(safe-area-inset-*)` 适配 |

---

## 6. 无障碍（A11y）

| 项 | 要求 |
|----|------|
| 色彩对比度 | 所有文字与背景对比度 ≥ 4.5:1（符合 WCAG AA） |
| 语义化 | 状态徽章使用 `role="status"`，图纸信息使用 `<dl>/<dt>/<dd>` |
| 屏幕阅读 | "ACTIVE" / "DEPRECATED" 文字本身即为状态信息，无需额外 aria 属性 |
| 键盘支持 | Retry 按钮支持 Tab 聚焦，Enter 触发 |
| 动效控制 | 骨架屏动效在 `prefers-reduced-motion: reduce` 媒体查询下关闭 |

---

## 7. 国际化对照表

| Key | 英文 | 中文 |
|-----|------|------|
| page.title | Drawing Status | 图纸状态 |
| status.active | ACTIVE | 有效 |
| status.deprecated | DEPRECATED | 已作废 |
| status.activeMarkupCount | · {n} Active Markups | · {n} 条活跃局部更新 |
| field.drawingCode | Drawing Code | 图纸编号 |
| field.drawingName | Drawing Name | 图纸名称 |
| field.category | Category | 分类 |
| field.version | Version | 版本 |
| field.approvedAt | Approved At | 审批通过时间 |
| active.defaultHint | This drawing version is currently in use. | 此图纸版本为当前有效版本。 |
| active.markupWarning | There are {n} active markups. Please consult the design team for the latest changes before proceeding with construction. | 该图纸存在 {n} 条活跃局部更新。请在施工前向设计团队确认最新改动。 |
| deprecated.supersededBy | Superseded by | 已被替代为 |
| deprecated.latestVersion | Latest Version | 最新版本 |
| deprecated.warning | This version has been superseded. Do NOT use this drawing for construction. Please obtain the latest version from your project team. | 此版本已作废。请勿按此图纸施工，请向项目团队索取最新版本。 |
| error.invalidQrTitle | QR Code Not Valid | 二维码无效 |
| error.invalidQrBody | The QR code on this drawing is invalid or has expired. Please contact your project administrator. | 此图纸上的二维码无效或已失效。请联系项目管理员。 |
| error.networkTitle | Failed to Load Status | 加载失败 |
| error.networkBody | Please check your connection and try again. | 请检查网络后重试。 |
| error.retry | Retry | 重试 |

> **语言判定**：依据 `Accept-Language` Header；`zh-*` → 简体中文，其他 → 英文。**不依赖**登录态或用户偏好。

---

## 8. 验收标准

### ACTIVE 状态页
- [ ] 绿色徽章正确展示 "ACTIVE" 全大写文字
- [ ] `activeMarkupCount > 0` 时徽章内显示 `· {n} Active Markups` 橙色附加信息
- [ ] `activeMarkupCount = 0` 时徽章仅显示 "ACTIVE"，无附加信息
- [ ] 图纸信息区按 Code / Name / Category / Version / Approved At 顺序展示 5 行
- [ ] Category 按语言正确显示（Architectural / 建筑 等）
- [ ] 默认文案与 Markup 警示文案根据 count 正确切换

### DEPRECATED 状态页
- [ ] 红色徽章正确展示 "DEPRECATED" 全大写文字
- [ ] 徽章**不显示**任何 Markup 数量或附加信息
- [ ] 已扫版本信息卡显示被扫版本（非最新）
- [ ] "Superseded by" 分隔线 + 文字正确展示
- [ ] 最新版本信息卡仅显示 `Latest Version` + `Approved At` 两行
- [ ] 红色警示文案正确展示（"Do NOT use..."）

### 错误与异常
- [ ] 1003006001 错误渲染 "QR Code Not Valid" 错误页，无任何图纸信息泄露
- [ ] 错误页**不提供** Retry 按钮
- [ ] 网络超时 / 失败显示 "Failed to Load Status" 页面与 Retry 按钮
- [ ] 骨架屏加载时长 < 1s（正常网络下）

### 隐私合规
- [ ] 页面不加载任何图纸文件或附件 URL
- [ ] 页面无登录 / 注册 / 下载 APP 入口
- [ ] 页面不写入 Cookie 或 LocalStorage
- [ ] 无任何第三方埋点或分析脚本

### 响应式
- [ ] 移动端 ≤ 480px：容器占满，左右 16px 内边距
- [ ] 桌面端 ≥ 480px：容器 480px 居中
- [ ] iOS Safari 安全区正确适配

### 国际化
- [ ] 浏览器 `Accept-Language: zh-CN` 时显示中文
- [ ] 其他语言显示英文
- [ ] 所有文案根据语言正确切换

### 无障碍
- [ ] 所有文字对比度 ≥ 4.5:1
- [ ] Retry 按钮可通过键盘 Tab + Enter 触发
- [ ] `prefers-reduced-motion: reduce` 时关闭骨架屏动画

---

## 9. 相关文档

| 文档 | 说明 |
|------|------|
| [REQ-006-shared.md](../../../requirements/shared/REQ-006-shared.md) | 二维码生成规则、公开状态 API |
| [REQ-006-h5.md](../../../requirements/h5/REQ-006-h5.md) | H5 公开状态页功能需求 |
| [UI-REQ-006-pc.md](../pc/UI-REQ-006-pc.md) | PC 端 QR 管理 UI 规范 |
| [REQ-003-shared.md](../../../requirements/shared/REQ-003-shared.md) | Category 枚举来源 |
