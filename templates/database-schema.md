# 数据库设计：{{DOMAIN_NAME}}

> 数据库选型：{{POSTGRESQL | MYSQL | MONGODB | ...}} {{VERSION}}

## 命名规范

| 规则 | 示例 |
|------|------|
| 表名：snake_case，复数 | `users`, `order_items` |
| 主键：`id` | `id UUID PRIMARY KEY` |
| 外键：`{table}_id` | `user_id`, `order_id` |
| 时间戳：`{verb}_at` | `created_at`, `deleted_at` |
| 布尔字段：`is_{adjective}` | `is_active`, `is_deleted` |
| 索引：`idx_{table}_{column}` | `idx_users_email` |

---

## 模块：{{MODULE_NAME}}

### {{TABLE_NAME}} — {{TABLE_DESCRIPTION}}

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, NOT NULL | |
| | | | |
| | | | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | |
| deleted_at | TIMESTAMPTZ | NULL | 软删除 |

**索引**：

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| idx_{{table}}_{{column}} | | BTREE | |
| | | | |

**约束**：

| 约束名 | 类型 | 字段 | 说明 |
|--------|------|------|------|
| | UNIQUE | | |
| | CHECK | | |
| | FK | | |

**SQL**：

```sql
CREATE TABLE {{table_name}} (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE NULL
);

-- 索引
CREATE INDEX idx_{{table}}_{{column}} ON {{table_name}}(...);

-- 注释
COMMENT ON TABLE {{table_name}} IS '';
COMMENT ON COLUMN {{table_name}}.{{column}} IS '';
```

---

## 模块：{{MODULE_NAME}}

<!-- 按模块/限界上下文组织，每模块一个表格区，重复上述结构 -->

---

## 迁移策略

### 版本管理
<!-- 使用什么迁移工具？Flyway / golang-migrate / Alembic / ... -->

### 迁移原则
- [ ] 所有迁移可回滚（expand-contract 模式）
- [ ] 禁止 `DROP COLUMN`，改用软删除 + 延迟清理
- [ ] 大表迁移分批执行，避免锁表
- [ ] 每次迁移含数据验证 SQL

### 迁移模板

```sql
-- 迁移: {{VERSION}}_{{DESCRIPTION}}
-- 执行时间: 预计 < 100ms / 需分批执行
-- 回滚: {{ROLLBACK_PLAN}}

-- Step 1: 新增字段（允许 NULL）
ALTER TABLE ... ADD COLUMN ... ;

-- Step 2: 回填数据
UPDATE ... SET ... = ... WHERE ... ;

-- Step 3: 设置 NOT NULL（确认数据完整后）
ALTER TABLE ... ALTER COLUMN ... SET NOT NULL;
```
