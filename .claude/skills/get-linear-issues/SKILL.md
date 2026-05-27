---
name: get-linear-issues
description: Fetch and list all Linear issues for a given team and project. Use when the user wants to see, list, pull, or fetch Linear issues/tickets for a project, asks "what issues are in <project>", or runs this skill with optional team/project arguments. Defaults to team "Claude-Linear-Github" and project "Linear-to-github" when none are given.
---

# Get Linear Issues

List every issue belonging to a Linear team + project.

## Requirements

Needs the Linear MCP server (`mcp__linear-server__*`). If those tools are unavailable, stop and tell the user to run `claude mcp add linear --transport sse https://mcp.linear.app/sse` and authenticate via `/mcp`.

## Inputs

The user may pass team and/or project as arguments, in any of these forms:

- `team=<name|id> project=<name|id>`
- two bare values (`<team> <project>`)
- plain prose ("issues for the Magnus team in the Onboarding project")

Resolve them as follows:

- **Team** — if not given, default to `Claude-Linear-Github`.
- **Project** — if not given, default to `Linear-to-github`.

If the user gave only one value and it's ambiguous which it is, ask which it refers to before continuing.

## Workflow

1. **Resolve team.** Call `mcp__linear-server__list_teams` and match the given (or default) name/ID. If no match, stop and tell the user the team wasn't found, and list the available team names.
2. **Resolve project.** Call `mcp__linear-server__list_projects` scoped to that team (`{ team: <teamId> }`) and match the given (or default) name/ID. If no match, stop and report it, listing the project names available on that team.
3. **Fetch issues.** Call `mcp__linear-server__list_issues` with `{ team: <teamId>, project: <projectId>, limit: 250 }`. If the result has `hasNextPage`/a cursor, page through with `cursor` until all issues are collected. Do not silently truncate.
4. **Present** the results (see below).

## Output

Print a short header naming the resolved team and project, the total count, then a table of issues sorted by identifier:

| ID | Title | Status | Assignee | Labels |
|----|-------|--------|----------|--------|

- Use the issue identifier (e.g. `CLA-3`), not the UUID.
- Show the workflow state name in Status.
- Show "—" for an unassigned issue.
- After the table, list any blocked-by relationships if present, and end with the project URL.

If there are zero issues, say so plainly instead of printing an empty table.

## Notes

- This skill is read-only — never create, update, or delete issues here.
- `list_issues` accepts `team` and `project` as either a name, ID, or slug, so resolving to IDs first (steps 1–2) is the reliable path and lets you fail loudly on a bad name.
- Defaults exist so the skill runs with no arguments; an explicit argument always overrides the default.
