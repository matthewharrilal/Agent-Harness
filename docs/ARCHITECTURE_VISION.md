# Agent Harness: Canonical Architecture Vision

**Single source of truth for the 5-layer architecture.** Every document in this repository points here for architecture context.

**Last updated:** January 30, 2026

---

Here's how the whole thing looks structurally, layer by layer, then as an integrated system.

## The 5 Layers (Individual)

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
│  Status: Level 3 — "don't double down on MCP yet"               │
│                                                                  │
│  The user types: claude                                          │
│  Everything below is invisible.                                  │
│                                                                  │
│  Three mechanisms (probably composing, not competing):            │
│                                                                  │
│  ┌─ MCP Server ──────────┐  ┌─ Hooks ──────────────────────┐   │
│  │ .mcp.json spawns      │  │ .claude/settings.json         │   │
│  │ harness as child proc │  │ 12 lifecycle events:          │   │
│  │                       │  │ SessionStart, PreToolUse,     │   │
│  │ Tools exposed:        │  │ PostToolUse, SubagentStart,   │   │
│  │ • delegate_to_gemini  │  │ Stop, SessionEnd, etc.        │   │
│  │ • shared_memory_rw    │  │                               │   │
│  │ • check_rate_limits   │  │ CAN observe, CANNOT return    │   │
│  │ • route_task          │  │ results to Claude's convo     │   │
│  │                       │  │                               │   │
│  │ CAN return results    │  │ Used for: dashboard events,   │   │
│  │ CANNOT see convo hist │  │ monitoring, side-effects      │   │
│  │ 25K output cap        │  │                               │   │
│  │ 60-120s timeout       │  │                               │   │
│  └───────────────────────┘  └───────────────────────────────┘   │
│                                                                  │
│  ┌─ CLAUDE.md ───────────────────────────────────────────────┐  │
│  │ Behavioral steering. Tells Claude WHEN to use tools:      │  │
│  │ "When task >100K tokens, use delegate_to_gemini"          │  │
│  │ "When researching broad topics, use delegate_to_gemini"   │  │
│  │ No runtime capability — static instructions only.         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ALTERNATIVE STILL OPEN: Could Bash + file I/O replace MCP?     │
│  gemini -p 'task' > result.md → Claude reads file.              │
│  Clunky but zero infrastructure. Not yet prototyped.             │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Integrated System (How Layers Connect)

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

---

## The Dependency Reality

Memory is labeled Layer 3 but is the actual foundation. Everything depends on it.

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

---

## What's Locked vs Open (Status Map)

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

## The Communication Architecture (Session 3)

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

  Control flow: Hierarchical (orchestrator controls)
  Context flow: Mesh (subtasks share directly)
  Status: ░░ INVESTIGATION LANE — exploring direct
          subagent-to-subagent messaging
```

---

## The Genuine Gap (~40%)

What existing tools (PAL MCP + CCProxy + CodexBar) cover vs what remains:

```
EXISTING TOOLS COVER (~55-60%):
  ✓ Manual delegation (PAL MCP clink tool)
  ✓ Rule-based routing by token count/model (CCProxy)
  ✓ Rate limit visibility (CodexBar menu bar widget)
  ✓ Auto-failover on 429 (claude-code-mux)
  ✓ Real-time inter-agent messaging (hcom)
  ✓ Parallel Claude instances (Conductor Build)

THE GENUINE GAP (~40%):
  ✗ Semantic task-type routing (by what the task IS)
  ✗ Persistent cross-provider memory (across sessions)
  ✗ Proactive rate limit prediction (before 429)
  ✗ Cross-model critique orchestration (veto pattern)
  ✗ The integrated invisible experience (all above, together)

OPEN QUESTION: Is the ~40% gap worth building for?
  → Zero-code baseline test has NEVER been done
  → Could invalidate or validate the entire project
```

---

*This document is the canonical architecture reference. All other documents point here for diagrams and status. If the architecture changes, update HERE first, then propagate.*
