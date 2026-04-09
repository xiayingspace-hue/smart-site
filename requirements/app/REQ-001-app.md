# APP 移动端 — 登录和首页功能需求

> **端**: APP 移动端（iOS & Android）
> **共享需求**: [REQ-001-shared.md](../shared/REQ-001-shared.md)（业务规则、API 接口、会话管理）
> **本文档仅包含**: APP 端特有的 UI 布局、交互方式、样式、技术栈和非功能需求

## 基本信息

- **需求ID**: REQ-001-app
- **需求标题**: APP移动端登录和首页功能
- **产品**: SMART SITE SYSTEM
- **平台**: APP 移动端（iOS & Android）
- **技术框架**: UNIAPP（基于 Vue 2）
- **优先级**: 高
- **状态**: 草稿

## 技术说明

> 后端技术栈和系统架构详见 [REQ-001-shared.md](../shared/REQ-001-shared.md)

### APP 端技术栈
- **主框架**: UNIAPP（跨平台移动开发框架）
- **核心技术**: Vue 2（MVVM 框架）
- **编程语言**: JavaScript（ES6+）
- **样式预处理**: SCSS（Sassy CSS）
- **包管理工具**: npm（Node Package Manager）

### 平台支持
- **iOS**: 通过 UNIAPP 打包为 iOS 应用（支持 iOS 12+）
- **Android**: 通过 UNIAPP 打包为 Android 应用（支持 Android 8+）
- **一套代码**: 同一套 Vue 2 代码库可编译为 iOS 和 Android 版本

### 开发工具
- **编辑器**: VS Code + UNIAPP 扩展或 HBuilderX
- **包管理**: npm（安装依赖、管理版本）
- **调试**: UNIAPP 的模拟器或真机调试
- **打包**: UNIAPP 官方打包或云端打包服务

### APP 端项目结构
- **前端代码**: 使用 JavaScript 编写业务逻辑
- **样式代码**: 使用 SCSS 编写样式，可利用变量、混合、嵌套等高级特性
- **依赖管理**: 通过 package.json 和 npm 管理所有依赖包
- **编译流程**: 开发时使用热更新，打包时由 UNIAPP 编译为平台特定代码

## 需求描述

### 背景与目标

为了满足平台用户随时随地查看工地人员、设备等相关信息的需求，开发移动端 APP 应用。首先实现用户登录认证和首页基础框架，为后续功能（人员信息、设备信息等）奠定基础。

### 用户故事

```
作为平台用户
我想要 在移动端 APP 上登录并进入首页
以便 我可以随时随地查看工地的数据（人员、设备等信息）
```

## 功能需求

### 1. 登录页面

#### 页面整体布局
- **页面分为上下两个区域**：
  - **上半部分 — 品牌展示区域**：浅灰色背景，展示品牌 logo 和插画
  - **下半部分 — 表单卡片区域**：白色圆角卡片，覆盖在品牌区域下方，包含登录表单和操作按钮

#### 品牌展示区域（上半部分）
- **顶部工具栏**：
  - **左上角**：语言切换按钮（图标为"文A"样式的多语言切换图标），点击可切换 APP 显示语言（中文 / English）
  - **右上角**：显示 "Smart Site" 文字标识，左侧带有刷新/加载图标
- **品牌 Logo**：
  - 居中显示品牌名称 **"NOVASYNC"**（深色字体，粗体）
  - Logo 下方显示 **"S M A R T  S I T E"**（红色字体，字母间距加大）
- **插画区域**：
  - 居中展示一张智慧城市风格的 3D 插画图（包含建筑群、绿化、水景等元素）
  - 插画位于品牌 Logo 下方，与白色表单卡片有部分重叠
- **背景**：浅灰色渐变背景（#F5F7FA 至 #FFFFFF），底部有浅色波浪形装饰曲线

#### 表单卡片区域（下半部分）
- **卡片样式**：
  - 白色背景（#FFFFFF）
  - 顶部左右圆角：20-24px
  - 与品牌展示区域略有重叠（向上覆盖约20px）
  - 内边距：左右各 24px，上部 30px，下部 20px
