---
name: reset-environment
description: Reset the local DevFlow Docker environment to a known-good state (clean rebuild), wait for db_migrations to finish applying migrations, and only then verify the app actually serves — index must return HTTP 200 — remediating with composer install and then root-cause investigation if it does not. Invoked by develop-ticket and pr-review at their Environment Reset Gates.
argument-hint: "base_branch=<BRANCH> return_branch=<BRANCH> [health_url=<URL>]"
model: sonnet
effort: high
# Docker + curl only — no MCP is reachable during an environment reset.
# NOTE: this skill is invoked mid-run by develop-ticket and pr-review, and a skill's
# allowed-tools also applies to what it invokes. The orchestration primitives below
# (Task/Agent/Skill/AskUserQuestion) are therefore kept so the restriction cannot clamp
# the calling workflow — the restriction that matters here is "no MCP", not "no tools".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Task
  - Agent
  - Skill
  - AskUserQuestion
---

> **Target project configuration.** Read the target repository's `CLAUDE.md` /
> `AGENTS.md` first. Resolve GitHub `{owner}/{repo}` from its remote or selected PR,
> the Jira account from the authenticated connector, and the local health URL from
> project configuration (ask if missing; never substitute a production URL).
> Commands and paths below describe a reference PHP/CodeIgniter + Prisma + Docker
> layout. Verify services, schemas, base branch and Playwright paths against the
> target project before executing; use its documented equivalents where different.


# Skill: Reset Local Environment (clean rebuild + health verification)

Shared procedure for putting the local Docker environment into a known-good state **and confirming it actually serves** before development or app-executing tests run. `develop-ticket` and `pr-review` invoke this at their Environment Reset Gates instead of duplicating the steps.

> **Plugin scoping note.** This copy ships inside the `devflow-toolkit` plugin. The `prisma-migrations` skill referenced below (root-cause step, "DB not migrated / empty") is a sibling in this same plugin — invoke it in its scoped form (`devflow-toolkit:prisma-migrations`) if the bare name doesn't resolve.

---

## Inputs

- `base_branch` — the branch to rebuild from (the ticket/PR base: a release branch if one exists, else `develop`).
- `return_branch` — the branch to return to after the reset (the ticket branch, or the PR branch).
- `health_url` — optional. The URL to health-check. Resolve from the target repository configuration; ask if missing. There is no built-in default.

---

## Mandatory opt-in ask

This is an implicit, mandatory **ask** every time a caller reaches its reset gate — the user opts in or out each run. Asking is mandatory; running the reset only happens on `yes`.

Ask exactly:

```text
Do you want me to reset the local environment first (clean rebuild)? This will:
  1. checkout the base branch
  2. docker compose down -v        (tears down containers AND removes volumes — local DB/data is wiped)
  3. docker compose up -d --build  (fresh rebuild)
  4. wait for db_migrations to finish applying migrations (HQ -> master -> every client DB)
Then I'll verify the app is serving (index returns 200) and return to <return_branch>. (yes / no)
```

- `no`: continue against the current environment as-is. Skip the rest of this procedure.
- `yes`: run the reset **and** the health verification below.
- anything else: re-ask the same yes/no question.

Do not skip the ask even if the environment "looks fine" — drift (stale containers, leftover fixtures, ephemeral in-container config edits) is exactly what this prevents.

---

## Reset (on `yes`)

Run, in order, from the repo root:

```bash
git checkout <base_branch>
docker compose down -v
docker compose up -d --build
git checkout <return_branch>
```

Wait for `up -d --build` to finish and containers to report healthy before continuing — then go to the `db_migrations` wait below. **Do not health-check yet.**

Notes:
- `docker compose down -v` is destructive to **local data only** (volumes) — never to the repo, and not a git action. The yes/no answer is the authorization; it does not need a separate git hard gate.
- After the reset the local DB/volumes are fresh — all test data/fixtures must be re-seeded by the caller before functional/UI testing.

---

## Wait for `db_migrations` (MANDATORY — runs before any health check)

`docker compose up -d --build` starts `db_migrations` alongside the app, but it is a **one-shot task container**, not a long-running service: its entrypoint runs `npm ci`, then `prisma migrate deploy` for HQ -> master -> every client DB, then exits. Because `down -v` wiped the volumes, the first `up` after a reset has *every* migration to apply — that takes minutes, and the portal container is already accepting requests the whole time.

