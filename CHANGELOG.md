# Smart Site — 需求变更日志

> 按需求编号记录每个 REQ 的功能范围、涉及端、对已有需求的影响。
> 快速了解"每个需求做了什么"以及"改了什么"。
>
> **最后更新**：2026-05-24

---

## REQ-015 · BCA 月度人力数据提交（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: 无

### 新增内容
- PC 端新增 BCA 月度人力数据提交功能，支持工地管理员发起月度数据提交任务
- 支持分批次发送（单月数据量大），清晰展示每批次的发送状态（待发送/发送中/成功/失败）
- 支持对失败批次或失败记录进行单独重试，确保最终数据完整上报
- 支持查看提交任务历史（按月份）及批次明细
- 平台管理员可全局配置 BCA 接口凭证和自动提交开关
- 自动提交：支持按月定时自动发起提交任务（可配置）

### 对已有需求的影响
- 无（首次引入 BCA 上报模块）

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-015-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-015-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-015-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-015.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-015-pc.md` |

---

## REQ-014 · 图纸管理用户反馈迭代（REQ-003/004/005 第一版用户反馈汇总）

- **状态**: 草稿
- **涉及端**: PC / APP
- **依赖**: REQ-003、REQ-004、REQ-005

### 新增内容
- **FB-001**（REQ-003）：上传新版本时允许修改 Drawing Code（图纸编号本身），同项目唯一性校验，历史版本保留原值不变
- **FB-002**（REQ-003）：图纸版本详情新增附件（Attachments）区域，支持任意文件类型上传；SE 可下载，上传/删除权限与 PDF 管理一致
- **FB-003**（REQ-004）：`+Markup` 入口从操作列移入 Markup 历史弹框，弹框内新增 `+ Add Markup` 按钮
- **FB-004**（REQ-004）：Markup 历史列表新增"Drawing Code"列，由系统在创建 Markup 时自动记录，只读展示
- **FB-005**（REQ-004）：SE 角色在 PC 端和 APP 端完全隐藏 Markup 相关所有入口（`v-if` 控制），接口层同步返回 403
- **FB-006**（REQ-003/REQ-005）：SE 视角只显示 Drawing Code，隐藏系统版本号（Ver）字段；非 SE 角色同时显示两者

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | `drawing_versions` 表 `drawing_code` 允许随新版本更新；新增 `drawing_attachments` 表；SE 响应中过滤 `ver` 字段 |
| **REQ-004** | `markups` 表新增 `drawing_code` 字段（创建时自动写入）；SE 角色接口层 403；Markup 历史弹框新增 `+ Add Markup` 入口 |
| **REQ-005** | SE PC 端图纸列表及版本详情不再显示系统版本号（Ver） |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-014-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-014-pc.md` · `outputs/ui/app/UI-REQ-014-app.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-014-pc.md` · `outputs/frontend/app/FRONTEND-REQ-014-app.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-014.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-014-pc.md` · `outputs/qa/app/QA-REQ-014-app.md` |

---

## REQ-013 · SE"我的任务"菜单与任务执行页面（APP端）

- **状态**: 草稿
- **涉及端**: APP
- **依赖**: REQ-001、REQ-012

### 新增内容
- APP 端首页新增"Progress Management"模块，SE 可见"我的任务"入口
- APP 首页底部 notification 入口，点击后进入通知中心，顶部显示两个 Tab：
  - Todo（待办）：展示所有与用户相关的待办事项，支持一键跳转
  - 消息通知：展示系统消息、任务变更、问题处理等通知
- SE 专属"我的任务"页面：展示 CM 分配的任务列表，支持筛选、搜索
- 任务详情页：工序要求、计划/实际时间、优先级、进度填报、工单反馈、问题上报等
- 消息通知：新任务分配、变更、作废、逾期预警、问题处理反馈等

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-001** | APP 首页新增 Progress Management 模块入口 |
| **REQ-012** | CM 分配任务给 SE 后，SE 在此页面接收并执行 |

