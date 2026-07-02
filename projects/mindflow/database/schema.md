# 数据库设计：MindFlow 个人知识管理系统

> 数据库选型：本地 SQLite 3.x（Offline-First 真理源）+ 云端 PostgreSQL 15+（备份与同步中介）
>
> **核心原则**：本地 SQLite 是真理源（Source of Truth），所有数据操作先写入本地，再通过 Sync Layer 推送云端。云端数据库作为跨设备同步中介和备份存储，不承担搜索和图谱查询负载。

---

## 命名规范

| 规则 | 示例 |
|------|------|
| 表名：snake_case，复数 | `notes`, `note_tags` |
| 主键：`id` | `id UUID PRIMARY KEY` |
| 外键：`{table}_id` | `note_id`, `user_id` |
| 时间戳：`{verb}_at` | `created_at`, `deleted_at` |
| 布尔字段：`is_{adjective}` | `is_daily_note`, `is_broken` |
| 索引：`idx_{table}_{column}` | `idx_notes_updated_at` |

---

## 本地 SQLite 与云端 PostgreSQL 差异总览

| 维度 | 本地 SQLite | 云端 PostgreSQL |
|------|-----------|----------------|
| 数据类型 | `TEXT` 存 UUID, `TEXT` 存 ISO-8601, `INTEGER` 存布尔 | 原生 `UUID`, `TIMESTAMPTZ`, `BOOLEAN` |
| 全文搜索 | FTS5 虚拟表（本地搜索） | 无（搜索由本地承担） |
| 变更日志 | 单用户设备级 `change_log` | 全局 `change_log`（所有用户） |
| ID 生成 | 客户端 UUID v4 | 服务端 UUID v4 / BIGSERIAL |
| 加密 | 平台加密存储（iOS Keychain / Android EncryptedSharedPreferences） | TLS 1.3 传输 + AES-256 静态加密 |
| 外键约束 | 启动时 `PRAGMA foreign_keys = ON` | 原生支持 |
| 触发器 | 支持 | 支持 |
| 分区 | 不支持 | 可按 `user_id` 哈希分区 |
| JSON | 不支持（TEXT 存储） | JSONB |

---

### 时间戳策略

| DB | 存储格式 | 原因 |
|-----|---------|------|
| 本地 SQLite | `TEXT` ISO-8601 UTC (如 `2026-07-02T14:30:00.000Z`) | 人类可读、可排序、跨平台一致 |
| 云端 PostgreSQL | `TIMESTAMP WITH TIME ZONE` | 原生时区支持、高效范围查询 |

---

## 核心表定义

---

### 1. `users` — 用户账户

**说明**：用户注册信息、认证凭据、偏好设置和订阅状态。云端主表，本地仅缓存当前用户信息和偏好。

**本地 SQLite Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | TEXT | PK | UUID v4 |
| email | TEXT | NOT NULL, UNIQUE | 登录邮箱 |
| password_hash | TEXT | NULL | bcrypt 哈希（Email 注册时使用） |
| auth_provider | TEXT | NOT NULL, DEFAULT 'email' | `google` \| `apple` \| `email` |
| auth_provider_id | TEXT | NULL, UNIQUE | OAuth 提供商返回的用户 ID |
| display_name | TEXT | NULL | 用户显示名称 |
| avatar_url | TEXT | NULL | 头像 URL |
| subscription_tier | TEXT | NOT NULL, DEFAULT 'free' | `free` \| `pro` |
| preferences | TEXT | NOT NULL, DEFAULT '{}' | JSON 字符串存储用户偏好 |
| locale | TEXT | NOT NULL, DEFAULT 'zh-CN' | 语言偏好 |
| last_synced_at | TEXT | NULL | ISO-8601，上次同步时间 |
| created_at | TEXT | NOT NULL | ISO-8601 |
| updated_at | TEXT | NOT NULL | ISO-8601 |
| deleted_at | TEXT | NULL | ISO-8601，软删除时间 |

