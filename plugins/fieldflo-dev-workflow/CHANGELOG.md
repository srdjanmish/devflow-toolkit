# Changelog

All notable changes to the `fieldflo-dev-workflow` plugin are documented in this file.

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
