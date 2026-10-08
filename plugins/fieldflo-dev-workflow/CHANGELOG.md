# Changelog

All notable changes to the `fieldflo-dev-workflow` plugin are documented in this file.

## [1.3.0] - 2026-10-08

### Added
- **`pr-review`: Hard Gate #7, Jira defect classification (new Step 8b, run just before the final output).** On `Bug`, `Defect` and `Release Defect` tickets the four classification fields (`Component`, `Found By`, `Root Cause Category`, `Escape Point`) are mandatory in Jira, and they kept being left empty after review. A finished review is the point where the evidence needed to fill them is already in hand.
  - The step reads the current values and live allowed options from `editmeta`, and proposes a value for each empty field with a one-line, evidence-backed rationale (ticket source, file path, `git blame` commit).
  - It writes nothing until the user replies `set` for those exact values, consistent with the "never write outside the codebase without per-action approval" rule.
  - It also runs when a review ends early (for example, when the PR is already merged).
  - Filled fields are never overwritten. Approval never extends to other fields such as `Root Cause Notes`.
- The step gives derivation rules per field. The key rule: `Regression` only when a recent, identifiable commit broke behavior that worked before. A latent flaw in long-standing code is `Coding`. Escape Point is the earliest stage that should reasonably have caught the defect.
- It also documents the write-payload gotcha: `components` must be sent **by id** (`[{"id": "…"}]`). A name-based payload is rejected with `You do not have permission to create new components` even when the name matches an existing component exactly (observed on CSE-8798).
- `pr-review` Step 9 final output now reports the classification state (set / declined / n/a).

### Changed
- **Playwright verification runs always use headless Chrome.** The committed `tests/aqa/playwright.config.ts` sets `headless: CONFIG.CI`, and `CONFIG.CI` defaults to `process.env.CI === 'true'`. Every local temp-spec run therefore opened a visible Chrome window.
  - The temp config template in `fieldflo-test-verifier` now sets `use: { ...base.use, headless: true }`. The spread is required: a bare `use: { headless: true }` would drop the Desktop Chrome profile, the viewport, the locale/timezone, `testIdAttribute` and the trace/video/screenshot-on-failure settings.
  - Setting `CI=true` is explicitly not the way to get headless, because it also turns on `forbidOnly` and `slowMo: 30`.
  - The old "`--headed` while debugging" allowance is removed. A failing spec is debugged from its retained trace, video and screenshot, or through the browser-MCP lane.
  - Verified with Playwright 1.56.1: the `headless` fixture reports `true`, CDP `Browser.getVersion` reports `HeadlessChrome/141.0.7390.37`, and viewport, locale and timezone still come from the base config.
- `develop-fieldflo` Step 11 now lists the headless override as the fourth load-bearing temp-config line and carries the "always headless, never `--headed`" rule. The `pr-review` UI-lane rule says the same.

## [1.2.0] - 2026-09-04

