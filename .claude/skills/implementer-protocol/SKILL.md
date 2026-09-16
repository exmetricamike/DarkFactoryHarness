---
name: implementer-protocol
description: How the coordinator talks to the configured reviewer/implementer backend - resolving the profile, self-contained prompts, review/implement/fix blocks, verdict parsing. Load before any backend call.
---

# Implementer protocol

The coordinator is whichever model is executing these commands. The **reviewer** and the
**implementer** are separate configured roles — they may be the same backend or two different ones.

## Step 0 — resolve the role before the first call

Read `harness.config.json`: `roles.<reviewer|implementer>` → `profiles.<id>`. Load
`.claude/docs/adapters/<adapter>.md` for the profile(s) the run actually uses, and nothing else.
Record the resolved profile id in the WP log the first time you call it.

Everything below is backend-agnostic. Commands, session handling and failure strings come from the
adapter doc; round limits, timeouts and the prompt budget come from the profile's `limits`.

- Same profile for both roles → one session per WP, reused for review, implementation and fixes.
- Different profiles → **separate sessions**: the reviewer's session for review rounds, the
  implementer's for implementation and fixes. The implementer never inherits the reviewer's context,
  so its first prompt carries the frozen spec in full, as always.
- A WP never changes profile mid-flight. The only exception is the configured `fallback`, which
  starts a fresh session and is recorded in the WP log and the commit trailer.

## Golden rules

1. **Self-contained prompts.** The backend starts cold and its sandbox root is the *repo*, not this
   harness. Never write "see active-project/wps/WP-003.md" — inline the full text. Build the prompt
   as a file, pipe it on stdin.
2. **One session per WP per role**, reused across rounds. Store the id in `active-project/wps/WP-XXX.log.md`.
3. **Always capture output to a file** so you read the reply as a file instead of scraping the terminal.
4. **The backend is a collaborator, not an executor.** It reads the real repo with fresh eyes; you
   wrote the spec and are attached to it. Ask for critique before implementation, take its objections
   seriously, resolve them in the spec. You still own scope and the final decision — but you earn that
   by answering its points, not by ignoring them.

## The order of operations, always

`write spec → reviewer reviews it → resolve every issue in the spec → VERDICT: READY → implement`

Never skip the middle. An unreviewed spec goes to no implementer, including you.

## Prompt envelope

Every prompt file uses this skeleton. Sections in this order — the required output format goes last,
because the last instruction is weighted heaviest.

```markdown
# Project: <name>
<3-6 lines: what the product is, who uses it, the current phase>

## Stack and conventions (authoritative)
<paste the relevant slice of active-project/PROJECT.md: language, framework, versions, test runner, lint, dir layout, naming, error/logging conventions>

## Repo you are working in
<REPO_PATH> (<frontend|backend>) — branch <branch>
Related repo (do not edit): <OTHER_REPO_PATH> — <how they talk: REST/GraphQL/…, contract location>

## Already built (do not redo, do not refactor)
<one line per DONE WP>

## Work package WP-XXX: <title>
<the full frozen WP spec, inlined verbatim>

## Constraints
- Stay inside this WP's scope. Out-of-scope improvements: list them, do not implement them.
- Do not change public contracts (API routes, schemas, shared types) unless this WP says to.
- Follow existing patterns in the repo over your own preferences.
- Ship tests with the code: <test command>. All tests must pass before you report done.

## Your task
<REVIEW BLOCK or IMPLEMENT BLOCK — see below>

## Required output format
<the matching verdict block — see below>
```

When the profile's `capabilities.network` is false, every IMPLEMENT prompt also carries:

> Your sandbox has no network. If a dependency is missing, stop and report `PARTIAL` saying which
> one. **Never** satisfy an import by pointing `PYTHONPATH`, `NODE_PATH` or any other loader at
> packages outside this repository — another project's virtualenv or `node_modules` on this machine
> is not this project's environment, and a suite that passes against it proves nothing about this
> one. Do not read, copy from, or write to any directory outside this repo.

Left unsaid, a blocked backend will find a sibling project's environment on disk and run the suite
green against it. It usually reports this honestly in `NOTES`, which is the only reason it gets
caught — so **always read `NOTES`, and always check which interpreter the `TESTS:` line used.**
Anything it reached outside the repo is an incident: scrub the other project's identity out of the WP
log before committing, and never let it into `LESSONS.md` (invariant 9).

Provision the environment yourself — venv, `npm install`, migrations — before the implement call,
from the coordinator side where the network works, and say so in the prompt.

