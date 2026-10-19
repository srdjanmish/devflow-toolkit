---
name: prisma-migrations
description: Prisma schema/migration workflow for DevFlow's master/client/HQ databases — edit the schema, generate the migration via Docker, review the generated SQL, then deploy. Use whenever a task requires a database schema change; never write raw SQL migrations.
---

> **Target project configuration.** Read the target repository's `CLAUDE.md` /
> `AGENTS.md` first. Resolve GitHub `{owner}/{repo}` from its remote or selected PR,
> the Jira account from the authenticated connector, and the local health URL from
> project configuration (ask if missing; never substitute a production URL).
> Commands and paths below describe a reference PHP/CodeIgniter + Prisma + Docker
> layout. Verify services, schemas, base branch and Playwright paths against the
> target project before executing; use its documented equivalents where different.


# Skill: Prisma Migration

When a task requires a database schema change, always use Prisma — never write raw SQL migrations.

## How migrations work in this project

There are three Prisma schemas:
- `migrations/master/schema_master.prisma` — shared master database (shared, e.g. cert names, companies)
- `migrations/client/schema_client.prisma` — per-company databases (e.g. certs, employees)
- `migrations/hq/schema_hq.prisma` — HQ database

Migrations are managed via Docker:
```bash
docker-compose run db_migrations build [master|client|hq] [migration_name]
```

This runs `prisma migrate dev` internally and generates a new migration file.

## Step-by-step workflow

### 1. Edit the Prisma schema
Add/modify the model in the correct `.prisma` file:
- Master DB change → `migrations/master/schema_master.prisma`
- Client (per-company) DB change → `migrations/client/schema_client.prisma`
- HQ DB change → `migrations/hq/schema_hq.prisma`

### 2. Generate the migration
```bash
docker-compose run db_migrations build master add_default_note_to_cert_names
docker-compose run db_migrations build client add_cert_number_to_employee_certificate
```

Replace `master`/`client` and the migration name accordingly.

This creates a new timestamped folder under `migrations/master/migrations/` or `migrations/client/migrations/` with a `migration.sql` file.

### 3. Review the generated SQL
Prisma auto-generates the SQL. Always review it before committing.

### 4. Deploy (applied automatically in CI/CD)
```bash
docker-compose run db_migrations        # runs prisma migrate deploy on all DBs
docker-compose run db_migrations bulk   # parallel bulk run for client DBs
```

## Naming conventions
- Use snake_case for migration names
- Be descriptive: `add_default_note_to_cert_names`, `add_cert_number_to_employee_certificate`
- Never use a generic name like `update` or `fix`

## Important rules
- **Never write raw SQL migration files** — always go through Prisma schema first
- **Never use `prisma db push`** — always use `prisma migrate dev` via the build command
- After editing the schema, always run `build` to create the migration file, even in local dev
- The `export` command pulls the current DB schema back into the `.prisma` file (reverse — use only to sync after manual DB changes)