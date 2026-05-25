---
doc_type: ui_spec
req_id: REQ-007C-pc
version: 0.2.5
status: draft
generated_from: REQ-007C-pc@0.2.5
generated_at: 2026-05-25
owner: ""
---

# UI 设计说明：PC 端 — 图纸版本历史抽屉

> **本文档供 UI 设计师及其 agent 使用，产出视觉稿与交互稿**。
>
> - 输入：REQ-007C-pc.md（主）、REQ-007-shared（业务规则）
> - 平台：PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> - 设计令牌参考：UI-REQ-001-pc § 3.2
> - 不重复定义数据字段（参见 data-contract.md）

---

## 0. 溯源块（Traceability）

| 项 | 值 |
|---|---|
| 来源需求 | REQ-007C-pc @ v0.2.5 |
| 覆盖用户故事 | US-007C-001 |
| 覆盖 AC | AC-007C-001 ~ AC-007C-018 |
| 上次同步时间 | 2026-05-25 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

管理人员可在版本历史抽屉中查看图纸每个版本的完整审批状态演进，并通过 Actions 列 [Details] 按钮访问附件与局部更新（Part Print）信息。

### 1.2 设计原则

1. **层叠抽屉**：Details 二级抽屉叠加在版本历史抽屉之上，空间利用高效。
2. **4 阶段可视化**：通过 2×2 卡片 + 步骤条，一眼看出当前版本在审批流程中所处的阶段。
3. **状态驱动渲染**：4 个卡片的内容完全由 `approvalStatus` 驱动，设计稿需覆盖全部 5 种状态组合。
4. **操作与展示解耦**：[Details] 触发二级抽屉与版本行展开/折叠完全独立，互不影响。

---

## 2. 信息架构

### 2.1 入口

图纸列表页任意图纸行 → [History] 按钮 → 版本历史抽屉

### 2.2 层级

```
图纸列表页
└── 版本历史抽屉（720px）
    ├── 版本行（可展开 → 4 阶段卡片 + 步骤条）
    └── Actions 列 [Details] 按钮点击 → Details 二级抽屉（480px，叠加）
        ├── 顶部版本信息区（Status / Ver / Description / Submission / 主文件行）
        ├── Part Print Tab（只读 Markup 列表）
        └── Attachments Tab（附件列表，上传人可上传/删除）
```

---

## 3. 页面 / 组件清单

### 3.1 版本历史抽屉（主列表）

**关联 Story**：US-007C-001
**关联 AC**：AC-007C-001、AC-007C-002、AC-007C-011

#### 规格

| 属性 | 值 |
|------|----|
| 宽度 | **720px** |
| 位置 | 从右侧滑入，叠加于图纸列表页之上 |
| 标题格式 | `Version History — {drawingCode} {drawingName}` |
| 关闭方式 | 右上角 [✕] 或点击遮罩 |

#### 主列表布局

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│  Version History — ARCH-001 首层平面图                                              [✕]  │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│  Version  │ RFA No.       │ Status              │ Designer  │ Confirmed │ QR      │ Actions     │
│───────────│───────────────│─────────────────────│───────────│───────────│─────────│─────────────│
│  ▶ V3     │ BEN-2026-001  │ ✅ Approved         │ 张三      │ 5/12      │ [View]  │ [Details]   │
│  ▶ V2     │ BEN-2026-010  │ ⏳ Pending External │ 李四      │ —         │ —       │ [Details]   │
│  ▶ V1     │ —             │ ❌ Int. Rejected    │ 张三      │ —         │ —       │ [Details]   │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 列定义

