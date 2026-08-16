# Evals, Harness, Loops, Graphs — The Layer Above Building

> **What this is:** everything you need to know about the current frontier of AI engineering — the layer that sits above "build a RAG system" and "build an agent." You are done building things. The next problem is **making what you built reliably ship, measure, and improve**. This is the 2026 playbook: evals, harness engineering, loop engineering, graph engineering, judges, observability, and the adjacent systems that make agents trustworthy enough for production.
>
> **The one sentence:** agents pass evals and fail production; the industry's current problem is not building agents, it's *evaluating them honestly, controlling them safely, and improving them in a loop* — and every hype word in this doc is a piece of that machine.

---

## Part 0 — Why This Layer Exists (The Current Problem)

### The Problem

RAG was solved. Agents were solved. Anyone can build both in a weekend with a framework. What *isn't* solved:

- **Reliability** — agents are probabilistic systems: the same request can take different execution paths and still fail silently.
- **Measurement** — you cannot improve what you cannot score, and scoring multi-step agent behavior is genuinely hard.
- **Trust** — production means humans and money are on the line; you need approval gates, budgets, permission boundaries.
- **Drift** — models, tools, prompts, and user behavior all change; yesterday's good agent is today's broken one.
- **The gap:** nearly half of agentic AI projects are projected to be cancelled (industry estimate, 2026) — not because the demos failed, but because evaluation and governance were treated as overhead instead of infrastructure.

**The industry answer:** "evaluation engineering" has emerged as a distinct discipline combining ML, software QA, observability, and governance. The tools and vocabulary below *are* that discipline.

### The Three-Layer Mental Model (keep this in your head the whole time)

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 3: IMPROVEMENT LOOP (hill-climbing)              │
│  traces → datasets → evals → prompt/tool/model updates  │
├─────────────────────────────────────────────────────────┤
│  LAYER 2: EVALUATION & OBSERVABILITY (measurement)      │
│  offline evals, online evals, tracing, judges, metrics   │
├─────────────────────────────────────────────────────────┤
│  LAYER 1: THE AGENT SYSTEM (what you build)             │
│  the loop (LLM ↔ tools) running inside a graph          │
│  inside a harness (permissions, budgets, validation)    │
└─────────────────────────────────────────────────────────┘
```

You already know Layer 1's innards (RAG, agents). This doc is Layers 2 and 3 plus the *outer shell* of Layer 1 (the harness). Everything new and hyped in 2026 lives in those layers.

### The Key Insight That Unlocks Everything

> **Evals are code, not benchmarks.** A benchmark is a fixed public test comparing models. An eval is a *software artifact* — a dataset, a runner, scorers, and a CI gate — that your team owns, updates, and treats with the same rigor as unit tests.

Everything else in this doc is an elaboration of that sentence.

---

## Part 1 — Evals (the core discipline)

### 1.1 What an eval is made of

Every eval has four parts:

1. **Dataset** — golden cases. Input + expected behavior (reference answer, rubric, or acceptance criteria). 20–50 high-quality cases beats 1,000 scraped ones.
2. **Runner** — executes your agent against each case (possibly in a sandboxed environment for agentic tasks).
3. **Scorer** — grades the output: exact match, code execution, human, or LLM-as-judge with a rubric.
4. **Gate** — a threshold wired into CI/CD: merge is blocked if score drops below X, or per-case failures are triaged.

### 1.2 The taxonomy you must know

| Axis | Split | Meaning |
|---|---|---|
| **Offline vs Online** | Offline = fixed dataset in CI; Online = scoring live production traffic | Offline catches regressions, is repeatable; online catches drift and unseen failures. **Use both — never choose.** |
| **Outcome vs Trajectory** | Outcome = did the final answer succeed? Trajectory = was the *path* correct? | A good final answer via a garbage path is a time bomb. Trajectory evals check tool selection, step utility, plan quality, and route correctness. |
| **Reference-based vs Reference-free** | Do you have a gold answer? | Reference-free (faithfulness, groundedness, self-consistency) is required for agentic tasks where no single right answer exists. |
| **Model-graded vs Code-graded** | LLM-as-judge vs deterministic checks | Deterministic checks (schema, tool args, state invariants) are free and reliable — use them first; use judges only where judgment is genuinely needed. |

### 1.3 The metric zoo (and what each one is actually for)

**RAG metrics (your warm-up):**
- **Faithfulness / groundedness** — is the answer supported by retrieved context? (RAGAS: faithfulness, context precision/recall)
- **Answer relevance** — does the answer address the question?
- **Retrieval precision/recall** — did we fetch the right chunks?

**Agent metrics (the real game):**
- **Task success rate** — did the session end in goal completion? (The headline number.)
- **Trajectory quality** — was the sequence of steps sensible? (Subjective → judge with rubric.)
- **Tool selection quality** — did it pick the right tool? (Node-level, easily scored per span.)
- **Tool/parameter correctness** — did it call tools with valid args? (Code-gradable — free.)
- **Action advancement** — did each step move toward the goal or spin? (Detects loops and non-contributing actions; pruning these cuts tokens and latency.)
- **Plan quality** — for planning agents: did the plan skip authentication/validation steps? (High-severity fault class.)
- **System efficiency** — latency, token usage, cost per task, tool call count. (Cheap, objective, always tracked.)
- **Step utility** — per-node contribution to the final outcome. (Enables targeted fixes instead of guesswork.)
- **Hallucination / refusal / safety** — content-level guards.

**Rule:** never optimize a single metric. Balance accuracy, reliability, speed, cost, and UX — the metric set is a *system*, and each metric must tie to an operational decision (approve deploy / rollback / alert).

### 1.4 The eval pyramid (where to spend effort)

```
        ▲  A few hard end-to-end scenarios
      ▲ ▲  Trajectory + outcome evals on agent sessions
    ▲ ▲ ▲  Per-node evals (tool correctness, routing, plan)
  ▲ ▲ ▲ ▲  Unit evals (prompts, parsers, retrieval, tools)
