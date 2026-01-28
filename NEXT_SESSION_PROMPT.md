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

`CLAUDE.md` at the project root is auto-loaded by Claude Code — you already have it. No need to read it again.

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
End-of-session research found the competitive landscape is closer than expected. CCProxy does rule-based routing with Max subscription support (~65% coverage). claude-code-mux does auto-failover with OAuth (~60%). PAL MCP + CCProxy + CodexBar combined gets ~55-60% of the vision for zero custom code. The genuinely novel gap is ~40%: semantic task routing, persistent cross-provider memory, proactive rate prediction, cross-model critique. **Is that 40% worth building for? Or should I just install the existing tools?** See docs/RESEARCH.md § "COMPETITIVE LANDSCAPE DEEP-DIVE" for the full tool-by-tool analysis. This is the question everything else hangs on.

### 1. How Do API Gateways Actually Do "Intelligent Routing"?
OpenRouter, Portkey, LiteLLM — I'm told they do "intelligent routing." How, concretely? What's the mechanism? Is it keyword matching, embeddings, learned routing, or just rule-based fallback? And is Cursor an example of this kind of tool — where you pick from a list of models — or is it something different? Where does Cursor land in the 5 tool categories listed at the bottom of this document? Answering this helps me understand what "routing" even means in practice and whether what exists is close to what we're describing.