### 文档清单
| 类型 | 文件 |
|------|------|
| APP 端需求 | `requirements/app/REQ-013-app.md` |
| UI 设计 | `outputs/ui/app/UI-REQ-013-app.md` |

---

## REQ-012 · CM"我的任务"菜单与任务执行页面（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001、REQ-010、REQ-011

### 新增内容
- PC 端 Progress Management 新增"我的任务（CM）"菜单，仅 CM 可见
- 任务列表页：展示 PM 分配给 CM 的所有任务，支持筛选、搜索、分页，与 PM 端布局一致
- 新建任务弹框：支持引用工序模板或自定义新建，自动关联上级任务
- 任务拆解：CM 可将任务分配/分解给 SE，弹框支持批量新建
- 问题汇报：操作列及任务详情页均有"汇报问题"入口，提交后通知 PM
- 消息通知：SE 分配通知、任务变更、作废、进度反馈、逾期预警等全场景通知

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-010** | 新增 CM 专属菜单入口 |
| **REQ-011** | PM 分配任务给 CM 后，CM 在此页面接收并分解任务 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-012-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-012-pc.md` |
| 前端开发 | （待生成） |
| 后端开发 | （待生成） |
| QA 测试 | （待生成） |

---

## REQ-011 · PM"我的任务"菜单与任务执行页面（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001、REQ-010

### 新增内容
- PC 端 Progress Management 新增"我的任务（PM）"菜单，仅 PM 可见
- 任务列表页：展示 Planning 分配给 PM 的所有任务，布局与 Planning 端 Master Program 一致
- 新任务高亮显示，顶部显示新任务数量提示，支持一键筛选
- 新建任务弹框：从右侧侧滑，需关联 Planning 主计划任务，支持批量新建
- 任务分配/分解给 CM，弹框支持批量操作
- 问题汇报：操作列及任务详情页均有"汇报问题"入口，提交后通知 Planning
- Planning 可在消息中心和 Master Program 任务详情页"问题汇报"Tab 查看处理
- 分级进度填报：SE 填报 → CM 审核 → PM 查看，PM 无需手动填报
- 消息通知：覆盖分配、变更、作废、进度反馈、问题汇报、逾期预警等全场景

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-010** | 新增 PM 专属菜单入口 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-011-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-011-pc.md` |
| 前端开发 | （待生成） |
| 后端开发 | （待生成） |
| QA 测试 | （待生成） |

---

## REQ-010 · 项目进度管理菜单与任务分配页面布局（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001、REQ-008、REQ-009

### 新增内容
- PC 端 Progress Management 新增"Master Program"菜单（Planning 专用）
- Master Program 任务表格：Activity ID、Activity Name、WBS、计划/实际时间、Deviation、Status、Assigned PM、操作等
- 新增/分配任务弹框：从右侧侧滑，样式与系统风格一致
- 任务分配（Planning → PM）：分配按钮、批量分配
- 任务编辑/作废：操作列支持编辑、作废，含权限控制和二次确认
- 任务状态管理：未分配/已分配/进行中/已完成/已延期/作废，颜色高亮
- Deviation 字段：自动计算计划与实际偏差，颜色区分延期/提前/如期
- 消息通知：分配/变更/作废/进度/异常等场景全覆盖，含消息内容模板

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-008** | 工序模板在任务新建/拆解时被引用 |
| **REQ-009** | 项目级模板数据作为任务拆解的基础 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-010-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-010-pc.md` |
| 前端开发 | （待生成） |
| 后端开发 | （待生成） |
| QA 测试 | （待生成） |

---

## REQ-009 · 项目层级字典与工序配置副本机制（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-008

### 新增内容
- 项目管理后台 Progress Management 下新增 Process Settings 子菜单
- 支持从公司层级一键复制字典配置和工序模板到项目层级
- 项目层级拥有独立的字典和工序配置副本，可自由增删改，不影响公司层级和其他项目
- 复制操作前弹窗确认，操作日志记录
- 页面结构与公司层级一致（专业 Tab、构件类型、工序表格三级联动）

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-008** | 项目级模板数据来源于公司级，复制后独立维护 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-009-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-009-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-009-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-009.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-009-pc.md` |

