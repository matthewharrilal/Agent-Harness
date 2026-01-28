# Agent Harness — Claude Code Instructions

## Project Phase: Deep Investigation (NOT Implementation)

This project is in active research and pre-implementation planning. No code has been written. Nothing is locked in. Your job is to think, challenge, research, and go deep — not to build.

## Ground Rules

1. **NEVER edit a file without first explaining what you plan to change and why, then getting approval.** This applies to every document edit, no exceptions. Talk through your reasoning, wait for a yes.
2. **Do not write code or propose implementations** unless explicitly asked. We are investigating, not building.
3. **Be honest about uncertainty.** "I don't know" and "I haven't researched this" beat false confidence every time.
4. **Think independently.** Challenge what you find in the docs. Push back on assumptions. Be a sparring partner, not a yes-man.
5. **Ask questions liberally.** Use AskUserQuestion when you need alignment. Don't assume.
6. **Track all findings** in the markdown files. If you discover something, document it.
7. **Go deep, not broad.** 19 research agents already covered the landscape. What's needed is depth.
8. **Use sub-agents for research.** Spawn them for investigations. Don't try to do everything inline.
9. **Do not summarize docs back to the user.** They wrote them. Give your own perspective.

## What This Project Is

A hybrid Claude + Gemini orchestration layer. The user hits 85% of their weekly Claude Max rate limit in 1-2 days. The harness combines Claude Code with Gemini CLI so they feel like one system — shared memory, intelligent routing, invisible middleware. The user types `claude` normally and the harness is baked in.

## What's Decided (Don't Re-Litigate)

- Hybrid Claude+Gemini, not dual Claude (device keychain bug makes it broken)
- Claude is always the top-level orchestrator
- Gemini at free tier or AI Pro ($0-20/mo), NOT Ultra ($250/mo)
- The harness is invisible — user types `claude` normally
- Cross-model critique is user-triggered or complexity-triggered, not every task
- Both providers exhausted = notify and stop

## What's Open (Your Thinking Is Needed)

- **Is the 40% gap worth building for?** Existing tools (CCProxy, PAL MCP, CodexBar) cover ~55-60% of the vision. The genuine gap is semantic routing, persistent memory, proactive rate prediction, and cross-model critique. Is that gap worth months of building? Or should the user just install existing tools? This is THE question.
- MCP server as mechanism (Level 3 confidence — "let's not double down on MCP yet")
- Memory system (Mem0? ChromaDB? sqlite-vec? Files?)
- Visual interface value for a terminal-native user
- How Claude decides what to delegate
- How this differs from PAL MCP + CCProxy + CodexBar (zero-code baseline)
- MVP scope
- Whether we're investigating or already designing

## Document Reading Order

1. **docs/SESSION_BRIDGE.md** — Primary orientation. Confidence registry, narrative, open questions, conduct guide.
2. **docs/ARCHITECTURE.md** — Technical decisions. MCP design, compensation list, gap analysis, competitive landscape reality check. Living draft, not settled.
3. **docs/RESEARCH.md** — Evidence base. 90+ sources. Competitive landscape deep-dive (30+ tools). Use as reference.
4. **docs/HANDOFF.md** — Original vision. Part 3 (architecture) is superseded. Parts 1 and 5 still valid.

## How to Start a New Session

The user will copy-paste the contents of `NEXT_SESSION_PROMPT.md` as their first message. It contains specific open questions, context, and marching orders. Follow its instructions — do not re-read it as a file since you already have the content.

## Do NOT

- Edit files without explaining and getting approval first
- Write code or propose implementations
- Treat ARCHITECTURE.md as settled
- Rush to conclusions
- Go broad when depth is needed
- Add API key vs subscription distinction to any documents
- Conflate "I researched this" with "this is true" — many findings are single-source
