# Agent Harness

A hybrid Claude + Gemini orchestration layer that makes two AI providers feel like one unified system.

## The Problem

Claude Max 20x ($200/mo) gives substantial inference capacity — but heavy users hit 85% of weekly rate limits in 1-2 days. When that happens, work stops. Getting a second Claude account doesn't help: a [device-level keychain bug](https://github.com/anthropics/claude-code/issues/19456) makes dual-Claude broken on one machine (6+ GitHub issues, unfixed as of January 2026).

## The Solution

Combine Claude Code with Gemini CLI into a single invisible middleware layer. The user types `claude` normally. Behind the scenes:

- **Intelligent task routing** — Claude handles precision work (debugging, architecture, long-horizon agents). Gemini handles breadth work (1M-token codebase analysis, multimodal, web-grounded research).
- **Shared memory** — Both providers read/write to the same external memory system. No context loss when switching.
- **Rate limit awareness** — The harness knows which provider has capacity and routes accordingly.
- **Cross-model critique** — Based on [arxiv 2601.14351](https://arxiv.org/abs/2601.14351) ("Team of Rivals"): single-agent self-review degrades accuracy (<60%), while cross-model critique achieves 90%. Claude and Gemini catch each other's blind spots.
- **Ambient monitoring** — CodexBar-style menu bar widget and/or localhost dashboard for rate limits, costs, and routing analytics.

Claude always orchestrates. Gemini is the specialist. The harness is the sous chef that coordinates both kitchens.

## Current Status: Deep Investigation Phase

**No code has been written.** This project is in active research and pre-implementation planning. We are investigating architecture, evaluating tradeoffs, and answering open questions before building anything.

### What Exists

| Document | Purpose |
|----------|---------|
| `SESSION_BRIDGE.md` | Primary continuity document. Narrative history, decision confidence registry, open questions, investigation roadmap, session conduct guide. **Read this first.** |
| `ARCHITECTURE.md` | Technical decisions and reflections. MCP server architecture, 21-item Gemini compensation list, Swarms/TeammateTool research, gap analysis (61 gaps). Living draft. |
| `RESEARCH.md` | Evidence base from 19 research agents across 3 rounds. 90+ sources. Device rate limit analysis, Gemini CLI gap audit, ecosystem map, dashboard patterns. |
| `HANDOFF.md` | Original vision from claude.ai conversation. Part 3 (dual-Claude architecture) is superseded. Parts 1 and 5 (vision, requirements) still valid. |
| `NEXT_SESSION_PROMPT.md` | Comprehensive handoff prompt for a fresh Claude Code instance to pick up the investigation. |

### Key Decisions Made

- **Hybrid Claude+Gemini** — not dual-Claude (device keychain bug), not API keys (subscription value)
- **Claude always orchestrates** — Gemini is a specialist subprocess, never the lead
- **Invisible middleware** — runs as MCP server + hooks inside interactive Claude Code, not a CLI wrapper
- **CLI-first** — terminal is the control plane; dashboard is ambient monitoring only
- **Cross-model critique on demand** — user-triggered or complexity-triggered, not every task

### Key Open Questions

1. **MCP server vs Bash + file I/O** — Why is an MCP server better than `gemini -p 'task' > result.md`?
2. **Visual interface value** — What does a dashboard provide a terminal-native user that CLI output cannot?
3. **Zero-code baseline** — Does PAL MCP + CodexBar + CLAUDE.md already cover 70% of this?
4. **Single-provider exhaustion** — When only Claude is rate-limited, can Gemini handle the workload?
5. **Memory system** — Mem0 vs ChromaDB vs sqlite-vec vs plain files?

### Research Highlights

- **Device keychain bug**: macOS Keychain namespace collision (`Claude Code-credentials`) causes rate limits from Account A to bleed into Account B. [6+ GitHub issues](https://github.com/anthropics/claude-code/issues/19456), unfixed.
- **Claude Code Swarms**: Hidden feature-flagged multi-agent system (TeammateTool, 13 operations, 5 org patterns). Claude-only — complements the harness, doesn't replace it. [Source](https://github.com/mikekelly/claude-sneakpeek)
- **Team of Rivals paper**: Cross-model critique (Claude reviews Gemini, Gemini reviews Claude) achieves 90% accuracy vs <60% for self-review. Validates the hybrid approach. [arxiv 2601.14351](https://arxiv.org/abs/2601.14351)
- **21 Gemini gaps identified**: Plan mode (4), subagent/task (4), tool bridging (2), MCP compat (3), config/memory (3), headless integration (3), behavioral parity (2).
- **61 architectural gaps cataloged**: 13 blockers, 38 complicators, 10 nice-to-have. None resolved yet.

## Architecture (Working Hypothesis — Not Locked In)

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

## Contributing

This project is in its investigation phase. If you're interested in the hybrid Claude+Gemini orchestration problem, the research documents in this repo contain comprehensive analysis of the landscape, existing tools, and architectural considerations.

## Budget

$200 (Claude Max 20x) + $0-20 (Gemini free/AI Pro) = **$200-220/mo**

## Inspiration

- [@omarsar0's custom agent orchestrator](https://x.com/omarsar0) — "a control center for my agents"
- [PAL MCP Server](https://github.com/BeehiveInnovations/pal-mcp-server) (10K+ stars) — provider abstraction layer
- [CodexBar](https://github.com/steipete/CodexBar) — macOS menu bar rate limit widget
- [arxiv 2601.14351](https://arxiv.org/abs/2601.14351) — "If You Want Coherence, Orchestrate a Team of Rivals"
