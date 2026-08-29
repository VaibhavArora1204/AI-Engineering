# Production Agent Engineering Practice 2026

## Harness Engineering

## Agent = Model + Harness: The 6-Layer Production Playbook

Based on Mitchell Hashimoto's engineering methodology, OpenAI's Codex field report, Martin Fowler's guides-and-sensors taxonomy, Anthropic, LangChain, and Cursor engineering materials

*Independently compiled, August 2026 — not affiliated with Google, OpenAI, Anthropic, or HashiCorp — and not endorsed*

```
+------------------+     +------------------+     +------------------+
|     GUIDES       |     |   AGENTIC LOOP   |     |    SENSORS       |
|  AGENTS.md       |     |                  |     |  Linters         |
|  Rule files      +---->+  Plan > Execute  +---->+  Tests           |
|  Constraints     |     |  Verify > Fix    |     |  Validators      |
|  Examples        |     |  Retry / Escalate|     |  LLM-as-judge    |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         v                        v                        v
+--------+---------+     +--------+---------+     +--------+---------+
|    MEMORY        |     |   PERMISSIONS    |     | OBSERVABILITY    |
|  State files     |     |  Tool budgets    |     |  Trace logs      |
|  Artifacts       |     |  Write limits    |     |  Cost tracking   |
|  Decision log    |     |  Approval gates  |     |  Trip wires      |
+------------------+     +------------------+     +------------------+
```

*Fig. 1. The six-layer agent harness architecture. Guides and sensors form the steering loop (feedforward + feedback). Memory, permissions, and observability form the runtime foundation. The agentic loop sits at the center, bounded by all five supporting layers.*

---

**Abstract** — 95% of enterprise AI agents never reach production. They demo well, pass budget review, and then quietly die in pre-production. The gap has a name: harness engineering. This note presents the complete six-layer architecture that separates shipping agents from abandoned prototypes. The formula: Agent = Model + Harness. The model provides reasoning. The harness provides everything else: guides that prevent known failures, sensors that catch new ones, an agentic loop with bounded retries, persistent memory, enforced permissions, and full observability. We compile evidence from five sources showing harness changes alone produce larger gains than model upgrades: a 44-point benchmark swing on the same model, a 25-rank improvement on Terminal Bench with zero model change, and a million-line codebase with zero manually written code.

**Index Terms** — Harness engineering, agent harness, AI agents, guides and sensors, agentic loop, AGENTS.md, CLAUDE.md, verification, observability, ratchet principle, capability budgets.

---

## I. THE PROBLEM: DEMOS THAT NEVER SHIP

*Why 95% of AI agents fail before production*

A chatbot produces responses. An agent produces outcomes. The difference is not the model. The difference is the infrastructure that turns a probabilistic language model into a reliable, bounded, observable system. Every AI coding tool generates code now. Yet more code does not mean better code. Research from Faros found that AI adoption is producing changes that are larger, more complex, and carry a wider blast radius than before [8]. The convincing surface quality of AI-generated output makes errors cognitively expensive to detect.

The agents fail security review, miss observability requirements, hallucinate in edge cases, and lack the governance every enterprise needs. The gap between what an agent can do in a demo and what it must do in production has become the defining engineering problem of 2026.

### A. The Three Eras

The industry spent 2023-2024 optimizing prompts: phrasing, examples, chain-of-thought. It spent 2025 designing context systems: RAG, MCP, memory, retrieval. By February 2026, practitioners converged on a third discipline that subsumes both: harness engineering.

**TABLE I — THREE ERAS OF AI ENGINEERING**

| Era | Focus | Optimizes | Limitation |
|---|---|---|---|
| Prompt Eng. (2023-24) | Single turn | Phrasing, examples | One interaction |
| Context Eng. (2025) | System context | RAG, MCP, memory | What model sees |
| Harness Eng. (2026) | Full environment | Execution, safety | Entire runtime |

Each era subsumes the previous one. A harness contains context pipelines and prompts, but also verification, permissions, observability, and state persistence. Prompt engineering shapes what the model says. Context engineering shapes what the model sees. Harness engineering shapes what the model can do, what survives failure, what is allowed to happen, and what constitutes successful completion [3].

### B. Who This Playbook Is For

This note is for engineers, operators, founders, and small teams who already use AI agents and want them to work reliably without constant supervision. No custom agent framework is required. The architecture is useful for coding agents, research agents, operations agents, and any workflow where an LLM must act autonomously.

---

## II. THE FORMULA: AGENT = MODEL + HARNESS

*How one blog post defined 2026*

On February 5, 2026, Mitchell Hashimoto, co-founder of HashiCorp and creator of Terraform, Vagrant, and Ghostty, published a blog post titled "My AI Adoption Journey" [1]. In it he described a discipline he developed while working with AI coding agents:

> "Anytime you find an agent makes a mistake, you take the time to engineer a solution such that the agent never makes that mistake again."
> — Mitchell Hashimoto [1]

Days later, OpenAI engineer Ryan Lopopolo published a field report [2]. His team spent five months building a production product with zero manually written lines of code. The codebase reached one million lines across roughly 1,500 automated pull requests. The humans were not writing code. They were designing the environment that made reliable code generation possible. The tagline: "Humans steer. Agents build."

The formula crystallized: Agent = Model + Harness. Martin Fowler published a systematic analysis [3], Ethan Mollick reorganized his framework around "Models, Apps, and Harnesses" [4], and the term entered the core AI engineering vocabulary.

### A. The Evidence That Harness Beats Model

**TABLE II — HARNESS-ONLY PERFORMANCE GAINS**

