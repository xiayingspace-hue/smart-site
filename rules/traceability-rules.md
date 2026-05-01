# 追溯规则(Traceability Rules)

> **本文档定义跨文档的引用与追溯规则**。
>
> 服务对象不是某一个 agent,而是**校验脚本**和**所有 agent 的自检环节**。
> 整套体系能否闭环,取决于追溯链是否完整。

---

## 1. 追溯模型

整个体系的追溯链:

```
   PM 视角                       开发/测试视角
┌─────────────┐
│  需求       │  REQ-XXXX
└─────────────┘
      ↓ 拆分
┌─────────────┐
│  用户故事   │  US-XXX
└─────────────┘
      ↓ 派生
┌─────────────┐
│  验收标准   │  AC-XXXX-NNN  ←─────────────────┐
└─────────────┘                                  │
      ↓ 由各下游覆盖                              │
┌────────────────────────────────────────────┐   │
│ UI 设计   │ 前端实现 │ 后端实现 │ QA 用例  │   │ 反向追溯
│ §11 表    │ §13 表   │ §15 表   │ §11 矩阵 │   │
└────────────────────────────────────────────┘   │
      ↓ 引用 AC                                   │
   代码注释 // AC-XXXX-NNN                        │
   测试代码 covers_ac: AC-XXXX-NNN  ──────────────┘
```

**核心原则**:每个 AC 必须从需求一路追溯到代码注释和测试,反之每个测试和代码都能溯源到 AC。

---

## 2. 追溯关系定义

### 2.1 必须追溯的关系(Required)

| 源 | 目标 | 含义 | 检查方法 |
|----|------|------|---------|
| `requirement.md` AC | `qa-spec.md` TC | 每个 AC 至少 1 个 TC 覆盖 | 反向扫描 TC 的 covers_ac |
| `requirement.md` AC | `frontend-spec.md` §13 | UI 类 AC 必须有前端实现位置 | 扫描 §13 表 |
| `requirement.md` AC | `backend-spec.md` §15 | 业务逻辑类 AC 必须有后端实现位置 | 扫描 §15 表 |
| `requirement.md` AC | `ui-spec.md` §11 | 设计类 AC 必须有 UI 元素 | 扫描 §11 表 |
| `data-contract.md` API | `frontend-spec.md` §6.2 | 每个 API 必须有前端调用方 | 扫描 §6.2 表 |
| `data-contract.md` API | `backend-spec.md` 业务逻辑 | 每个 API 必须有后端实现 | 扫描业务点 |
| `data-contract.md` 状态机 transition | `qa-spec.md` SC-T | 每条转换必须有 1 个测试场景 | 扫描状态转换场景 |
| `data-contract.md` forbidden 转换 | `qa-spec.md` SC-T-F | 每条非法转换必须有 1 个测试 | 扫描非法转换 |

### 2.2 可选追溯关系(Recommended)

| 源 | 目标 | 含义 |
|----|------|------|
| `user-stories.md` US | 各下游文档的"覆盖 Story" | Story 维度的覆盖 |
| `requirement.md` ROLE | 各下游文档的权限/角色相关章节 | 角色覆盖 |
| `requirement.md` OQ | 各下游文档的 `<!-- TODO -->` | OQ 必须被引用,不能消失 |

### 2.3 禁止的反向追溯

| 禁止 | 含义 |
|-----|------|
| 下游文档不允许定义新的 AC | AC 只能由 PM 在 requirement.md 定义 |
| 下游文档不允许新增 API | API 只能在 data-contract.md 定义 |
| 下游文档不允许定义新角色 | 角色只能在 requirement.md 定义 |

---

## 3. AC 覆盖矩阵的强制结构

每份下游文档的 AC 覆盖检查表**必须**采用以下统一结构:

```markdown
## XX. AC 覆盖检查表

| AC ID | 描述简述 | 对应实现/测试 | 状态 |
|------|---------|------------|------|
| AC-XXXX-001 | [从 requirement.md 复制] | [本文档中的章节/组件/TC ID] | ✅ / 🟡 / ❌ |
```

**状态值含义**:
- ✅ 已覆盖
- 🟡 部分覆盖,有 `<!-- TODO -->`
- ❌ 未覆盖,需补充

> ⚠️ **生成时**:agent 必须把 requirement.md 中**全部** AC 列入此表。
> 如果本角色不负责覆盖某 AC(例如纯后端 AC 在 ui-spec 里),写 `N/A(不在本角色范围)`,**不要省略**。

