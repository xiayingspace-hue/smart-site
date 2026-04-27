# Smart Site — 需求变更日志

> 按需求编号记录每个 REQ 的功能范围、涉及端、对已有需求的影响。
> 快速了解"每个需求做了什么"以及"改了什么"。

---

## REQ-013 · SE"我的任务"菜单与任务执行页面（APP端）

- **状态**: 草稿
- **涉及端**: APP
- **依赖**: REQ-001、REQ-012

### 新增内容
- APP端首页新增"Progress Management"模块，SE可见"我的任务"入口
- APP首页底部notification入口，点击后进入通知中心，顶部显示两个Tab：
  - Todo（待办）：展示所有与用户相关的待办事项，支持一键跳转
  - 消息通知：展示系统消息、任务变更、问题处理等通知
- SE专属"我的任务"页面：展示CM分配的任务列表，支持筛选、搜索
- 任务详情页：工序要求、计划/实际时间、优先级、进度填报、工单反馈、问题上报等
- 消息通知：新任务分配、变更、作废、逾期预警、问题处理反馈等

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-001** | APP首页新增Progress Management模块入口 |
| **REQ-012** | CM分配任务给SE后，SE在此页面接收并执行 |

### 文档清单
| 类型 | 文件 |
|------|------|
| APP端需求 | `requirements/app/REQ-013-app.md` |
| UI设计 | `outputs/ui/app/UI-REQ-013-app.md` |

---

## REQ-012 · CM"我的任务"菜单与任务执行页面（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001、REQ-010、REQ-011

### 新增内容
- PC端Progress Management新增"我的任务（CM）"菜单，仅CM可见
- 任务列表页：展示PM分配给CM的所有任务，支持筛选、搜索、分页，与PM端布局一致
- 新建任务弹框：支持引用工序模板或自定义新建，自动关联上级任务
- 任务拆解：CM可将任务分配/分解给SE，弹框支持批量新建
- 问题汇报：操作列及任务详情页均有"汇报问题"入口，提交后通知PM
- 消息通知：SE分配通知、任务变更、作废、进度反馈、逾期预警等全场景通知

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-010** | 新增CM专属菜单入口 |
| **REQ-011** | PM分配任务给CM后，CM在此页面接收并分解任务 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC端需求 | `requirements/pc/REQ-012-pc.md` |
| UI设计 | `outputs/ui/pc/UI-REQ-012-pc.md` |
| 前端开发 | （待生成） |
| 后端开发 | （待生成） |
| QA测试 | （待生成） |

---

## REQ-011 · PM"我的任务"菜单与任务执行页面（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001、REQ-010

### 新增内容
- PC端Progress Management新增"我的任务（PM）"菜单，仅PM可见
- 任务列表页：展示Planning分配给PM的所有任务，布局与Planning端Master Program一致
- 新任务高亮显示，顶部显示新任务数量提示，支持一键筛选
- 新建任务弹框：从右侧侧滑，需关联Planning主计划任务，支持批量新建
- 任务分配/分解给CM，弹框支持批量操作
- 问题汇报：操作列及任务详情页均有"汇报问题"入口，提交后通知Planning
- Planning可在消息中心和Master Program任务详情页"问题汇报"tab查看处理
- 分级进度填报：SE填报→CM审核→PM查看，PM无需手动填报
- 消息通知：覆盖分配、变更、作废、进度反馈、问题汇报、逾期预警、退回、审批、评论、完成确认、分解合并等全场景

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-010** | 新增PM专属菜单入口 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC端需求 | `requirements/pc/REQ-011-pc.md` |
| UI设计 | `outputs/ui/pc/UI-REQ-011-pc.md` |
| 前端开发 | （待生成） |
| 后端开发 | （待生成） |
| QA测试 | （待生成） |

---

## REQ-010 · 项目进度管理菜单与任务分配页面布局（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001、REQ-008、REQ-009

### 新增内容
- PC端Progress Management新增"Master Program"菜单（Planning专用）
- Master Program任务表格：Activity ID、Activity Name、WBS、计划/实际时间、Deviation、Status、Assigned PM、操作等
- 新增/分配任务弹框：从右侧侧滑，样式与系统风格一致
- 任务分配（Planning→PM）：分配按钮、批量分配
- 任务编辑/作废：操作列支持编辑、作废，含权限控制和二次确认
- 任务状态管理：未分配/已分配/进行中/已完成/已延期/作废，颜色高亮
- Deviation字段：自动计算计划与实际偏差，颜色区分延期/提前/如期
- 消息通知：分配/变更/作废/进度/异常等场景全覆盖，含消息内容模板

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-008** | 工序模板在任务新建/拆解时被引用 |
| **REQ-009** | 项目级模板数据作为任务拆解的基础 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC端需求 | `requirements/pc/REQ-010-pc.md` |
| UI设计 | `outputs/ui/pc/UI-REQ-010-pc.md` |
| 前端开发 | （待生成） |
| 后端开发 | （待生成） |
| QA测试 | （待生成） |

