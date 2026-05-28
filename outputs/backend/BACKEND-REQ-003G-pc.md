---
doc_type: backend_spec
req_id: REQ-003G-pc
version: 0.1.0
status: draft
generated_from: REQ-003G-pc@0.2.0
data_contract_ref: data-contract-003G-pc.md@0.1.0
generated_at: 2026-05-28
owner: ""
---

# 后端开发说明：PC 端 — 图纸管理区域配置与双模式 SE 分配（单选 + 按区域批量）

> **本文档供后端开发工程师及其 agent 使用**。
>
> ⚠️ **重要约定**：
> - 数据模型与 API 字段定义完全引用 `data-contract-003G-pc.md`，本文档不重复定义。
> - 业务规则、状态转换、事务边界、幂等性、审计要求在本文档详述。
> - 代码注释中必须标注覆盖的 AC ID。

---

## 0. 溯源块

| 项 | 值 |
|---|---|
| 来源需求 | REQ-003G-pc @ v0.2.0 |
| 数据契约 | data-contract-003G-pc.md @ v0.1.0 |
| 覆盖 Story | US-003G-001、US-003G-002、US-003G-003、US-003G-004 |
| 覆盖 AC | AC-003G-001 ～ AC-003G-010b |

---

## 1. 功能概述

本次后端需交付：
1. **DrawingArea（图纸区域）CRUD 接口**：支持项目管理员在项目维度下新建、查询、编辑、软删除区域。
2. **DrawingAreaSE（区域-SE 绑定）管理接口**：支持查询区域内 SE 列表和保存区域 SE 绑定变更（diff 模式）。
3. **按区域批量查询 SE 接口**：供前端 [Select by Area] 功能使用，返回多个区域 SE 的并集（自动去重）。
4. **权限控制**：所有区域操作接口须校验 `drawing:area-config` 权限，项目隔离。

---

## 2. 技术栈

### 2.1 已有技术栈（继承）

- 语言：Java / Node.js（以项目既有为准）
- 框架：Spring Boot / Express（以项目既有为准）
- 数据库：PostgreSQL
- 缓存：Redis
- 认证：JWT（Bearer Token）
- 迁移工具：Flyway

### 2.2 本需求新增依赖

| 库 | 用途 | 评估 |
|---|------|------|
| 无新增 | 全部在现有技术栈内实现 | — |

---

## 3. 业务逻辑

### 3.1 获取项目区域列表（覆盖 AC-003G-001、AC-003G-010b）

```
1. 校验权限：JWT 解析 userId，校验 userId 在 projectId 下具有 drawing:area-config 权限；无权限 → 403
2. 校验 projectId 存在且当前用户归属该项目；否则 → 403
3. 查询 drawing_areas WHERE project_id = :projectId AND deleted_at IS NULL
4. 按 created_at DESC 排序
5. 关联查询每个区域的 SE 数量（COUNT drawing_area_ses WHERE area_id = :areaId）
6. 支持可选 name 模糊搜索过滤（ILIKE %:q%）
7. 返回区域列表（含 seCount）
```

### 3.2 新建区域（覆盖 AC-003G-002、AC-003G-003）

```
1. 校验权限：同 3.1
2. 校验输入：
   - areaName 非空，最大 50 字符
   - 同 projectId 下无同名区域（deleted_at IS NULL 范围内，区分大小写或不区分见业务决策）
   - 若重名 → 返回 409，业务码 AREA_NAME_DUPLICATE
3. INSERT drawing_areas(area_id=UUID, project_id, area_name, created_by=userId, created_at=NOW())
4. 写审计日志：action=AREA_CREATED, resource_id=areaId
5. 返回新创建的区域对象（含 areaId、areaName、seCount=0、createdAt）
```

### 3.3 编辑区域名称（覆盖 AC-003G-002、AC-003G-003）

