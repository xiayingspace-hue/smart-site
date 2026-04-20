# Smart Site — 需求变更日志

> 按需求编号记录每个 REQ 的功能范围、涉及端、对已有需求的影响。
> 快速了解"每个需求做了什么"以及"改了什么"。

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
  └── REQ-003 工程图纸管理（核心模块）
        ├── REQ-004 图纸局部更新 Markup
        │     └── REQ-005 SE PC 端图纸访问（扩展 REQ-003 + REQ-004）
        ├── REQ-006 图纸二维码与公开状态页
        └── REQ-007 图纸两级审批流程（扩展 REQ-003，影响 REQ-006）
```