---

## REQ-008 · 公司层级工序模板配置（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001

### 新增内容
- Company Insights 视角下，左侧导航 Project Progress > Process Settings 页面
- 专业 Tab 栏（来自公司字典，仅展示，不可增删）
- 左侧构件类型列表（来自字典，仅展示）
- 右侧工序表格：支持新增、编辑、删除、禁用/启用、批量添加、排序
- 工序权重字段，用于后续进度汇总计算
- 删除/禁用逻辑：引用状态校验，已用于实际任务仅可禁用，不可物理删除
- 专业 Tab、构件类型、工序三级联动

### 对已有需求的影响
- 无（首次引入工序模板模块）

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-008-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-008-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-008-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-008.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-008-pc.md` |

---

## REQ-007E · 内部审批人图纸待办详情查看与原文件下载（PC端）

- **状态**: 草稿（v0.3.0，2026-05-23）
- **涉及端**: PC
- **依赖**: REQ-007-shared、REQ-007A-pc、REQ-003A-pc

### 新增内容
- Todo 列表**仅展示指派给当前登录用户**的内部审批待办，后端以 `assigneeId = 当前用户` 过滤，不可绕过
- 点击待办记录右侧 **[Detail]** 按钮，从右侧弹出**详情侧滑弹框（Detail Drawer）**
- 详情弹框展示与设计人员上传时完全一致的全量字段：Drawing Code、Drawing Name、Category、Description、系统版本号、Version Note、上传人、上传时间，以及原始文件行（文件名可点击在新标签页 inline 打开）
- 底部提供 **[Download Original File]** 按钮，触发浏览器强制下载，命名规则：`{drawingCode}-V{versionNo}-original.{ext}`
- 下载后弹框保留，审批人可在弹框内继续执行 [Approve] / [Reject]（交互详见 REQ-007A-pc）

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-007A-pc** | Todo 列表过滤规则收严（仅展示自己名下的待办）；[Detail] 按钮补充原卡片内联展示 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-007E-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-007E-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-007E-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-007E.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-007E-pc.md` |

---

## REQ-007D · DC 配置页面（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-007-shared