| 列 | 内容 | 宽度建议 | 说明 |
|----|------|---------|------|
| Version | `V1 / V2 / V3 …`，左侧有 ▶ 展开箭头 | 80px | 点击展开/折叠 4 阶段卡片 |
| RFA No. | 外部审批报审编号（`DrawingApproval.submissionRefNo`，`phase=EXTERNAL`） | 140px | 外部审批尚未发起时显示 `—` |
| Status | 状态标签（5 态，颜色见 §4.1） | 160px | — |
| Designer | 设计人员（上传人）姓名 | 90px | — |
| Confirmed | `x/y`（已确认/总人数） | 90px | 仅 `APPROVED` 版本显示；其他状态显示 `—` |
| QR | `[View]` 链接 | 70px | 仅 `APPROVED` 且 QR 已生成时显示；其他显示 `—` |
| Actions | 操作按钮入口 | 100px | [Details] 按钮：点击打开 Details 二级抽屉（§3.3）；与版本行展开/折叠无关 |

#### Actions 列按钮规格

| 按钮 | Element UI 样式 | 图标 | 点击行为 | 可见角色 |
|------|---------------|------|---------|---------|
| [Details] | `el-button size=mini type=default` | `el-icon-document` | 打开 Details 二级抽屉（§3.3） | 管理员 / Drawing 团队 / DC / 内部审批人 / 设计人员 |

#### 主列表状态（5 态）

| 状态 | 视觉表现 |
|-----|---------|
| 空 | 空状态插图，"No version history available." |
| 加载 | 列表 Skeleton |
| 正常 | 主列表 + 可展开行 |
| 错误 | 错误提示，"加载失败，请刷新重试" |
| 极端数据 | 版本数 > 20 时分页，避免一次性渲染过多 4 阶段卡片 |

---

### 3.2 4 阶段展开卡片

**关联 AC**：AC-007C-003 ~ 010

#### 卡片网格规格

| 属性 | 值 |
|------|----|
| 布局 | 2×2 网格，4 张卡片等高 |
| 卡片间距 | 12px |
| 每卡片内边距 | 16px |

#### 各 approvalStatus 渲染规则

| 卡片 | PENDING_INTERNAL | INTERNAL_APPROVED | INTERNAL_REJECTED | APPROVED | EXTERNAL_REJECTED |
|------|:----------------:|:-----------------:|:-----------------:|:--------:|:-----------------:|
| ① Uploaded | ✓ 完成 | ✓ 完成 | ✓ 完成 | ✓ 完成 | ✓ 完成 |
| ② Internal | ⏳ 进行中 | ✓ 完成 | ✕ 驳回 | ✓ 完成 | ✓ 完成 |
| ③ External | — 等待 | ⏳ 进行中 | — 未到达 | ✓ 完成 | ✕ 驳回 |
| ④ Signed | — 等待 | — 等待 | — 未到达 | ✓ 完成 | — 未到达 |

#### ① 上传卡片（始终显示）

```
┌───────────────────────────────────┐
│ ① Uploaded                        │
│ 📄 {fileName}                     │
│ Designer: {designerName}          │
│ {uploadTime}                      │
│ Size: {fileSize}                  │
│               [Preview] [Download]│
└───────────────────────────────────┘
```

#### ② 内部审批卡片

| approvalStatus | 显示内容 |
|---------------|---------|
| PENDING_INTERNAL | ⏳ Pending · Approver: {approverName} · Assigned: {date} |
| INTERNAL_APPROVED / APPROVED / EXTERNAL_REJECTED | ✅ Approved · Approver: {approverName} · {approvedTime} · Comment: {comment}（有则显示） |
| INTERNAL_REJECTED | ❌ Rejected · Approver: {approverName} · {rejectedTime} · Comment: {comment} |

#### ③ 外部审批卡片

| approvalStatus | 显示内容 |
|---------------|---------|
| PENDING_INTERNAL | — Waiting for internal approval |
| INTERNAL_APPROVED（DC 用户） | ⏳ Pending · "Waiting for DC to submit external approval" · **[✅ Mark Result]** |
| INTERNAL_APPROVED（非 DC 用户） | ⏳ Pending · "Waiting for DC to submit external approval"（无 Mark Result 按钮） |
| APPROVED（Status A/B/D） | ✅ {statusLabel} · Marked by: {dcName} · Approval Date: {date} · Submission Ref No.: {ref} · 📎 Evidence [Download] · Remark: {remark}（有则显示） |
| EXTERNAL_REJECTED（Status C/E） | ❌ {statusLabel} · Marked by: {dcName} · {time} · Submission Ref No.: {ref} · Reason/Remarks: {comment}（无 Evidence [Download]） |
| INTERNAL_REJECTED | — Not reached（Internal approval failed） |

