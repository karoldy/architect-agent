# ADR-0004: 部署平台选型

## 状态

**已接受**

## 上下文

MindFlow 需要为以下组件选择合适的部署/托管平台：

1. **Web 前端**（React/TypeScript SPA + WASM SQLite）
2. **后端 API**（Go Sync API，处理 Push/Pull 同步请求）
3. **数据库**（PostgreSQL，存储用户数据、变更日志、版本历史）
4. **认证服务**（OAuth 2.0 Google/Apple + Email/密码）
5. **对象存储**（附件、备份文件）

选型约束：

- MVP 阶段成本控制在 $100/月以内
- 支持从 MVP（几百用户）平滑扩展到 GA（10,000+ DAU）
- 前端需要全球 CDN 加速（用户分布全球）
- 后端无需常驻长连接（MVP 为 REST 轮询，v1.5+ 才引入 WebSocket）
- 减少运维负担：团队规模小，优先选择托管服务而非自建基础设施
- 数据库需要支持 PostgreSQL（与本地 SQLite schema 兼容，数据类型映射清晰）

## 决策

### 决策 1：Web 前端托管在 Vercel

**Vercel** 作为 Web 前端（React SPA）的托管平台。

| 评估维度 | Vercel | Netlify | Cloudflare Pages | AWS Amplify |
|---------|--------|---------|-----------------|-------------|
| **边缘网络** | 100+ 全球节点 | 50+ 节点 | 330+ 节点（最广） | CloudFront 节点 |
| **Preview Deployments** | ✅ 原生支持，每个 PR 自动 | ✅ 支持 | ✅ 支持 | ✅ 支持 |
| **Serverless Functions** | ✅ 内置（边缘/Node） | ✅ 内置 | ✅ Workers | ✅ Lambda |
| **WASM 支持** | ✅ 良好的 WASM 内容类型 | ✅ | ✅ | ✅ |
| **React 框架适配** | ⭐ 最优（Next.js/Vite 原生） | 良好 | 良好 | 一般 |
| **免费额度** | 100 GB 带宽/月 | 100 GB 带宽/月 | 无限带宽（有上限） | 15 GB/月 |
| **中文用户访问速度** | 良好（香港/新加坡节点） | 一般 | 一般（大陆节点有限） | 良好（CloudFront 中国） |
| **GitHub 集成** | ⭐ 最优 | ⭐ 最优 | 良好 | 良好 |

**选择 Vercel 的关键理由**：

1. **React 生态最优适配**：Vite 项目零配置部署，自动识别框架并优化构建
2. **Preview Deployments**：每个 PR 自动生成独立预览 URL，Code Review 效率大幅提升
3. **边缘网络对 WASM 友好**：wa-sqlite（~1.5MB WASM）首次加载由 CDN 加速
4. **免费额度充裕**：MVP 阶段预期远低于 100GB 带宽上限
5. **团队熟悉度**：前端团队已有 Vercel 使用经验

### 决策 2：后端 API 部署在 Render

**Render** 作为 Go Sync API 的容器化托管平台。

| 评估维度 | Render | Fly.io | Railway | Heroku | 自建 VPS |
|---------|--------|--------|---------|--------|----------|
| **Go 原生支持** | ✅ | ✅ (Docker) | ✅ | ✅ (Buildpack) | 手动 |
| **自动 HTTPS** | ✅ 自动证书 | ✅ 自动证书 | ✅ 自动证书 | ✅ 自动证书 | 手动配置 |
| **零停机部署** | ✅ 滚动更新 | ✅ 滚动更新 | ✅ | ✅ Preboot | 手动 |
| **健康检查** | ✅ HTTP 端点 | ✅ TCP/HTTP | ✅ | ✅ | 手动 |
| **日志** | ✅ 内置 + 外部 | ✅ 内置 | ✅ | ✅ | 自建 |
| **自动扩缩** | ✅ (Pro) | ✅ (自动) | ❌ (手动) | ✅ (Performance) | 自建 |
| **价格（入门）** | $7/月 | $0/月（3 shared VM） | $5/月 | $5/月 | $5-20/月 |
| **亚洲区域** | ✅ 新加坡节点 | ✅ 新加坡/香港 | ❌ 仅美欧 | ❌ | 取决于 VPS |
| **私有网络** | ✅ | ✅ (WireGuard) | ❌ | ✅ | ✅ |
| **管理复杂度** | ⭐ 极低 | 低 | 极低 | 低 | 高 |

**选择 Render 的关键理由**：

1. **Go 服务零配置**：原生 Go runtime 支持，`go build` + 二进制启动，无需 Dockerfile（也支持 Docker）
2. **自动 HTTPS + 滚动更新**：开箱即用，无需配置 Nginx/Caddy 或 CI 脚本
3. **新加坡节点可选**：降低亚洲用户 API 延迟
4. **与 Supabase 同区域部署**：Render 新加坡 → Supabase 新加坡，内网低延迟
5. **成本可控**：Starter $7/月起步，按需升级

