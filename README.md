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

## The Solution (Investigation Phase)

Combine Claude Code with Gemini CLI into a single invisible middleware layer. The user types `claude` normally. Behind the scenes:

- **Semantic task routing** — Route by WHAT the task is using pre-trained routers (investigating RouteLLM/Aurelio). No choice made yet; neither tested on Claude+Gemini pair.
- **Cross-provider memory (architecturally central)** — Both providers read/write shared state. Memory serves double duty: cross-subtask context + cross-session persistence. Tool choice (Mem0 / sqlite-vec / files) still open.
- **Hybrid hierarchical + mesh communication** — Claude always orchestrates (hierarchical, locked). Subtasks share context directly (mesh). Exploring direct subagent-to-subagent messaging. Still evolving.
- **Veto-based critique** — Based on [arxiv 2601.14351](https://arxiv.org/abs/2601.14351): single-agent self-review degrades accuracy (<60%), cross-model veto critique achieves 90%. Critic rejects or approves; no consensus voting. (Locked.)
- **Silent CLI + optional dashboard** — Invisible in terminal, visible when you want it (CodexBar-style widget or localhost:3200). Depends on MCP decision (Level 3 confidence).

Claude always orchestrates. Gemini is the specialist. The harness is the sous chef that coordinates both kitchens.

## Current Status: Investigation Phase — Refined Lanes Established (Session 3 Complete)

**No code has been written yet.** 3 sessions of deep research (24+ agents total) have produced:

- **7 decisions LOCKED** (confidence level 5 — see `docs/ARCHITECTURE_VISION.md` for full status map)
- **3 investigation lanes refined** (confidence level 4 — direction established, no tool choices made)
- **5-layer structural vision** established as framework for investigation
- **10 hidden assumptions surfaced** and documented
- **Proposed build order** (contingent on tool selections still under investigation)

### What's Decided (Don't Re-Litigate)

| Decision | Confidence | Summary |
|----------|:----------:|---------|
| Hybrid Claude+Gemini | LOCKED (5) | Device keychain bug makes dual-Claude broken. $200-220/mo vs $400/mo. |
| Claude as orchestrator | LOCKED (5) | All production frameworks use fixed orchestrator. No dynamic rotation. |
| Veto pattern for critique | LOCKED (5) | arxiv paper: veto > consensus. Critic rejects, doesn't negotiate. |
| Invisible experience | LOCKED (5) | User types `claude` normally. Harness baked in via .mcp.json + hooks. |
| Gemini free/AI Pro tier | LOCKED (5) | $0-20/mo budget. NOT Ultra ($250/mo). |
| Both exhausted = notify and stop | LOCKED (5) | No infinite retry loops. Surface the constraint. |
| Critique: user-triggered or complexity-triggered | LOCKED (5) | Not every task gets 2x cost of cross-model review. |
| Communication: hybrid hierarchical + mesh | INVESTIGATION LANE (4) | Evolved from Blackboard. Exploring direct subagent messaging. |
| Semantic routing (RouteLLM/Aurelio) | INVESTIGATION LANE (4) | Pre-trained routers — neither tested on Claude+Gemini pair yet. |
| Silent CLI + dashboard | INVESTIGATION LANE (4) | Depends on MCP decision (Level 3). Visual interface not designed. |
| Memory is architecturally central | INVESTIGATION LANE (4) | Tool choice open (Mem0/sqlite-vec/files). |
| MCP mechanism | INVESTIGATION LANE (3) | "Don't double down on MCP yet." |

### What's Still Open

| Question | Why It Matters |
|----------|----------------|
| **Memory system** | Mem0 vs ChromaDB vs sqlite-vec vs plain files. Wrong choice wastes months. |
| **MCP vs Bash + file I/O** | Why is MCP better than `gemini -p 'task' > result.md`? Not yet prototyped. |
| **Zero-code baseline test** | PAL MCP + CCProxy + CodexBar covers ~55-60%. Is that enough? NEVER TESTED. |

### The 10 Hidden Assumptions (Adversarial Analysis)

Session 3 surfaced critical challenges to the approach:

1. **Zero-code baseline never tested** — Could invalidate entire project
2. **Routing approach under investigation** — RouteLLM/Aurelio identified but neither tested on Claude+Gemini pair
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

## Document Hierarchy

### Entry Point
| Document | Purpose |
|----------|---------|
| `docs/START_HERE.md` | **START HERE.** Locked decisions, investigation lanes, open questions, build order. |

### Architecture
| Document | Purpose |
|----------|---------|
| `docs/ARCHITECTURE_VISION.md` | **Canonical 5-layer architecture.** All diagrams, dependency graph, status map. |
| `docs/ARCHITECTURE.md` | Technical deep-dive. MCP design, compensation list, gap analysis. Living draft. |

### Reference
| Document | Purpose |
|----------|---------|
| `docs/PROJECT_CONTEXT.md` | Confidence registry, session history, conduct guide. |
| `docs/RESEARCH.md` | Evidence base (90+ sources). Competitive landscape, device bug, Gemini gaps. |
| `docs/TOOL_ANALYSIS.md` | Tool-by-tool analysis of 8 competing tools. |

### Illustrative (User Journeys)
| Document | Purpose |
|----------|---------|
| `docs/USER_JOURNEY.md` | Speculative Wednesday morning scenario — with vs without harness. |
| `docs/USER_JOURNEYS.md` | 5 competing tool scenarios — mental models and trade-offs. |
| `docs/CLAUDE_CODE_MUX_USER_JOURNEY.md` | Mid-session failover scenario with claude-code-mux. |

### Meta
| Document | Purpose |
|----------|---------|
| `CLAUDE.md` | Instructions for Claude instances. Ground rules, locked decisions, what's open. |

### Archived
| Document | Purpose |
|----------|---------|
| `docs/archive/HANDOFF.md` | Original vision. Parts 1 & 5 valid; Part 3 superseded. |
| `docs/archive/SESSION_2_SYNTHESIS.md` | Merged into START_HERE.md. |
| `docs/archive/NEXT_SESSION_PROMPT.md` | Superseded by START_HERE.md as entry point. |

## Research Highlights

### Session 3 Key Findings

- **Semantic routing without training**: RouteLLM (95% of GPT-4 perf at 14% calls) and Aurelio (~90% accuracy, sub-penny per 10K queries) both work out-of-box
- **Communication evolved to hybrid hierarchical + mesh**: Session 2's Blackboard pattern evolved — orchestrator controls flow (hierarchical), subtasks share context directly (mesh). Exploring direct subagent-to-subagent messaging.
- **Veto > consensus**: arxiv 2601.14351 found hierarchical veto more reliable than democratic voting
- **No production system rotates orchestrator**: Every framework uses fixed orchestrator + dynamic workers

### Previous Sessions

- **Device keychain bug**: macOS Keychain namespace collision causes rate limits to bleed between accounts. 6+ GitHub issues, unfixed.
- **Claude Code Swarms**: Hidden feature-flagged multi-agent system (TeammateTool, 13 operations). Claude-only — complements harness, doesn't replace it.
- **21 Gemini gaps identified**: Plan mode (4), subagent/task (4), tool bridging (2), MCP compat (3), config/memory (3), headless integration (3), behavioral parity (2).
- **61 architectural gaps cataloged**: 13 blockers, 38 complicators, 10 nice-to-have.

## Architecture (5-Layer Vision)

> **Full architecture diagrams:** See [`docs/ARCHITECTURE_VISION.md`](docs/ARCHITECTURE_VISION.md) for the complete 5-layer breakdown, integrated system flow, dependency graph, and locked/open status map.

```
USER types: claude (normal interactive mode)
  │
  ▼
CLAUDE CODE starts normally
  │
  ├── .mcp.json → spawns harness MCP server (Layer 5: CLI)
  │
  ├── CLAUDE.md → behavioral steering for delegation
  │
  ├── User prompt arrives
  │       │
  │       ▼
  │   Layer 1: SEMANTIC ROUTER
  │   "What kind of task is this?"
  │   (RouteLLM / Aurelio — investigating)
  │       │
  │       ▼
  │   Layer 2: DECISION ENGINE
  │   Combines: task type + rate limits + memory + history
  │   → Routes to Claude or Gemini
  │       │
  │   ┌───┴───┐
  │   ▼       ▼
  │  Claude  Gemini (via MCP delegation)
  │   │       │
  │   └───┬───┘
  │       ▼
  │   Layer 3: CROSS-PROVIDER MEMORY
  │   Stores findings, informs next routing decision
  │   (architecturally central — actual Layer 0)
  │
  ├── Hooks capture events → Layer 4 (dashboard/widget)
  │
  └── Result returned to user in normal conversation
```

**Dependency reality:** Memory (Layer 3) is labeled Layer 3 but is the actual foundation — routing needs memory, memory informs routing. See `docs/ARCHITECTURE_VISION.md` for the full dependency graph.

## Budget

$200 (Claude Max 20x) + $0-20 (Gemini free/AI Pro) = **$200-220/mo**

## Inspiration

- [@omarsar0's custom agent orchestrator](https://x.com/omarsar0) — "a control center for my agents"
- [PAL MCP Server](https://github.com/BeehiveInnovations/pal-mcp-server) (10K+ stars) — provider abstraction layer
- [CodexBar](https://github.com/steipete/CodexBar) — macOS menu bar rate limit widget
- [CCProxy](https://github.com/starbased-co/ccproxy) — rule-based routing with subscription support
- [arxiv 2601.14351](https://arxiv.org/abs/2601.14351) — "If You Want Coherence, Orchestrate a Team of Rivals"

---

*Last updated: January 30, 2026 (Session 4 — documentation audit and restructure)*
