# 前端开发说明文档 — H5 图纸二维码公开状态页

> **来源需求**: [REQ-006-h5.md](../../../requirements/h5/REQ-006-h5.md) + [REQ-006-shared.md](../../../requirements/shared/REQ-006-shared.md)
> **UI 设计参考**: [UI-REQ-006-h5.md](../../ui/h5/UI-REQ-006-h5.md)
> **产品**: SMART SITE SYSTEM
> **平台**: 移动端 H5（无需登录；兼容桌面浏览器）
> **生成日期**: 2026-04-17

---

## 1. 功能概述

实现扫描图纸二维码后打开的 H5 公开状态页，面向**非系统用户**，无需登录即可查看图纸版本状态。

| 功能模块 | 说明 |
|---------|------|
| 公开状态页路由 | `/public/drawing-status/:publicToken` |
| 状态展示 | ACTIVE（绿）/ DEPRECATED（红）/ 错误页 三种状态 |
| Markup 警示 | ACTIVE 且 `activeMarkupCount > 0` 时展示橙色警示 |
| 国际化 | 根据浏览器 `Accept-Language` 自动切换中英文 |
| 隐私合规 | 无 Cookie / LocalStorage / 第三方脚本 |
| 错误处理 | 无效 token 错误页、网络超时重试 |

---

## 2. 技术栈

| 项目 | 规范 |
|------|------|
| 框架 | 建议使用 **Vue 3 + Vite**（轻量、移动端友好，独立于 PC 管理端） |
| 路由 | Vue Router 4（Hash 或 History 模式均可） |
| HTTP | `fetch` API 或 Axios |
| 状态管理 | 无需 Vuex / Pinia（页面为独立无状态页面） |
| 国际化 | Vue I18n（或轻量自研，仅 2 种语言约 20 个 key） |
| 样式 | CSS Variables + 原生 CSS 或 UnoCSS；不引入大型 UI 库，保持 bundle 体积 < 100KB |
| 构建目标 | 现代浏览器（ES2019+）、iOS Safari 12+、Android Chrome 80+ |
| 无依赖 | 不集成任何埋点、分析、广告、客服 SDK |

> **为什么独立于 PC 管理端？** H5 公开页无登录、体积敏感、用户群完全不同（非系统用户），独立部署可避免加载 PC 管理端的 Element UI、Vuex 等大体积依赖。

---

## 3. 文件与目录结构

```
smart-site-h5/
├── index.html
├── vite.config.ts
├── src/
│   ├── main.ts                             # 入口
│   ├── App.vue                             # 根组件（仅 <RouterView />）
│   ├── router/
│   │   └── index.ts                        # 路由配置
│   ├── views/
│   │   └── DrawingStatusPage.vue           # 公开状态页（主视图）
│   ├── components/
│   │   ├── StatusBadge.vue                 # Active / Deprecated 状态徽章
│   │   ├── InfoCard.vue                    # 图纸信息卡（Label/Value 列表）
│   │   ├── AlertBox.vue                    # 警示文案组件
│   │   ├── SkeletonLoader.vue              # 加载骨架屏
│   │   └── ErrorState.vue                  # 错误 / 无效 Token 状态
│   ├── api/
│   │   └── public-status.ts                # 公开状态 API 封装
│   ├── i18n/
│   │   ├── index.ts                        # 语言检测与初始化
│   │   ├── en.ts                           # 英文语言包
│   │   └── zh.ts                           # 中文语言包
│   ├── utils/
│   │   └── format.ts                       # 日期格式化
│   └── styles/
│       ├── tokens.css                      # 设计令牌（CSS Variables）
│       └── reset.css                       # 基础重置样式
└── public/
    └── favicon.ico
```

---

## 4. 路由配置

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory('/public/'),
  routes: [
    {
      path: '/drawing-status/:publicToken',
      name: 'DrawingStatus',
      component: () => import('@/views/DrawingStatusPage.vue'),
      props: true
    },
    {
      path: '/:pathMatch(.*)*',
      redirect: { name: 'DrawingStatus', params: { publicToken: 'invalid' } }
    }
  ]
})

export default router
```

**部署路径**: `https://{H5_HOST}/public/drawing-status/{publicToken}`

