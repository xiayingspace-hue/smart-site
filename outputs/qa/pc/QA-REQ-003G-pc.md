---
doc_type: qa_spec
req_id: REQ-003G-pc
version: 0.1.0
status: draft
generated_from: REQ-003G-pc@0.3.0
data_contract_ref: data-contract-003G-pc.md@0.1.0
generated_at: 2026-05-28
owner: ""
---

# QA 测试说明：PC 端 — 图纸管理区域配置与双模式 SE 分配（单选 + 按区域批量）

> **本文档供 QA 工程师及其 agent 使用**。
>
> ⚠️ **核心原则**：
> - 每个 AC 至少派生 1 条 TC，TC 描述显式标注覆盖的 AC ID。
> - 测试用例 ID 全局唯一。
> - 接口测试字段定义引用 data-contract-003G-pc.md。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003G-pc @ v0.2.0 |
| 数据契约 | data-contract-003G-pc.md @ v0.1.0 |
| 覆盖 Story | US-003G-001、US-003G-002、US-003G-003、US-003G-004 |
| 覆盖 AC | AC-003G-001 ～ AC-003G-013 |
| 测试环境 | dev / staging |

---

## 1. 测试目标

验证图纸管理区域配置（CRUD）、区域 SE 绑定管理，以及 [Assign] 弹框双模式 SE 选择（单选 Individual + 按区域批量 By Area，含混合使用）的功能正确性、权限隔离、幂等性和边界行为，确保所有 AC 100% 覆盖。

---

## 2. 测试策略

### 2.1 测试金字塔

| 层级 | 占比 | 由谁写 | 框架 |
|-----|-----|-------|------|
| 单元测试 | 60% | 前端/后端开发 | Vitest / JUnit 5 |
| 集成测试 | 25% | 开发 + QA | Testcontainers（后端）/ MSW（前端） |
| 契约测试 | 5% | QA | schemathesis |
| E2E 测试 | 10% | QA | Cypress |

### 2.2 测试范围

**必测**：
- 区域 CRUD（新建、编辑、删除、列表查询）
- 区域名唯一性校验（同项目内不重名）
- 区域 SE 绑定（Add、Remove、批量 diff 保存）
- [Select by Area] 按钮置灰（无区域时）
- 按区域批量 Add SE（并集 + 幂等去重）
- 双模式混合使用（批量 Add 后手动 Add/Remove）
- 权限控制（管理员 vs 非管理员）
- 区域配置页空状态

**不测**（本期）：
- APP 端区域配置（不适用）
- 区域作为图纸筛选维度（非本期需求）
- SE 账号管理本身（用户管理模块范围）
- OQ-001（SE 加入区域通知）：PM 未确认，暂不测试

---

## 3. 测试场景总览

### 3.1 主流程场景

| 场景 ID | 场景描述 | 涉及 Story | 优先级 |
|--------|---------|-----------|-------|
| SC-001 | 管理员进入区域配置页，创建新区域 | US-003G-001 | P0 |
| SC-002 | 管理员编辑已有区域名称 | US-003G-001 | P1 |
| SC-003 | 管理员删除区域（含二次确认） | US-003G-001 | P0 |
| SC-004 | 管理员为区域添加 SE，保存 | US-003G-002 | P0 |
| SC-005 | 管理员从区域移除 SE，保存 | US-003G-002 | P1 |
| SC-006 | 管理员在 [Assign] 弹框勾选整区域批量 Add SE | US-003G-003 | P0 |
| SC-009 | 管理员在下拉面板单独勾选特定 SE（不选整区域） | US-003G-003 | P0 |
| SC-010 | 管理员在下拉面板混合选择（整区域 + 单个 SE） | US-003G-003 | P0 |
| SC-007 | 管理员先按区域批量 Add，再手动单独 Remove 和 Add | US-003G-004 | P0 |
| SC-008 | 区域配置页空状态显示 | US-003G-001 | P1 |

