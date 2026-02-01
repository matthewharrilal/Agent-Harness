# Naming and Scope: What Each Name Commits You To Building

> Date: January 31, 2026 (Session 5)
>
> **Why this matters:** The name isn't cosmetic — it defines the boundaries of what you're building. If the name is wrong, the scope creeps back to the 5-layer vision without anyone noticing.

---

## Names That Don't Fit Anymore

**"Harness"** — implies you're wrapping both tools, sitting on top, controlling them. That's an orchestration layer. You're not building that — Claude orchestrates itself.

**"Hybrid AI orchestration system"** — implies multi-model task routing, decomposition, planning, coordination. Most of that is handled natively by the tools. This name overpromises.

**"Smart rate-limit manager with a context bridge"** — this is the honest description of what's actually unique. But it's a description, not a name.

---

## The Naming Spectrum

Each name implies a different project with different scope, different responsibilities, and different build effort.

```
Bridge → Relay → Overflow → Continuity Layer
 narrow                              wide

 "sync context"  →  "package tasks"  →  "auto-route on rate"  →  "unify all tools"
  when I switch      when I delegate      so I never hit walls     regardless of why
```

---

## Concrete Scenarios for Each Name

### "Bridge"

**What you'd build:** A context translation layer between Claude and Gemini. The core problem it solves is: Claude knows things about your project that Gemini doesn't, and vice versa. The bridge keeps them in sync.

**Concrete scenario:** You've been working with Claude for 3 hours on a refactor. Claude understands your architecture, the decisions you've made, the files you've touched. You hit the rate limit. You open Gemini CLI. Without the bridge, Gemini starts cold — it knows nothing. With the bridge, Gemini opens and already has a synthesized context: "Here's the project, here's what was being worked on, here's the current state of the refactor, here are the decisions that were made and why." You keep working without re-explaining everything.

**What it does NOT do:** It doesn't decide when to switch. It doesn't route tasks. It doesn't predict rate limits. You manually switch, and the bridge makes the handoff smooth.

**Unique value:** Context continuity across providers. That's it. One job.

---

### "Relay"

**What you'd build:** A bridge (above) plus intelligence about *what* to hand off. It doesn't just preserve context — it packages specific tasks for the receiving model.

**Concrete scenario:** You're working with Claude. Claude decomposes a feature into 5 subtasks. You're approaching your rate limit (you check CodexBar manually). You tell the relay: "hand off subtasks 3 and 4 to Gemini." The relay doesn't just dump context — it packages each subtask with the relevant files, the dependencies, the acceptance criteria, and the context from subtasks 1-2 that Gemini needs. When Gemini finishes, the relay brings the results back into Claude's awareness, formatted so Claude can review and integrate them.

**What it does NOT do:** It doesn't decide *which* subtasks go to Gemini — you do. It doesn't predict rate limits — you check manually. It doesn't route automatically. You're the dispatcher; the relay is the logistics.

**Unique value:** Context continuity + intelligent task packaging for handoff. Two jobs.

---

### "Overflow"

**What you'd build:** A relay (above) plus rate limit awareness. The core metaphor is a water tank — Claude is the primary tank, Gemini is the overflow tank. When the primary is approaching capacity, work spills over automatically.

**Concrete scenario:** You're working with Claude. You don't think about rate limits at all. Behind the scenes, the overflow monitor tracks your consumption. At 70% capacity, it starts flagging subtasks that are "overflow-safe" — tasks where Gemini's quality is good enough (boilerplate generation, test scaffolding, documentation, straightforward CRUD). At 85%, it starts actually routing those tasks to Gemini silently. You never hit the wall. Your session feels continuous. Claude handles the hard stuff, Gemini handles the routine stuff, and you didn't have to think about any of it.

**What it does NOT do:** It doesn't orchestrate the overall project. It doesn't decompose tasks — Claude does that. It doesn't manage project workflow. Its intelligence is narrow: *when* to overflow and *what* is safe to overflow.

**Unique value:** Context continuity + task packaging + proactive rate prediction + automatic routing of suitable work. This is the "smart rate-limit manager with a context bridge."

---

### "Continuity Layer"

**What you'd build:** Something that goes beyond rate limits. The core problem it solves isn't "Claude is running out of capacity" — it's "I work across multiple AI tools and my context is fragmented."

**Concrete scenario:** You start a feature in Claude Code. You switch to Gemini CLI to use Conductor's structured planning. You switch back to Claude to execute. You open Cursor for a quick inline edit. Throughout all of this, the continuity layer maintains a unified project memory — decisions made, files changed, architectural context, task state. Every tool you open already knows what happened in the other tools. It's not about rate limits at all. It's about making multi-tool workflows feel like one continuous session.

