---
doc_type: ui_spec
req_id: REQ-001-pc
version: 0.1.3
status: draft
generated_from: REQ-001-pc.md@0.2.0
generated_at: 2026-05-02
owner: ""
---

# UI 设计说明：PC 管理端 — 登录页面

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：[REQ-001-pc.md](../../../requirements/pc/REQ-001-pc.md)（主）
> - 输出引用：Figma 链接、交互原型链接
> - 不重复定义数据字段（参见 REQ-001-shared.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-001-pc @ v0.2.0 |
| 覆盖用户故事 | US-001、US-002、US-003 |
| 覆盖 AC | AC-001-pc-001 ~ AC-001-pc-008 |
| 上次同步时间 | 2026-05-02 |

> ⚠️ 当 REQ-001-pc.md 版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

为 PC 管理端提供简洁、安全的登录页面，支持 MAINCON / SUBCON 双角色切换，通过 URL 自动识别租户，无需用户手动输入企业账号。

### 1.2 设计原则

1. **简洁优先**：登录页不展示无关信息，最小化用户认知负担
2. **品牌一致**：主色 `#1890FF`（`--Primary-Color`），品牌文字全大写，与整体管理后台视觉统一
3. **错误即时**：校验失败必须有明确的视觉反馈（红色边框 + 文字提示）
4. **沉浸背景**：全屏城市夜景背景，登录卡片浮于其上，突出专业感

---

## 2. 信息架构

### 2.1 页面层级

```
登录页（/login）
└── 登录卡片
    ├── 品牌标识区（SMART CONSTRUCTION / BACK OFFICE）
    ├── 语言切换行（Account Login Page + 文A 图标）
    ├── Tab 区（MAINCON / SUBCON）
    ├── 登录表单（账号 + 密码）
    ├── 记住密码
    └── Login 按钮
```

### 2.2 导航与入口

| 入口位置 | 链接到 | 触发场景 |
|---------|-------|---------|
| Login 按钮（成功） | 管理后台首页 | 登录成功后路由跳转 |
| 语言图标（文A） | — | 弹出语言下拉菜单（⚠️ 必须是 click 触发的下拉菜单，选项：English / 中文；**禁止**直接在行内并排显示语言文字按钮） |

> ⚠️ **语言切换区域默认态说明**：
> - 页面默认只渲染**一个「文A」图标**，当前语言名称和备选语言名称**均不显示在页面上**
> - 点击「文A」图标后，弹出下拉菜单，菜单中才出现 `English` / `中文` 两个选项
> - 下拉收起后，页面恢复为仅显示「文A」图标的状态
> - **错误实现示例**（禁止）：`文A  English  中文` 三个元素并排显示在页面上

---

## 3. 页面 / 组件清单

### 3.1 登录页（/login）

**关联 Story**: US-001、US-002、US-003
**关联 AC**: AC-001-pc-001 ~ AC-001-pc-008

#### 布局结构

