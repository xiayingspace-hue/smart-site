# 前端开发说明生成规则(Frontend Rules)

> **本文件定义如何从需求/契约/UI 文档生成 `frontend-spec.md`**。
>
> 前端 agent 的工作:把 UI spec 翻译成"组件结构 + 状态分层 + API 调用计划",并明确性能/弱网/埋点策略。

继承:`global-rules.md`、`traceability-rules.md`

---

## 1. 输入契约

| 输入 | 用于哪一节 |
|-----|---------|
| `ui-spec.md §3` 页面/组件清单 | §3 路由设计、§4 组件结构 |
| `ui-spec.md §3` 各页面权限可见性 | §3 路由权限守卫 |
| `ui-spec.md §3` 状态(5 态) | §4 组件实现要点、§9 错误边界 |
| `ui-spec.md §11` AC 覆盖表 | §13 AC 覆盖检查表(继承) |
| `data-contract.md §4` API 接口 | §6 API 接入 |
| `data-contract.md §4.2` 错误码 | §6.3 错误处理映射 |
| `data-contract.md §1` 实体字段 | §6 类型生成依据 |
| `data-contract.md §7` constants | 引用,不重复 |
| `requirement.md §9.1` 性能 | §7 性能预算 |
| `requirement.md §9.4` 兼容性 | §7、§8 弱网策略 |
| `requirement.md §9.5` 可观测性 | §10 埋点设计 |
| `requirement.md §14` 成功指标 | §10 埋点反推 |

---

## 2. 输出章节

| 章节 | 必填? |
|-----|------|
| 0 溯源块 | ✅ |
| 1 功能概述 | ✅ |
| 2 技术栈(继承 + 新增) | ✅ |
| 3 路由设计 | ✅ |
| 4 组件结构 | ✅ |
| 5 状态管理 | ✅ |
| 6 API 接入 | ✅ |
| 7 性能预算 | ✅ |
| 8 弱网与离线策略 | ⭕(若兼容性涉及移动端则必填) |
| 9 错误边界与降级 | ✅ |
| 10 埋点设计 | ✅ |
| 11 国际化 | ⭕ |
| 12 测试要求 | ✅ |
| 13 AC 覆盖检查表 | ✅ |
| 14 与后端联调约定 | ✅ |
| 15 验收条件 | ✅ |
| 16 变更历史 | ✅ |

---

## 3. 转换规则

### 3.1 路由设计(§3)

对 `ui-spec.md §3` 每个页面,生成路由项:

| ui-spec 页面类型 | 路由模式 |
|---------------|---------|
| 列表页 | `/{module}/{resource}` |
| 详情页 | `/{module}/{resource}/:id` |
| 创建页 | `/{module}/{resource}/new` |
| 编辑页 | `/{module}/{resource}/:id/edit` |
| 子资源 | `/{module}/{parent}/:pid/{child}` |
| 浮层(Modal) | 不单独路由,在父路由 query 加 `?modal=xxx` |
| 多 Tab | 用 query `?tab=xxx`,不切路由 |

**权限守卫**:

| ui-spec 中的权限 | frontend-spec 路由处理 |
|--------------|--------------------|
| 角色对该页有访问权 | 路由守卫校验 user.role |
| 角色无访问权(隐藏) | 路由不暴露入口 + 守卫拦截 |
| 资源所有权 | 进入页面后再校验,不在路由层 |

**URL state 同步**(强制):
- 列表页筛选条件 → query string
- 分页页码 → query string
- 当前 Tab → query string
- 不写入 URL 的:Modal 内部表单数据、滚动位置

### 3.2 组件结构(§4)

#### 3.2.1 组件树推导

从 ui-spec.md 每个页面的"布局"描述,生成组件树。**职责划分原则**:

- 数据展示组件不调用 API(由父级或 hook 提供数据)
- 表单组件不直接 submit(交由父级处理 mutation)
- 业务逻辑放 hook(`use{Domain}Logic`),组件只做渲染

#### 3.2.2 每个新建组件必填字段

```markdown
#### `<ComponentName>`

**关联 AC**: AC-XXX-XXX
**Props**: <TS 接口>
**职责**: <做什么>
**不该做**: <边界,防止职责膨胀>
```

#### 3.2.3 命名规范

