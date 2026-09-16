---
description: Where the project stands - backlog, repo state, next action
---

# /df-status

1. Read `active-project/BACKLOG.md` and the `roles` block of `harness.config.json`.
2. Per repo in `active-project/PROJECT.md`: `git -C <repo> status --short` and `git -C <repo> log --oneline -5`.
3. If a WP is `BUILDING`/`VERIFY`, read the tail of its `active-project/wps/WP-XXX.log.md`.

Report exactly this, nothing more:

```
Progress: <n>/<total> DONE  |  in flight: <WP-XXX state> | none
Backends: reviewer <profile-id> / implementer <profile-id>
Repos:    be <branch> <clean|N dirty files> @<sha>
          fe <branch> <clean|N dirty files> @<sha>
Blocked:  <WP-XXX — one-line reason> | none
Open spec items: <n>
Next:     <exact command to run>
```

If the working trees are dirty and no WP is in flight, say so plainly — that is drift and it needs the user's attention before the next build.