```
┌─────────────────────────────────────────────────────────────┐
│                   (全屏城市夜景背景图)                         │
│              ┌─────────────────────────┐                    │
│              │  SMART CONSTRUCTION     │                    │
│              │  BACK OFFICE            │                    │
│              │  Account Login Page 文A  │                    │
│              │  MAINCON    SUBCON      │                    │
│              │  ─────────              │                    │
│              │  [账号输入框]            │                    │
│              │  [密码输入框        👁]  │                    │
│              │  ☑ Remember the password│                    │
│              │  [     Login     ]      │                    │
│              └─────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

#### 状态（必填 5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空（URL 无效） | tenantCode 不存在 | 全屏错误提示，无登录卡片 | "Invalid access URL, please contact your administrator" |
| 加载中 | 点击 Login，接口请求中 | 按钮变 Loading（旋转图标），禁止点击 | "Login"（含 Spinner） |
| 正常 | tenantCode 有效，表单待填写 | 登录卡片完整渲染 | — |
| 错误（前端校验） | 点击 Login，字段为空 | 空字段边框变红（#FF4D4F），下方显示 "required" | "required" |
| 错误（接口失败） | 接口返回认证失败 | Toast 弹出，按钮恢复 | 后端返回错误信息 |
| 极端数据 | 账号 / 密码超长输入 | 输入框不应溢出卡片宽度，超长文字截断显示 | — |

#### 权限可见性（对应 REQ-001-pc §3）

| UI 元素 | MAINCON 管理员 | SUBCON 管理员 |
|--------|:---:|:---:|
| MAINCON Tab | 显示（默认选中） | 显示（不可登录） |
| SUBCON Tab | 显示 | 显示（可选中） |
| 语言切换图标 | 显示 | 显示 |

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色语义

| 用途 | Token | 值 |
|-----|-------|----|
| 品牌主色 / 按钮背景 / Tab 选中 | `--Primary-Color`（DS 原生）/ `--color-primary` | `#1890FF` ✅ Figma |
| 按钮 Hover | `--color-primary-hover` | 待确认（估算 `#096dd9`） |
| 按钮 Active | `--color-primary-active` | 待确认（估算 `#0050b3`） |
| 错误状态 | `--color-error` | `#FF4D4F` |
| 输入框默认边框 | `--border-color`（DS 原生） | `#D8E2F0` ✅ Figma |
| 输入框 聚焦边框 | `--Primary-Color`（DS 原生） | `#1890FF` ✅ Figma |
| 输入框 背景 | `--fill-color-blank`（DS 原生） | `#FFF` ✅ Figma |
| 输入框 placeholder 文字 | `--text-color-placeholder`（DS 原生） | `#B7C1D1` ✅ Figma |
| Checkbox 边框 | `--border-color`（DS 原生） | `#D8E2F0` ✅ Figma |
| Checkbox 背景 | `--fill-color-blank`（DS 原生） | `#FFF` ✅ Figma |
| Remember Password 文字 | `--Primary-Color`（DS 原生） | `#1890FF` ✅ Figma |
| Tab 选中文字 / 选中底线 | `--Primary-Color`（DS 原生） | `#1890FF` ✅ Figma |
| Tab 底线（未选中侧） | `--text-color-placeholder`（DS 原生） | `#B7C1D1` ✅ Figma |
| 品牌主标题 / 副标题文字 | `--text-color-primary-dark`（DS 原生） | `#344050` ✅ Figma |
| 语言行标题文字（Account Login Page） | `--text-color-primary-dark`（DS 原生） | `#344050` ✅ Figma |
| 语言图标 | `--color-icon-muted` | `#666666` |
| 卡片背景 | `--Vertical-Menu-White`（DS 原生） | `#FFFFFF` ✅ Figma |
| 页面背景 | — | `linear-gradient(70deg, rgba(8,24,38,0.60) 17.79%, rgba(0,37,66,0.60) 65.66%, rgba(3,147,255,0.60) 91.96%)` ✅ Figma |

### 4.2 间距与尺寸

