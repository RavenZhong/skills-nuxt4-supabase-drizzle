---
name: nuxt4-drizzle-schema-workflow
description: Use when adding, changing, or reviewing Drizzle ORM database schemas, relations, migrations, database constraints, indexes, or server service data access in a Nuxt 4 project using Supabase Postgres. Keeps schema files, migration SQL, and Drizzle metadata aligned without adding project-specific business assumptions.
---

# Nuxt 4 Drizzle Schema Workflow

Use this skill for Drizzle schema and migration work in Nuxt 4 projects backed by Supabase Postgres. Keep the workflow generic and adapt business rules to the target project.

For Chinese maintenance notes, read `references/zh-notes.md` only when the user asks for rationale, improvement history, or Chinese documentation.

## First Read

Before changing schema behavior, inspect:

1. Project instructions such as `AGENTS.md`.
2. Existing architecture or code style docs, if present.
3. `package.json`.
4. `drizzle.config.ts`.
5. `database/index.ts`.
6. `database/schema.ts`.
7. Existing `database/schemas/*.table.ts` files.
8. Relevant `server/services/**` files.

Verify facts from source files before claiming that a table, relation, dependency, or config is missing.

## Core Rules

- Keep Nuxt 4 + Nitro + Drizzle + Supabase Postgres responsibilities separated.
- Keep `database/` outside `server/`.
- Keep Drizzle schema and migrations synchronized.
- Put database access behind server services.
- Expose only public Supabase keys to the frontend.
- Inspect generated migration SQL and Drizzle metadata before treating a migration as ready.

The root-level `database/` convention is intentional. Drizzle Kit does not automatically understand Nuxt `~` aliases, so schema files under `server/` can fail unless extra alias handling is added. In Nuxt layer projects, each layer can also have its own `server/` directory, which makes database imports from a root `server/` path unclear.

## Schema Change Workflow

1. Confirm the requested table or field belongs to the target project's scope.
2. Add or update one focused table file under `database/schemas/*.table.ts`.
3. Use `pgTable` with snake_case database names and idiomatic TypeScript export names.
4. Add indexes, unique constraints, checks, JSONB, timestamps, UUID defaults, and nullability according to the actual data contract.
5. Decide whether database foreign keys are appropriate based on the target project's conventions.
6. Export the table from `database/schema.ts`.
7. Add or update Drizzle `relations(...)` in `database/schema.ts` when query relations are needed.
8. Update server services to import from the central database exports.
9. Add or update focused tests when the change affects behavior or constraints.

## Migration Workflow

Generate migrations with the project's existing Drizzle command. Common default:

```bash
pnpm drizzle-kit generate
```

After generation:

- Inspect the generated SQL.
- Inspect `database/migrations/meta`.
- Confirm SQL, metadata, and TypeScript schema describe the same change.
- Do not hand-edit migration SQL without keeping schema and metadata strategy coherent.
- Tell the user whether the migration was generated, applied, or still needs to be applied.

## Service Access Pattern

Server services should own database reads and writes. API handlers should stay thin: parse request input, resolve auth/context where the project requires it, call services, and return responses.

Preferred import shape:

```ts
import { db } from '~~/database';
import { users } from '~~/database/schema';
```

Use project-local aliases when they differ.

## Review Checklist

Before reporting completion:

- Does every changed table export from `database/schema.ts`?
- Do relations match the intended query model?
- Are generated SQL and Drizzle metadata reviewed?
- Are schema defaults, nullability, and timestamps intentional?
- Are sensitive business queries kept out of frontend code?
- Did verification run, or is there a clear reason it could not run?
