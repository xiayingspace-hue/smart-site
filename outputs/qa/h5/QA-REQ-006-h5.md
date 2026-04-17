# QA 测试说明文档 — REQ-006 H5 图纸二维码公开状态页

> **来源需求**: REQ-006-shared + REQ-006-h5
> **平台**: H5 移动端浏览器（无需登录；兼容桌面浏览器）
> **生成日期**: 2026-04-17

---

## 1. 测试目标

验证扫描图纸二维码后打开的 H5 公开状态页，确保 ACTIVE / DEPRECATED / 错误三种状态正确展示，以及隐私合规、多语言、响应式等非功能需求。

---

## 2. 测试场景与用例

### 场景 1：ACTIVE 状态页

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-H5-01 | 版本 `isCurrent=true, isDeprecated=false` | 手机扫描该版本 QR 码 | 页面显示绿色 `ACTIVE` 徽章 |
| TC-006-H5-02 | 同上，`activeMarkupCount = 3` | 查看徽章区域 | 徽章内显示 `· 3 Active Markups`（橙色 `#EF6C00`） |
| TC-006-H5-03 | 同上，`activeMarkupCount = 0` | 查看徽章区域 | 徽章内仅显示 `ACTIVE`，无附加信息 |
| TC-006-H5-04 | `activeMarkupCount > 0` | 查看底部文案区域 | 显示橙色警示块："⚠️ There are N active markups. Please consult the design team..." |
| TC-006-H5-05 | `activeMarkupCount = 0` | 查看底部文案区域 | 仅显示 "This drawing version is currently in use."，无橙色警示块 |
| TC-006-H5-06 | ACTIVE 版本 | 查看信息区域 | 按顺序展示 Drawing Code → Drawing Name → Category → Version → Approved At |

### 场景 2：DEPRECATED 状态页

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-H5-07 | 版本 `isDeprecated=true` | 手机扫描该版本 QR 码 | 页面显示红色 `DEPRECATED` 徽章 |
| TC-006-H5-08 | 同上 | 查看徽章区域 | 徽章内**不显示** Markup 数量 |
| TC-006-H5-09 | 同上 | 查看已扫版本信息 | 按顺序展示 Code、Name、Category、Version、Approved At |
| TC-006-H5-10 | 同上 | 查看 "Superseded by" 区域 | 显示最新版本号（Latest Version）和审批时间（Approved At） |
| TC-006-H5-11 | 同上 | 查看底部文案 | 红色警示块："⚠️ This version has been superseded. Do NOT use this drawing for construction..." |

### 场景 3：无效 Token 错误页

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-H5-12 | 使用不存在的 publicToken | 访问 URL | 显示 ❌ 图标 + "QR Code Not Valid" + 说明文案 |
| TC-006-H5-13 | 使用格式错误的 token | 访问 URL | 同上，统一错误页 |
| TC-006-H5-14 | 错误页 | 查看页面 | **不提供** `[Retry]` 按钮（token 无效为永久性问题） |
| TC-006-H5-15 | 错误页 | 查看页面信息 | 不暴露任何项目、图纸或人员信息 |

### 场景 4：加载与网络异常

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-H5-16 | 正常网络 | 扫码打开页面 | 先显示居中 skeleton loader，数据返回后渲染状态页 |
| TC-006-H5-17 | 模拟网络超时（> 10s） | 扫码打开页面 | 显示错误提示 + `[Retry]` 按钮 |
| TC-006-H5-18 | 模拟网络断开 | 扫码打开页面 | 显示错误提示 + `[Retry]` 按钮 |
| TC-006-H5-19 | 超时/断网后恢复网络 | 点击 `[Retry]` 按钮 | 重新发起请求，成功后正确渲染 |

