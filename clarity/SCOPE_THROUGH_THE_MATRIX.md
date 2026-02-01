# Scope Through the Matrix

**Session 6 — January 31, 2026**

A walkthrough of project scope derived from the 7-capability implicit harness matrix. The user explained their understanding of what the project should own vs. what Claude Code should own. Each claim was evaluated for accuracy, durability, and layering.

---

## The User's Understanding (Explained Back)


> "We would want Claude to decompose the tasks given they have some maturity and it's only going to get better."

**Correct.** And here's why: Decomposition is capability #1 on the matrix. Claude Code already does this at Basic maturity (Plan Mode, TaskCreate with dependency graphs). Swarms will make it better. It's a single-provider capability — Claude decomposes work for Claude's own subagents. There's no cross-provider angle here. If we built our own decomposition layer, we'd be competing with Anthropic on their own territory, and they will always win that race because they control the model and the tool. Let them own it.

---

> "We could also let Claude classify them."

**Correct, with a nuance.** Claude Code currently has ZERO classification (maturity: None on the matrix). But it will build this. Internal task classification — "this subtask needs Opus, that one is fine for Haiku" — is a natural evolution of their dispatching system. The classification that matters for US is different: not "which Claude tier?" but "which provider?" That's the next point, and it's right.

---

> "Where we come in is figuring out what task is best suited for which model, having this dynamic routing mechanism, but also understanding the rate limits."

**Mostly right but partially tangled.** Two things are being combined that need to be separated:

**Thing 1: Semantic routing ("this task is best for Gemini because of its 1M context window").** This IS cross-provider. Claude will never build "send this to Gemini." This is moated. Durable value — yes.

**Thing 2: Rate-aware routing ("Claude is at 80%, save capacity for later").** This is where it gets complicated. The description was good — not just "send coding tasks to Claude because Claude is best at coding," but "consider the rate budget and the remaining work before deciding." That's genuinely intelligent routing. But here's the honest answer on durability:

The rate-aware part is **partially durable.** The cross-provider aspect (routing TO Gemini) is moated — Anthropic won't build that. But the rate prediction itself (tracking usage, forecasting when you'll hit limits) is something Anthropic COULD build for their own system. They already have `/status`. They could add "you have approximately X capacity remaining this session." What they WON'T do is say "...so you should send this task to Gemini instead."

So the durable piece is specifically: **the decision to route elsewhere based on rate awareness.** The rate tracking feeds a cross-provider decision. The tracking is absorbable; the decision is moated.

The example — "even though this is a coding task, should we send it to Gemini to save Claude capacity for harder work later?" — that's a cross-provider resource allocation decision. Anthropic will never make that decision for you. That IS durable value.

**But here's the pushback:** This routing intelligence depends on rate limits being painful enough to matter. If Anthropic doubles capacity in 6 months, the "save Claude for later" calculus changes — you might never need to save anything. The routing logic is durable in principle (Anthropic won't build it) but fragile in practice (the pain point it solves might shrink). That's why it's the on-ramp, not the foundation.

---

> "When it comes to communication between subtasks between different models, we'd handle this — creating bilateral communication rather than up back to an orchestrator."

**RIGHT about the problem but the thinking about the solution is getting ahead of itself.** Here's the untangling:

The problem identified is real: If Claude dispatches a subtask to Gemini, and Gemini discovers something mid-task ("this file has a bug that affects subtask 3"), that information needs to get back to Claude AND to the other subtasks. In Claude's current hub-and-spoke model, everything goes through the parent. Parent is the bottleneck.

But here's where you need to stop and think about layers. "Bilateral communication between subtasks rather than up back to an orchestrator" — that's a mesh communication pattern. And here's the critical question: **whose subtasks?**

- **Claude's subtasks talking to each other** (Haiku subagent A talks to Sonnet subagent B): That's Claude Code's problem. Swarms is addressing this. Single-provider. Not our territory.
- **Claude's subtask talking to a Gemini subtask**: THAT is our territory. Cross-provider communication. Moated.
- **Gemini subtasks talking to each other**: That's Gemini's problem. Not our territory.

