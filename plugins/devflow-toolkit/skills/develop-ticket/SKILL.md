---
name: develop-ticket
description: Orchestrated DevFlow Jira ticket workflow with model routing across Opus, Sonnet, and Haiku.
argument-hint: "ticket_id=<TICKET_ID> [epic_id=<EPIC_ID>] [pr_id=<PR_ID>]"
model: sonnet
effort: high
---

> **Target project configuration.** Read the target repository's `CLAUDE.md` /
> `AGENTS.md` first. Resolve GitHub `{owner}/{repo}` from its remote or selected PR,
> the Jira account from the authenticated connector, and the local health URL from
> project configuration (ask if missing; never substitute a production URL).
> Commands and paths below describe a reference PHP/CodeIgniter + Prisma + Docker
> layout. Verify services, schemas, base branch and Playwright paths against the
> target project before executing; use its documented equivalents where different.


# Skill: Develop DevFlow From Ticket

Run this skill for a DevFlow Jira ticket from context gathering through implementation, approval-gated testing, and final review.

> **Plugin scoping note.** This copy ships inside the `devflow-toolkit` plugin. Every subagent named below by its bare name (`devflow-context-analyst`, `devflow-planner`, `devflow-architect-reviewer`, `devflow-implementer`, `devflow-mechanical-editor`, `devflow-test-verifier`) must be invoked with the fully scoped `subagent_type`: `devflow-toolkit:<agent-name>` (e.g. `devflow-toolkit:devflow-planner`). Likewise, invoke the sibling `reset-environment` skill in its scoped form (`devflow-toolkit:reset-environment`). If a bare-name invocation fails to resolve, retry with the scoped form — do not treat that failure as the dependency being absent. `technical-analysis` (Step 3) is **not** bundled in this plugin — it stays a bare-name reference and is exactly what Hard Gate #0 below checks for.

This skill is the workflow controller. Repository rules live in `CLAUDE.md`. Model-specific execution lives in `.claude/agents/*.md`.

---

## Required Input

Required:

- `ticket_id`

Optional:

- `epic_id`
- `pr_id`

Usage:

```text
/develop-ticket ticket_id=PROJ-5877 epic_id=PROJ-5800 pr_id=<optional>
```

If `epic_id` is missing:

1. Use the Jira parent/epic if available.
2. If no parent/epic exists, use `ticket_id` as the root folder.

Work folder:

```text
agents/context/{ROOT_ID}/{TICKET_ID}/
```

Required files:

```text
Analysis.md
Changes.md
TestReport.md
SubagentLog.md
```

A UI ticket also keeps its temporary Playwright spec and `playwright.tmp.config.ts` here (Step 11) — that is deliberate, so `tests/aqa/` is never touched.

Do not store secrets, tokens, credentials, or raw PII in these files.

---

## Model Routing

Use this routing unless there is a strong reason to override.

| Phase | Agent | Model | Effort | Purpose |
|---|---|---:|---:|---|
| Context synthesis | `devflow-context-analyst` | Opus | xhigh | Jira/Slack/Notion/PR synthesis, AC extraction, ambiguity/risk. |
| Planning | `devflow-planner` | Sonnet | high | Code inspection, affected files, implementation plan, tests. |
| High-risk review/escalation | `devflow-architect-reviewer` | Opus | xhigh | DB/schema/security/API/tenant/cross-system decisions. |
| Implementation | `devflow-implementer` | Sonnet | high | Normal code changes. |
| Mechanical edits | `devflow-mechanical-editor` | Haiku | default | Exact low-risk rewrites, docs, formatting, simple view copy. |
| Testing | `devflow-test-verifier` | Sonnet | high | Test data, browser validation, static checks, failure triage. |
| Final review | `devflow-architect-reviewer` | Opus | xhigh | Merge readiness, AC/security/regression review. |

Escalate to Opus xhigh when any of these appear:

- unclear or conflicting ACs
- DB schema/migration decisions
- auth/permissions/security
- tenant isolation
- PII
- API contract compatibility
- HQ/mobile/integration impact
- high-risk payroll behavior
- repeated implementation/test failures

Use Haiku only when the plan is explicit and the edit is mechanical.

---

## Subagent Usage Tracking

This skill delegates each phase to a subagent (see Model Routing). Track **every** delegation so the run can end with a complete usage report (Step 13 → `## Subagent Usage`).

Maintain a running ledger at:

```text
agents/context/{ROOT_ID}/{TICKET_ID}/SubagentLog.md
```

Rules:

1. Log one entry **every time** a subagent is invoked — append it at the moment of delegation, and update its `Outcome` when the subagent returns.
2. Append as you go. Do **not** reconstruct the log from memory at the end — re-routes, escalations, and retries each count as a separate entry and are easy to miss after the fact.
3. Record only what the orchestrator reliably knows. **Do not invent token counts or cost figures** — per-subagent token/cost usage is not reliably available, so this report tracks who/what/why/outcome, not token accounting.
4. Note any deviation from the Model Routing table (a different agent or model/effort than the default) and the reason.
5. Do not store secrets, tokens, credentials, or raw PII in the log.
6. **Time each delegation.** Capture `Start` immediately before invoking the subagent and `End` immediately after it returns, using:

   ```bash
   date '+%Y-%m-%d %H:%M:%S'
   ```

   `Duration` is wall-clock `End − Start` (format `mm:ss`, or `hh:mm:ss` if over an hour). This is **wall-clock time around the delegation and includes orchestration overhead — it is not pure subagent compute time.** Label it as wall-clock; never present it as billed/compute time.

`SubagentLog.md` format — one row per invocation, in order:

```markdown
# {TICKET_ID} Subagent Log

| # | Step | Agent | Model / Effort | Purpose | Start | End | Duration | Outcome |
|---|---|---|---|---|---|---|---|---|
| 1 | Step 2 — Context | devflow-context-analyst | Opus / xhigh | Jira+Slack+Notion synthesis, AC extraction | 2026-06-18 09:21:04 | 2026-06-18 09:24:38 | 03:34 | success |
```

Field meaning:

- `#` — invocation order (1, 2, 3 …).
- `Step` — the workflow step that triggered the delegation.
- `Agent` — subagent type used.
- `Model / Effort` — the model and effort actually used (flag overrides vs the routing table).
- `Purpose` — one line on what the subagent was asked to do.
- `Start` / `End` — wall-clock timestamps captured via `date` at delegation and at return.
- `Duration` — wall-clock `End − Start` (`mm:ss`); includes orchestration overhead, not pure compute time.
- `Outcome` — one of: `success`, `revised`, `escalated → <agent>`, `routed back → <agent>`, `failed`.

---

## Hard Gates

Only these gates should stop the workflow for user input:

0. Dependency check at the very start of the run, before Step 0 — see "Hard Gate #0 — Dependency Check" below. Only asks if something required is missing or unavailable in the current session.
1. Jira status transition when ticket is `Ready for Dev` **or** `To Do` is imperative.
2. Branch creation or destructive git action is imperative.
3. Missing ticket description, or ACs that exist but aren't viable (not concrete/testable) — offer the `/technical-analysis` skill vs. a lightweight local draft (see Step 3).
4. Pre-implementation approval.
5. Post-code testing approval.
6. Any material blocker involving scope, security, tenant isolation, schema, API contract, data integrity, or destructive operations.
7. Committing or pushing any change is imperative. Never run `git add`/`git commit`, `git push`, `git commit --amend`, rebase, reset, cherry-pick, or force-push until the user explicitly approves that specific commit/push. Making edits to files is allowed after the pre-implementation approval (Hard Gate #4); turning those edits into commits is a separate action that always requires its own explicit approval.
8. Environment reset offer (see "Environment Reset Gate" below) — ALWAYS ask before starting code analysis (Step 4), before starting implementation (Step 8), and before starting testing (Step 10/11). Asking is mandatory; running the reset only happens if the user says yes.
9. PR Feedback Resolution Mode entry (Step 2.5) — when an existing PR has unresolved CoPilot comments / requested changes, ALWAYS ask before entering resolution mode, and only after an independent viability analysis. The post-resolution testing ask reuses Gate #5.

Do not stop for ordinary implementation details that can be safely inferred from existing project patterns. Record safe assumptions in `Analysis.md`.

---

## Hard Gate #0 — Dependency Check (always runs first, before Step 0)

Before Step 0, verify every dependency this skill relies on is actually available **in this session** — do not assume from a previous run just because it worked before. This is cheap: it's a check against information already in your context (available subagent types, available skills) plus a look at your connected tools (MCP) and, only if a PR is in scope, the local `gh` CLI.

### 1. Required subagents
Confirm each of these is available as a subagent type in this session:
- `devflow-context-analyst`
- `devflow-planner`
- `devflow-architect-reviewer`
- `devflow-implementer`
- `devflow-mechanical-editor`
- `devflow-test-verifier`

### 2. Required skills
Confirm each is available as an invocable skill in this session:
- `reset-environment` — mandatory; invoked at the Environment Reset Gate (Steps 4, 8, 10/11).
- `technical-analysis` — conditional; only offered if Step 3 triggers, but verify it's present anyway so that offer is real rather than a dead end.

### 3. Required MCP-backed capabilities
Confirm each capability is reachable through a connected MCP (match by what the tool does, not an exact tool-name prefix — different setups may expose it under different server/tool names):
- **Jira/Atlassian** — mandatory, Step 1.1 and Step 2.
- **Slack** — mandatory, Step 2.
- **Notion** — mandatory, Step 2.
- **MySQL** — preferred for Step 10; Docker CLI is an existing documented fallback, so treat this one as **degraded, not blocking** if absent.
- **Chrome DevTools / browser automation** — required for the Step 11 UI-testing requirement as the
  **debug/selector-discovery** path (the primary path is a CLI Playwright run, which needs only
  Node + the AQA project, not an MCP). Treat as **degraded, not blocking** if absent: a passing
  Playwright run still verifies, but a *failing* one cannot be diagnosed in-browser — say so
  rather than silently downgrading the evidence.

Note the limit of this check: a tool being listed means the MCP is *connected/exposed*, not that it's *authenticated*. A Jira MCP can appear available and still fail on first real call if the session isn't logged in — this gate catches "missing," not "misconfigured." Treat an auth failure hit later in the run the same as a missing dependency would have been treated here (flag it, don't silently degrade).

### 4. Required CLI (only when a PR is in scope)
- `gh` on PATH — needed whenever `pr_id` is provided or a PR is found (Step 1.2, Step 2.5). Check with `gh --version` only when a PR is actually in scope for this run; skip the check otherwise.

### 5. Playwright runner (only when the ticket has a UI surface)

Step 11 verifies UI through a CLI Playwright run, which needs the AQA project's own dependencies.
`tests/aqa/node_modules` is **gitignored and frequently absent on a fresh machine**. Check only
when the ticket actually touches UI; otherwise skip:

```bash
node --version
[ -x tests/aqa/node_modules/.bin/playwright ] && echo ok || echo "missing — run: cd tests/aqa && npm install"
```

Treat as **degraded, not blocking**: it is a one-time `npm install` (plus
`npx playwright install chromium` for the browser binary), not a missing capability. Install it
when reached rather than failing the gate — but say so, because the first run pays that cost.

### Gate behavior

- **Everything present:** proceed straight to Step 0. Do not report a clean bill of health — a passing check is silent.
- **Anything missing or unavailable:** stop before Step 0 and report it plainly — one line per missing item, naming what's missing and which step(s) it would block or degrade. Then ask exactly:

  ```text
  Do you want to continue, work will be limited? (yes / no)
  ```

  - `yes`: continue, but each missing dependency degrades its dependent step(s) when reached — e.g. no Notion MCP means Step 2 skips the Notion search and says so in `Analysis.md` rather than silently treating it as "nothing found"; no `devflow-architect-reviewer` means high-risk work cannot get its mandatory review — flag this again loudly if the ticket turns out to need it.
  - `no`: stop the run here. Do not proceed to Step 0.
  - anything else: re-ask the same yes/no question.

---

## Environment Reset Gate (ask before analysis AND before development AND before testing)

Before you begin **code analysis** (Step 4), again before you begin **executing development** (Step 8), and again before you begin **executing testing** (Step 10/11), ALWAYS apply the shared **`reset-environment`** skill. That skill owns the mandatory yes/no ask, the clean-rebuild commands, **and the post-rebuild health verification** (index must return HTTP 200 → remediate with `composer install` in the portal container → root-cause investigation if still failing). Invoke it with:

- `base_branch` = the ticket base branch (a matching release branch if one exists, else `develop`) — **exception at the Step 4 invocation:** the ticket branch was just cut from base in Step 1 and has no edits yet, so at that point pass the **ticket branch itself** as `base_branch` (i.e. `base_branch == return_branch`). There is nothing to reset "to base" that the current branch doesn't already equal — this just runs a plain rebuild in place instead of swapping branches.
- `return_branch` = the ticket branch
- `health_url` = the local portal at the test slug (from the target repository configuration; no built-in default)

This is an implicit, mandatory **ask** every run — the user opts in or out each time; asking is mandatory, the reset runs only on `yes`. After a `yes` reset the local DB/volumes are fresh — re-seed all test data/fixtures before Step 10. Do not skip the ask even if the environment "looks fine" — drift (stale containers, leftover fixtures, ephemeral in-container config edits) is exactly what this prevents. `docker compose down -v` is destructive to local data only (volumes), never to the repo, so it does not need Hard Gate #2 — the yes/no answer is the authorization.

The Step 4 ask exists so the planner inspects **accurate current behavior** (UI/API/DB state) rather than environment drift left over from a previous ticket, before any analysis of the code to be changed begins.

---

## Step 0 — Initialize Work Folder

Create or reuse:

```text
agents/context/{ROOT_ID}/{TICKET_ID}/
```

If existing files are present:

1. Read `Analysis.md`, `Changes.md`, and `TestReport.md` if present.
2. Continue from the latest completed step.
3. Check for new Jira comments, Slack context, PR feedback, and code changes since the last run.
4. Do not duplicate previous analysis.

---

## Step 1 — Jira Status and Branch Setup

### 1.1 Jira Status

Use Jira MCP to read the ticket status before branching.

If status is `Ready for Dev` **or** `To Do`, in any case, always ask exactly (substituting `{CURRENT_STATUS}` with the status actually read from Jira — `Ready for Dev` or `To Do`):

```text
The Jira ticket {TICKET_ID} is in status {CURRENT_STATUS}. Should I transition it to In Progress before proceeding? (yes / no)
```

- `yes`: transition to `In Progress`, then continue.
- `no`: continue without changing Jira.
- anything else: re-ask the same yes/no question.

Match the status name case-insensitively and ignoring separator/spacing variants (`Ready For Dev`, `Ready for Dev`, `To Do`, `TO DO`) — Jira board configs differ.

If status is anything else, continue without transition.

### 1.2 Git/PR State

Run:

```bash
git branch --show-current
git status --short
```

If `pr_id` is provided or an existing PR is found:

1. Check out the PR branch.
2. Pull latest changes.
3. Read all PR comments, requested changes, and review feedback.
4. Treat the current PR diff as existing state.
5. Do **not** decide here whether to re-run full analysis. Defer that to Step 2.5: if the PR has **unresolved CoPilot comments or requested changes**, the run may enter **PR Feedback Resolution Mode** (lightweight — resolve viable comments, no from-scratch re-development). Otherwise, continue the normal workflow with the PR context folded into analysis.

If already on a branch matching the ticket:

```bash
git fetch origin
git pull origin <current-branch>
```

If not on a ticket branch:

1. Read Jira `release`.
2. `git fetch origin` to update remote refs.
3. Determine the base branch:
   - If a matching release branch exists (e.g. `web-26-06-11-x1`), use it as the base.
   - If no release branch exists, use `develop`.
4. Ask before creating the ticket branch (Hard Gate #2). Confirm the branch name and the base branch with the user.
5. After approval, create the branch the classic way — explicitly `git checkout -b` off the base so it is a new, independent local branch (never a detached/aliased ref):

   ```bash
   git checkout <base-branch>
   git pull origin <base-branch>
   git checkout -b <ticket-branch> <base-branch>
   ```

   Branch name convention: `{TICKET_ID}-{short-slug}`
   (e.g. `PROJ-6596-inspection-report-columns`). Do not append the release to the branch name — the release is the base branch, not part of the ticket branch name.

6. On the first push, set the upstream to the ticket branch itself — never the base:

   ```bash
   git push -u origin <ticket-branch>
   ```

Important — avoid the "push maps to base branch" bug:

- Always create branches with `git checkout -b <ticket-branch> <base-branch>`. Do not create them implicitly (e.g. via tooling that leaves the new branch tracking the base/release branch).
- The ticket branch's upstream must be `origin/<ticket-branch>`, not `origin/<base-branch>`. If `git push` ever targets the base branch, the upstream is wrong — fix it with `git push -u origin <ticket-branch>` (or `git branch --set-upstream-to=origin/<ticket-branch>`).
- Verify with `git rev-parse --abbrev-ref --symbolic-full-name @{u}` before pushing if unsure; it must print `origin/<ticket-branch>`.

Never create a new branch without explicit user approval.

---

## Step 2 — Context Gathering

Use `devflow-context-analyst`.

Gather and summarize:

### Jira MCP

Mandatory:

- title
- description
- acceptance criteria
- comments
- status
- assignee if relevant
- parent/epic
- release field
- subtasks
- linked issues
- attachments/FFRD if available

### Slack MCP

Mandatory bounded search:

- ticket ID
- exact feature/form/module names
- 2–4 strong keywords from the ticket

Extract only decisions, clarifications, constraints, and scope changes.

### Notion MCP

Mandatory bounded search:

- feature docs
- architecture notes
- process docs relevant to the ticket

### GitHub / PR

Mandatory only if `pr_id` exists or PR is found:

- PR description
- changed files
- review comments
- requested changes
- unresolved conversations

Write to `Analysis.md`:

```markdown
# {TICKET_ID} Analysis

## Source Summary
## Jira Requirements
## Acceptance Criteria
## External Context
## Existing Work / PR Context
## Ambiguities
## Risks
## Initial Scope
```

---

## Step 2.5 — PR Feedback Resolution Mode (conditional)

**Trigger:** an existing PR for this ticket has **unresolved CoPilot review comments** or **requested changes** (from CoPilot, a human reviewer, or an automated tool) that are still open. If there is no PR, or the PR has no open feedback, skip this step and continue to Step 3.

When triggered, do **not** jump straight into resolving the comments, and do **not** restart a full re-development / re-analysis of the ticket. Follow this order.

### 2.5.1 — Independent viability analysis (mandatory, before any gate)

Delegate an **independent** assessment of the open feedback to `devflow-architect-reviewer` (Opus xhigh). The prompt must be **skeptical, not a rubber-stamp**: for each unresolved comment / requested change, judge — against the actual code, the ACs, and existing project patterns — whether the change is **actually correct and worth making** ("viable"), or whether the comment is wrong, stale, out of scope, or noise.

Record the result in `Analysis.md` under a `## PR Feedback Triage` section, one row per open comment:

```markdown
| # | Comment (source) | File:Line | Verdict | Rationale |
|---|---|---|---|---|
| 1 | … (CoPilot) | … | viable / not viable / needs-user-decision | independent reasoning vs code + ACs |
```

- **viable** — the comment is correct; a concrete, in-scope fix follows from it.
- **not viable** — the comment is wrong, stale, out of scope, or would regress behavior. **Do not resolve it.** Record why. This reason is surfaced to the user (and may become a reply on the PR only via the normal, separately-gated GitHub action — never auto-posted here).
- **needs-user-decision** — ambiguous, or a scope/behavior change; surface to the user rather than deciding.

### 2.5.2 — Resolution-mode entry gate (Hard Gate — always ask)

Present the triage table and ask exactly:

```text
This PR has open review feedback. Independent analysis says: <N viable, M not viable, K need your decision>.
I can enter PR Feedback Resolution Mode: resolve only the viable comments using the existing context files (no from-scratch re-development), leaving the not-viable ones with a documented reason.
Proceed in resolution mode? (yes / no / or tell me which comments to act on)
```

- `yes`: proceed to 2.5.3 with the viable set.
- `no`: skip resolution mode; continue the normal workflow (Step 3 onward) if the user still wants development, or stop.
- specific instruction: act on exactly the comments the user names.

Do not enter resolution mode without this explicit approval.

### 2.5.3 — Lightweight resolution (no re-development)

For the approved viable comments, delegate to `devflow-implementer` (or `devflow-mechanical-editor` for trivial/exact edits) per the Model Routing table — no new agent or routing is introduced here. Rules:

1. **Reuse existing context** — the current `Analysis.md` / `Changes.md` if present; otherwise the PR comments + diff + ACs are the minimal context. Do **not** re-run Steps 3–7 (missing-description gate, full technical analysis, requirements validation, full plan) unless a comment genuinely exposes a design gap that cannot be resolved locally — in which case stop and return to normal planning.
2. Make **only** the minimal edits each viable comment calls for. No adjacent cleanup, no scope expansion.
3. Apply the same implementation rules as Step 8 (Query Builder/binds, escape rendered data, `json_encode()` for PHP→JS, Prisma workflow for schema, no direct edits to generated API JSON).
4. Update `Changes.md` with what changed per comment, and mark each row in the `## PR Feedback Triage` table as resolved / not-resolved (with reason).
5. **Never commit or push, and never auto-post to the PR.** Resolving comments leaves edits in the working tree only. Pushing to update the PR, or replying to / resolving conversations on GitHub, are separate outward actions that each require explicit user approval (Hard Gate #7) — resolution mode does not authorize them.

### 2.5.4 — Post-resolution testing gate

After the viable comments are resolved, do **not** silently finish. Route into the normal **Step 9 (Post-Code Testing Approval Gate)** → Steps 10–11: ask the user whether to test, and run it if approved. If any resolved comment touched UI code, the mandatory UI-testing requirement in Step 11 applies — a UI change cannot be marked verified without an actual UI run (Playwright-first, per Step 11). Then produce the Step 13 final output, including the Subagent Usage ledger for the viability + resolution delegations.

---

## Step 3 — Missing Description / AC Gate

Run this step if Jira has no meaningful description, **or** if ACs exist but aren't viable — not concrete, not observable/testable, or self-contradictory.

This is a guarded ask — never auto-run either path. Ask exactly:

```text
{TICKET_ID} has [no description / ACs that aren't testable — name which]. I can:
  a) run the `/technical-analysis` skill — root-cause + codebase research, published as linked Confluence pages with a CS-readable Jira description and concise, testable ACs
  b) draft a lightweight local description/AC pair myself, kept in this working folder only (nothing published)
Which do you want? (technical-analysis / lightweight / skip)
```

- `technical-analysis`: invoke the `/technical-analysis` skill with `{TICKET_ID}`. That skill owns its own internal approval gate before writing to Jira/Confluence — do not bypass or duplicate it here.
  - If it publishes: the Jira description is now the source of truth. Restart this skill from Step 1 to re-read the ticket fresh.
  - If the user declines its publish step: that skill's local drafts live at `agents/context/{TICKET_ID}/analysis.md` and `solution.md` — a **flat** path, distinct from this skill's own nested `agents/context/{ROOT_ID}/{TICKET_ID}/` work folder. Read those two files as the source of truth and continue from Step 2 instead of restarting.
- `lightweight`: continue with the draft flow below.
- `skip`: continue with the current ticket as-is.

**Lightweight draft flow** (only when the user picks `lightweight`):

Use context from Step 2 to draft:

```text
agents/context/{ROOT_ID}/{TICKET_ID}/TicketDescription.md
```

Structure:

```markdown
# {TICKET_ID} — {Ticket Title}

## Description
Plain-English CS-readable description.

## Acceptance Criteria
1. Observable behavior.
2. Observable behavior.
```

Present the draft inline and ask:

```text
I drafted a CS-readable description and ACs at agents/context/{ROOT_ID}/{TICKET_ID}/TicketDescription.md.

Reply accept to restart the skill from Step 1 using this file as the local source of truth.
Reply skip to continue with the current ticket as-is.
Any other reply will be treated as feedback and I will revise the draft.
```

Do not continue past this gate until the user replies `accept` or `skip` (or, on the outer question, `skip`).

---

## Step 4 — Technical Analysis

**First apply the Environment Reset Gate** — before inspecting any code for this change, ask the user whether to reset the local Docker environment (see "Environment Reset Gate" above). At this point the ticket branch is fresh from base (Step 1, no edits yet), so the reset — if accepted — runs directly on the current branch with no base/return swap needed.

Use `devflow-planner`.

Inspect only code needed for the ticket scope:

- routes
- controllers
- models
- views
- JavaScript
- services/libraries
- Prisma schema if DB-related
- API docs if API-related
- tests if relevant

Document in `Analysis.md`:

```markdown
## Existing Implementation
- Request/UI/API flow
- Current behavior
- Relevant files

## Gap Analysis
- Already done
- Missing
- Needed changes

## Edge Cases
- AC-derived
- Existing data pattern edge cases
- Empty/boundary/negative cases
- Permission boundaries

## Security Assessment
- XSS
- SQL injection
- CSRF
- command/path/deserialization/upload risks
- PII
- tenant isolation
- secrets/data leakage

## Risk Assessment
- backwards compatibility
- HQ/mobile/integration impact
- data integrity
- performance
```

If high-risk design is needed, use `devflow-architect-reviewer` before Step 5.

---

## Step 5 — Requirements Validation

Create an AC matrix in `Analysis.md`:

```markdown
## Acceptance Criteria Validation

| # | Acceptance Criterion | Current Status | Implementation Needed | Test Evidence Needed |
|---|---|---|---|---|
| 1 | ... | Done / Missing / Needs clarification | ... | ... |
```

Stop and ask only if an AC is materially ambiguous or conflicts with other requirements.

---

## Step 6 — Implementation Plan

Use `devflow-planner`.

Write `Changes.md`:

```markdown
# {TICKET_ID} Changes

## Planned Files
| File | Change | Reason |
|---|---|---|

## Database Changes
- none / schema file / migration name

## API Changes
- none / endpoint / request / response / compatibility

## UI Changes
- none / affected page / component

## Security Measures
- XSS:
- SQL injection:
- CSRF:
- PII:
- tenant isolation:
- secrets:
- data leakage:

## Test Plan
| AC | Scenario | Data Needed | Method |
|---|---|---|---|
```

---

## Step 7 — Pre-Implementation Approval Gate

Output a numbered implementation plan and stop.

The plan must include:

1. Scope confirmation:
   - exact form/feature
   - exact module/code area
   - user roles/permissions in scope
2. Files to modify/create.
3. DB changes or `none`.
4. API changes or `none`.
5. UI changes or `none`.
6. Risks/dependencies.
7. Security measures mapped to this change.
8. Testing approach: URL/slug, users/roles, AC coverage, DB verification.

End exactly with:

```text
Reply `approved` to proceed with implementation. Any other reply will be treated as feedback to revise the plan.
```

Do not edit application code, Prisma schemas, migrations, or generated docs until the user replies with the literal word `approved`.

Allowed before approval:

- reading files
- querying DB read-only
- updating `Analysis.md`
- updating `Changes.md`
- drafting `TicketDescription.md`

---

## Step 8 — Implementation

After `approved`, **first apply the Environment Reset Gate** — ask the user whether to reset the local Docker environment before you start writing/executing code (see "Environment Reset Gate" above). Then choose the implementation agent:

- Use `devflow-implementer` for normal code changes.
- Use `devflow-mechanical-editor` only for exact low-risk mechanical changes.
- Use `devflow-architect-reviewer` before edits if the change became high-risk.

Implementation rules:

1. Follow `Changes.md`.
2. Make minimal scoped edits.
3. Match existing CodeIgniter/project conventions.
4. Use Query Builder/binds for DB access.
5. Escape rendered data.
6. Use `json_encode()` for PHP-to-JS values.
7. Follow Prisma migration workflow for schema changes.
8. Do not update API generated JSON directly.
9. Keep `Analysis.md` and `Changes.md` current.
10. If the plan becomes wrong, stop and return to planning.
11. Do not commit or push. `approved` authorizes editing files, not committing them. Leave changes in the working tree; staging, committing, and pushing happen only after the user explicitly approves the commit/push (Hard Gate #7).

For schema changes:

1. Edit Prisma schema.
2. Generate migration via Docker.
3. Review generated SQL.
4. Apply migration.
5. Verify schema/SQL sync.
6. Document migration name and SQL review result.

---

## Step 9 — Post-Code Testing Approval Gate

After code changes, do not test yet.

Output:

1. Changed files and logic summary.
2. AC-based scenarios to test.
3. Proposed test approach:
   - URL/slug
   - user roles
   - data needed
   - UI/API flow
   - DB verification
   - static checks/tests
4. Ask exactly:

```text
Should I proceed with testing this change?
```

Wait for approval before testing.

---

## Step 10 — Test Data Verification

**First apply the Environment Reset Gate** — once testing is approved (Step 9), ask the user whether to reset the local Docker environment before you start executing tests (see "Environment Reset Gate" above). If the user accepts the reset, the local DB/volumes are wiped — re-seed all required test data afterward.

Use `devflow-test-verifier`.

Before functional testing:

1. Verify database/file data covers every AC.
2. Prefer MySQL MCP for DB checks.
3. If MySQL MCP is unavailable, use Docker CLI.
4. Generate missing test data only when required.
5. Document records/data used in `TestReport.md`.

---

## Step 11 — Testing

Use `devflow-test-verifier`.

**UI changes MUST be exercised for real (mandatory).** If the change touches any UI element (`application/views/`, view templates, `.js`, `.css`, rendered output, or user-facing behavior), a UI test is **required** — the ticket **cannot be verified** without it.

**Prefer a Playwright test run through the CLI over interactive browser automation.** Do not drive the browser one action at a time — click-by-click verification is slow, non-reproducible, and leaves no re-runnable artifact. The agent definition (`devflow-test-verifier.md` → "UI verification") owns the full procedure; the rules that bind this step:

- **Write the temporary spec into the ticket's own work folder**, `agents/context/{ROOT_ID}/{TICKET_ID}/` — both `tmp.{TICKET_ID}.<feature>.test.ts` and its `playwright.tmp.config.ts`. This leaves `tests/aqa/` **completely untouched** and keeps the spec beside `Analysis.md` / `Changes.md` / `TestReport.md`.
- **Retention:** specs in the ticket context folder **stay** — they are part of the ticket record. A temp spec written **anywhere else must be deleted** when the run finishes (pass or fail), and `git status` must come back clean of it. Generated artifacts (`.pw-artifacts/`, and any `playwright-report/` / `test-results/`) are always deleted, including from the context folder.
- **Temporary specs are allowed; changing the AQA suite is not.** Do not create, edit, delete, or move any existing spec, page object, component, fixture, seed, config, or doc under `tests/aqa/` — read them freely, change none. No `qase(N, …)` id, no `@REGRESSION`/`@SMOKE`/`@SANITY` tags, no Qase PATCH (an outward-facing write the Step 9 approval does not cover). If the flow cannot be driven without changing the suite, stop and report it.
- The temp config is **not optional** and four of its lines are load-bearing — `projects: undefined` (the committed config's folder-anchored buckets match nothing outside `tests/ui/<domain>/`, so the spec would report `0 tests in 0 files`), `tsconfig: '../../../../tests/aqa/tsconfig.json'` (without it the `@/` aliases fail and it again reports `0 tests`), `outputDir` + `reporter: [['list']]` (otherwise report dirs land at the repo root and the inherited Qase/Datadog reporters stay live), and `use: { ...base.use, headless: true }` (the base config's `headless: CONFIG.CI` is `false` locally and would open a visible Chrome window). The agent file carries the exact contents.
- **Chrome always runs headless.** Never pass `--headed` and never set `headless: false`, not even while debugging. A failing spec is debugged from its retained trace, video and screenshot, or through the browser-MCP debug lane below.
- Run with the Qase reporter forced off — it is **enabled by default** when `QASE_SKIP` is falsy, so an unguarded run pushes results to Qase TestOps:
  ```bash
  cd tests/aqa
  # one-time, node_modules is gitignored: npm install && npx playwright install chromium
  QASE_SKIP=true ./node_modules/.bin/playwright test \
    --config=../../agents/context/{ROOT_ID}/{TICKET_ID}/playwright.tmp.config.ts \
    --reporter=list --workers=1 --retries=0
  ```
  Use `./node_modules/.bin/playwright`, not bare `npx playwright` — `npx` may resolve a cached global Playwright that dies with `MODULE_NOT_FOUND` on `src/config.ts`.
- Test local portal slug, configured in the target repository. Never production or sandbox URLs.
- **Use Chrome DevTools MCP only when the Playwright test fails** — to inspect DOM, console, network, or visual state — or to discover selectors, debug unexpected state, do visual verification, or read console/network errors. Then fix and re-run Playwright.
- **When you are in the browser, batch.** No click-by-click there either: `take_snapshot` (not `take_screenshot`) for uid handles, `fill_form` to set many fields in one call, `evaluate_script` to assert state without touching the UI, and `browser_batch` (claude-in-chrome) to run a whole navigate→click→type→screenshot sequence in one round trip. Never trigger a JS dialog — it freezes the extension.
- If the UI genuinely cannot be exercised locally (e.g. feature-flag gated off, no test-data path), do **not** mark the UI verified — record the reason and flag the affected AC as **not UI-verified** so it is never silently treated as passing.

Run relevant checks:

```bash
make lint
make codesniff
make phpstan
```

Run focused tests where relevant:

```bash
composer test
make aqa
```

Do not run broad/expensive suites unless relevant or requested.

Validate:

- every AC
- edge cases
- permissions
- regression risk
- data persistence
- tenant isolation
- security checklist

---

## Step 12 — Final Review

Use `devflow-architect-reviewer`.

Review:

1. AC coverage.
2. Diff vs plan.
3. Security.
4. Tenant isolation.
5. DB/API compatibility.
6. UI behavior.
7. Tests and evidence.
8. Regression risk.
9. Documentation completeness.

If required fixes remain, route back to `devflow-implementer` or `devflow-mechanical-editor`.

Capture generalizable lessons: if the review (or any PR comment from CoPilot, a human reviewer, or an automated tool) surfaces a recurring pitfall, missed convention, or security gap that should not be repeated on future tickets, propose adding it to the right home per `CLAUDE.md` → "Continuous Learning from Reviews" (this skill, the matching `.claude/agents/*.md`, or `CLAUDE.md`). Always ask the user to approve the addition before writing it — never edit a skill, agent, or `CLAUDE.md` silently.

---

## Step 13 — Final Output

Return:

```markdown
## Summary
## Files Changed
## Acceptance Criteria Coverage
## Test Results
## Security Review
## Risks / Follow-ups
## Documentation Path
## Subagent Usage
```

For `## Subagent Usage`, **print the full result inline in the final output** (not just a pointer to the file). It must contain:

1. **The complete ordered ledger table**, reproduced from `SubagentLog.md`, including the `Start`, `End`, and `Duration` columns for every invocation:

   ```markdown
   | # | Step | Agent | Model / Effort | Purpose | Start | End | Duration | Outcome |
   |---|---|---|---|---|---|---|---|---|
   | 1 | Step 2 — Context | devflow-context-analyst | Opus / xhigh | … | … | … | 03:34 | success |
   | … |
   | — | **Total** | — | — | — | — | — | **NN:NN** | — |
   ```

   The final **Total** row sums the `Duration` column (total wall-clock across all subagents).

2. A short summary under the table:
   - Total subagent invocations.
   - Per-agent count (e.g. `context-analyst ×1, planner ×2, implementer ×3, test-verifier ×1, architect-reviewer ×1`).
   - Per-model count (`Opus ×N, Sonnet ×N, Haiku ×N`).
   - Total wall-clock time across all subagents, and the longest single delegation.
   - Escalations / re-routes and why.
   - Deviations from the Model Routing table (where the actual agent or model/effort differed from the default, and the reason).

Durations are wall-clock (include orchestration overhead) — label them as such. Do not include token or cost figures — they are not reliably available per subagent (see Subagent Usage Tracking).

Do not claim completion unless all applicable ACs have been implemented and either tested or clearly marked as not tested with the reason. Any AC covering UI behavior requires an actual UI run (Step 11 — Playwright-first) to count as tested — a UI change with no such run must be reported as **not verified**, never as passing.

Do not commit or push while producing this output. Leave all changes in the working tree. If the work is ready to commit, present the diff summary and ask the user for explicit approval before running `git commit` or `git push` (Hard Gate #7). Only after that approval may a commit/push be made.