### 新增内容
- Project Settings 下新增 DC Configuration 子页面（仅具备 `drawing:dc-config` 权限可访问）
- 展示当前项目所有已配置 DC 成员列表，支持新增、删除
- 全量覆盖模式：每次保存以当前列表为准，历史任务不受影响
- 内部审批通过后系统自动向全部已配置 DC 推送外部审批 Todo，无需上传人手动指定

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-007A** | 内部审批通过后通知对象由"手动指定"改为"项目 DC 配置表" |
| **REQ-007B** | DC 收到 Todo 任务的前提是已在此页面完成配置 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-007D-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-007D-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-007D-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-007D.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-007D-pc.md` |

---

## REQ-007C · 版本历史抽屉 4 阶段生命周期视图（PC端）

- **状态**: 草稿（v0.2.3，2026-05-23）
- **涉及端**: PC
- **依赖**: REQ-007-shared、REQ-007A-pc、REQ-007B-pc

### 新增内容
- 图纸版本历史抽屉升级为可展开的 4 阶段生命周期视图：① 上传 → ② 内部审批 → ③ 外部审批 → ④ 签字版
- 每阶段卡片展示：责任人、操作时间、审批结果、文件链接（原始文件 / 签字版 / 审批凭证）
- ③ 外部审批卡片展示具体 Status of Approval（A/B/D/C/E），与纸质报审表对应
- DC 可在 ③ 外部审批卡片直接点击 [Mark Result] 发起标记（与 Todo 操作一致）
- 版本历史主列表新增 **Attachments 列**（📎 n），点击打开二级弹框
- 二级弹框（480px 宽抽屉）顶部展示：Status、Ver (System)、Description、Submission Ref No.、Submission Subject、Uploaded by、Upload Date 及主文件行；**不含 Drawing Code**
- 二级弹框内含 **Part Print / Attachments 双 Tab**：
  - **Part Print Tab**：只读展示基于当前版本发布的 Markup（局部更新）列表
  - **Attachments Tab**（默认激活）：附件上传/下载/删除，上传/删除仅限本版本上传人

### 变更记录
| 版本 | 日期 | 变更摘要 |
|------|------|---------|
| v0.2.3 | 2026-05-23 | 二级弹框新增 Part Print / Attachments 双 Tab |
| v0.2.2 | 2026-05-23 | 二级弹框顶部移除 Drawing Code，新增 Description / Submission Ref No. / Submission Subject |
| v0.2.1 | 2026-05-23 | ③ 外部审批卡片对齐 Status A–E |
| v0.2.0 | 2026-05-06 | 新增 Attachments 列及二级弹框（REQ-014 FB-002） |
| v0.1.0 | 2026-05-05 | 初稿：4 阶段生命周期视图 |

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003C** | 版本历史抽屉标题及查阅确认面板标题移除 Drawing Code，对齐 REQ-003A 列表设计 |
| **REQ-003** | 原版本历史列表升级为 4 阶段生命周期视图，UI 结构调整 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-007C-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-007C-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-007C-pc.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-007C-pc.md` |

---

## REQ-007B · DC 外部审批 Todo 与标记 Dialog（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-007-shared、REQ-007A-pc、REQ-003D-pc

### 新增内容
- PC Todo 列表新增"外部审批"类型任务（仅 DC 角色可见）
- DC 可在 Todo 中直接下载原始文件，提交 Bentley 后回传签字版 PDF + 审批凭证
- [Mark Result] Dialog：填写 Status of Approval（A/B/D/C/E）、Submission Ref No.、Submission Subject、Submission Description（始终必填）
- Status A/B/D（通过类）：额外上传签字版文件 + 审批凭证 + 外部审批日期，触发版本生效 + QR 生成
- Status C/E（驳回类）：填写 Remarks，通知设计人员重新上传
- 操作完成后 Todo 任务自动关闭，历史存档可查

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-007A** | 内部审批通过后自动创建 DC 外部审批 Todo 任务 |
| **REQ-006** | 外部审批通过时同步触发 QR 码生成（替代原"审批通过后异步"逻辑） |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-007B-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-007B-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-007B-pc.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-007B-pc.md` |

---

## REQ-007A · 内部审批 Todo 调整（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-007-shared、REQ-003A-pc

### 新增内容
- 将 PC Todo 列表中的"审批"任务明确区分为"内部审批"，与旧单级审批任务进行视觉区分
- 内部审批人通过后，弹框提示"下一步将由 DC 负责外部审批，版本暂不生效"
- 驳回时弹框同旧流程，驳回理由必填，通知设计人员重新上传
- Todo 列表仅展示**指派给当前登录用户**的待办（与 REQ-007E 联动）

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003B** | 单级审批 Todo 交互升级为两级，内部审批通过后版本不再直接生效 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-007A-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-007A-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-007A-pc.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-007A-pc.md` |

---

## REQ-007 · 图纸两级审批流程（内部审批 + 外部审批）

- **状态**: 草稿
- **涉及端**: PC（全部操作）/ APP（内部审批 Todo）
- **依赖**: REQ-003、REQ-006

