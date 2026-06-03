# Nuxt 4 Supabase Drizzle Skills

Reusable Codex skills for integrating Supabase Postgres and Drizzle ORM into Nuxt 4/Nitro projects.

中文文档请查看 [README.zh-CN.md](README.zh-CN.md)。如果你更习惯中文说明，可以先阅读中文文档了解安装方式、核心规则和设计原因，再查看各 skill 的英文 `SKILL.md`。

This repository is intentionally generic. It extracts reusable engineering workflow from a real Nuxt 4 project, but does not include any project-specific business tables, payment models, user settings, or product rules.

## Skills

### `nuxt4-supabase-drizzle-setup`

Use when setting up or repairing the foundational Nuxt 4 + Supabase Postgres + Drizzle integration.

Covers:

- Supabase agent skill prerequisite installation.
- Nuxt runtime config boundaries.
- Drizzle package dependencies.
- `database/` directory contract.
- `drizzle.config.ts`.
- Drizzle client entrypoint.
- First verification steps.

### `nuxt4-drizzle-schema-workflow`

Use when adding, changing, or reviewing Drizzle schema files, relations, indexes, constraints, or migrations.

Covers:

- One-table-per-file schema organization.
- Central `database/schema.ts` exports and relations.
- Schema and migration consistency.
- Migration SQL and Drizzle metadata review.
- Server service data access pattern.

### `nuxt4-supabase-drizzle-security-review`

Use when reviewing security boundaries in a Nuxt 4 + Supabase + Drizzle project.

Covers:

- Public vs server-only runtime config.
- Supabase service role key exposure risks.
- Database URL exposure risks.
- Protected business data access through Nitro/server services.
- Supabase Auth boundary checks.
- RLS and exposed schema review prompts.
- Migration security review.

## Prerequisite Supabase Skills

Before using these skills in a project, install the official Supabase agent skills if they are not already available:

```bash
npx skills add supabase/agent-skills
```

During selection, make sure these two skills are installed:

- `supabase`
- `supabase-postgres-best-practices`

These repository skills intentionally rely on the official Supabase skills for current Supabase product guidance and Postgres best practices.

## Recommended Installation

Clone or copy the skill folders into your Codex skills directory:

```text
~/.codex/skills/
  nuxt4-supabase-drizzle-setup/
  nuxt4-drizzle-schema-workflow/
  nuxt4-supabase-drizzle-security-review/
```

For a project-local setup, place them under:

```text
<project>/.codex/skills/
```

## Core Rules

The three skills share these core rules:

- Keep Nuxt 4 + Nitro + Drizzle + Supabase Postgres responsibilities separated.
- Keep `database/` outside `server/`.
- Keep Drizzle schema and migrations synchronized.
- Put database access behind server services.
- Expose only public Supabase keys to the frontend.
- Inspect generated migration SQL and Drizzle metadata before treating a migration as ready.

## Why `database/` Lives at the Project Root

The root-level `database/` convention is deliberate.

Drizzle Kit runs outside Nuxt's runtime path resolver and does not automatically understand Nuxt `~` aliases. If schema files live under `server/` and rely on Nuxt aliases, migration generation can fail unless extra alias handling is added.

Nuxt layer projects can also contain multiple `server/` directories. If database code lives under one `server/` directory, layers that need database access may end up with unclear cross-server imports. A root-level `database/` directory gives schema, relations, migrations, and the Drizzle client a clear shared home.

## Drizzle Client Environment Loading

The recommended `database/index.ts` reads `process.env.NUXT_DATABASE_URL` directly and throws if it is missing.

Do not use `useRuntimeConfig()` inside `database/index.ts`. Nuxt documents `useRuntimeConfig()` as a composable that should be called in an appropriate Nuxt context, such as a server handler, plugin, route middleware, or setup function. The root-level database module can also be loaded by Drizzle Kit, TypeScript tooling, and other Node processes outside Nuxt runtime.

## TypeScript Support for `database/`

Projects should add a dedicated `tsconfig.database.json` for root-level database files:

```json
{
  "extends": "./.nuxt/tsconfig.server.json",
  "compilerOptions": {
    "esModuleInterop": true
  },
  "include": [
    "database/**/*.ts"
  ]
}
```

Then add this project reference to the root `tsconfig.json`:

```json
{
  "path": "./tsconfig.database.json"
}
```

Also make sure `@types/node` exists in `devDependencies`, because the database entrypoint reads `process.env` and may use Node APIs.

## 中文说明

每个 skill 都包含一份中文说明文档，位于：

```text
references/zh-notes.md
```

这些中文文档用于记录经验沉淀、设计原因和后续优化方向。英文 `SKILL.md` 是 agent 实际执行时优先加载的工作说明。
