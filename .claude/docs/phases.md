# Phase gates

Each `/df-*` command must leave its gate satisfied before the next phase starts.

| Command | Phase | Gate to pass before next |
|---|---|---|
| `/df-intake [repos]` | 1. Read all of `project/intake/`, interrogate, discover the stack | `project/SPEC.md` has zero OPEN items |
| `/df-plan` | 2. Work-package decomposition | `project/BACKLOG.md` exists, WPs ordered + dependency-clean |
| `/df-spec WP-XXX` | 3. Write the WP spec, have Codex review it, resolve what it flags | Codex `VERDICT: READY` → WP state `SPECCED` |
| `/df-build WP-XXX` | 4. Codex implements, you verify, you commit | WP state `DONE` |
| `/df-run` | 5. Unattended loop: spec→build every WP, then prove the product runs | backlog done + `project/MORNING.md` written |
| `/df-status` | anytime | — |
| `/df-retro` | 6. Morning: distil the night into cross-project lessons | `LESSONS.md` updated + committed, candidates emptied |
| `/df-pause` | credits low, or you must stop | `project/RESUME.md` written, wake-up scheduled |
| `/df-resume` | after a pause | checkpoint verified against the repos |