| Source | Model | Benchmark | Before | After | Delta |
|---|---|---|---|---|---|
| Masood [6] | Claude Sonnet 4.5 | GAIA | 30.91% | 74.55% | +43.64 pts |
| LangChain [7] | Same model | Terminal Bench | 30th | 5th | +25 ranks |
| Hashline [9] | 16 LLMs | Coding bench | Baseline | Improved | Harness only |
| OpenAI [2] | Codex agents | Production | 0 lines | 1M lines | Zero manual |

Hold the model fixed and swap only the harness. The same Claude Sonnet 4.5 swings from 30.91% to 74.55% on GAIA: a 43.64-point spread attributable entirely to the harness [6]. This is larger than most model upgrades deliver.

### B. Inner Harness vs. Outer Harness

Frontier AI labs build the inner harness: foundational safety layers, native tool-calling, and context windows embedded in the base models. The engineering moat for a product team is the outer harness: custom configuration, environmental routing, testing frameworks, and situational guidelines built by your team around the model. This playbook focuses on the outer harness [5].

---

## III. LAYER 1: GUIDES

*Feedforward controls that prevent known failures before execution*

Guides are the instructions an agent reads before starting work. They are feedforward controls: they shape behavior before execution. Martin Fowler and Birgitta Böckeler introduced the guides-and-sensors taxonomy that has become the canonical vocabulary for harness components [3].

### A. What Guides Contain

The primary guide files are AGENTS.md, CLAUDE.md, and .cursorrules. Each line represents a past failure converted into a permanent prevention mechanism. Hashimoto's AGENTS.md for Ghostty has accumulated rules one line at a time, each addressing a specific agent error [1].

**Template 1 — Minimum Guide File**

```
PROJECT: [name]
LANGUAGE: [primary language]
BUILD: [exact build command]
TEST: [exact test command]
LINT: [exact lint command]

RULES:
- Never modify /config without asking
- Run tests after every code change
- Use [pattern] for [case]

ANTI-PATTERNS:
- [specific past failure, dated]
- [another observed failure mode]
```

Guides should contain executable actions and observable criteria, not motivational language. Replace "do thorough research" with named sources, stopping rules, and citation requirements. Replace "write good code" with a linter command, test command, and specific patterns.

### B. The Ratchet Principle

Hashimoto's methodology creates a ratchet: every failure permanently improves the system. The six-step loop: (1) Agent makes a mistake. (2) Identify the failure class, not the symptom. (3) Determine the strongest fix layer: guide, sensor, tool, or permission. (4) Encode the fix. (5) Verify it prevents recurrence. (6) Monitor for regression. A prompt patch fixes one conversation. A guide rule fixes every future run. A sensor catches the error class automatically. An environment constraint makes the error structurally impossible [1].

### C. Guide Hygiene

Guides accumulate. Without maintenance, they become contradictory, redundant, or stale. Version guide files. Review them monthly. Remove rules that sensors now enforce automatically. Group rules by category. Date each entry so you can trace when it was added and why. A guide file with 200 undated lines is not a harness. It is technical debt.

**Guide Quality Questions**

- Can the agent verify each rule without subjective judgment?
- Does every rule trace to an observed failure?
- Are any two rules contradictory?
- Could any rule be replaced by a linter or test (sensor)?
- When was each rule last validated against current behavior?

### D. Guides as Organizational Memory

A guide file is not just for agents. It is the encoded knowledge of how your system works, what has failed before, and what must never happen again. New team members read AGENTS.md and immediately understand the constraints. When an engineer leaves, their corrections survive in the guide file. The ratchet preserves institutional knowledge that would otherwise live only in someone's head.

OpenAI's internal practice treats the guide file as the system of record [2]. The file is not a suggestion. It is the primary source of truth for how the agent should behave in this project. When the guide and the conversation conflict, the guide wins. This inversion, giving a file more authority than a conversation, is what makes the harness durable.

### E. The Cost of Not Having Guides

Without a guide file, every new session starts from zero. The agent must infer the build system, coding patterns, forbidden operations, and project structure from the codebase alone. It will make the same mistakes that were corrected last week. It will use patterns that were banned last month. It will modify files that should be read-only. Every correction made in conversation is lost when the session ends. The guide file is what makes corrections permanent.

---

## IV. LAYER 2: SENSORS

*Feedback controls that catch failures after execution*

Sensors verify output after execution. They are feedback controls: they detect problems that guides could not prevent. The combination of feedforward guides and feedback sensors creates the steering loop that makes continuous improvement possible [3].

**TABLE III — SENSOR TYPES AND RELIABILITY**

| Type | Example | Speed | Cost | Determinism |
|---|---|---|---|---|
| Computational | Linter, type checker | Fast | Free | Deterministic |
| Computational | Unit test suite | Fast | Free | Deterministic |
| Computational | Schema validator | Fast | Free | Deterministic |
| Inferential | LLM-as-judge | Slow | Expensive | Non-deterministic |
| Inferential | AI code review | Slow | Expensive | Non-deterministic |

Computational sensors run on every change. Inferential sensors add semantic judgment but cost more and return non-deterministic verdicts. Design principle: computational sensors first. Add inferential sensors only for checks that cannot be expressed as deterministic rules [3].

### A. Self-Verification Pattern

The strongest harness pattern gives the agent access to its own sensors. After each step, the agent runs a pre-defined test suite and loops failures back with the error text. This is not the model judging its own quality. It is the model executing external, deterministic checks and acting on the results.