**不选 Railway 的理由**：缺少亚洲区域部署，数据库耦合度高（Railway 内置 DB 而非外部 Supabase）。
**不选 Fly.io 的理由**：虽然免费额度好，但 Fly.io 的 Anycast 网络在部分区域稳定性不如 Render，且需要自行管理 Dockerfile。
**不选自建 VPS 的理由**：HTTPS 证书、滚动更新、健康检查、日志收集等需自行搭建，运维负担与团队规模不匹配。

### 决策 3：数据库、认证、存储统一使用 Supabase

**Supabase** 作为托管 PostgreSQL 数据库 + Auth + Storage 的统一后端平台。

| 评估维度 | Supabase | Firebase | 自建 PostgreSQL | Neon | PlanetScale |
|---------|----------|----------|----------------|------|-------------|
| **数据库** | PostgreSQL 15+ | Firestore（NoSQL） | 自行管理 | PostgreSQL | MySQL（Vitess） |
| **认证** | ✅ OAuth 2.0 + Email | ✅ | 自行集成 | — | — |
| **存储** | ✅ S3 兼容 | ✅ | 自行集成 | — | — |
| **RLS** | ✅ PostgreSQL RLS | Firestore Rules | 自行配置 | ✅ | ❌ |
| **实时订阅** | ✅ CDC + WebSocket | ✅ | 自行实现 | — | — |
| **本地开发** | ✅ Supabase CLI | ✅ Emulator | — | — | — |
| **免费额度** | 500 MB DB / 2 项目 | 1 GB 存储 / 50K 读 | — | 0.5 GB DB | 1 生产分支 |
| **SQL 兼容** | ⭐ 完整 PostgreSQL | PostgreSQL 子集（受限） | 完整 | 完整 | MySQL 方言 |
| **价格（Pro）** | $25/月 | $30/月（Blaze） | $20-50/月 VPS | $19/月 | $29/月 |
| **数据导出** | ✅ pg_dump 完整导出 | ❌ 受限 | ✅ | ✅ | ✅ |
| **亚洲区域** | ✅ ap-southeast-1 | ✅ asia-east1 | 取决于 VPS | ✅ | ❌ |

**选择 Supabase 的关键理由**：

1. **完整 PostgreSQL**：与本地 SQLite schema 高度兼容，数据类型映射清晰（TEXT ↔ VARCHAR, INTEGER ↔ BOOLEAN）
2. **RLS 行级安全**：数据库级别的用户数据隔离，API 层无需每个查询都手工加 `WHERE user_id = ?`
3. **一体化 BaaS**：Auth + Storage + Database 在同一平台，减少多服务集成的认证/授权复杂度
4. **Supabase CLI 本地开发**：`supabase start` 一键启动本地 PostgreSQL + Auth + Storage，与生产环境一致
5. **无厂商锁定**：基于标准 PostgreSQL，数据可随时 `pg_dump` 导出迁移
6. **与本地 SQLite 的镜像架构**：云端 Supabase PostgreSQL 作为同步中介，与本地 SQLite 形成清晰的镜像关系

### 决策 4：平台间统一使用亚太区域

**所有三平台统一部署在新加坡区域（ap-southeast-1）**：

| 平台 | 区域 | 配置方式 |
|------|------|---------|
| Vercel | 边缘网络全球自动分发 | 无需配置，CDN 自动就近 |
| Render | Singapore（ap-southeast-1） | 创建服务时选择 |
| Supabase | ap-southeast-1（新加坡） | 创建项目时选择 `ap-southeast-1` |

统一区域的好处：
- Render ↔ Supabase 在同一区域内网通信，延迟 < 5ms
- 数据库连接池效率最高
- 对中国、东南亚用户延迟最优（相比美西减少 100ms+ RTT）

### 决策 5：移动端不涉及平台选型

iOS（SwiftUI）和 Android（Jetpack Compose）客户端通过 App Store / Google Play 分发，不涉及 Web 部署平台。移动端直接与 Render API 和 Supabase Auth 通信，不经过 Vercel。

## 备选方案

### 备选 A：全部使用 Vercel（前端 + Serverless Functions）

将后端 API 也放在 Vercel，使用 Vercel Serverless Functions。

| 对比项 | Vercel Serverless | Render 常驻服务 |
|--------|------------------|----------------|
| 冷启动延迟 | 50-500ms（取决于运行时） | 无（常驻进程） |
| 长连接支持 | ❌ 不支持 WebSocket | ✅ 支持 |
| 同步循环 | 需外部 Cron 触发 | 内置 goroutine 定时器 |
| Go 支持 | ✅ 支持 | ✅ 原生 |
| 连接池维护 | 无法维护（无状态） | 可以维护 pgBouncer 连接 |
| 成本 | 按请求计费 | 固定月费 |