### 3.2 异常场景

| 场景 ID | 场景描述 | 关联 AC |
|--------|---------|--------|
| SC-E01 | 新建区域名称与已有区域重名 | AC-003G-003 |
| SC-E02 | 新建区域名称为空 | AC-003G-002（边界） |
| SC-E03 | 新建区域名称超 50 字符 | AC-003G-002（边界） |
| SC-E04 | 删除有 SE 绑定的区域，SE 绑定同步失效 | AC-003G-004 |
| SC-E05 | 无区域时 [Select by Area] 按钮置灰 | AC-003G-006 |
| SC-E08 | 下拉面板搜索无结果时显示空态 | AC-003G-011（边界） |
| SC-E06 | 批量 Add 时所选区域 SE 与已 Assigned 列表重复（幂等） | AC-003G-008 |
| SC-E07 | 管理员未点 [Save] 直接关闭弹框，变更不提交 | 通用 |

### 3.3 权限场景

| 场景 ID | 场景描述 |
|--------|---------|
| SC-P01 | 普通业务人员进入图纸管理页，不可见 [Area Config] 入口 |
| SC-P02 | Site Engineer 进入图纸管理页，不可见 [Area Config] 入口 |
| SC-P03 | DC 进入图纸管理页，不可见 [Area Config] 入口 |
| SC-P04 | 未登录用户直接访问 `/drawing/area-config` 路由，跳转登录页 |
| SC-P05 | 普通业务人员直接调用区域 CRUD 接口，返回 403 |

### 3.4 状态转换场景

| 场景 ID | 描述 | 测试要点 |
|--------|------|---------|
| SC-T01 | DrawingArea: active → deleted（软删除） | 删除后列表不可见；drawing_area_ses 查询时自动失效 |
| SC-T02 | 软删除后同名区域可重新创建 | 唯一索引只在 deleted_at IS NULL 范围内生效 |

---

## 4. 测试用例详述

### TC-003G-001-01

```yaml
tc_id: TC-003G-001-01
covers_ac: [AC-003G-001]
scenario: SC-P01, SC-P02, SC-P03
priority: P0
test_level: e2e
preconditions:
  - 项目存在，有管理员账号 A、普通业务人员账号 B、SE 账号 C
test_data:
  - 账号 B（角色：业务人员），账号 C（角色：SE）

steps:
  given: 账号 B 或 C 登录系统
  when: |
    1. 进入图纸管理页面
  then:
    - 页面顶部操作区不显示 [Area Config] 按钮
    - 直接访问 /drawing/area-config 路由返回 403 或跳转无权限页

cleanup:
  - 无
```

---

### TC-003G-001-02

```yaml
tc_id: TC-003G-001-02
covers_ac: [AC-003G-001]
scenario: SC-P01
priority: P0
test_level: e2e
preconditions:
  - 管理员账号已登录

steps:
  given: 管理员进入图纸管理页面
  when: |
    1. 查看页面顶部操作区
  then:
    - [Area Config] 按钮可见且可点击
    - 点击后成功跳转到区域配置页
```

---

### TC-003G-002-01

```yaml
tc_id: TC-003G-002-01
covers_ac: [AC-003G-002]
scenario: SC-001
priority: P0
test_level: e2e
preconditions:
  - 管理员已登录，项目内无现有区域
test_data:
  - areaName: "Zone A"

steps:
  given: 管理员在区域配置页
  when: |
    1. 点击 [+ Add Area]
    2. 输入 Area Name = "Zone A"
    3. 点击 [Save]
  then:
    - 弹窗关闭
    - 区域配置列表新增 "Zone A" 一行
    - 该行 SE Count 显示 "0 SEs"
    - Snackbar 提示 "Area saved successfully."

cleanup:
  - 删除测试创建的区域
```

---

### TC-003G-002-02

