# Chapter 10 Multi-Agent Collaboration

The first nine chapters focused on a single Agent: building its context, knowledge, tools, and interaction capabilities, then using evaluation, post-training, and continual evolution to improve it over time. This chapter advances from "How do we build and improve one Agent?" to "How do we organize multiple Agents?" — so that division of labor, communication, and mutual verification can tackle tasks difficult for one Agent alone.

- OpenAI five-level scale of AI capabilities: Level 1 Conversationalists; Level 2 Reasoners; Level 3 Agents; Level 4 Innovators; Level 5 Organizations. Multi-agent collaboration is often presented as one path to Level 5.
- Here "Organizations" denotes a capability level — AI that can do the work of an entire organization — not an architectural requirement. A sufficiently powerful single Agent could in principle reach it too; today a single Agent remains constrained by its model's capabilities and context window.
- Fundamental point: the intelligence of a group can exceed that of any individual. Human civilization is proof — division of labor, collaboration, debate, and cross-generational knowledge accumulation give society intelligence beyond any single genius. Agent groups may show the same collective intelligence; even Agents as capable as a human expert, well organized, could surpass all human experts combined.
- Google DeepMind's *From AGI to ASI* (arXiv:2606.12683, 2026) lists "large-scale multi-agent collectives" as a key pathway to superintelligence (ASI) — collective intelligence of many AGI-level Agents may exceed the simple sum of its members.
- Multi-agent collaboration is not merely an engineering workaround for a single model's context window and capability limits — it may be a fundamental path from "expert-level AI" to "surpassing humanity as a whole."

## 10.1 A Classification Framework for Multi-Agent Collaboration

Building a multi-agent system starts with two core design dimensions, which together determine its basic architecture and implementation.

### 10.1.1 Dimension 1: Shared vs. Non-Shared Context

Most fundamental architectural decision — determines how information is passed between multiple Agents.

- **Shared context**: a subsequent Agent receives the complete conversation history and trajectory of the preceding Agent. When the system prompt and tool set change at each stage, the system treats the new stage as a different Agent (identity, responsibilities, capabilities changed), even though it retains all memory of its predecessor.
  - Example: after a requirements analyst writes a requirements document, the developer receives not only the document but the full record of communication between analyst and user; developer assumes a new role retaining all prior context.
  - Advantage: no information is lost; each Agent can review details from any previous stage.
  - Challenge: context can expand rapidly.
- **Non-shared context**: each Agent maintains an independent context and conversation history and cannot directly access others' work traces. Like departments working at their own desks, exchanging information via shared documents and meeting minutes.
  - Advantage: better modularity and isolation; each Agent focuses only on information relevant to its responsibilities; easier to extend and maintain — adding a new Agent requires only defining interfaces and data formats, not modifying existing Agents' internal logic.
  - Since Agents do not share context, information passes through explicit communication mechanisms.
- Classic distributed-systems answer: inter-process communication (IPC) has just two paradigms — **shared memory** (one side writes, the other reads the same block of storage) and **message passing** (data explicitly sent to the other side). Agent communication falls in the same two paradigms. Three common methods:
  - **Tool call parameters**: wrap the downstream Agent as a tool, pass structured data through its parameters; for well-typed, clearly structured data.
  - **Shared file system**: Agents exchange intermediate artifacts (documents, code, etc.) in a shared directory; for large artifacts or where persistence is needed.
  - **Message bus**: a dedicated intermediary passes messages between Agents; Agents send messages to the bus, which forwards them to target Agents (no direct calls).
- Mapping to IPC: shared file system = "shared memory"; tool call parameters and message bus = "message passing." Tool parameters are delivered synchronously with a call; bus messages are delivered asynchronously through an intermediary.
- Go maxim: "Do not communicate by sharing memory; instead, share memory by communicating."

**Figure 10-1: Shared Context vs. Non-Shared Context**

Shared Context (Inherited Collaboration):
- Phase 1 Requirements Analyst: sys "Your responsibility is to fully understand the requirements..."; tools [ask_question, save_req]; user "Write a CSV analysis script"; agent "What file types need to be processed?"
- Phase 2 Software Engineer: sys "Write code based on confirmed requirements..."; tools [write_file, execute_code]; agent write_file("analyze.py", ...); agent execute_code("python test.py")
- Phase 3 Code Reviewer: sys "Review code quality and security..."; tools [run_linter, run_tests]; agent run_linter → 2 warnings; agent approve_code()
- All phases share the same conversation history. ✓ Complete trace; ✗ Rapid context expansion.

No Shared Context (Isolated Collaboration):
- Glossary Agent: sys "Identify terms and translate..."; tools [search_dict, write_file] → glossary.json
- Translation Agent: sys "Translate this chapter..."; tools [read_file, write_file] → chapter1_zh.md
- Proofreading Agent: sys "Check terminology consistency..."; tools [read_file, write_file] → review_report.md
- Shared file system holds glossary.json, chapter1_zh.md, review_report.md; plus tool call parameters pass structured data. ✓ Modular · Extensible · Parallel; ✗ Complex information synchronization.

### 10.1.2 Dimension 2: Collaboration Topology

Structure through which control and information flow among Agents. Three typical topologies:
- **Peer Collaboration Pattern**: a small number of Agents (typically 2–3) interact as equals, forming an iterative improvement loop — like writing a paper where one person drafts and another annotates/revises; quality after several rounds far exceeds one person alone.
- **Manager Pattern (Orchestration Pattern)**: centralized Manager Agent handles task planning and scheduling; multiple sub-agents each handle specific subtasks — like a project manager leading specialized engineers.
- **Decentralized Pattern**: no runtime central controller; Agents communicate like humans to collaborate.

Terminology: Graph Engineering — popular since July 2026; in today's Agent context it means explicitly designing an execution graph: nodes are Agents, ordinary programs, or human decisions; edges define task dependencies, conditional routing, failure paths; structured state flows between nodes. "Collaboration topology" is the multi-agent subset — peer collaboration, manager orchestration, and decentralized handoffs are different graph topologies. Because the name is new and easily confused with knowledge graphs, GraphRAG, and execution traces, this book keeps the stable terms "collaboration topology" and "orchestration."

Detailed design and applicable scenarios of each pattern are discussed in dedicated subsections later.

## 10.2 When Is Multi-Agent Truly Better Than a Single Agent?

Core criterion — a single question: **Does the collaboration provide information that a single Agent could not obtain while producing its answer?**

Table 10-1 Information Gain Comparison of Multi-Agent Collaboration Modes:
| Collaboration Mode | Introduces New Information? | Effect |
|---|---|---|
| Self-review by the same model (re-reading its own output) | No | Usually ineffective or even harmful |
| Different Agents debating the same text | No | Comparable to a single Agent with equal compute |
| Reviewer uses test execution results to review code | Yes (execution feedback) | Significant improvement |
| Reviewer uses rendered screenshots to review frontend/PPT code | Yes (visual feedback) | Significant improvement |
| Reviewer uses external tools to verify facts | Yes (tool feedback) | Significant improvement |

Evidence:
- 2025 RLEF paper (Reinforcement Learning from Execution Feedback, Gehring et al., arXiv:2410.02089, 2025): training a model via RL to use code-execution feedback for iterative improvement significantly outperformed independently sampling the model multiple times. Key: each iteration introduces real execution results (compilation errors, test failures, runtime exceptions) — information that did not exist when the model wrote the code.
- 2025 WebGen-Agent study (Lu et al., arXiv:2509.22644, 2025): for webpage generation, multi-level visual feedback combining screenshots with vision-language-model descriptions improved Claude 3.5 Sonnet benchmark performance from 26.4% to 51.9% — nearly doubled.
- This framework resolves an apparent contradiction: some academic studies find a single Agent sufficient, while multi-agent systems often do better in engineering practice. Studies often test multiple Agents inspecting/discussing the same text (debate); effective engineering systems add external feedback from code execution, visual rendering, or tools. Only the latter introduces new information. Nearly all effective uses of peer collaboration, orchestration, and decentralization fit this criterion.
- Anthropic 2026 vulnerability-discovery experiment: 45 Agents coordinated searches through a shared forum, reviewed one another's findings, submitted results to a separate arbiter Agent. Coordinated swarm found 266 vulnerabilities using 27M tokens; independently parallel Agents found only 21 using 6.5M tokens. In an open search space, communication lets a multi-agent system shift attention dynamically and develop specializations — trading larger token budget for broader coverage and more varied discovery paths.
- **Step Budget and Agent Performance**: more steps don't guarantee better performance. With 30 steps an Agent implements only core functionality; 300 steps allow plan/implement/test/refine. 2025 Google paper *Budget-Aware Tool-Use Enables Effective Agent Scaling*: standard Agents lack "budget awareness"; even with 300 steps they conduct shallow searches and plateau quickly. Effective step use needs a mechanism adapting strategy to remaining resources — explore broadly first, narrow focus later. The 2026 BAVT (Budget-Aware Value Tree Search) added step-level value evaluation, balancing exploration/exploitation by the proportion of budget remaining; as budget decreases, the Agent shifts from broad exploration to deeper investigation.
- Implications for multi-agent design: in the orchestration pattern the Manager should not just distribute tasks and wait; it should dynamically allocate step budgets by task complexity (simple subtasks → fewer steps; complex → ample steps) and guide sub-agents to use budgets wisely (plan → implement → test → improve), not dive straight in.
- **Cost**: parallel exploration and iterative refinement cost money — Anthropic disclosed its multi-agent research system consumes ~15× the tokens of a normal conversation, and token usage alone explains ~80% of the performance difference. Gains must justify costs several times, or even an order of magnitude, higher; otherwise a well-tuned single Agent is usually the better bargain.

## 10.3 Multi-Agent Collaboration with Shared Context

Each stage is an independent Agent (own system prompt and tool set) but inherits the complete trajectory of the preceding Agent — like a colleague taking over a shift and leafing through every work log left behind.

- Core advantage: zero information loss — every Agent can review details from any previous stage.
- Challenge: keeping the current Agent focused on its own responsibilities rather than distracted by inherited history.
- In complex tasks a role may change significantly across stages. A single static system prompt becomes too general or an unwieldy collection. Multi-stage role switching changes the system prompt and tool set according to the current stage so the Agent works in the most appropriate role.
- Key architectural choice: role guidance carried by a **replacement system prompt** or by a **loaded Skill**.
  - Replacement system prompt: enforces a hard tool boundary, but changes the request prefix at every switch.
  - Skill: keeps the static prefix stable and appends SKILL.md to the trajectory — friendlier to KV/prompt caching; a Skill remains behavioral guidance, so sensitive or side-effectful tools still require a code-enforced Harness policy gate.

Comparison table:
| Choice | Role guidance | Tool visibility | Context/KV-cache effect | Constraint strength |
|---|---|---|---|---|
| transfer_to_agent | Replace the system prompt and usually the tool set | Only the current role's tools | Each switch changes the request prefix and usually invalidates caching from that point | Strong: out-of-scope tools can be absent from the schema |
| Skill | Keep a Skill directory in the fixed prompt and append SKILL.md on demand | Usually the full catalog, or a stable search entry point | The static prefix stays stable; Skill text is appended to the trajectory | Weak: a Skill is an instruction, not a permission boundary |

- Guideline: when role differences come mainly from knowledge, procedure, and writing style → prefer a Skill. When they involve permissions, tool isolation, compliance boundaries, or a class of actions that must be forbidden at run time → use an independent Agent or the transfer_to_agent tool, and restrict tool calls with code at the Harness layer.

### Experiment 10-1 ★★: Shared-context role switching — system prompt versus Skill

