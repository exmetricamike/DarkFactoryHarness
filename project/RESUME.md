# Resume checkpoint

Written: 2026-09-02, after WP-001.
Run: `/df-run`, unattended. Scope P0–P2, 15 WPs.

## Where the line is

**DONE 1/15.** WP-001 complete, committed, verified. Next runnable: **WP-002** (`TODO`, its only
dependency WP-001 is `DONE`). WP-007 is also unblocked and could run in parallel if the order
ever needs changing.

## Repo state

| Repo | Branch | HEAD | Clean? |
|---|---|---|---|
| backend `…/FamilyDashboard/E0ApiEngine_Backend` | `dark-factory/p0-p2` | `2aa266d` | yes, except untracked `docs/design-prompt.md` (the user's, left deliberately) |
| frontend `…/FamilyDashboard/Frontend` | `dark-factory/p0-p2` | `73adecd` (baseline only) | yes |
| harness `…/DarkFactoryHarness` | `main` | `22ad5b3` + uncommitted project docs | project/ docs dirty |

Baseline to diff the whole night against: backend `5ab2b54`, frontend `73adecd`.

## Environment (all verified working, no setup needed on resume)

- Postgres 18.3, `localhost:5432`, db **`familydashboard`**, user `postgres` / `admin`.
  `psql` at `C:/Program Files/PostgreSQL/18/bin/psql`.
- `DATABASE_URL` is set in the backend's gitignored `.env` (line 89). Do not edit `.env`.
- Backend `.venv` exists and is fully installed (Django 6.1, DRF 3.18, psycopg 3.3.5,
  django-pandas 0.6.7, pytest 9.1.1, ruff 0.16.5).
- Frontend has **no `node_modules`** yet — `npm install` is WP-007's first act.
- Codex CLI 0.152.0, authenticated.

## Commands that are known to work

```
cd <backend>
./.venv/Scripts/python.exe manage.py migrate
./.venv/Scripts/python.exe -m pytest              # 20 passed
./.venv/Scripts/ruff.exe check core config        # clean
./.venv/Scripts/python.exe manage.py runserver 8000 --noreload
bash project/checks/WP-001.sh                     # ALL PASS (needs the server up)
```

## Codex protocol reminders learned tonight

- `codex exec resume` takes **no** `--cd`, `-s`, `--approve-for-me`. `cd` into the repo first;
  pass `-c sandbox_mode="workspace-write" -c approval_policy="never"` for a build round.
- `rm -f` the `.out.md` before every call — a failed call leaves the previous round's reply and
  it reads as Codex repeating itself.
- **Codex's sandbox has no network.** It cannot `pip install` / `npm install`. Expect `PARTIAL`
  verdicts whose only defect is that; do the install yourself and re-run its commands. Never
  trust its `TESTS:` line — on WP-001 it ran green against a *different project's* site-packages.
- The Bash tool mangles backslashes inside quoted heredocs. Build prompt files by `cat`-ing
  pieces written with the Write tool, and avoid Windows paths inside heredoc'd Python.

## Standing instruction from the user — END OF RUN

Given 2026-09-02, mid-run: **"when done, merge all in main and commit and push all."**

This is explicit authorization to push, which the harness otherwise forbids. It applies **only
when the run is complete** (every WP `DONE` or `BLOCKED`, final integration pass done, morning
report written) — not per-WP. Execute in this order:

| repo | merge | remote (SSH verified working) |
|---|---|---|
| backend | `dark-factory/p0-p2` → `master` | `git@github.com:exmetricamike/FamilyDashboardBackend.git` |
| frontend | `dark-factory/p0-p2` → `master` | `git@github.com:exmetricamike/FamilyDashboardFrontend.git` |
| harness | already on `main` | `git@github.com:exmetricamike/DarkFactoryHarness.git` |

Note the two project repos use **`master`**, not `main`. Commit the harness's `project/` docs
first, then merge, then `git push origin <branch>` in each. Use a real merge commit
(`--no-ff`) so the night is one reviewable unit and stays revertible.

## Next action

`/df-spec WP-002` — finance schema, seed data, first migration. Ground it in the code as it
exists at `2aa266d`: `core/models.py` now holds `Base`, `Currency`, `Household`, `UserManager`,
`User`, `Invite` and nothing else, and `Currency` is imported from `core.models` by the new
`finance` app.