### 新增内容
- 将图纸审批从单级升级为**内部审批 → 外部审批**两级串行流程
- 新增角色：Document Controller (DC)，负责外部审批流转
- 上传人角色明确为**设计人员 (Designer)**，责任可追溯
- DC 从 Todo 下载原始文件 → 提交 Bentley 外部审批 → 回传签字版 + 凭证
- 版本审批状态扩展为 5 态：`PENDING_INTERNAL` → `INTERNAL_APPROVED` → `APPROVED`；驳回分内部/外部
- 新增 DC 配置页面（项目级，全量覆盖模式）
- 版本历史升级为可展开的 4 阶段生命周期视图（上传 → 内部审批 → 外部审批 → 签字版）
- Site Engineer 查看的是**外部签字版图纸**

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | 审批状态枚举 3→5 态；上传人角色变更；文件分为原始文件 + 签字版 |
| **REQ-004** | Markup 标注基于签字版 `signedFileUrl` |
| **REQ-005** | SE 在 PC 端看到签字版 |
| **REQ-006** | QR 生成时机从"审批通过后异步"改为"外部审批通过时同步"；QR 叠加基于签字版 |

### 文档清单
| 类型 | 文件 |
|------|------|
| 共享需求 | `requirements/shared/REQ-007-shared.md` |
| PC 端需求 | `requirements/pc/REQ-007-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-007-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-007-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-007.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-007-pc.md` |

---

## REQ-006 · 图纸二维码与公开状态查询

- **状态**: 草稿
- **涉及端**: PC（管理员查看/下载/重新生成 QR）/ H5（公开状态页）
- **依赖**: REQ-003、REQ-004

### 新增内容
- 图纸版本审批通过后自动生成 QR 码
- QR 叠加到 PDF 图纸指定位置，生成带码版 PDF（`pdfWithQrUrl`）
- 公开 H5 页面：扫码查看图纸版本状态、确认记录、活跃 Markup 数量（无需登录）
- PC 端管理员可查看 QR、下载带码 PDF、手动重新生成
- 公开访问令牌（`publicToken`）机制，无需登录即可查询

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | `DrawingVersion` 表新增 `publicToken`、`qrImageUrl`、`pdfWithQrUrl` 字段 |

### 文档清单
| 类型 | 文件 |
|------|------|
| 共享需求 | `requirements/shared/REQ-006-shared.md` |
| PC 端需求 | `requirements/pc/REQ-006-pc.md` |
| H5 端需求 | `requirements/h5/REQ-006-h5.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-006-pc.md` · `outputs/ui/h5/UI-REQ-006-h5.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-006-pc.md` · `outputs/frontend/h5/FRONTEND-REQ-006-h5.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-006.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-006-pc.md` · `outputs/qa/h5/QA-REQ-006-h5.md` |

---

## REQ-005 · Site Engineer PC 端图纸访问能力

- **状态**: 草稿
- **涉及端**: PC（SE 只读视图 + 站内通知）
- **依赖**: REQ-003、REQ-004

### 新增内容
- PC 端 Site Engineer 图纸列表（数据过滤规则与 APP 端一致）
- 图纸查阅确认（Confirm Reading）跨端同步：PC / APP 任一端确认，双端共享
- 局部更新 Mark as Read 跨端同步
- PC 端站内通知：局部更新发布时同步推送 PC 站内通知

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | 查阅确认 `deviceInfo` 字段同时支持 APP/PC |
| **REQ-004** | Mark as Read 跨端共享 |

### 文档清单
| 类型 | 文件 |
|------|------|
| 共享需求 | `requirements/shared/REQ-005-shared.md` |
| PC 端需求 | `requirements/pc/REQ-005-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-005-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-005-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-005.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-005-pc.md` |

---

## REQ-004 · 图纸局部更新（Markup）

- **状态**: 草稿
- **涉及端**: PC（发布局部更新）/ APP（接收通知、查阅）
- **依赖**: REQ-003

### 新增内容
- 设计人员在已有图纸版本上发布局部更新（Markup），无需走完整版本审批
- 定向通知已分配的 Site Engineer（App Push + 站内消息）
- SE 可标记已阅（Mark as Read）
- 局部更新有独立数据模型（`DrawingMarkup` 表），挂载在特定版本下

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | 在图纸版本上新增 Markup 关联；图纸详情页新增 Markup Tab |

