# 前端开发说明文档 — PC 端图纸二维码管理

> RUNTIME LIBRARY: 项目实现基于 Element UI（运行时）——所有实现必须使用 Element 组件或等价适配层。

> **来源需求**: [REQ-006-pc.md](../../../requirements/pc/REQ-006-pc.md) + [REQ-006-shared.md](../../../requirements/shared/REQ-006-shared.md)
> **UI 设计参考**: [UI-REQ-006-pc.md](../../ui/pc/UI-REQ-006-pc.md)
> **产品**: SMART SITE SYSTEM
> **平台**: PC 管理端（Vue 2 + Element UI，桌面浏览器，1280px+）
> **生成日期**: 2026-04-17

---

## 1. 功能概述

在现有 REQ-003-pc 图纸管理基础上扩展 QR 管理功能：

| 功能模块 | 说明 |
|---------|------|
| 版本历史 QR 列 | 在版本历史抽屉表格中新增 QR 列，显示 `[View]` / `⚠️ Not generated` / `—` 三种状态 |
| QR 详情抽屉 | 右侧 480px 抽屉，展示 QR 图像、生成时间、公开 URL、下载与重新生成操作 |
| 主列表 QR 状态指示 | 当前版本 QR 未生成时，版本号旁显示 ⚠️ 图标 |
| 图纸预览扩展 | PDF 预览优先加载 `pdfWithQrUrl`（带 QR 水印） |
| 审批 Toast 文案调整 | 审批通过 Toast 改为 "Drawing approved. QR code is being generated..." |

---

## 2. 技术栈与约定

| 项目 | 规范 |
|------|------|
| 框架 | Vue 2 + Vue Router + Vuex（沿用现有） |
| UI 组件库 | Element UI 2.x |
| HTTP | Axios，`@/utils/request.js`（统一封装） |
| 文件下载 | 使用 `<a download>` 或 `window.open` 下载 OSS 静态文件；对需后端鉴权的下载使用 Blob 方式 |
| 剪贴板 | 使用 `navigator.clipboard.writeText()`，降级使用 `document.execCommand('copy')` |
| 权限控制 | 复用现有 `v-has-permission` 指令（`drawing:upload` / `drawing:view`） |
| 国际化 | Vue I18n，新增 key 见 §8 |

---

## 3. 文件与目录结构

```
src/
├── views/
│   └── drawing-management/
│       ├── index.vue                        # 已存在，REQ-003 主列表（扩展 ⚠️ 图标）
│       └── components/
│           ├── VersionHistoryDrawer.vue     # 已存在，扩展 QR 列
│           └── DrawingQrDrawer.vue          # 新增：QR 详情抽屉
├── api/
│   └── drawing-qr.js                        # 新增：QR 相关 API 封装
└── utils/
    └── clipboard.js                          # 新增：剪贴板工具（若项目无）
```

---

## 4. 版本历史 QR 列扩展

### 4.1 扩展点

复用现有 `VersionHistoryDrawer.vue` 组件，在表格列配置中追加 QR 列（见 UI § 3.1）。

### 4.2 列渲染逻辑

```vue
<el-table-column label="QR" width="120" align="center">
  <template slot-scope="{ row }">
    <!-- APPROVED + QR 已生成 -->
    <el-button
      v-if="row.approvalStatus === 'APPROVED' && row.qrGeneratedTime"
      type="text"
      @click="openQrDrawer(row)"
    >
      View
    </el-button>

    <!-- APPROVED + QR 未生成 -->
    <el-tooltip
      v-else-if="row.approvalStatus === 'APPROVED' && !row.qrGeneratedTime"
      content="QR generation failed or in progress. Click to regenerate."
      placement="top"
    >
      <span
        class="qr-not-generated"
        :class="{ 'clickable': hasRegeneratePermission }"
        @click="hasRegeneratePermission && handleQuickRegenerate(row)"
      >
        ⚠️ Not generated
      </span>
    </el-tooltip>

    <!-- PENDING / REJECTED -->
    <span v-else class="text-muted">—</span>
  </template>
</el-table-column>
```

### 4.3 快速重新生成（从 ⚠️ 直接触发）

