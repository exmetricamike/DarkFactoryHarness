# Decisions log

Append-only. Every call made without the user, plus the calls the user made at intake so the
morning conversation has one place to look. Grade these in the morning — the grades feed
`/df-retro`.

Format: `## D-NN — <question>` · **Decision** · **Why** · **Reversal cost** · **Grade:** _(user)_

---

## Phase 1 — `/df-intake`, 2026-09-02

### Made by the user

**D-01 — Is the frontend in scope?** Yes, both repos. The frontend follows the Claude Design
canvas linked from `project/intake/claude-design-prompt.txt`, which outranks `design-prompt.md`
on visual questions.

**D-02 — How far may the template be stripped?** Strip + reset migrations. No data to preserve.

**D-03 — Which phases?** P0–P2 (net worth · cash flow + FX · forecast). P3/P4 deferred.

**D-04 — §12 Q5/Q6/Q9?** All three as the spec recommends: Employment stays net-only, Business
stays a black box, `fiscal_year_start_month` is dropped.

**D-07 — Python environment?** A `.venv` inside the backend repo.

**D-08 — What does "the product runs" mean?** Postgres via `DATABASE_URL`, not SQLite.

### Made by Claude

## D-05 — Which manifest is authoritative, `pyproject.toml` or `requirements.txt`?
**Decision:** `pyproject.toml`. `requirements.txt` is deleted.
**Why:** The two describe different projects. `pyproject.toml` is `social-poster-backend` — the
template this repo actually is. `requirements.txt` is UTF-16-encoded and lists firebase-admin,
matplotlib, fpdf2, jazzmin and pytz: the fingerprint of an older, unrelated app whose inert
folders (`main_app/`, `users/`, `errors/`) are still lying around. Keeping both means every
future dependency question has two answers.
**Caveat found:** the spec's §5 claim that *"`django-pandas` is already a dependency"* is true of
`requirements.txt` and of the interpreter on PATH, but **false of `pyproject.toml`**. It is
therefore added explicitly rather than assumed.
**Reversal cost:** one file, recoverable from git.
**Grade:**

## D-06 — Sketch vs Claude Design canvas
**Decision:** The canvas supersedes `Layout.pptx` (the `findash.png` the design brief refers to).
The pptx stays as provenance only; its category labels — "Broad Money", "Cryptocurrency" — are
not binding.
**Why:** The canvas is the realised output of the brief the pptx seeded, and the user named it as
the thing to follow. The label mismatch is a non-issue by design: kinds are renameable rows.
**Reversal cost:** none — nothing is built on it yet.
**Grade:**

## D-09 — Postgres driver
**Decision:** `psycopg[binary]` (psycopg 3) in `pyproject.toml`.
**Why:** Django 6's recommended driver. `psycopg2` 2.9.12 exists in the global interpreter and
would also work, but the `.venv` installs from `pyproject.toml`, which should name exactly one.
**Reversal cost:** one line + a reinstall.
**Grade:**

## D-10 — Frontend state management
**Decision:** Neither Redux Toolkit nor Zustand. Both are dropped.
**Why:** The template ships both, which is already a smell. The finance app needs neither: by
design each screen is backed by exactly one endpoint (§6 — `/api/dashboard/` and
`/api/months/{period}/` are denormalised precisely so the client does no assembly), and the only
cross-screen state is `?period` and `?currency`, which §5.9 already requires to live in the URL.
A store here would be a second place for numbers to disagree with the server.
**Reversal cost:** adding one back is an `npm install` and a provider. Cheap, and the shape of the
API means it should never be needed.
**Grade:**

## D-11 — Frontend dependency pruning
**Decision:** Prune `package.json` to what the finance app imports — but **in the first frontend
WP, after the canvas is read**, not now.
**Why:** The template carries MUI, FontAwesome, Heroicons, Tabler, Lucide, react-icons, dnd-kit,
react-social-icons, framer-motion, vaul and more — several complete UI systems layered on each
other. Pruning is obviously right, but *which* primitives survive is decided by the Claude Design
canvas, so doing it before reading the canvas means guessing and then re-doing it. Keep for
certain: Next, React, Tailwind v4, `recharts`, `next-themes`.
**Reversal cost:** `npm install` restores anything cut.
**Grade:**

## D-12 — The run creates its own Postgres database
**Decision:** The build creates a fresh database (working name `familydashboard`) rather than
reusing `intensity`, the database named in the backend's existing `.env`.
**Why:** That `.env` belongs to the unrelated older project whose folders are still in the repo.
Its `DB_*` variables are not even read by `config/settings.py`, which only looks at
`DATABASE_URL`. Pointing the new app at someone else's database to save one `CREATE DATABASE` is
the kind of shortcut that destroys data at 3am.
**Reversal cost:** none. `DROP DATABASE` if unwanted.
**Grade:**

