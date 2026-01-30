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

---

## Architecture (5-Layer Vision)

Here's how the whole thing looks structurally, layer by layer, then as an integrated system.

> **Canonical source:** [`docs/ARCHITECTURE_VISION.md`](docs/ARCHITECTURE_VISION.md)

### The 5 Layers (Individual)

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: SEMANTIC ROUTER                                       │
│  Status: INVESTIGATION LANE — no tool chosen                    │
│                                                                  │
│  Input:  raw user prompt ("refactor auth module to use JWT")     │
│  Output: task classification + confidence score                  │
│                                                                  │
│  ┌─────────────────┐     ┌──────────────────────┐               │
│  │   RouteLLM      │ OR  │   Aurelio Semantic    │               │
│  │                 │     │   Router              │               │
│  │ Pre-trained on  │     │                      │               │
│  │ strong vs weak  │     │ Embedding similarity  │               │
│  │ model pairs     │     │ against user-defined  │               │
│  │                 │     │ route utterances      │               │
│  │ ~50ms, 90-95%   │     │ ~100ms, 70-85%        │               │
│  │ No training     │     │ No training           │               │
│  └────────┬────────┘     └──────────┬───────────┘               │
│           │                         │                            │
│           └────────────┬────────────┘                            │
│                        ▼                                         │
│           { type: "code", confidence: 0.91 }                     │
│                                                                  │
│  UNKNOWN: Neither tested on Claude+Gemini pair.                  │
│  UNKNOWN: How to handle ambiguous tasks (code AND research).     │
│  DEPENDENCY: Needs memory (Layer 3) for historical routing data. │
└─────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2: STATE-AWARE DECISION ENGINE                           │
│  Status: CONCEPTUAL — no implementation                         │
│                                                                  │
│  THE INTEGRATION POINT — this is where "one system" lives       │
│                                                                  │
│  Inputs (4 signals):                                             │
│  ┌──────────────┐ ┌──────────────┐ ┌────────────┐ ┌──────────┐ │
│  │ Route signal  │ │ Rate limits  │ │ Memory     │ │ Subtask  │ │
│  │ from Layer 1  │ │ both provs   │ │ from L3    │ │ context  │ │
│  │              │ │              │ │            │ │          │ │
│  │ "code" 0.91  │ │ Claude: 42%  │ │ "last time │ │ "Task A  │ │
│  │              │ │ Gemini: 88%  │ │  Claude did │ │  found 3 │ │
│  │              │ │              │ │  this in 1  │ │  issues" │ │
│  │              │ │              │ │  pass"      │ │          │ │
│  └──────┬───────┘ └──────┬───────┘ └─────┬──────┘ └────┬─────┘ │
│         │                │               │              │       │
│         └────────────────┴───────┬───────┴──────────────┘       │
│                                  ▼                               │
│                    ┌──────────────────────┐                      │
│                    │   ROUTING DECISION   │                      │
│                    │                      │                      │
│                    │  → Claude (handle)   │                      │
│                    │  → Gemini (delegate) │                      │
│                    │  → Split (decompose) │                      │
│                    │  → Critique (both)   │                      │
│                    └──────────────────────┘                      │
│                                                                  │
│  THIS LAYER DOES NOT EXIST YET. It's the hardest part.          │
│  How these 4 signals combine is undefined.                       │
│  Weighting? Priority order? Overrides? All open.                │
└─────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3: CROSS-PROVIDER MEMORY                                 │
│  Status: INVESTIGATION LANE — no tool chosen                    │
│  Role: ARCHITECTURALLY CENTRAL (double duty)                    │
│                                                                  │
│  Duty 1: Cross-subtask context                                   │
│  ┌──────────┐    writes    ┌──────────┐    reads    ┌─────────┐ │
│  │ Task A   │─────────────►│ MEMORY   │◄────────────│ Task B  │ │
│  │ (Claude) │              │          │             │(Gemini) │ │
│  └──────────┘              └──────────┘             └─────────┘ │
│  "Task A found 3 security issues" → Task B sees this directly   │
│                                                                  │
│  Duty 2: Cross-session persistence                               │
│  ┌──────────┐    writes    ┌──────────┐    reads    ┌─────────┐ │
│  │ Tuesday  │─────────────►│ MEMORY   │◄────────────│Wednesday│ │
│  │ session  │              │          │             │ session  │ │
│  └──────────┘              └──────────┘             └─────────┘ │
│  "JWT decision: 3 formats, sunset 1 in Q2" persists             │
│                                                                  │
│  Candidates:                                                     │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐│
│  │ Mem0           │  │ sqlite-vec    │  │ Files (AGENTS.md)     ││
│  │ Semantic       │  │ Lightweight   │  │ Simplest              ││
│  │ extraction +   │  │ vector search │  │ Both CLIs read .md    ││
│  │ vector store   │  │ + SQLite      │  │ natively              ││
│  │ Needs LLM      │  │ BYO embed     │  │ No semantic search    ││
│  │ 150-500MB RAM  │  │ 30-50MB RAM   │  │ 0 MB overhead         ││
│  └───────────────┘  └───────────────┘  └───────────────────────┘│
│                                                                  │
│  BLOCKING: Router (L1) needs memory for historical accuracy.     │
│  BLOCKING: Decision Engine (L2) needs memory for context.        │
│  This is effectively Layer 0 — everything depends on it.         │
└─────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4: VISUAL INTERFACE                                       │
│  Status: CONCEPTUAL — not designed                               │
│                                                                  │
│  Two surfaces, one data source:                                  │
│                                                                  │
│  ┌─ Ambient (always visible) ─────────────────────────────────┐ │
│  │  macOS Menu Bar Widget (CodexBar-inspired)                  │ │
│  │  ┌───────────────────────────────────────────┐              │ │
│  │  │ ● Claude: 42% │ Gemini: 88% │ $0.50 today│              │ │
│  │  └───────────────────────────────────────────┘              │ │
│  │  Zero interaction cost. Glance and keep working.            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─ Deep analysis (on demand) ────────────────────────────────┐ │
│  │  localhost:3200 (Conductor Build-inspired)                  │ │
│  │  ┌──────────┬──────────┬──────────┬──────────┬───────────┐ │ │
│  │  │ Activity │ Routing  │  Costs   │  Rate    │   Plans   │ │ │
│  │  │  Feed    │ Analytics│ Tracker  │  Limits  │  Approval │ │ │
│  │  └──────────┴──────────┴──────────┴──────────┴───────────┘ │ │
│  │  Open when you want to investigate. Never required.         │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Data pipeline: Hooks → HTTP POST → SQLite → SSE → UI           │
│  DEPENDS ON: Layer 5 (hooks for event capture)                   │
│  DEPENDS ON: MCP decision (Level 3) for what events exist        │
└─────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────┐
│  LAYER 5: CLI EXPERIENCE                                         │
│  Status: INVESTIGATION — three-tier reliability model emerging   │
│                                                                  │
│  The user types: claude                                          │
│  Everything below is invisible.                                  │
│                                                                  │
│  THREE-TIER RELIABILITY MODEL (Session 4 insight):               │
│                                                                  │
│  ┌─ Tier 1: CLAUDE.md (Soft Steering, ~60-80% reliable) ────┐  │
│  │ "When task >100K tokens, use delegate_to_gemini"          │  │
│  │ "When researching broad topics, consider Gemini"          │  │
│  │ Always read by Claude. Claude decides whether to follow.  │  │
│  │ Zero runtime cost beyond context window tokens.           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ Tier 2: Hooks (Hard Enforcement, 100% reliable) ────────┐  │
│  │ UserPromptSubmit: inject rate state + memory context       │  │
│  │ PreToolUse:       block tool calls if rate-limited         │  │
│  │ PostToolUse:      track usage, update counters             │  │
│  │ Stop:             persist session summary to memory         │  │
│  │                                                            │  │
│  │ GUARANTEED — fires on every lifecycle event.               │  │
│  │ Zero LLM cost. This is the harness's critical path.       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ Tier 3: MCP Tools (Capability Extension, ~50-88%) ──────┐  │
│  │ delegate_to_gemini    (active delegation)                  │  │
│  │ request_critique      (cross-model veto)                   │  │
│  │ query_memory          (deep semantic lookup)               │  │
│  │ check_rate_status     (explicit rate query)                │  │
│  │                                                            │  │
│  │ OPTIONAL — Claude chooses when to invoke.                  │  │
│  │ System doesn't break if these aren't called.              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  KEY INSIGHT: Hooks guarantee the floor (critical functions      │
│  always execute). MCP tools raise the ceiling (optional          │
│  enhancements). CLAUDE.md provides behavioral context.           │
│  The system works even if Claude never calls an MCP tool.        │
│                                                                  │
│  HOOKS + MCP SPLIT:                                              │
│  Function          │ Hook (guaranteed)    │ MCP (optional)       │
│  Rate awareness    │ UserPromptSubmit     │ check_rate_status    │
│  Memory injection  │ UserPromptSubmit     │ query_memory         │
│  Rate enforcement  │ PreToolUse: block    │ —                    │
│  Usage tracking    │ PostToolUse          │ —                    │
│  Delegation        │ —                    │ delegate_to_gemini   │
│  Critique          │ —                    │ request_critique     │
│  Persistence       │ Stop: write memory   │ —                    │
└─────────────────────────────────────────────────────────────────┘
```

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

### The Dependency Reality

```
The layers aren't a stack — they're a dependency graph:

                    ┌──────────────────────┐
                    │  Layer 3: MEMORY     │ ◄── EVERYTHING DEPENDS ON THIS
                    └──────────┬───────────┘
                               │
                  ┌────────────┼────────────┐
                  │            │            │
                  ▼            ▼            ▼
           ┌──────────┐ ┌──────────┐ ┌──────────┐
           │ Layer 1   │ │ Layer 2   │ │ Layer 5   │
           │ ROUTER    │ │ DECISION  │ │ CLI       │
           │           │ │ ENGINE    │ │           │
           │ needs     │ │ needs     │ │ needs     │
           │ history   │ │ all of    │ │ mechanism │
           │ from mem  │ │ the above │ │ decision  │
           └─────┬─────┘ └────┬──────┘ └─────┬─────┘
                 │            │              │
                 └────────────┼──────────────┘
                              │
                              ▼
                       ┌──────────┐
                       │ Layer 4   │ ◄── DEPENDS ON ALL OTHERS
                       │ VISUAL    │     (observes everything)
                       └──────────┘

  CIRCULAR DEPENDENCY:
  - Router needs memory (past routing outcomes)
  - Memory needs router (to know what to store)
  - Decision engine needs both
  - All three need the CLI mechanism (Layer 5)
    to actually execute anything