```yaml
tc_id: TC-003G-002-02
covers_ac: [AC-003G-010b]
scenario: SC-008
priority: P1
test_level: e2e
preconditions:
  - 管理员已登录，项目内无任何区域

steps:
  given: 管理员进入区域配置页
  when: |
    1. 查看页面内容
  then:
    - 显示空状态插图
    - 显示文字 "No areas configured. Click [+ Add Area] to get started."
    - [+ Add Area] 按钮正常可用
```

---

### TC-003G-003-01

```yaml
tc_id: TC-003G-003-01
covers_ac: [AC-003G-003]
scenario: SC-E01
priority: P0
test_level: e2e
preconditions:
  - 已存在区域 "Zone A"
test_data:
  - 重复名称: "Zone A"

steps:
  given: 管理员在区域配置页
  when: |
    1. 点击 [+ Add Area]
    2. 输入 Area Name = "Zone A"
    3. 点击 [Save]
  then:
    - 弹窗未关闭
    - 输入框下方显示 inline 错误提示 "Area name already exists."
    - 列表中不新增重复区域

cleanup:
  - 无（区域未创建）
```

---

### TC-003G-004-01

```yaml
tc_id: TC-003G-004-01
covers_ac: [AC-003G-004]
scenario: SC-003, SC-E04
priority: P0
test_level: e2e
preconditions:
  - 已存在区域 "Zone B"，该区域已绑定 SE1、SE2、SE3 共 3 名
test_data:
  - areaName: "Zone B"，绑定 SE x3

steps:
  given: 管理员在区域配置页，"Zone B" 行可见
  when: |
    1. 点击 "Zone B" 行的 [Delete]
    2. 确认弹窗内容包含 "Zone B" 名称
    3. 点击 [Delete]（危险按钮）
  then:
    - "Zone B" 从列表消失
    - Snackbar 提示 "Area deleted."
    - 打开 [Assign] 弹框的 [Select by Area] 下拉，"Zone B" 不再出现
    - 原绑定的 SE1、SE2、SE3 不受影响（在其他区域的绑定保持）

cleanup:
  - 无（区域已删除）
```

---

### TC-003G-005-01

```yaml
tc_id: TC-003G-005-01
covers_ac: [AC-003G-005]
scenario: SC-004
priority: P0
test_level: e2e
preconditions:
  - 已存在区域 "Zone A"（无绑定 SE），项目内有 SE1～SE5 共 5 名
test_data:
  - 待添加: SE1、SE2、SE3

steps:
  given: 管理员在区域配置页
  when: |
    1. 点击 "Zone A" 的 [Manage SE]
    2. 弹框打开，Left: In Area(0)，Right: Available(5)
    3. 在 Available 中 Add SE1、SE2、SE3
    4. 点击 [Save]
  then:
    - 弹框关闭
    - "Zone A" 行 SE Count 更新为 "3 SEs"
    - Snackbar 提示 "Area SE configuration saved."
    - 重新打开 Manage SEs，In Area 列表包含 SE1、SE2、SE3

cleanup:
  - 从区域移除 SE1、SE2、SE3
```

---

### TC-003G-006-01

```yaml
tc_id: TC-003G-006-01
covers_ac: [AC-003G-006]
scenario: SC-E05
priority: P0
test_level: e2e
preconditions:
  - 项目内无任何已配置区域
  - 存在一张 ACTIVE 状态图纸

steps:
  given: 管理员打开该图纸的 [Assign] 弹框
  when: |
    1. 查看 Available 面板顶部 [Select by Area ▾] 按钮
  then:
    - 按钮呈置灰（disabled）状态
    - Hover 时显示 Tooltip: "No areas configured. Go to Area Config to set up."
    - 点击无响应
```

---

### TC-003G-007-01

