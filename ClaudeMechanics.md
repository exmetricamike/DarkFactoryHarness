# Claude Mechanics — how context, tokens and billing actually work

Study notes on the machinery underneath Claude Code and the Claude API: what gets sent,
what gets billed, and how to write prompts and harnesses that match the design instead of
fighting it.

Pricing and model facts below are current as of **2026-06-24** (the cached table in the
`claude-api` skill). Verify against <https://docs.claude.com/en/docs/about-claude/pricing>
before you build arithmetic on them.

---

## 1. The one fact everything follows from

**The API is stateless. There is no server-side conversation.**

Every call to `POST /v1/messages` carries the *entire* conversation — system prompt, tool
definitions, every prior user turn, every assistant reply, every tool result — as a fresh
payload. The model has no memory between calls. "Continuing a conversation" means *you*
replay the whole transcript and append one message.

Everything else in this document is a consequence of that single design choice:

- Why token cost grows with the square of turn count.
- Why prompt caching exists and is the largest cost lever there is.
- Why byte-stability of the prefix matters more than prompt wording.
- Why `CLAUDE.md` is cheap and a stray `cat` of a big file is expensive.

---

## 2. Anatomy of a request

The rendered prompt is assembled in a **fixed order**:

```
tools  →  system  →  messages[]
```

| Segment | Contents | Volatility |
|---|---|---|
| `tools` | Tool definitions (JSON schemas). Position 0. | Should be frozen |
| `system` | Top-level system prompt. In Claude Code: the harness prompt + `CLAUDE.md`. | Should be frozen |
| `messages[]` | The whole transcript: user turns, assistant turns, tool calls, tool results | Grows every turn |

That order is the reason the architectural advice all points the same way: **stable content
first, volatile content last**. Anything dynamic placed early poisons everything after it
(§5).

### Roles inside `messages[]`

- `user` — your input, and tool results.
- `assistant` — model output, including `tool_use` blocks and `thinking` blocks.
- `system` — *mid-conversation* operator instructions (Opus 5, Opus 4.8, Fable 5/5.1,
  Mythos 5/5.1; **not** Sonnet 5). This is the cache-preserving way to change instructions
  mid-flight, and it is the non-spoofable operator channel — text injected into a user turn
  can be forged by anything that writes to user-visible input; a `role: "system"` message
  cannot.

---

## 3. What you are billed for

Four meters, not one. Every response carries a `usage` object:

| Field | Meaning | Price |
|---|---|---|
| `input_tokens` | Tokens processed at full price — the **uncached remainder only** | 1× base input |
| `cache_creation_input_tokens` | Tokens written into the cache this request | 1.25× (5-min TTL) / 2× (1-hour TTL) |
| `cache_read_input_tokens` | Tokens served from cache this request | **0.1×** base input |
| `output_tokens` | Everything the model generated, thinking included | Output rate (~5× input) |

> **Total prompt size = `input_tokens` + `cache_creation_input_tokens` + `cache_read_input_tokens`.**
> `input_tokens` alone is *not* the size of your prompt. An agent that ran for hours and
> reports `input_tokens: 4000` sent far more than 4K — the rest was cache reads.

### Current model prices (per million tokens)

| Model | ID | Context | Input | Output |
|---|---|---|---|---|
| Claude Fable 5.1 | `claude-fable-5-1` | 1M | $10.00 | $50.00 |
| Claude Opus 5 | `claude-opus-5` | 1M | $5.00 | $25.00 |
| Claude Sonnet 5 | `claude-sonnet-5` | 1M | $2.00 | $10.00 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | $1.00 | $5.00 |

So on Opus 5: a cache read is **$0.50/MTok**, a 5-minute cache write **$6.25/MTok**, a
1-hour cache write **$10.00/MTok**. (Fable 5.1 is the outlier: cache reads there are
0.025×, i.e. $0.25/MTok.)

### Two things people get wrong

- **Thinking tokens are output tokens.** They are billed at the output rate whether or not
  you can see them. `thinking.display: "omitted"` (the default on Opus 5 and the whole
  5-family) hides them; it does not make them free. `effort` is the knob that changes how
  many there are — `max_tokens` is not.
