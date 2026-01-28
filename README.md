# Agent Harness

A hybrid Claude + Gemini orchestration layer that makes two AI providers feel like one unified system.

## The Problem

Claude Max 20x ($200/mo) gives substantial inference capacity — but heavy users hit 85% of weekly rate limits in 1-2 days. When that happens, work stops. Getting a second Claude account doesn't help: a [device-level keychain bug](https://github.com/anthropics/claude-code/issues/19456) makes dual-Claude broken on one machine (6+ GitHub issues, unfixed as of January 2026).

## The Solution (Under Investigation)

Combine Claude Code with Gemini CLI into a single invisible middleware layer. The user types `claude` normally. Behind the scenes:

- **Intelligent task routing** — Claude handles precision work (debugging, architecture). Gemini handles breadth work (1M-token codebase analysis, multimodal, research).
- **Shared memory** — Both providers read/write to the same external memory system.
- **Rate limit awareness** — The harness knows which provider has capacity and routes accordingly.
- **Cross-model critique** — Based on [arxiv 2601.14351](https://arxiv.org/abs/2601.14351): single-agent self-review degrades accuracy (<60%), while cross-model critique achieves 90%.
- **Ambient monitoring** — CodexBar-style menu bar widget and/or localhost dashboard for rate limits, costs, and routing analytics.

Claude always orchestrates. Gemini is the specialist. The harness is the sous chef that coordinates both kitchens.

## Current Status: Deep Investigation Phase

**No code has been written.** This project is in active research and pre-implementation planning. We are investigating architecture, evaluating tradeoffs, and answering open questions before building anything.

### The Honest Competitive Landscape

End-of-session research (3 deep agents, 30+ tools evaluated) revealed the landscape is **closer to this vision than initially expected**. Several existing tools cover parts of what we described:

| Tool | What It Does | Coverage |
|------|-------------|:--------:|
| [CCProxy](https://github.com/starbased-co/ccproxy) | Rule-based routing with Max subscription support (token count, thinking mode, tool use) | ~65% |
| [claude-code-mux](https://github.com/9j/claude-code-mux) | Rust proxy with OAuth support, auto-failover, 18+ providers, <1ms overhead | ~60% |
| [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) | macOS menu bar proxy with OAuth subscription support and real-time quota tracking | ~60% |
| [PAL MCP](https://github.com/BeehiveInnovations/pal-mcp-server) | Manual delegation from Claude Code to Gemini, conversation threading, context revival | ~45% |
| [hcom](https://github.com/aannoo/hcom) | Hook-based inter-agent messaging between Claude Code and Gemini CLI | ~35% |

**Best zero-code combo today: PAL MCP + CCProxy + CodexBar = ~55-60% of the vision.**

### The Genuine Gap (~40%)

What does NOT exist anywhere:

1. **Semantic task-type routing** — "This is a code review, send to Claude. This is research, send to Gemini." Existing tools route by token count or model name, not by what the task IS.
2. **Persistent cross-provider memory** — A knowledge store both Claude and Gemini read/write to across sessions and days, with semantic search.
3. **Proactive rate limit prediction** — Every tool REACTS to 429 errors. Nobody PREDICTS "Claude is at 80%, route the next task to Gemini preemptively."
4. **Cross-model critique** — Nobody implements "Claude generates, Gemini reviews" automatically. The Team of Rivals pattern is unimplemented anywhere.
5. **The integrated "sous chef"** — All of the above working together invisibly inside interactive Claude Code.

### The Core Question

**Is the 40% gap worth building for, or should we start with the 60% that exists and see if it's enough?**

The zero-code baseline test (install PAL MCP + CCProxy + CodexBar, use them for a week) would answer this from lived experience rather than theory. This is likely the most important next step.

## Documents

| Document | Purpose |
|----------|---------|
| `SESSION_BRIDGE.md` | Primary continuity document. Narrative history, decision confidence registry, open questions, investigation roadmap, session conduct guide. **Read this first.** |
| `ARCHITECTURE.md` | Technical decisions and reflections. MCP server architecture, 21-item Gemini compensation list, Swarms/TeammateTool research, gap analysis (61 gaps). Living draft — not settled. |
| `RESEARCH.md` | Evidence base from 19 research agents across 3 rounds. 90+ sources. Device rate limit analysis, Gemini CLI gap audit, ecosystem map, competitive landscape deep-dive. |
| `HANDOFF.md` | Original vision from claude.ai conversation. Part 3 (dual-Claude architecture) is superseded. Parts 1 and 5 (vision, requirements) still valid. |
| `NEXT_SESSION_PROMPT.md` | Comprehensive handoff prompt for a fresh Claude Code instance to pick up the investigation. |
| `CLAUDE.md` | Instructions for any Claude instance working on this repo. Ground rules, project phase, reading order. |

### Key Decisions Made

- **Hybrid Claude+Gemini** — not dual-Claude (device keychain bug), not API keys (subscription value)
- **Claude always orchestrates** — Gemini is a specialist subprocess, never the lead
- **Invisible middleware** — runs as MCP server + hooks inside interactive Claude Code, not a CLI wrapper
- **CLI-first** — terminal is the control plane; dashboard is ambient monitoring only
- **Cross-model critique on demand** — user-triggered or complexity-triggered, not every task

### Key Open Questions

1. **Is the 40% gap worth building for?** Or does PAL MCP + CCProxy + CodexBar cover enough?
2. **MCP server vs Bash + file I/O** — Why is an MCP server better than `gemini -p 'task' > result.md`?
3. **Visual interface value** — What does a dashboard provide a terminal-native user that CLI output cannot?
4. **Single-provider exhaustion** — When only Claude is rate-limited, can Gemini handle the workload?
5. **Memory system** — Mem0 vs ChromaDB vs sqlite-vec vs plain files?

### Research Highlights

- **Device keychain bug**: macOS Keychain namespace collision (`Claude Code-credentials`) causes rate limits from Account A to bleed into Account B. [6+ GitHub issues](https://github.com/anthropics/claude-code/issues/19456), unfixed.
- **Claude Code Swarms**: Hidden feature-flagged multi-agent system (TeammateTool, 13 operations). Claude-only — complements the harness, doesn't replace it. [Source](https://github.com/mikekelly/claude-sneakpeek)
- **Team of Rivals paper**: Cross-model critique achieves 90% accuracy vs <60% for self-review. [arxiv 2601.14351](https://arxiv.org/abs/2601.14351)
- **21 Gemini gaps identified**: Plan mode (4), subagent/task (4), tool bridging (2), MCP compat (3), config/memory (3), headless integration (3), behavioral parity (2).
- **61 architectural gaps cataloged**: 13 blockers, 38 complicators, 10 nice-to-have. None resolved yet.
- **Competitive landscape**: 30+ tools evaluated. Best zero-code combo (PAL+CCProxy+CodexBar) covers ~55-60%. Genuine gap is ~40%.

## Architecture (Working Hypothesis — Level 3 Confidence, Not Locked In)

```
YOU type: claude (normal interactive mode)
  |
  v
CLAUDE CODE starts normally
  |
  ├── Reads .mcp.json → spawns "agent-harness" MCP server
  │   Tools: delegate_to_gemini, shared_memory_read/write,
  │          check_rate_limits, route_task
  │
  ├── Reads CLAUDE.md → behavioral instructions for delegation
  │
  ├── Native subagents (.claude/agents/) → Claude-to-Claude work
  │
  ├── Hooks → event capture to dashboard
  │
  └── Dashboard server (background process)
      localhost:3200 or CodexBar-style menu bar widget
```

## Budget

$200 (Claude Max 20x) + $0-20 (Gemini free/AI Pro) = **$200-220/mo**

## Inspiration

- [@omarsar0's custom agent orchestrator](https://x.com/omarsar0) — "a control center for my agents"
- [PAL MCP Server](https://github.com/BeehiveInnovations/pal-mcp-server) (10K+ stars) — provider abstraction layer
- [CodexBar](https://github.com/steipete/CodexBar) — macOS menu bar rate limit widget
- [CCProxy](https://github.com/starbased-co/ccproxy) — rule-based routing with subscription support
- [arxiv 2601.14351](https://arxiv.org/abs/2601.14351) — "If You Want Coherence, Orchestrate a Team of Rivals"
