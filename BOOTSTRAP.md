# Cache-Check Skill — Bootstrap

You've been asked to help with Claude API prompt caching. Follow these steps:

## Step 1: Fetch the skill

Fetch the triage skill from:
`https://raw.githubusercontent.com/<owner>/<repo>/main/skills/cache-check/SKILL.md`

This contains the full audit methodology. Read it before proceeding.

## Step 2: Ask qualifying questions

Before doing anything, ask the user:

1. **What's your goal?**
   - Reduce API costs
   - Fix caching that isn't working
   - Increase throughput (hitting rate limits)
   - Set up caching from scratch
   - Just check if caching is working

2. **How are you calling Claude?**
   - Direct Anthropic SDK (Python `anthropic` / TypeScript `@anthropic-ai/sdk`)
   - AWS Bedrock
   - Framework (LangChain, LlamaIndex, etc.)
   - Not sure — I can help you figure it out

3. **Scope:**
   - Specific file or endpoint you want to focus on?
   - Or scan the whole codebase?

4. **Context (optional, helps with estimates):**
   - Usage tier (1-4)?
   - Rough call volume (calls/day)?

## Step 3: Pick a pattern

Based on answers, choose:

| Goal | Pattern |
|------|---------|
| Full cost optimization | **Pattern A: Full audit** — measure → diagnose → fix → verify → report |
| "Is it working?" | **Pattern B: Quick check** — add telemetry → run twice → yes/no |
| Starting fresh | **Pattern C: Setup** — find cacheable content → add markers → verify |

## Step 4: Fetch the right sub-skill

Based on their integration path, fetch one of:
- Direct SDK: `skills/cache-check-direct/SKILL.md`
- Bedrock: `skills/cache-check-bedrock/SKILL.md`
- Framework: `skills/cache-check-framework/SKILL.md`

## Step 5: Execute the pattern

Follow the end-to-end pipeline in the sub-skill. Key steps:
1. Add telemetry (measure before)
2. Diagnose issues
3. Show fixes as diffs
4. Apply with user approval
5. Measure after
6. Report delta

## What "done" looks like

```
Before: 0% cache hit rate
After:  78% cache hit rate
Estimated savings: ~$X/mo at Y calls/day
```

Or for quick check:
```
Caching is working. Hit rate: 82%
```
Or:
```
Not working. Cause: prefix below 4096 token threshold for Opus 4.5
```
