# FieldFlo Dev Workflow — Claude Code plugin

Internal Claude Code plugin bundling the FieldFlo Jira-ticket dev workflow so a
teammate can get it all with one install instead of copying loose files.

## What's included

- **Skills** (`plugins/fieldflo-dev-workflow/skills/`)
  - `develop-fieldflo` — the ticket orchestrator (context → plan → approval → implement → test → review). Now starts every run with a **Hard Gate #0 dependency check** — before Step 0 it verifies the required subagents, skills, and MCP-backed capabilities are actually available in the session, and if anything's missing it stops and asks `Do you want to continue, work will be limited? (yes / no)` before proceeding.
  - `pr-review` — orchestrated PR review from a Jira ticket, triaged by size × risk.
  - `prisma-migrations` — the Prisma schema/migration workflow (never raw SQL).
  - `update-openapi` — keeps `frontend/apidocs/*.yaml` and the generated TS client in sync with API changes.
  - `reset-environment` — shared clean-rebuild + health-check procedure used by both orchestrators.
- **Subagents** (`plugins/fieldflo-dev-workflow/agents/`) — all 8: `fieldflo-context-analyst`, `fieldflo-planner`, `fieldflo-architect-reviewer`, `fieldflo-implementer`, `fieldflo-mechanical-editor`, `fieldflo-test-verifier`, `fieldflo-pr-comment-composer`, `fieldflo-pr-functional-reviewer`.

## What's NOT included (install manually if you want them)

`fieldflo-rock-builder`, `last-worked-on`, `omc-reference`, `payroll-*`,
`pr-review-swarm` (deprecated, absorbed by `pr-review`), `report-builder-dev`,
`technical-analysis`, `what-i-did` — left out on purpose per the source setup.
`technical-analysis` is still *referenced* by `develop-fieldflo` (Step 3, optional
path) — if it's not installed, Hard Gate #0 will flag it and ask before continuing;
it doesn't block the rest of the workflow.

Note: `update-openapi` is also already checked into `portal.fieldflo.com` itself
at `.claude/skills/update-openapi/SKILL.md`, so anyone with that repo cloned
already has it via git. It's bundled here too for teammates working from `hq.fieldflo.com`
/ `FFClientDocs`, or who want it available outside that one repo checkout.

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