```javascript
async handleQuickRegenerate(row) {
  await this.$confirm(
    this.$t('qr.regenerate.confirmBody', { versionNo: row.versionNo }),
    this.$t('qr.regenerate.confirmTitle'),
    { type: 'warning', confirmButtonText: 'Regenerate' }
  )
  try {
    await regenerateQr({ drawingVersionId: row.id })
    this.$message.success(this.$t('qr.regenerate.success'))
    this.$emit('refresh')  // 触发父组件刷新版本列表
  } catch (err) {
    this.$message.error(err.message)
  }
}
```

---

## 5. QR 详情抽屉 `DrawingQrDrawer.vue`

### 5.1 Props

| Prop | 类型 | 说明 |
|------|------|------|
| `visible` | Boolean | 抽屉显隐，`.sync` 双向绑定 |
| `drawingVersionId` | Number | 目标版本 ID，父组件传入 |

### 5.2 数据状态

| 变量 | 类型 | 说明 |
|------|------|------|
| `qrInfo` | Object \| null | QR 详情对象（`/drawing/version/qr` 返回） |
| `loading` | Boolean | 数据加载态 |
| `regenerating` | Boolean | 重新生成 loading |
| `copied` | Boolean | 复制成功态（控制 Tooltip 临时展示） |

### 5.3 数据加载

```javascript
watch: {
  visible(val) {
    if (val && this.drawingVersionId) {
      this.fetchQrInfo()
    }
  }
},
methods: {
  async fetchQrInfo() {
    this.loading = true
    try {
      this.qrInfo = await getVersionQr(this.drawingVersionId)
    } finally {
      this.loading = false
    }
  }
}
```

### 5.4 页面结构

```vue
<template>
  <el-drawer
    :visible.sync="visible"
    :title="$t('qr.drawer.title')"
    direction="rtl"
    size="480px"
    :destroy-on-close="true"
  >
    <div v-loading="loading" class="qr-drawer-body">
      <!-- QR 图像 -->
      <div class="qr-image-wrapper">
        <img
          v-if="qrInfo?.qrImageUrl"
          :src="qrInfo.qrImageUrl"
          class="qr-image"
          alt="Drawing QR Code"
        />
      </div>

      <!-- 元信息 -->
      <div class="meta-row">
        <span class="label">{{ $t('qr.drawer.version') }}</span>
        <span class="value">{{ qrInfo?.versionNo }}</span>
      </div>
      <div class="meta-row">
        <span class="label">{{ $t('qr.drawer.generated') }}</span>
        <span class="value">{{ formatDateTime(qrInfo?.qrGeneratedTime) }}</span>
      </div>

      <!-- Public URL -->
      <div class="section-label">{{ $t('qr.drawer.publicUrl') }}</div>
      <div class="url-box">
        <span class="url-text">{{ qrInfo?.publicStatusUrl }}</span>
        <el-button type="text" size="mini" @click="handleCopy">
          {{ $t('qr.drawer.copy') }}
        </el-button>
      </div>

      <!-- 操作按钮 -->
      <el-button
        type="primary"
        class="action-btn"
        :disabled="!qrInfo?.qrImageUrl"
        @click="handleDownloadImage"
      >
        {{ $t('qr.drawer.downloadImage') }}
      </el-button>

      <el-tooltip
        :disabled="!!qrInfo?.pdfWithQrUrl"
        :content="$t('qr.drawer.pdfDisabledHint')"
        placement="top"
      >
        <el-button
          class="action-btn"
          :disabled="!qrInfo?.pdfWithQrUrl"
          @click="handleDownloadPdf"
        >
          {{ $t('qr.drawer.downloadPdf') }}
        </el-button>
      </el-tooltip>

      <el-button
        v-has-permission="'drawing:upload'"
        class="action-btn"
        icon="el-icon-refresh"
        :loading="regenerating"
        @click="handleRegenerate"
      >
        {{ $t('qr.drawer.regenerate') }}
      </el-button>
    </div>
  </el-drawer>
</template>
```

### 5.5 关键方法