### 文档清单
| 类型 | 文件 |
|------|------|
| 共享需求 | `requirements/shared/REQ-004-shared.md` |
| PC 端需求 | `requirements/pc/REQ-004-pc.md` |
| APP 端需求 | `requirements/app/REQ-004-app.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-004-pc.md` · `outputs/ui/app/UI-REQ-004-app.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-004-pc.md` · `outputs/frontend/app/FRONTEND-REQ-004-app.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-004.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-004-pc.md` · `outputs/qa/app/QA-REQ-004-app.md` |

---

## REQ-003E · 图纸上传 AI 自动识别页信息（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-003A-pc、REQ-003-shared

### 新增内容
- 新建图纸上传弹窗增强：设计人员填写 Drawing Description、选择 Category 并上传 PDF 文件后，AI 自动识别每页图框中的 Drawing No 和 Drawing Name
- 识别结果以**只读列表**形式展示（含页码、缩略图、Drawing No、Drawing Name），供设计人员核对
- AI 识别结果自动填入 Drawing Code / Drawing Name 输入框（可编辑），设计人员确认后提交
- **每次提交仍创建一条 Drawing 记录**（一个 PDF = 一条记录），与 REQ-003A 一致
- AI 识别期间 Drawing Code / Name 输入框及 Submit 按钮置灰
- AI 识别失败时降级：橙色提示 + 输入框恢复可编辑 + [Re-upload] 按钮

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003A** | 新建图纸上传弹窗增加 AI 识别结果列表区域和自动填入逻辑；上传新版本场景不受影响 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-003E-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-003E-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-003E-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-003E.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-003E-pc.md` |

---

## REQ-003D · 项目管理员图纸 SE 分配（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-003B-pc

### 新增内容
- 图纸列表操作列新增 [Assign] 按钮（图纸状态为 ACTIVE 时可用）
- [Assign] 弹框：展示当前分配的 SE 列表，支持添加/移除项目内 SE 成员
- 保存后，被新增的 SE 收到 App Push + 站内通知，被移除的 SE 图纸不再可见
- 分配记录可在版本历史/查阅确认记录中追溯（见 REQ-003C）

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | 将 REQ-003 的"SE 分配机制"拆解为独立子需求，操作入口和交互更明确 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-003D-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-003D-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-003D-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-003D.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-003D-pc.md` |

---

## REQ-003C · 图纸版本历史与查阅确认记录（PC端）

- **状态**: 草稿（v0.1.1，2026-05-23）
- **涉及端**: PC
- **依赖**: REQ-003A-pc、REQ-003B-pc

### 新增内容
- 项目管理人员在 PC 端查看每张图纸的完整版本历史列表及每次审批结果
- 版本历史抽屉：版本号、上传人、上传时间、审批状态、审批人、审批意见（驳回时显示）
- 查阅确认记录面板：当前有效版本中每个被分配 SE 的确认状态（已确认/未确认）、确认时间
- 版本历史抽屉标题格式：`{drawingName} — Version History`（不含 Drawing Code）
- 查阅确认面板标题格式：`{drawingName} — SE Confirmation（V{n}）`（不含 Drawing Code）
- （注：版本历史视图在 REQ-007C 中升级为 4 阶段生命周期视图）

