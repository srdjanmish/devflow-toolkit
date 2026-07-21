# Changelog

All notable changes to the `fieldflo-dev-workflow` plugin are documented in this file.

## [1.0.1] - 2026-07-22

### Changed
- `fieldflo-test-verifier`: added `mcp__chrome-devtools` and `mcp__claude-in-chrome` tool access, so the agent can drive its own browser-based UI verification steps instead of relying on the orchestrator for them.

## [1.0.0] - 2026-07-20

### Added
- Initial release of the plugin.
- `develop-fieldflo` orchestrator skill (with a startup dependency-check gate), plus `pr-review`, `prisma-migrations`, `update-openapi`, and `reset-environment` skills (dependencies for work).
- `fieldflo-*` subagent team: `fieldflo-architect-reviewer`, `fieldflo-context-analyst`, `fieldflo-implementer`, `fieldflo-mechanical-editor`, `fieldflo-planner`, `fieldflo-pr-comment-composer`, `fieldflo-pr-functional-reviewer`, `fieldflo-test-verifier`.
