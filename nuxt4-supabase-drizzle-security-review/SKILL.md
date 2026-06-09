---
name: nuxt4-supabase-drizzle-security-review
description: Use when reviewing a Nuxt 4 project that uses Supabase and Drizzle ORM for security risks around public keys, service role keys, database URLs, frontend database access, Supabase Auth boundaries, RLS, migrations, server-only utilities, internal APIs, or Postgres objects created through Drizzle migrations.
---

# Nuxt 4 Supabase Drizzle Security Review

Use this skill to review security boundaries in Nuxt 4 + Supabase + Drizzle projects. Pair it with the `supabase` and `supabase-postgres-best-practices` skills when Supabase-specific docs or current product guidance are needed.

For Chinese maintenance notes, read `references/zh-notes.md` only when the user asks for rationale, improvement history, or Chinese documentation.

## First Read

Inspect the actual project before making claims:

1. `AGENTS.md` or other project instructions.
2. `package.json`.
3. `nuxt.config.ts`.
4. `.env.example` or documented environment variable files, if present.
5. `database/index.ts`.
6. `drizzle.config.ts`.
7. `database/schema.ts` and recent migrations.
8. `server/api/**`, `server/services/**`, and `server/utils/**` files that touch auth, database access, internal APIs, webhooks, or admin behavior.

## Core Rules

- Keep Nuxt 4 + Nitro + Drizzle + Supabase Postgres responsibilities separated.
- Keep `database/` outside `server/`.
- Keep Drizzle schema and migrations synchronized.
- Put database access behind server services.
- Expose Supabase public keys to the frontend only when the project uses Supabase Auth, Supabase client SDK, or direct frontend Supabase access.
- Inspect generated migration SQL and Drizzle metadata before treating a migration as ready.

Treat root-level `database/` placement as part of the security and maintainability boundary. Drizzle Kit does not automatically resolve Nuxt `~` aliases from schema files, and Nuxt layer projects can have multiple `server/` directories. Keeping database code at the root reduces alias workarounds and ambiguous cross-layer imports.

## Security Review Areas

### Runtime secrets

Check that:

- `NUXT_DATABASE_URL` is server-only.
- `NUXT_SUPABASE_SERVICE_ROLE_KEY` is server-only when the project uses Supabase Admin/Auth management.
- Only public Supabase URL/key values appear under `runtimeConfig.public` when the project uses Supabase Auth, Supabase client SDK, or direct frontend Supabase access.
- No service role key, database URL, or admin secret appears in frontend code, public assets, client plugins, or logs.

### Database access

Check that protected business data is accessed through Nitro/server services, not directly from frontend Supabase clients.

Flag patterns where frontend code:

- Queries protected application tables directly.
- Sends trusted authorization fields such as arbitrary user IDs for protected writes.
- Depends on browser-side checks for server-enforced permissions.

### Supabase Auth boundary

Check that server-side code verifies Supabase sessions or access tokens before protected operations, according to the project's auth model.

Flag patterns where:

- `user_metadata` or user-editable claims decide permissions.
- Admin privileges depend only on frontend state.
- Token validation is skipped for protected API routes.

### RLS and exposed schemas

For tables in exposed Supabase schemas, check whether RLS and policies match the access model. Use the `supabase` skill for current product guidance before making detailed RLS recommendations.

Pay special attention to:

- Views that may bypass RLS.
- Security definer functions in exposed schemas.
- UPDATE policies that forget the required SELECT visibility.

### Migrations and Postgres objects

Review generated migration SQL for:

- Unexpected dropped columns or tables.
- Unsafe views or functions.
- Missing indexes for newly exposed query paths.
- Constraints that do not match the TypeScript schema.
- SQL that diverges from Drizzle metadata.

## Output Format

Lead with findings, ordered by severity. Include file and line references when possible.

Use this structure:

```text
Findings
- [Severity] File:line - Issue and impact.

Open Questions
- Anything that cannot be confirmed from the repository.

Summary
- Brief security posture summary and verification performed.
```

If no issues are found, say so clearly and list any remaining unverified areas.
