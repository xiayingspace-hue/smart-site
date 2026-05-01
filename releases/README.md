# 发布快照归档

> **用途**：每次正式版本发布时，将当前 `requirements/` 和 `outputs/` 的快照存入本目录，用于历史版本追溯。
> 与 `VERSIONS.md`（版本规划 + REQ 状态总表）的区别：本目录是**归档**，`VERSIONS.md` 是**活文档**。

## 目录结构

```
releases/
├── v0.1.0/                    # 版本快照
│   ├── requirements/          # 该版本的需求文档副本
│   ├── outputs/               # 该版本的下游文档副本
│   └── CHANGELOG.md           # 该版本的变更摘要
├── v0.2.0/
│   └── ...
└── README.md                  # 本说明文件
```

## 操作流程

### 创建新版本快照
1. 在 `releases/` 下创建新文件夹，命名为 `vX.X.X`
2. 从主工作区复制该版本相关的 `requirements/` 和 `outputs/` 文档
3. 编写 `CHANGELOG.md` 记录该版本包含的 REQ 及主要变更
4. 在 `VERSIONS.md` 的"版本发布历史"中更新状态为 `✅ 已发布`

### 查看历史版本
- 某版本的需求文档：`releases/vX.X.X/requirements/`
- 某版本的下游文档：`releases/vX.X.X/outputs/`
- 某版本的变更摘要：`releases/vX.X.X/CHANGELOG.md`

### 对比版本差异
直接对比不同版本文件夹下的同名文档，或使用 `git diff` 比较对应 tag。
