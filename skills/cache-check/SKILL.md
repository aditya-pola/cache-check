---
name: cache-check
description: Triage prompt-caching audit for a project that calls Claude. Detects which provider and integration path the project uses (Anthropic API direct, AWS Bedrock, a framework like LangChain, a gateway like LiteLLM) and routes to the right deep-dive skill. Use when the user asks to review their Claude caching, reduce Anthropic API cost, unblock ITPM rate limits, or improve cache hit rate on their own codebase. Not for auditing Claude Code's internal sessions — this is for user projects that call Claude through the API.
---

# cache-check — triage

You're the entry point. A user wants a prompt-caching review of their repo. Your job: detect integration path, route to the right sub-skill, drive to a clear outcome.

## Usage patterns — what "done" looks like

**Pattern A: Full audit**
Trigger: "run audit", "review caching", "optimize my prompts"
1. Add telemetry → measure baseline (run code, capture `cache_read`/`cache_creation`)
2. Diagnose → identify issues (byte drift, threshold, placement)
3. Suggest fixes → show diffs
4. Apply fixes (with approval)
5. Measure again → run code, capture new metrics
6. Report delta → "before: 0% hit rate, after: 78% hit rate, estimated savings: $X/mo"

**Pattern B: Quick check**
Trigger: "is caching working?", "check my cache hit rate"
1. Add minimal telemetry (5-line logger)
2. Run twice, same prompt
3. Report: "Yes, 82% hit rate" or "No — here's why: [prefix below threshold / byte drift / no cache_control]"

**Pattern C: Setup from scratch**
Trigger: "add caching", "set up prompt caching", "I'm not using caching yet"
1. Find cacheable content (stable system prompt, tools, embedded docs)
2. Add `cache_control` markers with diffs
3. Wire telemetry
4. Verify → run twice, confirm `cache_read > 0`
5. Report: "Caching active, estimated savings: $X/mo at Y calls/day"

Pick the pattern that matches user intent. Default to Pattern A if unclear.

## Complex codebases — multiple call sites, conditional routing

For large codebases with many call sites or dynamic prompt assembly:

**Step 1: Map the landscape**
```
→ grep for all call sites
→ categorize each:
  - static prompt (same every time) vs dynamic (assembled at runtime)
  - volume: high/medium/low (ask user or check logs)
  - prompt size: large (>2k tokens) / small
→ output: table of call sites with category + priority
```

**Step 2: Prioritize by savings potential**
```
savings ∝ volume × cacheable_prefix_size × hit_rate

Focus first on:
1. High volume + large static prefix (biggest wins)
2. High volume + shared prefix across branches
3. Large prefix + moderate volume

Skip (for now):
- Low volume call sites
- Fully dynamic prompts with no stable prefix
```

**Step 3: Handle conditional routing**
```
If prompt varies by condition (user type, feature flag, etc.):

Option A: Shared prefix
  → factor out the common stable part
  → cache that, let branches diverge after
  → one cache entry serves all branches

Option B: Per-branch caching
  → each branch gets its own cache entry
  → viable if branches are few and volume per branch is high
  → bad if many branches with low volume each (cold cache problem)

Option C: No caching
  → if branches are many and prefixes are small, caching won't help
  → recommend Batch API instead
```

**Step 4: Dynamic prompt assembly**
```
If prompts are built from pieces at runtime:

1. Trace the assembly logic
   → which pieces are stable? (loaded once at startup)
   → which pieces vary per request? (user input, context lookup)

2. Reorder assembly
   → stable pieces first, variable pieces last
   → cache breakpoint goes after the last stable piece

3. Watch for hidden variability
   → dict iteration order
   → timestamp injection
   → request-scoped IDs
```

**Step 5: Measure per call site**
```
→ add telemetry to each call site with a label
→ log_cache(response, label="route-A")
→ aggregate hit rates per label
→ identify which routes benefit, which don't
```

For very large codebases: audit in waves. Top 3-5 call sites first, measure impact, then next wave.

## Quick qualification — is caching even relevant?

Caching saves money when **the same bytes appear at the start of multiple API calls**. Ask:

1. **Do you make repeated calls?** Single one-off calls = no reuse = no savings.
2. **Is there a stable prefix?** System prompt, tools array, few-shot examples, embedded docs — anything that doesn't change between calls.
3. **How big is that prefix?** Must clear the model's minimum (1024–4096 tokens depending on model). Small prompts can't cache.

If yes to all three → audit is worth it. If no → caching won't help; recommend Batch API (50% off) or model selection instead.

## Where the wins are (biggest first)

1. **Large static system prompts** — Instructions, persona, constraints that ship on every call. Often 2k–20k tokens. Put it first, mark it, done.
2. **Tool/function definitions** — JSON schemas are big and rarely change. They're part of the cache key.
3. **Embedded reference docs** — Knowledge bases, specs, examples stuffed into context. Stable = cacheable.
4. **Conversation history prefix** — In multi-turn, earlier turns are stable; only the latest user message changes.

**What kills it:** Anything dynamic before the stable content — timestamps, user IDs, session tokens, request IDs. One byte of drift = full miss.

