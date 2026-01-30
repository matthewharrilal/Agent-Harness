# Session 3 Synthesis: The Architecture Takes Shape

**Date:** January 29, 2026
**Status:** Investigation Phase — Refined Lanes Established
**Entry Point:** This is where new sessions should start
**Session Duration:** ~3 hours of deep research with 5 parallel agents

---

## Architecture at a Glance

> **Full layer-by-layer detail:** See [`docs/ARCHITECTURE_VISION.md`](ARCHITECTURE_VISION.md) for all 5 individual layer boxes with candidates, unknowns, and dependencies.

### The Integrated System (How Layers Connect)

```
 USER types: claude
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│ CLAUDE CODE (interactive, normal experience)          LAYER 5   │
│ Reads CLAUDE.md → knows when to use harness tools               │
│ Reads .mcp.json → spawns harness MCP server                     │
│ Hooks fire on every event → captured by Layer 4                 │
└────────────────────────┬────────────────────────────────────────┘
                         │ user prompt arrives
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ SEMANTIC ROUTER                                       LAYER 1   │
│ "refactor auth module" → { type: "code", conf: 0.91 }          │
└────────────────────────┬────────────────────────────────────────┘
                         │ classification
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ DECISION ENGINE                                       LAYER 2   │
│                                                                  │
│  route signal ──┐                                                │
│  rate limits ───┤                                                │
│  memory ────────┼──► DECISION: Claude handles Task A + B         │
│  subtask ctx ───┘              Gemini handles Task C             │
│                                                                  │
└──────────┬───────────────────────────────┬──────────────────────┘
           │                               │
           ▼                               ▼
┌────────────────────┐          ┌────────────────────┐
│ CLAUDE (local)     │          │ GEMINI (delegated)  │
│ Task A: audit      │          │ Task C: 140K token  │
│ Task B: refactor   │          │ codebase analysis   │
│                    │          │                     │
│ Uses native tools  │          │ Spawned via MCP or  │
│ (Read, Edit, Bash) │          │ Bash (TBD)          │
└────────┬───────────┘          └──────────┬──────────┘
         │                                 │
         │       writes findings           │
         ▼                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│ CROSS-PROVIDER MEMORY                                 LAYER 3   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Task A findings: "3 security issues in auth.ts"        │    │
│  │  Task C findings: "47 call sites across 8 patterns"     │    │
│  │  Session memory: "JWT decision: 3 formats"              │    │
│  │  Routing history: "last time, Claude solved this in 1"  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ◄─── Task B reads Task A's findings directly (mesh) ───►       │
│  ◄─── Router reads past decisions (feedback loop) ───►           │
│  ◄─── Next session reads today's learnings ───►                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
         │
         │ events flow via hooks
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ VISUAL INTERFACE                                      LAYER 4   │
│                                                                  │
│  Menu bar: ● Claude 42% │ Gemini 88% │ $0.50                    │
│                                                                  │
│  Browser (optional):                                             │
│  "Task C → Gemini. Reason: 140K tokens. Cost: $0.08 vs $0.32"  │
└─────────────────────────────────────────────────────────────────┘
```

### What's Locked vs Open

```
██ LOCKED:
  Hybrid Claude+Gemini │ Claude orchestrates │ Veto critique
  Invisible experience │ Gemini free/$20 │ Exhaustion = stop

░░ INVESTIGATION LANE:
  Semantic routing │ Communication │ Visibility │ Memory tool │ MCP

   OPEN:
  Memory selection │ MCP vs Bash │ Zero-code baseline │ Decision engine
  Visual design │ MVP scope │ Build order validation
```

---

## Part 1: Where We Are Now

### The Core Problem (Never Lose Sight)

User hits 85% of weekly Claude Max limits in 1-2 days → stops working. The friction isn't just capacity — it's four compounding problems:

1. **Rate limits force me to stop** — Literal work stoppage at 429
2. **I don't know which model to use** — Waste tokens on wrong model
3. **Context-switching loses continuity** — Switching loses reasoning state
4. **I re-explain things constantly** — Each session starts cold