```yaml
tc_id: TC-003G-007-01
covers_ac: [AC-003G-007]
scenario: SC-006
priority: P0
test_level: e2e
preconditions:
  - 项目已配置 "Zone A"（含 SE1、SE2、SE3）和 "Zone B"（含 SE3、SE4）
  - Assigned 列表为空
test_data:
  - 选择区域: Zone A, Zone B

steps:
  given: 管理员打开图纸 [Assign] 弹框，Assigned 为空
  when: |
    1. 点击 [Select by Area ▾]
    2. 勾选 Zone A 和 Zone B
    3. 点击 [Add Selected Areas]
  then:
    - SE1、SE2、SE3、SE4 全部进入 Assigned 列表
    - SE3 只出现一次（去重）
    - Assigned 计数为 4
    - Available 计数相应减少 4
    - 下拉面板关闭
```

---

### TC-003G-008-01

```yaml
tc_id: TC-003G-008-01
covers_ac: [AC-003G-008]
scenario: SC-E06
priority: P0
test_level: unit
preconditions:
  - SE1 已在 Assigned 列表
  - SE1 同属 "Zone A"

steps:
  given: Assigned 列表中已有 SE1
  when: |
    1. 在 [Select by Area] 中选择 Zone A
    2. 点击 [Add Selected Areas]
  then:
    - SE1 不被重复添加
    - Assigned 列表中 SE1 仅出现一次
    - Assigned 计数不因 SE1 已存在而错误增加
```

---

### TC-003G-009-01

```yaml
tc_id: TC-003G-009-01
covers_ac: [AC-003G-009]
scenario: SC-007
priority: P0
test_level: e2e
preconditions:
  - 项目有 "Zone A"（含 SE1、SE2、SE3），SE4 存在但不属于任何区域

steps:
  given: 管理员打开图纸 [Assign] 弹框，Assigned 为空
  when: |
    1. 通过 [Select by Area] 选择 Zone A，批量 Add（SE1、SE2、SE3 进入 Assigned）
    2. 在 Assigned 列表中点击 SE2 的 [Remove]
    3. 在 Available 列表中找到 SE4，点击 [Add]
  then:
    - Assigned 列表包含 SE1、SE3、SE4，共 3 人
    - SE2 不在 Assigned 列表
    - 两种模式操作结果正确合并，无冲突

cleanup:
  - 取消弹框，不保存
```

---

### TC-003G-010a-01

```yaml
tc_id: TC-003G-010a-01
covers_ac: [AC-003G-010a]
scenario: SC-007
priority: P0
test_level: e2e
preconditions:
  - 项目有 "Zone A"（含 SE1、SE2、SE3），SE4 存在但不属于任何区域
  - 存在一张 ACTIVE 图纸

steps:
  given: 管理员打开图纸 [Assign] 弹框，Assigned 为空
  when: |
    1. 通过 [Select by Area] 选 Zone A，批量 Add（SE1、SE2、SE3 进入 Assigned）
    2. 手动 Remove SE2
    3. 手动 Add SE4
    4. 点击 [Save]
  then:
    - 保存成功
    - Snackbar 提示成功
    - SE1、SE3、SE4 收到图纸分配通知（或系统记录分配）
    - SE2 未收到通知

cleanup:
  - 取消该图纸的 SE 分配
```

---

### TC-003G-011-01

```yaml
tc_id: TC-003G-011-01
covers_ac: [AC-003G-011]
scenario: SC-009
priority: P0
test_level: e2e
preconditions:
  - 项目有 "Zone A"（含 SE1、SE2、SE3）
  - 当前 Assigned 列表为空

steps:
  given: 管理员打开图纸 [Assign] 弹框
  when: |
    1. 点击 [Select by Area ▾]，下拉面板打开
    2. 展开 Zone A，仅勾选 SE1 和 SE3（不勾选 SE2，不勾选区域行）
    3. 点击 [Add Selected (2)]
  then:
    - SE1、SE3 进入 Assigned 列表，计数为 2
    - SE2 不在 Assigned 列表
    - 下拉面板关闭

cleanup:
  - 取消弹框，不保存
```

---

### TC-003G-012-01

