# Harness vs Orchestration Pipeline

> Word-for-word reference from Session 5 (Feb 1, 2026). Visual presentation: [HARNESS_VS_PIPELINE.html](./HARNESS_VS_PIPELINE.html)

## Context: Why This Analysis Exists

The user asked: "If that taxonomy correlates to orchestration pipeline, what exactly is a harness? What is the harness in relation to that taxonomy? When somebody mentions an agent harness, I want to make sure that I have the same definition they're talking about compared to this orchestration pipeline and the implications and correlation between the two. What one does without the other and the value that one adds to another."

The scope was confirmed as:
1. The harness taxonomy laid out clearly — as a concrete set of capabilities you can point to
2. A side-by-side showing how the harness and orchestration pipeline relate — what each provides, where one depends on the other
3. What it looks like when you have a harness WITHOUT an orchestration pipeline vs WITH one — the concrete value-add
4. Where they intermingle — "I know it's not a simple agent taxonomy harness... I know it's an advanced intermingling and weaving of concepts throughout."

---

## Part 1: The Harness Taxonomy (Industry Standard)

A harness is **infrastructure**. It's the stuff that must exist for an agent to function at all. Phil Schmid's analogy is the cleanest: the model is the CPU, the context window is RAM, the harness is the operating system, the agent is the application. You don't run applications without an OS. You don't run agents without a harness.

The harness does not decide *what work to do*. It makes it possible to do work at all.

**The 8 capabilities that appear across multiple authoritative sources:**

| # | Harness Capability | What It Does (Plain English) | Who Says This |
|---|---|---|---|
| H1 | **Context Engineering** | Manages what the model can see right now — compresses old context, injects relevant info, keeps the window useful instead of cluttered | Anthropic, Salesforce, Parallel.ai, Phil Schmid |
| H2 | **Tool Integration** | Gives the agent hands — file operations, code execution, web search, API calls, MCP. Without this, the agent can only talk. | Anthropic, Google DeepMind, LangChain, Parallel.ai |
| H3 | **State & Session Persistence** | Keeps track of where things stand across time — progress files, checkpoints, recovery points. Work survives restarts. | Anthropic, Salesforce, Parallel.ai |
| H4 | **Memory** | Decides what to remember and forget across sessions — selective retention that informs future decisions. Not just storage, but retrieval. | Anthropic, Salesforce, Parallel.ai, Phil Schmid |
| H5 | **Guardrails & Permissions** | Prevents the agent from doing things it shouldn't — permission boundaries, input/output validation, safety checks, human-in-the-loop gates. | Anthropic, Salesforce, Google DeepMind, Parallel.ai |
| H6 | **Error Handling & Recovery** | Catches failures and retries intelligently — tool call errors, timeout recovery, defensive coding that keeps things running. | Google DeepMind, Salesforce, LangChain, Anthropic |
| H7 | **Planning & Decomposition Support** | Provides the scaffolding for the agent to break work into steps — task trackers, dependency graphs, scratchpads. Not the planning itself, but the infrastructure for planning. | LangChain, Parallel.ai, Anthropic |
| H8 | **Environment Reproducibility** | Ensures the agent starts in a known, correct state every time — initialization scripts, workspace setup, configuration. | Anthropic, LangChain, Phil Schmid |

---

## Part 2: Our Orchestration Pipeline (For Reference)

| # | Pipeline Step | What It Does |
|---|---|---|
| P1 | Decompose tasks | Break "build authentication" into subtasks: user model, routes, middleware, tests |
| P2 | Classify subtasks | Label each: "middleware is complex," "test scaffolding is boilerplate" |
| P3 | Dispatch to models | Send middleware to Claude, scaffolding to Gemini |
| P4 | Communicate between agents | When Gemini's user model is done, Claude's middleware needs to know |
| P5 | Handle conflicts | Claude and Gemini both modified app.js. Detect, resolve, or escalate. |
| P6 | Recompose results | Take output from both models, merge into one coherent result |
| P7 | Rate/memory/state | Track consumption, persist findings, maintain continuity |

---

## Part 3: How They Connect — The Pipeline Runs ON the Harness

Every orchestration step depends on harness infrastructure. Without the infrastructure, the step cannot execute.

| Pipeline Step | Depends On Which Harness Capabilities | Why It Can't Work Without Them |
|---|---|---|
| P1: Decompose | H1 Context Engineering + H7 Planning Support | You need the full request in context (H1) and a task tracker to put subtasks into (H7) |
| P2: Classify | H1 Context Engineering + H4 Memory | Classification needs the task description (H1) and historical routing outcomes to learn from (H4) |
| P3: Dispatch | H2 Tool Integration + H3 State Persistence | Dispatching means invoking another model via a tool (H2) and tracking which subtask went where (H3) |
| P4: Communicate | H3 State Persistence + H4 Memory | Subtasks share state through files, queues, or memory layers (H3, H4) |
| P5: Handle conflicts | H2 Tool Integration + H5 Guardrails + H6 Error Handling | Detecting conflicts needs file access (H2), resolution needs safety gates (H5), retries need recovery logic (H6) |
| P6: Recompose | H1 Context Engineering + H2 Tool Integration | Merging results requires fitting them into the context window (H1) and reading output files (H2) |
| P7: Rate/memory/state | H3 State + H4 Memory + H6 Error Handling | This step is partially harness, partially pipeline — it's the overlap zone |

---

## Part 4: The With/Without — What Breaks

**Scenario: Refactoring a payment processing module from monolith to microservice**

### Harness ONLY (Claude Code today, no orchestration pipeline)

