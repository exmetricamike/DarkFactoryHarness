---
description: Checkpoint now and schedule a resume - use when credits are running low or you have to stop
argument-hint: [optional: reset time, e.g. "18:30" from /usage]
---

# /df-pause — checkpoint and stop

Load the `continuity` skill and follow section A.

Short version, in order:

1. Stop at the nearest safe boundary. Verified-but-uncommitted work gets **committed first**.
2. Write `active-project/RESUME.md` in the skill's format. Be specific about the mid-flight detail — future-you has no memory of this session and may be resuming at 4am with nobody around.
3. `CronCreate` one-shot at **now + 4 hours** (or the time in `$ARGUMENTS` if given), prompt `/df-resume`, off-minute. Never ask the user for a reset time.
4. Report: done / in-flight / resume time / `/df-resume` as the manual fallback. Then stop — no new work.

Do not spend remaining budget on a tidy summary. The checkpoint file is the deliverable.
