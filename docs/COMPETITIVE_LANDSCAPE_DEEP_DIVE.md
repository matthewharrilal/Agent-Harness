# Competitive Landscape Deep Dive

## Purpose
This document captures deep research into each tool in the competitive landscape, answering specific questions about what they do, how they work, and where they fall short.

---

## The 8 Tools Analyzed

1. **CCProxy** - LiteLLM-based routing proxy
2. **claude-code-mux** - Rust proxy with OAuth and auto-failover
3. **PAL MCP** - MCP server with `clink` delegation tool
4. **hcom** - Hook-based inter-agent communication
5. **Portkey** - AI gateway with metadata routing
6. **OpenRouter** - Multi-model API gateway
7. **Conductor Build** - Parallel Claude instances in git worktrees
8. **CLIProxyAPI** - Menu bar quota tracking with round-robin

---

## Tool 1: CCProxy

### What Is CCProxy?
CCProxy is a Python-based proxy that sits between Claude Code and Anthropic's API. It uses LiteLLM under the hood. When you configure it, Claude Code sends requests to CCProxy (localhost:4000), and CCProxy decides which model/provider to forward to based on rules.

### How CCProxy Routing Works (The Mechanics)
CCProxy has exactly **4 built-in rule types**:

1. **TokenCountRule** - If total tokens > threshold, route to different model
2. **MatchModelRule** - If model name contains substring, route to different model
3. **ThinkingRule** - If request has "thinking" field, route to different model
4. **MatchToolRule** - If request uses specific tool (e.g., WebSearch), route to different model

### Why CCProxy Uses Static Heuristics (Not Semantic Routing)

**What CCProxy sees:**
- The raw API request (JSON)
- Token counts (computed via tiktoken)
- Model name requested
- Tools being invoked
- Presence of thinking parameters

**What CCProxy CANNOT see:**
- The meaning/intent of the task
- Whether it's "research" vs "coding"
- Task complexity beyond token count
- Historical patterns of success

**Example:**
```
Request 1: "Research the history of JWT security"
- CCProxy sees: 8 tokens, no tools, no thinking field
- CCProxy decision: Default route (Claude)

Request 2: "Debug this auth.ts file"
- CCProxy sees: 6 tokens, no tools, no thinking field
- CCProxy decision: Default route (Claude)

BOTH requests look identical to CCProxy. It cannot distinguish research from debugging.
```

**Why don't they do semantic routing?**

Semantic routing would require:
1. Parsing the message content
2. Running an LLM call to classify intent (expensive, adds latency)
3. OR training a lightweight classifier (requires labeled data)
4. OR using embeddings (requires embedding model, similarity search)

CCProxy's design philosophy is **zero overhead, transparent proxy**. Adding semantic routing would:
- Add 50-200ms latency per request
- Require additional API calls (cost)
- Require training data (complexity)

It's a design choice: CCProxy prioritizes simplicity and speed over intelligence.

### CCProxy: What It Does Well
- Token-based routing works for large context → Gemini scenarios
- Tool-based routing works for WebSearch → Perplexity scenarios
- Zero latency overhead (rules evaluate in <1ms)
- Works with Claude Max subscriptions

### CCProxy: What Breaks
- Cannot understand task intent
- No rate limit awareness
- No fallback when provider fails
- No shared state between requests

### CCProxy: With the Harness

| Gap | CCProxy Today | With Harness |
|-----|---------------|--------------|
| Semantic routing | None — routes by token count, tools, model name only | Classifies task intent (research vs coding vs debugging) automatically |
| Rate limit awareness | None — no visibility into quota consumption | Proactive prediction at 80%, auto-routes non-critical tasks |
| Failover | None — if provider fails, request fails | Automatic retry with backup provider |
| Shared state | None — stateless proxy, each request independent | Persistent memory across requests and sessions |
| Context handoff | N/A — doesn't track context | Transfers reasoning state when switching providers |

**Concrete scenario where CCProxy fails:**
```
You: "Research the latest XSS attack vectors and then fix our auth.ts"

CCProxy sees:
- 12 tokens
- No tools used
- No thinking field
- Default model name

CCProxy decision: Route to Claude (default)

Ideal behavior:
- "Research XSS vectors" → Gemini (research task, web search)
- "Fix auth.ts" → Claude (precision coding)

But CCProxy can't parse intent. It routes the entire request to Claude,
wasting Claude's quota on research that Gemini does better.
```

---

## Tool 2: claude-code-mux

### What Is claude-code-mux?
claude-code-mux is a Rust-based HTTP proxy that provides **automatic failover** between AI providers. When your primary provider (e.g., Anthropic) fails with a 429 or error, it automatically tries your backup providers.

### How claude-code-mux Works (The Mechanics)

1. You configure multiple providers with priorities:
   ```
   Priority 1: Anthropic (Claude API)
   Priority 2: OpenRouter (Claude via OpenRouter)
   Priority 3: Vertex AI (Claude via Google Cloud)
   ```

2. Your request arrives at the proxy
3. Proxy tries Priority 1 (Anthropic)
4. If Anthropic returns 429 → Proxy immediately tries Priority 2
5. If Priority 2 succeeds → Returns response to Claude Code
6. Claude Code never knows the switch happened

### The Context Continuity Problem (Explained)

**What happens during failover:**

```
Turn 1-5: Claude (Anthropic) is handling your debugging session
  - Claude has built up reasoning about the bug
  - Claude remembers hypotheses tried
  - Claude has a mental model of the codebase

Turn 6: Anthropic returns 429
  - claude-code-mux catches this
  - Switches to OpenRouter (still Claude model)
  - Forwards the same MESSAGE HISTORY

Turn 6 (continued):
  - OpenRouter Claude receives all messages from turns 1-5
  - BUT: This is a FRESH Claude instance
  - It has no "memory" of the reasoning process
  - It only sees the text of previous messages
```

