# Family Financial Dashboard — Specification

Status: **ACTIONABLE · OPEN ITEMS: 0**

Source material, in precedence order:
- `project/intake/familydashboard.md` — the behavioural spec, v0 draft, 1952 lines. **The authority
  on domain model, computation rules, API surface and workflows.** Cited below as `[src: fd §x.y]`.
- `project/intake/design-prompt.md` — the frontend design brief. Authority on visual intent,
  screen inventory and the demo figures. Cited as `[src: dp]`.
- `project/intake/claude-design-prompt.txt` — pointer to a Claude Design canvas
  (`Family Financial Dashboard.dc.html`). **Authority on the frontend's actual visual design**,
  outranking `design-prompt.md` where they differ, since it is the realised output of that brief.
- `..\Layout.pptx` (outside intake) — the original PowerPoint sketch `design-prompt.md` refers to
  as `findash.png`. Its pie and cash-flow chart are `ppt/media/image1.png` / `image2.png`.
  Superseded by the Claude Design canvas; kept for provenance.

This file is a consolidated restatement, not a replacement. Where a WP spec needs the full
argument for a decision, read the cited section of `familydashboard.md` — it carries the
reasoning, and the reasoning is why the design holds.

---

## 1. Product summary

A private web app for **one family** to track everything they own, owe, earn and spend, and to
project cash and net worth 5–10 years out. Answers three questions at any date, past or future:
what do we own and owe; what came in and went out; what will come in and go out. `[src: fd §1]`

Updated by hand in about **ten minutes at each month end**. One or two adults, once a month.
Not a consumer fintech app, not a trading terminal — "a well-made domestic instrument, a good
thermostat, a nice ledger book". Calm, precise, quietly substantial. Nothing gamified. `[src: dp]`

Explicitly not: tax filing, budgeting/envelope enforcement, multi-family SaaS, trade execution,
accounting-grade double-entry. `[src: fd §1]`

### The six principles that carry the design `[src: fd §2]`

| | |
|---|---|
| **P1** | One `Account` table, not one per asset class. Discriminated by an `AssetKind` **row**. |
| **P2** | NAV comes from `Valuation` rows. Transactions never change NAV. |
| **P3** | One signed `Transaction` with optional `from_account`/`to_account`. `NULL` = the outside world. **Income/expense only when exactly one side is internal**; both internal = transfer. |
| **P4** | Projections are computed from `RecurringRule` rows in memory, never stored. |
| **P5** | Cash flow and value change are different things and are never mixed. A revaluation is not income. |
| **P6** | The month is the atomic period and the past is always editable. No locking, no audit trail, no soft deletes, no forced reconciliation. |

Every simplification in §10 follows from P6.

---

## 2. Actors & permissions `[src: fd §9.2]`

Three roles on `User`, all household-scoped. The household is the **only** ownership unit —
no per-member aggregates, no `OwnershipStake`, no per-account permissions.

| Role | Can |
|---|---|
| `OWNER` | everything, plus invite/remove members and edit household settings |
| `EDITOR` | read and write all financial data |
| `VIEWER` | read everything, write nothing |

- Scoping lives in `access.py`; a cross-household read raises `NotFound` (**404, never 403**) so
  existence never leaks.
- `VIEWER` is one DRF permission class: `request.method in SAFE_METHODS or user.role != VIEWER`,
  enforced server-side. The frontend hides write affordances as a courtesy, never as the control.
- Invites, password reset and JWT refresh come from the template unchanged.

---

## 3. Domain model `[src: fd §3]`

All models inherit the template's `Base` (UUID PK, `created_at`, `updated_at`).
Money = `Decimal(18,2)`. Quantities and rates = `Decimal(28,10)`. Financial dates are
`DateField` — dated, not timestamped. `Currency` is a two-value `TextChoices` enum
(`EUR`, `USD`) with a DB `CheckConstraint` on every currency field.

### 3.1 `Household` + `User` (renamed from the template's `Organization`)
```
Household(Base)  name, base_currency (default EUR), default_inflation_pct (default 2.0)
User(AbstractUser)  household FK, role OWNER|EDITOR|VIEWER
```
**Decision D-04:** `fiscal_year_start_month` is **not** carried over — it was the last piece of
speculative schema and nothing reads it. `[src: fd §12 Q9]`

### 3.2 `AssetKind` — a per-household table, seeded `[src: fd §3.2.1]`
```
AssetKind(Base)  household FK, slug, label, colour, display_order,
                 has_nav (default True), is_liability (default False), is_liquid (default False)
                 unique_together (household, slug)
```
Every household owns its **own eight rows**, copied from a template list on creation. A user
creating "Ancient Coins" from the sidebar produces the same row shape as "Real Estate".

> **The rule that makes user kinds safe: no function in `finance/services/` may branch on a slug.**
> Liability treatment reads `kind.is_liability`. Flow-only reads `kind.has_nav`. Cash tiles and
> `balance_forecast` read `kind.is_liquid`. Enforced by a grep test (§9).

Creation exposes **two fields**: a name, and one radio (*something you own* / *something you owe*).
`has_nav` and `is_liquid` are editable in settings afterwards, never at creation. `colour` is
auto-assigned by `display_order`; `slug` is derived. Deletion is `PROTECT`.

### 3.3 `Account` `[src: fd §3.2.2]`
```
Account(Base)  household, name, kind FK(PROTECT), currency, institution,
               valuation_method MANUAL|SUM_OF_POSITIONS|AMORTIZED|NONE,
               linked_account self-FK (null), closed_on (null), details JSON,
               csv_mapping JSON (P4 only), notes
```
- `closed_on` is the sold-house fix: `nav()` returns 0 for periods after closure, so a closed
  account still contributes correctly to every historical month and nothing is rewritten.
  `[src: fd §3.2.4]`
- `linked_account` exists for exactly one thing: a debt account naming what it secures, so the
  property page can show net equity. **Not** a sidebar nesting mechanism. `[src: fd §3.2.6]`
- **Never reinstate** `sign`, `include_in_net_worth`, `is_active`, `opened_on` or
  `display_order` — all five were deleted with reasons. `[src: fd §3.2.3]`