**Never curl the health URL while migrations are still running.** A request in that window proves nothing: it can 500 on a half-migrated schema, or return a perfectly good 200 from a page that happens not to touch the not-yet-migrated tables. Both readings are noise, and the second one is worse — it hands a "verified" environment back to the caller before the DB is actually ready.

### 1. Follow the run

```bash
docker compose logs -f db_migrations
```

The log prints its phases: `Migrate` -> `HQ` -> `MASTER` -> `CLIENTS` (then one line per client database).

### 2. Confirm it actually finished (authoritative check)

Log output alone is not the signal — the container's terminal state is:

```bash
cid=$(docker compose ps -aq db_migrations)
docker inspect -f 'status={{.State.Status}} exit={{.State.ExitCode}}' "$cid"
```

- `exited 0` → migrations applied. **Only now** continue to Health verification.
- `running` → still working, **or** stuck on a failed migration (see step 3 — always check the log before assuming "just slow").
- `exited` with a non-zero code → migrations failed. Do not health-check; go to step 3.

Poll rather than guessing a duration:

```bash
until [ "$(docker inspect -f '{{.State.Status}}' "$(docker compose ps -aq db_migrations)")" = "exited" ]; do
  docker compose logs --tail=5 db_migrations
  sleep 10
done
docker inspect -f 'status={{.State.Status}} exit={{.State.ExitCode}}' "$(docker compose ps -aq db_migrations)"
```

### 3. Failure mode: a failed migration does **not** exit

`migrations/entrypoint.sh` traps a failed `prisma migrate deploy`, touches `/tmp/.failed_migration`, and then **loops forever** printing:

```text
Waiting to fix migrations, after it's done, run 'rm /tmp/.failed_migration'
```

So "container still running" is ambiguous — a long wait is either honest work or a held failure. Read the log before concluding:

```bash
docker compose logs --tail=50 db_migrations
```

If that waiting line is present, the migration **failed** and the real error is immediately above it. Then:

1. Fix the cause — see the `prisma-migrations` skill, including its approved `migrate diff` + `migrate deploy` fallback for a broken pre-existing migration.
2. Release the hold so the container can exit: `docker compose exec db_migrations rm /tmp/.failed_migration`
3. The service is declared `restart: on-failure:3`, so once it exits non-zero Compose restarts it and it **re-attempts the migrations on its own**, up to 3 attempts. Watch the log: if the restart applies them cleanly, just wait for `exited 0`. If the 3 attempts are exhausted, run them yourself with `make db` (`docker compose run --rm db_migrations`).
4. Re-check the terminal state from step 2.

Never health-check, and never hand control back to the caller, while `db_migrations` is in this held state — an unmigrated DB is exactly the condition this gate exists to catch.

---

## Health verification (MANDATORY after `db_migrations` has exited 0)

A successful `docker compose up` does **not** guarantee the app serves (missing vendor, boot fatals, unmigrated DB, etc.). After containers are healthy **and `db_migrations` has exited 0**, verify and remediate in this exact order. **Do not hand control back to the caller until the index returns HTTP 200** — or the failure is escalated to the user as a blocker.

### 1. Check the index returns 200

```bash
curl -k -s -o /dev/null -w "%{http_code}\n" <health_url>
```

- `200` → environment is good. Continue (re-seed test data as needed).
- anything else → go to step 2.

### 2. Mandatory remediation — `composer install` in the portal container

```bash
docker compose exec -T portal composer install
```

Then re-run the step-1 check.

- `200` → good. Continue.
- still not `200` → go to step 3.

### 3. Mandatory root-cause investigation

If `composer install` did not fix it, you **must** find and fix the cause before continuing — never proceed with a non-serving environment. Investigate:

```bash
docker compose ps                        # any container unhealthy / restarting?
docker compose logs --tail=150 portal    # PHP/app errors, missing config, fatal on boot
```

Common causes and fixes:
- **Boot fatal** (missing config/env, missing constant in `parameters.php`) → fix the config, restart the service.
- **DB not migrated / empty** → the `db_migrations` gate above should have caught this; re-check its terminal state, then apply migrations via the Docker migration workflow (`prisma-migrations` skill) and re-check.
- **Container unhealthy / restarting** → read its logs, fix the reported error, `docker compose up -d` again.
- **vendor still incomplete** → step 2 should have handled it; if it recurs, read the composer errors in the log and resolve them (e.g. auth/lock issues).

Re-check the index after each fix. Only continue once it returns `200`. If it genuinely cannot be made to return 200, **stop and report** the failing check + relevant logs to the user as a **material blocker** — never silently continue on a broken environment.
