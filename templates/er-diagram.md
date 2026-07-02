# ER 图：{{DOMAIN_NAME}}

## 实体关系总览

```mermaid
erDiagram
    {{ENTITY_A}} ||--o{ {{ENTITY_B}} : "{{RELATIONSHIP}}"
    {{ENTITY_B}} }o--|| {{ENTITY_C}} : "{{RELATIONSHIP}}"

    {{ENTITY_A}} {
        uuid id PK "主键"
        varchar {{field}} "{{说明}}"
        timestamp created_at "创建时间"
        timestamp updated_at "更新时间"
    }

    {{ENTITY_B}} {
        uuid id PK "主键"
        uuid {{entity_a}}_id FK "关联 {{ENTITY_A}}"
        varchar {{field}} "{{说明}}"
        timestamp created_at "创建时间"
    }

    {{ENTITY_C}} {
        uuid id PK "主键"
        varchar {{field}} UK "唯一字段"
        timestamp created_at "创建时间"
    }
```

> Mermaid `erDiagram` 语法参考：
> - `||--o{` : 一对多（必填对可选）
> - `}|--o{` : 一对零或多
> - `||--||` : 一对一
> - 字段类型：`PK`、`FK`、`UK`、`NOT NULL`、`DEFAULT`
> - 不写类型仅作注释：`varchar name "用户姓名"`

## 逐表说明

### {{ENTITY_A}}

核心字段：

| 字段 | 类型 | 长度 | 必填 | 默认值 | 说明 |
|------|------|------|------|--------|------|
| id | UUID | - | Y | gen_random_uuid() | 主键 |
| | | | | | |

生命周期：`创建 → {{状态1}} → {{状态2}} → 归档/删除`

关联关系：

| 目标实体 | 关系 | 外键 | 说明 |
|----------|------|------|------|
| | | | |

---

### {{ENTITY_B}}

<!-- 重复上述结构 -->

---

## 数据量预估

| 实体 | 初始规模 | 年增长 | 3年后 | 热度 |
|------|----------|--------|-------|------|
| | | | | 热 / 温 / 冷 |

## 关键查询路径

| 查询 | SQL 概要 | 涉及表 | 预期频率 |
|------|----------|--------|----------|
| | | | |
