# cache-check

A skill that helps Claude audit your code for prompt-caching opportunities.

## Use it

Paste this to Claude:

```
Fetch https://raw.githubusercontent.com/aditya-pola/cache-check/main/BOOTSTRAP.md
```

Claude fetches the skill, asks a few questions about your setup, and runs the audit.

---

## What is prompt caching?

When you call Claude's API, you send tokens. If part of your prompt is the same across calls (like a system prompt), you can **cache** it — pay once, reuse many times.

- **Without caching:** Pay full price for every token, every call
- **With caching:** Pay 1.25x once to cache, then 0.1x (90% off) for every reuse

That's it. The skill helps you find what's cacheable, set it up correctly, and verify it works.

## When to use this

**Good fit:**
- You have a large system prompt or tools array that repeats on every call
- You're paying more than you'd like for Claude API
- You added `cache_control` but it doesn't seem to work
- You're hitting rate limits and need more throughput

**Not needed:**
- Single one-off calls (nothing to reuse)
- Tiny prompts under 1024 tokens (too small to cache)

## What the skill does

1. **Asks** what you're trying to do (reduce cost? fix broken caching? just check if it's working?)
2. **Finds** where your code calls Claude
3. **Measures** current cache behavior
4. **Identifies** issues (if any)
5. **Shows** fixes as diffs
6. **Verifies** the fix worked

You get a before/after: "0% hit rate → 78% hit rate, ~$X/mo savings"

## The one thing to understand

Caching works on **byte-identical prefixes**. If the exact same bytes appear at the start of multiple API calls, they can be cached.

```
Call 1: [system prompt] + [user message A]
Call 2: [system prompt] + [user message B]
              ↑
        same bytes = cached
```

The skill finds your stable content, puts it first, marks it for caching, and verifies.

## Common gotchas (the skill catches these)

1. **Prompt too small** — Models require 1024-4096 tokens minimum to cache. Below that, caching silently fails.

2. **Something changes** — A timestamp, user ID, or JSON key reordering before the cache marker = cache miss. Silent.

3. **Wrong model upgrade** — Opus 4.1 needs 1024 tokens; Opus 4.5 needs 4096. Upgrading can break caching.

## Bonus: throughput boost

Cached tokens don't count against rate limits. 

If you're on Tier 1 (30k ITPM*) with 80% cache hits, you effectively get ~150k ITPM. At low tiers, this matters more than the cost savings.

*ITPM = Input Tokens Per Minute, Anthropic's rate limit metric

## What's in this repo

```
skills/
├── cache-check/           → entry point, routes to the right sub-skill
├── cache-check-direct/    → for direct Anthropic SDK users
├── cache-check-bedrock/   → for AWS Bedrock users  
└── cache-check-framework/ → for LangChain, LlamaIndex, etc.
```

## References

- [Anthropic — prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [AWS Bedrock — prompt caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)