**What's preserved:** The literal messages you exchanged
**What's LOST:**
- The model's internal reasoning state
- The "hypothesis chain" Claude was building
- The debugging approach being used
- Subtle context about what was tried and why

**Why doesn't mux use Mem0 or shared memory?**

claude-code-mux is a **stateless proxy**. By design:
- Each request is independent
- No database, no storage, no persistence
- It just forwards requests and catches errors

Adding Mem0 would require:
- A running database/service
- Logic to extract and store reasoning state (hard problem)
- Logic to inject that state into new provider's context
- Significant architectural changes

**Could they add it?** Yes, but it would change the tool's nature from "simple proxy" to "orchestration layer."

### claude-code-mux: What It Does Well
- Seamless failover on 429 errors
- Supports 18+ providers
- OAuth authentication support
- <1ms routing overhead, ~5MB memory

### claude-code-mux: What Breaks
- No semantic routing
- No shared memory/state
- Reasoning continuity lost during failover
- No rate limit prediction

### claude-code-mux: With the Harness

| Gap | claude-code-mux Today | With Harness |
|-----|----------------------|--------------|
| Context continuity | Messages preserved, reasoning state lost | Shared memory captures reasoning, hypotheses, approach |
| Rate limit awareness | None — reacts to 429, doesn't predict | Proactive prediction, routes before hitting limits |
| Semantic routing | None — priority-based failover only | Classifies task type, routes to appropriate model |
| Memory | Stateless — no persistence between requests | Persistent memory across sessions |
| Proactive routing | None — only acts on failure | Routes preemptively based on task type and quota |

**Concrete scenario where claude-code-mux fails:**
```
Turn 1-5: You're debugging a subtle race condition with Claude
  - Claude has identified 3 potential causes
  - Claude eliminated 2, focusing on the third
  - Claude is about to propose a fix

Turn 6: Anthropic returns 429 (rate limit hit)

claude-code-mux behavior:
  - Catches the 429
  - Switches to Priority 2 (OpenRouter)
  - Forwards message history to OpenRouter Claude

What happens next:
  - OpenRouter Claude receives the message log
  - But this is a FRESH instance — no reasoning state
  - It re-reads the messages, starts debugging from scratch
  - Might pursue the same dead-ends Claude already eliminated
  - You lose 5-10 minutes of debugging progress

With the harness:
  - Shared memory captured: "Investigating race condition. Eliminated
    hypothesis A (timing issue) and B (lock scope). Currently testing
    hypothesis C (async initialization order)."
  - New provider instance receives synthesized context
  - Continues from hypothesis C, not from scratch
```

---

## Tool 3: PAL MCP Server

### What Is PAL MCP?

**PAL MCP** (Provider Abstraction Layer - Model Context Protocol) is an MCP server with 10K+ GitHub stars that runs as a child process inside Claude Code. It extends Claude Code with additional tools, most importantly the `clink` tool that enables Claude to delegate work to other AI CLIs like Gemini, Codex, or other Claude instances.

**Repository:** https://github.com/BeehiveInnovations/pal-mcp-server

**The core idea:** Claude Code is powerful but has limitations (200K context, no web search, no multimodal). PAL lets Claude call out to OTHER AI CLIs that have different strengths, then bring the results back into Claude's conversation.

### What Is the `clink` Tool?

**`clink`** = "CLI Link" — a tool that spawns external CLI processes from within Claude Code.

**The mechanism step-by-step:**

1. **You're in an interactive Claude Code session**
2. **Claude decides to delegate** (based on your instructions or explicit request)
3. **Claude calls the `clink` MCP tool** with parameters:
   ```json
   {
     "cli_name": "gemini",
     "role": "security-analyzer",
     "prompt": "Analyze these files for vulnerabilities",
     "files": ["auth.ts", "config.ts"]
   }
   ```

4. **PAL MCP receives the tool call** and:
   - Loads Gemini config from `conf/cli_clients/gemini.json`
   - Loads role-specific system prompt from `systemprompts/clink/security-analyzer.md`
   - Prepares the combined prompt

5. **PAL spawns a subprocess:**
   ```bash
   gemini -p "SYSTEM: You are a security auditor...
              USER: Analyze these files for vulnerabilities..."
           --output-format json
           --yolo
           --telemetry false
   ```

6. **Gemini CLI runs in isolation:**
   - Headless mode (no interactive UI)
   - Same working directory as Claude
   - Can read the files you specified
   - Produces JSON output

7. **PAL captures Gemini's output** and returns it to Claude as a tool result

8. **Claude sees the result in the conversation:**
   ```
   Gemini found 3 security issues:
   1. JWT not validated (auth.ts:42)
   2. SQL injection risk (db.ts:89)
   3. Missing rate limiting (api.ts:15)
   ```

**Key parameters clink accepts:**

| Parameter | Type | Example | Purpose |
|-----------|------|---------|---------|
| `cli_name` | string | "gemini" | Which CLI to spawn |
| `role` | string | "security-analyzer" | Loads role-specific system prompt |
| `prompt` | string | "Analyze auth.ts" | The task for Gemini |
| `files` | array | `["auth.ts"]` | Files to include in context |
| `temperature` | number | 0.5 | Model parameter forwarding |

### What Invokes Delegation to Gemini?

**Delegation is MANUAL in PAL.** There's no automatic routing. Two patterns:

**Pattern 1: Explicit User Instruction**
```
You: "Use Gemini to analyze this codebase"
Claude: "I'll delegate to Gemini..." [calls clink]
```

**Pattern 2: Claude's Own Judgment (via CLAUDE.md)**

You write instructions in `CLAUDE.md`:
```markdown
## Delegation Strategy
- When analyzing >50 files, delegate to Gemini (larger context)
- For research tasks, delegate to Gemini (web search)
- For precision debugging, handle locally
```

Claude reads these and decides autonomously:
```
Claude (reasoning): "80 files to analyze. My instructions say delegate."
Claude: [calls clink tool]
```

### When Should You Delegate to Gemini?

