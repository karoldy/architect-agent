# 🏗️ Architect Agent

> AI 协同的软件架构设计工作空间 — 系统化产出数据库设计、API 契约、系统架构与部署方案

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-active-success.svg)](https://github.com/turbo.su/architect-agent)
[![Design](https://img.shields.io/badge/design-Mermaid-ff69b4.svg)](https://mermaid.js.org)

---

## 这是什么？

Architect Agent 是一个**架构设计工作空间**，由 Claude 作为架构师协同完成软件项目的系统化设计。与普通代码仓库不同，这里的产出不是运行时代码，而是**设计文档、ER 图、API 规范、架构决策记录（ADR）**—— 是软件构建前的"蓝图"。

### 核心理念

| 理念 | 说明 |
|------|------|
| 🤖 **AI 协同设计** | Claude 作为架构师，负责分析需求、设计方案、记录决策 |
| 📐 **结构化产出** | 每个项目遵循统一目录结构，数据库/API/架构/决策 四大支柱 |
| 📊 **图表先行** | 所有设计文档使用 Mermaid 绘制图表，纯文本即可渲染 |
| 📝 **决策可追溯** | 每个关键选型有 ADR 记录，回答"为什么这样建" |
| 🚀 **部署有方** | 默认部署栈：Vercel（前端）+ Render（后端）+ Supabase（数据库） |

---

## 目录结构

```
architect-agent/
├── README.md                  # 本文件
├── CLAUDE.md                  # Claude Code 工作空间配置
├── LICENSE                    # MIT License
├── projects/                  # 📁 各项目的架构设计产出
│   └── <project-name>/
│       ├── README.md          # 项目概述、背景、约束
│       ├── database/
│       │   ├── schema.md      # 表结构与索引策略
│       │   └── er-diagram.md  # ER 图（Mermaid）
│       ├── api/
│       │   ├── graphql/       # GraphQL schema
│       │   └── rest/          # REST API 定义（OpenAPI 3.1）
│       ├── architecture/
│       │   ├── system.md      # 系统架构总览（领域模型 + C4 图）
│       │   ├── components.md  # 组件/服务说明
│       │   └── deployment.md  # 部署拓扑
│       └── decisions/         # 架构决策记录（ADR）
│           └── 0001-xxx.md
├── templates/                 # 📋 设计文档模板（快速启动新项目）
└── .claude/
    └── agents/                # 自定义 Agent 定义
```

---

## 当前项目

| 项目 | 状态 | 简介 |
|------|------|------|
| [MindFlow](projects/mindflow/) | 🟢 架构设计阶段 | 个人知识管理笔记系统 — Offline-First、双向链接、知识图谱、多端同步 |

---

## 设计范畴

### 每个项目覆盖的领域

| 领域 | 产出物 | 工具/格式 |
|------|--------|----------|
| **数据库设计** | 表结构、ER 图、索引策略、迁移方案 | Mermaid `erDiagram` |
| **API 设计** | GraphQL schema、RESTful OpenAPI 3.1 规范、错误码 | OpenAPI / GraphQL SDL |
| **系统架构** | 领域模型（DDD）、C4 模型图、组件拓扑、技术选型 | Mermaid `flowchart` / `sequenceDiagram` |
| **部署方案** | 部署拓扑图、CI/CD 流水线、成本预估 | Mermaid `flowchart` |
| **决策记录** | ADR（Architecture Decision Records） | Markdown |

### 常用图表速查

| 场景 | Mermaid 语法 | 示例用途 |
|------|-------------|---------|
| ER 图 | `erDiagram` | 实体关系建模 |
| 时序图 | `sequenceDiagram` | API 调用流程、同步协议 |
| 架构图 | `flowchart TB/LR` | 系统拓扑、组件关系 |
| 状态图 | `stateDiagram-v2` | 同步状态机、编辑器状态 |
| C4 模型 | `flowchart` 近似 | 上下文图、容器图 |

---

## 工作流

```
理解需求 ──→ 需求澄清 ──→ 方案设计 ──→ 评审迭代 ──→ 定稿输出
  │              │              │              │              │
  │ 梳理业务      │ 对模糊点      │ 产出数据库    │ 根据反馈      │ 最终设计文档
  │ 用户角色      │ 主动提问      │ API 架构      │ 调整方案      │ + 图 + ADR
  │ 核心流程      │ 不臆测        │ 设计文档      │               │
```

1. **理解需求** — 梳理业务场景、用户角色、核心流程
2. **需求澄清** — 对模糊点主动提问，不臆测
3. **方案设计** — 产出数据库 schema、API 定义、架构图、ADR
4. **评审迭代** — 根据反馈调整，记录决策变更
5. **定稿输出** — 生成最终设计文档，交付给工程团队

---

## 设计原则

- **务实优先** — 选型基于实际需求，避免过度设计。MVP 优先，渐进演进
- **图表先行** — 一张好图胜过千言万语。所有文档必须包含 Mermaid 图表
- **决策可审查** — 每个关键技术选型都有 ADR，记录上下文、备选方案、后果
- **标准化** — 命名、格式、图表风格跨项目保持一致
- **部署默认栈** — 无特殊要求时，统一使用 Vercel + Render + Supabase

---

## 快速开始

### 启动新项目

```bash
# 1. 从模板创建项目目录
cp -r templates/ projects/<new-project>/

# 2. 编辑项目 README，填入项目背景与需求
$EDITOR projects/<new-project>/README.md

# 3. 告知 Claude 项目需求，它会：
#    - 分析需求，提出澄清问题
#    - 产出 database/schema.md + er-diagram.md
#    - 设计 API 契约（REST OpenAPI 或 GraphQL）
#    - 绘制架构图（C4 上下文图 + 容器图）
#    - 编写 ADR 记录关键决策
```

### 模板文件

| 模板 | 用途 |
|------|------|
| `templates/README.md` | 项目概述模板 |
| `templates/database-schema.md` | 表结构设计模板 |
| `templates/er-diagram.md` | ER 图模板 |
| `templates/openapi.yaml` | REST API OpenAPI 模板 |
| `templates/graphql-schema.graphql` | GraphQL schema 模板 |
| `templates/system-architecture.md` | 系统架构总览模板 |
| `templates/components.md` | 组件/服务说明模板 |
| `templates/deployment.md` | 部署拓扑模板 |
| `templates/adr.md` | 架构决策记录模板 |

---

## 可用 Agent

仓库配置了专门的 Agent 用于复杂架构任务：

| Agent | 专长 | 何时使用 |
|-------|------|---------|
| 🏗️ **Backend Architect** | 数据库架构、API 设计、云基础设施 | 设计 schema、定义 API 端点、规划部署方案 |
| 🏛️ **Software Architect** | DDD 领域建模、架构模式选型、ADR | 划分限界上下文、架构风格决策、权衡分析 |

> **分工原则**：Backend Architect 回答"怎么建"，Software Architect 回答"为什么这样建"。

---

## 部署平台默认选型

除非项目有特殊要求，所有架构设计中的部署方案默认采用：

| 层级 | 平台 | 典型场景 |
|------|------|---------|
| 🖥️ **前端** | [Vercel](https://vercel.com) | Web SPA/SSR 托管、全球 CDN、Preview Deployments |
| ⚙️ **后端** | [Render](https://render.com) | Go/Node/Python API 服务、容器化部署、自动 HTTPS |
| 🗄️ **数据库** | [Supabase](https://supabase.com) | PostgreSQL、Auth (OAuth 2.0)、Storage、RLS |

---

## 技术栈偏好

在架构设计中优先考虑的技术选型：

| 层级 | 首选 | 备选 | 备注 |
|------|------|------|------|
| 前端框架 | React / TypeScript | Vue / Svelte | 生态最完善 |
| 移动端 | SwiftUI + Jetpack Compose | React Native / Flutter | 追求原生体验时 |
| 后端语言 | Go | TypeScript (Node/Bun) | 性能 + 简洁 |
| API 风格 | REST（OpenAPI 3.1） | GraphQL | 按需选择，MVP 优先 REST |
| 数据库 | PostgreSQL | SQLite（本地）/ MySQL | 关系型优先 |
| 缓存 | Redis | — | 需要时才引入 |
| 消息队列 | — | Redis / RabbitMQ | MVP 阶段尽量避免 |

---

## 贡献

本仓库由 Claude（AI 架构师）与人类协同维护。工作方式：

1. 在 `projects/` 下创建新项目目录
2. 填写项目 README 模板，描述需求和约束
3. Claude 分析需求并产出设计文档
4. 人类审查设计，提出修改意见
5. Claude 迭代优化，直到设计定稿

---

## License

[MIT](LICENSE) © 2026 Architect Agent