```yaml
tc_id: TC-003G-012-01
covers_ac: [AC-003G-012]
scenario: SC-010
priority: P0
test_level: e2e
preconditions:
  - 项目有 "Zone A"（含 SE1、SE2）和 "Zone B"（含 SE3、SE4）
  - 当前 Assigned 列表为空

steps:
  given: 管理员打开图纸 [Assign] 弹框
  when: |
    1. 点击 [Select by Area ▾]，下拉面板打开
    2. 勾选 Zone A 区域行（整区域，SE1 + SE2 全选）
    3. 展开 Zone B，仅勾选 SE3（不勾选 SE4）
    4. 点击 [Add Selected (3)]
  then:
    - SE1、SE2、SE3 进入 Assigned 列表，计数为 3
    - SE4 不在 Assigned 列表
    - 下拉面板关闭

cleanup:
  - 取消弹框，不保存
```

---

### TC-003G-013-01

```yaml
tc_id: TC-003G-013-01
covers_ac: [AC-003G-013]
scenario: SC-010
priority: P1
test_level: unit
preconditions:
  - "Zone A" 含 SE1、SE2、SE3

steps:
  given: 下拉面板已展开，Zone A 下所有 SE 均未选中
  when: |
    1. 仅勾选 Zone A 下的 SE1
  then:
    - Zone A 区域行 Checkbox 显示 indeterminate（半选）态
    - SE2、SE3 的 Checkbox 保持未选状态
    - 底部 [Add Selected (1)] 计数显示 1

cleanup:
  - 无
```

---

### TC-003G-E-001

```yaml
tc_id: TC-003G-E-001
covers_ac: [AC-003G-002]
scenario: SC-E02
priority: P1
test_level: integration
preconditions:
  - 管理员已登录

steps:
  given: 管理员打开 Add New Area 弹窗
  when: |
    1. 不填写 Area Name，直接点击 [Save]
  then:
    - 表单校验提示 Area Name 必填
    - 接口未被调用（或调用后返回 400）
    - 弹窗未关闭
```

---

### TC-003G-E-002

```yaml
tc_id: TC-003G-E-002
covers_ac: [AC-003G-002]
scenario: SC-E03
priority: P1
test_level: integration
preconditions:
  - 管理员已登录
test_data:
  - areaName: 51 个字符的字符串 "AAAA...A"（> 50）

steps:
  given: 管理员打开 Add New Area 弹窗
  when: |
    1. 输入 51 字符区域名，点击 [Save]
  then:
    - 输入框限制最多 50 字符，超出部分无法输入（前端截断）；或接口返回 400
    - 区域未创建
```

---

### TC-003G-E-003

```yaml
tc_id: TC-003G-E-003
covers_ac: [AC-003G-005]
scenario: SC-E07
priority: P1
test_level: e2e
preconditions:
  - 已存在区域 "Zone A"（无绑定 SE），项目内有 SE1

steps:
  given: 管理员打开 "Zone A" 的 Manage SEs 弹框
  when: |
    1. 在 Available 中 Add SE1
    2. 点击 [Cancel] 或右上角关闭图标（不点 Save）
  then:
    - 弹框关闭
    - "Zone A" 的 SE Count 仍为 0（变更未保存）
    - 再次打开弹框，In Area 列表为空
```

---

## 5. 边界值与等价类

### 5.1 Area Name 边界

| 等价类 | 边界值 | 期望 |
|-------|-------|-----|
| 有效 | 1 字符 | 接受，创建成功 |
| 有效 | 50 字符 | 接受，创建成功 |
| 无效 | 0 字符（空） | 拒绝，必填校验 |
| 无效 | 51 字符 | 拒绝，长度超限 |
| 有效 | 含中文 | 接受，正确存储展示 |
| 有效 | 含空格（首尾） | 服务端 trim 后处理 |
| 有效 | 含特殊符号（`-`、`/`、`()`） | 接受 |
| 无效 | 仅空白字符 | 拒绝（trim 后为空） |

### 5.2 文本字段 fuzzing