| Scenario | Why Gemini? | Example |
|----------|-------------|---------|
| **Large codebase (>50 files)** | Gemini has 1M context vs Claude's 200K | "Audit all 87 auth files" |
| **Multimodal input** | Gemini handles images, PDFs, Figma | "Analyze this mockup" |
| **Research/investigation** | Gemini has built-in Google Search | "Latest XSS attack vectors" |
| **UI/frontend work** | Gemini is stronger at frontend | "Build responsive dashboard" |
| **Rate limit headroom** | Independent quota pools | Claude at 85%, Gemini fresh |
| **Rapid prototyping** | Speed over precision | "Quick MVP of feature X" |

### The No-Memory-Across-Sessions Problem (Deep Dive)

**Day 1:**
- You ask Gemini (via clink) to analyze auth module
- Gemini finds: "JWT expiration bug, rate limiting needed"
- Claude's session ends

**Day 2:**
- New Claude session, you ask to continue the analysis
- Claude calls clink again
- PAL spawns a NEW Gemini CLI process
- **This Gemini has ZERO memory of Day 1**
- Gemini re-analyzes from scratch, wastes tokens
- Might produce contradictory findings

**Why doesn't PAL have persistent memory?**
- Each `clink` call spawns a FRESH subprocess
- Gemini CLI itself is stateless
- PAL doesn't store results between calls
- The 3-hour/20-turn conversation threading only works WITHIN a continuous session

**Implications:**
- Manual context restoration required ("Yesterday you found X, Y, Z...")
- Tokens wasted re-analyzing same code
- Risk of inconsistent findings
- No learning over time

### PAL MCP: What It Does Well

1. **Clean delegation interface** — MCP protocol means structured tool calls, not string parsing
2. **Role-based prompting** — Pre-configured system prompts for different task types
3. **Result integration** — Gemini's output flows naturally into Claude's conversation
4. **Multiple CLI support** — Not just Gemini; can configure Codex, other Claude instances
5. **10K+ stars** — Battle-tested, production-ready

### PAL MCP: What Breaks (Limitations)

1. **No persistent memory** — Each clink call is stateless; Day 2 doesn't remember Day 1
2. **No automatic delegation** — You must explicitly call clink or write CLAUDE.md rules
3. **25K token output limit** — MCP tool responses capped; large analyses get truncated
4. **2-minute timeout** — Long-running Gemini tasks may timeout
5. **No rate limit awareness** — Doesn't know if Claude/Gemini are approaching limits
6. **Context passed, not shared** — Gemini sees files you specify, not Claude's reasoning

### PAL MCP: With the Harness

| Gap | PAL Today | With Harness |
|-----|-----------|--------------|
| Memory across sessions | None | Persistent store (file-based or Mem0) auto-injects prior context |
| Automatic delegation | Manual clink calls | Semantic router classifies tasks, routes automatically |
| Rate limit awareness | None | Proactive prediction, switches providers at 80% |
| Context handoff | Files only | Shared memory includes reasoning, decisions, state |
| Output limits | 25K cap | Stream to file, read in chunks |

---

## Tool 4: hcom (Hook-based Inter-Agent Communication)

### What Is hcom?

**hcom** (Hook Communication) is a real-time messaging layer that enables Claude Code and Gemini CLI (or other agents) to communicate with each other using Claude Code's native hook system and a SQLite message bus.

**Repository:** https://github.com/aannoo/hcom

**The core idea:** When multiple AI agents work on the same codebase, they need to know what each other is doing. Without communication, they might duplicate work, overwrite each other's changes, or produce conflicting outputs. hcom solves this by creating a shared event log that all agents can read and write to.

### How Agent Messaging Works (Deep Dive)

**The Architecture:**
```
Claude Code ←→ Hooks ←→ hcom (SQLite) ←→ Hooks ←→ Gemini CLI
```

**What IS a "message" in hcom?**

A message is NOT a text chat. It's a **structured event notification**:
```json
{
  "event_type": "file_write",
  "file_path": "src/middleware/rateLimit.ts",
  "agent": "claude",
  "timestamp": "2026-01-28T10:30:46Z",
  "session_id": "session_xyz"
}
```

**The technical mechanism:**

1. **Hooks** — Claude Code's native hook system (`~/.claude/hooks/`) runs scripts on events
2. **SQLite** — The message bus, with schema like:
   ```sql
   CREATE TABLE messages (
     id INTEGER PRIMARY KEY,
     timestamp DATETIME,
     agent TEXT,
     event_type TEXT,
     payload JSON,
     read BOOLEAN
   );
   ```
3. **Polling** — Each agent's hook periodically queries SQLite for new messages

**Who initiates messages?**

The **hook system initiates**, not the agents directly. When Claude writes a file:
1. Claude's `PostToolUse` hook fires automatically
2. The hook script sees "file was written" and posts to SQLite
3. Claude didn't explicitly "message" Gemini — the hook did it

### Concrete Timeline: Real-Time Collaboration

