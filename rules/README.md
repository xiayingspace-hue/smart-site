# 转换规则体系(Rules)

本目录包含**所有 agent 的转换规则**。每份 rules 服务一个 agent,定义它如何从上游文档生成下游文档。

---

## 文件清单

| 文件 | 服务的 Agent | 输出文档 | 类型 |
|-----|-------------|---------|------|
| `global-rules.md` | **所有 agent 共享** | - | 基础规则 |
| `traceability-rules.md` | 校验脚本 + 所有 agent 自检 | - | 追溯规则 |
| `glossary-rules.md` | glossary-agent | `glossary.md` | 角色规则 |
| `user-stories-rules.md` | user-stories-agent | `user-stories.md` | 角色规则 |
| `data-contract-rules.md` | data-contract-agent | `data-contract.md` | 角色规则 |
| `ui-rules.md` | ui-agent | `ui-spec.md` | 角色规则 |
| `frontend-rules.md` | frontend-agent | `frontend-spec.md` | 角色规则 |
| `backend-rules.md` | backend-agent | `backend-spec.md` | 角色规则 |
| `qa-rules.md` | qa-agent | `qa-spec.md` | 角色规则 |

与 `templates/` 的关系:
- `templates/` 定义文档**长什么样**(结构、字段、章节)
- `rules/` 定义文档**怎么生成**(从哪读、怎么转、怎么校验)

---

## 规则文件的统一结构

每份角色规则按以下骨架组织,便于交叉对照:

```
1. 角色定义           - agent 是谁、做什么、不做什么
2. 输入契约           - 必读文件、必读章节、输入校验
3. 输出章节映射表     - 输出每节 ← 输入哪些章节(机器可读)
4. 转换规则详述       - 每节的具体转换逻辑、派生规则、缺失策略
5. 命名约定           - 本 agent 专属的命名规则
6. 校验清单           - 生成完成后的自检项
7. 禁止事项           - 明令禁止的行为(anti-patterns)
8. 与下游协作         - 下游 agent 期望从本输出得到什么
9. 变更历史
```

---

## Agent 之间的依赖图

```
                    ┌─────────────┐
                    │ requirement │ (PM 写)
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
       ┌─────────┐   ┌──────────┐   ┌──────────────┐
       │glossary │   │user-stor.│   │data-contract │
       │ agent   │   │  agent   │   │   agent      │
       └────┬────┘   └────┬─────┘   └──────┬───────┘
            │             │                │
            └─────────────┴────┬───────────┘
                               ↓
              ┌────────┬───────┼────────┬─────────┐
              ↓        ↓       ↓        ↓
          ┌──────┐ ┌──────┐ ┌──────┐ ┌─────┐
          │ ui   │ │front │ │back  │ │ qa  │
          │agent │ │agent │ │agent │ │agent│
          └──────┘ └──────┘ └──────┘ └─────┘

执行顺序:
1. PM 写 requirement.md
2. glossary + user-stories + data-contract 三者基本并行
   (但 data-contract 强烈建议有 TL/架构师评审)
3. ui / frontend / backend / qa 四者基本并行
   (frontend 强依赖 ui-spec 和 data-contract;
    qa 强依赖 data-contract,可滞后 1 拍以拿到 ui/frontend/backend spec)
4. 校验:scripts/validate.py 跑追溯报告
```

---

## 规则的层级关系

```
┌──────────────────────────────────────────────┐
│  global-rules.md                              │
│  - 所有 agent 共享的基础规则                   │
│  - YAML front matter / 溯源块 / ID 规范       │
│  - 缺失策略 / SSoT 纪律 / 输出格式            │
└──────────────────────────────────────────────┘
                    ↑ 继承
┌──────────────────────────────────────────────┐
│  traceability-rules.md                        │
│  - 跨文档追溯规则,所有 agent 自检时遵守       │
│  - AC ↔ TC 双向映射、覆盖率、影响分析          │
└──────────────────────────────────────────────┘
                    ↑ 继承
┌──────────────────────────────────────────────┐
│  {role}-rules.md                              │
│  - 角色专属规则                                │
│  - 输入章节映射、转换函数、校验项              │
└──────────────────────────────────────────────┘
```

下层规则**继承且不得违反**上层规则。冲突时上层优先。

---

## 关键设计原则

### 1. 派生而非创造

规则是机器可执行的,不是人读的指南。每条规则都说清楚:**从哪读 → 怎么转 → 写到哪 → 缺信息怎么办**。

下游 agent 的输出**必须能回溯到上游某项**。规则强制每个产物挂 ID:

- 每个 TC 挂 AC
- 每个 API 挂 AC
- 每个 UI 组件挂 Story 或 AC
- 每个状态转换挂 ac_refs

不能回溯的产物 = 多余产物,删除或登记新 AC。

### 2. 不编造,显式 TODO

agent 看到上游缺信息时:
- PM 显式标 OQ → 标 `<!-- TODO: 等待 OQ-XXX -->`
- 上游遗漏 → 标 `<!-- MISSING: §X.Y 未填 -->`
- 引用不存在 → **报错并停止**

绝不编造默认值。"看起来合理就填"是 bug 之源。

### 3. 单一事实源(SSoT)

