---
description: Show, switch and smoke-test the backends bound to the reviewer and implementer roles
argument-hint: [profile] | [--reviewer X] [--implementer Y] | [--smoke]
---

# /df-implementer $ARGUMENTS — who is building tonight

Reads and writes `harness.config.json`. Schema: `.claude/docs/adapters/_contract.md`.
This is the one place the backend changes. Never edit a role binding from inside a phase command.

## No arguments — report

Read `harness.config.json` and print exactly this:

```
reviewer:    <profile-id>  (<adapter>, <label>)  <model or "default">  <verified|UNVERIFIED>
implementer: <profile-id>  (<adapter>, <label>)  <model or "default">  <verified|UNVERIFIED>
limits:      review <n> rounds, fix <n> rounds, timeout <n>s, prompt budget <n chars or "none">
sandbox:     <workspace-jailed | NOT sandboxed — prompt containment only>   network: <yes|no>
secrets:     <env_file: which env_required vars are set | none needed>
fallback:    <profile-id or "none">
preflight:   <not run this session | PASS <what you ran> | FAIL <the error>>
available:   <the other profile ids in the file>
```

Never print the value of a secret — only whether it is set.

## `<profile>` — switch both roles

Set `roles.reviewer` and `roles.implementer` to that profile id. Unknown id → list the valid ones and
stop; never invent a profile.

## `--reviewer X` / `--implementer Y` — switch one role

Either or both. A split configuration is normal: the reviewer reads the repo and argues about the
spec, the implementer writes the code, and they need not be the same model or even the same adapter.

After any switch: re-validate the JSON, print the report above, and run `--smoke` on whatever changed.

**Never switch a role while a WP is in flight** (`BUILDING`/`VERIFY` in the backlog). Finish or block
the WP first — a WP that changes backend mid-flight has no reviewable provenance. State that and stop.

## `--smoke` — prove it works before the night does

Per distinct profile bound to a role, in order:

0. If the profile names an `env_file`, source it (`set -a; . ./<env_file>; set +a`) and check every
   `env_required` var is non-empty. Missing → report which one and stop there for that profile. A
   missing key found now is a nuisance; found at 3am it is a dead night.
1. Run `preflight.check` from the profile. Fail → print `preflight.hint` and stop there for that profile.
2. Load `.claude/docs/adapters/<adapter>.md` and run its §1 preflight call against a repo:
   the first repo in `active-project/PROJECT.md`, or this harness directory if no project is active.
3. Read the output file. Empty or missing → **FAIL**, and say so plainly. A version print is not a
   reachability check, and an exit code of 0 with an empty reply is the most common false pass.
4. Record the result and the timestamp in your report. A profile that just passed for the first time
   → set its `verified: true` in `harness.config.json` and say you did.

One cheap call each. It costs seconds now and saves the whole night: a backend that cannot be reached
at 23:00 means the run happens entirely on the `continuity` §B takeover ladder, and you find that out
at 03:00 with half a backlog left.

## Adding a profile

Follow `.claude/docs/adapters/_contract.md` — add the profile with `verified: false`, be honest in
`capabilities`, then `--smoke` it. A profile that has never passed a smoke test is not usable in an
unattended run; say so rather than letting `/df-run` discover it.

Secrets go in `.env` (git-ignored, template in `.env.example`), named in the profile's `env_required`.
Local profiles need none — that is one of the reasons to try one.

## Recording it

The active profile ids belong in the WP log, the commit trailer (`Implemented-by:`), and the morning
report. When comparing backends across runs, that trail is the experiment's only evidence.
