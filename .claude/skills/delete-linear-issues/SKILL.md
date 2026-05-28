---
name: delete-linear-issues
description: "Delete" Linear issues by moving them to the Canceled state — one issue by ID/name, or every issue in a team+project when none is given. Use when the user wants to delete, cancel, remove, clear out, or wipe Linear issues/tickets, e.g. "delete CLA-3", "cancel all issues in <project>", "clear the Linear board".
---

# Delete Linear Issues

Cancel Linear issues — a single one by identifier/name, or all issues in the project when none is named.

## Important: there is no hard delete

The Linear MCP server has no delete/archive tool. This skill "deletes" by moving issues to the **Canceled** state via `mcp__linear-server__save_issue`. This is reversible and preserves history. Say this plainly to the user — do not imply the issues are gone forever.

## Requirements

Needs the Linear MCP server (`mcp__linear-server__*`). If unavailable, stop and tell the user to run `claude mcp add linear --transport sse https://mcp.linear.app/sse` and authenticate via `/mcp`.

## Inputs

- **Target** (optional) — an issue identifier (`CLA-3`) or a title/name. If given, only that issue is cancelled. If omitted, ALL issues in scope are cancelled.
- **Team** — defaults to `Claude-Linear-Github`. Arg overrides.
- **Project** — defaults to `Linear-to-github`. Arg overrides.

## Workflow

1. **Resolve team + project.** `mcp__linear-server__list_teams` → match team name/ID. `mcp__linear-server__list_projects` (`{ team }`) → match project. If either is missing, stop and report it, listing what's available.

2. **Resolve the Canceled state.** Call `mcp__linear-server__list_issue_statuses` (`{ team }`) and grab the id of the state whose `type` is `canceled`. If none exists, stop and tell the user.

3. **Collect the target issues.**
   - **Target given:** if it looks like an identifier, fetch via `mcp__linear-server__get_issue`. Otherwise `mcp__linear-server__list_issues` (`{ team, project }`) and match by title (case-insensitive). If the name matches more than one issue, list the matches and ask which one — do not cancel all of them.
   - **No target:** `mcp__linear-server__list_issues` (`{ team, project, limit: 250 }`), paging through any cursor. Skip issues already in a `canceled` state.

4. **Confirm once.** Print the exact list of issues that will be cancelled (`ID — Title — current status`) and the count. Require a single explicit yes before proceeding. If the user runs the no-target form, make the blast radius unmistakable (e.g. "This will cancel ALL N issues in <project>."). This is the ONLY confirmation — never re-ask per issue.

5. **Cancel all at once.** After the single yes, cancel every confirmed issue in one batch: issue all the `mcp__linear-server__save_issue` calls (`{ id, state: <canceledStateId> }`) together in a single turn (parallel tool calls), not one-at-a-time with a pause between each. Do not prompt or check in again mid-batch.

6. **Report.** List each cancelled issue as `ID — Title — ✅ Canceled`, and note anything skipped (already canceled) or failed.

## Notes

- This is the only write this skill performs — it never edits titles, descriptions, labels, or comments.
- Issues already Canceled are left as-is and reported as skipped.
- One name resolving to multiple issues is always a clarify-first situation, never a bulk cancel.
