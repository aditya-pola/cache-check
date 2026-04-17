---
name: cache-check-framework
description: Prompt-caching audit for projects that use Claude through a framework — LangChain (`langchain_anthropic`), LlamaIndex (`llama_index.llms.anthropic`), the Claude Agent SDK, or similar abstractions. Static audit is inherently limited because the framework constructs API requests dynamically; this skill focuses on what CAN be checked statically and recommends runtime observability for the rest. Invoked by `cache-check` after detecting framework usage.
---

# cache-check-framework

Be honest up front: static audits are weaker here than for direct SDK use. The framework owns a significant portion of the request shape, and you often can't see the final `cache_control` placement without running the code. Lead with this — don't oversell what you can find.

## End-to-end pipelines

### Pattern A: Full audit
```
1. IDENTIFY framework + version
   → check requirements.txt / package.json
   → note: caching support varies by version

2. CHECK if framework passes cache_control through
   → LangChain: recent versions support content blocks with cache_control
   → LlamaIndex: check docs for your version
   → Agent SDK: user system prompts cacheable, internal scaffolding managed by SDK

3. ADD telemetry (framework-specific)
   → LangChain: print(response.response_metadata.get('usage', {}))
   → LlamaIndex: print(response.raw) or model-specific metadata
   → OR use Helicone proxy (ANTHROPIC_BASE_URL swap) to see real values

4. MEASURE baseline
   → run twice with identical large system prompt
   → capture cache_read_input_tokens

5. DIAGNOSE (if 0)
   → framework might strip cache_control — use HTTP proxy (mitmproxy) to inspect actual request
   → if stripped: upgrade framework or bypass for hot path

6. FIX
   → add cache_control via framework's supported method
   → OR recommend dropping to raw SDK for the hot call site

7. MEASURE after

8. REPORT
   → "Framework passes caching through: yes/no"
   → "Hit rate: X%"
   → if framework blocks caching: "Consider raw SDK for this call site"
```

### Pattern B: Quick check
```
1. ADD Helicone proxy (easiest for frameworks)
   → set ANTHROPIC_BASE_URL=https://anthropic.helicone.ai
   → add Helicone-Auth header

2. RUN twice

3. CHECK Helicone dashboard
   → shows real cache hit rate regardless of what framework exposes
   → "Working" or "Not working — framework may be stripping cache_control"
```

### Pattern C: Setup from scratch
```
1. CHECK framework version supports caching
   → if not: upgrade or use raw SDK

2. FIND stable content you control
   → system prompt, few-shot examples, context docs

3. ADD cache_control via framework's API
   → LangChain: use content blocks in SystemMessage
   → show framework-idiomatic diff

4. ADD telemetry
   → Helicone proxy recommended (framework-agnostic truth)

5. VERIFY
   → run twice
   → check hit rate

6. REPORT
   → "Caching active through [framework]"
   → "If hit rate stays 0, consider raw SDK bypass for hot paths"
```

**Key difference for frameworks:** You can't trust static analysis. Always verify with runtime telemetry. Helicone proxy is the lowest-friction way to see truth.

### Complex codebases — chains, agents, routers

Frameworks add their own routing complexity:

```
1. MAP framework-level routing
   → LangChain: chains, agents, routers — each may hit Claude differently
   → identify which components make Claude calls
   → trace: what prompt does each component send?

2. LABEL at framework level
   → Helicone custom headers: add route labels
     headers={"Helicone-Property-Route": "summarize-chain"}
   → aggregates in dashboard by route

3. SHARED PROMPT TEMPLATES
   → frameworks often use PromptTemplate / ChatPromptTemplate
   → find the templates, identify stable vs variable parts
   → if template has {user_input} at the end: stable prefix is cacheable
   → if template interpolates throughout: harder to cache

4. AGENT LOOPS
   → agents make multiple Claude calls per user request
   → each iteration may have different context (tool results)
   → caching helps if: system prompt + tools are stable across iterations
   → measure: are early iterations cache writes, later iterations cache reads?

5. ESCAPE HATCH
   → if framework routing makes caching impossible
   → identify the hot path (highest volume component)
   → consider: bypass framework for that one call, use raw SDK
   → keep framework for everything else
```

## The framework caching landscape

Brief mental model:

- **LangChain** (`langchain_anthropic.ChatAnthropic`): caching pass-through depends on version. In recent versions, `cache_control` can be set on message content via `additional_kwargs` or by constructing `SystemMessage` / `HumanMessage` with explicit content blocks. Check the user's `langchain-anthropic` version and consult its changelog.
- **LlamaIndex** (`llama_index.llms.anthropic.Anthropic`): similar story — caching support has been added incrementally; check version and LlamaIndex docs.
- **Claude Agent SDK** (`@anthropic-ai/claude-agent-sdk`): manages its own prompt structure. User-level system prompts are cacheable; internal agent scaffolding caching is managed by the SDK. Framework users have less direct control here.
- **Custom framework / in-house wrapper**: audit the wrapper itself — it's effectively a thin layer on top of the direct SDK.

## Audit procedure

### 1. Identify the framework + version