| 类型 | 命名 | 例 |
|-----|------|---|
| 页面组件 | `{Resource}{Action}Page` | `OrderListPage` |
| 容器组件 | `{Resource}{Subject}` | `OrderTable` |
| 业务 hook | `use{Resource}{Action}` | `useOrderUpload` |
| 工具函数 | 动词开头 | `formatOrderStatus` |

### 3.3 状态分层(§5)

**强制规则**:每条状态必须显式归类,不允许"看心情放 useState 还是 store"。

| 状态归类决策 |
|------------|
| 来自服务端、需要缓存/失效 → React Query / SWR |
| 来自服务端、写完即弃(如 mutation 结果) → mutation state |
| 跨页面/跨组件共享、非服务端 → 全局 store(Zustand 等) |
| 影响 URL 可分享性 → URL query string |
| 表单字段 → React Hook Form / VeeValidate |
| 仅当前组件内、生命周期与组件一致 → useState |

**常见错误**:
- ❌ 把服务端数据塞进全局 store(导致缓存失效混乱)
- ❌ 把 URL 该承载的筛选条件放本地 state(刷新丢失)
- ❌ 把表单临时输入放全局 store(导致跨页面污染)

#### 3.3.1 缓存 Key 工厂(强制)

```ts
export const {domain}Keys = {
  all: ['{domain}'] as const,
  lists: () => [...{domain}Keys.all, 'list'] as const,
  list: (filters: Filters) => [...{domain}Keys.lists(), filters] as const,
  details: () => [...{domain}Keys.all, 'detail'] as const,
  detail: (id: string) => [...{domain}Keys.details(), id] as const,
};
```

#### 3.3.2 缓存失效策略

| mutation | 失效 |
|----------|-----|
| 创建资源 | `lists()`(整个列表族) |
| 更新资源 | `detail(id)` + `lists()` |
| 删除资源 | `detail(id)` + `lists()` |
| 状态机动作 | `detail(id)` + 必要时 `lists()` |

### 3.4 API 接入(§6)

#### 3.4.1 类型生成(强制)

- **不在前端手写 API 类型**,从 OpenAPI / data-contract 自动生成
- 工具:`openapi-typescript` 或同类
- 位置:`src/api/generated/`
- CI 校验:类型与 data-contract 不一致 → 编译失败

#### 3.4.2 错误码 → 用户提示映射

对 `data-contract.md §4.2` 每个错误码,生成用户可见文案:

| 错误码类型 | 映射规则 |
|----------|---------|
| 4xx 业务错误 | 具体文案,告诉用户怎么办 |
| 401 | 全局拦截,跳登录页(不显示给用户) |
| 403 | 浮层提示 + 返回 |
| 404 | 空状态页 |
| 409 冲突 | 模态对话框 + 选项 |
| 429 | toast"操作太频繁" |
| 5xx | toast"服务暂时不可用" + 重试 |
| 网络错误(无响应) | 顶部 banner |

文案要求引用 ui-spec.md §9 的文案规范。

### 3.5 性能预算(§7)

#### 3.5.1 默认指标(若需求未指定)

| 指标 | 默认目标 |
|-----|---------|
| FCP | < 1.5s (P95) |
| LCP | < 2.5s (P95) |
| TTI | < 3.5s (P95) |
| 单 chunk gzip | < 200KB |
| 总 JS gzip | < 1MB |
| 列表渲染 1000 行 | < 500ms |

需求 §9.1 有更严格的指标 → 覆盖默认值。

#### 3.5.2 优化手段决策

| 触发条件 | 优化措施 |
|--------|---------|
| 路由数 > 3 | 路由级 code splitting |
| 列表可能 > 100 行 | 虚拟滚动 |
| 大依赖(> 100KB) | dynamic import |
| 高频更新组件 | useMemo / useCallback / React.memo,谨慎使用 |
| 高频接口 | React Query staleTime ≥ 30s |
| 移动端 | 图片懒加载 + 响应式图片 |

### 3.6 弱网与离线(§8)

**仅当 requirement §9.4 标注移动端或弱网场景时输出**。否则填"不适用"。

| 场景 | 策略 |
|-----|------|
| 请求超时(默认 30s) | 抛超时 + 用户可重试 |
| 上传中断 | 分片续传(若 data-contract 支持) |
| 接口失败但有缓存 | 显示缓存数据 + 顶部"网络较慢" |
| 完全离线 | 全屏提示,引导切到已缓存功能 |

