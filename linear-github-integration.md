# Linear ↔ GitHub Integration — Setup & Usage

How to wire Linear's GitHub app to a repo so PRs and issues stay in sync automatically. Hand this to a teammate when onboarding a new repo or environment.

## Why this exists

When set up correctly, this integration gives you:

- **Linkage:** any PR whose body contains a magic-verb reference to a Linear issue (e.g. `Closes CL-12`) shows up in that issue's sidebar within seconds of opening, without any manual action.
- **Auto-close on merge:** merging that PR into the configured close-trigger branch moves the Linear issue to its team's completed state. No human has to touch Linear's status.

This lets us treat the GitHub PR merge as the single source of truth for "work is done" — Linear status follows from it, instead of being a parallel thing humans have to remember to update.

## Prerequisites

- **Linear:** admin on the workspace.
- **GitHub:** admin (or "Manage integrations" rights) on the org that owns the target repo.

If you're not admin on either side, get someone who is — the install requires permissions a regular member doesn't have.

## Step 1 — Install Linear's GitHub app

1. In Linear, open **Settings → Integrations → GitHub** (left sidebar under "Features").
2. If the target org isn't already listed under **Connected organizations**, click **Connect**. (If it is listed and shows the green **Connected** dot, skip to Step 2.)
3. GitHub OAuth opens in a new window. Sign in if prompted, then pick the **org that owns the target repo** (not your personal account, unless the repo is personal).
4. On the **Repository access** screen, choose either:
   - **All repositories** — Linear sees every current and future repo in the org. Simplest; recommended for small orgs.
   - **Only select repositories** — tick the specific repos you want Linear to access. Use this if you want to limit scope.
5. Click **Install** (or **Save** if updating an existing install).

You'll land back on Linear's GitHub integration page, with the org showing as **Connected**.

## Step 2 — Verify the connection

On the Linear GitHub integration page:

- The target org appears under **Connected organizations** with a green **Connected** dot.
- If you picked "Only select repositories", expand the org's row (caret on the right) and confirm the target repo is in the list. If not, go to `github.com/organizations/<org>/settings/installations` → Linear Code → **Configure** → tick the repo → Save.

**Do NOT create a throwaway PR to test this.** The first real PR raised through the normal workflow will exercise the magic verb. A throwaway PR risks someone forgetting to discard it.

## Step 3 — PR conventions (what makes the magic work)

For Linear to link and close an issue from a PR, the PR's **body** must contain a magic-verb line referencing the Linear issue ID. Branch name and PR title are NOT used as linkage signals (they're human-meaningful and don't embed issue IDs in our convention).

Accepted verbs (case-insensitive, with or without `-s` / `-d`):

- `Close <ID>` / `Closes <ID>` / `Closed <ID>`
- `Fix <ID>` / `Fixes <ID>` / `Fixed <ID>`
- `Resolve <ID>` / `Resolves <ID>` / `Resolved <ID>`

Convention for this project: use **`Closes <ID>`** for consistency. Example: a PR body ending with `Closes CL-12` will link to issue CL-12 and close it on merge.

**One PR closes exactly one Linear issue.** If your change spans multiple issues, that's a signal the scope is wrong — either split the PR per issue, or merge the Linear issues first so a single ID covers the work. Do not list multiple `Closes` IDs in one PR body.

## Step 4 — Configure PR → status automations per team

Linear closes (and otherwise transitions) issues via **per-team Pull Request automations**, not a global setting. You configure them once per team.

Path: **Linear → Settings → Teams → \<your-team\> → Workflows & automations → Pull request automations**.

### Default rules (recommended)

| PR event | Linear action | Why |
|---|---|---|
| Draft PR opened | No action | Drafts are work-in-progress; don't disturb status |
| **PR opened (non-draft)** | **→ In Progress** | Confirms work has started; redundant-but-safe if the agent already set this |
| PR review request or activity | → In Review | Signals reviewer engagement |
| PR ready for merge | No action | Optional; could go to "Ready to merge" if you have one |
| **PR merged** | **→ Done** | The trunk close-on-merge trigger |

