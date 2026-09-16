# Phase gates

Each `/df-*` command must leave its gate satisfied before the next phase starts.

| Command | Phase | Gate to pass before next |
|---|---|---|
| `/df-intake [repos]` | 1. Read all of `active-project/intake/`, interrogate, discover the stack | `active-project/SPEC.md` has zero OPEN items |
| `/df-plan` | 2. Work-package decomposition | `active-project/BACKLOG.md` exists, WPs ordered + dependency-clean |
| `/df-spec WP-XXX` | 3. Write the WP spec, have the reviewer review it, resolve what it flags | `VERDICT: READY` → WP state `SPECCED` |
| `/df-build WP-XXX` | 4. The implementer implements, you verify, you commit | WP state `DONE` |
| `/df-run` | 5. Unattended loop: spec→build every WP, then prove the product runs | backlog done + `active-project/MORNING.md` written |
| `/df-status` | anytime | — |
| `/df-implementer [profile]` | anytime | active roles preflighted and recorded |
| `/df-retro` | 6. Morning: distil the night into cross-project lessons | `LESSONS.md` updated + committed, candidates emptied |
| `/df-pause` | credits low, or you must stop | `active-project/RESUME.md` written, wake-up scheduled |
| `/df-resume` | after a pause | checkpoint verified against the repos |
| `/archive-project [name]` | project closed | `active-project/` empty, snapshot under `projects-archive/` |
| `/reactivate-project <name>` | starting a previously archived project | snapshot moved back into `active-project/` |