- **表单字段**（从上到下排列）：
  - **企业账号（Company Account）**：
    - 字段标签：**"Company Account"**（黑色，14-16px，粗体）
    - 输入框样式：**下划线式输入框**（无边框，仅底部有灰色分隔线 #E0E0E0）
    - Placeholder: "Please enter company account"（灰色，14px）
    - 验证错误提示：红色文字 "Please enter company account" 显示在输入框下方分隔线之下
    - 高度：44-48px
    - 排序：第一位（顶部）
  - **用户账号（User Account）**：
    - 字段标签：**"User Account"**（黑色，14-16px，粗体）
    - 输入框样式：同企业账号（下划线式）
    - Placeholder: "Please enter user name"（灰色，14px）
    - 验证错误提示：红色文字 "Please enter user name" 显示在输入框下方分隔线之下
    - 高度：44-48px
    - 上边距：16px（与上一个字段间距）
  - **密码（Password）**：
    - 字段标签：**"Password"**（黑色，14-16px，粗体）
    - 输入框样式：同上（下划线式），输入类型为密码（掩码显示）
    - Placeholder: "Please enter password"（灰色，14px）
    - 验证错误提示：红色文字 "Please enter password" 显示在输入框下方分隔线之下
    - 高度：44-48px
    - 上边距：16px
- **操作按钮区域**：
  - **Sign In 按钮**：
    - 文本：**"Sign In"**（白色文字，16px，粗体）
    - 背景色：主品牌蓝色（#6892FF 渐变至 #8B9AFF，或纯色 #6979F8）
    - 宽度：占表单区域宽度的约 75%（左侧对齐）
    - 高度：48-52px
    - 圆角：24-26px（全圆角 / 胶囊形）
    - 上边距：30px（与密码字段间距）
  - **扫码登录按钮**：
    - 位于 Sign In 按钮右侧
    - 图标：QR 码扫描图标（线框风格）
    - 按钮样式：正方形，48-52px × 48-52px
    - 边框：1px 蓝色边框（与 Sign In 按钮同色系）
    - 背景色：白色/透明
    - 圆角：12px
    - 点击功能：打开摄像头扫描 QR 码进行快速登录
- **底部协议文本**：
  - 居中显示，上边距：16px
  - 第一行：**"By clicking 'Login'"**（灰色，12px）
  - 第二行：**"you agree to our [Terms of Conditions] and [Privacy Policy]"**（灰色，12px）
  - 其中 **"Terms of Conditions"** 和 **"Privacy Policy"** 为蓝色可点击链接
  - 点击 "Terms of Conditions"：跳转到服务条款页面（WebView 或外部浏览器）
  - 点击 "Privacy Policy"：跳转到隐私政策页面（WebView 或外部浏览器）

#### 输入验证
- 企业账号为必填项，长度 >= 2 个字符
- 用户账号为必填项，长度 >= 3 个字符
- 密码为必填项，长度 >= 6 个字符
- 点击 Sign In 时，进行实时验证并在对应字段下方显示红色错误提示
  - 企业账号为空：显示 "Please enter company account"
  - 企业账号过短：显示 "Company account must be at least 2 characters"
  - 用户账号为空：显示 "Please enter user name"
  - 用户账号过短：显示 "User name must be at least 3 characters"
  - 密码为空：显示 "Please enter password"
  - 密码过短：显示 "Password must be at least 6 characters"
  - 租户不存在：显示 "Tenant does not exist or is disabled"
  - 账号或密码错误：显示 "Incorrect account or password, please try again"

#### 语言切换功能
- 点击左上角语言切换按钮，弹出语言选择列表（如弹窗或下拉菜单）
- 支持语言：中文（简体）、English
- 切换语言后，页面所有文本（标签、Placeholder、按钮文字、错误提示、协议文字）即时切换
- 用户选择的语言偏好保存到本地存储，下次打开 APP 时自动应用