### Changed
- **UI verification is now Playwright-first across the whole plugin.** `fieldflo-test-verifier` previously drove Chrome DevTools MCP click-by-click for every UI check — slow, non-reproducible, and it left no re-runnable artifact. The agent now derives the flow from the implementation, writes a Playwright spec, and runs the *complete* test through the CLI. Chrome DevTools MCP is demoted to the debug lane: used only when the spec fails (DOM/console/network/visual state), or for selector discovery and visual verification.
- `fieldflo-test-verifier`: **temporary specs are explicitly permitted; changing the committed AQA suite is not.** The rule is now stated as an allow/forbid pair rather than a blanket "never touch", so the permitted case isn't read as prohibited. Nothing that already exists under `tests/aqa/` may be created, edited, deleted, or moved; a temp spec never gets a `qase(N)` id, a `@REGRESSION`/`@SMOKE`/`@SANITY` tag, or a Qase PATCH (an outward-facing write not covered by the testing approval); and if a flow cannot be driven without changing a page object, the agent stops and reports instead. Promoting a temp spec into the real suite stays a separate, explicitly-requested task.
- **The temp spec now lives in the ticket's own work folder, not in `tests/aqa/`.** Both `tmp.{TICKET_ID}.<feature>.test.ts` and its `playwright.tmp.config.ts` are written to `agents/context/{ROOT_ID}/{TICKET_ID}/`, which leaves the AQA project completely untouched — nothing is added to it, so there is nothing to clean up there — and keeps the spec beside the `Analysis.md` / `Changes.md` / `TestReport.md` that explain it. Verified working end to end (`1 test in 1 file` from a config four levels outside `tests/aqa/`).
- **Retention is now explicit and differs by location.** Specs in the ticket context folder **stay** — they are ticket record. A temp spec written **anywhere else must be deleted** when the run finishes, pass or fail, with `git status` confirmed clean of it. Generated run artifacts (`.pw-artifacts/`, plus any `playwright-report/` / `test-results/`) are always deleted, including from the context folder.
- `fieldflo-test-verifier`: documents the **throwaway runner config** the temp spec requires, and why it is not optional. `playwright.config.ts` declares `testDir: './tests/ui'` *plus* a fully-enumerated `projects` array whose `testMatch` globs are folder-anchored; none covers `tests/ui/tmp/` (the only all-folder project, `flaky`, is gated behind `grep: /@FLAKY/`). Verified against the real config: a temp spec run through the committed config reports `Error: No tests found. Total: 0 tests in 0 files`, while the same spec through a `playwright.tmp.config.ts` that keeps the base config but sets `projects: undefined` lists `1 test in 1 file`. Three further lines in that config are load-bearing and were each verified: **`tsconfig`** pointing at `tests/aqa/tsconfig.json` (without it the spec's `@/` alias imports fail to resolve and it reports `0 tests in 0 files` again — tested in both directions); **`outputDir`**, because with the config outside `tests/aqa/` Playwright otherwise writes `playwright-report/` and `test-results/` to the **repo root**, where no `.gitignore` covers them (observed); and **`reporter: [['list']]`**, which overrides the base config's inherited html/blob/Qase/Datadog reporter array as a second line of defence alongside `QASE_SKIP=true`.
- `fieldflo-test-verifier`: pins two run-time details that otherwise cost a cycle each — `QASE_SKIP=true` is **mandatory** on the command line (the Qase reporter is gated on that variable and is *enabled* whenever it is falsy, so an unguarded local run pushes results to Qase TestOps), and the runner must be invoked as `./node_modules/.bin/playwright`, not bare `npx playwright`, which can resolve a cached global Playwright that dies with `MODULE_NOT_FOUND` on `src/config.ts`.
- **`fieldflo-test-verifier`: the "no click-by-click" rule now also governs the debug lane.** Added concrete batching guidance for both browser MCPs: `take_snapshot` (preferred over `take_screenshot`) for uid handles, `fill_form` to set many fields in a single call instead of per-field `fill`/`click`, `evaluate_script` to assert page state without touching the UI (`waitForStableDom: false` for read-only scripts, `filePath` to keep large output out of context), and `browser_batch` / `javascript_tool` on the claude-in-chrome side for multi-step sequences in one round trip. Reiterates the standing rule never to trigger a JS dialog, which freezes the extension.
- `develop-fieldflo`: Step 11 rewritten to match — Playwright-first, temp spec + temp config, `QASE_SKIP=true`, browser MCP as the failure/debug path, and a batching reminder. Step 2.5.4 and the Step 13 completion rule reworded from "browser exercise" to "an actual UI run", so the wording no longer implies the browser MCP is the only qualifying evidence.
- `pr-review`: the UI lane row, the mandatory-UI-lane rule, the ✅ Approved criterion, the UI verification rule, and the closing reminder all realigned to the same Playwright-first contract, so the two skills and the shared agent no longer contradict each other.
- `develop-fieldflo`: the Step 0 work-folder listing now notes that a UI ticket also keeps its temp spec and runner config there, so the extra files aren't mistaken for stray output.

### Note
- This release pairs with a matching amendment to the repo's shared `Portal/CLAUDE.md`, whose standing rule ("do not create, edit, delete, or run AQA / Playwright specs … unless the user explicitly asks") otherwise contradicted the workflow head-on. The amendment carves out exactly this case — a temporary, ticket-scoped spec in the ticket's work folder — and leaves every other part of the prohibition intact, including broad suite runs (`make aqa`, `regression:*`), which still need an explicit ask. That file lives in the portal repo, not in this plugin.
- `develop-fieldflo`: Hard Gate #0 reclassifies **Chrome DevTools / browser automation** as *degraded, not blocking*. The primary UI path is a CLI Playwright run needing only Node and the AQA project, so a missing browser MCP no longer reads as a blocked UI requirement — but a *failing* spec then cannot be diagnosed in-browser, and the gate says to report that rather than silently downgrade the evidence.

### Added
- `develop-fieldflo`: **a fifth Hard Gate #0 dependency check — the Playwright runner**, checked only when the ticket has a UI surface. `tests/aqa/node_modules` is gitignored and routinely absent on a fresh machine, which turns the first UI verification into a confusing failure. The gate probes `node --version` and `tests/aqa/node_modules/.bin/playwright`, and classifies a miss as degraded-not-blocking: it is a one-time `npm install` plus `npx playwright install chromium`, to be installed when reached and called out because the first run pays that cost.

## [1.1.1] - 2026-08-17

### Changed
- `develop-fieldflo`: **the Step 1.1 status-transition gate now fires for `To Do` as well as `Ready for Dev`.** Previously a ticket sitting in `To Do` fell into the "anything else → continue without transition" branch, so the run branched and implemented while Jira still showed the ticket as untouched — the move to `In Progress` only happened if the user noticed and asked for it by hand. Both statuses now mean the same thing for this workflow: work is starting, so offer the transition. Hard Gate #1 is reworded to match.
- `develop-fieldflo`: the gate's question no longer hardcodes the status name. It interpolates `{CURRENT_STATUS}` from what was actually read via the Jira MCP, so a `To Do` ticket isn't asked about as though it were `Ready for Dev`. Status matching is also specified as case-insensitive and tolerant of spacing/separator variants (`Ready For Dev` / `Ready for Dev` / `To Do` / `TO DO`), since board configurations differ on the exact rendering.

## [1.1.0] - 2026-07-27

### Added
- **Per-agent MCP scoping — every subagent now gets exactly the MCP tools its role needs, and nothing else.** Previously only `fieldflo-test-verifier` had any MCP access, so agents whose instructions named an MCP as mandatory (Jira for context analysis, MySQL for DB verification) silently degraded to whatever the orchestrator had already gathered. Every grant below is read-only: no `editJiraIssue`/`transitionJiraIssue`/`addCommentToJiraIssue`, no Confluence create/update, no `slack_send_message*`, no `notion-create*`/`notion-update*`, no `sentry__update_issue`/`execute_sentry_tool`. Slack grants deliberately exclude `slack_search_users`/`slack_read_user_profile` (PII).
  - `fieldflo-context-analyst` and `fieldflo-pr-functional-reviewer`: Jira read (6), Confluence read (4), Slack read (4), Notion read (3).
  - `fieldflo-architect-reviewer`: Jira read (6) for AC/merge-readiness, `context7` for API-contract verification, Sentry read (5) for production error context.
  - `fieldflo-planner`: `context7` plus `mysql_query` for schema/table inspection.
  - `fieldflo-implementer`: `context7`, so it can consult official SDK/framework docs before implementing against them.
  - `fieldflo-test-verifier`: `mysql_query` added — the agent's own procedure said "Prefer MySQL MCP for DB checks" but the tool was never granted. Keeps `chrome-devtools` + `claude-in-chrome`.
  - `fieldflo-mechanical-editor` and `fieldflo-pr-comment-composer`: intentionally zero MCP, now documented in-file so the exclusion isn't mistaken for an oversight (the comment composer reaches GitHub through the `gh` CLI, never an MCP).
- `reset-environment`: **a mandatory `db_migrations` wait gate between the rebuild and the health check.** `db_migrations` is a one-shot task container that applies HQ -> master -> every client migration after `down -v` wipes the volumes, and it takes minutes while the portal container already answers requests — so an early `curl` either 500s on a half-migrated schema or returns a misleading 200 from a page that doesn't touch the pending tables. The gate follows the run, then treats the container's terminal state (`exited 0`) as the only authoritative signal, with a poll loop instead of a guessed duration.
- `reset-environment`: handling for the held-failure mode. A failed `prisma migrate deploy` does **not** exit — `migrations/entrypoint.sh` touches `/tmp/.failed_migration` and loops forever printing "Waiting to fix migrations", so "still running" is ambiguous and the log must be read before concluding it's just slow. Documents the release step (`rm /tmp/.failed_migration`), the `restart: on-failure:3` auto-retry that follows it, and `make db` as the manual fallback.

### Changed
- `reset-environment`: the opt-in ask now lists the migration wait as step 4, so the user's `yes` covers it; the health-verification section is gated on `db_migrations` having exited 0; and the "DB not migrated / empty" root cause now points back at the new gate. Skill `description` updated to match.
- `reset-environment`: added `allowed-tools` — Docker + curl only, no MCP reachable during a reset. The list intentionally keeps the orchestration primitives (`Task`/`Agent`/`Skill`/`AskUserQuestion`), because this skill is invoked mid-run by `develop-fieldflo` and `pr-review` and a skill's `allowed-tools` also applies to what it invokes; the restriction that matters here is "no MCP", not "no tools".

## [1.0.1] - 2026-07-22

### Changed
- `fieldflo-test-verifier`: added `mcp__chrome-devtools` and `mcp__claude-in-chrome` tool access, so the agent can drive its own browser-based UI verification steps instead of relying on the orchestrator for them.

## [1.0.0] - 2026-07-20

### Added
- Initial release of the plugin.
- `develop-fieldflo` orchestrator skill (with a startup dependency-check gate), plus `pr-review`, `prisma-migrations`, `update-openapi`, and `reset-environment` skills (dependencies for work).
- `fieldflo-*` subagent team: `fieldflo-architect-reviewer`, `fieldflo-context-analyst`, `fieldflo-implementer`, `fieldflo-mechanical-editor`, `fieldflo-planner`, `fieldflo-pr-comment-composer`, `fieldflo-pr-functional-reviewer`, `fieldflo-test-verifier`.