---

## 5. 主视图 `DrawingStatusPage.vue`

### 5.1 组件结构

```vue
<template>
  <div class="page">
    <h1 class="page-title">{{ t('page.title') }}</h1>

    <SkeletonLoader v-if="state === 'loading'" />

    <ErrorState
      v-else-if="state === 'invalid'"
      type="invalid"
    />

    <ErrorState
      v-else-if="state === 'error'"
      type="network"
      @retry="fetchStatus"
    />

    <template v-else-if="state === 'success'">
      <!-- ACTIVE 状态 -->
      <template v-if="data.status === 'ACTIVE'">
        <StatusBadge
          type="active"
          :markup-count="data.activeMarkupCount"
        />
        <InfoCard :fields="activeFields" />
        <p v-if="data.activeMarkupCount === 0" class="default-hint">
          {{ t('active.defaultHint') }}
        </p>
        <AlertBox
          v-else
          type="warning"
          :text="t('active.markupWarning', { n: data.activeMarkupCount })"
        />
      </template>

      <!-- DEPRECATED 状态 -->
      <template v-else>
        <StatusBadge type="deprecated" />
        <InfoCard :fields="deprecatedCurrentFields" />
        <div class="divider">
          <span>{{ t('deprecated.supersededBy') }}</span>
        </div>
        <InfoCard :fields="latestVersionFields" />
        <AlertBox type="danger" :text="t('deprecated.warning')" />
      </template>
    </template>
  </div>
</template>
```

### 5.2 数据状态

```typescript
type PageState = 'loading' | 'success' | 'invalid' | 'error'

interface StatusData {
  drawingCode: string
  drawingName: string
  drawingCategory: string
  versionNo: string
  approvedTime: string
  status: 'ACTIVE' | 'DEPRECATED'
  activeMarkupCount: number | null
  latestVersion: { versionNo: string; approvedTime: string } | null
}

const state = ref<PageState>('loading')
const data = ref<StatusData | null>(null)
```

### 5.3 加载逻辑

```typescript
import { useRoute } from 'vue-router'
import { getPublicStatus } from '@/api/public-status'

const route = useRoute()
const publicToken = route.params.publicToken as string

async function fetchStatus() {
  state.value = 'loading'
  try {
    const controller = new AbortController()
    const timeoutId = setTimeout(() => controller.abort(), 10000)
    const res = await getPublicStatus(publicToken, controller.signal)
    clearTimeout(timeoutId)
    if (res.code === 1003006001) {
      state.value = 'invalid'
    } else if (res.code === 0) {
      data.value = res.data
      state.value = 'success'
    } else {
      state.value = 'error'
    }
  } catch (err) {
    state.value = 'error'
  }
}

onMounted(fetchStatus)
```

### 5.4 字段映射

```typescript
const activeFields = computed(() => [
  { label: t('field.drawingCode'), value: data.value?.drawingCode },
  { label: t('field.drawingName'), value: data.value?.drawingName },
  { label: t('field.category'), value: translateCategory(data.value?.drawingCategory) },
  { label: t('field.version'), value: data.value?.versionNo },
  { label: t('field.approvedAt'), value: formatDate(data.value?.approvedTime) }
])

const deprecatedCurrentFields = computed(() => activeFields.value)

const latestVersionFields = computed(() => [
  { label: t('deprecated.latestVersion'), value: data.value?.latestVersion?.versionNo },
  { label: t('field.approvedAt'), value: formatDate(data.value?.latestVersion?.approvedTime) }
])
```

---

## 6. 组件规格

### 6.1 `StatusBadge.vue`

