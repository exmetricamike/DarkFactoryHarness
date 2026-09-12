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
- **L-001** Before writing a numeric budget, row-count, or capacity assumption into a spec, verify it against the real current state (actual query sites, actual seeded row counts) — don't carry it forward unchecked from an earlier planning document. ⟨a multi-night unattended full-stack rebuild, Django/DRF + Next.js; 2 review rounds and one spec written against a fixture with zero rows in the domain it described⟩ hits:1 conf:med
- **L-002** When a spec states a global constraint, name its exemptions in the same sentence — a rule and its exception stated separately will contradict each other the moment someone reads only one of them. ⟨same project shape; one review round spent resolving a spec that both banned and required the same thing⟩ hits:1 conf:med
- **L-003** After amending a spec in response to review, grep the whole document for the value you just changed — a fix applied in one place often leaves the old literal standing in three or four others. ⟨same project shape; a full extra review round caused entirely by the coordinator's own leftover literals⟩ hits:1 conf:med
- **L-004** When a spec needs an exact formula whose correctness depends on a library's internal semantics (e.g. where a chart scale maps a category to a pixel), state the intent in words and let implementation/browser verification settle the arithmetic — don't pin a formula built on an assumption about the library that might be wrong. ⟨same project shape; a wrong pixel formula was implemented faithfully and reproduced a defect three prior review rounds had already removed⟩ hits:1 conf:med
- **L-005** Treat an approved design mock's own copy/microcopy as a claim to verify against the data model, not text to paste verbatim — a mock can assert something ("every figure is last month's") the underlying data cannot actually guarantee. ⟨same project shape⟩ hits:1 conf:med

## build
- **L-006** When a permission/guard class is only ever reached through a route gated by an even stricter check, no route's own test can exercise it — write a direct unit test against the guard class itself. ⟨same project shape⟩ hits:1 conf:med
- **L-007** When a feature adds a backwards or relative lookup keyed by date/index (e.g. reading `period − 1`), re-audit every other lookup keyed by the same axis — the new read can make a previously-unreachable gap reachable on day one. ⟨same project shape⟩ hits:1 conf:med
- **L-008** A dev server left running from earlier in the session does not necessarily reflect code a later Codex round just wrote — restart it and hit a live endpoint before trusting its response, especially across fix rounds or resumed sessions. ⟨same project shape; caused two false "still broken" verification results in one night before the server was restarted⟩ hits:2 conf:med

## verify / run
- **L-009** When an acceptance check bans something a later, already-planned WP will legitimately introduce, narrow the assertion to name the WP that sanctions the exception rather than letting a correct future change fail an old check. ⟨same project shape⟩ hits:1 conf:med
- **L-010** Automated fixtures and e2e suites typically only ever exercise one seeded, happy-path state. Deliberately drive at least one out-of-fixture state (an empty/unpopulated period, an un-seeded parameter) as part of every UI WP's verification — this is where real defects hide that a green suite walks straight past. ⟨same project shape; a real display defect (a parent row and its child rows making contradictory claims about the same absent data) was caught only this way, after every automated check had passed⟩ hits:2 conf:med
- **L-011** If a frontend auto-attaches a bearer token to every request, exclude public/auth endpoints (register, login, password reset) from it explicitly, and verify by testing with a deliberately stale/invalid token already in storage — not just a clean session — since an auth-gated framework can 401 a public endpoint before its own permission check ever runs, permanently locking a user out. ⟨same project shape⟩ hits:1 conf:med
- **L-012** A green run does not prove the code path under test actually executed — a test can silently skip when its dependency is absent, a REPL can swallow an error and still hit a later print statement, or a check can be structurally incapable of failing. Mutation-test any new verification script once (break the thing it's supposed to catch, confirm it goes red) and confirm the services it depends on were actually live, before trusting a PASS from it. ⟨same project shape; caught a script that reported "ALL PASS" on a section that could never have failed, and a shell-piped script that silently swallowed errors after an unblanked line and still printed success⟩ hits:2 conf:med

## coordination (Codex, budget, continuity)
- **L-013** Commit each WP once it's independently verified before starting the next build in the same repo — starting the next build on top of an uncommitted previous one mixes two WPs into one working tree and can make one WP's failure look like it belongs to the other. ⟨same project shape⟩ hits:1 conf:med
- **L-014** Never fully overwrite a persistent log/memory file that is the only record of past work (especially one that lives outside version control) — always append/edit it. An overwrite where an edit was meant destroys history with no way back. ⟨same project shape; a WP's own review-round history was permanently destroyed this way⟩ hits:1 conf:med
- **L-015** When a spec review needs to verify a contract (an API shape, a serializer's fields) that lives in a different repo than the one being changed, give the reviewer read access to the repo that owns the contract, not just the one being modified — otherwise a factual claim about the other side of the boundary goes unverified. ⟨same project shape; found a BLOCKER (a serializer silently dropping a field) only because the reviewer could read the other repo, then reused the same setup deliberately on the next WP⟩ hits:1 conf:med
- **L-016** When polling an agent's log file for a completion sentinel, match the concrete value you expect, not a line shape that could also appear in the prompt's own echoed template — a prompt that shows the required output format can itself satisfy a loose match. ⟨same project shape⟩ hits:1 conf:med
- **L-017** A CLI tool's flag compatibility can differ between its subcommands (e.g. a `resume`/`fork` subcommand vs. the main command) and can change between minor versions. Re-check `--help` for the exact subcommand you're calling — especially right after noticing a version bump — rather than assuming a previously-working invocation still works. ⟨same project shape; hit twice in one project as the CLI moved versions mid-run⟩ hits:2 conf:med

---

## Known friction — recurring pain with no rule yet

Where nights actually get lost, recorded before anyone knows the fix. A friction line that
recurs on a second project and finds a fix graduates into a lesson above; one that never
recurs gets deleted.

| # | Situation | Project shape | Seen | Cost so far |
|---|-----------|---------------|------|-------------|
| 1 | Browser-automation clicks (both coordinate-based and element-ref-based) intermittently fail to register with no visible error, silently no-op'ing a form submit | a Next.js frontend verified via `claude-in-chrome` | 1 project | One wasted manual-verification attempt; no real impact since the automated e2e suite already covered the same case and passed |

---

## Candidates — this project, not yet distilled

Appended during the night, cheap and unfiltered. `/df-retro` promotes, merges or drops every
line here and leaves the section empty. Anything still sitting here at the start of a new
project is dropped — an undistilled candidate is noise, not memory.

_(none — distilled by /df-retro on 2026-09-12)_