### REVIEW block (spec-review round)

```markdown
## Your task
You are the senior engineer who will implement this. Review the spec against the actual code in
this repo before a line is written. Do NOT modify any files.

Be critical — this review exists to catch what the author missed, and the author would much rather
hear it now than after the code is written. Concretely:
(a) Is anything ambiguous, i.e. could two competent engineers read it differently?
(b) Does it contradict what already exists — schema, contracts, naming, patterns, assumptions?
(c) Is anything missing that would block or slow you: migration, config, auth, error paths, edge cases?
(d) Is the scope right for one session, and is the ordering of the steps actually workable?
(e) What would you do differently, and why?

Disagreement is useful. If the approach is wrong, say so plainly and propose the alternative.
If it is genuinely sound, say READY — do not invent issues to seem thorough.

## Required output format
End your reply with exactly this block and nothing after it:

VERDICT: READY | ISSUES
ISSUES:
1. [BLOCKER|GAP|SUGGESTION] <one-line problem> -> <concrete proposed resolution>
2. ...
(omit the list entirely when VERDICT is READY)
```

**BLOCKER means the spec is wrong or unimplementable as written** — the coordinator must fix the
spec, not argue past it. GAP means something uncovered. SUGGESTION means an alternative worth weighing.

Round n+1 re-sends the amended spec plus a resolutions list, so the reviewer sees what actually changed:

```markdown
## Resolutions from your last review
1. ACCEPTED — spec §4 now specifies <x>.
2. DISPUTED — <file:line> shows <evidence>; re-judge with that in mind.
3. DECLINED — <reason it stays out of scope>; moved to §3 Out / a follow-up WP.

Re-review the spec below as amended. Same output format.
```

### IMPLEMENT block

```markdown
## Your task
Implement this work package in this repo. Write the code and the tests. Run the test command and
the lint command; fix what you break. Do not commit — the coordinator commits.

## Required output format
End your reply with exactly this block and nothing after it:

VERDICT: DONE | PARTIAL | BLOCKED
FILES: <path> (added|modified|deleted), ...
TESTS: <exact command run> -> <pass/fail counts>
NOTES: <anything the coordinator must know: assumptions made, out-of-scope items found, follow-ups>
```

Drop the `TESTS:` line from the required format when the profile's `capabilities.runs_commands` is
false — you run the tests either way, so do not ask for a claim the backend cannot make.

### FIX block (verification failed)

```markdown
## Your task
Your previous implementation of WP-XXX failed verification. Fix it. Do not start new work.

### Failures
<paste exact command + exact output, trimmed to the relevant lines>

## Required output format
<same as IMPLEMENT block>
```

## Parsing the reply

Read the out file and take the **last** `VERDICT:` line.

- No `VERDICT:` line → **one** re-ask on the same session: "Reply with only the required verdict
  block for the work you just did. Do not redo anything." Count it as a round. Still no verdict →
  treat as `BLOCKED`, record the last 20 lines of the reply in the log, do not retry blindly.
- `PARTIAL`/`BLOCKED` → read `NOTES`, decide: FIX round, amend the spec, or mark the WP `BLOCKED`.
- Never trust `TESTS:` — you re-run the tests yourself. It is a claim, not evidence.

## Round limits

From the active profile's `limits` (`codex-cloud`: 3 and 3).

- Spec review: `spec_review_rounds`. Then you decide the final spec and move on; log the unresolved points.
- Fix rounds after failed verification: `fix_rounds`. Then mark `BLOCKED` and escalate with the failure output.

A re-ask for a malformed verdict counts against the limit.

## Failure handling

Adapter-specific strings and exit codes are in the adapter doc's §7. Generic responses:

| Symptom | Action |
|---|---|
| non-zero exit / empty out file | Report the stderr verbatim. Do not retry more than once. Quota signature → `continuity` §B. |
| it asks a question instead of delivering | Answer it inline in the next round. Count it as a round. |
| it edited files outside the WP scope | `git -C <repo> diff --stat`, then FIX round: "revert changes to <paths>, they are out of scope". |
| it committed despite instructions | Leave the commit; verify it, amend the message to convention if needed. Note it in the log. |
| session id lost | Adapter doc §4, then §5. |
| repeated garbage after `fix_rounds` | Profile has a `fallback` → start a fresh session on that profile, record the switch. No fallback → `BLOCKED`, next WP. |

Overwrite the prompt and out files each round; the durable record is the WP log, appended every round.
