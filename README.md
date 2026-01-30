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

> **Full layer-by-layer detail:** See [`docs/ARCHITECTURE_VISION.md`](docs/ARCHITECTURE_VISION.md) for the complete 5-layer breakdown with individual layer boxes, candidates, unknowns, and dependencies.

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
                    ┌──────────────────────┐
                    │  Layer 4: Visual     │
                    │  (dashboard/widget)  │
                    └──────────┬───────────┘
                               │ reads events from
                    ┌──────────▼───────────┐
                    │  Layer 5: CLI        │
                    │  (MCP + hooks)       │
                    └──────────┬───────────┘
                               │ provides tools for
          ┌────────────────────┴────────────────────┐
          │                                         │
┌─────────▼──────────┐              ┌───────────────▼───────┐
│  Layer 1: Router   │              │  Layer 2: Decision    │
│  (classification)  │──────────────│  Engine (integration) │
└─────────┬──────────┘  informs     └───────────┬───────────┘
          │                                     │
          │          ┌─────────────────┐         │
          └──────────│  Layer 3:       │─────────┘
                     │  MEMORY         │
                     │  (actual        │
                     │   Layer 0)      │
                     └─────────────────┘

  CIRCULAR DEPENDENCY:
  - Router needs memory (past routing outcomes)
  - Memory needs router (to know what to store)
  - Decision engine needs both
  - This must be resolved in implementation:
    start with files (no memory), add memory
    incrementally as routing data accumulates
```

### What's Locked vs Open

```
STATUS KEY:
  ██ LOCKED (Confidence 5 — definitional, don't re-litigate)
  ░░ INVESTIGATION LANE (Confidence 4 — direction set, tool choice open)
     OPEN (Confidence 1-3 — needs investigation)

DECISIONS:
  ██ Hybrid Claude+Gemini (not dual Claude)
  ██ Claude as top-level orchestrator
  ██ Veto pattern for critique (not consensus)
  ██ Invisible experience (user types `claude` normally)
  ██ Gemini free tier or AI Pro ($0-20/mo, NOT Ultra)
  ██ Rate limit exhaustion: notify and stop
  ██ Cross-model critique: user-triggered or complexity-triggered

  ░░ Semantic routing approach (RouteLLM / Aurelio)
  ░░ Communication pattern (hybrid hierarchical + mesh)
  ░░ Visibility model (silent CLI + optional dashboard)
  ░░ Memory is architecturally central (tool choice open)
  ░░ MCP mechanism (Level 3 — "don't double down yet")

     OPEN: Memory system tool selection
     OPEN: MCP vs Bash + file I/O
     OPEN: Zero-code baseline test (never done)
     OPEN: Decision engine design
     OPEN: Visual interface design
     OPEN: MVP scope
     OPEN: Build order validation
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
| **MCP vs Bash + file I/O** | Why is MCP better than `gemini -p 'task' > result.md`? Not yet prototyped. |
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

*Last updated: January 30, 2026 (Session 4 — architecture visual format update)*
