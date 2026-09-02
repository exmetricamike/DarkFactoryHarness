# Backlog
Updated: 2026-09-02 | DONE 2/15

Scope: P0–P2 (net worth · cash flow + FX · 5–10 year forecast). P3/P4 deferred — see below.
`be` = `E0ApiEngine_Backend` · `fe` = `Frontend`.

## Order

| ID | Title | Repos | Depends | State | Codex session | Notes |
|----|-------|-------|---------|-------|---------------|-------|
| WP-001 | Backend environment + template strip | be | — | **DONE** `2aa266d` | 01a061e2 | `manage.py migrate` runs against Postgres and `pytest` is green on an app that is now only auth, invites, RBAC and a `Household`. |
| WP-002 | Finance schema, seed data, first migration | be | WP-001 | **DONE** `e14fe9a` | 01a06230 | Creating a household seeds its 8 asset kinds (canvas colours) and ~30 flow categories; every model in §3 exists with its constraints. |
| WP-003 | Valuation, FX, loan schedule + the golden fixture | be | WP-002 | TODO | — | `net_worth()` returns gross/debt/net for any past month from real `Valuation` rows, and the hand-computed fixture proves it. |
| WP-004 | Cash-flow services + `unexplained_change` | be | WP-003 | TODO | — | A month's income, expenses and transfers come back correctly classified, with the mortgage split into interest and principal. |
| WP-005 | Projection + balance forecast | be | WP-003 | TODO | — | Recurring rules and the loan schedule expand into a 10-year projection with a minimum, its month, and the first negative month. |
| WP-006 | Finance API foundation + demo household | be | WP-003 | TODO | — | `manage.py seed_demo` builds the canvas's household, and `/api/kinds/` + `/api/accounts/?period=` serve the sidebar already totalled. |
| WP-007 | Frontend strip, design system, shell, auth | fe | WP-001 | TODO | — | You can log in and land on the dark IBM Plex shell with a 308px sidebar — empty, but unmistakably this product. |
| WP-008 | Sidebar + dashboard | be, fe | WP-006, WP-007 | TODO | — | The dashboard renders: sidebar with values and stale marks, net-worth tile, allocation donut, cash tiles, and the "needs closing" prompt. |
| WP-009 | Cash-flow chart | be, fe | WP-004, WP-005, WP-008 | TODO | — | 24 months of stacked bars with the net-worth line, actuals solid and projections dimmed either side of a labelled divider. |
| WP-010 | Month close | be, fe | WP-004, WP-005, WP-008 | TODO | — | The ten-minute monthly ritual works end to end: prefilled balances, a flow checklist, the unexplained line, one wholesale save. |
| WP-011 | Account detail | be, fe | WP-008 | TODO | — | Any account opens the same page: value history, its flows, an editable valuation list, and net equity where a mortgage is linked. |
| WP-012 | Forecast + rules | be, fe | WP-005, WP-008 | TODO | — | The forecast page answers "will we run out of money", marks the low point, and lists the rules driving it. |
| WP-013 | Cash-flow ledger + settings | be, fe | WP-004, WP-008 | TODO | — | The full transaction ledger with filters and a category breakdown, plus settings for household, members, kinds, categories and FX. |
| WP-014 | Onboarding + empty state | be, fe | WP-010 | TODO | — | A brand-new user registers and reaches a dashboard with real numbers without ever seeing an empty screen that looks broken. |
| WP-015 | Clear the Django 7 deprecation warnings | be | WP-001 | TODO | — | The suite runs without warnings, so a real one is never lost in the noise: `EMAIL_BACKEND` migrated to MAILERS, `send_mail(fail_silently=)` replaced. (Found during WP-001; 5 warnings in kept template code.) |

State: `TODO` → `SPECCED` → `BUILDING` → `VERIFY` → `DONE` | `BLOCKED`
Codex-down only: `SPECCED-UNREVIEWED`, `DONE*` — both are debt, drain before starting new WPs.

## Milestones

- **M1 — the backend can value a household.** WP-001..003. The template is gone, the schema
  exists, and `net_worth()` returns the right three numbers for any month against a
  hand-computed fixture. Nothing is visible yet; everything downstream is unblocked.
- **M2 — every computation in the spec works.** WP-004..006. Flows classify, the mortgage
  splits, the forecast projects, and `seed_demo` builds the canvas's household so the numbers
  can be read out of the API by hand.
- **M3 — the dashboard is real.** WP-007..009. Log in and see the design: sidebar, totals,
  donut, cash tiles, the prompt, and the 24-month chart. **This is the first point the project
  looks like the thing it is.**
- **M4 — the product is usable.** WP-010. The month close is the spine; with it the family can
  actually run their finances. Everything before this is scaffolding for this screen.
- **M5 — the app is complete for P0–P2.** WP-011..014. The remaining screens, and a new
  household can get in from zero.

## Risk flags

- **WP-001 is the largest and the one most likely to overrun.** Renaming `Organization` →
  `Household` and the three roles cuts across models, `access.py`, `permissions.py`, every view,
  every serializer and the whole test suite at once, while half the suite is being deleted. It
  must end green or nothing after it can be verified. If it stalls, the fallback is to keep the
  template's names internally and alias at the API boundary — ugly, reversible, and it keeps the
  line moving.
- **WP-003's golden fixture is the highest-leverage artifact in the build** and the easiest to
  get subtly wrong. Its expected figures must be computed by hand and written down *before* the
  code runs, or every later test is just asserting that the code agrees with itself.
- **WP-009 carries decision D-14 item 4** — the one place the design canvas is being overridden
  in a way that changes how a chart looks. Its spec must put that to Codex explicitly rather
  than smuggling it through.
- **WP-013 is the most droppable.** If the night runs short, the ledger and settings pages are
  what to cut: no acceptance criterion depends on them, and the sidebar already covers creating
  a category (A10).

## Deferred / later

- **P3** — `Position`, `Quote`, `manage.py refresh_quotes`, `SUM_OF_POSITIONS` valuation. Needs a
  quote provider chosen; the source spec calls this its one genuinely hand-waved dependency.
- **P4** — CSV import + `CategoryRule`, the `net_worth_bridge` waterfall, PDF export, alerts.
- Light theme (D-15 committed to dark only).
- Redux/Zustand (D-10 — neither; server state per screen, view state in the URL).
