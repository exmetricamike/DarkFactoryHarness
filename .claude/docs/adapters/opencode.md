# Adapter: opencode

`verified-against: NOTHING YET` — OpenCode is not installed on this machine, so the commands below
come from the official docs (opencode.ai/docs/cli, /docs/config, /docs/providers), not from a run.
**Treat every command here as a hypothesis until `/df-implementer --smoke` passes.** Three specifics
are flagged CONFIRM below; settle them on the first smoke run and replace this line with the real
`opencode --version`.

Placeholders: `<REPO>` target repo, `<H>` harness dir (absolute), `<P>` prompt file, `<O>` out file.
`<FLAGS>` is built from the active profile's `invoke` block:

| `invoke` key | Flag |
|---|---|
| `model` | `-m <provider/model>` — always provider-prefixed, e.g. `lmstudio/qwen/qwen3-coder-30b` |
| `config_file` | not a flag: exported as `OPENCODE_CONFIG=<H>/<path>` |
| `auto: true` | `--auto` (auto-approve permissions; required unattended) |
| `agent` | `--agent <name>` (omit when null) |
| `extra_args` | appended verbatim |

`OPENCODE_CONFIG` layers the harness's provider definitions over the user's own OpenCode setup
instead of replacing it. The files live in `.claude/adapters/opencode/`; never edit
`~/.config/opencode/opencode.json` from the harness.

## 1. Preflight

```bash
opencode --version
[ -n "<env_file>" ] && set -a && . "<H>/<env_file>" && set +a
<profile preflight.check>
cd "<REPO>" && OPENCODE_CONFIG="<H>/<config_file>" \
  opencode run <FLAGS> "Reply with exactly one line naming this repo's primary language. Do not modify any files." \
  > /tmp/smoke.out.md 2>/tmp/smoke.err
```

Empty `smoke.out.md`, or an error naming the provider or the model id, means the profile is not
usable tonight. A local endpoint that answers `/v1/models` but not a completion usually means the
model is not loaded — run the profile's `preflight.hint`.

**CONFIRM #1:** whether `opencode run` accepts the prompt on **stdin** (`- < <P>`). The documented
form is a positional message, so the harness passes `"$(cat <P>)"`, which is safe for prompts up to
`ARG_MAX` (~1 MB on macOS — far above any prompt budget here). If stdin works, prefer it.

## 2. Session start (first call of a WP)

```bash
set -a; [ -n "<env_file>" ] && . "<H>/<env_file>"; set +a
cd "<REPO>" && OPENCODE_CONFIG="<H>/<config_file>" \
  opencode run <FLAGS> --title "WP-XXX <role>" --format json \
  "$(cat "<H>/active-project/wps/WP-XXX.prompt.md")" \
  > "<H>/active-project/wps/WP-XXX.json" 2>"<H>/active-project/wps/WP-XXX.err"
```

Then extract the assistant's text into `<O>` so the rest of the protocol reads one file as usual.
Apply the profile's `limits.call_timeout_ms` as the Bash timeout; a local 30B model will use most of it.

## 3. Session resume (every later round)

```bash
rm -f "<H>/active-project/wps/WP-XXX.out.md"          # the stale-reply trap applies here too
cd "<REPO>" && OPENCODE_CONFIG="<H>/<config_file>" \
  opencode run <FLAGS> --session "<SESSION_ID>" "$(cat "<H>/active-project/wps/WP-XXX.prompt.md")" \
  > "<H>/active-project/wps/WP-XXX.out.md" 2>"<H>/active-project/wps/WP-XXX.err"
```

`--continue` (`-c`) resumes the most recent session instead, and is the fallback when the id was not
captured — safe here only because the harness runs one WP at a time (invariant 3). `--fork` branches
off a resumed session; the harness does not use it.

> **The stale-reply trap.** Redirection truncates the out file even when the command fails, which is
> safer than codex's `-o`, but a *partial* write still reads like a real reply. `rm -f` first, check
> the exit code, and check the file is non-empty before parsing a verdict out of it.

## 4. Session id capture

**CONFIRM #2:** the exact field holding the session id in `--format json` output. Inspect the first
smoke reply (`python3 -m json.tool < WP-XXX.json | head -40`), find the id, and write the extraction
here as a one-line `jq`/`python3` command. Until then: capture nothing, use `--continue` for
follow-up rounds, and record `Session: --continue (id field not yet confirmed)` in the WP log.

## 5. No-session fallback

Sessions exist (`capabilities.sessions: true`). If both `--session` and `--continue` prove unreliable,
set `sessions: false` on the profile and re-inline the spec, the prior round's issue list or failure
output, and the current `git -C <REPO> diff` into every prompt. Slower, never wrong.

## 6. Write scope

**There is no workspace sandbox.** This is the material difference from the codex adapter, and the
reason these profiles carry `capabilities.sandboxed: false`. `--dir` sets the working directory; it is
not a jail. OpenCode runs with your user's permissions, and `--auto` approves its own actions.

Consequences, all mandatory:

- Run it with `cd "<REPO>"` and `--dir` pointing at that repo, so its default working set is right.
- Keep the "do not read, copy from, or write to any directory outside this repo" paragraph in every
  prompt — for codex it prevents a mistake, here it is the actual control.
- The diff review in `/df-build` step 3 is the only thing that catches a violation. Also check
  `git -C <OTHER_REPO> status` on a cross-repo project before committing.
- Think before pointing this at a machine holding client work or credentials. That is a judgement
  about the machine, not about the model.

`capabilities.network` is `true` for these profiles: unlike codex, OpenCode can install dependencies,
so the no-network block is omitted from IMPLEMENT prompts — but the coordinator still provisions the
environment itself, because an implementer inventing its own dependency tree is a diff problem.

## 7. Failure signatures

**CONFIRM #3:** the exact error strings. Fill these in from the first real failures rather than
guessing; until then, treat any non-zero exit with an empty out file as backend-down after one retry.

| Symptom | Reading |
|---|---|
| exit 0, out file written, no `VERDICT:` | bad reply — one re-ask, then BLOCKED |
| `connection refused` / `fetch failed` / `ECONNREFUSED` on a local port | server not running → `preflight.hint`, retry once, then `continuity` §B |
| provider or model id not found | `invoke.model` disagrees with the config file's `models` block or with what the server serves — a config error, not an outage. Fix it; do not retry. |
| `401` / `invalid api key` on OpenRouter | `.env` not sourced or the key is wrong → `continuity` §B, and say which env var |
| `429` / `rate limit` / `insufficient credits` (OpenRouter) | quota → `continuity` §B |
| exit 0, out file empty, no error | usually a model that returned nothing for an over-long prompt: check the prompt against `limits.max_prompt_chars` before retrying |
