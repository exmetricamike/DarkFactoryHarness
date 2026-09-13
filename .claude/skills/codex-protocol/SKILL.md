---
name: codex-protocol
description: Exact Codex CLI usage - commands, session persistence, self-contained prompts, reply parsing. Load before any codex exec call.
---

# Codex protocol

Codex CLI `0.152.0` (flags below re-verified against `codex exec --help` at this version). Verify with `codex --version`; if the major/minor differs, re-check `codex exec --help` before trusting the flags below.

A version print is not a reachability check. Before an unattended run, prove auth with one cheap call:

```bash
codex exec --cd "<REPO_PATH>" -s read-only -o smoke.out.md \
  "Reply with exactly one line naming this repo's primary language. Do not modify any files."
```

An empty `smoke.out.md` means the night would have run entirely on the Codex-down ladder (`continuity` skill §B) — find that out now, not at 1am.

## Golden rules

1. **Self-contained prompts.** Codex starts cold and its sandbox root is the *repo*, not this harness. Never write "see active-project/wps/WP-003.md" — inline the full text. Build the prompt as a file, pipe it on stdin.
2. **One Codex session per WP**, reused for review rounds, implementation and fixes. Store the session id in `active-project/wps/WP-XXX.log.md`.
3. **Always capture output** with `-o` so you can read the reply as a file instead of scraping the terminal.
4. **Codex is a collaborator, not an executor.** It reads the real repo with fresh eyes; you wrote the spec and are attached to it. Ask for critique before implementation, take its objections seriously, and resolve them in the spec. You still own scope and the final decision — but you earn that by answering its points, not by ignoring them.

## The order of operations, always

`write spec → Codex reviews it → resolve every issue in the spec → Codex says READY → implement`

Never skip the middle. An unreviewed spec goes to no implementer, including you.

## First call of a WP (creates the session)

```bash
codex exec --cd "<REPO_PATH>" -s workspace-write --approve-for-me --json \
  -o "<HARNESS>/active-project/wps/WP-XXX.out.md" \
  - < "<HARNESS>/active-project/wps/WP-XXX.prompt.md" \
  | tee "<HARNESS>/active-project/wps/WP-XXX.jsonl"
```

Capture the session id straight after:

```bash
grep -oE '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' \
  "<HARNESS>/active-project/wps/WP-XXX.jsonl" | head -1
```

Write it into the WP log as `Codex session: <uuid>`. If the grep is empty, fall back to `codex exec resume --last` for the *immediately* following call and record `session: --last (id capture failed)`.

## Follow-up calls (same session)

**`codex exec resume` does NOT accept `--cd`, `-s/--sandbox` or `--approve-for-me`.** Its usage is
`codex exec resume [OPTIONS] [SESSION_ID] [PROMPT]`, and the only options that matter here are
`-o` and `-c`. Passing `--cd` fails with `error: unexpected argument '--cd' found` and exit 2.
Set the working directory with a shell `cd` instead, and use **absolute paths** for `-o` and the
stdin redirect, because the `cd` changes what relative paths mean:

```bash
rm -f "<HARNESS>/active-project/wps/WP-XXX.out.md"          # see the stale-reply trap below
cd "<REPO_PATH>" && codex exec resume "<SESSION_ID>" \
  -o "<HARNESS>/active-project/wps/WP-XXX.out.md" \
  - < "<HARNESS>/active-project/wps/WP-XXX.prompt.md"
```

Sandbox mode carries over from the session; override it with `-c sandbox_mode="workspace-write"`
if a resumed session needs to write and the first call was `-s read-only`.

> **The stale-reply trap.** `-o` is only written when the call succeeds. A failed resume leaves
> the *previous* round's `.out.md` sitting there, and reading it looks exactly like Codex
> repeating itself verbatim — which is the tell. **Always `rm -f` the `.out.md` before the call
> and check the exit code**, or you will "resolve" the same issue list twice.

Overwrite `.prompt.md` and `.out.md` each round; the durable record is the WP log, which you append to every round.

