---
name: reset-environment
description: Reset the local FieldFlo Docker environment to a known-good state (clean rebuild) and verify the app actually serves — index must return HTTP 200 — remediating with composer install and then root-cause investigation if it does not. Invoked by develop-fieldflo and pr-review at their Environment Reset Gates.
argument-hint: "base_branch=<BRANCH> return_branch=<BRANCH> [health_url=<URL>]"
model: sonnet
effort: high
---

# Skill: Reset Local Environment (clean rebuild + health verification)

Shared procedure for putting the local Docker environment into a known-good state **and confirming it actually serves** before development or app-executing tests run. `develop-fieldflo` and `pr-review` invoke this at their Environment Reset Gates instead of duplicating the steps.

> **Plugin scoping note.** This copy ships inside the `fieldflo-dev-workflow` plugin. The `prisma-migrations` skill referenced below (root-cause step, "DB not migrated / empty") is a sibling in this same plugin — invoke it in its scoped form (`fieldflo-dev-workflow:prisma-migrations`) if the bare name doesn't resolve.

---

## Inputs

- `base_branch` — the branch to rebuild from (the ticket/PR base: a release branch if one exists, else `develop`).
- `return_branch` — the branch to return to after the reset (the ticket branch, or the PR branch).
- `health_url` — optional. The URL to health-check. Default: `https://portal.fieldflo.dev/dev1` (local portal, default dev slug). The caller may override with the slug/URL relevant to its work.

---

## Mandatory opt-in ask

This is an implicit, mandatory **ask** every time a caller reaches its reset gate — the user opts in or out each run. Asking is mandatory; running the reset only happens on `yes`.

Ask exactly:

```text
Do you want me to reset the local environment first (clean rebuild)? This will:
  1. checkout the base branch
  2. docker compose down -v        (tears down containers AND removes volumes — local DB/data is wiped)
  3. docker compose up -d --build  (fresh rebuild)
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

Wait for `up -d --build` to finish and containers to report healthy before continuing.

Notes:
- `docker compose down -v` is destructive to **local data only** (volumes) — never to the repo, and not a git action. The yes/no answer is the authorization; it does not need a separate git hard gate.
- After the reset the local DB/volumes are fresh — all test data/fixtures must be re-seeded by the caller before functional/UI testing.

---

## Health verification (MANDATORY after a successful `up`)

A successful `docker compose up` does **not** guarantee the app serves (missing vendor, boot fatals, unmigrated DB, etc.). After containers are healthy, verify and remediate in this exact order. **Do not hand control back to the caller until the index returns HTTP 200** — or the failure is escalated to the user as a blocker.

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
- **DB not migrated / empty** → apply migrations via the Docker migration workflow (`prisma-migrations` skill), then re-check.
- **Container unhealthy / restarting** → read its logs, fix the reported error, `docker compose up -d` again.
- **vendor still incomplete** → step 2 should have handled it; if it recurs, read the composer errors in the log and resolve them (e.g. auth/lock issues).

Re-check the index after each fix. Only continue once it returns `200`. If it genuinely cannot be made to return 200, **stop and report** the failing check + relevant logs to the user as a **material blocker** — never silently continue on a broken environment.
