# PC 管理端 — 工程图纸管理

> **端**: PC 管理端（桌面浏览器 Web 管理后台）
> **共享需求**: [REQ-003-shared.md](../shared/REQ-003-shared.md)（业务规则、数据模型、API 接口、审批流程）
> **本文档仅包含**: PC 端特有的页面布局、交互方式、上传流程、审批 Todo、版本历史与确认记录查看

## 基本信息

- **需求ID**: REQ-003-pc
- **需求标题**: PC 端工程图纸上传、审批与管理
- **产品**: SMART SI### 6. 版本历史抽屉

#### 6.1 触发方式SYSTEM
- **平台**: PC 管理端（桌面浏览器）
- **技术框架**: Vue 2 + Element UI
- **优先级**: 高
- **状态**: 草稿

---

## 需求描述

### 背景与目标

PC 管理端是图纸管理的主要操作端，面向 Drawing 团队、审批人员和项目管理人员。主要职责：
- Drawing 团队在此上传新版图纸并发起审批
- 审批人员在 Todo 列表中处理待审批图纸
- 项目管理人员查看图纸版本历史、审批记录和 Site Engineer 的查阅确认情况
- 审批驳回后，设计人员在此收到站内消息通知并重新上传

### 用户故事

```
作为 Drawing 团队成员
我想要 在 PC 端上传新版图纸并指定审批人
以便 图纸能进入正式审批流程，审批通过后自动对现场人员发布
```

```
作为 审批人员
我想要 在 PC 端 Todo 列表中快速看到待审批的图纸并完成审批
以便 不遗漏审批任务，保障图纸管理流程的顺畅运转
```

```
作为 项目管理人员
我想要 在 PC 端查看每张图纸的版本历史、审批记录和 Site Engineer 查阅确认情况
以便 对图纸管理全流程保持可见性，及时跟进未确认人员
```

```
作为 项目管理人员
我想要 在 PC 端为每张图纸指定需要查阅的 Site Engineer
以便 确保只有与该图纸相关的现场工程师收到通知并看到图纸，减少信息干扰
```

---

## 功能需求

### 1. 入口位置

- **侧边栏菜单**: Drawing Management（图纸管理）
- 作为独立一级菜单项，图标为图纸/文件夹图标

### 2. 图纸列表页

#### 2.1 页面布局

