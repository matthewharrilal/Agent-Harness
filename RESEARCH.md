# Agent Harness Research Report
## January 27, 2026 -- Landscape Survey + Deep-Dive Updates

---

## THE HEADLINE: This Is The White Space

**Nobody has built what you want.** After surveying 40+ tools, frameworks, and projects across the agent orchestration ecosystem, no single project combines:
- Multi-account routing with seamless failover
- Shared memory across agent instances
- Agent orchestration with subagent coordination
- Visual dashboard for monitoring and control

The pieces exist separately. The assembly is the opportunity.

---

## CRITICAL FINDINGS: DEVICE-LEVEL RATE LIMITING

### Verdict: NO Intentional Device Fingerprinting. YES, a Client-Side Bug Produces That Effect.

**Confidence: HIGH** -- confirmed by 6+ GitHub issues, reverse engineering analysis, and official docs.

**What Anthropic actually does:**
- Rate limits are enforced **per-organization/account, server-side**, using a token bucket algorithm
- Anthropic spokesperson confirmed limits are based on **token consumption**, not device identity
- Statsig stableID is used for **feature flags and telemetry only**, not rate limiting
- Reverse engineering (Reid Barber's blog) confirmed no rate limit fingerprinting in the client

**What creates the illusion of device-level tracking:**

A chain of three client-side bugs causes rate limits from Account A to bleed into Account B on the same machine:

| Bug | Description | Issue |
|-----|-------------|-------|
| **Keychain namespace collision** | All accounts write to the same macOS Keychain entry: `Claude Code-credentials`. Switching accounts overwrites tokens. | [#19456](https://github.com/anthropics/claude-code/issues/19456) |
| **Missing `-U` flag** | `security add-generic-password` calls lack the update flag, causing exit code 45 when entry exists | [#10665](https://github.com/anthropics/claude-code/issues/10665) |
| **In-memory session caching** | Even after programmatically swapping Keychain credentials, Claude Code keeps using the old session | [#20549](https://github.com/anthropics/claude-code/issues/20549) |

**Smoking gun evidence (Issue #12379):** The CLI works perfectly with the second account while the VS Code extension stays blocked on the same machine. If Anthropic were fingerprinting devices server-side, the CLI would be blocked too. This proves the bug is in the client, not the server.

**Another smoking gun (Issue #12786):** `claude.ai/settings` shows Account B at 4% usage, yet Claude Code reports "weekly limit reached." The server knows the correct usage; the client is sending wrong credentials.

### GitHub Issues Documenting This Bug

| Issue | Title | Date | Status |
|-------|-------|------|--------|
| [#3857](https://github.com/anthropics/claude-code/issues/3857) | Two accounts on different remote servers share usage limit | Jul 2025 | Closed - Not Planned |
| [#5001](https://github.com/anthropics/claude-code/issues/5001) | Two Claude Pro account limits overlapping | Aug 2025 | Closed - Duplicate |
| [#7892](https://github.com/anthropics/claude-code/issues/7892) | 5-hour limit persists after account switch | Sep 2025 | Closed - Duplicate |
| [#12190](https://github.com/anthropics/claude-code/issues/12190) | Usage reset timers synchronize across accounts | Nov 2025 | Closed - Duplicate |
| [#12379](https://github.com/anthropics/claude-code/issues/12379) | Account switching failure with server-side session caching | Nov 2025 | Related |
| [#12786](https://github.com/anthropics/claude-code/issues/12786) | Rate limit incorrectly applied switching Max accounts | Dec 2025 | Closed - Duplicate |

### Fix Status (as of Jan 27, 2026): UNFIXED

- **Issue #19456** (Keychain namespace collision): OPEN, no Anthropic response
- **Issue #20131** (Multi-account profile support feature request): OPEN, no Anthropic response
- **No confirmed fix in changelog** for any of the three root causes

### Workaround Table

| Workaround | Credential Isolation | Rate Limit Isolation | Confidence | Complexity |
|-----------|---------------------|---------------------|------------|------------|
| `CLAUDE_CONFIG_DIR` | Yes | **Unreliable** (broken per #5001) | Confirmed broken | Low |
| CCS tool (`@kaitranntt/ccs`) | Yes | Unclear | Speculative | Low |
| **Hybrid OAuth + API Key** | **Yes** | **Yes** (different billing systems) | **Likely** | Medium |
| Separate macOS users | Yes | Likely | Speculative | Medium |
| Docker containers | Yes | Likely | Speculative | High |
| Cloud VMs (native install) | Yes | Yes (should work) | Likely | High |
| VS Code Remote SSH | Yes | **Unreliable** (per #3857) | Contradicted | Medium |

**Best workaround:** Use OAuth for one account (Max subscription) and `ANTHROPIC_API_KEY` for the other (pay-as-you-go API). These use completely separate auth and billing pathways, eliminating the keychain collision entirely.

### Bottom Line for the Harness

Running two Claude Max accounts on one MacBook is **currently broken** due to client-side bugs. This makes the hybrid Claude+Gemini approach significantly more attractive -- different providers = no collision possible.

---

## PIVOT: CLAUDE + GEMINI HYBRID HARNESS

### Why This Is Now the Recommended Approach

| Factor | Two Claude Max Accounts | Claude Max + Gemini |
|--------|------------------------|---------------------|
| Device collision | **Broken** (keychain bug) | **None** (different providers) |
| TOS risk | Ambiguous (rate limit circumvention) | **Clean** (two different services) |
| Model diversity | None (same model) | **Complementary strengths** |
| Rate limit pools | Broken on same device | **Fully independent** |
| Downtime correlation | **Correlated** (same provider) | Independent |
| Rate reset cycle | Weekly (both) | Weekly (Claude) + **Daily (Gemini)** |

### Gemini CLI Capabilities (January 2026)

**Version:** v0.25.2 stable / v0.26-0.27 nightly | **License:** Apache 2.0 (fully open source)
> Note: Earlier research referenced v0.23.0/v0.24.0. Updated to reflect latest stable as of Jan 27, 2026.

#### Claude Code vs Gemini CLI Feature Comparison

| Feature | Claude Code | Gemini CLI |
|---|---|---|
| **Default Model** | Claude Sonnet 4 / Opus 4.5 | Gemini 2.5 Pro / 3 Pro |
| **Context Window** | ~200K tokens | **1M tokens** |
| **Config File** | `CLAUDE.md` | `GEMINI.md` (configurable for `AGENTS.md`) |
| **MCP Support** | Yes | Yes (arguably cleaner JSON config) |
| **Built-in Web Search** | No (requires tool/MCP) | **Yes** (Google Search grounding) |
| **Subagents** | **Mature** (Task tool, parallel) | Experimental (flag-gated, PR #4883) |
| **Plan Mode** | Yes (EnterPlanMode tool) | **Yes** (inspired by Claude, Issue #4666) |
| **Parallel Execution** | **Native** (Task tool) | Sequential tools; shell-spawn workaround |
| **Git Integration** | **Deep** (commits, diffs, worktrees) | Basic (history, diffs) |
| **Multimodal Input** | Text & code | **Screenshots, PDFs, Figma, video, Docs** |
| **Headless/CI Mode** | `--print` | `gemini -p` |
| **Safety/Permissions** | Confirm prompts | Policy engine + YOLO mode |
| **Code Quality** | **Higher precision**, fewer bugs | Faster generation, 4x more control-flow errors |
| **IDE Integration** | VS Code | VS Code + JetBrains |
| **Open Source** | No | **Yes** (Apache 2.0) |

#### What Gemini CLI Lacks vs Claude Code
- **Native subagents:** Not built-in yet. PR #4883 attempted but closed. Issue #3132 tracks it.
- **Parallel tool execution:** Issue #14963 tracks this. Currently sequential.
- **TodoWrite equivalent:** No native task tracking.
- **Mature agent loop:** Claude Code's is more battle-tested for long-horizon tasks.

#### What Gemini CLI Has That Claude Code Doesn't
- **1M token context window** -- whole-codebase analysis
- **Built-in Google Search** grounding
- **Multimodal input** (screenshots, PDFs, Figma exports, video)
- **Open source** (inspectable, modifiable)
- **Free tier** (1,000 requests/day)
- **Agent Skills** system (modular plugins)

### Google AI Subscription Tiers

| Tier | Price | Key Features | Best For |
|------|-------|-------------|----------|
| **Free** | $0 | Gemini 3 Flash, 1,000 req/day | Light delegation tasks |
| **Google AI Pro** | $19.99/mo | Gemini 3 Pro, Deep Research, 2TB | **Best value for hybrid harness** |
| **Google AI Ultra** | $249.99/mo | Highest limits, 30TB, YouTube Premium, Veo 3.1 | Overkill for coding-only |

**Recommendation: DO NOT get Google AI Ultra ($250/mo).** AI Pro ($20/mo) or the free tier is sufficient for the Gemini leg of a hybrid harness. Total cost: $200 (Claude Max) + $0-20 (Gemini) = **$200-220/mo** instead of $450/mo for Claude + Ultra.

**Critical distinction:** Google subscriptions boost Gemini CLI limits but are separate from pay-as-you-go API access. For programmatic usage, a separate API key from Google AI Studio or Vertex AI may be more flexible.

### Model Routing Guide: When to Use Which

**Route to Claude (Opus 4.5 / Sonnet 4) when:**
- Complex debugging (methodical, sustained reasoning)
- Long-horizon autonomous agents
- Safety-critical code
- Backend logic & architecture
- Multi-file refactoring
- Tasks requiring mature subagent/parallel infrastructure

**Route to Gemini (3 Pro / 2.5 Pro) when:**
- Large codebase comprehension (1M token context)
- Multimodal tasks (screenshots, PDFs, Figma)
- UI/frontend work
- Rapid prototyping
- Web-grounded research (built-in Google Search)
- Google ecosystem integration (Vertex AI, Firebase, BigQuery)
- "Second opinion" or cross-validation tasks

**Critical Gemini weakness:** 4x higher control-flow mistake rate than Claude. Overconfident -- claims fixes worked when they didn't. Always verify Gemini's code output.

### Existing Hybrid Claude+Gemini Projects

Multiple projects already implement hybrid patterns:

| Project | What It Does | Stars/Status |
|---------|-------------|-------------|
| **[claude-octopus](https://github.com/nyldn/claude-octopus)** | Multi-AI orchestrator: Claude + Gemini + Codex with Double Diamond methodology | Active |
| **[PAL MCP Server](https://github.com/BeehiveInnovations/pal-mcp-server)** | Provider Abstraction Layer: unified context across Claude, Gemini, Codex | 10,200+ stars |
| **[gemini-cli-orchestrator](https://github.com/dnnyngyen/gemini-cli-orchestrator)** | MCP server: Claude Code delegates analysis to Gemini | Active |
| **[claude-code-router](https://github.com/musistudio/claude-code-router)** | Routes Claude Code to Gemini, OpenRouter, DeepSeek, Ollama | Active |
| **[claude-code-proxy](https://github.com/1rgs/claude-code-proxy)** | `PREFERRED_PROVIDER=google` routes to Gemini models | Active |
| **[Portkey AI Gateway](https://portkey.ai/)** | 200+ provider gateway with fallbacks, load balancing, observability | Enterprise |

### Hybrid Architecture Options (Ranked by Effort)

#### Tier 1: Quick Win (Today)
**Claude Code + Gemini CLI subagent via Bash**

Create `.claude/agents/gemini-analyzer.md` that delegates tasks to Gemini CLI via `gemini -p "task"`. Use Gemini's free tier. Zero additional cost.

#### Tier 2: Mature Hybrid (1-2 days)
**Claude Code + PAL MCP Server or gemini-cli-orchestrator**

Install an existing MCP server that bridges Claude and Gemini. Gets you:
- Unified conversation threading across models
- Cross-CLI subagent spawning
- Context revival when one model hits limits
- File-based prompt handling to bypass MCP token limits

Add Google AI Pro ($20/mo) for higher Gemini limits. Total: $220/mo.

#### Tier 3: Full Orchestration (1-2 weeks)
**Google ADK orchestrator with Claude + Gemini backends via LiteLLM**

Build a custom ADK application:
```python
# Claude agent for code editing
code_editor = LlmAgent(
    model=LiteLlm(model="anthropic/claude-opus-4-5-20251101"),
    name="code_editor"
)
# Gemini agent for research/analysis
researcher = LlmAgent(
    model="gemini-3-pro",
    name="researcher"
)
```

Provides maximum control: custom routing, evaluation, state management, deployment flexibility.

### Google ADK vs Claude Agent SDK

| Feature | Google ADK | Claude Agent SDK |
|---|---|---|
| Open Source | Yes (Apache 2.0) | No (proprietary) |
| Languages | Python, Go, TypeScript, Java | Python, TypeScript |
| Multi-Model | **Yes** (via LiteLLM -- can use Claude) | Anthropic models only |
| Workflow Agents | Sequential, Parallel, Loop | Native parallel Task |
| MCP Support | Yes | Yes |
| A2A Protocol | Yes | No |
| Built-in Eval | Yes | No |
| Dev UI | Web UI (localhost:4200) | Terminal only |
| Deployment | Local, container, Cloud Run, Vertex | Local CLI only |

**ADK can use Claude as a backend** via LiteLLM, making it viable as the orchestration layer for a hybrid harness. However, Google's built-in tools (SearchTool etc.) only work with Gemini models.

### A2A Protocol Status
Announced April 2025, donated to Linux Foundation June 2025. **Adoption has been sluggish** -- ecosystem consolidated around MCP instead. For inter-agent communication, **MCP is the safer bet** since both Claude Code and Gemini CLI support it natively.

---

## DASHBOARD PATTERNS & CLI-FIRST INTERFACE DESIGN

### The Dominant Pattern

Every serious agent orchestrator follows the same architectural spine: **tmux sessions for isolation, a TUI for control, and an optional web dashboard for monitoring**. The CLI is the control plane; the dashboard is the observation plane.

### CLI-First Tools Studied

| Tool | Stars | Control Plane | Session Isolation | Key Insight |
|------|-------|--------------|-------------------|-------------|
| **[Claude Squad](https://github.com/smtg-ai/claude-squad)** | 5.6K+ | Bubble Tea TUI | tmux + git worktrees | Gold standard. Daemon for unattended operation. `n` create, `j/k` navigate, `Enter` attach. |
| **[Agent Deck](https://github.com/asheshgoplani/agent-deck)** | Active | Bubble Tea TUI + CLI | tmux + git worktrees | Fuzzy search across sessions. MCP Socket Pool reduces memory 85-90% for 20+ sessions. |
| **[Claude-Flow](https://github.com/ruvnet/claude-flow)** | 13K+ | CLI (26 commands, 140+ subcommands) | Process-based | Swarm topology with "queens." **WARNING:** 85% mock implementations per community audit. |
| **[Agent Farm](https://github.com/Dicklesworthstone/claude_code_agent_farm)** | Active | tmux controller | tmux panes | Pane-per-agent. Auto-recovery with adaptive idle timeout. State in JSON. |
| **[NTM](https://github.com/Dicklesworthstone/ntm)** | Active | tmux commands | tmux named panes | `ntm dashboard`, `ntm status`, `ntm view` for visual management. |

### Dashboard/Observability Tools Studied

| Tool | Type | Key Feature | Self-Hostable |
|------|------|------------|:------------:|
| **[disler's hooks-observability](https://github.com/disler/claude-code-hooks-multi-agent-observability)** | Vue + SQLite | Closest to our needs. Hook scripts → HTTP POST → Bun → SQLite → WebSocket → Vue. 920 stars. | Yes |
| **[multi-agent-dashboard](https://github.com/TheAIuniversity/multi-agent-dashboard)** | React + Tailwind | Monitors 68+ agents. Haiku generates task summaries. WebSocket (port 8766). | Yes |
| **[Langfuse](https://langfuse.com/)** | Open source (MIT) | Industry standard LLM observability. Trace → span → generation hierarchy. Agent graph visualization. Cost tracking with tiered pricing. | Yes |
| **[Helicone](https://www.helicone.ai/)** | Gateway + observability | One-line integration (change base URL). Sessions, anomaly detection. SOC 2 compliant. 10K free req/mo. | Partially |
| **[AgentNeo](https://github.com/VijayRagaAI/agentneo)** | Execution graphs | Traces LLM calls, tool usage, agent interactions. Graph visualization at localhost:3000. | Yes |
| **[SigNoz Claude Dashboard](https://signoz.io/docs/dashboards/dashboard-templates/claude-code-dashboard/)** | Pre-built template | 9 panels: sessions, cost (USD), P95 duration, token usage, success rate, quota usage, model distribution. Via OpenTelemetry. | Yes |

### What Developers Actually Want (RedMonk Survey, Dec 2025)

From [Kate Holterhoff's research](https://redmonk.com/kholterhoff/2025/12/22/10-things-developers-want-from-their-agentic-ides-in-2025/):

1. **MCP integrations** -- seamless connections to external systems
2. **Dashboard monitoring / multi-agent orchestration** -- which agents are working on what, ability to pause/redirect/terminate
3. **Delegation of entire workflows** -- from "write this function" to "build this feature while I review another PR"
4. **Human-in-the-loop controls** -- approval gates, configurable autonomy, audit trails
5. **Reliability** -- "flow state is sacred, and nothing destroys it faster than a sluggish response or mid-task crash"

### Common Pain Points (CLI-Only Experiences)

- No visibility into multiple agents running simultaneously
- Rate limit surprises with no warning
- Cost blindness -- no real-time spend awareness
- Context loss when agents hit limits
- No routing transparency in multi-model setups

### What's Missing in Existing Tools (Our Opportunity)

| Gap | Description |
|-----|-------------|
| **Routing decision transparency** | No tool shows WHY Claude vs Gemini was chosen for a given task |
| **Cross-provider rate limit dashboard** | No unified view of Claude AND Gemini limits |
| **Plan approval workflow** | Approve/reject a plan before execution, model-agnostic |
| **Comparative cost analytics** | "This task cost $0.45 on Claude, would have cost $0.12 on Gemini" |
| **Routing optimization suggestions** | "Based on last 100 tasks, routing X to Gemini would save 40%" |

### Event Capture Mechanisms

**Claude Code hooks** (12 lifecycle events): `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PostToolUseFailure`, `SubagentStart`, `SubagentStop`, `Stop`, `PreCompact`, `Notification`, `SessionEnd`. All receive JSON via stdin with session_id, transcript_path, cwd, permission_mode.

**Claude Code stream-JSON**: `claude -p "task" --output-format stream-json` emits NDJSON with every token, turn, and tool interaction. Can chain: `--input-format stream-json` accepts NDJSON for modular agent pipelines.

**Gemini CLI stream-JSON**: `gemini -p "task" --output-format stream-json` emits JSONL with event types: `init`, `message`, `tool_use`, `tool_result`, `error`, `result`. Includes `stats` field with per-model API and token usage.

**Proven pipeline**: Both CLIs → HTTP POST event collector → SQLite → SSE/WebSocket → Dashboard UI.

### Technology Recommendations

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Event collection | Bun/Node HTTP server | Receives hook POSTs + stream-json pipes |
| Storage | SQLite | Lightweight, zero-config, sufficient for local |
| Real-time push | SSE | Simpler than WebSocket for read-mostly dashboard |
| Frontend | Next.js (static export) or React + Vite | Proven by existing projects |
| Visualization | React Flow + Recharts + react-d3-tree | Routing trees, metrics, decision viz |
| Optional desktop | Tauri wrapper | 20-40MB RAM vs Electron's 200-400MB |

**SSE over WebSocket:** Dashboard is read-mostly. SSE has automatic reconnection, works over standard HTTP, is simpler. For plan approval (the one interactive element), use a separate REST endpoint.

### Claude Code Analytics API

[Anthropic provides an Analytics API](https://platform.claude.com/docs/en/build-with-claude/claude-code-analytics-api) for organizations:
- Track sessions, lines of code added/removed, commits, PRs
- Acceptance/rejection rates for different tools
- Estimated costs and token usage by model
- Free for all organizations with Admin API access

---

## GEMINI CLI DETAILED GAP AUDIT

### Maturity Summary

| Capability | Claude Code | Gemini CLI | Gap Severity |
|---|---|---|---|
| Plan Mode | Production | Nightly/experimental | **High** |
| Subagent orchestration | Production (Task tool) | Experimental | **High** |
| Core file/search/shell tools | Production | Production | Low |
| Notebook editing | Production (NotebookEdit) | Missing (Issue #6930) | **High** |
| User question tool | Production (AskUserQuestion) | In development (PR #16988) | Medium |
| Task management (CRUD) | Production (TaskCreate etc.) | Partial (write_todos) | **High** |
| Headless/programmatic | Good | **Excellent** | None (Gemini advantage) |
| MCP support | Production | Production (schema issues) | Medium |
| Memory/config | Manual | **Richer** (save_memory, /memory) | None (Gemini advantage) |

### Plan Mode Status (Gemini)

**In nightly builds only** (v0.26-0.27 range). Stable is v0.25.2. Key PRs:
- `feat(plan): implement simple workflow` (PR #17326)
- `feat(plan): persistent approvalMode setting` (PR #17350)
- `feat(plan): Shift+Tab Mode Cycling` (PR #17177)
- `feat(plan): add 'communicate' tool kind` (PR #17341)
- `feat(plan): remove subagent invocation from plan mode` (PR #17593)

**Critical gap:** No `--plan-mode` flag for headless invocation. Enforcement may be prompt-based, not tool-level. No structured plan output in JSON format.

### Subagent Status (Gemini)

**Experimental.** Framework exists in `packages/core/src/agents/`. Key elements:
- **CodebaseInvestigator**: Read-only agent. JSON reports with summaries and exploration traces.
- **TOML-based agent definitions**: `.gemini/agents/*.toml`. Loader currently ignores `[inputs]` section.
- **Event-driven scheduler**: PR #17567 (nightly).
- **Per-agent tool refactoring**: PR #17346.

**Critical gap:** No programmatic Task tool equivalent. No parallel agent spawning. No background execution with monitoring. No task dependencies or structured lifecycle.

### MCP Schema Compatibility Issues

1. **`$defs`/`$ref` rejection** (Issue #13326): Gemini API rejects these. Anthropic's FastMCP generates them automatically.
2. **Multi-type arrays crash** (Issue #2654): `type: ["string", "number"]` causes TypeError.
3. **Schema stripping**: Gemini auto-removes `$schema`, `additionalProperties`, `anyOf` defaults.
4. **Tool name sanitization**: Invalid chars → underscores, truncated to 63 chars.

### Gemini-Exclusive Tools (No Claude Code Equivalent)

- `save_memory`: Persists facts to `~/.gemini/GEMINI.md`
- `list_directory` (ReadFolder): Explicit directory listing
- `read_many_files`: Batch file reading
- `activate_skill`: On-demand skill activation

### AGENTS.md as Shared Config

Both Claude Code and Gemini CLI can read `AGENTS.md`. Gemini configured via:
```json
{ "context": { "fileName": ["AGENTS.md", "GEMINI.md"] } }
```
OpenAI Codex also switched to `AGENTS.md`. Emerging as a universal standard.

### Full 21-Item Compensation List

Documented in ARCHITECTURE.md. Categories: Plan Mode (4), Subagent/Task (4), Tool Bridging (2), MCP Compatibility (3), Configuration/Memory (3), Headless Integration (3), Behavioral Parity (2).

---

## ECOSYSTEM MAP

### Multi-Account Routing Tools

| Tool | Stars | What It Does | Multi-Account | Rate Limit Handling | Privacy |
|------|-------|-------------|---------------|--------------------:|---------|
| **claude-balancer** | Active | Load balancer for OAuth/API accounts | Yes | Tier-aware failover | Moderate logging |
| **better-ccflare** | Active | Enhanced proxy with 7+ providers | Yes | Priority-based with 30min buffer | Heavy logging, had security vulns |
| **ccNexus** | 676 | Cross-platform API gateway (Go) | Yes (endpoints) | Failover rotation | Local-first |
| **VibeProxy** | Active | macOS menu bar proxy app | Yes, round-robin | Auto-failover | TOS risk warning |
| **ccproxy (starbased)** | 148 | LiteLLM-based interceptor | Limited | Via LiteLLM | Minimal |
| **LiteLLM** | Very high | General-purpose LLM gateway | API keys only | Cooldown + retry | Configurable |
| **claude-code-hub (zsio)** | 153 | Multi-tenant API proxy | Yes | Weight-based LB | PostgreSQL analytics |
| **OpenRouter** | 4.2M users | Unified API for 400+ models | Yes | Intelligent routing + fallbacks | Configurable ZDR |

### Memory Systems

| System | Self-Hostable | Multi-Agent R/W | Memory Type | Maturity | Best For |
|--------|:------------:|:---------------:|-------------|----------|----------|
| **Mem0** | Yes (Apache 2.0) | Yes (namespaces) | Hybrid: vector + graph + KV | Production-ready | Best all-around |
| **Letta (MemGPT)** | Yes (open source) | Yes (Conversations API) | Tiered: core + recall + archival | Growing | Open-source commitment |
| **Graphiti (Zep engine)** | Yes (Apache 2.0) | Build-your-own | Temporal knowledge graph | Active, 20K stars | Temporal reasoning |
| **LangMem** | Yes (library) | Backend-dependent | Semantic + episodic + procedural | Active | LangChain ecosystem |
| **Cognee** | Yes (open source) | Backend-dependent | Triple: vector + graph + relational | Approaching v1.0 | Domain-specific memory |
| **EverMemOS** | Yes (Apache 2.0) | Unknown | Brain-inspired 4-layer | Very new (Jan 2026) | Research frontier |
| **Redis** | Yes | Yes (battle-tested) | Multi-type | Very mature | Infrastructure layer |

### Agent Orchestration Frameworks

| Framework | Language | Subagents | Parallel Exec | MCP Support | Multi-Backend | Best For |
|-----------|----------|:---------:|:-------------:|:-----------:|:------------:|----------|
| **Claude Agent SDK** | Python, TS | Yes (native) | Yes | Yes | Bedrock, Vertex, Foundry | Foundation for Claude harness |
| **Google ADK** | Python, TS, Go, Java | Yes | Yes | Yes | Gemini, LiteLLM (Claude) | **Multi-model orchestration** |
| **LangGraph** | Python, TS | Yes | Yes | Via tools | Yes (LiteLLM) | Fastest, fewest tokens |
| **CrewAI** | Python | Yes (role-based) | Yes | Limited | Yes | Declarative role-based teams |
| **AutoGen** | Python | Yes (conversational) | Yes | Limited | Yes | Flexible multi-agent convos |
| **PydanticAI** | Python | Agent handoffs | Yes | Yes + A2A | Yes | Type-safe, production-grade |
| **smolagents** | Python | Yes | Yes (parallel tool calls) | Yes | Yes (LiteLLM) | Minimal, code-first |

### Dashboard / Interface Projects

| Project | Type | What It Shows | Most Relevant Feature |
|---------|------|--------------|----------------------|
| **AgCluster Container** | Next.js web dashboard | Real-time tool execution, task tracking | Closest to orchestrator vision |
| **Claude Code by Agents** | Electron desktop | Multi-agent @mentions, session cards | Multi-machine coordination |
| **Claude Code Agent Farm** | tmux TUI | 20-50 agents, heartbeat monitoring | Scale (50 concurrent agents) |
| **Claude-Flow** | CLI + MCP | 64 agents, swarm topologies | Most ambitious orchestrator |
| **Claude Squad** | Go TUI (tmux) | Isolated workspaces, git worktrees | Mature (5.6K stars) |
| **AgentNeo** | React dashboard | Execution graphs, traces | Observability visualization |

### Observability Tools

| Tool | Type | Best For | Self-Hostable |
|------|------|---------|:------------:|
| **Langfuse** | Open source (MIT) | Full observability, data control | Yes |
| **AgentNeo** | Open source | Execution graph visualization | Yes |
| **LangSmith** | Managed | Best debugging, zero overhead | No |
| **Helicone** | Gateway-based | 1-line setup, fastest integration | Partially |
| **Braintrust** | Evaluation-first | Fast traces, evaluation loops | Partially |

---

## THE FRONTIER (January 2026)

### "2026 Is Agent Harnesses"
Industry consensus (Philipp Schmid/HF, Sequoia Capital, Aakash Gupta) that the **harness** -- not the model -- is where differentiation happens. Manus rewrote their harness 5 times (same models); each rewrite improved reliability. Vercel removed 80% of tools and got better results.

### Claude Code Swarms / TeammateTool (Deep Research, Jan 27 2026)

**Source:** [github.com/mikekelly/claude-sneakpeek](https://github.com/mikekelly/claude-sneakpeek) (687 stars, MIT), @NicerInPerson tweet (534K views, Jan 24 2026)

Claude Code contains a fully implemented but feature-flagged multi-agent orchestration system:
- **TeammateTool** with 13 operations: spawnTeam, discoverTeams, requestJoin, approveJoin, rejectJoin, write, broadcast, requestShutdown, approveShutdown, rejectShutdown, approvePlan, rejectPlan, cleanup
- **5 organization patterns**: Hive (task queue), Specialist (rigid roles), Council (debate), Pipeline (sequential), Watchdog (monitoring)
- **File-based coordination**: `~/.claude/teams/{name}/`, `~/.claude/tasks/{name}/`
- **3 spawn backends**: in-process, tmux, iTerm2
- **claude-sneakpeek** patches feature flag check (`I9() && qFB()` → always true) and injects system prompts via `tweakcc` subsystem
- **Team mode marked "legacy"** in claude-sneakpeek v1.6.3 — may indicate Anthropic restructured the code
- **Boris Cherny** (Claude Code dev) at Tokyo Meetup Q&A: "very interested in 'Long running' and 'Swarm'" — at demo stage internally

**Official Claude Code subagents** (production, documented at code.claude.com/docs/en/sub-agents):
- Custom subagents in `.claude/agents/` with YAML frontmatter
- Model routing: `model: haiku` / `model: opus`
- Tool restrictions, permission modes, foreground/background execution
- Subagents CANNOT spawn other subagents (no recursion)

**Harness verdict:** Swarms is Claude-only. No Gemini routing, no external memory, no cross-provider capability. **Complements the harness** (validates orchestration pattern) but does NOT replace it. The harness adds the cross-provider bridge that Swarms cannot provide. Use official subagents (production) for intra-Claude work; harness MCP for cross-provider.

Additional sources:
- [byteiota.com/claude-code-swarms](https://byteiota.com/claude-code-swarms-hidden-multi-agent-feature-discovered/)
- [HN discussion](https://news.ycombinator.com/item?id=46743908)
- [Architectural comparison: Claude Flow V3 vs TeammateTool](https://gist.github.com/ruvnet/18dc8d060194017b989d1f8993919ee4)
- [Claude Code subagents docs](https://code.claude.com/docs/en/sub-agents)

### Claude Cowork (Jan 12, 2026)
Anthropic launched Cowork -- a non-technical agent product built on Claude Agent SDK. Spawns multiple Claude instances for concurrent subtask execution.

### Interoperability Protocols
- **MCP** (Anthropic): Model Context Protocol for tool/context integration -- **THE standard to use**
- **A2A** (Google): Agent-to-Agent protocol -- adoption sluggish, consolidating under MCP
- **ACP** (IBM, merged with A2A under Linux Foundation): Lightweight REST-based agent communication

### Blackboard Pattern Resurgence
The blackboard architecture (shared state as coordination medium) is seeing strong resurgence in multi-agent LLM systems. Key benefits: token efficiency, scalable coordination, agents join/leave dynamically. The "dual-agent workflow" (Gemini detects, Claude fixes, shared via postbox files) is a modern implementation of this pattern.

### Context Engineering
Emerging as a distinct discipline from prompt engineering. Elvis (@omarsar0): "Context windows are NOT the bottleneck -- context engineering IS."

### Wild Card: Google Antigravity
Google Antigravity (launched Nov 2025) provides free access to Claude Opus 4.5 and Gemini 3 Pro in an agent-first IDE. Community-built proxy (`antigravity-claude-proxy`) lets you run Claude models in Claude Code powered by Antigravity tokens. Currently in preview with unpublished limits -- worth monitoring but not a primary backend.

---

## UPDATED RECOMMENDED ARCHITECTURE

Based on ALL research including the device rate limit deep-dive and hybrid feasibility study:

```
YOUR MACBOOK
  |
  ├── Claude Code (Claude Max 20x -- $200/mo)
  │   ├── Primary orchestrator for ALL tasks
  │   ├── Handles: debugging, architecture, backend logic, long-horizon agents
  │   ├── Plan mode, native subagents, parallel execution
  │   └── MCP server connections (shared tools)
  |
  ├── Gemini CLI (Free tier or AI Pro -- $0-20/mo)
  │   ├── Specialist subagent, invoked BY Claude Code
  │   ├── Handles: large codebase analysis, multimodal, research, UI work
  │   ├── Plan mode (yes, has it), Google Search grounding
  │   └── Same MCP servers (shared tool layer)
  |
  ├── Bridge Layer (pick one):
  │   ├── Option A: PAL MCP Server (easiest, 10K+ stars)
  │   ├── Option B: gemini-cli-orchestrator MCP server
  │   ├── Option C: .claude/agents/gemini-analyzer.md subagent
  │   └── Option D: Custom LiteLLM/Portkey gateway
  |
  ├── Shared Memory (Mem0 or Letta, self-hosted)
  │   ├── Both agents read/write same memory
  │   ├── Per-project namespaces
  │   └── Redis as infrastructure
  |
  ├── Shared State (File-based blackboard)
  │   ├── ./postbox/ for inter-agent communication
  │   ├── Plans and task queues in markdown/JSON
  │   └── Git as synchronization mechanism
  |
  └── Dashboard (Phase 2)
      ├── Agent activity feed
      ├── Cross-provider rate limit monitoring
      ├── Task/plan visualization
      └── Combined cost tracking
```

**Total cost: $200-220/mo** (vs $400+ for two Claude accounts or $450 for Claude + Ultra)

---

## RESOLVED QUESTIONS

1. ~~**Device-level tracking**~~: **RESOLVED.** Not real device fingerprinting. Client-side bug chain (keychain collision + missing -U flag + in-memory caching). Unfixed as of Jan 2026.

2. ~~**Can two Claude Max accounts work on one machine?**~~: **RESOLVED.** Currently broken due to client bugs. Use the hybrid approach instead.

3. ~~**Does Gemini CLI have plan mode?**~~: **RESOLVED.** Yes, since late 2025. Inspired by Claude Code's plan mode.

4. ~~**Does Gemini CLI have subagents?**~~: **RESOLVED.** Experimental only. Claude Code's are more mature. Claude should orchestrate, Gemini should be the specialist.

## REMAINING OPEN QUESTIONS

1. **Memory granularity**: Per-project namespaces in Mem0/Letta, or unified global memory?

2. ~~**Interface priority**~~: **RESOLVED.** CLI-first. Dashboard is a secondary monitoring window (a browser tab at localhost:3200). Technology: Next.js/React + SSE + SQLite. The dashboard does NOT control agents -- it observes them.

3. ~~**Gemini Pro quota reality**~~: **RESOLVED.** Ultra gives ~33% more requests for 12.5x the price. Users report Pro-model sub-quota exhausted after 2-3 hours on Ultra. Start with free tier ($0), upgrade to Pro ($20/mo) if needed. Do NOT get Ultra ($250/mo).

4. **Google ADK as orchestration layer**: Could replace Claude Code as the orchestrator in Tier 3. Worth prototyping?

5. **MCP tool unification**: Which MCP servers should both agents share? Which should be agent-specific? Schema translation layer needed for `$defs`/`$ref` compatibility (Issue #13326).

6. ~~**Fork vs build**~~: **RESOLVED.** Composed architecture. Custom orchestration layer using existing tools as dependencies. No single project is close enough to fork. ~12-19 day MVP timeline. TypeScript recommended. See ARCHITECTURE.md for full rationale.

7. **Cross-provider rate limit detection**: How does the harness detect Gemini's current usage? CLI doesn't expose rate limit headers. May need request counting + inference from `--session-summary`.

8. **MCP server lifecycle**: If both CLIs share MCP servers, who manages the server processes?

---

## SOURCES (Key References)

**Device Rate Limit Research:**
- [Issue #12786 - Rate limits across accounts](https://github.com/anthropics/claude-code/issues/12786)
- [Issue #19456 - OAuth Keychain collision](https://github.com/anthropics/claude-code/issues/19456)
- [Issue #3857 - Shared limits on remote servers](https://github.com/anthropics/claude-code/issues/3857)
- [Issue #5001 - Pro account limits overlapping](https://github.com/anthropics/claude-code/issues/5001)
- [Issue #7892 - 5-hour limit persists after switch](https://github.com/anthropics/claude-code/issues/7892)
- [Issue #20131 - Multi-account profile feature request](https://github.com/anthropics/claude-code/issues/20131)
- [Issue #20549 - In-memory credential caching](https://github.com/anthropics/claude-code/issues/20549)
- [Reid Barber - Reverse Engineering Claude Code](https://www.reidbarber.com/blog/reverse-engineering-claude-code)
- [Issue #7151 - Statsig/OpenAI acquisition concerns](https://github.com/anthropics/claude-code/issues/7151)
- [Anthropic Rate Limits Documentation](https://platform.claude.com/docs/en/api/rate-limits)

**Hybrid Claude+Gemini:**
- [claude-octopus](https://github.com/nyldn/claude-octopus)
- [PAL MCP Server](https://github.com/BeehiveInnovations/pal-mcp-server)
- [gemini-cli-orchestrator](https://github.com/dnnyngyen/gemini-cli-orchestrator)
- [claude-code-router](https://github.com/musistudio/claude-code-router)
- [claude-code-proxy](https://github.com/1rgs/claude-code-proxy)
- [Portkey Claude Code Integration](https://portkey.ai/docs/integrations/libraries/claude-code)
- [Dual-Agent Workflow (Medium)](https://medium.com/@slayerfifahamburg/the-dual-agent-workflow-how-to-pair-gemini-cli-and-claude-code-for-autonomous-code-evolution-f8f94900b6fc)
- [Hybrid Workflow: Spawning Gemini from Claude Code](https://paddo.dev/blog/gemini-claude-code-hybrid-workflow)
- [Gemini CLI Advisor for Claude Code](https://github.com/jezweb/gemini-cli-advisor-for-claude-code)

**Gemini CLI:**
- [Gemini CLI GitHub](https://github.com/google-gemini/gemini-cli)
- [Gemini CLI Agent Skills](https://geminicli.com/docs/cli/skills/)
- [Gemini CLI MCP Docs](https://geminicli.com/docs/tools/mcp-server/)
- [GEMINI.md Docs](https://geminicli.com/docs/cli/gemini-md/)
- [Plan Mode Issue #4666](https://github.com/google-gemini/gemini-cli/issues/4666)
- [SubAgent Issue #3132](https://github.com/google-gemini/gemini-cli/issues/3132)
- [Shipyard: Claude Code vs Gemini CLI](https://shipyard.build/blog/claude-code-vs-gemini-cli/)
- [Composio: Gemini CLI vs Claude Code](https://composio.dev/blog/gemini-cli-vs-claude-code-the-better-coding-agent)

**Google AI Plans:**
- [Google AI Plans pricing](https://one.google.com/intl/en/about/google-ai-plans/)
- [Gemini CLI Quotas](https://geminicli.com/docs/quota-and-pricing/)
- [9to5Google: AI Pro/Ultra features](https://9to5google.com/2026/01/16/google-ai-pro-ultra-features/)

**Google ADK:**
- [ADK Overview](https://google.github.io/adk-docs/get-started/about/)
- [ADK: Claude via LiteLLM](https://google.github.io/adk-docs/agents/models/litellm/)
- [ADK: Agent Teams](https://google.github.io/adk-docs/tutorials/agent-team/)

**Agent Harness Trend:**
- [Aakash Gupta - "2025 Was Agents, 2026 Is Agent Harnesses"](https://aakashgupta.medium.com)
- [Philipp Schmid - Agent Harness Importance](https://www.philschmid.de/agent-harness-2026)

**Multi-Provider Gateways:**
- [LiteLLM Claude Code Tutorial](https://docs.litellm.ai/docs/tutorials/claude_responses_api)
- [OpenRouter Claude Code Integration](https://openrouter.ai/docs/guides/guides/claude-code-integration)
- [Portkey AI Gateway](https://github.com/Portkey-AI/gateway)

**Tools:**
- [claude-balancer](https://github.com/snipeship/claude-balancer)
- [AgCluster Container](https://github.com/whiteboardmonk/agcluster-container)
- [Claude-Flow](https://github.com/ruvnet/claude-flow)
- [Claude Squad](https://github.com/smtg-ai/claude-squad)
- [Agent Deck](https://github.com/asheshgoplani/agent-deck)
- [Agent Farm](https://github.com/Dicklesworthstone/claude_code_agent_farm)
- [NTM (Named Tmux Manager)](https://github.com/Dicklesworthstone/ntm)
- [AWS CLI Agent Orchestrator](https://aws.amazon.com/blogs/opensource/introducing-cli-agent-orchestrator-transforming-developer-cli-tools-into-a-multi-agent-powerhouse/)

**Memory:**
- [Mem0](https://github.com/mem0ai/mem0)
- [Letta](https://github.com/letta-ai/letta)
- [Graphiti](https://github.com/getzep/graphiti)

**Observability & Dashboards:**
- [Langfuse](https://langfuse.com/)
- [Langfuse Agent Graphs](https://langfuse.com/docs/observability/features/agent-graphs)
- [disler's hooks-observability](https://github.com/disler/claude-code-hooks-multi-agent-observability)
- [multi-agent-dashboard](https://github.com/TheAIuniversity/multi-agent-dashboard)
- [Helicone](https://github.com/Helicone/helicone)
- [AgentNeo](https://github.com/VijayRagaAI/agentneo)
- [SigNoz Claude Code Dashboard](https://signoz.io/docs/dashboards/dashboard-templates/claude-code-dashboard/)
- [SigNoz Claude Code + OpenTelemetry](https://signoz.io/blog/claude-code-monitoring-with-opentelemetry/)
- [LLM-Use Intelligent Router](https://github.com/llm-use/llm-use)
- [Red Hat LLM Semantic Router](https://developers.redhat.com/articles/2025/05/20/llm-semantic-router-intelligent-request-routing)
- [LangSmith Dashboards](https://docs.langchain.com/langsmith/dashboards)
- [Dev-Agent-Lens (Arize)](https://arize.com/blog/claude-code-observability-and-tracing-introducing-dev-agent-lens/)

**CLI & Headless:**
- [Claude Code Headless Mode](https://code.claude.com/docs/en/headless)
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide)
- [disler's hooks-mastery](https://github.com/disler/claude-code-hooks-mastery)
- [Gemini CLI Headless Mode](https://geminicli.com/docs/cli/headless/)
- [Claude Code Analytics API](https://platform.claude.com/docs/en/build-with-claude/claude-code-analytics-api)

**Developer Experience:**
- [10 Things Developers Want from Agentic IDEs (RedMonk)](https://redmonk.com/kholterhoff/2025/12/22/10-things-developers-want-from-their-agentic-ides-in-2025/)
- [Shipyard: Track Claude Code Usage](https://shipyard.build/blog/claude-code-track-usage/)

**Technology Choices:**
- [Electron vs Tauri (DoltHub)](https://www.dolthub.com/blog/2025-11-13-electron-vs-tauri/)
- [Why Tauri Instead of Electron (Aptabase)](https://aptabase.com/blog/why-chose-to-build-on-tauri-instead-electron)
- [WebSockets vs SSE (Ably)](https://ably.com/blog/websockets-vs-sse)
- [React Flow (Dagre Layout)](https://reactflow.dev/examples/layout/dagre)

**Multi-Agent Research:**
- [arxiv 2601.14351 - "If You Want Coherence, Orchestrate a Team of Rivals"](https://arxiv.org/abs/2601.14351)

**Multi-Account Workarounds:**
- [CCS Tool](https://ccs.kaitran.ca/)
- [Asterisk Multi-Account Manager](https://github.com/juddflamm/asterisk)
- [JCMC Blog: Multiple Claude Code Accounts](https://jcmc.dev/posts/2025/10/multiple_claude_code_accounts/)

**Gemini CLI Gap Audit:**
- [Gemini CLI Releases](https://github.com/google-gemini/gemini-cli/releases)
- [Codebase Investigator Discussion #11375](https://github.com/google-gemini/gemini-cli/discussions/11375)
- [Agent Framework Discussion #12832](https://github.com/google-gemini/gemini-cli/discussions/12832)
- [Custom Sub-Agents Guide](https://website-nine-gules.vercel.app/blog/how-to-create-custom-sub-agents-gemini-cli)
- [MCP Schema $defs Issue #13326](https://github.com/google-gemini/gemini-cli/issues/13326)
- [MCP Multi-type Schema Issue #2654](https://github.com/google-gemini/gemini-cli/issues/2654)
- [.mcp.json Support Issue #13765](https://github.com/google-gemini/gemini-cli/issues/13765)
- [AGENTS.md Discussion #1471](https://github.com/google-gemini/gemini-cli/discussions/1471)
- [AskUser Issue #15995](https://github.com/google-gemini/gemini-cli/issues/15995)
- [Task Tool Issue #7532](https://github.com/google-gemini/gemini-cli/issues/7532)
- [NotebookEdit Issue #6930](https://github.com/google-gemini/gemini-cli/issues/6930)
- [Gemini CLI Changelog](https://geminicli.com/docs/changelogs/)

---

---

## MCP-BASED HARNESS INVOCATION (Round 3 Research)

### The Architecture Shift

The harness is NOT a CLI wrapper spawning headless processes. It is an **MCP server that runs inside interactive Claude Code**. User types `claude` normally. The harness is baked in via `.mcp.json`, hooks, and `CLAUDE.md`.

### How MCP Servers Work in Claude Code

- **Stdio transport**: Claude Code spawns the MCP server as a child process on session start. Server dies when session ends.
- **Tools visible as**: `mcp__agent-harness__delegate_to_gemini` etc.
- **Configuration**: `.mcp.json` at project root or `~/.claude.json` for global.
- **Environment variables**: Supports `${VAR}` and `${VAR:-default}` syntax.
- **Dynamic tool updates**: MCP `list_changed` notifications let server add/remove tools at runtime.

### Concrete MCP Server Example

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "agent-harness", version: "1.0.0" });

server.tool("delegate_to_gemini", "Delegate task to Gemini CLI", {
  prompt: z.string(),
  files: z.array(z.string()).optional(),
}, async ({ prompt, files }) => {
  // Spawn gemini -p subprocess, capture stream-json output
  // Return result to Claude's conversation
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

### MCP Limitations Inside Claude Code

- **Cannot access conversation history** -- only gets tool call inputs
- **Cannot modify system prompt at runtime** -- use CLAUDE.md for static instructions
- **Cannot intercept other tools' calls** -- use hooks for that
- **Tool output capped at 25K tokens** (configurable via `MAX_MCP_OUTPUT_TOKENS`)
- **Default timeout 60-120s** -- must set `MCP_TOOL_TIMEOUT=600000` for Gemini delegation
- **Cannot proactively send messages** -- only responds to tool calls

### Hooks for Event Capture

12 hook events in Claude Code: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PostToolUseFailure`, `SubagentStart`, `SubagentStop`, `Stop`, `PreCompact`, `Notification`, `SessionEnd`.

Hooks support regex matchers: `"matcher": "mcp__agent-harness__.*"` captures all harness tool calls.

Hook scripts receive JSON on stdin (session_id, tool_name, tool_input, etc.) and can POST to dashboard.

### PAL MCP Server (Reference Implementation)

PAL MCP (10K stars) implements this exact pattern:
- Claude Code spawns PAL as stdio child process
- PAL's `clink` tool spawns Gemini/Codex CLIs as subprocesses
- CLI configs in `conf/cli_clients/gemini.json`: `gemini --telemetry false --yolo -o json`
- Role-specific system prompts in `systemprompts/clink/`
- Context revival: if Claude's session clears, delegate to large-context model to resummarize

### CodexBar (Menu Bar Monitoring Widget)

[CodexBar](https://github.com/steipete/CodexBar) by Peter Steinberger -- macOS menu bar app (Swift, macOS 14+) showing real-time usage stats for Claude, Gemini, Codex, Cursor, Copilot.

Data sources:
1. OAuth API (reads Claude CLI credentials from macOS Keychain)
2. Web API (browser cookies fallback)
3. CLI PTY (spawns `claude` and parses text output)
4. Local JSONL logs at `~/.config/claude/projects/**/*.jsonl` for cost tracking

User wants a similar ambient monitoring widget. Could be a menu bar app or browser page.

---

## META-ORCHESTRATION RESEARCH (Round 3)

### Key Finding: No Production System Uses Dynamic Orchestrator Rotation

Every major framework uses a fixed orchestrator with dynamic worker selection:
- **AutoGen SelectorGroupChat**: GroupChatManager is fixed; speakers rotate
- **CrewAI Hierarchical**: Manager agent is fixed
- **Google ADK**: Single-parent tree hierarchy (enforced by code)
- **LangGraph**: The graph (code) IS the orchestrator
- **Manus AI**: Claude orchestrates, other models execute

### NeurIPS 2025 "Puppeteer" Paper (arXiv:2505.19591)

Most advanced dynamic orchestration research. Uses RL to train an orchestrator policy. Key finding: the orchestrator identity stays fixed -- what changes is its BEHAVIOR (which agents it selects). Token usage actually decreases over training.

### MAST Failure Taxonomy (NeurIPS 2025, arXiv:2503.13657)

1600+ traces across 7 frameworks. 14 failure modes:
- 41.77% from system design
- 36.94% from inter-agent misalignment
- 21.29% from verification/termination

Most dangerous for dynamic orchestration: **responsibility diffusion** (each agent thinks the other validated something, but nobody did) and **recursive delegation loops** ($47,000 failure case documented).

### Decision for Our Harness

**Claude always orchestrates.** Subtask routing to Gemini vs Claude is dynamic. The FSM router (or Claude's own judgment via CLAUDE.md instructions) decides per subtask. Future v2 could add task-type-based orchestrator selection at the code level.

---

## VISUAL INTERFACE RESEARCH (Round 3)

### What Conductor Build Actually Does

Conductor (YC-backed, Apple Silicon only) runs multiple Claude Code instances in parallel git worktrees:
- Create workspace → assign task → agents run simultaneously → review diffs → create PR
- Not a workflow designer. A parallelization multiplier.
- The New Stack review: "You don't sit and watch the LLM work -- you move on to creating another workspace."

### The @omarsar0 (Elvis) Pattern

Elvis builds custom dashboards on TOP of Claude Code via hooks. CLI-primary with monitoring bolted on. His architecture: orchestrator agent → specialized sub-agents (search, coder, KB retriever, analyst, verifier, writer), each with clean context. Orchestrator delegates and integrates.

### CLI vs Visual: The Honest Tradeoff

| Visual Wins | CLI Wins |
|-------------|----------|
| See 5+ agents simultaneously | Speed of interaction (keyboard > mouse) |
| Diff review across parallel branches | Flexibility/customization (infinite) |
| Debugging multi-agent failures | Maintaining flow state |
| Cost/token real-time tracking | No context-switching penalty |

**Complexity threshold:** 1-2 agents = CLI fine. 3-5 = TUI dashboard helps. 5-10+ = visual overview genuinely needed.

### What Visual Tools Do Wrong

- Break flow state (terminal → browser = 15-25 min refocus penalty per switch)
- Over-engineered (too many clicks for power users)
- Try to replace IDE instead of complement it
- Demo beautifully, break under real-world load (Composio "Stalled Pilot Syndrome" report)

### Our Approach

Hybrid: CLI-primary (terminal) + ambient monitoring (CodexBar-style menu bar widget OR localhost browser page). Never leave terminal for primary work. Dashboard for observability only.

---

## DEPENDENCY RESEARCH (Round 3)

### Recommended Core Stack (TypeScript)

| Package | Purpose | Why This One |
|---------|---------|-------------|
| `@modelcontextprotocol/sdk` v1.x | MCP server/client | Official SDK. Supports stdio + HTTP. zod peer dep. |
| `execa` v9.6 | Subprocess management | 18M+ weekly downloads. Stream conversion for NDJSON. |
| `p-queue` v8 | Concurrency control | Priority support, pause/resume, per-operation timeouts. |
| `tree-kill` v1.2.2 | Process cleanup | Cross-platform process tree termination. |
| `commander` v12 | CLI parsing | 100M+ downloads. Lightweight. |
| `zod` v3.22+ | Schema validation | MCP peer dep. Zero dependencies. |
| `better-sqlite3` v11 | SQLite driver | Fastest Node.js SQLite. Synchronous API. WAL mode. |
| `drizzle-orm` v0.36+ | Type-safe SQL | Zero overhead over raw queries. Great migrations. |
| `hono` v4 | HTTP server + SSE | 14KB. Built-in `streamSSE`. Bun-native. |
| `vitest` v3 | Testing | 10-20x faster than Jest for TypeScript. |

### What Existing Projects Use

- **Claude Squad** (Go): bubbletea + lipgloss + creack/pty + go-git + cobra
- **disler's observability**: Bun + SQLite + ws + Vue + Vite + Tailwind (extremely minimal)
- **PAL MCP** (Python): mcp + google-genai + openai + pydantic
- **claude-flow** (TypeScript): zod + hono + vitest (same stack we recommend)

### Key Insight from claude-flow's stack

Claude-flow uses the exact same core packages we identified (zod, hono, vitest). This validates the stack choice even though claude-flow's implementation is incomplete.

---

### MCP Apps (SEP-1865) — Evaluated Jan 27, 2026

**What it is:** First official MCP extension, announced Jan 26 2026. Allows MCP servers to return interactive HTML UIs rendered in sandboxed iframes within the host application. Joint effort by Anthropic, OpenAI, and community.

**Supported hosts:** Claude web, Claude Desktop, VS Code Insiders, Goose, ChatGPT, Postman.

**NOT supported:** Claude Code (terminal CLI — cannot render iframes).

**Harness verdict:** **Not relevant.** The harness runs inside Claude Code (terminal). MCP Apps require a graphical host. The harness is invisible middleware (backend orchestration), not interactive UI. The standard MCP Server pattern remains correct.

**Possible future use:** If a companion dashboard were built inside Claude Desktop alongside a web workflow, MCP Apps could power that UI. But this is a distant optional layer, not an architecture change.

Sources:
- [MCP Apps blog post](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/)
- [MCP Apps documentation](https://modelcontextprotocol.io/docs/extensions/apps)

---

## COMPETITIVE LANDSCAPE DEEP-DIVE (End of Session 1)

### The Honest Assessment: How Close Are Existing Tools?

Late-session research (3 agents, 30+ tools) revealed the landscape is closer to our vision than earlier research suggested. Here is the tool-by-tool assessment against 5 requirements: (1) subscription auth, (2) intelligent routing, (3) shared memory, (4) inside Claude Code, (5) rate limit failover.

| Tool | Sub Auth | Routing | Memory | In Claude Code | Rate Limit | Coverage |
|------|:--------:|:-------:|:------:|:--------------:|:----------:|:--------:|
| **CCProxy** (starbased) | Yes | Rule-based | No | Yes (proxy) | Partial (reactive) | ~65% |
| **claude-code-mux** | Yes (OAuth) | Priority-based | No | Yes (proxy) | Yes (failover) | ~60% |
| **CLIProxyAPI** | Yes (OAuth) | Round-robin | Partial | Yes (proxy) | Yes (quota tracking) | ~60% |
| **PAL MCP** | Separate | Manual only | Partial (3hr/20 turns) | Yes (MCP) | No | ~45% |
| **claude-code-router** | Mixed | By request type | No | Yes (proxy) | Limited | ~50% |
| **hcom** | Yes | No routing | Yes (messaging) | Yes (hooks) | No | ~35% |
| **Portkey** | Yes | Metadata-based | No | Yes (gateway) | Yes (retries) | ~30% |
| **OpenRouter** | Yes (forwarded) | Auto Router exists* | No | Yes (gateway) | Partial | ~25% |
| **Conductor Build** | Yes | No (Claude-only) | No (.context folder) | No (separate app) | No | ~15% |

*OpenRouter's Auto Router exists but Claude Code doesn't use it — Claude Code hardcodes model selection.

### What's Genuinely Novel (The ~40% Gap)

1. **Semantic task-type routing**: "This is a code review → Claude. This is research → Gemini." No tool routes by what the task IS. CCProxy routes by token count and model name. Not the same.
2. **Persistent cross-provider memory**: hcom does real-time messaging. PAL does 3hr conversation threads. Nobody has a persistent store both providers read/write to across sessions.
3. **Proactive rate limit prediction**: Every tool REACTS to 429 errors. Nobody PREDICTS "Claude is at 80%, route next task to Gemini preemptively."
4. **Cross-model critique orchestration**: Nobody implements "Claude generates, Gemini reviews" automatically. The Team of Rivals pattern is unimplemented.
5. **The integrated "sous chef"**: All of the above, working together invisibly inside interactive Claude Code.

### Best Zero-Code Combo Available Today

**PAL MCP + CCProxy + CodexBar** delivers ~55-60% of the vision:
- PAL: manual delegation and context revival
- CCProxy: automatic large-context routing to Gemini
- CodexBar: visual rate limit awareness

Gap: no semantic routing, no persistent memory, no proactive rate switching, no cross-model critique, context revival is lossy and time-limited.

### Tools Discovered This Round (Not in Earlier Research)

- **CCProxy** (starbased-co/ccproxy): LiteLLM-based proxy with Max subscription support and rule-based routing. TokenCountRule, ThinkingRule, MatchModelRule, MatchToolRule. Source: github.com/starbased-co/ccproxy
- **claude-code-mux** (9j/claude-code-mux): Rust-powered proxy, 18+ providers, OAuth support, ~5MB RAM, <1ms overhead. Source: github.com/9j/claude-code-mux
- **CLIProxyAPI** (router-for-me/CLIProxyAPI): macOS menu bar proxy with OAuth subscription support and real-time quota tracking. Source: github.com/router-for-me/CLIProxyAPI
- **hcom** (aannoo/hcom): Hook-based inter-agent communication between Claude Code and Gemini CLI. Transcript sharing, cross-agent search. Source: github.com/aannoo/hcom
- **Not Diamond**: AI model router using learned semantic routing (not rule-based). Best-in-class task-type routing but API-only, not integrated with Claude Code. Source: notdiamond.ai
- **Jenova AI**: Unified multi-model platform with persistent memory and intelligent routing — but it's a separate app, not CLI middleware. Source: jenova.ai

### Conductor Build Details

YC-backed, free Mac app by Melty Labs. Runs multiple Claude Code instances in parallel git worktrees with a polished GUI (diff viewer, file explorer, terminal, checkpoints, PR integration). Supports Claude Code + Codex + some Gemini via env vars.

**What it does NOT do**: No intelligent routing between providers, no rate limit awareness, no shared live memory between agents, no automated task distribution. Agents are fully isolated by design. Closed source. Mac-only. When rate-limited, "your workflow stalls."

**Verdict**: Conductor solves parallelism within Claude, not multi-provider orchestration. ~15% overlap with the harness vision.

---

## KNOWN RESEARCH GAPS

The following topics have been identified as needing investigation but have NOT been researched yet:

1. **Real-world MCP tool timeout behavior** — Has anyone run 5+ minute MCP tool calls in production Claude Code? What does the UX look like?
2. **Token economics modeling** — How many tokens does meta-work (routing, memory, delegation) consume per task?
3. **Conductor Build deep analysis** — Feature comparison, pricing, limitations vs our harness approach
4. **PAL MCP first-hand evaluation** — Does `clink` actually work well? Configuration? Limitations in practice?
5. **Gemini CLI headless output format** — No sample `stream-json` output documented
6. **Claude Code MCP output handling at >25K tokens** — What happens? Truncation? Error?
7. **Orchestrator system prompts** — What do Manus, Devin, Claude Code's own planner prompts look like?
8. **AGENTS.md spec** — What's actually in the format? What fields? Is there a formal spec?
9. **Latency benchmarks** — How long does `gemini -p` actually take for typical tasks?
10. **Anthropic rate limit query API** — Can you programmatically check current usage?

---

*Research conducted January 27, 2026 via 19 research agents across three rounds. Round 1 (9 agents): ecosystem survey, memory systems, multi-account routing, agent SDK comparison, dashboard interfaces, frontier research, device-level rate limits, Claude+Gemini hybrid feasibility, Gemini CLI capabilities. Round 2 (6 agents): Ultra vs Pro limits, Gemini CLI gap audit, fork vs build evaluation, arxiv paper analysis, CLI-first dashboard patterns, critical gap analysis. Round 3 (4 agents): MCP-based invocation, visual orchestration interfaces, meta-orchestration patterns, dependency stack research, Mem0 deep-dive.*

*If you are a new Claude or Gemini instance: read ARCHITECTURE.md first for decisions and reflections, then this file for evidence, then HANDOFF.md for original vision (note: its architecture section is superseded by the Claude+Gemini hybrid approach).*
