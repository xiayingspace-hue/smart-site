---
doc_type: ui_spec
req_id: REQ-007C-pc
version: 0.1.1
status: draft
generated_from: REQ-007C-pc@0.1.1
generated_at: 2026-05-24
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
| 来源需求 | REQ-007C-pc @ v0.1.1 |
| 覆盖用户故事 | US-007C-001、US-007C-002 |
| 覆盖 AC | AC-007C-001 ~ AC-007C-016 |
| 上次同步时间 | 2026-05-24 |

> ⚠️ 当来源需求版本变更时，本文档需更新此块，并 review 受影响章节。

---

## 1. 设计目标

### 1.1 核心目标

设计师和管理人员可在版本历史抽屉中查看图纸每个版本的完整审批状态演进，并能访问附件。

### 1.2 设计原则

1. **层叠抽屉**：Attachments 二级抽屉叠加在版本历史抽屉之上，空间利用高效。
2. **4 阶段可视化**：通过 2×2 卡片 + 步骤条，一眼看出当前版本在审批流程中所处的阶段。
3. **状态驱动渲染**：4 个卡片的内容完全由 `approvalStatus` 驱动，设计稿需覆盖全部 5 种状态组合。

---

## 2. 信息架构

### 2.1 入口

图纸列表页任意图纸行 → [Version History] 按钮（或版本列） → 版本历史抽屉

### 2.2 层级

```
图纸列表页
└── 版本历史抽屉（720px）
    ├── 版本行（可展开 → 4 阶段卡片 + 步骤条）
    └── Attachments 列单元格点击 → Attachments 二级抽屉（480px，叠加）
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
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Version History — ARCH-001 首层平面图                                      [✕]  │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Version │ Status              │ Designer │ Confirmed │ QR      │ Attachments    │
│──────────│─────────────────────│──────────│───────────│─────────│────────────────│
│  ▶ V3    │ ✅ Approved         │ 张三     │ 5/12      │ [View]  │ 📎 3           │
│  ▶ V2    │ ⏳ Pending External │ 李四     │ —         │ —       │ 📎 1           │
│  ▶ V1    │ ❌ Int. Rejected    │ 张三     │ —         │ —       │ 📎 0           │
└──────────────────────────────────────────────────────────────────────────────────┘
```

#### 列定义

| 列 | 内容 | 说明 |
|----|------|------|
| Version | `V1 / V2 / V3 …`，左侧有 ▶ 展开箭头 | 点击展开/折叠 4 阶段卡片 |
| Status | 状态标签（5 态，颜色见 §4.1） | — |
| Designer | 设计人员（上传人）姓名 | — |
| Confirmed | `x/y`（已确认/总人数） | 仅 `APPROVED` 版本显示；其他状态显示 `—` |
| QR | `[View]` 链接 | 仅 `APPROVED` 且 QR 已生成时显示；其他显示 `—` |
| Attachments | `📎 n`（n≥0） | **点击该单元格**打开 Attachments 二级抽屉（§3.3），与展开行无关 |

#### 状态（5 态）

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
| APPROVED | ✅ Approved · Marked by: {dcName} · Approval Date: {date} · 📎 Evidence: {evidenceFileName} [Download] · Remark: {remark}（有则显示） |
| EXTERNAL_REJECTED | ❌ Rejected · Marked by: {dcName} · {time} · Reason: {comment} |
| INTERNAL_REJECTED | — Not reached（Internal approval failed） |

#### ④ 签字版卡片

| approvalStatus | 显示内容 |
|---------------|---------|
| APPROVED | 📄 {signedFileName} · ✍️ Signed Drawing · Uploaded: {date} · Size: {size} · [Preview] [Download]（优先 pdfWithQrUrl > signedFileUrl） |
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

### 3.3 Attachments 二级抽屉

**关联 AC**：AC-007C-011 ~ 016

#### 规格

| 属性 | 值 |
|------|----|
| 宽度 | **480px** |
| 位置 | 从右侧滑入，**叠加在版本历史抽屉（720px）之上** |
| 触发方式 | 点击主列表任意版本行的 Attachments 列单元格（与版本行展开/折叠无关） |
| 关闭方式 | 右上角 [✕] |
| 关闭后 | 返回版本历史抽屉，版本历史抽屉保持原状 |

#### 布局

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Version History — · ARCH-001  V3                                   [✕]  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Drawing Code    ARCH-001                                                │
│  Status          ✅ Active                                               │
│  Ver (System)    V3                                                      │
│  Uploaded by     👤 张三                                                 │
│  Upload Date     2026-04-01 10:00                                        │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ 📄 arch001-v3.pdf                               [Download]       │    │
│  │ ✅ Approved by 王总工 · 2026-04-02 14:30                          │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Attachments (3)                                              [+]        │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ Filename         │ Type  │ Size    │ Uploaded    │ Uploaded by    │    │
│  │──────────────────│───────│─────────│─────────────│────────────────│    │
│  │ 结构计算书.xlsx   │ xlsx  │ 1.2 MB  │ 2026-04-01  │ 张三  [Delete] │    │
│  │ 施工说明.docx     │ docx  │ 0.5 MB  │ 2026-04-01  │ 张三  [Delete] │    │
│  │ arch001-v3.dwg   │ dwg   │ 8.3 MB  │ 2026-04-01  │ 张三  [Delete] │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 版本信息区字段