| 输入 | 期望 |
|-----|-----|
| 含 emoji（如 🅰） | 接受，正确存储和展示 |
| 含中文 | 接受 |
| SQL 注入（`' OR 1=1 --`） | 安全转义，正确存储，不执行 SQL |
| XSS（`<script>alert(1)</script>`） | 安全转义，展示时不执行 |
| 超长（51 字符） | 拒绝 |

### 5.3 areaIds 批量接口边界

| 等价类 | 值 | 期望 |
|-------|---|-----|
| 有效 | 1 个 areaId | 返回该区域 SE 列表 |
| 有效 | 50 个 areaId（上限） | 正常返回并集 |
| 无效 | 0 个（空数组） | 400 |
| 无效 | 51 个（超上限） | 400 |
| 无效 | 含非 UUID 字符串 | 400 |
| 无效 | 含不属于当前项目的 areaId | 403 |

---

## 6. 接口测试（契约测试）

### 6.1 通用契约校验

- 请求 schema 合法 → 响应 schema 合法
- 必填字段缺失 → 400
- 字段类型错误 → 400
- 权限不足 → 403
- 资源不存在 → 404
- 区域名重复 → 409

### 6.2 针对每个 API 的专项接口测试

#### API-003G-01：获取项目区域列表

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003G-01-01 | 正常请求 | 有效 projectId + 管理员 token | 200，返回区域列表 |
| TC-API-003G-01-02 | 无 token | 无 Authorization | 401 |
| TC-API-003G-01-03 | 非管理员 token | 业务人员 token | 403 |
| TC-API-003G-01-04 | 跨项目 projectId | 其他项目 ID | 403 |
| TC-API-003G-01-05 | 无区域 | 新项目 | 200，返回空数组 |

#### API-003G-02：新建区域

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003G-02-01 | 正常新建 | 不重复名称 | 201，返回区域对象 |
| TC-API-003G-02-02 | 重复名称 | 已存在的名称 | 409，AREA_NAME_DUPLICATE |
| TC-API-003G-02-03 | 名称为空 | `areaName: ""` | 400 |
| TC-API-003G-02-04 | 名称超长 | 51 字符 | 400 |
| TC-API-003G-02-05 | 未登录 | 无 token | 401 |
| TC-API-003G-02-06 | 无权限 | 业务人员 token | 403 |

#### API-003G-04：删除区域

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003G-04-01 | 正常删除 | 存在的 areaId | 204 |
| TC-API-003G-04-02 | 重复删除 | 已删除的 areaId | 404 |
| TC-API-003G-04-03 | 不存在的 areaId | 随机 UUID | 404 |
| TC-API-003G-04-04 | 跨项目 areaId | 其他项目区域 ID | 403/404 |

#### API-003G-07：保存区域 SE 绑定

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003G-07-01 | 正常保存（Add） | `{addSEIds:[id1,id2], removeSEIds:[]}` | 200 |
| TC-API-003G-07-02 | 正常保存（Remove） | `{addSEIds:[], removeSEIds:[id1]}` | 200 |
| TC-API-003G-07-03 | 幂等 Add（SE 已存在） | 重复 Add 同一 SE | 200，不重复插入 |
| TC-API-003G-07-04 | addSEIds 含无效 UUID | 非项目 SE | 400 |
| TC-API-003G-07-05 | 无权限 | 业务人员 token | 403 |

#### API-003G-08：按区域批量获取 SE 并集

| TC ID | 描述 | 输入 | 期望 |
|-------|------|-----|-----|
| TC-API-003G-08-01 | 单区域 | 1 个 areaId | 200，返回该区域 SE |
| TC-API-003G-08-02 | 多区域有重叠 SE | Zone A(SE1,SE2,SE3) + Zone B(SE3,SE4) | 200，返回 SE1～SE4（SE3 不重复） |
| TC-API-003G-08-03 | 空 areaIds | `[]` | 400 |
| TC-API-003G-08-04 | 超上限 areaIds | 51 个 ID | 400 |
| TC-API-003G-08-05 | 含已删除区域 ID | 软删除的 areaId | 400/404 |