```javascript
methods: {
  // 复制 URL
  async handleCopy() {
    await copyToClipboard(this.qrInfo.publicStatusUrl)
    this.$message.success(this.$t('qr.drawer.copied'))
  },

  // 下载 QR 图片
  handleDownloadImage() {
    const filename = `${this.qrInfo.drawingCode}-${this.qrInfo.versionNo}-qr.png`
    downloadFile(this.qrInfo.qrImageUrl, filename)
  },

  // 下载带 QR 的 PDF
  handleDownloadPdf() {
    const filename = `${this.qrInfo.drawingCode}-${this.qrInfo.versionNo}-with-qr.pdf`
    downloadFile(this.qrInfo.pdfWithQrUrl, filename)
  },

  // 重新生成
  async handleRegenerate() {
    await this.$confirm(
      this.$t('qr.regenerate.confirmBody', { versionNo: this.qrInfo.versionNo }),
      this.$t('qr.regenerate.confirmTitle'),
      { type: 'warning', confirmButtonText: 'Regenerate' }
    )
    this.regenerating = true
    try {
      this.qrInfo = await regenerateQr({ drawingVersionId: this.drawingVersionId })
      this.$message.success(this.$t('qr.regenerate.success'))
      this.$emit('qr-updated')  // 通知父组件刷新
    } catch (err) {
      this.$message.error(err.message)
    } finally {
      this.regenerating = false
    }
  },

  formatDateTime(iso) {
    if (!iso) return '—'
    return dayjs(iso).format('YYYY-MM-DD HH:mm')
  }
}
```

---

## 6. 主列表 ⚠️ 状态指示

扩展现有 `drawing-management/index.vue` 的 Version 列：

```vue
<el-table-column :label="$t('drawing.col.version')" width="120" align="center">
  <template slot-scope="{ row }">
    <span>{{ row.currentVersion }}</span>
    <el-tooltip
      v-if="row.currentVersionApproved && !row.currentVersionQrGenerated"
      :content="$t('qr.mainList.warningTooltip')"
      placement="top"
    >
      <i
        class="el-icon-warning-outline qr-warning-icon"
        @click.stop="openVersionHistory(row, row.currentVersionId)"
      />
    </el-tooltip>
  </template>
</el-table-column>
```

**触发条件**：
- 当前版本 `approvalStatus = APPROVED`
- 且 `qrGeneratedTime` 为空

**点击行为**：
- 打开版本历史抽屉（原有行为），并将选中版本滚动定位到视图中（可选实现）
- 使用 `.stop` 防止冒泡触发行点击

---

## 7. 图纸预览扩展（PDF 优先加载 pdfWithQrUrl）

修改现有图纸详情页的 `DrawingPreview` 组件文件 URL 解析逻辑：

```javascript
computed: {
  previewUrl() {
    // 优先 pdfWithQrUrl，降级 fileUrl
    return this.drawing.pdfWithQrUrl || this.drawing.fileUrl
  }
}
```

无需额外 UI 改动，QR 自然显示在 PDF 右下角。

---

## 8. 审批 Toast 文案调整

定位到现有审批 Todo 组件的 Approve 成功回调：

```javascript
// 原代码
this.$message.success('Drawing approved successfully.')

// 修改为
this.$message.success(this.$t('approval.toast.approvedWithQr'))
```

对应 i18n key：`approval.toast.approvedWithQr`

---

## 9. API 封装 `api/drawing-qr.js`

```javascript
import request from '@/utils/request'

// 获取版本 QR 信息
export const getVersionQr = (drawingVersionId) =>
  request.get('/drawing/version/qr', {
    params: { drawingVersionId }
  }).then(res => res.data)

// 重新生成 QR
export const regenerateQr = (data) =>
  request.post('/drawing/version/qr/regenerate', data)
    .then(res => res.data)
```

> **注**：返回字段 `qrGeneratedTime`、`pdfWithQrUrl` 可能为 null。组件侧需做空值判断。

---

## 10. 工具函数

### 10.1 `utils/clipboard.js`

```javascript
export async function copyToClipboard(text) {
  if (navigator.clipboard?.writeText) {
    await navigator.clipboard.writeText(text)
    return
  }
  // 降级方案
  const textarea = document.createElement('textarea')
  textarea.value = text
  textarea.style.position = 'fixed'
  textarea.style.opacity = '0'
  document.body.appendChild(textarea)
  textarea.select()
  document.execCommand('copy')
  document.body.removeChild(textarea)
}
```

### 10.2 `utils/download.js`（若已存在则复用）