The only communication layer we own is **the seam between providers.** When a Claude subtask needs to tell a Gemini subtask something (or vice versa), that goes through us. When Claude subtasks talk to each other, that's Swarms. Don't build communication infrastructure for single-provider subtasks — you'd be building what Swarms is already building.

**Does this conflict with Swarms?** No — IF you scope it correctly. Swarms handles Claude-to-Claude communication. We handle Claude-to-Gemini communication. They're different layers. It only conflicts if you try to build the orchestration of ALL subtasks (which is what the old "harness" framing implied). If you scope to just the cross-provider seam, Swarms is an ally, not a competitor. Swarms makes Claude's side better; we make the bridge between sides better.

---

> "Recomposing — building the context engineering and intelligentness of bubbling results back up, especially if they're handled by different models."

**Correct.** And this is durable specifically because the recomposing problem across providers is fundamentally different from recomposing within a provider.

When Claude recomposes results from its own subagents, the parent model speaks the same "language" — same tokenizer, same style, same assumptions. It's relatively natural.

When we recompose Gemini's output back into Claude's context, we need to translate. Gemini structures output differently, makes different assumptions, uses different conventions. The recomposition layer needs to take Gemini's work and present it to Claude in a way Claude can understand and build on. That's a cross-provider translation problem. Claude will never build this. Durable.

**Does this conflict with Swarms?** Same answer as communication. Swarms recomposes Claude-to-Claude results. We recompose Gemini-to-Claude results. Different layers. No conflict.

---

## Meta-Analysis: "Am I thinking about things at the right layer?"

**What's being done right:** Correctly walking through each capability on the matrix and asking "is this ours or Claude's?" The instincts are good — decomposition is Claude's, classification is Claude's, cross-provider routing is ours, cross-provider communication is ours. That layering is correct.

**Where it gets tangled:** Two different time horizons are being blended in the same thought. When talking about routing, the thinking is about TODAY's pain (rate limits, capacity management). When talking about communication and recomposing, the thinking is about the ARCHITECTURE (how models talk to each other). And when bringing up Swarms, the thinking is about WHAT'S COMING. Those are three different questions being held all at once.

The reason it feels messy is that three questions are being asked simultaneously for every capability:
- "What should we build?"
- "What will Claude absorb?"
- "Does this conflict with Swarms?"

That's actually the right set of questions — but they need to be answered **in order, not in parallel.**

---

## The Four-Question Framework

These questions resolve the scope cleanly when answered sequentially. Each one depends on the answer to the previous one.

### Question 1: What's ours vs. Claude's?

Answer: Only the cross-provider parts of capabilities 3, 4, 6, and 7.

| # | Capability | Owner | Why |
|---|---|---|---|
| 1 | Decomposing | **Claude's** | Single-provider. Plan Mode + Swarms. They control the model and tool. |
| 2 | Classifying | **Claude's** | Single-provider internal routing. Natural evolution of their dispatch. |
| 3 | Dispatching | **Ours (cross-provider only)** | Claude dispatches to Claude tiers. We dispatch to Gemini. |
| 4 | Communication | **Ours (cross-provider only)** | Claude handles Claude-to-Claude (Swarms). We handle Claude-to-Gemini. |
| 5 | Conflict handling | **Claude's** | Git-based, checkpoints improving. Single-provider. |
| 6 | Recomposing | **Ours (cross-provider only)** | Claude recomposes Claude results. We translate Gemini results back to Claude. |
| 7 | State management | **Ours (cross-provider parts)** | Rate-aware cross-provider decisions are ours. Rate tracking itself is absorbable. Cross-provider memory is ours. |

### Question 2: Does anything we're building conflict with Swarms or coming features?

Answer: **No — IF we stay on the cross-provider side of the line.**

- Swarms is single-provider orchestration. We're cross-provider. Different layers.
- Swarms makes Claude's side better; we make the bridge between sides better.
- It only conflicts if we try to build orchestration of ALL subtasks (the old "harness" framing).

### Question 3: Of what's ours, what's durable vs. fragile?