#### 扫码登录流程
1. 用户点击扫码登录按钮（Sign In 按钮右侧的 QR 码图标）
2. 系统请求摄像头权限（如未授权则提示用户授权）
3. 打开 QR 码扫描界面
4. 用户扫描 Web 端或管理后台生成的登录 QR 码
5. 扫描成功后，将 QR 码中的令牌信息发送到后端进行验证
6. 验证通过后，完成登录并跳转到首页
7. 验证失败时，提示用户 "QR code is invalid or expired, please try again"

#### 登录流程（账号密码登录）
1. 用户在 Company Account 输入框输入企业账号（即租户名称 / 租户 ID）
2. 用户在 User Account 输入框输入用户名
3. 用户在 Password 输入框输入密码
4. 点击 "Sign In" 按钮
5. 前端验证企业账号、用户账号、密码是否为空及格式是否正确
6. 如果验证不通过，在对应字段下方显示红色错误提示文字
7. 如果验证通过，提交到后端进行租户和身份认证
8. 显示加载动画（Sign In 按钮显示 loading 状态，禁止重复点击）
9. **登录成功**：
   - 自动导航到 APP 首页
   - 本地保存认证令牌（token）和租户标识（tenant ID）
   - 保存用户选择的语言偏好
10. **登录失败**：
    - 显示错误提示，根据失败原因：
      - "Tenant does not exist or is disabled"（租户验证失败）
      - "Incorrect account or password, please try again"（身份验证失败）
      - 或显示后端返回的具体错误信息
    - 密码框清空，企业账号和用户账号保留，方便重试

#### 样式要求
- **整体背景**：浅灰色渐变（#F5F7FA → #FFFFFF），带有轻微波浪曲线装饰
- **表单卡片**：白色背景（#FFFFFF），顶部圆角 20-24px，轻微阴影（box-shadow）
- **输入框**：下划线样式（underline），无边框，仅底部有 1px 灰色分隔线（#E0E0E0）
- **字段标签**：黑色（#333333），14-16px，粗体
- **Placeholder 文字**：灰色（#CCCCCC），14px
- **错误提示文字**：红色（#FF4D4F / #E74C3C），12-13px，显示在字段分隔线下方
- **Sign In 按钮**：蓝色渐变（#6892FF → #8B9AFF），白色文字，全圆角（胶囊形）
- **扫码按钮**：白色背景，蓝色边框，QR 码图标
- **协议链接文字**：蓝色（#6892FF），可点击，带下划线或无下划线
- **品牌 Logo**："NOVASYNC" 深色粗体，"SMART SITE" 红色字母间距加大

### 2. APP首页（Work Space）

#### 页面整体布局
- **页面结构**（从上到下）：
  - **顶部导航栏**：显示菜单按钮和页面标题
  - **安全标语横幅**：全宽图片横幅，展示安全相关的名言引用
  - **功能模块区域**：按项目分组的功能入口卡片（可上下滚动）
  - **底部 Tab 导航栏**：固定在页面底部的三个 Tab 按钮

#### 顶部导航栏
- **左侧**：汉堡菜单按钮（三条横线 ≡ 图标）
  - 点击从左侧滑出项目选择面板（Select Project）
- **中间**：页面标题 **"Smart Site"**（黑色，16-18px，粗体，居中显示）
- **右侧**：无按钮（或预留位置）
- **导航栏样式**：
  - 白色背景（#FFFFFF）
  - 底部有 1px 浅灰色分隔线（#F0F0F0）
  - 高度：44-48px（不含状态栏）

#### 安全标语横幅（Banner）
- **位置**：导航栏下方，左右各有 16px 外边距
- **样式**：
  - 圆角卡片：圆角 10px
  - 背景：深色抽象渐变图（蓝色流体风格背景图片）
  - 高度：约 110px
  - 上边距：16px（与导航栏间距）
