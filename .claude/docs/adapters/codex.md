# Adapter: codex

`verified-against: codex-cli 0.149.0` (flags checked against `codex exec --help` / `codex exec resume --help`).
Different major/minor → re-check those two help outputs before trusting anything below, and update this line.

Placeholders: `<REPO>` target repo path, `<H>` this harness dir (absolute), `<P>` the WP prompt file,
`<O>` the WP out file. `<FLAGS>` is built from the active profile's `invoke` block:

| `invoke` key | Flag |
|---|---|
| `model` | `-m <model>` (omit when `null`) |
| `oss: true` | `--oss` |
| `local_provider` | `--local-provider lmstudio\|ollama` |
| `sandbox` | `-s <mode>` |
| `approve_for_me: true` | `--approve-for-me` |
| `config_overrides: ["k=v", …]` | one `-c k=v` each |

A profile with an `env_file` is called with it sourced first: `set -a; . "<H>/<env_file>"; set +a`.
Local profiles (`oss: true`) need no key at all.

## 1. Preflight

```bash
codex --version
codex exec --cd "<REPO>" <FLAGS> -s read-only -o /tmp/smoke.out.md \
  "Reply with exactly one line naming this repo's primary language. Do not modify any files."
```

An empty `smoke.out.md` means the night would have run entirely on the backend-down ladder
(`continuity` §B). Find that out now, not at 1am. For a local provider, also confirm the server is
up and the model loaded with the profile's `preflight.check` before the smoke call.

## 2. Session start (first call of a WP)

```bash
codex exec --cd "<REPO>" <FLAGS> --json \
  -o "<H>/active-project/wps/WP-XXX.out.md" \
  - < "<H>/active-project/wps/WP-XXX.prompt.md" \
  | tee "<H>/active-project/wps/WP-XXX.jsonl"
```

Give the Bash tool the profile's `limits.call_timeout_ms`. Longer than 10 min → `run_in_background: true`
and collect the reply from `<O>`.

## 3. Session resume (every later round)

**`codex exec resume` does NOT accept `--cd`, `-s/--sandbox` or `--approve-for-me`** — passing `--cd`
fails with `error: unexpected argument '--cd' found`, exit 2. Set the directory with a shell `cd`, and
use absolute paths for `-o` and the stdin redirect, because the `cd` changes what relative paths mean:

```bash
rm -f "<H>/active-project/wps/WP-XXX.out.md"          # stale-reply trap, see below
cd "<REPO>" && codex exec resume "<SESSION_ID>" \
  -o "<H>/active-project/wps/WP-XXX.out.md" \
  - < "<H>/active-project/wps/WP-XXX.prompt.md"
```

Sandbox, model and provider carry over from the session. Override with `-c`, e.g.
`-c sandbox_mode="workspace-write"` when the first call was `-s read-only`.

> **The stale-reply trap.** `-o` is only written when the call succeeds. A failed resume leaves the
> *previous* round's `.out.md` in place, and reading it looks exactly like the backend repeating
> itself verbatim — which is the tell. Always `rm -f` the out file first and check the exit code.

## 4. Session id capture

```bash
grep -oE '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' \
  "<H>/active-project/wps/WP-XXX.jsonl" | head -1
```

Write it to the WP log as `Session: <uuid>` and into the backlog row. Empty grep → use
`codex exec resume --last` for the *immediately* following call only, and record
`Session: --last (id capture failed)`.

## 5. No-session fallback

Not needed — this adapter has sessions (`capabilities.sessions: true`). If capture fails twice,
treat the WP as sessionless for the rest of its life: every prompt re-inlines the spec, the prior
round's issue list or failure output, and the current `git -C <REPO> diff`.

## 6. Write scope

`-s workspace-write` confines writes to the repo passed with `--cd`, plus any `--add-dir`.
One repo per call. A WP touching both repos = two calls (backend first unless the spec says
otherwise), or one call with `--add-dir <OTHER_REPO>` when the change must be atomic across both.
`--skip-git-repo-check` only if the target is not a git repo — it should be; fix that instead.

**No network in the sandbox.** Package installs fail. Provision the environment yourself before the
implement call, and forbid the workaround explicitly in every IMPLEMENT prompt (the wording is in the
`implementer-protocol` skill). A blocked Codex will otherwise find a sibling project's virtualenv or
`node_modules` on disk and run the suite green against it.

## 7. Failure signatures

| Symptom | Reading |
|---|---|
| exit 0, `.out.md` written, no `VERDICT:` | bad reply — one re-ask, then BLOCKED |
| exit ≠ 0 within seconds, no file changes, stderr matches `usage limit`/`rate limit`/`quota`/`429`/`too many requests`/`insufficient`/`try again (after\|in)`/`resets (at\|in)` | backend down → `continuity` §B |
| exit ≠ 0, `error: unexpected argument` | a flag from §3 was passed to `resume` — fix the call, not the prompt |
| exit 0, `.out.md` empty or unchanged mtime | failed call with a stale file — see the trap in §3 |
| local provider: `connection refused`, `Failed to connect`, empty model list | server not running or model not loaded → run `preflight.hint`, retry once, then §B |
| stream stalls with no output past the timeout | kill it, treat as one failed round, resume the session with the same prompt once |

## 8. Providers other than the hosted default

**Local models — supported directly.** `--oss --local-provider lmstudio|ollama` is built in; set
`oss: true` and `local_provider` on the profile and pass the served model id with `-m`. The id must
match what the server actually serves (`lms ls`, `ollama list`) — a wrong id fails as a provider
error, not as a bad answer, so it is a config bug and never worth a retry. Everything else is
unchanged: same agent loop, same `workspace-write` sandbox, same sessions. That is what makes a
local-vs-hosted comparison on this adapter a clean one-variable experiment.

**OpenRouter — unverified, and here is the specific doubt.** It needs a custom provider in
`~/.codex/config.toml`:

```toml
[model_providers.openrouter]
name = "OpenRouter"
base_url = "https://openrouter.ai/api/v1"
env_key = "OPENROUTER_API_KEY"
```

then `-c model_provider="openrouter" -m <openrouter/model-id>`, with `OPENROUTER_API_KEY` sourced
from `.env`. The catch: OpenAI's config reference documents `wire_api = "responses"` as the only
supported protocol, and OpenRouter serves chat-completions. If the smoke call fails on the wire
format, that is the reason, and it is not something the harness can work around — use the
`opencode-openrouter` profile instead, where OpenRouter is a first-class provider.

Verify before relying on either: the `model_providers` keys above are `name`, `base_url`, `env_key`,
`wire_api`, `http_headers`, `query_params`, per learn.chatgpt.com/docs/config-file/config-reference.
