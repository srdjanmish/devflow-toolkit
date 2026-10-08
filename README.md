# FieldFlo Dev Workflow — Claude Code plugin

Internal Claude Code plugin bundling the FieldFlo Jira-ticket dev workflow so a
teammate can get it all with one install instead of copying loose files.

## What's included

- **Skills** (`plugins/fieldflo-dev-workflow/skills/`)
  - `develop-fieldflo` — the ticket orchestrator (context → plan → approval → implement → test → review). Now starts every run with a **Hard Gate #0 dependency check** — before Step 0 it verifies the required subagents, skills, and MCP-backed capabilities are actually available in the session, and if anything's missing it stops and asks `Do you want to continue, work will be limited? (yes / no)` before proceeding.
  - `pr-review` — orchestrated PR review from a Jira ticket, triaged by size × risk. On defect tickets it closes with a gated Jira classification step: it proposes `Component`, `Found By`, `Root Cause Category` and `Escape Point`, and writes them only on `set`.
  - `prisma-migrations` — the Prisma schema/migration workflow (never raw SQL).
  - `update-openapi` — keeps `frontend/apidocs/*.yaml` and the generated TS client in sync with API changes.
  - `reset-environment` — shared clean-rebuild procedure used by both orchestrators: rebuild, **wait for `db_migrations` to finish** (one-shot container; a failed migration holds instead of exiting), then health-check the index for a 200.

### UI verification is Playwright-first (1.2.0)

Both orchestrators verify UI through a **CLI Playwright run**, not click-by-click browser
automation. `fieldflo-test-verifier` writes a *temporary* spec plus a throwaway
`playwright.tmp.config.ts` **into the ticket's own work folder**
(`agents/context/{ROOT_ID}/{TICKET_ID}/`), so `tests/aqa/` is never touched at all.

- **Temporary specs are allowed; changing the committed AQA suite is not.** Nothing existing
  under `tests/aqa/` is created, edited, deleted, or moved, and a temp spec never gets a
  `qase(N)` id, a `@REGRESSION`/`@SMOKE`/`@SANITY` tag, or a Qase PATCH.
- **Retention:** specs in the context folder **stay** as part of the ticket record. A temp spec
  written anywhere else is **deleted** when the run finishes; generated artifacts always are.
- **Always headless (1.3.0).** The committed AQA config sets `headless: CONFIG.CI`, which is
  `false` locally. The temp config therefore overrides it with `use: { ...base.use, headless: true }`,
  and `--headed` is never used. A failing spec is debugged from its retained trace, video and
  screenshot, or through the browser MCP.
- Chrome DevTools MCP is reserved for debugging a failing spec and for selector discovery, and
  even there the agent is told to batch (`take_snapshot` → `fill_form` → `evaluate_script`, or
  `browser_batch`).

`Portal/CLAUDE.md`'s standing "don't touch AQA/Playwright" rule carries a matching exception, so
the repo rule and the plugin no longer contradict each other.

**Requires** the AQA project's dependencies, which are gitignored and often absent on a fresh
machine — one-time, and Hard Gate #0 now checks for it whenever the ticket has a UI surface:

```bash
cd tests/aqa && npm install && npx playwright install chromium
```

## What's NOT included

`technical-analysis` is referenced by `develop-fieldflo` (Step 3, optional path)
but is not bundled in this plugin — if it's not installed, Hard Gate #0 will
flag it and ask before continuing; it doesn't block the rest of the workflow.

## Installing (teammate side)

```text
/plugin marketplace add <git-url-or-path-to-this-repo>
/plugin install fieldflo-dev-workflow@fieldflo-dev-workflow-marketplace
```

`<git-url-or-path-to-this-repo>` is wherever you push this folder (private GitHub/GitLab/Bitbucket repo, or a local path for testing). This repo currently only exists locally — push it to your corporate git host first, then share that URL.

## Known gap — verify after first install

Anthropic's plugin docs confirm that plugin-bundled subagents get a **scoped
name** (`<plugin-name>:<agent-name>`), but don't explicitly document how a
skill's own prose instructions resolve a *bare*-name subagent/skill reference
once that skill is running from inside a plugin. To be safe, every bundled
skill here has a "Plugin scoping note" telling the model to use the scoped
form (`fieldflo-dev-workflow:fieldflo-planner`, `fieldflo-dev-workflow:reset-environment`,
etc.) when a bare name doesn't resolve. **Do one smoke-test run** (e.g.
`/fieldflo-dev-workflow:develop-fieldflo ticket_id=<a test ticket>`) after
installing to confirm subagent delegation actually resolves — if it errors on
an unknown subagent type, that confirms the scoped form is required exactly as
these notes assume.

## Duplicate-definition warning

If you (or a teammate) still have the loose, non-plugin copies of these skills
under `~/.claude/skills/*` and agents under `~/.claude/agents/*`, installing
this plugin creates a **second, plugin-scoped copy** alongside them. Claude
Code doesn't merge the two — you'll have both a bare `develop-fieldflo` and a
`fieldflo-dev-workflow:develop-fieldflo`, which can be confusing about which
one is current. Consider removing the loose copies once the plugin is
installed and verified, so there's one source of truth.