```
1. 校验权限：同 3.1
2. 校验 areaId 存在于该项目且未软删除；否则 → 404
3. 校验新 areaName 非空，最大 50 字符
4. 校验同 projectId 下无其他区域（areaId ≠ 当前）同名（deleted_at IS NULL）；重名 → 409 AREA_NAME_DUPLICATE
5. UPDATE drawing_areas SET area_name=:areaName, updated_at=NOW() WHERE area_id=:areaId
6. 写审计日志：action=AREA_UPDATED
7. 返回更新后的区域对象
```

### 3.4 删除区域（覆盖 AC-003G-004）

```
1. 校验权限：同 3.1
2. 校验 areaId 存在于该项目且未软删除；否则 → 404
3. （可选，见 OQ-005）检查是否有未完成图纸分配依赖该区域 SE；当前版本跳过此检查，直接允许删除
4. BEGIN TRANSACTION
   a. UPDATE drawing_areas SET deleted_at=NOW() WHERE area_id=:areaId  （软删除区域）
   b. DrawingAreaSE 记录无需删除，查询时 JOIN 过滤 deleted_at IS NULL 的区域即可失效
   c. （注意：不硬删除 drawing_area_ses，保留历史记录，查询层过滤）
5. COMMIT
6. 写审计日志：action=AREA_DELETED
7. 返回 204 No Content
```

> ⚠️ 软删除后 `drawing_area_ses` 记录保留但查询时自动失效（通过 JOIN drawing_areas WHERE deleted_at IS NULL 过滤）。

### 3.5 获取区域 SE 列表（In Area）（覆盖 AC-003G-005）

```
1. 校验权限：同 3.1
2. 校验 areaId 归属 projectId 且未删除；否则 → 404
3. 查询 drawing_area_ses JOIN users WHERE area_id = :areaId
   - 过滤非活跃用户（user.status = 'active' AND user 未被移出该项目）
4. 可选 name 模糊搜索（ILIKE）
5. 返回 SE 列表（userId, displayName, roleTag）
```

### 3.6 获取项目可用 SE 列表（Available，供 ManageSEs 弹框右栏使用）

```
1. 校验权限：同 3.1
2. 查询项目成员列表中角色为 SiteEngineer 且 status=active 的用户
3. 排除已在该区域的 SE（LEFT JOIN drawing_area_ses，过滤 area_id = :areaId）
4. 返回可用 SE 列表
```

### 3.7 保存区域 SE 绑定（批量 diff）（覆盖 AC-003G-005）

```
1. 校验权限：同 3.1
2. 校验 areaId 归属 projectId 且未删除
3. 接收 body: { addSEIds: string[], removeSEIds: string[] }
4. 校验 addSEIds 中每个 userId 均为项目内有效 SE；否则 → 400
5. BEGIN TRANSACTION
   a. INSERT drawing_area_ses(area_id, user_id, added_by, added_at) FOR EACH addSEId
      - ON CONFLICT (area_id, user_id) DO NOTHING（幂等）
   b. DELETE FROM drawing_area_ses WHERE area_id=:areaId AND user_id IN (:removeSEIds)
6. COMMIT
7. 写审计日志：action=AREA_SE_UPDATED, payload={added, removed}
8. 返回更新后的 In Area SE 列表（或 200 OK with updated count）
```

### 3.8 按区域批量获取 SE 列表（供 [Select by Area] 使用）（覆盖 AC-003G-006、AC-003G-007、AC-003G-008）

```
1. 校验权限：校验 userId 具有图纸分配权限（drawing:assign 或 drawing:area-config）
2. 接收 body/query: { areaIds: string[] }
3. 校验所有 areaIds 均归属 projectId 且未删除
4. 查询 drawing_area_ses JOIN users WHERE area_id IN (:areaIds) AND user.status = 'active'
5. 对结果按 userId 去重（取 DISTINCT userId），返回 SE 并集列表
6. 返回：{ ses: [{userId, displayName, ...}] }
   - 注意：此接口只返回 SE 列表供前端展示，前端负责与已 Assigned 列表做二次去重（幂等合并）
```

---

## 4. 状态机实现