- **`max_tokens` is a backstop, not a tuning knob.** The model never sees it. Hitting it
  truncates mid-thought and you pay for everything generated up to the cut. In Anthropic's
  coding runs a 16,384 cap ended 15% of Opus 5's attempts, none of them solved — cheaper per
  attempt, no better per *solved task*. Set 64,000 for agentic work and stream.

---

## 4. Prompt caching — the mechanism

### The invariant

**Prompt caching is a prefix match. Any byte change anywhere in the prefix invalidates
everything after it.**

The cache key is derived from the exact bytes of the rendered prompt up to each
`cache_control` breakpoint. One differing byte at position N kills every breakpoint at
position ≥ N. Not "mostly the same" — *byte-identical*.

### Declaring it

```json
"cache_control": {"type": "ephemeral"}
"cache_control": {"type": "ephemeral", "ttl": "1h"}
```

The first is the default 5-minute TTL; the second is the 1-hour TTL.

- Maximum **4 breakpoints** per request.
- Goes on any content block: system text, tool definitions, `text`, `image`, `tool_use`,
  `tool_result`, `document`.
- Top-level `cache_control` on `messages.create()` auto-places on the last cacheable block.
  Simple, and correct for a single growing conversation. It does *not* help when many
  independent conversations share a static prefix — that needs an explicit breakpoint on the
  shared part.

### Minimum cacheable prefix (model-dependent)

Below the minimum, the marker is silently ignored — no error, just
`cache_creation_input_tokens: 0`.

| Model | Minimum |
|---|---:|
| Opus 5, Fable 5, Fable 5.1, Mythos 5/5.1 | 512 tokens |
| Opus 4.8, Sonnet 5, Sonnet 4.6/4.5 | 1,024 tokens |
| Opus 4.7, Haiku 3.5 | 2,048 tokens |
| Opus 4.6, Opus 4.5, Haiku 4.5 | 4,096 tokens |

Not monotonic across generations. A 3K-token prompt caches on Opus 5 and silently does not
on Haiku 4.5.

### Choosing the TTL

A cache read **refreshes the timer for free**, on either TTL. Lifetime is measured from the
*start* of the request that writes or reads it — generation time counts against it, so a
4-minute generation leaves ~1 minute for the next request to start.

| Start-to-start gap between requests sharing the prefix | TTL |
|---|---|
| Under 5 min (continuous traffic, tight agent loops) | 5-minute — every request refreshes it; strictly cheaper |
| 5–60 min (a human replying after 20 min; a long side-task) | 1-hour — the only window where the 2× write pays off |
| Over an hour | Neither. Re-warm on a schedule, or accept the miss |

**Break-even:**

- 5-min TTL: 2 requests. `1.25× + 0.1× = 1.35×` beats `2×` uncached.
- 1-hour TTL: 3 requests. `2× + 0.2× = 2.2×` beats `3×` uncached.

### Scope and isolation

Caches are **per-model** and **per-workspace** (per-organization on Bedrock and Vertex).
Never shared across organizations. Switching models mid-conversation forfeits the entire
cache with no escape hatch — which is exactly why a cheap-model subagent starts cold rather
than inheriting the parent's prefix.

### The 20-block lookback window

Each breakpoint walks backward **at most 20 positions** looking for a prior entry. A run of
consecutive `tool_use` blocks counts as one position, and so does a run of consecutive
`tool_result` blocks — so parallel tool calls are safe. A turn that appends more than 20
positions of *other* content (long sequential tool loops, many text/image blocks) pushes the
previous entry out of the window, and every request silently rewrites the whole
conversation with byte-identical payloads.

Fix: an intermediate breakpoint every ~15 positions in long turns.

### Concurrency

A cache entry becomes readable only after the first response **begins streaming**. N
parallel requests with identical prefixes all pay full price — none can read what the others
are still writing.

For fan-out: send 1 request, await the first streamed token, *then* fire the remaining N−1.

### Pre-warming

`max_tokens: 0` runs prefill, writes the cache, returns immediately with `content: []` and
zero output tokens billed. Put the breakpoint on the last block shared with the real request
(system or tools) — **not** on the placeholder user message.

Worth it only when all three hold: first-request latency is user-visible; the shared prefix
is large; and there is a quiet moment before traffic (startup, worker boot, post-deploy).
Skip it entirely when traffic is continuous — the first real request warms it and a separate
warm call is a pure extra write.

---

## 5. Silent invalidators