```

- Bottom layers are cheap, deterministic, and catch most bugs.
- Top layers are expensive but catch the failures that matter.
- Debugging works top-down; building works bottom-up.
- **Span-level evaluation** is the 2026 unlock: score each step of a trace and link the score to the span, so failures are attributable to a node, not a vibe.

### 1.5 Eval reliability — the second-order problem

The eval itself is a system and can lie:

- **Judge instability** — LLM judges change answers under pressure, repetition, and persuasion ("wiggle"; see Part 4). Same input, different verdicts across turns and levels.
- **Stale suites** — offline suites rot within weeks; production behavior shifts, golden cases stop representing reality. The #1 complaint of YC agent builders: keeping offline suites up to date is the impossible task.
- **Contamination / overfitting** — improve on your 30 cases, break production. Your dataset must be continuously refreshed from real traces.
- **Model grade inflation** — frontier models are also better at *being tested*; short take-home style tasks lose discriminative power as models improve. (Anthropic, Jan 2026: design **AI-resistant evals** — longer-horizon, tool-building, environment-understanding tasks that stay discriminating.)
- **Self-preference bias** — judges favor their own family of models. Use independent judge models or ensembles.

**The fix discipline:** every score is suspect until reproduced twice; human review on a sample; refresh datasets from production logs monthly; keep a "canary" set of cases the eval must not overfit.

### 1.6 The 2026 framework landscape (know your options)

| Tool | Open source | Orientation | Notes |
|---|---|---|---|
| **DeepEval** | ✅ Apache-2.0 | Code-first, pytest-style CI evals | 50+ metrics, the most complete OSS agent eval framework; treat eval like unit tests. |
| **Braintrust** | freemium | Eval-first scoring + AI gateways | Great dataset management + "compare agents on same data". |
| **Arize Phoenix** | ✅ Elastic-2.0 | OTel-native observability + trajectory evals | Self-hostable, strong tracing; heavier ML-team bent. |
| **RAGAS** | ✅ Apache-2.0 | RAG metrics | Now has agent goal/tool metrics too; reference-free. |
| **LangSmith** | commercial | Traces + evals + deployment, native LangChain/LangGraph | The eval loop is wired into tracing; Engine = trace-analysis agent for hill-climbing. |
| **OpenAI Evals** | ✅ MIT | Offline registry-based grading | Hosted product goes **read-only Oct 31, 2026, shuts down Nov 30, 2026** — migrate to Promptfoo; repo stays usable. |
| **Promptfoo** | ✅ | OSS eval for LLM apps, CLI/CI | The recommended migration target; strong red-teaming too. |
| **Galileo** | commercial | Agent-first enterprise QA | Distilled judge models (Luna-2) instead of frontier judges; per-step agentic evals. |
| **Langfuse / Weave (W&B)** | ✅ / freemium | Observability + evals | LLM observability platforms that grew eval layers. |
| **AWS Bedrock AgentCore Evaluations** | GA Mar 2026 | Infrastructure-level agent evals | Trajectory scoring + task completion checks wired into the runtime platform — evals as part of the control plane, not a bolt-on. |

**Selection rule:** align with your architecture, not the feature list. LangGraph stack → LangSmith. Self-hosted OTel → Phoenix. Code-first CI → DeepEval/Promptfoo. Enterprise governance → Galileo/Braintrust.

### 1.7 Benchmarks vs frameworks — don't confuse them

- **Frameworks** (above) *test your agent*.
- **Benchmarks** (below) *compare models* on fixed task sets.
- **Benchmarks that matter in 2026:** SWE-Bench (software engineering), tau-bench/τ-bench (tool-agent benchmark), Terminal-Bench (terminal agents), GAIA (general assistants), WebArena (web agents), plus task-specific suites.
- Use benchmarks for *model selection*; never as your product's eval. Your product's eval is your own dataset.

---

## Part 2 — The Harness (the shell around the model)

### 2.1 What a harness is

> **The harness is everything around the model that isn't the model** — and the part you actually ship. Runtime code, tool definitions, permissions, budgets, validation, context management, approval gates. Strong models don't mean reliable execution; the harness is where reliability is made or lost.

The industry's own framing (Anthropic, Apr 2026, *Trustworthy Agents in Practice*): govern autonomous agents through five principles —

1. **Human control** — approval gates at irreversible or risky actions; human interrupts.
2. **Value alignment** — constraints written into the harness, not hoped for in the prompt.
3. **Secure interactions** — tool permissions, least privilege, secrets handling, prompt-injection defenses.
4. **Transparency** — the agent's reasoning and actions are inspectable.
5. **Privacy** — data boundaries enforced by code.

### 2.2 What the harness concretely owns

- **Typed tools** — schemas the model must conform to; invalid calls rejected by the runtime, not by the prompt. (This is why typed tool contracts beat free-form tool use.)
- **Permissions & approval gates** — irreversible actions (send email, delete data, spend money, deploy) require human approval. Gate = suspend, present plan, confirm.
- **Budgets** — token budgets, cost ceilings, tool-call counts, time limits. Kill switches.
- **Validation layers** — output schema validation, policy checks, guardrails on top of the model.
- **Context management** — what goes into the context window, what gets compacted, what is off-limits. (The prompt is no longer the contract; the context pipeline is.)
- **Recovery** — retries, fallbacks, graceful degradation when tools fail or the model loops.
- **Sandboxing** — execute code/tools in constrained environments (containers, read-only FS, network egress control).

### 2.3 The core rule (from the harness literature)

> **When things fail, fix the harness first.** The million-line experiment (OpenAI, 2025) and the ~15k-LOC FastAPI-class systems both show: most agent failures are environmental — bad tool contracts, missing permissions, unvalidated state — not model stupidity. A production agent is 80–95% harness code; the model is a component inside it.

### 2.4 MCP as harness boundary

The Model Context Protocol (MCP) standardizes how tools/servers plug into the harness. Treat MCP as:
- A **connector layer** (tools, resources, prompts) — not a reason to skip harness concerns.
- A **trust boundary** — every MCP server is a potential prompt-injection surface; validate, allow-list, and rate-limit.
- The reason harnesses got their own engineering discipline: when tools are external and dynamic, permissioning and validation can't live in the prompt.

### 2.5 Rollout safety

Harness engineering includes *how you ship*: shadow mode (run, don't act), canary users, gradual autonomy escalation (read-only → suggest → auto-execute), rollback triggers tied to eval gates. The loop version of this is "approval gates now, autonomy later" — you can always relax gates after trust is earned; tightening them later is painful.

---

## Part 3 — Loops (loop engineering / "loopcraft")

### 3.1 The insight

> The potential in agents is in **the loops you build around them**, not the agent itself. (swyx's "loopcraft"; LangChain's "The Art of Loop Engineering", June 2026; Karpathy, Steipete, and others all converge on the same conclusion.)

A loop is a cycle where output of one pass feeds input of the next. You already know the innermost one (the agent runtime loop). The craft is stacking *additional* loops with different periods and different purposes.

### 3.2 The four loops you must know

**Loop 1 — The agent loop (runtime).** LLM ↔ tools until task done. Period: milliseconds–seconds.
- Safety: exit conditions, `max_iterations` caps, recursion limits. **If the LLM decides when to stop, you always add a backup cap.** Loops fail in four ways: LLM never wants to stop; no-exit node loops (crash certain); hidden multi-node cycles (A→B→C→A, easy to miss); tool-call repetition (same tool, tiny tweaks, hoping for a new answer). Mitigations: state counters, "tasks done" checks, hard caps at call time.

**Loop 2 — The eval loop (offline).** Change something (prompt, tool, graph edge, model) → run the suite → compare → keep or revert. Period: per PR / per change. This is CI for agents. The gate: score delta and per-case diffs.

**Loop 3 — The observability loop (online).** Production traces → sampled into datasets → curated into eval cases. Period: continuous. This is how the offline suite *stays fresh* — the cure for stale evals. Failure samples, user feedback, and interesting traces all flow here.

**Loop 4 — The hill-climbing loop (improvement).** Traces feed an *analysis agent* that detects problems and proposes config changes (prompt edits, tool fixes, grader tweaks) → changes land back inside Loop 1/2. Period: hours–days. (LangSmith Engine is the reference implementation; the return arrow reaches *inside* the agent loop and updates it.)

**Loop 5 — The learning loop (optional, advanced).** For open-weight models: eval outcomes and trace data become *training signal* for RL fine-tuning. Same loop, slower period, bigger payoff.

```
         ┌─────────────────────────────────────────────┐
         │  Loop 4 (hill-climbing): traces → analysis   │
         │      agent → prompt/tool/grader updates      │
         │  ┌───────────────────────────────────────┐   │
         │  │ Loop 2 (eval): change → suite → gate  │   │
         │  │  ┌───────────────────────────────┐    │   │
         │  │  │ Loop 3 (observability):        │    │   │
         │  │  │  traces → datasets → evals     │    │   │
         │  │  │  ┌─────────────────────────┐   │    │   │
         │  │  │  │ Loop 1 (agent runtime): │   │    │   │
         │  │  │  │  LLM ⇄ tools            │   │    │   │
         │  │  │  └─────────────────────────┘   │    │   │
         │  │  └───────────────────────────────┘    │   │
         │  └───────────────────────────────────────┘   │
         └─────────────────────────────────────────────┘