```javascript
export function downloadFile(url, filename) {
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.target = '_blank'
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
}
```

> **跨域限制**：若 OSS CDN 未配置 `Content-Disposition: attachment` 且跨域，`download` 属性无效，浏览器直接打开文件。建议后端在返回 OSS URL 时使用预签名 URL 并指定 attachment 头。

---

## 11. Vuex 集成（可选）

若项目统一管理图纸状态，可将 `qrInfo` 缓存到 Vuex：

```javascript
// store/modules/drawing-qr.js
const state = {
  qrCache: {}  // { drawingVersionId: qrInfo }
}

const mutations = {
  SET_QR(state, { drawingVersionId, qrInfo }) {
    Vue.set(state.qrCache, drawingVersionId, qrInfo)
  }
}

const actions = {
  async fetchQr({ commit, state }, drawingVersionId) {
    if (state.qrCache[drawingVersionId]) {
      return state.qrCache[drawingVersionId]
    }
    const qrInfo = await getVersionQr(drawingVersionId)
    commit('SET_QR', { drawingVersionId, qrInfo })
    return qrInfo
  }
}
```

> 在重新生成后需 `commit('SET_QR', ...)` 更新缓存，避免显示旧数据。

---

## 12. 国际化（i18n）Key

```
qr.column.title              → "QR" / "二维码"
qr.column.view               → "View" / "查看"
qr.column.notGenerated       → "⚠️ Not generated" / "⚠️ 未生成"
qr.drawer.title              → "Drawing QR Code" / "图纸二维码"
qr.drawer.version            → "Version" / "版本"
qr.drawer.generated          → "Generated" / "生成时间"
qr.drawer.publicUrl          → "Public URL" / "公开链接"
qr.drawer.copy               → "Copy" / "复制"
qr.drawer.copied             → "Copied to clipboard" / "已复制到剪贴板"
qr.drawer.downloadImage      → "Download QR Image (PNG)" / "下载二维码图片（PNG）"
qr.drawer.downloadPdf        → "Download PDF with QR" / "下载含二维码的 PDF"
qr.drawer.regenerate         → "Regenerate QR" / "重新生成二维码"
qr.drawer.pdfDisabledHint    → "Only PDF drawings support embedded QR" / "仅 PDF 图纸支持内嵌二维码"
qr.regenerate.confirmTitle   → "Regenerate QR" / "重新生成二维码"
qr.regenerate.confirmBody    → "Regenerate QR for version {versionNo}? ..." / "为版本 {versionNo} 重新生成二维码？..."
qr.regenerate.success        → "QR regenerated successfully" / "二维码已重新生成"
qr.mainList.warningTooltip   → "QR not generated for current version. Click to view." / "当前版本的二维码未生成，点击查看"
approval.toast.approvedWithQr → "Drawing approved. QR code is being generated..." / "图纸已审批通过，二维码正在生成..."
```

---

## 13. 权限控制

| 操作 | 权限 | 前端处理 |
|------|------|---------|
| 查看 QR（[View] / 打开抽屉） | `drawing:view` | 基础权限，所有管理端用户具备 |
| 重新生成 QR | `drawing:upload` | `v-has-permission="'drawing:upload'"` 控制按钮显隐 |
| 下载 QR 图片 / 带 QR 的 PDF | `drawing:view` | 无额外限制 |

---

## 14. 错误处理

| 错误码 | 前端表现 |
|--------|----------|
| 1003006002 | Toast 错误："Cannot generate QR for unapproved version." 按钮恢复可点击 |
| 1003006003 | Toast 错误："QR generation failed. Please try again later." 按钮恢复 |
| 1003006004 | Toast 错误："Drawing version not found." 关闭抽屉并刷新父列表 |
| 1003006005 | Toast 错误："QR not available for this version." 关闭抽屉 |
| 网络错误 | Toast 错误："Network error. Please try again." |

---

## 15. 样式规范（SCSS 片段）

