---
doc_type: ui_spec
req_id: REQ-007D-pc
version: 0.1.1
status: draft
generated_from: REQ-007D-pc@0.1.1
generated_at: 2026-05-24
owner: ""
---

# UI 设计说明：PC 端 — DC Configuration 页面

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-007D-pc.md（主）、REQ-007-shared（业务规则）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc § 3.2
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007D-pc @ v0.1.1 |
| 覆盖用户故事 | US-007D-001、US-007D-002 |
| 覆盖 AC | AC-007D-001 ~ AC-007D-010 |
| 上次同步时间 | 2026-05-24 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

管理员在 Settings 页面中配置项目的 Document Controller（DC）列表，确保外部审批流程有专属负责人。

### 1.2 设计原则

1. **零 DC 预警**：若 DC 列表为空，系统显示显眼警告横幅，提示流程无法推进。
2. **连续添加**：Add DC 弹窗在每次添加成功后保持打开，支持批量配置。
3. **权限门禁**：无 `drawing:dc-config` 权限时，侧边栏菜单项直接隐藏，直接访问跳转 403。

---

## 2. 信息架构

### 2.1 入口

侧边栏 Settings → DC Configuration（需 `drawing:dc-config` 权限）

### 2.2 页面层级

```
Settings 模块
└── DC Configuration 页面
    ├── DC 列表（主内容区）
    ├── Add Document Controller 弹窗（§3.2）
    └── Remove DC 二次确认 Dialog（§3.3）
```

---

## 3. 页面 / 组件清单

### 3.1 DC Configuration 页面（主列表）

**关联 Story**：US-007D-001
**关联 AC**：AC-007D-001、AC-007D-002、AC-007D-006~008

#### 主布局

```
┌──────────────────────────────────────────────────────────────────┐
│ ⚙️ DC Configuration                                  [+ Add DC] │
├──────────────────────────────────────────────────────────────────┤
│  Document Controllers for current project:                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Name       │ Configured By │ Configured At │ Action        │  │
│  │─────────────│───────────────│───────────────│───────────────│  │
│  │  陈小明      │ Admin         │ 2026-04-01    │ [Remove]      │  │
│  │  刘文静      │ Admin         │ 2026-04-01    │ [Remove]      │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ⓘ At least one DC is required for the drawing approval process. │
└──────────────────────────────────────────────────────────────────┘
```

#### 列定义

| 列 | 内容 |
|----|------|
| Name | DC 用户姓名 |
| Configured By | 配置操作人姓名 |
| Configured At | 配置时间 `YYYY-MM-DD` |
| Action | [Remove] 按钮 |

#### 状态（5 态）

| 状态 | 触发条件 | 视觉表现 | 文案 |
|-----|---------|---------|-----|
| 空（无 DC） | 移除最后一个 DC 后 | 警告横幅 + 空状态提示 | `"⚠️ No DC configured. Internal approvals cannot proceed to external approval. Please add at least one Document Controller."` |
| 加载 | 列表请求中 | 表格 Skeleton | — |
| 正常 | 已配置 ≥1 个 DC | 如布局图所示 | — |
| 错误 | 接口失败 | 错误提示 + 重试 | "Failed to load DC list. Retry" |
| 极端数据 | DC 数量 > 10 | 表格内分页 | — |

#### 权限可见性

| UI 元素 | 业务人员/管理员 | DC 自身 | 普通用户 |
|--------|:--------------:|:-------:|:-------:|
| 整个页面（侧边栏菜单） | ✅ | ❌ | ❌ |
| [+ Add DC] | ✅ | — | — |
| [Remove] | ✅ | — | — |

> **AC-007D-001**：无权限用户直接访问 URL 时返回 403 页面，侧边栏菜单不显示此入口。

---

### 3.2 Add Document Controller 弹窗

**关联 AC**：AC-007D-003 ~ 005、AC-007D-009

#### 布局

```
┌──────────────────────────────────────────┐
│ Add Document Controller            [✕]   │
├──────────────────────────────────────────┤
│ Search: [____________________________]   │
│                                          │
│ Available Users:                         │
│  ┌──────────────────────────────────┐    │
│  │ 王磊                    [Add]    │    │
│  │ 赵强                    [Add]    │    │
│  │ 孙丽                    [Add]    │    │
│  └──────────────────────────────────┘    │
│                                          │
│                           [Close]        │
└──────────────────────────────────────────┘
```