| 信息 | 唯一定义位置 |
|-----|-----------|
| 数据字段、API、状态机 | `data-contract.md` |
| 业务术语 | `glossary.md` |
| 验收标准 | `requirement.md` §8 |
| 用户故事 | `user-stories.md` |

下游文档**只引用、不重复**。

### 4. 章节锚定

文档章节结构由 `templates/` 固定,agent 不允许:
- 删除章节
- 重排章节
- 改章节标题

只允许填充章节内容。这是为了让校验脚本能机械定位。

### 5. 角色边界清晰

每份 rules 都有"职责边界"和"不做"。例如:
- ui-agent 不输出代码
- frontend-agent 不定义 API 字段
- backend-agent 不写 UI 描述
- qa-agent 不生成可运行测试代码(那是 test-code-agent 的事)

---

## 项目目录结构(建议)

```
your-project/
├── README.md
├── glossary.md                   # 全局术语表(项目级)
├── requirements/                 # PM 写的需求(每个一个子目录)
│   └── REQ-XXXX-[名称]/
│       ├── requirement.md
│       ├── data-contract.md
│       └── user-stories.md
├── outputs/                      # 各角色 agent 生成的下游文档
│   └── REQ-XXXX-[名称]/
│       ├── ui-spec.md
│       ├── frontend-spec.md
│       ├── backend-spec.md
│       ├── qa-spec.md
│       └── .trace-report.md      # 校验脚本输出
├── templates/                    # 文档模板(8 份)
├── rules/                        # 本目录(9 份规则)
├── schemas/                      # OpenAPI / JSON Schema(可选)
└── scripts/
    ├── validate.py               # 校验文档结构 + AC 追溯
    ├── trace.py                  # 输出 AC 覆盖率报告
    └── generate.py               # 调用各 agent 生成下游文档(可选)
```

---

## 使用方式

### 对 agent 开发者(写 prompt 的人)

把对应 rules 文件作为 system prompt 的一部分,例如 ui-agent 的 prompt 应包含:

```
[system prompt]
你是 ui-agent。请严格遵守:
1. /rules/global-rules.md
2. /rules/traceability-rules.md
3. /rules/ui-rules.md

[user prompt]
请基于以下输入生成 ui-spec.md:
- requirement.md: ...
- data-contract.md: ...
- glossary.md: ...
```

### 对 PM(使用 agent 的人)

不需要读 rules,**只需写好 requirement.md**。规则会保证 agent 按一致方式产出下游文档。

但建议你了解:
- §5 信息缺失处理(知道 agent 会标 TODO 而不是编造)
- §10 通用校验清单(知道 agent 会自检什么)

### 对校验脚本

`scripts/validate.py` 应:
1. 加载所有文档
2. 按 traceability-rules.md 校验追溯链
3. 按 global-rules.md §10 校验文档结构
4. 输出 ERROR/WARNING 报告
5. 在 CI 中作为质量门禁

---

## 反模式(常见错误)

❌ **在 ui-rules 里规定 API 字段格式**
→ 那是 data-contract-rules 的事,不要跨域

❌ **让 agent 自动决定昂贵技术决策**
→ 例如换状态管理库、加新 MQ。应该标 TODO 让人类决策

❌ **rules 里只写"应该"不写"怎么"**
→ "提取设计目标" 不是规则,"从 §1.2 第一句作为设计目标" 才是

❌ **跳过缺失处理,直接给默认值**
→ 例如"PM 没写性能要求,默认 P95 < 200ms"

❌ **不同 rules 重复定义同一规则**
→ 共享规则放 global,不要在角色 rules 里各写一遍

---

## 演进路径

这套 rules 是 v0.1.0 初版。建议演进:

**v0.2:**
- 用真实需求跑一轮,记录 agent 实际产出与预期的差距
- 把发现的"agent 自由发挥"问题补成新规则

**v0.3:**
- 把规则中可形式化的部分(如字段命名、ID 格式)抽成 JSON Schema
- 引入校验脚本(scripts/validate.py)

**v1.0:**
- 规则稳定,有黄金标准示例,有 CI 校验
- 团队所有需求都按这套流程执行

---

## FAQ

**Q: rules 文件本身需要版本控制吗?**
A: 需要。rules 是"对 agent 的合同",改 rules 等于改流程。建议遵循语义化版本,大改时通知所有 agent 重跑。

**Q: 我们团队没有"data-contract"这个文档,可以跳过吗?**
A: 可以,但代价是前后端 QA 各自从需求"理解"接口,联调成本会显著上升。强烈建议至少在涉及前后端协作的需求中引入。

**Q: 如果团队还没准备好做这么严格的追溯,从哪开始?**
A: 优先级从高到低:
1. 先用 `requirement.md` 模板的 AC 编号系统(§8)
2. 让 QA 在每个 TC 中标 covers_ac
3. 加 data-contract,让前后端有共同的字段定义
4. 最后加追溯校验脚本

**Q: agent 执行时如果某条规则做不到怎么办?**
A: 在生成的文档底部加一节"规则偏离记录",显式说明哪条规则跳过了、理由、风险。**不要静默违反**。

---

## 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | YYYY-MM-DD | | 初稿:9 份规则文件 |
