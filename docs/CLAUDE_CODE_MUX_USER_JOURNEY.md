# Claude Code Mux: User Journey Scenario

> **Architecture context:** This failover scenario shows behavior WITHOUT the harness's semantic routing (Layer 1) and persistent memory (Layer 3). See [`docs/ARCHITECTURE_VISION.md`](ARCHITECTURE_VISION.md) for how the harness would improve this.

## Context
claude-code-mux is a Rust proxy providing auto-failover on 429s across 18+ providers. It's stateless by design — each request independent, no conversation history preservation between providers, no reasoning state recovery. Messages flow through intact. State doesn't.

---

## The Scenario: Mid-Session Failover

**Time: 2:45 PM. You're 25 minutes into a complex debugging task.**

You're working on a race condition in your payment module. You've spent the last 25 minutes with Claude gathering evidence:
- Read 12 files across transaction processing
- Generated 3 hypotheses about timing windows
- Debugged two false leads
- Claude kept context through all of this via conversation history

Your conversation looks like this:
```
[Turn 1] You: "Analyze the payment module for race conditions. Start with order.ts and transaction.ts"
         Claude: [Reads both files, surfaces timing gaps]

[Turn 2] You: "The gap in line 487 — what's the timing window there?"
         Claude: [Analyzes the specific window, suggests instrumentation]

[Turn 3] You: "I instrumented it. Here's the log..." [You paste 200 lines of logs]
         Claude: [Parses logs, identifies pattern, suggests fix]

[Turn 4] You: "Write the fix to order.ts"
         Claude: [Edits file, explains change]

[Turn 5] You: "Run the test suite"
         Claude: [Runs tests, shows 4 failures, analyzes failure patterns]

[Turn 6] You (current): "The failures are all in the async settlement path. Trace how..."
```

At the start of Turn 6, Claude's token window is tight. You're at 89K tokens in the 200K context window. The next request (your question about the settlement path) will consume ~3K tokens to serialize the failure logs + your question. Claude's response would need ~8K tokens to trace the issue end-to-end.

The mux checks Claude's rate limit. **You're at 94% of your daily quota.** The request arrives. Claude's API returns `429 — rate_limit_exceeded`.

---

## What Happens WITH Claude Code Mux

The proxy intercepts the 429 immediately. Decision: **route to Gemini.**

The mux follows this principle: **stateless passthrough, not intelligent recovery.**

```
Request arrives at proxy (Turn 6 prompt, ~3K tokens)
  ↓
Claude returns 429
  ↓
Mux checks: is Gemini available? (Yes, 42% daily quota)
  ↓
Mux forwards SAME request to Gemini API
  ↓
Mux returns result to Claude Code as if it came from Claude
  ↓
You see Gemini's response in your conversation
```

**What you experience:**