```scss
.qr-drawer-body {
  padding: 24px;

  .qr-image-wrapper {
    text-align: center;
    margin-bottom: 24px;

    .qr-image {
      width: 240px;
      height: 240px;
      border: 1px solid var(--color-border-base, #DCDFE6);
      border-radius: 4px;
    }
  }

  .meta-row {
    display: flex;
    justify-content: space-between;
    margin-bottom: 12px;

    .label {
      font-size: 13px;
      color: var(--color-text-secondary, #909399);
      width: 88px;
    }
    .value {
      font-size: 14px;
      color: var(--color-text-primary, #303133);
    }
  }

  .section-label {
    font-size: 13px;
    color: var(--color-text-secondary, #909399);
    margin: 16px 0 8px;
  }

  .url-box {
    background: #F5F7FA;
    border-radius: 4px;
    padding: 12px;
    margin-bottom: 24px;
    font-family: Monaco, Consolas, monospace;
    font-size: 13px;
    word-break: break-all;
    position: relative;

    .url-text {
      display: inline-block;
      max-height: 3em;
      overflow: hidden;
      text-overflow: ellipsis;
    }
  }

  .action-btn {
    width: 100%;
    height: 40px;
    margin-bottom: 12px;
  }
}

.qr-not-generated {
  color: #F57F17;
  font-size: 13px;

  &.clickable {
    cursor: pointer;
    &:hover { text-decoration: underline; }
  }
}

.qr-warning-icon {
  color: #F57F17;
  font-size: 14px;
  margin-left: 4px;
  cursor: pointer;

  &:hover { opacity: 0.8; }
}

.text-muted {
  color: var(--color-text-secondary, #909399);
}
```

---

## 16. 验收标准

### 版本历史 QR 列
- [ ] APPROVED + `qrGeneratedTime` 不为空时显示 `[View]` 链接
- [ ] APPROVED + `qrGeneratedTime` 为空时显示 `⚠️ Not generated` 橙色文字
- [ ] PENDING / REJECTED 版本显示 `—`
- [ ] 点击 `[View]` 打开 QR 详情抽屉
- [ ] 点击 `⚠️ Not generated`（`drawing:upload` 权限）直接触发重新生成
- [ ] 无 `drawing:upload` 权限时 `⚠️ Not generated` 仅显示，不可点击

### QR 详情抽屉
- [ ] 抽屉右侧滑出，宽度 480px
- [ ] QR 图像正确加载（240 × 240 px）
- [ ] Version / Generated 字段正确显示
- [ ] Public URL 使用等宽字体，最多 2 行显示
- [ ] Copy 按钮点击后剪贴板包含正确 URL，Toast 提示 "Copied to clipboard"
- [ ] Download QR Image 下载 PNG 文件，文件名 `{drawingCode}-{versionNo}-qr.png`
- [ ] Download PDF with QR 下载带 QR 水印的 PDF，文件名 `{drawingCode}-{versionNo}-with-qr.pdf`
- [ ] `pdfWithQrUrl` 为空时 Download PDF with QR 按钮置灰，hover 显示 tooltip
- [ ] Regenerate QR 按钮仅对 `drawing:upload` 用户可见
- [ ] 点击 Regenerate QR 弹出二次确认 MessageBox
- [ ] Regenerate 成功后抽屉内 QR 图像与 Generated 时间刷新
- [ ] Regenerate 成功后父组件 QR 列同步刷新
- [ ] Regenerate 时按钮进入 loading 禁用状态

### 主列表 ⚠️ 图标
- [ ] 当前版本 APPROVED 且 QR 未生成时显示 ⚠️ 图标
- [ ] Hover ⚠️ 显示 tooltip 文案
- [ ] 点击 ⚠️ 打开版本历史抽屉（不触发行点击）
- [ ] QR 正常生成后 ⚠️ 图标消失

### 图纸预览
- [ ] PDF 预览优先加载 `pdfWithQrUrl`
- [ ] `pdfWithQrUrl` 为空时降级加载 `fileUrl`
- [ ] PDF 内 QR 出现在每页右下角

### 审批 Toast
- [ ] 审批通过后 Toast 文案为 "Drawing approved. QR code is being generated..."
- [ ] Toast 持续 3000ms 后自动消失

### 错误处理
- [ ] 1003006002 ~ 1003006005 错误码正确展示 Toast 并恢复按钮状态
- [ ] 网络错误展示统一错误 Toast

### 国际化
- [ ] 所有新增 key 在 en.js / zh.js 两份语言包中均存在
- [ ] 切换语言后所有 QR 相关文案正确切换