```
10:30:00 USER INPUT
  You: "Implement rate limiting middleware for the API"

10:30:15 CLAUDE STARTS WORK
  Claude: "I'll create a rate limiting module..."

10:30:45 CLAUDE FINISHES WRITING
  Claude writes: ~/project/src/middleware/rateLimit.ts (125 lines)
  PostToolUse hook fires automatically

10:30:46 CLAUDE → MESSAGE BUS
  hcom hook captures the file_write event
  SQLite INSERT: {
    agent: "claude",
    event_type: "file_write",
    file: "src/middleware/rateLimit.ts",
    timestamp: "10:30:46"
  }

10:30:47-10:30:48 ASYNCHRONOUS GAP
  Claude is idle, waiting for your next input
  Gemini has NOT been invoked yet

10:30:48 GEMINI'S HOOK POLLS
  Gemini's background hook runs (on timer, every 500ms-2sec)
  Queries: SELECT * FROM messages WHERE read=false
  Finds the file_write event

10:30:49 GEMINI REACTS
  Gemini's hook sees: "Claude wrote rateLimit.ts"
  Based on its configuration, decides to review it
  Spawns: `gemini -p "Analyze rateLimit.ts for bugs"`

10:31:15 GEMINI ANALYZES
  Gemini reads rateLimit.ts
  Finds: "Race condition in line 42-44"
  Finds: "Missing error handling for Redis timeout (line 78)"

10:31:31 GEMINI → MESSAGE BUS
  Gemini's hook posts back:
  INSERT: {
    agent: "gemini",
    event_type: "review_comment",
    file: "src/middleware/rateLimit.ts",
    findings: [
      {issue: "race_condition", line: 42, severity: "high"},
      {issue: "missing_error_handling", line: 78, severity: "medium"}
    ]
  }

10:31:32 CLAUDE'S HOOK POLLS
  Claude's hook checks SQLite
  Finds Gemini's review_comment event

10:31:33 CLAUDE SEES FEEDBACK
  Hook injects into Claude's context: "Gemini found 2 issues..."
  Claude reads the review and fixes line 42

10:31:45 CYCLE REPEATS
  Claude saves updated file
  New file_write event posted
  Gemini can re-review if configured to do so
```

**Total wall time: ~100 seconds.** Most of it is model processing and polling gaps.

### What Async Messaging Unlocks

**1. Parallel Awareness**
- Without messaging: You run Claude, get result, manually run Gemini to review
- With messaging: Both agents know about each other's work automatically

**2. Collision Detection**
- Without messaging: Both agents edit same file → merge conflict
- With messaging: Claude writes auth.ts, Gemini sees message, avoids touching auth.ts
- **Note: Detection is AFTER-the-fact** — both edit, then see collision

**3. Feedback Loops**
- Without messaging: Claude writes → you review → you tell Claude → Claude fixes (3 steps)
- With messaging: Claude writes → Gemini reviews → result injected → Claude fixes (1 step)

**4. Coordination Without Explicit Orchestration**
- Without messaging: "Claude, do API. Gemini, do tests." (explicit division)
- With messaging: Claude works on API, Gemini sees progress, proactively tests changes

### Messaging vs Routing vs Orchestration

| Capability | Messaging (hcom) | Routing | Orchestration |
|-----------|-----------------|---------|---------------|
| **Real-time awareness** | Yes | No | Yes |
| **Decide WHO does work** | No | Yes | Yes |
| **Prevent collisions** | No (detect only) | No | Yes |
| **Share context** | Partial (events) | No | Yes |
| **Handle failures** | No | Partial | Yes |
| **Require both agents running** | Yes | No | No |

**Key insight:** Messaging is lightweight but passive. It tells agents what happened but doesn't decide what should happen. Orchestration is heavy but active.

### hcom: What It Does Well

1. **Real-time inter-agent awareness** — Agents know what each other is doing
2. **Collision detection** — Both agents notified if they touch same file
3. **Persistent event log** — SQLite preserves history across polling gaps
4. **Native integration** — Works with Claude Code's existing hook system
5. **Lightweight** — Just SQLite + hook scripts, no heavy infrastructure

### hcom: What Breaks (Limitations)

1. **Messaging only, NOT routing** — Doesn't decide who handles what task
2. **Collision detected, not prevented** — Both agents edit, THEN see the collision
3. **No automatic context injection** — Messages are events, not synthesized context
4. **Requires both agents running** — Gemini must be active to receive messages
5. **Polling delays** — 500ms-2sec gaps between event and reaction
6. **No failure handling** — If Gemini's review times out, messages pile up

### hcom: With the Harness

| Gap | hcom Today | With Harness |
|-----|------------|--------------|
| Task routing | None — agents decide themselves | Semantic router assigns tasks to appropriate agent |
| Collision prevention | Detect after | File locking before write |
| Context injection | Events only | Synthesize events into actionable context |
| Rate limit awareness | None | Include capacity info in messages |
| Failure recovery | None | Retry logic, escalation chains |

---

## Tool 5: Portkey

### What Is Portkey?

**Portkey** is an AI gateway — a cloud service that sits between your application and AI providers (Anthropic, OpenAI, Google, etc.). Every request to your AI provider goes through Portkey first, where it can be routed, logged, cached, and monitored.

**Website:** https://portkey.ai

**The core idea:** Instead of calling `api.anthropic.com` directly, you call `api.portkey.ai`. Portkey intercepts the request, applies your routing rules, adds observability, and forwards to the appropriate provider.

**Architecture:**
```
Your App → Portkey Gateway → Anthropic / OpenAI / Google / etc.
              ↓
         Logging, Caching,
         Routing, Cost Tracking
```

### What "Requires Upstream Logic" Means (Deep Explanation)

**The Problem You Want to Solve:**
"Route research tasks to Gemini, coding tasks to Claude"

**What Portkey CAN do:**
- Route based on METADATA you attach to each request
- Conditional logic: `IF metadata.task_type == "research" THEN route to Gemini`
- Complex conditions: `IF metadata.plan == "premium" AND metadata.tokens > 100000`

**What Portkey CANNOT do:**
- Look at your prompt and understand it's a "research task"
- Classify task intent from message content
- Make semantic decisions based on what the task IS

**So what does "upstream logic" mean?**

YOU must write code that classifies the task BEFORE calling Portkey. Then you TELL Portkey the classification via metadata.

```python
# Step 1: YOUR UPSTREAM LOGIC (you write this)
task = "Research the history of JWT security"
task_type = classify_task(task)  # YOUR function returns "research"
# This could be keyword matching, embeddings, or even an LLM call

# Step 2: CALL PORTKEY WITH METADATA
response = portkey.chat(
    messages=[{"role": "user", "content": task}],
    metadata={"task_type": task_type}  # YOU tell Portkey it's "research"
)

# Step 3: PORTKEY ROUTES BASED ON YOUR METADATA
# Portkey sees metadata.task_type == "research" → routes to Gemini
```