- `details` is validated against a per-slug allowlist in `finance/seed.py`; unknown keys are
  rejected, nothing else is coerced. **If a service function reads it, it is a column, not a JSON
  key.** No `jsonschema` dependency. `[src: fd §3.2.5]`

### 3.4 `Valuation` — the month-end snapshot `[src: fd §3.3]`
```
Valuation(Base)  account FK, period DateField, value Decimal, source MANUAL|QUOTE|IMPORT, note
                 unique_together (account, period)
```
`period` is the **first day of the month**, meaning "value as at the end of that month" — chosen
because it is exactly what Django's `TruncMonth` produces, so flows and NAV join with no date
maths. Normalised in `save()`. Liabilities store `value` **positive** and are negated at
aggregation. Re-entering a month is `update_or_create`. **Staleness is derived, never stored.**

### 3.5 `FxRate` — one dated series `[src: fd §3.4.1]`
```
FxRate(Base)  as_of DateField unique, usd_per_eur Decimal(28,10), source MANUAL|IMPORT|FEED
```
`fx.convert(amount, from, to, on_date)` uses the rate whose `as_of` is nearest-earlier. ~15 lines,
dict cache per request. **A date with no rate at or before it raises** — never assume 1.0.
Core from P1: an aggregate summing EUR and USD without it is meaningless, not approximate.

### 3.6 `Position` / `Quote` (P3 — out of scope this run) `[src: fd §3.4.2]`
Not built in this run. If built later: `Position` carries **no currency** (the `Quote` does),
there is no security master, and positions are not a trade log. `valuation_method =
SUM_OF_POSITIONS` therefore has no implementation this run and must not be offered in the UI.

### 3.7 `FlowCategory` `[src: fd §3.5.2]`
```
FlowCategory(Base)  household FK, name, parent self-FK (null), colour
```
**One level of nesting, enforced in `clean()`.** `direction` and `is_recurring_expectation` were
deleted and must not come back — categories are genuinely bidirectional (insurance is an expense
until the claim pays), and recurrence is what `RecurringRule` is for. The picker orders by
usage frequency in the current direction instead of filtering.

### 3.8 `Transaction` `[src: fd §3.5.4]`
```
Transaction(Base)  household, date (a real day), amount (always > 0), currency,
                   from_account (null), to_account (null), category (null),
                   status ACTUAL|PROJECTED, source MANUAL|IMPORT|RULE, rule FK (null),
                   description, external_id (P4)
                   CHECK: from_account IS NOT NULL OR to_account IS NOT NULL
                   unique_together (household, external_id) where external_id != ""
```
Direction comes from the account pair, never from a sign or a flag. **A category is required when
either side is `NULL`, optional otherwise** — enforced in the serializer, not the DB, so an
import can land uncategorised and be fixed later. `[src: fd §3.5.1]`

Hard delete, no soft-delete column. No split transactions — one payment across two categories is
two rows.

### 3.9 `RecurringRule` `[src: fd §3.6]`
```
RecurringRule(Base)  household, name, amount, currency,
                     from_account, to_account, category,
                     every_n_months (default 1, 0 = one-off),
                     start_period, end_period (null), escalation_pct (default 0)
```
`every_n_months` replaces `frequency` + `interval` + `day_of_month` + `weekday` + `is_active`.
Expansion is integer arithmetic on `month_index = year*12 + month` — **no date is ever
constructed**, so the "31 January + 1 month" bug cannot exist. `WEEKLY` is deliberately gone.

### 3.10 `LoanTerms` — one generator, three consumers `[src: fd §3.6.3]`
```
LoanTerms(Base)  account O2O, principal, annual_rate, term_months,
                 first_payment_period, payment_from_account FK, payment_type ANNUITY|LINEAR
```
`schedule()` yields `(period, payment, interest, principal_part, balance_after)`. The last row
absorbs rounding so the loan ends at exactly zero. Consumers: `nav()` for `AMORTIZED`, the
interest/principal split in `cash_flow()`, and `project()`. A loop, not the closed-form formula.

> **The trap: never create a `RecurringRule` for a loan payment.** `LoanTerms` already emits that
> stream; a rule beside it double-counts the mortgage in every forecast. The rule form must
> exclude `AMORTIZED` accounts from its `to_account` picker, and a test asserts none does.

`INTEREST_ONLY` and `BULLET` are not built. Fixed rate only.

### 3.11 Seed data `[src: fd §3.8]`
Plain Python lists in `finance/seed.py`, applied once on household creation — **data, not
migrations**. Eight `AssetKind` rows:

| slug | label | colour | has_nav | is_liability | is_liquid |
|---|---|---|---|---|---|
| `CASH` | Cash | green | ✓ | | **✓** |
| `EMPLOYMENT` | Employment | grey | | | |
| `EQUITY` | Equities | sky | ✓ | | |
| `DEBT` | Debt | red | ✓ | **✓** | |
| `REAL_ESTATE` | Real Estate | pink | ✓ | | |
| `CRYPTO` | Crypto | amber | ✓ | | |
| `METALS` | Gold | yellow | ✓ | | |
| `BUSINESS` | Business | violet | ✓ | | |

Plus ~30 `FlowCategory` rows in two levels (Income / Housing / Living / Transport / Health /
Education / Leisure / Financial). **No `Loan interest` category and no `Transfer` category** —
both are synthetic at read time, and seeding them would create rows a user can delete out from
under the aggregation. Every seeded row is an ordinary user row: renameable, deletable.

---

## 4. Computation rules — this is the actual product `[src: fd §4]`

Pure functions in `finance/services/` over querysets. No logic in views. No model instances leak
into serializers.

### 4.1 `nav_matrix` is the primitive; `nav()` is a one-line wrapper `[src: fd §4.1]`
This inversion is deliberate: the dashboard needs ~20 accounts × 24 months, and a `nav()` in a
nested loop is 480 queries. Making the bulk form primitive means the slow version cannot be
written by accident. Whole history in one query, reduced in Python with a dict — no `DISTINCT ON`
(ties us to Postgres), no window functions.

