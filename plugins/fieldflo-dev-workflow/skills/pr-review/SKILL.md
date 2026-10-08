---
name: pr-review
description: Orchestrated FieldFlo PR review from a Jira ticket — triages the change by size × risk, then engages only the subagents and model tier that fit. Casual-human inline comments, gated GitHub submission.
argument-hint: "ticket_id=<TICKET_ID> [epic_id=<EPIC_ID>] [pr_id=<PR_ID>] [profile=trivial|small|standard|high]"
model: sonnet
effort: high
---

# Skill: PR Review From Ticket (Orchestrated, Right-Sized)

Run this skill to review a FieldFlo PR using a Jira ticket as the source of truth. This skill is the **workflow controller (orchestrator)**: it gathers context, **triages the change**, fans out only as much review as the change warrants, composes inline comments, and submits to GitHub only behind an explicit approval gate.

Repository rules live in `CLAUDE.md`. Per-role review behaviour and accumulated FieldFlo review lessons live in `.claude/agents/*.md`.

> **Plugin scoping note.** This copy ships inside the `fieldflo-dev-workflow` plugin. Every subagent named below by its bare name (`fieldflo-context-analyst`, `fieldflo-pr-functional-reviewer`, `fieldflo-architect-reviewer`, `fieldflo-test-verifier`, `fieldflo-pr-comment-composer`) must be invoked with the fully scoped `subagent_type`: `fieldflo-dev-workflow:<agent-name>` (e.g. `fieldflo-dev-workflow:fieldflo-architect-reviewer`). Likewise, invoke the sibling `reset-environment` skill in its scoped form (`fieldflo-dev-workflow:reset-environment`). If a bare-name invocation fails to resolve, retry with the scoped form.

This skill assumes:

- The PR may be authored by another developer.
- No prior analysis exists — reconstruct what you need from MCPs.
- If the PR was reviewed before, **start the analysis from scratch** — do not reuse prior findings; note differences at the end.

> **Consolidation note:** This orchestrated `/pr-review` is the **primary** PR-review path and absorbs the parallel fan-out that `pr-review-swarm` used to provide. `pr-review-swarm` is **deprecated**.

> **Core principle — right-size the review.** Do not spend five Opus subagents on a two-file additive change, and do not skim a migration that touches tenant data. Step 2 triages the PR on two axes — **amount of change** and **required analysis (risk)** — and selects a Review Profile that decides how many lanes run, on which model, and what the orchestrator handles inline. The default posture is the **lightest profile that still covers the risk**.

---

## Required Input

Required:

- `ticket_id`

Optional:

- `epic_id`
- `pr_id`
- `profile` — force a profile (`trivial` | `small` | `standard` | `high`), overriding Step 2 triage. Omit to let the skill triage automatically.

Usage:

```text
/pr-review ticket_id=CSE-5877
/pr-review ticket_id=CSE-5877 profile=high      # force full fan-out
```

If `epic_id` is missing: use the Jira parent/epic if available, else `ticket_id` as the root folder.

If `ticket_id` is missing, return:

```text
Usage:   /pr-review ticket_id=<TICKET_ID> [epic_id=<EPIC_ID>] [pr_id=<PR_ID>] [profile=...]
Example: /pr-review ticket_id=CSE-5877
```

Work folder: `agents/context/{ROOT_ID}/{TICKET_ID}/`

Files this skill **may** produce (only those the chosen profile needs):

```text
Analysis.md          # context — only when a context subagent runs (Standard/High)
FunctionalReview.md  # only when the functional lane runs as its own agent
ArchitectReview.md   # the review lane's findings (always, for Small+)
UIReview.md          # only when the UI lane actually runs
InlineComments.md    # staged comments
inline-comments.json # gh-ready payload
PR_Review.md         # final synthesized report (always)
SubagentLog.md       # delegation ledger (always)
```

Do not store secrets, tokens, credentials, or raw PII in these files.

---

## Agents & Review Profiles

### Base agent roles