- **内容**：
  - 白色引用文字（左对齐，16px，粗体）：**"Safety brings first aid to the uninjured."**
  - 白色作者署名（左对齐，10px，常规，Size 10px）：**"F.S. Hughes"**
  - 文字位于卡片左侧，留有左内边距 38px，垂直居中
- **交互**（可选）：
  - 支持横向轮播（Swiper），自动轮播多条安全标语
  - 可手动左右滑动切换
  - 底部显示分页指示点（如有多条）

#### 功能模块区域（主内容区）
- **布局方式**：垂直滚动列表（ScrollView），内容按项目/模块分组展示
- **左右外边距**：16px
- **上边距**：16px（与 Banner 间距）

##### 项目功能卡片（如 "RWS Waterfront"）
- **卡片样式**：
  - 白色背景（#FFFFFF）
  - 圆角：12-16px
  - 轻微阴影（box-shadow: 0 2px 8px rgba(0,0,0,0.06)）
  - 内边距：上 20px，左右 16px，下 20px
  - 卡片间距：16px（多个卡片之间的垂直间距）
- **卡片标题**：
  - 项目名称（如 **"RWS Waterfront"**），黑色，16-18px，粗体
  - 左对齐，距离卡片顶部 20px
  - 标题下方有 16px 间距后显示功能图标网格
- **功能图标网格**：
  - 采用 **3 列网格** 布局（每行 3 个图标）
  - 每个功能入口包含：
    - **图标**：圆角正方形背景（浅灰色 #F5F7FA，圆角 16px，尺寸 60-64px × 60-64px），内含功能图标（青绿色 / teal 色，线框风格，24-28px）
    - **文字标签**：图标下方居中显示功能名称（灰黑色 #333333，12-14px，常规）
  - 图标之间的水平间距：均匀分布（等间距）
  - 图标之间的垂直间距（行间距）：20-24px
  - 图标与文字间距：8-10px

##### 功能入口列表 — "RWS Waterfront" 项目卡片
| 位置 | 图标名称 | 英文标签 | 功能说明 |
|------|----------|----------|----------|
| 第1行-左 | 人员项目图标 | **Project** | 进入项目信息页面 |
| 第1行-中 | 分包商图标 | **Subcontractor** | 进入分包商管理页面 |
| 第1行-右 | 人员列表图标 | **Person List** | 进入人员列表页面 |
| 第2行-左 | 设备列表图标 | **Equipment List** | 进入设备列表页面 |
| 第2行-中 | CCTV 图标 | **CCTV** | 进入监控摄像头页面 |
| 第2行-右 | 用户反馈图标 | **User Feedback** | 进入用户反馈页面 |
| 第3行-左 | 文档图标 | **Document** | 进入文档管理页面 |
| 第3行-中 | 设备调遣图标 | **Equipment departure** | 进入设备调遣/出场记录页面 |
| 第3行-右 | 耗材图标 | **Consumables** | 进入耗材管理页面 |

##### 功能入口列表 — "Equipment" 模块卡片
- **卡片样式**：同项目功能卡片
- **卡片标题**：**"Equipment"**（黑色，16-18px，粗体）
- **功能图标网格**：同上述 3 列网格布局
- **功能入口**：

| 位置 | 图标名称 | 英文标签 | 功能说明 |
|------|----------|----------|----------|
| 第1行-左 | 加油图标 | **Refueling** | 进入设备加油记录列表页（REQ-002-app） |

> **说明**：Equipment 模块为独立卡片，与项目功能卡片同级。后续可扩展更多设备管理相关的功能入口（如设备维修、设备检查等）。

##### 功能入口列表 — "Vehicle Delivery Booking" 模块卡片
- **卡片样式**：同项目功能卡片
- **卡片标题**：**"Vehicle Delivery Booking"**（黑色，16-18px，粗体）
- **功能图标网格**：同上述 3 列网格布局
- **功能入口**：车辆配送预约相关的子功能（具体功能后续细化）

> **说明**：功能模块区域可能包含多个卡片（按项目或业务模块分组），每个卡片内展示该项目/模块下的所有功能入口。用户可上下滚动浏览所有卡片。卡片数量和内容根据用户权限和项目配置动态展示。

