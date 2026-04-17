---
name: cache-check-bedrock
description: Prompt-caching audit for projects that call Claude models through AWS Bedrock (via `boto3` / `bedrock-runtime`). Bedrock's prompt caching is a distinct feature from Anthropic's direct API — different request shape, different model support, different pricing. Invoked by `cache-check` after detecting Bedrock usage. Do not invoke standalone unless certain the project is Bedrock-first.
---

# cache-check-bedrock

Bedrock is not Anthropic-direct. Same models, different caching surface. Keep that front of mind throughout.

## End-to-end pipelines

### Pattern A: Full audit
```
1. FIND call sites
   → grep for `bedrock-runtime`, `invoke_model`, `converse`, `anthropic.claude`
   → list each file:line
   → for complex codebases: categorize by volume/size/routing logic

2. ADD telemetry
   → after each call, extract from response with label:
     def log_bedrock_cache(response, label=""):
         usage = response.get('usage', {})
         cr = usage.get('cacheReadInputTokens', 0)
         cw = usage.get('cacheWriteInputTokens', 0)
         inp = usage.get('inputTokens', 0)
         print(f"[{label}] read={cr} write={cw} fresh={inp}")

3. MEASURE baseline
   → run code path twice (same inputs)
   → capture metrics

4. DIAGNOSE (if cache_read stays 0)
   → verify model ID supports caching on Bedrock (check AWS docs)
   → verify cachePoint marker present and correctly placed
   → dump request body, diff between calls

5. FIX with diffs
   → add cachePoint block marker (Bedrock syntax, not Anthropic's cache_control)
   → move dynamic content after the marker

6. MEASURE after
   → run twice again

7. REPORT
   → "Before: X. After: Y."
   → per-route breakdown if multiple call sites
   → link to Bedrock pricing for savings estimate
```

### Multiple call sites / conditional routing
```
Same principles as direct API:
1. Label each call site uniquely
2. Factor shared prefixes across branches
3. Aggregate hit rates per label
4. Prioritize high-volume routes with large stable prefixes
```

### Pattern B: Quick check
```
1. ADD telemetry to one call site
2. RUN twice
3. CHECK cacheReadInputTokens on run 2
   → >0: "Working"
   → =0: "Not working — check model support, marker syntax, or byte drift"
```

### Pattern C: Setup from scratch
```
1. VERIFY model supports caching
   → check Bedrock docs for your model ID + region

2. FIND stable content
   → system prompt, tools, context docs

3. ADD cachePoint marker
   → Bedrock uses block markers, not cache_control field
   → verify syntax against current Bedrock docs

4. ADD telemetry

5. VERIFY
   → run twice
   → confirm cacheReadInputTokens > 0

6. REPORT
   → "Caching active on Bedrock."
   → point to AWS pricing page for cost impact
```

## Key differences vs. direct Anthropic API

| Dimension | Direct Anthropic API | AWS Bedrock |
|---|---|---|
| Request shape | `cache_control: {type: "ephemeral"}` on content blocks | `cachePoint: {type: "default"}` block markers |
| TTL options | 5m default, 1h optional | 5m (as of last check — verify Bedrock docs for current options) |
| Breakpoints | Up to 4, places in `tools`/`system`/`messages` | Check Bedrock caching docs; structure differs |
| Pricing | Anthropic rates (1.25×/2× write, 0.1× read) | Bedrock's pricing page — verify multipliers |
| Model coverage | All non-deprecated Claude models | Subset — not every Claude version available on Bedrock supports caching |
| Rate limit relief | Cache reads don't count toward ITPM for non-deprecated models | AWS has its own quota system — relief behavior differs |

