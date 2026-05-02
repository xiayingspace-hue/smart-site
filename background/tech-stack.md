# 技术栈说明（Tech Stack）

> 本文档是 Smart Site System 各端技术栈的**单一事实源**。
> 所有需求文档的"技术说明"章节应引用本文档，不在各需求文件中重复列举。

---

## 1. PC 管理端

| 项目 | 规范 |
|------|------|
| 前端框架 | Vue 2 + Element UI |
| 编程语言 | JavaScript（ES6+） |
| 样式方案 | SCSS（Sassy CSS） |
| 构建工具 | Webpack |
| 包管理工具 | npm |
| 浏览器支持 | Chrome 80+、Firefox 78+、Safari 14+、Edge 80+ |
| 最低分辨率 | 1280×720px |
| 推荐分辨率 | 1920×1080px |

---

## 2. APP 移动端

| 项目 | 规范 |
|------|------|
| 主框架 | UNIAPP（跨平台移动开发框架） |
| 核心技术 | Vue 2（MVVM 框架） |
| 编程语言 | JavaScript（ES6+） |
| 样式预处理 | SCSS（Sassy CSS） |
| 包管理工具 | npm |
| iOS 支持 | iOS 12+ |
| Android 支持 | Android 8+ |
| 开发编辑器 | VS Code + UNIAPP 扩展 或 HBuilderX |

---

## 3. H5 端

> H5 为公开访问轻量页面，无需登录，独立于 PC 管理端部署，不引入 Element UI 等大体积依赖。

| 项目 | 规范 |
|------|------|
| 前端框架 | Vue 3 + Vite |
| 路由 | Vue Router 4 |
| HTTP | `fetch` API 或 Axios |
| 状态管理 | 无（独立无状态页面） |
| 国际化 | Vue I18n（仅中/英 2 种语言） |
| 样式 | CSS Variables + 原生 CSS 或 UnoCSS（无大型 UI 库） |
| Bundle 目标体积 | < 100KB |
| 构建目标 | ES2019+、iOS Safari 12+、Android Chrome 80+ |
| 隐私合规 | 无 Cookie / LocalStorage / 第三方脚本 |

---

## 4. 后端

| 项目 | 规范 |
|------|------|
| 编程语言 | Java |
| 架构 | Spring Cloud（微服务） |
| 数据库 | MySQL |
| 缓存 | Redis |
| 消息队列 | RabbitMQ |
| 搜索引擎 | Elasticsearch |
| 容器化 | Docker |
| API 风格 | 非 RESTful，Action-based（详见 `rules/backend-rules.md §8`） |
| API Gateway | `https://smartsite.mcc.sg` |

---

## 5. 跨端通用约定

| 项目 | 规范 |
|------|------|
| 认证方式 | JWT（Bearer Token） |
| 多租户 | URL 路径解析 `tenantCode`（PC）/ 表单输入（APP） |
| 请求 Header | `Authorization`、`X-Tenant-Id`、`Project-Id`、`lang` |
| 国际化默认语言 | English |
| 时区 | 服务器统一 UTC，前端按需格式化 |