```python
def verified_step(agent, task, sensors):
    result = agent.execute(task)
    for sensor in sensors:
        verdict = sensor.check(result)
        if not verdict.passed:
            result = agent.fix(result, verdict)
            if not sensor.check(result).passed:
                return escalate(task, verdict)
    return result
```

### B. Sensor Economics

A single linter rule costs nothing to run and prevents a class of review comments forever. A test validating API response structure costs milliseconds and prevents broken integrations. An LLM-as-judge evaluating "code quality" costs tokens and returns non-deterministic verdicts. Invest in computational sensors first. They compound faster and cost less.

### C. When to Add an Inferential Sensor

Add an LLM-as-judge only when the check requires semantic understanding that no deterministic rule can capture: tone of a customer email, correctness of a legal summary, quality of a design decision. When you do, treat it as an expensive advisory signal, not a gate. Log its verdicts, measure agreement with human review, and replace it with a deterministic check whenever a pattern becomes clear enough to encode as a rule.

### D. The Sensor Coverage Test

Before deploying any agent workflow unattended, verify: Can the agent check its own output without human inspection? Does every critical output have at least one computational sensor? Are sensor results logged for trend analysis? Is the sensor stable while the agent's behavior changes? Could a silent regression reach users without tripping any sensor? If any answer is no, add the missing sensor before enabling unattended operation.

### E. Building Sensor Suites Incrementally

Start with the sensors that already exist: the project's test suite, linter, and type checker. Wire them into the agent's execution loop so the agent runs them after every change. Only after these computational sensors are reliably catching errors should you consider adding inferential sensors. The cost curve matters: a ten-dollar-per-run LLM judge on a task that runs fifty times per day is five hundred dollars per day. A linter that catches the same class of error is free.

---

## V. LAYER 3: THE AGENTIC LOOP

*Plan, execute, verify, fix — with bounded retries and escalation*

The agentic loop is the execution engine at the center of the harness. It is not a single model call. It is a bounded cycle: plan the action, execute it with tools, verify the result against sensors, fix failures, and either advance or escalate. Every step is observable. Every retry is counted.

```python
def agentic_loop(task, agent, sensors, budget):
    plan = agent.plan(task)
    for step in plan.steps:
        for attempt in range(budget.max_retries):
            result = agent.execute(step)
            verdict = verify(result, sensors)
            if verdict.passed: break
            if not verdict.retryable:
                return escalate(step, verdict)
        else:
            return escalate(step, "budget exhausted")
    return synthesize(plan.results)
```

**TABLE IV — REQUIRED LOOP BOUNDS**

| Bound | Purpose | Default |
|---|---|---|
| Max retries per step | Prevent infinite loops | 3 attempts |
| Max wall-clock time | Prevent runaway execution | 30 minutes |
| Max token budget | Control model cost | 100K tokens |
| Max financial cost | Prevent bill shock | $5 per task |
| Max tool calls | Prevent tool abuse | 50 calls |
| Stopping condition | Return best artifact | Always defined |

When any budget is exhausted, the loop returns the best current artifact, completed work, unresolved issues, and a reason for stopping. It must not hide partial failure behind a fluent final answer.

### A. Escalation Is Not Failure

An agent that escalates correctly is more valuable than one that produces a confident wrong answer. The escalation packet should contain: the decision required from the human, the recommended option, alternatives already tested, cost of waiting, and the safest default action if no response arrives.

### B. The Background Agent Rule

Hashimoto: "I endeavor to always have an agent doing something at all times. If I'm coding, I want an agent planning. If agents are coding, I want to be reviewing" [1]. The loop should not require human attention during normal execution. The human provides intent, reviews results, and handles escalations. Everything between those moments is the harness.

---

## VI. LAYER 4: MEMORY AND STATE

*The model forgets every session. The harness remembers*

Every model call starts with an empty context window. The harness reconstructs relevant state. This is the fundamental asymmetry: the model has no durable memory. The harness provides continuity through explicit state management.

**TABLE V — STATE PERSISTENCE LAYERS**

| Layer | Persists | Survives | Implementation |
|---|---|---|---|
| Context window | Current turn | Nothing | Conversation buffer |
| Scratchpad | Current task | Tool calls | plan.md, todo.md |
| Artifact store | Cross-session | Restarts | Git, filesystem, S3 |
| Decision log | Cross-session | Restarts | decisions.jsonl |
| Knowledge graph | Permanent | Everything | Entity-relation store |

The simplest durable memory is the filesystem. A plan.md file, a decisions.md log, a progress.json checkpoint. The agent reads these at session start and updates them after every meaningful step. This is cheaper and more reliable than vector databases for most agent workflows.

### A. The Filesystem as Memory

The simplest durable memory is the filesystem. A plan.md file, a decisions.md log, a progress.json checkpoint. The agent reads these at session start and updates them after every meaningful step. This is cheaper, simpler, and more reliable than vector databases for most agent workflows. The file is the memory. The artifact is the state.

```python
def checkpoint(task, state, artifacts):
    with open(f"state/{task.id}.json", "w") as f:
        json.dump({
            "task_id": task.id,
            "status": state.status,
            "completed_steps": state.completed,
            "artifacts": [a.path for a in artifacts],
            "last_updated": datetime.now().isoformat()
        }, f)
```

### B. Compaction for Long-Running Agents

When an agent runs for hours, the context window fills up. OpenAI's engineering guide describes server-side compaction as a critical harness primitive: summarize older context, preserve recent actions and decisions, and keep the working set within the context budget [2]. The compaction itself is a model call, but the decision of when and what to compact belongs to the harness, not the agent. Routing accuracy improved from 73% to 85% by adding negative examples to the compacted context in OpenAI's Skill system [14].