**In other words:**
- Portkey is the **router** (it follows rules you define)
- YOU are the **classifier** (you determine the task type)
- Portkey does not understand task semantics

### Portkey and Claude Max Subscriptions

**Critical limitation:** Portkey requires API keys. Claude Max subscriptions use OAuth.

As of January 2026, Anthropic banned third-party tools from using OAuth-scoped credentials:
> "Claude Code Cripples Third-Party Coding Agents from using OAuth"

This means:
- If you have Claude Max ($200/mo) — you CANNOT use it through Portkey
- Portkey requires an Anthropic API key (pay-per-token, separate from Max)
- You'd be paying for Max AND API credits — double cost

### Portkey: What It Does Well

1. **Metadata-based conditional routing** — Complex IF/THEN rules on metadata fields
2. **Excellent observability** — Every request logged with costs, latency, tokens
3. **Caching** — Identical requests return cached responses (saves money)
4. **Failover and retries** — If Provider A fails, try Provider B
5. **Cost controls** — Per-team, per-project spending limits
6. **Enterprise security** — SOC 2, HIPAA, audit logs

### Portkey: What Breaks (Limitations)

1. **Cannot use Claude Max subscriptions** — OAuth banned by Anthropic
2. **No semantic understanding** — Routing is metadata-based, not intent-based
3. **Stateless** — No memory between requests
4. **You build the classifier** — Portkey routes, but you determine task type
5. **Cloud dependency** — Requests go through Portkey's servers

### Portkey: With the Harness

| Gap | Portkey Today | With Harness |
|-----|---------------|--------------|
| Semantic routing | Metadata only (you classify) | Harness classifies task type automatically |
| Subscription auth | API keys only | Works with Max subscription natively |
| Memory | Stateless | Persistent memory across requests |
| Classification | You build it | Built-in lightweight classifier |

---

## Tool 6: OpenRouter

### What Is OpenRouter?

**OpenRouter** is an API gateway that provides unified access to 300+ AI models from 60+ providers through a single API endpoint. Instead of integrating with Anthropic, OpenAI, and Google separately, you integrate once with OpenRouter.

**Website:** https://openrouter.ai

**The core idea:** One API, all models. You call `openrouter.ai/api/v1/messages` with any model name (Claude, GPT-4, Gemini, Llama, Mistral...) and OpenRouter routes to the right provider.

### OpenRouter vs Cursor: Understanding the Difference

The user asked: *"Is OpenRouter like Cursor in that case where it has access to these models?"*

**They're similar in that both give you access to multiple AI models. But they're architecturally different:**

| Aspect | Cursor | OpenRouter |
|--------|--------|------------|
| **What it is** | IDE (code editor) with AI built-in | API gateway (infrastructure) |
| **Interface** | Visual editor you type code in | API endpoint you call from code |
| **How you use it** | Open app, write code, AI assists | Call API from your application |
| **Model selection** | Dropdown menu in the UI | Change `model` field in API call |
| **Target user** | Developers wanting AI-assisted coding | Developers building AI apps/tools |
| **Runs where** | On your laptop (desktop app) | In the cloud (you call their API) |

**Analogy:**
- **Cursor** is like a restaurant — you sit down, order from a menu, food appears
- **OpenRouter** is like a food distributor — restaurants buy ingredients from them

### How Cursor Pricing Works (The API Budget Question)

The user asked: *"For Cursor, if I want to use Opus 4.5, I only have a specific API budget allocated for that?"*

**Cursor Pro ($20/month) includes:**
- **Unlimited "slow" requests** — Queued, may wait during peak times
- **500 "fast" requests per month** — Priority processing
- **Access to:** GPT-4, Claude Sonnet, Claude Opus, and others
- **Premium models (like Opus 4.5):** Count as fast requests

**Key point: There is NOT a per-model API budget.**

You get 500 fast requests TOTAL. Every model draws from the same pool:
- Use Opus 4.5 → 1 fast request
- Use GPT-4 → 1 fast request
- Use Sonnet → 1 fast request
- All count against your 500

**After 500 fast requests:**
- You can still use slow requests (unlimited, but queued)
- Or pay for more fast requests ($0.04 per request)

**Cursor is NOT paying per-token for you.** They've negotiated bulk deals with providers. Your $20/mo covers their costs up to your usage limits.

### How OpenRouter Pricing Works

**Two models:**

**1. OpenRouter Credits (prepaid)**
- You buy credits ($10, $50, $100)
- Each model has a price per 1M tokens
- Example: Claude Sonnet = $3/1M input, $15/1M output
- You pay as you go, credits deplete

**2. BYOK (Bring Your Own Key)**
- You provide your own API keys (e.g., from platform.anthropic.com)
- The provider bills you directly
- OpenRouter takes a 5% convenience fee
- Good if you already have API credits

### Can You Use Claude Max Through OpenRouter?

**No.** This is a critical limitation.

- **Claude Max ($200/mo)** = Subscription with OAuth authentication
- **OpenRouter** = Requires API keys
- **These are incompatible authentication methods**

If you try to use Max through OpenRouter:
- OpenRouter can't accept OAuth tokens
- You'd need a separate Anthropic API key
- That API key bills per-token, separate from Max
- You'd pay BOTH: Max subscription + API usage

### What Is OpenRouter's Auto Router?

OpenRouter has an "Auto Router" feature powered by NotDiamond:
- It learns which models perform best for which task types
- You send a request with `model: "auto"`
- Auto Router analyzes the prompt and picks the best model

**BUT there's a catch:** Auto Router requires 15+ labeled examples to train. You must provide training data before it works well.

### OpenRouter: What It Does Well

1. **Single endpoint for 300+ models** — One integration, all providers
2. **Easy model switching** — Just change the model name
3. **Auto Router** — Learned routing (with training data)
4. **Cost comparison** — See pricing across all models
5. **Fallback support** — If model A fails, try model B
6. **BYOK option** — Use your own API keys