**What it does NOT do:** It doesn't route, predict, or switch providers. It doesn't care *why* you switched tools. It just makes sure context follows you everywhere.

**Unique value:** Universal context persistence across any AI tool. Bigger scope than rate limits. Potentially more useful long-term, but also a much harder, more ambitious project.

---

## Harness vs. Overflow: The Full Comparison

### Harness (What We're NOT Building)

A harness **wraps both tools and controls them from above.** You don't talk to Claude or Gemini — you talk to the harness, and it decides what happens.

**Concrete scenario:** You type `harness "Build authentication for my Express app"`. The harness receives that prompt. It never reaches Claude directly. The harness:

1. **Decomposes the task itself** — breaks "build auth" into subtasks (user model, routes, middleware, tests, UI). The harness has its own decomposition logic, separate from Claude's plan mode.
2. **Classifies each subtask** — "user model creation is straightforward, route Gemini. Auth middleware is complex, route Claude. Test scaffolding is boilerplate, route Gemini."
3. **Dispatches to both models in parallel** — sends packaged prompts to Claude Code and Gemini CLI simultaneously, managing both processes.
4. **Manages communication between them** — Claude's middleware depends on Gemini's user model. The harness watches Gemini's output, validates it, then feeds it to Claude as input for the next task.
5. **Handles conflicts** — Claude's middleware and Gemini's routes both modified `app.js`. The harness detects the conflict, resolves or escalates it.
6. **Recomposes the result** — takes output from both models, merges branches, runs integration tests, presents you with a unified result.
7. **Manages rate limits, memory, context, project state** — all of it. The harness is the brain.

**You never interact with Claude or Gemini directly.** The harness is the interface. It's a full orchestration system that happens to use Claude and Gemini as execution backends.

---

### Overflow (What We ARE Building)

You **talk to Claude directly.** You type `claude` like you always do. Claude is the brain. The overflow sits quietly alongside, doing a narrow job.

**Same scenario:** You type into Claude Code: `"Build authentication for my Express app"`. Claude receives that directly — unchanged, unintercepted.

1. **Claude decomposes the task itself** — plan mode, tasks, subtasks. The overflow doesn't touch this.
2. **Claude starts executing** — writing the user model, routes, middleware. The overflow doesn't touch this either.
3. **The overflow watches your rate consumption** — in the background, it tracks that you've been going hard for 6 hours and you're at 75% of your weekly limit.
4. **At 80%, the overflow flags upcoming subtasks** — Claude's task list has "write test scaffolding" and "generate API docs" coming up. The overflow identifies these as overflow-safe.
5. **At 85%, the overflow routes those specific tasks to Gemini** — it packages the context (what Claude has built so far, what the tests need to cover, project conventions) and sends it to Gemini CLI.
6. **Gemini does the work, overflow brings results back** — the test files and docs come back into Claude's awareness. Claude continues with the hard tasks (security middleware, session management) without interruption.
7. **You never noticed.** Claude is still your interface. You're still typing into Claude. The overflow just extended your runway.

---

### The Difference in One Sentence

**Harness:** You talk to the harness, and it tells Claude and Gemini what to do.

**Overflow:** You talk to Claude, and the overflow quietly lends Gemini's capacity when Claude's is running low.

---

### Responsibility Comparison

| Responsibility | Harness | Overflow |
|---|---|---|
| Task decomposition | Harness does it | Claude does it |
| Routing decisions for every task | Harness decides | Only rate-pressure tasks |
| Being your primary interface | Yes — you talk to the harness | No — you talk to Claude |
| Managing both models as equal backends | Yes | No — Claude is primary, Gemini is overflow |
| Cross-model conflict resolution | Harness resolves | Minimal — overflow tasks are independent |
| Recomposition of parallel outputs | Complex merge from two models | Simple — bring Gemini's result back to Claude |
| Project state management | Harness owns it | Claude owns it |
| Working when rate limits aren't an issue | Yes — still routes for quality/cost | No — if Claude has capacity, overflow does nothing |

**The harness is a replacement for how you work. The overflow is an extension of how you already work.**

---

## Current Assessment

The capability subtraction analysis (see `WHAT_DOES_THIS_PROJECT_ACTUALLY_DO.md`) suggests **Overflow** is the honest sweet spot — it's where the unique value actually lives without overreaching into things other tools already do.

The **Continuity Layer** is interesting as a longer-term direction, because that value wouldn't evaporate if Anthropic changes rate limits. But it's a bigger, harder project.

The original "harness" vision was trying to be everything on the spectrum. That's no longer the plan.