---

## 7. 性能测试

### 7.1 测试目标

| 指标 | 目标 | 工具 |
|-----|------|-----|
| 区域配置列表加载 | ≤ 1s（P95） | k6 / JMeter |
| Manage SEs 弹框打开 | ≤ 1s（P95） | k6 |
| 批量 SE 获取接口（多区域） | ≤ 500ms（P95） | k6 |
| 批量 Add 区域 SE（本地操作） | 即时（< 16ms） | 浏览器 DevTools |

### 7.2 压测场景

| 场景 ID | 描述 | 并发 | 持续 | 通过条件 |
|--------|------|-----|-----|---------|
| PT-001 | 区域列表接口并发查询 | 50 并发 | 5min | P95 ≤ 1s，错误率 < 1% |
| PT-002 | 批量 SE 获取（5 个区域，每区域 50 SE） | 20 并发 | 3min | P95 ≤ 500ms |
| PT-003 | 保存区域 SE 绑定（diff 50 SE） | 10 并发 | 3min | P95 ≤ 500ms |

---

## 8. 安全测试

### 8.1 OWASP Top 10 覆盖

| 项 | 测试方法 |
|---|---------|
| A01 越权访问 | 用业务人员/SE token 调用区域配置接口，期望 403；跨项目访问，期望 403 |
| A03 注入 | Area Name 字段输入 SQL 注入字符串（`' OR 1=1 --`），验证正确转义 |
| A03 XSS | Area Name 输入 `<script>alert(1)</script>`，验证展示时不执行 |
| A07 鉴权失效 | 过期 token 调用接口，期望 401；伪造 token，期望 401 |
| A09 日志监控 | 删除区域操作后，验证审计日志记录 actor_id、areaId、时间 |

### 8.2 项目专项安全测试

| TC ID | 描述 | 期望 |
|-------|-----|-----|
| TC-SEC-001 | 用户 A（项目 1 管理员）访问项目 2 的区域列表 | 403 |
| TC-SEC-002 | 用户 A 调用删除接口删除项目 2 的区域 | 403/404 |
| TC-SEC-003 | areaIds 传入其他项目区域 ID 调用批量 SE 接口 | 403/400 |

---

## 9. 兼容性测试

| 维度 | 测试矩阵 |
|-----|---------|
| 浏览器 | Chrome 100+、Edge 100+、Safari 15+ |
| 移动端 | 不适用（PC 专属功能） |
| 屏幕分辨率 | 1280×720、1920×1080、2560×1440 |
| 网络 | 正常（100Mbps）、慢速（3G 模拟，区域列表加载超时处理） |

---

## 10. 测试数据

### 10.1 种子数据

| 数据集 | 用途 | 维护方式 |
|-------|------|---------|
| 项目管理员账号 | 权限验证 | fixtures/users.json |
| 普通业务人员账号 | 权限否定验证 | fixtures/users.json |
| SE 账号 × 10 | SE 列表操作 | fixtures/users.json |
| 预配置区域 × 3（Zone A、Zone B、Zone C）含 SE 绑定 | 批量分配测试 | fixtures/areas.json |

### 10.2 测试 fixtures

```
fixtures/
├── valid/
│   ├── area-create-valid.json      # 合法区域名称
│   └── area-se-binding-valid.json  # 合法 SE 绑定 diff
├── invalid/
│   ├── area-name-empty.json        # 空名称
│   ├── area-name-too-long.json     # 超 50 字符
│   └── area-name-duplicate.json    # 重名
└── security/
    ├── sql-injection-area-name.json
    └── xss-area-name.json
```

---

## 11. AC 覆盖矩阵