#### 底部 Tab 导航栏
- **固定在页面底部**，不随页面内容滚动
- **Tab 数量**：3 个
- **Tab 样式**：
  - 整体背景：白色（#FFFFFF）
  - 顶部有 1px 浅灰色分隔线（#F0F0F0）
  - 高度：50-56px（不含安全区域 / Safe Area）
  - 底部适配 iOS 安全区域（iPhone X 及以上机型的 Home Indicator 区域）
- **Tab 列表**（从左到右）：

| Tab 位置 | 图标 | 标签文字 | 功能说明 | 选中状态 |
|----------|------|----------|----------|----------|
| 左 | 四宫格图标（田字格 + 小星号） | **Work Space** | 当前页（首页 / 工作台） | 选中时图标和文字变为青绿色（teal，#00BFA5 / #26A69A） |
| 中 | 铃铛图标（通知图标） | **Notification** | 通知消息列表页 | 未选中时图标和文字为灰色（#999999） |
| 右 | 人物图标（用户轮廓） | **Profile** | 个人中心 / 个人资料页 | 未选中时图标和文字为灰色（#999999） |

- **Tab 交互**：
  - 点击切换 Tab，选中 Tab 的图标和文字变为青绿色（teal），其他 Tab 恢复灰色
  - 默认选中 **Work Space** Tab（首页）
  - 切换 Tab 时保持各页面状态（不重新加载）
  - Notification Tab 可显示未读消息数角标（红色圆点或数字）

#### 首页数据加载逻辑
1. 打开 APP 时检查本地是否有有效的登录令牌（token）和租户标识
   - 如果有：直接进入首页
   - 如果没有或已过期：跳转到登录页面
2. 进入首页后，调用后端 API 获取用户权限信息（`/system/auth/get-permission-info`）
3. 根据用户权限和关联项目，动态展示功能模块卡片：
   - 获取用户关联的项目列表（如 "RWS Waterfront"）
   - 获取每个项目下用户有权限访问的功能模块
   - 渲染对应的功能图标网格
4. 如果用户无任何项目权限，显示空状态提示："No project assigned. Please contact your administrator."
5. 功能入口点击后，导航到对应的功能详情页面（后续开发）

#### 项目选择面板（汉堡菜单展开后）
- 点击左上角汉堡菜单按钮（≡），从左侧滑出项目选择面板
- **面板顶部**：
  - **左侧**：返回按钮（＜ 箭头图标），点击关闭面板
  - **中间**：标题 **"Select Project"**（黑色，16-18px，粗体，居中显示）
- **项目列表**：
  - 列表样式：垂直列表，白色背景，每个项目占一行
  - 列表项样式：
    - 项目名称文字（黑色 / 深灰色，14-16px，常规），左对齐，左内边距 20px
    - 列表项高度：50-56px，垂直居中显示文字
    - 列表项之间有 1px 浅灰色分隔线（#F0F0F0）
  - **选中状态**：当前选中的项目行背景为浅青绿色 / 浅蓝色高亮（如 #E0F7FA / #E8F5E9），文字颜色变为青绿色（teal，#00BFA5 / #26A69A）
  - **未选中状态**：白色背景，黑色文字
  - 列表支持上下滚动（当项目数量较多时）
- **项目列表数据来源**：
  - 登录后从用户权限信息接口（`/system/auth/get-permission-info`）获取的 `projectInfos` 列表
  - 列表项显示项目全称（`fullName`）
- **示例项目列表**（根据用户权限动态展示）：
  - RWS Mario Kart Building
  - RWS Waterfront（当前选中）
  - Normanton Park
  - Rainforest North
  - One Bernam
  - Provence Residence
  - Hotel Mi
  - Rainforest South
  - North Gaia Turnstile
  - T311
  - MCC Innocentre
  - Sceneca Residence
  - The Landmark
  - LTA CR101
