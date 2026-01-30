# Agent Harness: Canonical Architecture Vision

**Purpose:** Single source of truth for the 5-layer architecture. All diagrams, dependency graph, and status map live here. Every other document points to this file for architecture context.

**Last updated:** January 30, 2026 (Session 4 — documentation audit)

---

## The 5 Layers (Individual)

Each layer is a distinct investigation area. No layer has a concrete implementation decided.

### Layer 1: Semantic Router

```
┌─────────────────────────────────────────────────┐
│  LAYER 1: SEMANTIC ROUTER                       │
│                                                  │
│  Purpose: Classify task intent without LLM call  │
│  Status:  ░░ INVESTIGATION LANE                  │
│                                                  │
│  Candidates:                                     │
│    - RouteLLM (90-95% accuracy, ~50ms)           │
│    - Aurelio Semantic Router (~85%, ~100ms)       │
│                                                  │
│  Inputs:  User prompt, task context              │
│  Outputs: Task classification (research/code/    │
│           debug/multimodal/etc.)                 │
│                                                  │
│  Unknowns:                                       │
│    - Neither tested on Claude+Gemini pair        │
│    - Cold-start behavior undefined               │
│    - Accuracy on ambiguous tasks (e.g.,          │
│      "refactor auth module" = code or research?) │
│                                                  │
│  Dependencies: Layer 3 (memory informs routing)  │
└─────────────────────────────────────────────────┘
```

### Layer 2: State-Aware Decision Engine

```
┌─────────────────────────────────────────────────┐
│  LAYER 2: STATE-AWARE DECISION ENGINE            │
│                                                  │
│  Purpose: Combine all signals into a single      │
│           routing decision                       │
│  Status:  OPEN (conceptual only)                 │
│                                                  │
│  Candidates: Custom (no off-the-shelf solution)  │
│                                                  │
│  Inputs:                                         │
│    - Task classification (from Layer 1)          │
│    - Rate limit headroom (both providers)        │
│    - Memory context (from Layer 3)               │
│    - Subtask sibling findings                    │
│    - Historical routing outcomes                 │
│                                                  │
│  Outputs: Routing decision                       │
│    - Target provider (Claude / Gemini)           │
│    - Confidence score                            │
│    - Routing rationale (for dashboard logging)   │
│                                                  │
│  Unknowns:                                       │
│    - Integration architecture not designed        │
│    - "One system, not three features" insight     │
│      means this layer IS the integration point   │
│    - Rate prediction algorithm undefined          │
│                                                  │
│  Dependencies: Layer 1, Layer 3                   │
└─────────────────────────────────────────────────┘
```

### Layer 3: Cross-Provider Memory

```
┌─────────────────────────────────────────────────┐
│  LAYER 3: CROSS-PROVIDER MEMORY                  │
│                                                  │
│  Purpose: Shared state across providers and      │
│           sessions (architecturally central)     │
│  Status:  ░░ INVESTIGATION LANE                  │
│                                                  │
│  Candidates:                                     │
│    - Mem0 (semantic extraction, dual storage)    │
│    - sqlite-vec (lightweight vector search)      │
│    - Plain files (AGENTS.md, .harness/*.json)    │
│    - ChromaDB (vector DB, simpler than Mem0)     │
│                                                  │
│  Inputs:                                         │
│    - Session findings, routing outcomes          │
│    - Cross-subtask context                       │
│    - User decisions and preferences              │
│                                                  │
│  Outputs:                                        │
│    - Routing signal to Layer 2                   │
│    - Context injection for delegated tasks       │
│    - Cross-session persistence                   │
│                                                  │
│  Unknowns:                                       │
│    - What to store, how long, when to query      │
│    - Staleness detection undefined               │
│    - Token overhead of memory reads              │
│                                                  │
│  Dependencies: None (but everything depends      │
│    on it — actual Layer 0)                       │
│                                                  │
│  DOUBLE DUTY:                                    │
│    1. Cross-subtask: Task B sees Task A findings │
│    2. Cross-session: Tomorrow knows today        │
└─────────────────────────────────────────────────┘
```

### Layer 4: Visual Interface