### C. Memory vs. Guides

Memory records what happened. Guides define what should happen. Do not rely on an agent remembering one correction from an old conversation when the rule matters to every future run. Promote stable preferences, policies, and procedures into explicit guide files. Leave temporary facts, task-specific discussion, and exploratory reasoning in memory or the task scratchpad.

### D. The Recovery Test

Close the agent session in the middle of a multi-step task. Reopen it. The agent should read its checkpoint, identify the last completed step, and resume from the next step without repeating completed work or asking the human to re-explain the task. If it cannot do this, the memory layer is insufficient. Add a checkpoint file.

---

## VII. LAYER 5: PERMISSIONS AND BUDGETS

*The model does not enforce safety. The harness does*

The model cannot restrict itself. It will use any tool it has access to, write to any file it can reach, and send any message it can compose. Permissions are not a model property. They are a harness property. The harness is the primary security boundary [3].

**Template 2 — Capability Budget**

```
ALLOW read:    src/**, tests/**, docs/**
ALLOW write:   src/**, tests/**
ALLOW execute: npm test, npm run lint
ASK before:    git push, npm publish, deploy
DENY:          rm -rf, DROP TABLE, send_email
RATE:          max 20 writes per task
COST:          max $5 per task
TIMEOUT:       30 minutes per task
```

**TABLE VI — DEFAULT ACTION POLICY**

| Action | Default | Reason |
|---|---|---|
| Read source files | Allow | Reversible observation |
| Write project files | Allow + log | Recoverable via git |
| Run tests / linters | Allow | No side effects |
| Push to remote | Ask | External visibility |
| Deploy to production | Human only | Hard to reverse |
| Send external message | Ask | Reputation impact |
| Delete data or files | Human only | Potentially irreversible |

### A. Prompt Injection Changes the Risk Model

An autonomous agent reads untrusted content: emails, web pages, documents, repository issues, user-submitted text. A malicious instruction embedded in content must not expand the agent's permissions, change its system policy, redirect secrets, or trigger external actions. The harness must separate trusted instructions (guide files, system prompts) from untrusted data (user input, retrieved documents) in every task [2], [6].

### B. Four Budget Dimensions

Four dimensions govern every agent action. Scope: which accounts, tools, files, and operations are available. Rate: the maximum writes, commits, or external calls per interval. Reversibility: whether the action can be rolled back safely; irreversible actions require human approval. Visibility: who is notified and which evidence is retained for audit. Encode all four before the first unattended run.

```python
def policy_gate(action, target, task):
    decision = policy.evaluate(action, target)
    if decision == DENY:
        log_denial(action, target, task)
        return BLOCKED
    if decision == ASK:
        return request_human_approval(action, target)
    if rate_limit.exceeded(action):
        return trip_wire(action, task)
    execute(action, target)
    append_audit_record(action, target, result)
    return COMPLETED
```

### C. The Principle of Least Privilege

Give the agent the smallest environment that can finish its job. If the agent needs to read source files and run tests, grant exactly those permissions. Do not grant write access to configuration files, network access to external services, or permission to install packages unless the task specifically requires it. Every unnecessary permission is an attack surface for prompt injection and a blast radius for agent errors.

---

## VIII. LAYER 6: OBSERVABILITY

*Track everything. Alert on drift. Catch regressions before users do*

A production harness requires telemetry. Without it, a faster agent can appear successful while silently omitting work that mattered. Required telemetry: start/finish timestamps, tool actions and outcomes, guide version and sensors used, attempt count and cost, produced artifacts, approvals and escalations.

**TABLE VII — TRIP WIRE TRIGGERS**

| Trip Wire Trigger | Indicates | Response |
|---|---|---|
| External write spike | Permission drift | Freeze writes |
| Same error repeats 3x | Guide or sensor gap | Escalate |
| Cost exceeds 2x average | Runaway loop | Pause and inspect |
| Sensor pass rate drops | Quality regression | Roll back |
| New domain contacted | Scope creep | Block and alert |
| Duration exceeds 3x | Agent stuck | Timeout |

**TABLE VIII — HARNESS HEALTH SCORECARD**

| Metric | Definition | Direction |
|---|---|---|
| Completion rate | Verified runs / started runs | Up |
| Rework rate | Runs manually corrected | Down |
| Escalation rate | Human pings per task | Down |
| Recovery time | Minutes from failure to safe state | Down |
| Cost per task | Tokens + tools per verified result | Down |
| Guide growth | New rules added per week | Declining |

### A. The Real Metric

Do not count model calls, tokens, or messages. Count completed tasks that required no manual intervention and still produced acceptable evidence. A system is improving when completion rises while rework, unnecessary approvals, cost per completed task, and recovery time fall. This metric prevents a visually impressive agent from hiding a large human coordination tax.

### B. Observability as Debugging Infrastructure

When an agent produces a wrong result, the first question is: where did it go wrong? Without structured logs, the answer requires replaying the entire conversation. With observability, you can trace the exact tool call that returned unexpected data, the exact sensor that passed when it should have failed, or the exact step where the agent ignored a guide rule. Observability is not overhead. It is debugging infrastructure.

### C. Cost Attribution

Track cost per task, not per day. A daily spend of $50 is meaningless without knowing whether it completed 100 tasks or 2. Cost per verified result is the metric that reveals whether the harness is improving. When a new guide rule reduces retries from 3 to 1, cost per task drops by 60%. When a new sensor catches errors that previously required human review, cost per verified result drops further. The harness pays for itself when the cost reduction exceeds the cost of maintaining it.

**Operational Review Questions**

