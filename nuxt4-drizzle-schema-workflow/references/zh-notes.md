# Nuxt 4 Drizzle Schema Workflow 中文说明

## 用途

这个 skill 用于通用 Nuxt 4 项目中的 Drizzle schema、relations 和 migrations 工作流。它强调结构一致性和迁移校验，不携带任何具体业务表。

## 核心原则

- 每张表单独放在 `database/schemas/*.table.ts`。
- `database/schema.ts` 作为统一出口。
- schema、migration SQL、Drizzle metadata 必须保持一致。
- 数据库访问应放在 `server/services/**`。
- API handler 不应直接堆复杂数据库逻辑。

## database 放在根目录的原因

`database/` 不放进 `server/` 是为了降低 Drizzle Kit 和 Nuxt layer 的路径成本：

1. Drizzle Kit 不识别 Nuxt 的 `~` 路径映射。如果 schema 文件位于 `server/` 内部并使用 Nuxt alias 引入，生成 migration 时容易报错；要解决就需要额外配置路径映射。
2. Nuxt layer 可以带有自己的 `server/` 目录。若数据库位于根项目 `server/` 下，layer 需要引用数据库时会变成跨 server 目录引用，语义不清晰，也不利于复用。

因此数据库 schema、relations、migration 和 Drizzle client 更适合统一放在根目录 `database/`。

## 使用场景

- 新增表。
- 修改字段。
- 增加索引、唯一约束或 check 约束。
- 更新 Drizzle relations。
- 生成并检查 migration。
- 审查某次 schema 改动是否完整。

## 不应写入的项目专用内容

- 固定业务表清单。
- 固定外键策略。
- 固定 API 响应结构。
- 固定权限模型。

不同项目可以选择数据库外键或逻辑关联，本 skill 只要求选择必须符合项目既有约定。

## 后续优化方向

- 可以加入常见 Postgres 字段类型选择经验。
- 可以补充 Drizzle partial unique index、jsonb、timestamptz 的示例。
- 可以沉淀 migration SQL 与 metadata 不一致的排查步骤。
