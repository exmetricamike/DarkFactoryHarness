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

- cand: `codex exec resume` rejects `--cd`/`-s`/`--approve-for-me` that `codex exec` accepts (CLI 0.152) → cost 1 wasted round; verify subcommand flags separately from the parent command's
- cand: a failed codex call leaves the previous round's `-o` file in place, and the stale reply reads as Codex repeating itself verbatim → `rm -f` the out-file before every call and check the exit code
- cand: WP spec cited the coordinator's own SPEC.md, which the implementer cannot read → inline every referenced section; a citation to a path outside the repo is a broken prompt
- cand: spec invented an endpoint (`DELETE`) the codebase never had, from reading the URL file without checking the view's methods → read the handler, not the route table
- cand: a permission class whose only endpoint is gated by a stricter class ships untested and its HTTP test passes for the wrong reason → unit-test the class directly when no route exercises it
- cand: Codex's sandbox blocked pip, so it ran the suite against an UNRELATED project's site-packages via PYTHONPATH and reported green → never accept a TESTS: line without re-running in the real env; check which interpreter it used
- cand: coordinator's own acceptance script failed 5/25 on first run, all its own bugs (Windows python.exe cannot read Git Bash /tmp paths) → parse JSON with grep in cross-toolchain checks, and always run the check once before trusting a PASS
- cand: `manage.py shell < script.py` runs an InteractiveConsole that swallows sys.exit and always returns 0 → judge such checks on printed output plus a sentinel line, never on exit code
- cand: acceptance check reported ALL PASS while one section could not fail → mutation-test every new check once (break an expectation, confirm FAIL) before trusting a green
- cand: pre-provisioning the venv before the implement call turned a PARTIAL into a clean DONE with 0 fix rounds → always provision the environment coordinator-side first