These aren't separate problems. They compound:
- Don't know which model → waste Claude tokens on research
- Waste tokens → hit limits faster
- Hit limits → switch to Gemini → lose context
- New session → re-explain everything → waste more tokens

**User's core insight (verbatim from HANDOFF.md):**
> "The only thing I'm really getting from two different accounts is the combined amount of inference capabilities. Everything else should be shared."

### The Research That Got Us Here

This session launched 5 parallel research agents:

| Agent | Mission | Key Output |
|-------|---------|------------|
| Task Decomposition | What frameworks exist? Build vs adopt? | Hybrid approach: implicit (CLAUDE.md) → semantic router |
| Inter-Agent Communication | How do subtasks talk? | Hybrid hierarchical + mesh (evolved from initial Blackboard finding) |
| LLM Routing | Can we route without training a model? | RouteLLM/Aurelio work out-of-box at 90-95% accuracy |
| Adversarial/Council | How does Team of Rivals fit? | Veto pattern, user-triggered, not consensus |
| Hidden Assumptions | What are we subconsciously committing to? | 10 critical challenges surfaced |

---

## Part 1.5: The Core Insight — One System, Not Three Features

The four compounding problems in Part 1 require a **single integrated system**, not separate features bolted together:

```
                  ┌─────────────────────────┐
                  │   SEMANTIC ROUTING      │
                  │   "What is this task?"  │
                  └───────────┬─────────────┘
                              │ informs
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
  │ RATE LIMIT    │   │ CROSS-PROVIDER │   │ SUBTASK       │
  │ PREDICTION    │   │ MEMORY         │   │ CONTEXT       │
  │ "Who has      │   │ "What do we    │   │ "What did     │
  │  capacity?"   │   │  already know?"│   │  sibling find?│
  └───────────────┘   └───────────────┘   └───────────────┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                  ┌───────────▼─────────────┐
                  │   ROUTING DECISION      │
                  │   Claude or Gemini?     │
                  └─────────────────────────┘
```

**Routing USES memory.** The semantic router doesn't just classify "is this research or code?" — it asks "what do we already know about similar tasks?" and "what did the last routing decision for this pattern produce?"

**Memory INFORMS routing.** If memory says "Claude spent 8 retries on this task type last week and Gemini solved it in one pass," that's routing signal, not just historical record.

**Rate limits CONSTRAIN routing.** A routing decision that says "send to Claude" but doesn't check capacity is useless. Rate prediction isn't a Phase 2 add-on — it's part of every routing decision.

**They're not separate features — they're one system.**

### Implications for Build Order

The phased build order in Part 4 is still valid as an **implementation sequence**, but the mental model should be:

- Phase 1: Build the routing decision engine (which needs capacity awareness from day one)
- Phase 2: Add rate prediction **into** the router (not beside it)
- Phase 3: Add memory **into** the router (it remembers past decisions)
- Phase 4: Wrap the unified system in MCP

Each phase **extends** the router, not **adds a new system**.

### The 5-Layer Architecture (Session 3 Vision)

| Layer | What | Investigation Status |
|-------|------|---------------------|
| **1. Semantic Router** | Aurelio or RouteLLM | Investigating — no choice made |
| **2. State-Aware Decision Engine** | Custom | Conceptual — consumes route + rate limits + memory + subtask context |
| **3. Cross-Provider Memory** | Mem0 / sqlite-vec / files | Investigating — no choice made. Architecturally central (double duty) |
| **4. Visual Interface** | CodexBar + Conductor Build inspired | Conceptual — not designed |
| **5. CLI Experience** | MCP server + hooks | Level 3 — "don't double down on MCP yet" |

Layer 2 is where the "one system, not three features" insight lives — the integration point. No concrete implementation decided for any layer.

> **Full architecture diagrams:** See [`docs/ARCHITECTURE_VISION.md`](ARCHITECTURE_VISION.md) for the complete 5-layer breakdown, integrated system flow, dependency graph, and status map.

---

## Part 2: Architectural Decisions LOCKED