```
┌─────────────────────────────────────────────────────────────────┐
│ 📄 Drawing Management    [Search]            [+ Upload Drawing] │
├─────────────────────────────────────────────────────────────────┤
│ ▼ 搜索面板（点击 Search 后展开，再次点击收起）                       │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Code:     [_______________]  Name:     [_______________]    │ │
│ │ Category: [________▼]        Status:   [________▼]         │ │
│ │                                        [Cancel]  [Search]  │ │
│ └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ Drawing Code │ Name │ Category │ Version │ Status │ Confirmed │ Updated │ Actions │
│─────────────────────────────────────────────────────────────────│
│ ARCH-001 │ 首层平面图 │ Architectural │ V3 │ 🟢 Active │ 5/12 │ 2026-04-01 │ [View] [Upload New Ver] [History] │
│ ARCH-002 │ 立面图    │ Architectural │ V1 │ 🟡 Pending │ —   │ 2026-04-02 │ [View] [History] │
│ STRU-001 │ 基础结构图│ Structural    │ V2 │ 🔴 Rejected│ —   │ 2026-03-28 │ [View] [Upload New Ver] [History] │
│ ...                                                             │
├─────────────────────────────────────────────────────────────────┤
│ Total: 45                              [< 1  2  3  4  5 >]     │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.2 筛选条件

> **交互说明**：顶部操作栏左侧放置 **[Search]** 按钮，右侧放置 **[+ Upload Drawing]**。点击 [Search] 在按钮正下方展开搜索面板，再次点击或点击 [Cancel] 收起面板，收起时保留已生效的筛选状态。

| 字段 | 类型 | 说明 |
|------|------|------|
| Code | 文本搜索 | 图纸编号精确或模糊搜索 |
| Name | 文本搜索 | 图纸名称模糊搜索 |
| Category | 下拉单选 | 按图纸分类筛选（全部 / Architectural / Structural / …） |
| Status | 下拉单选 | 按状态筛选（全部 / Active / Pending Approval / Rejected） |

搜索面板底部操作按钮：
- **[Search]**：以当前面板条件触发查询，列表刷新，面板保持展开
- **[Cancel]**：清空面板内所有输入并收起面板，列表恢复无筛选状态

#### 2.3 表格列定义

| 列名 | 说明 | 排序 |
|------|------|------|
| Drawing Code | 图纸编号 | ✅ |
| Name | 图纸名称 | — |
| Category | 分类 | — |
| Current Version | 当前版本号（如 V3） | — |
| Status | 状态标签（Active 绿 / Pending 黄 / Rejected 红） | — |
| Confirmed | 已确认/总人数（如 5/12），Pending/Rejected 时显示 — | — |
| Last Updated | 最后更新时间 | ✅ |
| Actions | 操作按钮组 | — |

#### 2.4 操作按钮逻辑

| 按钮 | 显示条件 | 权限 | 说明 |
|------|---------|------|------|
| View | 始终显示 | `drawing:view` | 在新标签页或抽屉中在线预览当前版本 |
| Upload New Version | Status ≠ PENDING_APPROVAL | `drawing:upload` | 上传新版本，Pending 时置灰 |
| History | 始终显示 | `drawing:history` | 打开版本历史抽屉 |
| Assign | 始终显示 | `drawing:assign` | 打开分配 Site Engineer 弹窗 |

---

### 3. 上传图纸（新建 / 新版本）

#### 3.1 触发方式

- 列表页右上角 **[+ Upload Drawing]** 按钮：上传全新图纸（新建）
- 列表行 **[Upload New Version]** 按钮：为已有图纸上传新版本

#### 3.2 上传弹窗（新建图纸）

```
┌──────────────────────────────────────┐
│ Upload New Drawing           [✕]     │
│                                      │
│ Drawing Code *  [________________]   │
│ Drawing Name *  [________________]   │
│ Category *      [________▼]          │
│ Description     [________________]   │
│                                      │
│ Drawing File *                       │
│ ┌──────────────────────────────────┐ │
│ │   📂  Drag file here or          │ │
│ │       Click to upload            │ │
│ │   PDF / DWG / DXF / PNG / JPG    │ │
│ │   Max 50MB                       │ │
│ └──────────────────────────────────┘ │
│                                      │
│ Version Note    [________________]   │
│ Approver *      [________▼]          │
│                                      │
│          [Cancel]  [Submit]          │
└──────────────────────────────────────┘
```

#### 3.3 上传弹窗（新版本）

与新建图纸弹窗相比：
- 标题改为 **"Upload New Version — {drawingCode} {drawingName}"**
- 隐藏 Drawing Code、Drawing Name、Category、Description 字段（继承自图纸主记录）
- 显示当前版本号（只读提示：Current Version: V2，新版本将为 V3）
- 其余字段相同（Drawing File、Version Note、Approver）

#### 3.4 上传交互

1. 文件选择后，显示文件名、文件大小预览
2. 点击 Submit 时前端校验必填项（Drawing File、Approver）
3. 显示上传进度条
4. 上传成功后：弹窗关闭，列表刷新，Snackbar 提示 "Drawing uploaded successfully. Pending approval."
5. 审批人的 Todo 列表自动出现新的待审批任务

---

### 4. 审批 Todo（PC 端）

#### 4.1 入口

- **顶部导航栏** Todo 图标（或通知角标），点击展开 Todo 下拉面板
- 或在侧边栏 Todo 模块中作为独立页面展示（与其他模块 Todo 合并）

#### 4.2 待审批图纸 Todo 卡片

```
┌─────────────────────────────────────────────────────┐
│ 📋 Drawing Approval Required                        │
│                                                     │
│ ARCH-001  首层平面图  V2                             │
│ Uploaded by: 李四  |  2026-04-01 10:00              │
│ Version Note: 修正轴网尺寸                           │
│                                                     │
│ [View Drawing]   [Approve]   [Reject]               │
└─────────────────────────────────────────────────────┘
```

#### 4.3 审批操作

**通过（Approve）**：
- 点击 [Approve] 按钮，弹出确认对话框：
  ```
  Confirm Approval?
  Once approved, this version will become the current active version,
  and all previous versions will be deprecated.
  [Cancel]  [Confirm]
  ```
- 确认后调用审批 API，列表刷新，Todo 消失，显示成功提示

**驳回（Reject）**：
- 点击 [Reject] 按钮，弹出驳回意见输入对话框：
  ```
  Reject Drawing Version
  Comment (Required): [________________________________]
                      [________________________________]
  [Cancel]  [Confirm Rejection]
  ```
- Comment 为必填，空白时 [Confirm Rejection] 按钮禁用
- 确认后调用审批 API，上传人收到站内消息通知

---

### 5. 分配 Site Engineer

#### 5.1 触发方式

点击图纸列表行的 **[Assign]** 按钮，弹出分配管理弹窗。

#### 5.2 分配弹窗

```
┌──────────────────────────────────────────────────┐
│ Assign Site Engineers — ARCH-001 首层平面图  [✕] │
├──────────────────────────────────────────────────┤
│ Search: [__________________________]             │
│                                                  │
│ Assigned (3)                                     │
│ ┌────────────────────────────────────────────┐   │
│ │  ✅ Wang Wei         Site Engineer  [Remove]│  │
│ │  ✅ Chen Ming        Site Engineer  [Remove]│  │
│ │  ✅ Li Hua           Site Engineer  [Remove]│  │
│ └────────────────────────────────────────────┘   │
│                                                  │
│ Not Assigned (9)                                 │
│ ┌────────────────────────────────────────────┐   │
│ │  ○  Zhang San        Site Engineer  [Add]  │   │
│ │  ○  Liu Yang         Site Engineer  [Add]  │   │
│ │  ○  Zhao Wei         Site Engineer  [Add]  │   │
│ │  ...                                       │   │
│ └────────────────────────────────────────────┘   │
│                                                  │
│                        [Cancel]  [Save]          │
└──────────────────────────────────────────────────┘
```

#### 5.3 交互逻辑

1. 弹窗打开时，调用 `/drawing/assign/list` 获取当前已分配和未分配列表
2. 已分配人员分组在上方，未分配人员分组在下方
3. 支持顶部搜索框，按姓名模糊过滤两组列表
4. 点击 **[Add]** 将该用户移入已分配分组（仅前端暂存，未提交）
5. 点击 **[Remove]** 将该用户移至未分配分组（仅前端暂存，未提交）
6. 点击 **[Save]**：收集已分配分组中所有用户的 ID，调用 `/drawing/assign`（全量覆盖），成功后弹窗关闭，提示 "Assignment updated successfully"
7. 若当前已无任何分配（已分配分组为空）且用户点击 Save，弹出二次确认："No Site Engineers assigned. This drawing will not be visible to anyone. Confirm?"
8. 点击 **[Cancel]** 或遮罩关闭弹窗，本次修改不保存

#### 5.4 字段与业务说明

| 字段 | 说明 |
|------|------|
| Assigned 列表 | 已分配该图纸的 Site Engineer，可点击 [Remove] 移除 |
| Not Assigned 列表 | 项目内拥有 `drawing:view` 权限但尚未分配该图纸的 Site Engineer，可点击 [Add] 添加 |
| Search | 实时过滤两组列表中的人员，不触发接口请求 |
| Save | 全量覆盖提交，以弹窗内当前已分配列表为准 |

> **注意**：新建图纸时默认无分配，管理员需手动操作分配弹窗才能让 Site Engineer 看到该图纸。建议在上传图纸弹窗的成功提示中引导管理员进行分配："Drawing submitted. Remember to assign Site Engineers so they can view this drawing."

---

### 6. 版本历史抽屉

#### 5.1 触发方式

点击列表行 **[History]** 按钮，从右侧滑入抽屉面板。

#### 6.2 抽屉内容

```
┌──────────────────────────────────────────┐
│ Version History                   [✕]   │
│ ARCH-001  首层平面图                      │
├──────────────────────────────────────────┤
│ V3  ● Current                            │
│ Uploaded: 李四  2026-04-01 10:00         │
│ Approved: 2026-04-01 14:30               │
│ Note: 修正轴网尺寸                        │
│ [View File]                              │
├──────────────────────────────────────────┤
│ V2  ✓ Deprecated                         │
│ Uploaded: 李四  2026-03-20 09:00         │
│ Approved: 2026-03-20 16:00               │
│ Note: 初版提交                            │
│ [View File]                              │
├──────────────────────────────────────────┤
│ V1  ✕ Rejected                           │
│ Uploaded: 李四  2026-03-15 14:00         │
│ Rejected: 2026-03-16 09:00               │
│ Rejection Comment: 图纸尺寸标注不完整     │
│ [View File]                              │
└──────────────────────────────────────────┘
```

版本状态标识：
- `● Current`：当前有效版本（蓝色）
- `✓ Deprecated`：已被新版本替代（灰色）
- `⏳ Pending`：待审批（黄色）
- `✕ Rejected`：已驳回（红色）

---

### 7. 查阅确认记录面板

#### 7.1 触发方式

在图纸详情页或版本历史抽屉中，点击 **"Confirmed: 5/12"** 链接，展开确认记录面板。

#### 7.2 确认记录面板

```
┌─────────────────────────────────────────────────────┐
│ Reading Confirmation — ARCH-001 V3          [✕]    │
│ Confirmed: 5 / 12 Site Engineers                    │
├─────────────────────────────────────────────────────┤
│ ✅ Confirmed (5)                                    │
│   Wang Wei        2026-04-01 16:20                  │
│   Chen Ming       2026-04-01 17:05                  │
│   Li Hua          2026-04-02 08:30                  │
│   Zhang Lei       2026-04-02 09:10                  │
│   Xu Fang         2026-04-02 10:45                  │
├─────────────────────────────────────────────────────┤
│ ⏳ Not Yet Confirmed (7)                            │
│   Liu Yang                                          │
│   Zhao Wei                                          │
│   Sun Qi                                            │
│   ...                                               │
└─────────────────────────────────────────────────────┘
```

---

### 8. 站内消息通知（审批驳回时）

设计人员（图纸上传人）在 PC 端顶部通知铃铛处接收站内消息：

| 字段 | 内容 |
|------|------|
| 通知标题 | Drawing Rejected |
| 通知内容 | Your drawing {drawingCode} {drawingName} V{n} was rejected. Comment: {comment} |
| 操作 | 点击通知跳转至该图纸的上传新版本弹窗（快速重新上传） |

---

## 验收标准

### 图纸列表
- [ ] 列表默认按 Last Updated 倒序展示
- [ ] 点击顶部 [Search] 按钮展开搜索面板，再次点击或点击 [Cancel] 收起面板
- [ ] Code、Name、Category、Status 筛选条件正确过滤结果，支持组合筛选
- [ ] [Cancel] 清空所有筛选条件并收起面板，列表恢复无筛选状态
- [ ] Status 标签颜色正确（Active 绿 / Pending 黄 / Rejected 红）
- [ ] Confirmed 列在 Active 状态时显示 x/y，其他状态显示 —
- [ ] Pending 状态时 [Upload New Version] 按钮置灰不可点击

### 上传功能
- [ ] 文件格式不符合时显示错误提示并阻止上传
- [ ] 文件大小超过 50MB 时显示错误提示
- [ ] 新建图纸时 Drawing Code 在项目内唯一校验
- [ ] 上传成功后列表刷新，状态显示为 Pending Approval
- [ ] 进度条正确显示上传进度

### 审批 Todo
- [ ] 上传触发后，审批人的 Todo 列表立即出现待审批任务
- [ ] [View Drawing] 可在线预览待审批的图纸文件
- [ ] [Approve] 点击后弹出确认对话框，确认后执行通过逻辑
- [ ] [Reject] 点击后弹出意见输入框，Comment 为空时 [Confirm Rejection] 禁用
- [ ] 审批完成后 Todo 任务消失，列表刷新

### 版本历史
- [ ] 版本按版本号倒序排列（最新在上）
- [ ] 当前版本标记 Current，已作废版本标记 Deprecated
- [ ] 驳回版本显示驳回意见
- [ ] [View File] 可在线预览该版本的文件

### 确认记录
- [ ] 已确认和未确认人员分组展示
- [ ] 确认人数与未确认人数之和等于**被分配该图纸**的 Site Engineer 总人数（非项目全体）

### 分配 Site Engineer
- [ ] 点击 [Assign] 打开分配弹窗，正确展示已分配和未分配两组列表
- [ ] [Add] / [Remove] 操作在弹窗内暂存，不立即提交
- [ ] 点击 [Save] 调用 `/drawing/assign` 接口，全量覆盖提交，成功后提示并关闭弹窗
- [ ] 已分配列表为空时点击 [Save]，弹出二次确认提示
- [ ] 取消分配后，对应 Site Engineer 在 APP 端图纸列表立即看不到该图纸
- [ ] 上传成功提示引导管理员进行分配操作

### 站内消息
- [ ] 审批驳回后，上传人收到站内消息通知
- [ ] 通知内容包含图纸编号、名称、版本号和驳回意见
- [ ] 点击通知可跳转至该图纸的上传新版本入口

---

## 相关需求

- [REQ-003-shared.md](../shared/REQ-003-shared.md) — 跨端共享业务规则与 API
- [REQ-003-app.md](../app/REQ-003-app.md) — APP 端查阅确认与审批 Todo