The costliest caching failure is silent: requests keep succeeding, the bill is just higher.
The usual shape is a *regression* — caching worked when written, then a later change to
prompt assembly broke it and nobody noticed for months.

| Pattern | Why it breaks caching |
|---|---|
| `datetime.now()` / `Date.now()` in the system prompt | Prefix changes every request |
| `uuid4()` / request IDs early in content | Every request is unique |
| `json.dumps(d)` without `sort_keys=True`; iterating a `set` | Non-deterministic bytes |
| Session/user ID interpolated into the system prompt | Per-user prefix, no sharing |
| Conditional system sections (`if flag: system += ...`) | Every flag combo is a distinct prefix |
| `tools=build_tools(user)` | Tools render at position 0 — nothing caches across users |
| Changing `thinking` or `effort` between requests | Always invalidates the messages cache |
| Switching models mid-conversation | Caches are model-scoped |
| Injecting a reminder then deleting it next turn | A history edit — cache misses from that point |

### The invalidation hierarchy

Not every change invalidates everything. Three tiers; a change invalidates its own tier and
below. In the table, "kept" means that cache survives:

| Change | Tools cache | System cache | Messages cache |
|---|---|---|---|
| Tool definitions (add/remove/reorder) | lost | lost | lost |
| Model switch | lost | lost | lost |
| System prompt content | kept | lost | lost |
| `tool_choice`, images | kept | kept | lost |
| Message content | kept | kept | lost |

The useful implication: **message-content changes never touch the tools+system cache.**
Appending to the conversation is cheap. Editing the system prompt is not.

Three rows have a cache-preserving escape hatch — move the change out of the top-level
request and into a `role: "system"` message inside `messages[]`, *after* the cached prefix:

| Top-level change | Cache-preserving form | Available on |
|---|---|---|
| System prompt content | `{"role": "system", "content": "..."}` message | Opus 5, Opus 4.8, Fable 5/5.1 — **today, no beta header** |
| Tool add/remove | `tool_addition` / `tool_removal` blocks | Opus 5+, beta `mid-conversation-tool-changes-2026-07-01` |
| `effort` change | system message with `content: []` + `output_config` | Fable 5.1, Opus 5, beta `mid-conversation-output-config-2026-07-01` |

---

## 6. Why agentic loops cost what they do

Each turn resends the whole growing conversation. A 40-turn task sends its first turn 40
times. **Task cost grows with roughly the square of turn count.**

Worked example, Opus 5, input only. Say the prefix starts at 15K tokens (system +
`CLAUDE.md` + tools) and each turn adds ~3K:

```
tokens sent across 30 turns = Σ (15K + 3K·n)  for n = 1..30
                            = 450K + 3K·465
                            = 1,845K tokens
```

- Uncached: 1.845M × $5/MTok = **$9.23**
- Cached (idealized): ~1,755K reads at $0.50 = $0.88, plus ~90K writes at $6.25 = $0.56
  → **~$1.44**

That idealized ratio (~6×) is an upper bound. **Anthropic's measured figure for real agent
loops is a 2.5×–3.7× reduction at 81–90% hit rates**; one issue-triage agent's bill fell
83% from caching alone. Real loops have misses, and output tokens don't cache at all.

The lesson that transfers to *any* harness: **the cheapest turn is the one you don't take**,
and the second cheapest is the one whose prefix was byte-identical to the last.

### The healthy-loop signature

In a steady multi-turn loop, the meters should look like this every request:

- `cache_read_input_tokens` — the whole prior prefix; **grows** turn over turn
- `cache_creation_input_tokens` — roughly the last assistant output plus the new input;
  **small** relative to the conversation
- `input_tokens` — just the tail after the last breakpoint

If `cache_creation_input_tokens` is near full conversation size every request, something
upstream is rewriting the prefix. Diff consecutive request payloads: the previous request's
prompt must reappear **unchanged** as a prefix of the next. Strip `cache_control` markers
before diffing — the moving marker always differs and is not the bug. The first divergence
inside the overlapping region is the invalidation point.

On the first-party API, cache diagnostics does this server-side: beta header
`cache-diagnosis-2026-04-07` on **every** request (fingerprints are only stored for requests
that carried it), then pass the previous response `id` as `diagnostics.previous_message_id`.

---