### Decision 1: Hybrid Claude+Gemini (Confidence: 5 — LOCKED)

**What:** Claude Max ($200/mo) + Gemini Free/Pro ($0-20/mo), NOT dual-Claude

**Why this is locked:**
- Device-level keychain bug makes dual-Claude broken (6+ GitHub issues)
- Not fixable by us — it's in macOS Keychain + Anthropic's CLI
- Cost: $200-220/mo vs $400/mo for broken dual-Claude

**The evidence:**
- macOS Keychain namespace collision under `Claude Code-credentials`
- Missing `-U` flag in `security add-generic-password`
- In-memory session caching bypasses keychain switching
- 6+ GitHub issues documenting the pattern, no fix in changelog

**What this means for architecture:**
- Gemini is the overflow provider, not a second Claude
- Different strengths: Claude (precision, debugging) vs Gemini (breadth, 1M context)
- Cross-provider memory becomes critical (they don't share context natively)

---

### Decision 2: Claude as Top-Level Orchestrator (Confidence: 5 — LOCKED)

**What:** Claude ALWAYS orchestrates. Subtask routing is dynamic, but orchestrator identity never changes.

**Why this is locked:**
- NeurIPS papers + MAST taxonomy (1600+ traces) confirm fixed orchestrator is production standard
- No production system dynamically rotates orchestrator role
- User explicitly confirmed this direction

**The research finding (verbatim from ARCHITECTURE.md):**
> "Research found that NO production multi-agent system dynamically switches the orchestrator role. Every framework (AutoGen, CrewAI, Google ADK, LangGraph, Manus AI) uses a fixed orchestrator with dynamic worker selection."

**What this means for architecture:**
```
Claude Code (orchestrator)
  ├── Decomposes tasks
  ├── Routes subtasks to Claude or Gemini
  ├── Monitors execution
  └── Synthesizes results

Gemini CLI (delegated subtask)
  └── Executes specific work
  └── Returns result to shared state
```

---

### Decision 3: Inter-Agent Communication — Hybrid Hierarchical + Mesh (Confidence: 4 — INVESTIGATION LANE)

**What:** Orchestrator controls flow (hierarchical). Subtasks share context directly (mesh). Session 3 evolved this from Session 2's Blackboard pattern, and investigation continues into direct subagent-to-subagent messaging through persistent messaging platforms.

**The user's insight (validated by research):**
> "What's the connective tissue? How do parallel subtasks communicate when distributed across models?"

**The user's parallel communication insight (verbatim, Session 3):**
> "What if subtasks communicate with each other in parallel, not just relay back to orchestrator? But still have some individualism, not just shared writes."

### Dual-Channel Architecture

The communication model separates two distinct channels:

| Channel | What Flows | Mechanism |
|---------|-----------|-----------|
| **Control flow** | Task assignment, completion signals, failures, synthesis triggers | Hub-and-spoke (orchestrator controls) |
| **Context** | Findings, intermediate results, shared learnings | Shared state (tasks read/write directly) |

**The Session 2 starting point (evolved in Session 3):**
> "The connective tissue between agents is NOT direct messaging. It's **shared state (Blackboard pattern)**:
> - Agents don't talk to each other — they read/write shared memory
> - Framework comparison (LangGraph, CrewAI, ADK, Swarm) confirms this
> - AgentMail was a red herring (email for humans, not agent-to-agent)
> - Conductor proves single-provider harnesses exist — ours adds the cross-provider layer"

**Why NOT other patterns:**

| Pattern | Problem |
|---------|---------|
| Pure hub-and-spoke | Bottleneck — orchestrator relays all context |
| Pure blackboard | No control flow — who synthesizes? |
| Full mesh P2P | Chaos — debugging nightmare, consistency issues |
| Direct messaging (AgentMail) | Red herring — that's for humans, not agent-to-agent |

**The architecture (Session 3 — bidirectional):**
```
           ┌──────────────────┐
           │  ORCHESTRATOR    │
           │  (Claude)        │
           └────────┬─────────┘
                    │ spawns subtasks, receives final results
                    │
      ┌─────────────┼─────────────┐
      │             │             │
  ┌───▼───┐   ┌────▼────┐   ┌────▼────┐
  │Task A │◄─►│ Task B  │◄─►│ Task C  │
  │(Claude)│   │(Gemini) │   │(Claude) │
  └───────┘   └─────────┘   └─────────┘
      │             │             │
      └─────────────┴─────────────┘
            shared context layer
```

**Traditional vs Session 3 Vision:**

| | Traditional (hub-and-spoke) | Session 3 vision (hybrid) |
|-|----------------------------|--------------------------|
| **Flow** | Task A → Orchestrator → Task C | Task A → shared context → Task C reads directly |
| **Orchestrator role** | Relays ALL intermediate results | Spawns, monitors, failures, synthesizes final |
| **Bottleneck** | Orchestrator becomes relay station | Orchestrator freed — control flow only |
| **Task awareness** | Tasks know only what orchestrator tells them | Tasks see sibling findings directly |

### Open Investigation: Direct Messaging

> The user is exploring whether subtasks can communicate through a persistent messaging platform (not just shared context reads/writes) — enabling direct peer-to-peer coordination that's stored for future reference. This goes beyond the Blackboard pattern into active message passing. Under investigation.

### What's Tricky (Implementation Challenges)

| Challenge | Problem | Potential Approaches |
|-----------|---------|---------------------|
| Polling vs events | How does Task C know Task A finished? | File watch, SQLite polling, Mem0 triggers |
| Stale data | Task C reads before Task A completes | Versioning, timestamps, "done" markers |
| When to synthesize | Orchestrator needs to know ALL tasks finished | Completion counters, sync points |

### Three Practical Options for Shared State

1. **Shared scratchpad file** (simplest): `.harness/context.json`, both providers read/write, risk of race conditions
2. **SQLite + polling** (structured): Atomic writes, queryable history, schema with task_id/status/findings
3. **Mem0** (semantic): Vector storage, "What did we learn about auth?" queries, more infrastructure

### Phased Approach

- Phase 1: Shared file (get it working)
- Phase 2: If concurrency issues, migrate to SQLite
- Phase 3: If semantic retrieval proves valuable, add Mem0

### Memory Serves Double Duty

Cross-provider memory isn't just a storage layer — it's the **foundation** of the communication pattern:

1. **Cross-subtask context:** Task B sees Task A's findings via shared memory, not orchestrator relay
2. **Cross-session persistence:** Tomorrow's session knows what today's session learned

This means memory is architecturally central from day one, even if implementation starts simple (files). The tool choice (Mem0 / sqlite-vec / files) is still open, but the architectural role is load-bearing.

**What this means for implementation:**
- Need a shared state layer (files, Mem0, or sqlite-vec)
- Both Claude and Gemini must be able to read/write it
- Orchestrator freed from context relay — focuses on control flow

---

### Decision 4: Semantic Routing Without Training (Confidence: 4 — INVESTIGATION LANE)

**What:** Route tasks to the right model using pre-trained routers. No custom model training.

**The user's requirement:**
> "I don't really want to train a model. Is there a wheel that's already been invented?"

**The research answer: YES, several wheels exist.**

#### Option A: RouteLLM (Recommended for accuracy)
- **Accuracy:** 90-95%
- **Latency:** ~50ms
- **Training required:** NO — pre-trained routers generalize to new model pairs
- **How it works:** Trained on GPT-4 vs Mixtral, but "routers have learned common characteristics of problems that distinguish between strong and weak models, which generalize to new model pairs"
- **Setup:** Point at Claude + Gemini, use default config

**Research finding (verbatim):**
> "RouteLLM: 95% of GPT-4 performance using only 14% GPT-4 calls"

#### Option B: Aurelio Semantic Router (Recommended for control)
- **Accuracy:** 70-85%
- **Latency:** ~100ms
- **Training required:** NO — uses embedding similarity
- **How it works (verbatim):**
  > "Convert queries to vectors, find similar past queries, route based on what worked. ~90% accuracy, sub-penny cost per 10K queries"

**Example implementation:**
```python
research_route = Route(name="research", utterances=[
    "analyze this codebase",
    "investigate why this is slow",
    "research JWT vulnerabilities"
])
code_route = Route(name="code", utterances=[
    "refactor this function",
    "debug the auth module"
])
router = SemanticRouter(routes=[research_route, code_route])
route = router(user_query)  # Returns "research" or "code"
```

#### Why NOT heuristics alone?

**Research finding (verbatim from SESSION_2_SYNTHESIS.md):**
> "The harness assumes 'route without LLM calls' (FSM routing). But how do you classify 'refactor auth module' as code vs research without asking a model?"

**Five unsolved problems with pure heuristics:**
1. Out-of-Distribution Generalization: Routers trained on chat fail on academic benchmarks
2. Non-Linear Decision Boundaries: Linear methods score 0.61-0.66, non-linear scores 0.74
3. Quality Estimation: Predicting response quality without seeing the response is hard
4. Interpretability Gap: We don't understand WHY routers make decisions
5. No Universal Optimal Router: Different routers win on different metrics

**Decision:** Phase 1: Integrate RouteLLM or Aurelio. Test with real tasks. No model training, no ML expertise required.

**Note:** Neither RouteLLM nor Aurelio has been tested on Claude+Gemini pair. Phase 1 investigation will validate accuracy on real tasks before any selection.

---

### Decision 5: Critique Pattern — Veto, Not Consensus (Confidence: 5 — LOCKED)

**What:** Cross-model critique uses veto authority, not democratic voting.

**The research finding (arxiv 2601.14351, verbatim from ARCHITECTURE.md):**
> "Single agent: 60% accuracy. Single agent + self-review: <60% (WORSE). Multi-agent cross-model critique: **90% accuracy**."

**Why veto beats consensus:**

| Approach | Token Cost | Accuracy | Failure Mode |
|----------|------------|----------|--------------|
| Consensus voting | 2-4x | +2.8% (knowledge tasks) | Agents can collude |
| Adversarial debate | 3x | +13.2% (reasoning tasks) | Expensive for routine work |
| **Veto/critique** | 1.5-2x | +30% (security, code) | Clear authority model |

**Research finding (verbatim from ARCHITECTURE.md):**
> "Hierarchical veto, not consensus voting. Critic can reject (triggers retry), not negotiate. Paper found veto > democratic consensus."

**The pattern:**
```
Claude generates solution
  ↓
Gemini critiques (doesn't generate alternative)
  ↓
If VETO: Claude revises (max 3 retries)
  ↓
If APPROVE: Done
  ↓
After 3 retries: Escalate to user
```

**When to trigger critique:**
- User types `/review` or `/critique` — always
- Automatic triggers (Tier 2):
  - Security-critical code (auth, encryption, SQL)
  - File count > 5 (large refactoring)
  - Context > 50K tokens
  - Architectural decisions

**When to SKIP critique:**
- Routine edits (typos, renames)
- Single file, <100 lines
- User says "quick iteration"

---

### Decision 6: Visibility Model — Silent CLI + Dashboard (Confidence: 4 — INVESTIGATION LANE)

**What:** Invisible in CLI experience, visible in dashboard when you want it.

**The user's clarification (verbatim from ARCHITECTURE.md):**
> "I want to type `claude` normally. Not headless mode. The harness should be baked into my Claude experience."

**The "invisible but observable" resolution:**
> "The harness is not a CLI wrapper. It is not a separate command you type. It is an MCP server + hooks system that runs inside your normal Claude Code experience. You type `claude` as you always do. The harness is baked in -- invisible, ambient, always present."

**The visibility clarification (verbatim from user, Session 3):**
> "I want it to be invisible from an experience standpoint, but that's where I think a visual interface would be useful if you do want that type of clarity and nuance to see what's happening behind the scenes. Which I think is where we kind of get inspiration from, like the Codex bar not only for rate limit but like a visual interface like we've seen with other like Conductor Build (not to be confused with Gemini Conductor) but like a visual interface like that as well. Yeah, so like Silent and Experience, but if I want that level of granularity and visibility, we built a visual interface for that."

**Key takeaways:**
- "Silent in Experience" = harness is invisible during normal CLI use
- "Visual for Granularity" = opt-in dashboard for debugging/understanding
- Conductor Build (explicitly NOT Gemini Conductor) = visual interface inspiration
- CodexBar = rate limit visibility inspiration specifically

**Two interfaces:**

1. **CLI (Silent):**
   - User types `claude` normally
   - Harness works invisibly via MCP server + hooks
   - No indication of routing unless you ask

2. **Dashboard (Optional visibility):**
   - CodexBar-style menu bar widget: `Claude: 76% │ Gemini: 22%`
   - Conductor Build-style web dashboard at localhost:3200 (not to be confused with Gemini Conductor)
   - Shows: routing decisions, cost, activity, critique history

**Why NOT "invisible only":**
- Can't debug routing decisions
- Can't understand why slow things are slow
- Leads to mistrust

**Why NOT "observable only":**
- Cognitive load on every task
- Breaks the "baked in" experience
- User doesn't want to think about routing

---

## Part 3: What's Still Open

### Open Question 1: Memory System Selection

**Candidates (from ARCHITECTURE.md):**
- **Mem0:** Semantic extraction, dual storage (SQLite + vectors), 91% latency savings
- **sqlite-vec:** Lighter weight, vector search, simpler
- **Files (AGENTS.md):** Simplest, Claude and Gemini both read .md naturally

**What we need:**
- Store: routing decisions, session findings, cross-session learnings
- Both providers read/write
- Semantic retrieval ("what did we learn about auth?")

**Research finding (verbatim):**
> "Recommendation not finalized. User wants more research before committing. Start simple (file-based), migrate to Mem0 or ChromaDB if needed."

**Next step:** Compare with real tasks. Files might be "good enough."

---

### Open Question 2: MCP vs Bash + File I/O

**The challenge (verbatim from PROJECT_CONTEXT.md):**
> "Why is MCP better than `gemini -p 'task' > result.md` followed by a Read?"

**MCP advantages:**
- Native tool integration (Claude sees `delegate_to_gemini` as a tool)
- Structured return (JSON, not raw text)
- Result marshalling into conversation

**Bash + files advantages:**
- Zero infrastructure
- Works today
- No 25K output limit, no timeout issues
- More visible in transcript

**What MCP delegation actually looks like (from Claude's perspective):**

```
User: "Analyze this large codebase for security vulnerabilities"

Claude's reasoning:
  "This task involves 180K tokens of code. My context window is 200K.
   I should delegate the analysis to Gemini (1M context).
   I'll use the delegate_to_gemini tool."

Claude's tool call:
  delegate_to_gemini(
    task: "Analyze auth.js, routes.js, database.js for security vulnerabilities",
    context: "Express API. Focus on OWASP top 10.",
    files: ["auth.js", "routes.js", "database.js"],
    output_format: "structured_report"
  )

  [MCP server receives call]
  [Spawns: gemini -p "Analyze..." --output-format stream-json]
  [Returns structured JSON to Claude]

Claude sees:
  {
    "status": "success",
    "findings": [
      {"severity": "HIGH", "file": "auth.js:42", "issue": "Missing rate limiting"},
      {"severity": "MEDIUM", "file": "database.js:78", "issue": "SQL injection risk"}
    ],
    "tokens_used": 12000,
    "provider": "gemini"
  }
```

**Compare to Bash + files equivalent:**
```
Claude runs: gemini -p "Analyze auth.js..." > /tmp/security_report.md
Claude reads: /tmp/security_report.md
Claude parses: Unstructured markdown
```

**Key MCP constraints to design around:**
- 25K token output limit (large analyses get truncated)
- 2-minute timeout (long Gemini tasks may timeout)
- Context in params (everything Gemini needs goes in tool parameters — expensive if large)

**Research finding (verbatim):**
> "The unexplored alternative: Could hooks alone work? A hook could spawn Gemini, write results to a file, and CLAUDE.md could instruct Claude to check `.harness/results/`. Clunky but no MCP server needed. This hasn't been investigated."

**Next step:** Prototype both. See which feels better in practice.

---

### Open Question 3: Zero-Code Baseline Test

**The critical gap (verbatim from Hidden Assumptions research):**
> "Zero-code baseline NEVER TESTED. Could invalidate entire project."

**What to test:**
- Install PAL MCP + CCProxy + CodexBar
- Write CLAUDE.md delegation rules
- Use for 1 week of real work
- Document what works and what breaks

**Why this matters (verbatim from PROJECT_CONTEXT.md):**
> "If the zero-code baseline is 'good enough,' the user saves months of work. If it's insufficient, the user will know EXACTLY what's missing because they'll have felt the gaps in their daily workflow."

---

## Part 3.5: The Build vs Adopt Line

Not everything needs building. The harness adds value ONLY where existing tools fall short.

### What's Already Solved (Adopt, Don't Build)

| Capability | Existing Tool | Status |
|------------|---------------|--------|
| MCP delegation to other models | PAL MCP | Production-ready |
| Rate limit monitoring | CodexBar | Production-ready |
| Shared memory / context persistence | Mem0 or file-based | Production-ready |
| Visual dashboard / observability | Langfuse or custom | Production-ready |

### What's Genuinely Unsolved (Harness Must Address)

| Capability | Why Existing Tools Don't Solve It |
|------------|-----------------------------------|
| Semantic task-type routing | PAL delegates but doesn't decide WHICH model. User still chooses. |
| Cross-model critique orchestration | No tool coordinates Claude→Gemini critique→Claude revision. |
| Proactive rate limit prediction | CodexBar shows current state, doesn't predict. |
| Seamless switching while keeping Claude interactive | Current tools require explicit model switching. |
| Failure recovery and checkpointing | No tool handles "Gemini timed out, resume from checkpoint." |

### Decision Framework

1. Does an existing tool do this? → Adopt it
2. Does this fill a gap in the unsolved list? → Build it
3. Neither? → Reject it (scope creep)

---

## Part 4: Build Order (Proposed)

**Note:** This build order assumes tool selections that are still under investigation. It represents a proposed sequence, not a committed plan. Phase 0 (select tools, test zero-code baseline) is implicit.

| Phase | What | Why First | Estimated Complexity |
|-------|------|-----------|---------------------|
| 1 | Semantic router (RouteLLM or Aurelio) | Proves routing works without LLM overhead | 2-3 days |
| 2 | Rate limit awareness layer | Proves proactive prediction adds value | 1-2 days |
| 3 | Shared memory (files → Mem0) | Proves cross-session context works | 2-3 days |
| 4 | MCP server wrapping all three | Unified experience for Claude | 1 week |
| 5 | Visual dashboard | When you want to see machinery | 1 week |

**Phase 1 is the smallest testable unit:** Can we classify tasks accurately without LLM calls? If yes, the whole system becomes viable.

---

## Part 5: The Hidden Assumptions (Adversarial Analysis)

The "Hidden Assumptions" research agent surfaced 10 critical challenges. Every assumption below is something the project is implicitly betting on — and none have been validated.

### The 10 Assumptions

| # | Assumption You're Making | The Challenge |
|---|--------------------------|---------------|
| 1 | "40% gap is worth building for" | But you've never tested if 55-60% baseline actually solves your friction |
| 2 | "FSM routing without LLM calls" | You've never designed it or tested accuracy. What if Claude should just decide in-band? |
| 3 | "MCP server is the right mechanism" | Bash + files works today, zero infrastructure. Why is MCP better? |
| 4 | "Harness should be invisible" | But you also want a dashboard showing routing decisions. These contradict. |
| 5 | "Claude always orchestrates" | What if Gemini should lead when Claude is exhausted? You have asymmetric costs. |
| 6 | "Cross-model critique for complex tasks" | How do you define "complex"? The trigger algorithm is undefined. |
| 7 | "Need a memory system (Mem0, etc.)" | What should it store? How does it inform routing? Criteria undefined. |
| 8 | "Designing for yourself is fine" | N=1 user research. Other power users might have different friction. |
| 9 | "Build from scratch" | What if you extend PAL MCP + CCProxy instead of new architecture? |
| 10 | "Problem is architectural" | What if it's epistemological or experiential? |

### The Deepest Challenge

> "You're designing architecture before validating that the problem the architecture solves is actually the problem you're experiencing."

Assumptions 1, 8, and 10 compound into a meta-problem: you might be building the wrong solution to the wrong problem for the wrong audience. Not because the research is bad — but because the research is answering "what's technically possible" before answering "what actually hurts."

### Recommended Validation Steps

Before committing to implementation:

1. **Test zero-code baseline for 1 week** with real work (PAL MCP + CCProxy + CodexBar)
2. **Map ACTUAL friction points** — not hypothetical ones, but what specifically went wrong each day
3. **Interview 5-10 other power users** — does N>1 confirm the same pain?
4. **THEN design** — with evidence of what the baseline couldn't solve

### Impact Assessment

| Impact Level | Assumptions | What Happens If Wrong |
|-------------|-------------|----------------------|
| **BLOCKER** | #1, #2, #3, #5, #7 | Project direction changes fundamentally |
| **DESIGN** | #4, #6 | Architecture needs rework but project survives |
| **VALIDATION** | #8, #9 | Scope or approach shifts |
| **META** | #10 | Entire framing questioned |

---

## Part 6: Document Map

| Order | File | Purpose | Status |
|-------|------|---------|--------|
| 1 | **START_HERE.md** | Start here. Current state. | NEW |
| 2 | ARCHITECTURE.md | Technical decisions, open questions | CURRENT |
| 3 | RESEARCH.md | Evidence base (90+ sources) | CURRENT |
| 4 | PROJECT_CONTEXT.md | Meta-layer, confidence registry | CURRENT |
| 5 | investigation-report.html | Visual competitive landscape | CURRENT |

**Archived (docs/archive/):**
- HANDOFF.md — Historical. Parts 1,5 extracted.
- SESSION_2_SYNTHESIS.md — Merged into this document.

---

## Part 7: Session Log

### Session 3 — January 29, 2026

**Duration:** ~3 hours

**What happened:**
- Launched 5 parallel research agents investigating task decomposition, communication, routing, critique, assumptions
- Deep-dived each dimension with web search and codebase analysis
- Surfaced 10 hidden assumptions challenging the approach
- Clarified user friction: all 4 problems compound each other
- Resolved visibility contradiction: silent CLI + optional dashboard

**Key investigation lanes refined:**
- Routing: Pre-trained semantic routers (investigating RouteLLM, Aurelio)
- Communication: Hybrid hierarchical + mesh (still evolving — exploring direct messaging)
- Critique: Veto pattern, user-triggered first
- Visibility: Silent CLI + opt-in dashboard

**What moved forward:**
- Memory system selection needs comparison
- MCP vs Bash+files needs prototype
- Zero-code baseline test still not done

**Next session should:**
- Consider zero-code baseline test first
- OR proceed to Phase 1: semantic router integration

---

## Part 8: For Future Sessions

### Quick Start (30 seconds)

1. **Read this file** — You're here
2. **Read ARCHITECTURE.md** — Technical decisions
3. **Ask user: "What do you want to focus on today?"**

### What NOT to Do

- Edit files without explaining first
- Write code unless asked
- Treat ARCHITECTURE.md as settled
- Skip the zero-code baseline test
- Rush to conclusions

### User's Values (from PROJECT_CONTEXT.md)

> "Thoroughness and exhaustiveness... Intellectual honesty — 'don't speak out of your ass'... Pushback when warranted — the user wants to be challenged, not agreed with"

---

*This document synthesizes Sessions 1-3. It is the primary entry point for new sessions. Challenge everything. The user wants depth, honesty, and independent thinking.*