```vue
<template>
  <div
    class="status-badge"
    :class="type"
    role="status"
  >
    <span class="dot" />
    <span class="label">{{ t(`status.${type}`) }}</span>
    <span
      v-if="type === 'active' && markupCount && markupCount > 0"
      class="markup-count"
    >
      · {{ t('status.activeMarkupCount', { n: markupCount }) }}
    </span>
  </div>
</template>

<script setup lang="ts">
interface Props {
  type: 'active' | 'deprecated'
  markupCount?: number | null
}
defineProps<Props>()
</script>

<style scoped>
.status-badge {
  background: #FFFFFF;
  border-radius: 12px;
  padding: 16px 20px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  font-size: 16px;
  font-weight: 600;
}
.status-badge .dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  margin-right: 8px;
}
.status-badge.active {
  color: var(--status-active-fg);
  background: var(--status-active-bg);
}
.status-badge.active .dot { background: var(--status-active-fg); }
.status-badge.deprecated {
  color: var(--status-deprecated-fg);
  background: var(--status-deprecated-bg);
}
.status-badge.deprecated .dot { background: var(--status-deprecated-fg); }
.markup-count {
  color: var(--status-warning-fg);
  font-size: 14px;
  font-weight: 500;
  flex-basis: 100%;
  margin-top: 6px;
}
</style>
```

### 6.2 `InfoCard.vue`

```vue
<template>
  <dl class="info-card">
    <div v-for="field in fields" :key="field.label" class="row">
      <dt class="label">{{ field.label }}</dt>
      <dd class="value">{{ field.value }}</dd>
    </div>
  </dl>
</template>

<script setup lang="ts">
interface Field { label: string; value: string | undefined }
defineProps<{ fields: Field[] }>()
</script>

<style scoped>
.info-card {
  background: #FFFFFF;
  border-radius: 12px;
  padding: 16px 20px;
  margin-top: 12px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
}
.row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  padding: 6px 0;
  gap: 16px;
}
.label {
  color: var(--color-text-secondary);
  font-size: 14px;
  flex-shrink: 0;
}
.value {
  color: var(--color-text-primary);
  font-size: 15px;
  text-align: right;
  word-break: break-all;
}
</style>
```

### 6.3 `AlertBox.vue`

```vue
<template>
  <div class="alert" :class="type" role="alert">
    <span class="icon">⚠️</span>
    <span class="text">{{ text }}</span>
  </div>
</template>

<script setup lang="ts">
defineProps<{
  type: 'warning' | 'danger'
  text: string
}>()
</script>

<style scoped>
.alert {
  border-radius: 8px;
  padding: 12px 14px;
  margin-top: 12px;
  display: flex;
  align-items: flex-start;
  font-size: 14px;
  line-height: 1.5;
  border-left: 3px solid;
}
.alert.warning {
  background: var(--status-warning-bg);
  color: var(--status-warning-fg);
  border-left-color: var(--status-warning-fg);
}
.alert.danger {
  background: var(--status-deprecated-bg);
  color: var(--status-deprecated-fg);
  border-left-color: var(--status-deprecated-fg);
  font-weight: 500;
}
.alert .icon {
  margin-right: 8px;
  flex-shrink: 0;
}
</style>
```

### 6.4 `ErrorState.vue`

```vue
<template>
  <div class="error-state">
    <span class="icon">{{ type === 'invalid' ? '❌' : '⚠️' }}</span>
    <h2 class="title">{{ title }}</h2>
    <p class="body">{{ body }}</p>
    <button
      v-if="type === 'network'"
      class="retry-btn"
      @click="$emit('retry')"
    >
      {{ t('error.retry') }}
    </button>
  </div>
</template>

<script setup lang="ts">
import { useI18n } from '@/i18n'
const props = defineProps<{ type: 'invalid' | 'network' }>()
const { t } = useI18n()
const title = computed(() =>
  props.type === 'invalid' ? t('error.invalidQrTitle') : t('error.networkTitle')
)
const body = computed(() =>
  props.type === 'invalid' ? t('error.invalidQrBody') : t('error.networkBody')
)
</script>
```

---

## 7. API 封装 `api/public-status.ts`

```typescript
export interface PublicStatusResponse {
  code: number
  msg: string
  data: StatusData | null
}

const API_BASE = import.meta.env.VITE_API_BASE || 'https://api.xxx.com'

export async function getPublicStatus(
  publicToken: string,
  signal?: AbortSignal
): Promise<PublicStatusResponse> {
  const response = await fetch(
    `${API_BASE}/public/drawing/status/${encodeURIComponent(publicToken)}`,
    {
      method: 'GET',
      headers: {
        'Accept-Language': navigator.language
      },
      signal,
      credentials: 'omit'  // 不携带任何 Cookie
    }
  )
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`)
  }
  return response.json()
}
```

**关键点**:
- `credentials: 'omit'` — 不发送 Cookie（合规要求）
- `AbortController` — 10s 超时
- 不引入 Axios（减少 bundle 体积）

---

## 8. 国际化 `i18n/`

### 8.1 语言检测

```typescript
// i18n/index.ts
import en from './en'
import zh from './zh'