### 变更记录
| 版本 | 日期 | 变更摘要 |
|------|------|---------|
| v0.1.1 | 2026-05-23 | 版本历史抽屉及查阅确认面板标题移除 Drawing Code，对齐 REQ-003A 列表设计 |
| v0.1.0 | 2026-05-04 | 初稿：从 REQ-003-pc 拆分 |

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | 将 REQ-003 的"版本历史与确认记录查看"拆解为独立子需求 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-003C-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-003C-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-003C-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-003C.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-003C-pc.md` |

---

## REQ-003B · 审批人图纸审批（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-003A-pc

### 新增内容
- 审批人通过右上角通知图标进入 Todo List，处理待审批图纸任务
- 详情侧滑弹框展示图纸信息（Drawing Code、版本说明、PDF 预览/下载）
- 弹框底部 [Approve] / [Reject] 按钮完成审批；驳回时理由必填
- （注：升级为两级审批后，此处为内部审批操作，通过后版本不立即生效，见 REQ-007A）

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | 将 REQ-003 的"审批操作"拆解为独立子需求 |
| **REQ-007A** | 内部审批 Todo 交互在 REQ-007A 中进一步调整，说明两级流程变化 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-003B-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-003B-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-003B-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-003B.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-003B-pc.md` |

---

## REQ-003A · 图纸上传与审批发起（PC端）

- **状态**: 草稿（v0.3.1，2026-05-23）
- **涉及端**: PC
- **依赖**: REQ-007-shared

### 新增内容
- 设计人员（Designer）在 PC 端图纸列表新建图纸或上传新版本
- 图纸主列表展示列：Description、Category、RFA No.、Subject of RFA、Current Version、Status、Confirmed、Total Markups、Last Updated、Actions；**不含 Drawing Code 列**
- 上传时选择文件（PDF）、填写 Drawing Code、版本说明，指定内部审批人
- 提交后触发两级审批流程（内部 → 外部），图纸版本进入 `PENDING_INTERNAL` 状态
- 内部审批驳回后，设计人员收到站内通知，可重新上传新版本
- 支持修改 Drawing Code（上传新版本时可选，同项目唯一性校验）
- 新建图纸流程集成 AI 自动识别（详见 REQ-003E），降级为手动填写路径

