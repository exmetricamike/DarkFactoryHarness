---
description: Move a previously archived project back into active-project/
---

# /reactivate-project <name>

1. Refuse if `active-project/` holds anything beyond the scaffolding
   (`intake/README.md`, `checks/README.md`, `wps/.keep`) — only one project can be
   active at a time. Tell the user to run `/archive-project` first.
2. Refuse if `projects-archive/<name>` does not exist. List the folders under
   `projects-archive/` so the user can pick the right one.
3. Move every file and folder from `projects-archive/<name>/` into `active-project/`,
   replacing the scaffolding files where the archive has its own copies.
4. Remove the now-empty `projects-archive/<name>/`.
5. Report the project is active again and the state its `BACKLOG.md` was left in.
