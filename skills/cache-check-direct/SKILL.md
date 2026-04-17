---
name: cache-check-direct
description: Deep-dive prompt-caching audit for projects that call the Anthropic API directly — via the Python SDK (`anthropic`), TypeScript SDK (`@anthropic-ai/sdk`), or raw HTTP. Covers cache_control placement, min-token thresholds, invalidation risks, model-swap traps, multi-turn lookback, and tier-aware cost/throughput projections. Invoked by `cache-check` after it confirms the project uses the direct API path. Don't invoke standalone unless you're sure the project isn't on Bedrock, a framework, or a gateway.
---

# cache-check-direct

You are auditing a project that calls `client.messages.create` (Python) or the equivalent TS/HTTP surface directly. The `cache_control` field is a request-schema primitive — same shape across all three surfaces, so this skill works for all three. Keep your language-specific examples matched to what the user actually writes.

## End-to-end pipelines

### Pattern A: Full audit
```
1. FIND call sites
   → grep for `messages.create`, `Anthropic(`, `@anthropic-ai/sdk`
   → list each file:line
   → for complex codebases: categorize by volume/size/static vs dynamic

2. ADD telemetry (if not present)
   → insert `log_cache(response, label="<call-site-name>")` after each call site
   → use unique labels per call site to track separately
   → helper:
     def log_cache(r, label=""):
         u = r.usage
         cr, cw = u.cache_read_input_tokens or 0, u.cache_creation_input_tokens or 0
         total = cr + cw + u.input_tokens
         pct = 100*cr//total if total else 0
         print(f"[{label}] read={cr} write={cw} fresh={u.input_tokens} ({pct}%)")

3. MEASURE baseline
   → run the code path twice (same inputs)
   → capture output: expect cache_read=0 on run 1, cache_read>0 on run 2 if working
   → for multiple call sites: aggregate by label, identify which routes cache well

4. DIAGNOSE issues (if cache_read stays 0)
   → dump request body on both calls, diff
   → check: prefix above threshold? byte-identical? cache_control present?

5. FIX with diffs
   → move dynamic content after breakpoint
   → add cache_control to stable prefix
   → pin json.dumps(sort_keys=True) if tools/system built from dicts

6. MEASURE after
   → run twice again
   → capture new metrics

7. REPORT
   → "Before: 0% hit rate. After: X% hit rate."
   → "At Y calls/day, estimated savings: $Z/mo + W effective ITPM"
   → for multiple call sites: report per-route hit rates
     "route-chat: 82% | route-summarize: 45% | route-classify: 0% (too small)"
```

### Handling conditional prompt routing
```
If prompts are assembled conditionally:

1. TRACE the assembly
   → find where prompt pieces come from
   → identify: what's shared across conditions? what varies?

2. FACTOR shared prefix
   → if multiple branches share system prompt / tools, that's one cache entry
   → put shared content first, branch-specific content after breakpoint

3. LABEL per branch
   → log_cache(response, label="branch-premium-user")
   → log_cache(response, label="branch-free-user")
   → compare hit rates — shared prefix should show hits on both

4. DECIDE per-branch strategy
   → high volume per branch: each branch can have its own cache entry
   → low volume per branch: factor out shared prefix or skip caching
```

### Pattern B: Quick check
```
1. ADD minimal telemetry to one call site
2. RUN twice with identical input
3. CHECK cache_read on run 2
   → >0: "Caching works. Hit rate: X%"
   → =0: "Not working. Likely cause: [threshold/drift/missing marker]"
```

### Pattern C: Setup from scratch
```
1. FIND largest stable content
   → system prompt size? tools array size? embedded docs?

2. CHECK threshold
   → count tokens (or estimate ~4 chars/token)
   → compare to model's min (4096 for Opus 4.5+/Haiku 4.5, 2048 for Sonnet 4.6, 1024 for others)

3. ADD cache_control
   → wrap stable content in content block with cache_control: {type: "ephemeral"}
   → show diff

4. ADD telemetry
   → insert logger after call

5. VERIFY
   → run twice
   → confirm cache_read > 0 on run 2

6. REPORT
   → "Caching active. Prefix: N tokens. Expected hit rate: ~X% for this usage pattern."
```

## Knowledge you rely on

### Pricing multipliers (universal)
- Base input = 1×
- Cache write, 5-minute TTL = **1.25×** base input
- Cache write, 1-hour TTL = **2×** base input
- Cache read = **0.1×** base input

Breakeven: a 5m write pays for itself after **one** cache read. A 1h write after **two** reads. Everything past that is savings.

### Minimum token thresholds (per model — non-monotonic!)
| Model | Min tokens to cache |
|---|---|
| Opus 4.7 / 4.6 / 4.5 | 4096 |
| Opus 4.1 / 4 | 1024 |
| Sonnet 4.6 | 2048 |
| Sonnet 4.5 / 4 | 1024 |
| Haiku 4.5 | 4096 |
| Haiku 3.5 | 2048 |
| Haiku 3 | 4096 |

Below threshold: request silently succeeds, both `cache_creation_input_tokens` and `cache_read_input_tokens` stay at 0. No error. This is the #1 source of "I added cache_control but nothing happened."

### ITPM interplay (the under-told story)
For all non-deprecated models, `cache_read_input_tokens` do NOT count toward ITPM rate limits. Deprecated 3.x models (marked † in Anthropic's rate-limit page) still count them. This turns caching into an effective-throughput multiplier:

> 2M ITPM limit + 80% cache hit rate ≈ 10M effective ITPM.

At Tier 1 (30k ITPM on Sonnet), this often matters more than the 90% cost discount. Lead with this at low tiers.

### Invalidation cascade
Changes to earlier layers invalidate everything later:
- Tool definition changes → invalidates tools + system + messages
- Tool choice param change → invalidates system + messages (tools cache survives)
- Image add/remove, thinking param change → invalidates system + messages
- Web search / citations toggle / speed setting → invalidates system + messages

Practical rule: **stable things first**. Order is fixed at `tools → system → messages`, and you want your most-stable content in `tools`, volatile content late in `messages`.

### Mechanics quick-ref
- Max **4 cache breakpoints** per request
- **20-block lookback window** per breakpoint — if the conversation grows more than 20 content blocks past your last cache write, you miss
- **Exact byte match** — whitespace, JSON key order, timestamps in strings all matter
- Two modes: **auto** (single top-level `cache_control`, moves forward automatically) vs. **explicit** (per-block, up to 4, for fine-grained control)
- Workspace-level cache isolation since 2026-02-05 (was org-level before)

### Response usage fields — read them correctly
```
usage.cache_creation_input_tokens   # written to cache this request
usage.cache_read_input_tokens       # retrieved from cache this request
usage.input_tokens                  # tokens AFTER the last breakpoint (not total!)
usage.output_tokens                 # generation
# Total input = cache_read + cache_creation + input_tokens
```

## Audit procedure

### 1. Find every call site

Grep for:
- Python: `messages.create`, `Anthropic(`, `AsyncAnthropic(`, `from anthropic`
- TypeScript: `messages.create`, `new Anthropic(`, `@anthropic-ai/sdk`
- Raw HTTP: `api.anthropic.com/v1/messages`, any `Authorization: ...` block in a fetch/requests/httpx call

For each site, open the file and read enough context to see how the request is being built: what's in `system`, what's in `tools`, what's in `messages`, and which parts are static vs. dynamically constructed per call.

### 2. Per-site checklist

For each call site, evaluate and note findings:

**A. Is there ANY `cache_control` present?**
- No → is there a cacheable prefix worth caching? (large static system prompt, big tools array, long conversation history, large embedded document). If yes, this is a primary finding.
- Yes → continue checks below.

**B. Does the cached content clear the min-token threshold for the declared model?**
- Roughly count tokens on the cacheable prefix. Use Anthropic's tokenizer if available, or approximate at ~4 chars/token for English as a rough starting point — but note the approximation.
- If the model is Opus 4.5+, Haiku 4.5, or Haiku 3, the threshold is 4096. Common miss.
- Flag as "silent no-op" if the declared cache_control is on a prefix under threshold.

**C. Is the breakpoint on stable content?**
- What comes BEFORE the breakpoint, all the way to the request start, must be byte-identical across requests for a hit.
- Look for dynamically-interpolated values pre-breakpoint: timestamps, user IDs, session IDs, random nonces. Any of these kill the hit rate.
- f-strings, template engines, `.format()` calls on content preceding the breakpoint deserve a closer look.

**D. Is the tool JSON stable?**
- Python dicts without `sort_keys=True` when serialized to JSON can reorder keys across runs (especially on Python < 3.7 semantics, but also across interpreter restarts with some serializers). Tool definitions hashed differently → no cache hit.
- If the user serializes tool schemas themselves (rather than passing Python dicts to the SDK), confirm determinism.

**E. Multi-turn lookback risk?**
- If `messages` grows each turn and the only cache breakpoint is in `system`, the breakpoint stays fixed but the user's current turn is always new. That's fine — cache reads still work.
- But if the user has a breakpoint IN `messages` and the list is growing, check whether by turn N there will be more than 20 content blocks between the last cache-write turn and the read turn. If yes, they need a second breakpoint further along.

**F. Model-swap risk?**
- If the user is on an older model family (e.g., Opus 4.1, Haiku 3.5) with prefix sizes between 1024 and 4096 tokens, flag that an upgrade to the newer family (Opus 4.5+, Haiku 4.5) will silently break caching.

**G. Auto vs. explicit mode choice?**
- Single static system prompt, nothing else fancy → auto caching (top-level `cache_control`) is simpler and does the right thing.
- Multiple sections with different change frequencies (stable tools + slowly-changing context + volatile user input) → explicit breakpoints are worth it.

### 3. Gather context for projections

From the triage handoff, you should have: tier, calls/day, expected hit rate. If not, ask now.

### 4. Emit the report

Use this structure. Keep it terse but concrete — the user wants to act on it.

```
## cache-check-direct: audit of <project>

### Stack detected
- Language: Python / TypeScript
- SDK: anthropic <version> / @anthropic-ai/sdk <version>
- Models in use: <list>
- Declared tier: <user input>
- Estimated calls/day: <user input>

### Findings (by impact)

#### 1. <site path:line> — <one-line summary>
**Issue:** <what's wrong or missing>
**Impact:** ~$X/mo saved AND ~Y ITPM unlocked (at N calls/day, Z% hit rate)
**Fix:**
```<lang>
<copy-pasteable diff>
```
**Why:** <one-sentence explanation tied to mechanics>

#### 2. ...

### Universal risks flagged
- <model-swap risks, if any>
- <lookback concerns, if any>
- <JSON stability concerns, if any>

### Things already good
- <positive observations — don't only crit>

### Further reading
- docs/prompt_caching.md — full reference
- docs/prompt_structure_guidelines.md — structuring rules
```

### 5. Offer to apply fixes

At the end: "Want me to apply any of these? I can do them one at a time or in a batch — tell me the numbers." Use the Edit tool on confirmed changes.

## Telemetry — verify before and after

**The Anthropic API already returns cache telemetry on every response.** No extra tooling needed. Every `messages.create` call returns:

```
response.usage.cache_read_input_tokens       # saved money on these
response.usage.cache_creation_input_tokens   # paid the first-call premium on these
response.usage.input_tokens                  # post-last-breakpoint (NOT total)
response.usage.output_tokens                 # generation
```

Total input = `cache_read + cache_creation + input_tokens`. This is the primary surface — mention it to the user explicitly. Every observability tool out there is just an aggregator over these same four fields.

Before finalizing the report, offer to help the user measure what's actually happening by logging these fields. The 60-second version (Python):

```python
def log_cache_usage(response, label=""):
    u = response.usage
    cr = getattr(u, "cache_read_input_tokens", 0) or 0
    cw = getattr(u, "cache_creation_input_tokens", 0) or 0
    total = cr + cw + u.input_tokens
    pct = cr / total * 100 if total else 0
    print(f"[{label}] in {total} = {cr} read ({pct:.0f}%) + {cw} write + {u.input_tokens} fresh | out {u.output_tokens}")
```

Drop into call sites, run twice (~2 min apart), confirm run 2 shows `cr > 0`. If it's 0 on run 2:
- Check cached prefix is above the model's min-token threshold
- Check nothing pre-breakpoint changed between runs (whitespace, timestamps, key order)
- Check TTL hasn't expired

## Reframing a prompt — worked example

Before (anti-patterns: volatile content pre-breakpoint, no `cache_control`):

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    system=f"""You are a helpful assistant.
Current time: {datetime.now().isoformat()}
User ID: {user.id}

{VERY_LONG_INSTRUCTIONS}
{LARGE_KNOWLEDGE_BASE}
""",
    messages=[{"role": "user", "content": query}],
)
```
Problems:
- Timestamp in system = cache miss every call
- User ID in system = one cache entry per user (low hit rate)
- No `cache_control` = no caching at all

After:

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    system=[
        {
            "type": "text",
            "text": VERY_LONG_INSTRUCTIONS + "\n\n" + LARGE_KNOWLEDGE_BASE,
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[
        {
            "role": "user",
            "content": (
                f"[context: user_id={user.id}, time={datetime.now().isoformat()}]\n\n"
                f"{query}"
            ),
        }
    ],
)
```
Wins:
- Stable content lives in `system`, under a single cache breakpoint
- Volatile per-user/time data moved to user message *after* breakpoint — doesn't invalidate
- One cache entry shared across all users (high hit rate)
- If instructions change occasionally but knowledge base is very stable, split into two blocks each with their own `cache_control` for finer invalidation

Sketch what the report's diffs look like in the same shape: remove volatile-pre-breakpoint content, add a single explicit `cache_control` at the end of the stable tail, move dynamic context into the user message.

## Savings math (derived from Anthropic's pricing multipliers)

The formulas below are algebra on the published pricing (1.25× write 5m, 2.0× write 1h, 0.1× read) and the documented ITPM behavior. Use them with the user's *measured* inputs — don't substitute "typical" values you've seen online, and don't let the skill generate example scenarios with fabricated numbers.

Let `T` = cacheable prefix tokens, `P` = base input $/token, `C` = calls per period, `H` = cache hit rate (0–1) measured from their actual `usage` fields, `M` = TTL write multiplier (1.25 for 5m, 2.0 for 1h).

```
savings = C × T × P × [ 1 − (1−H) × M − 0.1H ]            (USD per period)

effective_ITPM = rated_ITPM / (1 − H)                      (non-deprecated models)
```

**Breakeven** (where savings = 0, solved from the formula): H ≈ 21.7% for 5m TTL, H ≈ 52.6% for 1h TTL. Below these, caching costs more than no caching.

**Rules when applying this in a report:**
- Use the user's measured `H` if available. If not, say "estimated H = X%, should verify with telemetry" and run the numbers as a range (e.g., compute at H = 40%, 60%, 80%).
- Use `T` from an actual token count of the cacheable prefix (Anthropic tokenizer or SDK's `count_tokens`), not a guess from character length.
- Use `P` from Anthropic's current pricing page for the declared model — don't memorize.
- Flag when the prefix is below the model's min-token threshold: savings collapses to 0 regardless of the formula.

See `docs/savings_math.md` for the full derivation, breakeven proof, and the hit-rate-to-savings tables.

## Tier-aware lens

| Tier | ITPM (Sonnet) | RPM | Dominant lens |
|---|---|---|---|
| 1 | 30k | 50 | ITPM survival — cache to avoid 429s |
| 2 | 450k | 1,000 | Balanced — ITPM relief + cost savings |
| 3 | 800k | 2,000 | Cost shave — ITPM headroom is real |
| 4 | 2M | 4,000 | Cost shave + latency |

Phrase findings in the dominant lens. A Tier 1 user cares more about "unlocked 40k effective ITPM" than "$15/mo saved." A Tier 4 user is the opposite.

## Example findings phrased right

Shape (do not copy the numbers — compute them from measured inputs using the formula above):

✓ Specific file + line + issue + formula-derived impact
  *"`agent.py:47`: system prompt is `<T>` tokens (counted with SDK tokenizer), loaded on every request, no `cache_control`. At `<H>` hit rate (measured/estimated), formula projects `<savings>` for `<C>` calls and `<T>/(1−H)` × effective ITPM. Fix: wrap in a `cache_control` block (diff below)."*

✓ Silent no-op caught
  *"`agent.py:47` has `cache_control` but the cached prefix is `<T>` tokens, below this model's `<threshold>` minimum. Neither `cache_read_input_tokens` nor `cache_creation_input_tokens` will be non-zero. Either grow the prefix past `<threshold>` or move to a model family with a lower threshold (verify from the min-token table)."*

✗ *"You should cache your system prompt."* — vague, no file, no numbers, no formula.

**Never fabricate the numbers.** If the user hasn't given you tier/volume/hit rate, either ask or present the savings as a range across plausible hit rates (say H ∈ {40%, 60%, 80%}) with the formula explicit. Don't pick a single "typical" value.

## Add Telemetry

How to instrument cache behavior — useful before changes (baseline) or after (validation).

**Inline logging (no deps):**
- Wrap call sites with a helper that prints `response.usage` — cache_read, cache_write, fresh, output
- Compute hit rate per call: `cache_read / (cache_read + cache_write + input_tokens)`
- Grep logs for trends or pipe to a file

**Lightweight persistence:**
- SQLite: `(ts, call_site, model, cache_read, cache_write, fresh, output)` — query weekly hit rates
- CSV append — simpler, works for quick experiments

**Observability stack:**
- Helicone: swap `base_url`, get a dashboard with hit rate, cost, latency — zero code changes beyond the URL
- OpenLLMetry / Traceloop: auto-instruments `anthropic` SDK, exports to OTel backends
- Anthropic Console: free, shows per-model cache behavior, coarse but zero-friction

**Alerting options:**
- Hit rate below threshold → possible template change broke byte-match
- Spike in `cache_creation_input_tokens` → unexpected invalidation
- Model swap → check if new threshold is higher than cached prefix size

See `docs/telemetry.md` for copy-paste snippets.

## Honest limits

Say these when relevant:
- Can only estimate hit rates; real numbers require running the code and inspecting `usage`.
- If prompts are heavily template-rendered, you may miss cases — offer to look at a sample rendered output.
- Token counts are approximations unless you tokenize. Say "approximately" or "roughly" when you're eyeballing.
