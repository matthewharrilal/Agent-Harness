# User Journey Scenarios: Competitive Landscape

> **Architecture context:** The harness's 5-layer architecture is detailed in [`docs/ARCHITECTURE_VISION.md`](ARCHITECTURE_VISION.md). These tool scenarios show what existing tools cover and where the harness's ~40% gap begins.

Each tool below is presented through a concrete scenario, the mental model it enforces, its division of responsibility, and the trade-offs it embodies.

---

## 1. Portkey: Enterprise Metadata-Based Routing

### Scenario: Authentication Bug Investigation

**Setup:** You're debugging an OAuth token expiration issue. The task spans two subtasks: (1) research latest XSS vectors in OAuth libraries, (2) fix your auth module.

**How Portkey Works:**

```
You: "Research XSS vectors, then fix auth.ts"
 ↓
Your App (upstream classifier)
  Classifies: "This is two tasks"
  task_1 metadata = { type: "research", model_hint: "gemini" }
  task_2 metadata = { type: "coding", model_hint: "claude" }
 ↓
Portkey Gateway
  Reads metadata
  Route task_1 → Gemini (your rule: research → Gemini)
  Route task_2 → Claude (your rule: coding → Claude)
```

### Mental Model You Adopt
**"I am the classifier. Portkey is the enforcer."** You examine each request *before it reaches Portkey*, tag it with semantic intent via metadata fields, and Portkey blindly follows your routing rules. Portkey's strength is conditional logic enforcement; its weakness is that it cannot infer intent from raw prompts.

### Division of Responsibility
| Aspect | You | Portkey |
|--------|-----|---------|
| Understand task intent | Yes (you decide) | No |
| Write routing rules | Yes (complex IF/THEN) | Yes (executes them) |
| Attach metadata | Yes (required step) | No |
| Enforce rules | No | Yes |
| Handle failures | Partially (retries) | Yes |

