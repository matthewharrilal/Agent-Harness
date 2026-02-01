# What Does This Project Actually Do?

> Date: January 31, 2026 (Session 5)
>
> **How we got here:** Started by comparing Gemini Conductor to Claude's plan mode + task decomposition. Realized Conductor is a structured workflow extension (slash commands + prompt templates), not a prompt enrichment layer. Realized Claude's plan mode + task system + upcoming TeammateTool/swarm covers decomposition and orchestration natively. Stepped back to first principles: if we give every capability to the tool that does it best, what's left for this project?

---

## The Capability Subtraction

Strip away every capability that a better tool already handles:

| Capability | Who does it better | Status |
|---|---|---|
| Task decomposition | Claude (plan mode + tasks, soon swarm) | Native, maturing |
| Code execution | Claude Code / Gemini CLI | Both solid |
| Subagent orchestration | Claude (Task tool, TeammateTool coming) | Native |
| Project management workflow | Conductor (Gemini) / Plan mode (Claude) | Exists |
| Parallel execution | Conductor Build | Exists |
| Rate limit visibility | CodexBar | Exists |

After that subtraction, what's left?

---

## The Answer

**It's a rate-limit-aware context bridge between Claude Code and Gemini CLI.**

That's it. Not an orchestrator — Claude orchestrates. Not a decomposer — Claude decomposes. Not a project manager — those exist. It's a bridge that:

1. **Watches your Claude rate limit** and knows when you're approaching the wall
2. **Proactively routes suitable work to Gemini** before you hit it (not after, like CCProxy)
3. **Gives Gemini enough context** from Claude's ongoing work to be effective on the routed task
4. **Brings Gemini's results back** into Claude's awareness so continuity isn't broken

---

## Is That Useful?

Here's where we have to be honest rather than cheerleading.

### The case for yes

You burn through Claude Max in 1-2 days. That means 3-5 days per week you're either degraded, waiting, or manually context-switching to another tool. If a bridge handles that invisibly — you keep typing `claude`, it quietly offloads some work to Gemini, you never hit the wall, results come back seamlessly — that's a real quality-of-life improvement for your specific usage pattern. You're not an edge case; this is your *normal operating condition*.

### The case for "maybe not"

The zero-code baseline hasn't been tested. Just using Claude until limited, then manually opening Gemini CLI with a copy-pasted context summary, might get you 80% of the value with 0% engineering. The context bridge problem (translating Claude's internal understanding to something Gemini can use) is harder than it sounds and fragile — both tools update weekly. And if Anthropic changes rate limits, or ships multi-model support, the project's value could evaporate overnight.

### The case for no

"Cross-model routing" as the sole unique capability is thin. Nobody does it because nobody has proven it's worth the complexity. The routing classifier itself costs tokens and adds latency. And Gemini at free tier, while capable, is still a downgrade on harder tasks — so routing "suitable work" requires actually knowing what's suitable, which is itself a hard problem that may not have a clean answer.

---

## The Word "Harness" Is Wrong Now

A harness implies wrapping and controlling both tools. What you're actually describing is more like a **relay** or **bridge** — it sits between you and two providers, maintaining context continuity when work shifts from one to the other.

---

## The Uncomfortable Questions

These are the questions that actually matter before deciding whether to build:

1. **Have you tried the manual version?** Working in Claude until limited, then opening Gemini CLI with a context dump. If that's tolerable, the bridge isn't worth building. If it's painful enough that you avoid Gemini entirely and just wait for Claude to recover, that's signal that the bridge has real value.

2. **What's the actual quality delta?** If Gemini handles 70% of your tasks at equivalent quality, the bridge is valuable — it's offloading real work. If Gemini handles 30% at equivalent quality and you'd notice the degradation on the rest, the bridge is mostly just delaying your rate limit by a little.

3. **Is proactive routing actually better than reactive?** CCProxy switches *after* you're limited. That means Claude does 100% of work until the wall, then Gemini takes over. Your bridge would switch *before* the wall, meaning sometimes Gemini does work that Claude could have done. Is the smoothness worth the quality tradeoff?

4. **How much engineering are you willing to maintain?** Both tools ship updates constantly. A bridge between them is a third moving piece. That's ongoing maintenance tax.

---

## Honest Assessment

The project has a real use case for your specific situation (heavy user, predictable rate exhaustion, Gemini available as overflow). But it's narrower than the original 5-layer architecture vision suggested. The genuine unique value is:

- **Proactive rate prediction** (nobody does this)
- **Context continuity across providers** (nobody does this)
- **Invisible provider switching** (CCProxy does reactive, not proactive)

Everything else — decomposition, orchestration, planning, critique — has a native or existing solution that's better than what you'd build.

**The project isn't "a hybrid AI orchestration system." It's "a smart rate-limit manager with a context bridge."** And the question is whether that's exciting enough to build, or whether you'd rather just install CCProxy and live with reactive switching.

---

## Key Research That Led Here

### Gemini Conductor (what it actually is)
- External Gemini CLI extension by Google (not built-in), published Dec 17, 2025
- Six TOML slash commands with massive prompt templates (setup, newTrack, implement, status, revert, review)
- Writes persistent state to disk (product.md, tech-stack.md, workflow.md, spec.md, plan.md)
- Does NOT intercept or enrich freeform prompts — only activates via explicit slash commands
- Enforces TDD, coverage gates, phase verification through prompt engineering
- It's "follow this recipe" — prescriptive workflow enforcement

### Claude Plan Mode + Task Decomposition (what it actually is)
- Built-in mode toggle (Shift+Tab twice or `--permission-mode plan`)
- Restricts tools to read-only, injects system reminder preventing edits
- Model decides how to decompose work (freeform, not prescriptive)
- TaskCreate/TaskUpdate support dependency graphs, agent ownership
- Plan subagent researches in isolated context window
- It's "figure out the recipe, then follow it" — emergent structure from model reasoning

### Claude TeammateTool / Swarm (feature-flagged, not released)
- Found in Claude Code binary: `spawnTeam`, `assignTask`, `broadcastMessage`, `voteOnDecision`
- Five swarm patterns: The Hive, The Specialist, The Council, The Watchdog
- Combined with TaskCreate enables multi-agent coordination with dependency management
- Behind feature flags — signals Anthropic's direction but not usable yet

### The Comparison
- Conductor and Claude plan mode solve the same problem: preventing the model from diving into code without thinking
- Conductor is rigid and opinionated (external extension, explicit workflow)
- Claude's system is flexible and model-driven (built-in, emergent behavior)
- Both are fundamentally prompt engineering — neither has a separate "planning engine"
- Claude's system is more mature for decomposition and has subagent orchestration that Conductor lacks
