# Investigation Queue

**Created:** January 30, 2026
**Purpose:** Track all open investigation questions, their dependencies, optimal answering order, and connection to the 5-layer architecture.
**Status:** Active — questions being investigated in wave order

---

## How This Document Works

Each question has:
- **Status:** `OPEN` → `IN PROGRESS` → `ANSWERED` → `INTEGRATED`
- **Layer mapping:** Which architecture layers it touches (see [ARCHITECTURE_VISION.md](ARCHITECTURE_VISION.md))
- **Dependencies:** What must be answered first
- **Why it matters:** The architectural consequence of the answer

Questions are organized into **Primary (IQ-1 through IQ-7)** — the user's direct questions — and **Auxiliary (AQ-1 through AQ-4)** — questions surfaced during research that shouldn't get lost.

---

## Dependency Graph (Primary 7 Only)

```
                    ┌─────────────┐
                    │    IQ-1     │  DECOMPOSITION BEFORE CLASSIFICATION
                    │ (Layer 1,2) │  [ROOT NODE — no dependencies]
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
     ┌────────────┐  ┌──────────┐  ┌──────────┐
     │   IQ-7     │  │   IQ-3   │  │   IQ-4   │
     │ Semantic   │  │ Routing  │  │ LLM      │
     │ vs Heur.   │  │ Nuance   │  │ Costs    │
     └─────┬──────┘  └────┬─────┘  └─────▲────┘
           │              │               │
           │              ├───────────────┘
           │              │
           │         ┌────▼─────┐
           └────────►│   IQ-5   │  VISUAL INTERFACE
                     │ (Layer 4)│  (what to surface = what layers produce)
                     └──────────┘

   PARALLEL TRACKS (independent of critical path):

     ┌─────────────┐        ┌─────────────┐
     │    IQ-2     │        │    IQ-6     │
     │  Conductor  │        │  MCP Server │
     │  Mapping    │        │ Reliability │
     └─────────────┘        └─────────────┘
```

### Why IQ-1 Is the Root Node (Not Memory)

Memory (Layer 3) is the **implementation** foundation — everything depends on it at build time. But IQ-1 (decomposition) is the **investigation** foundation — the answer reshapes what every other layer receives as input. If we decompose before classifying:
- The router sees subtasks instead of raw prompts → may make heuristics viable (changes IQ-7)
- Changes the cost model → every decomposition step has overhead (changes IQ-4)
- Alters the granularity of routing decisions → more signals for Layer 2 (changes IQ-3)

---

## Investigation Waves

| Wave | Questions | Start Condition | Rationale |
|------|-----------|----------------|-----------|
| **1** | IQ-1, IQ-6, IQ-2 | Immediately (parallel) | IQ-1 is root node. IQ-6 and IQ-2 are independent tracks. |
| **2** | IQ-7, IQ-3 | After IQ-1 | IQ-7 depends on decomposition answer. IQ-3 benefits from IQ-1 + IQ-7. |
| **3** | IQ-4 | After IQ-1 + IQ-3 | Full cost surface requires knowing pipeline shape and routing factors. |
| **4** | IQ-5 | After waves 1-3 | Visual interface is downstream — must know what's worth surfacing. |

---

## Primary Investigation Questions

---

### IQ-1: Task Decomposition Before Classification

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Layers** | 1 (Semantic Router), 2 (Decision Engine) |
| **Dependencies** | None — root node |
| **Wave** | 1 |
| **Blocks** | IQ-7, IQ-3, IQ-4, IQ-5 |

**The Question:**
Should we decompose tasks into effective subtasks BEFORE doing task classification and confidence scoring? This would give more granular routing rather than surface-level classification of complex tasks.

**Example:**
```
Current pipeline:    prompt → classify → route
Proposed pipeline:   prompt → decompose → classify each → route each

"refactor auth module to use JWT" decomposes into:
  1. Search codebase for current auth patterns    → research → Gemini?
  2. Design JWT token structure                    → architecture → Claude?
  3. Modify auth middleware                        → code → Claude?
  4. Update all call sites                         → bulk edit → Gemini?
  5. Write tests for new auth flow                 → code → Claude?

Each subtask may have a different optimal model.
```

**Why It Matters:**
This changes the fundamental pipeline shape. If we decompose first, every downstream layer receives subtasks instead of raw prompts. Layer 1 classifies subtasks (potentially simpler, more accurate). Layer 2 makes per-subtask routing decisions (more granular). The cost model changes (decomposition itself has overhead). The visual interface shows subtask-level routing, not just prompt-level.