| 组件 | 属性 | 值 | 来源 |
|-----|------|----|------|
| 登录卡片 | display | `inline-flex` | Figma |
| 登录卡片 | flex-direction | `column` | Figma |
| 登录卡片 | align-items | `flex-start` | Figma |
| 登录卡片 | 内边距（上） | `48px` | Figma ✅ |
| 登录卡片 | 内边距（右） | `48px` | Figma ✅ |
| 登录卡片 | 内边距（下） | `60px` | Figma ✅ |
| 登录卡片 | 内边距（左） | `48px` | Figma ✅ |
| 登录卡片 | gap（sections 间距） | `32px` | Figma ✅ |
| 登录卡片 | 圆角 | `8px` | Figma ✅ |
| 登录卡片 | 背景色 Token | `--Vertical-Menu-White` = `#FFF` | Figma ✅ |
| 登录卡片 | 宽度 | `336px` | Figma ✅ |
| 登录卡片 | 阴影 | なし（无阴影） | Figma ✅ |
| 品牌标识区（容器） | display | `flex` | Figma ✅ |
| 品牌标识区（容器） | width | `184px` | Figma ✅ |
| 品牌标识区（容器） | flex-direction | `column` | Figma ✅ |
| 品牌标识区（容器） | justify-content | `center` | Figma ✅ |
| 品牌标识区（容器） | align-items | `flex-start` | Figma ✅ |
| 品牌主副标题间距 | gap | 无（两行文字同一文本块，间距由 `line-height: 14px` 控制） | Figma ✅ |
| 品牌区下边距（到 Account Login Page） | margin-bottom | `16px` | Figma ✅ |
| input-group（Tab + 表单 + Checkbox） | gap（内部各行间距） | `24px` | Figma ✅ |
| Tab 行（容器） | display | `flex` | Figma ✅ |
| Tab 行（容器） | align-items | `center` | Figma ✅ |
| Tab 行（容器） | flex | `1 0 0` | Figma ✅ |
| 两 Tab 整体（含底线） | display | `flex` | Figma ✅ |
| 两 Tab 整体（含底线） | width | `240px` | Figma ✅ |
| 两 Tab 整体（含底线） | align-items | `flex-start` | Figma ✅ |
| 两 Tab 整体（含底线） | border-bottom | `1px solid var(--text-color-placeholder, #B7C1D1)` | Figma ✅ |
| Tab 选中指示线 | height | `2px` | Figma ✅ |
| Tab 选中指示线 | align-self | `stretch` | Figma ✅ |
| Tab 选中指示线 | background | `var(--Primary-Color, #1890FF)` | Figma ✅ |
| 输入框 | display | `flex` | Figma ✅ |
| 输入框 | 高度 | `32px` | Figma ✅ |
| 输入框 | padding | `5px 95px 5px 16px` | Figma ✅ |
| 输入框 | align-items | `center` | Figma ✅ |
| 输入框 | align-self | `stretch`（与卡片同宽） | Figma ✅ |
| 输入框 | 圆角 | `4px` | Figma ✅ |
| 输入框 | border（默认态） | `1px solid var(--border-color, #D8E2F0)` | Figma ✅ |
| 输入框 | border（聚焦态） | `1px solid var(--Primary-Color, #1890FF)` | Figma ✅ |
| 输入框 | 背景色 Token | `--fill-color-blank` = `#FFF` | Figma ✅ |
| 输入框 | box-shadow（聚焦态） | `0 0 0 2px rgba(64, 158, 255, 0.00)` | Figma ✅ |
| 输入框 | **font-size** | **`14px`**（⚠️ 必须显式设置，禁止继承父级或浏览器默认 16px） | Figma ✅ §4.3 |
| 输入框 | **line-height** | **`22px`** | Figma ✅ §4.3 |
| 输入框 | **font-weight** | **`400`** | Figma ✅ §4.3 |
| 输入框 placeholder | **color** | **`#B7C1D1`**（`--text-color-placeholder`） | Figma ✅ §4.3 |
| 错误提示 | position | `absolute`（⚠️ **不占文档流**，不影响相邻输入框间距） | Figma |
| 错误提示 | margin-top | `2px`（相对输入框底部，偏移量） | 估算，需设计稿确认 |
| 错误提示 | 颜色 | `#FF4D4F`（`--color-error`） | Figma ✅ |
| input-group（⚠️ 间距说明） | gap | `24px` 为各行之间**唯一**间距来源；错误提示绝对定位，**不得**额外占据行高或 margin，否则视觉间距翻倍 | Figma ✅ |
| Remember Password 行 | display | `flex` | Figma ✅ |
| Remember Password 行 | align-items | `center` | Figma ✅ |
| Remember Password 行 | gap | `8px` | Figma ✅ |
| Remember Password 行 | align-self | `stretch` | Figma ✅ |
| Checkbox | display | `flex` | Figma ✅ |
| Checkbox | 宽度 | `14px` | Figma ✅ |
| Checkbox | 高度 | `14px` | Figma ✅ |
| Checkbox | align-items | `flex-start` | Figma ✅ |
| Checkbox | 圆角 | `2px` | Figma ✅ |
| Checkbox | border | `1px solid var(--border-color, #D8E2F0)` | Figma ✅ |
| Checkbox | 背景色 Token | `--fill-color-blank` = `#FFF` | Figma ✅ |
| Login 按钮 | display | `flex` | Figma ✅ |
| Login 按钮 | 宽度 | `240px`（固定，非全宽） | Figma ✅ |
| Login 按钮 | padding | `9px 20px` | Figma ✅ |
| Login 按钮 | justify-content | `center` | Figma ✅ |
| Login 按钮 | align-items | `center` | Figma ✅ |
| Login 按钮 | gap（icon 与文字间距） | `4px` | Figma ✅ |
| Login 按钮 | 圆角 | `4px` | Figma ✅ |
| Login 按钮 | 背景色 Token | `--Primary-Color` = `#1890FF` | Figma ✅ |

