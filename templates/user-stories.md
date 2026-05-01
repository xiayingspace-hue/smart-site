---
doc_type: user_stories
req_id: REQ-XXXX
version: 0.1.0
status: draft
generated_from: requirement.md@0.1.0
generated_at: YYYY-MM-DD
owner: ""
---

# 用户故事拆分:[需求标题]

> **本文档把需求拆解为可独立交付、可独立测试的最小单元(User Story)**。
>
> - 每个 Story 应在 1 个 sprint 内可完成。
> - 每个 Story 自带验收标准、依赖、估时。
> - 下游开发与测试以 Story 为单位执行,而不是以"模块"为单位。

---

## 1. 史诗(Epic)概览

| Epic ID | 名称 | 描述 | 包含 Story | 优先级 |
|--------|------|------|-----------|-------|
| EPIC-01 | | | | |
| EPIC-02 | | | | |

---

## 2. Story 拆分原则(INVEST)

本文档拆分时遵循 INVEST 原则:

- **Independent**: Story 之间尽量解耦,可独立开发
- **Negotiable**: 细节可调整,不是契约
- **Valuable**: 每个 Story 对最终用户有价值
- **Estimable**: 工作量可估
- **Small**: 1 个 sprint 内完成
- **Testable**: 可写出明确测试用例

---

## 3. Story 详述

### US-001:[Story 标题]

**Epic**: EPIC-XX
**优先级**: P0 / P1 / P2
**估时**: <!-- 例如 5 人日(前 X + 后 Y + QA Z) -->

**故事描述**:

```
作为 [角色]
我想要 [功能/行为]
以便 [价值/好处]
```

**前置依赖**:
<!-- 例如:
- 用户已登录
- 某前置 Story 已完成
-->

**所属流程节点**: 需求文档 §X.X 步骤 N

**关联 AC**: AC-XXXX-001, AC-XXXX-002

**验收要点**(详细 AC 见 requirement.md §8):
<!-- 简述本 Story 的验收要点,详细规范在需求文档 -->

**涉及接口**: API-XXX, API-YYY

**技术风险**:
<!-- 例如:某第三方依赖、性能瓶颈 -->

**已知问题/待定**:
<!-- 引用 OQ-XXX -->

---

### US-002:[Story 标题]

<!-- 按相同结构继续展开 -->

---

## 4. Story 依赖图

```
EPIC-XX:
  US-001 ─┬─ US-002 (依赖 001)
          └─ US-003 (依赖 001)

EPIC-YY:
  [需要 EPIC-XX.US-001 完成] → US-XXX
```

> 此处建议在生成时由 agent 自动绘制 mermaid 图,便于团队规划 sprint。

---

## 5. Sprint 排布建议

| Sprint | 包含 Story | 交付价值 |
|--------|-----------|---------|
| Sprint 1 | | |
| Sprint 2 | | |

---

## 6. Story 状态追踪

> 实际项目中建议同步到 Jira/PingCode 等系统。本表用作快照。

| Story ID | 状态 | Owner | 实际工时 | 关联 PR | 备注 |
|---------|------|-------|---------|--------|------|
| US-001 | TODO | - | - | - | - |

状态值: `TODO / IN_PROGRESS / IN_REVIEW / IN_TEST / DONE / BLOCKED`
