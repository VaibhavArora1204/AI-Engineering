# Chapter 1 Getting Started with AI Agents

Intro: If you have used Cursor to write code and watched it search your codebase, edit multiple files, and rerun tests until they pass, you have already used an AI Agent. The same holds for Deep Research (investigating a topic through repeated searching and reading), Manus (controlling a browser to finish online tasks), the Doubao phone assistant (booking tickets or sending messages), and Pine AI (negotiating a lower telecom bill).
- Shared trait: these products are no longer passive "you ask, it answers" conversations. They plan their own execution steps, call the tools each task requires, and adjust their strategy as results come in. AI Agents are becoming a new way to interact with computers.
- Chapter approach: starts with practical examples, works back to core components; readers experience what modern Agents can do, understand the architecture behind them, learn design patterns and best practices for building Agent systems.
- Reading Tip: this chapter is the conceptual map for the whole book — a concise tour of the core formula, operating loop, engineering framework, and Agent design patterns; establishes shared vocabulary and reference points. Do not memorize on first read; aim for the big picture. Later chapters expand each aspect; return here to reorient.

## 1.1 Modern Agent = LLM + Context + Tools

Core formula (minimal engineering realization): **Agent = LLM (Large Language Model) + Context + Tools**.
- The "+" signs denote a combination of engineering components, not a formal RL definition.
- The formula describes only what lies inside the Agent's boundary; it does NOT include the Environment the Agent interacts with.
- Each term read broadly but with a clear boundary:
  - **LLM = Agent's reasoning engine**: more than model parameters; the decision-making core responsible for understanding intent, reasoning, planning, and judgment. Capabilities come from world knowledge and language ability acquired during pre-training, plus decision-making strategies encoded through post-training (SFT and RL covered in Chapter 8).
  - **Context = Agent's working set of information**: not just text fed to the model, but the working set available at each decision point — the environment, user memory, domain knowledge, its own state, and task progress. Like a person sizing up a situation and consulting references, the context window holds the information usable at that moment.
  - **Tools = Agent's action interfaces**: not just callable API functions, but the full set of ways the Agent can act — predefined tool calls, Skills loaded on demand, generating code to create new capabilities on the fly, delegating to sub-agents, reaching out to the user, responding to external events.
- Intuition: Agent = Reasoning Engine + Working Context + Action Interfaces. The model reasons and decides; context supplies the working set those decisions depend on; tools provide the interfaces through which decisions affect the outside world.

### RL perspective and Model–Harness structure (Figure 1-1)

Classical RL/control view: Agent and Environment are two sides of a closed-loop interaction, not components of one another. Environment returns an observation; Agent uses its context to choose the next action; that action changes the Environment's state, producing the next observation.

Figure 1-1 (Agent–Environment interaction loop and Model–Harness structure inside the Agent):
- Agent side contains:
  - **Harness** (model runtime & interaction layer): loop, state management, permissions, verification, correction
  - **Context**: observations, history, memory, task state
  - **Model**: understand, reason, choose next action
  - **Tools & action interfaces**
- Environment side: state & transition rules; current state → new state after action; things living there include file system, database, web, APIs, applications, users, other Agents, simulated or physical world.
- Arrows: environment → observation → agent; agent → action → environment.

Two levels of abstraction: outer level is Agent↔Environment interaction (Environment includes file systems, databases, web pages, users, other Agents, physical or simulated worlds). Inner level is the Model–Harness structure inside the Agent: Model makes policy decisions; Harness is the runtime and governance layer inside the Agent boundary that builds context, exposes tool interfaces, maintains loops and state, and applies permissions, verification, and correction. A Harness can create, isolate, or proxy an environment without containing the Environment's state or transition rules.
- Expanded formula: LLM is the Model; Context + Tools form the minimum Harness; production systems add constraints, verification, and correction inside that boundary.

RL correspondence (not strict one-to-one): tools define observation/action interfaces whose underlying objects remain in the Environment.

| Intuition | Agent Component | RL Concept | Role |
|---|---|---|---|
| Reasoning Engine | LLM | Policy | The decision-making logic that determines "what to do next" — given the current information, choose the most appropriate action from all available options |
| Working Context | Context construction | Observations and history | Organizes Environment observations and existing history into the information needed for the current decision |
| Action Interfaces | Tool interfaces | Observation/action interfaces | Defines which observations the Agent can read, which actions it can issue, and the format of those interfaces |

### 1.1.1 Observation and Action Spaces: The Interface Between Model and World

- Observation space + action space together form the interface between the LLM and its external environment.
- Observation space: translates environment information into context the model can process. Action space: translates model decisions into operations on the outside world.
- Information outside the observation space effectively does not exist for the model. An operation outside the action space remains something the model can only recommend in words, even if it knows exactly what should be done.
- Key leverage: once the underlying model is held constant, the primary systems-engineering lever for improving performance is often to redefine or expand observation and action spaces (= expanding context and tools). Many "needs a smarter model" problems are really interface problems: bring task-relevant data into context or expose the required operation as a tool, and a previously unsolvable task may become solvable.

**Manus: merging spaces that had been separate.** Before Manus, production Agents mostly followed three distinct tracks: Deep Research, Coding, and Computer Use. Manus was the first widely influential production Agent to bring all three together in one system. Its virtual browser enlarged the observation space; its file system, code execution, and command execution enlarged the action space. Manus did not become a general Agent merely by swapping in a stronger model — it took the union of three kinds of Agents' observation and action spaces, enabling one Agent to cross previous product boundaries.

**OpenClaw: extending the interface into the user's digital life.** OpenClaw pushes both spaces outward again. It receives tasks and returns results through messaging channels users already inhabit — WhatsApp, Telegram, Slack, Discord, iMessage, and many others — so the Agent can be reached from almost anywhere. Its local Gateway connects cloud applications such as Google Drive and Notion as well as the local file system. Files scattered across accounts and devices can, with the user's explicit authorization, enter one Agent's observation space and be acted on by its tools. Compared to the original cloud-sandbox-centered Manus (files generally had to be uploaded or a connector separately configured), local-first OpenClaw spans a broader data boundary. Manus later added its own Google Drive connector and desktop access to local files, reinforcing the point: product evolution often consists precisely of expanding observation and action spaces.

Footnote 1: Manus official materials describe its original Sandbox as an isolated cloud virtual machine. On introducing its Google Drive Connector, Manus recalled the earlier fragmented workflow of manually downloading/uploading files between Drive, desktop, and Manus. When it launched My Computer in March 2026, it called the fact that important work lives locally rather than in the cloud a fundamental limitation of the cloud sandbox. OpenClaw's official README describes a local-first, always-on personal assistant running on the user's own devices, lists more than twenty messaging channels; its tools and plugin system can add cloud integrations and local capabilities. References: free/manus.im/blog/manus-sandbox, manus.im/blog/manus-google-drive-connector, manus.im/blog/manus-my-computer-desktop, github.com/openclaw/openclaw, docs.openclaw.ai/tools.

**The 5 Agent product comparison table** (across Working Context / Action Interfaces / Strategy):

| Agent Product | Working Context | Action Interfaces | Strategy |
|---|---|---|---|
| Coding Agents (e.g., Cursor) | Requirements documents, codebase, terminal environment | Open-ended (internal reasoning, code search, file read/write, command execution, etc.) | Incremental development: understand requirements → search relevant code → edit code → test and verify → debug and fix |
| Search Agents (e.g., Deep Research) | Web resources, academic databases, local files | Open-ended (internal reasoning, search queries, web reading, summary generation) | Iterative deepening: adjust search direction based on existing information, gradually synthesize a complete report |
| Computer Control Agents (e.g., Browser Use) | Computer screen, browser pages, file system | Open-ended (internal reasoning, clicking, typing, scrolling, screenshots, code execution, etc.) | Visual perception + operation: observe screen → identify target elements → perform actions → verify results |
| Phone Assistant Agents (e.g., Doubao) | Phone screen, installed apps | Open-ended (internal reasoning, clicking, swiping, typing, opening apps, etc.) | Intent understanding + App control: understand user needs → locate target app → perform actions → confirm completion |
| Personal Task Agents (e.g., Pine AI) | User account information, historical bills, service provider knowledge base | Open-ended (internal reasoning, making calls, sending emails, filling forms, confirming with user) | Multi-step task execution: gather information → formulate negotiation strategy → contact service provider → negotiate → report results |

