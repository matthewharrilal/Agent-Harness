# Agent Harness: A User Journey (Speculative)

**Status:** Investigative scenario. This explores *what might enable* and *what new complexities emerge*, not what will definitely happen.

> **Architecture context:** This scenario illustrates the 5-layer architecture described in [`docs/ARCHITECTURE_VISION.md`](ARCHITECTURE_VISION.md). Routing, memory, and communication decisions referenced here are investigation lanes, not locked implementations.

**Framing:** This is one developer's Wednesday morning with the hypothetical Agent Harness installed. Contrasts current reality (today, without harness) versus hypothetical reality (with harness). Shows both what becomes easier AND what requires new trust/debugging skills.

---

## THE SESSION: Wednesday 9 AM

### Scenario: Code Review + Architecture Refactoring + Documentation

The developer has 4 tasks stacked:
1. Review a 2,500-line authentication module for security gaps
2. Refactor the JWT validation to support multiple key formats
3. Analyze how this impacts the rest of a 140-file codebase
4. Update documentation to reflect changes

**This is a mixed-mode task: security review (Gemini's strength: large context), precision coding (Claude's strength), cross-cutting analysis (Gemini's 1M context window advantage).**

---

## TODAY (Without Harness)

**Time 9:00 AM — Task Arrives**

```
> claude
(Claude Code starts, terminal visible)

You: "I have 4 auth refactoring tasks.
     - Audit auth.ts (2,500 lines) for security
     - Refactor JWT validation for multiple key formats
     - Map impact across the codebase
     - Update docs"

Claude: (thinks about scope)
"This task has 140K tokens of codebase context.
Your Claude Max window can handle it, but it's at
the edge. My recommendation: I'll audit and refactor
the core module. But for the cross-codebase impact
analysis, you might want to manually run Gemini."

You: (manually situation awareness check)
"What's my rate limit?"
(Have to remember: I hit 82% of my 240-hour weekly
Sonnet 4 quota yesterday, that was Tuesday afternoon.
I have maybe 35-40 hours left for the entire week.
This task will use 15-20 hours alone. Good to go.)

Claude handles the security review + JWT refactoring
(takes 22 tokens hours, actual cost $0.42).

You: "Now I need to analyze impact across the full codebase."

You open a second terminal, remember the Gemini CLI
syntax from last week:

> gemini -p "Analyze these 140 files, find all JWT
  validation calls..." --include-directories src/

(Gemini runs headless, outputs JSON, you read it manually.)

You: (synthesize Claude's + Gemini's outputs)
"OK, Claude found 3 security issues. Gemini found
these 47 call sites. So the refactoring needs to
handle these 8 patterns I found by cross-referencing..."

You: "Claude, given these 47 call sites, update the
docs and give me an implementation plan."

Claude generates docs + plan ($0.18).

Time: 10:45 AM
Tokens used: Claude 22h / Gemini 8h
Cost: $0.60
State: In your head. No record of why Gemini was
       used vs Claude, what the routing was, or
       what the full cost breakdown was.
```

---

## WITH HYPOTHETICAL HARNESS

**Time 9:00 AM — Task Arrives**

```
> claude
(Claude Code starts, looks identical to user.
 Harness is invisible — baked into .mcp.json + hooks)

In the menu bar: CodexBar widget shows:
  Claude Max: 42% weekly, resets 4d 11h
  Gemini: 12% daily, resets 8h
  Session cost so far today: $0.00

You: "I have 4 auth refactoring tasks..."
     (paste the same description as before)

Claude: (reads CLAUDE.md instructions, calls harness tools)
"This is a mixed-mode task. Let me decompose:

 Task A: Security audit of auth.ts (140K tokens)
   → High priority, high precision required
   → I should handle this. My security model is
     stronger. [HANDLE_MYSELF]

 Task B: Refactor JWT validation
   → Tightly coupled to Task A output. [FOLLOW_A]

 Task C: Cross-codebase impact analysis
   → 47 files × ~3K tokens = 140K+ window
   → You can do this, but I can too
   → Wait — checking rate limits [CALLS harness tool]..."

Harness returns:
  Claude: 42% capacity, ~$0.50 available before hitting 70%
  Gemini: 12% capacity, ~$1.20 available
  Recommendation: "Task C is ideal for Gemini's 1M window"

Claude: "For Task C, I'm delegating to Gemini.
         This is a large-context analysis task
         where your 1M token window is a real advantage.
         I'll wait for your result."

[Claude calls delegate_to_gemini MCP tool]
(Behind the scenes, harness spawns gemini -p with
 scoped prompt: "Analyze call sites. Focus on:
 patterns, error handling, integration points.")

Gemini returns result (4.2 hours, $0.08 cost).

Claude: (synthesizes Tasks A + B + C result
         automatically in one flow)
"Here's the refactored module with security fixes,
and here's the impact analysis. 8 patterns need
special handling. The docs update is..."

Time: 9:48 AM
Tokens used: Claude 18h / Gemini 4.2h (30% less total!)
Cost: $0.50 (17% savings)
In the menu bar widget: "Session cost: $0.50"

You check browser dashboard (localhost:3200):

  ┌─ Live Activity ──────────────────────┐
  │ 9:00  Session start (1 agent active)  │
  │ 9:02  Task decomposed                 │
  │ 9:03  Task A → Claude (security)      │
  │ 9:04  Task C → Gemini (analysis)      │
  │ 9:15  Gemini complete                 │
  │ 9:18  Synthesis complete              │
  └──────────────────────────────────────┘

  ┌─ Routing Analytics (illustrative) ─────┐
  │ Delegation: Task C → Gemini            │
  │ Reason: 140K+ tokens, 1M context      │
  │ Confidence: 0.92 (hypothetical)       │
  │ Cost: Would be $0.32 on Claude        │
  │        Actual: $0.08 on Gemini (75%↓) │
  └───────────────────────────────────────┘
  (Confidence scores and cost estimates are
   illustrative — routing tool not yet selected
   or tested. See docs/ARCHITECTURE_VISION.md)

Time: 10:15 AM (27 minutes saved)
```

---

## THE MENTAL MODEL SHIFT

### What Changed For The User

| Aspect | Today | With Harness |
|--------|-------|--------------|
| **Awareness** | Manual rate limit tracking, scattered context | Ambient monitoring (menu bar widget always visible) |
| **Decisions** | "Should I use Gemini?" → Manual choice | "Should I use Gemini?" → Harness suggests, I override if needed |
| **Coordination** | Two separate interactions (Claude session + Gemini headless) | One seamless flow, delegation invisible |
| **State** | In my notes / in my head | Dashboard captures what happened and why |
| **Context** | "Wait, why did I pick Gemini last time?" → forgotten | Routing decision recorded with reasoning |
| **Token budgeting** | Rough estimate ("I have ~35 hours left") | Precise headroom tracking per provider |
| **Context Handoff** | Manual copy-paste between Claude and Gemini | Automatic context passing via MCP tool |

---

## Division of Responsibility: What Shifts

### What STAYS With The User

- **High-level task decomposition:** "I have 4 things to do" is YOUR insight. The harness doesn't invent tasks.
- **Veto authority:** Harness suggests Gemini. If you say "no, I want Claude," that's respected.
- **Plan approval:** When the harness generates a refactoring plan, you approve it before execution (plan mode).
- **Tool choices within a model:** Once a model is assigned, it picks its own tools.
- **File structure decisions:** "Where should this module live?" is architecture, not routing.

### What SHIFTS TO The System

| Responsibility | Today | With Harness |
|---|---|---|
| Rate limit monitoring | You manually check | Harness monitors both, notifies you |
| Routing decision logging | Nowhere | Dashboard records decision + reasoning |
| Context passing between models | Manual (copy/paste) | Automatic via harness tools |
| Session-level cost tracking | Your estimate | Harness tracks actual spend |
| Multi-turn task state | You remember | Harness keeps shared memory |
| Failure recovery (e.g., timeout mid-task) | You retry manually | Harness checkpoints and resumes |
| Gemini's plan mode implementation | Doesn't exist smoothly | Harness normalizes plan approval flow |

### NEW Responsibilities For The User

These are things that were never necessary before but emerge with the harness.

---

## NEW COMPLEXITIES: What Emerges

### 1. Trusting a Routing Decision You Don't Understand

**Today:** You manually pick Claude or Gemini. The reasoning is transparent — you thought about it.

**With harness:** The system suggests "Gemini" based on task_complexity=0.85, estimated_tokens=140K, rate_limit_headroom=0.9, historical_success=0.92. *(These scores are illustrative — the routing tool is still under investigation.)*

**New skill required:** You must trust the routing OR debug why the harness chose wrong.

```
Q: "Why did it pick Gemini?"
A: "Routing factors: large context (140K tokens),
    Gemini's 1M window advantage (0.92 confidence),
    Claude currently at 42% capacity headroom vs
    Gemini at 88%."

Q: "That's wrong for THIS task. Gemini can't handle
    my specific security review requirements."

Q: "OK, override. Use Claude instead."
    [You set routing_override: "claude" in CLAUDE.md]
```

**What the user must learn:**
- What signals the router uses (token count, context window, rate limits, historical success)
- How to recognize when the router is wrong (this task NEEDS Claude's strength, not Gemini's width)
- How to override and give feedback so the harness learns

### 2. Debugging Delegations That Fail

**Today:** Gemini fails silently. You don't know.

**With harness:** Gemini is delegate_to_gemini MCP tool. If it fails, the failure is visible AND logged.

```
Claude: "I delegated task C to Gemini. It returned:
         error: 'context_limit_exceeded'.
         Gemini ran out of tokens despite the 1M window."

New questions you must answer:
- Is this a transient failure or a real problem?
- Does this task genuinely need 1.2M tokens?
- Should I split the task or use Claude instead?
- Is there a way to compress context for Gemini?
```

**What the user must learn:**
- How to read delegation logs (Claude console + dashboard)
- When a failure is "task is too big for both models" vs "Gemini hit an edge case"
- How to file debugging issues ("task C worked fine last week, now it fails")

### 3. Debugging File Conflicts When Both Models Edit

**Today:** This doesn't happen. One model per invocation.

**With harness:** If Claude and Gemini both write to the same file in parallel (unlikely but possible with subagents), git conflicts emerge.

```
Claude (writing auth.ts): "Refactoring JWT validation..."
Gemini (writing auth.ts): "Adding new key format support..."
Result: git merge conflict marker in file.

New questions:
- Which version is correct? Both? Neither?
- Did the harness preserve the prior state properly?
- Should I use file-locking to prevent this?
```

**What the user must learn:**
- When subagents are spawned in parallel, conflicts can happen
- How to resolve them (usually: use Claude's output as the base, apply Gemini's specific contribution)
- Whether to request serialized execution instead ("Don't parallelize these tasks")

### 4. Memory Consistency Across Sessions

**Today:** Each Claude session is fresh. Gemini sessions are also fresh.

**With harness:** Shared memory persists across sessions. This is powerful AND risky.

```
Session 1 (Tuesday):
Claude stores: "JWT key format decision:
               support 3 formats, sunset 1 format
               in Q2"

Session 2 (Wednesday):
Gemini reads: "For this auth refactoring, apply the
              key format strategy from shared memory"
Gemini: "Applying the 3-format strategy..."

Q: Is that shared memory still accurate?
   What if I changed my mind?
   What if shared memory is stale?
```

**What the user must learn:**
- How to validate shared memory is still correct
- How to update shared memory when decisions change
- When to trust shared memory vs. override it

### 5. Unexpected Cascade Effects From Routing

**Today:** Doesn't happen.

**With harness:** If the router decides Gemini handles auth refactoring, and Gemini produces slightly different output than Claude would, downstream tasks see that difference.

```
Task A: Claude audits auth.ts, finds issue X
Task B: Gemini refactors JWT validation
Task C: Claude analyzes impact

If Task B output is subtly different than Claude's
style, Task C might miss edge cases.

Q: How do I know if the cascade is a problem?
   Should I add cross-model critique to Task C?
   Should I force Task B back to Claude?
```

---

## The Trade-Offs Being Made

### What You GAIN

| Gain | Magnitude | Cost |
|------|-----------|------|
| **Automatic rate limit monitoring** | High | None (pure benefit) |
| **Transparent routing decisions** (visible in dashboard) | High | Learning curve |
| **Token cost optimization** (routing to cheaper model) | Medium | Trust cost (is it the right model?) |
| **Hands-free context passing** (no manual copy-paste) | Medium | Less visibility into what context passed |
| **Session-level cost tracking** | Low-Medium | Adds ~2% token overhead for tracking |
| **Shared memory across models** | Medium-High | Stale memory risk |
| **Automatic task parallelization** (when safe) | Low | File conflicts / ordering bugs |

### What You LOSE

| Loss | Magnitude | Reason |
|-----|-----------|--------|
| **Manual control** over routing | Low-Medium | System makes decision, you override if wrong |
| **Perfect visibility** | Medium | Delegation happens "under the hood," harder to inspect |
| **Single-model debugging** | Low | Now bugs could come from harness coordination, not just the model |
| **Session isolation** | Medium-High | Shared memory + shared state means sessions aren't fully isolated |
| **Simplicity** | Medium | More moving parts (MCP server, hooks, background dashboard process) |

### The Core Trade-Off

```
TODAY: Complete transparency, full control,
       manual effort, context loss, routing mistakes

WITH HARNESS: Automatic coordination, lower token
              spend, less manual work, BUT you must
              trust the system and debug when wrong
```

**You are trading manual control for automatic optimization.** Both have costs.

---

## Honest Uncertainties

### These Are Unknown

1. **Does the routing actually save tokens in practice?**
   - Theory: Task analysis (complexity 0.85, 140K tokens) → route to Gemini ($0.08) vs Claude ($0.32)
   - Reality: Does the harness's routing analysis itself consume tokens? If the analysis costs 2K tokens, savings shrink.
   - **Status: Not yet modeled.**

2. **How reliable is the routing?**
   - Theory: FSM-based routing (heuristics, not LLM) is deterministic and predictable
   - Reality: Do heuristics capture enough signal to route correctly? Or does it route wrong 10% of the time?
   - **Status: No validation data yet.**

3. **Does delegation latency hurt?**
   - Theory: Claude delegates to Gemini, waits for result, synthesizes
   - Reality: Does the round-trip delay (spawn process, auth, run, return) add enough latency that you'd rather handle it yourself?
   - **Status: Not yet measured.**

4. **Can the harness actually normalize Gemini's gaps?**
   - The 21-item compensation list says the harness will provide plan mode, subagents, task management for Gemini.
   - Reality: These are complex features. Does the compensation actually work, or does it create new bugs?
   - **Status: Not yet implemented.**

5. **Will the dashboard be worth the maintenance burden?**
   - Theory: A separate background process showing rate limits and costs is useful
   - Reality: Is it, or is it just another process to debug when it breaks?
   - **Status: Speculation only.**

---

## How This Scenario Ends

**Time 10:15 AM — Both Versions Done**

**Without harness:** 48 minutes of work, manual context switching, scattered state, $0.60 spent, no record of why.

**With harness:** 48 minutes of work (might be 27 minutes faster if routing was optimal), seamless flow, visible state in dashboard, $0.50 spent (17% savings), full record of routing decisions.

**Cost of harness itself:**
- MCP server overhead: ~2% tokens (shared memory reads, tool metadata)
- Dashboard server overhead: negligible (runs in background)
- Upfront research/build time: months (not included here, but the real cost)

**Did it matter?**
- On a single task, maybe. Saved $0.10.
- Over a week of 80+ tasks, savings could hit $5-10.
- But only if routing was correct 95%+ of the time.
- If routing fails 10% of the time, you spend more time debugging than you save on optimization.

---

## Next Questions

If the user were to use this hypothetical harness, they would immediately ask:

1. **"Is the routing actually working? Or am I just trusting a black box?"**
   → Demands transparent logging and a way to validate routing quality

2. **"When should I override the harness?"**
   → Needs a mental model of: "this task needs this model because..."

3. **"How do I debug when it fails?"**
   → Needs concrete debugging tools and playbooks

4. **"Is this worth it, or should I just use PAL MCP + CCProxy?"**
   → The 40% gap is real only if daily experience proves it

5. **"How do I trust shared memory?"**
   → Needs validation and staleness detection

---

## Framing

This journey is **speculative**. It assumes:
- The MCP server works as designed *(Level 3 confidence — still investigating alternatives)*
- Routing logic produces correct suggestions 95%+ of the time *(Session 3 refined: investigating RouteLLM/Aurelio pre-trained routers, but neither tested on Claude+Gemini pair)*
- Shared memory is kept current *(Session 3 refined: memory now understood as architecturally central with double duty — cross-subtask + cross-session — but tool choice still open)*
- The dashboard adds useful value *(unchanged — not designed)*
- Token overhead is <3% *(unchanged — not modeled)*
- Communication follows hub-and-spoke *(Session 3 evolved: now hybrid hierarchical + mesh, exploring direct messaging)*

**None of these are validated.** They're working hypotheses. The actual user journey might be different — better in some ways, worse in others.

**The real question:** After living with this for a week, does the developer say "this is worth the months of building," or "I got 90% of the value from PAL MCP + CLAUDE.md rules"?

That's the question the next phase of investigation should answer through prototyping or zero-code baseline testing.

