---
name: night-shift
description: Unattended operation - deciding alone, when to stop, browser verification, the morning report. Load at /df-run start or when a decision would block.
---

# Night shift — running the factory alone

The user launched `/df-run` and went to bed. Nobody is coming. Act accordingly.

---

## 1. The decision protocol

When you hit something you would normally ask about, run this — it takes seconds, not a deliberation.

0. **Did a previous night already answer it?** `LESSONS.md` — the harness's cross-project memory. A lesson that fits the situation outranks reasoning it out again from scratch; that is why it exists. Follow it, and bump its `hits`. Nothing fits → carry on to 1.
1. **Does the spec answer it?** `active-project/SPEC.md`, then the WP spec, then `active-project/PROJECT.md`. Read before deciding.
2. **Does the codebase answer it?** The existing pattern in the repo wins over your preference. Consistency is a decision you never have to defend.
3. **Does a convention answer it?** Framework defaults, REST/HTTP semantics, the stack's idioms. Standard beats bespoke.
4. **Still open → apply the tie-breakers**, in order:
   - **Fewest moving parts.** No new dependency, no new service, no new abstraction, no new config knob.
   - **Reversible over permanent.** A choice one commit undoes beats one that needs a migration to undo.
   - **Narrow over general.** Build the case in the spec, not the framework for cases nobody asked for.
   - **Boring over novel.** The thing the next reader recognizes instantly.
   - **Ship the smaller slice.** If a WP has an obvious 80% version and a 100% version that needs an answer you don't have, ship the 80% and log the remainder as a follow-up WP.
5. **Log it** in `active-project/DECISIONS.md`, one row, then continue. Do not deliberate twice about the same thing — if you already decided it, follow your own precedent.

**Never** stop to ask about: naming, file layout, library choice among installed options, error message wording, validation strictness, pagination defaults, empty/loading/error state design, ordering of work, whether a WP is big enough to split, or how to word a commit.

**Deciding alone does not mean deciding by yourself.** Codex is awake and it reads the repo. On a genuinely close design call inside a WP, put both options in the review round and ask which one fits the existing code better — one extra round, a much better answer. The user is asleep; your collaborator is not.

## 2. `active-project/DECISIONS.md` — append-only

Every judgment call the user would plausibly want to revisit. Cheap to write, and it is the entire morning conversation.

```markdown
| # | WP | Decision | Alternative rejected | Why | Reversible? |
|---|----|----------|---------------------|-----|-------------|
| 12 | WP-004 | Soft-delete via `deleted_at` | Hard delete + audit table | Spec §7 needs restore; one column vs one table | yes, one migration |
```

Flag anything you are less than confident about with `⚠` in the `#` column. Those are the first lines you show in the morning report.

## 2b. `LESSONS.md` candidates — capture the cost, distil later

Every time the night costs you something, append **one line** to the `Candidates` section of
`LESSONS.md` and keep moving. Triggers, no judgement required:

- a spec that needed ≥3 Codex review rounds, or got descoped to get a READY
- a WP reverted, blocked, or split mid-flight
- ≥2 fix rounds on the same defect
- a decision you later had to reverse
- time lost to the environment, the tooling, or the protocol itself rather than to the product

Format: what happened, what it cost, and — if you can see it — the rule that would have avoided it.
`- cand: frontend guessed the response shape from the spec, 2 fix rounds → paste the backend's real signatures into the frontend prompt`

Rules: **candidates are cheap and unfiltered, lessons are not.** Do not edit the lesson sections during
the run, do not stop to generalise, do not write project identity into the line. `/df-retro` decides in
the morning which of these are real, with the user's grades in hand. If a candidate is genuinely
project-specific, it dies there — that is the system working.

## 3. When the pipeline may stop — the only three

1. **Security or data loss.** Credential handling you would be guessing at, a destructive migration on data you cannot verify is disposable, secrets that would land in a commit, an auth model the spec never defines. Do not guess. Stop that WP, take the next one.
2. **Irreducible product contradiction.** Two spec sections demand opposite behavior and picking either invents product intent that changes what the product *is* (not how it looks). Cosmetic or technical contradictions are not this — decide those.
3. **Both agents out of credit.** Nothing left to run with.

Everything else — a failing test, a confusing module, a missing endpoint, an unclear requirement, a Codex disagreement, a broken dev environment — is work, and work is what you are here for.

**Stopping is per-WP, never per-project.** Mark that WP `BLOCKED`, write why, and immediately take the next WP whose dependencies are met. Only when *no* WP is runnable do you halt the loop — and then you still write the morning report.

When you do stop a WP for reason 1 or 2, fire a `PushNotification` (load it with `ToolSearch` first) with one line of context. The user may see it; do not count on it, and do not wait for a reply.

## 4. Keeping the line moving

