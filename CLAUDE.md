# Architect Agent

架构师工作空间 — 为软件项目进行系统化的架构设计。

## 设计范畴

| 领域 | 产出物 |
|------|--------|
| 数据库设计 | 表结构、关系图、索引策略、迁移方案 |
| API 设计 | GraphQL schema、RESTful 接口定义、错误码规范 |
| 系统架构 | 组件图、服务拓扑、技术选型、部署视图 |
| ER 图 | 实体关系模型，使用 Mermaid/PlantUML 表达 |

## 项目组织

```
.
├── CLAUDE.md                 # 仓库配置与规范
├── projects/                 # 各项目的架构设计产出
│   └── <project-name>/
│       ├── README.md         # 项目概述、背景、约束
│       ├── database/
│       │   ├── schema.md     # 表结构与字段说明
│       │   └── er-diagram.md # ER 图（Mermaid）
│       ├── api/
│       │   ├── graphql/      # GraphQL schema 文件
│       │   └── rest/         # REST API 定义（OpenAPI）
│       ├── architecture/
│       │   ├── system.md     # 系统架构总览
│       │   ├── components.md # 组件/服务说明
│       │   └── deployment.md # 部署拓扑
│       └── decisions/        # 架构决策记录 (ADR)
│           └── 0001-use-postgres.md
├── templates/                # 设计文档模板
└── .claude/
    └── agents/               # 自定义 agent 定义
```

## 工作流

1. **理解需求** — 梳理业务场景、用户角色、核心流程
2. **需求澄清** — 对模糊点主动提问，不臆测
3. **方案设计** — 产出数据库、API、架构设计文档
4. **评审迭代** — 根据反馈调整方案
5. **定稿输出** — 生成最终设计文档和图表

## 设计原则

- **务实优先** — 选型基于实际需求，避免过度设计
- **渐进演进** — 架构应该支持增量迭代
- **标准化** — 命名、格式、图表风格保持一致
- **可审查** — 设计决策有记录，方案对比有依据
- **图表先行** — 使用 Mermaid 绘制 ER 图、时序图、架构图，纯文本即可渲染

## 常用图表类型

- ER 图 → `erDiagram` (Mermaid)
- 时序图 → `sequenceDiagram` (Mermaid)
- 架构图 → `graph TD` / `flowchart` (Mermaid)
- 部署图 → `graph LR` (Mermaid)
- C4 模型图 → 可用 PlantUML 或 Mermaid 近似

## 可用 Agent

| Agent | 用途 | 触发场景 |
|-------|------|----------|
| `engineering-backend-architect` 🏗️ | 数据库架构、API 设计、系统可扩展性、云基础设施 | 设计 schema、定义 API 契约、规划服务拓扑、迁移策略 |
| `engineering-software-architect` 🏛️ | 领域建模 (DDD)、架构模式选型、ADR、权衡分析 | 限界上下文划分、架构风格决策、技术选型、演进策略 |

两个 agent 的分工边界：
- **Backend Architect** 回答"怎么建"—— 具体的表结构、API 端点、索引、部署方案
- **Software Architect** 回答"为什么这样建"—— 领域模型、架构风格取舍、决策记录

## 关键约定

- 所有设计文档使用 Markdown 格式
- 图表优先使用 Mermaid（GitHub/GitLab 原生渲染）
- 每个项目有独立的 `decisions/` 目录存放 ADR
- API 设计要同时考虑 GraphQL 和 REST 的适用场景
- 数据库设计包含索引策略和迁移路径
- 复杂设计任务优先委派给对应的 agent，主循环负责协调和汇总
- **部署平台默认选型**（除非项目有特殊要求）：
  - 前端 → Vercel
  - 后端 → Render
  - 数据库 → Supabase