- Can you identify the most expensive task type this week?
- Can you identify which sensor caught the most errors?
- Can you identify which guide rule was violated most often?
- Can you trace any completed task from start to finish?
- Would you know within an hour if the agent started failing?

---

## IX. THE IMPROVEMENT LOOP

*Convert every failure into permanent infrastructure*

**TABLE IX — FAILURE CLASSIFICATION**

| Failure Class | Weak Fix (Avoid) | Strong Fix (Prefer) |
|---|---|---|
| Known bad pattern | Prompt reminder | Linter rule (sensor) |
| Missing context | Longer conversation | Guide file entry |
| Wrong tool use | In-chat correction | Permission boundary |
| Quality drift | Human review every run | Automated test suite |
| State loss | Re-explain everything | File-based checkpoint |
| Unsafe action | Hope it doesn't recur | Capability budget + deny |
| Cost overrun | Manual monitoring | Token budget + trip wire |

> "Prompts guide behavior. Environments prevent entire classes of failure."
> — Lauren Tan, Cursor [10]

The ratchet creates a virtuous cycle: each failure that gets fixed at a strong layer reduces the total number of failures in subsequent runs. Early in the harness lifecycle, guide growth is rapid because many common errors are being discovered. As the most frequent failure classes are covered, growth declines. A mature harness adds perhaps one new rule per week instead of five per day. This declining growth rate is the clearest signal that the harness is working.

### A. The Six-Step Engineering Loop

When an agent fails, follow this loop: (1) Reproduce the failure with the exact same input. (2) Classify the root cause: missing guide, absent sensor, permission gap, state loss, or observability blind spot. (3) Identify the strongest layer where the fix belongs. (4) Implement the fix at that layer. (5) Verify the fix on the original failing case. (6) Run the regression suite to ensure no existing capability broke. Skip no step. Skipping step 1 means you might fix a phantom. Skipping step 6 means you might break three things while fixing one.

### B. Converting Review Comments into Constraints

Lauren Tan at Cursor describes a powerful pattern: when a human reviewer writes the same review comment more than three times, that comment should become a structural constraint [10]. The progression: the first time, add a guide rule. The second time, verify the guide rule is being read. The third time, convert it to a sensor that blocks the agent from producing output that violates the rule. The fourth time should never happen.

**TABLE X — CONTROL RELIABILITY LADDER**

| Layer | Example | Reliability | Cost to Add |
|---|---|---|---|
| Memory | Prior correction in chat | Low | Zero |
| Prompt | Task instruction | Low-Medium | Minutes |
| Guide | AGENTS.md rule | Medium | Minutes |
| Sensor | Automated test | High | Hours |
| Environment | Permission, schema, CI | Highest | Hours-Days |

---

## X. BUILD PATH: SEVEN DAYS TO PRODUCTION

*Increase layers only after each one proves reliable*

**TABLE XI — SEVEN-DAY BUILD PATH**

| Day | Build | Exit Test |
|---|---|---|
| 1 | AGENTS.md with build, test, lint | Agent runs all 3 correctly |
| 2 | Add 3 guide rules from failures | Agent avoids all 3 anti-patterns |
| 3 | Add first computational sensor | Agent runs tests after every change |
| 4 | Wire agentic loop + retry budget | Agent retries, then escalates |
| 5 | Add file-based state checkpoint | Agent resumes after restart |
| 6 | Set permissions + cost budget | Agent cannot exceed scope |
| 7 | Structured logging + trip wire | Trip wire fires on simulated spike |

### A. Days 1-2: Guides First

Create one guide file with minimum viable content: project name, language, exact build/test/lint commands, three rules from known failures. Run the agent three times. Every failure becomes a new guide rule. Do not add sensors yet.

### B. Days 3-4: Sensors and Loop

Add the cheapest computational sensor: the existing test suite. Wire the agentic loop so the agent runs tests after every change, retries once on failure, escalates after two failed attempts. Verify the agent never claims success without passing tests.

### C. Days 5-6: Memory and Permissions

Add a JSON checkpoint the agent writes after every step and reads at session start. Set explicit permission boundaries. Run one task, close the session, reopen, verify the agent resumes without repeating work.

### D. Day 7: Observability

Add structured logging for every tool call and sensor result. Create one trip wire: alert when cost exceeds 2x the six-day average. Simulate a cost spike and verify the alert fires. The harness is now complete at minimum viable level.

### E. Week 2 and Beyond: The Expansion Pattern

After seven days, the harness has all six layers at minimum viable level. Expansion follows the ratchet: run the agent on real tasks. Every failure that occurs reveals which layer needs strengthening. If the agent makes the same error twice, the guide is missing a rule. If the agent produces subtly wrong output, a sensor is missing. If the agent exceeds scope, a permission is too broad. If you cannot diagnose a failure, observability is insufficient. Change one layer at a time. Measure the effect. Roll back if any scorecard metric regresses.

**Scale Gate — Expand Only When All Pass**

- Completion rate >= 80% without manual correction
- No task has exceeded cost budget
- Agent has correctly escalated at least once
- State checkpoint has survived at least one restart
- Trip wire has fired on at least one simulated anomaly
- Permission boundary has blocked at least one action

### F. When to Harden a Rule

Move a guide rule to a sensor when: the same review comment appears more than three times, the rule can be checked without subjective interpretation, violations create material rework or production risk, and the check can explain how to repair the violation. This is Lauren Tan's principle from Cursor: repeated human corrections are the signal to convert taste into a gate [10].

---

## XI. DECISION FRAMEWORK

