# SESSION BRIDGE — Hybrid Claude+Gemini Agent Harness
> **Project Phase:** Deep Investigation (NOT implementation. NOT planning. Investigating.)
> **Last Updated:** January 29, 2026 (end of Session 3)
> **Documents:** 4 total — HANDOFF.md, RESEARCH.md, ARCHITECTURE.md, this file
> **Reading Order:** SESSION_3_SYNTHESIS.md (start here) → This file → ARCHITECTURE.md → RESEARCH.md (as reference)
> **Session Count:** 3 completed (Sessions 1-2: ~19 research agents, Session 3: 5 parallel deep-dive agents)

---

## THE CORE PROBLEM (Never Lose Sight of This)

**The user hits 85% of their weekly Claude Max rate limit in 1-2 days.** When that happens, they stop working. They want to keep working.

The solution: combine Claude Code (Claude Max 20x, $200/mo) with Gemini CLI (free or $20/mo) so that when one runs dry or when one is better suited for a task, the other picks up. The two providers should feel like **one system** — shared memory, shared plans, shared context. The user should never notice the seam.

In the user's own words: *"The only thing I'm really getting from two different accounts is the combined amount of inference capabilities. Everything else should be shared."*

**Everything in this project serves that core problem.** If a feature or research thread doesn't help the user keep working with shared context across providers, it's scope creep.

### What Serves the Core vs What's Extra

| Serves the Core | Why |
|-----------------|-----|
| Hybrid Claude+Gemini model | Two providers = more capacity, no collision |
| Shared memory across providers | Context doesn't reset when switching |
| Intelligent delegation (Claude → Gemini for suitable tasks) | Gets more value from combined capacity |
| Rate limit monitoring | Know when you're running low, plan accordingly |
| Cross-model critique (Team of Rivals) | Makes output from BOTH providers higher quality — more value per token |
| Seamless experience (type `claude`, harness is invisible) | No friction, no workflow disruption |

| Selective Extras (Valuable, Not Core) | Why Keep |
|---------------------------------------|----------|
| Dashboard / ambient monitoring widget | Useful for rate limit awareness and cost tracking |
| Visual interface for routing analytics | Helps understand and optimize the system over time |
| Cross-model critique (adversarial review) | The arxiv paper insight is genuinely powerful — 60% → 90% accuracy |

| Scope Creep (Deprioritize) | Why It Crept In |
|----------------------------|-----------------|
| Mobile portability, remote access | HANDOFF.md's original requirements, but not the core problem |
| 23-vertical T-shape research agenda | Investigation expanded beyond the problem space |
| Zero-code baseline testing protocol | Implementation detail, not problem understanding |
| Dependency stack evaluation | Implementation detail |
| Full notification system (push to phone) | Nice-to-have, not core |
| Enterprise features (team collaboration, SOC 2) | Not relevant for one user |

---

## QUICK START: 30-SECOND ORIENTATION

**What is this project?** A system that makes Claude Code and Gemini CLI work as one. The user types `claude` normally. Behind the scenes, a harness gives Claude the ability to delegate tasks to Gemini when Gemini is better suited, share memory across both, and monitor rate limits. The user never leaves their terminal. The harness is invisible.

**What phase are we in?** Deep investigation. No code exists. We're understanding what we want to get out of this — the experience, the outcomes, the value — before deciding how to build it. Many findings feel solid but are working hypotheses. The continuing session should deepen, challenge, and explore.

**What is your role?** You are a research partner, not an implementer. Challenge what you find. Push back on assumptions. The user values intellectual honesty, depth, and independent thinking. Keep the focus on what the user GETS from this system, not implementation mechanics.

### What's Decided vs What's Open

