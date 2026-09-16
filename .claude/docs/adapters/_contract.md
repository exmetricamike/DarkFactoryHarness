# Adapter contract

An **adapter** is how the harness talks to one kind of backend (a CLI agent, an API, a local
runner). A **profile** is one configured instance of an adapter — adapter + model + limits.
`harness.config.json` binds the two roles, `reviewer` and `implementer`, to profiles.

Resolution, done once at the start of every phase command:

```
harness.config.json -> roles.<role> -> profiles.<id>.adapter -> .claude/docs/adapters/<adapter>.md
```

Load **only** the adapter doc(s) the active roles resolve to. If both roles resolve to the same
profile, that is one doc, one session, and the run behaves exactly as a single-backend run.

## Config schema

| Key | Meaning |
|---|---|
| `roles.reviewer` / `roles.implementer` | profile id. Same id for both = one backend does everything. |
| `profiles.<id>.adapter` | which adapter doc governs the mechanics |
| `profiles.<id>.label` | human line for reports and the morning report |
| `profiles.<id>.invoke` | adapter-specific launch parameters (model, provider, sandbox, flags) |
| `profiles.<id>.capabilities` | what the backend can do — the protocol branches on these |
| `profiles.<id>.limits` | round limits, call timeout, prompt budget |
| `profiles.<id>.preflight` | cheap reachability check + the hint to print when it fails |
| `profiles.<id>.fallback` | profile id to take over on repeated failure, or `null` for none |

### capabilities

| Flag | If false, the protocol must |
|---|---|
| `edits_files` | not send IMPLEMENT blocks to it — it can only review |
| `runs_commands` | not ask it for a `TESTS:` line; the coordinator runs everything |
| `sessions` | re-inline prior rounds into every prompt instead of resuming |
| `network` | tell it in every prompt that dependencies are pre-installed and it must not fetch |
| `context_tokens` | (number) size WPs and trim prompt sections to fit |

### limits

`spec_review_rounds` and `fix_rounds` replace the fixed 3/3 in the protocol.
`call_timeout_ms` is the Bash timeout for one call. `max_prompt_chars` is the prompt budget
(`null` = no guard).

## What an adapter doc must specify

Seven things, in this order, each with a copy-pasteable command. Anything an adapter cannot do
is stated as *cannot*, never left out — the protocol needs the negative as much as the positive.

1. **Preflight** — version check plus one cheap call that proves auth and reachability. A version
   print is not a reachability check.
2. **Session start** — first call of a WP. Takes repo path, prompt file, output file.
3. **Session resume** — follow-up rounds in the same context. If the backend has no sessions, say
   so here and point at 5.
4. **Session id capture** — how the id is obtained and what to do when capture fails.
5. **No-session fallback** — what a prompt must re-inline when context does not carry over.
6. **Write scope** — what the backend may read and write, how that is enforced, and what happens
   when it is asked to touch two repos.
7. **Failure signatures** — exact strings/exit codes for quota, unreachable endpoint, empty reply,
   and refusal, so `continuity` §B can tell "backend is down" from "backend did badly".

Plus a `verified-against:` line naming the exact tool version the commands were checked on.
On a version mismatch, re-check the tool's help output before trusting the commands.

## Adding an adapter

1. Write `.claude/docs/adapters/<name>.md` answering all seven points.
2. Add a profile to `harness.config.json` using it, with honest `capabilities`.
3. Preflight it (`/df-implementer --smoke`) before any unattended run.

Nothing else in the harness changes. If a new adapter needs a change to a phase command, the
contract is leaking — fix the contract instead.

## Example: a second profile on the same adapter

```json
"lmstudio-qwen3-coder": {
  "adapter": "codex",
  "label": "Qwen3-Coder 30B via LM Studio, driven by the Codex agent loop",
  "invoke": { "oss": true, "local_provider": "lmstudio", "model": "qwen/qwen3-coder-30b",
              "sandbox": "workspace-write", "approve_for_me": true, "config_overrides": [] },
  "capabilities": { "edits_files": true, "runs_commands": true, "sessions": true,
                    "network": false, "context_tokens": 32000 },
  "limits": { "spec_review_rounds": 2, "fix_rounds": 2, "call_timeout_ms": 1800000,
              "max_prompt_chars": 60000 },
  "preflight": { "check": "curl -sf http://localhost:1234/v1/models",
                 "hint": "lms server start && lms load qwen/qwen3-coder-30b" },
  "fallback": null
}
```

Same adapter, different model: the agent loop, sandbox and session handling are unchanged, only
the model underneath differs. That is the cheapest way to swap an implementer, and it isolates
the model as the single variable of the experiment.
