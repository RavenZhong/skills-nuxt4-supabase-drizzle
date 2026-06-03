# Nuxt 4 Supabase Drizzle Skills 中文说明

这是一组可复用的 Codex skills，用于在 Nuxt 4 / Nitro 项目中快速接入 Supabase Postgres 与 Drizzle ORM。

本仓库抽取自真实 Nuxt 4 项目的工程经验，但内容保持通用，不包含任何项目专用业务表、支付模型、用户设置或产品规则。

## Skills 列表

### `nuxt4-supabase-drizzle-setup`

用于搭建或修复 Nuxt 4 + Supabase Postgres + Drizzle 的基础集成。

覆盖内容：

- Supabase 官方 agent skills 前置安装。
- Nuxt runtime config 边界。
- Drizzle 依赖安装。
- `database/` 目录契约。
- `drizzle.config.ts` 配置。
- Drizzle client 入口。
- 初始验证步骤。

### `nuxt4-drizzle-schema-workflow`

用于新增、修改或审查 Drizzle schema、relations、索引、约束和 migrations。

覆盖内容：

- 每张表一个 schema 文件。
- `database/schema.ts` 统一导出 tables 和 relations。
- schema 与 migration 保持一致。
- migration SQL 和 Drizzle metadata 检查。
- server service 数据访问模式。

### `nuxt4-supabase-drizzle-security-review`

用于审查 Nuxt 4 + Supabase + Drizzle 项目的安全边界。

覆盖内容：

- public runtime config 与 server-only runtime config 的边界。
- Supabase service role key 暴露风险。
- 数据库连接串暴露风险。
- 受保护业务数据通过 Nitro/server services 访问。
- Supabase Auth 边界检查。
- RLS 与 exposed schema 审查提醒。
- migration 安全审查。

## 前置 Supabase Skills

使用本仓库 skills 前，如果环境中还没有 Supabase 官方 skills，先执行：

```bash
npx skills add supabase/agent-skills
```

命令行选择时确保安装：

- `supabase`
- `supabase-postgres-best-practices`

本仓库 skills 会依赖这两个官方 skills 提供最新 Supabase 产品指导和 Postgres 最佳实践，不重复维护其完整内容。

## 推荐安装方式

可以将这三个 skill 目录复制到全局 Codex skills 目录：

```text
~/.codex/skills/
  nuxt4-supabase-drizzle-setup/
  nuxt4-drizzle-schema-workflow/
  nuxt4-supabase-drizzle-security-review/
```

如果希望项目内局部生效，也可以放到：

```text
<project>/.codex/skills/
```

## 核心规则

这三个 skills 共享以下核心规则：

- Nuxt 4 + Nitro + Drizzle + Supabase Postgres 分层。
- `database/` 不放在 `server/` 里。
- schema 和 migrations 保持同步。
- 数据库访问放在 server services 后面。
- 前端只持有公开 Supabase key。
- migration 生成后检查 SQL 和 Drizzle metadata。

## 为什么 `database/` 放在项目根目录

`database/` 放在项目根目录是有意设计，不只是代码风格。

Drizzle Kit 运行在 Nuxt runtime 之外，不能自动识别 Nuxt 的 `~` 路径映射。如果 schema 文件放在 `server/` 下，并且依赖 Nuxt alias，生成 migration 时可能报错，除非额外处理路径映射。

Nuxt layer 项目也可能存在多个 `server/` 目录。如果数据库代码放在某一个 `server/` 目录下，其他 layer 需要访问数据库时会出现跨 server 目录引用，路径语义和所有权都不清晰。根目录 `database/` 更适合作为 schema、relations、migrations 和 Drizzle client 的共享位置。

## Drizzle Client 环境变量读取

推荐的 `database/index.ts` 直接读取 `process.env.NUXT_DATABASE_URL`，并在缺失时抛错。

不要在 `database/index.ts` 中使用 `useRuntimeConfig()`。Nuxt 文档将 `useRuntimeConfig()` 作为需要合适 Nuxt 上下文的 composable 使用，例如 server handler、plugin、route middleware 或 setup function。根目录 database module 也可能被 Drizzle Kit、TypeScript 工具或其他 Node 进程加载，这些场景不一定处在 Nuxt runtime 中。

推荐入口代码：

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

import * as schema from './schema';

const connectionString = process.env.NUXT_DATABASE_URL;

if (!connectionString) {
  throw new Error('Missing NUXT_DATABASE_URL for Drizzle connection.');
}

const queryClient = postgres(connectionString, {
  prepare: false,
});

export const db = drizzle(queryClient, { schema });
```

## `database/` 的 TypeScript 支持

项目应新增 `tsconfig.database.json`，用于让编辑器和类型检查正确识别根目录数据库文件：

```json
{
  "extends": "./.nuxt/tsconfig.server.json",
  "compilerOptions": {
    "esModuleInterop": true,
    "types": ["node"]
  },
  "include": [
    "database/**/*.ts"
  ]
}
```

这里必须显式写 `"types": ["node"]`。Nuxt 生成的 `.nuxt/tsconfig.server.json` / `.nuxt/tsconfig.node.json` 里可能有 `"types": []`；TypeScript 一旦看到 `compilerOptions.types`，就不会自动加载所有 `@types/*` 包，所以即使已经安装 `@types/node`，`process` 这类 Node 全局类型也可能不可见。

然后在根目录 `tsconfig.json` 的 `references` 中追加：

```json
{
  "path": "./tsconfig.database.json"
}
```

同时检查 `package.json` 中是否已有 `@types/node` 开发依赖。因为数据库入口会读取 `process.env`，并可能使用 Node API，所以缺少 `@types/node` 时编辑器可能无法正确识别 Node 全局类型。

## 每个 Skill 的中文说明

每个 skill 目录内还包含更细的中文说明文档：

```text
references/zh-notes.md
```

这些文档用于记录设计原因、经验沉淀和后续优化方向。英文 `SKILL.md` 是 agent 实际执行时优先加载的工作说明。
