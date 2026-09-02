# Project: Family Financial Dashboard

A private web app one family uses to track everything they own, owe, earn and spend, and to
project cash and net worth 5–10 years out. Updated by hand in ~10 minutes at each month end
via a single "month close" screen; the month is the atomic period and the past is always
editable. Not an accounting system — best-effort by design.

Built on an existing Django+DRF **template backend** (a stripped social-media planner) and an
existing Next.js **template frontend** from the same template pair. Both need the template's
domain removed and the finance domain added.

## Repos

| Role | Path | Branch | Language/Framework | Package manager |
|---|---|---|---|---|
| Backend | `C:\Users\micpa\Root\Development\FamilyDashboard\E0ApiEngine_Backend` | `master` | Python 3.14 · Django 6.0.4 · DRF 3.17 | pip (`pyproject.toml`) |
| Frontend | `C:\Users\micpa\Root\Development\FamilyDashboard\Frontend` | `master` (**zero commits**) | TypeScript · Next 16.2.9 · React 19.2.7 | npm (`package-lock.json`) |

Sibling reference material outside both repos: `..\Layout.pptx` (the dashboard sketch; its pie
and cash-flow chart are `ppt/media/image1.png` and `image2.png` inside it).

### Contract between them
REST/JSON over `/api/`, JWT bearer (`djangorestframework-simplejwt`), no shared type package —
the frontend hand-writes types in `lib/types.ts` against the backend serializers. Money crosses
the wire as **strings**, never JSON numbers (spec §9.1). CORS allows `localhost:3000`.
Two endpoints are deliberately denormalised, one per screen: `/api/dashboard/` and
`/api/months/{YYYY-MM}/`.

## Commands (per repo — exact, copy-pasteable)

Run from the repo root. **No `.venv` exists**; the Python on PATH
(`C:\Users\micpa\AppData\Local\Programs\Python\Python314\python.exe`) already has Django 6.0.4
and `django_pandas` importable, but **not `pytest`** — see Environment below.

| Repo | dev | test | lint | typecheck | build | migrate |
|---|---|---|---|---|---|---|
| Backend | `python manage.py runserver` | `python -m pytest` | `ruff check core config` | — | — | `python manage.py migrate` |
| Frontend | `npm run dev` (:3000) | `npx playwright test` (`e2e/`) | `npm run lint` | `npx tsc --noEmit` | `npm run build` | — |

Backend extras: `python manage.py seed_demo` (template demo data — will be replaced),
`python manage.py createsuperuser`.

## Conventions (inferred — cite the file each came from)

- **naming**: UUID PKs on every model via an abstract `Base(models.Model)` with
  `id`/`created_at`/`updated_at` (`core/models.py:13`). Apps are flat: `config/` project +
  one domain app.
- **views**: thin `APIView` classes, one module per resource under `core/views/`, wired by
  explicit `path()` entries in `core/urls.py` — no routers, no ViewSets, no `serializers` doing
  business logic (`core/views/org.py`).
- **services**: business logic in `core/services/*.py` as plain functions
  (`ai.py`, `billing.py`, `scheduler.py`, `transitions.py`).
- **tenancy/access**: `core/access.py` — `visible_*(user)` querysets and `get_*(user, pk)`
  helpers that raise DRF `NotFound` (404) on cross-tenant reads so existence never leaks.
- **permissions**: `core/permissions.py` — small `BasePermission` subclasses layered by role
  (`IsOrgUser` → `IsAdmin` → `IsAdminOrAgency`).
- **errors**: DRF exceptions (`NotFound`) for missing/forbidden; plain
  `Response({"detail": ...}, status=400)` for business rejections.
- **auth**: JWT, 60-min access / 7-day refresh (`config/settings.py`). `AUTH_USER_MODEL =
  core.User`, email as username, role on the user row.
- **tests**: `pytest` + `pytest-django`, `core/tests/test_<area>.py`, factory fixtures in
  `core/tests/conftest.py` (`make_org`, `make_user`, `auth_api`). ~106 tests in the template.
- **settings**: hand-rolled `.env` loader + `env()`/`env_bool()` helpers at the top of
  `config/settings.py`; every var has a working dev default. SQLite unless `DATABASE_URL`.
