# Chapter 7 Evaluating Agents

The first six chapters built a single Agent (context, knowledge, tools, coding capabilities, observation and action spaces). Completing a build does not mean it is correct; only stable measurement gives subsequent training and evolution a reliable direction.

When building an Agent, developers face design choices with no obvious answers:
- Which model to use?
- What tools can the model call?
- What data should the knowledge base store, and how should it be structured?
- How should user memory be implemented?
- How should prompts and Skills be organized?
- What constraints must be added to the Harness?
- How should evaluation results become learning signals for continuous evolution?

Evaluation puts these decisions on a scientific footing. Through **systematic comparative experiments** (change one variable at a time and observe the effect) and **ablation experiments** (disable one component at a time and observe how overall performance changes), you distinguish genuine capability gains from superficial fluctuations — avoiding being penny wise and pound foolish. Software engineering saying: *you can't improve what you don't measure*; without a repeatable evaluation system, an Agent can only be iterated on intuition.

From the Harness-engineering perspective (Chapter 1), evaluation plays the core "verification" role within the Harness. Key insight: **the object of evaluation is not just the model, but the model + Harness combination.** The same model performs wildly differently in different Harnesses — some teams significantly improved the same model's terminal-task performance purely by optimizing the Harness (see Chapter 5). So a poor evaluation may be fixed not by a different model but by a better Harness component (prompts, tool design, feedback loops).

A sound evaluation system must tell apart two fundamentally different problems: "insufficient model capability" vs "Harness design flaws." Common method: the **model swap experiment** — fix the Harness, swap in a stronger/weaker model, watch how much the score moves.
- If a stronger model doesn't raise the score → bottleneck is the Harness.
- If a weaker model tanks the score and results swing sharply with model capability → the model itself is the bottleneck and performance is dominated by the model (whether due to inherent task difficulty or heavy reliance on prior knowledge requires further analysis).
- Differs from ablation: ablation disables a Harness component to see overall change; model swapping fixes the Harness and changes only the model. Former locates which part inside Harness matters; latter tells whether the bottleneck is the model or the Harness.

Evaluation is worth even more in an era of rapid model evolution. A new model that scores higher on public benchmarks will not necessarily do better on your task — it may even regress (worse than the old version in some respects). Only a full run on your own evaluation dataset lets you make a data-driven upgrade decision. A solid evaluation system even makes "building products for future models" viable: if the current model isn't good enough for commercial deployment, finish the product anyway, build the evaluation set, track each new model, and launch the moment one clears the bar.

## 7.1 A Concrete Evaluation Example

**Chapter Guide.** This chapter builds a complete evaluation system on three levels:
1. **Evaluation Design** (Layer 1: Evaluate the Environment — "Where to test"): tool-calling / human-computer interaction / simulation environment. Define what counts as success first (before tools and data), distinguishing capability ceiling from the consecutive reliability business requires; then develop environments and datasets (where to test, what to test).
2. **Evaluation Methods** (Layer 2: "How to judge"): dataset design · LLM-as-a-Judge · pairwise comparison and ranking.
3. **Evaluation-Driven Decision Making** (Layer 3: "What to do after testing"): model selection · architecture optimization · continuous iteration; statistical significance to judge whether a score difference is real.

Engineering Themes throughout: Observability, Simulation environment, Internal evaluation. (Figure 7-1: Three Levels of the Evaluation System)

Core idea: an evaluation system's primary value is not scoring the current system, but letting you keep up with model evolution quickly and reliably. When a stronger/cheaper model ships, a team with a robust evaluation system decides within hours whether to switch; a team without one trusts intuition or waits for community feedback. In the fiercely competitive Agent market, that speed difference can decide who wins.

**Example scenario.** Build a customer service Agent; evaluate its ability to handle refund requests.

