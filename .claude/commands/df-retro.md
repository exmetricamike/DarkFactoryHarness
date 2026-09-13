---
description: Distil the night into cross-project lessons - run it after reviewing the morning report
argument-hint: [optional: "grades in chat" or nothing]
---

# /df-retro — turn this project's experience into harness memory

Run this **in the morning, after the user has read `active-project/MORNING.md`** and graded the decisions.
This is the only step that makes the next project cheaper than this one. It edits exactly one file
outside `active-project/`: `LESSONS.md`.

The user is present here — unlike the night, asking is allowed and cheap.

## Step 1 — collect the evidence (read, do not re-derive)

- `LESSONS.md` — the candidates section, and the existing lessons (you may be confirming or contradicting one).
- `active-project/DECISIONS.md` — every call made alone, with the `Grade` column the user filled in.
- `active-project/BACKLOG.md` — what ended `BLOCKED`, what got split, what was deferred.
- The `log.md` of any WP that took ≥3 spec rounds, ≥2 fix rounds, or got reverted. **Only those** —
  a smooth WP teaches nothing.

## Step 2 — get the grades

If `DECISIONS.md` has ungraded rows the user plausibly cares about (the `⚠` ones first), ask —
one `AskUserQuestion` call, batched, max 4, phrased as "was this the right call?" with the
alternative you rejected as an option. Ungraded after asking → leave it ungraded and move on.

**A graded decision outranks your own reading of the night.** If the user says a call was wrong,
the lesson is written from their verdict, not from your defence of it.

## Step 3 — distil

For each candidate line, each bad grade, and each ≥3-round or reverted WP, ask in order:

1. **Was this project-specific?** A quirk of this stack, this repo, this product → drop it. It is
   already in `active-project/`. Do not launder it into a general rule.
2. **Would the rule have changed the outcome on a project I have never seen?** No → drop it.
3. **Does an existing lesson already cover it?** Yes → bump `hits`, sharpen the wording if this
   night taught it better, and if the new context is a *different* project shape raise `conf`.
   Do not add a second line saying the same thing.
4. **Does it contradict an existing lesson?** Rewrite that lesson to the current answer, or delete
   it. Never leave two rules that fire on the same situation.
5. **Otherwise** → new lesson in the right phase section, in the file's one-line format, with the
   project shape written generically (see the header of `LESSONS.md` — shape and cost, never identity).
6. **Real pain, no rule yet?** → `Known friction` table instead. Recording "spec review kept stalling
   on migrations" honestly beats inventing a fix you have not tested.

Then enforce the ceiling: >30 lessons → merge or delete the weakest until it fits. Say which you cut.

## Step 4 — close

1. Empty the `Candidates` section of `LESSONS.md`. Nothing survives undistilled.
2. Commit `LESSONS.md` alone: `git add LESSONS.md && git commit -m "retro: <n> lessons after <generic project shape>"`.
   The lessons are the harness's memory — an uncommitted lesson is a forgotten one.
3. Report, short:

```
Retro — <generic project shape>
  graded:    <n> decisions (<n> good / <n> bad / <n> ungraded)
  new:       L-0xx <one line each>
  updated:   L-0xx (hits+1) …
  dropped:   <n> candidates (project-specific)
  friction:  <new or recurring rows>
  file:      <n>/30 lessons
```

Two or three real lessons is a good retro. Ten is a sign you promoted project detail — go back to Step 3.
