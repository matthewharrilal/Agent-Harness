# Agent Harness: Architecture Reflections & Design Decisions
## Living document -- tracks thought process, not just findings

---

## THE CORE REALIZATION (Updated Jan 27, 2026 - Late Session)

> **Confidence: Level 3 — Working Hypothesis.** The MCP server approach is architecturally sound but the user explicitly said "let's not double down on MCP yet." Alternatives (hooks-only, Agent SDK, Bash+file I/O) are not fully investigated. See PROJECT_CONTEXT.md § Decision Confidence Registry for the full confidence framework.

The harness is **not a CLI wrapper**. It is **not** a separate command you type. It is an MCP server + hooks system that runs inside your normal Claude Code experience. You type `claude` as you always do. The harness is baked in -- invisible, ambient, always present.

```
YOU type: claude (normal interactive mode, NOT headless)
  |
  v
CLAUDE CODE starts normally
  |
  ├── Reads .mcp.json → spawns "agent-harness" MCP server (stdio, on-demand)
  │   MCP server provides tools:
  │     - delegate_to_gemini (spawns gemini -p subprocess, returns result)
  │     - shared_memory_read / shared_memory_write
  │     - check_rate_limits (both providers)
  │     - route_task (intelligent routing analysis)
  │
  ├── Reads CLAUDE.md → instructions tell Claude WHEN to use harness tools
  │   "When task involves 100K+ tokens of codebase, use delegate_to_gemini"
  │   "When researching broad topics, use delegate_to_gemini"
  │
  ├── Reads .claude/settings.json → hooks fire on EVERY event:
  │   SessionStart → POST to dashboard server
  │   PreToolUse → log tool call (including MCP tool calls)
  │   PostToolUse → POST result to dashboard
  │   Stop → POST turn-complete
  │   SessionEnd → POST session-end
  │
  └── Dashboard server (separate process, always running in background)
      localhost:3200 or CodexBar-style menu bar widget
      Receives POSTed events from hooks
      Shows: rate limits, costs, agent activity, routing decisions
```

### The 5-Layer Architecture (Session 3 Vision)

| Layer | What | Investigation Status |
|-------|------|---------------------|
| **1. Semantic Router** | Aurelio or RouteLLM | Investigating — no choice made |
| **2. State-Aware Decision Engine** | Custom | Conceptual — consumes route + rate limits + memory + subtask context |
| **3. Cross-Provider Memory** | Mem0 / sqlite-vec / files | Investigating — no choice made. Architecturally central (double duty) |
| **4. Visual Interface** | CodexBar + Conductor Build inspired | Conceptual — not designed |
| **5. CLI Experience** | MCP server + hooks | Level 3 — "don't double down on MCP yet" |

Layer 2 is where the "one system, not three features" insight lives — the integration point. No concrete implementation decided for any layer.

> **Canonical architecture reference:** See [`docs/ARCHITECTURE_VISION.md`](ARCHITECTURE_VISION.md) for the definitive diagrams. This file contains technical deep-dive and implementation analysis.

---

### Why MCP Server, Not CLI Wrapper

**Original design (SUPERSEDED):** The harness wraps both CLI binaries, spawning `claude -p` and `gemini -p` in headless mode. The user types `harness "do this"`.

**Updated design (CURRENT):** The user explicitly wants the normal interactive Claude Code experience -- plan mode, subagents, text box, everything. They do NOT want headless mode. The harness must be baked INTO the interactive experience, not wrapped around it.

**The MCP server approach solves this:** Claude Code natively supports MCP servers via `.mcp.json`. When you start a Claude session, it automatically spawns any configured MCP servers as child processes. These servers provide tools that Claude can call. By making the harness an MCP server, Claude Code gains delegation and shared memory capabilities without changing the user experience at all.

**This is how PAL MCP Server (10K stars) works.** Claude Code spawns PAL as an MCP child process. PAL's `clink` tool spawns Gemini CLI as a subprocess. Result flows back to Claude. User sees it in their normal conversation.

### Other Implementation Options (Not Yet Ruled Out)

We are still investigating alternatives to MCP:
- **Claude Code hooks only** (no MCP server): Hooks could intercept events and trigger Gemini invocations. Simpler but less integrated -- can't return results to Claude's conversation.
- **Custom slash commands**: `/delegate-gemini "task"` as a skill. Simpler but manual.
- **Claude Agent SDK**: Build a full agent application that manages both models. More control but replaces Claude Code entirely.
- **Hybrid**: MCP for delegation tools + hooks for event capture + CLAUDE.md for behavioral steering. This is the most likely final design.

**More research needed on the tradeoffs.**

---

## DECISION: COMPOSED ARCHITECTURE (Not Fork, Not Pure Build)

**Date:** January 27, 2026

**Why not fork:** No single project is close enough. PAL MCP is a tool library, not a harness. Claude-Flow has 85% mock implementations. Claude Squad has AGPL license. AgCluster has no Gemini support.

**Why not build from scratch:** MCP protocol, CLI session management, and provider abstraction are all solved problems. No need to reinvent them.

**Decision:** Build a custom orchestration layer that uses existing tools as dependencies:
- Claude Code CLI (subscription auth, native)
- Gemini CLI (free OAuth, direct)
- PAL MCP Server (optional, API-key fallback)
- MCP protocol SDK (tool interface standard)
- Inspired by: Claude Squad (tmux patterns), Claude Octopus (routing heuristics), AgCluster (dashboard patterns)

**Critical constraint:** Anthropic's January 2026 OAuth crackdown means the harness MUST orchestrate the actual `claude` CLI binary to use subscription credits. Third-party OAuth is banned.

---

## DECISION: HYBRID CLAUDE+GEMINI (Not Dual Claude)

**Date:** January 27, 2026

**Why not two Claude Max accounts:**
1. Device-level rate limit bug makes it broken on one machine (keychain collision, unfixed)
2. No model diversity (same failure modes, same blind spots)
3. Correlated downtime risk (same provider)
4. $400/mo for redundancy, not complementary capability

**Why Claude + Gemini:**
1. Zero collision risk (different providers, different keychains)
2. Complementary strengths (Claude: precision/debugging, Gemini: 1M context/multimodal/search)
3. Independent rate pools with different reset cycles (weekly vs daily)
4. Clean TOS (two different services, no circumvention)
5. $200-220/mo instead of $400/mo

