# Working-state file map

Everything under `project/` is per-project working state and is git-ignored.

```
LESSONS.md                  cross-project memory: what previous nights taught. Read every phase;
                            written only by /df-retro. Never contains project identity.
project/intake/             USER-SUPPLIED source material: spec docs, mockups, data samples
                            [read-only — never edit; /df-intake consumes all of it]
project/PROJECT.md          repos, stacks, run/test/lint commands, conventions   [written by /df-intake]
project/SPEC.md             refined, actionable product spec + open-questions ledger
project/BACKLOG.md          WP table = progress source of truth
project/wps/WP-XXX.md       frozen WP spec handed to Codex
project/wps/WP-XXX.log.md   Codex session id, review rounds, decisions, verdicts
project/wps/WP-XXX.prompt.md  last prompt sent to Codex (self-contained)
project/wps/WP-XXX.out.md   last Codex reply (from codex -o)
project/checks/WP-XXX.*     YOUR independent acceptance check (never inside a repo)
project/RESUME.md           checkpoint: the one file that survives a session dying
project/DECISIONS.md        append-only log of every call you made without the user
project/MORNING.md          the report the user reads over coffee — written at the end of the run
```
