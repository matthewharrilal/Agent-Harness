# Agent Harness

A hybrid Claude + Gemini orchestration layer that makes two AI providers feel like one unified system.

## The Problem

Claude Max 20x ($200/mo) gives substantial inference capacity — but heavy users hit 85% of weekly rate limits in 1-2 days. When that happens, work stops. Getting a second Claude account doesn't help: a [device-level keychain bug](https://github.com/anthropics/claude-code/issues/19456) makes dual-Claude broken on one machine (6+ GitHub issues, unfixed as of January 2026).

**But the problem is actually four compounding frictions:**

1. **Rate limits force me to stop** — Literal work stoppage at 429
2. **I don't know which model to use** — Waste tokens on the wrong model
3. **Context-switching loses continuity** — Switching providers loses reasoning state
4. **I re-explain things constantly** — Each session starts cold

These compound: waste tokens → hit limits faster → switch providers → lose context → re-explain → waste more tokens.

## The Solution (Architecture Locked)

Combine Claude Code with Gemini CLI into a single invisible middleware layer. The user types `claude` normally. Behind the scenes:

- **Semantic task routing** — Route by WHAT the task is using pre-trained routers (RouteLLM/Aurelio), not heuristics. 90-95% accuracy, no custom training required.
- **Shared memory via Blackboard pattern** — Both providers read/write to shared state. Agents don't message each other; they read/write to common files or memory store.
- **Hub-and-spoke orchestration** — Claude always orchestrates (fixed). Gemini is a dynamic worker (spoke). All major frameworks (LangGraph, CrewAI, ADK, Swarm) converge on this pattern.
- **Veto-based critique** — Based on [arxiv 2601.14351](https://arxiv.org/abs/2601.14351): single-agent self-review degrades accuracy (<60%), cross-model veto critique achieves 90%. Critic rejects or approves; no consensus voting.
- **Silent CLI + optional dashboard** — Invisible in terminal, visible when you want it (CodexBar-style widget or localhost:3200).

Claude always orchestrates. Gemini is the specialist. The harness is the sous chef that coordinates both kitchens.

## Current Status: Implementation Ready (Session 3 Complete)

**No code has been written yet.** However, 3 sessions of deep research (24+ agents total) have produced:

- **6 architectural decisions LOCKED** (confidence level 5)
- **4 decisions at HIGH confidence** (level 4)
- **10 hidden assumptions surfaced** and documented
- **Proposed build order** for phased implementation

### What's Decided (Don't Re-Litigate)

| Decision | Confidence | Summary |
|----------|:----------:|---------|
| Hybrid Claude+Gemini | LOCKED (5) | Device keychain bug makes dual-Claude broken. $200-220/mo vs $400/mo. |
| Claude as orchestrator | LOCKED (5) | All production frameworks use fixed orchestrator. No dynamic rotation. |
| Hub-and-spoke + shared state | LOCKED (5) | Blackboard pattern. AgentMail was a red herring (for humans, not agents). |
| Veto pattern for critique | LOCKED (5) | arxiv paper: veto > consensus. Critic rejects, doesn't negotiate. |
| Semantic routing (RouteLLM/Aurelio) | HIGH (4) | Pre-trained routers work without custom training. 90-95% accuracy. |
| Silent CLI + dashboard | HIGH (4) | Invisible in terminal, observable via menu bar widget when wanted. |

### What's Still Open

| Question | Why It Matters |
|----------|----------------|
| **Memory system** | Mem0 vs ChromaDB vs sqlite-vec vs plain files. Wrong choice wastes months. |
| **MCP vs Bash + file I/O** | Why is MCP better than `gemini -p 'task' > result.md`? Not yet prototyped. |
| **Zero-code baseline test** | PAL MCP + CCProxy + CodexBar covers ~55-60%. Is that enough? NEVER TESTED. |

### The 10 Hidden Assumptions (Adversarial Analysis)

Session 3 surfaced critical challenges to the approach:

1. **Zero-code baseline never tested** — Could invalidate entire project
2. **FSM routing unsolved** — No algorithm for task classification without LLM
3. **MCP vs alternatives not compared** — Bash + files might be simpler and better
4. **"Invisible" and "observable" contradict** — Resolved: silent CLI + optional dashboard
5. **Claude orchestrates even when exhausted?** — Single-provider exhaustion undesigned
6. **Critique trigger undefined** — When exactly is 2x cost justified?
7. **Memory criteria undefined** — What to store, how long, when to query
8. **N=1 user research** — Only the author's workflow validated
9. **Build vs extend unclear** — Why not just extend PAL MCP?
10. **The 40% gap may not exist** — Maybe zero-code baseline is "good enough"

## The Honest Competitive Landscape

End-of-session research (30+ tools evaluated) revealed existing tools cover **~55-60%** of the vision:

| Tool | What It Does | Coverage |
|------|-------------|:--------:|
| [CCProxy](https://github.com/starbased-co/ccproxy) | Rule-based routing with Max subscription support | ~65% |
| [claude-code-mux](https://github.com/9j/claude-code-mux) | Rust proxy with OAuth, auto-failover, 18+ providers | ~60% |
| [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) | macOS menu bar proxy with real-time quota tracking | ~60% |
| [PAL MCP](https://github.com/BeehiveInnovations/pal-mcp-server) | Manual delegation from Claude to Gemini | ~45% |
| [hcom](https://github.com/aannoo/hcom) | Hook-based inter-agent messaging | ~35% |

**Best zero-code combo today: PAL MCP + CCProxy + CodexBar = ~55-60%**

### The Genuine Gap (~40%)

What does NOT exist anywhere:

1. **Semantic task-type routing** — Route by what the task IS, not metadata
2. **Persistent cross-provider memory** — Both models read/write across sessions
3. **Proactive rate limit prediction** — Before 429, not after
4. **Cross-model critique orchestration** — Team of Rivals pattern, automated
5. **The integrated invisible experience** — All above working together

## Proposed Build Order

| Phase | What | Why First |
|-------|------|-----------|
| 1 | Semantic router (RouteLLM or Aurelio) | Proves routing works without LLM overhead |
| 2 | Rate limit awareness layer | Proves proactive prediction adds value |
| 3 | Shared memory (files → Mem0) | Proves cross-session context works |
| 4 | MCP server wrapping all three | Unified experience for Claude |
| 5 | Visual dashboard | When you want to see machinery |

**Phase 1 is the smallest testable unit.** If semantic classification works, the whole system becomes viable.

## Documents

| Document | Purpose |
|----------|---------|
| `docs/SESSION_3_SYNTHESIS.md` | **START HERE.** Current state after 5-agent deep research. All locked decisions, open questions, build order. |
| `docs/ARCHITECTURE.md` | Technical decisions. MCP design, 21-item compensation list, gap analysis. Living draft. |
| `docs/SESSION_BRIDGE.md` | Meta-layer with confidence registry and session logs. |
| `docs/RESEARCH.md` | Evidence base (90+ sources). Use as reference. |
| `CLAUDE.md` | Instructions for Claude instances. Ground rules, reading order. |

**Archived:**
- `docs/archive/HANDOFF.md` — Original vision. Dual-Claude architecture superseded.
- `docs/archive/SESSION_2_SYNTHESIS.md` — Merged into SESSION_3_SYNTHESIS.md.

## Research Highlights

### Session 3 Key Findings

- **Semantic routing without training**: RouteLLM (95% of GPT-4 perf at 14% calls) and Aurelio (~90% accuracy, sub-penny per 10K queries) both work out-of-box
- **All frameworks converge on shared state**: LangGraph, CrewAI, ADK, Swarm all use Blackboard pattern — agents read/write shared memory, not direct messaging
- **Veto > consensus**: arxiv 2601.14351 found hierarchical veto more reliable than democratic voting
- **No production system rotates orchestrator**: Every framework uses fixed orchestrator + dynamic workers

### Previous Sessions

- **Device keychain bug**: macOS Keychain namespace collision causes rate limits to bleed between accounts. 6+ GitHub issues, unfixed.
- **Claude Code Swarms**: Hidden feature-flagged multi-agent system (TeammateTool, 13 operations). Claude-only — complements harness, doesn't replace it.
- **21 Gemini gaps identified**: Plan mode (4), subagent/task (4), tool bridging (2), MCP compat (3), config/memory (3), headless integration (3), behavioral parity (2).
- **61 architectural gaps cataloged**: 13 blockers, 38 complicators, 10 nice-to-have.

## Architecture

```
YOU type: claude (normal interactive mode)
  |
  v
CLAUDE CODE starts normally
  |
  ├── Reads .mcp.json → spawns "agent-harness" MCP server
  │   Tools: delegate_to_gemini, shared_memory_read/write,
  │          check_rate_limits, route_task
  │
  ├── Reads CLAUDE.md → behavioral instructions for delegation
  │
  ├── Semantic Router (RouteLLM or Aurelio)
  │   Classifies task → routes to Claude or Gemini
  │
  ├── Shared State (Blackboard pattern)
  │   Both providers read/write to common memory
  │
  ├── Hooks → event capture to dashboard
  │
  └── Dashboard server (background process)
      CodexBar-style menu bar widget or localhost:3200
```

## Budget

$200 (Claude Max 20x) + $0-20 (Gemini free/AI Pro) = **$200-220/mo**

## Inspiration

- [@omarsar0's custom agent orchestrator](https://x.com/omarsar0) — "a control center for my agents"
- [PAL MCP Server](https://github.com/BeehiveInnovations/pal-mcp-server) (10K+ stars) — provider abstraction layer
- [CodexBar](https://github.com/steipete/CodexBar) — macOS menu bar rate limit widget
- [CCProxy](https://github.com/starbased-co/ccproxy) — rule-based routing with subscription support
- [arxiv 2601.14351](https://arxiv.org/abs/2601.14351) — "If You Want Coherence, Orchestrate a Team of Rivals"

---

*Last updated: January 29, 2026 (Session 3)*
