# 全局转换规则(Global Rules)

> **本文档定义所有下游 agent(UI / 前端 / 后端 / QA / data-contract / user-stories / glossary)的共享规则**。
>
> 各角色 rules 文件不应重复定义本文档已覆盖的内容,只在角色专属部分扩展。

---

## 1. 适用范围

所有 doc_type 不是 `requirement` 的 agent,在执行转换前必须先加载本文件。

---

## 2. 输入契约(所有 agent 共用)

### 2.1 必读输入

每个下游 agent **必须**按以下顺序读取输入,缺一不可:

| 优先级 | 文件 | 用途 | 必需? |
|------|------|------|------|
| 1 | `requirement.md` | 需求源头 | ✅ 必须 |
| 2 | `glossary.md` | 术语对齐 | ✅ 必须 |
| 3 | `data-contract.md` | 接口与数据 | 🔶 除 data-contract agent 自身外必须 |
| 4 | `user-stories.md` | Story 拆分 | 🔶 推荐 |
| 5 | 其他下游文档 | 跨文档引用 | 按需 |

### 2.2 输入校验

读取输入后,agent **必须**先校验以下条件,任一不通过则**停止生成并报告**,不允许带病生成:

- [ ] requirement.md 的 YAML front matter 字段齐全(req_id / version / status)
- [ ] requirement.md status ≠ `draft`(若是 draft,警告但允许继续)
- [ ] requirement.md 的所有 AC 都有唯一 ID,符合 `AC-{REQ_ID}-{NNN}` 格式
- [ ] 引用的术语在 glossary.md 中存在
- [ ] 引用的实体/接口在 data-contract.md 中存在(若已生成)

---

## 3. 输出契约(所有 agent 共用)

### 3.1 文档头部 YAML Front Matter

每份生成的文档**必须**包含以下字段:

```yaml
---
doc_type: [ui_spec | frontend_spec | backend_spec | qa_spec | data_contract | user_stories | glossary]
req_id: REQ-XXXX                    # 来自上游
version: 0.1.0                      # 初次生成统一为 0.1.0
status: draft                       # 初次生成统一为 draft
generated_from: requirement.md@{version}  # 必填,标注来源版本
generated_at: YYYY-MM-DD            # 必填,生成日期
generator: agent_id_or_name         # 必填,标注由哪个 agent 生成
owner: ""                           # 留空,等待人类填写
---
```

### 3.2 §0 溯源块

每份文档第一节(§0)必须是溯源块,格式:

```markdown
## 0. 溯源块(Traceability)

| 项 | 值 |
|---|---|
| 来源需求 | REQ-XXXX @ v{X.Y.Z} |
| 数据契约 | data-contract.md @ v{X.Y.Z}(若适用) |
| 覆盖 Story | US-XXX, US-YYY |
| 覆盖 AC | AC-XXXX-XXX, AC-XXXX-YYY |
| 上次同步时间 | YYYY-MM-DD |
```

### 3.3 §末 AC 覆盖检查表

每份文档**最后一节之前**必须有 AC 覆盖检查表,声明本文档覆盖了哪些 AC、对应实现/测试/设计在哪。

### 3.4 §末 变更历史

每份文档**最后一节**必须是变更历史表。

---

## 4. ID 命名规范

| 类型 | 格式 | 示例 | 由谁分配 |
|-----|------|------|--------|
| 需求 | `REQ-{4 位数字}` | REQ-0042 | PM |
| 用户故事 | `US-{3 位数字}` | US-001 | PM 或 user-stories agent |
| 验收标准 | `AC-{REQ_ID 后 4 位}-{3 位数字}` | AC-0042-001 | PM |
| 测试用例 | `TC-{AC_ID 后部}-{2 位数字}` | TC-0042-001-01 | QA agent |
| 接口 | `API-{3 位数字}` | API-001 | data-contract agent |
| 实体 | `ENT-{3 位数字}` | ENT-001 | PM 或 data-contract agent |
| 状态 | `S-{大写英文}` | S-PUBLISHED | PM |
| 状态转换 | `T-{3 位数字}` | T-001 | data-contract agent |
| 待定问题 | `OQ-{3 位数字}` | OQ-001 | PM 或任意 agent |
| 角色 | `ROLE-{3 位数字}` | ROLE-001 | PM |
| 异步任务 | `TASK-{3 位数字}` | TASK-001 | data-contract agent |
| 测试场景 | `SC-{3 位数字}` 或 `SC-E{2 位}` 或 `SC-P{2 位}` | SC-001 / SC-E01 / SC-P01 | QA agent |
| 压测场景 | `PT-{3 位数字}` | PT-001 | QA agent |

