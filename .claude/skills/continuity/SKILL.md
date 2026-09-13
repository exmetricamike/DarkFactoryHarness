---
name: continuity
description: Checkpoint and resume when Claude or Codex runs out of credit. Load on a usage-limit warning, a codex quota failure, /df-pause or /df-resume.
---

# Continuity protocol

Two independent failure modes. They can happen at the same time; handle Claude's first.

---

# A. Claude approaching its credit limit

## Detection — what actually exists

You **cannot** read your own credit utilization. `/usage` is a REPL UI command: you cannot invoke it and its output never reaches you. No file under `~/.claude` holds the numbers. Do not pretend to measure it.

Trigger the pause on any of these, whichever comes first:

1. A usage-limit warning appears in the conversation (approaching-limit notice, "limit resets at ...", a degraded-model notice). **This is the primary trigger.**
2. The user runs `/df-pause`, or tells you the limit is near.
3. The user pastes `/usage` output — read the remaining % and the reset time from it.

Independently: **checkpoint after every WP regardless**. `active-project/RESUME.md` costs one file write and makes any unannounced cutoff lose at most one work package. A cheap checkpoint everywhere beats a clever threshold nowhere.

## On trigger — do this in order, and keep it short

You may have very little budget left. Do not start new work, do not run Codex, do not "just finish this one verification".

1. **Stop at the nearest safe boundary.** Safe = the repos are in a consistent state.
   - Codex is mid-implementation → let a call that is already in flight finish, then stop. Never abandon a half-written call you cannot verify.
   - You already verified and passed → **commit first**, then checkpoint. Uncommitted verified work is the one thing you cannot reconstruct.
   - Verification failed mid-way → do not commit; record the failure in the checkpoint and leave the tree dirty, noting exactly which files.
2. **Write `active-project/RESUME.md`** (format below). This is the only artifact guaranteed to survive.
3. **Schedule the wake-up 4 hours out** (see below). Do not ask the user for a reset time — assume they are asleep. Four hours is the window; it needs no confirmation.
4. **Report and stop.** One block: what is done, what is mid-flight, when the resume fires, and `/df-resume` as the manual fallback.

## RESUME.md format

Overwrite it every time; it is a snapshot, not a log.

```markdown
# RESUME — written <ISO timestamp>
Reason: credit-pause | codex-down | manual
Resume at: <ISO timestamp or "any time">

## Where we are
Phase: intake | plan | spec WP-XXX | build WP-XXX | idle
Backlog: <n>/<total> DONE. In flight: <WP-XXX, state> | none

## Repo state at checkpoint
| repo | branch | HEAD sha | clean? | if dirty: which files and why |

## Mid-flight detail  (omit if nothing is in flight)
- Codex session id: <uuid>
- Last Codex verdict: <READY|DONE|PARTIAL|BLOCKED> from <round n>
- Last thing I did: <one line>
- Last command I ran and its result: <one line>
- What was about to happen next: <one line>

## Next action, verbatim
<the exact command or step to take on resume, e.g. "/df-build WP-004 — re-run verification step 3, tests were failing on auth_test.py::test_expiry">

## Do not redo
<things already done that look undone: e.g. "WP-004 spec review is finished, do not re-review">

## Open decisions waiting on the user
<or "none">
```

## Scheduling the wake-up

Two mechanisms, use **both** — they fail in different ways.

1. **Durable**: `active-project/RESUME.md` on disk. Survives everything. This is the real one.
2. **Best-effort**: a one-shot cron in this session.

```
CronCreate(cron: "<M H DoM Mon> *", recurring: false,
           prompt: "/df-resume - scheduled resume after credit pause")
```

Pin minute/hour/day-of-month/month to **now + 4 hours**, computed from the actual current date — check it, and roll the day/month over correctly past midnight. Pick an off-minute (`:07`, `:23`), never `:00`/`:30`.

Overnight this is the mechanism that keeps the factory alive: the terminal sits idle, the cron fires, `/df-resume` picks up from `RESUME.md`, and the loop continues without anyone present. So the checkpoint must be good enough for a cold read — write it as if the reader knows nothing, because it does.