### 4.3 字体

> DS 字体家族：**Arial**（全局统一；Figma 中出现的 "Helvetica Neue" 一律视为 Arial）
> `letter-spacing: -0.01px`（DS 全局微调，见按钮文字确认值）

| 用途 | DS Token | 字号 | 行高 | 字重 | 颜色 | 来源 |
|-----|---------|------|------|------|------|------|
| 品牌主标题（SMART CONSTRUCTION） | `700 Bold` | 14px | 14px | 700 | `#344050`（`--text-color-primary-dark`） | Figma ✅ |
| 品牌副标题（BACK OFFICE） | `700 Bold` | 14px | 14px | 700 | `#344050`（`--text-color-primary-dark`） | Figma ✅ |
| 语言行标题（Account Login Page） | `700 SemiBold/B1-14-Base` | 14px | 22px | 700 | `#344050`（`--text-color-primary-dark`） | Figma ✅ |
| 语言图标 | 待确认 | 20px | — | — | `#666666` | 估算 |
| Tab 未选中文字 | `400 Regular` | 14px | 20px | 400 | `#5E6E82`（`--text-color-primary-light`） | Figma ✅ |
| Tab 选中文字 | `700 Bold` | 14px | 20px | 700 | `#1890FF`（`--Primary-Color`） | Figma ✅ |
| 输入框 placeholder | `400 Regular/B1-14-Base` | 14px | 22px | 400 | `#B7C1D1`（`--text-color-placeholder`） | Figma ✅ |
| 记住密码文字 | `400 Regular/B1-14-Base` | 14px | 22px | 400 | `#1890FF`（`--Primary-Color`） | Figma ✅ |
| Login 按钮文字 | `400 Regular/B1-14-Base` | 14px | 22px | 400 | `#FFF`（`--Vertical-Menu-White`） | Figma ✅ |

---

## 5. 响应式设计

| 断点 | 宽度 | 主要变化 |
|-----|------|---------|
| Desktop（目标） | ≥ 1280px | 登录卡片水平垂直居中，宽度 400~440px |
| Tablet | 768~1280px | 登录卡片可缩减至 360px，内边距适当收窄 |
| Mobile | < 768px | 超出最低支持范围（PC 端最低 1280×720px），不做额外适配 |

---

## 6. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| Tab 切换下划线移动 | 滑动 | 200ms | ease-in-out |
| 输入框聚焦边框变色 | 颜色过渡 | 150ms | ease |
| Login 按钮 Hover | 背景色变深 | 150ms | ease |
| Login 按钮 Loading | 旋转图标 | 持续 | linear |
| 错误提示出现 | 淡入 | 150ms | ease |
| Toast 出现 / 消失 | 淡入淡出 | 200ms | ease |
| 语言下拉菜单 | 展开 | 150ms | ease-out |

---

## 7. 无障碍（A11y）

- WCAG 等级：AA
- 颜色对比度：主色 `#4A90D9` on `#FFFFFF` 需检查 ≥ 3:1（大字体），错误红色 `#FF4D4F` on `#FFFFFF` 需 ≥ 4.5:1
- 键盘导航：Tab 键可在账号 → 密码 → 记住密码 → Login 按钮之间顺序切换；Enter 键可触发登录
- 焦点可见：聚焦态须有明显外轮廓（`outline: 2px solid #4A90D9`）
- 屏幕阅读器：错误提示使用 `aria-live="polite"` 通知；Loading 态按钮需 `aria-busy="true"`
- 密码切换按钮：需有 `aria-label`（"Show password" / "Hide password"）

---

## 8. 复用与新建组件清单

> 根据设计稿 Token（`--Primary-Color: #1890FF`、`--Vertical-Menu-White`），DS 基于 **Ant Design**，以下组件优先使用 Ant Design 现有实现。

