# 模板使用说明(README)

本目录包含一套**面向 agent 自动化的需求与设计文档模板**,服务于 PM → UI/前端/后端/QA 的全链路。

---

## 目录文件

| 文件 | 角色 | 说明 |
|-----|------|------|
| `requirement.md` | PM(主) | 需求文档,所有下游文档的源头 |
| `data-contract.md` | 后端架构师 / TL | **单一事实源**:数据模型、API 契约、状态机 |
| `user-stories.md` | PM + TL | 把模块需求拆分为可独立交付的故事 |
| `glossary.md` | 全员 | 术语表,统一项目内的业务词汇 |
| `ui-spec.md` | UI 设计师 | UI 设计稿生成依据 |
| `frontend-spec.md` | 前端开发 | 前端实现依据 |
| `backend-spec.md` | 后端开发 | 后端实现依据 |
| `qa-spec.md` | QA | 测试计划与用例依据 |

---

## 文档依赖关系

```
requirement.md ──┬─► user-stories.md
                 ├─► data-contract.md ──┬─► frontend-spec.md
                 │                      ├─► backend-spec.md
                 │                      └─► qa-spec.md
                 ├─► ui-spec.md ────────► frontend-spec.md
                 └─► glossary.md (全员引用)

下游文档变更不能反向影响 requirement.md。
data-contract.md 是前端/后端/QA 三方协调的枢纽。
```

---

## 推荐的项目目录结构

基于现有目录,建议如下扩展:

```
mcc-requirements/
├── requirements/              # PM 写的需求(每个需求一个子目录)
│   └── REQ-XXXX-[名称]/
│       ├── requirement.md
│       ├── user-stories.md
│       └── data-contract.md
├── templates/                 # 本目录(模板)
├── outputs/                   # 各角色 agent 生成的下游文档
│   └── REQ-XXXX-[名称]/
│       ├── ui-spec.md
│       ├── frontend-spec.md
│       ├── backend-spec.md
│       └── qa-spec.md
├── glossary.md                # 全局术语表(项目级,所有需求共享)
├── rules/                     # 转换规则配置
├── schemas/                   # 文档结构 schema(可选,用于校验)
├── examples/                  # 真实业务示例(参考)
└── scripts/
    ├── validate.py            # 校验文档结构、AC 追溯
    └── trace.py               # 输出 AC 覆盖率报告
```

---

## 关键机制

### 1. YAML Front Matter

每份文档顶部都有结构化元数据,机器可解析。**不要删,不要改字段名**。

```yaml
---
doc_type: requirement
req_id: REQ-XXXX
version: 0.1.0
status: draft
generated_from: requirement.md@0.1.0
---
```

### 2. AC ID 追溯系统

- PM 在 `requirement.md` §8 写 `AC-{REQ_ID}-{NNN}` 格式的验收标准
- 下游所有文档(UI/前端/后端/QA)在末尾的"AC 覆盖检查表"列出 AC → 对应实现位置
- QA 的每个 TC 必须显式 `covers_ac: [AC-XXX]`
- 代码注释中标注 `// AC-XXX`
- CI 中可跑校验:**每个 AC 都被覆盖、每个引用都有效**

### 3. 溯源块(Traceability Block)

下游文档头部都有 §0 溯源块,声明:

- 来源需求文档 + 版本号
- 覆盖的 Story / AC
- 上次同步时间

需求变更时可机械定位受影响的下游章节。

### 4. Open Questions(待定项)

- PM 在 §15 显式列出想不清楚的问题
- 下游 agent 看到 OQ 标记**不要编造**,在对应章节生成 `<!-- TODO: 等待 OQ-XXX 解决 -->`

### 5. 单一事实源(SSoT)

- **数据模型与 API 字段**:只在 `data-contract.md` 定义,其他文档只能引用
- **业务术语**:只在 `glossary.md` 定义
- **验收标准**:只在 `requirement.md` 定义,其他文档只能引用 ID

---

## 使用流程

### Step 1:PM 写需求

1. 复制 `requirement.md` 到 `requirements/REQ-XXXX-xxx/`
2. 填字段。**遇到不确定的事写到 §15 Open Questions,不要瞎填**
3. 同步更新 `glossary.md`(如有新术语)
4. 与 TL 共同产出 `data-contract.md`(API 设计)
5. 与 TL 共同拆分 `user-stories.md`

### Step 2:并行启动下游 agent

把 `requirement.md + data-contract.md + glossary.md` 喂给:

- UI agent → 产出 `ui-spec.md`
- 前端 agent → 产出 `frontend-spec.md`
- 后端 agent → 产出 `backend-spec.md`
- QA agent → 产出 `qa-spec.md`

下游 agent 按规则文件(`rules/`)转换。

### Step 3:校验

跑校验脚本:

- AC 全部被覆盖
- 字段引用有效(没人偷偷自己定义)
- 术语一致
- 溯源块版本号同步

### Step 4:执行落地

- UI 设计师按 `ui-spec.md` 出 Figma
- 开发按 `frontend-spec.md` / `backend-spec.md` 编码
- QA 按 `qa-spec.md` 写测试代码

---

## 版本号约定

- 语义化版本 `MAJOR.MINOR.PATCH`
- **MAJOR**:破坏性变更(字段删除、必填变化、API 路径变更)→ 必须通知所有下游
- **MINOR**:新增字段、新增 AC、新增功能
- **PATCH**:错别字、文案调整

---

## 反模式(不要这样做)

❌ 在 `frontend-spec.md` 重新定义 API 字段(应引用 `data-contract.md`)
❌ 把 PM 的待定问题"自动填上一个看起来合理的方案"
❌ 改下游文档时不更新溯源块的版本号
❌ 写"如登录功能"这种空泛 AC,不写 Given/When/Then
❌ 用同义词漂移术语
❌ 不给 TC 关联 AC ID

---

## FAQ

**Q: 我的需求很简单,需要写这么多吗?**
A: 8 份模板每份都按需填。简单需求 `data-contract.md` 可能就 1 个 API,`user-stories.md` 可能就 1 个 Story,但**结构必须保留**,因为下游 agent 按结构解析。

**Q: 下游 agent 跑出来的文档质量不达标怎么办?**
A: 大概率是上游 `requirement.md` 信息密度不够。先检查上游有没有 OQ、有没有遗漏字段,而不是反复 prompt 下游 agent。

**Q: 需求变更后下游文档怎么同步?**
A: 修改 `requirement.md` → 提升 version → 各下游 agent 重新生成或增量更新对应章节 → 更新各文档的溯源块。不要手改下游文档。

**Q: 数据契约谁来写?PM 还是后端?**
A: 建议 PM + 后端 TL 共同产出。PM 提业务字段,TL 决定技术细节(类型、索引、约束)。
