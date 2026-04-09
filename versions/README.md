# 版本快照说明

此目录用于存储每个版本的完整快照。

## 目录结构

```
versions/
├── v1.0.0/                    # 版本1.0.0快照
│   ├── requirements/          # 该版本的需求文档
│   ├── designs/               # 该版本的设计文档
│   ├── specifications/        # 该版本的各角色说明文档
│   │   ├── ui/
│   │   ├── frontend/
│   │   ├── backend/
│   │   └── qa/
│   └── CHANGELOG.md           # 该版本的变更日志
├── v1.1.0/
│   ├── requirements/
│   ├── designs/
│   ├── specifications/
│   └── CHANGELOG.md
└── README.md                  # 本说明文件
```

## 使用方式

### 创建新版本快照
1. 在 `versions/` 下创建新文件夹，命名为 `vX.X.X`
2. 从主工作区复制该版本的所有相关文档到该版本文件夹
3. 编写 `CHANGELOG.md` 记录该版本的变更
4. 在 `VERSIONS.md` 中更新版本发布历史

### 查看版本历史
- 查看各版本的需求文档: `versions/vX.X.X/requirements/`
- 查看各版本的设计文档: `versions/vX.X.X/designs/`
- 查看各版本的变更日志: `versions/vX.X.X/CHANGELOG.md`

### 对比版本差异
通过对比不同版本文件夹下的文档，可以了解版本之间的差异。
