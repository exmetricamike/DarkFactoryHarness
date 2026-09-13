# Dark Factory Harness

Claude coordinates, specs, reviews and tests. **Codex reviews every spec and writes the code.**
Drop your spec, mockups and supporting documents in `active-project/intake/`, point it at your two repos, and go to bed. Read the report in the morning.

## Setup

1. Have `codex` on PATH and logged in (`codex --version`).
2. Put your spec document, mockups, screenshots and any supporting material into `active-project/intake/` — see the README in there. Everything in that folder gets read.
3. Have your frontend and backend repos cloned locally, working trees clean.
4. For UI testing, have the Claude in Chrome extension connected and permitted for `localhost`.
5. Open Claude Code in this folder, and leave the window open overnight.

## Run it

| Step | Command | What you do |
|---|---|---|
| 1 | `/df-intake <frontend-repo> <backend-repo>` | Claude reads everything in `active-project/intake/` first, then asks only what's still missing. Answer until it stops. Repeat as long as it keeps asking — this is the step that decides quality. |
| 2 | `/df-plan` | Review the work-package list; approve, or ask to split/reorder. |
| 3 | `/df-run` | Go to bed. See below. |
| 4 | `/df-retro` | In the morning, after reading the report: grade the calls Claude made, and it turns the night into lessons for the next project. |
| — | `/df-status` | Where things stand, anytime. |
| — | `/df-pause` / `/df-resume` | Stop safely when credits run low, pick up later. See below. |
| 5 | `/archive-project` | Done with this project: moves `active-project/` into `projects-archive/`, ready for the next one. |
| — | `/reactivate-project <name>` | Bring a past project back into `active-project/`. Only works when `active-project/` is empty. |

Prefer one WP at a time? Use `/df-spec WP-001` then `/df-build WP-001` instead of `/df-run`.

## The night shift

`/df-run` is built for you to be asleep. From the moment you launch it, Claude is the manager: it decides spec ambiguities, design trade-offs and scope calls on its own — by software-engineering best practice, and when in doubt by whatever is simplest — and writes every call to `active-project/DECISIONS.md` instead of asking you. **It will not stop to ask questions.**

It doesn't just write code, either: it starts the app, hits the endpoints, drives the real UI in Chrome, and reads the console for errors. A work package with green tests and a blank screen is treated as failed. After the last package it does a clean-slate boot and walks every flow in your spec end to end.

A stalled pipeline is treated as the worst possible outcome. A blocked package, a failing test, a broken dev environment, a spec Codex keeps objecting to (which gets cut down until it's clean rather than forced through) — none of those halt the night; Claude routes around them and takes the next package. It halts only if every package is done or stuck, both agents are out of credit, or it hits a genuine security/data-loss risk or a spec contradiction it can't resolve without inventing what your product is supposed to be.

**In the morning, read `active-project/MORNING.md` first.** Bottom line, what needs your eyes (⚠ decisions and blocked packages), what was done and how it was verified, and the exact commands to run the product.

Leave the terminal open. That's what lets the credit-pause wake-up fire.

## Running out of credit

**Claude runs low.** Claude cannot read its own usage — `/usage` is a screen only you can see — so it goes by Claude Code's approaching-limit warnings. On one of those (or on `/df-pause`), it commits anything already verified, writes `active-project/RESUME.md`, schedules a wake-up **4 hours out**, and stops. No questions asked; you're asleep.

When the wake-up fires, Claude re-reads the checkpoint, reconciles it against the actual repos, and goes straight back into `/df-run`. Over a long night that's typically one pause and one self-restart.

> The wake-up only fires if the terminal stays open and idle. If it's closed, nothing happens — run `/df-resume` in the morning and it picks up exactly where it stopped. `active-project/RESUME.md` always survives.

**Codex runs out of tokens.** Claude keeps working alone, in this order: draft the remaining WP specs, write its own acceptance checks, then implement only *trivial* work packages (≤3 files, no schema, no auth, no dependency, no contract change). Those commits are tagged `Implemented-by: Claude` and marked `DONE*` in the backlog. When Codex is back, it reviews every unreviewed spec and every `DONE*` commit before any new work starts.

## It gets better with each project

The harness keeps a memory across projects in `LESSONS.md`, at the root — `active-project/` belongs to the
current job and gets replaced, that file does not. Every night, whenever something costs rounds (a spec
Codex kept objecting to, a reverted package, a defect that took three fixes, an hour lost to the dev
environment), Claude drops a one-line candidate into it and keeps working.

In the morning you read `active-project/MORNING.md`, and at the bottom it asks you to grade the judgment calls
it made while you slept — `good`, `bad`, or `bad: what I'd have done` in the `Grade` column of
`active-project/DECISIONS.md`. Then `/df-retro` distils: your grades plus the night's candidates become a handful
of general rules, and step 0 of every decision on the *next* project is checking whether one of them
already answers it.

Two things it will not do. It will not keep lessons that are only true of this product — those stay in
`active-project/` — and it will not write your product, client or repo names into `LESSONS.md`. A lesson records
the *shape* of the job it came from ("a fintech dashboard SPA on React + FastAPI, package touching auth")
and what the mistake cost, never what the thing was. The file is capped at 30 lessons; adding one past the
cap means merging or dropping the weakest, so it stays something readable at 2am rather than an archive.

If you start a new project by copying this harness, copy `LESSONS.md` with it. That file is the harness.

## Where to look

- `active-project/intake/` — your source material, exactly as you left it
- `active-project/SPEC.md` — the refined spec Claude built from it, with the decision log
- `active-project/BACKLOG.md` — progress, one row per work package
- `active-project/wps/WP-XXX.md` — what Codex was told to build
- `active-project/MORNING.md` — **read this first in the morning**
- `active-project/DECISIONS.md` — every call Claude made while you were away
- `active-project/RESUME.md` — the checkpoint: where things stood, what to do next
- `LESSONS.md` — what the harness learned on this and every previous project
- `active-project/wps/WP-XXX.log.md` — what Codex said, what Claude accepted or rejected, and why
- `projects-archive/` — closed-out projects, moved there by `/archive-project`; never read during a run

## Rules the harness enforces

- Codex writes the product code as a rule; Claude delegates rather than implements, and writes code itself only for limited fixes or when Codex is unavailable — flagged in the commit, re-reviewed by Codex later.
- **No work package is implemented on an unreviewed spec.** Claude writes the spec, Codex reviews it as a peer and flags what's ambiguous, contradictory or missing, Claude fixes the spec, and only a `READY` verdict unlocks implementation.
- Claude commits, and never pushes unless you ask.
- Nothing is committed until Claude has run the tests itself and passed its own independent check.
- Codex runs sandboxed to the repo it is working in (`workspace-write`).
