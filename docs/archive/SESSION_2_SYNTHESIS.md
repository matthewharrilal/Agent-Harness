# Agent Harness: Session 2 Synthesis (January 29, 2026)

> **⚠️ ARCHIVED — January 29, 2026**
>
> **This document has been merged into START_HERE.md** (formerly SESSION_3_SYNTHESIS.md)
>
> Session 2 findings are now incorporated into the comprehensive synthesis document.
> All key insights, confidence levels, and open questions have been preserved.
>
> **For current project state, start with:** `/docs/START_HERE.md`

---

**Status:** Critical Reflection Point
**Purpose:** Honest assessment of where we stand after the "sinew" and Conductor research

---

## Part 0: Current State Assessment

### What's NOW More Confident (Validated)

| Decision | Confidence | Why It Hardened |
|----------|------------|-----------------|
| **Hybrid Claude+Gemini, not dual-Claude** | LOCKED (5) | Device bug is real (6+ GitHub issues). Not fixable by us. |
| **Claude as top-level orchestrator** | LOCKED (5) | NeurIPS papers + MAST taxonomy (1600+ traces) confirm fixed orchestrator is right |
| **Gemini Free/Pro, not Ultra** | HIGH (4) | Math is conclusive: $220/mo << $400/mo, Ultra's 33% boost isn't worth 12.5x price |
| **Composed architecture** | HIGH (4) | 7 fork candidates evaluated. None > 50% complete. The gap is real. |
| **The 40% gap exists** | HIGH (4) | 40+ tools surveyed. PAL+CCProxy+CodexBar = 55-60%. The remaining 40% is genuinely novel. |
| **Conductor stacks WITH harness, not against** | NEW (4) | Conductor is Gemini's harness. Our harness sits ABOVE, coordinating both providers. |

### What's NOW Being Challenged (Needs Rethinking)

| Assumption | Challenge | Impact |
|------------|-----------|--------|
| **40% gap worth building** | Zero-code baseline NEVER TESTED | Could invalidate entire project |
| **MCP server is right mechanism** | "Bash + file I/O" question unanswered | Architecture could simplify dramatically |
| **FSM routing doesn't need LLM** | No algorithm for task classification | Might need Claude to decide routing anyway |
| **"Invisible" and "observable" compatible** | Contradictory goals stated | Design needs resolution |
| **Single-provider exhaustion handled** | Only "both exhausted" is designed | Most common scenario unspecified |

### What's STILL Missing (Research Gaps)

| Gap | Why It Would Change Direction |
|-----|-------------------------------|
| **Zero-code baseline test** | If "good enough," project becomes documentation, not engineering |
| **FSM router design** | If impossible without LLM, cost model breaks |
| **Memory system comparison** | Wrong choice wastes months |
| **Bash vs MCP comparison** | Could eliminate MCP complexity entirely |
| **Failure mode playbooks** | 61 gaps identified, 13 blockers, ~3 resolved |

### Key Insight from Today: The "Sinew" Answer

The connective tissue between agents is NOT direct messaging. It's **shared state (Blackboard pattern)**:
- Agents don't talk to each other — they read/write shared memory
- Framework comparison (LangGraph, CrewAI, ADK, Swarm) confirms this
- AgentMail was a red herring (email for humans, not agent-to-agent)
- Conductor proves single-provider harnesses exist — ours adds the cross-provider layer

---

## The Deeper Question: Genuine Domain Understanding

### What the User Is Really Wrestling With

**Not just "what should I build?"** but **"How do I develop the taste to know what's worth building?"**

The user wants to build something **irreplaceable** — where people say "I can't not use this now." The fear is building something redundant that reveals shallow understanding rather than genuine domain expertise.

### What Separates "Genuine Understanding" from "Surface Research"

| Surface Research | Genuine Understanding |
|------------------|----------------------|
| Solves problems *as described* | Solves problems *as experienced* |
| Architecture matches documentation | Architecture shaped by failures |
| Handles theoretical edge cases | Handles edge cases that actually happen |
| Uses standard terminology | Has named concepts others recognize as "finally, a word for that" |
| Builds what would impress a researcher | Builds what makes their own work better |

### The Authentic Signals the User Already Has

1. **Real friction**: 85% rate limits in 1-2 days (not borrowed frustration)
2. **Daily use**: Actually using Claude Code, not theorizing
3. **Deep research**: 19+ agents, 40+ tools surveyed
4. **Technical capability**: Can build, not just ideate

### What Research CANNOT Give