**TABLE XII — ARCHITECTURE DECISION FRAMEWORK**

| Problem | Start With | Do Not Add Yet |
|---|---|---|
| Known errors repeat | Guide rules | LLM-as-judge |
| Output quality varies | Computational sensors | Multi-agent review |
| Agent exceeds scope | Permission boundary | Full approval workflow |
| State lost between sessions | JSON checkpoint | Knowledge graph |
| Cost unpredictable | Token + cost budget | Dynamic pricing |
| Failures invisible | Structured logging | Full observability stack |
| Agent loops forever | Retry limit + timeout | Complex orchestration |

The correct starting point is always the simplest layer that addresses the observed failure. Run the agent, observe failures, and let the harness grow from real evidence. Do not build a complex harness before running the agent.

---

## XII. PRODUCTION CHECKLIST

**TABLE XIII — PRODUCTION READINESS CHECKLIST**

| # | Requirement | Failure If Missing |
|---|---|---|
| 1 | Guide file with build/test/lint | Agent guesses environment |
| 2 | 5+ guide rules from failures | Known errors repeat |
| 3 | Computational sensor per task | Errors reach users |
| 4 | Bounded retry + escalation | Infinite loops, runaway cost |
| 5 | State checkpoint to filesystem | Restart loses progress |
| 6 | Permission boundary | Agent exceeds scope |
| 7 | Token and cost budget | Unbounded spend |
| 8 | Structured logging | Failures invisible |
| 9 | Trip wire on cost/error rate | Regressions unnoticed |
| 10 | Trusted/untrusted input split | Prompt injection risk |
| 11 | Emergency stop + state preserve | Cannot safely halt |
| 12 | 3 successful unattended runs | Untested system deployed |

---

## XIV. HARNESS VS. ALTERNATIVES

*Why harness engineering displaced prompt and context engineering*

The progression from prompt engineering to context engineering to harness engineering is not a matter of fashion. Each discipline addresses a limitation the previous one cannot solve.

**TABLE XIV-B — THREE DISCIPLINES COMPARED**

| Challenge | Prompt Eng. | Context Eng. | Harness Eng. |
|---|---|---|---|
| Factual errors | Correct in prompt | RAG retrieval | Verification sensor |
| Forgets prior work | Summary in prompt | Memory system | File checkpoint |
| Wrong tool use | Instruct in prompt | Limit tool list | Permission policy |
| Quality varies | Add examples | Few-shot examples | Automated tests |
| Runs indefinitely | 'Be concise' | Limit context | Retry + cost budget |
| Errors repeat | Re-add correction | Store in memory | Guide rule (perm.) |
| Prod. incident | Add warning | Guardrail retrieval | Approval + trip wire |

The pattern is clear: prompt engineering addresses symptoms within a single turn. Context engineering addresses information gaps. Harness engineering addresses structural failures that neither prompts nor context can prevent. A prompt cannot enforce a permission. A retrieval pipeline cannot bound retries. A memory system cannot fire a trip wire. These are harness responsibilities.

### A. The Compounding Effect

The most important property of harness engineering is compounding. Every guide rule added today prevents a class of errors in every future run. Every sensor added today catches a class of defects in every future output. Every permission boundary added today blocks a class of incidents in every future session. Prompt corrections do not compound. They must be re-applied each conversation. Context configurations compound slowly because retrieval quality degrades as the corpus grows. Harness improvements compound rapidly because they are structural.

### B. When All Three Are Needed

A production agent needs all three disciplines working together. The prompt shapes the agent's reasoning style and task understanding. The context pipeline ensures the agent has the right information. The harness ensures the agent's actions are safe, verified, bounded, observable, and durable. Removing any layer weakens the system. But if forced to choose where to invest the next engineering hour, the harness almost always produces the largest marginal improvement. This is the lesson of the benchmark data: the model and prompt are the same. The harness is the variable.

---

## XV. LIMITATIONS AND COMMON MISTAKES

*What can go wrong even with a harness*

### A. The Over-Engineered Harness

A harness with 500 guide rules, 12 inferential sensors, 3 layers of LLM-as-judge review, and a 40-step approval workflow is not a production system. It is a bottleneck disguised as safety. The harness should be the minimum infrastructure that makes the agent reliable. Every additional layer adds latency, cost, and maintenance burden. If the agent spends more time checking its own work than doing it, the harness is too heavy.

### B. Guide Files That Never Get Pruned

Guide files accumulate one line at a time. Without pruning, they become contradictory, redundant, and eventually ignored. A common failure: the agent follows Rule 47 ("always add error handling") but violates Rule 183 ("keep functions under 20 lines") because the error handling adds 10 lines. The rules were written months apart by different people and never reconciled. Schedule monthly reviews. Remove rules that sensors now enforce. Consolidate rules that address the same failure class.

### C. Sensors That Test the Wrong Thing

A test suite that achieves 100% pass rate on every agent run is suspicious. Either the tests are too easy, or the agent has learned to produce output that passes tests without actually solving the problem. Adversarial tests matter: inputs that look correct but contain a hidden violation, outputs that pass schema validation but contain wrong data, code that builds and passes linting but has a logic error. The sensor suite should reject bad work, not just confirm good formatting.

### D. Memory Without Cleanup

State checkpoints accumulate disk space and stale data. A checkpoint from three weeks ago references files that no longer exist, decisions that were reversed, and artifacts that were superseded. The agent reads the stale checkpoint and makes decisions based on outdated state. Add expiration to checkpoints. Clean up completed task state. Keep only the most recent checkpoint per active task.

### E. The Harness Does Not Fix Bad Objectives