**Bottom line:** much of the advice in `cache-check-direct` transfers conceptually (cache stable prefixes, watch for invalidation from volatile content, avoid JSON key instability), but specifics (request syntax, exact pricing, rate-limit impact) must be verified against [AWS Bedrock's prompt caching documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html) rather than Anthropic's.

## Knowledge you rely on

### Universal mechanics that DO transfer from direct API
- Cache is a hash of the prefix → exact byte match required
- Invalidation cascades — changes earlier invalidate everything later
- Stable content first, volatile content last
- JSON key ordering matters
- Silent failures below minimum token thresholds (though the threshold may differ on Bedrock)
- Workspace/account isolation applies

### Things you MUST verify on Bedrock's docs before giving numbers
- Exact request field name (`cachePoint` block marker at time of last known info, but verify)
- Current pricing multipliers
- Which Claude model IDs on Bedrock support caching
- Minimum token thresholds (may differ from Anthropic-direct)
- TTL options available
- How cached tokens interact with Bedrock quotas vs. Anthropic ITPM rules

**Do not cite Anthropic's direct-API pricing or thresholds as authoritative for Bedrock.** Always say "per Bedrock's pricing page" and link.

## Audit procedure

### 1. Find call sites
Grep for:
- `bedrock-runtime` client instantiation: `boto3.client('bedrock-runtime', ...)` or `boto3.client("bedrock-runtime", ...)`
- Invocation methods: `invoke_model`, `invoke_model_with_response_stream`, `converse`, `converse_stream`
- Model IDs: `anthropic.claude-*` in the `modelId` argument
- Any explicit `cachePoint` or cache-related request body fields

### 2. Per-site checklist

**A. Which model is being invoked?** Claude model IDs on Bedrock have specific version suffixes. Note which one and check Bedrock docs for whether that specific version supports caching (not all do).

**B. Which invocation method?** The `converse` API and `invoke_model` have different request body structures. Caching configuration lives in different places.

**C. Where does the cache marker go?** At time of last known info, Bedrock uses a `cachePoint` block inserted into the messages/system content to mark where to cache. Verify current syntax against Bedrock docs, then check whether it's present and where it's placed.

**D. What's the stability story for content BEFORE the cache point?** Same question as direct API — anything dynamic pre-marker breaks the cache.

**E. Is the prefix long enough?** Verify current Bedrock minimum token threshold for the specific model. Do not assume it matches Anthropic-direct's thresholds.

### 3. Emit the report

Same report structure as `cache-check-direct`, but:
- Use Bedrock's actual request syntax in fix diffs (boto3 call shape, `cachePoint` markers, not `cache_control`)
- Quote savings in terms of Bedrock's pricing (include the link you used)
- Don't quote ITPM relief numbers — Bedrock quota mechanics are different. Say something like "reduces input token quota consumption" if accurate on Bedrock, and point to AWS's quota docs.

### 4. Recommend verification
Because Bedrock changes faster than this skill can be updated: end every Bedrock audit with a verification step. Have the user run their code with the fixes and inspect the response's `usage` block (Bedrock returns `cacheReadInputTokens` / `cacheWriteInputTokens` fields on compatible models) to confirm real caching behavior.

## When to redirect to direct API

If the user is using Bedrock only for billing/compliance reasons and their workload would benefit from the broader feature set (1h TTL, more model versions, tighter docs), mention that direct Anthropic API might be an option to evaluate. Don't push — respect that Bedrock is often chosen for enterprise/org reasons.

## Add Telemetry

Bedrock has its own observability stack. Suggest options based on AWS maturity:

**Minimal (response-level):**
- Log `cacheReadInputTokens` / `cacheWriteInputTokens` from the response body after each `invoke_model` or `converse` call
- Compute hit rate locally: `cache_read / (cache_read + cache_write + input_tokens)`

**AWS-native:**
- **CloudWatch Metrics** — Bedrock publishes invocation metrics; check if cache-specific dimensions are available for your model
- **CloudWatch Logs** — enable model invocation logging to capture full request/response including usage
- **X-Ray tracing** — if already instrumented, add cache fields as annotations for per-trace visibility

**Cross-cloud (if already using):**
- Datadog / New Relic Bedrock integrations — may surface cache metrics depending on version
- Custom metric emission: after each call, `put_metric_data` with `CacheHitRate` dimension per model/call site

**Alerting ideas:**
- CloudWatch alarm on hit rate below threshold
- Anomaly detection on `cacheWriteInputTokens` (spikes = unexpected invalidation)
- Cost anomaly via AWS Cost Explorer if caching suddenly stops working

Bedrock's caching telemetry surface is evolving — verify current field names against [Bedrock docs](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html).

## Honest limits

- This skill's authoritative source is Bedrock's docs, not Anthropic's. When specifics drift, trust Bedrock.
- AWS rolls caching support out gradually — always verify the exact model ID the user invokes supports caching in their region.
- Don't quote Anthropic-direct pricing numbers for Bedrock. Link to AWS pricing page.