| Status | Topic |
|--------|-------|
| **Decided (High Confidence)** | Hybrid Claude+Gemini, not dual Claude accounts |
| **Decided (High Confidence)** | Claude is always the top-level orchestrator |
| **Decided (High Confidence)** | Composed architecture (not fork, not pure build) |
| **Decided (High Confidence)** | Gemini free tier or AI Pro ($0-20/mo), NOT Ultra ($250/mo) |
| **Decided (High Confidence)** | The harness is baked into the interactive Claude experience — user types `claude` normally |
| **Decided (Medium)** | The harness is likely MCP server + hooks + CLAUDE.md — but NOT locked in |
| **Decided (Medium)** | Cross-model critique only on complex tasks or user-triggered |
| **Wide Open** | What the user experience actually looks and feels like |
| **Wide Open** | Memory system — what shared context looks like in practice |
| **Wide Open** | How Claude decides what to delegate to Gemini |
| **Wide Open** | What a visual interface / dashboard provides on top of the CLI |
| **Wide Open** | How this differs (or doesn't) from existing tools |
| **Wide Open** | What scope is in v1 vs later |

---

## THE NARRATIVE: HOW THINKING EVOLVED

This section tells the story of how we got here. Read it to understand not just WHAT was decided, but WHY, and where the thinking is still fragile.

### Chapter 1: The Origin Problem

The user has a Claude Max 20x subscription ($200/mo) and was hitting 85% of weekly rate limits in 1-2 days. The original idea was simple: get a second Claude Max account ($400/mo total) and route between them for more capacity.

Research immediately revealed this was broken. A chain of client-side bugs (macOS Keychain namespace collision under `Claude Code-credentials`, missing `-U` flag in `security add-generic-password`, in-memory session caching) makes running two Claude accounts on one MacBook unreliable. Six GitHub issues document this. Unfixed as of January 2026.

This killed the dual-Claude approach and forced a pivot.

### Chapter 2: The Hybrid Pivot

If two Claudes don't work on one machine, what about Claude + a different provider? Gemini CLI emerged as the complement:

- Different provider = zero keychain collision risk
- 1M token context window (vs Claude's ~200K) = excellent for large codebase analysis
- Built-in Google Search grounding = research tasks
- Free tier (1,000 req/day) = $0 additional cost
- Daily rate limit reset (vs Claude's weekly) = faster recovery
- Open source (Apache 2.0) = inspectable and modifiable

But Gemini has critical gaps: no production plan mode, no native subagents, no task management CRUD, no NotebookEdit, MCP schema compatibility issues. A 21-item compensation list was developed documenting exactly what a harness must provide to bridge these gaps.

**This pivot is high confidence.** The evidence is strong from multiple independent research threads. The user has confirmed this direction.

### Chapter 3: The "What Are We Building?" Clarification

Early discussions were fuzzy about what the harness actually IS. Is it a CLI tool? A dashboard? A proxy? An SDK application?

Three realizations clarified this:

1. **The harness is invisible middleware, not a UI.** The user's experience shouldn't change. They type `claude`, they work normally. The harness works underneath.

2. **The dashboard is a secondary monitoring window, not the primary interface.** Inspired by CodexBar (a macOS menu bar widget showing rate limits and costs across providers), the monitoring should be ambient — always visible with zero interaction cost. A full browser dashboard is available for deeper analysis but never required.

3. **The user does NOT want headless mode.** This was a critical late-session realization. Earlier designs assumed the harness would spawn `claude -p` and `gemini -p` in headless mode as subprocesses. The user explicitly said: "I want to type `claude` normally, not headless. The harness should be baked into my Claude experience." This changes the architecture from a CLI wrapper to an MCP server that runs inside Claude Code's interactive session.

**The MCP server approach is medium confidence.** It makes architectural sense and has a strong reference implementation (PAL MCP, 10K stars). But we haven't explored all alternatives (hooks-only, Agent SDK, custom slash commands), and the user explicitly said "let's not double down on MCP servers yet." More investigation is needed.

### Chapter 4: The Orchestration Model

Who runs the show? Research into meta-orchestration patterns (NeurIPS 2025 papers, framework analysis, failure taxonomy) revealed:

- **No production system dynamically rotates the orchestrator role.** Every framework (AutoGen, CrewAI, Google ADK, LangGraph, Manus AI) uses a fixed orchestrator with dynamic worker selection.
- **The "Puppeteer" paper (NeurIPS 2025)** trains a dynamic orchestrator via RL, but even it keeps the orchestrator identity fixed.
- **The MAST failure taxonomy** found 79% of multi-agent failures come from specification and coordination problems. Dynamic orchestrator switching creates new failure surfaces.

The user confirmed: "Claude always sits at the very top for initial decomposition. But subtask routing to Gemini vs Claude should be dynamic."

**This is high confidence.** Claude leads. Subtasks are routed dynamically. The routing logic (whether it's FSM-based, heuristic, or Claude's own judgment via CLAUDE.md instructions) is still open.

### Chapter 5: The Fork vs Build Question

Seven fork candidates were evaluated. None were close enough:

| Project | Why Not Fork |
|---------|-------------|
| Claude Squad (5.6K stars) | AGPL license. Session manager, not orchestrator. No Gemini. |
| PAL MCP (10K stars) | Tool library, not harness. Could be a dependency, not a fork base. |
| Claude-Flow (13K stars) | 85% mock implementations per community audit. |
| AgCluster | No Gemini. Dashboard only. |
| Claude Octopus | Methodology framework, not infrastructure. Thin implementation. |

The analogy: these tools are kitchen appliances (a blender, a mixer, an oven). What we need is a sous chef — someone who knows which appliance to use for which step, coordinates the workflow, and ensures the meal comes together. The appliances exist. The sous chef doesn't.

**Composed architecture is high confidence.** Build custom orchestration. Use existing tools as dependencies and inspiration. The specific dependencies are tentative (see Dependency Stack section in ARCHITECTURE.md).

### Chapter 6: Where We Are Now

We have extensive breadth (19 research agents covered the landscape). What we need now is **depth** on each vertical. The user described this as a T-shape: broad horizontal understanding with deep vertical investigation on each topic.

The verticals needing depth (not exhaustive — part of the investigation is discovering more):
1. MCP servers and alternatives for harness implementation
2. Memory systems (Mem0, ChromaDB, sqlite-vec, file-based)
3. Task decomposition — how Claude decides what to delegate and how
4. Orchestrator system prompts — what makes a good delegation prompt
5. Visual interface — what a concrete user journey looks like
6. Dashboard design — CodexBar-style widget vs browser page vs both
7. Failure modes — what breaks and how to recover
8. Security and credential management
9. Testing strategy for multi-agent systems
10. Cost optimization — minimizing token overhead for meta-work

**We are NOT ready for implementation planning.** There are 13 identified gaps that block implementation and 38 that complicate it. Many "decisions" are really "current best guesses." The investigation must continue.

---

## DOCUMENT RELATIONSHIP MAP

### How to Read Each Document

**ARCHITECTURE.md** (~880 lines) — **The current state of thinking.** Read this for decisions, reflections, and architectural reasoning. Organized by topic (core realization, composed architecture, hybrid model, abstraction problem, Team of Rivals pattern, compensation list, dashboard design, gap analysis, open questions). This is the most actively maintained document. BUT: not everything in it is equally confident. Some sections are well-reasoned; some are sketches. The Decision Confidence Registry below clarifies which is which.

**RESEARCH.md** (~970 lines) — **The evidence base.** Read this when you need sources, data, or detailed findings to support or challenge a position. Organized by topic with 90+ sources. Three major rounds of research. Not everything is equally relevant now — some findings led to dead ends that informed thinking by elimination. Key sections: device rate limit bug (foundational), hybrid harness feasibility, Gemini CLI gap audit, dashboard patterns, MCP invocation research, meta-orchestration, dependency stack.

**HANDOFF.md** (~330 lines) — **The original vision.** Read this to check whether current architecture has drifted from user goals. Written at the start of the project. **Part 3 (Proposed Architecture) is SUPERSEDED** — it describes a dual-Claude architecture that was abandoned after the device bug discovery. Parts 1 (Vision), 5 (Requirements), and 6 (Phases) are still relevant but need updating to reflect the hybrid approach.

**SESSION_BRIDGE.md (this file)** — **The meta-layer.** How to interpret the other three. Reading guide, narrative, confidence levels, and investigation roadmap. Read first, but don't treat as a substitute for the other documents.

### Known Contradictions Between Documents

1. **HANDOFF.md says "two Claude Max accounts" ($400/mo). ARCHITECTURE.md says Claude + Gemini ($200-220/mo).** The hybrid approach supersedes the dual-Claude approach. HANDOFF.md has not been updated to reflect this.

2. **HANDOFF.md says "Claude Agent SDK as foundation." ARCHITECTURE.md says MCP server + hooks.** The SDK approach was deprioritized because we want to keep the native interactive Claude Code experience. The MCP approach integrates without replacing Claude Code. BUT this is medium confidence — alternatives are still being investigated.

3. **Early ARCHITECTURE.md sections describe "wrapping both CLI binaries" in headless mode. Later sections describe MCP server inside interactive Claude.** The MCP approach is the current direction. The headless wrapper description is partially outdated. It still applies to how Gemini is invoked (Gemini always runs headlessly as a subprocess), but Claude Code runs interactively with the user.

4. **RESEARCH.md's "Recommended Architecture" diagram shows the harness as a separate entity between the user and the CLIs. ARCHITECTURE.md's updated core realization shows it baked INTO Claude Code via MCP.** The MCP version is more current, but the research architecture is still valid as a conceptual model — it just manifests differently than initially drawn.

---

## DECISION CONFIDENCE REGISTRY

Each significant direction has a confidence level. Use this to know what to trust, what to challenge, and what to investigate.

### Level 5 — Locked (Definitional)

**Hybrid Claude+Gemini, not dual Claude**
- Why locked: Device-level keychain bug makes dual-Claude broken on one machine. Confirmed by 6+ GitHub issues, reverse engineering analysis. Not fixable by us.
- What would change it: Anthropic fixes the keychain bug AND removes rate limit circumvention ambiguity. Unlikely near-term.
- See: RESEARCH.md § "CRITICAL FINDINGS: DEVICE-LEVEL RATE LIMITING"

**Claude is the top-level orchestrator**
- Why locked: User explicitly and repeatedly confirmed. Claude's agent loop is more mature. Research confirms no production system dynamically rotates orchestrators.
- What would change it: Gemini's agent ecosystem matures to parity. Unlikely in 2026.
- See: ARCHITECTURE.md § "DECISION: ORCHESTRATION HIERARCHY"

### Level 4 — High Confidence

**Composed architecture (not fork, not pure build)**
- Why high: 7 fork candidates evaluated. None close enough. Solved problems exist (MCP SDK, CLI tools). TypeScript stack validated by multiple reference implementations.
- What would change it: A new project emerges that's 80%+ of what we need with a permissive license.
- See: ARCHITECTURE.md § "DECISION: COMPOSED ARCHITECTURE"

**Gemini free tier or AI Pro, NOT Ultra ($250/mo)**
- Why high: Ultra gives 33% more requests for 12.5x the price. Pro-model sub-quotas exhaust in 2-3 hours even on Ultra. Claude Max provides far more sustained top-model usage.
- What would change it: Google restructures Ultra pricing or dramatically increases Pro-model quotas.
- See: ARCHITECTURE.md § "GOOGLE AI SUBSCRIPTION: ULTRA IS NOT WORTH IT"

**Cross-model critique reserved for complex/user-triggered tasks**
- Why high: User confirmed. Token economics support it — 2x cost per critiqued task is wasteful for routine work.
- What would change it: Evidence that routine tasks benefit enough from critique to justify the cost.

**Rate limit exhaustion: notify and stop**
- Why high: User confirmed. Matches native behavior. No automatic spend.
- What would change it: User changes preference.

### Level 4 — High Confidence (Added Session 3)

**Semantic routing approach (RouteLLM or Aurelio)**
- Why high: Research confirmed pre-trained routers achieve 90-95% accuracy without custom training. RouteLLM generalizes to new model pairs. Aurelio provides control via user-defined routes.
- What would change it: Testing reveals these don't work for Claude+Gemini routing in practice.
- See: SESSION_3_SYNTHESIS.md § "Decision 4: Semantic Routing Without Training"

**Communication pattern (Hub-and-spoke + shared state)**
- Why high: Framework comparison (LangGraph, CrewAI, ADK, Swarm) ALL use this pattern. AgentMail was a red herring. The "sinew" question is answered.
- What would change it: Discovery that shared state creates race conditions or data corruption.
- See: SESSION_3_SYNTHESIS.md § "Decision 3: Inter-Agent Communication"

**Critique pattern (Veto, not consensus)**
- Why high: arxiv 2601.14351 explicitly found veto > democratic consensus. Implementation is simpler. Token cost is lower.
- What would change it: Evidence that consensus produces better outcomes for coding tasks specifically.
- See: SESSION_3_SYNTHESIS.md § "Decision 5: Critique Pattern"

**Visibility model (Silent CLI + dashboard)**
- Why high: User explicitly clarified "baked in" means MCP server inside interactive Claude Code. Dashboard is for when you WANT visibility.
- What would change it: User testing reveals they want MORE or LESS visibility than designed.
- See: SESSION_3_SYNTHESIS.md § "Decision 6: Visibility Model"

### Level 3 — Working Hypothesis (Challenge These)

**MCP server as the harness mechanism**
- Current position: The harness runs as an MCP server inside interactive Claude Code. Claude Code auto-spawns it. It provides delegation tools.
- Why this level: Architecturally sound. Strong reference implementation (PAL MCP, 10K stars). But user explicitly said "let's not double down on MCP yet." Alternatives (hooks-only, Agent SDK, custom CLI) not fully investigated.
- What would change it: Discovery that MCP has fundamental limitations for our use case (e.g., tool timeout issues, context consumption, inability to maintain state across sessions).
- **INVESTIGATE FURTHER.** This is the single most impactful open question.
- See: ARCHITECTURE.md § "DECISION: INVOCATION MODEL"

**TypeScript as the implementation language**
- Current position: TypeScript for the MCP server, dashboard, and tooling.
- Why this level: Validated by reference implementations (claude-flow, disler's observability). Claude Code itself is TypeScript. But we haven't evaluated Go (Claude Squad), Python (PAL MCP), or Rust alternatives.
- What would change it: Performance requirements favor compiled languages. Or MCP SDK limitations in TypeScript.

**Dashboard as CodexBar-style widget + localhost browser page**
- Current position: Ambient menu bar widget for rate limits/costs. Full browser page for deep analysis.
- Why this level: User liked the CodexBar screenshot. But visual interface design is wide open — we haven't designed concrete user journeys or screens.
- What would change it: User testing reveals different monitoring needs than anticipated.

### Level 2 — Tentative (Alternatives Welcome)

**Specific MCP tools to expose**
- Current list: delegate_to_gemini, shared_memory_read/write, check_rate_limits, route_task
- Why tentative: These are sketches. Haven't been validated against real workflows. Missing many tools we haven't thought of.
- What would change it: User journey mapping. Actual prototyping.

**Specific dependency stack**
- Current: execa, p-queue, commander, hono, better-sqlite3, drizzle-orm, MCP SDK, vitest
- Why tentative: Researched but not tested together. Some choices may not compose well.
- What would change it: Prototyping. Discovering compatibility issues.

**21-item compensation list prioritization**
- Current: All 21 items documented. No prioritization for MVP.
- Why tentative: We know WHAT needs building but not in WHAT ORDER. MVP scope undefined.
- What would change it: User decides what matters first. Dependency analysis of the 21 items.

### Level 1 — Open (Needs Investigation)

**Memory system** — Mem0, ChromaDB, sqlite-vec, or file-based? Deep research done on Mem0. Others need equal depth. User interested but uncommitted.

**Task decomposition strategy** — How does Claude decide what to delegate vs handle itself? No orchestrator system prompts researched yet. How granular is decomposition?

**Visual interface specifics** — What exactly does a user see? What are the user journeys? What does the dashboard provide on top of the MCP server? What visual information adds value that CLI can't provide?

**Failure mode handling** — Mid-task failures, provider crashes, timeout cascades, conflicting edits. 61 gaps identified but no solutions designed.

**Security model** — Credential storage, shared memory encryption, dashboard authentication, process argument exposure.

**MVP scope** — Which features are in v1? What's the minimum that makes this useful?

### Level 0 — Unknown Unknowns (We Haven't Even Asked)

- How does the harness behave across different projects? Multi-project memory isolation?
- What happens when Claude Code or Gemini CLI auto-updates and breaks the interface?
- How do you develop/test the harness while using Claude Code as your development tool?
- What about offline/intermittent connectivity behavior?
- What about Gemini's sandbox model interaction with harness tool approval?
- How do multiple concurrent Claude Code sessions interact with a shared harness?

---

## RESEARCH DEBT: WHAT WE HAVEN'T VALIDATED

### Assumptions Under Load

These are things we're treating as true but haven't stress-tested:

1. **"MCP tool timeout can be extended to 10 minutes."** We've seen documentation suggesting `MCP_TOOL_TIMEOUT` is configurable. But has anyone actually run a 5-minute MCP tool call in production Claude Code? What happens to the user experience during a long tool call? Does Claude Code show a spinner? Can the user cancel?

2. **"Claude will reliably follow CLAUDE.md delegation instructions."** We assume that CLAUDE.md instructions like "when task exceeds 100K tokens, use delegate_to_gemini" will be consistently followed. But Claude can ignore instructions. How reliable is this in practice?

3. **"Gemini's headless output is parseable and stable."** We're building on `--output-format stream-json`. This is a pre-1.0 CLI. The JSONL schema could change between versions with no guarantee of backwards compatibility.

4. **"The sous chef gap is real."** We claim nobody has built what we're building. But maybe they have and we missed it in research. Or maybe nobody has built it because it's not actually useful — people prefer simple tools over unified orchestration.

5. **"Token overhead for harness meta-work is acceptable."** Every delegation call, memory read, rate limit check, and critique consumes tokens. We haven't modeled the total overhead or set a budget.

### The Skeptic's Challenges

If a smart, well-intentioned skeptic read our architecture, they would ask:

1. **"Why not just use PAL MCP directly?"** It already bridges Claude and Gemini. 10K stars. Why build a new harness instead of configuring PAL?
   - *Our answer:* PAL provides tools but doesn't orchestrate. It's a library, not a sous chef. But this answer needs stress-testing — maybe "just tools" is enough.

2. **"Is this over-engineered for one person?"** A 21-item compensation list, an FSM router, a dashboard, shared memory, cross-model critique... for one developer's workflow?
   - *Our answer:* Start simple, evolve. MVP won't include everything. But the scope IS ambitious and we need to be honest about that.

3. **"What if Claude Code adds native Gemini integration?"** Anthropic could build this functionality natively, making the harness obsolete.
   - *Our answer:* Possible but unlikely in 2026. Anthropic competes with Google. They're more likely to improve Claude than integrate Gemini.

4. **"You're planning to have Claude delegate to Gemini via MCP. But MCP tools can't access conversation history, can't modify the system prompt, and have a 25K token output limit. Aren't these fundamental limitations?"**
   - *Our answer:* Partially. We need to investigate how severe these limitations are in practice. This is Level 3 confidence for a reason.

5. **"The user said they want everything baked into the interactive Claude experience. But the most ambitious features (cross-model critique, parallel subagent execution, visual routing analytics) seem to need a standalone orchestrator, not an MCP server. Are you trying to fit a large system into a small interface?"**
   - *Our answer:* This tension is real and unresolved. The MCP server may not be sufficient for all features. Hybrid approaches (MCP for delegation + separate process for dashboard + hooks for monitoring) may be necessary. More investigation needed.

---

## BEFORE BUILDING ANYTHING: TRY THE ZERO-CODE BASELINE

This is important. Before writing a single line of code, the user should try this combination that exists TODAY:

1. **Install CCProxy** (starbased, rule-based routing) — automatic routing with Max subscription support, ~65% coverage alone
2. **Install PAL MCP Server** (10K stars, Python) — gives Claude Code the ability to delegate to Gemini CLI via `clink` tool
3. **Install CodexBar** (macOS menu bar app) — shows rate limits and costs for Claude + Gemini in the menu bar
4. **Write CLAUDE.md delegation rules** — instruct Claude when to use PAL's `clink` tool for Gemini delegation

**This gives you:** Rule-based routing, Gemini delegation from Claude Code, rate limit monitoring, and behavioral steering. For free. Today. No custom code.

**This does NOT give you:** Intelligent routing (you decide when to delegate, not the system), shared memory across models, cross-model critique, unified dashboard with routing analytics, Gemini gap compensation (plan mode, subagents), or cost comparison tracking.

**Why this matters:** If the zero-code baseline is "good enough," the user saves months of work. If it's insufficient, the user will know EXACTLY what's missing because they'll have felt the gaps in their daily workflow. Either outcome is valuable. The next session should help the user set this up and evaluate it.

---

## THE QUESTION THE USER KEEPS ASKING: "WHY HASN'T ANYONE BUILT THIS?"

This matters because the user has a gut feeling that something out there already does what they want. Let's be direct.

**Your gut is both right and wrong.**

Your gut is RIGHT that all the pieces exist. MCP delegation exists (PAL MCP). Multi-model spawning exists (gemini-cli-orchestrator). Rate limit monitoring exists (CodexBar). Agent dashboards exist (Langfuse, disler's hooks-observability). Shared memory exists (Mem0). Cross-model critique exists (arxiv paper pattern). Task routing exists (LLM-Use router). Every piece is real, open source, and works.

Your gut is WRONG that someone has assembled these into a unified system that works inside your Claude Code terminal. That assembly hasn't happened for four specific reasons:

1. **The dual-provider use case is genuinely new (January 2026).** Gemini CLI reached stable in late 2025. Claude's rate limits became acutely painful in late 2025. The "I'm rate-limited on Claude + Gemini CLI is good enough to help" combination is a 2025-2026 phenomenon. PAL MCP, gemini-cli-orchestrator, claude-code-router are ALL less than 6 months old.

2. **Most people solve rate limits by waiting.** The limit resets weekly. Most developers don't hit it. Those who do either wait, work on something else, or use a lighter model. "I need CONTINUOUS high-throughput agent work across providers" is a power-user need in a power-user niche.

3. **Cross-model orchestration is genuinely hard.** The 21-item compensation list proves this. Bridging plan mode, subagents, task management, MCP schema differences, tool name mapping, context budget management, and behavioral parity between two different agent systems is substantial engineering.

4. **The "invisible middleware inside Claude Code" constraint rules out 90% of solutions.** Most orchestrators REPLACE the CLI experience (Conductor, Claude Squad, ADK, CrewAI). You want to ENHANCE it while keeping it. This requires deep integration with Claude Code internals (MCP + hooks) AND Gemini CLI's headless mode. That's a narrow design space.

### How This Differs From Existing Tools (Feature Matrix)

| Feature | Our Harness | PAL MCP | Conductor Build | Claude Octopus | Claude Squad |
|---------|-------------|---------|-----------------|----------------|--------------|
| **What it is** | Invisible orchestration of Claude + Gemini | MCP tool library for multi-CLI spawning | Parallel Claude instances with visual oversight | Multi-AI brainstorming methodology | tmux session manager |
| **User types** | `claude` (normal, unchanged) | `claude` (normal, MCP tools available) | Conductor app (separate GUI) | `claude` (with special prompts) | `cs new` (separate TUI) |
| **Multi-model?** | Yes (Claude + Gemini, architected) | Yes (Claude, Gemini, Codex) | No (Claude only) | Yes (conceptual) | No (Claude only) |
| **Intelligent routing?** | Yes (task-type, rate limits, context) | No (user decides) | No (all Claude) | Conceptual only | No |
| **Rate limit awareness?** | Yes (cross-provider, failover) | No | No | No | No |
| **Shared memory?** | Yes (designed, TBD) | Limited (context sharing) | No (isolated worktrees) | No | No |
| **Cross-model critique?** | Yes (arxiv-informed veto) | No | No | Conceptual | No |
| **Dashboard?** | Yes (widget + browser) | No | Yes (Electron app) | No | TUI only |
| **Preserves Claude Code UX?** | Yes (MCP is invisible) | Yes (MCP is invisible) | No (separate app) | Partially | No (separate TUI) |
| **Implementation maturity** | None (design phase) | Production (10K stars) | Production (YC) | Thin | Production (5.6K stars) |

**The critical differentiator:** PAL MCP gives you a **tool** — you call `clink` and it runs Gemini. Our harness gives you a **system** — it decides WHEN to call Gemini, monitors both providers, shares context between them, critiques their output, and surfaces analytics. PAL is a Swiss Army knife. We're building a workshop foreman.

If the user's need is "sometimes use Gemini from Claude Code," **PAL MCP solves that today with zero custom code.** If the need is "intelligently coordinate Claude and Gemini as a unified system," PAL is a component in our harness, not a replacement. The zero-code baseline test (above) will clarify which need is real.

---

## WHAT THE VISUAL INTERFACE ACTUALLY LOOKS LIKE (Concrete Scenarios)

The user keeps asking "what would a visual interface entail?" Here are five real scenarios:

### Scenario 1: Normal coding session (80% of time)
You see: your terminal running Claude Code. In the macOS menu bar, a small icon (like CodexBar) shows a green dot. Click it: "Claude Max: 42% weekly, resets 4d 11h. Gemini: 12% daily, resets 8h. Session cost: $0.00. Active agents: 1." You never open the browser dashboard. You glance at the green dot and keep working.

### Scenario 2: Complex task with delegation
Claude calls `delegate_to_gemini`. Menu bar shows "2 agents active." If you open localhost:3200 in a browser, you see: (Left) timeline with Claude's thread and Gemini's thread running in parallel. (Center) live activity: "Claude: Edit auth.ts... Gemini: Analyzing 47 files..." (Right) routing card: "Task: Analyze auth patterns. Routed to Gemini. Reason: 47 files, ~180K tokens, exceeds Claude's window." You check once, go back to terminal.

### Scenario 3: Rate limit warning
Menu bar turns yellow. Click: "Claude Max: 82% weekly. At current rate, limit in ~3h. Gemini: 45% daily." Browser adds: "Recommendation: Route research tasks to Gemini to preserve Claude for code generation."

### Scenario 4: Both providers exhausted
Menu bar turns red. "Claude: Weekly limit reached. Resets Thu 3 PM. Gemini: Daily limit reached. Resets midnight." Browser: "3 tasks queued. Resume: midnight (Gemini first)." You close the laptop.

### Scenario 5: Cross-model critique
You trigger `/review` on security code. Claude delegates to Gemini for critique. Gemini returns: "2 issues — JWT expiration not enforced, missing rate limiting on login." Claude incorporates feedback. Browser shows: critique timeline, diff between first draft and post-critique, "2 issues caught by cross-model review."

**Key insight:** The visual interface is TWO things: (A) an ambient status indicator (menu bar, always visible, zero interaction cost) answering "is everything healthy?", and (B) a deep-analysis browser page answering "what happened and why?" when you want to investigate. Most sessions you only see (A).

---

## THE MCP QUESTION: ALL SIX OPTIONS (Not Just MCP)

SESSION_BRIDGE.md previously framed this as "MCP or not?" The real question is: "What does each mechanism do best, and how do they compose?"

| Option | Mechanism | Returns results to Claude? | Sees all events? | Persists across sessions? | Preserves interactive UX? |
|--------|-----------|---------------------------|-------------------|---------------------------|---------------------------|
| **1. MCP Server** | Claude spawns server, calls tools | **Yes** | No (only its own tools) | No (dies with session) | **Yes** |
| **2. Hooks Only** | Scripts fire on lifecycle events | **No** (observe only) | **Yes** (all tool calls) | No (per-session) | **Yes** |
| **3. Slash Commands** | User types `/delegate-gemini` | Yes (via Bash) | No | No | Partially (manual) |
| **4. Agent SDK App** | Standalone Python/TS application | Yes (full control) | Yes (it IS the system) | Yes | **No** (replaces Claude Code) |
| **5. Google ADK App** | Standalone with Gemini + LiteLLM | Yes (full control) | Yes | Yes | **No** (replaces Claude Code) |
| **6. Hybrid (MCP + Hooks + CLAUDE.md + Background Process)** | MCP for tools, hooks for events, separate process for persistence | **Yes** (via MCP) | **Yes** (via hooks) | **Yes** (via background process) | **Yes** |

**Option 6 is the actual answer.** Not "MCP server" alone. Each mechanism handles what it does best:
- **MCP server** handles: delegation tools that RETURN results to Claude's conversation
- **Hooks** handle: event capture for dashboard, monitoring side-effects
- **CLAUDE.md** handles: behavioral steering, delegation heuristics
- **Background process** handles: dashboard, persistent state, cross-session memory

**The unexplored alternative:** Could hooks alone work? A hook could spawn Gemini, write results to a file, and CLAUDE.md could instruct Claude to "check `.harness/results/` after delegation." Clunky but no MCP server needed. This hasn't been investigated.

**Key question for next session:** "What SPECIFICALLY does MCP provide over Bash + file I/O? When Claude delegates, it could just run `gemini -p 'task' > /tmp/result.md` via Bash and then Read the file. Why is an MCP server better than that?"

---

## PROVOCATIVE QUESTIONS FOR THE NEXT SESSION

These are designed to drive productive investigation, not confirm existing thinking. The next session should tackle these BEFORE doing more research.

### "Do You Actually Need This?" Questions

1. **"If you install PAL MCP + CCProxy + CodexBar + good CLAUDE.md rules today, you get ~55-60% of the value with zero custom code. What specific workflow fails that makes the other ~40% worth months of building?"**

2. **"You hit 85% of weekly limits in 1-2 days. But Claude Max 20x gives 240-480 hours/week of Sonnet 4. Are you hitting the weekly token cap, or the 5-hour session cap that resets after a break? These are different problems with different solutions."**

3. **"The harness has one user: you. A 21-item compensation list, FSM router, cross-model critique, shared memory, and dashboard for one person's workflow is a significant investment. What's the MINIMUM that makes your Wednesday morning better? Not the vision — the Wednesday improvement."**

### "MCP vs Everything Else" Questions

4. **"When Claude calls delegate_to_gemini via MCP, the server spawns gemini -p and returns results. But couldn't you just write a Bash command: `gemini -p 'analyze these files' > result.md` and Read the file? Why do you need an MCP server when Bash + file I/O achieves the same data flow?"**

5. **"PAL MCP's clink tool already works. gemini-cli-orchestrator already works. What SPECIFICALLY does a custom MCP server add? If the answer is 'routing + memory,' could those be SEPARATE MCP servers alongside PAL instead of replacing it?"**

6. **"MCP tools can't access conversation history. Claude must serialize ALL relevant context into tool parameters. For a complex delegation (debug this module across 12 files), that's 50K+ tokens as a tool parameter. Is MCP the right mechanism for large-context delegation?"**

### "Is the Sous Chef Real?" Questions

7. **"What if the right answer is a RECIPE BOOK (CLAUDE.md instructions), not a coordinator (MCP server)? What if writing better delegation rules achieves 90% of the value without infrastructure?"**

8. **"Conductor Build runs 5 parallel Claude instances. Our harness runs 1 Claude + 1 Gemini. Which produces more useful output per hour? Has anyone compared parallel-same-model vs serial-different-model?"**

9. **"The FSM router classifies tasks WITHOUT an LLM call to avoid token overhead. How do you classify 'refactor the auth module to use JWT' as code (Claude) vs research (Gemini) using only heuristics? What signals are available? Be concrete."**

### "Challenge the Architecture" Questions

10. **"You want invisible middleware. But invisible systems are the hardest to debug, understand when they break, and improve. Would you prefer a slightly-visible system (TUI sidebar showing routing decisions) over a truly invisible one?"**

11. **"Cross-model critique doubles cost and latency. The paper shows 60%→90% accuracy on benchmarks. For YOUR coding tasks, what's the expected improvement? Is 2x cost worth it when Claude alone is already 'good enough' for most work?"**

12. **"The routing decision model includes 'historical success rates.' But your first 100 tasks have no history. What's the cold-start strategy?"**

---

## INTER-DOCUMENT GAPS (Things That Don't Add Up)

The next session should resolve these contradictions:

1. **HANDOFF.md says "portable, works from road, mobile."** ARCHITECTURE.md designs everything as localhost-only. Either descope portability or design for remote access.

2. **HANDOFF.md says "single interface to manage everything."** ARCHITECTURE.md describes three interfaces (terminal + menu bar + browser page). Reconcile.

3. **HANDOFF.md says subagents are "load balanced" across accounts.** ARCHITECTURE.md uses "routing" (specialized backends) not "load balancing" (interchangeable backends). These are architecturally different. Which do we mean?

4. **ARCHITECTURE.md says "FSM routing, not LLM routing" but the routing model includes probability scores and estimated tokens.** How do you compute task_complexity: 0.85 without an LLM call? This is stated as a principle with no implementation research.

5. **ARCHITECTURE.md says dashboard is "optional" and "secondary" but designs 5 panels including an interactive plan approval queue.** A plan approval queue is primary for that workflow.

6. **One-provider exhaustion is unaddressed.** "Both exhausted → notify and stop" is decided. But what happens when ONLY Claude is exhausted and Gemini has capacity? This is the most common real-world scenario. Does the harness auto-route everything to Gemini? Does it ask the user? This gap is more practically important than the both-exhausted case.

7. ~~**Dashboard port inconsistency.**~~ **RESOLVED.** ARCHITECTURE.md now consistently uses localhost:3200.

8. ~~**Hook event count.**~~ **RESOLVED.** Both ARCHITECTURE.md and RESEARCH.md now correctly say 12 events.

---

## COMPLETE T-SHAPE DEPTH INVENTORY (23 Verticals)

The investigation has breadth. What it needs is depth on EACH of these verticals. The next session should pick 2-3 and go deep, not skim all 23.

### Previously Named (10)
1. MCP servers and alternatives — the hybrid architecture
2. Memory systems — Mem0, ChromaDB, sqlite-vec, file-based
3. Task decomposition — how Claude decides what to delegate
4. Orchestrator system prompts — what CLAUDE.md instructions look like
5. Visual interface — concrete user journeys
6. Dashboard design — CodexBar widget vs browser page
7. Failure modes — what breaks and how to recover
8. Security and credential management
9. Testing strategy for multi-agent systems
10. Cost optimization — token overhead budget

### Previously Unnamed (13)
11. **FSM router design** — What are the states, transitions, classification signals? How do you route without an LLM call?
12. **Gemini subprocess lifecycle** — spawn, monitor, timeout, cancel, cleanup. What happens when gemini -p hangs?
13. **MCP schema translation** — Resolving $defs/$ref, multi-type arrays, tool name sanitization. A distinct engineering problem.
14. **Context engineering for delegation** — How much context passes from Claude to Gemini? Full history? Summary? Files only? What format?
15. **Plan mode normalization** — Intercepting Gemini tool calls, enforcing read-only, translating plan format. Compensation items #1-4.
16. **Cross-session state persistence** — What survives when Claude's session ends? SQLite schema, retention policy, session identity.
17. **Version compatibility** — Both CLIs auto-update. stream-json format could change. How does the harness stay compatible?
18. **Token economics modeling** — How many tokens for meta-work (routing, memory reads, delegation overhead)? What's acceptable overhead?
19. **The bootstrapping paradox** — Developing the harness using the tool the harness enhances. Concrete development workflow needed.
20. **AGENTS.md and multi-config management** — How do CLAUDE.md, GEMINI.md, AGENTS.md relate? What goes where?
21. **Git state across models** — Branching, commit attribution, merge conflicts when both models edit files.
22. **Notification and alerting** — HANDOFF.md requires "notifications when work completes." Not designed. Push to phone?
23. **Zero-code baseline evaluation** — Try PAL MCP + CodexBar + CLAUDE.md before building anything. What works? What's missing?

---

## INVESTIGATION ROADMAP: WHERE TO GO NEXT

### FIRST 10 MINUTES OF A NEW SESSION

If you are a fresh Claude instance reading this for the first time:

1. **Read this file completely** — understand the project, phase, and conduct expectations
2. **Read ARCHITECTURE.md** — understand technical decisions and confidence levels
3. **Ask the user: "What do you want to focus on today?"** — don't assume. The user may have new priorities, new findings, or new questions since the last session
4. **Review the open questions below** — be ready to discuss any of them. The user's primary need for the next session is having their open questions answered with depth and honesty

Do NOT: start researching without asking, propose implementation, or make document edits without explaining first.

### Immediate (Start the Next Session With These)

1. **Try the zero-code baseline** — Install PAL MCP + CodexBar. Write CLAUDE.md delegation rules. Use it for a day. Document what works and what doesn't. This grounds everything in reality.

2. **Resolve the MCP vs Hybrid question** — Not "MCP or not?" but "What does each mechanism handle?" Flesh out Option 6 (hybrid). Answer: "Why can't Bash + file I/O replace MCP for delegation?"

3. **Map 3 concrete user journeys** — Walk through real daily tasks. Where does delegation happen? What does the user see? What breaks?

### Depth (After Immediate Threads)

4. Task decomposition and orchestrator prompts (vertical #3)
5. Memory system selection with criteria (vertical #2)
6. FSM router design — concrete states and transitions (vertical #11)
7. Failure mode playbook for the 13 blockers (vertical #7)
8. MVP scope — which features first? (vertical #7 in original list)

### Horizon (Keep in Mind)

9-23. The remaining verticals from the T-shape inventory above.

### Anti-Goals: What NOT to Do

- **Do not start writing code.** Investigation phase.
- **Do not finalize the MCP approach.** The hybrid option is under-explored.
- **Do not skip the zero-code baseline.** Real experience > hypothetical architecture.
- **Do not assume the 21-item list is complete or correctly prioritized.**
- **Do not rush to decisions.** The user values thoroughness over speed. Sitting with uncertainty is fine.
- **Do not treat ARCHITECTURE.md as settled.** It's a working draft of working hypotheses.

---

## SESSION CONDUCT GUIDE

### Mindset

You are a research partner continuing an investigation, not an implementer starting a build. The user has strong intuitions but also genuine uncertainty. They want you to:

- **Think independently.** Don't just agree with what you find in these documents. Challenge it.
- **Go deep, not broad.** We have breadth from 19 research agents. What we need is depth on each vertical.
- **Show your reasoning.** The user values seeing the thought process, not just the conclusion.
- **Be honest about uncertainty.** "I'm not sure because..." is better than false confidence.
- **Ask questions.** Use the AskUserQuestion tool liberally to check alignment.
- **Explain before editing.** ALWAYS talk through what you plan to change and WHY before modifying any document. The user wants to understand your reasoning and approve the direction before you touch files. This is a hard rule, stated explicitly twice.

### Interaction Style the User Values

- Thoroughness and exhaustiveness — the user repeatedly asked for things to be tracked, documented, and reflected upon
- Intellectual honesty — "don't speak out of your ass" (direct quote from early session)
- Pushback when warranted — the user wants to be challenged, not agreed with
- Narrative reasoning — show the train of thought, not just the destination
- Sub-agents for breadth — the user frequently asked for sub-agents to "think outside the box" and explore dimensions they weren't considering

### What the User Does NOT Value

- Premature convergence — "we're not even fully at the stage where we're planning implementation yet"
- Rushing to build — the user explicitly said investigation comes first
- Surface-level analysis — "there's so many nuances, so much depth we could go into"
- False confidence — the user wants to know what's uncertain
- Losing context — "make sure you're tracking things" was said 4+ times
- Making changes without explaining first — the user explicitly said "before you make changes, talk me through why you're making the changes before you" — this applies to ALL document and code edits

### Document Maintenance

Every session should:
1. Update ARCHITECTURE.md with new decisions, reflections, and pivots
2. Update RESEARCH.md with new findings and sources
3. Update this file's Decision Confidence Registry if confidence levels change
4. Update this file's Investigation Roadmap (mark completed, add new threads)
5. Add a session log entry to the appendix below

### If You're Running Low on Context

Before the session ends, update this file. At minimum: add a session log entry, update any changed confidence levels, and note where the investigation stopped.

---

## GLOSSARY

| Term | Meaning in This Project |
|------|------------------------|
| **Harness** | The orchestration system we're building. Not a UI, not a CLI tool — invisible middleware that coordinates Claude and Gemini. |
| **Compensation list** | The 21 capabilities the harness must provide to bridge Gemini's gaps vs Claude Code. See ARCHITECTURE.md. |
| **FSM routing** | Finite State Machine-based routing. Deterministic code decides task routing, not LLM prompts. From the arxiv paper. |
| **Team of Rivals** | Cross-model critique pattern from arxiv 2601.14351. Models with different failure modes catch each other's errors. |
| **Composed architecture** | Our approach: build custom orchestration, use existing tools as dependencies. Not fork, not pure build. |
| **CodexBar** | macOS menu bar widget (github.com/steipete/CodexBar) showing rate limits and costs. Inspiration for our ambient monitoring. |
| **PAL MCP** | Provider Abstraction Layer MCP Server (10K stars). Reference implementation of the MCP delegation pattern. |
| **Sous chef analogy** | Kitchen analogy explaining why existing tools don't solve our problem. They're appliances. We need a coordinator. |
| **Headless mode** | Running CLI tools with `-p` flag for programmatic use. Gemini always runs headlessly when delegated to. Claude does NOT (user wants interactive experience). |
| **AGENTS.md** | Emerging standard config file that both Claude Code and Gemini CLI can read. Potential shared config mechanism. |
| **Device bug** | macOS Keychain collision bug making dual-Claude accounts unreliable on one machine. The trigger for the hybrid pivot. |

---

## SESSION LOG

### Session 1 — January 27, 2026

**Context:** Continued from a claude.ai conversation. User provided HANDOFF.md as initial context.

**What happened:**
- Deployed 19 research agents across 3 rounds covering: ecosystem survey, memory systems, multi-account routing, agent SDKs, dashboards, frontier research, device rate limits, Claude+Gemini hybrid, Gemini CLI capabilities, Ultra vs Pro, gap audit, fork vs build, arxiv paper, dashboard patterns, MCP invocation, visual orchestration, meta-orchestration, dependencies, Mem0 deep-dive
- Discovered device-level keychain bug → pivoted from dual-Claude to hybrid Claude+Gemini
- Evaluated 7 fork candidates → decided on composed architecture
- Developed 21-item Gemini compensation list
- User clarified: wants interactive Claude Code experience, NOT headless → pivoted to MCP server approach
- User clarified: Claude always orchestrates, subtask routing is dynamic
- User clarified: cross-model critique is user-triggered or complexity-triggered
- Identified 61 architectural gaps (13 blockers, 38 complicators, 10 nice-to-have)
- Created ARCHITECTURE.md as living design document
- User emphasized: still in deep investigation phase. Don't rush to implement.

**What changed:**
- Architecture shifted from CLI wrapper to MCP server inside interactive Claude Code
- HANDOFF.md's dual-Claude architecture is now superseded
- The "invisible middleware" concept refined: harness is baked INTO Claude Code, not wrapped around it

**Late-session discovery: Claude Code Swarms / TeammateTool**
- Claude Code has a hidden, feature-flagged multi-agent system (TeammateTool) with 13 operations, task boards with dependencies, inter-agent messaging, and 5 org patterns (Hive, Specialist, Council, Pipeline, Watchdog)
- Unlockable via `claude-sneakpeek` repo (687 stars). NOT officially released or supported by Anthropic.
- **Verdict: Complements the harness, doesn't replace it.** Swarms is Claude-only — no Gemini routing, no external memory, no cross-provider anything. The harness's unique value (cross-provider bridge) is unaffected. Swarms validates the orchestration pattern.
- Official Claude Code subagents (custom subagents in `.claude/agents/`) are production-ready and provide model routing (haiku/sonnet/opus), tool restrictions, and background execution. The harness should work alongside these, not replace them.
- Source: github.com/mikekelly/claude-sneakpeek, @NicerInPerson tweet (534K views, Jan 24 2026)

**Where we stopped:**
- MCP server approach is leading candidate but user explicitly said to keep investigating alternatives
- Memory system is wide open (Mem0 researched in depth, alternatives need equal treatment)
- Visual interface is fuzzy — user wants to understand what value it adds beyond CLI
- Task decomposition and orchestrator prompts not yet researched
- MVP scope not defined
- No code written. No implementation planned. Investigation continues.

**Critical late-session finding: The landscape is closer than we thought.**
End-of-session research (3 deep agents, 30+ tools evaluated) revealed:
- **CCProxy** (starbased): Rule-based routing with Max subscription support, ~65% coverage
- **claude-code-mux**: Rust proxy with OAuth support and auto-failover, ~60% coverage
- **PAL MCP + CCProxy + CodexBar**: Combined zero-code setup gets ~55-60% of our vision
- **The genuinely novel gap is ~40%**: semantic task-type routing, persistent cross-provider memory, proactive rate limit prediction, cross-model critique
- **Conductor Build**: Claude-only parallelization. Does NOT route between providers, share memory, or detect rate limits. Different problem entirely.
- **Nobody has built the full integrated "sous chef"** — but the individual pieces are closer than our earlier research suggested
- **Zero-code baseline test is now even more important**: Install existing tools, use them, discover what's actually missing from experience
- Full competitive analysis documented in RESEARCH.md

**User's parting priority:** "Focus on the continuity document. Make sure everything is documented. My usage is about to run out."

### Session 2 — January 29, 2026 (Early)

**Context:** Continued investigation. Focused on "sinew" (connective tissue) and Conductor research.

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

### Session 3 — January 29, 2026 (Full)

**Context:** Deep research session with 5 parallel agents.

**What happened:**
- Launched 5 parallel research agents investigating:
  1. Task decomposition patterns
  2. Inter-agent communication (sinew)
  3. LLM routing without training
  4. Adversarial/council patterns (critique)
  5. Hidden assumptions and tunnel vision
- Deep-dived each dimension with web search and codebase analysis
- Surfaced 10 hidden assumptions challenging the approach
- Clarified user friction: all 4 problems compound each other
- Resolved visibility contradiction: silent CLI + optional dashboard

**Key decisions locked:**
- Routing: RouteLLM or Aurelio (no training, 90-95% accuracy)
- Communication: Hub-and-spoke + shared state (Blackboard pattern)
- Critique: Veto pattern, user-triggered first, not consensus
- Visibility: Silent CLI + dashboard when wanted

**What moved forward:**
- Memory system selection needs comparison (Mem0 vs files vs sqlite-vec)
- MCP vs Bash+files needs prototype
- Zero-code baseline test still not done

**Documentation reorganization:**
- Created SESSION_3_SYNTHESIS.md as new primary entry point
- Archived HANDOFF.md and SESSION_2_SYNTHESIS.md to docs/archive/
- Updated confidence registry with Session 3 findings

**Next session should:**
- Consider zero-code baseline test first
- OR proceed to Phase 1: semantic router integration

---

*This document is a bridge between sessions. It exists to prevent context loss and enable productive continuation. If you are reading this as a new AI session: welcome. Read the Quick Start above, then ARCHITECTURE.md, then proceed with the Investigation Roadmap. Challenge everything. The user wants depth, honesty, and independent thinking.*