**云端 PostgreSQL Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, DEFAULT gen_random_uuid() | |
| email | VARCHAR(320) | UNIQUE, NOT NULL | 邮箱唯一 |
| password_hash | VARCHAR(255) | NULL | bcrypt 哈希 |
| auth_provider | VARCHAR(20) | NOT NULL, DEFAULT 'email' | |
| auth_provider_id | VARCHAR(255) | NULL, UNIQUE | |
| display_name | VARCHAR(200) | NULL | |
| avatar_url | TEXT | NULL | |
| subscription_tier | VARCHAR(20) | NOT NULL, DEFAULT 'free' | |
| preferences | JSONB | NOT NULL, DEFAULT '{}' | 原生 JSON 格式 |
| locale | VARCHAR(10) | NOT NULL, DEFAULT 'zh-CN' | |
| last_synced_at | TIMESTAMPTZ | NULL | |
| login_attempts | INTEGER | NOT NULL, DEFAULT 0 | 登录失败计数 |
| locked_until | TIMESTAMPTZ | NULL | 账户锁定截止时间 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| deleted_at | TIMESTAMPTZ | NULL | 软删除 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 | 适用 |
|--------|------|------|------|------|
| idx_users_email | email | UNIQUE | 登录查询、邮箱查重 | Both |
| idx_users_auth_provider | auth_provider_id | UNIQUE | OAuth 登录回调匹配 | Both |
| idx_users_deleted_at | deleted_at | INDEX | 活跃用户过滤 | Both |

**SQL（PostgreSQL）**：

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(320) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NULL,
    auth_provider VARCHAR(20) NOT NULL DEFAULT 'email',
    auth_provider_id VARCHAR(255) NULL UNIQUE,
    display_name VARCHAR(200) NULL,
    avatar_url TEXT NULL,
    subscription_tier VARCHAR(20) NOT NULL DEFAULT 'free',
    preferences JSONB NOT NULL DEFAULT '{}',
    locale VARCHAR(10) NOT NULL DEFAULT 'zh-CN',
    last_synced_at TIMESTAMPTZ NULL,
    login_attempts INTEGER NOT NULL DEFAULT 0,
    locked_until TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ NULL
);

CREATE UNIQUE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX idx_users_auth_provider ON users(auth_provider_id) WHERE auth_provider_id IS NOT NULL AND deleted_at IS NULL;
```

**约束**：

```sql
-- Email 用户必须设置密码
ALTER TABLE users ADD CONSTRAINT chk_email_user_password
    CHECK (auth_provider != 'email' OR password_hash IS NOT NULL);