**否决理由**：同步引擎需要在后台维护定时轮询循环（goroutine 每 3s 触发 Pull），这是有状态的长生命周期进程，不适合 Serverless 的请求-响应模型。且 v1.5+ 计划引入 WebSocket 推送，Serverless 无法支持。

### 备选 B：全部使用 Supabase（Edge Functions 替代后端）

使用 Supabase Edge Functions（Deno）替代 Go 后端。

**否决理由**：
- Edge Functions 同样无法维护后台同步循环
- Deno 的 Go 生态不兼容
- Supabase Edge Functions 仍处于 Beta 阶段，不适合生产核心业务

### 备选 C：使用 Firebase 替代 Supabase

Firebase（Firestore + Auth + Storage）是一个更成熟的 BaaS 选项。

| 对比项 | Supabase | Firebase |
|--------|----------|----------|
| 数据库 | PostgreSQL（关系型） | Firestore（NoSQL 文档） |
| SQL 支持 | ✅ 完整 SQL | ❌ 无 SQL（需客户端查询） |
| Schema | 严格 Schema，与 SQLite 对齐 | 无 Schema |
| 数据可迁移性 | ✅ pg_dump 导出 | ❌ 受限导出 |
| 本地模拟 | ✅ Supabase CLI | ✅ Firebase Emulator |

**否决理由**：
- Firestore 是 NoSQL 文档数据库，与 MindFlow 的关系型数据模型（复杂 JOIN、关联表、FTS5 全文索引）不匹配
- 无法与本地 SQLite schema 对齐，导致双模型维护成本
- 数据导出困难，厂商锁定风险高

## 后果

### 正面后果

1. **运维负担极低**：三平台均为托管服务，无需管理服务器、证书、备份、扩缩容
2. **开发体验一致**：Supabase CLI 在本地提供与生产一致的 PostgreSQL + Auth，开发调试效率高
3. **成本可控**：MVP 阶段约 $53/月，免费额度覆盖初期用户
4. **扩展路径清晰**：Vercel 按带宽、Render 按实例规格、Supabase 按数据库容量线性升级
5. **亚洲用户延迟低**：统一新加坡区域 + Vercel 全球 CDN
6. **无厂商锁定**：PostgreSQL 标准数据导出 + Go 标准二进制 + React 标准构建产物
7. **部署自动化**：GitHub 集成 → Vercel 自动部署 + Render 自动部署

### 负面后果

1. **Supabase 成熟度低于 Firebase**：Auth 和 Storage 的 API 稳定性、文档完善度、社区规模略低于 Google Firebase
2. **三平台监控分散**：需要分别查看 Vercel Analytics / Render Logs / Supabase Dashboard，没有统一的 Observability 面板（可后续接入 Datadog/Sentry 解决）
3. **Render 冷启动**：免费方案在空闲时会休眠（约 50s 冷启动），需要 Pro 方案或设置 cron 保活
4. **Supabase RLS 调试复杂**：行级安全策略的调试和测试比应用层权限控制更复杂，尤其在复杂查询场景
5. **新加坡区域覆盖限制**：欧洲用户访问 Supabase 新加坡实例的延迟约 200ms，明显高于美西 150ms（可通过 Read Replica 后续优化）
6. **跨平台 CORS 配置**：Vercel（前端域）→ Render（API 域）→ Supabase（数据库域）的三层跨域配置需要仔细管理

### 缓解措施

| 风险 | 减轻措施 |
|------|---------|
| Supabase 稳定性 | Supabase 已在生产环境中被广泛验证（2024 年后 GA 成熟）；使用 Supabase Status 页面监控 |
| 监控分散 | MVP 先使用各平台内置监控；GA 阶段接入 Sentry（错误） + Grafana（指标） |
| Render 冷启动 | 使用 Pro 方案（$7/月）避免休眠；或设置外部 cron 每 10 分钟 ping `/health` |
| RLS 调试 | 在 Supabase 本地 CLI 中编写 RLS 策略的集成测试；使用 `supabase db lint` 静态检查 |
| 欧洲用户延迟 | MVP 用户主要在亚洲；后续可在 Supabase 添加 Read Replica 在欧洲 |
| CORS 配置 | 配置统一在 `render.yaml` 和 `vercel.json` 中管理，代码化 CORS 策略 |

## 相关 ADR

- [ADR-0001: 数据存储方案选型](0001-data-storage-strategy.md) — 决定了本地 SQLite + 云端 PostgreSQL 的双层存储架构
- [ADR-0002: Offline-First 同步架构](0002-offline-first-sync.md) — 决定了客户端驱动的同步引擎

## 参考

- Vercel Documentation: https://vercel.com/docs
- Render Documentation: https://render.com/docs
- Supabase Documentation: https://supabase.com/docs
- CLAUDE.md — 部署平台默认选型规则：前端 Vercel、后端 Render、数据库 Supabase
