# 前端开发说明文档 — REQ-014 图纸管理用户反馈迭代（APP 端）

> **来源需求**: `requirements/pc/REQ-014-pc.md`
> **产品**: SMART SITE SYSTEM
> **平台**: APP 端（NS Safety App / UNIAPP + Vue2，iOS & Android）
> **生成日期**: 2026-04-28

---

## 1. 功能概述

REQ-014 APP 端改动聚焦于 FB-002、FB-005、FB-006 三条反馈，FB-001/003/004 仅涉及 PC 端：

| 反馈编号 | 改动描述 | APP 是否涉及 |
|----------|----------|--------------|
| FB-001 | 上传时修改 Drawing Code | ❌ PC 端专属 |
| FB-002 | 版本附件区域（SE 可下载） | ✅ |
| FB-003 | +Markup 入口移至弹框 | ❌ PC 端专属 |
| FB-004 | Markup 列表新增 Drawing Code 列 | ❌ PC 端专属 |
| FB-005 | SE 完全隐藏 Markup 入口 | ✅ |
| FB-006 | SE 视角隐藏系统版本号（Ver） | ✅ |

---

## 2. 技术栈与约定

| 项目 | 内容 |
|------|------|
| 框架 | UNIAPP + Vue2 |
| 运行平台 | iOS / Android（原生打包） |
| UI 组件库 | uni-ui + 自定义组件 |
| 状态管理 | Vuex（全局 store 存储当前用户信息 `userInfo.role`） |
| 文件下载 | `uni.downloadFile` + `uni.openDocument` |
| 权限判断 | `this.$store.getters.isSE` 或 `userInfo.role === 'SE'` |

---

## 3. 文件与目录结构

```
pages/
  drawing/
    DrawingVersionDetail.vue     ← FB-002 附件区域、FB-006 Ver 隐藏
    DrawingList.vue              ← FB-005 Markup 入口隐藏、FB-006 Ver 列隐藏
components/
  drawing/
    AttachmentList.vue           ← FB-002 新增组件：附件列表
    AttachmentItem.vue           ← FB-002 新增组件：附件列表单项（含下载按钮）
```

---

## 4. 各反馈改动详情

---

### 4.1 FB-002 版本详情页 — 新增附件区域（Attachments）

**涉及文件**: `DrawingVersionDetail.vue`、`AttachmentList.vue`、`AttachmentItem.vue`

**改动说明**:

在图纸版本详情页的 PDF 信息卡片下方，新增附件（Attachments）模块：

```
┌────────────────────────────────┐
│  Drawing Code: DWG-001         │
│  Ver: v3（非 SE 可见）         │
│  上传时间: 2026-03-10          │
├────────────────────────────────┤
│  📎 附件（Attachments）        │
│  ─────────────────────────────│
│  文件名.pdf       [下载]       │
│  施工说明.docx    [下载]       │
│  （无附件时显示"暂无附件"）    │
└────────────────────────────────┘
```

**`AttachmentList.vue`** — 附件列表容器:

- `props`: `versionId: String`
- 生命周期 `mounted` 调用附件列表接口
- 渲染 `AttachmentItem` 列表
- 空态：显示"暂无附件"

**`AttachmentItem.vue`** — 单条附件:

- `props`: `attachment: Object`（含 `id`, `name`, `type`, `size`, `downloadUrl`）
- 点击 `[下载]` 按钮逻辑：
  1. 调用 `uni.downloadFile({ url: attachment.downloadUrl })`
  2. 下载成功后 `uni.openDocument` 打开文件
  3. 下载中显示 loading 状态，失败后 toast 提示

**`[下载]` 按钮权限**:
- 所有有版本访问权限的角色（含 SE）均显示 `[下载]` 按钮
- 上传 / 删除相关控件 APP 端不提供（仅 PC 端支持）

**接口**:

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/drawings/:drawingId/versions/:versionId/attachments` | 获取附件列表 |
| GET | `/api/drawings/:drawingId/attachments/:attachmentId/download` | 获取下载链接（redirect 或 presigned URL） |

---

### 4.2 FB-005 SE 角色完全隐藏 Markup 入口

**涉及文件**: `DrawingVersionDetail.vue`（或含 Markup 入口的版本详情/列表页）

**改动说明**:
- 找到 APP 端所有渲染 Markup 入口的位置（按钮、Tab、角标等）
- 统一使用 `v-if="!isSE"` 控制，不使用 `v-show`
- SE 登录后无法从任何 UI 路径进入 Markup 功能

```vue
<!-- 示例：Markup 历史查看按钮 -->
<view v-if="!isSE" class="markup-entry" @click="goMarkupHistory">
  查看 Markup
</view>
```

**`isSE` computed**:
```js
computed: {
  isSE() {
    return this.$store.state.user.role === 'SE'
  }
}
```

---

### 4.3 FB-006 SE 视角隐藏系统版本号（Ver）

**涉及文件**: `DrawingVersionDetail.vue`、`DrawingList.vue`

**版本详情页**:
- 非 SE：显示 `Drawing Code` + `Ver`（系统版本号）
- SE：`Ver` 行不渲染

```vue
<!-- 版本详情 -->
<view class="detail-item">
  <text class="label">Drawing Code</text>
  <text class="value">{{ versionDetail.drawingCode }}</text>
</view>
<view v-if="!isSE" class="detail-item">
  <text class="label">Ver</text>
  <text class="value">{{ versionDetail.ver }}</text>
</view>
```

**图纸列表（DrawingList.vue）**:
- 若列表卡片展示了版本号（Ver），SE 视角不渲染该字段
- Drawing Code 正常展示给 SE

---

## 5. 权限控制汇总

| 功能 | 普通角色 | SE |
|------|----------|----|
| 查看版本附件列表 | ✅ | ✅ |
| 下载附件 | ✅ | ✅ |
| 上传 / 删除附件 | ✅（有权限者） | ❌ |
| Markup 历史入口 | ✅ | ❌ `v-if` 隐藏 |
| 系统版本号（Ver）显示 | ✅ | ❌ `v-if` 隐藏 |
| Drawing Code 显示 | ✅ | ✅ |

---

## 6. 重点注意事项

- **下载兼容性**：iOS 需确认 `uni.openDocument` 对各文件类型（`.dwg`, `.rvt`, `.docx` 等）的支持情况；不支持格式可提示用户"请在 PC 端查看"
- **FB-005**：APP 端需全局检索 Markup 相关入口（搜索 `markup`、`Markup`），确保无遗漏隐藏点
- **FB-006**：`Ver` 隐藏使用 `v-if` 而非样式 `display:none`，避免留空白行
- **`isSE` 统一来源**：建议封装为全局 `store getter`，避免各页面重复写角色判断逻辑

---

> 本文档配合 `REQ-014-pc.md` 需求文档及 `UI-REQ-014-app.md` UI 说明文档使用。