---

## REQ-009 · 项目层级字典与工序配置副本机制（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-008

### 新增内容
- 项目管理后台Progress Management下新增Process Settings子菜单
- 支持从公司层级一键复制字典配置和工序模板到项目层级
- 项目层级拥有独立的字典和工序配置副本，可自由增删改，不影响公司层级和其他项目
- 复制操作前弹窗确认，操作日志记录
- 页面结构与公司层级一致（专业Tab、构件类型、工序表格三级联动）

### 对已有需求的影响
| 需求 | 影响 |
|------|------|
| **REQ-008** | 项目级模板数据来源于公司级，复制后独立维护 |

### 文档清单
| 类型 | 文件 |
|------|------|
| PC端需求 | `requirements/pc/REQ-009-pc.md` |
| UI设计 | `outputs/ui/pc/UI-REQ-009-pc.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-009-pc.md` |
| 后端开发 | `outputs/backend/pc/BACKEND-REQ-009-pc.md` |
| QA测试 | `outputs/qa/pc/QA-REQ-009-pc.md` |

---

## REQ-008 · 公司层级工序模板配置（PC端）

- **状态**: 草稿
- **涉及端**: PC
- **依赖**: REQ-001

### 新增内容
- Company Insights视角下，左侧导航Project Progress > Process Settings页面
- 专业Tab栏（来自公司字典，仅展示，不可增删）
- 左侧构件类型列表（来自字典，仅展示）
- 右侧工序表格：支持新增、编辑、删除、禁用/启用、批量添加、排序
- 工序权重字段，用于后续进度汇总计算
- 删除/禁用逻辑：引用状态校验，已用于实际任务仅可禁用，不可物理删除
- 专业Tab、构件类型、工序三级联动

### 对已有需求的影响
- 无（首次引入工序模板模块）

### 文档清单
| 类型 | 文件 |
|------|------|
| PC端需求 | `requirements/pc/REQ-008-pc.md` |
| UI设计 | `outputs/ui/pc/UI-REQ-008-pc.md` |
| 前端开发 | `outputs/frontend/FRONTEND-REQ-008-pc.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-008-pc.md` |
| QA测试 | `outputs/qa/QA-REQ-008-pc.md` |

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

## REQ-002 · 设备加油管理

- **状态**: 草稿
- **涉及端**: APP（数据录入）/ PC（数据查看与管理）
- **依赖**: REQ-001

### 新增内容
- APP 端加油信息录入（设备选择、油品、加油量、里程/工时、照片）
- PC 管理端加油记录列表、搜索、导出
- 加油数据统计（按设备、按时间段）
- 数据模型：`FuelRecord`、设备关联

### 对已有需求的影响
- 无（独立业务模块）

### 文档清单
| 类型 | 文件 |
|------|------|
| 共享需求 | `requirements/shared/REQ-002-shared.md` |
| PC 端需求 | `requirements/pc/REQ-002-pc.md` |
| APP 端需求 | `requirements/app/REQ-002-app.md` |
| UI 设计 | `outputs/ui/pc/UI-REQ-002-pc.md` · `outputs/ui/app/UI-REQ-002-app.md` |
| 前端开发 | `outputs/frontend/pc/FRONTEND-REQ-002-pc.md` · `outputs/frontend/app/FRONTEND-REQ-002-app.md` |
| 后端开发 | `outputs/backend/BACKEND-REQ-002.md` |
| QA 测试 | `outputs/qa/pc/QA-REQ-002-pc.md` · `outputs/qa/app/QA-REQ-002-app.md` |

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
  ├── REQ-002 设备加油管理（独立模块）
  ├── REQ-003 工程图纸管理（核心模块）
  │     ├── REQ-004 图纸局部更新 Markup
  │     │     └── REQ-005 SE PC 端图纸访问（扩展 REQ-003 + REQ-004）
  │     ├── REQ-006 图纸二维码与公开状态页
  │     └── REQ-007 图纸两级审批流程（扩展 REQ-003，影响 REQ-006）
  └── REQ-008 公司层级工序模板配置
        └── REQ-009 项目层级工序配置副本机制
              └── REQ-010 Master Program任务分配页面（Planning端）
                    ├── REQ-011 PM"我的任务"页面（PM端）
                    │     └── REQ-012 CM"我的任务"页面（CM端）
                    │           └── REQ-013 SE"我的任务"页面（APP端）
```