```

### 3.3 The event-driven loop (the "run in the background" loop)

The integrations layer: an event fires (webhook, schedule, document lands) → the agent runs inside the larger system → it updates a real system. The agent is a component, not an app you invoke. This is what turns a demo into automation. (LangSmith Deployment with cron/webhooks/Fleet channels; LangGraph with cron triggers and webhooks.)

### 3.4 Loop design rules

1. **Every loop needs an exit condition** — hard-coded, never model-decided alone.
2. **Give each loop a different period** — if everything runs at the same cadence you have no control hierarchy.
3. **Make the inner loop cheap, the outer loops expensive** — don't run LLM-judges on every step; sample at the outer loop.
4. **The return arrow must reach inside** — a loop that reports but can't update the thing it monitors is just telemetry with extra steps.
5. **Loops are where agents get good** — a graph that never re-evaluates its own behavior is a static artifact; the stacked loops are the improvement machinery.

---

## Part 4 — Graphs (graph engineering)

### 4.1 What graph engineering actually is

You have the LangGraph/Andrew-Ng/Karpathy material in your archive — here is the compressed truth: **an agent graph is a state machine with a checkpoint**. Nodes are functions with contracts, edges are data contracts (not arrows), and state is the part everyone underdesigns.

### 4.2 The twelve rules, compressed (from the graph engineering canon)

1. **Stop treating every "and then" as a dependency** — most steps are parallelizable; graph shape is a cost model, not a narrative.
2. **Give every node a contract** — declared input/output schemas; validate them.
3. **Treat edges as data contracts** — the edge carries state; what flows must be typed and checked.
4. **Learn the four shapes** — linear, fan-out/fan-in (parallel), router (conditional edges), loop (cycle with convergence) — almost every real graph is a composition of these.
5. **Fan out independent work, join deliberately** — parallelize where possible; merging is where bugs live (colliding keys, dropped data).
6. **Make routing inspectable** — conditional edges must log *why* they routed; invisible routing is undebuggable.
7. **Put verification on the edge** — check node output *before* it flows onward; a bad step should be caught between nodes, not at the end.
8. **State is the part most diagrams hide** — define state schema first, immutably, with a reducer per field. State bugs are the #1 source of "works in demo, breaks in prod."
9. **Add cycles only when they converge** — a loop must have a proven exit; recursion limits are mandatory (LangGraph: `recursion_limit` + state counters + task-done checks = the three safety layers).
10. **Design failure as a local event** — nodes fail and recover locally; retries, fallbacks, and error edges at each node, not a global catch.
11. **Topology is your cost model** — every node visit is a model call; cost and latency are shaped by the graph you drew. Redraw to prune.
12. **A production graph is a system** — instruments, checkpoints, and replays are part of the design, not afterthoughts.

### 4.3 LangGraph-specific knowledge (the 2026 reference stack)

- **State** — a shared object persisting across the whole execution; the agent's memory. Immutable per-step; checkpointed after every step.
- **Nodes** — functions/callables that modify state. **Edges** — flow of control. **Conditional edges** — routing on state → branches, loops, self-correction.
- **Checkpointers** — persistence per thread (Redis, Postgres, SQLite savers). Thread-level = short-term/working memory; store = cross-thread long-term memory (semantic memory with vector search).
- **Subgraphs** — a node can be a compiled graph; eval must traverse parent-child topology, not flatten it.
- **Interrupts / human-in-the-loop** — suspend execution, await approval, resume from checkpoint. The mechanism behind approval gates.
- **The unit of evaluation is the state transition**: `(state_before, node, state_after, edge_taken)` — not the final message. Message-level evals go flat while the state machine quietly degrades; checkpoint replay lets you replay any prefix for debugging and regression sets.

### 4.4 When a graph is the wrong answer

- One-shot/stateless tasks → just a call.
- Open-ended exploration → an agent loop, not a rigid graph.
- Small deterministic workflows → plain code beats a framework.
- Graphs exist to make control flow *explicit, inspectable, and safe* — if you don't need that, you're paying framework tax.

---

## Part 5 — Judges (LLM-as-a-judge done right)

### 5.1 The problem with judges

You already have the full "wiggle" study in your archive. The compressed version: **all LLM judges are unstable**, and the instability is not random — it's systematic and directional:

- **Wiggle** — judges flip verdicts under pressure: silence, repetition, adversarial persuasion, leading context. Binary scales wiggle more than Likert; higher-pressure conditions produce more flips.
- **When judges move, they usually move away from the right answer** — pressure pushes judges toward wrong verdicts, not toward better ones.
- **Wiggle rate is a model fingerprint** — some judge models are dramatically more stable than others on the same task; pick your judge empirically, per task, per scale.
- **Self-preference bias** — judges favor their own model family; use cross-family judges or ensembles.
- **Judge fragility is measurable** — baseline jury majority strength (multiple independent verdicts agreeing) is a simple reliability screen: low agreement → distrust the verdict.

### 5.2 Building a reliable judging layer

1. **Prefer code-graded checks first** — schema, tool args, state invariants, exact matches. Zero cost, zero wiggle.
2. **Rubrics over vibes** — structured scoring criteria with anchors; a rubric's *specificity* predicts judge stability more than judge size.
3. **Ensembles and majorities** — 3+ independent judge passes; disagreement is a signal to flag for human review, not to average blindly.
4. **Distilled judges** — small models trained to grade (e.g., Galileo's Luna-2) can beat frontier judges at lower cost and higher consistency for *their* rubric shape.
5. **Confidence + abstention** — judges should say "can't decide"; route low-confidence verdicts to humans.
6. **Hold out a human-annotated calibration set** — measure judge accuracy against it continuously; judges drift with model updates.
7. **Red-team the judge itself** — test your judge on deliberately adversarial cases; if your judge can be persuaded, your eval gate can be gamed.

### 5.3 Verifiers vs judges

A **verifier** checks something objective (executes code, checks a DB row, validates a schema, runs a test suite). A **judge** makes a subjective call. In agent systems, verifiers should do everything verifiable; judges only the remainder. Teams that invert this pay for it in flaky gates.

---

## Part 6 — Adjacent Topics (the extended family)

### 6.1 Observability / tracing

- **Tracing is the foundation of everything above** — no traces, no datasets, no online evals, no hill-climbing. If your agent doesn't emit traces (LLM calls, tool calls, state transitions, costs, latencies, per-span scores), you are flying blind.
- **The vocabulary:** traces (one session), spans (one step/node), attributes (metadata), events. **OpenTelemetry (OTel) is the emerging standard** — tool-agnostic, self-hostable (Phoenix, Langfuse), and increasingly the enterprise default.
- **What to trace:** inputs/outputs per node, tool calls + args + results, model + token counts + cost, routing decisions, checkpoints, judge scores. Link eval scores to spans → failures become attributable.

### 6.2 Context engineering (the discipline replacing prompt engineering)

- The prompt is no longer the contract — the **context pipeline** is: what enters the window, in what order, under what token budget, what gets compacted/evicted, what is forbidden.
- Core concepts: token budgets per section, memory hierarchies (working → episodic → semantic), retrieval patterns, compaction/summarization, progressive disclosure.
- This is where agent quality is actually won or lost in production — long-horizon agents fail from context rot (irrelevant history drowning the relevant facts) more than from model weakness.
- Adjacent to it: **memory systems** — thread-level (checkpointers), cross-thread (stores), semantic memory (vector search over past experiences), procedural memory (learned workflows). Long-term + short-term memory is now a first-class architecture concern, not a feature.

### 6.3 Eval-driven development (EDD)

- The workflow: **write the eval first, then the agent**. The eval defines "done." Every change is a hypothesis tested against the suite.
- Companions: **prompt/optimizer tooling** (agent-opt, ProTeGi, GEPA, PromptWizard, Optuna-based search) that hill-climb prompts and router prompts against the eval as the objective function.
- **Trace-driven dataset creation** is the pipeline: production traces → error-feed cluster → promote representative checkpoints into the regression set → run optimizer against it.

### 6.4 CI/CD and deployment for agents

- **Agent CI:** dataset linting → unit evals → trajectory evals → cost/latency gates → promotion. Same discipline as software CI, plus model-specific layers.
- **Deployment:** shadow mode → canary → full; feature-flagged autonomy levels; rollback = revert to previous graph/checkpoint (checkpoints make rollback trivial).
- **Monitoring:** online evals on sampled traces, alerting on score drops, latency spikes, tool error rates, policy violations. Alert thresholds must tie to operational decisions.
- **Security (the non-negotiable):** prompt injection defense at every tool boundary, least-privilege permissions, secrets isolation, audit logs of agent actions, rate limits. Governance is a feature of the harness, not a compliance afterthought.

### 6.5 The state of the world, 2026 (so you know what's real)

- **Frontier consensus:** loops around agents are the value; the agent is the commodity.
- **Evals are eating benchmarks:** per-turn, per-span, online evaluation in production is the frontier — "none of the big frameworks run per-turn blocking evals in production" is the documented gap being filled.
- **OpenAI Evals hosted product is retiring** (read-only Oct 31, 2026; shutdown Nov 30, 2026) — ecosystem consolidation toward Promptfoo/DeepEval/Braintrust-class tools.
- **AWS Bedrock AgentCore Evaluations GA (Mar 2026)** — evals becoming part of the runtime control plane, infrastructure-level.
- **AI-resistant evals** are a first-class research problem — evals must outrun the models being tested.
- **Agent harness engineering** is now a named discipline with its own best-practices corpus (typed tools, approval gates, context compaction, rollout safety, MCP connectors).

---

## Part 7 — What to Do This Week (the 30-day ramp)

1. **Instrument everything** — add tracing (OTel or LangSmith) to an existing agent. Export traces with costs, spans, routing decisions. (1–2 days)
2. **Build your first eval** — harvest 20–50 golden cases from *real* logs, not synthetic ones. Start with deterministic scorers (tool correctness, schema, task success). Add one LLM-judge trajectory eval with a written rubric. Wire it to CI as a merge gate. (3–5 days)
3. **Add the online loop** — sample production traces → curate failures into the dataset weekly. Measure eval score against a human-labeled canary set. (ongoing)
4. **Harden the harness** — typed tools, approval gates on irreversible actions, token/cost budgets, max-iteration caps on every loop, local failure handling per node. (1 week)
5. **Start the hill-climbing loop** — once traces + datasets + gates exist, an analysis pass (manual or engine-assisted) over failure clusters → targeted prompt/tool changes → verify against suite → ship. (ongoing)
6. **Then, and only then** — consider prompt optimizers, RL fine-tuning, multi-agent orchestrators. These are expensive; they only pay off on top of a working measurement system.

**Ordering principle:** measurement before optimization, harness before autonomy, offline before online, deterministic before judge-based.

---

## Part 8 — The Mentor's Honest Take

- **The hype is mostly real, but the emphasis is inverted.** Everyone talks about graphs and agents; the compounding work is evals, harness, and loops. The teams that ship in 2026 are not the ones with the cleverest agent — they're the ones with the most honest eval and the fastest improvement loop.
- **Evals are the bottleneck skill.** Anyone can demo an agent; almost nobody has a good golden dataset, a stable judge, and a CI gate. That's your differentiation.
- **You will be tempted to over-engineer the graph.** Resist. Start with the simplest graph that has explicit state, checkpointing, and verification on edges — the machinery you add later is for measurement, not for cleverness.
- **Your judge will betray you.** Every team discovers judge wiggle the hard way. Assume instability, build ensembles, prefer code checks, and keep a human-labeled calibration set from day one.
- **If you only internalize one sentence from this whole doc:** *build the loop that improves the loop* — traces → datasets → evals → changes → traces. Everything else is detail.

---

## Part 9 — Check Your Understanding

1. What are the four parts of every eval, and which one is most commonly neglected?
2. Outcome evals vs trajectory evals: an agent reaches the right answer by calling the wrong tool. Which eval catches this? Why does it matter?
3. Name the four failure modes of agent loops and the three safety layers that stop them.
4. Why is "state" the most under-designed part of a graph? What does checkpointing give you beyond memory?
5. Why is message-level (final-answer) evaluation "axis-blind" for graph systems? What is the unit of evaluation instead?
6. What are the five principles of trustworthy agents (Anthropic), and where does each one live — prompt or harness?
7. A judge flips its verdict when you ask it the same question twice with slightly different pressure. What are three things you can do about it?
8. Why must offline suites be continuously refreshed from production traces? What happens if they aren't?
9. What is the difference between a benchmark and an eval? Which one gates your merge?
10. Order these from innermost to outermost loop: observability loop, hill-climbing loop, agent runtime loop, eval loop.
11. What does "verification on the edge" mean, and why does it beat end-of-graph checks?
12. Why is the OpenAI Evals hosted product's retirement a useful signal about the ecosystem?

---

## Appendix A — Vocabulary Cheat Sheet

| Term | One-line meaning |
|---|---|
| **Eval** | Dataset + runner + scorer + gate; a software artifact, not a benchmark |
| **Harness** | Everything around the model that you actually ship: tools, permissions, budgets, validation, context |
| **Loop engineering** | Stacking feedback cycles of different periods around the agent |
| **Hill-climbing loop** | Traces → analysis → config changes that reach inside the agent |
| **Graph** | State machine with checkpoints; nodes with contracts, edges as data contracts |
| **Trajectory eval** | Scoring the path, not just the destination |
| **Outcome eval** | Scoring task completion |
| **Judge (LLM-as-a-judge)** | Model that grades with a rubric; unstable, needs ensembles + calibration |
| **Verifier** | Deterministic checker (executes code, validates schema, checks state) |
| **Wiggle** | Judge verdict instability under pressure/repetition |
| **Checkpoint** | Persisted graph state enabling replay, rollback, HITL resume |
| **Span** | One step/node in a trace; the unit of attributable failure |
| **Online eval** | Scoring live production traffic |
| **Context engineering** | Designing what enters the context window, under budget, in order |
| **EDD** | Eval-driven development: eval first, then agent |
| **MCP** | Protocol for connecting tools/servers; a trust boundary |
| **τ-bench / SWE-Bench** | Benchmarks that compare models, not your product |
| **Agent-resistant evals** | Tests designed to stay discriminative as models improve |

## Appendix B — Canonical Sources to Read Next

1. Anthropic — *Demystifying Evals for AI Agents* (trajectory vs outcome evals, reliable harnesses)
2. Anthropic — *Trustworthy Agents in Practice* (Apr 2026; the 5 governance principles)
3. Anthropic — *AI-Resistant Technical Evaluations* (Jan 2026; evals that outrun models)
4. LangChain — *The Art of Loop Engineering* (Jun 2026; the 4+ loops with LangSmith)
5. swyx — *Loopcraft: The Art of Stacking* (loops as the competitive moat)
6. LangGraph docs — state, checkpoints, recursion limits, interrupts, subgraphs
7. DeepEval / Promptfoo / Arize Phoenix docs — pick one and go deep on the eval lifecycle
8. The wiggle study (your archive) — judge epistemic stability under pressure
9. The graph engineering playbook (your archive — Andrew Ng / Karpathy) — the 12 rules
10. *Agent passes evals, fails production* — the axis-blindness pattern analysis (2026)