| 组件 | 来源 | Ant Design 对应组件 | 备注 |
|-----|------|-------------------|------|
| 文本输入框（bordered 样式） | Ant Design DS | `Input` | bordered 为默认变体，error 态通过 `status="error"` 支持，可直接使用 |
| 密码输入框（含眼睛图标） | Ant Design DS | `Input.Password` | 内置眼睛图标切换，可直接使用 |
| Checkbox | Ant Design DS | `Checkbox` | 选中色跟随 `--Primary-Color: #1890FF`，与设计稿一致，可直接使用 |
| 主按钮（固定宽 240px / Loading 态） | Ant Design DS | `Button type="primary"` | Loading 态通过 `loading` prop 支持；宽度需自定义为 `240px`（非 Ant Design 默认） |
| Tab（下划线样式） | Ant Design DS | `Tabs type="line"` | 下划线为默认变体，可直接使用 |
| Toast / 消息提示 | Ant Design DS | `message` / `notification` | 可直接使用 |
| 下拉菜单（语言切换） | Ant Design DS | `Dropdown` + `Menu` | 可直接使用 |

---

## 9. 文案规范

| 场景 | 文案（zh-CN） | 文案（en） |
|-----|-------------|----------|
| 品牌主标题 | SMART CONSTRUCTION | SMART CONSTRUCTION |
| 品牌副标题 | BACK OFFICE | BACK OFFICE |
| 语言行标题 | 账号登录页 | Account Login Page |
| Tab — 总承包商 | MAINCON | MAINCON |
| Tab — 分包商 | SUBCON | SUBCON |
| 账号 Placeholder | 请输入账号 | Please fill in account |
| 密码 Placeholder | 请输入密码 | Please fill in password |
| 必填提示 | 必填 | required |
| 记住密码 | 记住密码 | Remember the password |
| 登录按钮 | 登录 | Login |
| URL 无效提示 | 访问链接无效，请联系管理员 | Invalid access URL, please contact your administrator |

---

## 10. Figma 与原型链接

- Figma 设计稿：<!-- 填写 REQ-001 登录页 Frame 链接 -->
- 交互原型：
- 设计系统：

---

## 11. AC 覆盖检查表

| AC ID | 对应页面 / 组件 | 对应状态 / 交互 | 覆盖？ |
|------|--------------|--------------|------|
| AC-001-pc-001 | 登录页 | 正常态 → Loading → 跳转 | ✅ |
| AC-001-pc-002 | 登录页 / SUBCON Tab | Tab 切换 → 正常态 → 跳转 | ✅ |
| AC-001-pc-003 | 登录表单 | 错误态（前端校验） | ✅ |
| AC-001-pc-004 | 登录页 | 空（URL 无效）全屏错误 | ✅ |
| AC-001-pc-005 | 登录页 | 错误态（接口失败）Toast | ✅ |
| AC-001-pc-006 | 密码输入框 | 眼睛图标交互 | ✅ |
| AC-001-pc-007 | 记住密码 Checkbox | 选中 / 取消状态 | ✅ |
| AC-001-pc-008 | 语言切换 | 下拉菜单 → 全卡片文案更新 | ✅ |

---

## 12. 待定问题（Open Questions）

| OQ ID | 问题 | 影响 UI 哪部分 |
|------|------|--------------|
| OQ-001 | 记住密码的存储策略（REQ-001-pc §16） | Checkbox 文案是否需要注解说明仅记账号不记密码 |

---

## 13. 验收标准（本文档自身）

- [ ] Figma 稿覆盖登录页所有组件
- [ ] 每个组件均有 5 态设计稿
- [ ] 每个 AC 都能在 §11 表中标记 ✅
- [ ] 与现有 DS 的差异已列入 §8 并经评审
- [ ] 所有 OQ 在 §12 显式列出，未编造方案

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.3 | 2026-05-03 | agent | 在 §4.2 输入框行内补充 font-size/line-height/font-weight/placeholder color，防止 agent 跨表遗漏导致继承浏览器默认 16px |
| 0.1.2 | 2026-05-03 | agent | 补充语言切换默认态约束：页面只显示「文A」图标，禁止并排显示语言文字；明确下拉菜单仅在点击后展开 |
| 0.1.1 | 2026-05-03 | agent | 补充两条实现约束：①语言切换明确为 click 触发下拉菜单，禁止行内并排；②错误提示 position:absolute 不占文档流，gap:24px 为唯一间距来源 |
| 0.1.0 | 2026-05-02 | agent | 从 REQ-001-pc.md v0.2.0 抽取视觉规格，初稿 |
