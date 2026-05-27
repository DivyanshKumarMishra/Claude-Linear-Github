---
name: implement-linear-issue
description: Fetch a Linear issue by its identifier and implement it end to end in this repo — plan, branch, code, and report progress back to Linear. Use when the user gives a Linear issue ID (e.g. CLA-3) and wants it built/implemented, says "implement <issue>", "start on <issue>", "work on ticket <id>", or "pick up <id>".
---

# Implement Linear Issue

Take a Linear issue identifier, build it in this repo, and keep the issue updated as you go.

## Requirements

Needs the Linear MCP server (`mcp__linear-server__*`). If those tools are unavailable, stop and tell the user to run `claude mcp add linear --transport sse https://mcp.linear.app/sse` and authenticate via `/mcp`.

## Input

A single issue identifier, e.g. `CLA-3`. If none is given, run the `get-linear-issues` skill to list the project's issues, present them, and ask the user which one to implement. Do not guess.

## Workflow

1. **Select the issue.** If the user gave an identifier, use it. Otherwise invoke the `get-linear-issues` skill to fetch the issue list, show it, and have the user pick one (prefer issues labeled `ready-for-agent` / in Todo). Then fetch the full issue with `mcp__linear-server__get_issue` using the identifier to get the full title, description, labels, and `gitBranchName`. Read the entire description — these issues reference a source PRD (`PRD.md`); read that file too if it exists, plus any files the description names.

2. **Plan.** Draft a concrete implementation plan from the issue + PRD: which files to create/edit, the routes/contracts to honor, and the acceptance criteria you'll satisfy. Surface the plan to the user via EnterPlanMode/ExitPlanMode if plan mode fits, otherwise just lay it out.

3. **Grill on ambiguity only.** If the issue leaves genuine decisions open (naming, contract details, edge cases the PRD doesn't pin down), invoke `/grill-me` (the `grill-me` skill) to resolve them with the user before writing code. If the issue is fully specified, skip grilling and say so — don't manufacture questions.

4. **Branch.** Ask the user for the branch name to work on, offering the issue's `gitBranchName` as the default. Create/checkout that branch. (Note: this repo may not be initialized as git — if so, ask the user whether to `git init` or implement on the working tree as-is.)

5. **Mark In Progress.** Move the issue to the "In Progress" state: resolve the state id via `mcp__linear-server__list_issue_statuses` (`{ team: <teamId> }`) and call `mcp__linear-server__save_issue` with `{ id, state: <inProgressId> }`. Use TaskCreate/TaskUpdate to track the implementation steps locally.

6. **Implement.** Build the slice. Match the surrounding code's conventions. Honor every contract the issue specifies (routes, DTOs, status codes, guards, invariants). Run the project's tests/typecheck/lint where they exist and fix what you broke.

7. **Mark Done + comment.** When the work is complete and verified, move the issue to "Done" via `save_issue`, then post an implementation summary with `mcp__linear-server__save_comment` (`{ issueId, body }`): what was built, files touched, the branch name, and how it was verified. If work is incomplete, leave it In Progress and comment on the current state instead.

## Notes

- Only write to Linear at the two checkpoints above (status + summary comment). Don't spam the issue with intermediate comments.
- Never invent an admin route or weaken a stated security invariant to make a task pass — flag the conflict instead.
- One issue per run. If the user names several, confirm an order and do them one at a time.