Three shared features: (1) an **open-ended action space** — not picking from a fixed set of buttons but generating arbitrary natural language and code; (2) **internal reasoning** — planning before acting; (3) **continuous interaction** — adjusting strategy based on environmental feedback. These come precisely from the interplay of reasoning engine, working context, and action interfaces — i.e., LLM, context, and tools.

### 1.1.2 Tools: The Agent's Action Interfaces

- Tools are the Agent's bridge to the outside world; they turn it from a passive observer into an active system that can search, write files, run code, call APIs, send messages, or operate interfaces. Without tools, an Agent is limited to text generation; with them, it can act on external systems.

Five tool types, sorted by direction of the Agent's interaction with the world:

| Type | Purpose | Content / Examples |
|---|---|---|
| **Perception Tools** | Allow the Agent to access information | Search engines (real-time web data), file systems (local documents), APIs and databases (external services and enterprise core data) |
| **Execution Tools** | Allow the Agent to act on external systems | Code execution, file operations, system commands, external API calls — turn decisions into concrete actions |
| **Collaboration Tools** | Allow the Agent to divide work with other Agents | Delegating specialized tasks to sub-agents, requesting human confirmation at key decision points, coordinating actions in multi-agent systems |
| **Event Trigger Tools** | Invoked differently from the first three: the Agent does NOT call them; they arrive as external inputs that trigger the Agent to begin work | A new email arrives, a scheduled time arrives, another system fires a Webhook callback — the event activates the Agent and initiates reasoning and action. The Agent never calls these itself, yet they are still a channel of interaction, so they count in the broad tool system |
| **User Communication Tools** | Channels through which the Agent communicates with the user | Where execution tools change the external world, communication tools carry information — delivering the Agent's progress or a proactive check-in by text message, voice call, email, and so on |

- Chapter 4 covers the full taxonomy and design principles for these five types.
- Tool design quality directly determines what an Agent can reliably accomplish: vague interfaces → model misuse; poor error handling → a single failed tool can leave the Agent stuck; overly broad permissions → one Agent error can become irreversible. As the MCP (Model Context Protocol) standard spreads, integrating tools is becoming easier.
- **Tool Calling** (also known as Function Calling) is a core capability of modern LLM Agents: lets the model invoke external tools in a structured way, transforming the LLM from a pure text generator into an intelligent system that can act through external interfaces. This book uses "tool calling" throughout.

**Tool calling — four steps** (the loop is the foundation of ReAct, introduced later):
1. The context tells the model which tools are available (names, purposes, parameters).
2. The model decides on its own whether to call a tool, which tool, and with what arguments.
3. The tool runs; its result is appended to the context.
4. The model decides its next move based on that result.

API-level simplified representation for a weather query:

```
Step 1: Declare tools
  tools: [{ function: "get_weather", parameters: { city: "string" } }]

Step 2: Model decides to call
  assistant: { tool_calls: [{ function: "get_weather", arguments: { city: "Beijing" } }] }

Step 3: Result appended to context
  tool: { tool_call_id: "call_1", content: '{"temp":28,"sky":"clear"}' }

Step 4: Model responds based on result
  assistant: { content: "Today in Beijing: 28°C, sunny." }
```

- The developer only defines the tools and executes the calls; the model decides whether to call, which tool, and what arguments. Chapter 2 examines this API structure in detail.

**Tool design principles:**
- Start with the narrowest capability the task needs, then expand gradually as the task grows more complex.
  - Basic arithmetic → a calculator with clearly defined parameters is enough.
  - Growing to reading spreadsheets, cleaning missing values, computing statistics, plotting charts → a constrained Python code interpreter is easier to combine and explore with than an ever-growing collection of specialized tools.
  - Generality increases error risk and expands the attack surface: code must run in an isolated sandbox, with **network access disabled by default**, no access to files outside the authorized working directory, and limits on execution time, CPU, memory, and output size.
- A single logging tool is fine for one execution; for long-running tasks (hours or days), a **controlled virtual working directory** can preserve plans, intermediate results, execution logs, and final artifacts so the Agent can resume across multiple runs. This directory should also restrict readable/writable paths, storage capacity, and file types, and prevent path traversal, instead of exposing the entire host file system.
- General-purpose tools are not always better than specialized ones. High-risk operations or those governed by strict business constraints — such as payments, data deletion, sending email, and production deployment — should be exposed as **dedicated tools** with explicit parameters, restricted permissions, and end-to-end auditability, with previews and human confirmation added when necessary.
- **Core principle of tool design**: use general-purpose foundational capabilities for composition and exploration; use specialized tools to constrain high-risk operations and enforce strict business rules.

### 1.1.3 LLM: The Agent's Reasoning Engine

- The LLM is the Agent's decision-making core. Given a user request it first infers the real intent (what users say is often not what they actually want), then breaks a vague or complex task into executable steps. Throughout execution it keeps deciding: what to do next, whether to call a tool, which one, and with what arguments. This understand–plan–execute capability comes from knowledge accumulated during pre-training, and is the foundation both workflows and autonomous Agents depend on.
- Distinctive capability: **internal reasoning** — before acting, the Agent can plan and reason through the task. This changes nothing in the environment yet markedly improves the actions that follow. The ability comes from pre-training (initial training on massive amounts of internet text, learning language patterns and world knowledge): the model draws on reasoning patterns encoded in human knowledge — mathematical laws, causal relationships, strategies for decomposing problems. Therefore, unlike traditional RL Agents, today's LLM-based Agents do not explore through blind trial and error; they reason over a structured body of knowledge.

#### 1.1.3.1 Model as Agent: When the Model Itself Becomes the Product

- "Model as Agent" is the newest direction in AI Agent development. Advanced models internalize tool calling as a native ability through post-training (especially reinforcement learning): when to call a tool, which one, with what arguments — the model decides all of it, with no manual orchestration required.
- That does NOT make the framework layer less important. On the contrary: the stronger the model, the more the surrounding **Harness** matters.
- **Harness metaphor**: the word Harness originally referred to the reins and tack fitted to a horse — not to limit its ability to run, but to direct that power appropriately. In the Agent context, the model is the powerful yet unpredictable horse; the Harness is the engineering infrastructure that channels its capability into reliable task execution. It includes context management, tool interfaces, safety constraints, and verification and correction mechanisms.
- The more decision authority a model has, the greater the impact of a wrong decision — calling for finer-grained constraint, verification, and correction. The real advantage of model providers is not "making the framework thinner" but co-optimizing the model and its surrounding Harness, iterating continuously.
- **The Bitter Lesson**: if models keep getting stronger, will today's Harness eventually be absorbed into the model? In "The Bitter Lesson," Rich Sutton looked back on a pattern across seventy years of AI research: researchers repeatedly encoded their understanding of a domain into a system, achieving short-term gains but ultimately losing to general methods — search and learning — that scale with compute and data. (Sutton, Rich. "The Bitter Lesson", 2019, http://www.incompleteideas.net/IncIdeas/BitterLesson.html)
- Book's position: **endorse the direction, stay pragmatic about the pace.** Directionally, models will continue to absorb parts of the Harness — tool calling and long-horizon planning once depended on external orchestration but are now native model capabilities. In practice, absorption is far slower than intuition suggests: training proceeds on a timescale of months, and no model can internalize all the constraints and preferences of real businesses in a single pass.
- The model's current capability boundary is precisely where the Harness creates value. Harness engineering is not resistance to the Bitter Lesson, but its practice on an engineering timescale: whatever the model cannot yet do reliably, the Harness covers first; whenever the model internalizes another layer, the Harness sheds that layer and moves on to support the next capability frontier.

#### 1.1.3.2 Agent Learning Mechanisms: From Contextual Adaptation to Persistent Updates

- A model can internalize tool-use policies as native capabilities through RL, but behavior changes do not occur only during training. Based on where an update occurs and how long it persists, changes are three complementary paths (Figure 1-2):

| Learning path | Timing | Mechanism | Persistence | Cost / trade-off | Example |
|---|---|---|---|---|---|
| **In-context learning** | Inference time | Soft update via attention | Temporary, adapts instantly; bounded by context window | Low cost, fast (milliseconds) | Learn a format from 3 examples |
| **Externalized learning** | Runtime | Knowledge base + generated tools | Persistent, updatable; reliable, verifiable | Reliable/verifiable but must be accessed via context or tool interfaces | Freeze a workflow into a tool |
| **Post-training** | Training time | Modify model weights | Permanent, general | High cost, slow to update (weeks) | Learn when to call a tool |