| 字段 | 内容 |
|------|------|
| 标题 | `Version History — · {drawingCode}  {versionNo}` |
| Drawing Code | 图纸编号（只读） |
| Status | 状态标签（§4.1 颜色规范，只读） |
| Ver (System) | 系统版本号（只读） |
| Uploaded by | 头像 + 姓名（只读） |
| Upload Date | `DD-MM-YYYY HH:mm:ss`（只读） |
| 主文件行 | `📄 {fileName}` + 审批状态 `✅ Approved by {name} · {time}` + `[Download]` |

#### Attachments 表格列

| 列 | 内容 | 说明 |
|----|------|------|
| Filename | 原始文件名 | 点击文件名可下载（与行 [Download] 等效） |
| Type | 文件扩展名大写（XLSX / DWG …） | — |
| Size | 格式化大小（1.2 MB） | — |
| Uploaded | 上传日期 `YYYY-MM-DD` | — |
| Uploaded by | 上传人姓名 | — |
| 行操作 | [Download]（所有人）；[Delete]（仅本版本上传人） | [Delete] 需二次确认 |

#### [+] 上传按钮规则

- 仅本版本上传人（设计人员）可见；其他角色隐藏
- 点击触发文件选择器：任意文件类型，单文件 ≤ 50MB，单次最多 5 个
- 上传成功：表格即时追加行，标题计数 +1，主列表 Attachments 列数字同步更新

#### [Delete] 二次确认

```
┌──────────────────────────────────────────────────┐
│  Delete 结构计算书.xlsx?                           │
│  This action cannot be undone.                   │
│                      [Cancel]    [Delete]         │
└──────────────────────────────────────────────────┘
```

删除成功：行即时移除，标题计数 -1，主列表 Attachments 列同步减 1。

#### 状态（5 态）

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
| PENDING_INTERNAL | ⏳ Pending Internal | `--color-status-waiting` | `#E6A23C` |
| INTERNAL_APPROVED | ⏳ Pending External | `--color-status-waiting` | `#E6A23C` |
| INTERNAL_REJECTED | ❌ Int. Rejected | `--color-status-error` | `#F56C6C` |
| APPROVED | ✅ Approved | `--color-status-success` | `#67C23A` |
| EXTERNAL_REJECTED | ❌ Ext. Rejected | `--color-status-error` | `#F56C6C` |

### 4.2 间距与字体

| 组件 | 属性 | 值 |
|-----|------|----|
| 版本历史抽屉 | width | 720px |
| Attachments 二级抽屉 | width | 480px |
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
| Attachments 二级抽屉滑入/滑出 | slide-in-right | 250ms | ease-out |
| 4 阶段卡片展开/折叠 | collapse（高度动画） | 200ms | ease-in-out |
| 步骤条状态切换 | color transition | 200ms | linear |

---

## 6. 无障碍（A11y）

- WCAG 等级：AA
- 颜色对比度：状态标签文字与背景色对比度 ≥ 4.5:1
- 键盘导航：展开/折叠按钮支持 Enter/Space；Attachments 表格行支持 Tab
- 焦点可见：所有交互元素有 `focus-visible` 样式（2px outline）

---

## 7. 文案规范

| 场景 | 文案（en） |
|-----|----------|
| 版本历史空状态 | No version history available. |
| Attachments 空状态 | No Data |
| 加载失败 | Failed to load. Retry |
| 步骤条标签 | Uploaded / Internal / External / Signed |
| 未到达步骤 | Not reached |
| 等待步骤 | Waiting |
| 删除附件确认 | Delete {fileName}? This action cannot be undone. |

---

## 8. AC 覆盖检查表

| AC ID | 对应章节 | 覆盖? |
|------|---------|------|
| AC-007C-001 | §3.1 抽屉宽度与标题格式 | ✅ |
| AC-007C-002 | §3.1 列完整性（含 Confirmed / QR 逻辑） | ✅ |
| AC-007C-003 | §3.2 APPROVED — 4 卡片全完成 | ✅ |
| AC-007C-004 | §3.2 INTERNAL_APPROVED + DC — ③ Mark Result | ✅ |
| AC-007C-005 | §3.2 INTERNAL_APPROVED + 非 DC — ③ 无 Mark Result | ✅ |
| AC-007C-006 | §3.2 INTERNAL_REJECTED — ②驳回，③④ Not reached | ✅ |
| AC-007C-007 | §3.2 EXTERNAL_REJECTED — ③驳回，④ Not reached | ✅ |
| AC-007C-008 | §3.2 PENDING_INTERNAL — ②进行中，③④等待 | ✅ |
| AC-007C-009 | §3.2 ④ 优先 pdfWithQrUrl | ✅ |
| AC-007C-010 | §3.2 ④ pdfWithQrUrl 为 null 降级 signedFileUrl | ✅ |
| AC-007C-011 | §3.1 Attachments 列计数 📎 n | ✅ |
| AC-007C-012 | §3.3 触发方式与布局 | ✅ |
| AC-007C-013 | §3.3 下载附件 | ✅ |
| AC-007C-014 | §3.3 权限 [+] / [Delete]，非上传人隐藏 | ✅ |
| AC-007C-015 | §3.3 上传附件即时追加行，计数同步 | ✅ |
| AC-007C-016 | §3.3 删除附件二次确认，即时移除，计数同步 | ✅ |

---

## 9. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-07 | agent | 从 UI-REQ-007-pc.md 拆分，覆盖 REQ-007C-pc 版本历史部分 |
| 0.1.1 | 2026-05-24 | agent | 按来源需求独立成文件，完善字段与状态规范 |