| Lane | Agent | Default purpose |
|---|---|---|
| Context | `fieldflo-context-analyst` | Jira/Slack/Notion/PR synthesis, AC extraction. |
| Functional | `fieldflo-pr-functional-reviewer` | AC-by-AC coverage + functional defects on the finished diff. |
| Architecture / security | `fieldflo-architect-reviewer` | Code quality, **security (any violation = blocker)**, DB/Prisma, tenant, cross-system, regression, sensitive-data. Also the **combined** reviewer for Small PRs (add AC coverage to its prompt). |
| UI / behavioural | `fieldflo-test-verifier` | Playwright-first UI verification (temporary spec, run via CLI) + test-data verification. Browser MCP only to debug a failure. |
| Comments | `fieldflo-pr-comment-composer` | Dedup findings → casual-human inline comments + gh payload (never submits). |

### Profiles (Step 2 selects exactly one)

| Profile | When | Context | Review lanes | Comments |
|---|---|---|---|---|
| **Trivial** | **Non-code only** — docs/markdown/comments/copy/config; **0 risk triggers**. | orchestrator inline | **orchestrator inline — 0 subagents** | orchestrator inline |
| **Small** | ≤ ~3 application files **and** ≤ ~150 changed app lines **and** **0 risk triggers**. | orchestrator inline (+ one targeted Slack search only if the ticket cites prior discussion) | **1 subagent:** `fieldflo-architect-reviewer` @ **Sonnet**, prompted to do **AC coverage + code quality + security** in one pass. UI lane **only if** there is a UI surface **and** it is locally testable. | orchestrator inline |
| **Standard** | More than Small, but no risk trigger; or moderate logic across a few subsystems. | `fieldflo-context-analyst` @ **Sonnet** | **parallel:** `fieldflo-pr-functional-reviewer` @ Sonnet **+** `fieldflo-architect-reviewer` @ **Opus xhigh**; **+ UI** if UI surface & testable. | composer agent if >~5 findings across lanes, else orchestrator inline |
| **High / Large** | **Any risk trigger**; or > ~10 app files; or multiple subsystems; or the user asks for a thorough/deep review (or `profile=high`). | `fieldflo-context-analyst` @ **Opus xhigh** | **full parallel fan-out:** functional + architect + UI, all @ **Opus xhigh**. | composer agent |

**Selection rules (read these — they prevent the two failure modes):**

1. **Risk triggers dominate size, always.** If any risk trigger is present, the floor is **Standard** regardless of line count, and the triggering concern's lane runs at **Opus xhigh**. A one-line change to a migration or an auth check is *not* Small.
2. **Ties round UP.** When a PR sits between two profiles, pick the heavier one. The asymmetry is deliberate: over-reviewing wastes tokens; under-reviewing merges a bug or vuln into someone else's PR. Spend the tokens.
3. **Mid-review escalation.** If a Small/Standard lane surfaces a genuine high-risk finding (security, DB, tenant, API, payroll), escalate **that finding** to `fieldflo-architect-reviewer` @ Opus xhigh before forming the verdict, and log the re-route.
4. **`profile=` input overrides triage** but rules 1–3 still apply on top (you may always go heavier than forced).

### Risk triggers (ANY → minimum Standard; that concern → Opus xhigh)

- **DB:** `migrations/`, `*.prisma`, `*.sql`
- **API:** `apidoc/`, API DTOs (`application/api/`, `_*.php`), `Apirest_v*` controllers
- **Auth / permissions:** `application/Modules/Auth/`, `isGranted`, permission helpers, `MY_Controller` auth paths
- **Payroll:** payroll models/controllers/exports
- **Tenant / master:** master-DB queries, `companyUUID` / tenant-scoping changes
- **PII:** `application/Modules/PII/`, any new field storing personal data
- **New write / CSRF surface:** new POST/create/update/delete endpoints or forms
- **Dangerous patterns in the diff (the *unsafe* variant only):** raw/concatenated SQL or string-built `WHERE`/`ORDER BY`/`LIMIT`/`LIKE`; `unserialize(` on non-constant data; `exec(`/`shell_exec(`/`system(`/`passthru(`; file-upload handling; **raw, unescaped** echo/interpolation of DB/user values into HTML or `<script>`. A value rendered through `json_encode(..., JSON_HEX_*)` or `htmlspecialchars()` is the **safe** pattern and does **not** trip this trigger — the Small profile's combined reviewer already covers escaping correctness.

### UI surface

A UI lane is warranted only if the diff changes `application/views/`, view templates, `.js`, or `.css`.