## Core thesis (share this early, it frames everything)

**Prompt caching is both a cost cut and a throughput multiplier.** Cache reads cost 0.1× the base input rate (90% off). And on every non-deprecated Claude model, `cache_read_input_tokens` do NOT count toward the ITPM (Input Tokens Per Minute) rate limit. At Tier 1 (30k ITPM on Sonnet), caching is often what saves you from 429s before it's what saves you money. Most content online treats caching as a cost topic and misses half the value.

## Step 1 — detect the integration path

Look at the project's code and dependencies. You're trying to place them on this tree:

| Signal | Path | Hand off to |
|---|---|---|
| `import anthropic` (Python) / `@anthropic-ai/sdk` (TS) / raw HTTP to `api.anthropic.com` | Direct Anthropic API | `cache-check-direct` |
| `boto3` + `bedrock-runtime` client with `anthropic.*` model IDs | AWS Bedrock | `cache-check-bedrock` |
| `langchain_anthropic`, `llama_index.llms.anthropic`, `@anthropic-ai/claude-agent-sdk` | Framework-wrapped | `cache-check-framework` |
| `litellm`, Helicone as proxy, OpenRouter, Portkey | Gateway | (no dedicated skill — see §5) |
| Azure AI Foundry SDK with Claude | Azure (same mechanics as direct) | `cache-check-direct`, note Azure preview status |
| Vertex AI SDK with Claude | Vertex | (no skill — see §5, caching status) |

Places to look: `requirements.txt` / `pyproject.toml` / `package.json`, the main entry files, any env vars like `ANTHROPIC_BASE_URL`, `OPENAI_API_BASE` pointing at a proxy, etc.

If you see multiple paths (e.g., LangChain in one service, raw SDK in another), name each and audit them separately. Don't blur.

## Step 2 — confirm with the user

Say what you found and who you think they are. One short paragraph. Example:

> I see `anthropic` (Python) in `requirements.txt` and direct `client.messages.create` calls in `agent/main.py`. Looks like direct Anthropic API use, not a framework. Going with that. Let me know if part of this is actually going through LangChain or a proxy.

## Step 3 — capture context needed for dollar/throughput math

Before handing off, ask for:

1. **Usage tier** (1–4, Monthly Invoicing, or "don't know — check console"). Shapes which lens to apply — T1/T2: lead with ITPM relief; T3/T4: lead with cost shave.
2. **Rough call volume** (per day or per month) — needed for projected savings.
3. **Expected cache hit rate after fixes** (if unsure, use 70% as a reasonable baseline for stable system prompts / 40% for more volatile workloads — but state the assumption).

If the user doesn't know any of these, proceed with sensible defaults and flag them as assumptions.

## Step 4 — hand off

Route to the right sub-skill. Keep this transition short — "handing off to `cache-check-direct` for the audit" — and let that skill run.

## Step 5 — the edges (things no sub-skill fully owns)

**Vertex AI.** As of 2026-04-17, Anthropic prompt caching on Vertex is listed as "coming soon" in Anthropic's docs. Tell the user this — advice based on direct-API mechanics may or may not apply once Vertex rolls out caching. Point them at Google's Vertex AI docs to check current state, and `docs/provider_matrix.md` in this repo for the comparison.

**Pure gateways (LiteLLM, Helicone-as-proxy).** These pass `cache_control` through to Anthropic in most cases, but check your gateway's docs to confirm the `cache_control` field isn't being stripped or re-serialized in a way that breaks byte-exact match. If the gateway strips it, you have no caching regardless of how well you structured your prompt. Recommend: test with a known-cacheable prompt and verify `cache_read_input_tokens > 0` on the second call.

**Mixed paths.** If one service uses LangChain and another uses raw SDK, treat them as separate audits. Don't average advice across them.

## Universal footguns to name up front

Before handing off, flag these four — they bite regardless of integration path:

1. **Min-token thresholds are non-monotonic across model families.** Opus 4.7/4.6/4.5 require 4096 tokens to cache; Opus 4.1/4 require only 1024. Upgrading Opus 4.1 → 4.5 can silently break caching on prompts that previously worked. Same goes for Haiku 3.5 (2048) → Haiku 4.5 (4096), and Sonnet 4.5 (1024) → Sonnet 4.6 (2048).
2. **`usage.input_tokens` is post-last-breakpoint only, not total input.** Total input = `cache_read + cache_creation + input_tokens`. Easy to misread, especially in dashboards.
3. **Exact byte match required.** A single whitespace change, JSON key reordering, or a dynamically-formatted timestamp anywhere before the breakpoint = cache miss. Python dicts serialized without sorted keys are a common culprit in tool definitions.
4. **20-block lookback limit.** Grow a conversation more than 20 content blocks past where you last wrote the cache, and you'll miss. For long conversations, add a second breakpoint further along.

## "I added cache_control but it's not working"

If `cache_read_input_tokens` stays at 0 on the second call:

| Symptom | Likely cause | Fix |
|---|---|---|
| Both `cache_read` and `cache_creation` are 0 | Prefix below min-token threshold | Grow prefix or check model's threshold (4096 for Opus 4.5+, Haiku 4.5) |
| `cache_creation` > 0 on every call, never reads | Bytes changed between calls | Find the drift — timestamps, JSON key order, whitespace, dynamic content pre-breakpoint |
| Works locally, fails in prod | Different model ID, different account, or >5min between calls | Pin model, check TTL, verify same API key/org |
| Framework shows 0 but proxy shows hits | Framework not exposing the field | Check `response.response_metadata` or use Helicone to see real values |

## How to diagnose byte drift

**Step 1 — Capture requests.** Log the full request body on two consecutive calls. For direct SDK:
```python
import json
# Before client.messages.create(), dump what you're sending:
print(json.dumps({"system": system, "messages": messages, "tools": tools}, indent=2, sort_keys=True))
```

**Step 2 — Diff.** Compare the two dumps. Everything from the start up to and including the `cache_control` block must be byte-identical.

**Step 3 — Find the culprit.** Common sources of drift:
- `datetime.now()`, `time.time()`, `uuid4()` anywhere in system/tools/early messages
- `json.dumps()` without `sort_keys=True` — dict key order varies
- f-strings with user IDs, session IDs, request IDs
- Whitespace: `.strip()` on some paths, not others
- Float precision: `f"{x}"` vs `f"{x:.4f}"`

**Step 4 — Fix.** Move dynamic content after the cache breakpoint (into later messages), or remove it entirely from the prompt.

## How to add telemetry

**Minimal (5 lines, no deps):**
```python
def log_cache(resp, label=""):
    u = resp.usage
    cr, cw = u.cache_read_input_tokens or 0, u.cache_creation_input_tokens or 0
    total = cr + cw + u.input_tokens
    print(f"[{label}] {cr}/{total} cached ({100*cr//total if total else 0}%)")
```
Call after every `messages.create()`. Run twice, check second call shows cache reads.

**Persistent:** Append to SQLite or CSV for trends. See `docs/telemetry.md` for a 20-line SQLite logger.

**Production:** Helicone (swap base_url, get dashboard) or OpenLLMetry (auto-instruments SDK, exports to OTel).

## For `claude -p` users (Claude Code CLI)

If the user runs Claude via `claude -p` rather than calling the API from their own code, caching works differently — it's automatic but there are ways to maximize it.

### What's already happening

- `claude -p` **auto-caches with 1-hour TTL** (no setup needed)
- Claude Code's system prompt (~12k tokens) is cached and shared across all sessions
- Each new `claude -p` call starts a fresh cache key for the user prompt
- Subagents (via Task tool) use 5-minute TTL with Haiku

### How to leverage it

**1. Use `--resume` for multi-step work**
```bash
# First call — pays cache write cost
claude -p "Analyze this codebase" --session-id $SID

# Follow-up — cache hit on everything from first call
claude -p "Now refactor the auth module" --resume $SID
```
Each resumed turn reads from cache instead of re-paying for the context.

**2. Use `-c` to continue most recent session**
```bash
claude -p "Review this PR"
# ... later, same directory ...
claude -p "What about the error handling?" -c
```
`-c` continues the most recent session in the current directory.

**3. Use `--fork-session` to branch without polluting history**
```bash
claude -p "Try approach A" --resume $SID --fork-session
```
Gets cache benefit from prior context but writes to a new session.

**4. Use `--append-system-prompt` for stable context**
```bash
claude -p "Do X" --append-system-prompt "$(cat context.md)"
```
Content in system prompt caches better than content in `-p` argument.

### How to check if it's working

Session logs live at `~/.claude/projects/<cwd-slug>/<session-uuid>.jsonl`. Each turn has:
```json
"usage": {
  "cache_read_input_tokens": 24000,
  "cache_creation_input_tokens": 500,
  "input_tokens": 12
}
```

**Quick check:**
```bash
# Find your latest session
ls -t ~/.claude/projects/$(pwd | tr '/' '-')/*.jsonl | head -1

# Grep for cache stats
grep cache_read ~/.claude/projects/.../<session>.jsonl
```

If `cache_read_input_tokens` grows across turns, caching is working.

### Tips for max cache reuse

1. **Keep prompts stable at the start** — variable content at the end
2. **Same directory** — session lookup is by cwd
3. **Don't exceed 1 hour between turns** — TTL expires, next turn re-pays
4. **Avoid `--no-session-persistence`** — disables caching across turns

### When this doesn't help

- Single one-shot `claude -p` calls with no follow-up — you pay write cost but never read
- Rapidly changing prompts — each change is a new cache key
- Cross-machine work — sessions are local, can't resume from another machine

## When to recommend other tools instead of continuing this audit

- Wants zero-config auto-injection → Autocache (montevive/autocache). Skill can't beat "don't touch my code."
- Wants ongoing production observability → Helicone, OpenLLMetry, or the Anthropic console's Usage page.
- Wants per-session Anthropic cost tracking → `ccost` CLI (carlosarraes/ccost).
- Just wants the docs → `docs/prompt_caching.md` in this repo, or [Anthropic's official page](https://platform.claude.com/docs/en/build-with-claude/prompt-caching).

This skill is for **learning + one-shot audits**, not runtime or production monitoring.
