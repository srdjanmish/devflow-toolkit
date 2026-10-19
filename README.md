# DevFlow Toolkit — Claude Code plugin

Claude Code plugin for Jira-driven development, PR review, and verification.
Install the workflow skills and their supporting agents together.

## What's included

- **Skills** (`plugins/devflow-toolkit/skills/`)
  - `develop-ticket` — the ticket orchestrator (context → plan → approval → implement → test → review). Now starts every run with a **Hard Gate #0 dependency check** — before Step 0 it verifies the required subagents, skills, and MCP-backed capabilities are actually available in the session, and if anything's missing it stops and asks `Do you want to continue, work will be limited? (yes / no)` before proceeding.
  - `pr-review` — orchestrated PR review from a Jira ticket, triaged by size × risk. On defect tickets it closes with a gated Jira classification step: it proposes `Component`, `Found By`, `Root Cause Category` and `Escape Point`, and writes them only on `set`.
  - `prisma-migrations` — the Prisma schema/migration workflow (never raw SQL).
  - `update-openapi` — keeps `frontend/apidocs/*.yaml` and the generated TS client in sync with API changes.
  - `reset-environment` — shared clean-rebuild procedure used by both orchestrators: rebuild, **wait for `db_migrations` to finish** (one-shot container; a failed migration holds instead of exiting), then health-check the index for a 200.

### UI verification is Playwright-first (1.2.0)

Both orchestrators verify UI through a **CLI Playwright run**, not click-by-click browser
automation. `devflow-test-verifier` writes a *temporary* spec plus a throwaway
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

If the target repository restricts AQA/Playwright changes, confirm its rules allow
temporary verification specs outside the committed test suite before running them.

**Requires** the AQA project's dependencies, which are gitignored and often absent on a fresh
machine — one-time, and Hard Gate #0 now checks for it whenever the ticket has a UI surface:

```bash
cd tests/aqa && npm install && npx playwright install chromium
```

## What's NOT included

`technical-analysis` is referenced by `develop-ticket` (Step 3, optional path)
but is not bundled in this plugin — if it's not installed, Hard Gate #0 will
flag it and ask before continuing; it doesn't block the rest of the workflow.

## Installing (teammate side)

```text
/plugin marketplace add srdjanmish/devflow-toolkit
/plugin install devflow-toolkit@devflow-toolkit-marketplace
```

### Project setup

Read the target repository's `CLAUDE.md` / `AGENTS.md` before using a workflow.
Resolve its GitHub owner/repository from the selected PR or Git remote, and the
current Jira account from the authenticated connector. Never assume the plugin's
own GitHub repository is the application repository.

Configure the local health URL, development/base branch, Compose application and
migration services, Prisma schema paths, and Playwright project in the target
repository's instructions. The bundled PHP/CodeIgniter, Prisma master/client/HQ,
Docker and `tests/aqa` procedures are reference layouts: confirm they match the
project before running commands; adapt to documented equivalents or report a
missing dependency. Environment resets still require the skill's explicit opt-in.

### Upgrading from 1.x

Version 2.0.0 renames the plugin to `devflow-toolkit`, the ticket command to
`develop-ticket`, and agent prefixes to `devflow-`. Update existing invocations
and install the new marketplace/plugin names shown above. Old scoped names are
not aliases. Earlier commits and release tags retain their original content.

## Contributing

Open pull requests against `main` at
https://github.com/srdjanmish/devflow-toolkit. Include the workflow change and how
you verified it. Do not include credentials, internal URLs, or real ticket data.

## Known gap — verify after first install

Anthropic's plugin docs confirm that plugin-bundled subagents get a **scoped
name** (`<plugin-name>:<agent-name>`), but don't explicitly document how a
skill's own prose instructions resolve a *bare*-name subagent/skill reference
once that skill is running from inside a plugin. To be safe, every bundled
skill here has a "Plugin scoping note" telling the model to use the scoped
form (`devflow-toolkit:devflow-planner`, `devflow-toolkit:reset-environment`,
etc.) when a bare name doesn't resolve. **Do one smoke-test run** (e.g.
`/devflow-toolkit:develop-ticket ticket_id=<a test ticket>`) after
installing to confirm subagent delegation actually resolves — if it errors on
an unknown subagent type, that confirms the scoped form is required exactly as
these notes assume.

## Duplicate-definition warning

If you (or a teammate) still have the loose, non-plugin copies of these skills
under `~/.claude/skills/*` and agents under `~/.claude/agents/*`, installing
this plugin creates a **second, plugin-scoped copy** alongside them. Claude
Code doesn't merge the two — you'll have both a bare `develop-ticket` and a
`devflow-toolkit:develop-ticket`, which can be confusing about which
one is current. Consider removing the loose copies once the plugin is
installed and verified, so there's one source of truth.
