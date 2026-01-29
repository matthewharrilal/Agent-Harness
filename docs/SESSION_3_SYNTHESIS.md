# Session 3 Synthesis: The Architecture Takes Shape

**Date:** January 29, 2026
**Status:** Implementation Ready
**Entry Point:** This is where new sessions should start
**Session Duration:** ~3 hours of deep research with 5 parallel agents

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
| Inter-Agent Communication | How do subtasks talk? | Hub-and-spoke + shared state (Blackboard pattern) |
| LLM Routing | Can we route without training a model? | RouteLLM/Aurelio work out-of-box at 90-95% accuracy |
| Adversarial/Council | How does Team of Rivals fit? | Veto pattern, user-triggered, not consensus |
| Hidden Assumptions | What are we subconsciously committing to? | 10 critical challenges surfaced |

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

### Decision 3: Inter-Agent Communication — Hub-and-Spoke + Shared State (Confidence: 5 — LOCKED)

**What:** Orchestrator controls flow. Subtasks share context via shared memory, not direct messaging.

**The user's insight (validated by research):**
> "What's the connective tissue? How do parallel subtasks communicate when distributed across models?"

**The research validation (verbatim from SESSION_2_SYNTHESIS.md):**
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

**The architecture:**
```
Orchestrator spawns Task A, Task B, Task C
  │
  ├── Task A finishes → writes to SHARED_STATE.md
  │
  ├── Task B reads SHARED_STATE.md → sees A's findings directly
  │   (no orchestrator relay needed)
  │
  └── Orchestrator monitors, handles failures, synthesizes final
```

**What this means for implementation:**
- Need a shared state layer (files, Mem0, or sqlite-vec)
- Both Claude and Gemini must be able to read/write it
- Orchestrator freed from context relay — focuses on control flow

---

### Decision 4: Semantic Routing Without Training (Confidence: 4 — HIGH)

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

### Decision 6: Visibility Model — Silent CLI + Dashboard (Confidence: 4 — HIGH)

**What:** Invisible in CLI experience, visible in dashboard when you want it.

**The user's clarification (verbatim from ARCHITECTURE.md):**
> "I want to type `claude` normally. Not headless mode. The harness should be baked into my Claude experience."

**The "invisible but observable" resolution:**
> "The harness is not a CLI wrapper. It is not a separate command you type. It is an MCP server + hooks system that runs inside your normal Claude Code experience. You type `claude` as you always do. The harness is baked in -- invisible, ambient, always present."

**Two interfaces:**

1. **CLI (Silent):**
   - User types `claude` normally
   - Harness works invisibly via MCP server + hooks
   - No indication of routing unless you ask

2. **Dashboard (Optional visibility):**
   - CodexBar-style menu bar widget: `Claude: 76% │ Gemini: 22%`
   - Conductor Build-style web dashboard at localhost:3200
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

**The challenge (verbatim from SESSION_BRIDGE.md):**
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

**Why this matters (verbatim from SESSION_BRIDGE.md):**
> "If the zero-code baseline is 'good enough,' the user saves months of work. If it's insufficient, the user will know EXACTLY what's missing because they'll have felt the gaps in their daily workflow."

---

## Part 4: Build Order (Proposed)

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

The "Hidden Assumptions" research agent surfaced 10 critical challenges:

### Challenge 1: Zero-Code Baseline Never Tested
**Impact:** BLOCKER
> "If 'good enough,' project becomes documentation, not engineering"

The 40% gap calculation assumes PAL+CCProxy+CodexBar doesn't solve the friction. **But we've never actually used it.**

### Challenge 2: FSM Routing Keeps Appearing "Unsolved"
**Impact:** BLOCKER
> "No algorithm for task classification. Might need Claude to decide routing anyway"

We say "FSM routing, not LLM routing" but have never designed the state machine or tested its accuracy.

### Challenge 3: MCP vs Alternatives Not Compared
**Impact:** BLOCKER
> "Bash + file I/O question unanswered. Architecture could simplify dramatically"

### Challenge 4: "Invisible" and "Observable" Contradict
**Impact:** DESIGN
**Resolution:** Silent CLI + optional dashboard. But this needs explicit design.

### Challenge 5: Claude Always Orchestrates (Even When Exhausted?)
**Impact:** BLOCKER
> "What happens when ONLY Claude is exhausted and Gemini has capacity? This is the most common real-world scenario."

Current design only addresses "both exhausted → notify and stop."

### Challenge 6: Cross-Model Critique Trigger Undefined
**Impact:** FEATURE
> "Cross-model critique doubles cost and latency. Is 2x cost worth it when Claude alone is already 'good enough' for most work?"

"Complex tasks" — but how do we classify complexity without LLM call?

### Challenge 7: Memory System Criteria Undefined
**Impact:** BLOCKER
> "Deep research done on Mem0. Others need equal depth."

What are we storing? How does memory inform routing? Criteria never specified.

### Challenge 8: N=1 User Research
**Impact:** VALIDATION
> "N=1 user research (only themselves). No shipped artifact in this space. No community feedback loop."

Designing for yourself. Other power users might have different friction.

### Challenge 9: Building vs Extending Existing Tools
**Impact:** ARCHITECTURE
> "PAL provides tools but doesn't orchestrate. But this answer needs stress-testing — maybe 'just tools' is enough."

What if we extend PAL MCP + CCProxy instead of building new architecture?

### Challenge 10: Problem Might Not Be Architectural
**Impact:** META
> "What am I actually trying to change about my work? Not 'what system would be ideal,' but 'what pain point am I trying to remove?'"

Maybe it's epistemological (don't know when to use which) or experiential (hate context-switching).

---

## Part 6: Document Map

| Order | File | Purpose | Status |
|-------|------|---------|--------|
| 1 | **SESSION_3_SYNTHESIS.md** | Start here. Current state. | NEW |
| 2 | ARCHITECTURE.md | Technical decisions, open questions | CURRENT |
| 3 | RESEARCH.md | Evidence base (90+ sources) | CURRENT |
| 4 | SESSION_BRIDGE.md | Meta-layer, confidence registry | CURRENT |
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

**Key decisions locked:**
- Routing: RouteLLM or Aurelio (no training, 90-95% accuracy)
- Communication: Hub-and-spoke + shared state
- Critique: Veto pattern, user-triggered first
- Visibility: Silent CLI + dashboard when wanted

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

### User's Values (from SESSION_BRIDGE.md)

> "Thoroughness and exhaustiveness... Intellectual honesty — 'don't speak out of your ass'... Pushback when warranted — the user wants to be challenged, not agreed with"

---

*This document synthesizes Sessions 1-3. It is the primary entry point for new sessions. Challenge everything. The user wants depth, honesty, and independent thinking.*