## Flags

- `-s workspace-write --approve-for-me` — the project default. Codex edits the repo and runs commands unattended, sandboxed to the workspace.
- `--cd <REPO_PATH>` — one repo per call. A WP touching both repos = two calls (backend first unless the spec says otherwise), or one call plus `--add-dir <OTHER_REPO>` when the change must be atomic across both.
- Add `--skip-git-repo-check` only if the target is not a git repo (it should be — fix that instead).
- Long runs: give the Bash tool `timeout: 600000` and expect implementation calls to take minutes. If a call may exceed 10 min, run it with `run_in_background: true` and pick the result up from `.out.md`.
- A review-only call may use `-s read-only` if the user asks for extra safety; the project default keeps `workspace-write` and relies on the prompt's "do not modify files" instruction.

## Prompt envelope

Every prompt file uses this skeleton. Sections in this order — Codex weights the last instruction heaviest, so the required output format goes last.

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

Round n+1 re-sends the amended spec plus a resolutions list, so Codex reviews what actually changed:

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

### FIX block (verification failed)

```markdown
## Your task
Your previous implementation of WP-XXX failed verification. Fix it. Do not start new work.

### Failures
<paste exact command + exact output, trimmed to the relevant lines>

## Required output format
<same as IMPLEMENT block>
```

## Codex has no network — plan for it

Its sandbox blocks package installs. `pip install`, `npm install`, `poetry add` and friends will
fail on every WP that needs a new dependency. Two consequences, both learned the hard way:

1. **Do the install yourself, before the implement call.** Create the venv / run `npm install`
   from the coordinator side where the network works, then tell Codex in the prompt that the
   environment is already provisioned and it must not try.
2. **Forbid the workaround explicitly**, in every IMPLEMENT prompt:

   > Your sandbox has no network. If a dependency is missing, stop and report `PARTIAL` saying
   > which one. **Never** satisfy an import by pointing `PYTHONPATH`, `NODE_PATH` or any other
   > loader at packages outside this repository — another project's virtualenv or `node_modules`
   > on this machine is not this project's environment, and a suite that passes against it proves
   > nothing about this one. Do not read, copy from, or write to any directory outside this repo.

   Left unsaid, a blocked Codex will find a sibling project's environment on disk and run the
   suite green against it. It reports this honestly in `NOTES`, which is the only reason it gets
   caught — so **always read `NOTES`, and always check which interpreter the `TESTS:` line used.**

Anything it does reach outside the repo is an incident: scrub the other project's identity out of
the WP log before committing, and never let it into `LESSONS.md` (invariant 9).

## Parsing the reply

Read `WP-XXX.out.md` and take the **last** `VERDICT:` line in the file.
- No `VERDICT:` line → treat as `BLOCKED`, record the last 20 lines of the reply in the log, and tell the user. Do not retry blindly.
- `PARTIAL`/`BLOCKED` → read `NOTES`, decide: send a FIX round, amend the spec, or mark the WP `BLOCKED` and escalate to the user.
- Never trust `TESTS:` — you re-run the tests yourself.

## Round limits

- Spec review: **3 rounds** max. Then you decide the final spec and move on; log the unresolved points.
- Fix rounds after failed verification: **3** max. Then mark `BLOCKED` and escalate with the failure output.

## Failure handling

| Symptom | Action |
|---|---|
| non-zero exit / empty `.out.md` | Report the stderr to the user verbatim. Do not retry more than once. |
| Codex asks a question instead of delivering | Answer it in the next resume call, inline. Count it as a round. |
| Codex edited files outside the WP scope | `git -C <repo> diff --stat`, then FIX round: "revert changes to <paths>, they are out of scope". |
| Codex committed despite instructions | Leave the commit; verify it, amend the message to convention if needed. Note it in the log. |
| Session id lost | `codex exec resume --last` once; if it lands in the wrong thread, start a fresh session with a prompt that re-inlines the spec plus the current `git diff`. |
