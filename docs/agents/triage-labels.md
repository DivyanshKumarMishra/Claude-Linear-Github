# Triage Labels

This repo uses two independent label dimensions in Linear. An issue normally carries one label from each.

Resolve every label name below to its Linear label id at runtime via `mcp__linear__list_issue_labels` (scoped to the `Claude-Linear-Github` team). Do not hardcode ids — they differ per workspace. Cache the name→id map for the session.

## Dimension 1 — Triage labels (workflow state)

The skills speak in terms of five canonical triage roles. This table maps each role to the label string used in this repo.

| Canonical role    | Label in our tracker | Meaning                                  |
| ----------------- | -------------------- | ---------------------------------------- |
| `needs-triage`    | `needs-triage`       | Maintainer needs to evaluate this issue  |
| `needs-info`      | `needs-info`         | Waiting on reporter for more information |
| `ready-for-agent` | `ready-for-agent`    | Fully specified, ready for an AFK agent  |
| `ready-for-human` | `ready-for-human`    | Requires human implementation            |
| `wontfix`         | `wontfix`            | Will not be actioned                     |

When a skill mentions a role (e.g. "apply the AFK-ready triage label"), use the corresponding label string from this table.

## Dimension 2 — Type labels (nature of work)

These describe what kind of change the issue represents. Every PRD-derived issue gets exactly one.

| Type label    | Use when                                                                 |
| ------------- | ------------------------------------------------------------------------ |
| `Feature`     | New capability that did not exist before                                 |
| `Improvement` | Enhancement or change to behavior that already exists                    |
| `Bug`         | Fixing incorrect behavior in something that already exists               |

### How to choose the type label

- A slice from a PRD describing brand-new functionality → `Feature`.
- A slice that extends, tightens, or refactors existing behavior → `Improvement`.
- A slice that corrects a defect → `Bug`.

When unsure between `Feature` and `Improvement`, ask: does this code path exist today? If no → `Feature`; if yes → `Improvement`.

## Applying both

A typical PRD-derived, AFK-ready issue carries `ready-for-agent` + `Feature`. Because Linear's `update_issue` replaces the whole label set, always send the union of both label ids (plus any already present) — see `issue-tracker-linear.md`.

Edit the right-hand columns above if the actual Linear label names differ from these strings.