> **Status of Approval 标签对照**（来源 REQ-007B-pc）：
>
> | DrawingApproval.status | 图标 | 显示文案 |
> |----------------------|------|---------|
> | A | ✅ | A – Approved / No Exception Taken |
> | B | ✅ | B – Approved with comment, resubmission required |
> | D | ✅ | D – For Record Purpose |
> | C | ❌ | C – Revise And Resubmit |
> | E | ❌ | E – Others: {othersReason} |

#### ④ 签字版卡片

| approvalStatus | 显示内容 |
|---------------|---------|
| APPROVED | �� {signedFileName} · ✍️ Signed Drawing · Uploaded: {date} · Size: {size} · [Preview] [Download]（优先 `pdfWithQrUrl` > `signedFileUrl`） |
| INTERNAL_APPROVED | — Waiting for external approval |
| 驳回类 | — Not reached |
| PENDING_INTERNAL | — Waiting for internal approval |

> **AC-007C-009**：[Download] 文件 URL 优先使用 `pdfWithQrUrl`；
> **AC-007C-010**：`pdfWithQrUrl` 为 null 时降级使用 `signedFileUrl`。

#### 底部步骤条规范

```
📄 Uploaded ✓  →  🔍 Internal [✓/⏳/✕/—]  →  🌐 External [✓/⏳/✕/—]  →  ✍️ Signed [✓/—]
```

| 步骤状态 | 颜色 | 符号 |
|---------|------|------|
| 完成 | 绿色 `#67C23A` | ✓ |
| 进行中 | 橙色 `#E6A23C` | ⏳ |
| 驳回/失败 | 红色 `#F56C6C` | ✕ |
| 未到达 | 灰色 `#C0C4CC` | — |

---

### 3.3 Details 二级抽屉

**关联 AC**：AC-007C-011 ~ 018

#### 规格

| 属性 | 值 |
|------|----|
| 宽度 | **480px** |
| 位置 | 从右侧滑入，**叠加在版本历史抽屉（720px）之上** |
| 触发方式 | 点击主列表任意版本行 Actions 列的 **[Details] 按钮**（与版本行展开/折叠无关） |
| 关闭方式 | 右上角 [✕] |
| 关闭后 | 返回版本历史抽屉，版本历史抽屉保持原状 |

#### 布局

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Version History — · 首层平面图  V3                                 [✕]  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Status              ✅ Active                                           │
│  Ver (System)        V3                                                  │
│  Description         包含外墙及核心筒轮廓线                              │
│  Submission Ref No.  BEN-2026-001                                        │
│  Submission Subject  首层平面图 Rev.3 外部审批                           │
│  Uploaded by         👤 张三                                             │
│  Upload Date         2026-04-01 10:00                                    │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ 📄 arch001-v3.pdf                               [Download]       │    │
│  │ ✅ Approved by 王总工 · 2026-04-02 14:30                          │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌──────────────┬──────────────────────────────────────────────────┐    │
│  │ Part Print(2)│ Attachments (3)                                  │    │
│  └──────────────┴──────────────────────────────────────────────────┘    │
│                                                                          │
│  ── Part Print Tab 激活时 ─────────────────────────────────────────      │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ [Active]  A轴节点详图修正                                         │    │
│  │ Apr 8, 2026 · 张三                                               │    │
│  │ A-C轴 / 3-5层                                                   │    │
│  │ A轴与3轴交叉节点详图已更新，新增钢筋排布说明...                   │    │
│  │ 📎 node-detail.pdf                                               │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ [Merged → V4]  外墙保温层厚度修正                                 │    │
│  │ Apr 5, 2026 · 张三                                               │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│  （无局部更新时显示 "No Data"）                                           │
│                                                                          │
│  ── Attachments Tab 激活时 ─────────────────────────────────────────     │
│  Attachments (3)                                              [+]        │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ Filename              │ Type  │ Size    │ Uploaded    │ Uploaded by│  │
│  │───────────────────────│───────│─────────│─────────────│────────────│  │
│  │ 结构计算书.xlsx        │ xlsx  │ 1.2 MB  │ 2026-04-01  │ 张三 [Del] │  │
│  │ 施工说明.docx          │ docx  │ 0.5 MB  │ 2026-04-01  │ 张三 [Del] │  │
│  │ arch001-v3.dwg         │ dwg   │ 8.3 MB  │ 2026-04-01  │ 张三 [Del] │  │
│  └──────────────────────────────────────────────────────────────────┘    │
│  （无附件时表格内显示 "No Data"）                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 版本信息区字段