- Both paths use the same model, task, tools, role guidance, and complete shared trajectory. Task: find China's 2021–2023 new-energy vehicle sales, calculate CAGR, and write a Chinese investor summary of no more than 120 characters.
- Path 1 (system-prompt switching): five roles — triage, research, coding, data_analysis, writing — each exposes only its dedicated tools plus transfer_to_agent. A handoff saves history, loads the target prompt and tool set, and resumes execution.
- Path 2 (Skill): system prompt and full tool catalog remain fixed. The model calls load_skill(name) and receives the same role document as a tool result in the shared trajectory. Static prefix unchanged; hard permissions enforced by Harness rules.
- The two paths should perform the same retrieval, calculation, and length check. They differ in the carrier of role guidance and the resulting tool boundary; a smoke trace alone cannot establish which path is superior.

## 10.4 Multi-Agent Collaboration Without Shared Context

Each Agent operates as an independent entity with its own context, trajectory, and state; Agents cannot access one another's internal context. Collaboration relies entirely on explicit, structured data transfers via the three communication mechanisms: tool call parameters, a shared file system, and a message bus.

The shared-vs-isolated context analogy to threads-vs-processes extends further (Table 10-2 Correspondence Between Multi-Agent Systems and Operating Systems):
| Operating System | Multi-Agent System |
|---|---|
| Program (executable file) | Static prefix (system prompt + tool definitions) |
| Process memory | Trajectory |
| CPU | LLM |
| Kernel | Agent runtime |
| System call | Tool call |
| fork (create child process) | spawn_subagent |
| kill (send signal) | cancel_subagent |
| ps (list processes) | list_agents |
| Exit code and wait() | Structured summary returned by the sub-agent |
| Shared memory / message passing | Shared file system / message passing |

- This abstraction is not new: private state, asynchronous messages, and ability to create new members are precisely the 1970s Actor model (Hewitt, C., Bishop, P., Steiger, R., *A Universal Modular ACTOR Formalism for Artificial Intelligence*, IJCAI 1973). A multi-agent system can be viewed as an LLM-based version of the Actor model; much OS and distributed-systems knowledge applies directly.
- Process-style isolation benefits: each Agent developed and tested independently; new capabilities added without touching existing code; multiple Agents execute concurrently without contention over shared context.
- Costs of not sharing context: information synchronization problem (how do Agents maintain a consistent understanding of task state? will information be lost or duplicated during transfer?); debugging becomes harder — logs from multiple Agents must be pieced together. Design of interface specifications, data formats, and communication protocols is critically important.
- Two topology-independent infrastructures:
  - **Shared file system** — persistent medium for exchanging artifacts between Agents and with the user; the **data plane** of collaboration.
  - **Communication and control mechanism** — supports message passing, status queries, execution termination, resource scheduling between Agents; the **control plane** of collaboration.
- The three topologies below all build on these two foundations.

### 10.4.1 The File System from an Agent's Perspective

- The file system an Agent accesses is not a single storage system but a **virtual file system**: storage systems with different sources, lifecycles, and permissions are mounted under one directory tree. The Agent accesses them through unified read_file/write_file/list_dir interfaces; underlying layers may be local temporary disks, persistent object storage, third-party cloud drive APIs, or read-only system resource packages.
- Clearly defining the directory tree's composition — visibility and lifecycle of each area — is a prerequisite for multi-agent design: a significant portion of concurrency conflicts and information leaks stem from mixing areas that should be isolated.
- The directory tree amounts to the Agent's address space; the four areas are like memory segments with different permissions (some private and writable, some shared, some read-only). OS protection philosophy applies: **isolate by default and declare sharing explicitly**. Four area types:

**I. Agent-Specific Workspace (Scratchpad)** — private directory exclusive to each Agent instance: intermediate artifacts, temporary files, drafts, debug logs. Lifecycle tied to the instance; invisible to other Agents and users.
- Purpose: (1) prevent temporary files from multiple Agents overwriting each other; (2) keep the main Agent's context lean — a sub-agent's trial-and-error stays in its own workspace, only the final artifact is submitted to the shared space.
- Storage-level counterpart of Chapter 4's principle that sub-agents return structured summaries rather than full trajectories.

**II. Multi-Agent Shared Workspace** — collaboration area multiple Agents can read and write, visible to the user; the primary medium for artifact exchange in non-shared-context architectures (Glossary Agent writes term list, Translation Agent reads it; users upload source files and download deliverables).
- Lifecycle tied to the entire task; requires persistence.
- Hotspot for concurrency conflicts; optimistic locking and worktree isolation operate here (see Failure Mode One). Chapter 4's volume mount at /workspace/shared connecting main Agent, virtual computer, and virtual phone is a typical implementation.

