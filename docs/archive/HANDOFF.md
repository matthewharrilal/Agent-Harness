# Claude Agent Harness: Comprehensive Handoff Document

> **⚠️ ARCHIVED — January 29, 2026**
>
> **This document has been superseded by SESSION_3_SYNTHESIS.md**
>
> This was the original handoff document from a claude.ai conversation. It has been archived because:
> - The dual-Claude architecture (Part 3) was abandoned after the device keychain bug discovery
> - The hybrid Claude+Gemini architecture is now documented in ARCHITECTURE.md
> - Key insights from Parts 1 and 5 have been extracted into SESSION_3_SYNTHESIS.md
>
> **For current project state, start with:** `/docs/SESSION_3_SYNTHESIS.md`

---

> **STATUS: PARTIALLY SUPERSEDED (Updated Jan 27, 2026 — End of Session 1)**
>
> This was the original handoff document from a claude.ai conversation. Key changes since:
> - **Dual-Claude architecture → Hybrid Claude+Gemini** (device keychain bug kills dual-Claude on one machine)
> - **CLI wrapper → MCP server inside interactive Claude Code** (user wants to type `claude` normally)
> - **$400/mo → $200-220/mo** (Claude Max $200 + Gemini free/$20)
> - **Claude Agent SDK as foundation → MCP server + hooks + CLAUDE.md** (SDK replaces Claude Code; MCP augments it)
>
> **Read SESSION_BRIDGE.md first.** It is the current source of truth for project state, confidence levels, and session conduct.
> Parts 1 (Vision), 2 (Journey), and 5 (Requirements) of this document remain useful.
> **Part 3 (Proposed Architecture) is SUPERSEDED** — see ARCHITECTURE.md for current design.

**Date:** January 27, 2026
**Purpose:** Complete context transfer for building a unified agent orchestration system ~~across two Claude Max 20x accounts~~ combining Claude Max + Gemini CLI
**Next Environment:** Claude Code CLI
**Status:** ~~Research & Architecture Phase — Ready for Deep Tool Discovery~~ Deep Investigation Phase — See SESSION_BRIDGE.md

---

## EXECUTIVE SUMMARY

This document represents a complete handoff from a claude.ai conversation to Claude Code CLI. ~~The user has two Claude Max 20x accounts ($400/month combined) and wants to build an **agent harness** that makes them function as a **single unified experience**.~~ **Updated: The user has Claude Max 20x ($200/mo) + Gemini CLI (free/$20/mo) = $200-220/mo. See ARCHITECTURE.md.** The only thing that should be "split" between ~~accounts~~ providers is inference capacity—everything else (memory, plans, context, subagent coordination) should be shared through external infrastructure the user controls.

**This is NOT just load balancing. This is building a custom agent orchestration layer.**

---

## PART 1: THE VISION

### 1.1 Core Goal

Create a unified agent experience where:

- **Two Claude Max accounts** provide combined inference capacity (avoiding rate limits)
- **Shared memory layer** ensures both accounts have the same context and history
- **Shared planning system** means any agent can read/update the current plan
- **Subagent coordination** works across both accounts seamlessly
- **The user experiences ONE system**, not two siloed accounts

The user explicitly stated: *"The only thing I'm really getting from two different accounts is the combined amount of inference capabilities."*

### 1.2 Inspiration

The user shared screenshots of a custom "Claude Orchestrator" built by @omarsar0 (elvis on X/Twitter) with these characteristics:

- **18 simultaneous sessions** visible in a unified dashboard
- **Projects sidebar** for organization
- **Active sessions tracking** with real-time status
- **Servers & Ports management** (multiple services running)
- **Tool use visualization** showing agent actions in real-time
- **Built primarily on Claude Agent SDK** but flexible to use other frameworks

Key quote from the creator: *"I've been building my own agent orchestrator... It's tuned to how I work. It looks like an IDE, but it's not. It's a control center for my agents... I can add and remove whatever I like. I research from it. I write from it. I code from it. I do everything there... No limits. 24/7 personalized agents doing all kinds of work."*

This represents the north star: a personal agent control center that transcends the limitations of any single account or interface.

### 1.3 Why Build This?

Current limitations of using two separate accounts:

| What's Siloed | Impact |
|---------------|--------|
| Memory | Account 1 doesn't know what Account 2 learned |
| Projects | Can't share project context between accounts |
| Conversation History | Complete context loss when switching |
| Subagents | Can't coordinate agents across accounts |
| Rate Limits | Each account has separate limits that don't communicate |

**The harness solves all of this** by externalizing shared state to infrastructure you control.

---

## PART 2: CONVERSATION JOURNEY (How We Got Here)

### 2.1 Phase 1: The Rate Limit Problem

The user initially came with a practical problem: hitting 85% of weekly Claude Max limits in 1-2 days. Considered getting a second account for overflow capacity.

**Initial question:** How do I route between two Claude Max accounts to get more inference?

### 2.2 Phase 2: Proxy Tool Research

