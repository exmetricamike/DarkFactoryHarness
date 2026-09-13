# Closed-out projects

Each subfolder here is a finished `active-project/` snapshot, put here by `/archive-project`.

Folder name: `<YYYY-MM-DD>-<slug-of-project-title>`, derived automatically from
`PROJECT.md` and the archive date. No manual naming needed.

**This is an archive, not working state.** During any harness run, only `active-project/`
is read or considered — never a folder under here. Open something in here only when the
user explicitly asks to look at, compare against, or reactivate a past project.

`/reactivate-project <folder-name>` moves a snapshot back into `active-project/`. It only
succeeds if `active-project/` is empty — only one project can be active at a time.
