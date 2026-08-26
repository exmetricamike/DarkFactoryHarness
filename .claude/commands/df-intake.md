---
description: Phase 1 - ingest the initial spec, analyze the provided repos to infer the stack, interrogate the user until the spec is actionable
argument-hint: [path to spec document] [frontend repo path] [backend repo path]
---

# /df-intake — make the spec actionable

Input: `$ARGUMENTS` (may be empty — then ask for the spec path and the two repo paths).

Goal: produce `project/PROJECT.md` (stack truth) and `project/SPEC.md` (product truth, zero OPEN items).
Do not decompose into work packages here. That is `/df-plan`.

## Step 1 — ingest

1. Copy/normalize the user's spec into `project/SPEC.md` verbatim first (keep their words; you will annotate, not rewrite away).
2. Read it fully. List what it does and does not answer.

## Step 2 — infer the stack from the repos (do not ask what the code can tell you)

For **each** repo, read — do not guess:
- manifest + lockfile (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `*.csproj`) → language, framework, versions, package manager
- scripts/targets → dev, build, test, lint, typecheck, migrate commands
- `README`, `CLAUDE.md`, `AGENTS.md`, `.editorconfig`, linter config → conventions
- directory layout, 2-3 representative source files → naming, error handling, logging, module patterns
- test setup → runner, location, style (unit/integration/e2e)
- infra → Dockerfile, compose, CI workflows, `.env.example`, migrations dir
- git: `git -C <repo> log --oneline -15`, current branch, `git status --short`

Write `project/PROJECT.md`:

```markdown
# Project: <name>
One-paragraph product summary.

## Repos
| Role | Path | Branch | Language/Framework | Package manager |
### Contract between them
<REST/GraphQL/tRPC/…, where the contract lives, how types are shared or duplicated>

## Commands (per repo — exact, copy-pasteable)
| Repo | dev | test | lint | typecheck | build | migrate |

## Conventions (inferred — cite the file you inferred each from)
- naming: … (src/…)
- errors: …
- logging: …
- state/data access: …
- styling/UI: …
- auth: …
- tests: …

## Environment
services (db, cache, queue), env vars, ports, seed/fixture story.

## Uncertain — confirm with user
- [ ] …
```

Anything you could not determine goes in **Uncertain**, never invented.

## Step 3 — gap analysis of the spec

Score the spec against this checklist. Every unanswered item becomes an OPEN item.

- **Users & roles**: who are the actors, what may each do, auth model, tenancy (single/multi)
- **Core entities**: the data model nouns, their relations, lifecycle/state transitions, ownership
- **Flows**: the top user journeys end-to-end, including the unhappy paths
- **Screens/surfaces**: which UI screens exist, what each shows and does
- **API surface**: which operations the frontend needs; sync vs async; pagination/filtering
- **Rules & invariants**: validation, permissions, limits, money/units/timezones
- **External integrations**: providers, auth, sandbox/live, failure behavior
- **Non-functional**: expected scale, latency budget, availability, retention, audit
- **Compliance/privacy**: PII, GDPR, data residency, deletion
- **Ops**: environments, deploy target, observability, migrations/backfills
- **Out of scope**: what this project explicitly will NOT do
- **Acceptance**: how the user will judge the product is working (per flow)

## Step 4 — interrogate the user

Ask with `AskUserQuestion`, **max 4 questions per call**, highest-leverage first (things that change the data model or the architecture come before things that change a screen).
Rules:
- Always offer a concrete recommended default as the first option, marked `(Recommended)`, inferred from the repos and the domain. Cheap for the user to say "yes".
- Never ask what the repos already answered.
- Never ask two questions that collapse into one decision.
- Keep looping until every checklist item is Resolved or the user marks it Deferred.
- Batch related questions; don't drip one at a time.

After each answer round, immediately fold the answers into `project/SPEC.md` — never keep decisions only in the chat.

## Step 5 — write the refined spec

`project/SPEC.md` final shape:

```markdown
# <Product> — Specification
Status: ACTIONABLE | OPEN ITEMS: <n>

## 1. Product summary  (what, for whom, why now)
## 2. Actors & permissions
## 3. Domain model            (entities, fields, relations, states — this is what Codex builds against)
## 4. User flows              (numbered, with unhappy paths)
## 5. Surfaces / screens
## 6. API surface             (operation list; detail lives in the WP specs)
## 7. Rules & invariants
## 8. Integrations
## 9. Non-functional & compliance
## 10. Out of scope
## 11. Acceptance criteria    (per flow, observable, testable)

## Decision log
| # | Question | Decision | Date | Source |

## Open items
| # | Question | Why it matters | State (OPEN/DEFERRED) |
```

## Gate

Do not proceed to `/df-plan` while any item is `OPEN` (DEFERRED is fine, with the deferral recorded).
Finish by telling the user: the resolved/deferred counts, the top 3 risks you see, and `Next: /df-plan`.

**This is the one phase that needs the user.** It is the evening conversation before the night shift, and every question answered here is a decision you don't have to make alone at 3am. Ask thoroughly — cheap now, expensive later.

If intake is somehow reached unattended (the user launched `/df-run` with an unrefined spec): do not stall. Answer every OPEN item yourself with the `night-shift` tie-breakers, mark each `DECIDED (unattended)` in the decision log with a `⚠`, and carry on to `/df-plan`. A guessed spec that ships is worth more than a perfect spec nobody wrote.