A perfectly engineered harness around a poorly defined objective produces reliable garbage. If the acceptance criteria are wrong, the sensors will validate wrong output. If the guide rules encode incorrect patterns, the agent will reliably produce incorrect results. The harness amplifies the objective and evaluation chosen by the builder. If the system optimizes the wrong thing, the harness increases the scale of the error.

**TABLE XV — COMMON HARNESS MISTAKES**

| Mistake | Symptom | Fix |
|---|---|---|
| Too many rules | Slow, contradictory | Prune monthly |
| No inferential sensors | Subtle errors pass | Add LLM-as-judge |
| No computational sensors | Expensive LLM-judge | Add linter, tests first |
| Stale checkpoints | Outdated state | Expire old checkpoints |
| No trip wires | Regressions unnoticed | Cost + error alerts |
| Wrong criteria | Reliable wrong output | Review criteria |
| All-or-nothing perms | Blocked or unrestricted | Graduated scope |

---

## XVI. WHEN NOT TO BUILD A HARNESS

*Not every agent interaction justifies engineering overhead*

A harness earns its cost when the agent runs the same workflow repeatedly, failures have real consequences, the agent operates without human supervision, or multiple sessions must preserve state. Single-turn queries, creative brainstorming, exploratory conversations, and one-off tasks do not benefit from guides, sensors, or checkpoints.

**TABLE XVI — HARNESS DECISION FILTER**

| Scenario | Harness? | Why |
|---|---|---|
| One-off question | No | No recurrence, no consequence |
| Creative brainstorming | No | Output is exploratory, not verifiable |
| Daily code review agent | Yes | Repeats daily, errors have consequences |
| Research digest agent | Yes | Runs weekly, must cite sources |
| Customer support triage | Yes | External-facing, must be reliable |
| Personal note-taking | No | Low stakes, no external effect |
| CI/CD pipeline agent | Yes | Runs on every commit, failures block team |
| Exploratory data analysis | Maybe | If reused, yes. If one-off, no |

The test: would you notice if the agent silently produced a wrong result? If yes, you need a sensor. Would you need to explain the same context again next session? If yes, you need memory. Would a mistake create external consequences? If yes, you need permissions. If none of these apply, the conversation itself is the harness.

---

## XVII. HARNESS ENGINEERING FOR MULTI-AGENT SYSTEMS

*When one agent is not enough*

The six-layer harness applies to individual agents. When multiple agents collaborate, the harness extends to cover the boundaries between them. Each agent needs its own guides, sensors, and permissions. The system additionally needs: typed handoffs between agents, a shared state model, a routing policy, and an independent verifier that no agent can override.

### A. Typed Handoffs

When Agent A passes work to Agent B, the handoff should be a typed interface: what artifact is ready, where it lives, what has been verified, and what remains unresolved. A free-form summary like "done, looks good" must not advance a production workflow. The receiver should be able to open the artifact and independently verify it. The SpaceXAI Playbook documents this pattern extensively: every handoff includes task identity, artifact pointer, evidence, assumptions, and deadline [11].

### B. Shared Memory vs. Shared Context

Multiple agents sharing a conversation create context pollution: each agent's reasoning, errors, and corrections fill the context window of every other agent. A better pattern: agents share a structured state model (a ledger, a graph, a database) and read only the state relevant to their current task. The Anthropic Knowledge Graph Cookbook demonstrates this pattern: workers read and write to a shared graph while the orchestrator's context stays clean [12].

### C. The Verifier Must Be Independent

The agent that produced an artifact should not be the sole judge of its quality. Producer bias is real: the generating agent has context and incentives that skew its judgment. Use deterministic sensors where possible and a separate verifier agent for material work. The verifier reports failures without silently rewriting output. The router then sends failure evidence back to the producer or escalates after the retry budget is exhausted.

**TABLE XVII — MULTI-AGENT HARNESS EXTENSIONS**

| Multi-Agent Layer | Single Agent Equivalent | What It Adds |
|---|---|---|
| Typed handoffs | Self-verification | Contract between agents |
| Shared state model | File checkpoint | Cross-agent memory |
| Routing policy | Agentic loop | Task assignment rules |
| Independent verifier | Computational sensor | Producer-consumer separation |
| Escalation protocol | Human approval | System-level escalation |

---

## XVIII. CONCLUSION

The model provides the intelligence. The harness determines whether that intelligence becomes a product or remains a demo. The evidence is empirical: harness-only changes produce performance gains as large as model upgrades. A 44-point benchmark swing. A 25-rank improvement. A million lines of production code with zero manually written. The model was the same. The harness was the only variable.

The formula Agent = Model + Harness is an engineering specification. The six layers are the minimum viable infrastructure for any agent that must work without constant supervision. Guides prevent known failures. Sensors catch new ones. The agentic loop bounds execution. Memory persists state. Permissions enforce safety. Observability enables debugging and improvement.

The practical recommendation is Hashimoto's ratchet: every time the agent makes a mistake, engineer a solution so that mistake never happens again. Do not patch the prompt. Fix the harness. Start with one guide file. Add one sensor. Set one permission boundary. Log one metric. Wire one trip wire. The harness accumulates. The agent improves. The result is not a smarter model. It is a system that can be trusted, inspected, and improved.

The three eras of AI engineering, prompt engineering, context engineering, and harness engineering, are not alternatives. They are layers. A production agent needs all three. But the harness is the layer that determines whether the system ships. A perfect prompt inside a broken harness produces unreliable results. A mediocre prompt inside a strong harness produces consistent, improvable results. The harness wins.