### 场景 5：多语言

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-H5-20 | 浏览器语言设为 `zh-CN` | 扫码打开页面 | 页头显示 "图纸状态"，状态徽章显示 "有效" / "已作废"，标签为中文 |
| TC-006-H5-21 | 浏览器语言设为 `en-US` | 扫码打开页面 | 页头显示 "Drawing Status"，状态徽章显示 "ACTIVE" / "DEPRECATED"，标签为英文 |
| TC-006-H5-22 | 浏览器语言设为 `ja-JP`（非中文/英文） | 扫码打开页面 | 默认显示英文 |
| TC-006-H5-23 | 中文环境，ACTIVE + 3 个 Markup | 查看徽章附加信息 | 显示 "· 3 条活跃局部更新" |
| TC-006-H5-24 | 中文环境，Category = "Structural" | 查看 Category 字段 | 显示 "结构"（中文枚举值） |

### 场景 6：响应式

| ID | 前置条件 | 测试步骤 | 预期结果 |
|----|---------|---------|---------|
| TC-006-H5-25 | 手机屏幕（375px 宽） | 扫码打开页面 | 内容满宽显示，左右各 16px 边距，信息完整可见 |
| TC-006-H5-26 | 平板屏幕（768px 宽） | 访问同一 URL | 内容区域居中，最大宽度 480px |
| TC-006-H5-27 | PC 浏览器（1440px 宽） | 粘贴 URL 访问 | 内容居中，布局不变形 |

---

## 3. 隐私合规测试

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| PV-006-H5-01 | 页面内容 | 不展示图纸文件、附件 URL、下载链接 |
| PV-006-H5-02 | 页面内容 | 不展示项目名称 |
| PV-006-H5-03 | 页面内容 | 不展示任何人员姓名（上传人、审批人等） |
| PV-006-H5-04 | 页面交互 | 无登录入口、注册入口、APP 下载引导 |
| PV-006-H5-05 | 浏览器存储 | 页面不写入 Cookie 或 LocalStorage（DevTools 验证） |
| PV-006-H5-06 | 网络请求 | 无第三方埋点、分析、广告脚本的网络请求（DevTools Network 验证） |
| PV-006-H5-07 | API 响应 | 接口返回数据中无 fileUrl、tenantId、userId 等敏感字段 |

---

## 4. 界面验收

| ID | 检查项 | 预期结果 |
|----|--------|---------|
| UI-006-H5-01 | ACTIVE 徽章 | 绿色背景 `#E8F5E9`，绿色文字 `#2E7D32`，圆角 8px |
| UI-006-H5-02 | DEPRECATED 徽章 | 红色背景 `#FFEBEE`，红色文字 `#C62828`，圆角 8px |
| UI-006-H5-03 | Markup 附加信息 | 橙色 `#EF6C00`，14px |
| UI-006-H5-04 | ACTIVE 底部警示块 | 橙色底 `#FFF3E0`，圆角 8px |
| UI-006-H5-05 | DEPRECATED 底部警示块 | 红色底 `#FFEBEE`，圆角 8px |
| UI-006-H5-06 | 页头字号 | 18px，600 字重，居中 |
| UI-006-H5-07 | 信息区 Label | 14px，次要色 |
| UI-006-H5-08 | 信息区 Value | 15px，主要色 |

---

## 5. 非功能测试

| 类型 | 测试点 |
|------|-------|
| 性能 | 页面首次加载（含 API 请求）< 2s（4G 网络） |
| 限流 | 同一 IP 60 次/分钟后返回 HTTP 429 |
| 安全 | 修改 URL 中的 publicToken 为其他值，返回统一错误页（不泄露信息） |
| 安全 | 页面 HTML 源码中无敏感信息 |
| 兼容性 | iOS Safari 12+、Android Chrome 80+、微信内置浏览器 |

---

## 6. 验收标准

- [ ] ACTIVE 版本扫码后正确展示绿色徽章和图纸信息
- [ ] `activeMarkupCount > 0` 时显示橙色 Markup 提示和底部警示块
- [ ] `activeMarkupCount = 0` 时不显示 Markup 相关信息
- [ ] DEPRECATED 版本扫码后显示红色徽章和最新版本信息
- [ ] 无效 token 统一展示错误页，不暴露任何信息
- [ ] 网络异常时显示错误提示和 Retry 按钮
- [ ] 中英文根据浏览器语言自动切换
- [ ] 移动端和 PC 端访问布局均正常
- [ ] 页面不写入 Cookie / LocalStorage，无第三方脚本
- [ ] 限流规则生效
