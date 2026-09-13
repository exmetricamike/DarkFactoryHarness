# LESSONS.md protocol

This harness runs on project after project.
It should be better at the tenth than at the first, and the only thing that carries across is `LESSONS.md` at the repo root — `active-project/` gets wiped or replaced, that file does not.

- **Read it at the start of every phase command**, before you decide anything.
It is capped at 30 one-line lessons so this stays cheap.
- **Step 0 of any decision is "did a previous night already answer this?"** A lesson that applies outranks your own reasoning from scratch — that is the whole point of having it.
- **Capture a candidate the moment something costs you.** A spec that took ≥3 review rounds, a WP reverted or blocked, ≥2 fix rounds on the same defect, a decision you had to reverse, an hour lost to environment or tooling.
One line appended to the `Candidates` section of `LESSONS.md`, then keep working — do not stop to philosophise, and do not promote it to a lesson mid-run.
- **`/df-retro` in the morning does the distilling**, with the user's grades on `active-project/DECISIONS.md` in hand.
Their verdict on a decision beats your own account of it.
- **General rules only, and never project identity.** The lesson must be actionable on a project this harness has never seen; the context is the project's *shape* ("a fintech dashboard SPA on React + FastAPI, WP touching auth"), never client names, product names, repo paths or proprietary domain terms.
Anything true only of this product belongs in `active-project/`, not here.
