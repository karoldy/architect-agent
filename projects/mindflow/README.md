# MindFlow — 个人知识管理型笔记本

**状态**：架构设计阶段
**来源**：`business-manager-agent/projects/personal-notebook/`
**架构负责人**：待分配
**创建日期**：2026-07-02

---

## 1. 项目概述

MindFlow 是一款面向知识工作者的个人笔记工具，核心价值主张：**让记录像 Apple Notes 一样轻量快速，让知识沉淀像 Obsidian 一样结构化**。

### 核心差异化能力

| 维度 | 目标体验 |
|------|----------|
| 记录速度 | 从 App 打开到开始输入 < 2 秒 |
| 知识关联 | `[[双向链接]]` 原生支持，自动维护反向链接 |
| 知识发现 | 力导向知识图谱，可视化笔记网络 |
| 离线能力 | Offline-First 架构，无网环境下完整可用 |
| 多端同步 | Web + iOS + Android，P95 同步延迟 < 3 秒 |

### 目标用户

- **知识积累者**（产品经理、设计师）：每天 5-8 条笔记，痛点"记过的东西找不回来"
- **研究型写作者**（研究员、学者）：需要概念关联，痛点"手机记录电脑对接不上"
- **轻量创作者**（独立开发者、内容创作者）：关注记录流畅度，痛点"碎片想法无法串联"

---

## 2. 技术约束与关键决策

> 以下决策已在 PRD 中明确，架构设计需在此基础上展开。

| 决策 | 选择 | 理由 |
|------|------|------|
| 编辑器方案 | Markdown 块级编辑器（非富文本） | 实现复杂度低，与"快速记录"定位一致 |
| 数据架构 | 本地优先（Offline-First），SQLite 为真理源 | 离线可用，数据主权 |
| 冲突策略 | LWW（Last-Writer-Wins），MVP 阶段 | 降低实现复杂度，v2 考虑 CRDT |
| 跨平台方案 | 各端原生开发（非 RN/Flutter） | 极致性能和原生体验，尤其文本编辑 |
| 开发顺序 | Web 优先 → iOS 跟随 → Android 追赶 | Web 调试效率最高，iOS 用户付费意愿强 |
| 知识图谱 | 力导向图（Fruchterman-Reingold），本地渲染 | 动态适应关系变化，无需服务端预计算 |
| 全文搜索 | SQLite FTS5 本地索引 | 离线可用，无需服务端搜索 |
| 同步协议 | 增量同步，本地变更队列 → 云端 | 仅传输变更部分，降低带宽和延迟 |

### 技术栈

| 端 | 技术 |
|----|------|
| Web | React / TypeScript |
| iOS | SwiftUI |
| Android | Kotlin / Jetpack Compose |
| 后端 | 待决策（自建 vs Supabase/Firebase） |
| 本地存储 | SQLite + FTS5 |
| 认证 | OAuth 2.0 (Google/Apple) + Email/密码 |
| Markdown 解析 | Web: unified/remark, iOS: Down, Android: Markwon |
| 知识图谱 | Web: D3.js, iOS: GraphKit, Android: 待定（WebView 过渡） |

---

## 3. 待解决的关键架构问题

| # | 问题 | 负责人 | 截止 | 架构影响 |
|---|------|--------|------|----------|
| Q1 | 数据存储方案：自建 vs Supabase/Firebase | 工程负责人 | W2 | 决定后端 API 形态、数据库选型、运维成本 |
| Q2 | 离线冲突策略细节：LWW 字段级行为、冲突日志格式 | 工程负责人 | W4 | 决定同步引擎设计、版本历史存储方案 |
| Q3 | 定价模型：免费版 vs 付费版功能边界 | 产品经理 | W6 | 影响 feature flag 设计、用户表结构 |
| Q4 | 知识图谱布局算法选型与性能基线 | 前端/客户端 TL | W8 | 影响图谱渲染架构和性能优化策略 |
| Q5 | 数据隐私合规（GDPR/个保法） | 法务 + PM | W4 | 影响数据加密、存储地域、删除策略 |

---

## 4. 设计产出物

| 领域 | 文档 | 状态 |
|------|------|------|
| 数据库设计 | `database/schema.md` | ✅ 已完成 |
| ER 图 | `database/er-diagram.md` | ✅ 已完成 |
| API 设计 (REST) | `api/rest/openapi.yaml` | ✅ 已完成 |
| 同步协议 | `architecture/sync-protocol.md` | ✅ 已完成 |
| API 设计 (GraphQL) | `api/graphql/schema.graphql` | ⏸️ 暂缓 (P2 — MVP 仅需 REST) |
| 系统架构总览 (领域模型 + 上下文图 + 架构风格选型 + C4 图) | `architecture/system.md` | ✅ 已完成 |
| 组件/服务说明 (组件拓扑 + 引擎详解 + 数据流) | `architecture/components.md` | ✅ 已完成 |
| 部署拓扑 | `architecture/sync-protocol.md` (同步协议已覆盖云-端交互) | ✅ 已完成 |
| ADR-0001 — 数据存储方案选型 | `decisions/0001-data-storage-strategy.md` | ✅ 已完成 |
| ADR-0002 — Offline-First 同步架构 | `decisions/0002-offline-first-sync.md` | ✅ 已完成 |
| ADR-0003 — LWW 冲突策略适用性分析 | `decisions/0003-lww-conflict-strategy.md` | ✅ 已完成 |

---

## 5. 时间线（架构视角）

| 里程碑 | 周次 | 日期 | 架构交付物 |
|--------|------|------|-----------|
| 技术架构评审通过 | W6 | 2026-08-11 | 系统架构设计文档、技术选型决策记录、API 规范 |
| 后端基础设施就绪 | W6 | 2026-08-11 | API 端点可用、认证系统可跑、数据库 Schema 确认 |
| 核心编辑引擎完成 (Web) | W10 | 2026-09-08 | 本地存储方案验证、同步引擎原型 |
| Internal Alpha | W12 | 2026-09-22 | 架构验证、性能基线数据 |

---

## 6. 相关文档

- [PRD](../../business-manager-agent/projects/personal-notebook/prd.md)
- [用户故事](../../business-manager-agent/projects/personal-notebook/user-stories.md)
- [功能清单](../../business-manager-agent/projects/personal-notebook/feature-list.md)
- [待办事项](../../business-manager-agent/projects/personal-notebook/todo.md)

## 7. 本任务完成的交付物

| 文件 | 说明 |
|------|------|
| `database/schema.md` | 完整的表结构定义，包含本地 SQLite 与云端 PostgreSQL 的差异说明、索引策略、FTS5 全文索引虚拟表定义、约束、外键关系、迁移策略 |
| `database/er-diagram.md` | Mermaid ER 图，包含所有实体关系、逐表字段说明、生命周期、关联关系、数据量预估 |
| `api/rest/openapi.yaml` | OpenAPI 3.1 规范，包含 Auth(注册/登录/Token/OAuth/密码重置)、Notes(CRUD+批量同步+版本)、Notebooks(树状CRUD+排序)、Tags(CRUD+重命名+合并)、Search(服务端搜索)、Sync(Push/Pull/设备注册)、Export/Import、User 共 30+ 端点 |
| `architecture/sync-protocol.md` | 详细同步协议设计，包含设备注册与认证、本地变更日志机制、增量 Push/Pull 格式、首次全量同步特殊处理、LWW 冲突裁决流程、重试与幂等性、时钟偏移处理、同步状态机 |