本需求无复杂状态机。DrawingArea 仅有两态：
- `active`（未删除，`deleted_at IS NULL`）
- `deleted`（软删除，`deleted_at IS NOT NULL`）

唯一转换：`active → deleted`（软删除），不可逆。实现直接在 Service 层处理，无需独立状态机类。

---

## 5. 事务边界

| 操作 | 事务边界 | 说明 |
|-----|---------|------|
| 新建区域 | ✅ 单事务 | INSERT + 审计日志 |
| 编辑区域名称 | ✅ 单事务 | UPDATE + 审计日志 |
| 删除区域 | ✅ 单事务 | UPDATE deleted_at（区域软删除）+ 审计日志；drawing_area_ses 无需在同一事务删除 |
| 保存区域 SE 绑定（批量 diff） | ✅ 单事务 | INSERT（幂等）+ DELETE + 审计日志，必须原子 |

---

## 6. 幂等性要求

| API | 幂等键 | 策略 |
|-----|-------|-----|
| 新建区域 | projectId + areaName（业务唯一约束） | DB 唯一索引（project_id, lower(area_name)）防重复，409 返回 |
| 编辑区域 | areaId | 幂等（相同值 UPDATE 无副作用） |
| 删除区域 | areaId | 软删除幂等：已删除再次调用 → 404 |
| 保存区域 SE 绑定 | (area_id, user_id) | DB 唯一约束 + `ON CONFLICT DO NOTHING` |
| 按区域批量获取 SE | 无状态变更 | GET/POST 纯查询，天然幂等 |

---

## 7. 异步任务与事件

本需求无异步任务队列。

**可选副作用**（见 OQ-001，PM 确认后实现）：
- SE 被加入区域时，若 PM 确认需要站内通知，则在保存区域 SE 绑定后发布 `DrawingAreaSEAdded` 事件，由通知服务消费。
- 发布方式：Outbox 模式，在同一事务内写 `outbox_events`，独立调度器发送。

---

## 8. 数据库设计

### 8.1 表结构

```sql
-- 引用 data-contract §1.1
CREATE TABLE drawing_areas (
  area_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id   UUID NOT NULL REFERENCES projects(project_id),
  area_name    VARCHAR(50) NOT NULL,
  created_by   UUID NOT NULL REFERENCES users(user_id),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at   TIMESTAMPTZ
);

-- 唯一约束：同项目下区域名不重复（活跃区域）
-- 注意：PostgreSQL 部分唯一索引（仅 deleted_at IS NULL 范围）
CREATE UNIQUE INDEX uidx_drawing_areas_project_name
  ON drawing_areas(project_id, lower(area_name))
  WHERE deleted_at IS NULL;

CREATE INDEX idx_drawing_areas_project_id
  ON drawing_areas(project_id)
  WHERE deleted_at IS NULL;

-- 引用 data-contract §1.2
CREATE TABLE drawing_area_ses (
  id        BIGSERIAL PRIMARY KEY,
  area_id   UUID NOT NULL REFERENCES drawing_areas(area_id),
  user_id   UUID NOT NULL REFERENCES users(user_id),
  added_by  UUID NOT NULL REFERENCES users(user_id),
  added_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (area_id, user_id)
);

CREATE INDEX idx_drawing_area_ses_area_id ON drawing_area_ses(area_id);
CREATE INDEX idx_drawing_area_ses_user_id ON drawing_area_ses(user_id);
```

### 8.2 数据库迁移

- 工具：Flyway
- 命名：`V{时间戳}__{描述}.sql`
  - `V20260528001__create_drawing_areas.sql`
  - `V20260528002__create_drawing_area_ses.sql`
- 原则：只前进不回退，新增字段必须有默认值或允许 NULL
- 索引创建用 `CONCURRENTLY`（生产环境）

---

## 9. 缓存策略