These rules apply to any PR that's linked to a team issue (via the magic verb in the PR body — see Step 3).

### Multi-trunk repos (long-lived `dev` / `staging` branches)

By default, the rules above fire on merges into the GitHub repo's default branch. If your repo has additional long-lived branches that should ALSO close issues on merge (scenario (a) — each branch is its own destination), use **Branch-specific rules → Add branch** on the same screen:

1. Click **Add branch**.
2. Enter the branch name (`dev`, `staging`, etc.).
3. Set the per-branch event rules — usually a duplicate of the default set, with **On PR merge → Done**.
4. Save. Repeat for each additional close-trigger branch.

There's some UI ambiguity about whether the top-level rules apply to all branches or only the default branch. The safe approach is to **explicitly add a branch-specific rule for every long-lived branch you want close-on-merge from** — that way the behavior is unambiguous regardless of how Linear interprets the global rules.

> **Note:** branch-specific rules only matter if the branch actually exists in the repo and PRs target it. You don't need to pre-configure rules for branches you don't have yet — add them when the branch is created.

### Auto-close automations (separate section, on the same screen)

Below the PR automations, there's an **Auto-close automations** section with toggles for closing parent issues, sub-issues, and stale issues. These are independent of GitHub and not required for the PR ↔ issue sync — leave them at your team's preference.

### Gap — "PR closed without merge" is NOT automated

Linear's automation panel covers PR draft-open, open, review request, ready-for-merge, and merge — but **not "PR closed without merging"**. If a PR is closed without being merged (wrong approach, force-pushed reset, scope rejected), Linear does nothing. The issue stays in whatever state the last fired rule left it (usually In Review).

**Manual convention for this project:** when you close a PR without merging, also flip the linked Linear issue back to **In Progress** in the same sitting. This keeps the board honest — In Review should only ever mean "there's an open PR awaiting review."

If you reopen work later via the `implement-linear-issue` skill, its Mark-In-Progress step will set the state correctly anyway. The manual flip is only needed if you close the PR and walk away.

## Branch and PR naming convention (for context)

Branches and PR titles in this project are intentionally human-meaningful and **do not embed Linear issue IDs**. Branch format:

```
<type>/<descriptive-slug>
```

Where `<type>` is one of `feature`, `improvement`, `bug`, `refactor`, `chore`. Examples:

- `feature/auth-register`
- `bug/cookie-secure-prod`
- `refactor/extract-token-service`
- `chore/upgrade-prisma-v6`

PR titles follow the same human-readable principle — describe the change, not the issue number.

This is why the **PR body's magic-verb line is the only link** between PR and Linear issue. It is therefore non-negotiable: every PR that closes a Linear issue must end with a `Closes <ID>` line.

## What to do when something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| PR doesn't appear in Linear issue sidebar | App not installed on this repo, OR magic verb missing/typo'd | Check Linear → Settings → Integrations → GitHub; confirm repo is in the access list. Re-check the PR body for `Closes <ID>` |
| PR appears in sidebar but issue doesn't close on merge | PR merged into a non-default branch, OR magic verb wrong shape | Check which branch was merged into. If non-default and you need close-on-merge, configure additional close-trigger branches (see Step 4 TODO) |
| Wrong issue closed | Typo in issue ID, or PR body referenced multiple IDs | Reopen the wrongly-closed issue in Linear; fix and re-merge a follow-up PR if needed |
| Integration disappeared / authorization revoked | Someone reset the GitHub app install | Re-run Step 1 |

## Related docs

- `docs/agents/issue-tracker-linear.md` — Linear conventions used in this project
- `docs/agents/triage-labels.md` — issue label taxonomy
- `.claude/skills/implement-linear-issue/SKILL.md` — the agent skill that creates PRs following these conventions