- **lint**: ruff, line-length 120, `select = ["E","F","I","UP","B"]` (`pyproject.toml`).
- **frontend styling**: Tailwind v4 + Radix primitives (shadcn-shaped), `next-themes` for
  light/dark, `recharts` for charts, `lucide-react` icons. Route groups: `app/(auth)/`,
  `app/dashboard/`. Shared helpers in `lib/` (`api.ts`, `format.ts`, `types.ts`, `ui.tsx`).
- **frontend state**: both `@reduxjs/toolkit` + `react-redux` **and** `zustand` are installed —
  the template uses both; the finance app should pick one.
- **frontend e2e**: Playwright, `e2e/smoke.spec.ts`, config in `playwright.config.ts`.

## Environment

- **Services**: SQLite dev DB (`db.sqlite3`, committed, holds template demo data). Postgres via
  `DATABASE_URL` (`psycopg2-binary` listed in the legacy `requirements.txt` only). No Redis, no
  Celery — the spec's only background job is `manage.py refresh_quotes` on cron (P3).
- **Ports**: backend 8000, frontend 3000.
- **Env vars**: `.env` present (not committed, holds real-looking values), `.env.example`
  committed. Template vars (`FAKE_LLM`, `LLM_*`, `STRIPE_*`, `PUBLISH_DRY_RUN`, `MEDIA_BACKEND`)
  become dead once the template domain is stripped.
- **Seed/fixtures**: template `seed_demo` command + `core/fixtures/guideline_templates.json`.
  The finance app seeds instead from plain Python lists in `finance/seed.py` (spec §3.8) —
  8 `AssetKind` rows + ~30 `FlowCategory` rows, applied on household creation.
- **TLS certs** for `runserver_plus` are committed (`localhost+2.crt/.key`, `localhost+3.pem`)
  but `django-extensions` is not in `INSTALLED_APPS`.

## Repo debt found during intake (facts, not opinions)

1. **Two contradictory manifests.** `pyproject.toml` is `social-poster-backend` (Django>=5.1,
   anthropic, openai, stripe, pytest, ruff). `requirements.txt` is UTF-16 and describes a
   *different, older* project (firebase-admin, matplotlib, fpdf2, jazzmin, django-pandas,
   psycopg2). The installed interpreter matches `requirements.txt` (Django 6.0.4), not
   `pyproject.toml`. The spec's claim that "`django-pandas` is already a dependency" is true of
   the installed env and of `requirements.txt`, **false of `pyproject.toml`**.
2. **`pytest` is not installed** in the interpreter on PATH, so the template's ~106 tests cannot
   currently run. Nothing was verified green during intake.
3. **Inert legacy folders** in the backend: `E0ApiEngineBackend/`, `main_app/`, `users/`,
   `errors/`, `global_utils/`, `scripts/`, plus `static/admin/` — none in `INSTALLED_APPS`,
   mostly `.pyc`. The template README says they can be deleted.
4. **The frontend repo has zero commits** and no `node_modules`. Nothing is baselined.
5. `README.md` points at `docs/social_poster_spec.md`, which does not exist in the repo.
6. `db.sqlite3` is committed despite `.gitignore` listing it.

## Decided at intake (see `SPEC.md` decision log D-01…D-11)

- **Both repos in scope.** Frontend follows the Claude Design canvas.
- **Strip the template + reset migrations.** Delete the social-poster domain and its tests, rename
  `Organization`→`Household` and the roles, drop all migrations and `db.sqlite3`, regenerate a
  clean initial migration. Keep auth, invites, RBAC, tenancy, throttling, media.
- **Target: P0–P2.** P3 (quotes) and P4 (CSV import, bridge) deferred.
- **Python: a `.venv` in the backend repo**, `pip install -e ".[dev]"`. The build's first WP
  creates it.
- **Runs on Postgres via `DATABASE_URL`.** Verified present: **PostgreSQL 18.3** on
  `localhost:5432`, user `postgres`, password `admin`, `psql` at
  `C:/Program Files/PostgreSQL/18/bin/psql`. The run creates its **own** database and never
  touches an existing one. `psycopg[binary]` goes into `pyproject.toml`.
- **Delete `requirements.txt`** (UTF-16, describes an unrelated older project). Add
  `django-pandas` and `psycopg[binary]` to `pyproject.toml` explicitly.

- **Frontend is dark-theme-only**, IBM Plex Sans/Mono/Serif, gold accent used for exactly two
  things. Design tokens and per-screen layouts are in `SPEC.md` §5.0/§5.10; the canvas itself is
  saved at `project/design-canvas.html`.

## Still uncertain

_Nothing. `/df-plan` is unblocked._