| 字段 | 内容 | 说明 |
|------|------|------|
| 标题 | `Version History — · {drawingName}  {versionNo}` | 不含 Drawing Code |
| Status | 状态标签（§4.1 颜色规范，只读） | — |
| Ver (System) | 系统版本号，如 `V1`、`V3`（只读） | — |
| Description | 图纸描述（来自 `Drawing.description`，只读） | 无内容时显示 `—` |
| Submission Ref No. | 外部报审编号（`DrawingApproval.submissionRefNo`，`phase=EXTERNAL`，只读） | 外部审批尚未发起时显示 `—` |
| Submission Subject | 外部报审主题（`DrawingApproval.submissionSubject`，`phase=EXTERNAL`，只读） | 外部审批尚未发起时显示 `—` |
| Uploaded by | 上传人头像 + 姓名（只读） | — |
| Upload Date | 上传时间，格式 `DD-MM-YYYY HH:mm:ss`（只读） | — |
| 主文件行 | `📄 {fileName}` + 审批状态 `✅ Approved by {name} · {time}` + `[Download]` | 点击 [Download] 下载原始文件（`fileUrl`） |

---

#### Tab 导航规格

默认激活 **Attachments** Tab。

| Tab | 标题格式 | 数量来源 | Element UI 组件 |
|-----|---------|---------|----------------|
| Part Print | `Part Print ({n})` | 该版本关联的 Markup 总数（含 ACTIVE 与 MERGED） | `el-tabs` |
| Attachments | `Attachments ({n})` | 该版本的附件总数 | `el-tabs` |

---

#### Part Print Tab 规格

**关联 AC**：AC-007C-017、AC-007C-018

- 只读展示，**无任何发布、删除、汇总操作按钮**
- 按发布时间倒序排列
- 每条 Markup 以卡片形式展示：

| 卡片字段 | 视觉规格 |
|---------|---------|
| 状态标签 | `ACTIVE` → 绿色 `#67C23A`；`Merged → V{n}` → 灰色 `#909399` |
| 标题 | 14px / font-weight 600 |
| 元信息行 | 发布日期（`MMM D, YYYY`）· 创建人姓名，12px / `#909399` |
| 受影响区域 | 灰色 `el-tag size=mini`，`affectedArea` 字段；无内容时隐藏该行 |
| 说明文字 | 最多 3 行，超出显示"…Show more"可展开；13px / `#606266` |
| 附件列表 | `📎 {fileName}`，点击可下载；12px |

- 无 Markup 时显示空状态 `"No Data"`

---

#### Attachments Tab 规格

**关联 AC**：AC-007C-012 ~ 016

**[+] 上传按钮规则**：

- 位置：`Attachments (n)` 标题右侧图标按钮（`el-icon-plus`）
- **仅本版本上传人（设计人员）可见**；其他角色隐藏整个按钮
- 点击触发文件选择器：支持任意文件类型，单文件 ≤ 50MB，单次最多 5 个
- 上传成功：表格即时追加新行，Tab 标题计数同步 +1
- 上传失败：Toast 提示错误（如"文件超过 50MB 限制"）