| 数据 | 缓存层 | TTL | 失效时机 |
|-----|-------|-----|---------|
| 项目区域列表（含 SE Count） | Redis | 5min | 区域新建/编辑/删除时主动失效 |
| 区域 SE 列表 | Redis | 5min | 保存区域 SE 绑定时主动失效 |
| 项目 SE 可用列表 | 无缓存（频繁变动） | — | 每次请求实时查询 |

**缓存 Key**：
```
drawing:area:list:{projectId}
drawing:area:se:{areaId}
```

**缓存击穿防护**：大量并发请求同一 Key 时使用 Redis Mutex（SETNX + TTL）防止同时回源。

---

## 10. 并发控制

| 场景 | 策略 | 说明 |
|-----|------|------|
| 区域名称唯一性校验 + 插入 | DB 唯一索引（部分索引）| 依赖 DB 约束而非应用层 check-then-act，避免竞态 |
| 批量 diff 保存区域 SE 绑定 | DB 唯一约束 + ON CONFLICT DO NOTHING | 防并发重复插入 |
| 删除区域并发 | 软删除幂等，重复调用返回 404 | 无需额外锁 |

---

## 11. 审计日志

| 操作 | 必须审计 | 审计内容 |
|-----|---------|---------|
| 新建区域 | ✅ | actor_id, action=AREA_CREATED, resource_type=DrawingArea, resource_id=areaId, payload={areaName} |
| 编辑区域名称 | ✅ | actor_id, action=AREA_UPDATED, resource_id=areaId, payload={oldName, newName} |
| 删除区域 | ✅ | actor_id, action=AREA_DELETED, resource_id=areaId |
| 保存区域 SE 绑定 | ✅ | actor_id, action=AREA_SE_UPDATED, resource_id=areaId, payload={added:[userId], removed:[userId]} |

**保留期限**：随项目归档（≥ 3 年）。

---

## 12. 性能要求

| 接口 | P95 延迟 | 备注 |
|-----|---------|------|
| GET 项目区域列表 | ≤ 200ms | 单项目 ≤ 100 区域，无需分页 |
| POST 新建/编辑区域 | ≤ 300ms | 含唯一性校验 |
| DELETE 区域（软删除） | ≤ 200ms | |
| GET/POST 区域 SE 列表 | ≤ 300ms | 单区域 ≤ 200 SE |
| POST 按区域批量获取 SE 并集 | ≤ 500ms | 多区域 IN 查询 + 去重 |
| POST 保存区域 SE 绑定 | ≤ 500ms | 批量 diff 写入 |

**关键优化**：
- 区域列表 SE Count 通过子查询或 JOIN COUNT 一次性返回，禁止 N+1。
- 按区域批量获取 SE 使用 `SELECT DISTINCT ... WHERE area_id IN (...)` 单条 SQL，禁止循环查询。

---

## 13. 安全要求

### 13.1 输入校验

- 所有外部输入走 DTO 校验（JSR-303 / class-validator）
- `areaName`：非空、最大 50 字符，trim 后校验
- `areaIds`（批量接口）：数组元素均为合法 UUID，最大 50 个（防滥用）
- `addSEIds` / `removeSEIds`：均为合法 UUID，最大 200 个
- SQL 参数化查询，禁止字符串拼接

### 13.2 鉴权与权限

- 鉴权：JWT Bearer Token，Spring Security / Express middleware 统一校验
- **每个区域配置接口**须校验 `drawing:area-config` 权限；无权限 → 403
- **行级权限**：所有查询强制注入 `project_id = :projectId` 过滤，防止跨项目读取
- **按区域批量获取 SE 接口**：校验调用者具有 `drawing:assign` 或 `drawing:area-config` 权限之一

### 13.3 限流

- 用户级：600 req/min（通用）
- 批量接口（按区域批量获取 SE）：120 req/min（单用户）

---

## 14. 可观测性

### 14.1 日志

- 结构化 JSON 日志，必含 `trace_id`、`request_id`、`user_id`、`project_id`
- INFO：区域 CRUD 成功、SE 绑定保存成功
- WARN：区域名重复（业务拒绝）
- ERROR：事务失败、DB 异常

### 14.2 Metrics

