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

## Document Reading Order (Updated Jan 29, 2026)

1. **docs/SESSION_3_SYNTHESIS.md** — **START HERE.** Current state after 5-agent deep research. All locked decisions, open questions, and build order.
2. **docs/ARCHITECTURE.md** — Technical decisions and 21-item compensation list. Living draft.
3. **docs/SESSION_BRIDGE.md** — Meta-layer with confidence registry and session logs.
4. **docs/RESEARCH.md** — Evidence base (90+ sources). Use as reference when you need to verify or challenge findings.

**Archived (do not start here):**
- `docs/archive/HANDOFF.md` — Historical. Dual-Claude architecture superseded.
- `docs/archive/SESSION_2_SYNTHESIS.md` — Merged into SESSION_3_SYNTHESIS.md.

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