**Attachments 表格列**：

| 列 | 内容 | 说明 |
|----|------|------|
| Filename | 原始文件名 | 点击文件名可下载（与行 [Download] 等效） |
| Type | 文件扩展名大写（XLSX / DWG …） | — |
| Size | 格式化大小（1.2 MB） | — |
| Uploaded | 上传日期 `YYYY-MM-DD` | — |
| Uploaded by | 上传人姓名 | — |
| 行操作 | [Download]（所有人可见）；[Delete]（仅本版本上传人可见） | [Delete] 需二次确认 |

**[Delete] 二次确认 Dialog**：

```
┌──────────────────────────────────────────────────┐
│  Delete 结构计算书.xlsx?                           │
│  This action cannot be undone.                   │
│                      [Cancel]    [Delete]         │
└──────────────────────────────────────────────────┘
```

- 删除成功：行即时移除，Tab 标题计数 -1

**Attachments Tab 状态（5 态）**：

| 状态 | 视觉表现 |
|-----|---------|
| 空（无附件） | 表格内显示 "No Data"，[+] 按钮仍可见（若有权限） |
| 加载 | 表格 Skeleton |
| 正常 | 如布局图所示 |
| 错误 | Toast 提示文件加载/上传/删除失败 |
| 极端数据 | 附件数 > 20 时表格内分页，不截断 |

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色语义（审批状态 5 态）

| 状态值 | 标签文案 | 颜色 Token | 色值 |
|--------|---------|-----------|------|
| `PENDING_INTERNAL` | ⏳ Pending Internal | `--color-status-waiting` | `#E6A23C` |
| `INTERNAL_APPROVED` | ⏳ Pending External | `--color-status-waiting` | `#E6A23C` |
| `INTERNAL_REJECTED` | ❌ Int. Rejected | `--color-status-error` | `#F56C6C` |
| `APPROVED` | ✅ Approved | `--color-status-success` | `#67C23A` |
| `EXTERNAL_REJECTED` | ❌ Ext. Rejected | `--color-status-error` | `#F56C6C` |

### 4.2 间距与字体

| 组件 | 属性 | 值 |
|-----|------|----|
| 版本历史抽屉 | width | 720px |
| Details 二级抽屉 | width | 480px |
| 4 阶段卡片网格 | gap | 12px |
| 4 阶段卡片 | padding | 16px |
| 4 阶段卡片标题 | font-size | 13px |
| 4 阶段卡片标题 | font-weight | 600 |
| 4 阶段卡片内容 | font-size | 13px |
| 4 阶段卡片内容 | color | `#606266` |
| 步骤条文字 | font-size | 12px |
| Attachments 表格 | font-size | 13px |

---

## 5. 微交互与动效

| 场景 | 动效 | 时长 | 缓动 |
|-----|------|-----|-----|
| 版本历史抽屉滑入/滑出 | slide-in-right | 300ms | ease-out |
| Details 二级抽屉滑入/滑出 | slide-in-right | 250ms | ease-out |
| 4 阶段卡片展开/折叠 | collapse（高度动画） | 200ms | ease-in-out |
| 步骤条状态切换 | color transition | 200ms | linear |
| Tab 切换 | 默认 Element UI el-tabs 动效 | — | — |

---

## 6. 无障碍（A11y）

- WCAG 等级：AA
- 颜色对比度：状态标签文字与背景色对比度 ≥ 4.5:1
- 键盘导航：展开/折叠按钮支持 Enter/Space；Attachments 表格行支持 Tab；[Details] 按钮支持 Enter
- 焦点可见：所有交互元素有 `focus-visible` 样式（2px outline）

---

## 7. 文案规范

| 场景 | 文案（en） |
|-----|----------|
| 版本历史空状态 | No version history available. |
| Part Print 空状态 | No Data |
| Attachments 空状态 | No Data |
| 加载失败 | Failed to load. Retry |
| 步骤条标签 | Uploaded / Internal / External / Signed |
| 未到达步骤 | Not reached |
| 等待步骤 | Waiting |
| 删除附件确认 | Delete {fileName}? This action cannot be undone. |
| 文件上传超大提示 | File exceeds the 50MB limit. |
| RFA No. 未发起 | — |
| Description 为空 | — |