- **交互逻辑**：
  1. 点击汉堡菜单按钮（≡），面板从左侧滑入，同时右侧主页面区域覆盖半透明黑色遮罩层
  2. 用户在列表中点击某个项目名称：
     - 该项目行变为选中高亮状态
     - 面板自动关闭（向左滑出消失），遮罩层消失
     - 首页内容刷新为所选项目的功能模块卡片
     - 将选中的项目 ID 保存到本地存储，后续 API 请求在 Header 中传递 `Project-Id`
  3. 点击右侧空白区域（遮罩层）：面板关闭，不切换项目
  4. 点击左上角返回按钮（＜）：面板关闭，不切换项目
  5. 支持向左滑动手势关闭面板
- **面板样式**：
  - 宽度：屏幕宽度的约 75-80%
  - 白色背景（#FFFFFF）
  - 右侧有半透明黑色遮罩层（背景色 rgba(0,0,0,0.4)），点击遮罩可关闭面板
  - 滑入/滑出动画：300ms 缓动动画（ease-in-out）

#### 样式要求
- **页面背景**：浅灰色（#F5F7FA / #F8F9FB）
- **卡片背景**：白色（#FFFFFF），圆角 12-16px，轻微阴影
- **功能图标背景**：浅灰色圆角正方形（#F5F7FA / #F0F2F5）
- **功能图标颜色**：青绿色 / teal（#00BFA5 / #26A69A），线框风格（outlined）
- **标题文字**：黑色（#333333），16-18px，粗体
- **标签文字**：灰黑色（#333333 / #666666），12-14px，常规
- **Tab 选中色**：青绿色（#00BFA5 / #26A69A）
- **Tab 未选中色**：灰色（#999999 / #BDBDBD）

### 3. 会话管理

> 详见 [REQ-001-shared.md](../shared/REQ-001-shared.md) — 第 3 节「会话管理」

**APP 端特有说明**：
- Token 存储在安全存储中（iOS Keychain、Android Keystore）
- 使用 UNIAPP 的 `uni.setStorageSync` / `uni.getStorageSync` 管理非敏感数据
- 使用 UNIAPP 条件编译处理 iOS 和 Android 的存储差异

## API 接口定义

> 详见 [REQ-001-shared.md](../shared/REQ-001-shared.md) — 第 4 节「API 接口定义」
>
> 所有 API 接口定义（登录、分包商登录、Token 刷新、获取用户权限信息、登出）均为跨端共享，本文档不再重复。

## 验收标准

### 登录页面
- [ ] 页面上半部分显示 NOVASYNC 品牌 Logo 和 SMART SITE 文字标识
- [ ] 页面上半部分显示智慧城市风格的 3D 插画
- [ ] 左上角显示语言切换按钮（文A 图标），点击可切换中英文
- [ ] 右上角显示 "Smart Site" 文字标识
- [ ] 下半部分显示白色圆角表单卡片
- [ ] 表单卡片包含 Company Account、User Account、Password 三个输入字段
- [ ] 输入框为下划线样式（underline），非边框样式
- [ ] 每个输入字段有正确的英文标签和 Placeholder 文本
- [ ] Sign In 按钮为蓝色渐变、胶囊形（全圆角）样式
- [ ] Sign In 按钮右侧显示扫码登录按钮（QR 码图标）
- [ ] 底部显示协议文字 "By clicking 'Login' you agree to our Terms of Conditions and Privacy Policy"
- [ ] Terms of Conditions 和 Privacy Policy 为蓝色可点击链接
- [ ] 企业账号为空时提交显示红色错误提示 "Please enter company account"
- [ ] 用户账号为空时提交显示红色错误提示 "Please enter user name"
- [ ] 密码为空时提交显示红色错误提示 "Please enter password"
- [ ] 企业账号长度 < 2 时提交显示错误提示
- [ ] 用户账号长度 < 3 时提交显示错误提示
- [ ] 密码长度 < 6 时提交显示错误提示
- [ ] 输入的租户不存在时显示错误提示 "Tenant does not exist or is disabled"
- [ ] 输入正确信息后，点击 Sign In 按钮显示加载状态
- [ ] 登录成功后自动跳转到首页
- [ ] 登录失败时显示错误提示，且密码框清空
- [ ] 登录后本地保存认证令牌和租户标识
- [ ] 语言切换功能正常，切换后页面所有文本即时更新
- [ ] 扫码登录按钮点击后能正确打开摄像头扫描界面
- [ ] 扫码登录成功后能正确跳转到首页

