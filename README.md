# Smart Site — 需求与文档自动化仓库

> **一句话说明**：PM 写一份需求文档，AI agent 自动生成 UI / 前端 / 后端 / QA 四份下游文档，全程有规则约束、有追溯链、有版本管理。

---

## 🗺️ 新人快速导览

**不知道从哪里读起？按顺序看这 5 个文件：**

| 步骤 | 文件 | 读完你会知道 |
|------|------|------------|
| 1 | `background/project-overview.md` | 这是什么产品、三端分别是什么 |
| 2 | `MULTI-PLATFORM.md` | 三端（PC / APP / H5）如何分文件夹管理 |
| 3 | `rules/README.md` | 整套 agent 规则体系如何运作 |
| 4 | `scripts/GENERATE.md` | PM 如何触发一次完整的文档生成 |
| 5 | `VERSIONS.md` | 当前有哪些 REQ，各自处于什么状态 |

---

## 📁 目录结构全览

```
smart-site/
│
├── background/                  # 项目背景（新人必读）
│   ├── project-overview.md      # 产品简介、目标、角色
│   ├── business-context.md      # 业务背景
│   ├── key-decisions.md         # ⭐ 关键设计决策（DEC-001~007）
│   ├── market-analysis.md       # 市场分析
│   └── user-research.md         # 用户研究
│
├── requirements/                # PM 编写的需求文档（唯一事实源）
│   ├── shared/                  # 跨端共享业务规则（REQ-XXX-shared.md）
│   ├── pc/                      # PC 管理端需求（REQ-XXX-pc.md）
│   ├── app/                     # APP 移动端需求（REQ-XXX-app.md）
│   └── h5/                      # H5 移动端需求（REQ-XXX-h5.md）
│
├── outputs/                     # ⚡ 由 agent 自动生成，不要手动大改
│   ├── ui/
│   │   ├── shared/              # ⭐ 跨端共享：设计 Token + 组件注册表
│   │   ├── pc/                  # PC 端 UI 说明（UI-REQ-XXX-pc.md）
│   │   ├── app/                 # APP 端 UI 说明
│   │   └── h5/                  # H5 端 UI 说明
│   ├── frontend/
│   │   ├── pc/                  # PC 端前端说明（FRONTEND-REQ-XXX-pc.md）
│   │   ├── app/                 # APP 端前端说明
│   │   └── h5/                  # H5 端前端说明
│   ├── backend/                 # 后端说明，不分端（BACKEND-REQ-XXX.md）
│   ├── qa/
│   │   ├── pc/                  # PC 端测试用例（QA-REQ-XXX-pc.md）
│   │   ├── app/                 # APP 端测试用例
│   │   └── h5/                  # H5 端测试用例
│   └── shared/                  # 数据契约（DATA-CONTRACT-REQ-XXX.md）
│
├── rules/                       # Agent 生成规则（规定文档怎么生成）
│   ├── README.md                # ⭐ 规则体系总览，必读
│   ├── global-rules.md          # 所有 agent 共享的基础规则 + 项目专属约定
│   ├── ui-rules.md              # UI agent 规则
│   ├── frontend-rules.md        # 前端 agent 规则
│   ├── backend-rules.md         # 后端 agent 规则
│   ├── qa-rules.md              # QA agent 规则
│   ├── data-contract-rules.md   # 数据契约 agent 规则
│   ├── traceability-rules.md    # 追溯规则（AC → 代码 → 测试的闭环）
│   ├── glossary-rules.md        # 术语表 agent 规则
│   └── user-stories-rules.md    # 用户故事 agent 规则
│
├── templates/                   # 文档模板（规定文档长什么样）
│   ├── requirement.md           # 需求文档模板（PM 写需求时使用）
│   ├── ui-spec.md               # UI 说明模板
│   ├── frontend-spec.md         # 前端说明模板
│   ├── backend-spec.md          # 后端说明模板
│   ├── qa-spec.md               # QA 测试用例模板
│   ├── data-contract.md         # 数据契约模板
│   ├── glossary.md              # 术语表模板
│   └── user-stories.md          # 用户故事模板
│
├── scripts/                     # PM 操作指令
│   ├── GENERATE.md              # ⭐ 生成一个 REQ 全套下游文档的完整指令
│   └── IMPACT-ANALYSIS.md       # ⭐ 需求变更时分析影响范围的指令
│
├── releases/                    # 发布快照归档（每次版本发布时存入）
│
├── README.md                    # 本文件
├── VERSIONS.md                  # ⭐ REQ 状态总表 + 版本规划
├── CHANGELOG.md                 # 各 REQ 的功能变更日志
└── MULTI-PLATFORM.md            # 多端管理规范
```

---

## 🔄 核心工作流

```
PM 写需求                触发生成                  下游文档
─────────────────       ──────────────────        ──────────────────────
requirements/           scripts/GENERATE.md  →    outputs/shared/       (数据契约)
  REQ-XXX-pc.md    →    粘贴给 AI agent      →    outputs/ui/pc/        (UI 说明)
                                             →    outputs/frontend/pc/  (前端说明)
                                             →    outputs/backend/      (后端说明)
                                             →    outputs/qa/pc/        (测试用例)
```

**需求变更时**：使用 `scripts/IMPACT-ANALYSIS.md` 分析哪些下游文档需要重新生成。

---

## 📐 关键设计原则

| 原则 | 说明 |
|------|------|
| **单一事实源** | `requirements/` 是唯一权威，`outputs/` 全部从它派生 |
| **追溯闭环** | 每个 AC → UI 页面 → 前端实现 → 后端接口 → QA 测试用例，全链路可追溯 |
| **语义引用** | 文档只写语义名（如 `"primary"`），不硬编码具体值（如 `#6892ff`） |
| **平台独立** | PC / APP / H5 三端独立维护，Token 和设计语言不继承 |
| **agent 可重复执行** | 相同输入 + 相同规则 = 相同输出，任何 agent 可接手继续工作 |

---

## ⭐ 最重要的几个文件

| 文件 | 为什么重要 |
|------|-----------|
| `rules/global-rules.md §14` | 项目所有技术约定的集中地（技术栈/枚举/错误码/API规范） |
| `background/key-decisions.md` | 解释"为什么这样设计"，防止新人推翻已有决策 |
| `outputs/ui/shared/UI-COMPONENT-REGISTRY.md` | 已有 UI 组件注册表，生成新文档前必查 |
| `scripts/IMPACT-ANALYSIS.md §二` | 跨 REQ 依赖关系图 |

---

## 📊 当前 REQ 覆盖状态

详见 `VERSIONS.md` — REQ 文档版本状态总表。

| 状态 | 含义 |
|------|------|
| ✅ 同步 | 需求与下游文档版本一致 |
| 🔴 下游缺失 | 需求存在，下游文档未生成（当前：REQ-015） |
| 🟠 下游过期 | 需求已更新，下游文档未重新生成 |
| 🟡 草稿 | 需求文件仍为 draft |

---

## 🏗️ 架构备忘

本仓库当前为单仓结构。未来计划分离为：
- `smart-site`（本仓库）— 业务需求
- `smart-site-design` — 设计体系（Token + 组件规范）
- `ai-doc-framework` — agent 规则框架（rules + templates）

触发时机和分离成本见 `rules/global-rules.md §14.10`。

---

*最后更新：2026-05-01*