Check `requirements.txt` / `pyproject.toml` / `package.json` for:
- `langchain-anthropic` (and `langchain-core`, `langchain`) with version pins
- `llama-index` + any Anthropic-specific subpackage
- `@anthropic-ai/claude-agent-sdk`
- Others (Haystack, DSPy, custom wrappers)

Note the version. Caching support has been evolving; version pins matter.

### 2. Identify the integration points users CAN control

Even in a framework, these are typically yours to configure:
- **System prompt content** — usually the biggest cacheable block
- **Tools / functions array** — if the framework supports caching here, this is often the biggest stable prefix
- **Few-shot examples embedded in messages** — static, cacheable if framework passes them through
- **Any long context documents attached to every call** — RAG context, knowledge base inserts

Search the codebase for how these are constructed. Flag them as "audit surface."

### 3. Check how the framework is invoked

For LangChain specifically, look at how `ChatAnthropic` is instantiated and how messages are constructed. Look for:
- Explicit `cache_control` in message content blocks (LangChain supports this in recent versions — the user can pass raw content blocks)
- `extra_body` / `additional_kwargs` carrying cache-related fields
- Absence of any caching configuration — common in defaults-only code

For LlamaIndex: look at LLM instantiation and any custom prompt templates.

### 4. Verify it actually works

This is the critical step — framework users cannot trust static analysis alone.

**Recommend the user run one of:**

- **Quick verification script:** call the framework with a known large stable system prompt, twice. Print `response.response_metadata['usage']` (LangChain) or equivalent. On the second call, look for `cache_read_input_tokens > 0`. If zero, caching isn't working — either the framework isn't passing `cache_control` through or it's being placed wrong.

- **Runtime observability:** route traffic through Helicone (proxy-based, shows cache hit rate), or instrument with OpenLLMetry / Traceloop. These give continuous visibility.

- **Direct inspection of the outgoing HTTP request:** point `ANTHROPIC_BASE_URL` at a local proxy (e.g., `mitmproxy`, `Caddy` with a logging reverse proxy) and confirm the actual bytes sent to Anthropic contain `cache_control` where expected.

### 5. Emit the report

Keep it modest in claims. Structure:

```
## cache-check-framework: audit of <project>

### Framework detected
- Framework: <LangChain / LlamaIndex / Agent SDK / custom>
- Version: <version pin>
- Known caching support at this version: <yes / partial / no / unknown>

### Static findings (what I could check)
- <system prompt shape + size>
- <tools array shape + size>
- <attached context blocks>
- <presence/absence of cache_control in any user-controlled content>

### Recommendations
1. <specific user-controlled content to add cache_control to, with framework-idiomatic syntax>
2. <version upgrade if needed>
3. **Verify with a runtime check** (see below) — framework caching cannot be confirmed statically.

### Verification steps (important!)
- <specific script or observability recommendation>

### Consider dropping to raw SDK for hot paths
If one call site dominates cost or ITPM usage, it may be worth using the `anthropic` SDK directly for that specific call to gain full cache_control precision. The framework is a productivity win for most calls; for the 1-2 hot paths, direct is often worth the complexity.
```

## When to switch tracks

- If the user's framework version doesn't support `cache_control` at all, the real recommendation is **upgrade the framework** or **bypass it for the hot path**.
- If the user has heavy custom templating with timestamps / user IDs pre-system-prompt, that's a content issue the framework can't fix. Fix the template.
- If the user wants to measure real hit rates without changing code, recommend Helicone proxy (ANTHROPIC_BASE_URL swap, no code changes). Skill can't beat that for production visibility.

## Add Telemetry

Framework users need telemetry more than anyone — you can't trust static analysis to confirm caching works.

**Per-framework access to usage:**
- **LangChain:** `response.response_metadata.get('usage', {})` — contains `cache_read_input_tokens` if the framework passes it through
- **LlamaIndex:** check `response.raw` or model-specific metadata for usage dict
- **Agent SDK:** agent-level callbacks may expose per-turn usage; check SDK docs

**Quick validation script:**
- Call the framework twice with identical large system prompt
- Print usage on both calls
- Run 2 should show `cache_read_input_tokens > 0` — if zero, framework isn't caching (or isn't passing markers through)

**Runtime observability (recommended for framework users):**
- **Helicone proxy** — set `ANTHROPIC_BASE_URL`, framework doesn't need to change. Dashboard shows real hit rate. Best friction/value ratio.
- **OpenLLMetry / Traceloop** — auto-instruments LangChain + LlamaIndex. Exports cache fields to your OTel backend.
- **Local HTTP proxy** (mitmproxy, Caddy) — for debugging. Confirm actual outbound requests contain `cache_control` where expected.

**Alerting ideas:**
- Hit rate trend over time (Helicone dashboard or custom OTel query)
- Framework version bump → re-validate caching behavior (frameworks change how they pass fields)
- Compare usage before/after template changes — catch regressions early

Point users at `docs/telemetry.md` for code snippets. Emphasize: *framework caching must be verified at runtime, not assumed from static config.*

## Honest limits

- Static analysis of framework usage gives you surface indicators, not confirmed behavior.
- Framework caching support is a moving target; your knowledge might be stale. Tell the user to check the framework's current docs for the specific version they're on.
- For complex agent frameworks (multi-step tool loops, sub-agents), caching interacts with the framework's own prompt restructuring in ways not always documented.