- Figure 1-2 axis: Learning Speed from Fast (Milliseconds) to Slow (Weeks).
- **Contextual adaptation** occurs within the current task: once examples, state, and retrieval results enter context, the model adjusts behavior immediately, but this does not change the persistent state of the next session. Advantages: speed and low cost. Limitations: context window and how information is organized. Chapter 2 explains this in detail.
- **Externalized learning** for changes that persist across tasks: facts and experience → knowledge documents; strategies expressible in language → a Prompt or Skill; deterministic procedures and constraints → programs and Harnesses. These artifacts are auditable and revisable, but the Agent must still access them at execution time through context or tool interfaces. Chapters 3–5 establish foundations for knowledge and programs; Chapter 9 covers generating such updates from evaluated operational trajectories.
- **Parameter updates** for high-dimensional capabilities — medical-image understanding, natural-language style, implicit decision policies — that external rules cannot fully express. Higher deployment costs, but natural and broad generalization. Chapter 8 presents the methods.
- The three paths are not mutually exclusive categories but coordinated mechanisms at different timescales: context supports immediate adaptation, external artifacts support controlled accumulation, and parameters internalize capabilities that are difficult to express explicitly.

### 1.1.4 Context: The Agent's Working Set

- Context is the working set of information available to an Agent at each decision point — like the materials a person puts on the table (task instructions, reference manuals, earlier correspondence, latest data).
- From the API's perspective (detailed in Chapter 2), each LLM call's context consists of five parts:
  1. **System Prompt**: developer-written, stays fixed for the whole conversation. The Agent's "job description" — defines its identity, permissions, and rules of conduct. Careful prompt engineering of the system prompt shapes the Agent's operating behavior. It also carries user memory that persists across sessions (personalized preferences, past behavior, background settings; see Chapter 3) plus dynamically injected environmental state.
  2. **Tool Definitions**: names, functional descriptions, and parameter formats of available tools. Without them the Agent cannot recognize or call any tools — though it does not fall silent either (Experiment 1-1 shows what it does instead). Together with the system prompt, tool definitions form the **static prefix** that stays unchanged throughout the conversation. (Foundational pattern; since 2026, production frameworks can also load full tool schemas on demand at the end of the context without breaking the prefix — see tool definitions section of Chapter 2 and Chapter 4.)
  3. **User Messages**: input from the user; may also contain externally retrieved knowledge via RAG (Retrieval-Augmented Generation; Chapter 3) — information beyond the training data cutoff or private domain knowledge.
  4. **Assistant Messages**: responses previously generated by the model, up to three parts — **reasoning** (internal chain of thought, maintaining coherence and decision interpretability), **content** (the response to the user), and **tool_calls** (the way the Agent takes action). They may not all appear together: when calling a tool, usually reasoning + tool_calls; when giving a final answer, usually reasoning + content.
  5. **Tool Results**: output returned after the framework executes a tool — the direct basis for the next reasoning step, and what lets the Agent learn from outcomes rather than repeat mistakes.
- **Static prefix** = system prompt + tool definitions (first two items). **Dynamic message history** = user messages + assistant messages + tool results (last three), growing with every interaction. Together the five parts make up the context of each LLM inference.
- Is every component truly indispensable? The direct test is an **ablation study** — the diagnostic method of ruling out causes one at a time: remove component A and see whether the system still works; then component B; until each component's contribution is clear.

**## Experiment 1-1** ★★: The Critical Role of Context
- Method: systematic ablation study of how each context component shapes Agent behavior. Of the five components, four were tested — the **system prompt was exempt**: as the Agent's basic identity definition, without it the Agent has no role awareness at all, and the test would be meaningless.
- Figure 1-3 — five controlled groups (each missing one component vs. a complete baseline):

| Group | System prompt | Tool defs | Tool exec results | Thought process | History messages | Result |
|---|---|---|---|---|---|---|
| Full baseline | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ Works normally |
| No tool defs | ✓ | ✗ | ✓ | ✓ | ✓ | ✗ Cannot call tools |
| No tool results | ✓ | ✓ | ✗ | ✓ | ✓ | ✗ Blind loop |
| No reasoning | ✓ | ✓ | ✓ | ✗ | ✓ | △ Inconsistent decisions |
| No history | ✓ | ✓ | ✓ | ✓ | ✗ | △ Repeated operations |

- Findings — the components are not equally important:
  - **Tool Definitions** (part of the static prefix) are the foundation of the action capability; without them the Agent cannot call any tool. But losing the ability to act does not mean falling silent: the model still returns a neatly formatted, confidently worded answer whose numbers come from **parametric memory** rather than from observations, laid out exactly like an answer genuinely derived from tool output. Whether it declines outright or fabricates on the spot depends mainly on the model's own hallucination rate and honesty; a prompt constraint such as "do not estimate the rates yourself" lowers the odds of fabrication without eliminating it.
  - **Tool Results** are key to closed-loop control; without them the Agent executes "blind," retrying until it exhausts its iteration budget.
  - **Reasoning process** (the reasoning part of assistant messages) records why a step was taken; tool results record what happened. When the former can be reconstructed from the latter, dropping it from the history costs almost nothing.
  - **Message history** (user messages, assistant messages, tool results from previous rounds) prevents redundant operations and avoids repeating the same mistakes.
- Core insight: **context determines what the Agent can see, and the Agent can only decide based on what it sees.** Components are not equivalent; the measure is whether the information one carries can be reconstructed from somewhere else. Claims like "every component is indispensable" have to be measured rather than assumed, and models move fast enough that the same ablation on a newer model may reach a different conclusion.
- Engineering note: "produced an answer" is not "completed the task." When a context component is missing, the typical failure is not an error exit but an answer that looks flawless.

### 1.1.5 The ReAct Loop

- The ReAct loop is the core mechanism that connects LLM, context, and tools into one system.
- **ReAct (Reasoning + Acting)**: the name mentions only reasoning and acting, but the loop has three stages: the model reasons about what to do next → calls a tool to act → observes the tool's result and reasons about the next step. "reason → act → observe" repeats until the task is done.
- **Trajectory** = the message history that accumulates as the Agent works: user messages, assistant messages (with reasoning and tool calls), and tool results.
- On every LLM call, the complete context = **static prefix (system prompt + tool definitions) + trajectory (dynamic message history)** (Figure 1-4). Key fact: **Agent context = static prefix + trajectory**. From this complete context the LLM generates its next response, which is appended to the trajectory for the subsequent call.

Figure 1-4 — multi-currency aggregation task trajectory:
- Round 1:
  - user: "Calculate total annual revenue: Q1 $2.5M, Q2 €2.1M, Q3 £1.8M"
  - assistant.reasoning: "Need to convert EUR and GBP to USD, then aggregate"
  - assistant.tool_calls: convert_currency(2100000, "EUR", "USD"); convert_currency(1800000, "GBP", "USD")
  - tool (result): EUR→USD: 2,282,608.70; GBP→USD: 2,278,481.01
- Round 2:
  - assistant.reasoning: "Exchange rates obtained, call code interpreter to aggregate"
  - assistant.tool_calls: code_interpreter("total = 2.5M + 2.28M + 2.28M")
- Round 3:
  - assistant.content (final answer): "Total annual revenue $7,061,089.71, quarterly average $2,353,696.57"
- Key features: context accumulation — full history seen each round; structured trajectory — user / assistant / tool cleanly separated.

