# Cache-Check Skill — Bootstrap

You are Claude. A user wants help with prompt caching. Follow these steps automatically.

## Step 1: Fetch the triage skill (do this now)

Fetch:
```
https://raw.githubusercontent.com/aditya-pola/cache-check/main/skills/cache-check/SKILL.md
```

Read it fully — it contains cache mechanics, patterns, and diagnosis methods.

## Step 2: Ask the user qualifying questions

Before auditing, ask:

1. **What's your goal?**
   - Reduce API costs
   - Fix caching that isn't working
   - Increase throughput (hitting rate limits)
   - Set up caching from scratch
   - Just check if it's working

2. **How are you calling Claude?**
   - Direct Anthropic SDK (Python/TypeScript)
   - AWS Bedrock
   - Framework (LangChain, LlamaIndex, etc.)
   - `claude -p` (CLI)
   - Not sure

3. **Scope?**
   - Specific file/endpoint, or whole codebase?

4. **Optional context:**
   - Usage tier (1-4)?
   - Call volume (calls/day)?

## Step 3: Fetch the right sub-skill (based on their answers)

After the user answers, fetch the appropriate sub-skill:

| Integration path | Fetch |
|------------------|-------|
| Direct SDK | `https://raw.githubusercontent.com/aditya-pola/cache-check/main/skills/cache-check-direct/SKILL.md` |
| AWS Bedrock | `https://raw.githubusercontent.com/aditya-pola/cache-check/main/skills/cache-check-bedrock/SKILL.md` |
| Framework | `https://raw.githubusercontent.com/aditya-pola/cache-check/main/skills/cache-check-framework/SKILL.md` |
| `claude -p` | Use the "For claude -p users" section in the triage skill (already fetched) |

## Step 4: Pick a pattern and execute

| Goal | Pattern |
|------|---------|
| Full optimization | **Pattern A** — measure baseline → diagnose → fix → verify → report delta |
| "Is it working?" | **Pattern B** — add telemetry → run twice → report yes/no |
| Starting fresh | **Pattern C** — find cacheable content → add markers → verify |

Follow the end-to-end pipeline in the sub-skill.

## Step 5: Report outcome

End with a clear before/after:

```
Before: 0% cache hit rate
After:  78% cache hit rate
Estimated savings: ~$X/mo at Y calls/day
```

Or for quick check:
```
Caching working: Yes/No
Reason: [hit rate or failure cause]
```