You open Claude Code, describe the task. The harness kicks in:
- **H1 works:** Context engineering compresses your conversation, keeps relevant files visible
- **H2 works:** Claude reads files, edits code, runs tests via tools
- **H7 works:** Plan mode breaks the refactor into subtasks with dependencies
- **H5 works:** Permission gates prevent destructive operations

Then you hit the wall:
- **6:30 PM Monday:** Rate limit. Work stops. No P3 (dispatch) exists — nobody routes anything to Gemini. You manually open Gemini CLI, but Gemini has zero context from Claude's 4 hours of work. You spend 20 minutes re-explaining. Gemini's output uses different conventions. You manually paste it back into Claude.
- **Tuesday morning:** New session. H4 (memory) is manual — CLAUDE.md has whatever you wrote last. Context from Monday is gone except what's in git. You spend 20 minutes re-explaining the architecture decisions Claude made yesterday.
- **Tuesday afternoon:** Rate limit again. Same manual Gemini workaround. Compounding loop: hit limit, switch manually, lose context, re-explain, waste tokens, hit limit faster.

**The harness works fine as infrastructure. The problem is there's no workflow for cross-provider work.**

### Orchestration Pipeline ONLY (hypothetical — no harness)

The semantic router correctly classifies the task (complexity: 0.88, type: code_refactoring). The decision engine correctly decides to route the codebase analysis to Gemini's 1M context window.

Then nothing happens:
- P3 (Dispatch) fires, but there's no H2 (Tool Integration) — the pipeline can't actually invoke Gemini. It has no subprocess management, no MCP tools, no CLI access.
- P4 (Communicate) fires, but there's no H3 (State Persistence) — nowhere to write shared state files. No filesystem access.
- P7 (Memory) fires, but there's no H4 — no storage layer, no retrieval system.

**A brain with no hands.** The pipeline knows exactly what to do but can't do any of it. Every decision requires infrastructure that doesn't exist.

### BOTH Together (harness + orchestration pipeline)

Same task. Monday morning:
- You describe the task to Claude. **H7** (planning support) provides TaskCreate. Claude decomposes it (P1). The **pipeline** classifies subtasks (P2) — codebase analysis is 180K tokens of file reading, perfect for Gemini's context window.
- **P3** dispatches the analysis to Gemini using **H2** (MCP tool integration spawns Gemini CLI). Result: 3K tokens consumed instead of 40K. Claude's rate budget is preserved.
- Monday afternoon: Claude builds the service (precision coding work). Simultaneously, Gemini migrates 23 call sites (repetitive work). **P4** handles communication — Gemini discovers a missing API endpoint mid-migration, writes to shared memory (**H3** provides the file I/O). Claude's hook picks it up (**H1** injects it into context). Claude adds the endpoint.
- The security-critical webhook handler triggers **P5** — automatic cross-model critique. Gemini catches a race condition Claude missed. **H5** (guardrails) gates the merge until the fix is verified.
- **P6** recomposes: Gemini's migration output is translated into Claude's format and merged into the codebase.
- Tuesday morning: New session. **P7** feeds **H4** — 800 tokens of memory injected automatically. Zero re-explanation. All decisions from Monday (including ones Gemini made and ones Gemini's critique caught) are available.

**The harness provides the infrastructure. The pipeline makes the decisions. Neither works without the other.**

---

## Part 5: Where They Intermingle — The 5 Overlap Points

The harness and pipeline aren't cleanly separated layers you can draw a line between. They intermingle at specific points:

| Where They Touch | What the Harness Provides | What the Pipeline Decides |
|---|---|---|
| **Rate limits** | H3: Store usage counters. H6: Handle 429 errors, retry with backoff. | P7: Should we route to a different model based on remaining capacity? |
| **Memory** | H4: Store findings, persist state, provide retrieval. | P4/P7: What should we share between subtasks? What's worth remembering for tomorrow? |
| **State** | H3: Checkpoint progress, enable crash recovery. | P7: When to checkpoint, what constitutes a recovery point. |
| **Planning** | H7: TaskCreate, dependency graphs, scratchpads. | P1/P2: How to decompose, how to classify each piece. |
| **Tool access** | H2: MCP, Bash, file ops — the raw ability to invoke things. | P3: Which tool, which model, for which subtask. |

The infrastructure **enables**. The pipeline **decides**.

---

## Part 6: What This Means for the Project

When someone says "agent harness," they mean H1-H8 — the infrastructure. Claude Code already IS a strong harness for single-provider work:

| Harness Capability | Claude Code Coverage |
|---|---|
| H1 Context Engineering | **High** — compaction, fork, /context |
| H2 Tool Integration | **High** — Read, Edit, Bash, Grep, MCP, 170+ tools |
| H3 State Persistence | **Moderate** — git, checkpoints, sessions (but sessions die) |
| H4 Memory | **Low** — CLAUDE.md is manual. No semantic memory. |
| H5 Guardrails | **High** — permission modes, approval gates |
| H6 Error Handling | **Moderate** — retries exist, no cross-provider recovery |
| H7 Planning Support | **Moderate-High** — Plan mode, TaskCreate, dependency graphs |
| H8 Environment Setup | **High** — CLAUDE.md, .mcp.json, hooks |

Our project does two things:

1. **Extends the harness** with cross-provider infrastructure — H2 (tool access to Gemini), H3 (cross-provider state), H4 (cross-provider memory). This is harness work.
2. **Builds the orchestration pipeline** — the P1-P7 workflow that uses the extended harness to route, communicate, and recompose across providers. This is pipeline work.

The "AgentHarness" repo name was half right — we ARE extending the harness. But we're also building something on top of it (the pipeline) that the industry doesn't have a name for yet.