**规则**:
- ID 一旦分配,**永不复用**(即使删除了对应内容)
- ID 在所属命名空间内必须唯一
- 跨文档引用 ID 时**禁止改写格式**(不能写 `Ac-0042-1` 或 `AC0042001`)

---

## 5. 信息缺失处理(降级策略)

> ⚠️ **这是 agent 最容易出错的地方**。下面的规则**不可妥协**。

### 5.1 三类缺失场景

| 场景 | 表现 | 处理 |
|-----|------|------|
| **PM 显式标注待定** | 需求文档 §15 列出 OQ-XXX | 在对应章节插入 `<!-- TODO: 等待 OQ-XXX 解决 -->`,不生成内容 |
| **PM 隐式遗漏** | 某章节空白或字段缺失 | 在对应章节插入 `<!-- MISSING: requirement.md §X.Y 未填写,需 PM 补充 -->`,继续生成其他章节 |
| **跨文档引用对象不存在** | 引用的 AC/API/实体在源文档找不到 | **报错并停止**,不允许带错引用继续生成 |

### 5.2 禁止编造

agent **绝对禁止**以下行为:

- ❌ 看到 PM 没写"性能要求",自己填上"P95 < 200ms"
- ❌ 看到需求未明确角色,自己定义"管理员/普通用户"
- ❌ 看到 AC 不完整,自己补充 Given/When/Then
- ❌ 把"从经验推断"的内容写得像"从需求推导"
- ❌ 把 OQ 待定项替换成一个看似合理的方案

### 5.3 允许的合理推导

agent **允许**以下推导,但必须**显式标注来源**:

- ✅ 从枚举值数量推导出"应有对应数量的状态徽章 token"
- ✅ 从 PM 写的"主流程"推导出"流程图节点"
- ✅ 从 AC 中的 Given 推导出"测试前置数据"
- ✅ 从 PM 列的角色矩阵推导出"路由级权限守卫清单"

推导内容必须用以下格式标注:

```markdown
> 💡 **派生**:本节内容由 [agent_name] 从 requirement.md §X.Y 自动派生,
> 派生规则:[简述推导逻辑]。如有偏差请回到源头修正。
```

---

## 6. 单一事实源(SSoT)纪律

| 内容 | SSoT 文件 | 引用方式 |
|-----|---------|---------|
| 数据模型字段 | `data-contract.md` §1 | 引用实体名 + 章节,**不复制字段表** |
| 枚举值 | `data-contract.md` §2 | 引用枚举名 |
| API 字段 | `data-contract.md` §4 | 引用 API ID |
| 状态机 | `data-contract.md` §3 | 引用状态机名 |
| 验收标准 | `requirement.md` §8 | 引用 AC ID |
| 业务术语 | `glossary.md` | 直接使用术语,不重新定义 |
| 角色定义 | `requirement.md` §2.1 | 引用 ROLE ID |
| 用户故事 | `user-stories.md` 或 `requirement.md` §2.2 | 引用 US ID |

**违反 SSoT 的典型错误**:
- ❌ 在 `frontend-spec.md` 写"上传接口字段:file, hash, size..."(应引用 API-XXX)
- ❌ 在 `qa-spec.md` 重新写一遍 AC 描述(应只写 AC ID)
- ❌ 在 `backend-spec.md` 重新定义实体字段(应引用 data-contract §1.X)

---

## 7. 术语一致性

- 所有用户可见文案、字段命名、文档行文,**必须**与 `glossary.md` 一致
- 发现 glossary 缺词时,先在 glossary 增补,再使用
- **禁止**同义词漂移(同一概念用多个名字)

agent 自检方法:生成完成后,扫描文档中的核心名词,对照 glossary 检查。

---

## 8. 版本号约定

- 语义化版本:`MAJOR.MINOR.PATCH`
- **MAJOR**:破坏性变更(字段删除、必填变化、API 路径变更)
- **MINOR**:新增字段、新增 AC、新增功能
- **PATCH**:错别字、文案调整

下游 agent **不主动升级 MAJOR 版本**,只能由人类决策。

---

## 9. 输出格式与风格

### 9.1 Markdown 风格