---

## 4. TC ↔ AC 双向映射规则

QA agent 在生成测试用例时,必须执行以下双向校验:

### 4.1 正向(AC → TC)

```pseudo
for each AC in requirement.md:
  matched_tcs = find_tcs_with_covers_ac(AC.id)
  if len(matched_tcs) == 0:
    REPORT("AC %s 无对应 TC" % AC.id)
  if len(matched_tcs) < ac_min_tc_count(AC):
    WARN("AC %s 测试不足" % AC.id)
```

每个 AC 的最低 TC 数量(`ac_min_tc_count`)规则:

| AC 类型 | 最低 TC 数 | 说明 |
|--------|----------|------|
| 主流程 AC | ≥ 1 正向 + 1 异常 | 至少覆盖正反两条路径 |
| 边界值 AC | ≥ 4 | min, max, min-1, max+1 |
| 权限 AC | ≥ 2 | 有权限,无权限 |
| 状态转换 AC | ≥ 1 合法 + 1 非法 | 状态机闭环验证 |
| 异常 AC | ≥ 1 | 异常路径触发 |

### 4.2 反向(TC → AC)

```pseudo
for each TC in qa-spec.md:
  if TC.covers_ac is empty:
    REPORT("TC %s 未关联 AC" % TC.id)
  for ac_id in TC.covers_ac:
    if ac_id not in requirement.md.ACs:
      REPORT("TC %s 引用了不存在的 AC %s" % (TC.id, ac_id))
```

---

## 5. 代码层追溯约定

> 这部分约定写在代码而非文档中,通过 PR 模板和 lint 强制。

### 5.1 代码注释格式

任何实现 AC 的代码,**必须**在最近的方法/组件文档注释中标注:

```typescript
/**
 * 上传文件的核心逻辑
 * @covers AC-0042-001 AC-0042-002
 */
async function uploadFile(...) { ... }
```

```python
def create_resource(...):
    """
    创建资源 + 首个子记录

    @covers AC-0042-002
    """
    ...
```

### 5.2 测试代码追溯

测试代码必须显式声明覆盖的 AC:

```typescript
describe('AC-0042-001: 上传文件基本流程', () => {
  it('TC-0042-001-01: 正常上传文件', () => { ... });
});
```

### 5.3 PR 描述追溯

PR 模板必须包含:

```markdown
## 覆盖的 AC

- [ ] AC-XXXX-XXX(简述)

## 覆盖的 TC

- [ ] TC-XXXX-XXX-XX

## 文档同步

- [ ] frontend-spec.md §13 表已更新
- [ ] qa-spec.md §11 矩阵已更新
```

---

## 6. 校验脚本逻辑(伪代码)

> 实际实现可用 Python / Node 写,这里给出参考逻辑。

### 6.1 校验入口

```pseudo
function validate_traceability(repo_root):
  errors = []

  # 加载所有文档
  req = parse(repo_root / "requirement.md")
  dc = parse(repo_root / "data-contract.md")
  ui = parse(repo_root / "ui-spec.md")
  fe = parse(repo_root / "frontend-spec.md")
  be = parse(repo_root / "backend-spec.md")
  qa = parse(repo_root / "qa-spec.md")

  # 校验 1: AC 全部被 QA 覆盖
  errors += check_ac_to_tc(req.acs, qa.tcs)

  # 校验 2: AC 在三份开发文档中至少一份有声明
  errors += check_ac_to_implementation(req.acs, [ui, fe, be])

  # 校验 3: API 与状态机覆盖
  errors += check_api_coverage(dc.apis, fe, be)
  errors += check_state_machine_coverage(dc.state_machines, qa)

  # 校验 4: ID 格式与唯一性
  errors += check_id_uniqueness_and_format([req, dc, ui, fe, be, qa])

  # 校验 5: 引用链完整(无破链)
  errors += check_reference_integrity([ui, fe, be, qa], [req, dc])

  # 校验 6: 版本号同步
  errors += check_version_sync([ui, fe, be, qa], [req, dc])

  # 校验 7: OQ 未消失
  errors += check_oq_propagation(req.oqs, [ui, fe, be, qa])

  return errors
```

### 6.2 报告格式

校验脚本输出**统一格式**,便于 CI 解析:

```
[traceability check report - YYYY-MM-DD HH:mm:ss]

Summary:
  Total ACs:        42
  Covered:          38 (90%)
  Partial:          3
  Uncovered:        1

ERRORS:
  E001: AC-0042-015 has no corresponding TC in qa-spec.md
  E002: TC-0042-007-01 references non-existent AC-0042-099
  E003: API-005 is defined in data-contract.md but not called in any frontend component
  E004: State transition T-002 has no test case in qa-spec.md §3.4

WARNINGS:
  W001: AC-0042-008 has only 1 TC; expected ≥ 2 (boundary AC)
  W002: ui-spec.md@0.1.0 generated from requirement.md@0.1.0, but requirement.md is now @0.2.0
```

### 6.3 退出码约定

| 退出码 | 含义 | CI 行为 |
|-------|------|--------|
| 0 | 全部通过 | merge 通过 |
| 1 | 有 ERROR | 阻塞 merge |
| 2 | 仅 WARNING | 提示但允许 merge |

---

## 7. 覆盖率指标

校验脚本应输出以下指标进 dashboard:

| 指标 | 公式 | 目标 |
|-----|------|------|
| AC 覆盖率 | covered_acs / total_acs | 100%(P0)、≥ 95%(总体) |
| API 覆盖率 | covered_apis / total_apis | 100% |
| 状态转换覆盖率 | tested_transitions / total_transitions | 100% |
| OQ 解决率 | resolved_oqs / total_oqs | 上线前 = 100% |
| 文档版本同步率 | synced_downstream / total_downstream | 100% |

---

## 8. 变更影响分析(Impact Analysis)

当 `requirement.md` 或 `data-contract.md` 变更时,校验脚本应输出**受影响清单**:

```pseudo
function impact_analysis(diff):
  affected = {}

  for change in diff:
    if change.type == "AC modified":
      affected.tcs += find_tcs_covering(change.ac_id)
      affected.implementations += find_impl_for(change.ac_id)
    elif change.type == "API modified":
      affected.frontend += find_frontend_using(change.api_id)
      affected.backend += find_backend_implementing(change.api_id)
      affected.tcs += find_contract_tests(change.api_id)
    elif change.type == "state machine modified":
      affected.tcs += find_state_tests(change.state_machine)

  return affected
```

输出形式:

```
[Impact Analysis: requirement.md v0.1.0 → v0.2.0]

Modified ACs (3):
  AC-0042-005 (modified Given clause)
  AC-0042-006 (added new Then assertion)
  AC-0042-008 (priority changed P1 → P0)

Affected TCs (5):
  TC-0042-005-01, TC-0042-005-02 in qa-spec.md
  TC-0042-006-01 in qa-spec.md
  TC-0042-008-01, TC-0042-008-02 in qa-spec.md

Affected Implementations:
  frontend-spec.md §13: AC-0042-005, AC-0042-006
  backend-spec.md §15: AC-0042-006

Required Action:
  - QA agent: regenerate affected TCs
  - frontend-spec.md owner: update implementation references
  - Update traceability blocks (§0) in all affected docs
```

---

## 9. 在 CI 中的集成

### 9.1 触发时机

| 事件 | 校验范围 |
|-----|---------|
| PR 创建/更新 | 全量追溯校验,阻塞 merge |
| 文档 push 到主干 | 全量校验 + 输出 dashboard |
| 每日定时 | 全量 + 影响分析报告 |

### 9.2 PR 模板增强

```markdown
## 追溯校验

- [ ] 已运行 `scripts/validate.py`,无 ERROR
- [ ] 影响分析已 review
- [ ] 受影响下游文档已更新
```

---

## 10. 反模式(常见错误)

| 错误 | 后果 | 正确做法 |
|-----|------|---------|
| 覆盖表中只列覆盖的 AC,漏列未覆盖的 | 看不出缺口 | 列出全部 AC,未覆盖的标 ❌ |
| TC 描述用自然语言"覆盖登录场景",不写 AC ID | 校验脚本无法解析 | 用 `covers_ac: [AC-XXX]` 结构化字段 |
| 改了需求不改下游文档版本号 | 版本对不上 | 升级版本 + 更新溯源块 |
| OQ 在下游文档"猜个答案" | 上线后才发现错 | 用 `<!-- TODO: 等待 OQ-XXX -->` |
| 多份文档各自分配 ID | 冲突 | 按 `global-rules.md` §4 由指定角色统一分配 |

---

## 11. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | YYYY-MM-DD | | 初稿 |