| Our Capability | Durable or Fragile? | Why |
|---|---|---|
| Cross-provider semantic routing | **Durable** | Anthropic won't build "send this to Gemini" |
| Rate-aware routing decision | **Partially durable** | The decision is moated; the pain point (rate limits) might shrink |
| Cross-provider communication | **Durable** | Anthropic won't bridge Claude-Gemini subtask communication |
| Cross-provider recomposing | **Durable** | Anthropic won't translate Gemini output for Claude consumption |
| Cross-provider memory | **Durable** | Anthropic won't make context portable to Gemini |
| Rate tracking/prediction | **Fragile** | Anthropic could build this for their own system |

### Question 4: What do we build first?

**Not ready to answer until Questions 1-3 are settled.** This is the build order question. It depends on knowing what's ours (Q1), that it doesn't conflict with what's coming (Q2), and which parts are durable vs. fragile (Q3). Only then can you make an informed build order.

---

## The Key Insight

The project's scope is defined by one principle: **we own the seam between providers.** Everything on Claude's side of the seam is Claude's. Everything on Gemini's side is Gemini's. The seam itself — routing across it, communicating across it, translating across it, remembering across it — is ours.

This is why "harness" is the wrong name. A harness wraps something. We don't wrap Claude or Gemini. We connect them at the seam.



No. "Harness" is no longer accurate, and here's the specific reason why.                                                                                                                                                                                     
                                                                                                                                                                                                                                                               
  A harness wraps and controls one thing. It manages the full lifecycle of what's inside it. The original "AgentHarness" name implied we'd sit on top of Claude Code and manage all seven capabilities — decomposition, classification, dispatching,           
  communication, conflicts, recomposing, state. That's what a harness does: it's the outside layer controlling the inside layer.                                                                                                                               
                                                                                                                                                                                                                                                               
  But look at what we just established through the matrix. Of the seven capabilities, we gave back five to Claude:                                                                                                                                             
                                                                                                                                                                                                                                                               
  - Decomposing — Claude's                                                                                                                                                                                                                                     
  - Classifying — Claude's                                                                                                                                                                                                                                     
  - Conflict handling — Claude's                                                                                                                                                                                                                               
  - Single-provider dispatching — Claude's                                                                                                                                                                                                                     
  - Single-provider communication — Claude's (Swarms)                                                                                                                                                                                                          
                                                                                                                                                                                                                                                               
  We don't wrap Claude. We don't control Claude. Claude manages itself. We only touch the cross-provider seam — the four places where work crosses from one provider to another.                                                                               
                                                                                                                                                                                                                                                               
  A harness wraps one thing. We connect two things. Those are fundamentally different roles.                                                                                                                                                                   
                                                                                                                                                                                                                                                               
  What we actually do:                                                                                                                                                                                                                                         
  - Route work across the provider boundary                                                                                                                                                                                                                    
  - Translate context so each provider understands the other's output                                                                                                                                                                                          
  - Remember across providers so context isn't lost at the seam                                                                                                                                                                                                
  - Communicate between subtasks that live on different sides                                                                                                                                                                                                  
                                                                                                                                                                                                                                                               
  Every function we own is about the boundary between Claude and Gemini, not about controlling either one. The project is definitionally a connector, not a wrapper.                                                                                           
                                                                                                                                                                                                                                                               
  The most accurate terms for "the thing that manages the boundary between two independent systems" would be something like relay, bridge, or broker — not harness, not orchestrator, not proxy. Each implies connection and mediation between two independent 
  sides rather than top-down control over one side.                                                                                                                                                                                                            
                                                                                                                                                                                                                                                               
  The earlier spectrum from your previous session — Bridge, Relay, Overflow, Continuity Layer — those were all scoping decisions about how much we do at the seam. But they all share the same architectural role: we are the seam layer. The question was     
  never "do we wrap Claude?" It was always "how intelligent is the connection between Claude and Gemini?"                                                                                                                                                      
                                                                                                                                                                                                                                                               
  "Harness" was the right name when we thought we'd manage the whole lifecycle. Now that we know we only own the seam, it's the wrong shape of word entirely.   
