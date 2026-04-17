# cache-check

Audit your Claude API usage for prompt-caching savings. Measure → diagnose → fix → verify.

## Use it — paste this to any Claude

```
Fetch https://raw.githubusercontent.com/aditya-pola/cache-check/main/BOOTSTRAP.md
```

That's it. Claude will fetch the bootstrap, ask you qualifying questions, and run the right audit for your setup.

---

## You already have cache telemetry (and probably don't know it)

Every single response from the Anthropic API returns cache metrics in the `usage` object. No proxy, no observability tool, no instrumentation. They come back on every call, for free.

```python
response = client.messages.create(...)
u = response.usage

u.cache_read_input_tokens       # tokens served from cache this call
u.cache_creation_input_tokens   # tokens written to cache this call
u.input_tokens                  # tokens AFTER the last breakpoint (NOT total input)
u.output_tokens                 # generation
# Total input = cache_read + cache_creation + input_tokens
```

**Read these fields in prod, not a dashboard.** A dashboard adds latency to your iteration loop; four print statements close it. If you're trying to figure out "is my caching even working?", the answer is in your next API response — not in a tool you haven't installed yet. See `docs/telemetry.md` for the 60-second verification script.

The observability tools (Helicone, OpenLLMetry, Anthropic Console) are aggregators on top of these same fields. Useful at scale; overkill for "does caching work on this prompt?"

## Why this exists

The prompt-caching tool space is crowded — [Autocache](https://github.com/montevive/autocache) handles zero-config injection, [Helicone](https://docs.helicone.ai/gateway/concepts/prompt-caching) handles production observability, [ccost](https://github.com/carlosarraes/ccost) handles spend tracking. What's missing isn't more code. It's a **single clear voice** that:

- Teaches you the real mechanics (not just "add `cache_control`")
- Audits *your* repo with context the docs don't have
- Gives **tier-aware** advice — a Tier 1 user fights ITPM limits; a Tier 4 user shaves dollars
- Names the **non-obvious footguns** you'll hit

That's a good fit for a skill, not another tool.

## The four footguns nobody talks about

1. **Min-token thresholds are non-monotonic across model families.** Opus 4.7/4.6/4.5 require 4096 tokens to cache; Opus 4.1/4 require only 1024. Upgrading Opus 4.1 → 4.5 can silently break caching on prompts that worked before. Same trap on Sonnet 4.5 → 4.6 (1024 → 2048) and Haiku 3.5 → 4.5 (2048 → 4096). No error — both cache fields just stay at 0.

2. **`usage.input_tokens` is not total input.** It's tokens *after the last breakpoint*. Total input = `cache_read + cache_creation + input_tokens`. Easy to misread on dashboards; a 100k-token cached prompt with a 50-token user message shows `input_tokens: 50`.

3. **Exact byte match required.** A whitespace change, JSON key reordering (Python dicts serialized without `sort_keys`!), or a timestamp interpolated pre-breakpoint = cache miss. Silent.

4. **20-block lookback limit.** If a growing conversation gets more than 20 content blocks past your last cache write, new calls miss. Long multi-turn chats need a second breakpoint further along, not just one at the top.

## Caching is a throughput multiplier

On every current (non-deprecated) Claude model, `cache_read_input_tokens` do NOT count toward your ITPM rate limit. Example:

> Tier 1 Sonnet (30k ITPM) + 80% cache hit rate ≈ **150k effective ITPM**

At low tiers, this often matters more than the 90% cost discount. Most "prompt caching saves you 90%!" content glosses over it.

## What's in this repo

```
skills/
├── cache-check/              → triage skill, routes to the right deep-dive
├── cache-check-direct/       → Anthropic API direct (Python / TS / REST)
├── cache-check-bedrock/      → AWS Bedrock (different caching surface)
└── cache-check-framework/    → LangChain / LlamaIndex / Agent SDK
docs/
├── prompt_caching.md         → legacy research notes (context-specific)
├── prompt_structure_guidelines.md → seven reframing principles
├── savings_math.md           → formulas, breakeven analysis, worked $ examples
├── telemetry.md              → copy-paste snippets to measure real cache behavior
├── landscape.md              → survey of existing tools — when to use which
└── provider_matrix.md        → Anthropic vs. Bedrock vs. Vertex vs. Azure
```

## Install

Copy the skills into your Claude Code skills directory:

```bash
cp -r skills/cache-check* ~/.claude/skills/
```

Or install per-project under `.claude/skills/` in your repo.

Once installed, open Claude Code in the project you want to audit and ask something like:

> *Can you review this repo for prompt caching opportunities?*

Claude Code will match the `cache-check` skill, triage your project's integration path, hand off to the right deep-dive, and produce a report.

## When to use this vs. other tools

| You want | Use |
|---|---|
| One-shot audit + learn the mechanics | **this skill** |
| Zero-config auto-injection of cache_control | [Autocache](https://github.com/montevive/autocache) |
| Ongoing production cache-hit-rate observability | [Helicone](https://docs.helicone.ai/) or [OpenLLMetry](https://www.traceloop.com/openllmetry) |
| Per-session Claude API spend tracking | [ccost](https://github.com/carlosarraes/ccost) |
| A broader Claude-cost-optimization guide (Batch + Cache + Extended Thinking) | [sstklen/claude-api-cost-optimization](https://github.com/sstklen/claude-api-cost-optimization) |
| Official docs | [Anthropic — prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) |

See `docs/landscape.md` for a longer honest comparison.

## Limitations (read before expecting miracles)

- **Requires Claude Code.** Not CI-friendly; not a headless CLI. If you want a static linter, reach for something else.
- **Estimates, not measurements.** Projected $/mo and ITPM deltas are estimates — real numbers come from running your code and inspecting `usage`. The skill tells you to verify.
- **Framework audits are weaker than direct-SDK audits.** See `skills/cache-check-framework/SKILL.md` — the audit recommends runtime verification precisely because static analysis has limits here.
- **Bedrock facts can drift.** AWS updates prompt caching on its own cadence. The Bedrock skill always links you to AWS docs as authoritative.

## Contributing / feedback

This is a work-in-progress personal project. If you find a footgun not listed, a framework version behaving oddly, or an existing tool that should be in the comparison, open an issue.

## References

- [Anthropic — prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Anthropic — Messages API reference](https://platform.claude.com/docs/en/api/messages)
- [Anthropic — rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- [Anthropic cookbooks — prompt_caching.ipynb](https://github.com/anthropics/anthropic-cookbook/blob/main/misc/prompt_caching.ipynb)
- [AWS Bedrock — prompt caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)