### 3.7 错误边界(§9)

强制规则:
- 路由级 ErrorBoundary
- 关键功能(如文件上传、表单提交)各自 ErrorBoundary
- 错误必须上报(Sentry 或同类)
- 用户必须看到"出错了 + 重试"提示,不允许静默失败

### 3.8 埋点设计(§10)

#### 3.8.1 埋点最小集(每个 Story 至少)

| 事件类型 | 触发 |
|--------|-----|
| `{action}.start` | 用户开始操作 |
| `{action}.success` | 操作成功 |
| `{action}.fail` | 操作失败(必带 error_code) |
| `{page}.view` | 页面打开 |

#### 3.8.2 反推规则

对 `requirement.md §14` 每个成功指标,反推埋点:

| 指标类型 | 推断埋点 |
|---------|---------|
| 转化率(X 到 Y) | X 页 view + Y 操作 success,各 1 个 |
| 操作时长 | start + success/fail,带 duration_ms |
| 失败率 | fail 事件 + error_code 维度 |
| DAU | 关键页 view |

#### 3.8.3 埋点字段规范

```ts
{
  event_name: string,           // 'order.upload.success'
  occurred_at: string,           // ISO 8601
  user_id: string,
  session_id: string,
  // 业务属性,每个事件 ≤ 10 个,扁平结构
}
```

### 3.9 测试要求(§12)

> 团队当前未强制要求单元测试，以下为建议项，由开发自行评估引入。

| 层级 | 建议程度 | 说明 |
|-----|---------|------|
| 单元 | 建议（非强制） | 工具函数、复杂 hooks、表单校验逻辑优先考虑 |
| 集成 | 建议（非强制） | 含 API 调用的关键交互流程 |
| E2E | 建议（非强制） | 由 QA 主导，覆盖核心业务流程 |

---

## 4. AC 覆盖反查(§13)

对 `ui-spec.md §11` 中标记 ✅ 的 AC,在 frontend-spec 中找到具体实现位置:

```markdown
| AC ID | 实现位置 | 测试覆盖 | 状态 |
|------|---------|---------|------|
| AC-XXX-001 | `<UploadComponent>` + `useUpload` hook | upload.test.tsx | TODO |
| AC-XXX-007 | (后端逻辑,无前端体现) | - | N/A |
```

---

## 5. 降级与待定

| 情况 | 处理 |
|-----|------|
| 技术栈未指定 | 用团队公认默认,标注"待 TL 确认" |
| 性能指标未指定 | 用 §3.5.1 默认,标注"待评审" |
| 埋点指标未指定 | 用 §3.8.1 最小集,在 OQ 中提示 PM 补充成功指标 |
| 国际化要求未指定 | 默认仅 zh-CN,文案中预留 i18n key |
| 兼容性未指定 | Chrome/Edge/Safari 最新两版,标注"待评审" |

---

## 6. 校验清单(自检)

- [ ] 每个路由都有权限处理说明
- [ ] 每个新建组件有 Props/职责/不该做
- [ ] 每条状态都明确归类(§5 决策树)
- [ ] 缓存 key 工厂已定义
- [ ] 每个 mutation API 都有失效 key
- [ ] 每个 data-contract 错误码都映射到用户提示
- [ ] 性能预算每项都有数值(默认或需求指定)
- [ ] 每个 §14 成功指标都有对应埋点
- [ ] AC 覆盖表全部处理
- [ ] 没有重复定义 API 字段类型(必须从 contract 生成)

---

## 7. 不要做的事

- ❌ 在 frontend-spec 里手写 API 字段类型(必须从 contract 生成)
- ❌ 把服务端数据放全局 store
- ❌ 用 React Context 做服务端缓存
- ❌ 跳过 URL state 同步(列表筛选不写 URL → 刷新丢失)
- ❌ 在组件内直接 fetch(必须走统一 client)
- ❌ 静默失败(任何失败必须有用户反馈)
- ❌ 把性能优化前置(过早 useMemo / memo,先写正确再测瓶颈)
- ❌ 给所有组件加 ErrorBoundary(粒度过细)
- ❌ 埋点字段嵌套(扁平化,便于分析)
