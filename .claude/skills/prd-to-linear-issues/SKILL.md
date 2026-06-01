---
name: prd-to-linear-issues
description: Convert a PRD file into Linear issues using tracer-bullet vertical slices. Use when the user provides a PRD file path and wants it broken into Linear tickets, or says "turn this PRD into Linear issues", "publish PRD to Linear", "create Linear tickets from PRD".
---

# PRD to Linear Issues

Take a PRD file and publish it as a set of independently-grabbable Linear issues, sliced as tracer bullets.

This skill requires the Linear MCP server. If `mcp__linear__*` tools are not available, stop and tell the user to run `claude mcp add linear --transport sse https://mcp.linear.app/sse` and authenticate via `/mcp`.

## Process

### 1. Read the PRD

The user passes a path to a PRD file as the skill argument. Read it in full. If no path was given, ask for one. Do not infer a PRD from conversation context — this skill is file-driven.

### 2. Resolve Linear context

Before drafting issues, figure out where they're going:

- **Team is hardcoded to `Claude-Linear`.** Call `mcp__linear__list_teams`, find the team whose name matches `Claude-Linear`, and use its ID. Do NOT ask the user. If no team with that name exists, stop and tell the user the team is missing.
- **Project is hardcoded to `Claude-to-linear-issues`.** Call `mcp__linear__list_projects` (scoped to that team), find the project whose name matches `Claude-to-linear-issues`, and use its ID. Do NOT ask the user. If no project with that name exists, stop and tell the user the project is missing.
- Call `mcp__linear__list_issue_labels` for the team so you can suggest relevant labels.

Cache the team ID, project ID, and label IDs for the publish step.

### 3. Explore the codebase (optional)

If the PRD references a specific area of the code and you have not already read it, skim it. Issue titles and descriptions should use the project's domain vocabulary, not generic phrasing copied from the PRD.

### 4. Draft vertical slices

Break the PRD into **tracer bullet** issues. Each issue is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

Slices may be 'HITL' or 'AFK'. HITL slices require human interaction (architectural decision, design review). AFK slices can be implemented and merged without human interaction. Prefer AFK over HITL where possible.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
</vertical-slice-rules>

### 5. Quiz the user

Present the proposed breakdown as a numbered list. For each slice show:

- **Title**: short descriptive name
- **Type**: HITL / AFK
- **Blocked by**: which other slices (if any) must complete first
- **PRD sections covered**: which parts of the PRD this addresses

Also surface the Linear destination: chosen team, project, and proposed labels.

Ask the user:

- Does the granularity feel right? (too coarse / too fine)
- Are the dependency relationships correct?
- Should any slices be merged or split further?
- Are the correct slices marked as HITL vs AFK?
- Is the team / project / label set correct?

Iterate until the user explicitly approves. Do NOT create any Linear issues before approval.

### 6. Publish to Linear

For each approved slice, call `mcp__linear__create_issue` with the team ID, project ID, labels, title, and description (using the template below).

Publish in **dependency order** (blockers first) so you can reference real Linear issue identifiers (e.g. `ENG-123`) in the "Blocked by" field of dependent slices.

After publishing each issue, capture its identifier and URL. At the end, report back to the user a list of `IDENTIFIER — Title — URL`.

<issue-template>
## Source PRD

`<relative/path/to/prd.md>`

## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

Avoid specific file paths or code snippets — they go stale fast. Exception: if the PRD encoded a decision more precisely than prose can (state machine, schema, type shape), inline only the decision-rich parts.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Blocked by

- `ENG-XXX` — title of blocking issue

Or "None — can start immediately" if no blockers.
</issue-template>

Do NOT modify the PRD file itself, and do not create any parent/epic issue unless the user asks for one.