### 变更记录
| 版本 | 日期 | 变更摘要 |
|------|------|---------|
| v0.3.1 | 2026-05-23 | 明确图纸列表不含 Drawing Code 列，作为下游 REQ-003C / REQ-007C 的对齐依据 |

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-003** | 将 REQ-003 的"上传与发起审批"拆解为独立子需求，审批流程升级为两级 |
| **REQ-003C** | 列表不显示 Drawing Code，版本历史抽屉/确认面板标题同步移除 Drawing Code |
| **REQ-007C** | F-006 二级弹框标题及顶部信息区同步移除 Drawing Code |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-003A-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-003A-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-003A-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-003A.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-003A-pc.md` |

---

## REQ-003 · 工程图纸管理（版本管理、审批、查阅确认）

- **状态**: 草稿
- **涉及端**: PC（上传、审批、管理）/ APP（SE 查阅确认）
- **依赖**: REQ-001

### 新增内容
- 图纸 CRUD + 版本管理（自动版本号递增）
- 单级审批流程：上传 → 指定审批人 → 通过/驳回 → 版本生效
- Site Engineer 分配机制：管理员为图纸指定可查阅的 SE
- 查阅确认（Confirm Reading）：SE 确认已阅，记录设备信息
- 审批通过后推送已分配 SE（App Push + 站内消息）
- 数据模型：`Drawing`、`DrawingVersion`、`DrawingSeAssignment`、`DrawingReadConfirmation`

### 对已有需求的影响
- 无（基础图纸模块，首次引入）

### 文档清单
| 类型 | 文件 |
|------|------|
| 共享需求 | `requirements/shared/REQ-003-shared.md` |
| PC 端需求 | `requirements/pc/REQ-003-pc.md` |
| APP 端需求 | `requirements/app/REQ-003-app.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-003-pc.md` · `outputs/ui/app/UI-REQ-003-app.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-003-pc.md` · `outputs/frontend/app/FRONTEND-REQ-003-app.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-003.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-003-pc.md` · `outputs/qa/app/QA-REQ-003-app.md` |

---

## REQ-002 · PC 管理后台框架（Header / Sidebar / Main Content）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001

### 新增内容
- 登录后进入管理后台统一页面框架（Header + 左侧 Sidebar + Main Content 区域）
- Header：Logo、项目切换器、通知图标（Todo List 入口）、用户头像/退出
- Sidebar：按角色权限动态渲染菜单项（MAINCON 管理员 / SUBCON 管理员），支持折叠
- Main Content：路由占位区，各功能模块页面在此渲染
- 最低支持 1280×720px 桌面浏览器，不做移动端适配

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-001** | 登录成功后跳转至此框架页面 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC 端需求 | `requirements/pc/REQ-002-pc.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-002-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-002-pc.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-002-pc.md` |

---

## REQ-001 · 登录和首页功能

- **状态**: 草稿
- **涉及端**: APP / H5 / PC（全端）
- **依赖**: 无

### 新增内容
- SaaS 多租户登录（租户名 + 用户名 + 密码）
- Token 会话管理（JWT，自动刷新）
- 项目切换机制（多项目归属）
- 各端首页布局（PC Dashboard、APP 功能入口、H5 轻量首页）
- 国际化支持（中/英）
- 基础权限体系

### 对已有需求的影响
- 无（系统基础模块，所有后续需求依赖此模块的登录、权限、租户体系）

### 文档清单
| 类型 | 文件 |
|------|------|
| 共享需求 | `requirements/shared/REQ-001-shared.md` |
| PC 端需求 | `requirements/pc/REQ-001-pc.md` |
| APP 端需求 | `requirements/app/REQ-001-app.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-001-pc.md` · `outputs/ui/app/UI-REQ-001-app.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-001-pc.md` · `outputs/frontend/app/FRONTEND-REQ-001-app.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-001.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-001-pc.md` · `outputs/qa/app/QA-REQ-001-app.md` |

---

## 需求依赖关系总览

```
REQ-001 登录和首页（基础）
  ├── REQ-002 PC 管理后台框架（Header / Sidebar / Main Content）
  ├── REQ-003 工程图纸管理（核心模块）
  │     ├── REQ-003A 图纸上传与审批发起（PC端）
  │     │     ├── REQ-003E 图纸上传 AI 自动识别页信息（PC端，增强 REQ-003A 上传弹窗）
  │     │     └── REQ-003B 审批人图纸审批（PC端）
  │     │           ├── REQ-003C 图纸版本历史与查阅确认记录（PC端）
  │     │           └── REQ-003D 项目管理员图纸 SE 分配（PC端）
  │     ├── REQ-004 图纸局部更新 Markup
  │     │     └── REQ-005 SE PC 端图纸访问（扩展 REQ-003 + REQ-004）
  │     │           └── REQ-014 图纸管理用户反馈迭代（迭代 REQ-003 + REQ-004 + REQ-005）
  │     ├── REQ-006 图纸二维码与公开状态页
  │     └── REQ-007 图纸两级审批流程（扩展 REQ-003，影响 REQ-006）
  │           ├── REQ-007D DC 配置页面（PC端）
  │           ├── REQ-007A 内部审批 Todo 调整（PC端，依赖 REQ-007D）
  │           │     └── REQ-007E 内部审批人待办详情查看与原文件下载（PC端）
  │           ├── REQ-007B DC 外部审批 Todo 与标记 Dialog（PC端，依赖 REQ-007A）
  │           └── REQ-007C 版本历史抽屉 4 阶段生命周期视图（PC端，依赖 REQ-007A/B）
  └── REQ-008 公司层级工序模板配置
        └── REQ-009 项目层级工序配置副本机制
              └── REQ-010 Master Program 任务分配页面（Planning 端）
                    ├── REQ-011 PM"我的任务"页面（PM 端）
                    │     └── REQ-012 CM"我的任务"页面（CM 端）
                    │           └── REQ-013 SE"我的任务"页面（APP 端）
REQ-015 BCA 月度人力数据提交（独立模块，无依赖）
```