## 7. Strategies — as a Claude Code user

Claude Code is one long agentic loop, so §6 is your bill. The levers you actually control:

**Turn count is the primary cost driver.** Every message you send re-bills the whole
transcript. Ten small clarifying exchanges cost more than one well-formed request. Front-load
the spec: give the full task, the constraints, and the acceptance criterion in one message.
This is also why batching independent tool calls into one response matters — it is the same
saving on the model's side.

**Tool output is permanent.** A `cat` of a 2,000-line file, or an unfiltered `grep`, enters
the transcript and rides in every subsequent request forever. It is far more expensive than
anything in `CLAUDE.md`. Prefer `sed -n '120,180p'`, `rg` with a path filter, `head`. Reading
the *right* 40 lines beats reading the file.

**Keep `CLAUDE.md` behavioral, not documentary.** It sits in the frozen prefix, which is the
cheapest real estate available (0.1× after the first turn) — but it is re-billed on every
single request for the life of the session. Rules that change behavior earn their place;
documentation a reader could derive from the code does not. This repo's `CLAUDE.md` is
~2,800 tokens ≈ 280 tokens-equivalent per cached turn. That is fine. A 20K-token one is not.

**Load instructions lazily.** Skills and slash commands are the right pattern precisely
because they enter context only when invoked. This repo has ~66KB of `.claude/skills/` and
`.claude/commands/` — 7× the size of `CLAUDE.md` — and pays for none of it until a command
runs. Subdirectory `CLAUDE.md` files work the same way.

**`/clear` is the cheapest optimization in the tool.** A finished task's transcript is dead
weight in every subsequent request. New task → new context. `/compact` is the middle option
when you need continuity but not the full history.

**Don't edit `CLAUDE.md` mid-session expecting effect.** The loaded copy is a snapshot in the
prefix. Changing the file on disk does nothing until restart or `/clear`. (`#` appends write
to both.)

**Subagents start cold and are billed separately.** A subagent gets a fresh prefix and shares
no cache with you. That is a *feature* when the sub-task is reading-heavy — the subagent
absorbs a hundred thousand tokens of file contents and hands back one paragraph, and your
main loop never pays to carry them. It is waste when the sub-task is small: you paid a full
cold prefix to save nothing. Spawn for bulky, self-contained exploration; do small things
inline.

**Long gaps cost real money.** Walk away for 90 minutes and the cache is gone; the next
message pays full price for the entire transcript plus a fresh write. If you are stepping
away mid-task, `/clear` and restarting later is often cheaper than resuming cold into a huge
transcript.

---

## 8. Strategies — as an API builder

Levers in the order they should be applied. **Free wins before tradeoffs** — never trade
accuracy for cost until the free levers are exhausted.

### 8.1 Caching (largest lever, always on)

Design the prompt-building path around prefix stability. Classify every input:

| Stability | Where it belongs |
|---|---|
| Never changes | Early, before any breakpoint |
| Per-session | After the global prefix |
| Per-turn | At the end |
| Per-request (timestamps, UUIDs) | **Eliminate**, or move to the very end |

Then place breakpoints at the stability boundaries. The robust shape for agent loops is one
explicit breakpoint on the static prefix, plus top-level automatic caching for the tail.

**Shared prefix, varying suffix** is the pattern people get backwards. Put the breakpoint at
the end of the *shared* portion, not the end of the whole prompt — otherwise every request
writes a distinct entry and nothing is ever read:

```json
"messages": [{"role": "user", "content": [
  {"type": "text", "text": "<shared context>", "cache_control": {"type": "ephemeral"}},
  {"type": "text", "text": "<varying question>"}
]}]
```

**Fork operations must reuse the parent's exact prefix.** Summarizers, compactors and
sub-agents that rebuild `system`/`tools`/`model` with any difference miss the parent's cache
entirely. Copy verbatim, append at the end.

**Verify from `usage`, not from code review** — and re-verify after every prompt-assembly
change. Ship a standing assertion: a second identical request must show
`cache_read_input_tokens > 0`.

### 8.2 Input hygiene — progressive disclosure

Send what the task needs; let the model fetch the rest.

- Large reference doc in every prompt → put it behind a tool. *Skip when* most calls consult
  most of it — a doc in the cached prefix is cheap.
