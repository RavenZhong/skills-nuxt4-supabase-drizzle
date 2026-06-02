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
- Expose only public Supabase keys to the frontend.
- Inspect generated migration SQL and Drizzle metadata before treating a migration as ready.

## Recommended Dependencies

Check `package.json` first. Add missing packages only when the project does not already provide an equivalent:

```bash
pnpm add @supabase/supabase-js drizzle-orm postgres
pnpm add -D drizzle-kit dotenv
```

Use the repository's package manager if it is not `pnpm`.

## Runtime Config Contract

Configure these values in `nuxt.config.ts` or the project's existing runtime config layer:

```ts
runtimeConfig: {
  supabaseServiceRoleKey: process.env.NUXT_SUPABASE_SERVICE_ROLE_KEY,
  db: {
    url: process.env.NUXT_DATABASE_URL,
  },
  public: {
    supabaseUrl: process.env.NUXT_PUBLIC_SUPABASE_URL,
    supabaseKey: process.env.NUXT_PUBLIC_SUPABASE_KEY,
  },
}
```

Key boundaries:

- `NUXT_PUBLIC_SUPABASE_URL`: browser-safe Supabase project URL.
- `NUXT_PUBLIC_SUPABASE_KEY`: browser-safe public key only.
- `NUXT_SUPABASE_SERVICE_ROLE_KEY`: server-only; never put it under `runtimeConfig.public`.
- `NUXT_DATABASE_URL`: server-only Postgres connection string for Drizzle.

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

const config = useRuntimeConfig();
const queryClient = postgres(config.db.url, {
  prepare: false,
});

export const db = drizzle(queryClient, { schema });
```

If the file can be loaded outside Nitro runtime, add a safe fallback to `process.env.NUXT_DATABASE_URL`.

## First Verification

After setup:

1. Run the project's type check or build preparation command.
2. Run `pnpm drizzle-kit generate` only after at least one table exists.
3. Inspect generated SQL and `database/migrations/meta`.
4. Run the smallest available database or service test.
5. Report any remaining environment variable or Supabase dashboard action.
