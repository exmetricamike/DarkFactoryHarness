---
description: Phase 3 - write one WP spec, have the reviewer review it, resolve the issues, freeze it
argument-hint: WP-XXX
---

# /df-spec $ARGUMENTS — author and harden one WP spec

Load the `implementer-protocol` skill before the first backend call; its step 0 resolves `roles.reviewer` from `harness.config.json`.
Preconditions: the WP exists in `active-project/BACKLOG.md`, its `Depends` are all `DONE`. Otherwise stop and say why.

Running unattended (inside `/df-run`): every gap you find in the spec is yours to fill. Decide with the `night-shift` tie-breakers — simplest thing that satisfies the outcome — write the decision into the WP spec as if it had always been there, and log it in `active-project/DECISIONS.md`. A WP spec with an open question in it is an unfinished spec.

## Step 1 — ground yourself in the current code

Read, at HEAD, in the target repo(s): the modules this WP will touch, the nearest existing analogue (a sibling feature built the same way), the test patterns, and `git -C <repo> log --oneline -10`.
The spec must describe *deltas to the code that exists now*, not to the code you imagined at planning time.

## Step 2 — write `active-project/wps/WP-XXX.md`

```markdown
# WP-XXX: <title>
Repos: <be:/path, fe:/path> | Depends: <WP-…> | State: DRAFT

## 1. Outcome
One sentence a user could verify: "<actor> can <do X> and sees <Y>."

## 2. Context
Why now, which spec sections (§n) this implements, what already exists that it builds on
(name the actual files/modules), and the analogue to copy the pattern from.

## 3. Scope
### In
- <bullet per deliverable>
### Out (explicitly not this WP)
- <bullet — kill the obvious scope creep before the reviewer finds it>

## 4. Design constraints
Decisions already made; the implementer must not relitigate them.
- data: <tables/collections/fields/migrations, with exact names and types>
- contract: <endpoints or operations: method, path, request, response, status codes, errors>
- UI: <routes/screens, states: loading/empty/error/success>
- auth: <who may call what>
- config: <new env vars, defaults>

## 5. Step plan (suggested; the implementer may improve the how, not the what)
1. …

## 6. Acceptance criteria
Numbered, observable, each independently checkable.
1. Given <state>, when <action>, then <observable result>.
### The implementer must ship these tests
- <test name/case> — <what it proves>
### Coordinator's independent check
- <the command/probe I will run myself, distinct from the implementer's tests>

## 7. Risks / edge cases
- <what will bite: races, nulls, permissions, migrations on existing data, pagination, timezones>

## 8. Non-goals for the reviewer
Do not propose: <refactors, alternate stacks, extra abstraction> — this project ships thin slices.
```

Quality bar before you hand it over: every noun exists (table, field, route, component names are literal, not "e.g."), every acceptance criterion is falsifiable, no sentence starts with "should probably".

## Step 3 — spec review (mandatory — this is the gate)

**No WP is implemented on an unreviewed spec.** Not by the implementer, not by you. The reviewer is a senior engineer reading the real repo with fresh eyes; you wrote the spec and are attached to it. Its review is the cheapest bug-prevention in the whole pipeline — a spec fix costs one round, the same mistake found after implementation costs three.

Build `active-project/wps/WP-XXX.prompt.md` using the envelope + **REVIEW block** from `implementer-protocol`, inlining the whole spec. First round creates the reviewer's session; capture the profile id and the session id into `active-project/wps/WP-XXX.log.md` and BACKLOG.

Ask for real critique, not a rubber stamp: what is ambiguous, what contradicts the code, what is missing that would block implementation, what this will break elsewhere. **Invite disagreement explicitly** — a review that returns READY on the first pass with nothing to say is a review you should push back on once ("what would you have done differently?") before trusting it.

For each returned issue: **verify the claim in the code first, then resolve it in the spec.**

| Type | How you resolve it |
|---|---|
| BLOCKER (contradicts the code, missing schema/contract, impossible order) | Check it against the repo. True → **amend the spec** and send the corrected version for another pass. False → reply in the next round with the file:line that disproves it and let it re-judge. Never leave a blocker standing. |
| GAP (edge case, error path, migration, permission not covered) | In scope → amend the spec so it is covered. Out of scope → move it to §3 Out explicitly, and add it to the backlog's Deferred list. Either way it is written down, not ignored. |
| SUGGESTION (different pattern, more abstraction, wider refactor) | Judge on merit: does it remove a real risk, shrink the diff, or match an existing repo pattern? Then take it. Does it only add scope or indirection? Decline it **in writing, with the reason** — the reviewer sees your reasoning in the next round and can push back. |

Rules that do not bend:

- **A flagged inconsistency gets fixed in the spec.** "Noted, proceeding anyway" is not a resolution.
- **You may not skip to implementation while a BLOCKER is unresolved.** If you truly cannot resolve one, shrink the WP until the blocker no longer applies and log the descope. Descoping is a resolution; overruling is not.
- Never accept a scope increase just because the reviewer offered it — but say why you declined.

Round n+1 prompt = the amended spec + a short resolutions list: `1. accepted, spec §4 updated. 2. disputed: <file:line evidence>. 3. declined: <reason>.`

**Loop until `VERDICT: READY`.** Rounds are cheap; rework is not.

After the profile's `spec_review_rounds` without READY, do not keep circling and do not bulldoze: pick the smallest spec the reviewer has no blockers against — cut the contested scope into a follow-up WP — and send that reduced spec for a confirming pass. A smaller reviewed WP beats a larger unreviewed one. Log the split in `active-project/DECISIONS.md`.

## Step 4 — freeze

Only reachable with a `VERDICT: READY` on the current text of the spec (or a logged descope the reviewer confirmed). If you are here without one, go back to Step 3.

1. Set `State: FROZEN` in `WP-XXX.md`, WP state `SPECCED` in BACKLOG. Record the READY round number next to it.
2. Append to `active-project/wps/WP-XXX.log.md`: reviewer profile id, session id, per-round issue → resolution → reason, anything descoped.
3. Report: outcome sentence, what the reviewer caught and how you fixed it, what you declined and why, `Next: /df-build WP-XXX`.

If the reviewer catching things makes a WP take four rounds, that is the system working. The rounds are the point.
