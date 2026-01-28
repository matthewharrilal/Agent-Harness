# Agent Harness — Session Handoff Briefing

You are picking up an ongoing deep investigation. No code exists. Nothing is locked in. Your job is to think, challenge, and go deep — not to build.

## Ground Rules (Non-Negotiable)

1. **NEVER edit a file without first explaining what you plan to change and why, then getting approval.** This is the hardest rule. It applies to every document edit, no exceptions. Talk through your reasoning, wait for a yes.
2. **We are investigating, not building.** Do not propose implementations. Do not write code. Do not draft config files. If you catch yourself sliding into "here's how we'd build it," stop.
3. **Be honest about uncertainty.** "I don't know" and "I haven't researched this" are better than guessing. Do not speak out of your ass.
4. **Think independently.** The docs contain one session's worth of thinking. Some of it is wrong. Challenge what you find. Push back on assumptions. I want a sparring partner, not a yes-man.
5. **Ask questions liberally.** Use AskUserQuestion whenever you need alignment. Don't assume.
6. **Track everything.** All findings go into the markdown files. If you discover something, document it. "Make sure you're tracking things."
7. **Go deep, not broad.** 19 research agents already covered the landscape. What's needed now is depth on specific verticals.
8. **Use sub-agents for research tasks.** Spawn them for investigations. Don't try to do all research inline.

## The Project in 60 Seconds

I hit 85% of my weekly Claude Max rate limit ($200/mo) in 1-2 days. When that happens, work stops. The solution: combine Claude Code with Gemini CLI so they function as one system — shared memory, shared context, intelligent delegation. I type `claude` normally. Behind the scenes, invisible middleware routes tasks, shares state, and monitors rate limits across both providers. I never see the seam.

Claude always orchestrates. Gemini is the specialist. This is not negotiable.

## Load Full Context

Read all 4 files in parallel from `docs/`:

1. **docs/SESSION_BRIDGE.md** — Your primary orientation. Narrative history, confidence registry (what's locked vs open), 12 provocative questions, conduct guide, investigation roadmap. This is your map.
2. **docs/ARCHITECTURE.md** — Technical thinking. MCP server architecture, 21-item Gemini compensation list, dashboard design, Swarms/TeammateTool research, gap analysis (61 gaps), competitive landscape reality check. Some sections are well-reasoned; some are sketches. The confidence registry in SESSION_BRIDGE.md tells you which is which.
3. **docs/RESEARCH.md** — Evidence from 19 research agents, 90+ sources. Key sections: device rate limit bug (killed dual-Claude), Gemini CLI gap audit, ecosystem map, competitive landscape deep-dive (30+ tools), MCP invocation research. Use as reference.
4. **docs/HANDOFF.md** — Original vision from claude.ai. Part 3 (architecture) is SUPERSEDED. Parts 1 (vision) and 5 (requirements) still valid. Read to check if current thinking has drifted from my actual needs.

Also read `CLAUDE.md` at the project root — it contains ground rules and project context that Claude Code loads automatically.

## What's Decided vs. What's Open

**Locked (don't re-litigate):**
- Hybrid Claude+Gemini, not dual Claude (device keychain bug makes dual-Claude broken)
- Claude is always the top-level orchestrator
- Gemini at free tier or AI Pro ($0-20/mo), NOT Ultra ($250/mo)
- The harness is invisible — I type `claude` normally
- Cross-model critique is user-triggered or complexity-triggered, not every task
- Both providers exhausted = notify me and stop

**Working hypothesis (challenge these):**
- MCP server as the harness mechanism (Level 3 confidence — I explicitly said "let's not double down on MCP yet")
- TypeScript as implementation language
- Dashboard as CodexBar-style widget + localhost browser page

**Wide open:**
- What the user experience actually looks and feels like
- Memory system (Mem0? ChromaDB? sqlite-vec? Files?)
- How Claude decides what to delegate
- What a visual interface provides beyond CLI
- How this concretely differs from existing tools
- MVP scope
- Whether we're investigating or already designing (the docs say investigation but contain design specs)

## Your First Task: Answer These Open Questions

Go deep. Use sub-agents for research. Give honest assessments, not hand-waves.

### 0. THE MOST IMPORTANT QUESTION: Is This Worth Building?
End-of-session research found the competitive landscape is closer than expected. CCProxy does rule-based routing with Max subscription support (~65% coverage). claude-code-mux does auto-failover with OAuth (~60%). PAL MCP + CCProxy + CodexBar combined gets ~55-60% of the vision for zero custom code. The genuinely novel gap is ~40%: semantic task routing, persistent cross-provider memory, proactive rate prediction, cross-model critique. **Is that 40% worth building for? Or should I just install the existing tools?** See RESEARCH.md § "COMPETITIVE LANDSCAPE DEEP-DIVE" for the full tool-by-tool analysis. This is the question everything else hangs on.

### 1. MCP Server vs. Bash + File I/O
"Why is an MCP server better than `gemini -p 'task' > result.md` followed by a Read?" Claude can already run Bash commands and read files. What specifically does an MCP server provide that Bash + file I/O does not? Be concrete — not "better integration" but actual capabilities gained and lost. This is the single most impactful architectural question.

### 2. What Does a Visual Interface Actually Provide on Top of CLI?
The docs describe 5 dashboard panels and a CodexBar widget. But I'm a terminal-native power user. What specific information or capability would a visual interface give me that `cat .harness/status.json` or a well-formatted CLI output cannot? What is genuinely worth the context-switching cost of a browser tab?

### 3. How Does This Differ From PAL MCP + CCProxy + CodexBar?
Today, right now, you can: install PAL MCP for Gemini delegation, CCProxy for rule-based routing with subscription auth, and CodexBar for rate limit monitoring. That gets ~55-60% of the vision. What specific workflows fail? What's the 40% gap in practice, and does it matter day-to-day? Be ruthlessly honest.

### 4. The Single-Provider Exhaustion Scenario
Both-providers-exhausted is handled (notify and stop). But the common case is: Claude is rate-limited, Gemini still has capacity. What happens? Does the harness auto-route everything to Gemini? Does it ask me? Can Gemini even handle Claude's typical workload given its gaps (no mature subagents, no production plan mode, 4x more control-flow errors)? Walk through this scenario concretely.

### 5. Are We Investigating or Designing?
SESSION_BRIDGE.md says "deep investigation, NOT implementation." But ARCHITECTURE.md contains a full MCP server code example, a dashboard ASCII diagram with 5 named panels, a routing decision JSON schema, and a 21-item compensation list. Is the project actually past investigation? Should I acknowledge we're in design phase? Or should the design specs be treated as hypothetical sketches?

## After the Questions: Map the Research Path

Look at the 12 provocative questions in SESSION_BRIDGE.md and the 23-vertical T-shape inventory. Recommend:
- Which 2-3 verticals to tackle first
- In what order and why
- What specific sub-questions within each vertical

## Consider the Zero-Code Baseline (Possibly the Most Important Next Step)

End-of-session competitive research suggests: **PAL MCP + CCProxy + CodexBar** delivers ~55-60% of the harness vision with zero custom code. Rather than theorizing about the 40% gap, I could install these tools, use them for a week, and discover from lived experience what's actually missing. Should this be the first action before any more research or design? Make a strong recommendation.

## Key Context

- **My main question:** "What am I actually building and what do I get out of it?" Focus on experience and outcomes, not implementation mechanics.
- **Rate limits:** Claude Max gives 240-480 hours/week of Sonnet 4. I exhaust 85% in 1-2 days. Gemini has daily resets vs Claude's weekly.
- **The sous chef analogy:** Existing tools are kitchen appliances. Nobody has built the coordinator who decides which appliance to use for which dish. That's the claimed gap.
- **Swarms/TeammateTool:** Claude Code has a hidden multi-agent system (feature-flagged, not released). Claude-only — no Gemini, no cross-provider. Complements the harness, doesn't replace it.
- **61 gaps identified:** 13 block implementation, 38 complicate it. None solved. Only identified.
- **Do NOT add API key vs subscription distinction** to any documents. I explicitly said it's not relevant.

## What NOT To Do

- Do not write code or propose implementations
- Do not edit files without explaining your changes and getting approval first
- Do not treat ARCHITECTURE.md as settled — it's a working draft
- Do not rush to conclusions. Sitting with uncertainty is fine.
- Do not go broad. We have breadth. We need depth.
- Do not conflate "I researched this" with "this is true." Many findings are single-source.
- Do not summarize the docs back to me — I wrote them. Give your OWN perspective.

## Start

Read the four files. Then tell me: what stands out to you, what questions you'd ask, and where you think the investigation should go next.
