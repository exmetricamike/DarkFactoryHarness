---
description: Phase 4 - hand a frozen WP to Codex, verify the result independently, commit
argument-hint: WP-XXX
---

# /df-build $ARGUMENTS — implement, verify, commit

Load the `codex-protocol` skill.

**Precondition, hard:** WP state is `SPECCED`, the spec is `FROZEN`, and its log records a Codex `VERDICT: READY` with no unresolved BLOCKER. Anything else → go run `/df-spec` and come back. This holds whoever implements it, Codex or you.

## Step 1 — pre-flight

Per target repo: `git -C <repo> status --short` must be clean and note `HEAD` sha — that is your diff baseline. Unattended and the tree is dirty? Account for the changes yourself: commit them if they are a finished previous WP, stash them if they are stray, revert them if they are broken. Log it; do not wait for the user.
Create/switch to the WP branch if the project uses branches (`PROJECT.md` says); otherwise work on the current branch.

## Step 2 — implement

Prompt = envelope + **IMPLEMENT block**, spec inlined verbatim, plus a "Already built" list from the DONE WPs.
Resume the WP's Codex session. Set BACKLOG state `BUILDING`. Expect minutes: `timeout: 600000`, or background it.
Cross-repo WP: backend call first, then frontend, passing the *actual* implemented contract (paste the real route signatures/types Codex produced) into the frontend prompt. Never let the frontend guess.

If the call fails on quota/rate-limit, load the `continuity` skill §B: record what Codex already wrote (`git status`/`git diff`), do not commit a half-finished WP, mark the WP `BLOCKED` with reason `codex-down`, and switch to the takeover ladder.

## Step 3 — verify (state `VERIFY`) — this is your core job, do not shortcut it

Run all four. Record actual output, not impressions.

1. **Diff review**: `git -C <repo> diff --stat <baseline>..` then read the full diff.
   Reject-worthy: files outside scope, secrets/keys committed, deleted tests, `any`/silenced types where the repo doesn't allow it, TODO stubs presented as done, error paths swallowed, dependencies added that the WP never authorized, copy-paste of an existing helper instead of reuse.
2. **Their tests**: run the repo's test + lint + typecheck commands from `PROJECT.md` yourself. Codex's `TESTS:` line is a claim, not evidence.
3. **Your independent check** — the one from spec §6, written by you into `project/checks/WP-XXX.*` (never inside the repo). Prefer a real probe over a unit test: start the service and curl the endpoint, drive the UI, inspect the DB row. It must be able to fail: confirm it fails against the pre-WP baseline (or reason explicitly why that is impossible).
4. **Run the actual product** — mandatory for any WP touching the frontend, and for any backend WP with a reachable route. `night-shift` skill §5 has the procedure: start the services in the background, poll for readiness, `curl` the new routes including their failure paths, then drive the UI with the `claude-in-chrome` tools and read the console and network for errors the DOM hides. Green tests over a blank screen is a failed WP.
5. **Acceptance criteria**: walk §6 one by one, mark each PASS/FAIL with the evidence line. Then judge it as a user would — empty states, error states, labels, contrast, a window at a normal size. Defects a user would notice go back as a FIX round; they are not polish.

## Step 4 — outcome

- **All pass** → Step 5.
- **Anything fails** → FIX round (max 3): exact command + exact output, no interpretation, no suggested patch unless the fix is unambiguous. Re-verify from Step 3 in full each time. After 3 failed rounds → if the tree is worse than the baseline, revert this WP's commits (`git revert`, never `reset --hard` on committed work); mark `BLOCKED`, log the failure evidence, and **take the next WP** — unattended, you never halt the line for one package.
- **Passes but the diff is wrong-shaped** (out-of-scope files, dead abstraction, duplicated helper) → FIX round to trim it. A green suite is not permission to keep bloat.

## Step 5 — commit (only now)

Per repo, only the files this WP touched:

```bash
git -C <repo> add -A && git -C <repo> commit -m "WP-XXX: <title>

<1-3 lines: what changed and why>
Spec: project/wps/WP-XXX.md
Tests: <command> -> <result>"
```

Never `git push` unless the user asks. Cross-repo: commit backend then frontend, same WP id in both subjects.

## Step 6 — close out

1. BACKLOG: state `DONE`, update the `DONE n/total` header.
2. Append to `project/wps/WP-XXX.log.md`: fix rounds, verification evidence, commit shas per repo, follow-ups discovered.
3. Any follow-up worth doing → new WP row (`TODO`) at the end of the backlog. Never silently absorb it into the next WP.
4. Rewrite `project/RESUME.md` (`continuity` skill format). One write, and a hard cutoff now costs nothing.
5. Report: outcome sentence, commit shas, what you verified and how, anything deferred, `Next: /df-spec WP-YYY`.