### OpenRouter: What Breaks (Limitations)

1. **Cannot use Claude Max subscriptions** — API keys only, no OAuth
2. **Auto Router requires training** — 15+ examples minimum
3. **Claude Code hardcodes model selection** — Auto Router not exposed
4. **No shared memory** — Stateless, each request independent
5. **5% BYOK fee** — Adds cost on top of provider pricing

### OpenRouter: With the Harness

| Gap | OpenRouter Today | With Harness |
|-----|------------------|--------------|
| Subscription auth | API keys only | Works with Max subscription natively |
| Auto routing | Requires 15+ training examples | Zero-config semantic routing |
| Integration with Claude Code | Model selection hardcoded | MCP server exposes routing |
| Memory | Stateless | Persistent memory layer |

---

## Tool 7: Conductor Build

### What Is Conductor Build?

**Conductor Build** (by Melty Labs, YC-backed) is a macOS Electron app that runs multiple Claude Code instances in parallel, each isolated in its own git worktree. It provides a visual dashboard to monitor and manage all agents simultaneously.

**Website:** https://conductor.build
**Backing:** Y Combinator

**The core idea:** If you have 5 independent features to build, why work on them sequentially? Conductor lets you spin up 5 Claude Code instances, each working on a different feature in isolation, and then merge all the work back together.

### Does Conductor CREATE or DISCOVER Agents?

**Conductor CREATES the agents. The UI is the controller, not a viewer.**

This is a critical architectural distinction:

**What Conductor does:**
1. You open Conductor (macOS app)
2. You add your GitHub repo
3. You click "Create 5 agents"
4. **Conductor creates** 5 git worktrees (isolated branch copies)
5. **Conductor spawns** 5 Claude Code CLI instances (new processes)
6. **Conductor manages** them throughout their lifecycle
7. Dashboard shows what Conductor created and controls

**What Conductor does NOT do:**
- Scan for existing Claude Code instances
- Detect "ambient" processes
- Wrap externally-started agents
- Read any kind of stack trace

**If you manually run `claude` in a terminal outside Conductor, Conductor will NOT see it.**

### The Process Hierarchy

```
Conductor Build (Electron app — YOUR entrypoint)
  │
  ├── Creates: git worktree at .git/worktrees/agent-1/
  │   └── Spawns: Claude Code CLI instance #1 (PID 1234)
  │
  ├── Creates: git worktree at .git/worktrees/agent-2/
  │   └── Spawns: Claude Code CLI instance #2 (PID 1235)
  │
  ├── Creates: git worktree at .git/worktrees/agent-3/
  │   └── Spawns: Claude Code CLI instance #3 (PID 1236)
  │
  ├── Creates: git worktree at .git/worktrees/agent-4/
  │   └── Spawns: Claude Code CLI instance #4 (PID 1237)
  │
  └── Creates: git worktree at .git/worktrees/agent-5/
      └── Spawns: Claude Code CLI instance #5 (PID 1238)
```

**Conductor is the parent process.** When Conductor exits, it cleans up the child processes.

### How Git Worktrees Enable Isolation

**What's a git worktree?**
A worktree is a lightweight checkout of a different branch in a separate directory. Each worktree has its own file state but shares the same .git repository.

```
~/project/                          # Main checkout (branch: main)
~/project/.git/worktrees/agent-1/   # Agent 1 (branch: feature-auth)
~/project/.git/worktrees/agent-2/   # Agent 2 (branch: feature-api)
~/project/.git/worktrees/agent-3/   # Agent 3 (branch: feature-tests)
```

**Why this matters:**
- Agent 1 can modify auth.ts without affecting Agent 2's copy
- No merge conflicts during development
- Each agent has independent file state
- When done, Conductor helps merge all branches

### What the Dashboard Shows

**The dashboard displays what Conductor CREATED:**

1. **Agent status** — Running, idle, errored (for each of 5 instances)
2. **Terminal output** — Live stream from each Claude Code instance
3. **File changes** — What files each agent modified
4. **Diffs** — Comparison between agent branch and main
5. **Merge preview** — What the PR will look like
6. **Progress timeline** — Visual representation of work

**The dashboard does NOT show:**
- Claude Code instances started manually elsewhere
- External tools or agents
- Anything Conductor didn't explicitly create

### The Rate Limit Problem

**5 agents = 5x rate limit burn**

All 5 agents use your single Claude Max subscription:
- Claude Max gives you ~240-480 hours/week of Sonnet 4
- 5 parallel agents consume this 5x faster
- What normally lasts a week lasts 1-2 days

**What happens when you hit limits:**
- Agents 1-3 might be working fine
- Agents 4-5 start hitting 429 errors
- Conductor has NO fallback logic
- Agents 4-5 stall silently or error out
- You have to manually restart or wait for reset

### Conductor: What It Does Well

1. **True parallel execution** — 5 agents on 5 CPU cores simultaneously
2. **Git isolation via worktrees** — No file conflicts during development
3. **Visual dashboard** — See all agents' progress at once
4. **Merge coordination** — Helps combine branches when done
5. **No manual git commands** — Worktree management is automatic
6. **YC-backed, actively developed** — Real company, real support

### Conductor: What Breaks (Limitations)

1. **Claude-only** — No Gemini, no GPT-4, no model selection
2. **No rate limit awareness** — 5 agents burn quota 5x faster, no throttling
3. **No cross-agent communication** — Agents can't see what each other is doing
4. **No shared memory** — Each agent is fully isolated
5. **No semantic task routing** — You manually assign tasks
6. **Silent failures** — Agents stall on 429 without clear notification
7. **macOS only** — No Windows, no Linux, no web

### Conductor: With the Harness

| Gap | Conductor Today | With Harness |
|-----|-----------------|--------------|
| Model support | Claude only | Claude + Gemini + others |
| Rate limits | 5x burn, no awareness | Quota-aware throttling, load balancing |
| Cross-agent comms | None | Shared message bus (hcom-style) |
| Task routing | Manual assignment | Semantic routing to appropriate model |
| Failure handling | Silent stalls | Retry logic, graceful degradation |
| Memory sharing | None | Shared memory layer across agents |

