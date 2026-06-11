---
name: nuxt4-supabase-drizzle-setup
description: Use when setting up, checking, or repairing the foundational integration of Supabase Postgres and Drizzle ORM in a Nuxt 4/Nitro project. Covers Supabase agent skill installation, package dependencies, runtime config boundaries, Drizzle config, database directory layout, and first verification steps. Does not define project-specific business tables.
---

# Nuxt 4 Supabase Drizzle Setup

Use this skill to establish the basic Nuxt 4 + Nitro + Drizzle + Supabase Postgres structure. Keep the work generic: do not copy domain tables, payment models, user settings, or other project-specific business rules into a new project.

For Chinese maintenance notes, read `references/zh-notes.md` only when the user asks for rationale, improvement history, or Chinese documentation.

## Required Supabase Skills

Before implementation, check whether the project or user environment already has the Supabase skills installed. If they are missing, tell the user to run:

```bash
npx skills add supabase/agent-skills
```

During the command-line selection, ensure these two skills are selected:

- `supabase`
- `supabase-postgres-best-practices`

After installation, use those skills for Supabase-specific documentation checks, security guidance, and Postgres best practices. Do not duplicate their full content in this skill.

## Core Rules

- Keep Nuxt 4 + Nitro + Drizzle + Supabase Postgres responsibilities separated.
- Keep `database/` outside `server/`.
- Keep Drizzle schema and migrations synchronized.
- Put database access behind server services.
- Expose Supabase public keys to the frontend only when the project uses Supabase Auth, Supabase client SDK, or direct frontend Supabase access.
- Enable RLS for every Supabase Postgres table in exposed schemas; each Drizzle `pgTable(...)` schema should call `.enableRLS()`.
- Inspect generated migration SQL and Drizzle metadata before treating a migration as ready.

## Recommended Dependencies

Check `package.json` first. Add missing packages only when the project does not already provide an equivalent:

```bash
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit dotenv @types/node
```

Use the repository's package manager if it is not `pnpm`.

`@types/node` is required for clean TypeScript support when root-level database files read `process.env` and use Node APIs. If the project already has it as a dev dependency, do not install it again.

Install `@supabase/supabase-js` only when the project uses Supabase Auth, the Supabase JavaScript SDK, or direct frontend Supabase access:

```bash
pnpm add @supabase/supabase-js
```

## Runtime Config Contract

For database-only Supabase Postgres + Drizzle usage, only `NUXT_DATABASE_URL` is required. The Drizzle client reads it from `process.env`, so it does not need to be placed in `runtimeConfig` unless other server code also reads it from Nuxt runtime config.

Configure these Supabase client/Auth values only when the project uses Supabase Auth, the Supabase JavaScript SDK, or direct frontend Supabase access:

```ts
runtimeConfig: {
  supabaseServiceRoleKey: process.env.NUXT_SUPABASE_SERVICE_ROLE_KEY,
  public: {
    supabaseUrl: process.env.NUXT_PUBLIC_SUPABASE_URL,
    supabaseKey: process.env.NUXT_PUBLIC_SUPABASE_KEY,
  },
}
```

Key boundaries:

- `NUXT_DATABASE_URL`: server-only Postgres connection string for Drizzle.
- `NUXT_PUBLIC_SUPABASE_URL`: browser-safe Supabase project URL; needed only for Supabase client/Auth usage.
- `NUXT_PUBLIC_SUPABASE_KEY`: browser-safe public key only; needed only for Supabase client/Auth usage.
- `NUXT_SUPABASE_SERVICE_ROLE_KEY`: server-only; needed only for Supabase Admin/Auth management use cases, and never under `runtimeConfig.public`.

## Directory Contract

Prefer this structure:

```text
database/
  index.ts
  schema.ts
  schemas/
  migrations/

server/
  api/
  services/
  validators/
  utils/
```

Required responsibilities:

- `database/index.ts`: create and export the Drizzle client.
- `database/schema.ts`: export tables and relations.
- `database/schemas/*.table.ts`: define individual table schemas.
- `database/migrations/`: store generated migrations.
- `server/services/**`: own business database access.

Keep `database/` at the project root because Drizzle Kit runs outside Nuxt's runtime path resolver. Placing schema files under `server/` often introduces Nuxt `~` aliases that Drizzle Kit cannot resolve without extra configuration. Root-level `database/` also avoids ambiguous imports in Nuxt layer setups where multiple `server/` directories may exist.

## Drizzle Config

Create or update `drizzle.config.ts` to read the database URL from environment variables and point at the central schema file:

```ts
import * as path from 'node:path';
import { config } from 'dotenv';
import { defineConfig } from 'drizzle-kit';

const env = process.env.NODE_ENV ?? 'development';
config({ path: path.resolve(process.cwd(), `.env.${env}`) });

export default defineConfig({
  dialect: 'postgresql',
  schema: './database/schema.ts',
  out: './database/migrations',
  dbCredentials: {
    url: process.env.NUXT_DATABASE_URL || '',
  },
});
```

Adapt dotenv loading to the repository's existing environment naming.

## Drizzle Client

Create `database/index.ts` as the single server-side Drizzle entrypoint:

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

Do not use `useRuntimeConfig()` in `database/index.ts`. Nuxt documentation presents `useRuntimeConfig()` as a composable that should be called in an appropriate Nuxt context, such as a server handler, plugin, route middleware, or setup function. The root-level `database/` module is also loaded by Drizzle tooling and TypeScript outside Nuxt runtime, so `process.env.NUXT_DATABASE_URL` is the simpler and more reliable source for the Drizzle connection.

## TypeScript Project Reference

Add a dedicated `tsconfig.database.json` so editors and type checks understand root-level database files as server-side TypeScript:

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

Keep `"types": ["node"]` even when `@types/node` is installed. Nuxt-generated server and node tsconfigs may set `"types": []`; in TypeScript, once `compilerOptions.types` is set, TypeScript does not automatically load every `@types/*` package. An inherited empty `types` array can hide Node globals such as `process` unless this database-specific tsconfig opts back into Node types.

Then add it to the root `tsconfig.json` references:

```json
{
  "references": [
    {
      "path": "./.nuxt/tsconfig.app.json"
    },
    {
      "path": "./.nuxt/tsconfig.server.json"
    },
    {
      "path": "./.nuxt/tsconfig.shared.json"
    },
    {
      "path": "./.nuxt/tsconfig.node.json"
    },
    {
      "path": "./tsconfig.database.json"
    }
  ],
  "files": []
}
```

If the project has a custom `tsconfig.json`, preserve existing references and add only the `./tsconfig.database.json` entry.

## First Verification

After setup:

1. Run the project's type check or build preparation command.
2. Run `pnpm drizzle-kit generate` only after at least one table exists.
3. Inspect generated SQL and `database/migrations/meta`.
4. Run the smallest available database or service test.
5. Report any remaining environment variable or Supabase dashboard action.