- Intuition about what matters vs what's noise
- Muscle memory for failure modes (haven't built and failed yet)
- Taste (knowing when something feels right)
- Calibration against other power users' reality

### The Non-Obvious Problems Only Power Users Encounter

1. **The Momentum Problem**: Every new session starts cold. The cost isn't tokens — it's rebuilding shared understanding.
2. **The Critique Timing Problem**: Critique is valuable at specific moments, not constantly.
3. **The Partial Success Problem**: 70% correct + 30% wrong is worse than clean failure.
4. **The Trust Calibration Problem**: Verification effort must match actual risk.
5. **The Interruptibility Asymmetry**: Interrupting AI mid-task = "start over" disguised as "pause."

### The "Everyone Thinks X, Actually Y" Insights

| Everyone Thinks | Actually |
|-----------------|----------|
| More capable models = better | Capability without context = sophisticated mistakes |
| Multi-agent = parallel speedup | Coordination overhead often exceeds savings |
| Memory systems help | Wrong memories are worse than no memories |
| Routing saves money | Routing decisions consume context; trivial routing loses money |
| Cross-model critique catches errors | Catches obvious errors; subtle ones need same-model self-critique |

---

## Pivot Options Analysis

### If Not the Full Harness, What Demonstrates Domain Understanding?

**Option 1: The Routing Layer ("TaskRouter")**
- Build the BEST semantic router for multi-model orchestration
- Defensibility: Medium-High (unsolved problem — "no universal optimal router exists")
- Weakness: Hard to classify without LLM overhead

**Option 2: The Memory Layer ("CrossMem") — RECOMMENDED**
- Cross-session, cross-provider persistent memory
- Defensibility: HIGH (Switzerland play — Anthropic/Google compete, neither builds cross-provider)
- Directly serves user's stated need: "Everything else should be shared"
- Novelty is integration, not the memory itself

**Option 3: The Critique Layer ("TeamOfRivals")**
- Automated cross-model critique (60%→90% accuracy)
- Defensibility: Low (2x cost for something reserved for complex tasks)
- Doesn't solve capacity problem — reduces it

**Option 4: The Observability Layer ("RouteViz")**
- Dashboard showing WHY routing decisions were made
- Defensibility: Medium (genuine gap — no tool shows decision rationale)
- Supporting tool, not main event

**Option 5: The Context Layer ("ContextBridge")**
- Context engineering across models/sessions
- Defensibility: Medium ("context engineering IS the bottleneck")
- Sleeper candidate but hard to explain value

---

## User's Clarified Direction (January 29)

### What to Build First

**An MCP server that lives inside Claude Code providing:**
1. Intelligent routing based on task decomposition
2. Rate limit awareness (current state of both providers)
3. The "sinew" / connective tissue between subtasks distributed to different models

This is NOT the full harness. It's the **Control Plane layer** — the brain that makes routing decisions.

### Why Memory AND Routing?

User is interested in both. The distinction:

**Memory** is defensible because it's the "Switzerland play":
- Anthropic and Google compete — neither will build cross-provider memory
- Lower technical risk (memory systems exist; integration is the novelty)
- Directly serves "everything else should be shared"

**Routing** is harder but demonstrates deeper understanding:
- "No universal optimal router exists" — genuinely unsolved
- Shows you understand the non-obvious tradeoffs
- More technically ambitious

**The synthesis:** Build routing-aware memory. The MCP server needs shared state (memory) to make intelligent routing decisions. They're not separate — routing USES memory.

### Success Criteria (User-Defined)

1. **Solves MY friction elegantly** — If I can't stop using it, others will feel the same
2. **Others adopt it** — Stars, users, external validation

This means: Start by solving YOUR problem deeply. Adoption follows from authentic utility.

---

## Intelligent LLM Routing — Deep Research

### The Four Major Routing Paradigms

**1. Classifier-Based (RouteLLM, HybridLLM)**
- Train a model to predict which LLM performs best
- RouteLLM: 95% of GPT-4 performance using only 14% GPT-4 calls
- HybridLLM: 40% fewer calls to large model with no quality drop

**2. Embedding-Based / Semantic (Aurelio Labs, RoRF)**
- Convert queries to vectors, find similar past queries, route based on what worked
- ~90% accuracy, sub-penny cost per 10K queries

**3. Bandit Algorithms (MixLLM, PILOT)**
- Treat model selection as explore/exploit problem
- MixLLM: 97.25% of GPT-4 quality at 24.18% of cost

**4. Cascading (FrugalGPT)**
- Try cheapest model first, escalate if quality insufficient
- Match GPT-4 performance with 98% cost reduction

### Why Routing Is HARD (Unsolved Problems)

1. **Out-of-Distribution Generalization**: Routers trained on chat fail on academic benchmarks
2. **Non-Linear Decision Boundaries**: Linear methods score 0.61-0.66, non-linear scores 0.74
3. **Quality Estimation**: Predicting response quality without seeing the response is hard
4. **Interpretability Gap**: We don't understand WHY routers make decisions
5. **No Universal Optimal Router**: Different routers win on different metrics

### The FSM Routing Question

The harness assumes "route without LLM calls" (FSM routing). But how do you classify "refactor auth module" as code vs research without asking a model?

**Possible approaches:**
- Keyword heuristics ("research", "analyze" → Gemini)
- Token count / context window estimate
- Historical success rates per task type
- Let Claude decide via CLAUDE.md instructions (skip FSM entirely)

**This is an acknowledged GAP needing more research.**

---

## The Layering Model

### The Stack — Where Everything Sits

```
┌─────────────────────────────────────────────────────────────────┐
│ CONTROL PLANE (Who/Why/When)              ← THE HARNESS LIVES HERE
│ ├─ Semantic task-type routing             ← NOBODY does this yet
│ ├─ Cost optimization logic                ← NOBODY does this yet
│ ├─ Cross-model critique orchestration     ← NOBODY does this yet
│ ├─ Proactive rate limit prediction        ← NOBODY does this yet
│ └─ Unified approval policies              ← NOBODY does this yet
├─────────────────────────────────────────────────────────────────┤
│ ROUTING/DISPATCH PLANE                    ← CROWDED (PAL MCP, CCProxy)
│ ├─ Provider selection (which API)
│ ├─ Context serialization for hand-off
│ └─ Concurrent invocation / spawning
├─────────────────────────────────────────────────────────────────┤
│ PERSISTENCE PLANE                         ← SPARSE
│ ├─ Conductor (.md files) — single-provider only
│ └─ Cross-session memory — EMPTY
├─────────────────────────────────────────────────────────────────┤
│ OBSERVATION PLANE                         ← PARTIAL
│ ├─ CodexBar (rate limits)
│ └─ Routing analytics — EMPTY
├─────────────────────────────────────────────────────────────────┤
│ CLI/EXECUTION LAYER
│ ├─ Claude Code + PAL MCP
│ ├─ Gemini CLI + Conductor
│ └─ CCProxy interception
└─────────────────────────────────────────────────────────────────┘
```

### Key Insight: Conductor vs Harness

**Gemini Conductor** sits at the Persistence + Planning layer for a **single provider**:
- Context-as-code (specs, plans in markdown)
- Workflow structure (Spec → Plan → Implement)
- Brownfield project support

**The Harness** sits at the **Control Plane — above Conductor**:
- Decides Claude vs Gemini for each task
- Manages persistent memory across BOTH providers
- Orchestrates cross-model critique
- Predicts rate limit exhaustion

**They don't conflict. They stack.** Conductor could be a component the harness uses when delegating to Gemini.

---

## The 40% Genuine Gap

What NOBODY does yet:
1. **Semantic task-type routing** — Route by WHAT the task is, not metadata
2. **Persistent cross-provider memory** — Both models read/write, spans sessions
3. **Proactive rate limit prediction** — Before 429, not after
4. **Cross-model critique orchestration** — Team of Rivals pattern, automated
5. **The invisible integrated experience** — All above working together seamlessly

---

## The Immersion vs Research Question

### Can Research Substitute for Years of Immersion?

**What research gives:**
- Landscape awareness, technical knowledge, pattern recognition, gap identification

**What research cannot give:**
- Intuition about what matters, muscle memory for failure modes, taste

### The User's Honest Position

**Advantages:**
- Real friction (not imagined)
- Daily use of primary tool
- Serious research investment
- Technical capability to build

**Disadvantages:**
- N=1 user research (only themselves)
- No shipped artifact in this space
- No community feedback loop
- No failed attempts to learn from

**Assessment:** "Informed outsider" stage, not "deep insider." Knows more than 95% of people, but breakthrough builders are in 99th percentile — that last 4% is earned through building and failing.

### Paths to Accelerate Immersion

1. **Ship something ugly in 2 weeks** — Learn more from 3 real users than 2 more months of research
2. **Interview 5-10 power users** — Calibrate intuition against others' reality
3. **Contribute to existing projects** — PAL MCP, CCProxy, etc. to learn constraints
4. **Document publicly and be wrong** — Let community correct blind spots

---

## Next Steps

1. **Build an MCP server** inside Claude Code with:
   - Intelligent routing based on task decomposition
   - Rate limit awareness
   - Shared state ("sinew") between subtasks

2. **Research FSM routing further** — How to classify without LLM calls?

3. **Test zero-code baseline** — Install PAL+CCProxy+CodexBar, use for a week, document what breaks

4. **Compare memory systems** — Mem0 vs ChromaDB vs sqlite-vec vs files

---

## Session Log

**Session 2 — January 29, 2026**

**What happened:**
- Clarified Conductor's role (single-provider harness, stacks with our harness)
- Researched "sinew" / connective tissue → answer is Blackboard pattern (shared state, not messaging)
- Deep-dived intelligent LLM routing (4 paradigms, unsolved problems)
- Explored "genuine domain understanding" vs surface research
- User clarified direction: MCP server for routing + memory, solves own friction, others adopt

**Key insights:**
- Conductor IS a harness for Gemini — ours sits above
- AgentMail was a red herring (email for humans)
- FSM routing without LLM is an open gap
- Memory and routing are not separate — routing USES memory

**User's success criteria:**
1. Solves MY friction elegantly
2. Others adopt it
