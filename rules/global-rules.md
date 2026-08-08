# 全局转换规则(Global Rules)

> **本文档定义所有下游 agent(UI / 前端 / 后端 / QA / data-contract / user-stories)的共享规则**,
> 并收录全项目的命名与书写约定(§4、§9)。
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
| 2 | `requirements/shared/ROLES-shared.md` | 角色标准名与角色禁用词 | ✅ 必须 |
| 3 | `data-contract.md` | 接口与数据 | 🔶 除 data-contract agent 自身外必须 |
| 4 | `user-stories.md` | Story 拆分 | 🔶 推荐 |
| 5 | `background/project-overview.md` §文档覆盖范围 | 了解本仓库的覆盖边界 | ✅ 必须 |
| 6 | 其他下游文档 | 跨文档引用 | 按需 |

> ⚠️ 第 5 项不可跳过。本仓库创建于项目开发中期,产品已上线的多数模块
> (机械设备、物料、环境、安全、质量、车辆进出场、工地文档、工地反馈、BIM)
> **在 `requirements/` 下没有需求文档**。详见 §11 第 9 条。

### 2.2 输入校验

读取输入后,agent **必须**先校验以下条件,任一不通过则**停止生成并报告**,不允许带病生成:

- [ ] requirement.md 的 YAML front matter 字段齐全(req_id / version / status)
- [ ] requirement.md status ≠ `draft`(若是 draft,警告但允许继续)
- [ ] requirement.md 的所有 AC 都有唯一 ID,符合 `AC-{REQ_ID}-{NNN}` 格式
- [ ] 引用的角色名是 `ROLES-shared.md` §1 的标准名,且未使用其 §2 禁用词
- [ ] 引用的实体/接口在 data-contract.md 中存在(若已生成)

---

## 3. 输出契约(所有 agent 共用)

### 3.1 文档头部 YAML Front Matter

每份生成的文档**必须**包含以下字段:

```yaml
---
doc_type: [ui_spec | frontend_spec | backend_spec | qa_spec | data_contract | user_stories]
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

## 4. 命名规范

### 4.1 常用缩写

| 缩写 | 全称 | 含义 |
|-----|------|------|
| AC | Acceptance Criteria | 验收标准 |
| US | User Story | 用户故事 |
| TC | Test Case | 测试用例 |
| OQ | Open Question | 待定问题 |
| API | Application Programming Interface | 应用编程接口 |
| DTO | Data Transfer Object | 数据传输对象 |
| SSoT | Single Source of Truth | 单一事实源 |

> 项目专属缩写在此续填。

### 4.2 ID 命名

| 类型 | 格式 | 示例 | 由谁分配 |
|-----|------|------|--------|
| 需求 | `REQ-{3 位数字}[子需求字母]` | REQ-003、REQ-003A | PM |
| 用户故事 | `US-{3 位数字}` | US-001 | PM 或 user-stories agent |
| 验收标准 | `AC-{REQ 编号}-{3 位数字}` | AC-003H-001 | PM |
| 测试用例 | `TC-{AC 编号}-{2 位数字}` | TC-003H-001-01 | QA agent |
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
- 跨文档引用 ID 时**禁止改写格式**(不能写 `Ac-003H-1` 或 `AC003H001`)
- `ROLE-XXX` 是**需求文件内的局部编号**,不跨文档通用。跨文档引用角色须使用标准名,
  见 `requirements/shared/ROLES-shared.md` §1

### 4.3 文件命名

> 端标识取值:`shared` / `pc` / `app` / `h5`。多端拆分原则见 §15。

| 类型 | 格式 | 示例 |
|-----|------|------|
| 需求文件 | `REQ-{编号}-{端标识}.md` | `REQ-003A-pc.md`、`REQ-007-shared.md` |
| 数据契约 | `DATA-CONTRACT-REQ-{编号}-{端标识}.md` | `DATA-CONTRACT-REQ-003H-pc.md` |
| UI 说明 | `UI-REQ-{编号}-{端标识}.md` | `UI-REQ-003A-pc.md` |
| 前端说明 | `FRONTEND-REQ-{编号}-{端标识}.md` | `FRONTEND-REQ-003A-pc.md` |
| 后端说明 | `BACKEND-REQ-{编号}.md`(不分端) | `BACKEND-REQ-003A.md` |
| 测试用例 | `QA-REQ-{编号}-{端标识}.md` | `QA-REQ-003A-pc.md` |

> **例外**:不属于任何单个 REQ 的跨端共享文档,采用 `{主题}-shared.md` 命名,
> 如 `requirements/shared/ROLES-shared.md`(全项目角色定义)。

### 4.4 字段命名

- 数据库字段:`snake_case`
- API 字段:`snake_case`(对外契约统一,前端 client 内部转 camelCase)
- 前端组件 prop:`camelCase`
- TS 类型:`PascalCase`

### 4.5 枚举值命名

- 全大写,下划线分隔(如 `IN_PROGRESS`)
- 不要用数字编码
- 枚举的**取值**定义在 data-contract §2,本节只规定命名形式

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
| 验收标准 | `requirement.md` §9 | 引用 AC ID |
| 角色(跨文档) | `requirements/shared/ROLES-shared.md` §1 | 引用标准名,**不用 ROLE-XXX** |
| 角色(需求文件内) | `requirement.md` §4.1 | 局部编号,仅本文件内有效 |
| 命名与书写规范 | 本文件 §4、§9 | 直接遵循,不另立规则 |
| 用户故事 | `user-stories.md` 或 `requirement.md` §4.2 | 引用 US ID |

**违反 SSoT 的典型错误**:
- ❌ 在 `frontend-spec.md` 写"上传接口字段:file, hash, size..."(应引用 API-XXX)
- ❌ 在 `qa-spec.md` 重新写一遍 AC 描述(应只写 AC ID)
- ❌ 在 `backend-spec.md` 重新定义实体字段(应引用 data-contract §1.X)

---

## 7. 术语一致性

- 角色名**必须**取自 `requirements/shared/ROLES-shared.md` §1 标准名
- 字段与枚举命名**必须**遵循本文件 §4.4 / §4.5
- 发现缺少标准角色名时,先在 `ROLES-shared.md` 增补,再使用
- **禁止**同义词漂移(同一概念用多个名字)

agent 自检方法:生成完成后,扫描文档中出现的角色名,对照 `ROLES-shared.md` §1 与 §2 检查。

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

**全角 / 半角**:中文语境下的括号统一用全角 `（）`。

| ❌ 不要用 | ✅ 应该用 |
|---------|---------|
| `Document Controller (DC)` | `Document Controller（DC）` |
| `Site Engineer (SE)` | `Site Engineer（SE）` |

> 当前 `REQ-007-shared` 使用半角,其余文件使用全角。

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
- [ ] 角色名与 `ROLES-shared.md` §1 一致,未命中其 §2 禁用词

### 10.4 缺失项校验

- [ ] 所有 OQ 都已在文档中以 `<!-- TODO: 等待 OQ-XXX -->` 标注
- [ ] 没有编造 PM 未提供的内容
- [ ] 所有 `<!-- MISSING -->` 标注都准确指向源文档章节

---

## 11. 禁止事项(Anti-patterns)

agent **绝对不允许**:

1. ❌ 修改源文档(requirement.md / ROLES-shared.md / data-contract.md)
2. ❌ 删除模板中的章节(可留空,但不能删)
3. ❌ 改变章节顺序与编号
4. ❌ 跳过 AC 覆盖检查表
5. ❌ 在不确定时编造默认值
6. ❌ 用同义词替换 `ROLES-shared.md` 已定义的角色标准名
7. ❌ 生成超出本角色职责的内容(例如 UI agent 不应输出 SQL)
8. ❌ 输出"建议增加 XXX 章节"这类元评论(应直接修改本 rules 反馈)
9. ❌ **因某模块在 `requirements/` 下找不到文档,就推断该功能不存在**

   本仓库只覆盖图纸线与工序进度线,其余已上线模块暂无需求文档
   (见 `background/project-overview.md` §文档覆盖范围)。由此**不得**推导出:

   - "该字段无其他模块引用,可以安全变更/删除"
   - "该接口无其他调用方,可以改签名"
   - "该枚举值无其他使用场景,可以收窄取值"
   - "系统中不存在 XX 功能,因此无需考虑其交互"

   涉及跨模块影响的判断,标 `<!-- TODO: 需确认 XX 模块的使用情况 -->` 交由人类核实,
   不要基于文档缺失做出边界假设。部分已上线功能的设计决策记录在
   `background/key-decisions.md`,可作为线索但不完整。

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

## 14. 项目专属约定

<!-- MISSING: 本节被 README.md 与 background/key-decisions.md 共 8 处引用
     (§14、§14.4、§14.5、§14.10),但内容从未写入本文件。
     按引用处的描述,本节应包含:技术栈约定、枚举、错误码、API 规范、
     设计 Token 语义值、仓库分离方案的触发时机与成本。
     在补齐之前,上述引用均为断链。 -->

---

## 15. 多端拆分原则

> 本项目包含 PC / APP / H5 三端,共享同一套后端服务与业务逻辑。
> 各端技术栈见 `background/tech-stack.md`(技术栈的单一事实源),本节不重复。

### 15.1 目录与端标识

| 端 | 端标识 | 目标用户 |
|----|-------|---------|
| PC 管理端 | `pc` | 系统管理员、项目经理 |
| APP 移动端 | `app` | 工地管理人员(现场) |
| H5 移动端 | `h5` | 工地工人、分包商 |
| 跨端共享 | `shared` | — |

`requirements/` 与 `outputs/` 均按端标识分子目录;`outputs/backend/` 不分端。
文件命名规范见 §4.3。

### 15.2 shared 与各端的职责边界

写入 `shared/`:

- 业务规则与业务流程定义
- API 接口定义(端点、请求/响应格式、错误码)
- 数据模型与字段定义
- 权限与角色定义
- 会话管理逻辑(Token、租户上下文)
- 字段校验规则(不含 UI 表现)

写入各端(`pc/` / `app/` / `h5/`):

- 页面布局与结构
- 视觉样式与设计令牌
- 交互方式与手势
- 端特有功能(如 APP 扫码登录、H5 微信授权、PC 批量操作)
- 端特有的非功能需求(屏幕适配等)

### 15.3 引用而非复制

各端需求文档**引用** shared 文档中的业务规则与 API 定义,不得重复表述：

```markdown
## 业务规则

> 详见 [REQ-001-shared.md](../shared/REQ-001-shared.md) — §3「登录业务规则」
```

agent 在生成下游文档时,若发现某端文档重复定义了 shared 中已有的业务规则,
应标注 `<!-- MISSING: 与 REQ-XXX-shared.md §N 重复定义 -->` 并以 shared 为准。

### 15.4 设计体系分层

跨端设计系统(品牌标识、品牌色、语义色、字体家族、图标库)为三端共同基线;
各端在此之上独立维护自己的设计语言(圆角、触控尺寸、信息密度、导航模式),
**Token 不跨端继承**。

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | YYYY-MM-DD | | 初稿 |
| 0.2.0 | 2026-08-08 | XIA YING | 新增 §15 多端拆分原则(由 MULTI-PLATFORM.md 并入);§14 标记为缺失章节 |
| 0.3.0 | 2026-08-08 | XIA YING | §2.1 将 `background/project-overview.md` §文档覆盖范围 列为必读输入;§11 新增第 9 条,禁止因需求文档缺失而推断功能不存在或系统边界 |
