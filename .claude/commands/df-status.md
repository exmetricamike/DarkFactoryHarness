---
description: Where the project stands - backlog, repo state, next action
---

# /df-status

1. Read `project/BACKLOG.md`.
2. Per repo in `project/PROJECT.md`: `git -C <repo> status --short` and `git -C <repo> log --oneline -5`.
3. If a WP is `BUILDING`/`VERIFY`, read the tail of its `project/wps/WP-XXX.log.md`.

Report exactly this, nothing more:

```
Progress: <n>/<total> DONE  |  in flight: <WP-XXX state> | none
Repos:    be <branch> <clean|N dirty files> @<sha>
          fe <branch> <clean|N dirty files> @<sha>
Blocked:  <WP-XXX — one-line reason> | none
Open spec items: <n>
Next:     <exact command to run>
```

If the working trees are dirty and no WP is in flight, say so plainly — that is drift and it needs the user's attention before the next build.