State it once in your report, without hedging it into noise:

> Resume scheduled for <time>. It fires only if this terminal stays open; `active-project/RESUME.md` holds the state either way.

Never claim the resume is guaranteed. Never register an OS-level scheduled task unless the user asks for it.

---

# B. Codex out of tokens

## Detection

A `codex exec` call fails, or its output contains a quota signal. Match case-insensitively on stderr, exit code, and `.out.md`:

`usage limit` · `rate limit` · `quota` · `429` · `too many requests` · `insufficient` · `try again (after|in)` · `resets (at|in)`

Distinguish it from a normal failure: a quota failure produces no code changes and usually fails within seconds. Confirm with **one** retry after ~60s. If it fails the same way, declare Codex down; log it in `active-project/RESUME.md` (`Reason: codex-down`) and in the WP log with the exact message and any reset time you can parse.

If the failure lands **mid-implementation** (files already changed), first run `git -C <repo> status --short` and `git -C <repo> diff` and record what exists. Do not commit a half-finished WP. Note in the WP log which files are partial, so the resumed Codex session knows what it left behind.

## While Codex is down — the takeover ladder

Work down it, top first. Never skip ahead to code because the queue is boring.

1. **Spec ahead.** Write `active-project/wps/WP-XXX.md` for every remaining WP whose dependencies are DONE or will be. Mark each `State: SPECCED-UNREVIEWED` and add `Pending: Codex spec review` to its backlog row. **These are drafts, not frozen specs** — the review gate is owed and unpaid, so nothing gets implemented from them while Codex is down except what clears the trivial gate below. Review your own drafts against the code as hard as you can in the meantime; it is not a substitute, it is what you have.
2. **Write the acceptance checks.** For each unreviewed spec, write your independent check into `active-project/checks/WP-XXX.*`. Outside the repos, always.
3. **Implement trivial WPs only.** See the gate below.
4. **Nothing left → stop.** Write RESUME.md, schedule a retry, report. Do not invent work to look busy.

## The trivial-WP gate

You may write code when you judge it necessary, but delegation is the default for a reason: Codex reviews the repo with fresh eyes, and code you both wrote and reviewed has had one pair of eyes, not two. With Codex down you lose that second pair — so what you take on alone stays small and obvious.

Implement a WP yourself while Codex is out only if it passes **every** test:

- touches <= 3 files
- adds no dependency
- no schema, migration, or data backfill
- no auth, permission, money, or PII path
- no change to a public contract (route, response shape, shared type, event)
- no new abstraction, no refactor of existing code
- the whole diff is obvious enough that a reviewer needs no explanation

Typical: config and env samples, docs/README, static copy and i18n strings, lint/CI config, a single pure helper plus its test.

**When in doubt, it is not trivial.** A WP is not trivial because it is urgent, and not trivial because you are bored waiting.

If it passes: implement it, run the repo's own tests and lint, run your independent check, then commit with the trailer that marks it:

```
WP-XXX: <title>

<what and why>
Spec: active-project/wps/WP-XXX.md
Tests: <command> -> <result>
Implemented-by: Claude (Codex unavailable — pending Codex review)
```

Backlog state becomes `DONE*` (the star means Claude-built). Add it to the review queue in RESUME.md.

## When Codex returns

Before starting any new WP, drain the debt in this order:

1. Resume the interrupted WP's Codex session with a FIX/continue prompt that inlines the current `git diff`, so it knows what it left half-done.
2. Run the deferred spec-review round for every `SPECCED-UNREVIEWED` WP — one call per WP. Fold the issues in as usual, then mark them `SPECCED`.
3. Have Codex review every `DONE*` commit (`git -C <repo> show <sha>`, "review this, do not rewrite it unless it is wrong"). Fix what it finds, then drop the star.

Only then continue the normal loop.

## Probing for recovery

Do not poll in a tight loop — each probe costs your credits, not Codex's.

- If you parsed a reset time: wait for it, then probe once.
- Otherwise: probe at +60 min, then hourly, max 5 probes, then stop and tell the user.
- A probe is the next real call you needed to make anyway. Never burn a call on "are you back?".