---

## Tool 8: CLIProxyAPI

### What Is CLIProxyAPI?

**CLIProxyAPI** is a local proxy server with a companion macOS menu bar widget (called **Quotio** or **CodexBar** depending on version) that provides two things: (1) visual quota tracking across AI providers, and (2) round-robin load balancing across multiple accounts or API keys.

**Repository:** https://github.com/nicobako/cliProxyAPI (and related menu bar tools)

**The core idea:** You want to see at a glance how much quota you've used across Claude, Gemini, OpenAI, etc. — and you want to automatically spread load across multiple accounts when you have them.

**Architecture:**
```
Claude Code → CLIProxyAPI (localhost:8080) → Anthropic/Google/OpenAI
                    ↓
            Menu Bar Widget
         (Claude 60% | Gemini 20%)
```

### How CLIProxyAPI Works (The Mechanics)

**Quota Tracking:**

1. CLIProxyAPI intercepts every request going to AI providers
2. It logs token usage from responses (input tokens + output tokens)
3. It maintains running totals per provider per time window
4. Menu bar widget polls this data and displays percentages

**What the menu bar shows:**
```
┌─────────────────────────────────────┐
│  Claude: ████████░░ 78%             │
│  Gemini: ██░░░░░░░░ 15%             │
│  OpenAI: ░░░░░░░░░░  3%             │
│  ─────────────────────              │
│  Today: $4.23 spent                 │
└─────────────────────────────────────┘
```

**Round-Robin Load Balancing:**

If you have multiple Anthropic API keys (e.g., from different accounts):
```yaml
accounts:
  - name: "personal"
    api_key: "sk-ant-xxx1"
  - name: "work"
    api_key: "sk-ant-xxx2"
  - name: "project"
    api_key: "sk-ant-xxx3"
```

CLIProxyAPI rotates through them:
- Request 1 → personal account
- Request 2 → work account
- Request 3 → project account
- Request 4 → personal account (cycle repeats)

**Why round-robin?** Each account has independent rate limits. 3 accounts = 3x the effective quota.

### The Visibility vs Action Gap

**What CLIProxyAPI shows you:**
- Real-time quota percentages
- Historical usage graphs
- Cost accumulation
- Which account is being used

**What CLIProxyAPI does NOT do:**
- Act on quota thresholds ("Route to Gemini when Claude hits 80%")
- Understand task type ("This is research, use Gemini")
- Predict exhaustion ("At current rate, you'll hit limits in 2 hours")
- Proactively switch providers

**The gap is: visibility without agency.**

You SEE that Claude is at 85%. But CLIProxyAPI won't DO anything about it. YOU must manually decide to switch tasks or providers.

### Reactive vs Proactive: The Critical Distinction

**Reactive behavior (what CLIProxyAPI does):**
```
Request → Provider → 429 Error → Failover to next account
```
You've already HIT the limit. The request failed. Then it tries another account.

**Proactive behavior (what the harness would do):**
```
Request → Check: "Claude at 82%" → Route to Gemini → Success
```
You NEVER hit the limit. The system routes preemptively.

**Why this matters:**

1. **Failed requests waste time** — You wait for timeout, then retry
2. **Context is lost** — The request that failed might have been mid-conversation
3. **Rate limit penalties** — Some providers penalize you for hitting limits
4. **User experience** — "429 Too Many Requests" is jarring vs seamless routing

### The Round-Robin Session Problem

**What round-robin loses:**

When CLIProxyAPI rotates through accounts, each request potentially goes to a different account. For API calls, this usually doesn't matter. But for Claude Code sessions:

```
Turn 1: Account A (Claude has context)
Turn 2: Account B (different billing, but same model)
Turn 3: Account C
Turn 4: Account A again
```

**Problems:**
1. **Caching benefits lost** — Anthropic caches prompts per-account; rotating loses this
2. **Session coherence** — Some providers track conversation state per-account
3. **Audit confusion** — Hard to trace which conversation used which account

**For your use case:** You don't have multiple API keys — you have one Claude Max subscription. Round-robin isn't applicable.

### CLIProxyAPI: What It Does Well

1. **Glanceable quota visibility** — Menu bar always shows current usage
2. **Multi-account load distribution** — Spreads load if you have multiple API keys
3. **Post-hoc failover** — If one account fails, tries the next
4. **Cost tracking** — See total spend per provider per day
5. **Lightweight** — Menu bar widget, minimal overhead
6. **Works with Claude Max** — Tracks subscription usage (not just API)

### CLIProxyAPI: What Breaks (Limitations)

1. **Reactive only** — Fails over AFTER 429, not before
2. **No semantic routing** — Doesn't understand task type
3. **No proactive thresholds** — Can't say "switch at 80%"
4. **Round-robin loses caching** — Session coherence compromised
5. **Display only** — Shows data but doesn't act on it
6. **No cross-session memory** — Quota tracking only, no content memory
7. **Single-provider focus** — Designed for multiple accounts of SAME provider

### CLIProxyAPI: With the Harness

| Gap | CLIProxyAPI Today | With Harness |
|-----|-------------------|--------------|
| Proactive routing | None — displays quota, doesn't act | Routes at 80% threshold automatically |
| Semantic awareness | None — treats all requests equally | Routes research to Gemini, coding to Claude |
| Threshold triggers | None — just displays percentages | Configurable: "At 75%, route non-critical to Gemini" |
| Prediction | None — shows current state only | "At current rate, 2.3 hours until limit" |
| Action on data | Display only | Data drives routing decisions |
| Cross-provider routing | Same provider only (multi-account) | Routes between Claude AND Gemini based on task |

