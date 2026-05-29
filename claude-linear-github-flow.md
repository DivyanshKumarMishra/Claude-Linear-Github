# Claude → Linear → GitHub Flow

End-to-end workflow for turning a product requirements document (PRD) into shipped code, with Linear tracking and GitHub PRs kept in sync automatically. Share this with the team to align on how work moves from idea to merge.

## Why this flow

- **Single source of truth per step.** PRD → Linear issues → branches → PRs → merge. Each artifact has one owner; no parallel "Linear status" updates done by hand alongside the PR.
- **Reviewable agent work.** Claude implements one Linear issue at a time, reports back, and waits for human approval at three explicit gates (commit, PR open, merge). Nothing ships without a person saying so.
- **Auto-closure without polling.** Linear's GitHub integration closes the linked issue when the PR merges. No standup ritual to update statuses.

## End-to-end at a glance

```
PRD file                    skill: /prd-to-linear-issues
   │
   ▼
Linear issues               (vertical-slice tracer bullets, one per shippable change)
   │
   ▼ (user picks an issue)
Branch                      skill: /implement-linear-issue <ID>
Implementation              (Claude codes; user reviews on working tree)
   │
   ▼ (user says "ship it")
Commit
   │
   ▼ (user says "open PR")
GitHub PR                   (body ends with `Closes <ID>` → linked to Linear)
   │
   ▼ (review, then merge into main)
Linear issue auto-closes    (via Linear's GitHub integration)
```

## Skill 1 — `/prd-to-linear-issues <path/to/PRD.md>`

Turns a PRD file into a set of independently-grabbable Linear issues sliced as **tracer bullets** — thin vertical slices that cut through every layer (schema → API → UI → tests), not horizontal layer-only chunks.

**Brief steps:**

