---
name: implement-linear-issue
description: Fetch a Linear issue by its identifier and implement it end to end in this repo — plan, branch, code, and report progress back to Linear. Use when the user gives a Linear issue ID (e.g. CLA-3) and wants it built/implemented, says "implement <issue>", "start on <issue>", "work on ticket <id>", or "pick up <id>".
---

# Implement Linear Issue

Take a Linear issue identifier, build it in this repo, and keep the issue updated as you go.

## Requirements

Needs the Linear MCP server (`mcp__linear-server__*`). If those tools are unavailable, stop and tell the user to run `claude mcp add linear --transport sse https://mcp.linear.app/sse` and authenticate via `/mcp`.

## Prerequisites

**Linear's GitHub app must be installed on the target repo** before step 10 (open PR) will close the Linear issue on merge. Without it, the `Closes <ISSUE-ID>` line in the PR body is just text — no linkage, no auto-close.

How to check / install:

1. **In Linear:** Settings → Integrations → GitHub. If the target repo's org appears as "Connected" (and the repo itself is in the selected-repos list), you're set. Otherwise click **Connect**.
2. **GitHub OAuth:** sign in, choose the org that owns the repo, pick "Only select repositories", tick the target repo, **Install**.
3. **Confirm in Linear:** the integration page should now list the connected GitHub org and show the repo as accessible. No test PR needed — the first real PR from step 10 will exercise the `Closes` magic verb.

If step 10 is reached and the app is not installed, stop and walk the user through this setup before pushing the PR — don't open a PR that won't auto-close the issue, because that defeats the entire reason step 7 forbids marking the issue Done from Claude.

We rely **only** on the default magic-verb auto-close (`Closes`/`Fixes`/`Resolves` on merge). We do not require any configurable Linear workflow automations (e.g. "PR opened → In Review") — those are optional sugar.

## Input

A single issue identifier, e.g. `CLA-3`. If none is given, run the `get-linear-issues` skill to list the project's issues, present them, and ask the user which one to implement. Do not guess.

## Workflow

1. **Select the issue.** If the user gave an identifier, use it. Otherwise invoke the `get-linear-issues` skill to fetch the issue list, show it, and have the user pick one (prefer issues labeled `ready-for-agent` / in Todo). Then fetch the full issue with `mcp__linear-server__get_issue` using the identifier to get the full title, description, labels, and `gitBranchName`. Read the entire description — these issues reference a source PRD (`PRD.md`); read that file too if it exists, plus any files the description names.

2. **Branch — always do this BEFORE planning, grilling, or any implementation work.** Set up the working branch in this exact order, so no code/decisions ever happen on a stale or wrong base:
   1. **Ask the user which base branch to branch off** (default `main`, but always confirm — they may want to stack on top of a feature branch).
   2. `git checkout <baseBranch>` then `git pull` to make sure the base is up to date with the remote.
   3. **Ask the user for the new branch name**, offering the issue's `gitBranchName` as the default.
   4. `git checkout -b <newBranch>` off the freshly-pulled base.

   (Note: this repo may not be initialized as git — if so, ask the user whether to `git init` or implement on the working tree as-is. If there are uncommitted local changes on the current branch when this step starts, stop and ask the user what to do — don't clobber their work with a checkout.)

3. **Plan.** Draft a concrete implementation plan from the issue + PRD: which files to create/edit, the routes/contracts to honor, and the acceptance criteria you'll satisfy. Surface the plan to the user via EnterPlanMode/ExitPlanMode if plan mode fits, otherwise just lay it out.

4. **Grill on ambiguity only.** If the issue leaves genuine decisions open (naming, contract details, edge cases the PRD doesn't pin down), invoke `/grill-me` (the `grill-me` skill) to resolve them with the user before writing code. If the issue is fully specified, skip grilling and say so — don't manufacture questions.

5. **Mark In Progress.** Move the issue to the "In Progress" state: resolve the state id via `mcp__linear-server__list_issue_statuses` (`{ team: <teamId> }`) and call `mcp__linear-server__save_issue` with `{ id, state: <inProgressId> }`. Use TaskCreate/TaskUpdate to track the implementation steps locally.

6. **Implement.** Build the slice. Match the surrounding code's conventions. Honor every contract the issue specifies (routes, DTOs, status codes, guards, invariants). Run the project's tests/typecheck/lint where they exist and fix what you broke. **Do NOT commit** at this stage — leave changes on the working tree so the user can review, revise the strategy, or send you back for another pass without piling up throwaway commits.

7. **Report — don't close.** When you believe the work is complete and verified, post an implementation summary with `mcp__linear-server__save_comment` (`{ issueId, body }`): what was built, files touched, the branch name, how it was verified, and any deviations from the spec. Then **stop and surface the work to the user for review**. Leave the issue in "In Progress". **Never move the issue to "Done" yourself** — closing is always the user's call, not yours, even if they say "looks good".

8. **Iterate on feedback.** The user may request changes, debate the approach, or ask for a different strategy. Keep iterating on the working tree (no commits yet). Post a new Linear comment only if a round of changes meaningfully changes what was reported in step 7 — otherwise stay silent on Linear and just talk in chat.

9. **Commit on explicit approval.** Only when the user explicitly says to commit (e.g. "ship it", "commit this", "looks good, commit"), create the commit(s) — prefer a single squashed commit per issue unless the user asks otherwise, so the branch ends up with one clean commit per Linear issue. Still do NOT touch the issue's status — the user marks it Done themselves.

10. **Open PR on explicit approval (separate step).** Commit and PR are two separate user commands; do not auto-open a PR after committing. When the user explicitly says to open the PR (e.g. "open PR", "raise PR", "create PR"):
    1. Read the target repo from `git remote get-url origin`. If origin is missing or non-GitHub, stop and tell the user.
    2. Push the branch with `-u` if not already tracking a remote.
    3. **Ask the user for the PR title** — do not auto-generate it. Offer the Linear issue title as a default they can accept or override.
    4. Auto-generate the PR body from the implementation summary already posted to Linear plus a link to the Linear issue. End the body with a literal `Closes <ISSUE-ID>` line (e.g. `Closes CL-1`) so Linear's GitHub integration auto-closes the issue when the PR merges. **This is exactly why step 7 forbids marking the issue Done in Linear — closing must come from the merge, not from Claude.**
    5. Create the PR with `gh pr create --title "<user title>" --body "<generated body>"` and return the PR URL.

## Notes

- **Never auto-commit. Never auto-open a PR. Never mark an issue Done.** All three require explicit, separate user approval. Closing the issue is owned by Linear's GitHub integration via the PR merge — not by you.
- Only write to Linear at the checkpoints above (In Progress on start, summary comment after implementation, follow-up comments only if the work materially changes). Don't spam the issue with intermediate comments.
- Never invent an admin route or weaken a stated security invariant to make a task pass — flag the conflict instead.
- One issue per run. If the user names several, confirm an order and do them one at a time.