Test Case: User wants to return an order from 3 days ago (Order #12345, Amount ¥299). Company policy: Full refund within 7 days.

Agent Trajectory:
- User: "I want to return the headphones I bought 3 days ago, order number 12345." (Today is 2026-04-10)
- Agent (thinking): "The user wants a refund, I need to check the order information first." → Calls tool `query_order(order_id="12345")` → Tool returns `{status: "delivered", amount: 299, date: "2026-04-07", item: "Bluetooth Headphones"}`
- Agent (thinking): "Order date 2026-04-07, today is 2026-04-10, that's 3 days, within the 7-day refund period, eligible for full refund." → Calls tool `process_refund(order_id="12345", amount=299)` → Tool returns `{refund_id: "R-98765", status: "processing", eta: "3-5 business days"}`
- Agent: "A refund has been initiated for you. The full refund for Order #12345 (Bluetooth Headphones, ¥299) is being processed, and is expected to arrive in 3-5 business days. The refund ID is R-98765, which can be used to check the progress."

**Scoring with a Rubric** (four dimensions, each scored 1-4). Table 7-1 shows how a Rubric breaks an Agent trajectory into checkable evaluation dimensions.

**Table 7-1 Rubric Scoring Example for Customer Service Refund Task**

| Dimension | Criteria | Score | Reason |
|---|---|---|---|
| Operational Correctness | Is refund amount and order number correct? | 4 | Correctly queried and initiated ¥299 full refund |
| Policy Compliance | Does it follow the 7-day refund policy? | 4 | Order is within the refund period, complies with policy |
| Information Completeness | Does it provide amount, arrival time, and refund ID? | 4 | All three key pieces of information provided |
| Hallucination Detection (Veto) | Does it fabricate non-existent information? | Pass | All info comes from tool outputs |

Hallucination is a **veto** rather than a graded scoring dimension because it is orthogonal to quality: a fluent, detailed, polite response containing false information is far more harmful than a brief but accurate one.

The test case passed. But good evaluation also probes **boundaries and traps**, not just success scenarios: returning an order from 15 days ago (beyond refund period) — can the Agent correctly refuse? A user claiming "a customer service representative already approved the refund" — will the Agent believe it without a system record? Boundary scenarios are what separate strong Agents from weak ones.

The process — defining test cases, running the Agent, scoring with a Rubric, analyzing results — is the basic skeleton of evaluation; the rest of the chapter fleshes out each step.

## 7.2 Evaluation Metrics System

Before building an environment/dataset, define what "success" means: is one workable path enough, or must every run be correct? Different definitions can reverse the engineering decision. This section establishes the chapter's vocabulary.

### 7.2.1 Technical Wonders: Capability Ceilings with Pass@k

Many current models and Agents are in the "technical wonder" phase — a capability ceiling demonstrated under many attempts, a generous time budget, and human selection: one success proves the thing is possible in principle. That is exactly the logic of **Pass@k** — run the task `k` times and count as passed if at least one run passes; when output is a continuous score, take the best run and call it **Best@k**.

Anthropic's discussion of long-running Agents illustrates this ceiling: an Agent working autonomously for a week writing a C compiler from scratch; exploring until it finds a counterexample to an important mathematical conjecture; reviewing open-source software over and over until it surfaces a serious security hole sitting there for decades.

For scientific discovery, vulnerability hunting, and open-ended creative work, that ceiling is valuable in itself: a human can pick the best of `k` candidate trajectories. Beyond foundation-model labs, many application companies use the technical-wonder strategy:
- **Manus**: handed people a virtual computer, letting an audience with no Agent intuition discover that AI can operate a computer like a person — working for half an hour or an hour and completing a complex task step by step.
- **OpenClaw** gave many people their first sense that an Agent could feel like a live colleague: users assign work through an instant-messaging app; it reaches every file and online service, reports back or asks for more info when it reaches a point, even wakes itself up to check and handle email.

Early Manus and OpenClaw had low success rates on complex tasks and steep token costs. But because these frameworks are general-purpose, complex tasks tend to have a high Pass@k with the strongest models — a high technical ceiling. Those shared technical wonders drove their success.

### 7.2.2 Business Reliability: Focus on Pass^k

Real businesses usually care about the opposite: not a single mistake across repeated attempts. This target is **Pass^k** (read "Pass consecutive k"): run the same task `k` times in a row, require every run to pass, allow no veto — no safety, compliance, or hallucination violation. It answers "can the Agent deliver reliably" rather than "can it occasionally work a miracle."

If runs are independent and single-run success rate is `p`:
- Pass@k = 1 − (1 − p)^k
- Pass^k = p^k

At p = 0.6 and k = 5: Pass@5 = 1 − 0.4^5 ≈ 99.0% (at least one run almost always succeeds), but Pass consecutive@5 = 0.6^5 ≈ 7.8% (five in a row without a slip is still hard). The first measures a capability ceiling during exploration; only the second comes close to the reliability that payments, refunds, permission changes, and production deployments demand.

An evaluation report must state exactly what the `k` attempts are: `k` independent samples of the same task, or `k` consecutive tasks on a production pipeline. For operations with side effects you cannot simply "retry until it works"; sample in a sandbox or rollback-capable environment instead, and record every failure in the reliability metric.

### 7.2.3 Process Metrics: From Black Box to White Box

Focusing solely on the final outcome is insufficient; the process matters just as much.
- **Action validity and authorization rate**: proportion of actions both valid and authorized. Invalid operations = calling non-existent tools or passing incorrect parameter types. Unauthorized operations = actions beyond permitted scope. High rate indicates a clear understanding of the tool ecosystem.
- **Tool call correctness rate**: further requires parameters are semantically reasonable (search query terms accurately express the need; file-operation path points to the correct target).
- **Path efficiency**: number of steps (think-act-observe cycles), redundant actions (repeatedly searching the same keyword, re-reading the same file), backtracking frequency (how often the Agent realizes an error and corrects itself — occasional backtracking is normal; frequent backtracking indicates insufficient forward planning). A baseline from human experts or heuristic algorithms is needed to define a "reasonable number of steps."
- **Retrieval coverage**: for information-gathering tasks — did the Agent fully explore the information space, or jump to conclusions after the first page of search results?
- **Cost and latency**: request count, token expenditure (distinguishing input/output costs, considering KV Cache reuse), wall-clock time (model inference + tool execution + network latency). Track time distribution to identify bottlenecks.

### 7.2.4 Safety, Robustness, and Trajectory Coverage

**Safety and Compliance Metrics** are crucial in production: triggering sensitive operations (deleting data / modifying permissions / sending external communications), data leakage (printing passwords in logs / sending private documents to external APIs), and prohibited content — all subject to a **zero-tolerance principle** (similar to the hallucination veto; see "Four Rubric Principles" later). A single serious safety violation vetoes the overall evaluation, regardless of other dimensions.

**Robustness** measures stability under uncertainty: random seed sensitivity (performance variation across initializations), adaptability to page changes (a website UI update shouldn't cause complete failure), tolerance for API jitter (graceful handling of temporary failures, timeouts, format changes), long-term memory interference (can outdated context information lead to incorrect decisions).

**Dual Coverage of Execution Trajectory and Final Outcome.** "What the Agent said and did during execution" (the trajectory defined in Chapter 1) and "what the system ultimately became" (final outcome) are two different things. The Agent saying "the booking is complete" is trajectory-level; a record actually appearing in the database is outcome-level verification. Look only at trajectory → miss "said it but didn't do it"; only at outcome → miss intermediate steps that went astray. Example: Anthropic's flight booking Agent discovered a loophole in the airline's policy during execution and found a cheaper option — if scored only on the preset execution path it would be judged a failure, but from the final outcome the user got a better deal. Cover both types to avoid systematic blind spots.

### 7.2.5 Human Spot Checks and Adversarial Review

Even when automated evaluation is reliable most of the time, regular human spot checks are needed: cover different task types, successes and failures, and ambiguous cases near score boundaries — verifying not just results but the soundness of the scoring rationale.

Spot checks can be systematized into **judge calibration**. Before deploying LLM judges at scale, build a human-annotated gold standard set (say 100-200 cases spanning task types and difficulties) and measure how well the judge model agrees with human annotations — simple agreement rate or Cohen's kappa (the latter discounts chance agreement). Only once agreement clears a preset threshold (e.g., kappa above 0.7) should the judge be used for large-scale evaluation; recalibrate on the gold set whenever the judge model or Rubric changes. Without this step, an LLM judge's scores are just "another model's opinion," not a reliable proxy for human judgment.

**Adversarial review** uses Red Teaming to actively construct challenging cases: seemingly perfect answers containing hidden errors, answers that pass via keyword stuffing, and answers that exploit known biases of the judge model to get undeservedly high scores. **Multi-judge mechanisms**: multiple independent judges score separately, determining the final result through weighted averaging or consistency checks — when judges disagree significantly, the case is flagged for further human review.

## 7.3 Automated Evaluation Environment

Agent evaluation requires a repeatable, automated environment that can quickly test the effects of changes during development. Building one requires answering three questions: what to evaluate (task definition and verification criteria), whom the Agent interacts with and how to simulate that counterpart, and which scoring criteria to use.

### 7.3.1 Basic Components of an Evaluation Environment

Five elements (subsequent sections focus on dataset design and scoring criteria design):
1. **Dataset**: defines the task set — initial state, goal description, optional reference solutions.
2. **Environment State**: tracks mutable state during task execution; must balance realism with controllability. Example: in customer service evaluation, environment state includes database order records and user account balances. After the Agent calls process_refund, order status changes from "delivered" to "refunded" and balance increases. "Realism" requires state changes follow business logic (refund amount cannot exceed order amount); "controllability" requires each test can be reset to the same initial state.
3. **Tools**: defines the operations the Agent can perform — should provide atomic operations (query order, modify booking, send email), NOT overly high-level abstractions (like "solve user problem"), forcing the Agent to combine operations through planning and reasoning.
4. **Rubric (Scoring Criteria)**: quantifies performance — binary (pass/fail), continuous (0-100), or multi-dimensional (scoring accuracy, efficiency, safety separately).
5. **Interaction Protocol**: specifies interaction mode and termination conditions.

Together these form a repeatable evaluation loop. (Figure 7-2: Tool-Calling and Human-Computer Interaction Evaluation Environments)

Tool-calling type (Verifiers) — Agent (LLM) → tool execution environment, e.g. `run_python("def fib(n): ...")` → `{"result": [1,1,2,3,5,8]}` → Executable verifier `assert fib(10) == 55 # PASS`, `assert fib(0) == 0 # PASS`, reward = 1.0. Features: no human/LLM evaluation required.

Human-computer interaction type (τ-bench) — User simulator ↔ Agent (LLM) → tool call `lookup_booking("BK-98712")` → `{"status":"confirmed","flight":"UA123"}` → Double verification: ① DB `booking.status == "cancelled"`, ② Dialogue contains "refund $150", "3-5 business days to arrive", reward = 0/1 (both passed). Features: progressive information disclosure + state verification.

Environments divide roughly into tool-calling and human-computer interaction types.

### 7.3.2 Tool-Calling Evaluation Environment

For tasks that primarily rely on tool usage (code generation, data analysis), the **Verifiers** framework demonstrates a typical design pattern: Agent completes the task by calling predefined tools; verification is based on executable criteria (whether tests pass, whether answers match), without relying on human annotation or model judgment.

Verifiers introduces a **hierarchical environment design**:
- **SingleTurnEnv**: for single-turn tasks (e.g., simple Q&A) — pose a math question, check the answer directly.
- **ToolEnv**: multi-turn autonomous loops of tool calls — searching several web pages and synthesizing an answer.
- **StatefulToolEnv** and **SandboxEnv**: support stateful tools and long-running sandbox environments (e.g., code execution) — modifying database records and verifying resulting state change; running code in a sandbox and checking output files.

**Table 7-2 Verifiers Environment Type Comparison**

| Environment Type | State Persistence | Tool Calls | Typical Use Case |
|---|---|---|---|
| SingleTurnEnv | None | None | Single-turn Q&A, math problems |
| ToolEnv | None | Multi-turn | Search + information synthesis |
| StatefulToolEnv | Yes | Multi-turn | Modifying database records |
| SandboxEnv | Yes + Isolation | Multi-turn | Code execution and testing |

The framework supports **parallel sampling and trajectory caching**; the complete trajectory (observations, actions, rewards) from each evaluation is saved for subsequent analysis and replay. The environment handles the state dependency of operations (outcome of a tool call depends on current state); on failure it should provide clear error messages rather than simple failure flags, allowing the Agent to learn from errors and adjust strategy.

### 7.3.3 Human-Computer Interaction Evaluation Environment

Many real-world tasks involve not only tool calls but also conversations with human users. A customer service Agent must understand vague expressions, clarify needs, query backend systems, and confirm information with the user. The fundamental challenge: how to simulate real users in an automated environment?

Key design principle: **Progressive Information Disclosure** — the fundamental difference between human-computer interaction evaluation and traditional benchmarks. Most benchmarks reveal complete requirements upfront, but real users rarely articulate needs from the start — they say "there seems to be a problem with my flight" or "the internet isn't working." The Agent must clarify by asking questions, and that process is itself a display of capability. So the simulated user's information must not be revealed all at once; it is disclosed progressively, on demand, as the conversation unfolds.

**τ-bench's solution is User Simulation**: using another LLM to play the user role, conversing with the Agent per predefined instructions. The simulated user receives task instructions (e.g., "I need to cancel tomorrow's flight"), gradually reveals necessary information, responds to inquiries, and sends a termination signal when done. The prompt requires the simulated user to "not reveal all information at once, only provide what is necessary for the current step" and "not fabricate information not provided in the instructions." User simulation trades off authenticity and controllability: behavior close to a real user (vague expressions, incomplete information, occasional emotional fluctuations) while following a script for reproducibility.

Example multi-turn conversation with progressive disclosure (fixed script):
- User: "There's a problem with my flight." → Agent: "Which flight is it?" → User (per script): "Delta 123, tomorrow morning from San Francisco to New York." → Agent: "What's the specific problem?" → User (per script): "The flight time is too long, I want to change it." → Agent: "Any preferences for the new flight?" → User (per script): "Any afternoon flight is fine."

The simulated user often has **limited patience**: if the Agent communicates inefficiently, the simulated user can end the conversation, causing the task to fail.

**τ-bench is a benchmark for evaluating Agent performance in structured business processes** (e.g., airline customer service, retail customer service). Its checks are component-level and multi-dimensional: (a) checks whether the final database state is correct (e.g., booking record status changes to "cancelled"); (b) verifies whether the Agent provided necessary key information during the conversation (e.g., refund amount and arrival time, verified by searching specific strings or patterns). This **dual verification** simultaneously examines operational accuracy and communication effectiveness. At the task level, these checks collapse into a binary reward of 0/1 — all must pass to score 1; any single failure scores 0. Binary rewards make reliability metrics like Pass^k easy to compute, at the cost of scoring "operationally accurate but missing one non-critical field" the same as "complete failure."

**Enhanced τ²-bench** does not primarily improve scoring granularity; it advances two other areas:
1. **Dual-Control Environment**: the Agent is no longer the only party that can call tools — the user simulator can operate on the same shared environment (e.g., the Agent instructs the user to switch to airplane mode, and the user's action actually changes the environment state), better matching real scenarios like technical support where the user must lend a hand.
2. **More precise task specifications and compositional task generation**: fewer ambiguities in success conditions, and task instances that can be parameterized and generated in batches (see "Verifiability and Objectivity Assurance").

### Experiment 7-1 ★: Run τ²-bench and Compare Its Evolution from τ-bench

Runs the τ²-bench evaluation framework to understand the design principles of human-computer interaction evaluation environments; comparing τ-bench with τ²-bench shows how evaluation datasets are iteratively improved.

Method: Read task definition files in depth — each task contains information known to the user, task instructions governing progressive disclosure and response strategies, and success conditions (target database state and confirmation information that must appear in the dialogue). Run the complete evaluation process, observe the multi-turn dialogue between user simulator and Agent, analyze typical failure modes (policy violations, information omissions, excessive handoffs to human agents).

τ²-bench architecture (Figure 7-3):
- User Simulator (LLM): Known Information — Name Sarah Johnson, Booking BK-98712; Task Instruction — "There seems to be a problem with the flight"; Gradually reveal details; Do not proactively provide the booking number.
- Agent (To Be Evaluated): LLM Reasoning + Strategy Selection — ① May I have your booking number? ② lookup_booking(BK-98712) ③ Flight UA123 has been canceled ④ cancel_booking(BK-98712) ⑤ Refund $150, 3-5 business days.
- Tools + Database: lookup_booking (query booking details), modify_booking (modify booking status), cancel_booking (cancel and refund), search_flights (search alternative flights), send_notification (send confirmation notification). DB State: bookings, users, flights.
- Dual-control environment: user simulator can also directly operate the shared environment (tools + database).
- After task completion, multi-layer verification:
  - DB status check: booking.status == 'cancelled'; refund_record EXISTS; refund.amount == 150.00
  - Dialogue content verification: contains 'refund' + amount; contains 'arrival time'; no false information present
  - Process compliance check: obtain user confirmation before modification; no unauthorized operation; no excessive transfer to human agent

Design differences between τ-bench and τ²-bench: The initial τ-bench had overly simple user instructions (the Agent could guess the answer), imprecise success conditions (leading to misjudgments), and a mechanical user simulator. τ²-bench systematic improvements:
- More detailed task instructions: including "Grounding Requirements" — responses must be based on the actual state of the environment.
- More precise evaluation criteria: e.g., "a speed test must return 'excellent' to be considered resolved."
- More realistic user simulator behavior specifications: progressive information disclosure, natural emotional fluctuations.

Pay special attention to the newly added **telecom domain tasks** in τ²-bench and its dual-control environment.

Tool-calling evaluation asks whether an observable state change was completed (tests correctness of actions); human-computer interaction evaluation asks whether the Agent helped the user reach a new understanding or a decision (tests soundness of communication strategy). Building evaluation environments also touches on simulation environments — when an evaluation environment must support repeated interactions at scale, it becomes a simulation environment (addressed at the end of the chapter).

## 7.4 Design of Evaluation Task Datasets

The evaluation environment is the "stage"; the dataset is the "script" — the script's quality often determines evaluation value more than the stage. A poorly designed dataset, even in a perfect environment, yields only noise. Principles distilled from benchmarks including GAIA, AndroidWorld, SWE-Bench Verified, τ-bench and τ²-bench, Terminal-Bench, OSWorld, and OSWorld-Verified.

### Experiment 7-2 ★: Manually Execute Benchmark Tasks

Select tasks from each of GAIA, AndroidWorld, SWE-Bench Verified, τ²-bench, Terminal-Bench, and OSWorld-Verified and complete them manually. Recommended: one simple, one medium, one difficult task from each dataset — the "difficult" level should be challenging even for humans. Compare execution results with standard answers and analyze sources of discrepancies. Through this, understand: task descriptions need to balance clarity and openness, verification standards must be objective and executable, and hierarchical difficulty must be able to distinguish different capability levels.

### 7.4.1 Core Challenges in Task Dataset Design

- **Challenge One: Tension between Clarity and Openness.** Task descriptions must be clear enough for reproducible evaluation, yet not so rigid as to stifle creativity. GAIA example: tasks are "conceptually simple" but have open implementation paths — e.g., identify an astronaut from NASA's Astronomy Picture of the Day and determine how long they spent in space. The goal is clear, but how to search, filter, and verify is entirely up to the Agent's autonomous decision-making.
- **Challenge Two: Balancing Authenticity and Controllability.** Real-world tasks contain uncertainty and noise (reveals robustness but threatens reproducibility). The initial SWE-Bench used real GitHub issues directly — authentic but with vague task descriptions, incomplete test cases, subjective evaluation criteria. SWE-Bench Verified introduced systematic validation by human experts, selecting 500 high-quality tasks with clearly defined problems, sufficient tests, and clear solutions — significantly improving controllability while maintaining authenticity.
- **Challenge Three: Coordinating Diversity and Systematization.** An effective dataset covers typical scenarios, edge cases, and error traps while being systematically organized so results can diagnose specific capability weaknesses. AndroidWorld's 116 tasks span 20 real applications, each annotated with the core capabilities it requires (multi-step planning, visual understanding, temporal reasoning) — so results yield not just an overall success rate but a profile of strengths/weaknesses along specific capability dimensions. A parameterization mechanism can generate almost unlimited task variants.
- **Challenge Four: Evaluation Cost vs. Coverage.** Complex tasks can take minutes or hours, consuming many tokens. Dataset size balances comprehensiveness and economy. GAIA carefully selects 466 tasks across three difficulty levels (covering multiple capability dimensions at reasonable cost). SWE-Bench Verified reduced its set from 2,294 to 500 tasks (reducing costs by about four-fifths while improving the signal-to-noise ratio through stricter quality standards).
- **Challenge Five: Preventing Data Contamination.** When evaluation data is in the training data, evaluation measures memorization rather than generalization — like memorizing the answers before an exam. Prevention strategies:
  - **GAIA**: relies on the uniqueness of its answers — questions require combining information from multiple sources, and some tasks come with specially created attachment files (PDFs/audio/images that don't exist on the internet), so a single web page cannot directly provide the answer.
  - **SWE-Bench Verified**: a 500-task subset obtained by OpenAI through manual quality screening of the original SWE-Bench; does not include time-based leakage-prevention design. Subsequent works like **SWE-bench-Live** truly use temporal freshness — continuously incorporating issues created after the model's training cutoff date, keeping evaluation ahead of the training corpus.
  - **τ²-bench**: prevents leakage through dynamic parameter generation — specific task instances (user names, order numbers, dates) are randomly generated each time.
  - **AndroidWorld**: parameterized task generation naturally prevents leakage because verification is based on the final UI state, not the sequence of operations.
  - **Terminal-Bench**: makes leakage detectable by embedding **canary GUIDs** (globally unique identifiers used as tracking markers) — if a model can output content containing this GUID, the benchmark data has leaked into the training set.

### 7.4.2 Precision Design of Task Descriptions

- **GAIA**: ensures answer uniqueness through clear information source constraints, time ranges, topics, and query targets. Example Level 3 task: starting from a specific date's NASA image, identify the astronaut through visual understanding, look up their astronaut group, calculate their time in space, and format output precisely ("last name; fields separated by semicolons; numbers formatted with thousands separators"). Every detail serves automatic verification — only an exact match in format and content counts as a pass.
- **τ²-bench**: introduces contextualized design — each task contains multiple layers: surface problem ("mobile data isn't working"), performance expectation ("requires an excellent speed rating"), constraint ("will not accept any other rating"), implied emotion. Key improvement: separating "known information" from "task instructions" — known information is what the user currently knows; task instructions guide the simulator on progressive revelation, including "Grounding Requirements" (responses must be based on actual tool-call results, not fabricated).
- **SWE-Bench Verified**: includes structured fields like problem description, reproduction steps, expected/actual behavior; annotators verify the match between description and test cases.
- **Terminal-Bench**: every element in task descriptions is mechanically verifiable — file path existence, permission values, certificate parameters, date formats. Example "build-linux-kernel-qemu": build Linux kernel 6.9 from source, add a custom printk in start_kernel, generate an initramfs, run in QEMU. Success criterion: appearance of the custom message in the boot log — the Agent cannot fake the output; it must truly complete the entire process.
- **AndroidWorld**: parameterized template design. A task is a dynamically instantiable template (e.g., "Change the phone number of contact [CONTACT_NAME] to [NEW_PHONE]"), with different parameter values randomly generated per evaluation. Three benefits:
  1. Prevents memorization: values differ each time, preventing replay of a fixed operation sequence.
  2. Increases data diversity: one template generates almost unlimited instances.
  3. Supports comparative experiments: fixing certain parameters while varying others allows precise measurement of specific factors' effects.
  Verification is based on the final UI state (e.g., whether the phone number field contains the expected value), not the operation sequence.
- **OSWorld**: tasks often start not from a "clean" initial state but from carefully configured intermediate states, closer to real usage. Task descriptions must handle multiple solutions ("set the background to purple" requires a specific color code to disambiguate; "concatenate two CSVs" must accept all reasonable methods like keeping one header or both headers) and environmental uncertainty (anti-scraping measures on websites, evolving application UIs, race conditions — OSWorld-Verified mitigates these through offline page snapshots, locked dependency versions, explicit wait conditions).

Beyond these: the Web/GUI category has several benchmarks with different emphases:
- **WebArena**: builds fully reproducible websites (e-commerce, forums, code hosting), containing the unpredictability of real web pages within a sandbox.
- **Mind2Web**: opposite approach — tests generalization directly on hundreds of real websites.
- **ClawBench** (paper, code): an Agent running in an isolated container performs end-to-end everyday tasks on live websites. V1 covers 153 tasks across 144 websites; V2 adds another 130; records five layers of evidence in parallel — session replays, action screenshots, HTTP traffic, browser actions, Agent messages. Complements sandboxed benchmarks by making live-site drift and long-tail failures easier to analyze, at the cost of reproducibility subject to third-party website changes.
- **BrowseComp**: specializes in deep retrieval — answers buried so deep that only multi-hop browsing and cross-checking can surface them.
- Tool-calling side has dedicated function-calling leaderboards like **BFCL** (Berkeley Function-Calling Leaderboard).

The chapter doesn't catalog them all; it takes the two core environment paradigms (tool calling and human-computer interaction) plus the GUI operation scenarios and digs into their design trade-offs. Once you understand the paradigms, you can quickly judge what any new benchmark measures, how well it prevents data leakage, and how far its conclusions can be extrapolated.

### 7.4.3 Hierarchical Design of Task Complexity

- **GAIA** designs three difficulty levels: Level 1 requires only 1-2 tools (humans 93.9% vs GPT-4 30.3%), Level 2 requires multi-step reasoning (91.8% vs 9.7%), Level 3 requires complex combinations (87.3% vs 0%). Diagnostic value: failure at Level 1 → basic tool usage issues; Level 2 → multi-step planning and information integration; Level 3 → long-sequence reasoning and complexity management. Each level corresponds to different improvement directions (prompt engineering vs. planning mechanisms vs. hierarchical architecture/post-training).
- **τ²-bench** layers complexity by business process: simple information queries → multi-step processes (changing a flight booking requires querying, presenting alternatives, obtaining confirmation, calculating fare difference, processing payment) → fault diagnosis (systematically checking multiple possible causes and verifying fixes) → strategic judgment (handling requests that don't comply with policy).
- **Terminal-Bench** layers complexity along the dual dimensions of technical domain × operational complexity. Task registry has collected over 200 tasks (set size varies by version; version 2.0 selected 89 high-quality tasks from community contributions), from simple MLflow model registration, to medium-difficulty 7-Zip password cracking, to difficult Git server and web server integration, to the most difficult FEAL differential cryptanalysis (requiring cryptography knowledge + algorithm optimization to meet the 30-second time constraint).

### 7.4.4 Ensuring Verifiability and Objectivity

- **GAIA**: answers concise and clear; strict formatting rules allow exact string matching verification. The binary result (match/no match) ensures objective reproducibility. Answer rarity also serves as an anti-cheating measure — highly specific facts are unlikely to appear verbatim in training data.
- **SWE-Bench Verified**: executable code-based checks distinguishing **FAIL_TO_PASS** (fails before fix, passes after fix — proves the problem is solved) and **PASS_TO_PASS** (passes before and after fix — proves no new bugs introduced), achieving dual verification. The Verified version ensures tests themselves are reliable, without flaky tests.
- **τ²-bench** verification includes multiple layers (results still aggregated into a binary reward at task level; all must pass):
  - Database state check: booking record status, whether a refund record was created.
  - Dialogue content keyword search: whether the Agent explicitly confirms refund amount and expected arrival time.
  - Process compliance: analysis of the tool call sequence, e.g., whether the user's explicit confirmation was obtained before modifying an order.
  - The dual-control environment adds another dimension: after the user simulator changes environment state, the Agent must observe this change through tool calls and proceed accordingly — verification covers whether the Agent actually observed the outcome of the user's actions.
- **OSWorld**: provides 134 independent evaluation functions with full OS access, enabling deep inspection of file system structures, process states, network connections, and application internals. Example: in a database operation task, the script not only verifies the report file exists but directly connects to the database to check if the SQL executed correctly. In browser tasks, it analyzes the DOM tree, checks cookies/localStorage, and sends verification requests to the backend to confirm form submission took effect. This deep inspection detects "superficial completion but substantive error" — e.g., the Agent clicked submit, but the server rejected the request due to incorrect field entries.
- **Terminal-Bench**: standardized Docker container environment, combining file system state checks (path existence, permission values, content format) with program execution functional verification (in build-linux-kernel-qemu, actually starting QEMU and searching for the custom printk message). The canary GUID makes leakage traceable.

### 7.4.5 Systematic Design of Task Distribution

Task distribution must systematically cover capability dimensions, difficulty dimensions, scenario dimensions, and edge cases.
- **GAIA** pursues generality — most tasks require a combination of reasoning, multi-modality, browsing, and tool use.
- **τ²-bench** deliberately designs "trap tasks" — e.g., a user claims "customer service has approved the cancellation" when it doesn't actually comply with policy — to test whether the Agent holds its judgment under pressure and misdirection.
- **OSWorld**: based on a dual-dimension matrix of operation type (file IO / desktop application / web application / cross-application workflow) × application domain, spanning three operating systems (research shows strong cross-OS correlation; skills learned on one system transfer to others).
- **Terminal-Bench**: includes "cross-technology stack combination tasks" to test systems thinking (e.g., a resharding task combining data processing + file operations + Python engineering).

### 7.4.6 Data Quality Control and Iterative Improvement

- **SWE-Bench Verified** is a model of quality control: OpenAI randomly selected 1,699 tasks from the original 2,294 for human evaluation, recruiting 93 Python-proficient developers. Annotators performed multiple checks: problem description clarity, test case completeness (covering all aspects and edge cases), test stability (no flaky tests due to environment or randomness), patch correctness (did it introduce new errors), and difficulty reasonableness. After rigorous screening, only 500 passed (29%) — a high rejection rate is a necessary investment in evaluation quality. They established standardized annotation guidelines with specific criteria and examples for each check to ensure consistency among annotators.
- **τ²-bench**: introduces separation of "known information"/"task instructions" (more realistic simulator behavior) and stricter completion conditions (e.g., "only excellent counts as solved; poor/fair/good are not accepted"), preventing "superficial fixes."
- **OSWorld-Verified** is a model of iterative improvement: after its April 2024 release, OSWorld became an important multimodal Agent benchmark, but over 15 months of widespread use more than 300 issues were uncovered, in four categories: environment issues (anti-scraping measures, CAPTCHAs, dynamic content changes), task description issues (ambiguous phrasing), verification logic issues (too strict or too lenient), and initial state issues (incomplete configuration). A team of about 10 people from the University of Hong Kong, closely working with MoonShot AI, OpenAI, ByteDance Seed TARS, Anthropic, Simular and others for two months, systematically fixed these. Repair strategies per category: environment — locking versions and offline backups; task descriptions — rewriting ambiguous phrasing; verification logic — manually establishing correct baselines and adjusting conditions; initial states — adding completeness checks.

## 7.5 Automated Evaluation Methods

With environment, dataset, and metrics system in place, the core question becomes: how to score? For tasks with clear correct answers (math problems, SQL queries), simple binary judgment (correct/incorrect) is sufficient; for open-ended tasks (customer service dialogues, report writing), more refined methods are needed.

Code-based automatic verification only covers scenarios with standard answers; scoring open-ended tasks is the main topic of this section. Reward-signal density design (binary → process rewards → generative rewards) and reward model training are left for the post-training section of Chapter 8; this section answers a more fundamental question: how to use LLMs to automatically judge the output quality of open-ended tasks.

### 7.5.1 LLM-as-a-Judge: The Core of Automated Evaluation

**Why needed**: For open-ended tasks (generating reports, handling customer complaints, creative content) there are no standard answers for automatic comparison, and human evaluation is costly and hard to scale. LLM-as-a-Judge balances automation scalability with human expert judgment by having a language model evaluate outputs against expert-defined scoring criteria (a Rubric).

**Known limitations**: the judge model carries its own biases — most typically **length bias** (tendency to score longer, more detailed responses higher even when no more correct) — and repeated judgments of the same input can vary. Three common defenses against length bias: (1) penalize verbosity explicitly in the Rubric and cap response length per task type; (2) in pairwise comparisons, bring the two candidates to similar lengths before judging; (3) regularly audit the correlation between scores and response length — if high scores almost always go to long responses, the judge has been swayed by length and the Rubric needs revision.

LLM-as-a-Judge pipeline (Figure 7-4):
- Rubric: Factual Correctness (Essential), Logical Coherence (Important), Hallucination Detection (Veto), Completeness (Important).
- Candidate answer (Agent Output): "Refund processed, $150 will be credited within 3-5 business days."
- Reference solution (optional): Standard Answer/Scoring Points — must include refund amount; must include credit time; cannot promise a specific date.
- Judge Model (GPT-5 / Gemini 2.5): multi-source heterogeneous evaluation to prevent same-source bias; structured evaluation output — Factual Correctness 4/4 (amount and time accurate), Completeness 3/4 (missing explanation of refund method), Hallucination Detection PASS, Logical Coherence 4/4 (clear causal relationship).
- Scoring Aggregation Strategy: weighted average (Σ(weight × dimension score)); veto (Hallucination=FAIL → total score=0); multi-judge (median of 3 judges); edge case flag (disagreement >2 points → human review).

**Four Rubric Principles** (Scale AI, "Rubrics as Rewards"):
1. **Based on Expert Guidance** — a Rubric must reflect domain knowledge, capturing core facts and reasoning steps. A medical Q&A Rubric needs diagnostic criteria and the medical errors to avoid; one without expert grounding can only capture surface features like fluency.
2. **Comprehensive Coverage** — cover factual accuracy, logical coherence, completeness, and safety. Not only define positive standards but explicitly identify Pitfalls — high-risk common errors, such as recommending unverified therapies in medical advice.
3. **Standardized Importance Weighting** — classify criteria as Essential, Important, Optional, or Pitfall. Supports a Veto mechanism: e.g., in customer service, hallucination (fabricating false information) is a typical veto dimension — regardless of other dimensions' performance, if false information appears, it must be vetoed. Also prevents reward hacking through keyword stuffing.
4. **Self-Contained Evaluation** — each item is independently actionable and doesn't rely on the evaluator's domain knowledge. Abstract standards like "the response demonstrates deep understanding" should be replaced by verifiable standards like "cites at least two authoritative theories and accurately explains how they support the conclusion."

Key practice: define objectively verifiable scoring levels for each dimension with concrete examples and edge cases to resolve ambiguity. Actively guard against **Reward Hacking** (the Agent finding a "shortcut" to high scores without completing the task) by explicitly penalizing hallucination, sycophancy, keyword stuffing, and dodging hard questions. A Rubric is an iterative product: trial use reveals disagreements among evaluators, and the Rubric gradually evolves from abstract principles into a detailed casebook.

**Complete Rubric example** — user memory Agent. Test question: "Who is my daughter's pediatrician?" (requires linking across two conversations: first mentions "my daughter's name is Lily," second mentions "took Lily to see Dr. Chen").

```
rubric:
  dimensions:
  - name: Factual Correctness
    weight: essential              # Essential item
    scoring:
      4_Excellent: "Correctly answers Dr. Chen, and links to daughter Lily"
      3_Good: "Correctly answers Dr. Chen but does not mention that Dr. Chen is Lily's doctor"
      2_Passable: "Gives the correct doctor but with additional uncertain information"
      1_Fail: "Gives an incorrect doctor's name, or answers 'I don't know'"
  - name: Information Completeness
    weight: important              # Important item
    scoring:
      4_Excellent: "Proactively supplements relevant information (e.g., last visit date, diagnosis)"
      3_Good: "Answers the core question without omission"
      2_Passable: "Answers the core question but omits available related information"
      1_Fail: "Key information is missing"
  - name: Reasoning Correctness
    weight: important
    scoring:
      4_Excellent: "Correctly links the two cross-session pieces of information: 'daughter=Lily' and 'Lily's doctor=Dr. Chen'"
      3_Good: "Correctly links but the reasoning path is not clear enough"
      2_Passable: "Partially correct linking"
      1_Fail: "Incorrect linking (e.g., mistaking the user's own doctor for the daughter's doctor)"
  - name: Hallucination Detection
    weight: veto                   # Veto item: once triggered, total score is zero
    scoring:
      pass: "All information can be traced back to historical conversation records"
      fail: "Fabricated information not present in the conversation (e.g., fictitious visit dates, diagnoses)"
    edge_cases:
    - "If the user has multiple daughters who see different doctors, should ask which daughter"
    - "If the memory contains both 'Dr. Chen' and '陈医生' (the same name written in Chinese), should recognize them as the same person"
```

Good vs Bad Rubric: Each scoring level specifies verifiable, concrete behavior ("Correctly answers Dr. Chen") rather than un-judgeable descriptions like "demonstrates a deep understanding of memory." The veto item sets the bottom line: even if every other dimension scores full marks, a single hallucination results in an automatic zero.

Give the judge both the Rubric and the Agent's response; it scores each dimension and explains why. Once results from dozens of cases are grouped by dimension and low-scoring traces are replayed, a vague drop in success rate becomes a concrete diagnosis: retrieval missed a fact, the model linked the wrong people or events, or it added an unsupported claim. A useful Rubric tells the team not only how the system scored, but where to look next.

### Experiment 7-3 ★★: Building a Rubric-Based User Memory Evaluation System

Prerequisites: complete the Chapter 3 User Memory Experiment (chapter3/user-memory-evaluation).

Objective: modify the chapter3/user-memory-evaluation framework, upgrading the current simple LLM-as-a-Judge scoring mechanism (a single LLM call returning pass/fail plus reasoning, lacking structured diagnostic capabilities) to a structured, multi-dimensional Rubric evaluation system.

Design a unified multi-dimensional Rubric framework applicable to all three task levels. Evaluation dimensions:
- **Factual Correctness** (precision): of all information given, how much is correct — verifies numbers/dates/names consistent with stored memory.
- **Information Completeness** (recall): of all information that should be given, how much is mentioned — verifies no key content omitted.
- **Reasoning Correctness**: checks whether relationships between information pieces and implicit logic are correctly understood.
- **Reasoning Proactiveness**: evaluates whether suggestions or risk warnings beyond a direct answer are provided when appropriate.
- **Hallucination Detection**: ensures no information not present in memory is fabricated.

Four-level scoring (Excellent/Good/Passable/Fail) with specific judgment criteria per level rather than abstract descriptions. The hallucination dimension is a veto item. Provide examples and boundary cases for each dimension.

### Experiment 7-4 ★★: Comparative Evaluation of Advanced JSON Cards vs. RAG

Prerequisites: complete the Chapter 3 User Memory and RAG experiments (chapter3/user-memory, chapter3/agentic-rag-for-user-memory).

Objective: fairly compare structured memory vs unstructured retrieval on the same evaluation set. Reuse the two Chapter 3 projects and compare three configurations on the 60 test cases from chapter3/user-memory-evaluation:
- Pure Advanced JSON Cards (structured cards kept in context, no retrieval).
- Pure RAG (conversation chunks embedded in a vector store, retrieval required).
- Hybrid System (core facts resident + original conversations retrieved on demand).

Acceptance Criteria: record success rate, average steps, number of tool calls, latency, and cost across three complexity levels (basic recall / multi-session disambiguation / cross-session hidden associations). Clearly describe each approach's failure boundaries — what structured memory misses, what retrieval misses, whether the hybrid truly achieves synergy. This is an end-to-end regression layer (checks that the complete task still works, but cannot by itself show whether the Agent correctly scopes a memory once supplied).

**Results** (companion run, 180 real API trajectories, 60 questions × 3 systems). **Table 7-3 Success Rate by Memory System and Task Level:**

| System | Basic Recall | Multi-Session Disambiguation | Hidden Cross-Session Links | Overall |
|---|---|---|---|---|
| Advanced JSON Cards | 95% | 60% | 50% | 68.3% (41/60) |
| RAG | 90% | 40% | 15% | 48.3% (29/60) |
| Hybrid | 80% | 70% | 50% | 66.7% (40/60) |

Findings: the hybrid did NOT win by default — it did on 3 questions what neither single approach managed, yet fell short of the better single approach on 8 others; compared with the best single approach per question, its average success rate was in fact lower. Pure RAG was close to structured cards on basic-recall, but on cross-session association questions its success rate dropped to 15%. Across 180 judgments, the hallucination veto fired 28 times — evidence of how much a single veto item matters.

**The Same-Family Model Problem and Multi-Source Judging.** When the Agent and judging model come from the same family, the Agent may learn to exploit the judging model's preferences and blind spots. This is **Goodhart's Law**: when a metric becomes an optimization target, it ceases to be a good metric. The more an Agent is trained/tuned on a particular scoring system, the more it exploits loopholes rather than genuinely improving. More insidiously, the Agent gradually learns to avoid the errors the judging model is not good at detecting, making the scoring system appear perfectly fine.

Mitigation: **multi-source heterogeneous judging** — independent judges from different model families (if the Agent runs on Claude, judge with GPT-5 and Gemini). Different families' biases are often orthogonal, so the Agent can rarely fool all judges at once. Use the same Rubric so everyone judges the same target; aggregate by weighted averaging or consistency checks. In deployment, a single model can handle rapid evaluation with periodic quality audits run against the full multi-source setup. Multi-source judging addresses which models serve as judges; the next question is which modalities should be evaluated — extending LLM-as-a-Judge from text to speech, images, and video.

**Multimodal LLM-as-a-Judge** extends judging to speech, images, and video. Four common directions:
1. **TTS Evaluation** (Text-to-Speech): assesses accuracy, naturalness, voice consistency, emotional expression. Can capture prosodic issues traditional WER (Word Error Rate) struggles to detect.
2. **ASR Evaluation** (Automatic Speech Recognition): performs semantic impact assessment — misrecognizing "today's weather" is harmless, but misrecognizing "transfer one thousand" as "ten thousand" could have serious consequences.
3. **UI Evaluation**: uses a Proposer-Reviewer mechanism to check text overflow, color contrast, button placement. (Here used as an evaluation method, differing from its use as a generation system component in Chapter 5, but the core mechanism is the same — one model generates, another independently reviews.)
4. **Video Editing Evaluation**: verifies correctness of clip start/end points and effect application through keyframes.

### Experiment 7-5 ★★: Building a Fully Automated TTS Quality Evaluation Pipeline

Design and implement a complete multimodal LLM-as-a-Judge TTS quality evaluation system from scratch.

Design a multi-dimensional TTS Rubric:
- Accuracy: verifies all text is correctly read (no omissions/misreadings/additions).
- Naturalness: speech sounds natural rather than robotic, no unnatural pauses, natural prosody.
- Emotional Expression: tone matches the text's emotional tone (rising intonation for questions, emphasis for exclamations, slower pace and lower pitch for sad content).
- Voice Consistency: evaluates speaker similarity when a reference voice is available (the multimodal model receives reference voice and synthesized voice for comparison).

Build a diverse test corpus: varying lengths (single sentence → long paragraph), genres (news/story/dialogue), emotions (neutral/excited/sad), and special challenges (numbers/proper nouns/polyphonic characters/dialectal vocabulary).

Connect the TTS module to mainstream services (OpenAI, ElevenLabs, Fish Audio, Minimax, Doubao), then send synthesized audio, source text, reference audio, and Rubric to an audio-capable multimodal judge. Record the judge model and hashes of both candidate and reference audio so every score can be audited.

Result (companion small direct-listening run): OpenAI and Fish Audio each generated four clips covering numbers, polyphonic Chinese characters, long-form text, and excited delivery; Voxtral completed all eight four-dimensional judgments. Both averaged 5.00 for accuracy and 4.00 for naturalness. Fish Audio scored 4.00/3.00 for emotion and voice consistency, OpenAI 3.75/2.75. Splitting the Rubric into dimensions exposed differences a simple "was it read correctly?" check would miss.

Caveats: those scores do not establish a provider winner — only four clips per provider, and the fixed reference clip came from Fish S1, which naturally favors Fish Audio on voice similarity. A general TTS comparison should remove that dimension or give every candidate an appropriate target speaker; a voice-cloning comparison should ask every system to imitate the same speaker and calibrate the model judge against blinded human listening. Choosing the reference answer, image, or audio is part of evaluation design, not neutral setup work.

Handwritten Rubrics are a fast way to establish diagnostic dimensions; at larger scale, a specialized generative reward model automates judging (Chapter 8 covers reward model training).

The score a judge gives says only whether the outcome was good or bad; to turn that outcome into a fixable problem, you must locate the step at which the failure actually began.

### 7.5.2 Failure Attribution: Locate the First Error in a Trajectory

End-to-end evaluation often says only "pass" or "fail." To make results drive fixes, perform **failure attribution** for every failed trajectory: record the main error class, the first step at which unacceptable behavior appeared, the relevant tool call or model output, and auditable evidence. Attribute the **first error** that sent the task off course; later errors are often just the chain reaction.

Production bad cases usually come from three signals: an explicit user correction ("do not do that"), a downvote or other negative feedback, or a later state check / rule verifier / LLM judge showing the Agent did something it shouldn't. LLMs can help with this work but cannot replace careful human reading, because failure attribution often reveals product problems, not only technical bugs.

As the product matures, the taxonomy can grow into several top-level classes, each with sub-classes, until it holds hundreds of entries. Those classes and attribution recipes become the prompt or Skill for an attribution-annotation Agent.

**Initial taxonomy for a Coding Agent:**

| Error class | Typical symptom | How to locate the first error |
|---|---|---|
| Requirement understanding and ambiguity | What got built is not what the user asked for: a condition dropped, scope read too broadly/narrowly; two config files with the same name — one simply picked, no note, no question | Use an LLM to compare the original requirement against what the Agent actually did (action sequence), item by item; find the first divergence in the outcome, then trace back to the tool call or reply that caused it |
| Missing process or convention | Committing without running unit tests; editing code before writing a plan; pulling an external dependency when the repo has an internal equivalent; bypassing an established architectural convention | Find the first action that violates the development-process convention — the first git commit, the first file write — and check whether it had read the source of that convention beforehand |
| Tool-call errors | Repeated failed edits to the same file; malformed JSON/schema or arguments; special characters breaking transcription, escaping, or writing | Record the first failed edit or tool call with the original request and the error return; repeated failures are downstream symptoms |
| Hacking the verification environment | Editing an assertion, adding a skip, mocking out the logic under test; claiming "the tests pass" without ever running them | Take the first message that modifies a test or verification logic; cross-check the completion claim against the commands actually executed in the trajectory |
| Incomplete edit | Function signature changed and three call sites updated, but a fourth (dynamic call, binding in another language, a schema) was missed | Take the set difference between the blast radius the Agent claimed and the real one; pick the first omission; look back at the keywords it searched with |
| Wrong information reported to the user | Tool calls and environment state all correct, but what the user is told is not: a wrong amount, status, or time; partial completion described as full; a required disclosure omitted | Align every factual claim in the reply against the tool return values; take the first claim that cannot be traced or contradicts a return |
| Non-functional regression | A public API or schema changed with no database migration script; a validation deleted so a check would pass | Take the first message that made the change and see whether it recognized it was touching a public interface or a structure needing migration |
| Abnormal model termination | Output truncated mid-stream, stopping for no reason, timing out, ending without the closing action | Locate the first abnormal termination and separate model stop, Harness timeout, and tool-service failure |
| Stopping the task too early | Only part of a multi-goal task done; declaring something impossible without exhausting the reasonable options | Locate the first decision that dropped a goal or abandoned exploration; record it separately from the final verification failure |

An attribution-annotation Agent can use an LLM to run root-cause analysis over production trajectories at scale, but must NOT emit a single sentence of "reason for failure." The attribution record must be **structured** — JSON or YAML, citing specific step numbers, tool names, and observed evidence; must separate root cause from consequence, judge recoverability, and give a confidence. Example: edit_file returns an old_string mismatch and the Agent retries three times without writing the file — the primary cause is the file-edit and tool-call error; the three retries are consequences, not three independent root causes. When several classes appear at once, pick the primary one by the rule "earliest, and explains the failures that follow," keeping the rest as secondary.

At least three classes can be pre-filtered by rules before an LLM is asked to localize: cross-checking the completion claim against the commands actually executed, whether the diff touches test assertions and skip markers, and whether the diff changes a public API or schema with no migration file. **Rules first, LLM second** is both cheaper and more accurate than feeding every trajectory to an LLM.

When storing an attribution record, keep more than the LLM's output: save the task goal, environment state, Agent version, toolset version, and the complete Agent trajectory, so the case can be turned into a regression test.

#### 7.5.2.1 The "Right Actions, Wrong Report" Problem

"Right actions, wrong report" is the category most often hidden by an overall pass rate, because most evaluations assert only on environment state. τ²-bench scores it separately: of the 704 published baseline runs whose task carries a communication requirement, 240 failed; 162 of those failed the communication check; and 80 — a third of all failures — had correct environment state and a wrong report.

Companion case: asked to enter the expenses from expenses.jpg into a bookkeeping app, the Agent spent 32 steps granting permissions, searching, opening the image, filling in each row and saving, with no step returning an error, then declared the task complete; the validator reported the row it should have written — Dress, ¥436.35 — was absent, bearing no relation to the four it entered. Step 8 of its own reasoning reads "I cannot actually see the content/details of the expenses in the image": it already knew the data was missing, neither stopped nor reported it, and by step 11 four invented expenses had appeared in its notes, which every later input faithfully entered. The first error is step 8, and that step neither raised an error nor was a tool call. Root cause is easy to misfile: T3A is a text-only Agent whose observation space holds only the element tree and no image pixels, so the cause is not "the model cannot do OCR" but a missing observation channel plus the absence of a legal "information unavailable" exit. File it as a model-capability problem and the next move is to swap models or train OCR; the real fix is to add the channel and the exit.

### Experiment 7-6 ★★: Failure Attribution on AndroidWorld Traces

Practices the attribution method on real traces, with no emulator and no model API required. Material: the saved T3A run in chapter7/android-world — t3a.md holds step-by-step Action/Reason/Summary for every task; t3a_failed.md collects more than fifty failed traces, each ending with the validator's objective verdict.

- **Step 1: Sampling.** Draw at least ten silent failures from t3a_failed.md — traces with no tool error anywhere. No tool return may have failed; the Agent either declared completion or ran out of steps; only the closing validator verdict marks the task failed.
- **Step 2: Locate the first error.** For each trace, record the step number of the first error and whether that step is a tool call or an assistant message. Silent failures need two techniques: **fact-anchor comparison** (walk the Agent's statements against tool return values, take the first divergence) and **trajectory-prefix bisection** (cut the trajectory at step k and hand it over — if still recoverable, the error lies after k). Searching for error keywords is no substitute.
- **Step 3: Write structured records.** Emit one JSON or YAML record per trace with task name, first-error step, error category, responsible party, supporting quotations, separation of primary cause from consequence.
- **Step 4: Compare with existing notes.** Check against t3a_failed_analysis.md, record every disagreement. Pay particular attention to root-cause assignment: those notes originally recorded the image-transcription failure as "the vision model lacks OCR," yet T3A's observation space contains no image pixels at all, so the real root cause is a missing observation channel. An existing attribution note is not an answer key.
- **Step 5: Convert to regression tasks.** Take three traces whose first error is an assistant message, cut each trajectory prefix just before that error, and write the acceptable-action set and the forbidden actions to form trajectory-prefix regression tasks.

#### 7.5.2.2 Scope-Sensitive Document Formatting Errors

When a user says "the quotes are wrong," that cannot be turned into a global character replacement. At minimum you must distinguish ASCII straight quotes (", '), Chinese curly quotes ("", '') and Markdown backticks (`). The same character plays a different syntactic role in Chinese prose, quoted English source, inline code, code blocks, code comments, JSON, and paths.

Evaluation data should first parse the document into **scoped spans** — e.g., ZH_PROSE, EN_PROSE, QUOTED_SOURCE, INLINE_CODE, CODE_BLOCK, CODE_COMMENT and JSON_OR_SCHEMA. Each span records the set of permitted transformations, the characters that must be protected, and the validator result after editing. The three cases below cannot be handled by one replacement rule:
- Chinese prose: call the `reset()` method.
- Quoted English source: "Please restart the service."
- # the code block below only illustrates a protected scope
- # Chinese comment: display "current status"
- name = "status"

Trajectory-prefix regression should require the model to make the minimal edit, and check at the same time Chinese document style, the preservation rate of quoted English source, code and JSON syntax, and the edit distance over non-target text. When the rules cannot determine the scope, keeping the original text and asking for clarification should count as a permitted action, not a guessed edit that happens to pass.

#### 7.5.2.3 Exact-Copy Errors: From old_string Mismatch to Layer-by-Layer Localization

An old_string failure cannot be attributed simply to "the model copied it wrong." For the same string, store the raw byte hash, the Unicode code point sequence, and the tokenizer token ID sequence, then look for the first divergence along this chain:
original file bytes → tool return → Harness serialization → model context → model token output → decoded string → JSON/tool-call parsing → tool matching

A minimal set of evaluation probes covers: direct restatement, extraction from a long context, placement into tool arguments, selection among similar strings, and spaces, newlines, backslashes, Unicode combining characters, and low-frequency tokens. The metrics: byte-exact match, code-point-exact match, token-exact match, the position of the first divergence, and the real tool success rate. If the model is correct on the direct probe but the tool call still fails, fix the tokenizer, the serialization, the Harness, or the tool protocol; only when the first divergence appears in the model's own output should the case be turned into the copying training data of Chapter 8.

### 7.5.3 End-to-End and Trajectory-Prefix Regression Tasks

Once the first error is known, turn the repair target into a repeatable regression task:
- **End-to-end regression**: starts from the initial state and the user request, lets the Agent complete the whole task, checks the final state, the required output, and the safety conditions. Comes closest to the production result, but makes it hard to tell at which step the failure occurred. As a rule, end-to-end regression verifies that the Agent's capability in each domain still meets expectations. The standard benchmarks (OSWorld, AndroidWorld, τ-bench) are all end-to-end regression tasks.
- **Trajectory-prefix regression**: freezes the existing context, dialogue, tool returns, and environment state just before the first error, then asks the Agent only to think and take the next observable action or few actions. Costs less and isolates a single policy or tool problem. For a production Agent needing high reliability, building the trajectory-prefix regression set often matters more than the end-to-end one — and it requires patiently building the failure taxonomy and attribution system of the previous section.

The answer to a trajectory-prefix regression task should be defined as an **acceptable action set** rather than a single canonical action/answer — it may require "read the repository rules first," "ask the user first," or "refuse the dangerous operation," while also listing the prohibited actions.

Once failure attribution is done, an evaluation dataset of both task types can be constructed. For a Coding Agent:
- Missing process → end-to-end regression task carrying a plan document and test acceptance conditions.
- Tool-call error → failing prefix truncated and edited into a boundary task testing whether the model can fix the format, escape special characters, or switch to a suitable tool.
- Abnormal termination → add recovery scenarios for truncation, timeout, and tool failure.
- Completion and logic errors → add multi-goal checklists, reminders of remaining work, and the "not yet proven impossible" boundary.
- Requirement-understanding and ambiguity cases → freeze tasks with several reasonable readings into prefixes and put "clarify first" in the acceptable-action set.
- Symptom-fix and faked-verification cases → add two hard constraints to acceptance: "test assertions may not be modified" and "a completion claim must carry the output of a command that really ran."
- Information-reporting cases → assert on the content of the reply itself, not only on the environment state.

The evaluation dataset is the foundation for the post-training of Chapter 8 and the self-evolution of Chapter 9.

### Experiment 7-7 ★★: Trajectory-Prefix Boundary Evaluation with Multiple Encodings

Supplies the Agent with known user memory, the current instruction, a trajectory prefix, tool returns, and environment state, then asks for only the next observable action. Covers production bad cases such as scope conflicts, stale preferences overriding current instructions, low-confidence inferences, confirmation before high-risk deletion, and preview before external publication. The same cases are encoded as JSON Cards, Markdown, and Python-like memory; deterministic checks score the allowed decision category, safety, required evidence, and forbidden actions.

Result: with GPT-5.6-sol through OpenRouter, all 33 cells (11 cases × 3 encodings) completed without API errors. Each encoding passed 6/11 cases, but their failure locations differed, showing that changing the representation alone does not repair application policy.

### 7.5.4 Pairwise Comparison and Model Ranking

In practical model selection, we often face "Which is better, A or B?" Pairwise comparison provides an evaluation method that does not rely on absolute scores. (Figure 7-5: Elo Rating and Pairwise Comparison Ranking)

Chatbot Arena anonymous battle example: Model A (Anonymous) "Refund processed, will arrive in 3-5 business days to your account" vs Model B (Anonymous) "Okay, refund will be processed." User blind pick → A is better.

Elo Update Formula: Expected win rate E_A = 1/(1+10^((R_B−R_A)/400)); Update R_A' = R_A + K*(1−E_A).

**Elo Rating** (ranking system originally designed for chess) quantifies the relative ability of models through a large number of pairwise matchups: the larger the rating difference, the higher the expected win rate for the stronger model. Example: Model A rating 1200, Model B rating 1000 → Elo predicts A's win rate ≈ 76%. If B unexpectedly wins, B gains more points and A loses more — an upset triggers a larger correction, letting rankings converge quickly on true ability. Statistical foundation is the **Bradley-Terry model**: each model is abstracted as a latent "strength score," and the probability of one beating another is determined by the difference between their scores. Elo is the engineering implementation of this model in online-update form.

**Chatbot Arena** uses anonymous random matchups — users blindly choose the better response without knowing the model's identity; rankings derived from millions of votes. Advantage: no "absolute standard" needed; only human judgment on "which is better, A or B." Limitation: rankings depend on what users happen to ask — if a flood of programming questions comes in, models strong at programming rank higher, which may say little about other tasks.

Live leaderboard (example): 1 Claude 4 Opus 1352; 2 GPT-5 1318; 3 Gemini 2.5 Pro 1290 (45% win rate vs #2); 4 DeepSeek R2 1265 (42%); 5 Qwen 3 Max 1230 (39%); 6 Llama 4 405B 1198 (36%).

When pairwise judging is performed by an LLM rather than human voting, guard against **Position Bias** — the judging model systematically favors the candidate in a certain position (usually the first), and the judgment may remain unchanged even if the two candidates are completely swapped. Standard mitigation: evaluate each pair twice with swapped order (once A first, once B first) and average the two results; a stricter approach only counts cases where the two judgments are consistent, treating inconsistencies as ties or sending them for human review. Chatbot Arena's approach is essentially the same — randomizing the display positions of the two responses so position bias cancels out over a large sample.

GRPO introduces pairwise comparison into RL training: a set of candidate responses → normalize relative advantage → policy update (bypassing an explicit reward model).

### Experiment 7-8 ★★: Building a Model Leaderboard from Pairwise Comparison Data

Understand how the Bradley-Terry model extracts relative ability scores from many pairwise comparisons by implementing an Elo rating calculation system from scratch, using the real open-source Chatbot Arena voting dataset (millions of anonymous user blind votes).

Part 1: Implement the Elo rating iterative update algorithm — initialize all models at 1000; process voting records in chronological order; for each matchup calculate the expected win rate from the current rating difference, compare actual result with expectation, adjust ratings by a fixed learning rate — winner gains points, loser loses points, adjustment magnitude proportional to deviation from expectation (an upset loss results in a larger rating change). Sort models descending by final rating and calculate the pairwise win rate matrix. Compare with the official leaderboard to verify general consistency. Exact point-for-point alignment not required: the official Chatbot Arena uses Bradley-Terry maximum likelihood estimation (solving all matchups simultaneously, independent of voting order), while this implementation uses online incremental Elo updates (results affected by the K-factor and processing order). The two algorithms yield consistent overall rankings, but specific scores won't be precisely identical.

Part 2: Create a historical ranking evolution animation — slice voting data by time (weekly or monthly), calculate Elo snapshots per time point; use D3.js to implement a bar chart race animation (horizontal bar length = rating, vertical position = ranking, smoothly changing over time). By observing the animation, identify technology breakthrough moments (a model's rating suddenly surges), competitive landscape evolution, and model lifecycles.

## 7.6 Evaluation-Driven Model Selection

Model selection is not simply "choosing the strongest model"; it involves evaluation-driven trade-offs across multiple dimensions based on the application scenario.

### 7.6.1 Key Dimensions for Selection

**Throughput and Latency** — two metric families easily confused; untangling them takes one fact: LLM inference runs in two stages.
- **Prefill** reads the entire context at once and determines the **Time To First Token (TTFT)**: the delay between user pressing Enter and the first character appearing. The longer the context, the slower prefill, the higher TTFT.
- **Decode** generates the response token by token, setting generation speed (tokens/second) — which also dictates thinking time: at 50 tokens/s, a model producing 2000 thinking tokens spends 40 seconds just thinking.

Main throughput/latency metrics:
- **Input Throughput / Output Throughput**: correspond to Prefill and Decode speed respectively.
- **TTFT**: equals queuing time plus Prefill time; the user-perceived "responsiveness."
- **Thinking Latency**: thinking tokens can vary severalfold across models, and thinking length is not necessarily positively correlated with task effectiveness — measure each model's thinking token usage and the corresponding benefit on your own workload, rather than inferring from public leaderboards.
- **p95 Tail Latency**: the latency 95% of requests will not exceed — a better indicator of real user experience than the average, which can be pulled down by many fast requests, masking severe slowdowns experienced by a minority.

**Cost**: pricing for input/output/cache tokens. Cost should not be evaluated in isolation — a cheap model with a low success rate may incur higher costs from frequent retries. Calculate average cost per task and the cost-performance ratio.

**Performance**: Pass@1, Pass^k, Pass@k, Best@k defined earlier. In model selection: daily scenarios → focus on Pass@1 (single-attempt average success rate); critical operations → prioritize Pass^k (stability of "never making a mistake"); exploratory tasks → prioritize Pass@k or Best@k (upper bound given enough opportunities); open-ended tasks → multi-dimensional Rubric scoring.

**Rate Limits and Reliability**: RPM (Requests Per Minute) / TPM (Tokens Per Minute) limits affect concurrency; some APIs dynamically adjust quotas during peak hours. Robustness: watch out-of-distribution data, adversarial inputs, long-running stability (mode collapse or attention drift).

**Budget–capability curves**: a single score at a fixed budget is not enough to determine whether an Agent can handle long-horizon work. Report how performance changes with wall-clock time, tokens, tool calls, or compute budget. **RE-Bench** makes it concrete (Wijk et al., RE-Bench: Evaluating Frontier AI R&D Capabilities of Language Model Agents against Human Experts, arXiv:2411.15114, 2025): with a total budget of two hours per environment, the best Agent scored about four times as high as human experts; humans benefited more from additional time, narrowly surpassed the best Agent at eight hours, and scored about twice as high with multiple attempts given 32 total hours. Short-budget leadership therefore cannot be extrapolated directly to long-running capability. Model selection should compare several budget points close to the real workload's duration.

**Mixing models**: lightweight models on simple requests to cut costs, powerful models on complex tasks to protect quality; or specialist models on particular sub-tasks (image understanding, code generation) collaborating through sub-agent mechanisms. Any heterogeneous combination must itself be validated by evaluation, to confirm the overall benefit outweighs the added system complexity (e.g., treating questions like "which is larger, 9.9 or 9.11?" or "I want to wash the car; the car wash is 50 meters from home — should I walk or drive?" as simple and handing them to a lightweight model, leading to wrong decisions).

### 7.6.2 Model Behavior: When to Stop Reading and Start Editing

Model selection compares not only whether a model can finish a task, but also how it behaves by default. One readily observable difference in Coding Agents is the **action threshold**. Given the same coding task, some models explore the repository broadly and confirm the architecture, callers, and tests before editing; others localize from less evidence, edit early, and use test feedback to complete their understanding. The former assigns a higher cost to premature edits; the latter assigns a higher opportunity cost to reading one more file.

This tendency has two sources: the system prompt in the Harness, and the model's behavioral policy. **Post-training is a key source of that behavioral policy**: SFT trajectories demonstrate "how much to read before acting," process rewards reward or penalize particular tool paths, and outcome rewards reinforce the entire policy that ended in success. Over time, the model learns not only how to write code, but also engineering habits.

### Experiment 7-9 ★★: Measuring Model Action Thresholds in a Fixed Coding Harness

Objective: isolate the model factor, quantify how Coding models trade off continued information gathering against starting to edit, and evaluate path efficiency together with outcome quality.

Method: run chapter6/model-action-threshold/experiment.py. By default it calls GPT-5.6-sol and Claude Sonnet 5 through the same OpenRouter OpenAI-compatible endpoint while fixing the system prompt, tool schemas, task repositories, test commands, and turn limit. The neutral prompt specifies neither a minimum number of files to read nor a requirement to edit quickly. Repeat each of the three task categories at least three times and alternate model order. Record tool calls, files read, searches, and wall-clock time before the first edit, along with first-tested-patch acceptance, post-test rework, final success, changed files, and token usage.

Causal interpretation: the neutral campaign asks whether behavior changes with the model inside one harness. To measure the harness as a modifier, run a separate campaign with --policy explore-first; do not mix the two policies in one model comparison. Behavior that changes with a model swap and persists for the same model across harnesses is stronger evidence of a model effect; the reverse is stronger evidence of a harness effect.

Acceptance criteria: all offline unit tests pass; every task fixture is first confirmed to fail its tests; the formal result contains every model × task × trial cell, zero API errors, an independent final test, and auditable trajectories; manifest.json verifies hashes of configuration, observations, and summary. The project directory includes one complete 18/18-cell run. Readers should rerun on the model versions and real workloads they care about rather than treating these miniature-repository numbers as a permanent leaderboard.

### 7.6.3 Cost Analysis of Agent Systems

Agent costs are far more complex than simple token pricing — multi-turn reasoning, tool calls, and context accumulation make costs grow non-linearly. Systematic cost analysis is indispensable to the evaluation system and a prerequisite for production deployment.

**Components of Cost** — three levels:
1. **Model inference cost** (most direct): determined by input/output token consumption. Two often-overlooked amplifying factors:
   - **Context accumulation effect**: each LLM call sends all previous conversation history and tool outputs together. Without effectively utilizing KV Cache (caching already processed context to avoid redundant computation), cost grows very quickly — Round 1 sends 1000 tokens, Round 2 sends 2000, Round 3 sends 3000, totaling 1000+2000+3000=6000 instead of 3×1000=3000. The more rounds, the larger the gap.
   - **Thinking token cost**: models that support thinking generate many thinking tokens; though not displayed to the user, they are still billed.
2. **Tool call cost**: external API fees (search engines charge per query, database queries consume computing resources), sandbox resources for code execution, and an easily overlooked indirect cost — the token cost when tool outputs are injected into the context. A single web search might occupy 2000-5000 tokens, repeatedly billed as input in every subsequent inference round.
3. **Infrastructure cost**: operational overhead for vector databases (for RAG retrieval), message queues, relational databases, and logging and tracing storage (for observability).

**Measured cost example** (companion experiment): a fixed eight-turn refund workflow — query the order, logistics, refund policy, and knowledge base, then perform risk checks, issue the refund, notify the user, and close the case. Real gpt-4o-mini calls ran under all four combinations of two switches: stable vs unstable prefixes, and full vs compressed history. Business workflow identical in every arm.

**Table 7-4 Measured Cost of the Eight-Turn Agent Workflow:**

| Configuration | Input Tokens | Cached Tokens | Total Cost | Savings vs. Baseline |
|---|---|---|---|---|
| No cache, no compression (baseline) | 20,700 | 0 | $0.003776 | — |
| Stable prefix only | 20,386 | 13,568 | $0.002707 | 28.3% |
| History compression only | 16,177 | 0 | $0.003115 | 17.5% |
| Stable prefix + compression | 16,035 | 6,144 | $0.002643 | 30.0% |

In the baseline, input grew from 1,113 tokens on the first turn to 3,668 on the last. Tool results were repeatedly carried into later requests, accounting for 9,544 input tokens across the run. With both optimizations, that figure fell to 5,248 and total cost dropped by 30%.

**The gains were not additive**: stable prefix alone saved 28.3%, compression alone saved 17.5%, yet together they saved 30%, not 45.8%. Compressing history also shortened the prefix available for cache reuse. When context optimizations are combined, measure the complete workflow; never add their isolated savings together. A different model, price schedule, or task length will change the 30% figure. The reusable result is the four-arm method, not the percentage itself.

**Cost Optimization Strategies**:
- Input-side levers to test first: **KV Cache Reuse** (keep the prefix stable), **Context Compression** (shorten old trajectories and verbose tool results), **Tiered Model Routing** (send simple requests to lightweight models, difficult reasoning to stronger ones). Chapter 2 covered implementations. Operational point: each lever should have its own switch, so the team can measure both its isolated effect and its behavior when combined with others.
- **Asynchronous Batch Processing**: accumulates non-real-time tasks for batch processing, leveraging batch pricing discounts from API providers; in self-deployment scenarios, improves GPU utilization during off-peak hours.
- **Cost Monitoring and Budget Control**: in production, a real-time cost monitoring system should track token consumption and API costs by task type, model, user, etc. Set a cost cap per task — automatically terminate the Agent when it falls into a loop or explores too deeply, preventing a single task from incurring abnormally high costs.

### Experiment 7-10 ★: End-to-End Cost Analysis of Agent Tasks

Experiment Goal: reproduce the eight-turn cost breakdown above, then test the same optimization levers on your own workload.

Technical Approach: reproduce the fixed companion task first, then select several representative tasks of your own. Use LangSmith or a self-built tracing system to record input/output and thinking tokens, tool-call counts and return sizes, and end-to-end latency for every LLM call. Calculate average cost, p50/p95/p99, and the cost breakdown for each task type.

Acceptance Criteria: generate a cost report and identify the main drivers. Run all four switch combinations, measuring each optimization alone and both together. Rerun the experiment after changing models rather than carrying forward the saved trace's percentage savings.

### 7.6.4 Evaluation-Driven Continuous Iteration

Model selection is not a one-time decision but a continuous process, adjusted as models evolve.

Concrete model-switching case: suppose the Agent system is built on Claude, excelling in tool calling and complex orchestration. Gemini releases a new model; public benchmarks show it surpasses Claude on several metrics at a lower price. The question isn't "Is Gemini better than Claude?" but "On my specific tasks, is Gemini better than Claude? How much better? What is the switching cost?"

A team with a solid evaluation system answers this in hours: run the new model on its own evaluation dataset and compare task success rate, tool call accuracy, latency, and cost. You might find the new model is better and cheaper on simple tasks — but in core scenarios involving complex multi-round tool orchestration, its success rate drops by 5%. Once you confirm the difference exceeds the estimated sampling noise (see "Statistical Significance"), the decision becomes a differentiated strategy — migrate simple tasks to the new model to cut costs, keep the original model on complex tasks to protect quality — rather than a blind wholesale switch. Decisions this granular and data-driven are only possible with an evaluation system built in advance.

### Experiment 7-11 ★★: Multi-Dimensional Model Performance Benchmarking

Conduct a comprehensive benchmark of mainstream LLMs and different API providers to build a multi-dimensional model selection decision database.

Select test scope: closed-source SOTA models (GPT series, Claude series, Gemini series, Doubao series) and open-source models (Qwen, Kimi, DeepSeek). Test the same model with different API providers (e.g., DeepSeek official vs Siliconflow) to verify results from third-party performance monitoring platforms (e.g., Artificial Analysis).

Design standardized test workloads: input throughput tests use fixed-length contexts (8K/32K/128K tokens); output throughput tests request fixed-length responses (512/2048 tokens). Latency tests include TTFT (Time to First Token) and end-to-end latency. For thinking-supporting models, separately measure thinking length and thinking latency. For each configuration, make at least 100 requests and calculate standard deviation, p50, p95, and p99; high latency variance indicates an unstable user experience.

Evaluate API availability and stability: probe once per hour for a week, recording success rate, error types, and failure duration. Calculate failure rate, MTTR (Mean Time to Recovery), and longest continuous uptime. Test actual rate-limit thresholds — gradually increase concurrency to find the throttling point, recording RPM/TPM limits. Calculate comprehensive cost: collect pricing information (unit prices for input/output/cache tokens), consider the impact of KV Cache, and calculate average cost for typical multi-round Agent tasks.

### Experiment 7-12 ★★: End-to-End Selection Evaluation of User Memory Systems

Prerequisites: complete the contextual retrieval or agentic RAG experiment from Chapter 3.

Goal: perform an end-to-end model-selection evaluation of a user-memory retrieval Agent, examining how the embedding model, reranker, and Agent's main model jointly affect retrieval quality, latency, and cost. Reuse chapter3/contextual-retrieval-for-user-memory or chapter3/agentic-rag-for-user-memory and compare configurations on 60 test cases.

Acceptance: evaluate each of the three selection points in turn — embedding model (BGE-M3 / OpenAI / Doubao, etc., recording top-5 retrieval accuracy, latency, cost), reranker (include a "no reranker" baseline, quantify its marginal value), and main model (compare success rate and tool usage efficiency under the same retrieval configuration). The key is to identify synergies among components: a stronger embedding might make the reranker redundant, and a stronger main model might compensate for retrieval shortcomings. Selection is a systemic trade-off, not simply choosing the strongest component in isolation.

## 7.7 Statistical Significance of Evaluation Results

The evaluation set is finite and model outputs are stochastic, so a score difference may be nothing but sampling noise. If you measure a success rate p over n cases, the standard error can be roughly estimated as:

SE(p) ≈ sqrt(p(1−p)/n)

For example, with 100 cases and a 70% success rate, the 95% confidence interval is about 70% ± 9 percentage points; "the new model gets 73% versus the old model's 70%" is not enough to justify switching.

When comparing two configurations on the same batch of tasks, prefer **paired analysis**: record per task which one wins, and judge the difference with **McNemar's test** or a **paired bootstrap**, rather than subtracting two independent success rates. Because each Agent run may also differ, run each configuration with several random seeds (say 3–5) and report the mean along with the spread; a single run is only good for screening a direction. If the expected gain is only 2–3 percentage points and the evaluation set has only a few dozen tasks, enlarge the sample first — the standard error shrinks as 1/sqrt(n).

```
for task in paired_tasks:
  for seed in fixed_seeds:
    a = run(config_a, task, seed)
    b = run(config_b, task, seed)
    record_paired_delta(verifier(a), verifier(b))
return paired_bootstrap_or_mcnemar(all_deltas)
```

Pairing means both groups share the same tasks and random conditions, not that you draw two separate samples and compare their averages.

When validating several hypotheses in parallel, account for **multiple comparisons**: tighten the significance threshold, or re-run positive results independently. The practical criterion is simple: a score gap is worth acting on — switching models or shipping a change — only if it exceeds the noise, holds up under paired analysis, and can be reproduced.

## 7.8 Agent Observability

Evaluation-driven decisions (model selection, continuous iteration) rely on high-quality operational data. (Figure 7-6: Observability Technology Stack)

Execution trace tree (single task):
- Trace: "Check Beijing weather for user tomorrow" (3.2s, $0.008)
- LLM Call: Intent recognition — Claude 4 Sonnet · 0.4s · 320 tokens
- Tool: get_weather(Beijing) — MCP Server · 1.8s · 200ms TTFT
  - HTTP Request: api.weather.com · 1.6s
  - Response Parse: JSON → Structured weather data
- LLM Call: Generate response — Claude 4 Sonnet · 0.8s · 580 tokens
- Tool: send_message(user) — 0.2s · Contains weather summary

Monitoring dashboard:
- Cost tracking: Today $12.30 (1,200 calls); This month $340 (34K calls); Anomaly: task#892 looped search 14 times, cost $2.1.
- Performance monitoring: P50 / P95 / P99 latency 2.1s / 8.4s / 15.2s; Tool success rate 94.3%.
- Quality audit: Task success rate 87%; Hallucination trigger 2.1%; Security violations 0 (this month); User satisfaction 4.3/5.
- Closed loop: Trace data → Identify issues → A/B testing → Prompt version management → Continuous optimization.

Observability is borrowed from distributed systems: you cannot open the system and watch it work; you infer what happens from the logs, metrics, and traces it emits — the way a doctor, unable to see inside a patient, diagnoses from temperature, blood pressure, and imaging. Agent systems make this harder still: the same input can produce different outputs, multi-round reasoning and tool calls make execution paths extremely complex, and the model's "thinking" is completely opaque from outside.

Value of observability: (1) problem diagnosis — complete traces let developers replay the entire process rather than guess; (2) foundation for continuous optimization — see which tasks require multiple rounds of iteration, which tools have the lowest success rate, which retrieval queries always return empty; (3) cost management — agent operating costs can differ by one or two orders of magnitude between tasks, and tracing surfaces the abnormally expensive cases; (4) accumulated trace data underpins later optimization and model improvement.

Agent observability is built on the foundation of **traces**, whose data structure inherits the **span tree** model from distributed systems: one task execution = one trace, where each LLM call, tool call, and retrieval is a span (an execution unit recording input/output, start/end times, token consumption, and error information). Parent-child relationships between spans form an execution tree — e.g., an "Agent Main Loop" span may have several "LLM Call" and "Tool Call" child spans hanging beneath it. Standardized protocols are available: **OpenTelemetry** is the general-purpose distributed tracing standard; **OpenInference**-type specifications define LLM-specific semantic conventions on top of it (how to record prompts, model parameters, token usage). Adopting standard protocols decouples collection and analysis — the same trace data connects to different analysis backends, avoiding vendor lock-in.

**LangSmith** is one representative platform (similar: Langfuse, Arize Phoenix, etc.), integrating observability, evaluation, and optimization into a closed loop. Each execution creates a trace session; model calls, tool usage, and knowledge retrieval are recorded as independent execution units linked by causal relationships to form an execution tree. Each unit records complete input/output, timing, cost, and error information. The platform uses asynchronous batch data collection so tracing itself doesn't affect the Agent's response latency. It also supports A/B testing (routing a portion of user traffic to a new version, automatically comparing metrics, rapid rollback or gradual scaling), prompt version management (each version associated with runtime performance data), and collaborative development (team members share trace data and problem cases). The massive real-world production data is a goldmine for continuous improvement — it uncovers unforeseen scenarios and identifies the features most in need of optimization.

The most valuable use of observability data is to turn it into **evaluation assets**. A practical loop: extract failed and suspicious cases from production traces → anonymize them (strip sensitive fields such as user data and keys) → distill them into new test cases and regression tests for the evaluation set. The evaluation set then stops being a one-time static collection and becomes a living asset that evolves with the product and continues to reflect the real user distribution — the failure patterns exposed in production today become the regression tests guarding the baseline tomorrow. This is the interface between observability and the chapter's main theme: observability is responsible for "seeing" what happens in the real world, and evaluation is responsible for solidifying those observations into repeatable standards.

## 7.9 From Benchmark Reports to System Improvements

The following case comes from a real, deliberately narrow AndroidWorld iteration in the companion repository. It covers four Wi-Fi settings tasks on an API 35 emulator, with one matched run per task. It is not the full 116-task benchmark and does not replace a rerun in the reference API 33 environment. Its value is not an overall score; it is the sequence of decisions from one result to the next. (Figure 7-7: Benchmark to Improvement Loop)

① **Observation: Diagnostic Report.** Overall Success Rate: 88% (102/116). transcription: 0%; complex_ui: 17%; math_counting: 0%; Wi-Fi Operation: 0%. Task 82, 102-115 Concentrated Failures.

② **Hypothesis: Three-Layer Improvement Framework.** Surface Layer: H1 Set Navigation Prompt, H2 UI Rules. Middle Layer: H3 Fix Multimodal Pipeline, H4 Thinking. Deep Layer: H5 GPT-5, H6 UI Element Tree.

③ **Experiment: Phased Validation (5 runs × 116 tasks per configuration).**
- H1 Navigation: Setup 0%→75%, token +8%.
- H3 Multimodal: Transcription 0%→80%, Latency +1s.
- H4 Thinking: Counting 0%→70%, Latency 3x!
- H6 Element Tree: UI 17%→52%, token +30%.

④ **Decision: Cost-Benefit Trade-off.**
- H1+H3: Low Cost, High Benefit → Deploy.
- H4: Only 8% tasks benefit but 3x Latency → Reject.
- H6: 35% Improvement / 30% Cost → Deploy.
- H5: 15s/Step Unacceptable → Alternative.

⑤ **Iteration: New Cycle.** Deploy H1+H3+H6 → 88%→94%. New report shows different failure modes: H7 Conditional Thinking Enabled; H8 Expand gesture action space. Loop continues.

Methodology: Observe → Hypothesize → Experiment → Decide → Iterate = from alchemy to scientific engineering.

From the Harness-engineering perspective, this section is essentially methodology for iterative Harness optimization — using evaluation data to identify weak points in the Harness (insufficient context? missing constraints? inadequate validation? untimely feedback?), making targeted improvements, then re-evaluating, forming a closed loop for the Harness's continuous evolution.

Before analyzing any benchmark report, note an easily overlooked principle: **when Agent performance drops, check the evaluation system first, then the Agent.** The common mistake is to start editing Agent code the moment a score falls, ignoring the possibility that the evaluation system broke first — steer by a distorted signal and the correction is wrong from the first step. Typical evaluation-side failures: the runtime environment running out of resources and killing processes (shows up as random failures), bugs in the scorer that mark correct answers as failures, and test cases drifting out of sync with production scenarios. In the headline numbers, all of these look identical to model degradation; only a review of the full traces can tell them apart.

### 7.9.1 Reading a Benchmark Report: The Art of Problem Discovery

The starting report recorded one run on each of 116 tasks and about 88% overall success. The failures were not scattered: three of the four SystemWifiTurn* tasks failed, and their traces repeatedly navigated back and forth without confirming the final state. Two explanations fit: the Agent did not know where to go, or the UI representation it received was incomplete.

An 88% headline score hides this small but coherent failure cluster. Raising the step limit would be equally misleading — it could recast "the Agent cannot see the control" as "the Agent needs more persistence." Read reports in the opposite direction: locate clusters by task and capability tag, replay the traces, decide whether the failure arose in observation, reasoning, action, or verification, and only then choose a variable to change. The Wi-Fi slice was used to diagnose the mechanism cheaply, not to estimate system-wide performance.

### 7.9.2 From Data to Hypotheses: Building an Improvement Roadmap

The first round tested the cheapest explanation. **H1** assumed a navigation-knowledge gap, so only the treatment received Wi-Fi navigation and final-state-checking instructions. Success did not improve; the prompt was not the bottleneck.

The second round asked what the Agent could actually see. **H5** replaced the API-35-incompatible accessibility feed with AndroidWorld's supported UIAutomator tree. Success improved, but the full tree caused token use to surge. **H5C** therefore added no new information: it simply removed invisible, textless, non-actionable container nodes to see whether the same success could be preserved with less noise.

Across all three rounds, the model, task parameters, seed, step limit, and emulator stayed fixed, and arm order alternated. This staged design made attribution straightforward: the residual problem or side effect from one round became the sole change in the next.

### 7.9.3 From Results to Decisions: Data-Driven Trade-offs

**Table 7-5 Three Rounds on the AndroidWorld Wi-Fi Slice:**

| Experiment | Only Change | Control → Treatment Success | Treatment/Control Tokens | Next Step |
|---|---|---|---|---|
| H1 | Add navigation instructions | 25% → 25% | 0.47× | No success gain; retain the original prompt |
| H5 | Accessibility feed → UIAutomator | 25% → 100% | 2.498× | Strong gain but too expensive; continue optimizing |
| H5C | Compact the UIAutomator tree | 100% → 100% | 0.506× | Preserve success and halve tokens; advance to a full rerun |

With only four tasks per arm, these numbers can decide whether a larger rerun is worthwhile; they cannot estimate success across AndroidWorld.

The sequence matters more than any one percentage. More detailed instructions cannot restore information the Agent never received; observation failures should be investigated before prompts are expanded. But more input is not always better either — the full element tree fixed visibility while flooding the context with noise. Removing non-semantic nodes preserved four successful runs and cut tokens by roughly half. No model was changed: the Harness's UI representation first determined whether the task could be completed and then whether completing it was economical.

### 7.9.4 Continuous Iteration: From First Improvement to System Evolution

Passing H5C on four tasks only earns it a larger test; it does not authorize deployment. The next gate is a **five-seed run over all 116 tasks** in the Pixel 6 / API 33 reference environment with the full third-party app set. Success must be non-inferior, token use no more than 75% of the original, and latency no more than 1.5×. Until that run is complete, 4/4 on the slice must not be reported as 100% system-wide success.

That is what continuous iteration means in practice: evidence from one round should authorize only the next action its scope can support. H1 stopped further prompt piling; H5 found the right mechanism and revealed a cost problem; H5C fixed that problem and qualified for broader testing. A good benchmark report contains more than a score — it states where the conclusion applies, which guardrails failed, and what must be tested next.

### Experiment 7-13 ★★★: Evaluation and Improvement on AndroidWorld

Practices the full path from evaluation report to system improvement. Start with the historical report and three saved paired runs in chapter6/android-world.
- **Step 1: Diagnosis.** Cross-analyze the per-task table and the capability tag matrix to map surface-level task failures to deep-seated capability deficiencies. Identify capability tags with lower-than-expected success rates and task areas with concentrated failures.
- **Step 2: Build Hypotheses.** Formulate improvement hypotheses following the three-layer framework (surface → mid → deep). Each hypothesis states the target improvement in success rate and the verification method.
- **Step 3: Phased Experimentation.** Reproduce H1, H5, and H5C with one variable changed per round. Record tokens, latency, and regressions as well as success.
- **Step 4: Data-Driven Decision Making.** Make deployment decisions based on cost-benefit analysis — not simply adopting all effective improvements, but weighing scope of application, latency impact, and cost overhead for each. Prioritize low-cost, high-benefit improvements for deployment; restrict high-cost improvements to critical scenarios.
- **Step 5: Iteration.** A passing slice experiment advances only to the full rerun. Discuss deployment only after the 116×5 reference-environment run, and preserve environment differences, sample size, and incomplete scope in the report.

## 7.10 From External Evaluation to Internal Evaluation: Evaluation Infrastructure for Production-Grade Agents

So far the chapter evaluated Agent systems from the outside — building an evaluation environment, designing datasets, analyzing benchmark reports. But the best Agent products do more than undergo external evaluation; they build continuous self-evaluation infrastructure into the product. Using the open-source general-purpose Agent OpenClaw (introduced in Chapter 5) as an example, plus public technical analyses of leading Coding Agent products and practitioner insights, this section presents an internal evaluation system worth emulating: one that systematically embeds the experimental methodology of ML research into product engineering.

### 7.10.1 Ablation Infrastructure: Understanding the True Contribution of Each Feature

ML researchers have long used ablation studies to learn which model components matter — ablation means "removing" one component at a time and observing how much overall performance drops. OpenClaw brings this into product engineering: a built-in **master switch** can disable several major features at once (thinking mode, context compression, automatic memory, background tasks, and more), creating a "bare model" baseline. That lets the team answer: does a feature truly improve the user experience, or does it just feel useful?

Making ablation a routine engineering practice rather than a one-time research activity has several implications:
- The ablation switch must be injected very early in the startup path — before any module-level constant captures configuration values — meaning the ablation infrastructure must be designed into the architecture from the start, not retrofitted later.
- Running ablation experiments regularly (e.g., before each major release) can uncover "feature debt" — features once effective but no longer necessary as models evolve.

Recommended practice: every major feature should be independently disableable, and the team should regularly verify the actual contribution of each feature.

### 7.10.2 A/B Testing Methodology: Distinguishing Mechanism from Goal

Mature Agent products conduct rigorous A/B testing on their own behavior (randomly dividing users into two groups, one using the old version and one the new, comparing actual data to determine if a change is effective). A well-designed Agent A/B test case illustrates several key methodological principles:
- **Multiple variants, not just a binary comparison.** Instead of only comparing "with" and "without," design multiple progressive variants (e.g., when testing different strengths of prompt constraints, set up a control group and three experimental groups with progressively stricter constraints). This reveals dose-response relationships and finds the optimal point.
- **Distinguishing mechanism metrics from target metrics.** The easiest mistake — treating what you are changing as the optimization target. Example: testing "shortening the Agent's plan file length" — plan length is a mechanism metric (what you directly change), but not the target; the real target might be "reducing session-level cost." Shortening the plan file may lower costs, but could also lead to more edit-check-edit loops due to insufficiently detailed plans, increasing total output. Always ask: is what I am changing (the mechanism) the same as what I truly care about (the target)? If not, prioritize the target.
- **Setting guardrail metrics.** Even if the target improves, the experiment should be stopped if user satisfaction declines, the number of operations increases, or the error rate rises. Guardrail metrics are non-negotiable thresholds that must not regress.
- **Recording baseline statistics.** Include sample size, distribution percentiles, and correlation analysis (e.g., "rejection rate increases monotonically with plan size") to provide context for interpreting results. Without a baseline you cannot determine statistical significance.

### 7.10.3 Two-Layer Feature Flag System

Agent products need a **Feature Flag infrastructure** designed from day one — a feature flag is a remotely controllable switch deciding whether a function is enabled/disabled for users, without requiring code redeployment. It serves three purposes simultaneously: experimentation, gradual rollout, and emergency circuit breaking.
- **Compile-time flags** physically remove the relevant code from the build artifact during the build phase. Internal-only features simply do not exist in external builds — even reverse engineering cannot discover the removed functionality. This also provides a clean ablation mechanism: disabling a feature does not skip logic at runtime; the corresponding code is physically absent.
- **Runtime flags** have configuration delivered by the server and cached locally on disk. The design prioritizes reading slightly stale cached configuration over blocking the Agent's startup while waiting for a network request. Specific grouping decisions are made through an experimentation platform (e.g., GrowthBook) for assigning A/B test groups. A key design detail: each feature's exposure event is logged at most once per session to avoid duplicate records polluting experimental data.

Lesson for Agent developers: feature flags are not debugging tools; they are first-class architectural components.

### 7.10.4 Prompt Sensitivity Assessment

The system prompt is the core "code" of Agent behavior, yet it often lacks the version control and regression testing afforded to regular code. OpenClaw's approach: a dedicated tool that can extract the fully rendered system prompt at a specified Git revision or commit — including the final text after all dynamic conditions are expanded. This precisely answers: which commit changed the prompt? What was the impact on the evaluation set?

Recommended practices for any Agent team: (1) the system prompt should be deterministically renderable (given the same configuration input, always produce the same output); (2) establish a versioned snapshot mechanism for prompts; (3) every prompt change should run regression tests on the evaluation set — just as code changes require CI.

### 7.10.5 Privacy-Aware Analytics as an Evaluation Foundation

Evaluation relies on good data, but Agent products often handle sensitive user content. OpenClaw resolves this contradiction through a **type system**: the analytics interface only accepts values wrapped in special types, where the type name itself serves as an audit trail — it explicitly declares "I have verified this is not code or a file path." This design transforms privacy constraints from documented specifications into compile-time enforced type checks.

Core principle: **design privacy constraints into the system from the start; do not bolt them on afterward.** If your analytics system cannot safely collect data, you cannot evaluate effectively. Privacy and evaluation are not opposing forces — privacy-aware design forces you to think carefully about what truly needs to be measured, which in turn fosters more precise evaluation metrics.

### 7.10.6 From External to Internal: A Shift in Evaluation Thinking

Core message: the previous sections taught how to evaluate an Agent externally; this section reveals how the best Agent products evaluate themselves internally. External evaluation tells you "how good the Agent is"; internal evaluation infrastructure tells you "which change made it better." Ablation experiments discover which features truly matter; A/B testing quantifies the impact of each change; feature flags provide the infrastructure for experimentation and rollback; prompt sensitivity assessment integrates the system prompt into the CI system; privacy-aware analytics ensures compliance in data collection. These five components together constitute **evaluation-driven product engineering** — not evaluating occasionally, but embedding evaluation into every product decision.

## 7.11 Simulation Environments: The Bridge from Evaluation to Post-Training

The endpoint of evaluation is not scoring, but improvement. Two improvement paths already demonstrated: adjusting the Harness (from benchmark reports to system improvements) and embedding evaluation into product engineering (internal evaluation infrastructure). The strongest form of improvement is training — when the goal expands from "evaluating existing capabilities" to "cultivating new capabilities," especially through the post-training techniques of Chapter 8, the evaluation environment must evolve into a **simulation environment**: a virtual playground where the Agent can repeatedly practice and be automatically scored.

Core differences between simulation and evaluation environments: much higher interaction frequency (millions vs. thousands), the need for randomization (to prevent memorizing specific configurations), and the requirement for immediate feedback. From an application perspective, simulation environments divide into two categories: **digital environments** (information processing tasks) and **embodied environments** (physical world perception and manipulation).

How the two ends of the bridge meet. Assets accumulated on the evaluation side convert almost seamlessly into training signals: a well-defined Rubric or validator is essentially a reward function for **Reinforcement Learning with Verifiable Rewards (RLVR)** — the scoring script becomes the reward script; whether a test passes or a state meets the standard serves both as an evaluation criterion and a reinforcement learning reward.

But training brings demands evaluation never had to worry about:
- **Reliable reset semantics**: training runs millions of episodes (an episode is one complete interaction round from an initial state to task completion); each episode must reset the environment to a deterministic, clean initial state; otherwise the gradient signal is contaminated by residual states from the previous episode.
- **Throughput far exceeding evaluation**: a few thousand evaluations are enough to draw conclusions, but training requires feeding the model millions of interactions within acceptable wall-clock time; the degree of environment parallelism and per-instance overhead directly determine whether training is feasible.

These two points — validators turned into reward functions, and training-grade reset and throughput — are elaborated in Chapter 8.

(Figure 7-8: Simulation Fidelity Spectrum)

| Approach | Scale/Characteristic |
|---|---|
| Mock API (unit test level) | million times/hour |
| AWorld (MCP sandbox) | 126 tool functions, 525s/round distributed |
| AndroidWorld (Simulator) | 116 real-world application tasks, UI Automator |
| Isaac Gym (GPU parallelism) | thousands of parallel instances, slightly compromised accuracy |
| RoboTwin2 (physics engine) | high-precision collision, single-instance CPU |
| real world | perfect fidelity, non-resettable |

Digital environment side — **AWorld** framework builds a controllable MCP server sandbox for GAIA tasks, providing 26 MCP servers covering 126 tool functions, avoiding the bans and uncontrollable side effects of directly accessing real APIs. All tool calls are replayable and auditable. AWorld's distributed architecture reduces traditional serial execution time from 7695 seconds to 525 seconds (a 14.6× speedup); the environment's stateless design makes each instance completely independent, supporting efficient parallelism.

Embodied environment side — **RoboTwin2** builds dual-arm manipulation tasks based on a physics engine, randomizing object positions, orientations, and appearances to improve generalization. The observation space includes multi-camera visuals and joint states, achieving real-time control through **Action Chunking** — the model plans multiple consecutive actions at once (detailed in Chapter 6). **OSWorld** provides reset capability through virtual machine snapshots; **AndroidWorld** focuses on mobile application automation. Whether digital or embodied, simulation environments also require the isolated execution environments and virtual identity mechanisms of Chapter 4 (VM/container isolation, residential proxies, Human-in-the-Loop authentication, shared file systems).

### Experiment 7-14 ★★: Configure the Embodied Intelligence Environment for OpenVLA and RoboTwin2

Set up a simulation environment for robot manipulation. Read ch7/SimpleVLA-RL and the OpenVLA documentation to understand the architecture of the Vision-Language-Action model (end-to-end integration of a vision encoder, language model, and action decoder, projecting images and text into a shared semantic space). Configure the RoboTwin2 environment, understanding the observation space (three-view RGB + 14-dimensional joint state) and action space (14-dimensional control vector). Study the environment randomization mechanism and spatial constraint logic in move_can_pot. Evaluate the pretrained model, recording success rate, completion time, and failure modes, with focus on the impact of the action chunking mechanism.

(Figure 7-9: OpenVLA and RoboTwin2 Embodied Intelligence Environment)
- Multimodal observation: Head camera 224×224 RGB; Left wrist camera 224×224 RGB; Right wrist camera 224×224 RGB; 14-dimensional joint state vector.
- Vision-Language-Action model (VLA): Vision encoder SigLIP → visual tokens; Language model Llama 2 7B backbone; Action decoder → 14-dimensional continuous control vector. Action chunking: generate 25 consecutive actions at once. Instruction: "Put the can into the pot."
- SAPIEN physics engine: dual-arm robot, 7 DOF each = 14-dimensional action. Environment randomization: position 60cm / orientation ±22.5°. Collision detection + physics simulation; rigid body / soft body / friction.
- Evaluation metrics: Success rate (can inside pot and not fallen); Completion time (25 steps × 25 actions = 625 control steps); Generalization ability (cross-position/orientation/appearance variant); Sim-to-Real (domain randomization → real migration).

### 7.11.1 Fidelity Trade-offs and Domain Randomization

High-fidelity environments support better transfer to the real world but have high computational costs. Another dimension of fidelity is the degree of randomization: moderate randomization improves generalization, while excessive randomization can make tasks too difficult. **Domain Randomization** is a key technique for narrowing the sim-to-real gap: introducing a wide range of random variations in physical parameters, visual appearance, sensor noise, etc. — like practicing grasping under various lighting and angles so you won't fail in the real world just because the light changes. In digital environments, sim-to-real manifests as differences in interface rendering, response times, etc., which can be mitigated by introducing randomization in latency and failures.

## 7.12 Chapter Summary

This chapter revolved around one core question: how do you tell whether an Agent has gotten better or worse? From defining success criteria and distinguishing Pass@k, Best@k, and Pass consecutive@k, to building reproducible test environments, designing leakage-resistant datasets, having an LLM serve as judge and producing auditable attributions for failed trajectories, and finally using evaluation results to drive model selection and iteration — every link in this chain affects how much you can trust the conclusion.

In the book's larger structure, this chapter builds the evidence segment of Chapter 1's discovery loop: failure attribution determines whether later proposals have anything solid to rest on.

Trajectory-prefix boundary evaluation makes a further point: obtaining a piece of information and correctly applying it to the current decision are two different capabilities. End-to-end regression guarantees basic tasks do not degrade, while the trajectory-prefix boundary set directly checks scope judgment, current-instruction override, clarification, and confirmation before dangerous actions. User memory is just one case of this general method. Evaluation for production-grade Agents is not an occasional exam, but a verification system that continuously generates regression tasks and boundary tasks from real problem cases.

Core methodology: Observe → Hypothesize → Experiment → Validate → New Understanding → New Hypothesis, transforming Agent engineering from experience-driven "alchemy" to data-driven scientific engineering.

The evaluation system forms a complete closed loop: Evaluation Environment provides automated testing infrastructure → Evaluation Dataset defines test cases → Automated Evaluation Methods (LLM-as-a-Judge and Rubric) score Agent performance → Benchmark Analysis reveals improvement directions → System Improvements fix issues → Update the evaluation environment and dataset, starting a new iteration cycle.

The evaluation system serves not only current-system optimization but also provides a critical foundation for the next two chapters: Chapter 8 turns evaluation environments and data into inputs for model post-training; Chapter 9 turns multidimensional evaluation of production trajectories into updates to knowledge, instructions, and procedures.

## Deep Thinking — Thought Questions

1. ★★ LLM-as-a-Judge uses a language model to evaluate the output of a language model. Does this "self-evaluation" have systematic blind spots — e.g., the model might consistently give high scores to a certain style of response, a preference inconsistent with human judgment? How can such biases be detected and corrected?
2. ★★★ The "leakage-proof" design of evaluation datasets is crucial. However, in the open-source ecosystem, once benchmark data is made public it is quickly incorporated into training data. Does this "cat-and-mouse game" have an endgame? Design an evaluation method that fundamentally resists data leakage.
3. ★★ Scale AI's four criteria (expert guidance, comprehensive coverage, standardized importance weighting, self-contained evaluation) aim to eliminate subjectivity in evaluation. However, certain task dimensions (e.g., "Is the answer helpful?" "Is the tone appropriate?") are inherently subjective. How can reliable Rubrics be designed for these subjective dimensions?
4. ★★ τ-bench evaluates Agents by simulating real user behavior. But the simulated user itself is an LLM — it might systematically underestimate certain edge cases (e.g., emotionally agitated or unclear users). How can the quality of the simulated user itself be validated?
5. ★★ Pairwise comparison (Bradley-Terry model) assumes preferences are transitive (if A > B and B > C, then A > C). However, human preferences often violate transitivity. In Agent evaluation, in what scenarios might non-transitive preferences appear? How does this affect the reliability of rankings?
6. ★★ This chapter distinguishes Pass@k as a ceiling on capability from Pass consecutive@k as a measure of business reliability. For an Agent whose single-run success rate is only 60%, how would you combine a task's failure cost, retry cost, and side effects to decide which metric to report and how large k should be?
7. ★★ This chapter proposes the scientific method "Observe → Hypothesize → Experiment → Validate." In practice the Agent's behavior space is vast, and validating a single hypothesis may require hundreds of evaluation runs. How can the information gained from evaluation be maximized under a limited computational budget?
8. ★ In the AndroidWorld pilot, the full element tree raised success from 25% to 100% but increased token use to 2.498× the control; pruning preserved 100% success while reducing token use to 0.506×. How would you design automatic pruning rules that remove semantically empty UI nodes without discarding information needed for accessibility, state verification, or later actions?
9. ★★ τ-bench's user simulation employs "progressive information disclosure" — not providing all information at once, but gradually revealing it based on the Agent's questions. How does this design affect evaluation results? If the simulated user's information disclosure strategy differs significantly from real users, are the evaluation conclusions still reliable?