- Tool recaps in the system prompt → delete. Schemas already render into the request.
- Heavy tool schemas → `defer_loading: true` + tool search. Pays once schemas exceed ~10K
  tokens (MCP servers get there fast); below that the search step is overhead.
- Images/PDFs at full resolution → pre-downscale. Vision cost scales with **pixel area**
  (~1 token per 28×28 patch), not information content. 1280×720 caps an image near 1,200
  tokens.
- Chained tool calls whose intermediates don't matter → programmatic tool calling; only the
  filtered result enters context. Reported 24% fewer input tokens on agentic search, with a
  *higher* score.
- Broad data-dump tools → narrow accessors. `get_policy(claim_id)`, not `get_all_policies()`.
- Prompts written for an older model → audit them. On a support-desk eval, prompts written
  for Opus 4.8 cost **36% more per ticket** on Opus 5 for no accuracy gain.

**Caveat governing all of the above:** a smaller prefix is not automatically a cheaper task.
Deferred context means discovery turns. Validate against an eval.

### 8.3 Loop hygiene

- **Context editing is a context-window tool, not a savings lever.** Every clearing pass
  rewrites the cached conversation and works *against* caching. In Anthropic's measured run
  it cost more than it saved. Use it to make room; trigger rarely, clear in large batches.
- **Compaction** (server-side summarize-and-continue, beta `compact-2026-01-12`, default
  trigger 150K) needs sessions long enough to fire; where it did on a long triage run it cut
  the bill a further 38%. Critical: append `response.content` back, not just the text —
  compaction blocks carry the state.
- **Prune at natural boundaries**, keeping the message array byte-identical between prunes,
  so each prune is one cold miss rather than a new miss every turn.

### 8.4 Batch API

**50% off every token — including cache reads and writes. The discounts stack.** The
second-largest free lever for unattended work: evals, backfills, scheduled jobs. Results
within 24 hours (an expiry, not an SLA). Single-shot — no mid-batch tool loop. Keep
user-facing work synchronous.

### 8.5 Effort — the first real tradeoff

`output_config: {effort: "low"|"medium"|"high"|"xhigh"|"max"}` — inside `output_config`, not
top-level. Default is `high`.

Which workloads repay higher effort is a property of the workload: coding and long-horizon
agentic work respond strongly; chat, classification and high-volume routes often don't and
do fine at `low`. **Measure per route, not globally.** Judge cost *per completed task*, not
per request — a cheaper request that needs three more turns is not cheaper.

Before building a multi-model cost cascade, measure the simpler alternative: **the most
capable model at lower effort**. Lower effort on a newer model often matches high effort on
the previous generation, and one model means one cache namespace — a cascade forfeits cache
reuse across its models.

---

## 9. Quick reference

| Question | Answer |
|---|---|
| Is history stored server-side? | No. Every request resends everything. |
| What does a cache hit cost? | 0.1× base input (0.025× on Fable 5.1) |
| What does a cache write cost? | 1.25× (5-min) / 2× (1-hour) |
| How many breakpoints? | 4 max |
| Smallest cacheable prefix, Opus 5? | 512 tokens |
| Does a cache read extend the TTL? | Yes, free, measured from request start |
| Break-even, 5-min TTL? | 2 requests |
| Break-even, 1-hour TTL? | 3 requests |
| Render order? | `tools` → `system` → `messages` |
| Does changing message content break the system cache? | No |
| Does changing the system prompt break the messages cache? | Yes — use a `role: "system"` message instead |
| Does switching models keep the cache? | No. Model-scoped, no escape hatch. |
| Are thinking tokens billed? | Yes, at the output rate, visible or not |
| Is `max_tokens` a cost knob? | No. It's a truncation backstop. Use `effort`. |
| Batch discount? | 50% on everything, stacks with caching |
| Biggest single lever? | Caching — 2.5×–3.7× on measured agent loops |
| Biggest lever in Claude Code specifically? | Fewer turns, smaller tool output, `/clear` |

---

## Sources

- `claude-api` skill: `shared/prompt-caching.md`, `shared/cost-optimization.md`
  (bundled with Claude Code)
- <https://docs.claude.com/en/docs/build-with-claude/prompt-caching>
- <https://docs.claude.com/en/docs/about-claude/pricing>
- Run `/claude-api cost-optimize` against a real codebase for a measured audit.