function detectLang(): 'en' | 'zh' {
  const acceptLang = navigator.language || 'en'
  return acceptLang.startsWith('zh') ? 'zh' : 'en'
}

const locale = detectLang()
const messages = locale === 'zh' ? zh : en

export function t(key: string, params?: Record<string, string | number>): string {
  let text = messages[key] || key
  if (params) {
    Object.entries(params).forEach(([k, v]) => {
      text = text.replace(`{${k}}`, String(v))
    })
  }
  return text
}

export function useI18n() {
  return { t, locale }
}
```

### 8.2 语言包

**`i18n/en.ts`** 与 **`i18n/zh.ts`** 按 [UI § 7 国际化对照表](../../ui/h5/UI-REQ-006-h5.md) 维护（约 20 个 key）。

### 8.3 Category 翻译

```typescript
const CATEGORY_MAP: Record<string, { en: string; zh: string }> = {
  ARCHITECTURAL: { en: 'Architectural', zh: '建筑' },
  STRUCTURAL:    { en: 'Structural',    zh: '结构' },
  MECHANICAL:    { en: 'Mechanical',    zh: '机电' },
  ELECTRICAL:    { en: 'Electrical',    zh: '电气' },
  PLUMBING:      { en: 'Plumbing',      zh: '给排水' },
  CIVIL:         { en: 'Civil',         zh: '土建' },
  OTHER:         { en: 'Other',         zh: '其他' }
}

export function translateCategory(category?: string): string {
  if (!category) return '—'
  // 后端可能直接返回已翻译的显示值或枚举 key，两种都兼容
  const upperKey = category.toUpperCase()
  const entry = CATEGORY_MAP[upperKey]
  if (entry) return entry[locale]
  return category  // 后端已返回显示值时直接透传
}
```

---

## 9. 设计令牌 `styles/tokens.css`

```css
:root {
  --color-primary: #409EFF;
  --color-text-primary: #303133;
  --color-text-regular: #606266;
  --color-text-secondary: #909399;
  --color-border-light: #EBEEF5;

  --bg-app: #F5F7FA;

  --status-active-bg: #E8F5E9;
  --status-active-fg: #2E7D32;
  --status-deprecated-bg: #FFEBEE;
  --status-deprecated-fg: #C62828;
  --status-warning-bg: #FFF3E0;
  --status-warning-fg: #EF6C00;

  --radius-card: 12px;
  --radius-alert: 8px;
  --shadow-card: 0 1px 4px rgba(0, 0, 0, 0.06);
}

body {
  margin: 0;
  padding: 0;
  background: var(--bg-app);
  color: var(--color-text-primary);
  font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', 'Helvetica Neue', sans-serif;
  font-size: 15px;
  line-height: 1.5;
}

.page {
  max-width: 480px;
  margin: 0 auto;
  padding: 16px 16px 24px;
  padding-bottom: calc(24px + env(safe-area-inset-bottom));
}

.page-title {
  text-align: center;
  font-size: 18px;
  font-weight: 600;
  color: var(--color-text-primary);
  margin: 16px 0 24px;
}

.divider {
  display: flex;
  align-items: center;
  gap: 12px;
  margin: 20px 0 16px;
  font-size: 13px;
  color: var(--color-text-secondary);
}
.divider::before,
.divider::after {
  content: '';
  flex: 1;
  height: 1px;
  background: var(--color-border-light);
}