### APP 首页
- [ ] 登录成功后能正确加载首页（Work Space）
- [ ] 顶部导航栏左侧显示汉堡菜单按钮（≡ 图标）
- [ ] 顶部导航栏中间居中显示 "Smart Site" 标题
- [ ] 导航栏下方显示安全标语横幅（Banner），包含引用名言和作者署名
- [ ] Banner 采用深色抽象渐变背景，圆角卡片样式
- [ ] Banner 下方显示按项目分组的功能模块卡片
- [ ] 功能模块卡片显示项目名称标题（如 "RWS Waterfront"）
- [ ] 功能图标以 3 列网格布局展示，每个包含图标和文字标签
- [ ] 功能图标为青绿色（teal）线框风格，背景为浅灰色圆角正方形
- [ ] "RWS Waterfront" 卡片包含 9 个功能入口：Project、Subcontractor、Person List、Equipment List、CCTV、User Feedback、Document、Equipment departure、Consumables
- [ ] 功能模块区域下方显示 "Equipment" 模块卡片，包含 Refueling 功能入口
- [ ] "Equipment" 模块卡片下方显示 "Vehicle Delivery Booking" 模块卡片
- [ ] 页面内容区域可上下滚动浏览所有功能模块卡片
- [ ] 底部固定显示 3 个 Tab：Work Space、Notification、Profile
- [ ] 默认选中 Work Space Tab，图标和文字为青绿色
- [ ] 未选中 Tab 的图标和文字为灰色
- [ ] 点击 Tab 能正确切换页面
- [ ] 点击汉堡菜单按钮能从左侧滑出项目选择面板
- [ ] 项目选择面板顶部显示返回按钮（＜）和标题 "Select Project"
- [ ] 项目选择面板显示用户关联的所有项目列表
- [ ] 当前选中的项目行显示高亮背景
- [ ] 点击项目列表中的某个项目后，面板自动关闭，首页刷新为所选项目内容
- [ ] 点击面板右侧空白区域（遮罩层）后面板关闭，不切换项目
- [ ] 点击返回按钮（＜）后面板关闭，不切换项目
- [ ] 首页根据用户权限动态展示功能模块（无权限的功能不显示）
- [ ] 无项目权限时显示空状态提示

### 会话管理

> 业务逻辑验收标准详见 [REQ-001-shared.md](../shared/REQ-001-shared.md) — 验收标准「会话管理」

**APP 端补充验收**：
- [ ] 关闭 APP 后重新打开，如果 token 和租户标识有效自动进入首页
- [ ] 关闭 APP 后重新打开，如果没有 token 或 token 过期返回登录页面
- [ ] Token 存储在 iOS Keychain / Android Keystore 安全存储中
- [ ] 非敏感数据使用 `uni.setStorageSync` 存储，符合 UNIAPP 规范

## 非功能需求

- **性能**: 
  - 登录页加载时间 < 1s
  - 登录请求响应时间 < 2s
  - 首页加载时间 < 1s

- **安全性**: 
  - 登录时使用 HTTPS 加密传输
  - 密码字段不保存在日志或调试信息中
  - Token 存储在安全存储（iOS Keychain、Android Keystore）中
  - Token 刷新与过期策略详见 [REQ-001-shared.md](../shared/REQ-001-shared.md)

- **可访问性**: 
  - 按钮和输入框有清晰的焦点状态
  - 错误提示文本清晰可读
  - 支持键盘导航（iOS）、辅助功能（Android）

