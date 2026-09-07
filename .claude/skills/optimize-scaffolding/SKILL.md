---
name: optimize-scaffolding
description: Audit and refactor a project's CLAUDE.md/AGENTS.md and skill descriptions to cut always-loaded context. Use for "optimize scaffolding", "clean up CLAUDE.md", "reduce token cost of this project".
---

# Optimize Scaffolding

Goal: shrink the bytes this project injects into **every** request, so each turn costs less.

What counts as always-loaded, and therefore in scope:

- `CLAUDE.md` / `AGENTS.md` / `CLAUDE.local.md`, at root and in subdirectories.
- Every file they pull in with `@path` — `@` imports are **eager**, they load with the file that references them.
- `name` + `description` frontmatter of every skill in `.claude/skills/` and agent in `.claude/agents/` — this metadata preloads at session start; skill bodies do not.

Not in scope, but flag if large: MCP tool schemas, hooks output, plugin skills.

Target directory: the path given in the invocation arguments, else the current working directory.

## Step 1 — Inventory

Report path, line count and approx tokens (chars / 4) for every always-loaded file above, plus a total.
Rough targets: root `CLAUDE.md` under ~200 lines / ~5k tokens; whole always-loaded set under ~10k tokens.
Nested `CLAUDE.md` files load only when their subtree is touched, so count them separately from the root budget.

## Step 2 — Classify

Every section lands in one bin:

1. **Behavioral** — changes what the agent does, applies broadly, rarely changes (commit conventions, "never edit CHANGELOG.md", coding philosophy). Keep inline.
2. **Reference** — architecture notes, file-by-file walkthroughs, tool inventories, anything derivable from the code, anything only one kind of task needs. Move out.
3. **Dead** — stale references to removed files, duplication across CLAUDE.md/AGENTS.md/nested files, content already in git history or auto-generated files. Delete.

## Step 3 — Restructure (plan only, write nothing yet)

Move each bin-2 section out, then replace it with **one line naming the file as a plain path, never `@path`**.
An `@` reference re-imports the content eagerly and saves nothing — the whole point is that the agent reads the file only when the task needs it.

Choose the destination by shape:

- "How to do X" — repeatable procedure, checklist, workflow: `.claude/skills/<name>/SKILL.md`. Loads on invocation.
- "How X works" — passive reference: `.claude/docs/<topic>.md`. Loads when read.
- Applies only to one subtree: `<subtree>/CLAUDE.md`. Loads when that subtree is touched.

Also:

- Merge CLAUDE.md/AGENTS.md duplicates into whichever file the project's tooling actually loads; do not leave two authoritative copies.
- Trim skill and agent `description` frontmatter to one line that says when to invoke. Long descriptions cost on every request; the body is free until used.

## Step 4 — Add cost-discipline directives

Add a short section, only if the project has no equivalent already:

- Batch independent tool calls into one turn.
- Read narrow slices (line ranges, targeted grep), not whole files — tool output is permanent context and outweighs anything in this file.
- Never re-read a file just edited.
- Fix small bounded things inline; delegate to a subagent only for high-volume exploration whose raw output is not needed again.
- Keep responses proportional; no unrequested summaries or restated context.

Keep it under ~10 lines. It is durable behavior, so it belongs inline.

## Step 5 — Present the plan, wait

Show, before touching anything:

- Always-loaded token count now vs. projected.
- The bin table: `file:line → bin → destination`.
- A per-section summary of what moves where.

Do not paste the full new file contents into the reply — that is the same tokens twice.
Write on approval, then show the diff.
Never edit CHANGELOG.md or other auto-generated files; flag them instead.

## Step 6 — Verify

Report new token counts and the delta.
If it is a git repo, `git diff --stat` on the touched files.
Do not commit unless asked.

Tell the user the change takes effect on the next `/clear` or restart in that project: the loaded copy of CLAUDE.md is a snapshot in the request prefix, and editing it mid-session both does nothing and invalidates the session cache.