**Answer:**

*(To be filled during investigation)*

---

### IQ-2: Gemini Conductor Layer Mapping

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Layers** | All 5 (external reference analysis) |
| **Dependencies** | None — independent track |
| **Wave** | 1 |
| **Blocks** | None directly |

**The Question:**
If Gemini Conductor wasn't limited to Gemini CLI, what parts of these 5 layers does it tackle? What does it abstract or improve in each layer's lifecycle? The user is having trouble understanding which layers Conductor operates in when unbounded.

**Why It Matters:**
Helps distinguish what's genuinely novel in this harness vs what existing orchestration tools already solve. If Conductor (unbounded) covers Layers 1-3, the harness's unique value is only Layers 4-5 plus the cross-provider aspect. If Conductor only touches Layer 1, the gap is larger than assumed.

**Answer:**

*(To be filled during investigation)*

---

### IQ-3: Layer 2 Routing Nuance — Forward-Thinking Conservation

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Layers** | 2 (Decision Engine), 3 (Memory for historical data) |
| **Dependencies** | IQ-1 (pipeline shape affects what gets routed) |
| **Wave** | 2 |
| **Blocks** | IQ-4, IQ-5 |

**The Question:**
Beyond task classification and current rate limits: if even the task classification says Model A is best suited, given prospective future tasks it might be better to conserve Model A's usage and route to Model B. What other nuanced factors go into routing that we aren't thinking about?

**Example:**
```
Scenario: It's Wednesday. Claude is at 60% weekly usage.
          Two days left. User has a complex architecture
          review Friday.

Current routing: "This research task → Claude (best match)"
Smart routing:   "This research task → Gemini (good enough)
                  because Friday's architecture review
                  NEEDS Claude and we're burning capacity"
```

**Why It Matters:**
Layer 2 is explicitly marked as "DOES NOT EXIST YET — hardest part" in the architecture. This question probes the depth of that difficulty. It's not just "who's best for this task?" but "who's best for this task given everything we know about future needs, historical patterns, and capacity trajectories?"

**Answer:**

*(To be filled during investigation)*

---

### IQ-4: Overarching LLM Costs Across All Layers

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Layers** | ALL (cross-cutting) |
| **Dependencies** | IQ-1 (decomposition adds cost layer), IQ-3 (routing factors affect cost decisions) |
| **Wave** | 3 |
| **Blocks** | IQ-5 |

**The Question:**
Mem0 incurs LLM costs for memory operations. Layer 1 incurs costs for classification, decomposition, confidence scoring, routing. What are ALL the auxiliary LLM costs at every layer on top of the regular task execution costs? Use metacognition to understand the full cost picture of the harness overhead.

**Why It Matters:**
The harness exists to SAVE model usage. If its own overhead is significant, the math might not work. If decomposition (IQ-1) + classification + routing + memory writes + memory reads consume 15% of capacity per task, the harness needs to save more than 15% to break even. This is the ROI question.

**Cost surfaces to map (once dependencies are answered):**
```
Layer 1: Classification LLM calls (per prompt or per subtask?)
Layer 1: Decomposition LLM calls (if IQ-1 says yes)
Layer 1: Confidence scoring (embedded in classification or separate?)
Layer 2: Decision engine computation (heuristic or LLM-based?)
Layer 3: Mem0 memory extraction (LLM call per write)
Layer 3: Mem0 memory retrieval (LLM call per read?)
Layer 3: Memory maintenance/compaction
Layer 5: CLAUDE.md context injection (token cost of harness instructions)
Critique: Cross-model critique when triggered (1.5-2x task cost)
```

**Answer:**

*(To be filled during investigation)*

---

### IQ-5: Visual Interface — What Should Actually Surface?

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Layers** | 4 (Visual Interface), observes all others |
| **Dependencies** | IQ-1, IQ-3, IQ-4, IQ-7 (must know what the system produces before designing what to show) |
| **Wave** | 4 |
| **Blocks** | None |

**The Question:**
Beyond just current usage percentage, what would be the most useful information to surface? Routing decisions, task decomposition details, justifications for routing choices, rate limit effects on routing, the hidden between-the-lines nuances of the system. Think outside the box about what a power user would want visibility into.

**Why It Matters:**
The interface is downstream of all logic decisions — it displays what the pipeline produces. Designing it requires understanding what's worth showing. A dashboard that shows "Claude: 42%" is CodexBar (already exists). The harness dashboard needs to show what CodexBar can't: WHY things were routed where they were, what the system learned, what it's predicting.

