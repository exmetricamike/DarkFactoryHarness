# Lessons — what previous nights taught this harness

Cross-project memory. **This file is not project-specific and never leaves the harness repo** —
it travels to the next project, `project/` does not. Read it at the start of every phase command;
it is kept short on purpose.

## How to write a lesson

One line, in the phase section it applies to:

```
- **L-012** <the rule, imperative, general> ⟨<generic project shape>; <what it cost>⟩ hits:3 conf:high
```

- **Rule** — must be actionable on a project this harness has never seen. "Paste the real route
  signatures into the frontend prompt" is a lesson. "The `/orders` endpoint returns a cursor" is not.
- **Context in ⟨⟩** — the *shape* of the project it came from, never its identity: no client or product
  names, no repo paths, no proprietary domain terms. "a fintech dashboard SPA (React + FastAPI)",
  "a multi-tenant billing backend (Django)", "an internal ops tool with no test suite". Plus what the
  incident actually cost — fix rounds, a reverted WP, an hour of the night.
- **hits** — times this lesson has been applied or re-confirmed since it was written. Bump it when it fires.
- **conf** — `high` (seen on ≥2 projects, or once with unambiguous evidence) · `med` (once, clear cause)
  · `low` (a hunch that earned a line; delete it if the next project does not confirm it).

## How to keep it useful

- **Ceiling: 30 lessons.** Adding the 31st means merging two or deleting the weakest — lowest `hits`,
  oldest, or superseded. A file nobody can read at 2am teaches nothing.
- **Contradicted by experience → rewrite or delete it.** Do not stack a counter-lesson on top of a
  wrong one; there is one current answer per situation.
- **Graded by the user → that grade wins** over your own reading of the night.
- Never record here what belongs in `project/` (a decision about this product, a repo quirk,
  a WP follow-up). This file only holds things that will still be true on the next project.

---

## intake / plan
_(nothing yet)_

## spec
_(nothing yet)_

## build
_(nothing yet)_

## verify / run
_(nothing yet)_

## coordination (Codex, budget, continuity)
_(nothing yet)_

---

## Known friction — recurring pain with no rule yet

Where nights actually get lost, recorded before anyone knows the fix. A friction line that
recurs on a second project and finds a fix graduates into a lesson above; one that never
recurs gets deleted.

| # | Situation | Project shape | Seen | Cost so far |
|---|-----------|---------------|------|-------------|

---

## Candidates — this project, not yet distilled

Appended during the night, cheap and unfiltered. `/df-retro` promotes, merges or drops every
line here and leaves the section empty. Anything still sitting here at the start of a new
project is dropped — an undistilled candidate is noise, not memory.

