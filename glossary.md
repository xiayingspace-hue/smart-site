---
doc_type: glossary
req_id: GLOBAL
version: 0.6.0
status: draft
owner: ""
---

# 项目命名与缩写规范

> **本文档只收录与业务无关的书写约定**：技术缩写、ID 与文件命名、排版规则。
>
> **不收录**：
>
> | 内容 | 归属 |
> |------|------|
> | 角色定义与角色禁用词 | `requirements/shared/ROLES-shared.md` |
> | 单个需求内部的业务术语 | 该需求 §6 核心实体与数据生命周期 |
> | 状态枚举取值、字段定义 | data-contract §2 / §1 |

---

## 1. 技术术语缩写

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

---

## 2. 命名规范

### 2.1 ID 命名

| 类型 | 格式 | 示例 |
|-----|------|------|
| 需求 | `REQ-{3 位数字}[子需求字母]` | REQ-003、REQ-003A |
| 用户故事 | `US-{3 位数字}` | US-001 |
| 验收标准 | `AC-{REQ 编号}-{3 位数字}` | AC-003H-001 |
| 测试用例 | `TC-{AC 编号}-{2 位数字}` | TC-003H-001-01 |
| 接口 | `API-{3 位数字}` | API-001 |
| 实体 | `ENT-{3 位数字}` | ENT-001 |
| 状态 | `S-{大写英文}` | S-PUBLISHED |
| 转换 | `T-{3 位数字}` | T-001 |
| 待定问题 | `OQ-{3 位数字}` | OQ-001 |
| 角色 | `ROLE-{3 位数字}` | ROLE-001 |

> `ROLE-XXX` 是**需求文件内的局部编号**，不跨文档通用。
> 跨文档引用角色须使用标准名，见 `requirements/shared/ROLES-shared.md` §1。

### 2.2 文件命名

> 端标识取值:`shared` / `pc` / `app` / `h5`。多端拆分原则见 `rules/global-rules.md` §15。

| 类型 | 格式 | 示例 |
|-----|------|------|
| 需求文件 | `REQ-{编号}-{端标识}.md` | `REQ-003A-pc.md`、`REQ-007-shared.md` |
| 数据契约 | `DATA-CONTRACT-REQ-{编号}-{端标识}.md` | `DATA-CONTRACT-REQ-003H-pc.md` |
| UI 说明 | `UI-REQ-{编号}-{端标识}.md` | `UI-REQ-003A-pc.md` |
| 前端说明 | `FRONTEND-REQ-{编号}-{端标识}.md` | `FRONTEND-REQ-003A-pc.md` |
| 后端说明 | `BACKEND-REQ-{编号}.md`（不分端） | `BACKEND-REQ-003A.md` |
| 测试用例 | `QA-REQ-{编号}-{端标识}.md` | `QA-REQ-003A-pc.md` |

> **例外**：不属于任何单个 REQ 的跨端共享文档，采用 `{主题}-shared.md` 命名，
> 如 `requirements/shared/ROLES-shared.md`（全项目角色定义）。

### 2.3 字段命名

- 数据库字段:`snake_case`
- API 字段:`snake_case`(对外契约统一,前端 client 内部转 camelCase)
- 前端组件 prop:`camelCase`
- TS 类型:`PascalCase`

### 2.4 枚举值命名

- 全大写,下划线分隔(如 `IN_PROGRESS`)
- 不要用数字编码
- 枚举的**取值**定义在 data-contract §2，本表不重复

---

## 3. 排版规范

### 3.1 全角 / 半角混用

| ❌ 不要用 | ✅ 应该用 |
|---------|---------|
| `Document Controller (DC)` | `Document Controller（DC）` |
| `Site Engineer (SE)` | `Site Engineer（SE）` |

> 中文语境下的括号统一用全角 `（）`。当前 `REQ-007-shared` 使用半角,其余文件使用全角。

---

## 4. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | YYYY-MM-DD | | 初稿 |
| 0.2.0 | 2026-08-08 | XIA YING | 填充角色术语与禁用词：确立图纸线 7 个标准角色名；登记 10 个角色同义词为禁用词；标注「业务人员 / 普通业务人员」含义相反的高危混淆 |
| 0.3.0 | 2026-08-08 | XIA YING | 新增「文件命名」小节（由 MULTI-PLATFORM.md 并入）；需求 ID 格式修正为 `REQ-{3 位数字}[子需求字母]` |
| 0.4.0 | 2026-08-08 | XIA YING | 移除「业务术语」（术语绝大多数属单个需求，归各需求 §6 核心实体）与「状态术语」（原要求与 data-contract §2 完全一致，属重复定义） |
| 0.5.0 | 2026-08-08 | XIA YING | 删除上述两节的占位说明并重排章节编号 |
| 0.6.0 | 2026-08-08 | XIA YING | 角色定义与角色禁用词拆出至 `requirements/shared/ROLES-shared.md`（依据 `rules/global-rules.md` §15.2「权限与角色定义」属 shared 内容）。本文件更名为「项目命名与缩写规范」，只保留与业务无关的书写约定；章节重排为 §1 技术缩写 / §2 命名规范 / §3 排版规范 / §4 变更历史 |