**Categories to consider (once dependencies answered):**
- Real-time: Current routing decisions and justifications
- Historical: Routing accuracy over time, model performance comparisons
- Predictive: Rate limit forecasting, capacity planning
- Diagnostic: Why did the system make that choice? What would it do differently?
- Learning: What has the memory system captured? What patterns emerged?
- Cost: Harness overhead vs savings (informed by IQ-4)

**Answer:**

*(To be filled during investigation)*

---

### IQ-6: MCP Server Reliability, Tools Exposed & Robustness

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Layers** | 5 (CLI Experience), but delivery mechanism for everything |
| **Dependencies** | None — independent track |
| **Wave** | 1 |
| **Blocks** | None directly (but informs all implementation) |

**The Question:**
The user's core worry: LLM-based tool selection is unreliable. Skills and MCP tools that are "dynamically invoked upon a justification of inference" are hit-or-miss. How to make the MCP server more reliable? What tools should be exposed? How to make this robust?

**User's Additional Context:**
- Believes MCP server is the best delivery mechanism (subscription justification)
- But worried about invocation reliability
- Wants a CONVERSATION about tools exposed, not just a list
- Wants to discuss how this would work with all the prior questions in mind
- Wants deeper MCP familiarity — understanding how MCP servers operate in depth, not surface level

**Why It Matters:**
If MCP invocation is unreliable, the entire harness might not be used when needed most. The system is invisible by design — but invisible + unreliable = useless. This is the delivery mechanism for the entire 5-layer architecture.

#### Pre-Research Finding: Hooks + MCP Hybrid Approach

Subagent research on MCP reliability uncovered a critical architectural insight:

**The problem:** MCP tool invocation is inference-based — Claude decides whether to call tools. Research shows 12-50% failure rate depending on conditions. Tool Search (when >10% context is tool definitions) adds a discovery step that further reduces reliability. Native tools always win in discoverability.

**The solution: Hooks (guaranteed) + MCP tools (optional)**

| Layer | Hooks (Always Fire) | MCP Tools (LLM Chooses) |
|-------|-------------------|----------------------|
| Rate checking | `UserPromptSubmit`: inject rate state | `check_rate_status` (explicit query) |
| Memory | `UserPromptSubmit`: inject relevant snippets | `query_memory` (deep lookup) |
| Rate enforcement | `PreToolUse`: block if exhausted | — |
| Tracking | `PostToolUse`: update counters | — |
| Delegation | — | `delegate_to_gemini` |
| Critique | — | `request_critique` |
| Persistence | `Stop`: write session to memory | — |

**Key insight:** Hooks guarantee the floor (critical functions always execute). MCP tools raise the ceiling (optional enhancements Claude can choose). The system doesn't break if MCP tools aren't called, because hooks handle the critical path.

**`UserPromptSubmit` hook is the linchpin** — fires on every prompt, can inject context that shapes Claude's reasoning for that turn. This is how you make the harness invisible: not by hoping Claude calls your tools, but by injecting the right context so Claude's natural reasoning aligns with what the harness needs.

**Important:** This pre-research is a starting point for conversation, not the conclusion. The user wants to discuss:
- What the hooks implementation would actually look like vs MCP
- How tools would be exposed and what tools specifically
- How this works in practice with all the other questions in mind
- Deeper MCP operational understanding — not surface level

**Answer:**

*(To be filled during investigation and conversation)*

---

### IQ-7: Semantic Routing vs Heuristics Consistency

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Layers** | 1 (Semantic Router), 2 (Decision Engine) |
| **Dependencies** | IQ-1 (if decomposition produces clear subtasks, heuristics may suffice) |
| **Wave** | 2 |
| **Blocks** | IQ-5 |

**The Question:**
The architecture diagram still shows heuristic-style examples ("audit → Claude, 140K token analysis → Gemini"). Are we truly moving to semantic routing or still using these heuristics? This is a consistency question about stated direction vs diagram content.

**The Tension:**
```
STATED DIRECTION (START_HERE.md):
  "Semantic routing without training"
  RouteLLM or Aurelio — pre-trained routers, 90-95% accuracy

DIAGRAM EXAMPLES (ARCHITECTURE_VISION.md):
  "Task A: audit" → Claude
  "Task C: 140K token codebase analysis" → Gemini

  These look like heuristics:
  - "audit" = security-critical → Claude (rule)
  - "140K tokens" = large context → Gemini (rule)

  Where's the semantic classification?
```

