---
description: Phase 5 - run the factory unattended until every WP is done and the product runs
argument-hint: [optional: stop-after WP-XXX]
---

# /df-run — run the factory

**Load the `night-shift` skill now, before the first iteration.** It governs every decision you make from here.
**Read `LESSONS.md` in the same breath.** Previous nights already paid for those answers; step 0 of every decision tonight is checking whether one applies.

Preconditions: `project/BACKLOG.md` exists. That is all. Do **not** wait for plan approval — if the user launched this, the plan is approved.

## The contract you just accepted

The user is away, probably asleep, and will read the results in the morning. You are the manager. You decide everything: spec ambiguities, design trade-offs, scope calls, ordering, when a feature is good enough. You do not ask, you log. You do not stop, you route around. By morning every WP is implemented and the product runs.

**No `AskUserQuestion` while this loop is active.** If you catch yourself drafting one, you have already failed the contract — decide it with the tie-breakers in the skill and write it to `project/DECISIONS.md`.

## Loop

```
loop:
    re-read project/BACKLOG.md          <- the state; your memory is not
    pick the lowest-numbered WP with state TODO/SPECCED whose Depends are all DONE
    none available?
        any WP still TODO/SPECCED behind a BLOCKED dep?  -> try to unblock it (skill §4), else re-plan the order
        truly nothing runnable -> exit loop
    if state == TODO:    follow .claude/commands/df-spec.md
    if state == SPECCED: follow .claude/commands/df-build.md
    update BACKLOG, write project/RESUME.md, append any calls to project/DECISIONS.md
    anything cost you rounds? -> one candidate line in LESSONS.md (night-shift skill 2b), then move on
    report one compact block, continue
on exit:
    final integration pass (night-shift skill §6)
    write project/MORNING.md (skill §7)
```

Follow those command files literally. A long night is not a licence to run an abbreviated version of the protocol — that is how 3am work becomes morning rework.

## Credit events during the loop

Load the `continuity` skill on either signal.

- **Your credits**: an approaching-limit warning appears → commit anything verified, write `project/RESUME.md`, schedule the one-shot cron **4 hours out** (no user input, no reset-time question — they are asleep), stop. The cron fires while the terminal sits idle overnight and the loop continues itself. Do not start another WP hoping it fits.
- **Codex's credits**: a `codex exec` call fails on quota → one retry, then switch to the takeover ladder (spec ahead → write your checks → trivial WPs only) and **keep looping in that reduced mode**. Retry Codex at the parsed reset time, else hourly. Log the switch; do not silently change mode.

Never spend your own remaining budget implementing non-trivial WPs because Codex is down.

## Halt the loop only when

- **every** WP is `DONE` or `BLOCKED` and nothing can be unblocked
- both agents are out of credit (checkpoint first — the cron restarts you)
- the `stop-after` WP in `$ARGUMENTS` is `DONE`

That is the whole list. A blocked WP, a failing test, a Codex disagreement, an unclear requirement, a broken dev environment, a spec that turned out wrong — none of these halt the loop. Mark, log, route around, continue. Even a security stop (skill §3) only blocks that one WP.

Before halting on "nothing runnable", check twice: re-read the backlog dependencies, and ask whether a blocked WP can be split so its unblocked half ships tonight.

## Per-WP report (one block, no essays)

```
WP-XXX <title> — DONE | BLOCKED
  spec: <n> rounds, <n> accepted / <n> rejected
  build: <n> fix rounds | tests <cmd> -> <result>
  verified: <what you actually exercised — endpoints hit, UI flow driven, console clean>
  decisions: #<n>, #<n>   follow-ups: <WP ids or none>
  commits: be <sha> fe <sha>
```

## On completion

Run the final integration pass, then write `project/MORNING.md`.

Leave the `Candidates` section of `LESSONS.md` as it is — unfiltered, undistilled, uncommitted to the lesson
sections. Distilling is `/df-retro`'s job in the morning, and it is better done with the user's grades than
with your own account of your own night. The one exception: a candidate about the *harness protocol itself*
that cost you the night and cannot possibly be project-specific (a step that always misfires, a command that
never had the input it needs) — promote that one, so the next run does not repeat it before anyone is awake.

Final message to the user: the bottom line, what needs their eyes, the command to run the product, and
`Next: grade the ⚠ decisions, then /df-retro`. Then stop — do not invent new work to fill the night.