```

---

### 2. `devices` — 设备注册

**说明**：记录用户注册的设备信息，用于同步身份校验和设备识别。

**本地 SQLite Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | TEXT | PK | 设备 UUID（客户端生成，持久化存储） |
| device_name | TEXT | NOT NULL | 设备显示名称（如 "小陈的 MacBook Pro"） |
| device_type | TEXT | NOT NULL | `web` \| `ios` \| `android` |
| is_current | INTEGER | NOT NULL, DEFAULT 1 | 布尔，当前设备标记 |
| created_at | TEXT | NOT NULL | ISO-8601 |

**云端 PostgreSQL Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 设备 UUID（客户端生成） |
| user_id | UUID | NOT NULL, FK -> users(id) ON DELETE CASCADE | |
| device_name | VARCHAR(200) | NOT NULL | |
| device_type | VARCHAR(20) | NOT NULL | |
| last_seen_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 上次活跃时间 |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | 可选吊销设备 |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| idx_devices_user_id | user_id | BTREE | 查询用户的所有设备 |
| idx_devices_last_seen | last_seen_at | BTREE | 清理过期设备 |

**SQL（PostgreSQL）**：

```sql
CREATE TABLE devices (
    id UUID PRIMARY KEY,  -- 客户端生成的 UUID，非服务端
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    device_name VARCHAR(200) NOT NULL,
    device_type VARCHAR(20) NOT NULL CHECK (device_type IN ('web', 'ios', 'android')),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_devices_user_id ON devices(user_id);
CREATE INDEX idx_devices_last_seen ON devices(last_seen_at);
```

---

### 3. `notes` — 笔记

**说明**：核心实体。包含笔记标题、正文 Markdown、归属关系、同步版本号和软删除/回收站状态。

**本地 SQLite Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | TEXT | PK | UUID v4 |
| title | TEXT | NOT NULL, DEFAULT '' | 笔记标题，最多 500 字符 |
| body | TEXT | NOT NULL, DEFAULT '' | Markdown 正文，应用层限制 50000 字 |
| is_daily_note | INTEGER | NOT NULL, DEFAULT 0 | 布尔，是否为每日笔记 |
| device_id | TEXT | NOT NULL | 最后修改设备 UUID |
| sync_version | INTEGER | NOT NULL, DEFAULT 1 | 单调递增版本号，用于乐观并发和冲突裁决 |
| sync_updated_at | TEXT | NOT NULL | ISO-8601，用于 LWW 冲突裁决的服务器时间戳 |
| checksum | TEXT | NOT NULL, DEFAULT '' | SHA-256(title + body) |
| is_deleted | INTEGER | NOT NULL, DEFAULT 0 | 布尔，逻辑删除标记 |
| trashed_at | TEXT | NULL | ISO-8601，放入回收站时间 |
| deleted_at | TEXT | NULL | ISO-8601，软删除时间（超过 30 天物理删除） |
| created_at | TEXT | NOT NULL | ISO-8601 |
| updated_at | TEXT | NOT NULL | ISO-8601 |

**备注**：
- `sync_version` 在每次本地保存时递增，同步时用于检测远程版本是否较旧
- `sync_updated_at` 由服务器在确认同步时设置，是 LWW 裁决的权威时间
- `trashed_at` 与 `deleted_at` 分离：用户删除时设 `trashed_at`，回收站中可见；系统清理时设 `deleted_at`
- 笔记不直接包含 `notebook_id` 字段，而是通过 `note_notebooks` 关联表支持多笔记本归属

**云端 PostgreSQL Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | 客户端生成的 UUID |
| user_id | UUID | NOT NULL, FK -> users(id) ON DELETE CASCADE | |
| title | VARCHAR(500) | NOT NULL, DEFAULT '' | |
| body | TEXT | NOT NULL, DEFAULT '' | |
| is_daily_note | BOOLEAN | NOT NULL, DEFAULT false | |
| device_id | UUID | NOT NULL | |
| sync_version | INTEGER | NOT NULL, DEFAULT 1 | |
| sync_updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | LWW 裁决时间 |
| checksum | VARCHAR(64) | NOT NULL, DEFAULT '' | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT false | |
| trashed_at | TIMESTAMPTZ | NULL | |
| deleted_at | TIMESTAMPTZ | NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**云端索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| idx_notes_user_deleted | user_id, deleted_at | BTREE | 按用户过滤活跃笔记 |
| idx_notes_sync_updated | user_id, sync_updated_at | BTREE | 增量同步游标查询 |
| idx_notes_trashed | user_id, trashed_at | BTREE | 回收站列表 |
| idx_notes_title_search | user_id, title | GIN (trgm) | 标题模糊查询（`[[` 联想） |

**云端 SQL**：

```sql
CREATE TABLE notes (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL DEFAULT '',
    body TEXT NOT NULL DEFAULT '',
    is_daily_note BOOLEAN NOT NULL DEFAULT false,
    device_id UUID NOT NULL,
    sync_version INTEGER NOT NULL DEFAULT 1,
    sync_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    checksum VARCHAR(64) NOT NULL DEFAULT '',
    is_deleted BOOLEAN NOT NULL DEFAULT false,
    trashed_at TIMESTAMPTZ NULL,
    deleted_at TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notes_user_deleted ON notes(user_id, deleted_at) WHERE deleted_at IS NULL;
CREATE INDEX idx_notes_sync_updated ON notes(user_id, sync_updated_at);
CREATE INDEX idx_notes_trashed ON notes(user_id, trashed_at) WHERE trashed_at IS NOT NULL;
CREATE INDEX idx_notes_title_search ON notes USING gin(title gin_trgm_ops);
```

---

### 4. `notebooks` — 笔记本/文件夹

**说明**：树状结构组织笔记，通过 `parent_id` 自引用实现层级。

**本地 SQLite Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | TEXT | PK | UUID v4 |
| name | TEXT | NOT NULL | 名称，最多 200 字符 |
| parent_id | TEXT | NULL, FK -> notebooks(id) ON DELETE SET NULL | 父笔记本 ID |
| sort_order | INTEGER | NOT NULL, DEFAULT 0 | 同级排序 |
| note_count | INTEGER | NOT NULL, DEFAULT 0 | 去规范化计数字段 |
| is_deleted | INTEGER | NOT NULL, DEFAULT 0 | |
| trashed_at | TEXT | NULL | |
| deleted_at | TEXT | NULL | |
| created_at | TEXT | NOT NULL | ISO-8601 |
| updated_at | TEXT | NOT NULL | ISO-8601 |

**云端 PostgreSQL Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | |
| user_id | UUID | NOT NULL, FK -> users(id) ON DELETE CASCADE | |
| name | VARCHAR(200) | NOT NULL | |
| parent_id | UUID | NULL, FK -> notebooks(id) ON DELETE SET NULL | |
| sort_order | INTEGER | NOT NULL, DEFAULT 0 | |
| note_count | INTEGER | NOT NULL, DEFAULT 0 | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT false | |
| trashed_at | TIMESTAMPTZ | NULL | |
| deleted_at | TIMESTAMPTZ | NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| idx_notebooks_parent | user_id, parent_id | BTREE | 查询子节点列表 |
| idx_notebooks_sort | user_id, parent_id, sort_order | BTREE | 同级排序查询 |

**SQL（PostgreSQL）**：

```sql
CREATE TABLE notebooks (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    parent_id UUID NULL REFERENCES notebooks(id) ON DELETE SET NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    note_count INTEGER NOT NULL DEFAULT 0,
    is_deleted BOOLEAN NOT NULL DEFAULT false,
    trashed_at TIMESTAMPTZ NULL,
    deleted_at TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notebooks_parent ON notebooks(user_id, parent_id);
CREATE INDEX idx_notebooks_sort ON notebooks(user_id, parent_id, sort_order);
```

---

### 5. `note_notebooks` — 笔记与笔记本多对多关联

**说明**：一条笔记可属于多个笔记本。这是多对多关联表。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| note_id | TEXT/UUID | NOT NULL, FK -> notes(id) ON DELETE CASCADE | |
| notebook_id | TEXT/UUID | NOT NULL, FK -> notebooks(id) ON DELETE CASCADE | |

**索引（复合主键）**：

```sql
-- Both SQLite and PostgreSQL
CREATE TABLE note_notebooks (
    note_id UUID NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
    notebook_id UUID NOT NULL REFERENCES notebooks(id) ON DELETE CASCADE,
    PRIMARY KEY (note_id, notebook_id)
);

CREATE INDEX idx_note_notebooks_notebook ON note_notebooks(notebook_id);
```

---

### 6. `tags` — 层级标签

**说明**：使用路径字符串（如 `科技/AI/大模型`）实现层级标签。标签全局唯一（同用户下）。

**本地 SQLite Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | TEXT | PK | UUID v4 |
| name | TEXT | NOT NULL | 标签名称（末段名称，如 `大模型`） |
| path | TEXT | NOT NULL, UNIQUE | 完整路径（如 `科技/AI/大模型`），全局唯一 |
| parent_id | TEXT | NULL, FK -> tags(id) ON DELETE SET NULL | 父标签 ID |
| note_count | INTEGER | NOT NULL, DEFAULT 0 | 去规范化计数 |
| is_deleted | INTEGER | NOT NULL, DEFAULT 0 | |
| deleted_at | TEXT | NULL | |
| created_at | TEXT | NOT NULL | ISO-8601 |
| updated_at | TEXT | NOT NULL | ISO-8601 |

**云端 PostgreSQL Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK | |
| user_id | UUID | NOT NULL, FK -> users(id) ON DELETE CASCADE | |
| name | VARCHAR(100) | NOT NULL | |
| path | VARCHAR(500) | NOT NULL | 完整路径 |
| parent_id | UUID | NULL, FK -> tags(id) ON DELETE SET NULL | |
| note_count | INTEGER | NOT NULL, DEFAULT 0 | |
| is_deleted | BOOLEAN | NOT NULL, DEFAULT false | |
| deleted_at | TIMESTAMPTZ | NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| idx_tags_path | user_id, path | UNIQUE | 路径唯一性 |
| idx_tags_parent | user_id, parent_id | BTREE | 查询子标签 |
| idx_tags_name_search | user_id, name | GIN (trgm) | 标签模糊搜索（`#` 联想） |

**约束**：

```sql
-- 标签名称不可含分隔符
ALTER TABLE tags ADD CONSTRAINT chk_tag_name_no_separator
    CHECK (name NOT LIKE '%/%');
```

---

### 7. `note_tags` — 笔记与标签多对多关联

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| note_id | TEXT/UUID | NOT NULL, FK -> notes(id) ON DELETE CASCADE | |
| tag_id | TEXT/UUID | NOT NULL, FK -> tags(id) ON DELETE CASCADE | |

```sql
CREATE TABLE note_tags (
    note_id UUID NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (note_id, tag_id)
);

CREATE INDEX idx_note_tags_tag ON note_tags(tag_id);
```

---

### 8. `note_links` — 双向链接索引

**说明**：存储笔记间的 `[[笔记标题]]` 引用关系。从笔记正文内容中解析提取。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | TEXT/UUID | PK | UUID v4 |
| source_note_id | TEXT/UUID | NOT NULL, FK -> notes(id) ON DELETE CASCADE | 来源笔记（含有 `[[...]]` 的笔记） |
| target_note_id | TEXT/UUID | NULL, FK -> notes(id) ON DELETE SET NULL | 目标笔记（被引用的笔记，断链时可为空） |
| target_title_snapshot | TEXT | NOT NULL | 创建链接时的目标标题快照，用于重命名检测 |
| context_snippet | TEXT | NOT NULL, DEFAULT '' | 链接前后各 50 字上下文片段 |
| is_broken | INTEGER/BOOLEAN | NOT NULL, DEFAULT false | 是否断链（目标被删除或重命名） |
| created_at | TEXT/TIMESTAMPTZ | NOT NULL | |

**索引**：

| 索引名 | 字段 | 用途 |
|--------|------|------|
| idx_links_source | source_note_id | 查询某笔记的所有出链 |
| idx_links_target | target_note_id | 查询某笔记的所有入链（反向链接） |
| idx_links_broken | is_broken, target_note_id | 断链检测 |

**SQL（PostgreSQL）**：

```sql
CREATE TABLE note_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_note_id UUID NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
    target_note_id UUID NULL REFERENCES notes(id) ON DELETE SET NULL,
    target_title_snapshot VARCHAR(500) NOT NULL,
    context_snippet TEXT NOT NULL DEFAULT '',
    is_broken BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_links_source ON note_links(source_note_id);
CREATE INDEX idx_links_target ON note_links(target_note_id);
CREATE INDEX idx_links_broken ON note_links(is_broken) WHERE is_broken = true;
```

---

### 9. `change_log` — 同步变更日志

**说明**：记录每次数据变更，作为同步队列的基础。本地 change_log 追踪单设备待同步的同级，云端 change_log 是全局变更历史供其他设备拉取。

**本地 SQLite Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | INTEGER | PK AUTOINCREMENT | 自增 ID，同步队列排序依据 |
| entity_type | TEXT | NOT NULL | `note` \| `notebook` \| `tag` \| `notebook_ref` \| `tag_ref` |
| entity_id | TEXT | NOT NULL | 受变更实体的 UUID |
| operation | TEXT | NOT NULL | `create` \| `update` \| `delete` |
| device_id | TEXT | NOT NULL | 变更发起设备 |
| timestamp | TEXT | NOT NULL | ISO-8601，变更发生时本地时间 |
| payload | TEXT | NOT NULL | JSON 序列化的变更数据快照 |
| version | INTEGER | NOT NULL | 变更后实体的 `sync_version` |
| checksum | TEXT | NULL | 变更后校验和（仅 note） |
| synced | INTEGER | NOT NULL, DEFAULT 0 | 布尔，是否已推送到云端 |
| sync_error | TEXT | NULL | 最近同步错误信息 |
| retry_count | INTEGER | NOT NULL, DEFAULT 0 | 重试计数，超过 10 停止自动重试 |
| created_at | TEXT | NOT NULL | ISO-8601 |

**云端 PostgreSQL Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGSERIAL | PK | 全局自增 ID |
| user_id | UUID | NOT NULL, FK -> users(id) ON DELETE CASCADE | |
| device_id | UUID | NOT NULL | |
| entity_type | VARCHAR(20) | NOT NULL | |
| entity_id | UUID | NOT NULL | |
| operation | VARCHAR(10) | NOT NULL | |
| server_timestamp | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 服务端接收时间戳（LWW 权威裁决依据） |
| client_timestamp | TIMESTAMPTZ | NOT NULL | 客户端声称的变更时间 |
| payload | JSONB | NOT NULL | 变更快照 |
| version | INTEGER | NOT NULL | |
| checksum | VARCHAR(64) | NULL | |

**索引**：

| 索引名 | 字段 | 用途 |
|--------|------|------|
| idx_changelog_user_sync | user_id, id | 按游标拉取变更 |
| idx_changelog_user_time | user_id, server_timestamp | 按时间范围拉取 |
| idx_changelog_entity | user_id, entity_type, entity_id | 特定实体的变更历史 |

**SQL（本地 SQLite）**：

```sql
CREATE TABLE change_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_type TEXT NOT NULL CHECK (entity_type IN ('note', 'notebook', 'tag', 'notebook_ref', 'tag_ref')),
    entity_id TEXT NOT NULL,
    operation TEXT NOT NULL CHECK (operation IN ('create', 'update', 'delete')),
    device_id TEXT NOT NULL,
    timestamp TEXT NOT NULL,
    payload TEXT NOT NULL,
    version INTEGER NOT NULL,
    checksum TEXT NULL,
    synced INTEGER NOT NULL DEFAULT 0,
    sync_error TEXT NULL,
    retry_count INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL
);

CREATE INDEX idx_changelog_synced ON change_log(synced, id);
CREATE INDEX idx_changelog_entity ON change_log(entity_type, entity_id);
```

**SQL（云端 PostgreSQL）**：

```sql
CREATE TABLE change_log (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    device_id UUID NOT NULL,
    entity_type VARCHAR(20) NOT NULL CHECK (entity_type IN ('note', 'notebook', 'tag', 'notebook_ref', 'tag_ref')),
    entity_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('create', 'update', 'delete')),
    server_timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    client_timestamp TIMESTAMPTZ NOT NULL,
    payload JSONB NOT NULL,
    version INTEGER NOT NULL,
    checksum VARCHAR(64) NULL
);

CREATE INDEX idx_changelog_user_sync ON change_log(user_id, id);
CREATE INDEX idx_changelog_user_time ON change_log(user_id, server_timestamp);
CREATE INDEX idx_changelog_entity ON change_log(user_id, entity_type, entity_id);
```

---

### 10. `version_history` — 版本历史

**说明**：每条笔记保留最近 30 个版本的内容快照。被冲突覆盖的版本也在这里保留。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | TEXT/UUID | PK | UUID v4 |
| note_id | TEXT/UUID | NOT NULL, FK -> notes(id) ON DELETE CASCADE | |
| title | TEXT/VARCHAR(500) | NOT NULL | 该版本时的标题 |
| body | TEXT | NOT NULL | 该版本时的正文 |
| version_number | INTEGER | NOT NULL | 版本号 |
| device_id | TEXT/UUID | NOT NULL | 该版本从哪个设备保存 |
| checksum | VARCHAR(64) | NOT NULL | |
| source | TEXT | NOT NULL, DEFAULT 'auto_save' | `auto_save` \| `manual` \| `conflict` \| `restore` |
| created_at | TEXT/TIMESTAMPTZ | NOT NULL | |

**索引**：

```sql
-- Unique constraint on (note_id, version_number)
-- Index on note_id for quick lookup
CREATE INDEX idx_version_history_note ON version_history(note_id, version_number DESC);
```

**SQL**：

```sql
CREATE TABLE version_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    note_id UUID NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    body TEXT NOT NULL,
    version_number INTEGER NOT NULL,
    device_id UUID NOT NULL,
    checksum VARCHAR(64) NOT NULL,
    source VARCHAR(20) NOT NULL DEFAULT 'auto_save',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(note_id, version_number)
);

CREATE INDEX idx_version_history_note ON version_history(note_id, version_number DESC);

-- 清理超 30 个版本的历史（保留最新 30 条）
-- 由应用层在每次保存版本历史后执行：
-- DELETE FROM version_history
-- WHERE note_id = ? AND id NOT IN (
--     SELECT id FROM version_history
--     WHERE note_id = ?
--     ORDER BY version_number DESC
--     LIMIT 30
-- );
```

---

### 11. `sync_cursor` — 同步游标

**说明**：记录每个设备上次成功同步位置，用于增量拉取。

**本地 SQLite Schema**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | INTEGER | PK | 仅有一行 |
| last_change_id | INTEGER | NOT NULL, DEFAULT 0 | 上次成功推送的 change_log.id |
| last_pull_cursor | INTEGER | NOT NULL, DEFAULT 0 | 上次成功拉取的云端 change_log.id |
| last_synced_at | TEXT | NULL | ISO-8601 |
| server_time_offset | REAL | NOT NULL, DEFAULT 0.0 | 设备时钟与服务器时钟偏移量（秒） |

**云端**：无需独立表，通过 `devices.last_seen_at` 和 `change_log.server_timestamp` 推算游标。

```sql
-- 本地 SQLite
CREATE TABLE sync_cursor (
    id INTEGER PRIMARY KEY CHECK (id = 1),
    last_change_id INTEGER NOT NULL DEFAULT 0,
    last_pull_cursor INTEGER NOT NULL DEFAULT 0,
    last_synced_at TEXT NULL,
    server_time_offset REAL NOT NULL DEFAULT 0.0
);

-- 初始化游标行
INSERT INTO sync_cursor (id, last_change_id, last_pull_cursor)
VALUES (1, 0, 0);
```

---

## 全文搜索（FTS5）

**说明**：仅在本地 SQLite 中维护，云端不作全文搜索。

### FTS5 虚拟表定义

```sql
CREATE VIRTUAL TABLE fts5_notes USING fts5(
    title,
    body,
    tag_paths,        -- 该笔记绑定的所有标签路径（空格分隔），用于 `tag:` 搜索
    content='',       -- 空 content 表：独立存储，无内容表同步
    tokenize='unicode61 remove_diacritics 2 tokenchars ''_-/'''
);

-- 注意：tokenchars 包含 '-' '_' '/'，确保标签路径 "科技/AI" 和复合词搜索正确
```

### FTS5 索引维护

**写入/更新索引**：

```sql
-- 笔记保存后，更新 FTS5 索引
INSERT INTO fts5_notes(rowid, title, body, tag_paths)
VALUES (
    (SELECT rowid FROM notes WHERE id = ?),
    ?,
    ?,
    (SELECT group_concat(t.path, ' ')
     FROM note_tags nt
     JOIN tags t ON t.id = nt.tag_id
     WHERE nt.note_id = ?)
);
```

**删除索引**（软删除或物理删除时）：

```sql
-- 通过 note_id 找到 rowid 后删除
DELETE FROM fts5_notes
WHERE rowid = (SELECT rowid FROM notes WHERE id = ?);
```

**全文搜索查询**：

```sql
-- 基础全文搜索
SELECT n.id, n.title, n.updated_at,
       rank AS relevance,
       snippet(fts5_notes, 1, '<mark>', '</mark>', '...', 30) AS body_snippet
FROM fts5_notes f
JOIN notes n ON n.rowid = f.rowid
WHERE n.deleted_at IS NULL
  AND fts5_notes MATCH ?
ORDER BY rank
LIMIT 20;

-- 标题联想搜索（用于 [[ 联想，不经过 FTS5）
SELECT id, title FROM notes
WHERE deleted_at IS NULL
  AND title LIKE '%' || ? || '%'
ORDER BY updated_at DESC
LIMIT 10;

-- 标签联想搜索
SELECT id, name, path FROM tags
WHERE deleted_at IS NULL
  AND path LIKE '%' || ? || '%'
ORDER BY note_count DESC
LIMIT 10;
```

---

## 数据量预估

| 实体 | 单用户初始 | 年增长 | 3 年后 | 热度 |
|------|----------|--------|-------|------|
| notes | 100 | 500-1000 | ~3000 | 热 |
| notebooks | 10 | 20-50 | ~150 | 温 |
| tags | 20 | 50-100 | ~300 | 温 |
| note_links | 50 | 500-2000 | ~6000 | 温 |
| note_tags | 200 | 1000-2000 | ~6000 | 温 |
| version_history | 1000 | 15000-30000 | ~90000 | 冷 |
| change_log | 5000 | 36000-72000 | ~200000 | 温（定期清理） |

---

## 关键查询路径

| 查询 | SQL 概要 | 涉及表 | 预期频率 |
|------|----------|--------|----------|
| 笔记列表（最近修改） | `SELECT * FROM notes WHERE deleted_at IS NULL ORDER BY updated_at DESC LIMIT 50` | notes | 极高 |
| 笔记本树加载 | `SELECT * FROM notebooks WHERE user_id=? AND deleted_at IS NULL ORDER BY parent_id, sort_order` | notebooks | 高 |
| 标签树加载 | `SELECT * FROM tags WHERE user_id=? AND deleted_at IS NULL ORDER BY path` | tags | 高 |
| 反向链接查询 | `SELECT * FROM note_links WHERE target_note_id=? AND is_broken=0` | note_links | 中 |
| 全文搜索 | FTS5 MATCH 查询 | fts5_notes | 高 |
| 增量同步（推送） | `SELECT * FROM change_log WHERE synced=0 ORDER BY id LIMIT 50` | change_log | 高 |
| 增量同步（拉取） | `SELECT * FROM change_log WHERE user_id=? AND id > ? ORDER BY id LIMIT 100` | change_log (云端) | 高 |
| 笔记按笔记本过滤 | `SELECT n.* FROM notes n JOIN note_notebooks nn ON n.id=nn.note_id WHERE nn.notebook_id=?` | notes, note_notebooks | 高 |
| `[[` 联想搜索 | `SELECT id, title FROM notes WHERE deleted_at IS NULL AND title LIKE '%keyword%' ORDER BY updated_at DESC LIMIT 10` | notes | 中 |
| 回收站笔记列表 | `SELECT * FROM notes WHERE user_id=? AND trashed_at IS NOT NULL ORDER BY trashed_at DESC` | notes | 低 |
| 断链检测 | `SELECT * FROM note_links WHERE is_broken=1` | note_links | 低（定时） |

---

## 迁移策略

### 本地 SQLite 迁移

- 使用客户端 Migration Runner（各端自行实现版本化迁移）
- 迁移文件命名：`YYYYMMDD_HHMMSS_description.sql`
- 关键原则：仅 ADD COLUMN 和 CREATE INDEX，不做 DROP 或 RENAME（向后兼容）

### 云端 PostgreSQL 迁移

- 使用 golang-migrate 或 Flyway 管理
- 严格遵循 Expand-Contract 模式：

```sql
-- 迁移示例：新增 sort_order 字段到 tags
-- 阶段 1: Expand（新增字段，允许 NULL）
ALTER TABLE tags ADD COLUMN sort_order INTEGER NOT NULL DEFAULT 0;

-- 阶段 2: 数据回填（如适用）
UPDATE tags SET sort_order = 0 WHERE sort_order IS NULL;

-- 阶段 3: Contract（应用程序已开始写入新字段后）
-- 不需要 ALTER，默认值已足够
```

### 回收站清理策略

```sql
-- 每天运行一次，物理删除 30 天前的回收站数据
-- PostgreSQL
DELETE FROM notes WHERE deleted_at IS NOT NULL AND deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM notebooks WHERE deleted_at IS NOT NULL AND deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM tags WHERE deleted_at IS NOT NULL AND deleted_at < NOW() - INTERVAL '30 days';

-- 同时级联清理关联表和索引
-- change_log 保留 90 天
DELETE FROM change_log WHERE server_timestamp < NOW() - INTERVAL '90 days';
```

### 版本历史清理策略

```sql
-- 每条笔记只保留最近 30 个版本
-- 由应用层在每次插入 version_history 后触发
DELETE FROM version_history vh
WHERE vh.note_id = ?
  AND vh.id NOT IN (
    SELECT vh2.id FROM version_history vh2
    WHERE vh2.note_id = ?
    ORDER BY vh2.version_number DESC
    LIMIT 30
  );
```