#### 交互规则

| 操作 | 行为 |
|------|------|
| Search 输入 | 实时模糊搜索 Available Users 列表（按姓名） |
| 点击 [Add] | 立即调用接口；成功后该用户从弹窗列表消失，主列表新增行；弹窗**保持打开**支持连续添加 |
| 接口失败 | Toast 报错，弹窗保留 |
| Available Users 为空 | 显示 `"All eligible users have been configured as DC."` |
| 点击 [Close] 或 [✕] | 弹窗关闭 |

---

### 3.3 Remove DC 二次确认 Dialog

**关联 AC**：AC-007D-006 ~ 008

#### 布局

```
┌──────────────────────────────────────────────────┐
│  Remove DC                                       │
│                                                  │
│  Remove {dcName} from DC list?                   │
│  They will no longer receive external            │
│  approval tasks.                                 │
│                                                  │
│             [Cancel]    [Confirm]                │
└──────────────────────────────────────────────────┘
```

#### 交互规则

| 操作 | 行为 |
|------|------|
| [Confirm] | loading → 接口调用 → 成功后主列表移除该行；若为最后一个 DC，显示空状态警告横幅 |
| [Cancel] | Dialog 关闭，主列表不变 |

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色

| 用途 | Token | 色值 |
|-----|-------|------|
| 警告横幅图标/文字 | `--color-status-warning` | `#E6A23C` |
| 警告横幅背景 | `--color-status-warning-bg` | `#FDF6EC` |

### 4.2 间距与字体

| 组件 | 属性 | 值 |
|-----|------|----|
| 页面标题 | font-size | 18px |
| 页面标题 | font-weight | 600 |
| 表格 | font-size | 14px |
| [+ Add DC] 按钮 | type | primary |
| [Remove] 按钮 | type | danger text |
| Dialog | width | 420px |

---

## 5. 无障碍（A11y）

- WCAG 等级：AA
- 键盘导航：弹窗内字段支持 Tab 键；Esc 关闭弹窗；[Confirm] 支持 Enter
- 焦点可见：所有交互元素有 `focus-visible` 样式

---

## 6. 文案规范

| 场景 | 文案（en） |
|-----|----------|
| 无 DC 警告横幅 | ⚠️ No DC configured. Internal approvals cannot proceed to external approval. Please add at least one Document Controller. |
| 可用用户列表为空 | All eligible users have been configured as DC. |
| Remove DC 确认文案 | Remove {dcName} from DC list? They will no longer receive external approval tasks. |
| 添加成功 Toast | {dcName} has been added as Document Controller. |
| 移除成功 Toast | {dcName} has been removed from DC list. |

---

## 7. AC 覆盖检查表

| AC ID | 对应章节 | 覆盖? |
|------|---------|------|
| AC-007D-001 | §3.1 权限：无权限 → 菜单隐藏 + 403 | ✅ |
| AC-007D-002 | §3.1 DC 列表正确展示 | ✅ |
| AC-007D-003 | §3.2 Available Users 过滤已配置 DC | ✅ |
| AC-007D-004 | §3.2 添加成功路径 | ✅ |
| AC-007D-005 | §3.2 连续添加（弹窗保持打开） | ✅ |
| AC-007D-006 | §3.3 Remove Dialog 文案含姓名 | ✅ |
| AC-007D-007 | §3.3 移除成功路径 | ✅ |
| AC-007D-008 | §3.1 空状态 — 移除最后一个 DC → 警告横幅 | ✅ |
| AC-007D-009 | §3.2 Search 实时模糊搜索 | ✅ |
| AC-007D-010 | 集成验证：配置 DC 后 Todo 下发正确 | ✅ |

---

## 8. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 UI-REQ-007-pc.md 拆分，覆盖 REQ-007D-pc DC 配置部分 |
| 0.1.1 | 2026-05-24 | agent | 按来源需求独立成文件，完善字段与状态规范 |
