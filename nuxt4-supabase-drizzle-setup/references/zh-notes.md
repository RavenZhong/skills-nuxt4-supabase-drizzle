# Nuxt 4 Supabase Drizzle Setup 中文说明

## 用途

这个 skill 用于在通用 Nuxt 4 项目中快速建立 Supabase Postgres + Drizzle ORM 的基础工程骨架。它只负责安装、环境变量边界、目录结构和初始验证，不负责具体业务表设计。

## 前置安装

如果环境中还没有 Supabase 官方 skills，先执行：

```bash
npx skills add supabase/agent-skills
```

命令行选择时确保勾选：

- `supabase`
- `supabase-postgres-best-practices`

这两个 skill 负责 Supabase 官方能力和 Postgres 最佳实践，本 skill 不重复维护这些内容。

## 保留的通用规则

- Nuxt 4 + Nitro + Drizzle + Supabase Postgres 分层。
- `database/` 不放在 `server/` 里。
- schema 和 migration 保持同步。
- server services 统一负责数据库访问。
- 前端只持有公开 Supabase key。
- migration 生成后检查 SQL 和 Drizzle metadata。

## database 放在根目录的原因

`database/` 建议放在项目根目录，而不是放进 `server/`，主要有两个原因：

1. Drizzle Kit 是独立工具，不运行在 Nuxt runtime 里，不能天然识别 Nuxt 的 `~` 路径映射。如果 schema 文件放在 `server/` 下，文件之间引用时容易出现 `~` alias；Drizzle Kit 不额外处理路径映射就会报错，额外处理又会增加维护成本。
2. Nuxt layer 场景下可能存在多个 `server/` 目录。如果 layer 也要使用数据库能力，再从 layer 的 `server/` 去引用根项目的 `server/`，路径语义会变得不清晰。根目录 `database/` 更适合作为跨 app/layer/server 共享的数据库层入口。

## database/index.ts 不建议使用 useRuntimeConfig

`database/index.ts` 建议直接读取 `process.env.NUXT_DATABASE_URL`，不要使用 `useRuntimeConfig()`。

原因是 Nuxt 文档把 `useRuntimeConfig()` 作为需要 Nuxt 上下文的 composable 使用，推荐在 server handler、plugin、route middleware 或 setup function 等合适上下文中调用。根目录 `database/` 既会被 Nuxt server services 引用，也可能被 Drizzle Kit、TypeScript 编辑器和其他 Node 工具加载，这些场景不一定处在 Nuxt runtime context 中。

推荐代码：

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

## TypeScript 配置

为了让编辑器和类型检查正确识别根目录 `database/**/*.ts`，需要增加 `tsconfig.database.json`：

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

这里必须显式写 `"types": ["node"]`。Nuxt 生成的 `.nuxt/tsconfig.server.json` / `.nuxt/tsconfig.node.json` 中可能存在 `"types": []`。在 TypeScript 里，只要配置了 `compilerOptions.types`，TS 就不会自动加载所有 `@types/*` 包；`types: []` 等于明确告诉 TS 不要引入任何全局类型包。所以即使 `package.json` 已经安装了 `@types/node`，`process` 这类 Node 全局类型仍然可能不可见，需要在 `tsconfig.database.json` 中重新声明 Node types。

然后在根目录 `tsconfig.json` 的 `references` 中增加：

```json
{
  "path": "./tsconfig.database.json"
}
```

如果项目的 `tsconfig.json` 已经有其他 references，保留原有内容，只追加这一项。

同时要检查 `package.json` 是否已有 `@types/node` 开发依赖。没有的话需要安装：

```bash
pnpm add -D @types/node
```

原因是 `database/index.ts` 推荐读取 `process.env.NUXT_DATABASE_URL`，根目录数据库文件也可能使用 Node API；缺少 `@types/node` 时，编辑器可能无法正确识别 `process` 等 Node 全局类型。

## 不应写入的项目专用内容

- 具体业务表。
- 当前项目的 API 响应格式。
- 当前项目的端口、测试账号、业务阶段。
- Stripe、积分、订阅、宠物等业务模型。
- 特定登录页面或 UI 流程。

## 后续优化方向

- 可以补充不同包管理器的命令差异。
- 可以补充 Nuxt runtimeConfig 在多环境下的模板。
- 可以沉淀常见接入失败案例，例如环境变量缺失、数据库 URL 暴露、Drizzle schema 路径错误。
