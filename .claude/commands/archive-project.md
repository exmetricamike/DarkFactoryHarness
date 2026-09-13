---
description: Close out the active project and move it into projects-archive/
---

# /archive-project [name]

1. Refuse if `active-project/` holds no real content beyond the scaffolding
   (`intake/README.md`, `checks/README.md`, `wps/.keep`) — nothing to archive.
2. Derive the archive folder name: `<today's date, YYYY-MM-DD>-<slug of the project title
   in active-project/PROJECT.md>`. If `[name]` was given, use it verbatim instead.
3. `mkdir projects-archive/<name>`, then move every file and folder under `active-project/`
   into it — including `intake/`, `checks/`, `wps/` — using `git mv` for anything git
   tracks and a plain move for everything else.
4. Recreate the empty scaffolding in `active-project/`: `intake/README.md`,
   `checks/README.md`, `wps/.keep` — copy them back from the archive folder you just
   filled, don't rewrite them.
5. Report the archive path and remind the user `active-project/` is now empty and ready
   for `/df-intake`.

Never touch `LESSONS.md` — it is cross-project and stays at the repo root.