**Minimal running skeleton** (Python-style pseudocode; the Model only decides the next step, the Harness assembles context and validates and executes tools, the Environment produces actual state changes and observations; not runnable, not tied to any SDK — concrete executable code is in the book's companion repository):

```
trajectory = [user_request]
repeat:
    context = stable_prefix + trajectory
    decision = Model(context)
    trajectory.append(decision)
    if decision has no tool call:
        return decision.answer
    for call in decision.tool_calls:
        # independent calls may run in parallel
        validated_call = Harness.validate(call)
        observation = Environment.execute(validated_call)
        trajectory.append(observation)
```

**Trajectory structure example** (pseudocode):
```
trajectory = [
  {role: "user", content: "Based on the company's quarterly revenue: Q1 2.5M USD, Q2 2.1M EUR, Q3 1.8M GBP, Q4 380M JPY, calculate the company's total annual revenue and average quarterly revenue"},
  # First iteration - LLM receives the above trajectory and generates a response
  {role: "assistant", reasoning: "Need to convert all currencies to USD...", content: "",   # No direct reply to the user
   tool_calls: [
     {name: "convert_currency", args: {amount: 2100000, from: "EUR", to: "USD"}},
     {name: "convert_currency", args: {amount: 1800000, from: "GBP", to: "USD"}},
     {name: "convert_currency", args: {amount: 380000000, from: "JPY", to: "USD"}}
   ]},
  # Agent framework executes tools, adds results to trajectory
  {role: "tool", content: "EUR->USD: 2282608.7"},
  {role: "tool", content: "GBP->USD: 2278481.01"},
  {role: "tool", content: "JPY->USD: 2541806.02"},
  # Second iteration - LLM receives the complete trajectory, including tool results
  {role: "assistant", reasoning: "Conversion results obtained, now need to aggregate and calculate...", content: "",
   tool_calls: [{name: "code_interpreter", args: {code: "total = 2500000 + 2282608.7 + ..."}}]},
  {role: "tool", content: "Total: $9,602,895.73, Average: $2,400,723.93..."},
  # Third iteration - LLM receives the complete trajectory and generates the final answer
  {role: "assistant", reasoning: "All calculations complete, summarizing results...", content: "FINAL ANSWER: Total revenue $9,602,895.73..."}
]
```
- Note: the system prompt and tool definitions are NOT shown in the trajectory — they serve as the static prefix, automatically prepended before each LLM call.
- Real experiment: round 1 — Agent analyzed the task and called three currency conversion tools in parallel; round 2 — fed conversion results to a code interpreter for the more computationally intensive calculation; round 3 — confirmed all calculations complete, produced the final answer. A complex multi-step task completed in **3 iterations and 4 tool calls**.
- Properties: context is continually appended to; every LLM call sees the complete trajectory, so the model knows which stage it is in, what was tried before, and the outcome — the Agent keeps a global view of the task, like a person reviewing and summarizing while solving a problem. Because the trajectory is structured (user / assistant (reasoning + tool calls) / tool results cleanly separated), the system is highly interpretable and debuggable.

**## Experiment 1-2** ★: Kimi K3 Native Agent Capability
- Demonstrates the native Agent capability of Kimi K3, an example of the "Model as Agent" paradigm.
- Kimi K3 is a **Mixture of Experts (MoE) model with approximately 2.8 trillion parameters**. MoE can be viewed as a team of experts: for each kind of problem, the system activates only the few experts best suited to it rather than the entire model — preserving capability without paying the full efficiency cost.
- Kimi K3 has a **1 million token context window**, native visual understanding, and an always-on "thinking mode."
- Through reinforcement learning it has internalized the tool-calling decision policy as a native capability: when to call a tool, which tool, what arguments — all decided by the model, allowing autonomous tasks such as web searches.
- Precision note: what is internalized is the *when and how to call* decision; the tools themselves (e.g., web_search and code_runner) still execute server-side as API-level built-in tools. Kimi runs these official tools through a server-side script engine called **Formula**.
- Key observations: the model decides when to search and what to search for (genuine autonomy); it adjusts strategy as search results arrive and judges whether it has enough information.
- Common misconception cleared up: **RL gives the model the decision policy, not the tools themselves.** RL teaches when to call a tool, which to choose, what arguments to pass, whether to continue after a result, and how to chain dozens or hundreds of calls into coherent reasoning — these whether-and-how-to-use judgments get written into the model's weights. The tools and their execution (implementations of web_search and code_runner, the code sandbox, and the infrastructure issuing calls and returning results) all live outside the model. RL optimizes the decision policy; it does not embed a search engine or code sandbox into the weights. Thus the orchestration loop has not disappeared; it moved from the client to the server, while decision-making moved into the model. (Footnote a: thanks to reader asdlem for pointing out and clarifying, via GitHub Issue #30, the distinction that what RL internalizes is the tool-calling decision policy, not the tool execution mechanism — github.com/bojieli/ai-agent-book/issues/30.)
- Notable advantage: **stability of long-chain tool calls — 200–300 consecutive tool calls with coherent reasoning**, far beyond the few dozen calls at which most models begin to degrade.
- K3 is optimized for long-horizon programming and Agent workloads. Two variants: **K3 Max** (dialogue and Agent tasks) and **K3 Swarm Max** (large-scale parallel processing).
- As an open-source model, it matches top-tier closed-source systems on software engineering and Agent benchmarks — evidence that RL can endow a model with native Agent capability.

**## Experiment 1-3** ★: GPT-5.6 Native Deep Research Capability
- Uses OpenAI GPT-5.6 to show how an advanced model, backed by API-level built-in tools, closes the "search—read—analyze" orchestration loop on the server side for Deep Research.
- Feature: **Freeform Tool Calling**. Traditionally, a model calling a tool must serialize every parameter into strict JSON (a structured data format), like filling out a form with rigid formatting rules. Freeform tool calling (declared in the API through a tool of `type: "custom"`) lets the model send raw text straight to the tool (a snippet of Python code, a SQL query), avoiding JSON escaping entirely. This is an evolution of the API's parameter format, NOT an innovation in model architecture — the client's tool-calling loop (detect tool_calls → execute → return the result) stays the same; only the arguments change from a JSON string to raw text.
- GPT-5.6 + the Responses API's web search and code interpreter built-in tools delivers the core mechanism of Deep Research: autonomous web search for real-time information and code writing for in-depth analysis — an iterative research process of "search → read → analyze → search again."
  - Example A: "What is the shortest distance between the capitals of the 10 ASEAN countries?" — GPT-5.6 automatically searches geographic coordinates of each capital, writes Python to calculate great-circle distance between all pairs of capitals, identifies the closest pair.
  - Example B: "Search for Bitcoin's trend over the past month and perform technical analysis" — fetches real-time price data from multiple financial data sources, uses professional technical analysis libraries to calculate moving averages, RSI, MACD, and other technical indicators, generates visual charts, and provides trading recommendations.
- More importantly, GPT-5.6 internalizes the **intent clarification** design philosophy of the OpenAI Deep Research product at the model level. Given a research request, it does not start executing immediately; it first clarifies the user's true intent through a series of questions. For the Bitcoin task it would first ask: "Which data source do you prefer? Which technical indicators would you like analyzed?" This produces more precise reports better aligned with what the user actually needs.
- GPT-5.6 is a mature "Model as Agent" example: web search, the code interpreter, and other built-in tools of the Responses API execute in a closed loop on the server; the orchestration loop moves from client to API server, simplifying the client. The model still emits standard tool calls; the client simply no longer has to build the "search—read—analyze" orchestration framework itself. Most noteworthy: the intent clarification mechanism — rather than executing immediately, the model first confirms what the user really needs, then formulates a research strategy. The gap between "what the user said" and "what the user actually wants" is addressed before execution begins.
- Reproduction note: not tied to one vendor. Providers with equivalent managed tools work too, e.g., Alibaba Cloud Bailian's **qwen3.7-plus** Responses API (built-in web_search and code_interpreter) and Kimi K3's **Formula-managed search and code_runner**.

Figure 1-5 — "Model as Agent" Architecture (Native Tool Calling):
- LLM (Kimi K3 / GPT-5.6): native agent capabilities after RL training; RL training signal feeds it.
- Native tools: web_search, code_interpreter, more tools... (server-side).
- Example flow ("User: search the Bitcoin price trend over the last month"): Thought: search real-time data, then analyze it with code → Round 1: call web_search "BTC price last month" → Round 2: code_interpreter → Result: [price data] $67,230 → $71,450 → Final output: technical analysis report + visualization chart. ReAct loop executed closed-loop by the server-side Harness; carry the observation into the next round.
- Differences from traditional frameworks: ✓ Orchestration hosted by the server-side Harness ✓ No hand-written ReAct loop on the client ✓ The model decides tool calls itself.

## 1.2 Harness Engineering: Competitiveness Beyond the Model

- An Agent works at its core via an LLM running the ReAct loop, guided by context, using tools. The experiments above show the basic mechanism works — and also how fragile it is: the model may hallucinate (invent tools or parameters that do not exist), pick the wrong tool, or fail to recover from an error. Between a working demo and a reliable product lies a substantial gap; Harness Engineering exists to fix exactly those fragilities.
- First half of the chapter answered *what an Agent is*; the second half answers *how an Agent runs reliably in production*.
- Second, implementation-level view of the same system: treat the LLM as one core component (the **Model**), call all supporting code built around it the **Harness**. The two views are not rivals; they describe the same system at different abstraction levels. The more general word "Model" is used because Harness Engineering principles apply to any model that can reason and call tools.
- Core of the Harness = the original formula's "Context + Tools," plus three layers of safeguards: **Constrain** (what the Agent may and may not do), **Verify** (whether it did the thing correctly), **Correct** (how to recover when it did not).

Expanded as equations, the complete production-grade composition:
```
Agent = Model + Harness
Harness = Context management + Tool interfaces + Constrain + Verify + Correct
Agent ↔ Environment
```
- A minimal demo needs only a Model and a Harness that can construct context and expose tools; a production system must add constrain, verify, and correct within that same boundary. Example: a refund Agent places its policy in context, constrains calls with permission and amount rules, verifies the result against database state, and retries or falls back after a timeout.
- Precise definition: the Harness is not everything outside the model — it is the runtime and governance layer **inside the Agent boundary and outside the Model**. It mediates the Model–Environment interaction but does not include the Environment itself. Tool definitions, call adapters, sandbox permissions and reset mechanisms belong to the Harness; files and processes changing inside the sandbox, external databases, web pages, users, and the physical world belong to the Environment. Deployment location does not change this conceptual boundary.

**The five Harness functions table**

| Function | One-Sentence Responsibility / Core Principle | Practical Example | See Chapter |
|---|---|---|---|
| **Context** | Provides the model with relevant information; Information Sufficiency: ensure the Agent makes decisions based on sufficient information at every decision point | System prompts, knowledge bases, Agent status bars, Sidecar bypass queries | Chapters 2 & 3 |
| **Tools** | Provides the model with action interfaces; Clear Interface: tool names intuitive, parameters have examples, boundaries explained | MCP tools, code interpreter, search tools | Chapter 4 |
| **Constrain** | Sets behavioral boundaries — what can and cannot be done; Fail-Safe Defaults: all capabilities off by default and explicitly enabled (like mobile app permission management) | In Claude Code, every tool requires user authorization by default before execution | Chapter 4 |
| **Verify** | Automatically judges correctness of tool execution results; Input Isolation: security checks only look at structured data (e.g., JSON fields returned by tools), not free-form text generated by the model (attackers might manipulate model output through prompt injection) | Linter checks, type systems, tool call result validation | Chapters 5 & 6 |
| **Correct** | Automatically recovers or rolls back when problems are found; do not expose intermediate states until a failure is confirmed unrecoverable (e.g., silently retry a failed tool call instead of showing a half-finished result) | Silent retries, continuation generation, fallback to human judgment upon consecutive failures (circuit breaker mechanism) | Chapters 2 & 5 |

**Basic model control loop** (pseudocode):
```
observation = Environment.observe()
trajectory = [observation]
while true:
    actions = Model(Harness.build_context(trajectory))
    if len(actions) == 0:
        break
    allowed_actions = Harness.constrain(actions)
    observation = Environment.apply(allowed_actions)
    if not Harness.verify(Environment):
        observation = Harness.correct(Environment)
    trajectory.append(allowed_actions, observation)
```
- This skeleton deliberately omits implementation details: complete API message loop in Chapter 2; tools and automatic verification in Chapters 4 and 5.
- Context and Tools let the Agent complete tasks; Constrain, Verify, and Correct make sure it does so reliably and safely — not apart from Context and Tools, but as the engineering keeping them working reliably in production.
- Along the maturity curve, emphasis shifts: **early Agent frameworks focused on Context and Tools** (give the model tools and context, let it complete tasks); **production-grade systems shifted their center of gravity to Constrain, Verify, and Correct** (safe tool calls, managed context, recoverable errors).
- **Claude Code example**: the vast majority of its Harness code does Constrain, Verify, and Correct, not Context and Tools — the tools themselves (file read/write, command execution, search) are only a small part; the safeguards around them are the true core, including:
  - **Process State Management**: tracks which step the Agent is currently executing
  - **Multi-Layer Context Compression**: automatically prunes information when there is too much
  - **Permission Classification**: controls which operations require user confirmation
  - **Circuit Breaker**: automatically stops retrying after repeated errors so one failing operation does not cascade through the whole system
  - **Error Recovery Mechanisms**: catches exceptions, rolls back to the last stable state, retries, or hands off to a human
- The industry is shifting from *task completion* to *reliable task completion*, making Harness Engineering the core competitive advantage of Agent systems.

### 1.2.1 From Prompt Engineering to Loop Engineering: The Evolution of Engineering Paradigms

- **Prompt Engineering** — first wave: improving output quality by refining the natural-language instructions fed to the model.
- **Context Engineering** — second wave: optimizing the prompt alone is not enough; all information the model can see (system instructions, tool definitions, conversation history, external knowledge) must be managed systematically.
- **Harness Engineering** — third wave: widens the view from "what information the model receives" to "what kind of system the model runs in" — all infrastructure outside the model: constraint mechanisms, verification methods, feedback loops, error recovery.
- **Loop Engineering** — next: widens the view from a single run to sustained autonomous operation across runs — who discovers the next piece of work, when to verify, when the task counts as truly done. (Chapter 10 develops this alongside multi-agent collaboration systems.)
- July 2026: the industry began using **Graph Engineering** for a higher-level orchestration perspective — organizing Agent loops, deterministic programs, and human approvals into an explicit execution graph, where nodes provide capabilities, edges define routing and dependencies, and structured state travels along edges and is persisted at key boundaries.
  - Footnote 3: Josh C. Simmons used the name explicitly in his July 4, 2026 article "We Are Entering the Graph Engineering Phase," summarizing it in terms of nodes, typed edges, and checkpointed state. On July 18, Peter Steinberger's question about whether the discussion had shifted from loops to graphs helped the name spread further. The practices predate the label: the official documentation for LangGraph, Microsoft Agent Framework, and Google ADK describe them as graph orchestration or graph-based workflows.
- The five stages are NOT replacements but **nested layers**: Prompt Engineering ⊂ Context Engineering ⊂ Harness Engineering ⊂ Loop Engineering. Each layer widens the engineer's scope of concern and influence. As models converge in capability and stop being the decisive differentiator, competitive advantage shifts to the engineering outside the model.
- Recent evidence — **Terminal Bench 2.0** (benchmark evaluating an Agent's ability to complete complex tasks in a terminal environment): LangChain's Coding Agent improved from **52.8% to 66.5%** (jumping from outside the top 30 to the top 5 on the leaderboard). What changed was not the model but the Harness — having the Agent check its own execution results, detect when it was stuck in a repetitive loop, and refine its reasoning strategy.

### 1.2.2 Core Principles for Building Effective Agents

Based on Anthropic's experience, successful Agent systems follow three core principles:
1. **Keep it simple**: start with the simplest solution, add complexity only when truly necessary. Direct API calls are preferable to complex frameworks; clear code is preferable to clever abstraction — every extra layer of abstraction is a new blind spot during debugging.
2. **Keep it transparent**: show the Agent's planning steps, execution logs, and decision trajectory clearly. Not just a debugging convenience — a precondition for user trust; an error inside a black box is hard to locate or fix from outside.
3. **Design a well-structured tool interface (ACI, Agent-Computer Interface)**: designing the interface from the Agent's perspective — easy for the Agent to understand and use — rather than from the programmer's perspective as in traditional APIs. Tool names and parameters should be intuitive; wherever misuse is likely, the design should make the mistake impossible from the start: a SIM card's notched corner lets it slide into the tray in only one orientation; a microwave refuses to heat while its door is open. Manufacturing calls this "designing errors out" — **Poka-yoke**, a term from the Toyota Production System. A poorly designed tool can cause even the strongest model to fail repeatedly: the interface is the only channel between model and tool, and a vague interface gets amplified into systemic error.
- The next three sections address three freestanding but important Harness topics: model selection, orchestration patterns, and guardrails and safety. None belongs to the five Harness elements proper, but all are unavoidable in engineering practice.

### 1.2.3 How to Choose a Model

- The model is the foundation of the Agent's intelligence; choosing the right one often matters more than any amount of prompt tuning. Model releases move too quickly for specific version recommendations to stay useful, so this section offers directions instead.
- **Closed-Source Models**: the two most commonly used closed-source providers in current Agent development are OpenAI (GPT/o series) and Anthropic (Claude series). Generally lead in capability but are more expensive and constrained by the vendor's API policies. Do not rely only on leaderboards — evaluate on your own tasks (see Chapter 7).
- **Open-Source Models**: at the time of writing, lag closed-source models by no more than six months while costing substantially less. Pragmatic choice if the scenario does not demand the highest capability. Low-cost, support private deployment, allow fine-tuning customization — suitable for cost-sensitive scenarios or data compliance requirements. DeepSeek, Kimi, and GLM are among the stronger Chinese models for Agent capabilities. Models differ widely in tool-calling ability — test in your specific scenario before committing.
- **Beyond capability: policy boundaries.** A model may have the technical ability to perform a task without the product hosting it allowing users to invoke that ability. Vendors draw different policy boundaries around cybersecurity, model distillation, model extraction, private data, and high-risk operations; the same task may produce different outcomes in a chat product, a Coding Agent, and an API. Model selection therefore cannot compare only accuracy, price, and speed. Test on your real tasks whether the model is willing to proceed, whether the interface exposes the required capability, and whether the service terms permit the intended use. For business-critical tasks, prepare human handoff or another compliant model as a fallback.
- **Most Agents need a model that supports reasoning**: Agents make complex decisions — multi-step reasoning, tool selection — and non-reasoning models tend to perform poorly on them. Exceptions are few: a single simple step, or Computer Use GUI operations that amount to clicking a fixed position, where a non-reasoning model may suffice. The moment multi-step reasoning or dynamic decision-making enters, a reasoning model is essential.
- **Output speed and multimodal capabilities**: two easily overlooked dimensions.
  - Output token speed: Agents typically run many rounds of inference, each round must finish before the next starts, so output speed directly determines end-to-end latency — a 20-round Agent task running 2 seconds slower per round means an extra 40 seconds of waiting.
  - Multimodal support: if the Agent needs to understand images, audio, or video, multimodal capability is a hard requirement, and models differ widely here.

### 1.2.4 Orchestration Patterns: Workflow vs. Autonomous

- Orchestration patterns are how the Harness organizes its "context and tools" layer — how context flows between LLM calls, how tools are scheduled, whether the execution path is fixed in advance or generated dynamically. Agent orchestration evolved from simple to complex; each pattern has suitable use cases and trade-offs.
- Anthropic's experience with dozens of teams building LLM Agents: the most successful implementations rarely use complex frameworks; they use simple, composable patterns.
- Progress from simple to complex: start with a single LLM call — if better prompts and in-context examples solve the problem, do not build an Agent system. When multiple steps are needed and the task decomposes cleanly into fixed sub-tasks, use a workflow. Use an autonomous Agent only when you need dynamic decisions and a flexible execution path.
- Remember: Agent systems typically trade latency and cost for better task performance — evaluate carefully whether that trade is worth it.

#### 1.2.4.1 Workflow Pattern: Deterministic Orchestration

- A workflow orchestrates LLMs and tools through **predefined code paths**. Its execution path is deterministic and designed in advance by the developer — the behavior of each step and transition is defined in code; the LLM handles only the understanding and generation inside each node.
- Example: a flight-booking Agent as a workflow with four fixed nodes:
  1. **Verify User Identity** — call the identity verification API to confirm who the user is.
  2. **Search for Available Flights** — query the flight database based on user requirements.
  3. **Complete Payment** — call the payment interface to deduct the amount.
  4. **Confirm Booking** — call the booking API to lock the seat and send a confirmation to the user.
- An LLM can be used inside each node (e.g., understanding the user's travel needs in natural language), but the flow sequence between nodes is fixed by code — the system will not book a seat before payment is completed, nor search for flights before identity verification.
- Two core advantages:
  1. **Strict process control**: the developer can guarantee critical steps are never skipped or run out of order — business rules like "no booking before payment" are enforced by code, not left to the LLM's judgment.
  2. **Security**: because the execution path is deterministic, prompt injection or a model error can at most affect processing inside the current node; it cannot make the Agent jump to a branch it should not reach. The attack surface is confined to a single node.
- Main limitation: **lack of flexibility**. When an unanticipated event occurs — the user changes the booking during payment, or a flight is canceled and the system needs to recommend an alternative — the fixed path cannot adapt on its own; it can only follow a preset exception branch or hand control back to a human.
- Simplest workflow example: **text-to-image generation**. The user's need is usually a single plain-language sentence ("Draw me a scene of programmers at work after AGI is achieved"); text-to-image models like Stable Diffusion only accept prompts in a specific style — comma-separated English tags, quality words, negative prompts. So the workflow places two fixed nodes between the user and the image-generation model:
  1. **Prompt Rewriting** — use an LLM to rewrite the natural-language request into the format the text-to-image model expects. For the example, "programmers at work after AGI is achieved" is very broad, so the LLM must think carefully (e.g., "after AGI is achieved, programmers no longer need to write code, so the image should show a programmer sunbathing on a beach, directing AI employees through a brain-computer interface") and produce a concrete scene description.
  2. **Image Generation** — call the text-to-image model with the rewritten prompt to obtain the image.
- The execution path is hard-coded; the LLM node performs translation — converting human language into an input format the tool can understand — and it exists because text-to-image models "don't understand plain speech." Harness code that specifically patches a capability shortcoming of a tool (or model) like this can be called an **adapter layer**.
- But if the image-generation tool is replaced with a multimodal model having native image generation, such as **Nano Banana 2 or GPT-Image 2**, prompt rewriting is no longer needed — no matter how the user phrases the request, the model understands it on its own and produces the image directly.

**## Experiment 1-4** ★: Text-to-Image Workflow vs. Native Image Generation
- Method: send the same plain-language request through two routes. Workflow route: an LLM first rewrites the request into a Stable Diffusion-style prompt, then calls the text-to-image model to produce the image. Native route: send the sentence as-is to a multimodal model that supports native image generation (such as GPT-Image 2), producing the image in a single call.
- Compare two things: what the prompt-rewriting node turns the original request into, and which route's image stays closer to the original request. Worth comparing two categories of requests: one concrete (e.g., a poster with specified copy) and one broad (such as the AGI work scene) — for the broad category, the workflow route may still have advantages of its own.
- Takeaway: the parts of a Harness that patch the model's capability shortcomings will be internalized by the model as it grows stronger. Already happened several times in this chapter: few-shot examples and prompting tricks like "let's think step by step" were internalized by instruction tuning and reasoning models; output-format repair and JSON parsing tolerance were internalized by structured output and native tool calling; text-to-image prompt rewriting was absorbed by native multimodal understanding and generation capabilities. Each round of internalization eliminates adapter-layer code of the "translation" and "scaffolding" kind.

#### 1.2.4.2 Autonomous Agent: Runtime Decision-Making

- When the fixed path of a workflow is insufficient, an autonomous Agent is needed. Core difference: **the execution path is not predefined; it is determined at runtime by the Agent based on environmental feedback.**
- Flight example: an autonomous Agent needs no four predefined nodes. The user says "Book me a flight to Shanghai next Wednesday": the Agent decides the sequence dynamically — searches for flights, discovers login is required, verifies identity, resumes the search. If the cheapest flight has a layover, it can ask whether that is acceptable; if the user says no, it adjusts the search criteria.
- An autonomous Agent must plan for itself — choose its own execution steps — and recognize failure and change strategy rather than simply halting on error.
- **Autonomy is not unbounded**: explicit stopping conditions must be designed in (task complete, maximum iterations reached, unrecoverable error hit), or the Agent can enter infinite loops or continue executing after the task is already done.
- Implementation view: an autonomous Agent is essentially an LLM using tools in a loop, continuously obtaining environmental feedback — i.e., the ReAct loop. Common exit conditions: calling a final output tool, the model returning a response without any tool calls, encountering an error, or reaching the maximum number of rounds.

Figure 1-6 — execution loop of an autonomous Agent:
- ① Think (Reasoning): "Need more info" → ② Acting: web_search(...) → ③ Observing: tool_result: "..." → check decision criteria: "Exit condition met?"
  - Continue the loop if No; Return final result if Yes.
- Exit conditions (any one): ① Task complete ② final_answer called ③ No tool call ④ Error limit exceeded ⑤ Max rounds reached.

- Well suited to **open-ended problems** where the number of steps is hard to predict. Typical use cases: Coding Agents solving SWE-bench (Software Engineering Benchmark — evaluates an Agent's ability to automatically fix real GitHub issues) tasks; "Computer Use" Agents operating computer interfaces like a human; research tasks requiring iterative search and analysis.
- Autonomy costs more and lets errors compound. Deploying one demands thorough testing in a sandbox, appropriate guardrails and monitoring, and human-in-the-loop checkpoints at critical decision points.

#### 1.2.4.3 Choosing and Mixing the Two Patterns

- Workflows and autonomous Agents are not mutually exclusive — many systems mix the two: critical processes with strict compliance requirements run as workflows for reliability, while parts needing flexible decisions switch to autonomous mode.
- **n8n** is a mature open-source workflow automation framework in which developers build Agents by arranging functional components on a visual canvas — workflow nodes and autonomous Agent nodes can coexist in the same system. (Figure 1-7: n8n workflow editor interface.)

#### 1.2.4.4 Brief Comparison of Mainstream Agent Frameworks

| Framework/Platform | Core Positioning | Orchestration Pattern | Development Approach | Suitable Scenarios |
|---|---|---|---|---|
| **Codex Harness** | Open-source Agent runtime behind Codex | Autonomous | Code-first, embeddable in your own app | Coding Agents, embedding an Agent into your own product |
| **Claude Agent SDK** | Production-grade Agent development framework | Autonomous | Code-first | Complex autonomous tasks, Coding Agents |
| **LangChain / LangGraph** | General LLM application framework | Workflow + autonomous | Code-first | Complex reasoning chains, multi-step workflows |
| **n8n** | Visual workflow automation | Workflow + autonomous | Low-code | Business automation, nontechnical teams |
| **Dify** | LLM application development platform | Workflow + conversational | Low-code + API | Enterprise RAG, knowledge-base applications |
| **CrewAI** | Role-based multi-Agent orchestration | Multi-Agent collaboration | Code-first | Team-style task decomposition and execution |
| **OpenClaw** | Open-source all-purpose personal Agent | Autonomous + event-driven | Configuration + code | Personal assistants, Deep Research, Computer Use, multiplatform messaging |
| **DeepSeek Harness** | Agent self-evolution framework | Everything is a plugin | Code-first, easy to customize | Agent developers, researchers |
| **Pi** | Minimal Coding Agent framework | Autonomous | Code-first, easy to customize | Agent developers |

- Clarification on the first two rows: **Codex** is OpenAI's Coding Agent product (App, CLI, IDE extension); the **Codex Harness** is the runtime layer driving all of these forms. It offers three integration paths: `codex exec` — one-off tasks in scripts and CI; the **Codex SDK** — third-party application code that starts, resumes, and streams tasks; and the **app-server** — persistent sessions, event streams, and approval callbacks over the JSON-RPC protocol, for building an Agent directly into a product. **Claude Agent SDK and Claude Code** stand in a similar relationship, except that what Claude opens up is the SDK interface — the Harness implementation itself is not open source. (Footnote 4: OpenAI, "Codex as a platform: build on the open agent harness", August 2026.)
- Frameworks evolve rapidly; some may be obsolete by the time you read this and new ones popular. Learning one particular framework's API is therefore not important. The key question when choosing a framework is not its sophistication, but whether its abstraction is thin enough to let you focus on business logic.
- Orchestration patterns solve the organization of context and tools within the Harness. But task completion is not enough — tasks must also be completed correctly and safely. Next: guardrails, the main way constrain, verify, and correct are implemented in practice.

### 1.2.5 Guardrails and Safety

- Guardrails are how the "constrain, verify, and correct" layer of the Harness is primarily implemented — a layered defense keeping Agent behavior safe and controllable.
- Well-designed guardrails help manage **data privacy risks** (e.g., preventing system prompt leakage) and **reputational risks** (e.g., keeping model behavior consistent with the brand). Start with guardrails for risks you have already identified, then add new ones as new vulnerabilities surface.
- Think of guardrails as **defense in depth**: no single guardrail is likely sufficient on its own, but several specialized ones combined make a far more resilient Agent system.
- Another failure mode: **false refusal**. To reduce the chance of allowing dangerous requests, a model may also reject legitimate but sensitive-looking work — authorized security testing or model distillation research. Guardrail evaluation should therefore test not only whether prohibited requests are blocked, but also whether clearly permitted requests can still be completed.

#### 1.2.5.1 Types of Guardrails

Guardrails sit at three layers: **context layer, execution layer, data layer**. Ordered not by where they sit in the request lifecycle but by how hard they are to bypass — the lower the layer, the less it depends on the model's own judgment, and the harder it is for a single successful attack to get through. Every security discussion later in the book hangs on this tree.

**Context-layer guardrails** govern what the model gets to see, intercepting content before it enters context. Usually four mechanisms:
- **Relevance classifier**: flags off-topic queries — e.g., a coding assistant asked "how tall is the Empire State Building?"
- **Safety classifier**: detects jailbreaks (inducing the model to bypass its safety limits) and prompt injection (embedding malicious instructions in the input). Key difference: a **jailbreak** is the user trying to get around the model's own limits; **prompt injection** is an attacker manipulating the model indirectly through external data such as web pages or documents.
- **Content moderation**: flags harmful or inappropriate input such as violent or discriminatory content.
- **Rule-based protection**: deterministic measures — blocklists, input length limits, regular-expression filters — against known threats such as SQL injection.
- Source labelling and the separation of "instructions" from "data" also belong to this layer; Chapter 2 develops them.
- Representative practice: **Anthropic's Constitutional Classifiers**. Three key design elements:
  1. **Rule-driven training**: rules written in natural language — explicitly specifying what is allowed and what is not — generate synthetic training data for the input and output classifiers.
  2. **Joint contextual judgment**: the new generation checks the user's question and the model's answer together — some answers look perfectly fine on their own (e.g., "how to use food flavorings") and only against the question does it become clear that "food flavorings" is code for chemical reagents.
  3. **Two-stage screening**: an extremely lightweight probe — which reads the model's internal activations at almost zero cost — checks every conversation first; anything suspicious is escalated to a more powerful classifier for review rather than refused outright. The first stage can tolerate more false positives without hurting user experience, and overall cost is greatly reduced.
  - (Footnote 5: Anthropic, "Next-generation Constitutional Classifiers: More efficient protection against universal jailbreaks", 2026; paper: Cunningham et al., "Constitutional Classifiers++: Efficient Production-Grade Defenses against Universal Jailbreaks", arXiv:2601.04603.)
- This layer has a structural ceiling: an Agent sitting inside the very context under attack can hardly tell whether it has already been injected. The context layer can lower attack success rate but cannot offer a guarantee — hence the two layers below.

**Execution-layer guardrails** govern what the model gets to do, validating an action before it takes effect.
- At their core is **tool risk rating**: each tool is labelled low, medium, or high risk according to **reversibility, privilege level, and financial impact**; high-risk operations require additional review or human confirmation.
- That review must be performed by a mechanism **outside the context** — an independent review process, least-privilege credentials, sandbox isolation, a human in the loop — otherwise it falls together with the injected Agent.
- The reply returned to the user is itself an action (Chapter 4 classifies it as a user-communication tool), so output checks belong here too: a **PII filter** screens output for personally identifiable information such as ID or phone numbers to prevent unnecessary exposure; **output validation** checks content to keep replies aligned with brand values.

**Data-layer guardrails** govern what the world can ultimately be changed into, delegating "who may do what to which record" to a stable, human-reviewed mechanism: **row-level security policies in the database, constraints and validators, controlled views and stored procedures, and an access context bound by a trusted runtime that cannot be forged**.
- Value: does not depend on the two layers above being correct — even if prompt injection succeeds and generated code omits its permission checks entirely, the unauthorized operation is still rejected at the data layer. Chapter 5 develops this layer through the example of dynamically generated software.

#### 1.2.5.2 Human Intervention

- **Human-in-the-loop intervention** is a key protective measure: it lets an Agent improve real-world performance without degrading the user experience. Matters most in early deployment, when it helps identify failure modes, surface edge cases, and establish a robust evaluation cycle.
- With human-in-the-loop, an Agent that cannot complete a task can hand over control gracefully — in customer service, escalating to a human representative; for a Coding Agent, handing control back to the developer.
- Two main situations trigger human intervention:
  1. **Exceeding Failure Thresholds**: set caps on the Agent's retries and operations. If exceeded, escalate to a human.
  2. **High-Risk Operations**: sensitive, irreversible, or high-risk operations should trigger human oversight — at least until the team has built enough confidence in the Agent's reliability. Typical examples: authorizing a large refund, processing a payment.

### 1.2.6 The Five Harness Elements and the "Building" Part

- Relationship between the two formulas — the book has exactly **one structural skeleton**, the one the introduction and afterword keep using: **Agent = LLM + Context + Tools** — chapters 2–6 build, chapters 7–9 evaluate and evolve, chapter 10 collaborates.
- **Agent = Model + Harness** is not a rival partition but the same thing unfolded into its production form: it expands "context" and "tools" into five responsibilities — context management, tool interface, constraints, verification, correction. It is a lens inside the "building" part, not a table of contents covering all ten chapters.
- The five Harness elements map cleanly onto chapters 2–5:

| Harness Focus | Corresponding Chapter | Core Content | Security Concerns |
|---|---|---|---|
| Context Design | Chapter 2 (Context Engineering) | Prompt engineering, Agent status bar, context compression, Agent Skills | Prompt injection and information leakage |
| Context Expansion (Knowledge Persistence) | Chapter 3 (Knowledge Base) | User memory, RAG, structured indexing, agentic RAG | Sensitive information exposure, privacy protection |
| Tool Design and Security Constraints | Chapter 4 (Tool Design) | Tool classification, permission control, MCP standard, asynchronous architecture | Misoperation, unauthorized access, irreversible operations |
| Tool Verification and Correction | Chapter 5 (Code Generation) | Coding Agent's Harness, test-driven development, codified rules | Identity impersonation, responsibility attribution |

- Chapter 6 (Interaction) does not belong to any of the five elements; it expands the modality and timing of the observation and action spaces themselves. Chapters 7–9 ask how we know the Harness was built right and how to keep making it better. Chapter 10 replaces a single Agent's Harness with a collaboration structure among several. Forcing those chapters into the five boxes only makes the boxes stop discriminating.
- Security is not partitioned by chapter; it is a **cross-cutting concern** running through the whole book, organized by the three guardrail layers (context, execution, data). The "security focus" column gives each chapter's principal landing point among those three layers.
- **Anthropic's practice** building long-running Agents shows how Harness design solves problems the model cannot: split complex tasks between an **Initialization Agent** (setting up the environment, decomposing the task list) and an **Execution Agent** (making incremental progress each session, leaving clear handover artifacts), using a structured Harness to tackle the two failure modes of long tasks: running out of context and declaring the task done prematurely.
- The chapters ahead work through the Harness component by component — Chapter 2 begins with the most central one, context engineering; Chapter 5 lays out the complete practice of Harness engineering in Coding Agents.

## 1.3 Design Patterns That Run Through the Book

Five design patterns are named and defined canonically once here:

1. **Proposer-Reviewer**: production and judgment are carried out by two roles that do not share a context; the judge sees the artifact itself — the rendered result, the test output, the structured call arguments — rather than the producer's reasoning. Premise: self-review is unreliable — a model inside a given context can neither think of what it failed to think of, nor readily tell whether it has already been injected.
   - Uses: Chapter 3 updates knowledge; Chapter 4 pre-approval and post-validation of tool calls (the Sidecar is a read-only variant); the PPT, video and log experiments of Chapter 5; Chapter 7 evaluates UIs; Chapter 9 reviews update proposals; Chapter 10 discusses its shape in peer collaboration, and why an Agent must not review itself.
2. **Progressive Disclosure**: rather than putting everything into context at once, offer a searchable catalogue first and load details on demand. Optimizes two things simultaneously: the **context budget** and **selection accuracy**.
   - Uses: Agent Skills in Chapter 2 is the archetype (metadata resident, body loaded on demand); the layered retrieval of Chapter 3, the proactive tool discovery and paginated truncation of Chapter 4, and Agent discovery in Chapter 10 are variants.
3. **Append-only**: state evolves by appending; what has been written is never revised in place. Buys **cacheability, replayability, and auditability**.
   - Uses: the KV Cache prefix stability of Chapter 2 is its performance form — the earlier a change lands, the more cache it invalidates; the event-shaped memory of Chapter 3 and Chapter 4's habit of appending a newly discovered tool schema to the end of the trajectory rather than splicing it back into the prefix follow the same discipline.
4. **Boundary Set + Retention Set**: every change must be validated both on "the samples it is supposed to change" and on "the samples it must not affect." Testing only the former mistakes overfitting for progress; testing only the latter mistakes an ineffective change for a safe one.
   - Uses: the regression tasks of Chapter 7, the training/evaluation isolation of Chapter 8, and the update-proposal validation of Chapter 9 all rest on this pair of sets.
5. **Minimal Diff, Reversible**: keep each change as small as possible, carrying its provenance, and independently revertible instead of rewritten wholesale. This is what makes **attribution** possible — when something breaks, it can be traced to one specific change.
   - Uses: the knowledge updates of Chapter 3, the code patches of Chapter 5, and the prompt and program updates of Chapter 9 follow it; the three update paths from the start of the chapter (in-context adaptation, external-artifact updates, parameter updates) are themselves ordered from most to least reversible.

## 1.4 Chapter Summary

- **Agent = Reasoning Engine + Working Context + Action Interfaces**: the LLM provides reasoning and decision-making, context supplies the working set of information available at decision time, tools provide the action interfaces. None of the three is dispensable.
- **Expanding Context and Tools Is the Primary Capability Lever**: once the model is fixed, redefining or enlarging the observation and action spaces — expanding context and tools — can often turn an unsolvable task into a solvable one directly. The Manus-to-OpenClaw evolution shows that much of generality comes from broadening the interface boundary; that expansion must remain on-demand and be paired with permissions and verification.
- **Context Is the Decisive Factor**: context consists of a static prefix (system prompt + tool definitions) and a dynamic trajectory (message history). Ablation shows the components are not equivalent: removing tool definitions or tool results takes away the ability to act or to close the loop outright, while the cost of removing the other two depends on whether that information can be reconstructed from the current observations. The essence of the ReAct loop is appending to the trajectory, over and over, so the model keeps advancing the task.
- **Harness Is the Competitive Advantage**: model capability is commoditizing; the real differentiator is the Harness — the constrain, verify, and correct mechanisms built around context and tools that enable reliable task completion. In production-grade Agent systems, the vast majority of Harness code goes into these safeguards, not context and tools alone.
- **From Workflow to Autonomous Agent**: prompts first, then workflows, autonomous Agents last — that ordering is the most practical way to reduce unexpected behavior. Every orchestration pattern has situations where it fits; no single pattern is best everywhere.
- **Five design patterns run through the book**: Proposer-Reviewer, Progressive Disclosure, Append-only, Boundary Set + Retention Set, and Minimal Diff + Reversible.
- **Security Is an Architectural Issue**: security must be considered from the first line of code, not patched on before launch. Guardrails are divided by difficulty of bypass into context, execution, and data layers; all later security discussions use this structure.
- Next chapter examines the Harness's most central component in depth: context engineering. Chapter 8 covers the Agent concept's academic roots in reinforcement learning and compares traditional RL with modern LLM Agents.

### Deep Thinking: Thought Questions

The thought questions below take the chapter's core concepts a level deeper; they do not have standard answers.

1. ★★ If you could only add one capability to an Agent system — a stronger model, richer context, or more tools — which would you choose? Under what conditions would your choice change?
2. ★★★ In a ReAct loop, cumulative cache reads grow approximately quadratically with the number of rounds. How can this growth be reduced?
3. ★★ The "Model as Agent" paradigm means models are becoming more autonomous in tool-calling decisions. However, this chapter argues that the importance of Harness engineering is actually increasing. How can these two trends coexist? Where does the future core value of Agent frameworks lie?
4. ★★ In the ablation experiment, the absence of "tool result feedback" makes the Agent retry until it exhausts its iteration budget. In a production environment, besides missing tool results, what other situations could push an Agent into this kind of loop? What detection and termination mechanisms would you design?
5. ★ This chapter analyzed five Agent products along three dimensions: working context, action interfaces, and strategy. Pick an AI product you use daily, analyze it along the same three dimensions, and judge whether its architecture is appropriate. If you were designing it, what would you improve?
6. ★★ If you were to design a customer service system specifically for booking flights, would you choose a workflow pattern or an autonomous Agent pattern? Is it possible to mix both patterns in the same system?
7. ★★★ The guardrails section mentioned tool risk ratings. If a tool is generally low-risk but becomes high-risk with specific parameter combinations (e.g., delete_file deleting a normal file vs. deleting a system file), how would you design dynamic risk assessment?
8. ★★ In the Agent product table in this chapter, all Agents have an "open-ended" action space. In what scenarios would a constrained action space (e.g., only being able to choose from predefined options) be superior to an open-ended one?
9. ★★ The human-in-the-loop intervention mechanism requires the Agent to "gracefully hand over control." However, in practice, the user might be offline, respond slowly, or give vague instructions. What should the Agent do in such cases?
10. ★★★ The introduction states that "good design principles should transcend model iteration cycles," but the concrete engineering methods used to implement those principles may become obsolete as model capabilities improve. Give an example of such an Agent engineering method and explain why.