The response takes 3-4 seconds longer (routing overhead + Gemini latency). From your perspective in Claude Code, it looks like a slightly slow response. Gemini reads the SAME messages, analyzes the SAME logs, and returns an analysis. It works. The response is different (Gemini's phrasing, slightly different emphasis), but accurate.

**What you don't get:**

- **Seamless continuity.** Gemini doesn't "know" the 25-minute context the way Claude does (different model, different training). It reads the same transcript and reaches similar conclusions, but with less nuance about YOUR debugging journey.
- **State recovery.** Claude had formed a hypothesis about the timing window depth. Gemini re-forms it independently from the same evidence. Same destination, different path.
- **Reasoning loss.** You lose Claude's explicit reasoning about *why* the settlement path is suspect. Gemini reasons about it too, but from scratch, not extending Claude's previous analysis.

**What you gain:**

- **Uninterrupted work.** No manual provider switching. You keep typing. The response appears. You continue debugging.
- **Automatic failover.** Zero decisions from you. The proxy handles it.
- **Transparent trade-off.** You see *that* a failover happened (your logs show Gemini API called). You can see *why* (rate limit hit). It's not invisible but it's automatic.

---

## The Mental Model You Adopt

**The proxy is a stateless message router, not a context manager.**

Think of it like this:

```
Your conversation transcript = the state
Claude/Gemini API = interchangeable processors

When Claude is exhausted, send the SAME transcript to Gemini.
Gemini processes it independently.
You get a different response from the same input.
```

This changes what you *accept* from the system:

- **You own conversation management.** If you need context at Turn 40, YOU reference Turn 10 in your prompt. The mux doesn't compress history or maintain "reasoning state" — it just routes messages.
- **Reasoning isn't continuous across failover.** Turn 5 (Claude's analysis) and Turn 6 (Gemini's analysis) are separate. Gemini doesn't "build on" Claude's reasoning — it reasons independently from the shared transcript.
- **Different models = different risk.** If Claude was wrong about a hypothesis, Gemini might catch it. But it might not. You're swapping one model's blindness for another's.

---

## Key Decision Points and What You're Aware Of

### Decision Point 1: "Did Failover Actually Happen?"
**When:** Turn 6 response arrives slower than usual.
**What you check:** The proxy logs (or a small indicator in Claude Code's footer, if implemented).
**What you learn:** "Claude 429'd. Routed to Gemini. Request served in 4.2s (vs Claude's usual 2.8s)."
**What you decide:** "Good enough. The response is correct. Keep working." OR "This response is different from what Claude would say. Let me re-prompt Claude when the limit resets."

### Decision Point 2: "Is the Response Trustworthy?"
**When:** Gemini provides the settlement path trace.
**What you check:** Is the analysis sound? Does it match the logs? Does it suggest a fix you can validate?
**What you decide:** "This is good analysis, I'll try the fix" OR "This feels different. Let me validate manually or wait for Claude to recover."
**The trade-off:** You cannot assume Gemini is "equivalent" to Claude on this task. You have to think for yourself about the result's validity. The mux gives you automatic routing, not automatic trust-transfer.

### Decision Point 3: "When Do I Switch Back?"
**When:** You hit another rate limit 10 minutes later, but now your quota has reset (daily boundary, or you hit Claude's limit again).
**What you check:** The proxy dashboard — Claude available again? Quota headroom?
**What you decide:** "Switch back to Claude" — either wait for Claude to recover, or proactively re-route to Claude for the next request.
**The constraint:** The mux doesn't auto-switch back. It only auto-failover. You have to actively prefer Claude when it's available again, or the mux keeps using Gemini. (This is an open design question — does the mux have a "preference ordering" or pure "first available"?)

---

## Division of Responsibility

**The mux handles:**
- Detecting 429 status code
- Checking secondary provider availability
- Forwarding request to secondary provider
- Returning response to Claude Code
- Logging the failover event

**You handle:**
- Maintaining conversation context (you reference prior turns if Gemini needs clarity)
- Validating responses from a different model (Gemini's analysis might differ)
- Deciding when to switch providers explicitly
- Tracking which provider served which response (important for debugging chains of thought)

**You don't handle:**
- Switching APIs manually — the proxy is transparent
- Rewriting prompts for the new provider — same request goes through
- Recreating context — the transcript is your context, not the mux's responsibility

---

## The Trade-Off: What You're Making

**Simplicity + Reliability vs. Context Continuity**

**What you gain:**
- Uninterrupted workflow (no manual `export PROVIDER=gemini`)
- Automatic failover (no decision paralysis)
- Transparent degradation (you can see that a failover happened and why)

**What you lose:**
- Seamless context continuity (Gemini doesn't "remember" Claude's reasoning)
- Reasoning state preservation (each model reasons from scratch)
- Model consistency (different tone, different emphasis, different blind spots)

**When this trade-off is optimal:**

- **Short, self-contained requests.** "Analyze these logs" is fine. Gemini reads the logs fresh, returns analysis. No prior context needed.
- **Stateless tasks.** Code generation, refactoring, file edits. The code is the context, not the conversation history.
- **Periodic rate limits, not sustained exhaustion.** If you hit Claude's limit for 1-2 requests, then recover, failover is minimal pain. If you stay exhausted for hours, the accumulated friction (different model per request) compounds.
- **You trust both models.** If you know Gemini's weaker on your domain (complex distributed systems, obscure bugs), failover is riskier.

**When this trade-off is NOT optimal:**

- **Long reasoning chains.** Debugging a 25-minute investigation where each turn builds on the last. Failover mid-chain breaks continuity.
- **Model-specific capabilities.** You chose Claude for its edge in precision code review. Gemini's 4x higher control-flow error rate makes the swap painful.
- **Trust-dependent work.** Security audits, production incident response. You can't afford to swap models mid-diagnosis.

---

## Contrast: WITHOUT Claude Code Mux

Same scenario. Turn 6. Claude returns 429.

**What you do manually:**

1. See the 429 error in Claude Code
2. Stop the conversation in Claude Code
3. Open a new terminal
4. Run `gemini -p "Analyze the payment module race condition. Here's the context:" > settlement_analysis.md`
5. Paste 200 lines of context into the shell
6. Wait for Gemini to respond
7. Read settlement_analysis.md
8. Copy insights back to Claude Code context
9. Resume the Claude conversation when quota resets in 8 hours

**Time cost:** 3-5 minutes of context switching + manual copying.

**Friction:** You have to decide:
- Pause Claude and switch to Gemini (lose thread continuity)
- Wait for Claude's limit to reset (pause work for 8 hours)
- Work on something unrelated (context switch, likely harder to resume)

**Trust cost:** The manual Gemini response is in a separate file. Its analysis isn't integrated into your Claude conversation. When Claude recovers, you have to manually synthesize the Gemini insights back into the Claude context.

**Work feels fragmented:** You're managing two conversation threads — Claude's and Gemini's — not one.

---

## What Changes With Persistent Memory

*(Investigative note: This is tentative.)*

If the harness had persistent cross-provider memory (Mem0, ChromaDB, etc.), the failover would improve:

```
Turn 5 ends. Claude's analysis is complete.
Memory system writes: "Hypothesis: race condition exists in settlement handler at line 487.
                      Evidence: async timing logs show 3ms window.
                      Confidence: 0.85."

Turn 6. Claude 429's.
Mux routes to Gemini + memory injection:
         "Prior hypothesis: [above]. Please continue analysis."

Gemini reads both the transcript AND the prior hypothesis.
Gemini understands it's building on Claude's work.
Gemini's response extends, not restarts.
```

**What changes:**
- Gemini has explicit context about Claude's reasoning, not just implicit via transcript
- Response quality improves (builds on prior analysis vs. repeats it)
- Fewer tokens wasted re-explaining the same hypothesis

**What doesn't change:**
- Gemini still reasons differently than Claude
- You still need to validate the results
- The mental model shifts from "stateless router" to "stateful handler with reasoning recovery"

This is speculative because the memory system isn't implemented yet. But it's the gap between "automatic failover that works" and "automatic failover that feels seamless."

---

## Summary: The User Experience

| Aspect | WITH Mux | WITHOUT Mux | WITH Mux + Memory |
|--------|----------|-----------|-------------------|
| **Interaction** | Type normally. Response takes 4s instead of 2.8s. Keep working. | See 429. Stop. Switch tools manually. | Type normally. Response takes 4s. Keep working. |
| **Context** | Transcript is shared. Reasoning restarts. | Transcript is in Claude, separate in shell. | Transcript + prior reasoning both passed. |
| **Response Quality** | Good. Independent analysis. | Good but isolated. | Better. Reasoning continues. |
| **Friction** | Low (automatic). | High (manual). | Very low (automatic + coherent). |
| **When It's Worth It** | Rate limits every few hours. | N/A — bad experience. | Rate limits frequent + need continuity. |

---

*This scenario is a working model based on claude-code-mux's documented architecture. Confidence: medium. The actual UX depends on implementation details (how visible is the failover? can you control provider preference? how does error handling work?). These should be tested against real usage before finalizing any design based on this model.*