Researched existing tools for multi-account routing:

| Tool | Notes |
|------|-------|
| better-ccflare | Most feature-complete, but documented security concerns (plaintext token storage, no auth, request logging) |
| ccflare (original) | Foundation for better-ccflare |
| LiteLLM | Enterprise-grade, 15k+ stars, but designed for API keys not OAuth |
| claude-balancer | Tier-aware distribution |
| ccNexus | Lightweight endpoint rotation |

**Key finding:** All these tools can route requests, but they can't sync memory, projects, or context. They're just proxies.

### 2.3 Phase 3: Privacy Analysis

User expressed strong privacy preferences. Research revealed what these tools store:

- **better-ccflare stores:** OAuth tokens (plaintext), full request bodies, full response bodies (base64), session data
- **Security documentation explicitly states:** "designed for local development and trusted environments"

**Key realization:** Even with hardening, proxy tools see everything. Custom minimal implementation (~100 lines) could do zero logging.

### 2.4 Phase 4: The Limitation Wall

Critical insight emerged: **Proxies can't create a unified experience.**

Two accounts remain fundamentally siloed on Anthropic's side:
- Account 1's Claude memory ≠ Account 2's Claude memory
- Account 1's projects ≠ Account 2's projects
- Switching accounts = complete context loss

**This is when the conversation pivoted to the harness architecture.**

### 2.5 Phase 5: The Harness Vision (Current)

If Anthropic's infrastructure can't unify the accounts, **build an external layer that does:**

- External memory (Mem0 or alternatives) → Shared context
- External planning (files, database) → Shared plans
- External task queue → Shared work coordination
- Orchestrator layer → Manages which account handles what
- Claude Agent SDK → The backbone for agent execution

**The accounts become interchangeable inference endpoints. Everything meaningful is external.**

---

## PART 3: PROPOSED ARCHITECTURE

> **SUPERSEDED.** The dual-Claude architecture below was abandoned after discovering a device-level keychain bug (macOS Keychain namespace collision under `Claude Code-credentials`) that makes running two Claude accounts on one machine unreliable. See ARCHITECTURE.md for the current design: Claude+Gemini hybrid with MCP server inside interactive Claude Code.

### 3.1 High-Level Architecture (HISTORICAL — DO NOT USE)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         YOUR AGENT HARNESS                                  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                      SHARED STATE LAYER                               │ │
│  │                                                                       │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │ │
│  │  │   Memory    │  │   Plans &   │  │    Task     │  │   Agent     │ │ │
│  │  │   System    │  │   Context   │  │   Queue     │  │  Registry   │ │ │
│  │  │ (External)  │  │  (External) │  │ (External)  │  │  (Config)   │ │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                     ORCHESTRATION LAYER                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │  │                  Primary Orchestrator                           │ │ │
│  │  │  • Reads shared memory before each action                       │ │ │
│  │  │  • Updates shared plans after each decision                     │ │ │
│  │  │  • Spawns and coordinates subagents                             │ │ │
│  │  │  • Monitors progress across all agents                          │ │ │
│  │  │  • Synthesizes results back to shared state                     │ │ │
│  │  │  • Decides which account to use for each task                   │ │ │
│  │  └─────────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                      SUBAGENT LAYER                                   │ │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐         │ │
│  │  │ Research  │  │  Coding   │  │  Writing  │  │  Analysis │  ...    │ │
│  │  │  Agent    │  │  Agent    │  │  Agent    │  │   Agent   │         │ │
│  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘         │ │
│  │  Each subagent:                                                       │ │
│  │  • Has isolated context window (efficient)                            │ │
│  │  • Reads from shared memory (unified)                                 │ │
│  │  • Writes results to shared state (persistent)                        │ │
│  │  • Can use EITHER account's inference (load balanced)                 │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                      INFERENCE LAYER                                  │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │  │              Account Router / Load Balancer                     │ │ │
│  │  │  • Routes requests to available account                         │ │ │
│  │  │  • Detects rate limits (429) and fails over                     │ │ │
│  │  │  • Session-sticky when beneficial                               │ │ │
│  │  │  • Tracks rate limit reset times                                │ │ │
│  │  │  • Zero logging (privacy-preserving)                            │ │ │
│  │  └─────────────────────────────────────────────────────────────────┘ │ │
│  │                          │                   │                        │ │
│  │                 ┌──────────────┐     ┌──────────────┐                │ │
│  │                 │  Claude Max  │     │  Claude Max  │                │ │
│  │                 │  Account 1   │     │  Account 2   │                │ │
│  │                 │   (OAuth)    │     │   (OAuth)    │                │ │
│  │                 └──────────────┘     └──────────────┘                │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                      INTERFACE LAYER                                  │ │
│  │  Options: Terminal UI | Web dashboard | Desktop app | CLI             │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Data Flow Example

**User makes a request: "Research AI agent frameworks and write a comparison report"**