---

## 8. AC 覆盖检查表

| AC ID | 描述 | 对应章节 | 覆盖? |
|------|------|---------|------|
| AC-007C-001 | 抽屉宽度 720px，标题格式 | §3.1 规格 | ✅ |
| AC-007C-002 | 主列表含 Version / RFA No. / Status / Designer / Confirmed / QR / Actions 列 | §3.1 列定义 | ✅ |
| AC-007C-003 | APPROVED — 4 卡片全完成 | §3.2 各 approvalStatus 渲染规则 | ✅ |
| AC-007C-004 | INTERNAL_APPROVED + DC — ③ Mark Result 可见 | §3.2 ③ 外部审批卡片 | ✅ |
| AC-007C-005 | INTERNAL_APPROVED + 非 DC — ③ 无 Mark Result | §3.2 ③ 外部审批卡片 | ✅ |
| AC-007C-006 | INTERNAL_REJECTED — ② 驳回，③④ Not reached | §3.2 各 approvalStatus 渲染规则 | ✅ |
| AC-007C-007 | EXTERNAL_REJECTED — ③ Status C/E，④ Not reached | §3.2 ③ 外部审批卡片 | ✅ |
| AC-007C-008 | PENDING_INTERNAL — ② 进行中，③④ 等待 | §3.2 各 approvalStatus 渲染规则 | ✅ |
| AC-007C-009 | ④ [Download] 优先 pdfWithQrUrl | §3.2 ④ 签字版卡片 | ✅ |
| AC-007C-010 | ④ pdfWithQrUrl 为 null 降级 signedFileUrl | §3.2 ④ 签字版卡片 | ✅ |
| AC-007C-011 | Actions 列 [Details] 按钮触发 Details 二级抽屉 | §3.1 Actions 列按钮规格 | ✅ |
| AC-007C-012 | Details 抽屉内容展示（顶部信息区 + Tab） | §3.3 版本信息区字段、Tab 导航规格 | ✅ |
| AC-007C-013 | 附件下载，文件名与上传时一致 | §3.3 Attachments 表格列 | ✅ |
| AC-007C-014 | [+] / [Delete] 仅本版本上传人可见 | §3.3 [+] 上传按钮规则、[Delete] 规则 | ✅ |
| AC-007C-015 | 上传成功即时追加行，Tab 计数 +1 | §3.3 [+] 上传按钮规则 | ✅ |
| AC-007C-016 | 删除附件二次确认，即时移除，计数 -1 | §3.3 [Delete] 二次确认 Dialog | ✅ |
| AC-007C-017 | Part Print Tab — 显示当前版本关联 Markup 列表 | §3.3 Part Print Tab 规格 | ✅ |
| AC-007C-018 | Part Print Tab — 只读，无发布/删除/汇总操作 | §3.3 Part Print Tab 规格 | ✅ |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.2.5 | 2026-05-25 | agent | 同步 REQ-007C-pc@0.2.5：主列表移除 Attachments 列，新增 RFA No. 列与 Actions 列（[Details] 按钮）；触发方式由"点击 Attachments 单元格"改为"点击 [Details] 按钮"；Details 二级抽屉顶部新增 Description / Submission Ref No. / Submission Subject 字段，移除 Drawing Code；新增 Part Print Tab 规格；更新 §2.2 层级、§3.1 布局/列定义、§3.3 全节、§8 AC覆盖表（011 新含义 + 017 + 018） |
| 0.1.1 | 2026-05-24 | agent | 按来源需求独立成文件，完善字段与状态规范 |
| 0.1.0 | 2026-05-07 | agent | 从 UI-REQ-007-pc.md 拆分，覆盖 REQ-007C-pc 版本历史部分 |