## D-13 — The design canvas
**Decision:** Read after `/design-login` and saved verbatim to `project/design-canvas.html`. Its
tokens, palette and per-screen layouts are extracted into `SPEC.md` §5.0 and §5.10.
**Why:** WP prompts must be self-contained (harness invariant 5) — Codex cannot reach the design
MCP. Extracting the tokens into the spec means a frontend WP prompt carries the palette and the
geometry inline, and the saved HTML is there when a detail is missing.
**Reversal cost:** none; re-pullable.
**Grade:**

## D-14 — Four conflicts between the canvas and the behavioural spec
**Decision:** Behaviour wins in all four; full table in `SPEC.md` §5.10.
1. The canvas's *"Autosaved as a draft."* label is deleted — `familydashboard.md` rules out
   autosave twice (§8.1.5, §11), and autosave against a wholesale PUT means a half-filled form
   overwrites the month with its own incomplete state.
2. The canvas draws a long-stale balance as a dashed **non-input** reading "carry". It becomes a
   real `<input>` styled to look identical until focused — otherwise an 11-months-stale house is
   permanently un-updatable, which breaks acceptance A5.
3. The empty state's *"Import a CSV"* button is omitted — CSV import is P4, out of scope.
4. Cash-flow bars stack by **source kind** (income) and **top-level category** (expense) in their
   own colours, not the canvas's two green/red tints.
**Why:** The user named the canvas the authority on *design*; `familydashboard.md` is the
authority on *behaviour*, and each of these four is a behavioural claim wearing a visual costume.
**Watch item:** (4) is the only one that visibly changes a chart. The canvas's legend shows two
swatches; eight kinds cannot be represented by two. The spec's argument — colour lives on the
`AssetKind` row precisely so pie, sidebar and bars cannot disagree (§7 of the source spec) —
outweighs the mock, but this is the call in this intake most worth a second opinion. **The
frontend chart WP must put it to Codex explicitly.**
**Reversal cost:** (1)–(3) are trivial. (4) is one chart component.
**Grade:**

## D-15 — Dark theme only
**Decision:** No light mode. `next-themes` dropped.
**Why:** The canvas is a single fully-resolved dark theme, and `design-prompt.md` explicitly
offered "light and dark themes, **or one theme committed to properly**". Building a light variant
means inventing a palette the designer did not, and doubling every colour decision for a
single-family app whose designer already chose.
**Reversal cost:** real but contained — the tokens are already centralised in §5.0, so a light
theme is a second token block, not a refactor.
**Grade:**

## D-16 — `AssetKind.colour` seed values come from the canvas
**Decision:** Seed the canvas's exact hexes (`SPEC.md` §5.0), not the design brief's colour names.
**Why:** Those hexes are what the donut, the sidebar swatches and the bars are drawn with. Seeding
anything else guarantees a mismatch the first time a household is created.
**Reversal cost:** one list in `finance/seed.py`.
**Grade:**

---

## Phase 2 — `/df-plan`, 2026-09-02

## D-17 — Backlog shape: 14 WPs, six backend foundations then vertical slices
**Decision:** Approved by the user as proposed. See `BACKLOG.md`.
**Why these three shaping calls, specifically:**
- **One schema, one migration (WP-002)** rather than three tranches by phase. The source spec
  calls `models.py` "one coherent schema"; splitting it P0/P1/P2 buys nothing and costs two
  extra migration rounds plus the chance of a half-built constraint.
- **`seed_demo` early (WP-006), not last.** It builds the canvas's exact household, so every
  frontend WP has data to render instead of hand-creating rows, and it exercises all three
  service modules before any UI exists.
- **Onboarding last (WP-014) despite being acceptance A1.** Its step 4 *is* the close screen
  (source spec §8.3), so it cannot precede WP-010, and `seed_demo` means nothing is blocked by
  its absence. Ordered by risk and unblocking, not by the order a user meets the screens.
**Considered and declined:** splitting WP-001 into delete-then-rename (offered; user kept it
whole), and cutting WP-013 up front rather than leaving it as the overnight cut line.
**Reversal cost:** the plan is a starting order, not a contract. Reordering, splitting or
merging overnight is expected — log it and update the backlog.
**Grade:**

## D-18 — WP-001 fallback if the rename overruns
**Decision:** If `Organization` → `Household` cannot be landed green, keep the template's
internal names and alias only at the API boundary, then continue. Do not stop the line for a
naming question.
**Why:** The rename crosses models, `access.py`, `permissions.py`, every view and serializer and
the whole test suite at once, while half that suite is being deleted — the single most likely
place for the night to stall. The alias is ugly and reversible in one later WP; a stalled
pipeline at 1am is neither.
**Reversal cost:** one rename WP, later, against a suite that is by then green and much smaller.
**Grade:**

---

## Open — needs the user

_None. `/df-spec WP-001` is unblocked._