```

### What's Locked vs Open

```
██████  = LOCKED (won't change)
░░░░░░  = INVESTIGATION LANE (direction set, no tool chosen)
          OPEN (undefined)

LOCKED:
  ██████ Claude+Gemini hybrid (not dual-Claude)
  ██████ Claude always orchestrates
  ██████ Gemini free/$20, not $250 Ultra
  ██████ Invisible experience (type `claude` normally)
  ██████ Critique = veto pattern, user-triggered
  ██████ Both exhausted = notify and stop

INVESTIGATION LANES:
  ░░░░░░ Semantic routing (RouteLLM vs Aurelio — neither tested)
  ░░░░░░ Communication (hybrid mesh — exploring direct messaging)
  ░░░░░░ Visibility (silent CLI + dashboard — not designed)
  ░░░░░░ Memory tool (Mem0 vs sqlite-vec vs files)
  ░░░░░░ CLI delivery (three-tier: hooks + MCP + CLAUDE.md emerging)

OPEN:
         Decision Engine logic (how 4 signals combine)
         Memory schema (what to store, retention, queries)
         Dashboard design (panels, tech, interactions)
         Zero-code baseline (never tested)
         MVP scope (which of 21 compensation items first)
         Communication mechanism (shared files vs messaging platform)
```

---

## Current Status: Investigation Phase (Session 3 Complete)

**No code has been written yet.** 3 sessions of deep research (24+ agents total) have produced:

- **7 decisions LOCKED** (confidence level 5 — see status map above)
- **5 investigation lanes refined** (confidence level 4 — direction established, no tool choices made)
- **5-layer structural vision** established as framework for investigation
- **10 hidden assumptions surfaced** and documented in [`docs/START_HERE.md`](docs/START_HERE.md)
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

### What's Still Open

| Question | Why It Matters |
|----------|----------------|
| **Memory system** | Mem0 vs ChromaDB vs sqlite-vec vs plain files. Wrong choice wastes months. |
| **CLI delivery mechanism** | Three-tier model emerging: hooks (guaranteed) + MCP (optional) + CLAUDE.md (behavioral). Hooks handle critical path; MCP tools are enhancements. Could the harness be hooks-only + PAL MCP? |
| **Zero-code baseline test** | PAL MCP + CCProxy + CodexBar covers ~55-60%. Is that enough? NEVER TESTED. |

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
| `docs/INVESTIGATION_QUEUE.md` | 7 primary + 4 auxiliary investigation questions. Dependency graph, wave ordering, status tracking. |
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

## Budget

$200 (Claude Max 20x) + $0-20 (Gemini free/AI Pro) = **$200-220/mo**

## Inspiration

- [@omarsar0's custom agent orchestrator](https://x.com/omarsar0) — "a control center for my agents"
- [PAL MCP Server](https://github.com/BeehiveInnovations/pal-mcp-server) (10K+ stars) — provider abstraction layer
- [CodexBar](https://github.com/steipete/CodexBar) — macOS menu bar rate limit widget
- [CCProxy](https://github.com/starbased-co/ccproxy) — rule-based routing with subscription support
- [arxiv 2601.14351](https://arxiv.org/abs/2601.14351) — "If You Want Coherence, Orchestrate a Team of Rivals"

---

*Last updated: January 30, 2026 (Session 4 — three-tier reliability model, investigation questions, hooks+MCP architecture)*