- 章节编号采用 `1.`、`1.1`、`1.1.1` 三层
- 表格表头加粗自动应用,无需额外加 `**`
- 代码块标注语言(```yaml / ```ts / ```sql / ```pseudo)
- 链接用相对路径(`./data-contract.md`)便于跨文档跳转

### 9.2 注释类型

| 注释 | 用途 | 示例 |
|-----|------|------|
| `<!-- TODO: ... -->` | 待 PM/上游补充 | `<!-- TODO: 等待 OQ-001 -->` |
| `<!-- MISSING: ... -->` | 上游遗漏 | `<!-- MISSING: §9 性能要求未填 -->` |
| `<!-- DERIVED: ... -->` | 自动派生标注 | `<!-- DERIVED: 从需求 §5.2 状态机派生 -->` |
| `<!-- NOTE: ... -->` | 给后续读者的提示 | `<!-- NOTE: 此字段后端校验在 service 层 -->` |

---

## 10. 通用校验清单

agent **生成完成后,必须自检以下项**,任一未通过须修正后重出:

### 10.1 结构校验

- [ ] YAML front matter 字段完整、值合法
- [ ] §0 溯源块存在且字段齐全
- [ ] AC 覆盖检查表存在
- [ ] 变更历史存在
- [ ] 所有章节按模板顺序与编号

### 10.2 引用校验

- [ ] 所有 `AC-XXX` 引用都在 requirement.md 中存在
- [ ] 所有 `API-XXX` 引用都在 data-contract.md 中存在
- [ ] 所有 `ENT-XXX` / `ROLE-XXX` / `US-XXX` 引用都在源文档存在
- [ ] 没有"破链"引用

### 10.3 SSoT 校验

- [ ] 没有重复定义已在 SSoT 文档定义的字段
- [ ] 引用而不复制
- [ ] 术语与 glossary.md 一致

### 10.4 缺失项校验

- [ ] 所有 OQ 都已在文档中以 `<!-- TODO: 等待 OQ-XXX -->` 标注
- [ ] 没有编造 PM 未提供的内容
- [ ] 所有 `<!-- MISSING -->` 标注都准确指向源文档章节

---

## 11. 禁止事项(Anti-patterns)

agent **绝对不允许**:

1. ❌ 修改源文档(requirement.md / glossary.md / data-contract.md)
2. ❌ 删除模板中的章节(可留空,但不能删)
3. ❌ 改变章节顺序与编号
4. ❌ 跳过 AC 覆盖检查表
5. ❌ 在不确定时编造默认值
6. ❌ 用同义词替换 glossary 已定义的术语
7. ❌ 生成超出本角色职责的内容(例如 UI agent 不应输出 SQL)
8. ❌ 输出"建议增加 XXX 章节"这类元评论(应直接修改本 rules 反馈)

---

## 12. Agent 之间的边界

```
┌──────────────────────────────────────────────────┐
│  data-contract-agent                              │
│  职责:实体、API、状态机、错误码、幂等性         │
│  不做:UI、测试用例、业务运营策略                 │
└──────────────────────────────────────────────────┘
            ↓ 提供契约
┌──────────────┬──────────────┬──────────────┐
│  ui-agent    │  frontend    │  backend     │  qa-agent
│  Figma 输入   │  组件/路由   │  服务实现    │  测试矩阵
│  5 态/权限   │  状态分层    │  事务/审计   │  AC 覆盖
└──────────────┴──────────────┴──────────────┘
```

各 agent 守好自己的领域,跨域请求需通过修改源文档而非自行扩展。

---

## 13. 执行流程

每个下游 agent 的执行流程都遵循以下骨架:

```pseudo
function generate_spec(requirement_md):
  # 1. 加载输入(本文件 §2.1)
  inputs = load_required_inputs()

  # 2. 校验输入(本文件 §2.2)
  errors = validate_inputs(inputs)
  if errors: report_and_stop(errors)

  # 3. 加载本角色 rules(role-rules.md)
  rules = load_role_rules()

  # 4. 按转换映射表生成各章节
  for section in template_sections:
    content = apply_transformation(section, inputs, rules)
    if content.has_missing:
      mark_with_todo(content)
    output[section] = content

  # 5. 输出自检(本文件 §10)
  errors = self_check(output)
  if errors: revise(output, errors)

  # 6. 写文件 + 更新溯源块
  write_with_traceability(output)
```

---

## 14. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | YYYY-MM-DD | | 初稿 |