- **If there is a UI surface and it is locally testable, the UI lane is MANDATORY** — a UI change cannot be marked verified/approved without an actual UI run. Do not skip it to save tokens.
- **The UI lane runs Playwright, not click-by-click browser automation.** The agent writes a *temporary* spec plus a throwaway `playwright.tmp.config.ts` into the ticket's own work folder (`agents/context/{ROOT_ID}/{TICKET_ID}/`, never `tests/aqa/`) and runs it via CLI, with Chrome always headless (never `--headed`). Temporary specs are allowed; changing the committed AQA suite is not — nothing existing under `tests/aqa/` is created, edited, deleted, or moved, and a temp spec is never tagged, committed to the suite, or Qase-synced. Specs in the context folder stay as ticket record; anything written elsewhere is deleted when the run finishes, and generated artifacts always are. Chrome DevTools MCP is the **debug/selector-discovery** path, used only when the spec fails; when in the browser, batch (`take_snapshot` → `fill_form` → `evaluate_script`, or `browser_batch`) rather than acting one step at a time. The agent definition owns the procedure.
- **Skip the UI lane (no agent) only if** there is no UI surface, or the surface genuinely cannot be exercised locally (feature-flag gated off, no test-data path). Record the skip reason — do not spawn an agent that can only report "couldn't test." In that case the UI is **not verified**: the verdict must reflect that (see Step 7), not silently pass.

---

## Static-Check Scoping (applies at EVERY profile)

This is orthogonal to profile and is the single biggest avoidable cost — **never run repo-wide static analysis for a small diff.**

- **Default everywhere:** scope static checks to the changed files.
  - `php -l` on each changed `.php` file. For Trivial/Small the **orchestrator runs this inline** — do not keep a subagent alive to lint a couple of files.
  - `phpcs` / `phpstan` on the changed paths only.
- **Full suites** (`make lint`, `make codesniff`, `make phpstan`, `composer test`, `make aqa`) run **only** when the diff is broad (many files/subsystems) or the user explicitly asks. A 2-file PR never triggers a repo-wide `make phpstan`.

---

## Subagent Usage Tracking

Track **every** delegation in `agents/context/{ROOT_ID}/{TICKET_ID}/SubagentLog.md` so the run ends with a complete usage report (Step 9).

Rules:

1. Log one entry **every time** a subagent is invoked — append at the moment of delegation, update `Outcome` when it returns.
2. Append as you go; re-routes/escalations/re-runs are each a separate entry.
3. **Do not invent token/cost figures** — track who/what/why/outcome only.
4. Note any deviation from the profile's default agent/model and why.
5. No secrets/PII in the log.
6. **Time each delegation** with `date '+%Y-%m-%d %H:%M:%S'` immediately before and after. `Duration` is wall-clock (`mm:ss`), includes orchestration overhead — label it as such.
7. **Record the chosen profile** as the first line of the log so the orchestration size is auditable.

Format:

```markdown
# {TICKET_ID} PR Review — Subagent Log
**Profile:** Small (2 app files, 0 risk triggers, UI gated off → 1 reviewer)

| # | Step | Agent | Model / Effort | Purpose | Start | End | Duration | Outcome |
|---|---|---|---|---|---|---|---|---|
| 1 | Step 4 — Review | fieldflo-architect-reviewer | Sonnet / high | Combined AC + quality + security | … | … | … | success |
```

`Outcome` ∈ `success`, `revised`, `escalated → <agent>`, `routed back → <agent>`, `failed`.

---

## Hard Gates

1. **PR cannot be resolved** — no linked PR and none found by branch. Stop, ask.
2. **Branch cannot be checked out.** Stop, ask.
3. **Missing description AND ACs** in Jira — stop, ask (cannot validate functional coverage without requirements).
4. **Material blocker** — ambiguity/conflict in scope, security, tenant isolation, schema, API contract, data integrity, or destructive operation.
5. **Posting to GitHub.** Never run `gh pr review`, `gh pr comment`, or any `gh api ... POST`/PATCH against the PR until the user explicitly approves that action. Preparing comments locally is allowed; publishing is always gated.
6. **Environment reset offer** (see "Environment Reset Gate" below) — ALWAYS ask before any review lane that **executes the running app** (a UI lane or functional verification against the live environment). Asking is mandatory; running the reset only happens if the user says yes. Pure static review (no app execution) does not need it.
7. **Jira defect classification** (see Step 8b). On a defect-type ticket (`Bug`, `Defect`, `Release Defect`), the run may not close out until `Component`, `Found By`, `Root Cause Category` and `Escape Point` are filled. Proposing values for the empty fields is mandatory. Writing them to Jira happens only on an explicit `set` for those exact values.