- `drawing_area_created_total`（Counter）：新建区域次数
- `drawing_area_deleted_total`（Counter）：删除区域次数
- `drawing_area_se_saved_total`（Counter）：保存 SE 绑定次数（labels: added_count, removed_count）
- `drawing_area_batch_se_fetch_duration_seconds`（Histogram）：批量获取 SE 耗时

### 14.3 Tracing

- 全链路 OpenTelemetry trace，跨服务 trace_id 透传
- 重点 Span：数据库查询、Redis 缓存操作

### 14.4 告警

| 告警项 | 阈值 | 等级 |
|-------|------|-----|
| 区域 CRUD 接口 P95 延迟 | > 1s | Warning |
| 批量 SE 获取接口错误率 | > 5% | Critical |
| 数据库连接池耗尽 | 等待队列 > 10 | Critical |

---

## 15. AC 覆盖检查表

| AC ID | 实现位置 | 测试覆盖 | 状态 |
|------|---------|---------|------|
| AC-003G-001 | 权限中间件 + `drawing:area-config` 校验 | 集成测试（403 场景） | TODO |
| AC-003G-002 | `AreaController.createArea` + `AreaService` | 单元 + 集成 | TODO |
| AC-003G-003 | `AreaService` 唯一性校验 → 409 | 单元 + 集成 | TODO |
| AC-003G-004 | `AreaService.deleteArea`（软删除，drawing_area_ses 级联失效） | 单元 + 集成 | TODO |
| AC-003G-005 | `AreaSEService.saveSEBindings`（diff 写入） | 单元 + 集成 | TODO |
| AC-003G-006 | `AreaController.listAreas`（空列表正常返回） | 单元 | TODO |
| AC-003G-007 | `AreaSEService.getBatchSEsByAreaIds`（并集去重） | 单元 | TODO |
| AC-003G-008 | `AreaSEService.getBatchSEsByAreaIds`（DISTINCT 去重） | 单元 | TODO |
| AC-003G-009 | 无后端逻辑（纯前端本地状态） | — | N/A |
| AC-003G-010a | `AssignService.saveAssignment`（最终保存完整列表） | 集成 | TODO |
| AC-003G-010b | `AreaController.listAreas`（空列表返回 200 + 空数组） | 单元 | TODO |

---

## 16. 测试要求

| 层级 | 框架 | 范围 | 覆盖率目标 |
|-----|------|------|-----------|
| 单元 | JUnit 5 / Jest | 业务逻辑、唯一性校验、软删除、SE 去重 | ≥ 80% |
| 集成 | Testcontainers（PG） | DB 操作、并发唯一约束、事务回滚 | 核心路径 100% |
| 契约 | schemathesis / Pact | 与前端接口契约 | 全部新增接口 |

---

## 17. 部署与回滚

- 部署方式：K8s 滚动更新
- DB schema 变更：先部署兼容版本（新表已存在，旧代码不访问），再切流量
- 回滚：
  - 应用层：回滚镜像版本
  - DB：新表/索引保留（不删除），Feature Flag 隐藏入口
- Feature Flag：`feature.drawing-area-config`（控制区域配置入口）、`feature.assign-select-by-area`（控制 Select by Area 按钮）

---

## 18. 验收条件

后端开发完成的判定：

- [ ] 所有 AC 在 §15 表中标记完成
- [ ] 单元 + 集成测试通过，覆盖率达标
- [ ] 契约测试与前端通过
- [ ] 压测达到 §12 目标
- [ ] 安全扫描无 high 项
- [ ] DB 迁移脚本已审核，`CONCURRENTLY` 索引可在生产执行
- [ ] 已部署到 dev，可联调

---

## 19. 变更历史

| 版本 | 日期 | 修改人 | 变更摘要 |
|-----|------|-------|---------|
| 0.1.0 | 2026-05-28 | agent | 初稿，覆盖 DrawingArea CRUD、DrawingAreaSE 绑定、按区域批量查询 SE 全部后端逻辑 |