**Concrete scenario where CLIProxyAPI falls short:**
```
10:00 AM - Menu bar shows: Claude 72%
  You see this, but you're in the middle of debugging
  You think "I'll switch to Gemini later"

10:45 AM - Menu bar shows: Claude 85%
  Still debugging, don't want to break flow

11:15 AM - Request fails with 429
  Menu bar shows: Claude 100%
  Your debugging session is interrupted
  You manually restart with Gemini, losing context

With the harness:
10:30 AM - Harness sees: Claude 78%
  "Next request is research (finding similar bugs)"
  Routes to Gemini automatically, preserving Claude quota

11:00 AM - Menu bar shows: Claude 82%
  Debugging continues uninterrupted on Claude
  Research tasks handled by Gemini in background

12:00 PM - Still working, no 429 errors
  Intelligent routing extended your Claude session
```

---

## The ~40% Gap Summary

### What EXISTS in current tools:
- Token-count routing (CCProxy)
- Auto-failover on 429 (claude-code-mux)
- Manual delegation (PAL MCP)
- Real-time messaging (hcom)
- Parallel instances (Conductor)
- Quota visibility (CLIProxyAPI)

### What NOBODY provides:
1. **Semantic task routing** — Understanding "this is research" vs "this is coding"
2. **Persistent cross-session memory** — Yesterday's findings available today
3. **Proactive rate prediction** — Act at 80%, not react to 429
4. **Automatic mid-task context handoff** — Preserve reasoning state during failover
5. **Cross-model critique orchestration** — Claude generates, Gemini reviews, loop back

---

## Terminology: Is "Harness" the Right Word?

An **agent harness** is infrastructure that wraps an LLM and manages what the model can't do itself:
- Execute tool calls
- Manage memory
- Structure workflows
- Handle long-running tasks

**Your concept is a hybrid:**
- Harness (memory, tool execution)
- Orchestrator (routing decisions)
- Proxy/Router (request interception)

**Better terms might be:**
- "Agent Orchestration Layer"
- "Multi-Model Harness"
- "Agentic Router"

---

## Key Clarifications (User Questions Answered)

### "Why don't these tools do semantic routing?"

**The short answer:** It requires an LLM call (expensive, adds latency) or a trained classifier (requires labeled data). Most tools prioritize simplicity and speed.

**Technical requirement for semantic routing:**
1. Parse the message content
2. Either: Run an LLM call to classify (50-200ms, costs tokens)
3. Or: Use embeddings + similarity search (requires pre-embedded examples)
4. Or: Train a lightweight classifier (requires labeled training data)

Most tools are designed as **transparent proxies** — they add minimal overhead. Semantic routing adds 50-200ms latency. That's a design tradeoff.

### "Why doesn't claude-code-mux use Mem0 for context continuity?"

claude-code-mux is a **stateless proxy** by design:
- Each request is independent
- No database, no storage, no persistence
- Just forwards requests and catches errors

Adding Mem0 would require:
- Running a database/service
- Extracting and storing reasoning state (hard problem)
- Injecting that state into new provider's context
- Significant architectural change from "proxy" to "orchestration layer"

**Could they add it?** Yes. But it would change the tool's nature entirely.

### "Is Cursor's pricing per-model?"

**No.** Cursor Pro gives you 500 fast requests total, regardless of model. Opus 4.5 uses 1 request. GPT-4 uses 1 request. Same pool.

### "What IS the clink tool?"

`clink` = CLI Link. It's a tool in PAL MCP that spawns external CLI processes:
- Claude calls `clink(cli_name="gemini", role="security-analyzer", prompt="...")`
- PAL spawns: `gemini -p "your prompt" --output-format json --yolo`
- Gemini runs, produces output
- PAL returns result to Claude's conversation

### "Who's messaging whom in hcom?"

The **hook system** messages, not the agents directly. When Claude writes a file:
1. Claude's `PostToolUse` hook fires automatically
2. Hook script posts event to SQLite: "Claude wrote auth.ts"
3. Gemini's hook polls SQLite, sees the event
4. Gemini reacts (reads file, reviews, posts response)

Agents don't explicitly "message" — the hooks capture events and broadcast them.

### "Does Conductor create agents or discover them?"

**Conductor creates them.** The UI is the control plane:
- You click "Create 5 agents"
- Conductor creates 5 git worktrees
- Conductor spawns 5 Claude Code processes
- Dashboard shows what Conductor created

There's NO ambient detection. If you run `claude` manually in a terminal, Conductor won't see it.

---

## Summary: The ~40% Gap

### What EXISTS in current tools:
| Capability | Tool(s) |
|------------|---------|
| Token-count routing | CCProxy |
| Auto-failover on 429 | claude-code-mux |
| Manual delegation | PAL MCP (clink tool) |
| Real-time messaging | hcom |
| Parallel instances | Conductor Build |
| Quota visibility | CLIProxyAPI, CodexBar |
| Metadata routing | Portkey |
| Multi-model access | OpenRouter |

### What NOBODY provides:
| Capability | Why It Matters |
|------------|----------------|
| **Semantic task routing** | Route by intent (research vs coding), not just metadata |
| **Persistent cross-session memory** | Yesterday's findings available today |
| **Proactive rate prediction** | Act at 80%, not react to 429 |
| **Automatic mid-task context handoff** | Preserve reasoning state during failover |
| **Cross-model critique orchestration** | Claude generates, Gemini reviews, loop back |
| **Subscription auth for routing** | Use Max subscription through routing layer |

---

## Terminology: Is "Harness" the Right Word?

An **agent harness** is infrastructure that wraps an LLM and manages what the model can't do itself:
- Execute tool calls
- Manage memory
- Structure workflows
- Handle long-running tasks

**Your concept is a hybrid:**
- Harness (memory, tool execution)
- Orchestrator (routing decisions)
- Proxy/Router (request interception)

**Better terms might be:**
- "Agent Orchestration Layer" — clearest
- "Multi-Model Harness" — extends the harness definition
- "Agentic Router" — if emphasizing routing