- **兼容性**: 
  - 需支持 iOS 12+
  - 需支持 Android 8+
  - 适配各种屏幕尺寸（小到 4.5 英寸，大到 6.7 英寸）
  - 支持竖屏和横屏显示（可选）

## 相关需求

- [REQ-002-app](./REQ-002-app.md)：设备加油记录（Equipment 模块 → Refueling 入口）
- 后续需求 1：Project（项目信息）模块详情页
- 后续需求 2：Subcontractor（分包商管理）模块详情页
- 后续需求 3：Person List（人员列表）模块详情页
- 后续需求 4：Equipment List（设备列表）模块详情页
- 后续需求 5：CCTV（监控摄像头）模块详情页
- 后续需求 6：User Feedback（用户反馈）模块详情页
- 后续需求 7：Document（文档管理）模块详情页
- 后续需求 8：Equipment departure（设备调遣）模块详情页
- 后续需求 9：Consumables（耗材管理）模块详情页
- 后续需求 10：Vehicle Delivery Booking（车辆配送预约）模块详情页
- 后续需求 11：Notification（通知消息）页面
- 后续需求 12：Profile（个人中心）页面

## 备注

- 登录页面默认显示语言为英文（English），支持通过左上角语言切换按钮切换为中文
- 品牌名称为 "NOVASYNC"，产品名称为 "SMART SITE"
- 登录页面表单字段名称对应关系：
  - Company Account = 企业账号（对应后端的 Tenant / 租户概念）
  - User Account = 用户账号（对应后端的 username）
  - Password = 密码（对应后端的 password）
- 输入框采用下划线样式（underline style），非传统边框式（bordered style）
- Sign In 按钮采用胶囊形（全圆角）蓝色渐变设计
- 扫码登录为新增功能，需后端配合提供 QR 码生成和验证接口
- 底部协议文字中的 "Terms of Conditions" 和 "Privacy Policy" 需要提供对应的 URL 链接
- 首页功能模块卡片的内容根据用户权限和项目配置动态加载
- 首页功能入口图标统一使用青绿色（teal）线框风格，与品牌视觉一致
- "Vehicle Delivery Booking" 模块的具体子功能需后续细化
- 安全标语 Banner 的内容可配置或轮播多条
- 底部 Tab 导航为 3 个：Work Space（工作台）、Notification（通知）、Profile（个人中心）
- 可考虑支持生物识别登录（指纹、面容识别）作为后续功能
- 可考虑支持第三方登录（如微信、支付宝）作为后续功能
- **UNIAPP 技术说明**：
  - 本项目使用 UNIAPP 框架实现跨平台移动应用开发
  - 基于 Vue 2 的单一代码库可编译为 iOS 和 Android 版本
  - **前端代码**: 使用 JavaScript（ES6+）编写，遵循 UNIAPP 和 Vue 2 的语法规范
  - **样式代码**: 使用 SCSS 编写，支持变量、混合、嵌套等高级特性，提高样式复用性和可维护性
  - **包管理**: 使用 npm 作为唯一的包管理工具，所有依赖通过 package.json 管理
  - **条件编译**: 使用 UNIAPP 的条件编译指令实现平台差异化（如 iOS 和 Android 特定功能）
  - **平台 API**: 调用 UNIAPP 提供的平台 API（如存储、权限、系统功能等），自动适配 iOS 和 Android
  - **打包流程**：
    - 开发阶段：使用 npm 安装依赖，通过 UNIAPP 开发服务器热更新调试
    - iOS 打包：通过 UNIAPP 打包为 .ipa 文件，可通过 App Store 或企业分发
    - Android 打包：通过 UNIAPP 打包为 .apk 文件，可通过 Google Play 或企业分发
  - **开发建议**：
    - 充分利用 SCSS 变量管理颜色、尺寸等设计令牌
    - 使用 npm script 定义常用命令（dev、build、test 等）
    - 合理组织项目文件结构，如 components、pages、utils、assets 等目录
    - 在开发时注意 iOS 和 Android 的屏幕尺寸和交互差异