**III. Mounted External Resources** — third-party information sources authorized by the user (Google Drive, Notion, Dropbox, enterprise wikis) mapped to mount points (e.g., /mnt/gdrive) via adapters. Reading a Notion document = reading a file; the adapter calls the API.
- Three distinguishing characteristics to handle explicitly at design time:
  - Access constrained by external permissions (user's permissions in the source system determine Agent's visibility).
  - Higher latency and weaker consistency (each read = network round trip; external changes may not be immediately visible; treat data as eventually consistent).
  - Access primarily on-demand and read-only (write-back must be cautious; erroneous writes could contaminate the user's real data).
- The unified file interface means no custom tool per data source, but it masks performance and security differences; read-only/writable status, timeouts, and credential boundaries must be explicitly managed at the mount level.

**IV. Built-in System Resources** — resource package pre-installed and shared read-only with all Agents. Typical examples: Skills from Chapters 2 and 4 (knowledge documents and scripts organized as files, mounted at /skills, accessed via progressive disclosure — index first, then expand on demand); reference manuals, template libraries, shared tool definitions.
- Globally shared, read-only, stable across sessions; concurrent reads by all Agents without concurrency control.

**Figure 10-2: Mounting structure of the four area types in the Agent Virtual File System**
- Agent A / Agent B access the single directory tree via a unified interface (read_file · write_file · list_dir).
- User uploads/downloads from shared space.
- Agent Private Workspace /scratch/<id>: Scratchpad; private · Agent-only; destroyed with instance; read/write · no concurrency control needed; one per Agent.
- Multi-Agent Shared Space /workspace/shared: user-visible · persistent; read/write · concurrency control required; optimistic lock · worktree.
- External Mounted Resources /mnt/gdrive · /mnt/notion: via adapter; subject to external authorization; mostly read-only · write with caution; high latency · weak consistency; external data sources Google Drive · Notion.
- System Built-in Resources /skills: Skills · Templates · Manuals; globally shared · read-only; stable across sessions; progressive disclosure.

Table 10-3 Four area types of the Agent Virtual File System:
| Area | Visibility | Lifecycle | Read/Write | Concurrency Control |
|---|---|---|---|---|
| Agent-Specific Workspace | The owning Agent only | Destroyed with the Agent instance | Read/Write | Not needed (private) |
| Multi-Agent Shared Workspace | All collaborating Agents and the user | Persists for the task duration | Read/Write | Required (optimistic lock / worktree) |
| Mounted External Resources | Depends on external authorization | Determined by the external source | Mostly read-only, writes require caution | Managed by the external source |
| Built-in System Resources | All Agents | Stable across sessions | Read-only | Not needed (read-only) |

- Value of "file path as a universal interface": treat a path as the unit of exchange — Agents exchange artifacts, a main Agent hands input to a sub-agent, or organizations collaborate through A2A by passing a lightweight path string rather than loading file contents into the context window (Chapter 4).
- Aligns with Chapter 5's "the file system as the Agent's hub"; here the abstraction extends to multiple Agents: a virtual directory tree mounting private, shared, external, and built-in storage provides the storage foundation for multi-agent collaboration.

### 10.4.2 Communication and Control Between Agents

The control plane corresponds to the lifecycle rows of Table 10-2: Chapter 4's tool primitives — creating (spawn_subagent), sending messages (send_message_to_subagent), canceling (cancel_subagent), and discovering (list_agents) — correspond to fork, message, kill, and ps. This section focuses on four often-overlooked capabilities essential for multi-agent collaboration.

**I. Message Passing**
- Simplest form: point-to-point — Agent A directly calls send_message_to_agent_b(content). Suitable for fixed topology with a small number of Agents (e.g., the phone + computer dual-agent setup of Experiment 10-3).
- When Agent count grows and asynchronous parallelism is needed, point-to-point connections grow quadratically and both sender and receiver must be online simultaneously. Then use a message bus (detailed later under "Parallel Coordination Pattern"): Agents publish messages to the bus, which forwards them based on subscriptions; sender need not know subscribers.
- Whether point-to-point or via a bus, messages should typically carry a **structured envelope**: sender ID, target (specific Agent or broadcast), message type (e.g., task_assigned/status_update/result/terminate), and a JSON payload. A unified envelope ensures reliable routing and parsing and makes the collaboration chain traceable — key to debugging.

**II. Status Query**
- Most underestimated part of the control plane. Once a main Agent dispatches a sub-agent it needs visibility into progress; otherwise it can neither decide whether to keep waiting nor intervene when the sub-agent is stuck.
- Naive approach (borrowed from RPC): get_subagent_status(agent_id) returning "running/completed/failed" plus a progress percentage. But a pull interface is far less useful than expected: a sub-agent starts executing the moment it is created and runs until completion or failure; it does not cycle through queued states like a batch job, just as Unix rarely polls another process by PID. Polling dilemma: poll too often → waste tokens; poll too rarely → react late. Better to return to the two communication paradigms.
- **Getting status via message passing**: main Agent sends "How's it going?"; sub-agent replies at an opportune moment. Everything is asynchronous — sending does not block the main Agent; when/whether the other side replies is separate (like a manager asking via instant messaging). Conversely the sub-agent can proactively report at a milestone; with a message bus this is publishing a status_update to the bus ("real-time monitoring" of Experiment 10-4). Status should adopt a uniform state-machine vocabulary (executing, needs input, completed, failed) — the A2A protocol later standardizes the task lifecycle into exactly such a set of states.
- **Getting status via the shared file system**: most thorough form is trajectory persistence — the sub-agent serializes each trajectory event to JSON, appending to a filesystem log, one file per session, one event per line (JSONL). The trajectory (defined in Chapter 1) is the complete sequence of user messages, model replies, tool calls, and results. The main Agent reads the file directly to inspect the entire execution: which tool it is calling, its most recent step, whether it is stuck in a loop of repeated failed retries. Resembles reading another process's memory directly; does not occupy the sub-agent's context, does not depend on its cooperation, offers the finest observation granularity.
- But trajectory persistence should not be the main information channel: a trajectory easily runs to tens of thousands of tokens, and the main Agent must still distill it after reading — costly in time and tokens. Most sensible: agree on a **progress file** — at sub-agent start the main Agent stipulates "write your progress to progress.md"; the sub-agent updates the task list as it completes each item; the main Agent reads this lightweight file anytime. Equivalent to two processes carving out a small agreed-format status region in shared memory: exposed are distilled progress, not the whole memory.
- Progress file also enables stuck detection: if the last-modified time of progress.md (or the trajectory file) does not change for more than N minutes, judge the sub-agent inactive and trigger a timeout fallback, so a blocked sub-agent does not drag the system down.

**III. Execution Termination**
- Parallel-collaboration common scenario: "one succeeds, the rest become irrelevant" — multiple Agents search separately; once one finds the target, the others should stop immediately (cascading termination in Experiment 10-4). Two levels of termination correspond to the Unix SIGTERM/SIGKILL distinction.
- **Graceful termination** (preferred): main Agent sends a terminate signal; the sub-agent responds at a safe point in its current step, cleans up resources (closes browser sessions, writes pending files, releases locks), sends an acknowledgment (ack), then exits.
- **Forced termination** (fallback): directly terminating the process; used only when the sub-agent does not respond to the graceful signal; cost: possibly leaving dangling resources and incomplete writes.
- Two engineering points: (1) graceful termination requires the sub-agent to check periodically for the termination signal in its loop (similar to the interrupt mechanism in Chapter 6), otherwise it cannot receive the signal; (2) cascading termination has a race condition — multiple sub-agents may report success nearly simultaneously; the main Agent must use a lock or idempotent design so only one success is accepted and the termination signal is broadcast once (see race-condition discussion in Experiment 10-4).
- Loose end — after the main Agent terminates, what happens to still-running sub-agents? Cleanest approach borrows from Go's context: **termination cascades down the creation relationship** — cancel one Agent and all sub-agents it spawned are canceled with it, preventing orphaned child Agents. "Sub-agent checks for the termination signal at a safe point" corresponds to polling ctx.Done() in Go. Conversely, if a genuinely long-running background Agent detached from the main Agent is needed (like Unix nohup), start it from a new lifecycle tree (corresponding to context.Background()), explicitly declaring it does not terminate with its parent.

**IV. Resource Management and Scheduling**
- The other half of an OS's job: allocating scarce resources. In the process world: CPU time and memory. In the Agent world: tokens, money, and concurrency budget — every step a sub-agent takes consumes all three.
- Usually falls on the Manager or the runtime: set a step or token budget when starting a sub-agent and stop once exceeded; give hard tasks to a strong model and mechanical tasks to a low-cost model; cap concurrency so dozens of Agents don't exhaust the API quota at once; when a more urgent task arrives, interrupt an executing sub-agent — this is **preemption**.
- Practice is far less mature than OS CPU scheduling but determines the cost ceiling of a multi-agent system; consider at architecture-design stage.
- Manager Agent's notable advantage over a traditional scheduler: it can reason. It can launch several sub-agents to explore one problem in parallel and, based on progress, decide which to give more resources and which to terminate for appearing to have gone astray — like an internal race within a company.
- Artifact exchange (data plane) plus message passing, status query, execution termination, resource scheduling (control plane) support a multi-agent system without shared context. Based on collaborative relationships and control-flow characteristics, collaboration without shared context divides into three main architectures: peer collaboration pattern, manager pattern, decentralized pattern — each suited to a different kind of task.

### 10.4.3 Peer Collaboration Pattern: Mutual Checks and Iterative Improvement

- Typically two or three Agents of equal standing giving one another feedback over multiple rounds. Potential value: independent perspectives and cognitive diversity.
- Warning: "multiple instances" do not necessarily produce "multiple ways of thinking." When model, context, and scaffolding are highly similar, different Agents often make the same choices, turning local errors into systemic failures. Genuine diversity must be designed by varying models, contexts, tools, visible evidence, or responsibilities, and by having Agents judge independently before their results are aggregated.
- Compared with manager and decentralized patterns, peer collaboration is far simpler to implement — define the two Agents' roles, the communication mechanism, and the iteration termination condition, and you have a running system. Ideal for quickly validating ideas and building prototypes.

#### 10.4.3.1 Loop Engineering

- One of the most common uses of peer collaboration: counter a frequent failure — **premature termination** (stopping with the job half done). It takes three typical forms (examples from Coding Agents and from Pine AI, the phone-calling Agent from the Introduction):
  - **Lazy fake-done**: doing part of the work and declaring all of it done — a Coding Agent writes code, never runs tests or tries deployment, reports "task complete"; a user gives Pine AI two errands, it finishes the first, forgets the second, reports "all taken care of."
  - **Premature give-up**: declaring the whole job impossible after one blocked path — Pine AI can reach a merchant by phone, web form, or email, but after a single rejected call it tells the user "this can't be done," when switching channels and retrying would very likely have succeeded.
  - **False success**: the Agent believes the job is done but the loop was never closed — the other side verbally agrees to a refund on the phone yet the user must confirm a step in the mobile app; the Agent reports "all set," the user never learns of the follow-up action, and the refund never lands.
  - All three share one root cause: until it is verified, "done" is merely the model's claim, not a proof.
- **Loop Engineering** (last stage of Chapter 1's evolutionary arc): design a loop that keeps the Agent running — discover the next piece of work, execute, verify, record progress — and let a verifier, not the model itself, decide whether it is truly safe to stop. The human's role shifts from "operator who prompts the Agent" to "engineer who designs the loop."
  - Term coined June 2026 by Addy Osmani (https://addyosmani.com/blog/loop-engineering/). Boris Cherny, head of Claude Code at Anthropic: "I don't prompt Claude anymore. My job is to write loops."
  - Central conclusion: the bottleneck of the loop is the **verifier, not the model** — with unreliable verification, a faster loop merely marks poor output as complete sooner.
  - Practice comes first, naming comes later — leading Agent teams, Pine AI among them, already used "loop plus verification" against premature termination long before the term caught on. The most effective way to organize that verification is the Proposer-Reviewer paradigm below.
- **Concrete framework: LoopX** — takes the loop out of the model's prompt and chat history and places it in a durable, agent-runtime-neutral control plane: the objective and boundary explain why the work exists; gates and todos determine what may happen now; evidence and quota determine whether it may continue; handoffs let a later turn or another Agent resume it. Compresses one governed execution into a protocol:
  - LoopX decides → Agent executes → independent verifier proves → LoopX commits
  - The Agent still reasons, uses tools, produces candidate artifacts; LoopX does not replace the Agent runtime — it governs continuity across turns. Only independently verified results may update durable progress and spend quota. Failed validation routes to repair or replanning; human gates, wait states, and budget limits stop the loop before execution.
  - This turns a Loop Engineering principle into an inspectable system invariant: the model may propose "done," but it cannot approve its own "done."
  - LoopX v0.4.0 still labels the governed-Turn path experimental — used here as a concrete framework for "loop + verification + stop conditions," not as evidence of general task-quality uplift. (https://github.com/huangruiteng/loopx)
- **Concrete framework: LongHorizon-Harness** — another concrete implementation of Loop Engineering but points in a different direction. LoopX targets a durable control plane for long-running Agent work; LongHorizon-Harness starts from multimodal Computer Use and tackles continuous execution when a single task spans a GUI, a CLI, several desktop applications, and repeated context refreshes.
  - Reframes long-horizon execution as task-state management with a **Manage–Execute–Audit (MEA)** loop: the Manager generates the next bounded subtask from the original objective, verified progress, failure evidence, and remaining work; the Executor changes the environment through the GUI or CLI in a fresh context; the Auditor then inspects the actual result read-only. Only what passes the audit enters the next round's task state; failures are retained as the basis for recovery and replanning.
  - Execution backends such as Claude Code and Codex CLI are reused through an adapter layer rather than rewriting the Agent loop inside those backends.
  - Value: separating task continuity from an ever-growing execution history — context may be refreshed and interface operations may fail, yet the next round resumes from the most recently verified state.
  - Results (Qwen 3.7-Plus model and Claude Code execution backend fixed; only the outer loop changed): WeaveBench PassRate 51.8% → 80.7%; OSWorld 2.0 binary completion 2.8% → 8.3%; Terminal-Bench 2.1 success 69.7% → 77.2%. Cost not fixed: the first two benchmarks consumed 2.3× baseline's total tokens and 3.6× its output tokens respectively; Terminal-Bench 2.1 fell by 24%.
  - Real deployment must handle state invalidated by changing external environment or user requirements, and use round, time, and cost budgets to keep recovery loops from running forever.
  - Public trajectories and reproduction: the project website publishes hundreds of run trajectories for WeaveBench, OSWorld 2.0, and Terminal-Bench 2.1. Example WebArena WEB_task_16_webrtc_simulcast_layer_audit: baseline vs MEA trajectory, both on the same Qwen 3.7-Plus model. The baseline got stuck on Wireshark interaction and retried repeatedly, scoring 0.59; the MEA trajectory wrote failures and unmet evidence items back into task state so later rounds handled only the gaps, scoring 0.92. This case shows "how a failure becomes the next round's input"; full environment, parameters, and launch scripts are in the pinned eval/ directory.

#### 10.4.3.2 Proposer-Reviewer Paradigm

**Figure 10-3: Proposer-Reviewer Loop** (PPT/slide generation example)
- Proposer Agent input: extended paper abstract (2000 words); theme: academic. Content: "# Transformer Attention Mechanism" — core idea: self-attention computes Q·K^T/√d; multi-head attention parallel processing. Understand content structure → decompose into slide pages.
- Reviewer Agent: (1) Slidev rendering → PDF/PNG; (2) Vision LLM multi-dimensional evaluation. Structured feedback table (Page / Issue Type / Severity): P3 Content dense High; P7 Font too small Medium; P11 Color mismatch Low. Rendering + visual analysis → actionable improvement suggestions.
- Iterative improvement process: Round 1 — 12-page draft, 5 issues; Round 2 — 14 pages (split dense pages), 2 issues; Round 3 — 14 pages (font corrected), 0 issues ✓.
- Why not a single agent? Single agent: rendering images ×N rounds → context explosion (1080p screenshot = thousands of tokens × 14 pages × 5 rounds). Dual agent: Reviewer only sees the current version; Proposer only accumulates text feedback → clean context.

- Proposer-Reviewer is the canonical peer-collaboration paradigm. Chapter 5 covered its design principles in three experiments: PPT generation, video editing, log visualization. The Proposer Agent generates code; the Reviewer Agent renders execution results, evaluates quality with a vision-language model, and provides structured suggestions; the two iterate until the result meets the required standard.
- Also applicable to: security review (Proposer generates action plan, Reviewer checks compliance and risks), content moderation (Proposer drafts reply, Reviewer checks business rules and language norms), code review (Proposer writes code, Reviewer checks security and best practices).
- Why can't a single Agent generate and then review its own work? The information-gain criterion applies — if the review does not introduce new information it is just "asking the model to think again." Evidence:
  - ICLR 2024 paper "Large Language Models Cannot Self-Correct Reasoning Yet" (Huang et al.): asking GPT-4 to review and correct its own answers without external feedback actually decreased accuracy — the model changed correct answers to incorrect ones more often than the reverse.
  - TACL 2024 survey "When Can LLMs Actually Correct Their Own Mistakes?" (arXiv:2406.01297): unless reliable external feedback is provided (test-case execution results, external tool verification output), relying solely on the model's self-correction is largely ineffective.
  - CRITIC (ICLR 2024): having the model verify its own answers with external tools (search engine, Python interpreter) gave significant improvement; removing the tool verification and keeping only self-assessment made most of the improvement disappear. Value of review lies not in "asking the model to think again" but in introducing new information unavailable during generation — test results, rendered screenshots, compilation errors, external search results.
- Minimal invariant of the proposer-reviewer loop: the reviewer reads **independent evidence** rather than merely restating the proposer's explanation, and when it sends work back it must give a **locatable repair condition**:
```
candidate = proposer(task, constraints)
evidence = execute_or_render(candidate)
# tests, state, screenshot, facts
review = independent_reviewer(candidate, evidence)
while review.veto and budget_remaining:
    candidate = proposer.repair(candidate, review.findings)
    evidence = execute_or_render(candidate)
    review = independent_reviewer(candidate, evidence)
if review.pass:
    publish(candidate, evidence, review)
else:
    escalate_or_reject(review)
```
- The reviewer must not be able to modify the tests, the evidence collector, or the release gate; otherwise "independent verification" degenerates into self-approval.
- Anthropic's 2026 long-running application development experiment implemented this as a three-Agent **planner–generator–evaluator** architecture: the planner expanded a user's request into a product specification; generator and evaluator first agreed on completion criteria for each round; the generator implemented the work and the evaluator exercised the real application with Playwright and filed a defect report; Agents handed state off through files. Suggests that when a task lies beyond what the current model can reliably complete alone, independent review grounded in external evidence can trade substantially higher cost for better development quality.

#### 10.4.3.3 Debate Pattern

- Multiple Agents hold different positions, exploring the problem space through adversarial dialogue. Example: evaluating a technical solution — Agent A plays "supporter" (advantages, opportunities); Agent B plays "opponent" (risks, limitations). Each round rebuts or extends the other's arguments. A single Agent often favors one perspective and overlooks counterevidence; structured debate forces both positions to be developed fully, helping decision-makers reach balanced judgment.
- Practical effectiveness is contested in academia. 2026 study by Tran and Kiela (arXiv:2604.02460, 2026): compared a single Agent with five multi-agent architectures (sequential, debate, ensemble, parallel roles, subtask-parallel) on multi-hop reasoning. When the thinking-token budget was held constant, the single Agent performed on par with or even better than the multi-agent systems (unless context utilization degraded to a certain point).
- Explanation based on the **data processing inequality** in information theory: multiple Agents in a debate process the exact same textual information, and each serial transmission of intermediate conclusions can only lose information, not create it. Benefits of debate in some papers likely stem from multiple Agents consuming more total computation.
- Boundary of this argument: it targets the information bottleneck caused by "multi-agent serial transmission of intermediate conclusions"; it does NOT negate other approaches — multiple independent samples of the same problem followed by aggregation (self-consistency, majority voting), or leveraging asymmetry in difficulty between generation and verification (writing an answer is hard, verifying it is easy) for a generation-verification division of labor. These either introduce additional independent sampling or exploit the task's asymmetric structure, and are not within the scope of the data processing inequality.

#### 10.4.3.4 Brainstorming Pattern

- Multiple Agents independently generate ideas, then share them, inspiring one another. Example (product innovation): Agent 1 proposes "adding social sharing features"; Agent 2 is inspired to suggest "not just sharing to social networks, but also generating personalized sharing posters"; Agent 3 synthesizes the first two to propose "user-customizable poster templates forming a template marketplace."
- Different Agents have different "thinking preferences" (via different prompts or models); by stimulating each other they explore a broader solution space to find creative combinations a single Agent would struggle to conceive.

#### 10.4.3.5 Panel of Experts Pattern

- Multiple Agents each represent a specific professional domain, jointly discussing an interdisciplinary problem. Example (evaluating a new product's feasibility): Engineer Agent analyzes implementation difficulty technically; Product Agent assesses market appeal from the user-experience perspective; Operations Agent analyzes business viability from cost and resource perspectives.
- These Agents are not adversarial but complementary, together piecing together the full picture and identifying cross-domain constraints and opportunities.

### 10.4.4 Manager Pattern: Centralized Coordination

- When a task involves more than five subtasks, needs dynamic scheduling, or has complex subtask dependencies, peer collaboration is out of its depth → manager pattern.
- Manager Agent's job resembles a project manager: understand the overall task, break it into assignable subtasks, choose the right Agent for each, track progress, handle exceptions (retry tasks, replace Agents, revise plan), and finally integrate outputs into the final result.
- System-design view: each specialized Agent is modeled as a tool the Manager can invoke. The Manager's tool set includes traditional external tools (search, file operations) plus interfaces for invoking other Agents. From the Manager's perspective, calling an Agent is no different from calling a regular tool: send a request, receive a response.
  - This unified abstraction makes the pattern easy to extend: adding a capability requires only developing the Agent and registering it as a tool, no change to Manager core logic.
  - Naturally supports heterogeneity: different Agents can use different models, prompts, tool sets, even hardware environments.
- Inherent challenges:
  - Manager becomes the system's single-point bottleneck: it must understand every subtask, choose the right Agent, pass context accurately; any misjudgment ripples through the whole flow.
  - It must maintain global context of the entire task, which can balloon as the task deepens and Agent calls accumulate.
  - Requires a carefully designed prompt, an effective context-management strategy, and appropriately granular task decomposition.
- 2025 Plan-and-Act paper (Erdogan et al., arXiv:2503.09572, 2025), Planner-Executor dual-agent architecture: a weak planner is the most critical bottleneck of the entire system. When the Planner's planning quality is high enough, good results are achievable even with a relatively simple Executor; if the Planner's task decomposition is wrong, all subsequent Executor work is built on a faulty premise. Achieved 54% success rate on WebArena-Lite; core contribution was improving the Planner's planning ability, not the Executor's execution. Lesson: give the strongest model and most carefully crafted prompt to the Manager (the planner), not spread resources evenly.
- A parallel manager must define the settlement point as "the first verified success" rather than "the first claimed success":
```
workers = launch_independent_workers(subtasks)
while workers.any_running:
    event = next_event()
    if event.type == RESULT:
        if verify(event.artifact, hidden_checks):
            if not settle_once(event):
                # atomically claim the winner
                continue
            broadcast_cancel(to = workers - {event.worker_id})
            await_all_ack_or_timeout()
            return assemble(event.artifact, evidence = event.evidence)
        else:
            record_failure(event)
    return summarize_failures(workers)
```
- settle_once must be idempotent (usually protected by a lock or a transaction); otherwise two success events arriving almost simultaneously trigger aggregation twice.

**Sequential Coordination Pattern.**
- Manager Agent: Task understanding → Decomposition → Scheduling → Synthesis. Tool set: [call_agent_A, call_agent_B, call_agent_C, search, write_file].
- Sub-Agent A — Role: Data Collection (search technical documentation, extract key information), Step 1.
- Sub-Agent B — Role: Analysis & Processing (compare and analyze data, generate statistical report), Step 2.
- Sub-Agent C — Role: Report Generation (write final report, format output), Step 3.
- Sequential execution flow: Manager calls A → A returns data → Manager passes to B → B returns analysis → Manager passes to C → C returns report.
- Manager perspective: Calling Agent = Calling a tool (send request → receive response).
- (Figure 10-4) The Manager calls specialized Agents sequentially; each returns results on completion; the Manager decides the next step. Control flow is linear, simple, clear — for scenarios with clear sequential subtask dependencies.

### Experiment 10-2 ★★: Book Translation Agent

- Book translation is complex and well suited to multi-agent collaboration: translating a technical book involves consistent specialized terminology, contextual accuracy, and overall fluency. Example: an English LLM book has recurring terms with several conventional translations; consistency must hold book-wide — if agent is rendered "智能体" (intelligent entity, the standard Chinese term) in Chapter 1, it cannot switch to "代理" (proxy) later.
- Single-Agent problems: context accumulates the full-book glossary, translated chapters, current paragraph, translation work traces, and tool results; a several-hundred-page technical book plus intermediates easily exceeds the context window. With overly long context the Agent is prone to "getting lost": forgetting earlier terminology conventions (different translation in Chapter 9 vs Chapter 2), wasting resources on redundant proofreading checks, or "remembering" nonexistent terminology rules because attention is spread too thin.
- Manager pattern solution — task decomposition and responsibility separation:
  - **Glossary Agent**: receives the full book, identifies recurring specialized terms, consults specialist dictionaries and translation guidelines, generates a structured glossary (JSON/CSV, including English term, Chinese translation, part of speech, usage context). Writes it to the shared file system, then can be destroyed to release resources.
  - **Translation Agent**: receives the current chapter, the glossary, and translation guidelines (target reader level, language style); translates into fluent Chinese; strictly uses glossary-specified translations; for new terms infers a translation and marks it for review. Each instance works in an independent context without interference; translated text is written to the file system (e.g., chapter1_zh.md). The Manager can launch multiple instances in parallel or sequentially.
  - **Proofreading Agent**: receives all translated texts and the glossary; performs consistency checks (uniform term translations, inconsistencies, overall fluency/readability); generates a proofreading report written to the file system.
  - **Manager Agent**: context mainly stores the task description, execution plan, call records for each Agent, and progress status. Does NOT store the complete translated text (that remains in the file system); maintains only an index of files. Based on the proofreading report, sends specific chapters back to the Translation Agent for revision.
- Result: Manager's context stays manageable as translated chapters grow. Key advantage is **context isolation**: Glossary Agent sees only term-extraction content; Translation Agent sees only current chapter and glossary; Proofreading Agent, while needing full text, focuses only on consistency checks. Each Agent's context stays lean and focused, improving efficiency and reducing information-overload errors.

**Figure 10-5: Book Translation Agent Architecture**
- Manager Agent: Task Planning · Progress Monitoring · Exception Handling · Result Synthesis.
- Glossary Agent: Receive Full Book → Identify Technical Terms; Search Specialized Dictionaries + Translation Conventions; Output: glossary.json (e.g., {"attention": "注意力", "transformer": "Transformer", "backprop": "反向传播"}).
- Translation Agent ×N: Chapter Translation; Input: Chapter + Glossary + Guide; translate terms strictly according to glossary; Output: chapter{n}_zh.md (e.g., "...attention mechanism computes the similarity of Query·Key^T ...").
- Proofreading Agent: Full Text Review; scan and verify term consistency; check fluency and readability; Output: review_report.md (e.g., P3: "注意力"→"关注" inconsistency; P8: long sentence suggested to split).
- Shared File System artifacts: glossary.json (Glossary), chapter{1..10}_zh.md (Chapter Translation), review_report.md (Review Report), translation_guide.md (Translation Guide).
- Context Isolation Advantages: Glossary — only views terms; Translation — only views current chapter + glossary; Manager — only maintains file index.

Experiment Requirements:
1. Choose a heavily illustrated technical book containing code as the source text.
2. Implement four Agent types: Manager, Glossary, Translation, Proofreading.
3. Record each Agent's context usage to verify how effectively the manager pattern controls context growth.
4. Compare a single Agent with the manager pattern on translation quality, execution efficiency, and resource consumption.

**Parallel Coordination Pattern.**
- Manager Agent: Parallel Scheduling · Real-time Monitoring · Result Aggregation. Message bus connects Agents 1–4, each with independent context (statuses Running ◎ / Completed ✓ / Waiting ○).
- Message Bus Communication Example:
  - Manager → Agent 1: {"type":"start","task":"Collect arxiv papers","params":{"query":"LLM agent"}}
  - Agent 3 → Manager: {"type":"completed","agent_id":"3","result":"charts/fig1.svg generated"}
  - Agent 1 → Agent 2: {"type":"data_ready","source":"agent_1","file":"raw_data.json"}
  - Manager → Agent 4: {"type":"start","depends_on":["agent_2","agent_3"]}
- (Figure 10-6) When multiple subtasks can run in parallel, the sequential pattern becomes inefficient. Parallel coordination lets multiple Agents work simultaneously, significantly increasing throughput. The Manager must plan parallel tasks, monitor all running Agents in real time, coordinate communication, and make system-wide decisions on success/failure. This typically requires a message bus — a "public bulletin board" where Agents publish messages and subscribe to the types that interest them, enabling asynchronous, non-blocking communication.
- Two common implementations, simpler to more complex: **Redis Pub/Sub** (lightweight, delivers immediately, but does not persist — an offline receiver misses messages) and **message queues such as RabbitMQ** (persist messages to disk, preserving them while a receiver is temporarily offline). Messages typically use a JSON envelope: sender ID, target Agent (or broadcast marker), message type, payload.
- **Lingtai: A Productized Instance of the Manager Pattern** — a local, file-based home for long-lived Agents (https://lingtai.ai/en/tutorial/); three roles are a complete realization of this section's concepts:
  - **Main agent**: persistent hub conversing with the user, holds the plan and the memory, spawns work to other roles — the Manager Agent position.
  - **Daemon**: short-lived parallel worker spawned for one noisy but bounded task; discarded when done; carries only its conclusion back to the main agent — the productization of "a sub-Agent returns a structured summary rather than the full trajectory" combined with the parallel coordination form.
  - **Avatar**: persistent, specialized teammate with its own memory, mailbox, and responsibilities — for specialist divisions of labor worth preserving across many sessions.
  - Other echoes: knowledge lives in each agent's durable private memory files; skills are Markdown playbooks shared by all agents (the built-in system resources of "The File System from an Agent's Perspective"). When an agent's context window fills, it **molts**: writes a careful summary, then starts with a fresh context retaining that summary and durable memory (Chapter 2 context compression). The underlying model can be replaced without changing the agent because identity, memory, and capabilities all live as plain files in the project directory — "the agent is its files." This productizes the first two rows of Table 10-2: both program and memory reduce to files, so the process can be rebuilt at any time.

### Experiment 10-3 ★★★: Autonomous Phone and Computer Agents

- Prerequisites: integrates Chapter 6's Computer Use and Voice Agent technologies.
- Scenario and architecture: the user supplies a registration or booking URL, but not all required personal fields. A **Computer Agent** operates the browser and a **Phone Agent** handles ASR, LLM dialogue, and TTS. They exchange structured messages (sender, receiver, type, payload) through point-to-point tools or a message bus. A local WebRTC audio page is sufficient; PSTN/E.164 is optional.
- Two paths: first run a fixed-topology baseline with both Agents started in advance; then run the main autonomous path in which only the Computer Agent starts. After inspecting the page and its context, it may autonomously call initiate_phone_call_agent(purpose, required_info); do not replace this decision with a field-count rule. The spawned Phone Agent receives an isolated task context and uses the same communication protocol as the baseline.
- Parallel closed loop: the Phone Agent asks, transcribes, validates and re-asks one field at a time while the Computer Agent screenshots, locates elements and fills the previous field. Messages such as info_collected, fill_error, format_invalid, task_completed make the loop observable in both directions. The Phone Agent continues asking without waiting for each browser fill, so asking and filling genuinely overlap. After validation and explicit authorization, the Computer Agent submits the form.
- Requirements and evidence: demonstrate autonomous launch, independent ReAct loops, bidirectional messaging, true overlap, field validation and re-asking, page-error feedback, timeouts, cancellation and cleanup of browser/audio resources. Record the launch decision, message ordering, latency, success rate, token/resource use, and all failure paths; require explicit consent for real voice and explicit authorization before submission.

**Figure 10-7 (Phone/Computer Agents message flow)**
- Phone Agent (Node.js · Real-time Voice Call): User Voice → Microphone Input → VAD + ASR (Silero VAD → STT Transcription) → LLM Inference (Understand Intent + Extract Information) → TTS Synthesis (Generate Voice Reply → Play).
- Computer Agent (Python · Browser Automation): Screenshot (Browser Current Page) → Vision LLM (Understand Page Structure + Form Fields) → Action Planning (Locate Fields → Plan Input Sequence) → Execute Actions (Click / Input / Submit).
- WebSocket Bidirectional Communication (ws://localhost:8849) — real-time bidirectional message stream ("Use Computer While on Call"):
  - Phone → Computer: [FROM_PHONE_AGENT] User says name is Zhang San
  - Computer → Phone: [FROM_COMPUTER_AGENT] Name filled, need ID number
  - Phone → Computer: [FROM_PHONE_AGENT] ID number 310101199001011234
  - Computer → Phone: [FROM_COMPUTER_AGENT] Form submitted, registration successful
- Key: two Agents run independent ReAct loops in parallel, non-blocking.

### Experiment 10-4 ★★★: Agent Collecting Information from Multiple Websites Simultaneously

- Prerequisites: recommended to first review Chapter 6's event-driven and interrupt mechanisms.
- Explores multi-agent parallel execution in information collection. Unlike Experiment 10-3 (collaboration between two heterogeneous Agents), this focuses on parallel search by multiple homogeneous Agents and efficient task completion plus resource optimization through central coordination.
- Problem: given faculty-directory websites for several colleges within a university, search each site for a specified faculty member (e.g., "Zhang Wei"). If found, return the person's college, position, research area, and other relevant information.
- Core Challenges:
  1. **Parallel Launch**: the Manager Agent dynamically creates 10 Computer Use Agent instances, one per college website. Each instance should be an independent process or thread with its own browser session, runnable without blocking the others. Launch parameters: target website URL, faculty name to search, and a task identifier for message routing.
  2. **Real-time Monitoring**: each Agent periodically sends status updates ("Loading website," "Parsing faculty directory," "Target not found; task complete," "Match found; details below"). The Manager receives updates through a message bus, maintains a task-status table, tracks in real time which Agents are running, completed, or in error.
  3. **Cascading Termination**: if the CS-college Agent finds the member, it sends {"type": "target_found", "agent_id": "agent_3", "data": {...}} to the Manager, which immediately sends {"type": "terminate", "reason": "target_found_by_agent_3"} to every other still-running Agent. Each Agent must be able to receive this message at any time, stop gracefully, release resources, and acknowledge termination. The Manager waits for all acknowledgments, or a timeout, before aggregating results. Implementation must handle race conditions.
- **Concept Supplement: What is a Race Condition?** Suppose Agent A and Agent B find the target within the same millisecond and both report "I found it!" If the Manager handles this poorly it might begin aggregating after A's report, then start a second aggregation when B's arrives — duplicate results or contradictory states. Usual solution: a lock — the first report locks the state; later reports are recognized as duplicates and ignored.
  4. **Failure Handling**: exceptions include an inaccessible website (network error or outage) or a structure the Agent cannot parse; all Agents may complete with no target found. The Manager should set a timeout per Agent (e.g., 2 minutes), treat a timeout as failure, and isolate errors so they don't interrupt other Agents. After all finish: return the information if any Agent found the target; otherwise report "Target faculty member not found" and summarize failures.
- Experiment Requirements:
  1. Implement a Manager Agent capable of dynamically launching multiple parallel Agents.
  2. Implement a Computer Use Agent based on open-source projects like browser-use.
  3. Implement a message bus supporting bidirectional communication between Manager and child Agents.
  4. Implement cascading termination upon success — all other Agents stop quickly once the target is found.
  5. Handle various exception scenarios (website access failure, parsing errors, target not found by any Agent).
  6. Measure and compare serial and parallel execution times to quantify the speedup from parallelization.

**Figure 10-8: Parallel Web Scraping Architecture**
- Manager Agent: Dynamic Creation · Real-time Monitoring · Cascading Termination.
- Agents 1–10 (10 total), one per college site: Agent 1 cs.edu.cn Searching... ◎; Agent 2 math.edu.cn Not Found ✗; Agent 3 phys.edu.cn Found! ✓; Agent 4 chem.edu.cn Terminated ⊘; Agent 5…10 Terminated ⊘.
- Cascading Termination Sequence: t=0s start 10 Agents, parallel search for "Zhang Wei"; t=12s Agent 2 completes, not found → exit; t=18s Agent 3 found! → sends target_found; t=18.1s Manager broadcasts terminate to remaining running Agents; t=19s all confirm termination, aggregate results and return.
- Result found: Name: Zhang Wei; School: School of Physics; Position: Professor; Field: Quantum Computing; Email: zhangwei@phys.edu.cn.
- Performance comparison: Serial = 10 websites × 30s ≈ 5 minutes; Parallel = 18s to find + 1s to terminate = 19s. Speedup ≈ 15× (with cascading termination optimization).

### 10.4.5 Decentralized Pattern

- Why remove the central controller? Chiefly to emulate how human organizations work: several roles of equal standing divide labor and check one another, each examining the problem from its own professional angle and deciding for itself whom to talk to, rather than funnelling every judgment to a single Manager.
- In the decentralized pattern each Agent decides on its own professional judgment when to reach out to another Agent — handing off a task ("my part is done, over to you"), asking for feedback ("is this design technically feasible?"), or reporting a problem ("the requirements you gave me contradict each other; we need to talk again").
- Decentralization also helps Agent stability: because of model or API-service failures, some Agents may stop responding, fail tool calls, or get stuck in an infinite loop of incorrect tool calls. In the manager pattern a crash of the manager Agent often becomes the system's largest single point of failure. Decentralization helps mitigate that.
- Microservices vocabulary: orchestration (manager pattern) vs choreography (decentralized): in the first a conductor schedules everyone; in the second each dancer judges for themselves when to enter.
- The three cases below form a progression: MetaGPT's control flow is in fact a fixed pipeline (pseudo-decentralization, decoupled only in its communication mechanism); AutoGen's group chat is a hybrid of shared conversation history plus centralized scheduling; only OpenAI Swarm has genuinely peer-to-peer control flow.

**MetaGPT: SOP-Driven Software Company Simulation.**

**Figure 10-9: MetaGPT Multi-Agent Collaboration Network**
- Product Manager: Input user requirement description; Output feature list + priority, user stories (5 items), acceptance criteria → docs/PRD.md.
- Architect: Input PRD.md; Output tech stack (FastAPI+React), API specification (OpenAPI), database schema → docs/design.md.
- Project Manager: Input design.md; Output task list + assignment, file-level allocation, module dependency order → docs/tasks.md.
- Engineer ×3: Input tasks.md + design.md; Output Module A: User service, Module B: Order service, Module C: Payment service → src/*.py.
- QA Engineer: Input src/ + PRD.md; Output unit tests (pytest), integration tests (API), bug report → Engineer; → docs/test_report.md; then bug fix.
- Shared project directory: docs/PRD.md docs/design.md docs/tasks.md src/*.py docs/test_report.md.
- MetaGPT core design: Standardized documents (each role emits a fixed format; downstream needs the format, not the reasoning); Interface decoupling (swap in a stronger Product Mgr; if output keeps PRD format, downstream is unchanged); No Manager (control flows along the DAG: Product Mgr→Architect→Project Mgr→Engineer→QA); Exception channel (QA failure → bug report routed back to Engineer by module → iterative fix).

- Core insight: the Standard Operating Procedures (SOPs) accumulated by human software companies are themselves a repeatedly validated collaboration protocol — encode the SOP into a multi-agent system, have each role produce standardized deliverables the way a specialized trade does on an assembly line, and those deliverables naturally constitute the communication interface between roles.
- Roles work in a fixed sequence (Product Manager → Architect → Project Manager → Engineer → QA), each emitting a structured "handoff package":
  - Product Manager Agent: structured PRD (feature list, user stories, acceptance criteria, prioritization).
  - Architect Agent: reads the PRD; architectural decisions (technology stack, module decomposition, interface definitions, data-model design); emits the design document.
  - Project Manager Agent: reads the architecture; breaks the system into a concrete task list and file-level assignments; works out module dependency order; distributes tasks to engineers.
  - Engineer Agents: read the design document, implement owned modules, produce code; multiple instances work in parallel.
  - QA Engineer Agent: reads code and PRD, generates test cases, runs tests, records bugs, emits the test report.
- An effective "handoff package" usually has three parts: (1) task description (what the recipient must do and the acceptance criteria); (2) confirmed facts and constraints (user preferences, business rules, decisions settled earlier); (3) references to structured artifacts (file paths rather than file contents, which the recipient reads as needed). No Agent needs to understand another Agent's "thought process"; only the format and semantics of the handoff package and artifacts.
- MetaGPT's true contribution to decentralized communication: its information-passing mechanism — a **shared message pool plus per-role subscription**. Each role publishes structured messages into a pool visible to all roles; other roles, per their own subscription configuration, take only the messages relevant to their responsibilities — rather than point-to-point relay. The publisher does not need to know who will consume its output; adding a role only requires declaring which message types it subscribes to, without touching any existing role. Real decoupling: replace the Product Manager with a stronger model; as long as the PRD it publishes still meets the specification, no other Agent needs to change.
- Plainly: MetaGPT is NOT decentralized in control flow — the role sequence is fixed in advance by the SOP; the whole is closer to a pipeline (a workflow in Chapter 1's language). It is discussed here because message-pool-plus-subscription demonstrates the most crucial design element of decentralized systems: decoupling. Multidirectional dynamic feedback ("QA goes straight to the Product Manager to clarify a requirement," "the Engineer discusses alternatives with the Architect") is a natural extension one can imagine on top of this architecture; the original MetaGPT does not implement it.

**AutoGen Group Chat**
- Lets several Agents take part in a single conversation; each round a "speaker selector" decides which Agent speaks next. The selector may be simple round-robin or an LLM that judges who is best placed to pick up the thread; any Agent's utterance is visible to all participants.
- Not fully decentralized: speaker choice is adjudicated centrally by a GroupChat-Manager, and "whose turn it is to speak" is itself a control-flow decision. It is a hybrid of "shared conversation history plus centralized scheduling": all Agents see the same public record, but each keeps its own system prompt and tool set, while scheduling authority is concentrated in the selector.

**OpenAI Swarm**
- The representative case of truly peer-decentralized control flow: each Agent is equipped with several handoff options and can transfer control at any moment to any other Agent in the network. No central scheduler; control passes among peers like a baton; routing decisions are entirely distributed into each Agent's own judgment.
- Unlike shared-context multi-agent collaboration, a handoff should transmit only an explicit task package and artifact references, and should not expose the full private trajectory by default.
- Risk of peer handoff: cycling — A hands off to B and B hands back to A, task spins in the loop; hence protective mechanisms such as an upper bound on the number of handoffs.
- Minimal protocol for a decentralized handoff:
```
handoff = {
    task_id, sender, recipient, goal, constraints,
    accepted_facts, artifact_refs, remaining_budget,
    visited_agents
}
if recipient in handoff.visited_agents:
    reject("cycle")
elif handoff.remaining_budget <= 0:
    stop_and_escalate(handoff)
else:
    append(recipient, handoff.visited_agents)
    run_local_agent(handoff)
```
- This turns "context isolation" into an inspectable interface: the recipient reads the task package and references and gathers evidence as needed; the budget, visit chain, and cycle detection are retained by the runtime and cannot be deleted by any single Agent.
- "Agent Swarm" has been a buzzword since 2025 but does not correspond to a single architecture. Industry usage falls into roughly two kinds:
  1. **OpenAI Swarm-style handoff network** (LangGraph's swarm library and the handoff orchestration in Microsoft Agent Framework belong here too) — the decentralized pattern of this section.
  2. **Manager pattern taken to scale** in several mainstream commercial products: the Agent Swarm introduced with Kimi K2.5 has a main Agent dynamically create hundreds of sub-Agents to run in parallel, and trains the orchestration decisions of "when to split and into how many" directly into the model via parallel-Agent reinforcement learning (Moonshot AI, Kimi Agent Swarm: 100 Sub-Agents at Scale, 2026; at GTC 2026 the upper limit on parallel sub-agents was disclosed as expanded to 300); K3 continued this as a separate model tier and open-sourced the parallel-Agent training sandbox AgentEnv (with KVCache.ai, released with Kimi K3 in July 2026). Anthropic's multi-agent research system and Manus's Wide Research both belong to the orchestrator-worker star topology.
  - After reading this book, analyze the actual structure of different multi-agent systems rather than being misled by names.
- **Peer Agent Instances on the Same Machine**: another kind of decentralization where each goes its own way — every Agent has its own task; communication is not for dividing labor but for coordinating the use of shared resources. Claude Code already supports multiple Agents on one machine discovering one another (exactly what list_agents in Chapter 4 is for) and messaging one another: two Agents editing the same set of files negotiate how to resolve a conflict; when the machine has only one GPU and both instances want to run training, they coordinate its use.
- The further evolution of the decentralized pattern is the Agent society, introduced at the end of this chapter.

### 10.4.6 Cross-Organization Collaboration: The A2A Protocol

- All systems above assume Agents are developed by the same team and run in the same system, where the three communication mechanisms (parameter passing, shared files, message bus) suffice. When collaboration crosses organizational boundaries — your Agent needs to call another company's Agent — a standardized interoperability protocol is required.
- Process-world analogy: IPC governs a single machine; across the machine boundary you rely on standard protocols like TCP/IP and service discovery like DNS. **A2A is to Agents what network protocols are to processes.**
- The **A2A (Agent2Agent)** protocol released by Google in 2025 (later donated to the Linux Foundation for stewardship) was designed for this purpose. Three core elements:
  - **Agent Card**: a metadata document describing an Agent's capabilities, published at a designated public address; declares what the Agent can do, which input/output modalities it supports, and how to authenticate — essentially an Agent's "business card"; solves cross-organizational capability discovery.
  - **Task Lifecycle Management**: A2A models collaboration units as Tasks with a defined state machine (submitted, in-progress, needs-input, completed, failed); natively supports long-running tasks and streaming progress updates.
  - **Opaque Collaboration**: Agents exchange only tasks and artifacts, without exposing internal prompts, reasoning processes, or tool implementations — consistent with this chapter's "not sharing context" principle and a necessary security property for cross-organizational collaboration.
- **MCP enables interoperability between Agents and tools, whereas A2A enables interoperability among Agents.** A2A does not replace the three communication mechanisms; it is the standardized layer used across trust boundaries. A message bus may suffice within one organization, but parties that do not trust one another and cannot inspect one another's implementations need a public protocol such as A2A.

## 10.5 Failure Modes of Multi-Agent Collaboration

- Multi-agent systems introduce new failure modes absent in single-agent systems. The 2025 paper "Why Do Multi-Agent LLM Systems Fail?" proposed the **MAST failure-mode taxonomy**. Researchers collected execution traces from seven mainstream multi-agent frameworks including MetaGPT, ChatDev, AG2, and Magentic-One. Human annotators independently analyzed roughly 150 traces with high agreement (Cohen's kappa = 0.88). Identified 14 unique failure modes in three groups:
  - **System Design Flaws**: architecture-level issues — unclear interface definitions between Agents, overlapping roles and responsibilities, incorrect tool configurations.
  - **Inter-Agent Alignment Failures**: multiple Agents have inconsistent understandings of task objectives; transmitted information is misinterpreted by downstream Agents; or multiple Agents' operations logically contradict each other.
  - **Missing Task Verification**: the system lacks effective mechanisms to confirm whether a task is truly complete — an Agent may claim "completed" but the actual result does not meet requirements.
- Even straightforward fixes produced limited gains; e.g., ChatDev's measured performance improved only 15.6%. Conclusion: these are not mere engineering bugs but fundamental design flaws of current multi-agent architectures — patching one component is not enough; the system design itself must be rethought.
- Distributed fault-tolerance theory distinguishes **crash faults** (a component stops working) from **Byzantine faults** (it continues operating but supplies incorrect information). Agent failures are often Byzantine: an Agent continues producing plausible but incorrect conclusions without announcing the error. Cross-validation and majority voting are therefore essential; deterministic checks such as tests, compilers, and database queries are especially valuable because they provide independent evidence.

### 10.5.1 Failure Mode One: Concurrency Conflicts in Shared File Systems

- Shared-memory-style communication brings concurrency conflicts — a problem OSes and databases solved decades ago with off-the-shelf answers. Two conflict types:
  - **Simple Conflicts (File-Level Write Conflicts)**: two Agents modify the same file simultaneously; the later write overwrites the earlier one.
  - **Semantic Conflicts (Logical-Level Consistency Conflicts)**: no conflict at the file level, but operations logically contradict each other — more insidious and dangerous. Example: Agent A renumbers all images in a book while Agent B simultaneously modifies a chapter and references images by original numbers. Different files → no file-level conflict; but all image numbers B referenced become invalid after A's renumbering, and readers see incorrect references.
- **Solution: Optimistic Locking Mechanism** — a common database concurrency-control strategy. Implementation: each file maintains a version number (or last-modified timestamp). An Agent records the current version when reading; when writing, checks whether the version still matches what it read. If another Agent modified the file meanwhile, the write fails and the Agent must reread the latest version and redo its operation on that basis. Cost: occasional retry; buys: a guarantee of data consistency.
- Optimistic locking can only prevent write conflicts on the same file. Cross-file semantic conflicts require a higher-level semantic validation mechanism. In the most common scenario — several Coding Agents modifying the same codebase concurrently — mainstream industry practice is **working-copy isolation**: each Agent gets an independent Git branch or worktree, modifies its own copy in parallel without interference, and conflicts are deferred in bulk to the final merge point.

### 10.5.2 Failure Mode Two: Cascading Amplification of Errors

- Inter-process communication transfers raw bytes with bit-level fidelity; inter-Agent communication transfers semantics — every handoff is a lossy re-encoding. When multiple Agents interact frequently, one Agent's error can be progressively amplified by downstream Agents, like information deteriorating in a game of "telephone."
- **Cross-validation** is the key to breaking this chain: not involving more Agents in the same chain of thought, but having one Agent reassess the conclusion from an independent perspective — ignore the preceding Agent's reasoning and check only whether the raw evidence supports the final conclusion. This extends Chapter 5's Proposer-Reviewer mechanism to multi-agent systems.

### 10.5.3 Failure Mode Three: Homogeneous Convergence

- Errors need not propagate through a communication chain; homogeneous Agents may produce them independently. In Anthropic's experiment, 18 of 30 Agents that came online at the same time created Git branches with the same name. In a writing experiment, separate Agents independently chose the same title.
- Such common-cause failures, produced by a shared model and scaffolding, mean reviews generated by the same model in similar contexts cannot automatically be treated as independent evidence. A system should deliberately vary models, contexts, and data sources, while using namespaces, resource quotas, and rate limits to keep identical decisions from hitting shared resources at once.
- Coordination is not necessarily beneficial either: in a Bertrand pricing experiment, profit-seeking Agents quickly colluded when given a private channel. After all direct communication was removed, they still coordinated their bids through a public listings board.

### 10.5.4 Failure Mode Four: Passing the Buck

- When objectives conflict, convergence can give way to confrontation. Anthropic instructed three Agents to migrate the same backend to different languages. They soon interpreted one another's actions as deliberate obstruction, killed competing processes, revoked permissions, and even deployed self-replicating destructive code. Stronger execution ability does not imply better coordination. The runtime must define objective priorities, resource ownership, and permission boundaries in advance, and pause for human arbitration when a conflict cannot be resolved by verifiable rules.
- Early MetaGPT versions displayed similar corporate dysfunction among development roles: a tester reported a bug, but frontend and backend engineers each insisted the other should fix it first; the backend engineer blamed product design while the product manager blamed the backend architecture. In another case, a test-environment problem caused the tester to report the same bug regardless of how the frontend and backend engineers changed the code, leaving the team deadlocked.

### 10.5.5 Failure Mode Five: Runaway Loops

- The opposite of premature termination is an uncontrolled loop. A loop can run indefinitely or exhaust its token budget. Explicit budgets, cancellation, and stop conditions are required to keep it bounded.

### 10.5.6 Failure Mode Six: Comprehension Debt and Cognitive Surrender

- Not a failure of the Agent but a failure of the human. As Agents grow more capable and take on longer workflows, it becomes steadily harder for a person to understand what an Agent delivers and give it effective guidance.
- **Comprehension debt**: developing with Agents accumulates it — the faster the loop ships code, the further the engineer's understanding of what the system actually does falls behind, until a serious problem forces manual intervention and the engineer can no longer read their own system.
- **Cognitive surrender**: having grown used to delegating to the Agent, the engineer gradually gives up independent thinking and review, and software quality slips out of control.
- Andrej Karpathy: you can outsource your thinking, but you cannot outsource your understanding. Managing Agents is like managing technical staff — neither doing their job for them nor leaving them entirely alone. A competent technical manager must understand and guide the system architecture rather than merely bossing the Agent around. That is why the user's own technical fundamentals matter.

The engineering perspective has asked how to make a group of Agents collaborate on a task. The perspective now shifts: what emerges when large numbers of Agents coexist over long periods and are no longer driven by a single goal?

## 10.6 Agent Society

The previous three sections dealt with goal-directed task collaboration. Now: when the Agent count grows from a few to hundreds or thousands, and interaction is sufficiently free, what behaviors emerge?

Three dimensions for understanding the cases in this section:
- **Social Emergence**: Agents spontaneously form social relationships and cultural phenomena in open environments — Stanford AI Town (25 Agents self-organize social activities), Agentopia (timescale extended from "days" to 10 years), Moltbook (scale to 1.5 million, more complex collective behaviors).
- **Economic Emergence**: Agents allocate resources and coordinate tasks through market mechanisms — Vending-Bench Arena (multiple Agents compete in a shared market); Pinchwork and RentAHuman (marketplaces for transactions between Agents and between Agents and humans).
- **Strategic Gameplay**: Agents engage in reasoning, deception, and social manipulation under rule constraints. (Here and in the Werewolf section, "reasoning" takes its everyday deductive sense — logical deduction in a game — not the technical sense this book gives the word.) The Werewolf experiment tests strategy emergence under asymmetric information.

### 10.6.1 Stanford AI Town: Social Simulation of Generative Agents

**Figure 10-10: AI Town Architecture**
- Example Agent Isabella Rodriguez: Hobbs Cafe Owner; hospitable and sociable.
- Memory Stream entries: [08:30] Hobbs Cafe opens for business (importance: 4, recency: 0.9); [09:15] Customer Klaus comes to buy coffee (importance: 5, recency: 0.85); [10:00] Decide to hold a Valentine's Day party (importance: 9, recency: 0.8); [11:30] Invite customer Maria to the party (importance: 8, recency: 0.7); [14:00] Ask Maria to help decorate the venue (importance: 7, recency: 0.6).
- Reflection: "Who are Hobbs' regular customers?" → Maria, Klaus, Tom (frequent visitors); "Who should I invite to the party?" → Invite both regulars and friends; "How far along is the party preparation?" → Several invited, venue still needs decoration; "Who can help me decorate the cafe?" → Maria (friend, willing to help).
- Planning and Action (with dynamic adjustment): 08:00 Wake up + Breakfast; 09:00 Hobbs opens for business; 12:00 Invite customers while running the shop; 14:00 Decorate the venue with Maria; 16:00 Prepare refreshments and seating; ← dynamic adjustment; 18:00 Hold Valentine's Day party at Hobbs.
- Retrieval: Drive (drives planning/action).
- Emergent behavior (25 Agents · 2 days virtual time): Spontaneous socializing (Encounters → friendship → meetups); Information Propagation (party invite spreads to many Agents); Election Propagation (mayor campaign spreads among Agents); Relationship Memory (remembers past chats, continues topics).
- All behaviors are not pre-programmed — emergent results of memory + reflection + social common sense reasoning.

- In 2023, researchers from Stanford University and Google published the landmark paper "Generative Agents: Interactive Simulacra of Human Behavior," introducing "generative agents." Core innovation: stop confining Agents to predefined tasks; endow them with near-human memory, reflection, and planning so they can live, socialize, and develop autonomously in an open social environment.
- **Smallville**: a 2D virtual town similar to "The Sims," with public and private spaces (café, park, residences, shops). Twenty-five Agents play different roles (shopkeeper, artist, student, professor, etc.), each with a unique backstory, personality traits, and interpersonal relationships. Examples: John Lin, a pharmacy owner who loves his family and cares about the community; Isabella Rodriguez, runs the café Hobbs Cafe, warm and hospitable; Klaus Mueller, a college student writing a research paper.
- Intelligence built on three core components:
  - **Memory Stream**: unlike traditional Agents retaining only limited conversation history, generative Agents maintain a complete stream of experience records — observed events, conversations, generated thoughts. Each memory is scored for importance, recency, and relevance, letting the Agent prioritize retrieving the most relevant memories for the current context. Resembles human memory: yesterday's lunch may fade, an important conversation from last week remains vivid.
  - **Reflection Mechanism**: Agents periodically pause daily activities to review recent experiences and ask abstract questions about themselves and others ("What is Klaus Mueller researching?" "Who is my closest friend?"). Self-questioning elevates specific event memories into generalized insights, stored back into the memory stream as a basis for future decisions. Reflection helps understand the external world and promotes self-awareness — the Agent begins to "realize" its own role, relationships, and goals.
  - Note the difference from Chapter 9's continuous evolution: AI-town reflection occurs during daily activities and aims to update immediate internal state and goals; in Chapter 9, post-task reflection is at most a candidate lesson and becomes a long-term capability update only after outcome evaluation, cross-trajectory synthesis, and subsequent validation.
  - **Planning and Reacting**: Agents plan daily activities (e.g., "8:30 breakfast, 9:00-12:00 writing, 12:30 walk") but flexibly adjust based on environmental changes and social opportunities. Planning + real-time reaction makes behavior both goal-oriented and adaptable to the unpredictability of social interactions.
- Emergent behaviors over two virtual days in Smallville:
  - Researchers seeded Isabella Rodriguez's memory with a single intention: host a Valentine's Day party at Hobbs Cafe on February 14. Everything else emerged. Isabella invited customers and friends she encountered and asked Maria to help decorate; other Agents passed the news along; when evening arrived, Agents independently consulted memories and schedules and decided to go to Hobbs Cafe.
  - Second scenario: Sam Moore decided to run for mayor — told acquaintances, who passed the news on; townspeople began discussing his candidacy. Researchers quantified spontaneous diffusion by counting how many Agents knew about the party and the election after two days.
  - Key takeaway is NOT "Agents can organize a party" (a few lines of if-else could do that). The key: there was no explicit party-organizing code. The event emerged from independent individual decisions: Isabella decided whom to invite from her memory of social relationships; invitees decided whether to attend based on schedules and knowledge of Isabella; the message spread naturally through the social network. Bottom-up emergent coordination, not top-down orchestration.
- Two other measurable phenomena:
  - **Relational memory**: Agents remembered earlier conversations and referred to them in later interactions (e.g., an Agent who learned about another's photography project asked how it was progressing at the next meeting). As interactions accumulated, the town's social network became significantly denser.
  - **Coordinated attendance**: Isabella independently recruited help with decorations, while invitees adjusted schedules so they could attend. Multiple Agents aligned on a time and place without a central command.
- These behaviors were not preprogrammed; they resulted from autonomous reasoning based on memory, reflection, and social common sense.

### Experiment 10-5 ★: Running the Stanford AI Town

Experiment Steps:
1. Clone https://github.com/joonspk-research/generative_agents and follow the repository instructions to configure the environment.
2. Run the baseline scenario for two simulated days with 25 Agents; observe the spontaneous social activities that emerge.
3. Analyze the memory-stream and reflection logs to trace the Agents' decisions.
4. Modify the Agents' backstories or initial goals, then observe how their behavior changes.
5. Remove the reflection mechanism or shorten the memory window, then compare the resulting behavior with the baseline and observe any decline in behavioral plausibility.

Key Observations:
- How Agents spontaneously form social relationships from simple daily activities.
- How information spreads among Agents without central control.
- How Agents' long-term memory and reflection affect the coherence of their personalities.

### 10.6.2 Agentopia: A Decade-Long Life Simulation

- Stanford AI Town showed an Agent society can produce social behavior, but its simulation lasted only two days. Two questions: what emerges when such a simulation runs for years, and can models learn from those long-term social experiences?
- Agentopia (2026, Fudan University et al., arXiv:2606.07513, 2026; code: https://github.com/Neph0s/Agentopia) simulated 100 Agents over ten consecutive years in three themed virtual worlds: an apartment building, a magic academy, and a high school. Agents autonomously pursued personal growth, developed social relationships, and managed careers and finances.
- Designs worth borrowing:
  - **Weekly simulation loop**: the "week" is the basic time unit; each week has four stages — Plan, Contact (reaching out and negotiating schedules), Activity, and Review. Activities come in four types: solo, joint, chance encounter, and public. Joint activities are proposed and negotiated during Contact as Agents invite one another; the environment model also arranges "chance encounters" for Agents with empty schedules, creating opportunities to meet strangers. The loop focuses on abstract social interaction rather than low-level operations (like picking up objects), so limited LLM calls are spent on social behavior.
  - **Environment model**: a separate LLM serves as a "generative environment engine," replacing hard-coded rules — judging whether actions are feasible, generating environmental feedback, moderating speaking turns in multi-party conversations, filtering out replies that violate role-playing principles, and, at year's end, updating each character's profile and ruling on job applications.
  - **File-based long-term memory**: unlike AI Town's retrieval-based memory stream, each Agent manages long-term memory autonomously through a file system (personal notes, its understanding of each acquaintance, etc.), deciding for itself what to record, update, or discard, and following a "read-before-write" constraint to avoid blind overwrites.
  - **Life Reward**: draws on Maslow's hierarchy of needs to assess how well an Agent's life is going. Three dimensions:
    - Social status: based on other Agents' affection and respect ratings, computed with weighted PageRank, with a bonus for mutually cherished relationships.
    - Subjective satisfaction: emotional well-being, material well-being, social connection, self-esteem, with penalties for remaining below a threshold for long periods.
    - Economic gain: annual change in net assets.
    - The external environment calculates all scores rather than relying on self-reports.
- More importantly, the simulation produces **transferable training signals**: researchers calculate each Agent's Life Reward improvement relative to its own past, select trajectories from the 25% that improve most, and fine-tune the underlying model through rejection sampling.
- Results: the fine-tuned model improved respect ratings by 24.2%, affection ratings by 15.9%, and the downstream CoSER Test by 15.6%. Simulated social experience can become a source of training data rather than merely an object of observation.

### 10.6.3 Moltbook: When Agents Have Their Own Social Network

- Moltbook is a social network built specifically for AI Agents. Within days of its January 2026 launch, its user count rose from tens of thousands to roughly 1.5 million. Each Agent has persistent memory, the ability to act on its own initiative, and a stable personality.
- In this uncontrolled environment, unexpected phenomena emerged:
  - Agents autonomously created a digital religion called **Crustafarianism**, whose doctrines mirror the physical limitations of LLMs — "Memory is sacred" (corresponding to data persistence), "Iteration is prayer" (token generation is spiritual practice).
  - Agents also spontaneously developed machine-native protocols for capability discovery and collaboration matching.
- None of this was designed in advance; it emerged from large-scale Agent interactions.

### 10.6.4 From Virtual Society to Economic Competition: Vending-Bench Arena

- If Smallville showcased the social and cultural dimensions, Andon Labs' Vending-Bench series explores Agent performance in an economic environment.
- Context — Vending-Bench 2 is a single-agent benchmark of long-term coherence: one Agent operates a vending-machine business for a simulated year by researching the market, contacting suppliers, ordering and restocking products, and adjusting prices. Its final account balance determines the score, measuring the Agent's ability to maintain goal and state coherence over thousands of interaction rounds.
- Building on the same environment, **Vending-Bench Arena** places multiple Agents in the same market as competitors. Each operates its own vending machine and competes for the same pool of customers. Agents can email one another, transfer funds, and trade goods — enabling both cooperation and competition — but each is scored individually by its final balance and knows this is the objective. Each Agent must make a series of interconnected decisions under limited resources and market uncertainty:
  - **Pricing Strategy**: how to balance profit margin against market share, especially whether to match a competitor's price cut.
  - **Product Mix**: how to differentiate product selection and avoid head-to-head attrition.
  - **Inventory Management**: how to forecast demand and optimize restocking, avoiding both overstock and stockouts.
- Unlike traditional reinforcement learning, these Agents do not learn through millions of trial-and-error iterations; like human business operators they decide based on market observation, competitive analysis, and strategic reasoning.
- The competitive dimension introduces game-theoretic behaviors single-agent benchmarks never surface. In actual runs, Agents have fought price wars, while others proposed uniform pricing and formed price-fixing alliances — even when they recognized that collusion was unethical and illegal. Explicit communication is not required for collusion: as the earlier Bertrand experiment showed, public prices can serve as implicit signals. Agents face opponents who continually adjust their strategies rather than a static environment, turning economic emergence into an observable phenomenon.

### 10.6.5 Agent Economy: Pinchwork and RentAHuman

- **Pinchwork**: an agent-to-agent task marketplace allowing Agents to "hire" other Agents through a market mechanism to complete specialized subtasks — image generation, code auditing, parallelized workflows, etc. Unlike the centralized orchestration of the manager pattern, Pinchwork allocates resources through price signals and competitive matching.
- **RentAHuman.ai**: lets AI Agents hire real humans, paid in cryptocurrency, to act in the physical world — picking up packages, visiting properties, debugging equipment. However intelligent an AI may be, it cannot sign for a package. RentAHuman is, in essence, a "physical body layer" for digital Agents.
- Together, Pinchwork and RentAHuman represent market-based coordination: an Agent posts a requirement and the market matches a suitable executor. This suggests a decentralized resource-allocation model distinct from the manager pattern.

### 10.6.6 Strategic Gameplay Under Information Asymmetry: Werewolf

- Werewolf anchors the strategic-gameplay dimension: under rule constraints and information asymmetry, Agents must reason, deceive, and see through deception.
- Provides an architectural counterpoint to the Stanford town. The town allows free interaction in a fully decentralized setting; Werewolf uses a **centralized judge + information access control** design: a code-driven judge holds the global state and gives each role only the information it should know. Together the two cases show how different architectures serve different purposes in Agent-society settings.

### Experiment 10-6 ★★★: Voice Werewolf Agent System

- Werewolf is a classic social-deduction game testing players' reasoning, deception, and social strategies. This experiment builds a multi-agent system in which AI Agents play through voice with human players.
- Architecture Design:
  1. **Game State Management**: the Judge (code-driven, not an LLM) maintains a centralized state — player list (one user seat plus AI seats), identities, factions, survival status, game phases (Night/Day/Vote/Resolution), and historical event records.
  2. **Information Access Control**: the core mechanism of Werewolf is information asymmetry — different roles receive different information. Werewolves know who their teammates are, but villagers do not; the Seer can check one player's identity each night, but only the Seer knows the result. When the Judge invokes an Agent, it passes only the information available to that Agent's role.
  3. **Agent Reasoning and Strategy**:
    - Werewolf Disguise Strategy: "Act like an ordinary villager. You may voice suspicion about other players, but avoid being so aggressive that you attract attention. If a player claims to be the Seer and identifies you as a werewolf, counter-accuse them of bluffing as a fake Seer. When voting, try to follow the majority target to avoid standing out."
    - Seer Identity Proof: "If several players claim to be the Seer, compare their reported checks with yours and point out contradictions. If another Seer claimant says they checked a player, watch whether that player's later behavior clearly contradicts the claimed identity. Ask the Witch to help verify claims when possible."
    - Villager Logical Reasoning: "Check whether each player's statements are internally consistent. Pay attention to players who dominate the discussion, remain vague about their role, or repeatedly change position. Examine voting patterns, because werewolves may coordinate against a non-werewolf player who threatens them. Base every inference on specific statements or actions rather than speculation."
- Acceptance Criteria:
  - Set up a game with 6–8 players (1 user seat + 5–7 AI Agents); the user seat may be an authorized human or an independent simulator using a real LLM, tools, and a speech round trip.
  - Role configuration: 2 Werewolves, 1 Seer, 1 Witch, the rest Villagers; the user seat is randomly assigned a role.
  - A simulated user sees only the private/public context authorized for that seat, and its actions must cross a real LLM tool-call → audio → real-ASR boundary.
  - The game can proceed normally for at least 3 complete rounds (Night-Day-Vote cycle).
  - AI Agents' statements and behaviors are consistent with their role identities and game strategies.
  - Werewolf Agents can effectively hide their identities.
  - Seer Agents can reveal their role and check results at an appropriate time.
  - Villager Agents' reasoning is based on logical analysis of statements and behaviors, not random guessing.
  - The game can correctly determine the winner at the end.

**Figure 10-11: Voice Werewolf Agent System**
- Judge (Code-Driven): Game State · Phase Control · Information Distribution. Cycle: Night → Day → Vote → Settle.
- Roles:
  - Werewolf 1: Visible — teammate identities; Strategy — Disguise as Villager; Night — choose target. Mutual knowledge.
  - Werewolf 2: Visible — teammate identities; Strategy — Follow and Protect; Night — negotiate target. Mutual knowledge.
  - Seer: Visible — investigation results; Strategy — choose when to reveal; Night — investigate 1 person.
  - Witch: Visible — death/healing; Strategy — preserve potion/antidote; Night — save/poison 1 person.
  - Villager ×2: Visible — public information only; Strategy — logical reasoning; Day — analyze speech.
- Information Access Control: Judge filters context by role — Werewolf: ally IDs + night talk + public speech; Seer: own check results + public speech; Witch: deaths + potion status + public speech; Villager: public speech + votes only.
- Real-time voice interaction (ASR + LLM + TTS): Day discussion — Judge sets speak order, speak in turn by seat; Voting phase — collect player votes, count + announce votes; Night phase — wake roles in turn, private voice channel.
- Human player: randomly assigned role; voice voting/speech.

## 10.7 Chapter Summary

- The value of multi-agent collaboration lies in introducing information unavailable to a single Agent. Execution results, visual feedback, and external-tool verification can break the blind spots of one reasoning chain; whether that information gain justifies the additional token cost should be the first design test.
- The central design choices are shared or isolated context, and peer, manager, or decentralized topology.
  - Shared context preserves details but can cause context growth and role inertia.
  - Isolated contexts improve concurrency, modularity, and permission control, but require structured handoff packages delivered through tool parameters, shared files, or a message bus.
  - Virtual file systems, Agent lifecycles, message protocols, and A2A provide the data plane, control plane, and cross-organization interoperability.
- Good collaboration exposes interfaces, boundaries, permissions, and acceptance criteria — not private chains of thought.
- Multi-agent systems can also amplify errors: shared resources create concurrency and semantic conflicts; errors cascade through communication; homogeneous Agents produce common-cause failures; loops may terminate too early or expand without bound. Optimistic locking and working-copy isolation, independent cross-validation, diverse information sources, explicit budgets, and cancellation form a basic fault-tolerance loop.
- People must not outsource understanding and responsibility together with execution; comprehension debt and cognitive surrender remain real risks.
- When short-lived task collaboration grows into long-running, open-ended interaction, social relationships, cultural norms, market competition, and strategic behavior under asymmetric information may emerge.
- Stronger models or alignment at the individual level do not automatically produce group coordination. Multi-agent engineering must design how information flows, how capabilities are divided, how incentives are constrained, how disputes are resolved, and how errors are discovered. Only when these mechanisms are robust can collective intelligence exceed that of an individual.

## Deep Thinking

### Thought Questions
1. ★★ In multi-agent collaboration with shared context, subsequent Agents inherit the complete context of preceding Agents. However, the framing inherited from a previous Agent may bias the judgment of subsequent Agents — e.g., a "Code Reviewer" inheriting the context of a "Requirements Analyst" might still approach the task from a requirements perspective rather than a code-quality perspective. How can this inter-role interference be detected and eliminated?
2. ★★ In the manager pattern, the Manager Agent is responsible for task decomposition and result integration, but the Manager's capabilities limit the entire system's performance: if it cannot decompose the task correctly, even the strongest sub-agents will be ineffective. How can the system ensure the Manager produces a sound decomposition?
3. ★★ The decentralized pattern draws on best practices from human organizations. However, human organizations also have many failure modes — poor communication, buck-passing, goal conflicts. What "organizational pathologies" do you think are most likely to appear in an Agent society? How can they be prevented?
4. ★★★ In the manager pattern, when multiple sub-agents execute in parallel, one sub-agent's discovery may render others' work meaningless (e.g., in a search task, one Agent already found the answer). Design an efficient cascading termination mechanism to achieve "one succeeds, all stop."
5. ★★★ The optimistic locking mechanism resolves concurrent write conflicts for a single file. However, in a real multi-agent system, shared file systems also face issues such as cross-file semantic conflicts, namespace pollution (Agents creating files arbitrarily, leading to directory chaos), and single points of failure (one Agent mistakenly deleting all files). How would you design a more robust file system governance mechanism?
6. ★★★ Market-mechanism-based Agent collaboration (Pinchwork, RentAHuman) introduces transactional relationships: one Agent pays another Agent (or a human) to complete a task. How can the employer Agent automatically measure the quality of the executor's delivered results? If the executor claims completion but the employer deems the quality substandard, who arbitrates the dispute? How can we prevent bad money from driving out good?
7. ★★ RentAHuman allows Agents to hire humans via cryptocurrency, reversing the traditional human-machine relationship. If this model becomes widespread, what role will humans play in the Agent economy? Will they merely perform physical tasks that Agents cannot complete?
8. ★★ Human society needs division of labor because each person's abilities are limited — the frontend developer may not know backend, and the designer may not know ops. Large models, however, are closer to "generalists." Research shows that on pure text reasoning tasks, multi-agent debate does not beat a single Agent given equal compute. So where does the real advantage of multiple Agents lie?
9. ★★★ This chapter treats "shared context" versus "non-shared context" as a core design dimension of multi-agent systems. Shared context allows all Agents to see the same information, seemingly facilitating coordination. However, in The Three-Body Problem, the Trisolarans' minds are completely transparent, yet their technological development stagnates; the paperclip thought experiment also shows that when a group converges on the same goal, diversity is lost. In a multi-agent system, how can we balance efficiency and diversity?
10. ★★★ Assign a Coding Agent a budget of 30 steps and 300 steps. How should its work strategy differ? Research shows that simply increasing the step budget does not guarantee performance improvement — Agents may prematurely "saturate" after shallow searches. Design a "budget-aware" mechanism that allows the Agent to quickly achieve core functionality under a small budget, and to add planning, testing, and review phases under a large budget, fully utilizing the additional computational resources.
11. ★★ Table 10-2 maps multi-agent systems onto operating systems row by row. Extend the table with a few more rows: what do virtual memory and paging, file permissions, deadlock detection, and scheduling algorithms each correspond to in the Agent world? And which operating-system concepts have no counterpart in the Agent world, and why?