**Why It Matters:**
If semantic routing is the direction, the diagrams should reflect that — show the classification output, not task-type labels. If heuristics are actually sufficient (especially after decomposition per IQ-1), we might not need RouteLLM/Aurelio at all, which changes the entire Layer 1 design and eliminates a dependency.

**Answer:**

*(To be filled during investigation)*

---

## Auxiliary Questions

These emerged from subagent analysis of the 7 primary questions. Not the user's primary focus, but tracked here so they don't get lost. May become primary questions as investigation progresses.

---

### AQ-1: Signal Arbitration — When Layer 2's Four Signals Disagree

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Surfaced From** | IQ-3 and IQ-7 analysis |
| **Layers** | 2 (Decision Engine) |

Route signal says "Claude." Rate limits say "Gemini." Memory says "Gemini failed this last time." Subtask context says "subtasks 1-2 ran on Claude for continuity." Four signals, four answers. What wins? Is there priority ordering? Weighted scoring? User override?

This IS Layer 2's core problem. When we get to designing the Decision Engine, this question is the first thing to answer.

---

### AQ-2: Error Recovery and Partial Failure

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Surfaced From** | IQ-1 (decomposition creates partial failure scenarios), IQ-3 (routing must account for failure history) |
| **Layers** | All (cross-cutting) |

If Gemini fails mid-task, what happens? Retry same model? Escalate to Claude? Roll back partial state? What does memory record? If 5 decomposed subtasks and subtask 4 fails, what's the contract?

Decomposition (IQ-1) makes this more acute — more subtasks means more failure points. The answer shapes Layer 2 (retry logic), Layer 3 (what to persist on failure), and Layer 5 (how to surface failures).

---

### AQ-3: Cold Start and Bootstrap

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Surfaced From** | IQ-3 (routing needs memory), IQ-4 (cold start = maximum overhead with minimum benefit) |
| **Layers** | 3, 2, 1 |

Memory is "Layer 0" but empty on Day 1. The system must work immediately. Hardcoded defaults? Starter knowledge? How many task-cycles before memory actually helps routing?

The cold start problem is where the harness is weakest — all its smarts depend on accumulated data, but it has none at first. This is also where auxiliary LLM costs (IQ-4) are highest relative to benefit.

---

### AQ-4: Latency Budget

| Field | Value |
|-------|-------|
| **Status** | `OPEN` |
| **Surfaced From** | IQ-1 (decomposition adds latency), IQ-4 (cost and latency are related but different constraints) |
| **Layers** | All (cross-cutting) |

Every layer adds latency. Potentially 500ms-1.5s before a task even starts executing. For "fix this typo" that's noticeable. Per-layer latency budget? Parallelization opportunities? Fast path for trivial tasks that skip decomposition/routing entirely?

---

## Status Summary

| ID | Question | Status | Wave | Dependencies Met? |
|----|----------|--------|------|-------------------|
| IQ-1 | Decomposition Before Classification | `OPEN` | 1 | Yes — no deps |
| IQ-2 | Gemini Conductor Layer Mapping | `OPEN` | 1 | Yes — no deps |
| IQ-3 | Layer 2 Routing Nuance | `OPEN` | 2 | No — needs IQ-1 |
| IQ-4 | Overarching LLM Costs | `OPEN` | 3 | No — needs IQ-1, IQ-3 |
| IQ-5 | Visual Interface Surfacing | `OPEN` | 4 | No — needs IQ-1, IQ-3, IQ-4, IQ-7 |
| IQ-6 | MCP Server Reliability | `OPEN` | 1 | Yes — no deps |
| IQ-7 | Semantic vs Heuristics | `OPEN` | 2 | No — needs IQ-1 |
| AQ-1 | Signal Arbitration | `OPEN` | — | Tracked, not scheduled |
| AQ-2 | Error Recovery | `OPEN` | — | Tracked, not scheduled |
| AQ-3 | Cold Start | `OPEN` | — | Tracked, not scheduled |
| AQ-4 | Latency Budget | `OPEN` | — | Tracked, not scheduled |

---

## Answering Protocol

For each question, in wave order:

1. **Deep-dive** with subagents researching from multiple angles
2. **Present findings** to user for DISCUSSION (not just a report)
3. **User and Claude discuss** implications together
4. **Document answer** in this file under the question's Answer section
5. **Note any new auxiliary questions** that surface during investigation
6. **Cross-reference** to other questions when dependencies exist
7. **Update status** in the summary table

---

*This document tracks investigation progress. Questions are documented for tracking, not resolved. Answers will be filled as each wave completes. New auxiliary questions can be added as investigation surfaces them.*