```
┌─────────────────────────────────────────────────┐
│  LAYER 4: VISUAL INTERFACE                       │
│                                                  │
│  Purpose: Optional observability layer           │
│  Status:  OPEN (not designed)                    │
│                                                  │
│  Candidates:                                     │
│    - CodexBar-style menu bar widget              │
│    - Conductor Build-inspired web dashboard      │
│    - localhost:3200 browser page                 │
│                                                  │
│  Inputs:                                         │
│    - Events from hooks (12 lifecycle events)     │
│    - Routing decisions from Layer 2              │
│    - Rate limit data from both providers         │
│    - Cost tracking data                          │
│                                                  │
│  Outputs:                                        │
│    - Ambient status (menu bar: green/yellow/red) │
│    - Deep analysis (routing analytics, traces)   │
│    - Plan approval queue                         │
│                                                  │
│  Unknowns:                                       │
│    - Concrete UI design not started              │
│    - Menu bar widget vs browser page vs both     │
│    - What information is genuinely worth the     │
│      context-switching cost                      │
│                                                  │
│  Dependencies: Layer 2, Layer 5 (hooks)          │
└─────────────────────────────────────────────────┘
```

### Layer 5: CLI Experience

```
┌─────────────────────────────────────────────────┐
│  LAYER 5: CLI EXPERIENCE                         │
│                                                  │
│  Purpose: Invisible integration into Claude Code │
│  Status:  ░░ INVESTIGATION LANE                  │
│           (Level 3 — "don't double down on MCP") │
│                                                  │
│  Candidates:                                     │
│    - MCP server + hooks + CLAUDE.md (leading)    │
│    - Hooks only (simpler, can't return results)  │
│    - Custom slash commands                       │
│    - Claude Agent SDK (replaces Claude Code)     │
│    - Hybrid of above (most likely)               │
│                                                  │
│  Inputs:                                         │
│    - User types `claude` normally                │
│    - .mcp.json auto-spawns harness               │
│    - CLAUDE.md provides behavioral steering      │
│                                                  │
│  Outputs:                                        │
│    - MCP tools: delegate_to_gemini,              │
│      shared_memory_read/write, check_rate_limits,│
│      route_task                                  │
│    - Hook events → dashboard                     │
│                                                  │
│  Unknowns:                                       │
│    - MCP vs Bash+files not prototyped            │
│    - MCP 25K output limit for large tasks        │
│    - MCP 2-min timeout for Gemini delegation     │
│    - Can hooks alone replace MCP for delegation? │
│                                                  │
│  Dependencies: All other layers feed into this   │
└─────────────────────────────────────────────────┘
```

---

## The Integrated System Flow

How the layers connect, from user prompt to output.

```
USER types: claude (normal interactive mode)
  │
  ▼
CLAUDE CODE starts normally
  │
  ├── .mcp.json → spawns harness MCP server (Layer 5)
  │
  ├── CLAUDE.md → behavioral steering for delegation
  │
  ├── User prompt arrives
  │       │
  │       ▼
  │   ┌─────────────────────────────────────────┐
  │   │  LAYER 1: SEMANTIC ROUTER               │
  │   │  "What kind of task is this?"            │
  │   │  (RouteLLM / Aurelio — investigating)    │
  │   └────────────────┬────────────────────────┘
  │                    │ task classification
  │                    ▼
  │   ┌─────────────────────────────────────────┐
  │   │  LAYER 2: DECISION ENGINE               │
  │   │  Combines:                               │
  │   │    + task type (from Layer 1)            │
  │   │    + rate limit headroom (both providers)│
  │   │    + memory context (from Layer 3)       │
  │   │    + historical success rates            │
  │   │  → Routing decision: Claude or Gemini    │
  │   └────────────────┬────────────────────────┘
  │                    │
  │        ┌───────────┴───────────┐
  │        ▼                       ▼
  │   Claude handles          Gemini handles
  │   (native tools)          (via MCP delegation)
  │        │                       │
  │        └───────────┬───────────┘
  │                    │
  │                    ▼
  │   ┌─────────────────────────────────────────┐
  │   │  LAYER 3: CROSS-PROVIDER MEMORY         │
  │   │  Stores: findings, decisions, outcomes   │
  │   │  Serves: context for next routing,       │
  │   │          sibling subtask awareness,      │
  │   │          cross-session persistence       │
  │   └─────────────────────────────────────────┘
  │
  ├── Hooks capture events → Layer 4 (dashboard)
  │
  └── Result returned to user in normal conversation
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

  ⚠️  CIRCULAR DEPENDENCY:
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