Resolution order for one `(account, period)`:
1. `closed_on` set and `period > month_of(closed_on)` → **0**
2. An explicit `Valuation` for **that exact month** → its value. **This outranks every method.**
3. Otherwise by `valuation_method`: `MANUAL` → latest valuation `period <= M`, carried forward and
   marked stale · `SUM_OF_POSITIONS` → Σ qty × last quote in the month (P3) · `AMORTIZED` →
   the schedule's `balance_after` · `NONE` → 0.

Cash is `MANUAL` like everything else — at month end you type the balance you can see. Deriving
it from the transaction stream would let one unrecorded expense silently corrupt the headline
number. `valuation_method = TRANSACTIONS` was removed and must not return.

### 4.2 Balance-sheet aggregates `[src: fd §4.2]`
- `net_worth(household, period, currency)` → `{gross_assets, total_debt, net}` — all three, since
  a net figure alone hides whether a household is wealthy or merely leveraged.
- `allocation(household, period)` → per-kind slices of **gross assets only**, in `display_order`,
  with label and colour from `AssetKind` so the pie can never disagree with the sidebar. Debt is
  shown separately, never as a negative slice. The **top-6 + "Other" rollup is presentational** —
  the API returns every kind.
- `net_worth_history(household, from, to, currency)` — the same `nav_matrix`, summed per month.

### 4.3 `cash_flow(household, from, to, group_by, depth=1, currency)` `[src: fd §4.3]`
Buckets by `TruncMonth(date)`. Income = exactly one internal side, inbound. Expense = exactly one
internal side, outbound. Transfers excluded from both, available as their own series.
`group_by ∈ category | account | kind`. Three behaviours belong **here**, not in callers:

1. **Amortised-account transfers are split** against that month's schedule into an interest
   expense and a principal transfer, with any excess over the scheduled payment counted as
   principal. The interest half has no `Transaction` row, so it reports under a **synthetic**
   `Loan interest` bucket. `[src: fd §3.5.3]`
2. **Uncategorised external flows** bucket under a synthetic `Uncategorised` group rather than
   being dropped — discarding them would make the totals lie.
3. **Conversion happens once per (bucket, currency) subtotal**, at the end. Never per transaction,
   never on a mixed-currency total. Sum within a currency, convert each subtotal, then add.

### 4.4 Where actuals stop and projections start `[src: fd §3.5.5]`
> **Rules generate only for months strictly after the current one.** The current month and every
> month before it are actuals only.

No matching of actuals to rule instances, no heuristics, no state. A past month never recorded
shows **zero, not an estimate** — showing rule output in the past is a lie about history.
The `rule` FK stays for provenance and for the close-screen prefill; no aggregate depends on it.

