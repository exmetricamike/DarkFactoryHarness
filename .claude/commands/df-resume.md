---
description: Pick up exactly where the last session stopped
---

# /df-resume — restore and continue

1. Read `active-project/RESUME.md`. If it is missing or says `Reason: none`, fall back to `/df-status` and stop — do not improvise a resume point.
2. **Verify the checkpoint against reality before trusting it.** Per repo: `git -C <repo> log --oneline -3` and `git -C <repo> status --short`. Compare with the recorded HEAD sha and clean/dirty state.
   - Matches → proceed.
   - HEAD moved, or dirty files the checkpoint doesn't mention → something ran in between. Read the diff and the log and account for it yourself. If it is Codex's leftovers from the interrupted WP, fold it into that WP's next round. If it is coherent work, keep it and note it. If it is junk that breaks the build, revert it. **Do not wait for the user** — this resume probably fired at 4am from a cron. Log what you found and what you did in `active-project/DECISIONS.md`, then continue.
3. Read `active-project/BACKLOG.md` and the in-flight WP's `active-project/wps/WP-XXX.log.md`.
4. Honour the **Do not redo** list. Repeating a finished Codex review round wastes both budgets.
5. If the checkpoint was `Reason: codex-down`, load the `continuity` skill and drain the debt in section B's "When Codex returns" order before any new WP.
6. Execute the **Next action, verbatim** line.
7. Once you are actually moving again, replace `active-project/RESUME.md` with `Reason: none` so a stale checkpoint can never be resumed twice.
8. **If the checkpoint came from a `/df-run` pause, re-enter `/df-run`** and keep going until the backlog is done. Load the `night-shift` skill first. Resuming into idleness wastes the whole point of the wake-up.

Report in three lines: where we stopped, what changed since, what you are doing now.