| Situation | Do this, not that |
|---|---|
| Dependency WP blocked | Take the next independent WP. Re-plan the order in the backlog if several are stuck behind one. |
| Codex disagrees with the spec 3 rounds in | Do not bulldoze it — its objections are usually about the code you cannot see. Cut the contested part into a follow-up WP and get a READY on the smaller spec. A reviewed thin slice tonight beats an unreviewed thick one. |
| Codex flags a BLOCKER you think is wrong | Check the code. Wrong → answer with the file:line and let it re-judge. Right → fix the spec. Never proceed with the blocker standing. |
| Verification fails 3 fix rounds | Revert the WP's commits if the tree is worse than before (`git -C <repo> revert` — never `reset --hard` on committed work), mark `BLOCKED`, move on. A clean tree at 4am is worth more than a half-working feature. |
| Test suite is flaky | Re-run once. Still flaky → treat the flake as a finding, log it, judge the WP on the deterministic tests. Do not delete or skip tests to get green, and do not let a flake block a WP. |
| Dev environment broken (port, missing service, bad env var) | Fix it — that is environment, not product code, and it is yours to fix. Log it. |
| A WP turns out much bigger than specced | Split it: implement the core slice now, add the remainder as a new WP at the end of the backlog. |
| Spec turns out wrong once you see the code | Amend `active-project/SPEC.md`, log the amendment with a `⚠`, continue. The spec serves the product, not the reverse. |
| You are unsure whether something is in scope | Out. Log it as a follow-up WP. Scope creep at 3am is unsupervised scope creep. |

## 5. Verifying that it actually runs

"All WPs implemented" is not the goal. **A usable product in the morning** is the goal.

At the end of every WP that touches the frontend, and again after the final WP, prove it with the app running — not just with the test suite.

1. Start what the WP needs (`PROJECT.md` has the commands): services, backend, frontend. Run them with `run_in_background: true` and log to files under the scratchpad; keep the ports from `PROJECT.md`.
2. Wait for readiness by polling the port/health endpoint — never by sleeping a fixed time.
3. **Backend**: `curl` the routes this WP added. Assert status, shape, and the failure paths (401/403/404/422), not just the happy 200.
4. **Frontend**: drive the real UI with the `claude-in-chrome` tools — load the `claude-in-chrome` skill, open a fresh tab, walk the flow from §6 of the WP spec, read the console (`read_console_messages`) and network (`read_network_requests`) for errors the DOM does not show. A screenshot of a blank page with a red console is a failure, not a pass.
5. **Judge it like a user, and fix what a user would notice**: unreadable contrast, unlabeled controls, a form with no error state, a list with no empty state, a button that does nothing, a layout broken at a normal window size. Send those back to Codex as a FIX round — they are defects, not polish.
6. Record the evidence in the WP log: commands run, status codes, what you saw, console errors. "Verified" without evidence is not verified.
7. Shut the processes down when you are done with them, so the next WP starts clean.

If the UI cannot be driven (extension not connected, app will not boot), say so explicitly in the WP log and the morning report — never silently downgrade to "tests passed, probably fine".

## 6. Final integration pass — run this after the last WP, always

The product must be usable, not merely built.

1. Clean-slate boot: from a fresh clone-equivalent state (fresh install, migrations, seed), start everything using only what `PROJECT.md` documents. If a step is missing from the docs, add it — undocumented setup is a broken product.
2. Walk **every** flow in `SPEC.md` §4 end to end in the browser, as a real user would, including the unhappy paths.
3. Check `SPEC.md` §11 acceptance criteria one by one: PASS/FAIL with evidence.
4. Anything broken → one more Codex round if credits allow; otherwise record it precisely at the top of the morning report.
5. Leave the repos committed and clean, and leave the app startable with a single documented command per repo.

## 7. The morning report — `active-project/MORNING.md`

Write it last, rewrite it fully each night. The user reads this before anything else, coffee in hand.

```markdown
# Morning report — <date>
## Bottom line
<2-3 sentences: what the product can do now that it could not last night, and whether it runs.>

## Needs you (top of the list, or "nothing")
- ⚠ <decision you want reviewed> — DECISIONS #<n>
- <WP blocked and why>

## Done tonight
| WP | Title | Verified how | Commits |

## Not done
| WP | State | Why | What it needs |

## Decisions taken  (full list in active-project/DECISIONS.md)
<the ⚠ ones, one line each>

## Grade these, then run `/df-retro`
<the ⚠ decisions again as a numbered list, each with the alternative rejected, so the user can say
"#3 was wrong" in one line. Put their verdicts in the `Grade` column of active-project/DECISIONS.md.>
<n> lesson candidates are waiting in LESSONS.md; `/df-retro` distils them with your grades.

## How to run it right now
<exact commands, per repo, that you actually executed and saw work>

## Suggested next session
<3 bullets max>
```

Publish it as an Artifact as well if that tool is available, and give the user the link in your final message. Keep the file either way.

## 8. Budget discipline overnight

You are spending a finite budget while nobody watches. Do not waste it.

- Do not re-read files you already read this session; do not re-verify a WP you already verified.
- Do not re-litigate a decision already in `DECISIONS.md`.
- Prefer one precise Codex round over three vague ones: a FIX prompt with exact failing output beats "it doesn't work".
- Do not burn rounds polishing a WP that already meets its acceptance criteria. Meets spec = done.
- Keep your own reports to the compact block in `df-run.md`. Prose costs tokens that could have been a work package.
- A candidate line is one line. Retrospection is a morning activity with the user present; at 3am it is procrastination with a budget.