The assignment check (Step 1) runs automatically and never blocks. This skill is **read-only w.r.t. the codebase and VCS** — it reviews; it never edits application code, commits, pushes, or rewrites git history. (The Environment Reset Gate's `git checkout` steps are existing-branch checkouts only — no branch creation/commit/push — so they're compatible with this read-only posture; `docker compose down -v` is local-data only, not VCS.)

---

## Environment Reset Gate (ask before executing the running app)

Before any review activity that **runs the app** (a UI lane via Chrome DevTools MCP, or functional verification against the live local environment), ALWAYS apply the shared **`reset-environment`** skill. That skill owns the mandatory yes/no ask, the clean-rebuild commands, **and the post-rebuild health verification** (index must return HTTP 200 → remediate with `composer install` in the portal container → root-cause investigation if still failing). Invoke it with:

- `base_branch` = the PR's target branch (a release branch like `web-26-07-02-x1`, else `develop`)
- `return_branch` = the PR branch
- `health_url` = the local portal at the slug the review will exercise (default `https://portal.fieldflo.dev/dev1`)

Implicit, mandatory **ask** each time the gate is reached — the user opts in or out; asking is mandatory, the reset runs only on `yes`. After a `yes` reset the local DB/volumes are fresh — re-seed any data the functional/UI check needs. Skip the ask only when the chosen profile does **no** app execution (e.g. Trivial/Small static-only reviews). `docker compose down -v` is local-data only (volumes), not a VCS action — compatible with this skill's read-only posture; the yes/no answer is the authorization.

---

## Step 0 — Initialize Work Folder

Create or reuse `agents/context/{ROOT_ID}/{TICKET_ID}/`. If a prior PR review exists here, acknowledge it, **redo from scratch**, and note differences at the end. Start `SubagentLog.md` with the header (profile line filled in after Step 2).

---

## Step 1 — Branch Setup, PR Resolution & Assignment (orchestrator)

### 1.1 — Assignment check (first)
Fetch the ticket (`getJiraIssue`), read `assignee`. If unassigned, assign to the current user (`srdjan@fieldflo.com`) via `editJiraIssue`. If it fails, note and continue — never block.

### 1.2 — Resolve the PR
Read linked PR(s)/branch from Jira; else `gh pr list --head <branch>`. Multiple PRs → review all. None → Hard Gate #1.