```
1. REQUEST ARRIVES → Orchestrator receives request
2. CONTEXT LOADING → Reads shared memory, loads relevant past context
3. PLANNING → Creates/updates plan in shared state
4. SUBAGENT SPAWNING → Routes agents to available accounts
5. SYNTHESIS → Collects results, updates shared memory
6. PERSISTENCE → All context persists in YOUR infrastructure
```

---

## PART 4: RESEARCH DIRECTIONS

> *Subsumed by RESEARCH.md (comprehensive report with 90+ sources) and SESSION_BRIDGE.md's T-shape inventory (23 verticals). The directions below were the starting points; see those documents for where research actually went.*

### 4.1 Memory Systems
- Mem0, LangMem, Zep, Motorhead, Custom solutions
- Vector databases: Qdrant, Pinecone, Weaviate, Chroma, Milvus

### 4.2 Planning & State Management
- File-based, Database, Message queues, Workflow engines

### 4.3 Orchestration & Agent SDKs
- Claude Agent SDK, OpenAI Agents SDK, LangGraph, CrewAI, AutoGen
- claude-flow, claude-code-by-agents, wshobson/agents

### 4.4 Multi-Account Routing
- Minimal secure proxy, OAuth token management, Rate limit detection

### 4.5 Subagent Architecture
- Leader/Worker, Pipeline, Debate, Watcher, Swarm, Hierarchical patterns

### 4.6 Interface Options
- Terminal UI, Web dashboard, Desktop app, VS Code extension, CLI

### 4.7 Model Context Protocol (MCP)
- Existing servers, Custom servers, MCP for orchestration/memory

---

## PART 5: USER REQUIREMENTS

### Hard Requirements
1. Privacy-First — All data under user control
2. Self-Hostable — Runs on user's infrastructure
3. Unified Experience — Two ~~accounts~~ providers feel like one
4. ~~Portable — Works from home, road, mobile~~ *Descoped from v1. Focus on local MacBook first.*
5. Extensible — Add and remove capabilities freely

### Soft Preferences
1. Prefer open-source tools
2. Minimal dependencies where possible
3. Well-documented, maintained projects
4. ~~Claude Agent SDK as foundation~~ *Deprioritized — SDK replaces Claude Code. Current direction: MCP server + hooks that augment Claude Code from inside.*
5. Open to custom components

### Success Criteria
- Seamless failover between ~~accounts~~ providers on rate limits
- No context loss when switching
- Subagents can use either ~~account~~ provider
- All history/plans/learnings persist externally
- ~~Single interface to manage everything~~ *Revised: Terminal is primary. Dashboard is ambient monitoring (menu bar widget + optional browser page).*
- ~~Notifications when work completes~~ *Descoped from v1.*
- ~~Works from anywhere~~ *Descoped from v1.*

---

## PART 6: IMPLEMENTATION PHASES

> *Outdated — these phases assumed dual-Claude architecture. See SESSION_BRIDGE.md's Investigation Roadmap for the current phase plan. We are still in deep investigation, not implementation.*

### Phase 1: Foundation (MVP)
- Research and select memory system
- Set up memory with vector storage
- Create minimal account router
- Basic orchestrator with shared memory
- CLI interface for testing

### Phase 2: Subagent Architecture
- Research subagent patterns in Claude Agent SDK
- Define roles and capabilities
- Implement spawning and parallel execution
- Result synthesis to shared state

### Phase 3: Planning System
- External plan storage format
- Plan CRUD operations
- Task queue for complex workflows
- Agent-driven plan updates

### Phase 4: Interface
- Monitoring dashboard
- Session management
- Notifications
- Portability

### Phase 5: Polish & Extension
- More specialized subagent types
- MCP integrations
- Reliability and recovery
- Documentation

---

## OPEN QUESTIONS

1. Memory granularity — per-conversation, per-project, or global? *Still open.*
2. Plan format — Markdown, JSON, or database records? *Still open.*
3. Subagent context — full history or summarized? *Still open.*
4. ~~Account selection — automatic, manual, or hint-based?~~ *Resolved: Claude always orchestrates. Subtask routing to Gemini is dynamic based on task type, context size, rate limits.*
5. Failure recovery — checkpointing strategy? *Still open.*
6. Concurrency limits — per-account agent limits? *Still open.*
7. Tool access — universal vs. role-specific? *Still open.*
8. ~~Interface priority — CLI-first or web-first?~~ *Resolved: CLI-first. Dashboard is secondary monitoring window (ambient menu bar widget + localhost browser page).*
9. Testing strategy for multi-agent systems? *Still open.*
10. Cost/usage tracking across accounts? *Still open — dashboard design addresses this.*

*See ARCHITECTURE.md § "OPEN ARCHITECTURE QUESTIONS" and SESSION_BRIDGE.md § "PROVOCATIVE QUESTIONS" for the current expanded question set.*

---

*Handoff generated January 27, 2026. Context transfer from claude.ai to Claude Code CLI. Updated end-of-session with supersession notices, descoped items, and resolved questions. See SESSION_BRIDGE.md for current project state.*