*(This replaced an earlier `last_closed_period` boundary that fired as soon as one balance was
typed, silently truncating that month's cash flow. Do not reinstate it.)*

### 4.5 Forward-looking `[src: fd §4.4]`
- `project(household, from, to, real=False)` — expands rules and `LoanTerms` schedules, applying
  each rule's `escalation_pct` compounded from `start_period`
  (`amount × (1 + pct/100) ** floor(months_since_start / 12)`). Returns the same shape as
  `cash_flow`. `real=True` discounts by `default_inflation_pct` — a display transform at the very
  end, applied to nothing stored.
- `balance_forecast(household, to, real=False)` — the question the forecast exists to answer.
  Starts from **cash NAV, a balance**, not a sum of flows, so an incompletely recorded month
  cannot corrupt it. Reads `kind.is_liquid`, never a slug. **Transfers count here and nowhere
  else** — `ING → Portfolio1` is not an expense but certainly reduces spendable cash. Returns the
  series plus `min_balance`, `min_month`, `first_negative_month`.

### 4.6 The bridge, and the one real check `[src: fd §4.5]`
`revaluation(a, M) = ΔNAV(a, M) − net_flow_into(a, M)` is a **definition, not an invariant** —
revaluation is the residual, so subtracting it from both sides tests nothing. Its value is
display: `net_worth_bridge()` returns the waterfall "net worth Aug → +net cash flow → +market
movement → net worth Sep" (P4 — out of scope this run).

`unexplained_change(household, period)` **is** checkable, and is narrower than it first looks:
- computed **only over accounts whose kind `is_liquid`** — a bank balance moves only because
  money moved; for a portfolio the same residual is just the market doing its job;
- **each account differenced in its own currency, only the residual converted** — otherwise a 1%
  euro move reports €90 of missing money every month and the diagnostic gets ignored;
- only accounts holding a valuation in **both** M and M−1 are included.

Shown as "unexplained −350". A nudge. It never turns red, never blocks a save, and is never
auto-corrected with a balancing entry. **On the golden fixture it must be exactly zero.**

### 4.7 Query budget `[src: fd §4.6]`
The dashboard is fixed-cost: accounts+kinds, valuations, transactions, rules, loans, fx = **six
queries** (eight with positions/quotes at P3). `assertNumQueries` on `/api/dashboard/` and
`/api/months/` is the regression test.

---

## 5. Surfaces / screens

**The Claude Design canvas is the authority on visual design**, saved locally as
`project/design-canvas.html` (pulled 2026-09-02 from Claude Design project
`31b75a0e-9ad2-4241-973a-ddfacc2b5b5d`). It holds six artboards: `01 Dashboard`, `02 Month close`,
`03 Account detail`, `04 Forecast`, `05 Empty state`, `06 Narrow dashboard`. Its `<script
type="text/x-dc">` block carries the exact demo data and the computed style objects.

What follows is the behavioural contract each screen must satisfy. Where the canvas differs on
layout, type, colour or component choice, **the canvas wins**. Where it differs on *behaviour*,
`familydashboard.md` wins — the four places that happens are listed in §5.10.

### 5.0 Design system (extracted from the canvas)

**A committed dark theme.** No light mode. `next-themes` can be dropped.

| Token | Value | Used for |
|---|---|---|
| `--bg-page` | `#0a0a0c` | the canvas behind everything |
| `--bg-shell` | `#14161a` | screen shell / content area |
| `--bg-sidebar` | `#0c0d10` | the sidebar, darker than the content |
| `--bg-card` | `#1a1d21` | every card and panel |
| `--bg-input` | `#0f1114` | input fields |
| `--bg-row-pending` | `#191b1f` | an unconfirmed flow row |
| `--border` | `#23262b` | screen and sidebar edges |
| `--border-card` | `#262a30` | card edges, section rules |
| `--border-hair` | `#1c1f24` | row separators inside a list |
| `--border-input` | `#2e3238` | inputs, secondary buttons |
| `--border-hover` | `#4a5057` | hover on secondary controls |
| `--text` | `#e7e5e0` | primary figures and headings — warm off-white, not pure white |
| `--text-strong` | `#dcdad5` | row labels |
| `--text-mid` | `#c7c5c0` / `#a9a7a2` | secondary labels |
| `--text-dim` | `#8d9399` / `#9aa0a7` | captions, axis labels |
| `--text-faint` | `#6b7178` / `#5d646b` | notes, stale marks, tick labels |
| `--accent` | `#d0a45a` (hover `#e0b970`) | **the "needs closing" prompt and the forecast's low point — nothing else** |
| `--accent-bg` / `--accent-border` / `--accent-text` | `#1c1a15` / `#4a3d21` / `#f0e3c8` | the prompt's panel |
| `--pos` | `#88cfae` | positive amounts |
| `--neg` | `#cd7361` (child rows `#b8705f`) | debt figures and negative amounts |

**Category colours** — these are the seeded `AssetKind.colour` values (§3.11), and the canvas is
the source of the hexes:

| Kind | Hex | | Kind | Hex |
|---|---|---|---|---|
| Cash | `#5cb98c` | | Crypto | `#d59a52` |
| Employment | `#7c848c` | | Gold | `#c9b25a` |
| Equities | `#57a5d8` | | Business | `#9b86d4` |
| Debt | `#cd7361` | | *(user kinds, e.g. Coins)* | `#63b0b6` |
| Real Estate | `#cf6f96` | | | |

**Typography** — IBM Plex, from Google Fonts:
`IBM Plex Sans` 400/450/500/600 for prose and labels · **`IBM Plex Mono` 400/450/500 for every
number, without exception** · `IBM Plex Serif` italic for exactly one line (the empty state's
"Nothing here yet — which is correct."). Body carries
`font-feature-settings:"tnum" 1, "lnum" 1`; inputs and buttons inherit `tnum`.

**Numbers**: negatives use the true minus `−` (U+2212), never a hyphen. "No NAV by nature" is the
em dash `—` (U+2014) in `--text-faint`. Whole units, `en-US` grouping, no cents. Every numeric
column is `text-align:right` with a fixed `min-width` (sidebar 78px, flow amounts 82px).

**Geometry**: desktop artboard 1440px, narrow 420px. Sidebar 308px. Radii 8px (screen) / 6px
(card) / 5px (tile, prompt) / 4px (button) / 3px (input, swatch). A category swatch is a 6×6px
square with `border-radius:1px` — never a circle, never an icon.

**Staleness** is a mono `3m` / `11m` label in `--text-faint` with a **dotted bottom border**
(`1px dotted #3d434a`) and `title="carried forward"` — quieter than the `·` dot §5.1 describes,
and the canvas's version is the one to build.

**Buttons**: primary is `#e7e5e0` on `#14161a` text; the prompt's CTA is the only `--accent`
button in the app; secondary is transparent with a `--border-input` outline. "+ Add category" is
a full-width **dashed** outline button pinned to the sidebar's bottom.

### 5.1 The sidebar `[src: fd §7.1]`
Two levels, never three: **kind → account**, ordered by `AssetKind.display_order` then account
name. Rendered from `GET /api/accounts/?period=` with kind rows **already totalled server-side** —
the client does no arithmetic.

- **Values in the sidebar, not just names.** The highest value-per-byte change in the whole UI.
- **A kind with exactly one account collapses to a single row** linking straight to that account.
  Adding a second account expands it automatically.
- **`EMPLOYMENT` shows `—`, not `0`.** "Nothing by nature" and "worth nothing" must not look
  alike. Driven by `kind.has_nav`. A kind with no valuation yet also shows `—`.
- **Liabilities render negative** in their kind's colour. The pie still excludes them.
- **A staleness dot `·`, not a warning**, on anything carried forward, month count on hover.
  Stale data is normal; mark, never nag.
- **Foreign-currency accounts show both** native and converted.
- **The sidebar is a function of the selected period** — browsing September 2024 shows the house
  you sold in 2025, at its 2024 value. This is what makes the whole app time-travel.
- **"+ Add category" lives at the bottom**, because that is where the user is standing when they
  realise the app has nowhere to put their coin collection. Two fields, then the kind appears
  immediately showing `—`.

### 5.2 Dashboard `[src: fd §7.2, dp]`
Four regions, all from one `GET /api/dashboard/` call: **totals tile** (gross assets, total debt,
net — net worth is the largest thing on the screen), **allocation pie** (top 6 + Other, full
breakdown as a table beneath), **cash-flow bars**, **cash tiles** (one per `is_liquid` account).

The cash-flow chart: months on X, stacked bars on the left axis — income above zero stacked by
source kind, expenses below zero stacked by top-level category — and net worth as a **line on the
right axis**. A **labelled vertical divider at the end of the current month**; projected months
render in the same colours at reduced opacity. Clicking a segment re-requests with `?depth=2`.

**Plus the "September needs closing" prompt, which outranks all four when showing.** This is the
single most important piece of UI in the app: if it is ignored the data goes stale and every
other number becomes confidently wrong. Real presence, not an alarm.

### 5.3 Month close — `/close/[period]` `[src: fd §8.1, dp]`
**The product's spine.** One screen, everything prefilled, every row skippable. If it is tedious
the product fails. Two stacked sections — **balances**, then **flows** — a summary line, and Save.

A currency row at the top: one **USD/EUR rate for the month**, applied to every USD account,
prefilled from the last known rate. Foreign accounts show native and converted side by side.

Four row types, and it matters that they look and behave differently `[src: fd §8.1.4]`:

| Row | Prefilled with | Shown as | Sent if untouched |
|---|---|---|---|
| Balance, `MANUAL` | last month's value, **as a grey hint beside the box** | empty box + "carried forward · N months" | `null` |
| Balance, `AMORTIZED` | this month's scheduled figure, **inside the box** | filled + "scheduled · edit to override" | `null` |
| Balance, `SUM_OF_POSITIONS` (P3) | computed from quotes, **inside the box** | filled + "from quotes" | `null` |
| Flow | every `RecurringRule` due this month | unticked row to confirm, edit or ignore | not created |

> **A carried-forward balance is never put inside the input.** If it were, one click would record
> last month's guess as this month's confirmed truth and staleness would become invisible.
>
> **A computed figure is shown inside the box but only sent if edited.** Untouched it posts
> `null`, so the computation stays live and a later schedule correction still applies.

The summary line is the only computed number on the screen:
`Liquid balances −550 · flows −200 · unexplained −350`. **Informational. Never red, never
blocking, never a validation failure.** Designing that restraint is the hardest part of the
screen.

Other rules `[src: fd §8.1.5]`: Tab moves down the balance column; **Enter on an empty box
accepts the carried-forward figure explicitly**, writing a real `Valuation` — a confirmation, not
a blank. A blank row is never an error. Any past month is reachable through the same screen.
Accounts and kinds are creatable **inline**. **Explicit Save with a navigate-away warning, never
autosave** — autosave plus a wholesale PUT would repeatedly overwrite the month with a
half-filled form's state.

### 5.4 "Closed" is a UI affordance only `[src: fd §8.1.2]`
Nothing in §4 depends on it. Derived, not stored: a month is closed when **every liquid account**
open in that month has a `Valuation` for it. Liquid only — requiring every account made the state
unreachable, since nobody revalues a house monthly, and the prompt would nag forever until the
user learned to ignore it. It drives exactly one thing: the dashboard prompt. `Save & mark closed`
is `Save` plus a stored **dismissal**, not a status.

### 5.5 Account detail — `/accounts/[id]` `[src: fd §7.3, dp]`
The same page for every kind — **no kind-specific components**, which is the test of whether
`AssetKind` was done properly. NAV history as a line, that account's flows as a table, its
valuation history as an editable list, its amortisation schedule where one exists, and for a
property its linked mortgage and the resulting net equity.

### 5.6 Forecast — `/forecast/` `[src: fd §7, dp]`
Projected monthly liquid balance out to 10 years as a line. **Mark the lowest point and its
month**; if the balance ever goes negative, that month is the single thing the user came for.
A today's-money / future-money toggle. A panel listing the recurring rules driving it.

### 5.7 Onboarding / first run — `/onboarding` `[src: fd §8.3]`
1. Register → creates `User` + `Household`, seeds `AssetKind` + `FlowCategory`, asks base currency.
2. "When do you want to start tracking?" — a month picker, default current month. **Stored
   nowhere**; it is simply the month onboarding navigates to.
3. Add accounts — repeating three-field rows (name, kind, currency) grouped under the seeded
   kinds. "+ new category" available exactly as in the sidebar.
4. **Opening balances = the close screen** for that month, every box empty, no "last month"
   column. No separate opening-balance concept, no `is_opening` flag — and the user learns the
   one screen they will use every month.
5. Optional recurring rules. Offered, skippable, better skipped.

**The FX rate is asked at step 4** if any account is USD, which makes §3.5's raise unreachable.
**Backfilling history needs no feature** — any past month is fillable through the same screen.

### 5.8 Empty state `[src: dp]`
A brand-new household with no accounts must not look broken. It should look like the beginning of
something and point clearly at adding the first account.

### 5.9 Cross-cutting frontend rules
- **Tabular figures everywhere, right-aligned in every column.** Numbers that do not line up are
  the fastest way to make a finance app feel amateur. `[src: dp]`
- **Whole currency units, no cents** — this app is deliberately approximate. `[src: fd P6]`
- Every category's colour comes from `GET /api/kinds/`, so pie, bars and sidebar cannot disagree.
- Every view reads `?period` and `?currency` from URL state, so any view is shareable.
- One chart library across the app (`recharts` is already installed).
- Avoid: gradient hero cards, glassmorphism, oversized rounded corners, emoji as icons,
  percentage-change badges on everything. `[src: dp]`

### 5.10 Canvas layouts, and the four places behaviour overrides it

**Layouts the canvas settles** (build these, they are not open questions):

- **Dashboard** — 308px sidebar, then content: header row (`Overview` + "last closed · August
  2026" + Forecast/Settings buttons) → the prompt panel → a 2-up grid of *net worth* and *asset
  distribution* → full-width cash flow → a 3-up row of cash tiles. Net worth is `52px` mono with
  a `€` glyph at `15px` beside it, gross assets and total debt split beneath a hairline. The
  allocation chart is a **donut** (conic-gradient, 158px, 34px inset hole showing `660k / gross`),
  with the legend as a vertical list of swatch · name · pct — not a table.
- **Cash flow chart** — 46px left axis · 900px plot · 52px right axis, 280px tall, zero line at
  140px, 24 columns of 26px with 12px gaps. Net worth is an SVG polyline: **solid for actuals,
  dashed (`4 4`, opacity .55) for projected**. The divider is a 1px `#8d9399` vertical rule with
  a mono label above it reading `SEP 2026 → PROJECTED`. Projected bars are the same colours at
  `opacity:0.4`. Month letters below, with the year printed under each January.
- **Month close** — **two columns side by side**, balances left, flows right (`1fr 1fr`, 44px
  gap). This supersedes `design-prompt.md`'s "two sections stacked". The FX rate is a bordered
  control in the header showing `USD / EUR`, the input, and a mono provenance note. Balances are
  grouped by kind under a swatch + uppercase kind name. Flows are checkbox rows; **confirmed rows
  dim to `--text-mid` on a transparent background, unconfirmed rows stay bright on
  `--bg-row-pending`** — that inversion is the "legible at a glance" requirement, and it is worth
  keeping exactly. A `transfer` pill marks internal transfers. The summary card carries the three
  figures, a plain-language sentence explaining "unexplained", then Save.
- **Account detail** — breadcrumb (`REAL ESTATE /`), title, then a right-aligned three-cell strip:
  Value · Mortgage1 · Net equity, split by hairlines. Below, `1fr 396px`: a value-and-mortgage
  chart (property line `#cf6f96`, mortgage line `#cd7361`, **the carried-forward segment dashed at
  opacity .5**) with a flows table beneath, and a valuation-history panel of editable rows on the
  right.
- **Forecast** — header with a two-button `Today's money / Future money` segmented toggle. Then
  `1fr 364px`: two summary tiles (**Lowest point** in the accent panel treatment, **End of
  horizon** in a plain card) over the projection chart, and a "Rules driving this" panel on the
  right. The chart is a `#57a5d8` line over a 10%-opacity fill, a `#4a5057` zero line, and the
  minimum marked by a dashed accent vertical, a 4px accent dot and a label reading
  `−4,200 · Mar 2029`.
- **Empty state** — sidebar with three hairline placeholders and the "+ Add category" button
  already present; content centred: the serif italic line, a `28px` headline, one paragraph, the
  primary CTA, and a row of starter suggestions beneath a hairline.
- **Narrow (420px)** — prompt, net worth card, then the sidebar's categories as a flat list
  inside a card. No sidebar, no charts.

**Where behaviour overrides the canvas** — four places, each one a real bug if built as drawn:

| # | The canvas shows | Build instead | Why |
|---|---|---|---|
| 1 | "Autosaved as a draft." beneath Save on the close screen | Delete that label. **Explicit Save only**, with a navigate-away warning. | `[src: fd §8.1.5, §11]` — autosave paired with a wholesale PUT (§6.2) means a half-filled form repeatedly overwrites the month with its own incomplete state. The spec rules it out twice. |
| 2 | A long-stale balance (`isCarried`) as a **dashed non-input box reading "carry"** | Render an `<input>` that *looks* exactly like that dashed box until it is focused or typed into, then becomes an ordinary input. | Every row must stay fillable — §5.3's "Enter accepts the carried-forward figure" and acceptance A5 both need a real input. A non-input makes an 11-months-stale house permanently un-updatable. |
| 3 | An **"Import a CSV"** button on the empty state | Omit it. | CSV import is P4 (§10), out of scope this run. Shipping a dead button is worse than shipping neither. |
| 4 | Cash-flow bars as **two tints of one green and one red**, legend "income / expense" | Stack **income by source kind and expenses by top-level category**, each segment in its own `AssetKind` / `FlowCategory` colour; legend lists the series present. Keep the canvas's geometry, divider, and `opacity:0.4` for projected. | `[src: fd §7]` — the whole point of colour living on the kind row is that pie, sidebar and bars cannot disagree. Two swatches cannot represent eight kinds. **Flagged for review: this is the one canvas override that changes how a chart looks, not just how it behaves.** |

Two further canvas details that are mock shorthand, not requirements: the close screen's
**"OTHER" balance group** (build one group per kind, as §5.3 says), and the dashboard header's
**"last closed · August 2026"** (derive it from the §5.4 coverage check — it is a label, and must
never be reintroduced as the actual/projection boundary that §4.4 deleted).

---

## 6. API surface `[src: fd §6]`

JWT as in the template. Every list endpoint household-scoped via `access.py`.

**In scope this run (P0–P2):**
```
GET  /api/dashboard/?period=&currency=        one call, backs the whole dashboard
GET  /api/months/{YYYY-MM}/                   one call, backs the whole close screen
PUT  /api/months/{YYYY-MM}/                   wholesale replace, one DB transaction

GET/POST     /api/kinds/                      PATCH/DELETE /api/kinds/{id}/   (DELETE is PROTECTed)
GET          /api/accounts/?period=&kind=     sidebar shape, pre-totalled per kind
POST         /api/accounts/                   GET/PATCH/DELETE /api/accounts/{id}/
GET          /api/accounts/{id}/summary/?from=&to=
GET/POST     /api/accounts/{id}/valuations/   DELETE /api/valuations/{id}/

GET/POST     /api/transactions/?from=&to=&account=&kind=&category=&status=&q=&page=
PATCH/DELETE /api/transactions/{id}/
GET/POST     /api/categories/                 PATCH/DELETE /api/categories/{id}/
GET/POST     /api/rules/                      PATCH/DELETE /api/rules/{id}/
GET/POST     /api/fx/?from=&to=

GET  /api/net-worth/?from=&to=
GET  /api/cash-flow/?from=&to=&group_by=&depth=
GET  /api/forecast/?to=&real=true|false
POST /api/onboarding/
```
**Deferred (P3/P4):** `/api/accounts/{id}/positions/`, `/api/transactions/import/*`,
`/api/category-rules/`, `/api/net-worth/bridge/`.

### 6.1 Conventions, decided once
- Every `period` is `YYYY-MM`; every `date` is `YYYY-MM-DD`. **No timestamps in the financial API.**
- **Money serialises as a string**, never a JSON number — a `Decimal` through a JS `Number` is a
  float again the moment it reaches the browser.
- Every response carries the `currency` it is expressed in; the UI never infers it.
- **Soft problems come back as a `warnings` array on a `200`** — an uncategorised flow, a nonzero
  unexplained change, a stale quote. Only malformed input is a `400`. The app does not get to
  decide when a household's data is good enough.
- Cross-household access is **404**, never 403.
- Lists page at 100. `/api/dashboard/`, `/api/months/` and `/api/accounts/` **do not page** —
  each backs exactly one screen.
- No GraphQL.

### 6.2 The month-close save contract `[src: fd §8.1.3]`
```
PUT /api/months/2026-09/
{ "fx_rate": "1.0840",
  "valuations": { "<account_id>": "11850.00", "<account_id>": null, ... },
  "flows": [ {id?, amount, from_account, to_account, category, date, description}, ... ] }
```
**The PUT replaces the month wholesale.** Every account the screen showed appears in
`valuations`, with an explicit `null` where the box was blank:

| Sent | Meaning | Effect |
|---|---|---|
| `"34000.00"` | a figure for this month | `update_or_create` the `Valuation` |
| `null` | blank — no figure this month | **delete** any existing `Valuation` for that month |
| `"0.00"` | genuinely worth nothing now | a `Valuation` of zero — **not the same thing** |

> **Blank and zero must never be conflated.** Blank means "carry forward what you knew"; zero
> means "I sold the gold". Confusing them silently zeroes a family's net worth.

Flows follow the same wholesale principle: rows with an `id` are updated, rows without are
created, rows previously in the month and absent from the payload are **deleted**.

**One DB transaction, all-or-nothing. Idempotent. Last write wins** (two family members editing
September concurrently clobber each other — accepted). **Warns, never blocks.**

---

## 7. Rules & invariants (the ones a build can violate silently)

1. No `finance/services/` function branches on an `AssetKind.slug`. **Grep test.**
2. No `RecurringRule` may target an `AMORTIZED` account. **Test + excluded from the UI picker.**
3. An explicit `Valuation` for a month beats every computed method, for **all** methods.
4. A missing FX rate **raises**; 1.0 is never assumed.
5. `Decimal` everywhere in the money path; float appears nowhere. **Never convert on write** — a
   stored USD amount stays USD forever so history re-reads correctly as rates change. `fx.py` is
   the only place conversion happens.
6. Liabilities store positive and negate at aggregation, in one place.
7. A category is required iff exactly one side of a transaction is `NULL`. Serializer, not DB.
8. `FlowCategory` nesting is one level, enforced in `clean()`.
9. Rules never generate at or before the current month.
10. `unexplained_change` is liquid accounts only, differenced natively, both-months-present only.
11. Cross-household reads return 404.
12. Display rounds to whole units; storage keeps two decimals.

### 7.1 Indexes `[src: fd §9.3]`
`Valuation(account, period)` unique · `Transaction(household, date)` ·
`Transaction(household, external_id)` unique partial (P4) · `FxRate(as_of)` unique.
Nothing else needs one.

---

## 8. Integrations

**None in scope for this run.** P3 would add a quote provider (unchosen — the spec's one
genuinely hand-waved external dependency); P4 would add CSV import. No Plaid/GoCardless/Tink.
No Celery, no Redis — the only background job the design ever needs is a cron'd
`manage.py refresh_quotes` at P3.

---

## 9. Non-functional, testing & compliance

**Scale**: one household, ~20 accounts, a decade of monthly valuations (~2,400 rows), a few
thousand transactions. The entire dataset fits in memory. The §4.7 query budget matters far more
than any index.

**Testing** `[src: fd §9.4]` — the golden fixture carries most of the weight: one household with
two properties, two portfolios, a mortgage with `LoanTerms`, a salary, a business and a USD
account, with **hand-computed** expected figures. Tests assert against those numbers, not against
the code's own output. Plus:

| Test | Catches |
|---|---|
| `unexplained_change == 0` on the fixture | aggregation and sign errors |
| `assertNumQueries` on `/api/dashboard/`, `/api/months/` | the N+1 the next serializer field creates |
| Recurrence expansion across year boundaries | off-by-one in `month_index` |
| `Σ principal == principal`, final balance `== 0`, `Σ payments == Σ interest + Σ principal` | schedule maths |
| Closed account: NAV `== 0` after `closed_on`, unchanged before | the sold-house bug |
| Month PUT: blank deletes, `0` stores zero | the blank-vs-zero conflation |
| No `RecurringRule` targets an `AMORTIZED` account | double-counted mortgages |
| `AssetKind` slugs never referenced in `finance/services/` (grep) | the rule that makes user kinds safe |
| Missing FX rate raises | a silent 1.0 |
| Cross-household read returns 404 | tenancy leak |

**Privacy**: one family's own financial data, self-hosted, no third-party processors in scope. No
PII beyond household members' emails, which the template already handles. Financial credentials
are never stored — there is no bank connection.

**Ops**: SQLite in dev, Postgres via `DATABASE_URL`. Deployment target undecided (Open item 3).

---

## 10. Out of scope `[src: fd §11]`

Not built, and each is additive when wanted: net-worth snapshot table · Celery/Redis · event
sourcing / audit-log table · per-asset-class Django apps · tax engine, capital-gains lots,
depreciation · `Scenario` model · `OwnershipStake` · Monte-Carlo projection · **period locking,
audit trail, soft deletes, reconciliation enforcement, double-entry balancing** (all five follow
from P6) · sub-monthly resolution · penny-accurate display · optimistic locking on the close ·
autosave · a stored "month closed" status · user-defined behaviour on custom kinds · kind merging
UI · security master / corporate actions / trade log · bulk history importer · per-account or
per-member permissions · notifications or emails beyond the in-app "needs closing" prompt.

**Deferred to a later run** (in the spec, not in this build): P3 `Position`/`Quote`/
`refresh_quotes`; P4 CSV import, `CategoryRule`, `net_worth_bridge` waterfall, PDF export, alerts.

---

## 11. Acceptance criteria

Observable, per flow. The product is done when a human can do all of these in a browser.

| # | Flow | Passes when |
|---|---|---|
| A1 | **First run** | Register → household created, 8 kinds + ~30 categories seeded, add 3 accounts incl. one USD, enter a FX rate and opening balances, land on a dashboard showing real numbers. |
| A2 | **Sidebar** | Shows kind → account with values; Employment shows `—`; a one-account kind is collapsed; Debt is negative; a carried-forward figure carries a staleness dot with a month count. |
| A3 | **Dashboard** | Totals tile shows gross/debt/net; pie shares sum to 100% of gross assets and match the sidebar's colours; cash tiles appear for liquid accounts only. |
| A4 | **The prompt** | With the current month unclosed, "September needs closing" is visible and links to the close screen. Filling every liquid account's balance makes it disappear. |
| A5 | **Month close** | A `MANUAL` row's box is empty with last month beside it; an `AMORTIZED` row's box is pre-filled and labelled "scheduled"; Enter on an empty box writes a real valuation; a blank box on save deletes any existing valuation while a typed `0` stores zero. |
| A6 | **Unexplained** | With balances moving −550 and flows netting −200, the summary reads "unexplained −350", in a neutral colour, and Save succeeds. |
| A7 | **Time travel** | Navigating to a past month re-renders the sidebar, dashboard and pie at that month's values. An account closed later still appears at its old value. |
| A8 | **Cash flow** | Bars show income above zero by kind and expenses below by category; a mortgage payment appears as interest expense **and** principal transfer, not as one line; a share purchase appears in neither income nor expense. |
| A9 | **Forecast** | Projects liquid balance to 10 years, marks the minimum and its month, and reports the first negative month if any. Toggling real/nominal changes the curve and nothing stored. |
| A10 | **Add a category** | "+ Add category" in the sidebar with two fields produces a working kind that appears in the sidebar, the pie, the close screen and the account page, with no code change. |
| A11 | **Access** | A `VIEWER` gets 200 on reads and 403 on writes. A user from another household gets **404**, not 403, on every object. |
| A12 | **Budget** | `/api/dashboard/` and `/api/months/` each stay within the §4.7 query budget under `assertNumQueries`. |

---

## Decision log

| # | Question | Decision | Date | Source |
|---|---|---|---|---|
| D-01 | Is the frontend in scope? | **Yes.** Both repos. The frontend follows the Claude Design canvas linked in `claude-design-prompt.txt`, which outranks `design-prompt.md` on visual questions. | 2026-09-02 | user |
| D-02 | How far may the template be stripped? | **Strip + reset migrations.** Delete campaigns, posts, sources, publishing, AI, billing, magic links, client view and their tests; rename `Organization`→`Household` and roles ADMIN/AGENCY/CLIENT→OWNER/EDITOR/VIEWER; delete all migrations and `db.sqlite3`; regenerate a clean initial migration. Keep auth, invites, RBAC, tenancy, throttling, media. No data to preserve. | 2026-09-02 | user |
| D-03 | Which phases this run? | **P0–P2**: net worth, cash flow + FX, and the 5–10 year forecast. P3 (quotes) and P4 (CSV import, bridge) deferred. | 2026-09-02 | user |
| D-04 | §12 Q5 / Q6 / Q9 | **All three as the spec recommends.** Employment = net take-home only, no gross/tax subsystem. Business = a black box with manual NAV paying ordinary flows. `fiscal_year_start_month` **dropped** as speculative schema. | 2026-09-02 | user |
| D-05 | Which manifest is authoritative? | `pyproject.toml`. `requirements.txt` is UTF-16 and describes an unrelated older project; it will be deleted, and any dependency the build actually needs (`django-pandas`) added to `pyproject.toml` explicitly. | 2026-09-02 | Claude (intake) |
| D-06 | Sketch vs canvas | The Claude Design canvas supersedes `Layout.pptx` / `findash.png`. The pptx stays as provenance only; its category labels ("Broad Money", "Cryptocurrency") are not binding — kinds are renameable by design. | 2026-09-02 | Claude (intake) |
| D-07 | Python environment | **A `.venv` in the backend repo.** `python -m venv .venv` then `.venv/Scripts/python -m pip install -e ".[dev]"`. Matches the template README; `.venv` is gitignored. First WP of the run must create it. | 2026-09-02 | user |
| D-08 | What "the product runs" means | **Postgres via `DATABASE_URL`.** Verified available: PostgreSQL 18.3 on `localhost:5432`, user `postgres`, password `admin` (`postgres_notes.txt`). The run creates its own database rather than touching any existing one, and ends with backend + frontend up and the §11 acceptance criteria walked in a browser. | 2026-09-02 | user |
| D-09 | Postgres driver | `psycopg[binary]` (psycopg 3) added to `pyproject.toml` — Django 6's recommended driver. `psycopg2` 2.9.12 also exists in the global interpreter and would work, but the `.venv` installs from `pyproject.toml` and should name one driver. | 2026-09-02 | Claude (intake) |
| D-10 | Frontend state management | **Neither Redux Toolkit nor Zustand.** Both are in the template's `package.json`; the finance app needs neither. Server state comes from `fetch` per screen (each screen has exactly one endpoint by design, §6), and `?period` / `?currency` live in URL state, which §5.9 already requires. Both libraries are dropped from `package.json` along with the rest of the template's unused UI stack. | 2026-09-02 | Claude (intake) |
| D-11 | Frontend dependency pruning | The template ships MUI, FontAwesome, Heroicons, Tabler, Lucide, react-icons, dnd-kit, react-social-icons and more. **Prune `package.json` to what the finance app actually imports** — the Claude Design canvas decides which UI primitives survive, so the prune happens in the first frontend WP, after the canvas is read. Keep: Next, React, Tailwind v4, `recharts`, `next-themes`, and whatever the canvas uses. | 2026-09-02 | Claude (intake) |


| D-13 | Design canvas retrieval | **Read and saved to `project/design-canvas.html`** (2026-09-02, via `DesignSync` after `/design-login`). Six artboards + the demo dataset. §5.0 and §5.10 extract its tokens and layouts so WP specs are self-contained and Codex never needs the MCP. | 2026-09-02 | Claude (intake) |
| D-14 | Canvas vs spec, 4 conflicts | Resolved in §5.10: **behaviour wins** in all four. (1) No autosave. (2) The "carry" box becomes a real input styled as that box. (3) No CSV button on the empty state. (4) Cash-flow bars stack by kind/category in their own colours, not two tints. **(4) is the one that changes a chart's appearance — the WP spec must call it out for Codex review.** | 2026-09-02 | Claude (intake) |
| D-15 | Theme | **Dark only, committed.** The canvas is a single fully-resolved dark theme and `design-prompt.md` explicitly permitted "one theme committed to properly". `next-themes` is dropped along with the rest of the template's unused deps (D-11). | 2026-09-02 | Claude (intake) |
| D-16 | `AssetKind.colour` seed values | Take the canvas's hexes verbatim (§5.0). They are what the pie, the sidebar and the bars are drawn with, so seeding anything else guarantees a mismatch on day one. | 2026-09-02 | Claude (intake) |

## Open items

_None. The spec is actionable._