1. Read the PRD file in full.
2. Resolve Linear destination (team + project; both hardcoded per project so the agent doesn't ask).
3. Optionally skim the codebase to use domain vocabulary in titles.
4. Draft vertical slices — each is demoable on its own, with explicit "blocked by" dependencies. Marked **AFK** (agent can finish without human) or **HITL** (needs human decision).
5. Present the breakdown to the user for approval (granularity, dependencies, AFK/HITL classification).
6. On approval, publish each slice as a Linear issue in dependency order, capturing identifiers so dependent issues reference the real `Closes` blockers.

**Output:** a list of `IDENTIFIER — Title — URL` for every issue created.

## Skill 2 — `/implement-linear-issue <ID>`

Picks up one Linear issue and builds it end-to-end in the repo. Keeps Linear in sync at exactly the right moments — and **never** at the wrong ones.

**Brief steps:**

1. **Fetch the issue** (title, description, labels, `gitBranchName`) + read the source PRD.
2. **Branch** — before any planning or coding:
   - Ask the user which **base branch** to branch off (default `main`, always confirm).
   - `git checkout <base>` + `git pull` so the base is fresh from remote.
   - Ask the user for the new branch name. Convention is `<type>/<descriptive-slug>` where `<type>` is `feature` / `improvement` / `bug` / `refactor` / `chore`. Branch names are human-meaningful — they do **not** embed the Linear issue ID. The user can override with no prefix at all.
   - `git checkout -b <newBranch>`.
3. **Plan** the implementation from the issue + PRD; surface to the user.
4. **Grill on ambiguity only** — if the issue leaves real decisions open, ask. Otherwise skip.
5. **Mark issue "In Progress"** in Linear.
6. **Implement.** Match repo conventions. Run tests/typecheck/lint. **Do not commit** — leave changes on the working tree so review can iterate without piling up throwaway commits.
7. **Report back on Linear** with an implementation summary comment (what was built, files touched, deviations, how it was verified). Leave the issue in "In Progress" — **never mark Done from Claude**.
8. **Iterate on user feedback** on the working tree. No commits during iteration. Update the Linear comment only if material changes happen.
9. **Commit on explicit approval** (user says "ship it" / "commit this"). One clean commit per issue.
10. **Open PR on explicit approval** (user says "open PR" / "raise PR"). The PR body is pulled from the latest Linear implementation comment + a Linear issue link + `Closes <ID>`. Before opening, verifies the base branch exists on the remote and is GitHub's default branch (so close-on-merge actually fires).

## Anatomy of a Linear issue

Every issue created by `/prd-to-linear-issues` follows the same shape so it's pickable by any engineer or agent without re-reading the PRD.

### Issue body template

```markdown
## Source PRD

`relative/path/to/PRD.md`

## What to build

A concise description of this vertical slice — end-to-end behavior, not
layer-by-layer implementation. Decision-rich snippets (state machines,
schemas, type shapes) inlined only when the PRD encoded them more
precisely than prose can.

## Acceptance criteria

- [ ] Concrete, demoable criterion 1
- [ ] Concrete, demoable criterion 2
- [ ] ...

## Blocked by

- `CL-XXX` — title of the blocking issue

Or "None — can start immediately" if no blockers.
```

The body is deliberately code-light — file paths and snippets go stale fast. The acceptance criteria are the contract; once they're all ticked, the slice is done.

### Labels

Labels are how triage and routing happen at a glance. Convention for this project:

| Label | Meaning |
|---|---|
| `ready-for-agent` | Fully specified, an AFK agent can pick it up and finish without further human input. Default for issues created by `/prd-to-linear-issues`. |
| `ready-for-human` | Decision-rich; needs a human to drive (architecture, design review, ambiguous tradeoffs). |
| `Feature` | New functionality. |
| `Improvement` | Enhancement to an existing feature. |
| `Bug` | Defect or regression. |
| `needs-triage` | Maintainer hasn't classified it yet. |
| `needs-info` | Waiting on the reporter for clarification — blocked until they reply. |
| `wontfix` | Decided not to address. |

Issues created from a PRD get `ready-for-agent` + a type label (`Feature` / `Improvement` / `Bug`) at minimum. Triage labels (`needs-triage` / `needs-info`) are added later if blockers appear.

### Lifecycle states

| State | When the issue is here |
|---|---|
| **Todo** | Created but not started. Waiting to be picked up. |
| **In Progress** | Picked up — agent or human is actively working. Set by `/implement-linear-issue` immediately after the branch is created, *before* any code is written. Also set by Linear's GitHub automation when a non-draft PR opens (no-op since we already set it). |
| **In Review** | A PR is open and a reviewer has been engaged. Set automatically by Linear when reviewer activity happens on the PR. |
| **Done** | PR merged into the default branch. Set automatically by Linear's GitHub integration via the PR body's `Closes <ID>` magic verb. **Never set manually from Claude.** |
| **Canceled** | Work abandoned without shipping. Set manually. |

### The implementation summary comment

After Claude finishes implementing (step 7 of `/implement-linear-issue`), it posts a single structured comment on the issue. This comment is the **single source of truth** for what was built — and later becomes the PR body verbatim. Shape:

```markdown
## Implemented

Branch: `feature/<slug>`
Commit: `<sha>` ("<commit subject>")    # filled in once the commit lands

### What's built

- Bullet per file or module touched, describing the contract delivered
  (not line-level changes). Example:
  "`src/config/env.schema.ts` — Zod schema validating ..."

### Deviations

- Anything that differs from the PRD or the issue spec, with a one-line
  justification. Example: "Prisma pinned to v6 because v7 moved
  datasource.url out of schema.prisma."
- Empty section if there were no deviations.

### Verified

- How the change was tested (typecheck, manual boot, specific test
  cases). One bullet per verification done.
```

**Why the structure matters:**

- Reviewer reads one comment, not a transcript.
- The same comment becomes the PR body, so PR and Linear stay in lockstep without anyone writing the summary twice.
- "Deviations" forces explicit flagging of any drift from the PRD, instead of letting it hide in the diff.
- "Verified" makes the acceptance criteria evidence-of-completion, not an honor system.

If iteration during review materially changes what was built, the comment is **updated in place** (not appended) so it always reflects the current state.

## Lifecycle discipline (locked-in rules)

These are non-negotiable in the skill — they enforce a clean review experience and prevent races with Linear's GitHub integration:

| Rule | Why |
|---|---|
| **Never auto-commit.** Only commit when the user explicitly says so. | Back-and-forth review must not pile up throwaway commits. One clean commit per issue. |
| **Never auto-open a PR.** Commit and PR are separate user commands. | Sometimes the user wants to commit but hold the PR for a related change. |
| **Never move a Linear issue to Done from Claude.** | Linear's GitHub integration auto-closes the issue when the linked PR merges. Closing from Claude too would race with the integration and could close prematurely (before review). |
| **One PR, one Linear issue.** Body has exactly one `Closes <ID>` line. | A change spanning multiple issues is a scope smell — split the PR or merge the issues first. |
| **Branch names and PR titles are human-meaningful (no issue ID embedded).** | Readable history; the PR body's `Closes <ID>` is the sole link to Linear. |

## Linear ↔ GitHub state machine

Configured per team in **Linear → Settings → Teams → \<team\> → Workflows & automations**. For the `Claude-Linear` team:

| GitHub event | Linear state transition |
|---|---|
| Draft PR opened | No action |
| **PR opened (non-draft)** | **→ In Progress** |
| Reviewer requested or activity | **→ In Review** |
| PR ready for merge | No action |
| **PR merged into default branch** | **→ Done** (closes the issue) |
| **PR closed without merging** | *Not automated* — manual flip back to In Progress |

Multi-trunk repos (long-lived `dev`/`staging` branches) need explicit per-branch rules added in the same panel — see `linear-github-integration.md`.

## What the manager sees on a given day

For any in-flight issue, the Linear state at a glance:

- **Todo** — waiting to be picked up.
- **In Progress** — someone (likely an agent + reviewer) is actively working; no PR yet, or PR was opened.
- **In Review** — reviewer engaged on a PR.
- **Done** — PR merged into the trunk branch; work shipped.

No one needs to manually update Linear — the state always reflects what's actually happening on GitHub.

## Related docs

- `linear-github-integration.md` — one-time setup of Linear's GitHub app, per-team automations, and multi-trunk configuration.
- `.claude/skills/prd-to-linear-issues/SKILL.md` — full skill definition.
- `.claude/skills/implement-linear-issue/SKILL.md` — full skill definition.
- `PRD.md` — the source PRD for this project.