.default-hint {
  font-size: 14px;
  color: var(--color-text-regular);
  margin-top: 16px;
}
```

---

## 10. 日期格式化 `utils/format.ts`

```typescript
export function formatDate(iso?: string): string {
  if (!iso) return '—'
  const date = new Date(iso)
  if (isNaN(date.getTime())) return '—'
  const y = date.getFullYear()
  const m = String(date.getMonth() + 1).padStart(2, '0')
  const d = String(date.getDate()).padStart(2, '0')
  return `${y}-${m}-${d}`
}
```

> 仅显示日期，不显示时间（符合 UI 规范）。

---

## 11. 性能与体积

| 项 | 目标 |
|----|------|
| JS bundle gzip | < 80 KB |
| 首屏渲染 | < 1.5s (4G 网络) |
| 骨架屏 | 请求 300ms 内不显示（避免闪烁） |
| 图片资源 | 无（纯文字页面，图标使用 Emoji） |
| CDN | 静态资源上 CDN，缓存 1 年 |
| SSR | 无需 SSR（扫码即访问，SEO 无意义） |

---

## 12. 安全与隐私

| 项 | 实现 |
|----|------|
| 无 Cookie | `fetch` 使用 `credentials: 'omit'` |
| 无 LocalStorage | 不写入任何存储 |
| 无第三方脚本 | `index.html` 仅包含自家资源；禁止引入 Google Analytics、Sentry、客服 SDK |
| CSP | 配置严格 CSP：`default-src 'self'; connect-src 'self' {API_BASE};` |
| XSS 防护 | Vue 模板自动转义；不使用 `v-html` |
| 错误上报 | 后端统一返回 1003006001，不暴露内部状态；前端日志仅在开发环境输出 |

---

## 13. 构建与部署

### 13.1 环境变量

```
VITE_API_BASE=https://api.xxx.com
VITE_PUBLIC_PATH=/public/
```

### 13.2 构建命令

```bash
pnpm build   # 产出 dist/
```

### 13.3 部署

- 静态资源部署到 CDN / 对象存储，路径 `/public/`
- Nginx 配置 `try_files $uri $uri/ /public/index.html` 支持前端路由 fallback
- HTTPS 必须（部分浏览器禁止 HTTP 页面访问相机扫码链接）

---

## 14. 验收标准

### ACTIVE 状态
- [ ] 绿色徽章 "ACTIVE" 正确展示
- [ ] `activeMarkupCount > 0` 时徽章附加橙色 `· {n} Active Markups`
- [ ] `activeMarkupCount = 0` 时徽章仅显示 "ACTIVE"
- [ ] 图纸信息卡正确展示 5 个字段
- [ ] Category 按语言正确翻译
- [ ] 无 Markup 时显示默认文案，有 Markup 时显示橙色警示块

### DEPRECATED 状态
- [ ] 红色徽章 "DEPRECATED" 正确展示
- [ ] 徽章不显示任何附加信息
- [ ] 已扫版本信息卡正确展示被扫版本
- [ ] "Superseded by" 分隔线居中展示
- [ ] 最新版本卡仅显示 Version + Approved At
- [ ] 红色警示文案正确展示

### 错误处理
- [ ] 1003006001 错误码展示无效 QR 错误页
- [ ] 错误页不暴露任何图纸信息
- [ ] 网络超时（>10s）展示网络错误页 + Retry 按钮
- [ ] Retry 点击后重新发起请求

### 国际化
- [ ] 浏览器 `Accept-Language: zh-CN` 时全页中文
- [ ] 其他语言时全页英文
- [ ] `Accept-Language` Header 传给后端接口

### 隐私合规
- [ ] `fetch` 请求 `credentials: 'omit'`，不携带 Cookie
- [ ] 无任何 LocalStorage / SessionStorage 写入
- [ ] 无第三方脚本加载
- [ ] CSP 策略生效

### 响应式
- [ ] 移动端 ≤ 480px 容器占满
- [ ] 桌面端 ≥ 480px 容器居中 480px
- [ ] iOS 安全区正确适配
- [ ] 横屏布局不变形

### 性能
- [ ] JS bundle gzip ≤ 80 KB
- [ ] 4G 网络首屏 < 1.5s
- [ ] 骨架屏在请求 > 300ms 时才显示（避免闪烁）

### 无障碍
- [ ] 徽章使用 `role="status"`，警示使用 `role="alert"`
- [ ] 信息卡使用 `<dl>/<dt>/<dd>` 语义化
- [ ] Retry 按钮 Tab + Enter 可触发
- [ ] `prefers-reduced-motion: reduce` 关闭骨架动画