### 2. Multi-Agent Managers: Who Invokes What?
Claude Squad and Conductor Build show multiple instances in a visual interface. But what's the source of truth? Does the GUI invoke and control the instances (it's the creator), or does it read state from somewhere underneath (it's a viewer)? Is there a stack trace or process state that these GUIs tap into, or is the visual interface the thing that spawns and manages the agents? This matters because it tells me whether "multi-agent management" is fundamentally a UI concern or an infrastructure concern — and whether a harness needs to provide that infrastructure layer for any UI to read from.

### 3. Provider Abstraction Layers: What Do You Actually Lose?
PAL MCP's `clink` tool lets Claude Code delegate to Gemini from inside a conversation. But it has no shared memory, no routing intelligence, no rate limit awareness — I manually decide what to delegate. Paint me concrete user journey scenarios showing what that actually looks like day-to-day. What breaks? What's friction? What do I lose by NOT having shared memory, routing intelligence, and rate limit awareness? And then contrast: what would those same scenarios look like WITH a full harness? I need to feel the gap, not just be told it exists.

### 4. The Competitive Landscape: What Each Tool Does AND Doesn't Do
The README lists tools (CCProxy, claude-code-mux, CLIProxyAPI, PAL MCP, hcom) with what they do and a coverage percentage. But I'm confused about what each one DOESN'T do — because some of them sound like they already do what we want.

**The specific confusion (use this as the starting point):** CCProxy's own README says: *"ccproxy unlocks the full potential of your Claude MAX subscription by enabling Claude Code to seamlessly use unlimited Claude models alongside other LLM providers like OpenAI, Gemini, and Perplexity. It works by intercepting Claude Code's requests through a LiteLLM Proxy Server, allowing you to route different types of requests to the most suitable model — keep your unlimited Claude for standard coding, send large contexts to Gemini's 2M token window, route web searches to Perplexity, all while Claude Code thinks it's talking to the standard API."* That sounds almost exactly like what we described wanting. Route by task type, keep the subscription, use multiple providers, invisible to the user. **Where am I wrong?** What specifically does CCProxy NOT do that makes us say it's only ~65% coverage? Be precise — don't just say "no semantic routing." Show me: here's a real workflow where CCProxy handles it fine, and here's a real workflow where it falls apart and what happens when it does.

Do this same does/doesn't analysis for EACH tool in the competitive landscape table. For each one: (a) concrete scenario where it works well, (b) concrete scenario where it falls short — what breaks, what's missing, what do I have to do manually, (c) what would that "falls short" scenario look like if the harness existed? This is how I'll understand what gap we're actually building for. Don't be abstract — walk me through real workflows.

### 5. MCP Server vs. Bash + File I/O
"Why is an MCP server better than `gemini -p 'task' > result.md` followed by a Read?" Claude can already run Bash commands and read files. What specifically does an MCP server provide that Bash + file I/O does not? Be concrete — not "better integration" but actual capabilities gained and lost. This is the single most impactful architectural question.

### 6. What Does a Visual Interface Actually Provide on Top of CLI?
The docs describe 5 dashboard panels and a CodexBar widget. But I'm a terminal-native power user. What specific information or capability would a visual interface give me that `cat .harness/status.json` or a well-formatted CLI output cannot? What is genuinely worth the context-switching cost of a browser tab?

### 7. How Does This Differ From PAL MCP + CCProxy + CodexBar?
Today, right now, you can: install PAL MCP for Gemini delegation, CCProxy for rule-based routing with subscription auth, and CodexBar for rate limit monitoring. That gets ~55-60% of the vision. What specific workflows fail? What's the 40% gap in practice, and does it matter day-to-day? Be ruthlessly honest.

### 8. The Single-Provider Exhaustion Scenario
Both-providers-exhausted is handled (notify and stop). But the common case is: Claude is rate-limited, Gemini still has capacity. What happens? Does the harness auto-route everything to Gemini? Does it ask me? Can Gemini even handle Claude's typical workload given its gaps (no mature subagents, no production plan mode, 4x more control-flow errors)? Walk through this scenario concretely.

### 9. Are We Investigating or Designing?
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

## The 5 Tool Categories (Reference for Questions 1-3)

Previous session research identified 5 distinct categories of tools in this space. They do different things. Understanding where each tool lands — and where our harness sits — is critical context for the open questions above.

**1. API Gateways (OpenRouter, LiteLLM, Portkey)**
Route API requests to different models. You send a prompt, they pick the best model or failover. They DO intelligent routing. They DO let you use Claude AND Gemini AND GPT together. But — they work with API keys and pay-per-token pricing. They do NOT work with your Claude Max subscription. You can't send your $200/mo subscription credits through OpenRouter. The OAuth crackdown killed that.

**2. Multi-Agent Managers (Claude Squad, Conductor Build)**
Run multiple instances of the SAME provider. Claude Squad = multiple Claude Code sessions in tmux. Conductor Build = multiple Claude Code instances in parallel git worktrees. They parallelize work. They do NOT route between different providers. They're multipliers, not routers.

**3. Model Switchers (claude-code-proxy, claude-code-router)**
Let you swap which model powers Claude Code. Point Claude Code at Gemini via OpenRouter. But you're using ONE model at a time — substituting, not combining.

**4. Provider Abstraction Layers (PAL MCP)**
The closest thing to what you want. PAL's `clink` tool lets Claude Code delegate to Gemini from inside a conversation. But it's a delegation tool, not a system — no shared memory, no routing intelligence, no rate limit awareness. You manually decide what to delegate.

**5. Orchestration Frameworks (LangGraph, CrewAI, Google ADK)**
Build custom multi-agent applications that CAN use multiple providers with shared memory. But they REPLACE Claude Code entirely — you build your own app. You lose the interactive Claude Code experience you use every day.

**Where the harness sits:** The intersection of subscription-based Claude Code + intelligent routing to Gemini + shared memory + invisible middleware + rate limit awareness. Each individual piece exists somewhere in these 5 categories. The specific combination — invisible, inside Claude Code, with all pieces working together — does not.

**Where does Cursor fit?** This is an open question for the next session to answer. Cursor lets you choose models (GPT-4, Claude, etc.) from a dropdown. Is it a model switcher? An API gateway? Something else? Understanding Cursor's architecture illuminates what "model selection" means in practice.

## Start

Read the four files. Then tell me: what stands out to you, what questions you'd ask, and where you think the investigation should go next.