### 1.3 — Check out the PR branch (review real code)
```bash
git fetch origin
git checkout <branch>
git pull origin <branch>
git log --oneline -3
```
Determine the base branch (the PR's target — a release branch like `web-26-07-02-x1`, else `develop`). Checkout failure → Hard Gate #2. Check out and pull only — never create branches, commit, push, or rewrite history.

---

## Step 2 — Triage & Profile Selection (orchestrator) — **the optimization step**

> **3-DOT GUARD (mandatory).** Compute every size/path signal from the **three-dot** diff / GitHub file list — the true PR contents — never the two-dot diff. A branch behind its base shows unrelated base-ahead files in `git diff base..HEAD`; reading those misclassifies a tiny PR as Large. Use:
> ```bash
> git diff <base>...HEAD --stat        # three dots
> gh pr view <pr> --json files          # authoritative file list
> ```

> **Ignored paths (not reviewed).** Exclude the **`agents/`** folder from the review entirely — it holds the per-ticket work-log / context docs (`agents/context/{ROOT_ID}/{TICKET_ID}/` — `Analysis.md`, `Plan.md`, `Investigation.md`, `Changes.md`, `TestReport.md`, etc.), not product code. `agents/` files do **not** count toward triage size signals, are **not** validated against ACs, receive **no** code/security review, and get **no** inline comments. Filter them out before computing any signal and before handing the file set to any lane. (A PR whose only changes are under `agents/` has nothing to review — say so and stop.)

1. Get the true file set and line counts (3-dot), **after dropping ignored paths (`agents/`)**. Separate the remaining **application files** from docs/tests/config.
2. Scan the diff for **risk triggers** (list above) and **UI surface**.
3. Select the profile from the table using the selection rules: risk triggers force ≥ Standard and Opus on that concern; ties round up; honor a forced `profile=`.
4. Write the chosen profile + the one-line rationale (file count, risk triggers found/none, UI surface yes/no) into `SubagentLog.md` and report it to the user before proceeding:

   ```text
   Triage: <Profile> — <N> app files, <risk triggers or "no risk triggers">, UI surface <yes/no, testable?>.
   Plan: <which lanes/agents will run, on which model; what the orchestrator handles inline>.
   ```

This single decision is what keeps the review proportionate. Spend the rest of the run executing only what the profile calls for.

---

## Step 3 — Context (profile-aware)

- **Trivial / Small:** the orchestrator already has Jira (Step 1) and the diff. Gather context inline — read the diff and the ticket. Run a **single targeted Slack search only if** the ticket cites prior discussion/decisions; skip Notion unless the ticket points to a doc. Do **not** spawn `fieldflo-context-analyst`. Write a short context note into `PR_Review.md` directly (no separate `Analysis.md`).
- **Standard:** delegate to `fieldflo-context-analyst` @ Sonnet → `Analysis.md` (Jira mandatory; Slack/Notion bounded; PR comments). Escalate to Opus if a risk trigger fired.
- **High / Large:** delegate to `fieldflo-context-analyst` @ Opus xhigh → `Analysis.md`.

**Missing-requirements gate:** if Jira has no meaningful description **and** no ACs → Hard Gate #3.

---

## Step 4 — Review (profile-aware)

**Environment Reset Gate:** if this profile includes a lane that **executes the running app** (a UI lane, or functional verification against the live environment), apply the Environment Reset Gate (see above) — ask the user whether to reset the local Docker environment before that lane runs. Static-only profiles skip the ask.

Dispatch independent lanes **concurrently** (single message, parallel `Agent` calls). Every lane prompt must (a) state the base branch and the 3-dot file set **with ignored paths already excluded (`agents/` — see Step 2)**, (b) tell the lane to **write its output file in one pass** (no "draft then finalize" round-trips), and (c) carry the relevant slice of context.

- **Trivial:** no review subagent. The orchestrator reviews the doc/config diff inline against the ticket, plus `php -l` on any changed `.php`.
- **Small:** **one** subagent — `fieldflo-architect-reviewer` @ **Sonnet** — prompted to cover **AC-by-AC coverage + code quality + security** in a single pass, writing `ArchitectReview.md`. Add the UI lane whenever there is a testable UI surface — mandatory for UI changes (see **UI surface**). The orchestrator runs `php -l` on changed files inline.
- **Standard:** `fieldflo-pr-functional-reviewer` @ Sonnet **+** `fieldflo-architect-reviewer` @ Opus xhigh, parallel; **+ UI** lane if a testable UI surface exists.
- **High / Large:** functional + architect + UI, all @ Opus xhigh, parallel.

For Standard/High the architect lane must explicitly cover: code quality & conventions, security (any violation = BLOCKER), DB/Prisma, cross-system impact, regression, sensitive-data. (The agent holds the FieldFlo lessons — don't restate them in the prompt.)

Apply **Static-Check Scoping** here: scoped to changed files by default; full suites only for broad diffs or on request.

---

## Step 5 — Comment Composition (profile-aware)

- **Trivial / Small, or ≤ ~5 findings:** the **orchestrator composes** the inline comments directly (few findings, no cross-lane dedup needed). Use the tone spec below verbatim — do not regress to linter voice.
- **Standard with >~5 findings / High:** delegate to `fieldflo-pr-comment-composer` to dedup across lanes, drop anything already in existing PR comments, and stage `InlineComments.md` + `inline-comments.json`.

Either way the comments are **staged, never posted** — submission is Step 8 behind the gate.

### Comment Tone & Style (MANDATORY on every path, inline or composer)

Write like a casual but technically sharp senior colleague — not a robot, not an essay.

**Do:** be direct and conversational; short sentences; show you get the intent ("I see what you're going for, but…"); suggest concrete fixes with a snippet when useful; use "we" / "our pattern for this is…"; add context ("the PII module handles this for us"); be encouraging on good work.
**Don't:** sound like a linter; use "it is recommended that"/"please be advised"; write walls of text; be condescending; use emojis unless the team does.

Each staged comment carries: `File`, `Line`, `Type` (FUNCTIONAL|SECURITY|QUALITY|DATABASE|REGRESSION|UI), `Severity` (LOW|MEDIUM|HIGH|CRITICAL), and the comment text. A finding whose line isn't in the diff goes in the review **body**, not as a diff-anchored comment.

---

## Step 6 — Report Synthesis (orchestrator)

Write `PR_Review.md`: Summary (ticket, PR, branch, base, profile, status, re-review) · Feature Understanding · Context Sources · AC Coverage table · Functional / Code-Quality / Security / Database / UI issues · Test data · Regression · Cross-System · Slack/Notion insights · Sensitive-data check · Required Changes · Reviewer agreement (findings caught by ≥2 lanes, when multiple lanes ran).

---

## Step 7 — Severity Classification

- ✅ **Approved** — no issues, all ACs covered, security clean. **A PR with UI changes may be Approved only if the UI was actually exercised** (the UI lane ran) — never on static grounds alone.
- ⚠️ **Needs Changes** — non-blocking issues only.
- ❌ **Blocked** — broken functionality, missing AC coverage, any security vulnerability, raw/non-Prisma DB change, Prisma out of sync, data leakage / PII violation, or cross-tenant isolation breach.

**UI verification rule:** a UI change is **not verified** until it has actually been run (Playwright-first, per the UI lane). If the UI lane ran, cite the evidence. If it was skipped because the surface wasn't locally testable, the UI portion is **not** ✅ Approved — report it as ⚠️ Needs Changes / explicitly-unverified with the reason, never a silent pass.

---

## Step 8 — Approval Gate & Submission (HARD GATE #5)

Present: final status + top issues; the staged `InlineComments.md` and count; the exact `gh` command(s); path to `PR_Review.md`. Ask exactly:

```text
Should I post these as inline comments on the PR? (Reply `post` to publish, or give feedback to revise.)
```

Only on `post` submit a single PR review (`gh api repos/FieldFlo/portal.fieldflo.com/pulls/{pr}/reviews`, `event=COMMENT`, `comments[]`, head SHA), verify with `gh pr view <pr> --comments`, report the URL. Feedback → revise (orchestrator or composer) and re-present.

---

## Step 8b — Jira Defect Classification (HARD GATE #7)

Runs after Step 8's gate is settled (posted, revised, or declined), right before the final output. It also runs when the review ends early, for example when the PR is already merged or the user declines the review. The gate itself only asks a question, so offering it never writes anything.

**Scope:** defect-type tickets only (`Bug`, `Defect`, `Release Defect`). For any other issue type, skip this step and say so in one line.

### 8b.1 — Read the current values and the allowed options

Fetch the ticket with `getJiraIssue`, `expand: "editmeta"`, and `fields: ["issuetype", "components", "customfield_11064", "customfield_11065", "customfield_11066"]`. If the response overflows, `jq` the saved file. Read the allowed options from `editmeta` every time and never from memory, because the option lists change.

| Field | Jira id | Write shape |
|---|---|---|
| Component | `components` | `[{"id": "<component id>"}]` |
| Found By | `customfield_11064` | `[{"value": "<option>"}]` |
| Root Cause Category | `customfield_11065` | `[{"value": "<option>"}]` |
| Escape Point | `customfield_11066` | `[{"value": "<option>"}]` |

Set `components` **by id**. A name-based payload (`{"name": …}`) is rejected with `You do not have permission to create new components` even when the name matches an existing component exactly (observed on CSE-8798).

If all four fields are already filled, report their values and continue. Never overwrite a filled field. If a filled value looks wrong, say so and leave it unchanged unless the user asks for the change.

### 8b.2 — Derive a proposal for each empty field from evidence

- **Found By** (who first surfaced the defect):
  - `Customer`: the ticket cites an Intercom conversation, a client tenant/portal URL, or a support reporter, or the issue happened in production.
  - `QA`: found during QA or release testing. This is typical for `Release Defect` tickets and QA-parented tickets.
  - `Developer`, `Automation` (AQA run or automated monitoring) or `Other` when the evidence says so.
- **Component:** the module where the fix lands. Map the changed application files to the closest, most specific component. For example, `views/forms/pdf/custom_form.php` maps to `Project / Custom Forms`, not `Project / Main Module`. Use one component unless the fix truly spans two.
- **Root Cause Category:** decide from the fix plus `git blame` / `git log` on the lines it changed.
  - `Regression` **only** when a recent, identifiable change broke behavior that worked before. Name that commit.
  - `Coding`: a latent logic flaw in long-standing code.
  - `Data`: the code is correct and unexpected tenant data triggered the failure.
  - `Validation`: input validation is missing.
  - `Requirements`: the code was built as specified and the spec was wrong or missing.
  - `Config`, `Design`, `Environment`, `Integration`, `Timezone`, `Not a Defect` and `Unknown` as the evidence dictates.
- **Escape Point:** the earliest stage that should reasonably have caught the defect.
  - `Dev Test`: an edge or boundary case of the code's own purpose that a developer could reproduce.
  - `Code Review`: the flaw is visible in the original diff.
  - `Release Testing`: a normal QA flow would have hit it.
  - `Automated Regression`: an area the AQA suite covers.
  - `Requirements`: the gap was in the spec.
  - `Not Catchable`: it depends on production-only data or conditions that no reasonable pre-release test would hit.

Each proposal carries a one-line rationale that cites its evidence (ticket source, file path, commit hash). When two options are close, name the runner-up and say why you rejected it.

### 8b.3 — Gate

Show the proposal table (field · proposed value · why) and ask exactly:

```text
Should I set these classification fields on {TICKET_ID}? (Reply `set` to write them, or give corrections.)
```

- Only `set`, or an equally explicit yes, authorizes the write. Call `editJiraIssue` with only the approved fields, then re-read the ticket to confirm the values landed. A rejected call writes nothing, so fix the payload and retry before reporting.
- If the user gives corrections, revise the proposal and ask again.
- If the user declines, write nothing. Record "Jira classification: not set (declined)" in `PR_Review.md` and in the final output.
- Approval covers these four fields on this ticket only. It never extends to other fields (e.g. `Root Cause Notes`), to status transitions, to comments, or to other tickets.

---

## Step 9 — Final Output

Return: Status · Top Issues · AC Coverage · Security Review · Cross-System/Regression Risks · Report Path · Inline Comments (count; staged/posted) · Jira Classification (set / declined / n/a) · **Subagent Usage**.

`## Subagent Usage` prints inline: the **chosen profile + rationale**, the full ledger from `SubagentLog.md` (with `Start`/`End`/`Duration` and a **Total** row), then a short summary — total invocations, per-agent and per-model counts, total wall-clock + longest delegation, escalations/re-routes, deviations from the profile defaults. Durations are wall-clock (label as such); no token/cost figures.

Don't claim completion unless every applicable AC was assessed and every lane the profile called for ran (or was explicitly skipped with a reason).

---

## Continuous Learning

If the review surfaces a generalizable lesson (recurring pitfall, missed convention, security gap — from a lane or an existing PR comment), propose capturing it per `CLAUDE.md` → "Continuous Learning from Reviews". **Always ask before writing it** — never edit a skill/agent/`CLAUDE.md` silently.

---

## Important Rules

- **Triage first, then right-size** — pick the lightest profile that still covers the risk; risk triggers force ≥ Standard; ties round up.
- **Triage from the 3-dot diff only** — never the 2-dot diff (base-ahead files misclassify the PR).
- **Ignore the `agents/` folder for review** — per-ticket work-log/context docs, not product code: excluded from triage signals, AC validation, code/security review, and inline comments.
- **Static checks scoped to changed files** at every tier — repo-wide `make` suites only for broad diffs or on request.
- **UI changes require an actual UI run** — run the UI lane whenever the surface is testable (mandatory); skip only when there's no UI surface or it isn't locally testable, and then report the UI as **not verified** (never a silent pass).
- **Lane prompts write output in one pass** — avoid draft/finalize round-trips.
- **Assignment first; never assume prior context; redo from scratch if previously reviewed.**
- **Always check out the PR branch locally.** Jira is the source of truth; validate every AC.
- **Security violations / raw-SQL DB changes are blockers.**
- **Never post to GitHub without explicit `post`** (Hard Gate #5). Never commit/push/rewrite history.
- **Defect tickets close with Jira classification** (Hard Gate #7): propose `Component`, `Found By`, `Root Cause Category` and `Escape Point` for any empty field, and write them only on `set`.
- **DB access only via MySQL MCP or Docker CLI.** Never store secrets/tokens/PII/credentials in any output.
- **Don't fabricate findings** — an empty lane is a valid result.
