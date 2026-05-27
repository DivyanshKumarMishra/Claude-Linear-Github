# Issue tracker: Linear

Issues and PRDs for this repo live as Linear issues. Use the Linear MCP tools (`mcp__linear__*`) for all operations.

Before any operation, confirm the Linear MCP server is connected. If `mcp__linear__*` tools are unavailable, stop and tell the user to add the Linear MCP server (`claude mcp add linear --transport sse https://mcp.linear.app/sse`) and authenticate via `/mcp`.

## Fixed routing for this repo

- **Team**: `Claude-Linear-Github` — resolve its ID at runtime via `mcp__linear__list_teams` (match by name). Do not hardcode the ID.
- **Project**: `Linear-to-github` — resolve its ID via `mcp__linear__list_projects` scoped to that team (match by name).

## Conventions

- **Create an issue**: `mcp__linear__create_issue` with `{ teamId, projectId, title, description, labelIds }`. The description is markdown.
- **Read an issue**: `mcp__linear__get_issue` with the issue id/identifier, plus `mcp__linear__list_comments` for its discussion.
- **List issues**: `mcp__linear__list_issues` with `{ teamId, projectId, state }` filters.
- **Comment on an issue**: `mcp__linear__create_comment` with `{ issueId, body }`.
- **Apply / remove labels**: `mcp__linear__update_issue` with the full desired `labelIds` array (see label note below).
- **Close an issue**: `mcp__linear__update_issue` setting `stateId` to a Done-type state. Find available states via `mcp__linear__list_issue_statuses` for the team.

Tool names follow the official Linear MCP server. Verify exact parameter names with one `list_*` call if a write fails — servers occasionally rename fields.

## Labels

**`mcp__linear__update_issue` replaces the entire label set** — you must pass the complete desired `labelIds` array every time, not just the one you want to add. To add a label, read the issue's current labels first, append the new id, and send the union. To remove one, send the array minus that id.

Resolve label names → ids via `mcp__linear__list_issue_labels` (scoped to the team). Cache them for the session.

This repo uses two independent label dimensions — see `triage-labels.md` for the full mapping:

1. **Triage labels** (workflow state): `ready-for-agent`, `needs-triage`, `needs-info`, `ready-for-human`, `wontfix`.
2. **Type labels** (nature of work): `Feature`, `Improvement`, `Bug`.

An issue normally carries one of each dimension (e.g. `ready-for-agent` + `Feature`).

## When a skill says "publish to the issue tracker"

Create a Linear issue under team `Claude-Linear-Github`, project `Linear-to-github`, with the appropriate triage label (`ready-for-agent` for PRD-derived, AFK-ready work) and the appropriate type label (`Feature` / `Improvement` / `Bug`).

## When a skill says "fetch the relevant ticket"

Call `mcp__linear__get_issue` with the issue id/identifier, then `mcp__linear__list_comments` for its discussion.

## When a skill says "apply the AFK-ready triage label"

Add the `ready-for-agent` label id to the issue's existing `labelIds` (union, not replace-with-just-this) and call `mcp__linear__update_issue`.
