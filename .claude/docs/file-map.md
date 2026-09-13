# Working-state file map

Everything under `active-project/` is the current run's working state and is git-ignored.
**During a run, only `active-project/` is read or considered.** `projects-archive/` is
closed-out projects only — touched by `/archive-project` and `/reactivate-project`, never
by any phase command.

```
LESSONS.md                          cross-project memory: what previous nights taught. Read every phase;
                                    written only by /df-retro. Never contains project identity.
active-project/intake/              USER-SUPPLIED source material: spec docs, mockups, data samples
                                    [read-only — never edit; /df-intake consumes all of it]
active-project/PROJECT.md           repos, stacks, run/test/lint commands, conventions   [written by /df-intake]
active-project/SPEC.md              refined, actionable product spec + open-questions ledger
active-project/BACKLOG.md           WP table = progress source of truth
active-project/wps/WP-XXX.md        frozen WP spec handed to Codex
active-project/wps/WP-XXX.log.md    Codex session id, review rounds, decisions, verdicts
active-project/wps/WP-XXX.prompt.md last prompt sent to Codex (self-contained)
active-project/wps/WP-XXX.out.md    last Codex reply (from codex -o)
active-project/checks/WP-XXX.*      YOUR independent acceptance check (never inside a repo)
active-project/RESUME.md            checkpoint: the one file that survives a session dying
active-project/DECISIONS.md         append-only log of every call you made without the user
active-project/MORNING.md           the report the user reads over coffee — written at the end of the run
projects-archive/<name>/            a closed-out project, same layout as active-project/
                                    [archive only — never read or written during a run]
```