### The Trade-Off and Workflow It Suits
**Trade-off:** Accuracy for transparency. You lose zero request data (Portkey never misclassifies intent because it doesn't try). You gain full visibility into routing reasons. But you must build and maintain the upstream classifier.

**Best for:** Teams with clear task taxonomies (research vs coding vs debugging), structured workflows where intent is known upfront, and organizations valuing observability over automation. Also: **you cannot use Claude Max subscriptions** — Portkey requires API keys, and Anthropic banned OAuth. You'd pay for both Max subscription + API credits.

**Without the harness:** Every request manually tagged. Errors compound when you misclassify. Routing only as smart as your upstream logic.

**With the harness:** Harness contains the classifier. Portkey still enforces rules, but harness auto-generates metadata. Classification offloaded.

---

## 2. OpenRouter: 300+ Models, Auto Router Needs Training

### Scenario: Rapid Feature Prototyping

**Setup:** You're building a quick MVP feature that needs sketches reviewed. You want to try GPT-4, Claude Opus, and Gemini in parallel to see which produces the best design.

**How OpenRouter Works:**

```
You: "Design a dashboard for user settings" (using claude/openrouter)
 ↓
OpenRouter API
  Your configured model: "claude-3.5-sonnet"
  Alternative: "model: auto" (learns best model)
 ↓
OpenRouter routes to appropriate provider
  If auto: analyzes your prompt, picks model based on training
  If specific: sends to Anthropic, OpenAI, Google, etc.
 ↓
Response flows back with usage tracking
```

### Mental Model You Adopt
**"One integration, infinite model access."** You stop thinking about provider friction and start thinking about model capabilities. The mental model is horizontal (300+ models on one interface) rather than vertical (Claude is the decision-maker). You pick a model like you pick a tool: by its strengths, not by API complexity.

### Division of Responsibility
| Aspect | You | OpenRouter |
|--------|-----|-----------|
| Choose a model | Yes | No (unless using auto) |
| Provide training examples for auto | Yes (15+ required) | No |
| Route requests | Yes (via model param) | Yes (forwards to provider) |
| Handle multi-provider auth | No | Yes |
| Support Claude Max | No (API keys only) | N/A (can't route to Max) |

### The Trade-Off and Workflow It Suits
**Trade-off:** Flexibility for complexity. You gain instant access to any model on earth. You lose the focused expertise of a single provider's CLI. And critically: **you cannot use Claude Max subscriptions** — OpenRouter requires API keys. Auto Router requires training data (15+ labeled examples minimum).

**Best for:** Developers building AI applications (not coding agents in Claude Code), teams evaluating models, prototyping where you want to try multiple models quickly. NOT for orchestrating Claude Max + Gemini, because Max subscriptions are incompatible with OpenRouter's auth model.

**Without the harness:** You manually pick models, manage API keys across providers, track training data for auto-router.

**With the harness:** Harness provides semantic routing without needing OpenRouter. Harness works WITH Claude Max (not through OpenRouter). Faster iteration.

---

## 3. hcom: SQLite Message Bus, Collision Detection (Not Prevention)

### Scenario: Parallel Code Review Loop

**Setup:** Claude is writing middleware, and you want Gemini to review it in real-time. Both agents are active simultaneously.

**How hcom Works:**

```
10:30:15 Claude writes: src/middleware/rateLimit.ts (125 lines)
        PostToolUse hook fires
        SQLite INSERT: {
          agent: "claude",
          event: "file_write",
          file: "src/middleware/rateLimit.ts"
        }

10:30:48 Gemini's polling hook detects new event
        Sees: "Claude wrote rateLimit.ts"
        Based on config: "Auto-review files Claude writes"
        Spawns: gemini -p "Analyze rateLimit.ts for bugs"

10:31:30 Gemini completes review, finds: race condition in line 42
        SQLite INSERT: {
          agent: "gemini",
          event: "review_comment",
          file: "src/middleware/rateLimit.ts",
          findings: [{issue: "race_condition", line: 42, severity: "high"}]
        }

10:31:33 Claude's hook polls, sees Gemini's review
        Injects into Claude's context
        Claude reads feedback and fixes line 42
```

### Mental Model You Adopt
**"Agents are aware of each other through event notifications."** You're not orchestrating — you're enabling discovery. Each agent has independent autonomy but reads an async event log. The mental model is **messaging/gossip** rather than command/control. Agents react based on what they see, not what they're told to do.

### Division of Responsibility
| Aspect | You | hcom |
|--------|-----|------|
| Decide who does what | No (agents decide) | No |
| Capture file changes | No (hooks auto-fire) | Yes |
| Store event log | No | Yes (SQLite) |
| Detect collisions | No | Yes (after writes) |
| Prevent collisions | No | No |
| Inject context into agents | Partially (via hook scripts) | No |

### The Trade-Off and Workflow It Suits
**Trade-off:** Autonomy for coordination. Agents are fully independent — they decide what to do based on observed events. But they cannot prevent conflicts (collision detection is after-the-fact, not before). Both agents can edit the same file simultaneously, discover the conflict, then reconcile.

**Best for:** Workflows where agents are naturally independent workers (one writes, one reviews, one tests) and where you want lightweight real-time awareness without heavy orchestration. NOT for preventing conflicts or enforcing strict serialization. **Key limitation:** messaging, not routing. Does not decide who handles tasks — only notifies what happened.

**Without the harness:** You manually coordinate agents or accept that conflicts happen. Context handoffs are manual.

**With the harness:** Harness adds orchestration on top of messaging. Harness prevents collisions via file locking. Harness synthesizes events into actionable context, not just raw notifications.

---

## 4. Conductor Build: 5 Parallel Claude Instances, 5x Rate Burn

### Scenario: Building 5 Features in Parallel

**Setup:** You have 5 independent features to build (auth module, API routes, tests, docs, frontend). Instead of building sequentially, you want 5 Claude Code instances working simultaneously.

**How Conductor Works:**

```
You open Conductor Build app
Click: "Create 5 agents"
 ↓
Conductor creates:
  - git worktree #1 (branch: feature/auth)
  - git worktree #2 (branch: feature/api)
  - git worktree #3 (branch: feature/tests)
  - git worktree #4 (branch: feature/docs)
  - git worktree #5 (branch: feature/frontend)
 ↓
Conductor spawns Claude Code CLI in each worktree
 ↓
Dashboard shows all 5 agents working simultaneously
```

### Mental Model You Adopt
**"Parallelism via isolation. Each agent owns a worktree."** You're not routing tasks to agents (agents are pre-created). You're dividing your codebase into independent branches, spinning up N agents, and collecting results. The mental model is **workspace multiplexing**, not intelligent routing.

### Division of Responsibility
| Aspect | You | Conductor |
|--------|-----|-----------|
| Decide which agent works on what | Yes (assign features) | No |
| Create isolated environments | No | Yes (git worktrees) |
| Spawn Claude Code instances | No | Yes |
| Manage rate limits across agents | No (you watch burn 5x faster) | No |
| Route subtasks within an agent | No | No (each agent is independent) |
| Merge results | Partially (you review diffs) | Yes (helps with PR) |

### The Trade-Off and Workflow It Suits
**Trade-off:** Throughput for rate limit overhead. You get 5x parallelism but burn Claude quota 5x faster. When Claude Max runs out (normally lasts a week), now lasts 1-2 days. No automatic failover, no cross-provider routing, no rate limit awareness.

**Best for:** Teams with independent, parallel tasks and deep Claude quota budgets. Features that don't depend on each other. MacOS only. **Critically:** Claude-only (no Gemini, no GPT-4). **Rate limit behavior:** Silent failures — agents stall when hitting limits with no clear notification.

**Without the harness:** You manually manage 5 agents, burn quota quickly, deal with 429 errors mid-task, no cross-provider fallback.

**With the harness:** Harness would add: rate limit awareness, automatic throttling, Gemini fallback when Claude is exhausted, cross-agent communication. Conductor + Harness = intelligent parallelism.

---

## 5. CLIProxyAPI: Menu Bar Quota Visibility, Display-Only Routing

### Scenario: Watching Your Quota Burn While Debugging

**Setup:** You're debugging a tricky race condition. The menu bar shows Claude at 72%, and you want to know when to switch to Gemini. You have multiple API accounts and want automatic failover.

**How CLIProxyAPI Works:**

```
Menu bar widget (top-right, always visible):
┌──────────────────────────────┐
│ Claude:  ████████░░  72%     │
│ Gemini:  ██░░░░░░░░  15%     │
│ Cost today: $3.87            │
└──────────────────────────────┘

CLIProxyAPI proxy (localhost:8080):
Request 1 → Account A (round-robin)
Request 2 → Account B
Request 3 → Account C
Request 4 → Account A (cycle repeats)

If Account A returns 429 (rate limited):
  CLIProxyAPI immediately retries with Account B
  Response flows back
  You may not even notice the switch
```

### Mental Model You Adopt
**"Real-time data visibility, no automatic routing."** You glance at the menu bar, see Claude is at 85%, and *you decide* whether to switch tasks. CLIProxyAPI shows you the information but doesn't *act* on it. It's a dashboard that occasionally does reactive failover.

### Division of Responsibility
| Aspect | You | CLIProxyAPI |
|--------|-----|-------------|
| Decide when to switch providers | Yes | No |
| Display quota percentages | No | Yes |
| Predict exhaustion | No | No |
| Route tasks proactively | No | No |
| Route tasks reactively (after 429) | No | Yes |
| Round-robin across accounts | No | Yes (if configured) |

### The Trade-Off and Workflow It Suits
**Trade-off:** Visibility for agency. You see everything happening. CLIProxyAPI acts on nothing. You get a perfect information display but no automatic decision-making. Failover only happens AFTER a request fails (reactive), never before (proactive).

**Best for:** Developers who want to monitor their quota burn and occasionally switch providers manually. Teams with multiple API accounts who want load distribution. **Key constraint:** Round-robin works best for identical-provider-multiple-accounts (3x Anthropic API keys), not cross-provider routing. **Status update:** Display-only means you must manually decide when to route elsewhere — no thresholds, no predictive logic.

**Without the harness:** You watch the menu bar, manually route tasks to Gemini when Claude is getting full. You hit 429 errors sometimes and experience interruptions.

**With the harness:** Harness observes the same quota data but acts on thresholds automatically. "At 80%, route next task to Gemini." Seamless, no manual intervention. Proactive, not reactive.

---

## Comparative Summary

### What Each Tool Does Well
| Tool | Strength | Constraint |
|------|----------|-----------|
| **Portkey** | Complex metadata-based rules, enterprise observability | Cannot use Claude Max, requires upstream classifier |
| **OpenRouter** | Instant access to 300+ models, one integration | Cannot use Claude Max, auto-router needs training data |
| **hcom** | Real-time agent awareness, lightweight messaging | Messaging only (no routing), collision detection not prevention |
| **Conductor** | True parallelism, git worktree isolation, visual dashboard | Claude-only, 5x rate burn, no cross-provider fallback |
| **CLIProxyAPI** | Quota visibility at a glance, reactive failover | Display-only (no proactive routing), manual decision-making |

### The ~40% Gap Each Leaves

| Gap | Portkey | OpenRouter | hcom | Conductor | CLIProxyAPI |
|-----|---------|-----------|------|-----------|-------------|
| **Semantic routing** (infer intent from task) | ✗ (metadata only) | ✗ (if no training data) | ✗ (no routing) | ✗ (manual assign) | ✗ (display only) |
| **Proactive rate limits** (act before 429) | ✗ (no awareness) | ✗ (no awareness) | ✗ (no awareness) | ✗ (burns 5x) | ✗ (reactive only) |
| **Persistent memory** (resume after switch) | ✗ (stateless) | ✗ (stateless) | ✗ (events only) | ✗ (isolated agents) | ✗ (quota only) |
| **Cross-model critique** (Claude → Gemini → Claude) | ✗ (no loop) | ✗ (no loop) | ~ (if you script) | ✗ (Claude-only) | ✗ (no routing) |
| **Claude Max support** | ✗ (API keys) | ✗ (API keys) | ✓ | ✓ | ✓ |

---

## Terminology Note

Each tool claims to solve "orchestration," but they actually solve different problems:
- **Portkey** = Policy enforcement (routing engine)
- **OpenRouter** = Provider abstraction (unified API)
- **hcom** = Event messaging (real-time awareness)
- **Conductor** = Parallelism multiplier (workspace isolation)
- **CLIProxyAPI** = Quota dashboard (visibility layer)

None is an "orchestrator" in the true sense (deciding what work to do, assigning it to agents, managing state, handling failures). They each cover one dimension of the orchestration problem.

### What the Harness Uniquely Provides

The Agent Harness sits at the intersection these tools leave empty. It provides **all five dimensions simultaneously**: semantic routing (understanding task intent), proactive rate limit management (acting before 429), persistent cross-provider memory (context survives across sessions and providers), cross-model critique (veto-based quality improvement), and invisible integration inside the existing Claude Code experience. No existing tool combines these. The Session 3 5-layer vision (Semantic Router → Decision Engine → Memory → Visual → CLI) is the structural framework for this integration — still under investigation, with no layer having a concrete implementation decided.

---

*User journeys compiled from TOOL_ANALYSIS.md. Each tool analyzed for constraints, dependencies, and workflow fit. Scenarios grounded in documented tool mechanics and real usage patterns.*
