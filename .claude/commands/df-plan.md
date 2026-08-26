---
description: Phase 2 - decompose the actionable spec into ordered work packages and write the backlog
---

# /df-plan — work-package decomposition

Preconditions: `project/SPEC.md` has zero OPEN items and `project/PROJECT.md` exists. If not, stop and run `/df-intake`.

## Sizing rule

One WP = **one Codex session**: a vertical slice a competent implementer finishes in one sitting.
Target ~5-15 files touched. If a WP needs more than ~15 files or two unrelated concerns, split it.
If it touches fewer than 2 files, merge it into a neighbour — coordination overhead is not free.

## Decomposition heuristics (in order)

1. **Foundations first**: schema/migrations, auth, config, shared contract types, app shell. These unblock everything.
2. **Then vertical slices**: each feature WP delivers backend + frontend for one user-visible capability, so it can actually be verified.
3. **Split by seam, not by layer** — avoid "all the endpoints" then "all the UI"; that defers integration risk to the end. Exception: the contract/schema foundation WPs.
4. **Isolate the risky/unknown** into its own early WP (third-party integration, novel algorithm, perf-sensitive path) so failure surfaces early.
5. **Cross-repo WPs**: allowed, but the WP must state the order (contract → backend → frontend) and both repos' acceptance.
6. **Dependencies are a DAG, no cycles.** If two WPs need each other, extract the shared piece into a third.

## Numbering

`WP-001`, `WP-002`, … in intended execution order. Never renumber later; new work gets the next free number regardless of where it belongs logically.

## Write `project/BACKLOG.md`

```markdown
# Backlog
Updated: <date> | DONE <n>/<total>

## Order
| ID | Title | Repos | Depends | State | Codex session | Notes |
|----|-------|-------|---------|-------|---------------|-------|
| WP-001 | … | be | — | TODO | — | |

State: TODO -> SPECCED -> BUILDING -> VERIFY -> DONE | BLOCKED

## Milestones
- M1 walking skeleton: WP-001..003 — <what the user can see/do when these land>
- M2 …

## Deferred / later
- <item> (from SPEC open items)
```

Each row's **Notes** column holds one line: the user-visible outcome of that WP. If you cannot write that line, the WP is not a vertical slice — reshape it.

## Then

1. Present the plan to the user as a compact table plus milestone summary, and flag any WP you are unsure about.
2. Ask for approval to proceed (`AskUserQuestion`: approve / reorder / split-merge specific WPs) — **only if the user is present**. If this ran from `/df-run` or any unattended path, skip the approval, note the plan in `project/DECISIONS.md`, and start building.
3. On approval: `Next: /df-spec WP-001` — or `/df-run` for the autonomous loop.

The plan is a starting order, not a contract. Overnight you may reorder, split, or merge WPs to keep the line moving — log it and update the backlog.

Do **not** write WP specs here. They are written just-in-time in `/df-spec`, so each one is grounded in the code as it actually exists at that moment.