---

## THE ABSTRACTION PROBLEM (Core Architectural Challenge)

The harness must make it invisible whether Claude or Gemini is handling a task. This means abstracting:

### 1. Unified Plan Mode

Claude Code plan mode:
- EnterPlanMode activates → blocks write tools, read-only exploration
- Plan file created → presented via ExitPlanMode
- User approves → execution begins

Gemini CLI plan mode:
- Plan mode via config → blocks write tools via policy engine
- `exit_plan_mode` tool presents plan for approval
- User approves/rejects/modifies → execution begins

**Harness responsibility:** Normalize the approval flow. User sees "here's a plan, approve it" regardless of which model made the plan. The harness translates between the two plan mode mechanics.

### 2. Unified Subagent Spawning

Claude Code: Task tool spawns typed agents (Bash, Explore, Plan, general-purpose). Each agent runs in parallel with independent context and scoped tool access. Returns results to parent.

Gemini CLI: No native equivalent. Experimental subagents (PR #4883) were attempted but closed. The CodebaseInvestigator is one built-in subagent. Shell-spawn workaround exists (`gemini -p` processes).

**Harness responsibility:** When routing a subagent task to Gemini, the harness spawns `gemini -p` processes with scoped instructions that mimic the Claude subagent pattern. The parent orchestrator sees the same interface regardless of which model is executing.

### 3. Unified Tool Schemas

| Claude Code Tool | Gemini CLI Equivalent | Gap? |
|-----------------|----------------------|------|
| Read | ReadFile, ReadFolder | Covered |
| Write | WriteFile | Covered |
| Edit | Edit | Covered |
| Glob | FindFiles | Covered |
| Grep | SearchText | Covered |
| Bash | Shell (run_shell_command) | Covered |
| WebFetch | WebFetch | Covered |
| WebSearch | GoogleSearch (built-in!) | Gemini advantage |
| Task (subagents) | **MISSING** | Harness must provide |
| EnterPlanMode | exit_plan_mode (different mechanics) | Harness normalizes |
| ExitPlanMode | exit_plan_mode | Different approval flow |
| AskUserQuestion | **In development** (PR #16428, Jan 2026) | Harness must provide |
| TaskCreate/Update/List/Get | WriteTodos (experimental, different) | Harness must normalize |
| NotebookEdit | **MISSING** (Issue #6930) | Harness must provide |

**Harness responsibility:** Normalize tool invocations so routing decisions don't break tool expectations. When the router sends a task to Gemini, translate Claude-style tool calls to Gemini equivalents.

### 4. Result Synthesis

Both models return results in different formats. The harness normalizes output so the parent orchestrator can consume it regardless of source. This ties into the arxiv paper's "perception-execution separation" -- only structured summaries cross agent boundaries, not raw dumps.

---

## THE "TEAM OF RIVALS" PATTERN (From arxiv 2601.14351)

### Key Insight: Self-Review Hurts, Cross-Model Critique Helps

Single agent: 60% accuracy. Single agent + self-review: <60% (WORSE). Multi-agent cross-model critique: **90% accuracy**.

This validates the entire hybrid approach. Claude and Gemini have different training data, different failure modes, different blind spots. When one hallucinates, the other is unlikely to hallucinate the same way.

### Implementation in Our Harness

```
Task arrives → Router assigns to Claude (code generation)
  → Claude generates code
  → Harness sends output to Gemini for critique
  → Gemini reviews: intent alignment, completeness, edge cases
  → Gemini approves → advance to user
  → Gemini rejects → Claude retries with Gemini's feedback
  → Max 3 retries → escalate to user
```

And the reverse for Gemini-generated work -- Claude catches Gemini's 4x higher control-flow error rate.

### Design Principles from the Paper

1. **Routing approach under investigation.** Session 2 proposed FSM (deterministic state machine). Session 3 is investigating pre-trained semantic routers (RouteLLM, Aurelio) which may be more accurate for ambiguous tasks like "refactor auth module." No approach decided.
2. **Hierarchical veto, not consensus voting.** Critic can reject (triggers retry), not negotiate. Paper found veto > democratic consensus.
3. **Context isolation.** Raw data never crosses agent boundaries. Only structured summaries. Prevents context contamination.
4. **Pre-declared acceptance criteria.** Before execution, the planner states what "success" looks like. Gives the critic concrete criteria to evaluate against.
5. **Checkpointing at phase transitions.** Full state serialized at each step. Resume after rate limits. Undo wrong paths.

### Seven Agent Roles Mapped to Our Harness

| Paper Role | Our Harness |
|-----------|------------|
| Planner (constructs execution DAG) | Claude Code (plan mode) |
| Executor (routes tasks) | FSM orchestrator (deterministic code) |
| Data Writers (do the work) | Claude for code, Gemini for research |
| Critics (veto authority) | Cross-model: Claude critiques Gemini, Gemini critiques Claude |
| Responders (human checkpoints) | User-facing interface layer |
| Summarizers (context compression) | Gemini (1M context advantage) |
| SME Experts (domain specialists) | Claude Opus for precision, Gemini for breadth |

### Connection to Claude Code's TeammateTool/Swarm Mode

**Updated Jan 27, 2026 (late session) — Deeper research via claude-sneakpeek repo**

Claude Code has a fully implemented but feature-flagged multi-agent system:
- **TeammateTool** with 13 operations: spawnTeam, discoverTeams, requestJoin, approveJoin, rejectJoin, write, broadcast, requestShutdown, approveShutdown, rejectShutdown, approvePlan, rejectPlan, cleanup
- **5 organization patterns**: Hive (task queue), Specialist (rigid roles), Council (debate), Pipeline (sequential chain), Watchdog (monitoring + auto-fix)
- **File-based coordination**: teams at `~/.claude/teams/{name}/`, messages at `~/.claude/teams/{name}/messages/{session-id}/`, tasks at `~/.claude/tasks/{name}/`
- **3 spawn backends**: in-process, tmux, iTerm2
- **Unlockable via** `claude-sneakpeek` (687 stars, MIT license) — patches feature flag checks in Claude Code binary

**Status**: NOT officially released. Marked as "legacy" in claude-sneakpeek v1.6.3. Boris Cherny (Claude Code dev) mentioned interest in "Long running and Swarm" at Claude Code Meetup Tokyo but no official launch.

**Verdict for harness**: **Complements, does NOT replace.**
- Swarms is Claude-only — no Gemini routing, no external memory, no cross-provider capability
- The harness's unique value (cross-provider bridge) is unaffected
- Official Claude Code subagents (`.claude/agents/`) are production-ready and cover most intra-Claude orchestration needs
- The harness should work ALONGSIDE Claude's native subagent system, adding the cross-provider layer on top

Key tension: TeammateTool's `voteOnDecision` = democratic consensus. The paper argues veto authority is more reliable. For our harness, we use the paper's approach: the critic model's rejection is final, not a vote.

The **Specialist** and **Pipeline** patterns map most closely to the paper's architecture. The harness could implement these patterns with Gemini as a specialist role (researcher, analyzer, critic) in a pipeline alongside Claude's roles (planner, coder, debugger).

**Architecture with Swarms context:**
```
Claude Code (interactive, orchestrator)
  |
  ├── Native subagents (Claude workers via official .claude/agents/)
  │   ├── Explore agent (haiku, read-only)
  │   ├── Code reviewer (sonnet)
  │   └── Test runner (haiku)
  |
  ├── MCP harness tools (cross-provider delegation)
  │   ├── delegate_to_gemini
  │   ├── shared_memory_read/write
  │   ├── check_rate_limits
  │   └── route_task
  |
  └── Hooks (event capture to dashboard)
```
Claude's native system handles Claude-to-Claude work. The harness MCP adds the Claude-to-Gemini bridge.

### Communication Pattern Evolution (Session 3 Update)

Session 2 proposed a Blackboard pattern (shared state, no direct messaging). Session 3 evolved this toward a **hybrid hierarchical + mesh** model:
- **Hierarchical:** Orchestrator (Claude) controls task assignment, failure handling, and final synthesis
- **Mesh:** Subtasks share context directly via a shared context layer — no orchestrator relay needed
- **Under investigation:** The user is exploring direct subagent-to-subagent messaging through persistent messaging platforms, going beyond passive shared-state reads into active message passing

Communication architecture is an active investigation area. Not locked.

---

## GOOGLE AI SUBSCRIPTION: ULTRA IS NOT WORTH IT

**Date:** January 27, 2026

### The Numbers

| Tier | Requests/Day | Pro-Model Prompts (Real) | Price |
|------|-------------|-------------------------|-------|
| Free | 1,000 | ~25-50 (Pro model) | $0 |
| AI Pro | 1,500 | ~50-150 (complex prompts) | $20/mo |
| AI Ultra | 2,000 | ~65-200 (complex prompts) | $250/mo |

Ultra gives you ~33% more requests than Pro for 12.5x the price. Users report Pro model quota exhausted after 2-3 hours on Ultra.

**Comparison to Claude Max 20x ($200/mo):** Claude gives 240-480 hours/week of Sonnet 4, 24-40 hours/week of Opus 4.5. Significantly more sustained top-model usage than AI Ultra.

**Decision:** Start with Gemini free tier ($0). If insufficient, upgrade to AI Pro ($20/mo). Do NOT get Ultra ($250/mo) -- the per-dollar value is terrible for coding-only usage.

**Total budget:** $200 (Claude Max) + $0-20 (Gemini) = **$200-220/mo** (well under $400 ceiling).

**Fallback:** If Gemini free/Pro quotas are insufficient for burst usage, use a Google AI Studio API key for pay-as-you-go overflow. Gemini 3 Pro: $2/M input, $12/M output. Budget ~$20-50/mo for occasional overflow.

---

## INTERFACE: CLI-FIRST WITH OPTIONAL DASHBOARD

The harness enhances the CLI, it doesn't replace it. The dashboard is a secondary monitoring window.

### What the Dashboard Shows (Read-Mostly)
- Real-time agent activity (which tools are being called, what files are being touched)
- Routing analytics (why this task went to Claude vs Gemini, historical patterns)
- Rate limit status for both providers (current usage, time to reset, remaining capacity)
- Cost tracking (per-session, per-agent, per-provider)
- Task timeline and history
- Plan review UI (optional -- can also approve in terminal)

### What the Dashboard Controls (Minimal)
- Pause/resume/cancel running agents
- Override routing decisions
- Budget alerts

### Technology
- Likely Next.js (proven by AgCluster)
- SSE for real-time updates (simpler than WebSockets)
- Captures CLI output via process management layer

---

## THE 21-POINT HARNESS COMPENSATION LIST

Based on the detailed Gemini CLI gap audit, here is every capability the harness must provide to make Gemini a seamless partner to Claude Code. Organized by category:

### A. Plan Mode Compensation (4 items)

1. **Hard tool blocking.** Intercept Gemini's tool calls and reject any write operation (`write_file`, `replace`, destructive shell commands) when plan mode is active. Gemini's enforcement is prompt-based (nightly only), not tool-level. The harness cannot trust it.

2. **Headless plan mode invocation.** Create a wrapper that sets `GEMINI_SYSTEM_MD` to a plan-mode system prompt, configures tool exclusions in a temporary `settings.json`, invokes `gemini -p` with the query, and parses the plan from the response text.

3. **Plan approval loop.** Implement the approve/reject/modify cycle in the harness, re-invoking Gemini with modification feedback until the user approves. Gemini's interactive Shift+Tab cycling is UI-only.

4. **Structured plan normalization.** Convert Gemini's text-based plan output into a structured format compatible with Claude Code's `ExitPlanMode` returns.

### B. Subagent/Task Compensation (4 items)

5. **Agent spawning via subprocess.** Invoke `gemini -p "..." --output-format stream-json` with task-specific system prompts (via `GEMINI_SYSTEM_MD`), tool restrictions (via temporary config), and directory scoping (via `--include-directories`).

6. **Parallel execution manager.** Run multiple Gemini CLI subprocesses concurrently, monitor their JSONL streams, aggregate results. Implement timeout and cancellation.

7. **Task lifecycle system.** Implement `TaskCreate`, `TaskUpdate`, `TaskList`, `TaskGet` equivalents with status tracking (pending/in_progress/completed/failed), dependency management (blockedBy/unblocks), session-scoped JSON persistence, and result storage.

8. **Agent type templates.** Pre-configure profiles that mirror Claude Code's agent types:
   - **Explore agent**: Read-only tools, returns analysis
   - **Plan agent**: Read + search tools, returns structured plan
   - **Bash agent**: Shell-only, returns command output
   - **General agent**: Full tool access

### C. Tool Bridging (2 items)

9. **NotebookEdit implementation.** Provide via MCP or shell wrapper. Must parse `.ipynb` JSON, support cell replacement/insertion/deletion by index or ID, handle code and markdown cell types. Options: `ipynb-ai-cli-editor` CLI or custom MCP server.

10. **AskUserQuestion shim.** Until Gemini's native AskUser tool reaches stable (PR #16988, PR #17344), detect when Gemini needs user input, present through harness UI, feed response back.

### D. MCP Compatibility (3 items)

11. **Schema translation layer.** Before connecting MCP servers to Gemini, preprocess tool schemas: resolve `$defs`/`$ref` references inline (critical -- Anthropic's FastMCP generates these), convert multi-type arrays to `anyOf`, strip `additionalProperties` and `$schema` fields.

12. **Unified MCP configuration manager.** Single source of truth generates `.mcp.json` for Claude Code and `settings.json` entries for Gemini CLI. Handles transport differences (`httpUrl` vs Claude's format).

13. **Tool name mapping registry.** Bidirectional map between Gemini's sanitized tool names (underscores, 63-char truncation) and original MCP tool names Claude Code uses.

### E. Configuration and Memory (3 items)

14. **Shared context file management.** Generate and maintain `CLAUDE.md`, `GEMINI.md`, and `AGENTS.md` from a canonical harness config. Configure Gemini to read `AGENTS.md` via `context.fileName` setting.

15. **Memory synchronization.** When either agent stores a persistent fact (Gemini's `save_memory` → `~/.gemini/GEMINI.md`), detect changes and propagate to shared context. Claude reads updated `AGENTS.md` on next invocation.

16. **Context budget management.** Gemini lacks auto-compaction. Monitor token usage via `--session-summary`, implement context windowing or summarization when approaching limits, selectively load context based on task relevance.

### F. Headless Integration (3 items)

17. **Gemini subprocess manager.** Spawns `gemini -p` with appropriate flags, parses `stream-json` JSONL events in real time, handles timeouts/errors/crashes, supports concurrent invocations with isolation, maps Gemini event types to normalized format.

18. **Session state bridge.** Each `gemini -p` invocation is stateless. The harness maintains conversation history for multi-turn interactions, injects prior context via stdin piping or system prompt, tracks session IDs.

19. **Output format normalization.** Convert Gemini's JSON output schema to match Claude Code's response format so consumers don't need to know which engine handled the task.

### G. Behavioral Parity (2 items)

20. **Approval mode mapping.** Map between Claude's tool-level confirmation and Gemini's global approval modes (`ask_user`, `auto_edit`, `yolo`). Enforce consistent behavior regardless of engine.

21. **Error handling normalization.** Normalize tool execution failures, model refusals, rate limiting/quota exhaustion, and context overflow across both providers.

### Gap Severity Summary

| Capability | Claude Code | Gemini CLI | Gap |
|---|---|---|---|
| Plan Mode | Production | Nightly/experimental | **High** |
| Subagent orchestration | Production (Task tool) | Experimental | **High** |
| Core file/search/shell tools | Production | Production | Low |
| Notebook editing | Production | Missing | **High** |
| User question tool | Production | In development | Medium |
| Task management (CRUD) | Production | Partial (write_todos) | **High** |
| Headless/programmatic | Good | **Excellent** | None (Gemini advantage) |
| MCP support | Production | Production (schema issues) | Medium |
| Memory/config | Manual | **Richer** (save_memory) | None (Gemini advantage) |

**Bottom line:** Gemini's core tools are at parity. Its headless mode and memory system are arguably superior. But the three pillars of Claude Code's agent ecosystem -- **plan mode, subagent orchestration, and task management** -- are exactly where Gemini is most immature. The harness's primary engineering challenge is bridging these three gaps.

---

## DASHBOARD ARCHITECTURE

**Date:** January 27, 2026

### The Dominant Pattern: CLI Control + Browser Observation

Every serious agent orchestrator follows the same spine: **tmux sessions for isolation, a TUI for control, and an optional web dashboard for monitoring**. The CLI is always the control plane; the dashboard is always the observation plane.

### Reference Implementations Studied

| Tool | Control Plane | Observation Plane | Key Insight |
|------|--------------|-------------------|-------------|
| **Claude Squad** | Bubble Tea TUI | Preview/Diff panes | Gold standard TUI. tmux + git worktrees for isolation. Daemon for unattended operation. |
| **Agent Deck** | Bubble Tea TUI + CLI | Status + fuzzy search | MCP Socket Pool shares connections across 20+ sessions (85-90% memory reduction). |
| **disler's observability** | Claude Code hooks | Vue dashboard + SQLite | Closest existing model. Hook scripts POST events → Bun server → SQLite → WebSocket → UI. |
| **Langfuse** | SDK integration | Web dashboard | Industry standard LLM observability. Trace → span → generation hierarchy. Agent graph visualization. |
| **SigNoz** | OpenTelemetry | Pre-built dashboard | Claude Code template with sessions, costs, P95 latency, quota usage, model distribution. |

### Our Dashboard: The Five Essential Panels

```
┌─────────────────────────────────────────────────────────┐
│  CLI (primary interface)                                │
│  $ claude (normal interactive mode — harness is baked in│
│                                                          │
│  ┌──────────────────────┐  ┌──────────────────────────┐ │
│  │ Routing Engine        │  │ Rate Limit Monitor       │ │
│  │ - task analysis       │  │ - Claude: 60% headroom   │ │
│  │ - model selection     │  │ - Gemini: 90% headroom   │ │
│  │ - reason logging      │  │ - cooldown timers        │ │
│  └──────────┬───────────┘  └────────────┬─────────────┘ │
│             │                            │               │
│  ┌──────────▼──────────────────────────▼──────────────┐ │
│  │ Event Bus (internal)                                 │ │
│  │ Captures: routing decisions, tool calls, costs,      │ │
│  │ file changes, rate limits, approvals, completions    │ │
│  └──────────────────────┬──────────────────────────────┘ │
└─────────────────────────┼───────────────────────────────┘
                          │ HTTP POST / SSE
┌─────────────────────────▼───────────────────────────────┐
│  Dashboard Server (localhost:3200)                       │
│  Bun/Node HTTP → SQLite → SSE push                      │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  [Activity] [Routing] [Costs] [Rate Limits] [Plans] │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Panel 1 -- Live Activity Feed:** Scrolling event log filtered by agent/session/event type. Tool calls, file changes, prompts. Modeled after disler's timeline.

**Panel 2 -- Routing Analytics:** Per-decision breakdown with model selection rationale, confidence score, historical comparison, cost differential. "This task cost $0.45 on Claude; would have cost $0.12 on Gemini." No existing tool does this for multi-model setups -- this is our unique value.

**Panel 3 -- Rate Limit Gauges:** Real-time bars showing Claude and Gemini usage vs limits, with cooldown timers and automatic failover indicators. Cross-provider view is a gap in existing tools.

**Panel 4 -- Cost Tracker:** Running totals per session, per day, per model. "Would have cost" comparisons. Routing optimization suggestions based on historical data.

**Panel 5 -- Plan Approval Queue:** When the harness generates a plan, it appears here for approve/reject/modify. This is the ONE interactive element.

### What NOT to Build in the Dashboard

- Do NOT build a plan editor. Edit plans in the CLI.
- Do NOT build agent control (start/stop/pause). Do that in the CLI.
- Do NOT replicate the terminal. The terminal IS the terminal.
- Do NOT build file editing or code review. That's what Claude Squad/Agent Deck do.

The dashboard is a **second monitor** -- you glance at it to check status, costs, and routing, but you never need to touch it to get work done.

### Technology Stack

```
Event Collection:  Bun/Node HTTP server (receives hook POSTs + stream-json pipes)
Storage:           SQLite (lightweight, zero-config, sufficient for local use)
Real-time Push:    SSE (simpler than WebSocket for read-mostly dashboard)
Frontend:          Next.js (static export) or React + Vite
Visualization:     React Flow (routing trees), Recharts (metrics), react-d3-tree (decisions)
Optional Desktop:  Tauri wrapper (20-40MB RAM vs Electron's 200-400MB)
```

**Why SSE over WebSocket:** Dashboard is read-mostly. SSE has automatic reconnection. Works over standard HTTP. For the rare interactive element (plan approval), use a separate REST endpoint.

**Why not Electron:** 200-400MB RAM, 100MB+ app size, 1-2 second startup for a monitoring window is absurd. Tauri (system WebView + Rust) or just a localhost browser tab.

### Event Capture Pipeline

```
Claude Code (hooks/stream-json) ──┐
                                   ├──> Event Collector (HTTP POST) ──> SQLite ──> SSE ──> Dashboard
Gemini CLI (stream-json)         ──┘
```

Claude Code hooks capture 12 lifecycle events: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PostToolUseFailure`, `SubagentStart`, `SubagentStop`, `Stop`, `PreCompact`, `Notification`, `SessionEnd`. All receive JSON via stdin with session_id, transcript_path, cwd, permission_mode.

Gemini CLI `--output-format stream-json` emits JSONL with event types: `init`, `message`, `tool_use`, `tool_result`, `error`, `result`.

### Routing Decision Data Model

Each routing decision should capture:
```json
{
  "timestamp": "...",
  "task_id": "...",
  "task_summary": "Refactor auth module",
  "task_complexity": 0.85,
  "routing_factors": {
    "estimated_tokens": { "claude": 45000, "gemini": 52000 },
    "rate_limit_headroom": { "claude": 0.6, "gemini": 0.9 },
    "task_type_match": { "claude": 0.95, "gemini": 0.7 },
    "cost_estimate": { "claude": 0.45, "gemini": 0.12 },
    "historical_success": { "claude": 0.92, "gemini": 0.84 }
  },
  "selected_model": "claude",
  "routing_reason": "High complexity code refactoring",
  "outcome": { "success": true, "actual_tokens": 38000, "actual_cost": 0.38 }
}
```

---

---

## WORKING HYPOTHESIS: INVOCATION MODEL (MCP Server, Not CLI Wrapper) — Level 3 Confidence

**Date:** January 27, 2026

**User's requirement:** "I want to type `claude` normally. Not headless mode. The harness should be baked into my Claude experience."

**Working hypothesis:** The harness is an MCP server + hooks, not a CLI wrapper. This is architecturally sound but the user explicitly said "let's not double down on MCP yet." Alternatives (hooks-only, Agent SDK, Bash+file I/O) are not fully investigated.

**How it works:**
1. User configures `.mcp.json` in project root to include the harness MCP server
2. User types `claude` -- normal interactive Claude Code starts
3. Claude Code auto-spawns the harness MCP server as a stdio child process
4. Claude sees harness tools (`delegate_to_gemini`, `shared_memory`, etc.) alongside its normal tools
5. `CLAUDE.md` instructs Claude when and how to use harness tools
6. Hooks capture events and POST to dashboard server
7. Dashboard runs as a separate background process (localhost web page or CodexBar-style menu bar widget)

**Key files needed:**
- `.mcp.json` -- declares the harness MCP server
- `CLAUDE.md` -- behavioral instructions for delegation
- `.claude/settings.json` -- hook configuration
- `agent-harness/src/server.ts` -- the MCP server itself
- `.claude/hooks/post-event.py` -- hook scripts for dashboard
- `dashboard/` -- separate status display process

**MCP tool timeout warning:** Default MCP tool timeout is 60-120 seconds. Delegating to Gemini can take longer. Must set `MCP_TOOL_TIMEOUT=600000` (10 minutes) and `MAX_MCP_OUTPUT_TOKENS=50000`.

---

## DECISION: ORCHESTRATION HIERARCHY

**Date:** January 27, 2026

**User's requirement:** "Claude always sits at the very top for initial decomposition. But subtask routing to Gemini vs Claude should be dynamic."

**Decision:** Static top-level orchestrator (Claude) + dynamic subtask routing.

```
Claude Code (always the orchestrator, always the "top turtle")
  |
  ├── Decomposes task into subtasks
  ├── For each subtask, decides: handle it myself, or delegate to Gemini?
  │   Decision factors:
  │     - Task type (code → Claude, research → Gemini, multimodal → Gemini)
  │     - Context size (>100K tokens → Gemini's 1M window)
  │     - Rate limit headroom (who has capacity right now?)
  │     - Historical success (which model does this type of task better?)
  │
  ├── Subtask → Claude: handles it directly using native tools
  └── Subtask → Gemini: calls delegate_to_gemini MCP tool
        → MCP server spawns gemini -p subprocess
        → Gemini executes in headless mode
        → Result returned to Claude's conversation
        → Claude synthesizes and continues
```

**Session 3 Mesh Communication Diagram:**
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

**Why not dynamic meta-orchestration (rotating who leads):**

Research found that NO production multi-agent system dynamically switches the orchestrator role. Every framework (AutoGen, CrewAI, Google ADK, LangGraph, Manus AI) uses a fixed orchestrator with dynamic worker selection. The NeurIPS 2025 "Puppeteer" paper trains a dynamic orchestrator via RL, but even it keeps the orchestrator identity fixed -- it just trains it to behave better.

For a 2-model system, the complexity of dynamic meta-orchestration creates failure surfaces (infinite delegation loops, responsibility diffusion, authority conflicts) with minimal benefit. Claude's agent loop is more mature. Let Claude lead.

**What the harness provides Gemini when delegated to:** The 21-item compensation list ensures Gemini can execute subtasks competently. But Gemini doesn't need to orchestrate -- it just needs to do its specific subtask well and return the result.

**Future (v2):** Task-type-based orchestrator selection at the FSM level. For research-heavy tasks, Gemini could lead with Claude as critic. But this is aspirational, not MVP.

---

## DECISION: CROSS-MODEL CRITIQUE POLICY

**Date:** January 27, 2026

**User's requirement:** "Reserved for when I trigger it or when complexity is enough to trigger it."

**Decision:** Cross-model critique is NOT default. It activates when:
1. User manually triggers it (e.g., `/review` command or explicit instruction)
2. The harness detects high-complexity tasks (architecture decisions, security-sensitive code, production deployments)

For routine tasks, Claude handles them and routes subtasks to Gemini without adversarial review. The cross-model critique overhead (2x tokens, 2x latency) is not worth it for simple edits.

---

## DECISION: RATE LIMIT EXHAUSTION BEHAVIOR

**Date:** January 27, 2026

**User's requirement:** "Whatever is native. Notify me, then stop. I'll decide when I come back."

**Decision:** When both providers are rate-limited:
1. Notify the user with reset timers for both providers
2. Stop processing
3. No automatic API fallback (user controls spend)

---

## COMPETITIVE LANDSCAPE REALITY CHECK (Added End of Session 1)

> **Critical finding**: The landscape is closer to our vision than earlier research suggested. Existing tools cover ~55-60% of what we described when combined. The genuinely novel gap is ~40%.

**Tools that exist and partially solve this:**
- **CCProxy** (starbased): Rule-based routing with Max subscription support. Routes by token count, thinking mode, tool use. ~65% coverage.
- **claude-code-mux**: Rust proxy with OAuth, auto-failover, 18+ providers. ~60% coverage.
- **PAL MCP**: Manual delegation + conversation threading + context revival. ~45% coverage.
- **hcom**: Hook-based inter-agent messaging between Claude and Gemini. ~35% coverage.

**What remains genuinely novel (the ~40% gap):**
1. Semantic task-type routing (by what the task IS, not metadata)
2. Persistent cross-provider memory (across sessions, with semantic search)
3. Proactive rate limit prediction (before 429, not after)
4. Cross-model critique orchestration (Team of Rivals pattern)
5. The integrated invisible experience (all above, working together)

**The honest question**: Is the 40% gap worth building for? Or should we use PAL + CCProxy + CodexBar and see if it's enough?

See RESEARCH.md § "COMPETITIVE LANDSCAPE DEEP-DIVE" for the full 30+ tool analysis.

---

## THE SOUS CHEF ANALOGY (Why This Hasn't Been Built)

The existing tools are kitchen equipment. Claude Code is a professional chef. Gemini is another chef with different specialties. Both have full kitchens.

| Tool | Kitchen Analogy | Why It's Not What We Need |
|------|----------------|--------------------------|
| **Claude Squad** | A pass-through window | Lets you peek at multiple chefs. Doesn't coordinate menus or share ingredients. |
| **PAL MCP** | A universal ingredient converter | Translates between recipe formats. Doesn't decide who cooks what. |
| **Claude-Flow** | A recipe book with 140 recipes | Most pages are blank. Table of contents is impressive; content is stubs. |
| **AgCluster** | A kitchen display screen | Shows what one chef is doing. No coordination, no routing. |
| **Claude Octopus** | A cooking methodology book | "Double Diamond" brainstorming for dishes. Good theory. Doesn't run a kitchen. |

**What we're building is a sous chef.** It knows which chef is better at which dish, assigns each dish to the right chef, shares prep work between kitchens, tracks who's busy, and makes sure the final meal comes out coordinated.

**Why nobody has built the sous chef yet:** Most people only have one chef (one provider). The dual-provider use case is emerging right now (Jan 2026) as rate limits push people to combine Claude + Gemini. The individual kitchen tools exist. The intelligent coordinator doesn't.

Claude Octopus is conceptually closest -- it has the right idea about multi-model coordination. But it's a methodology framework, not infrastructure. It describes HOW to think about multi-model work without providing the routing engine, compensation layer, shared memory, or monitoring that makes it actually function.

---

## VISUAL INTERFACE REFLECTION

**Date:** January 27, 2026

### What Visual Orchestrators Actually Do

Studied: Conductor Build, AgCluster, @omarsar0's orchestrator, Rivet, n8n, Langflow, Flowise, Langfuse, LangSmith.

**Key finding:** Visual orchestration tools for coding agents primarily add THREE things CLI lacks:
1. **Multi-agent overview on one screen** (Conductor: see 5 agents' progress simultaneously)
2. **Diff-first review across parallel branches** (Conductor: all diffs on one screen)
3. **Post-hoc execution traces** (Langfuse: hierarchical span views with latency/token overlays)

Visual workflow designers (Rivet, n8n, Langflow) solve a DIFFERENT problem -- node-based pipeline construction. Not relevant for coding agents.

### What We Should Steal

Even for a CLI-native user, these visual elements provide genuine value:
- **Agent status with health indicators** (red/yellow/green per agent)
- **Execution trace trees** (see the SHAPE of execution, not just text)
- **Cost/token overlay** (real-time per-agent spend)
- **Timeline/Gantt view** of agent activity (where are the bottlenecks?)
- **Aggregated diff view** from multiple agents

### What Visual Interfaces Do WRONG

- **Break flow state** -- terminal → browser → back = two context switches per check (15-25 min refocus penalty each)
- **Over-engineered** -- too many clicks for power users. `cs new --name auth --prompt "..."` is faster than a GUI wizard
- **Try to replace the IDE** instead of complementing it
- **Fragile under real-world load** -- demo beautifully, break in production

### Our Approach: CodexBar-Style Ambient Monitoring

The user showed a screenshot of [CodexBar](https://github.com/steipete/CodexBar) -- a macOS menu bar widget showing:
- Rate limit gauges with reset timers (session + weekly)
- Cost tracking (today + 30-day)
- Multi-provider tabs (Claude, Gemini, Codex, Cursor, Copilot)

**This is the right UX for CLI-native users.** Not a full browser dashboard. A menu bar widget you glance at. Always visible, zero context-switch cost.

The dashboard should have TWO forms:
1. **Menu bar widget** (like CodexBar) -- always visible, shows rate limits + costs + active agent count
2. **Full browser page** (localhost:3200) -- opened only when you want deeper analysis (routing decisions, execution traces, historical trends)

---

## MEMORY SYSTEM: MEM0 DEEP-DIVE

**Date:** January 27, 2026

### Memory Is Architecturally Central (Session 3 Insight)

Memory serves double duty in this architecture — it's not a secondary add-on:

1. **Cross-subtask context:** Task B sees Task A's findings via shared memory, not orchestrator relay. This enables the mesh communication pattern.
2. **Cross-session persistence:** Tomorrow's session knows what today's session learned. This enables continuity across rate limit resets.

The router needs memory (past routing outcomes). Subtasks need memory (sibling findings). Sessions need memory (prior decisions). Memory is load-bearing from day one. Tool choice (Mem0 / sqlite-vec / files) is still open, but the architectural role is locked.

### What Mem0 Self-Hosted Actually Is

Mem0 is primarily a **Python library** that uses an LLM to extract facts from conversations and store them in a vector database. It is NOT a standalone server by default, but CAN be deployed as a REST API.

### Architecture Options

| Setup | Components | RAM | Complexity |
|-------|-----------|-----|------------|
| **Library mode** | Python in-process + Qdrant embedded + SQLite history | 150-500 MB | Low |
| **REST API (Docker)** | FastAPI on port 8000 + Qdrant | 400 MB - 1 GB | Medium |
| **OpenMemory MCP** | Qdrant + MCP API + Next.js UI | 500 MB - 1.2 GB | Medium |
| **Full stack + Neo4j** | Above + Neo4j + Postgres | 2-3 GB | High |
| **Fully local (Ollama)** | Above + local LLM | 6+ GB | Very High |

### Critical Dependencies

- **Requires an LLM** for fact extraction (default: OpenAI `gpt-4.1-nano`, costs fractions of a cent per operation)
- **Requires an embedding model** (default: OpenAI `text-embedding-3-small`)
- **Vector store** defaults to embedded Qdrant (on-disk, no separate process needed)
- **History** stored in SQLite at `~/.mem0/history.db`
- **Can run fully local** with Ollama (but adds 5-6GB+ for the LLM model)

### Multi-Agent Access Pattern

Both Claude and Gemini wrappers can read/write to the same Mem0 instance using scoping:
```
user_id = "jane"           # The human developer
agent_id = "claude_code"   # or "gemini_cli"
run_id = "session_123"     # Specific session
```
Search by `user_id` only = shared memory. Search by `user_id` + `agent_id` = agent-scoped memory.

### Simpler Alternatives

| Option | RAM | API Keys? | Auto-Dedup? | Semantic Search? |
|--------|-----|-----------|-------------|-----------------|
| **Mem0 + ChromaDB** | 150-400 MB | Yes (LLM) | Yes | Yes |
| **ChromaDB alone** | 90-200 MB | No (built-in embeddings) | No | Yes |
| **sqlite-vec** | 30-50 MB | BYO embeddings | No | Yes |
| **File-based (AGENTS.md)** | 0 | No | No | No |

**Recommendation not finalized.** User wants more research before committing. Start simple (file-based), migrate to Mem0 or ChromaDB if needed.

---

## DEPENDENCY STACK (Researched)

### Core Dependencies (TypeScript)

| Package | Purpose | Size |
|---------|---------|------|
| `@modelcontextprotocol/sdk` | MCP server + client | ~200KB |
| `zod` | Schema validation (MCP peer dep) | ~60KB |
| `execa` | Subprocess management (Gemini CLI spawning) | ~50KB |
| `p-queue` | Concurrency control | ~10KB |
| `tree-kill` | Process tree cleanup | ~5KB |
| `commander` | CLI argument parsing (for dashboard server) | ~50KB |
| `better-sqlite3` | SQLite driver | Native addon |
| `drizzle-orm` | Type-safe SQL | ~100KB |
| `hono` | HTTP server + SSE (dashboard) | ~14KB |

### Nice-to-Have (v2)

| Package | Purpose |
|---------|---------|
| `ink` | React-based terminal UI |
| `recharts` | Dashboard charts |
| `react-flow` | Routing visualization |

### What NOT to Use

- **RxJS** -- backpressure issues with subprocess streams
- **PM2** -- overkill for personal tool
- **Prisma** -- too heavy for SQLite
- **Redis/NATS** -- distributed pub/sub unnecessary for single process
- **Electron** -- 200-400MB for a monitoring window is absurd
- **Jest** -- inferior TypeScript DX vs vitest

---

## CRITICAL GAP ANALYSIS (61 Gaps Identified)

A comprehensive architecture review identified 61 gaps: **13 block implementation**, **38 complicate it**, **10 are nice-to-have**.

### Blockers (Must Resolve Before Implementation)

| # | Gap | Category |
|---|-----|----------|
| 1 | HANDOFF.md describes dual-Claude; ARCHITECTURE.md describes Claude+Gemini. Need to formally retire the old architecture in HANDOFF.md. | Contradiction |
| 2 | Orchestrator identity (SDK vs FSM vs ADK vs MCP server) -- NOW PARTIALLY RESOLVED (MCP server approach) | Contradiction |
| 3 | Invocation mechanism -- NOW RESOLVED (type `claude` normally, harness is MCP + hooks) | UX |
| 4 | Dual provider rate limit exhaustion -- NOW RESOLVED (notify and stop) | Failure |
| 5 | Mid-task failure and recovery (partially written files, checkpointing) | Failure |
| 6 | Harness process crash (orphan cleanup, signal handling) | Failure |
| 7 | Shared memory technology not selected (Mem0 vs ChromaDB vs files vs SQLite) | State |
| 8 | Ephemeral vs persistent state boundary not defined | State |
| 9 | Credential storage not designed | Security |
| 10 | Git state management across models (conflicting commits?) | Cross-model |
| 11 | File locking for concurrent agents | Cross-model |
| 12 | No configuration management system | Missing component |
| 13 | MVP scope not defined (which of 21 compensation items are MVP?) | Scope |

### Key Complicators (Top 10)

- Interactive vs headless tension (Claude interactive, Gemini headless)
- Plan approval UX in CLI not specified
- Gemini CLI breaking changes (pre-1.0, interfaces may change)
- Cross-model critique loop failure modes
- SQLite concurrent write contention
- Routing layer latency (how to estimate task complexity without LLM call?)
- Context window consumption by MCP tool definitions
- MCP tool timeout limits for long Gemini tasks
- Testing strategy for multi-agent systems
- Bootstrapping paradox (developing harness while using it)

### Full Gap List

Documented in research session output. Categories: Contradictions (7), UX (6), Failure Modes (7), State Management (5), Security (5), Performance (5), Cross-Model (5), Token Economics (4), Missing Components (8), Development (4), Scope (3), Edge Cases (5).

---

## THINGS WE STILL NEED TO RESEARCH (T-Shape Depth)

The user described a T-shape: broad understanding across many topics (horizontal), with deep depth on each (vertical). Here are the "notches" along the horizontal that need depth:

1. **Task decomposition quality** -- How well can Claude decompose tasks for delegation? What system prompts and orchestrator patterns exist? How does Manus AI do it? Research needed.

2. **Subtask routing intelligence** -- How does Claude know to route to Gemini vs handle itself? Heuristics? Keyword matching? Learned patterns? What signals matter?

3. **MCP server vs alternatives** -- We landed on MCP but haven't exhausted alternatives (hooks-only, Agent SDK, custom CLI). More research needed.

4. **Memory system selection** -- Mem0 researched in depth. ChromaDB and sqlite-vec are lighter. File-based is simplest. Need user testing to determine which matters.

5. **Orchestrator system prompts** -- There are many existing orchestrator/planner system prompts in the wild (Manus, Devin, Claude Code itself). Research and catalog them.

6. **Failure mode handling** -- What happens when Gemini produces bad output? When the MCP tool times out? When context is lost mid-task? Each needs a concrete strategy.

7. **Version compatibility** -- Both Claude Code and Gemini CLI auto-update. The harness depends on specific interfaces (stream-json, hooks, MCP). What's the compatibility strategy?

8. **Cost optimization** -- How do we minimize token waste on meta-work (routing, summarization, critique) vs actual task execution? What's an acceptable overhead budget?

9. **Multi-project context** -- How does the harness handle switching between projects? Per-project memory? Per-project config? Global vs local state?

10. **Security model** -- The MCP server sees all prompts and responses. Where are credentials? Is shared memory encrypted? Is the dashboard authenticated?

---

## OPEN ARCHITECTURE QUESTIONS

### Resolved This Session

1. ~~**How does the harness intercept requests?**~~ **RESOLVED.** MCP server + hooks inside interactive Claude Code. User types `claude` normally.

2. ~~**Plan approval normalization**~~ **RESOLVED.** See compensation items #1-4.

3. ~~**Subagent spawning for Gemini**~~ **RESOLVED.** MCP tool `delegate_to_gemini` spawns `gemini -p` subprocess.

4. ~~**How does the user invoke the harness?**~~ **RESOLVED.** User types `claude` as normal. Harness is baked in via `.mcp.json` + hooks + `CLAUDE.md`.

5. ~~**Cross-model critique policy**~~ **RESOLVED.** User-triggered or complexity-triggered. Not every task.

6. ~~**Rate limit exhaustion**~~ **RESOLVED.** Notify and stop. User decides when to resume.

### Still Open

7. **Shared memory format:** Mem0, ChromaDB, sqlite-vec, or file-based? User wants more research.

8. **Cross-provider rate limit detection:** How does the harness know Gemini's current usage? May need request counting.

9. **MCP server lifecycle:** Claude Code spawns MCP servers as stdio child processes on session start. They die when session ends. Dashboard server must be separate (persists across sessions).

10. **Git state across models:** When Gemini handles a subtask via MCP delegation, it runs in headless mode in the same working directory. Need file-locking strategy.

11. **MCP tool timeout configuration:** Default 60-120s is too short. Must set `MCP_TOOL_TIMEOUT=600000`. How to make this transparent to user?

12. **Implementation alternatives to MCP:** Hooks-only, Claude Agent SDK, custom slash commands. Not fully ruled out. Need more research.

13. **Task decomposition patterns:** How does Claude best decompose tasks for delegation? What orchestrator system prompts exist in the wild?

14. **MVP scope:** Which of the 21 compensation items are in MVP? Which can be deferred?

---

## NOTE ON HANDOFF.MD

HANDOFF.md describes the original dual-Claude architecture (two Claude Max 20x accounts). That architecture is **SUPERSEDED** by the hybrid Claude+Gemini approach documented in this file. HANDOFF.md remains useful for:
- The original vision and user requirements (Parts 1, 5, 6)
- The research directions list (Part 4)
- The implementation phases outline (Part 6, needs updating)

But Part 3 (Proposed Architecture) with the dual-account diagram is no longer canonical. This ARCHITECTURE.md is the current source of truth for architectural decisions.

---

*Architecture reflections last updated: January 27, 2026 (late session). This is a living document tracking thought process, design decisions, and research reflections. Raw research lives in RESEARCH.md. Original vision lives in HANDOFF.md (note: architecture section is superseded).*

*If you are a new Claude or Gemini instance picking this up: read PROJECT_CONTEXT.md first for orientation, confidence levels, and session conduct. Then this file for decisions and reflections. Then RESEARCH.md for evidence and sources. Then HANDOFF.md for original vision and requirements (note: Part 3 is superseded). The four together give full context.*