The path forward is incremental: build one measured loop, make failures reversible, add one tool for one error class, split roles only when specialization adds signal, store artifacts before conversations, and scale only when the reducer is defined. Each pattern addresses a specific limitation of the previous stage. The progression is not mandatory, but it is directional.

A reliable harness-engineered system should make this statement true: every important output can be traced to a guide rule that shaped it, a sensor that verified it, a budget that bounded it, a checkpoint that preserved it, and a log that recorded it. When that statement is false, adding more model calls usually increases opacity. When it is true, the harness becomes a composable engineering mechanism rather than opaque behavior.

> "The engineers who thrive in 2026 are not the ones who write the most code. They are the ones who build the best environments for agents to write code in."

---

## XV. REAL-WORLD EVIDENCE

*Five case studies that prove the formula*

### A. OpenAI Codex: One Million Lines, Zero Manual Code

In February 2026, OpenAI engineer Ryan Lopopolo reported that a small team spent five months building a production product using only Codex agents [2]. The codebase reached one million lines across roughly 1,500 automated pull requests. No code was manually written. The humans designed the environment: guide files, test harnesses, review pipelines, and deployment gates. The team's tagline captured the new division of labor: "Humans steer. Agents build." The critical detail: they reported investing more time in the harness than they would have spent writing the code manually. The harness was the product.

### B. LangChain: 30th to 5th Without Changing the Model

In March 2026, the LangChain engineering team improved their coding agent's ranking on Terminal Bench 2.0 from 30th place to 5th place [7]. The model was unchanged. The improvement came entirely from harness optimizations: better guide rules, tighter sensor loops, improved error recovery, and more precise tool configurations. This result demonstrated that harness engineering is not a theoretical improvement. It produces measurable, competitive gains on standardized benchmarks.

### C. GAIA Benchmark: 44-Point Swing, Same Model

Dr. Adnan Masood documented a systematic comparison on the GAIA benchmark using identical instances of Claude Sonnet 4.5 [6]. With a minimal harness, the model scored 30.91%. With a production-grade harness including structured tool access, verification loops, and state management, the same model scored 74.55%. A 43.64-point improvement attributable entirely to the harness. This is larger than the performance gap between most adjacent model generations.

### D. Hashline: 15 Models Improved in One Afternoon

Security researcher Can.ac ran a systematic experiment across 16 different LLMs on a coding benchmark [9]. By changing only the harness, specifically the tool format and edit method, scores improved across nearly all models. The experiment was conducted in a single afternoon. No model was fine-tuned, retrained, or replaced. The harness change was a modification to how the tool called the model, not what the model knew.

### E. Ghostty: One Rule Per Failure

Mitchell Hashimoto's AGENTS.md for Ghostty, a terminal emulator, has accumulated rules one line at a time over months of agent-assisted development [1]. Each line traces to a specific agent error. The file serves as both a harness and a historical record. New agents reading the file inherit months of corrective knowledge in seconds. The ratchet in practice: the file grows when errors are new, and the growth rate declines as the most common failure classes are covered.

---

## APPENDIX: GLOSSARY

**TABLE XIV — KEY TERMS**

| Term | Operational Definition |
|---|---|
| Agent | Model + Harness: LLM with tools, memory, execution loop |
| Harness | Everything around the model: guides, sensors, loop, memory, permissions, observability |
| Guide | Feedforward control: instruction read before execution |
| Sensor | Feedback control: verification run after execution |
| Ratchet | Every failure becomes a permanent fix in the harness |
| Trip wire | Automated alert on aggregate behavior drift |
| Capability budget | Scope, rate, reversibility, visibility limits |
| Escalation | Structured handoff with decision packet + evidence |
| Inner harness | Safety layers built into base model by AI lab |
| Outer harness | Custom harness built by product team |

---

## REFERENCES

[1] M. Hashimoto, "My AI Adoption Journey," mitchellh.com, Feb. 5, 2026.

[2] R. Lopopolo, "Harness Engineering: Leveraging Codex in an Agent-First World," OpenAI, Feb. 11, 2026.

[3] B. Böckeler, "Harness Engineering for Coding Agent Users," martinfowler.com, Apr. 2026.

[4] E. Mollick, "A Guide to Which AI to Use in the Agentic Era," 2026.

[5] V. Trivedy, "The Anatomy of an Agent Harness," LangChain Blog, Mar. 10, 2026.

[6] A. Masood, "Agentic Harness Engineering," Google Cloud / Medium, Jun. 2026.

[7] LangChain Eng., "Improving Deep Agents with Harness Engineering," Feb. 17, 2026.

[8] Faros AI, "AI Engineering Report 2026: Acceleration Whiplash," Aug. 2026.

[9] Can.ac, "I Improved 15 LLMs at Coding in One Afternoon. Only the Harness Changed," 2026.

[10] L. Tan, "How Cursor Turned AI Agents Into Better Engineers," Aug. 12, 2026.

[11] Google Cloud, "Harness Eng. for Multi-Agent Systems Using Google ADK 2.0," Jul. 2026.

[12] Anthropic, "Building Effective AI Agents," anthropic.com, Dec. 2024.

[13] A. Osmani, "Agent Harness Engineering," addyosmani.com, Apr. 2026.

[14] OpenAI, "Codex Agent Best Practices," 2026.

[15] P. Schmid, "Trajectory-Capture as Competitive Advantage," 2026.

---

*Source Method. This document is an independent practical synthesis of the public engineering materials cited above. Product behavior may change. Recommendations, templates, pseudocode, and decision rules are the author's adaptation for implementation and are not official specifications of any organization mentioned. All diagrams are original.*
