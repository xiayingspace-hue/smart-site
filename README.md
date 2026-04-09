# MCC Requirements Automation Workflow

团队工作流程自动化系统。产品经理生成需求文档，自动为不同角色生成专有说明文档。

## 工作流程

1. 产品经理编写需求文档 → `requirements/`
2. 自动生成各角色说明文档 → `outputs/`
   - UI设计师文档
   - 前端开发文档
   - 后端开发文档
   - QA测试文档

## 目录结构

```
mcc-requirements/
├── requirements/           # 产品经理编写的需求文档
├── templates/              # 各种文档模板
│   ├── requirement.md      # 需求文档模板
│   ├── ui-spec.md          # UI说明文档模板
│   ├── frontend-spec.md    # 前端开发说明文档模板
│   ├── backend-spec.md     # 后端开发说明文档模板
│   └── qa-spec.md          # QA测试说明文档模板
├── outputs/                # 自动生成的各角色说明文档
│   ├── ui/
│   ├── frontend/
│   ├── backend/
│   └── qa/
├── rules/                  # 转换规则配置
│   ├── ui-rules.md
│   ├── frontend-rules.md
│   ├── backend-rules.md
│   └── qa-rules.md
└── scripts/                # 自动化脚本
```