| AC ID | 描述简述 | 覆盖的 TC | 状态 |
|------|---------|----------|------|
| AC-003G-001 | Area Config 入口仅管理员可见 | TC-003G-001-01、TC-003G-001-02 | TODO |
| AC-003G-002 | 新建区域成功 | TC-003G-002-01、TC-003G-E-001、TC-003G-E-002 | TODO |
| AC-003G-003 | 区域名不可重复 | TC-003G-003-01 | TODO |
| AC-003G-004 | 删除区域同步解绑 SE | TC-003G-004-01 | TODO |
| AC-003G-005 | 为区域添加 SE | TC-003G-005-01、TC-003G-E-003 | TODO |
| AC-003G-006 | 无区域时 [Select by Area] 置灰 | TC-003G-006-01 | TODO |
| AC-003G-007 | 整区域批量 Add SE 正常路径 | TC-003G-007-01 | TODO |
| AC-003G-008 | 按区域批量 Add SE 幂等性 | TC-003G-008-01 | TODO |
| AC-003G-009 | 按区域 Add 后仍可手动调整 | TC-003G-009-01 | TODO |
| AC-003G-010a | 双模式混合使用端到端 | TC-003G-010a-01 | TODO |
| AC-003G-010b | 区域配置页空状态 | TC-003G-002-02 | TODO |
| AC-003G-011 | 下拉面板单独勾选特定 SE | TC-003G-011-01 | TODO |
| AC-003G-012 | 下拉面板整区域 + 单 SE 混合选择 | TC-003G-012-01 | TODO |
| AC-003G-013 | 区域行 indeterminate 半选态 | TC-003G-013-01 | TODO |

---

## 12. 回归测试范围

| 受影响功能 | 影响原因 | 回归用例 |
|----------|---------|---------|
| REQ-003D-pc 图纸 [Assign] 弹框（原有单选流程） | F-006 在 Available 面板新增 UI 元素，需验证原有单选 Add/Remove 不受影响 | 原有 Assign 弹框 P0 用例全套 |
| REQ-003A-pc 图纸管理列表页（顶部操作区） | F-001 新增 [Area Config] 入口按钮，需验证不破坏现有 [Upload] 等按钮布局 | 图纸上传、筛选功能冒烟 |

---

## 13. 测试环境

| 环境 | 用途 | 数据 |
|-----|------|-----|
| local | 开发自测 | mock（MSW） |
| dev | 集成联调、E2E | 共享种子数据 |
| staging | 验收、压测 | 接近生产的匿名化数据 |

---

## 14. 验收条件

QA 测试完成的判定：

- [ ] 所有 P0 AC 100% 覆盖
- [ ] P0 用例全通过
- [ ] P1 用例 ≥ 95% 通过
- [ ] 性能指标达标（参见 §7.1）
- [ ] 安全扫描无 high 级问题
- [ ] 兼容性矩阵（Chrome、Edge、Safari）全绿
- [ ] 回归用例（REQ-003D-pc Assign 流程、REQ-003A-pc 列表页）全绿
- [ ] AC 覆盖矩阵 §11 全部 ✅

---

## 15. 已知问题与遗留

| 问题 ID | 描述 | 严重度 | 处理方案 |
|--------|------|-------|---------|
| OQ-001 | SE 被加入区域时的站内通知逻辑 PM 未确认，相关通知测试暂缓 | Low | PM 确认后补充 TC |
| OQ-005 | 删除区域时是否校验未完成图纸分配，当前版本直接允许删除，后续若加限制需补充 TC | Medium | 后续迭代补充 |

---

## 16. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-28 | agent | 初稿，覆盖 AC-003G-001 ～ AC-003G-010b 全部测试用例 |
| 0.2.0 | 2026-05-28 | agent | 同步 REQ-003G-pc v0.3.0：新增 TC-003G-011-01（单独选 SE）、TC-003G-012-01（混合选择）、TC-003G-013-01（indeterminate 态）；更新 AC-003G-007 描述；AC 覆盖矩阵补全至 AC-003G-013 |
