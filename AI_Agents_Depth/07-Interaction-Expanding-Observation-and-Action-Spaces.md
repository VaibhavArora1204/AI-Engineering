# Chapter 6 Interaction: Expanding the Observation and Action Spaces

Thesis (from Ch1): when the underlying model is fixed, the most effective system-engineering lever for improving an Agent's task performance is usually to redefine or expand its observation space and action space. Chapters 2-5 cashed that out — context engineering decides what goes into the observation; memory and knowledge bases stretch observation across sessions; tools define what the Agent can do; code generation lets it create new actions of its own. All prior expansions shared one premise: the Agent and the world take turns speaking (user finishes a sentence → Agent thinks, calls a few tools, replies; while thinking, the world is assumed to stand still). The premise is so natural it is rarely written down. This chapter removes exactly that premise.

## 6.1 Two Axes: Modality and Timing

- **Modality** decides the form of observation and action: does the Agent only read text, or also hear sound, see the screen, sense torque; can it only emit tokens, or also speak, click, and drive joints.
- **Timing** decides the rhythm of observation and action: does the Agent go fetch an observation, or does the world push it; must an action finish within one turn, or may it span turns, be interrupted midway, and be preempted by something more urgent.
- Previous chapters expanded the *content* of the two spaces; this chapter expands *modality* and *timing*.

| Space | Content (Ch 2-5) | Modality (this chapter) | Timing (this chapter) |
|---|---|---|---|
| Expanding observation space | Context engineering, memory and knowledge bases | Voice, screen, physical sensors | The world pushes, continuous streams |
| Expanding action space | Tools, code generation | Speaking, clicking, joint motion | Across turns, interruptible, preemptible |

- A model's training corpus is almost entirely turn-based (question→answer, tool call→tool result, one speaker finishes before the other begins), so the policy a model learns assumes the world will wait for it. The real environment does not wait: mail arrives while it is thinking, the user cuts in mid-sentence, the page has already changed between two screenshots, and the cup is knocked over while the arm is reaching for it.

Scale table (observation/action change at each timing scale):

| Scale | Scenario | Change on observation side | Change on action side |
|---|---|---|---|
| Seconds — days | Async and event-driven | The world wakes the Agent (mail, timers, callbacks) | Actions span turns: start now, finish later on an event |
| 10 ms — 1 s | Voice | Listen while speaking, without waiting for a full sentence | Think while speaking, interruptible, revisable midway |
| Sub-second — seconds | Computer Use | The screen keeps changing between frames | After acting, reality must be re-confirmed against the plan |
| Milliseconds | Robotics | Sensors stream back continuously | Actions are chunked: plan a little at a time, preemptible |

## 6.2 Async and Event-Driven: When the World Comes Looking for You

- The perception, execution, and collaboration tools of Chapter 4 are all invoked proactively by the Agent. Responding to external events that may arrive at any time requires an **event-driven asynchronous architecture**. The two remaining tool classes from Chapter 1 — **event-trigger tools** and **user-communication tools** — depend on this architecture.
- Modality does not change in this section (still text); only timing changes — the first step out of the turn-based world of the previous five chapters.

### 6.2.1 Why Asynchrony is Needed

- Analogy: **synchronous** = "do one thing before you can do the next"; **asynchronous** = "multiple things can happen concurrently." A traditional synchronous Agent architecture is like a single checkout counter (one customer at a time); a truly intelligent assistant is like a flexible secretary — with multiple pending items (emails, phone calls, visitors) it decides which to handle first by urgency and can pause and switch to a more urgent task mid-way.
- In synchronous mode the Agent must either wait for a background task before talking to the user, or wait for the conversation to end before processing a newly arrived event. It cannot deliver three core capabilities:
  1. **Asynchronous execution is the norm** — many tasks require long runtimes and should not block user interaction.
  2. **Dynamic judgment of event priority** — not all events are equal; the Agent must choose handling strategy: cancel the current operation (urgent), add it to a queue (routine), or process in parallel (independent lightweight query).
  3. **Fluency in interruption and resumption** — an interrupted conversation/task should resume naturally.
- The asynchrony paradigm collides with a fundamental fact about current LLMs: **training assumes synchrony** (after a tool call, the next message must be the tool result), while **real deployment demands asynchrony** (users interrupt at will, tasks progress concurrently, external events arrive before a tool returns). This "synchronous training / asynchronous deployment" contradiction runs through every engineering trade-off in the section.
- Solution: an **event-driven asynchronous architecture**. The system no longer actively and repeatedly checks for "new messages" (polling = inefficient), but automatically triggers processing logic when a new message arrives. All inputs, outputs, thought processes, and external interactions are uniformly modeled as an **event stream** — a sequence of event records arranged on a timeline.

**Figure 6-1: Event-Driven Asynchronous Agent Architecture**
- Event sources: Email `on_email_reply {"from":"alice@...", "subject":"Re:meeting"}`; Timer `on_timer_expire {"task_id":"daily_report", "scheduled":"09:00"}`; Webhook `on_webhook {"repo":"agent-lib", "event":"pr_merged"}`; User `on_user_message {"text":"Check tomorrow's weather for me"}`.
- Event queue: `user.input` (Priority: normal); `email.reply` (normal); `user.interrupt` (Priority: urgent!); `timer.trigger` (normal).
- Agent processing flow: Fetch event → Router (LLM determines urgency) → Append to trace (structured event format) → LLM inference (Observe → Think → Act) → Tool execution (async/sync dispatch, result handling, notify/respond/store) → Loop.

### 6.2.2 Implementing Event-Driven Mechanisms in OpenClaw

- OpenClaw (open-source framework) receives multi-channel messages through a **Gateway control plane** and routes them to the Agent runtime. Three built-in event-driven mechanisms:
  1. **Hooks**: respond to events in the Agent's lifecycle (session creation, reset); similar to event triggers in GitHub Actions.
  2. **Cron** (scheduled-task scheduler): periodic tasks per cron expressions (Unix scheduled-task syntax; e.g., `0 9 * * 5` = 9 AM every Friday).
  3. **Heartbeat** (Heartbeat Daemon): wakes the Agent every N minutes to check whether anything requires attention.
- These three give OpenClaw Agents the appearance of autonomy — even with the user offline, the Agent can generate reports on schedule, check system status, handle routine chores. Gateway already handles built-in channels (IM, web interface) in push fashion. Only **Cron and Heartbeat let the Agent act without a user message**, and both are time-driven: Heartbeat at fixed intervals, Cron at preset times; Hooks originate inside the framework.
- **The real gap**: third-party event sources beyond built-in channels (new email, external API callback, urgent notification). OpenClaw has no immediate ingress path; the Agent may only notice at the next Cron or Heartbeat tick.
- **PineClaw example** (Pine AI's OpenClaw plugin): Pine AI makes real phone calls on behalf of the user — negotiating bills, canceling subscriptions, handling insurance claims. User may need to intervene during a call:
  - **Real-time Identity Verification**: representative asks to verify account holder's identity; Pine needs the user to immediately provide a security code or one-time password (OTP).
  - **Three-Way Call Confirmation**: representative asks to speak directly with the account holder; Pine needs the user to answer the phone within seconds.
  - **Progress Sync and Decision Confirmation**: at a critical negotiation point (e.g., a price reduction proposal), Pine needs the user to confirm acceptance.
- With Heartbeat's periodic polling the user might not be notified while the representative is still waiting for the verification code; the representative hangs up and the call fails.
- **PineClaw's solution**: a **Channel mechanism** establishing a real-time event path between OpenClaw's Gateway and the Pine API. When a call connects, needs user input, or ends, the message is pushed immediately to the Agent, which handles it and notifies the user.
- Core value: true "proactive service" requires not only periodic checking but **events actively notifying the Agent**. Unifying all inputs (user messages, tool returns, external callbacks, scheduled triggers) into an event stream and driving thinking/actions through an event loop is the architectural foundation.

### 6.2.3 Event-Triggered Tools

- Entry points through which external events drive an Agent's actions. Without them the Agent only loops: think, call tools, output a result, wait for the user's next input. Three common types:
  1. **Timers (`set_timer`)** — events tied to physical time. Example: unanswered email → follow up after a while to ask about progress; call placed outside business hours → retry during the next business window. OpenClaw and Claude Code let an Agent wake itself at a specified time.
     - **One-shot timers** handle tasks with a specific time (user asks Saturday: "call the bank's mortgage department for a status update" → timer triggers the call "next Monday at 10:00 AM").
     - **Recurring timers** handle periodic tasks (checking server health every hour); they also provide polling for external services that cannot push progress updates.
     - OpenClaw's **Heartbeat is a systematized version** of the timer mechanism — basis of its "proactive service" capability.
  2. **Background Task Monitoring (`monitor_shell`)** — events from asynchronously executing tools or command-line tasks. If the Agent "stares at the command line" polling repeatedly, it burns tokens; if it waits for full completion, it misses critical problems as they unfold; if the command hangs it cannot intervene at all, stalling the task. Claude Code adds a **monitor tool** allowing the Agent to monitor new command-line output, including output containing specific keywords.
  3. **External Event Channels (`connect_channel`)** — push external events (new emails, API callbacks, IM messages) to the Agent in real time. The PineClaw Channel mechanism is a typical implementation.
- Design guidance: event-triggered tools should define **clear trigger conditions and filtering rules** to prevent irrelevant events waking the Agent and wasting compute; the **event payload should contain sufficient context** to minimize additional queries after being woken up.

### 6.2.4 User Communication Tools

- Arise as communication channels between Agents and users diversify. Many Agents (Claude Code, Manus) use a **native ReAct loop**: everything the Agent "says" (assistant message) is sent directly to the user, who must open a specific session in the app; the session often exposes the Agent's tool-call process.
- **OpenClaw breaks this pattern**: users need not perceive sessions or follow tool-call details; both user and Agent can send messages at any time instead of alternating one request with one response. This gives OpenClaw a "human-like presence," communicating asynchronously like a secretary. Rather than raw assistant messages, OpenClaw uses **dedicated messaging tools** whose messages can include images and files and can trigger push notifications based on urgency.
- Growing **multimodal communication**: structured card messages, reminder emails; some Agents experiment with **Generative UI** — using HTML and similar means to generate interactive interfaces for friendlier presentation.
- Design level: support **asynchronous messaging** (user may be offline), **read/unread status tracking**, and keep messages **consistent across channels**.
- **Multi-channel user communication and re-engagement**: the notification mechanism also serves as a re-engagement mechanism. Sending extends to instant messaging, SMS, email, phone calls, push notifications. The Agent picks the channel from urgency, user status, content nature, and user preferences — important messages not missed, redundant interruptions avoided.
- **Long-running tasks** → notify the user on completion to bring attention back; **periodic tasks** (daily summaries, weekly reports) → notifications build a regular interaction habit.
- User communication tools solve "how to reach the user"; the identity the Agent assumes on channels and the environment it acts in require a layer of virtual identity and isolated execution-environment infrastructure (next section).

### 6.2.5 Virtual Identity and Isolated Execution Environment

- Chapter 4 opened with Samantha in *Her* as an example of an Agent using tools to interact with the real digital world. A general-purpose assistant forces a key architectural choice: manage the user's personal accounts directly, or hold a **virtual identity of its own**?
  - Direct management looks convenient, but one Agent error or compromise exposes the user's entire digital identity.
  - Safer: an **independent virtual identity** — the way a secretary has their own office phone and mailbox — comprising dedicated communication accounts, storage, and computing environments. The Agent works on the user's behalf under a transparent, clearly declared identity. Transparency does not weaken trust; it can make communication more authentic.
- Virtual identities need **isolated execution environments**: virtual computers (VMs/containers) and virtual phones (Android emulators) give OS isolation plus full desktop/mobile capabilities.
  1. A virtual computer can run around the clock regardless of whether the user's device is online, and without disrupting the apps the user is operating.
  2. An Agent error at worst crashes the virtual environment, not the user's real device.
  3. Isolation prevents the Agent from freely accessing the user's local files.
- Two practical challenges:
  1. **Anti-bot mechanisms**: CAPTCHAs and IP reputation checks block automated access. Data-center-IP virtual environments are easily identified; normal access often requires configuring a **residential proxy network** (real household IPs).
  2. **Access to the user's real accounts**: when a task must log in as the user, use **Human-in-the-Loop authentication** — a VNC/RDP remote desktop where the user logs in personally, sees the full interface the Agent is operating, and understands why authentication is needed. The session token is reused within its validity period to avoid interrupting the user repeatedly, balancing autonomy and security.
- **Data exchange** uses a shared file system: volume mounts such as `/workspace/shared` connect the Agent, virtual computer, and virtual phone. Data is passed by **file-path reference**, not copied into context. Example: user uploads a CSV to the shared directory; the Agent analyzes it and saves a chart there; the Agent returns only the chart's path. Every handoff stays a lightweight path string.
- Summary: event-triggered tools let the world wake the Agent; user communication tools let the Agent reach the user; virtual identities with isolated execution environments let the Agent act independently and auditably. Remaining question: when multiple events converge on the same Agent instance simultaneously, how should they be handled?

### 6.2.6 Event Handling Mechanism

- A single Agent may face multiple concurrent events: a new user message, a tool result, a timer expiring, a collaboration request from another Agent. Handling them efficiently and correctly directly impacts performance and user experience.
- **Skeleton**: the event loop from concurrent programming. An asynchronous Agent is a long-running loop: each round takes a batch of events off the input queue, appends them to the trajectory, invokes the LLM once, executes the tools it decides to call, then returns to the top to wait for the next batch — the same structure as a Go goroutine reading from a channel inside `for { select { ... } }`.
- **Key property**: events are consumed only at the boundaries of each round. While the LLM reasons or a tool executes, a new event does not barge in and disrupt the step; it waits until the round reaches a **safe point** (end of a stretch of reasoning, return of a tool call), where all pending events are handled together. **Cancellation** follows the same discipline: rather than cutting off work at an arbitrary moment, it checks "have I been asked to stop?" at the safe point — precisely the role of `ctx.Done()` in Go.
- The three processing strategies differ only in how they treat the safe point: let the event wait for the next naturally-arriving safe point (**queued**); proactively manufacture an early safe point (**cancellation-based**); or start another loop that does not wait for the main loop's safe point (**parallel**).

**Structured Event Modeling**
- Handling requires understanding: a third-party message is not sent by the user to the Agent, yet the Agent must understand it, weigh its importance, and decide whether to step in. Model each input as a structured event:
  - **Source (who)**: the user themselves, a contact, a stranger, a system notification.
  - **Channel (how)**: phone call, SMS, instant message, email, social media, timer trigger, asynchronous tool call result, command-line monitoring status update.
  - **Content (what)**: message text, emotional tone, urgency, whether a reply is needed.
  - **Context (background)**: whether it's a reply to a previous conversation or new communication; relevance to the current task.
- Example (customer refund request email):
```json
{
  "source": {"type": "email", "sender": "client@example.com"},
  "channel": "gmail_webhook",
  "content": {"subject": "Refund Request", "body": "Order #12345, requesting a refund..."},
  "context": {"priority": "high", "customer_tier": "vip", "related_orders": ["#12345"]}
}
```
- Only when these dimensions are clearly modeled can the Agent maintain understanding in multi-party communication — avoiding mistaking user input for a tool result, or a tool result containing hidden instructions for a user command (**prompt injection**). Multi-threaded context management requires understanding relationships between threads: how a third-party message affects the user's mood, the user's role transitions across conversations, when to synthesize information from different threads into advice.
- The trigger ecosystem of workflow platforms such as **n8n** makes the point: webhooks, timers, email, database changes, file watchers — each trigger is one of the Agent's "senses" for perceiving the world. Once modeled uniformly in a structured format, the Agent can handle stimuli from different sources consistently; urgency determination and processing strategies rest on this unified modeling.

**Dynamic Processing Strategy Based on Urgency**
- Humans juggling tasks adapt strategy to urgency: an emergency → drop what they're doing; routine to-do → list for later. Agents should show the same intelligence.

**Figure 6-2: Three Strategies for Asynchronous Event Processing**
- **Cancellation (Urgent)**: time t1/t2/t3 — LLM reasoning... → `user.interrupt: "Stop!"` (lightning) → tool executing... (×canceled) → New LLM reasoning (including interrupt event + clear queue).
- **Queue-based (Normal)**: LLM → Tool execution (search_web) → user: "Only look at the last 1 month" (enqueued, waiting) → LLM comprehensive processing → Batch append: tool.result + user input.
- **Parallel (Independent)**: Main task: data analysis (long execution); user: "How is the weather today?" → Parallel LLM (weather) → Reply to user immediately; tag: [Parallel with main task].

- **Cancellation-Based Processing** (urgent events): essence is forcing a safe point early. Steps: (1) stop the current operation — if LLM reasoning, immediately cancel the streaming response; if a synchronous tool is executing, send a cancel signal; (2) drain the pending queue (remove all pending events); (3) append those events with the urgent event to the end of the trajectory; (4) immediately re-invoke the LLM with the updated complete trajectory to assess the situation. Example: user inputs "Stop! I said the wrong thing" while the Agent is about to perform a potentially erroneous operation → the Agent immediately sees the input, re-understands the true intent, and avoids executing the wrong action.
- **Queued Processing** (routine events): (1) add the event to the end of the queue without interrupting the current operation; (2) wait for the current operation to complete (LLM finishes reasoning, synchronous tool finishes); (3) when any tool call completes and returns a `tool.result`, check the queue; if non-empty, append all events to the trajectory at once; (4) the LLM processes the updated trajectory comprehensively. Enables batch processing and efficiency — e.g., while waiting for a search result, the user adds "only show results from the last month"; both events are presented to the LLM together, avoiding unnecessary round trips.
- **Parallel Processing** (independent, lightweight queries): e.g., "What's the weather like today?" during a large data-analysis task. Characteristics: unrelated to the main task, need a quick response, low execution cost. Cancellation would interrupt the main task; queued would make the user wait too long. The system assesses the query's independence and complexity, executes it in a parallel reasoning session, calls necessary tools, and returns the response immediately. The query and response are appended to the main task's trajectory clearly marked "executed in parallel with the main task" to avoid confusing the LLM.
- **Urgency Determination**:
  - Urgent: `user.interrupt`, `supervisor.instruction`, `agent.interrupt`, external triggers marked urgent (system alerts, payment failures).
  - Non-urgent: regular `user.input`, `agent.input`, `tool.result`, `timer.trigger`, regular external triggers.
- Hardcoded rules have limitations; event semantics dictate the method — "Stop immediately!" → cancellation-based; "What's the weather like today?" → parallel; "Send the report in Chinese" → queued. Recommended: a **lightweight classification LLM as an event router**, quickly determining strategy on arrival.
- A cancellation point must be a position where the tool or reasoning can wind down safely; an unfinished tool result is represented by an explicit placeholder and must **never be faked as a success**.

### Experiment 6-1 ★★★: Event-Driven Email Processing Agent

**Architecture (Figure 6-3)**:
- External event sources: Web/App (`on_user_message`), Email system (`on_email_reply`), GitHub (`on_github_pr_update`), Timer (`on_timer_expire`), Webhook (`on_webhook_received`), System alert (`on_resource_alert`).
- FastAPI server: HTTP endpoint `POST /events/{type}`.
- Event router: LLM determines urgency. Event queue: priority sorting.
- Agent loop: Fetch → Reason → Execute. Session management: multi-threaded context.
- MCP tool server: Perception tools (`search_web`, `read_file`, `read_webpage`, `parse_image`); execution tool (`code_interpreter`, `virtual_terminal`, `write_file`); collaboration tool (`browser_use`, `request_human_approval`); notification tool (`send_email`, `send_slack`, `send_im_notification`).
- Persistence layer: conversation history, event log, scheduled task, tool status, audit trail.
- Builds the simplest event-driven Agent: an **Automated Email Processing Assistant**. Monitors the inbox; each new email automatically triggers a processing workflow — classification, summarization, draft reply, and notifying the user if necessary. Most intuitive intro scenario: an external event (new email arrival) triggers a complete Agent thinking cycle.
- **Experiment Objective**: understand the core idea of event-driven architecture (Agent no longer waits passively for user input but acts on its own in response to external events); master the basic closed loop of event source registration, the event queue, and "event arrives → Agent processes → result delivered."
- **Event sources**: unified access for Email Events (`on_email_received` — periodic inbox check or push), IM/SMS Messages (`on_im_message`, `on_sms_message`), GitHub Events (`on_github_pr_update`, `on_github_issue_update`), Timer Triggers (`on_timer_expire` — daily summaries, weekly reports), Webhooks (`on_webhook_received`), System Events (`on_user_inactive`, `on_process_timeout`, `on_resource_alert`). All enter a unified queue, processed sequentially in arrival order; each triggers an independent thinking loop (read content → call relevant tools such as knowledge-base queries, attachment reading, email-history search → generate result such as classification labels, summaries, draft replies → notify the user or execute an action).
- **Validation Scenario**: monitor a test mailbox; simulate three emails — meeting invitation, customer complaint, marketing advertisement. Meeting invitation → auto-check calendar conflicts, draft accept/decline reply; complaint → extract key info, mark high priority, notify user; advertisement → auto-archive. No user intervention required.
- Demonstrates the simplest pattern (sequential queue processing); insufficient for interruptions during long-running tool executions or concurrent task management → deeper engineering challenges next.

### 6.2.7 Engineering Implementation: How to Make Synchronous Models Support Asynchronous Interruptions

- Experiment 6-1 only handles serial events. Back to the "synchronous training / asynchronous deployment" contradiction: when the user interrupts while a tool has not yet returned, how can the synchronous format accommodate it?
- Scenario: Agent drafts an email (tool call: search for contact information). Before the search returns, the user says "Wait, first check tomorrow's weather for me." In a synchronous ReAct loop, the Agent must wait for the search to return — the API requires "after issuing a tool call, the next message must be the tool result." But in the asynchronous real world, events can interrupt at any time. Expressing "asynchronous interruption" semantics under a "synchronous format" is exactly the problem this solution solves.
- **Engineering Expedient: An Asynchronous Implementation Simulating Synchronous Behavior.** Core idea: under normal conditions (no interruptions), let the LLM see a standard synchronous trajectory; only on interruption, insert placeholders to fix the format. **Five key rules:**
  - **Rule 1**: Immediately record the assistant message (thinking, content, and tool call) when the LLM produces it.
  - **Rule 2**: Record the tool result only when the tool call is complete. The trajectory is in a "partially completed" state during execution.
  - **Rule 3**: Interruptions during tool execution require placeholders. Generate a placeholder response for the unfinished tool (e.g., "The tool is executing in the background, please prioritize the new event"), append the interruption event, and re-invoke the LLM. From the LLM's perspective the assistant message still has a paired tool result.
  - **Rule 4**: Interruptions during LLM thinking directly discard the current thinking — do not write it to the trajectory; append the new event and start a new round of thinking.
  - **Rule 5**: Non-interrupting events enter the queue for batch processing; appended all at once only after the current cycle completes.
- **Worked example** (email drafting interrupted by weather query):
  1. Agent calls `search_contacts`; the assistant message is immediately written to the trajectory (Rule 1).
  2. Before the search returns, user sends "First check tomorrow's weather for me." User interruption → system generates a placeholder tool result for the unfinished `search_contacts` ("The tool is executing in the background, please prioritize the new event", Rule 3), appends the weather query, re-invokes the LLM. Trajectory format seen by the LLM is completely valid — assistant message and tool result perfectly paired.
  3. After answering the weather query, the original `search_contacts` result arrives and is appended as a new event (Rule 2). The Agent reads the contact info and continues drafting the email.
- **Core advantage**: under normal conditions the LLM sees a perfect synchronous trajectory (strict pairing, clear timeline, no placeholders/anomalous states) — the friendliest arrangement for synchrony-trained LLMs, preserving thinking quality. The placeholder (a necessary compromise) appears only when an interruption genuinely occurs.
- **Hallucination risk**: even though the placeholder states the tool "has not yet completed," the model may fabricate a tool result in later thinking — convincing itself the tool returned valid data and basing decisions on fabricated data. Cause: in the vast majority of training trajectories a tool call is immediately followed by the real result; the model never learned "the result hasn't come back yet." In practice, **interruptions are triggered only in truly urgent situations** (user explicitly requests a stop); non-urgent events are queued for batch processing.
- **Asynchronous Tool Interfaces Suitable for Existing Models**: since the synchronous assumption is difficult to break, embrace asynchronous semantics at the tool-interface design level.
  - Traditional design implies "call equals completion" (e.g., `phone_call` suggests "dial the phone, wait for the call to end, return the call log"). Under asynchrony, "initiation" and "completion" are decoupled:
    - `initiate_phone_call`: initiates a call, immediately returning a task identifier and initial status (e.g., "Call initiated, dialing..."); progress via event notifications (`phone_call_connected`, `phone_call_ended`).
  - The tool's **name and description themselves convey asynchronous semantics** — a model seeing `initiate_phone_call` infers "initiating" rather than "completing". Example description: "This tool initiates a phone call task handled by a sub-agent. It returns the task ID immediately upon successful initiation, allowing you to continue with other matters. A separate notification event will be sent when the call ends."
- **Attention Dispersion in Queue-Based Processing**: when processing batch events, the model often focuses only on the last event — root cause: it is trained to react to the most recent input, and batch events break that assumption. Two intervention levels:
  - **Prompt level**: "When you receive multiple consecutive events, please ensure you comprehensively consider all the information."
  - **Agent status bar markers** before each event:
    - `[Unprocessed Event 1/4] Tool result from database_query: ...`
    - `[Unprocessed Event 2/4] User supplementary note: Only look at Beijing data`
    - `[Unprocessed Event 3/4] System reminder: Report deadline is in 30 minutes`
    - `[Unprocessed Event 4/4] User asks: What's the progress?`
  - Plus an end summary: "There are 4 unprocessed events above, including 1 tool result, 2 user messages, and 1 system reminder. Please ensure your response covers all the information."

### 6.2.8 Deeper Contradictions and Future Directions

**Figure 6-4: Synchronous Training Paradigm vs. Asynchronous Deployment Reality**
- Training paradigm (strictly synchronous sequence): Observation (User: Check Beijing weather) → Thinking (Need to call weather tool) → Action (`get_weather(Beijing)`) → Observation (22°C, sunny). Constraint: `tool_call` → next must be `tool_result`, otherwise API error.
- Deployment reality (asynchronous events interleaved): assistant `tool_call: get_weather(Beijing)`; waiting... (tool execution ~5s); user interrupts "No need, check Shanghai's". Questions: when will `tool_result` arrive? how to ensure format?
- Patch solution: placeholder ("[Tool still executing, prioritize interruption]") fixes format + non-urgent events enqueued + only interrupt when truly urgent. **Fundamental solution: next-generation models need to be trained via RL in asynchronous environments.**

- Placeholders, asynchronous tool interfaces, and status bar markers all use prompt engineering to patch the same contradiction — a temporary expedient during a transitional period. **The real solution requires a paradigm shift at the model training level.**
- **VLA (Vision-Language-Action) models** in robotics already face similar challenges (unavoidable delay between perception and action); their success points the way for the evolution of Agent models. The next generation of models needs three core capabilities via **reinforcement learning in asynchronous environments**:
  1. **Understanding asynchronous interleaving of events in trajectories** (the most critical deficiency): current models expect a strictly synchronous sequence, but in a real asynchronous environment a tool call might be followed by a new user message rather than a tool result; thinking might be interrupted halfway, the intermediate state should be retained in the trajectory, and thinking should continue after the new message is processed rather than start over. The model must maintain clear understanding in "out-of-order" trajectories — which tool calls are still waiting for results, which thoughts are unfinished fragments.
  2. **Resuming interrupted tasks and thoughts**: when interrupted to handle an urgent event, the model must remember the unfinished task. Example: user asks about the weather while the Agent executes a data-analysis tool; after answering, the Agent should naturally wait for the data-analysis result. Especially important to avoid hallucinations where the model believes the interrupted tool call has completed.
  3. **Comprehensive processing of batch events**: when multiple events are appended in a batch, the model must consider all unprocessed information, not focus on the last one.
- Async RL training needs new infrastructure: an **asynchronous environment simulator** (generating delayed tool returns, random user interruptions, etc.) and **specialized rewards** for asynchronous capabilities (understanding out-of-order trajectories, resuming interrupted thoughts, avoiding hallucinations, comprehensive batch processing).
- **Continuous thinking need not wait for the next generation of models**: about two hundred lines of orchestration can turn an existing text-reasoning model into a continuous-time Agent. It upgrades Rule 4 — instead of discarding an interrupted partial thought, make the interaction one uninterrupted stream of thought. The runtime forcibly closes the model's current thinking block, injects a newly arrived observation (tool result, user interruption, or recognition update) as an ordinary message, and lets decoding continue.
- It uses a commonly wasted resource: a model can generate hundreds of tokens per second while a tool call or a user's utterance may take several seconds. The Agent can therefore **think while waiting** (continue from partial information, even start the next tool early) and **think while acting** (continue reasoning while producing output, correct itself midway through an action).

### Experiment 6-2 ★★★: Asynchronous Agent with Parallel Execution and Interruption Capabilities

**Figure 6-5** (Agent | Tool A | Tool B | Tool C | Trajectory): LLM launches 3 tools. Script A: 3% per second → 33s; Script B: 2% per second → 50s; Script C: 1% per second → 100s. A completed → Query B, C progress (B≈66%, C≈33%) → Cancel C → B completed → LLM: Integrate A+B results to generate report → A result + B result + C cancellation record.
**Key: placeholder injection + async completion event + `cancel_tool(task_id)` API.**

- Builds on Experiment 6-1's simple event queue; moves into the hard parts of asynchronous Agents: **parallel tool execution, execution cancellation, state management**. The Agent must manage multiple concurrent tasks, handle interruptions and recoveries, and make dynamic decisions based on real-time state.
  1. **Asynchronous Tool Execution**: async execution of time-consuming tools (at least 3-5 seconds), returning a placeholder immediately on initiation. Validation Scenario: Agent executes a long-running terminal command; user asks "What time is it now?" → Agent responds immediately, presents the analysis result when the command completes.
  2. **Event Queue and Batch Processing**: accumulates non-urgent events, appends them in a batch. Validation Scenario: long task running; user sends consecutive "Remember to reply in Japanese" and "Format it as a webpage" → when the task completes the Agent processes all events at once, generating a Japanese webpage.
  3. **Interruption Mechanism**: a user "stop" command immediately terminates the execution flow and cancels the asynchronous tool. Validation Scenario: user sends "Cancel." → Agent stops immediately; trajectory records the interruption event and the cancellation operation.
  4. **Cancellation and Status Query for Parallel Tools**: after completion the real result is injected as a new event; supports cancellation or progress query by task ID. Validation Scenario: "Run these three scripts simultaneously for me. Whichever finishes first, check the progress of the remaining scripts. If any hasn't exceeded 50%, cancel it." Three scripts output progress at 3%, 2%, 1% per second. The 3%-per-second script finishes in ~33s; the Agent queries the other two (~66% and ~33%), cancels the one under 50%; integrates results into a complete report after both complete.

- Asynchrony and event-driven execution let the world wake an Agent at any time, but assume the model can finish thinking before it responds. The next three sections challenge that assumption: when the environment changes as fast as or faster than model generation, "think first, then speak" becomes unacceptable latency.

## 6.3 Voice: The Most Natural Human-Machine Interface

- Voice is not merely text turned into sound. **Speaking is roughly four times faster than typing** and leaves the hands and eyes free, so it naturally places an Agent in a continuous input-output loop where the user may interrupt at any moment. **Dictation** converts speech into text; a **voice Agent** lets the user collaborate directly. Both support the whisper-coding workflow introduced earlier.
- Two directions: the user speaking to an Agent, and an Agent speaking to the outside world on the user's behalf. The **voice model** determines what the Agent can answer; the **interaction architecture** determines whether it can hear clearly, respond in time, hand over naturally, and complete confirmations and tool calls during a call. First examine interaction timing, then cognitive timing and expressive quality.

### 6.3.1 Interaction timing: from cascaded to full-duplex

- OpenAI's GPT-Live introduction describes three voice-interaction paradigms — cascaded, turn-based, and full-duplex[1]. Not a simple old-to-new replacement; they trade latency, cost, and observability differently:

| Paradigm | Core structure | Main advantage | Main limitation |
|---|---|---|---|
| Cascaded | VAD → ASR → LLM → TTS | Clear modules that are easy to replace and debug | Latency accumulates; paralinguistic information is lost at interfaces |
| End-to-end Omni | Native audio input and output with turn-based interaction | Lower latency; better preservation of tone, emotion, ambient sound | Still turn-based; training and debugging cost more |
| Full-duplex | Native audio input and output with continuous listening, speaking, and decision-making | Overlapping speech, natural interruption, continuous streams | Training, control, and evaluation are more complex |

- Common thread: escaping the assumption that people must speak one at a time, and escaping VAD's guess about who has the floor. Cascaded and Omni systems still divide interaction into turns; **full-duplex makes turn ownership a continuous model decision**.
- [1] OpenAI. Introducing GPT-Live. 2026-07-08. https://openai.com/index/introducing-gpt-live/ — the cascaded/turn-based/full-duplex taxonomy comes from the article's summary of three generations of ChatGPT Voice; its "end-to-end omnimodal (Omni)" term corresponds to "turn-based voice models."

### 6.3.2 Paradigm 1 · Cascaded pipeline

- Most commercial voice assistants still use a serial pipeline (Figure 6-6): VAD decides when the user has finished; ASR converts audio to text; the LLM understands and generates a reply; TTS speaks it. Modularity lets each component be optimized independently, but every boundary can add waiting time.

**Figure 6-6: Serial voice Agent pipeline**
- VAD (Voice Activity Detection, 500-800ms, Silero VAD on ONNX Runtime) → ASR (Automatic Speech Recognition, 50-200ms, Whisper / SenseVoice) → LLM (Language Model Inference, 100-500ms, GPT-4o / Doubao Flash) → TTS (Speech Synthesis, 200-500ms, Fish Audio S1, neural network synthesis). Serial blocking: each stage waits for the previous to complete. User speaks 2-3 seconds → VAD waits → ASR waits → LLM idle → all time gaps wasted. Tech stack: Node.js WebSocket + AudioWorklet (128 samples ≈ 2.7ms) + FFmpeg.
- Module bottlenecks: VAD decides whether speech has ended — silence thresholds add waiting and split turns incorrectly. ASR converts audio to text — recognition latency and loss of context. LLM understands, reasons, generates — time to first token; reasoning adds more waiting. TTS converts text to speech — first-packet synthesis and playback buffering.
- For a short reply without reasoning, VAD + ASR + LLM + TTS waiting accumulates serially (Figure 6-7). Real value depends on input length, model, hardware, network, and load.

**Figure 6-7: Latency waterfall for a serial response** — VAD wait 500-800ms; ASR transcription 50-200ms; LLM TTFT 100-500ms; LLM generation 100-300ms; TTS synthesis 200-500ms. Total response time: **best 950ms ≈ 0.9s; worst 2300ms ≈ 2.3s** (ideal no-load scenario).

**Figure 6-8: Queueing latency curve** — utilization ρ vs total latency: ρ=0.5 → 2s (user tolerance limit); ρ=0.8 → 5s (unacceptable); ρ→1 → ∞. Total latency = S/(1−ρ) with idle latency S=1s. In production ρ is typically 0.5-0.8 → actual latency is 2-5× idle latency. (Capacity planning outside this chapter's scope.)

### Experiment 6-3 ★: Build a traditional voice Agent
Connect a microphone, Silero VAD, local Whisper, a streaming LLM, and Fish S1 TTS over WebSocket to establish the cascaded baseline.

### 6.3.2.1 From serial to streaming perception

- The fully serial case (VAD → ASR → LLM → TTS one after another) has three problems:
  1. **Accumulated latency**: must wait through silence before confirming the end.
  2. **Lost information**: a voiced/unvoiced bit cannot express hesitation, emotion, backchannels, or ambient sound.
  3. **Broken context**: email addresses, names, and proper nouns may be split across chunks and misrecognized.
- Keeping the modular split, one optimization is **streaming perception** — each stage produces incremental results as early as possible:
  - **Streaming ASR**: once VAD detects the user has started speaking, the ASR model is called at fixed intervals to produce a provisional transcript; once VAD detects the user has finished, the final text is confirmed.
  - **LLM speculative execution**: the provisional transcript is sent to the LLM as soon as it exists; if the final text matches the provisional transcript the LLM is not called again; otherwise the earlier speculative thinking is cancelled and the LLM is called again.
  - **Segmented LLM output**: the first speakable sentence goes to TTS without waiting for the full reply.
  - **Incremental TTS**: audio chunks are returned continuously so later generation, synthesis, and playback overlap.
- A truly streaming ASR needs model-level support: Whisper's decoder is autoregressive but its encoder expects a complete audio segment, so it cannot simply be equated with a streaming model. An **LLM-based streaming-audio model** can emit text and semantic events from continuous audio, placing recognition and part of understanding in one model; it keeps conversation context from the beginning to the present and can use world knowledge for brands, names, and proper nouns.
- **Endpointing**: if the only goal is deciding whether the user has finished, endpointing can be built into the streaming recognizer — the model combines semantics and silence to judge whether an utterance is complete. **Training labels must contain only information visible at decision time**, or hindsight produces a judgment that cannot be reproduced online.
- The model can emit **acoustic-event markers** as well as words:
  - `speak_start/end`, `interrupt`: speech boundaries and interruption intent;
  - `emotion`: emotion and hesitation;
  - `laugh`, `sigh`, `noise`: paralinguistic and environmental sound.
- Together with text tokens these markers form one event stream. The Agent can detect hesitation, interruption, and environmental changes without compressing every sound into plain text.

### Experiment 6-4 ★: Simulate streaming voice perception with Qwen2-Audio
Qwen2-Audio is not itself a streaming model. Simulate continuous perception with increasing audio prefixes and compare it with 600 ms VAD + Whisper.

### 6.3.3 Paradigm 2 · End-to-end omnimodal models (Omni)

- Even with streaming perception, a cascade passes listening, thinking, and speaking through discrete interfaces; emotion, intonation, and ambient sound may be lost when audio becomes plain text. **Omni** uses one model to listen to audio, generate a reply, and speak it — preserving those signals, though at higher training cost (Figure 6-9). Advantage over Paradigm 1 mainly in latency and in understanding/generating non-text information.
- **Understanding side**: Omni models pick up on pauses in the voice. **Generation side**: richer paralinguistic information — singing, or delivering a line in a distinctive tone.
- Omni models still assume turn-taking and generally use VAD to assign the floor. A mid-utterance pause while the user reads out a string of digits can still be mistaken for the end of the turn.

**Figure 6-9: End-to-end omnimodal speech-model comparison**
- OpenAI Realtime (`gpt-realtime`): server-side VAD + turn detection; asynchronous function call; interruption stops current generation. Essentially still under the VAD framework. 2024 preview based on GPT-4o.
- Gemini Live (Gemini 2.0): VAD sensitivity configurable; interruption retains sent information; dialogue coherence optimization. Similar to OpenAI's strategy.
- Qwen3-Omni: Thinker-Talker MoE — Think (understand) + Express (generate); multi-codebook speech coding; first packet latency **234ms**; **22/36 benchmarks achieve SOTA**.
- Step-Audio 2: potential audio coding + RL; end-to-end speech token; paralinguistic understanding **83.09%**; CoT thinking + RAG + tools; surpasses Qwen2.5-Omni (44.18%).
- Common limitation: still reliant on turn detection. Insufficient turn detection: pause ≠ end of speech; needs proactive guidance; primitive interruption judgment; acknowledgments ≠ interrupts; noise misread as speech. Root cause: acoustic signals only — misses semantic meaning.
- Input/output architecture comparison: Traditional serial: Audio→VAD→ASR→Text→LLM→Text→TTS→Audio. OpenAI/Gemini: Audio→End-to-end model (built-in VAD)→Audio/Text. Qwen3-Omni: Audio/Image/Video→Thinker (MoE)→Talker→Speech Codec. Step-Audio 2: Raw Audio→Latent Encoder→LLM+RL→Audio+Text Tokens. Full-duplex Moshi: parallel dual audio streams + inner monologue text stream → no VAD / turn detection needed.

### Experiment 6-5 ★★: Run MiniCPM-o 4.5 locally — end-to-end versus self-cascade
Run MiniCPM-o 4.5 locally with thinking mode disabled, comparing direct answers from audio against a self-cascade that first transcribes and then answers with the same model. Measures whether audio information is preserved — not the "thinking while speaking" discussed later.

### 6.3.4 Paradigm 3 · Full-duplex interactive models

- Omni still divides conversation into "the user speaks" and "the model speaks," but simultaneous interpreting and similar tasks require overlap. A **full-duplex model does not presuppose turns**: it listens and speaks continuously and repeatedly decides whether to continue, pause, interrupt, or call a tool.
- **Kyutai's Moshi (2024)** — early research example. It models the user's and the model's audio streams in parallel, so overlapping speech and interruption can be natural behaviors.
- **Thinking Machines Lab** calls this an **Interaction Model**: interaction is built into the model instead of assembled around it with VAD and other external harnesses. Its **micro-turn mechanism** advances in short audio blocks, preserving silence, overlap, and interruption as continuous context. It can delegate the full conversation to a background reasoning model while it keeps the conversation alive, then incorporate the result at a suitable moment. (Thinking Machines Lab, "Interaction Models: A Scalable Approach to Human-AI Collaboration," 2026-05. https://thinkingmachines.ai/blog/interaction-models/)
- **OpenAI's GPT-Live** brings the full-duplex path to production scale: continuously processes input and generates output; can wait, backchannel, be interrupted, and handle realtime translation. Like the Interaction Model, it delegates complex work to a background model while the foreground model maintains the conversation.

### 6.3.5 Cognitive timing: realtime interaction and deep thinking

- Interaction quality and intelligence ceiling are different dimensions: the **foreground model** must respond while the user is still engaged; the **background model** can spend longer thinking. The three designs below are trade-offs, not a linear progression. The first two can wrap a cascade or Omni model; the third unifies deep reasoning and realtime expression within the same model.

### 6.3.5.1 Solution 1: Fast thinking for fillers, slow thinking for answers
- Fast thinking gives a holding response within a few hundred milliseconds while slow thinking performs a deeper derivation in the background. Simple questions may be processed twice; hard questions can produce **contradictions**: the fast model recommends a purchase, then the slow model discovers a key feature is missing. Root cause: **two independent instances thinking separately**.
- Figure 6-10 example ("Which plan to choose?"): fast thinking ~500ms (thinking off) → "Good price, recommend purchase"; slow thinking ~8s (thinking on) → "Lacks international roaming, not suitable". User experience: 0.5s then 8.0s → contradiction → user loses trust.
  - **Issue 1 — Overthinking simple problems**: "What day is it today?" → fast already correct yet slow still runs 8s.
  - **Issue 2 — Inconsistency between fast and slow**: independent thinking paths may have completely different assumptions.
  - **Improvement**: slow thinking as an "advisor" guiding behind the scenes — slow thinking → Agent status bar → fast thinking. No direct conflict, but communication is indirectly vague. Still fundamental limitations: fast thinking may misinterpret status bar hints ("Confirm price" → "Confirm with user" instead of "Recalculate"); cannot achieve natural "thinking while speaking."

### 6.3.5.2 Solution 2: Fast thinking for interaction, slow thinking for advice
- The background model sends advice through a status bar or dedicated interface while the foreground model keeps the conversation alive and decides how to phrase it. More stable than Solution 1, but communication is still indirect: the foreground can misunderstand advice and cannot see the background's intermediate reasoning. Before the background finishes, follow-up questions still rely on the foreground. It can naturally wait for a result, but it cannot truly think while speaking.

### 6.3.5.3 Solution 3: End-to-end unification of thinking and expression
- Internalizes reasoning directly in an end-to-end audio model. **Step-Audio R1** uses two complementary mechanisms:
  - **Modality-Grounded Reasoning Distillation (MGRD)**: grounds thinking in acoustic features.
  - **MPS dual-brain architecture**: lets planning and expression proceed in parallel.
  - The first helps the model think correctly; the second helps it speak in time.
- Ideally the model infers emotion from pitch, rhythm, and intonation rather than only from the transcript. MGRD selects reasoning traces that actually cite acoustic features, trains on them, and uses reinforcement learning to prevent guessing without thinking. MPS lets the planning brain continuously emit thought segments; the expression brain combines each segment with the partial reply and immediately generates speech. The pipeline runs in parallel, so the listener need not wait for the entire chain of reasoning before hearing the first sentence.

### 6.3.5.4 The trade-off between separated fast/slow thinking and end-to-end reasoning
- A unified model implements "thinking while speaking" most directly, but thinking and realtime expression must be retrained together; a decoupled design makes it easier to swap the background brain. These are trade-offs, not simple substitutes.
- As frontier reasoning models advance rapidly, **fast/slow separation captures the gains of each new generation of slow models directly**: the fast foreground model only listens, responds, and sustains the conversation with low latency; the slow background handles reasoning, planning, and tool use. When a stronger reasoning model arrives, only the background is replaced — the realtime voice system need not be retrained. A unified design binds reasoning and interaction to the same training cycle, so every upgrade must rebalance intelligence, response latency, and natural expression. Fast/slow separation is therefore not merely a latency compromise but a **modular choice letting interaction capability and the intelligence ceiling evolve independently**.
- This separation does not necessarily sacrifice task performance. **As of August 2026, Pine AI's voice Agent (separated fast/slow architecture) ranked first on the τ³-Voice Leaderboard**, ahead of realtime voice systems including Grok Voice and GPT-Realtime-2 — showing a decoupled architecture is not inherently inferior on tasks jointly testing deep reasoning and realtime conversation. (Pine AI. "The Most Natural Human-Computer Interface Is Your Voice." 2026-06-23, updated 2026-08-06. https://www.19pine.ai/blog/pine-ai-the-most-natural-human-computer-interface-is-your-voice)
- **"End-to-end model" is commonly used in two senses (independent axes):**
  1. **End-to-end speech path** (Section 6.3.3): the model receives audio and produces audio directly instead of connecting multiple models through discrete text. Both Omni and Interaction Models are end-to-end in this sense, but Omni models usually remain turn-based whereas Interaction Models can listen and speak at the same time — their architectures differ substantially.
  2. **End-to-end cognitive architecture** (this section): realtime interaction and deep reasoning either share state and are trained together within one model, or are split between a fast foreground model and a slow background model.
  - A system can have an end-to-end speech path while retaining fast/slow separation in its cognitive architecture; Thinking Machines Lab's delegation of complex tasks to a background reasoner is one such combination.

### 6.3.6 More human-like speech synthesis

- Traditional TTS can expose its machine identity by being too smooth and pausing too little. **Pauses, filler words, and occasional repetition signal uncertainty and thought** in human speech.
- The main LLM can emit **control markers** in addition to text, such as `THINKING`, `EMO:happy`, and `SPEED:0.8x`; TTS maps them to pauses, prosody, speaking rate, laughter, sighs, and other nonverbal audio.
- Implementation options: a TTS **trained to understand control markers**, or **voice cloning with reference clips** for different emotions and styles.

### Experiment 6-6 ★★: Control token-driven TTS with Fish Audio
Use Fish Audio S1 to build a multi-reference voice library and compare three configurations: no control markers, one reference clip, and multiple reference clips. The execution layer selects matching emotion, speaking rate, and style from the markers.

## 6.4 Computer Use: GUI Automation Agents

- Voice pushed the timing axis down to the millisecond, but its observation is still a one-dimensional stream of sound. **Computer Use** moves the same problem onto a two-dimensional screen: observation becomes continuously changing pixels; action becomes clicks and keystrokes at coordinates. Voice emphasizes when to speak; Computer Use emphasizes where to click next — plus a question that does not exist in voice interaction at all: **after an action executes, is reality still consistent with the plan?**
- Computer Use (also known as GUI automation) lets AI use software like a human by observing the screen and operating the mouse and keyboard — opening a browser to search, filling data in a spreadsheet, adjusting configurations in system settings. Its core is a **Perceive-Think-Act loop** (Figure 6-11):
  1. The Agent takes a screenshot of the current screen.
  2. A multimodal model receives the screenshot and task instruction, and outputs a thought and a specific action.
  3. The execution layer performs the action in the real environment (moving the mouse, clicking, typing text, etc.).
  4. It waits for the interface to respond, takes another screenshot, and enters the next loop iteration.
- Important distinction: **understanding the interface** (closer to multimodal understanding; measurable with one-shot screenshot question answering) vs **completing the task** (requires putting understanding and action generation into a closed loop that handles page loading, state changes, mistakes, and irreversible consequences). The challenge is not merely answering correctly about a screenshot, but **reconfirming after every step that reality still matches the plan**.

**Figure 6-11: Computer Use Agent's Perceive-Think-Act Loop**
- ① Screenshot: computer screen (desktop/browser/application); take screenshot after interface stabilizes.
- ② Model inference: input = screenshot + task instruction; output = Thought ("Search box is in the center of the screen") + Action (`click(512, 250)`, `type("weather")`).
- ③ Execute action: execution tool (xdotool / Playwright); mouse: move/click/drag; keyboard: input/hotkeys; scroll: up/down/left/right; wait: interface response.
- ④ Interface state changes → wait for stability → next screenshot.
- Typical scenario: completing a multi-step form may require **10-20 loops**.
- Three key design dimensions: **Action Space** (what operations the Agent can perform), **Visual Grounding** (how to find the target element in the screenshot), **Model Architecture** (how to generate the correct action from the screenshot).

### 6.4.1 Action Space Design

- Anthropic's reference implementation divides a complete interaction capability into three types of tools (Figure 6-12). A clear action-space design, but **not a private protocol** model providers must follow: as long as the Harness can translate the same screenshots, action constraints, and execution results into the messages and structured outputs the target model supports, Claude, open-weight vision models, and self-hosted endpoints can all drive the same Perceive-Think-Act loop.

**Figure 6-12: Computer Use Action Space**
- GUI operation tool (computer): Mouse — `mouse_move`, `left/right/middle_click`, `double/triple_click`, `left_click_drag`, `left_mouse_down/up`; Keys — `type` (character by character, 12ms interval), `key` (combination key), `hold_key` (long press); Scroll — 4 directions + modifier keys; Perception actions — Screen: screenshot → scale to training resolution; Cursor: `cursor_position` → (x, y); Wait — `wait` → wait for interface to stabilize.
- Command execution (bash): Term — persistent bash session, 120s timeout; sentinel string detection for completion.
- File editing (str_replace_editor): Ops — `view`, `create`, `str_replace`, `insert`, `undo_edit`.
- Coordinate scaling mechanism: actual resolution ↔ training resolution (XGA/WXGA/FWXGA); screenshot shrink → model inference → coordinate enlarge → xdotool execute.
- Typical execution flow (fill form): ① screenshot (capture page) → ② model inference (find name field) → ③ `mouse_move` (move to (324, 156)) → ④ `left_click` (get focus) → ⑤ `type` ("John Smith"). **Each action interval 2-5s (serial screenshot-recognize-think-click); 1/3 to 1/5 of human speed.**
- Details: GUI tool — mouse operations include moving (`mouse_move`), left/right/middle clicks, double/triple-click, dragging (`left_click_drag`), precise press/release (`left_mouse_down`/`left_mouse_up`); scrolling (`scroll`) supports four directions, combinable with modifier keys; keyboard includes character-by-character typing (`type`, 12ms interval to simulate real typing), key combinations (`key`, e.g., Ctrl+C), and key holding (`hold_key`). Perception actions: screenshot, `cursor_position`, `wait`.
- Command execution tool (bash): persistent bash terminal session with **120-second timeout**; **sentinel string** to detect command completion; maintains environment state across calls (after `cd`, the next call remains in that directory).
- File editing tool (str_replace_editor): safe editing through string matching; supports view, create, replace, insert, undo. More precise than overwriting an entire file; less likely to modify unrelated content accidentally.

### Experiment 6-7 ★: Running Computer Use (Anthropic Reference Path or Open-Model Path)
- Path A — Anthropic Computer Use Demo: container packages a complete Ubuntu desktop environment (browser, terminal, common tools). Frontend receives a task; backend sends instructions and screenshots to Claude and executes the mouse, keyboard, terminal, or editing actions returned by the model.
- Path B — example code in `chapter6/computer-use-open-model`: by default drives `browser-use` with the open-weight **Qwen3-VL 32B Instruct** model through the hosted OpenRouter API, or through self-hosted vLLM/SGLang and similar systems.

### 6.4.2 Visual Grounding

- In each loop iteration the model must accurately locate the target element in the screenshot ("Where is the search box?", "What are the coordinates of the submit button?"). Two main approaches:
  1. **Turn localization into a multiple-choice problem**: annotate interface elements with numbers; the model only selects one.
  2. **Pure coordinate prediction**: the model "looks" at the screenshot and reports coordinates directly, like a human.
- The multiple-choice approach has two implementations: **pure visual annotation** (the original Set-of-Mark — a segmentation model segments candidate regions in the image) and **structured element indexing** (DOM/Accessibility Tree — reads the interface's inherent structure).
- Common advantage: transforms the open-ended problem ("find the button in the screenshot and predict its coordinates") into a closed-ended one ("choose one from the already-annotated elements"). Like multiple-choice vs fill-in-the-blank on exams; the model only says "click [123]" instead of "click the button at screen coordinates (350, 464)." Direct coordinate prediction is especially hard: extensive training to get right, and prone to error across different screen resolutions.
- **Set-of-Mark: Visual Annotation Method** — original SoM proposed by **Microsoft Research in 2023**, initially to unlock the visual grounding capabilities of GPT-4V. Purely visual: uses image segmentation models (SAM, SEEM, etc.) to automatically segment candidate regions in the screenshot, overlays a numbered marker on each region; the model sees the numbered image, reports a number, and the system converts it to the region's center coordinates. No DOM or internal interface structure needed — equally applicable to native desktop software and game interfaces, as long as the segmentation model can identify candidate regions.
- **Structured Element Indexing: A Structured Implementation of the SoM Idea on the Web** — when the interface itself provides structured information, annotation can be more precise. Modern web pages define a complete element structure (the **DOM tree**) and semantic roles identifying buttons, input fields, and other controls; **accessibility trees** provide similar info for many desktop applications. Web Agent systems such as browser-use take exactly this route. Four steps:
  1. Obtain the structured representation (DOM tree) and accessibility info through the browser's debugging interface (**CDP — Chrome DevTools Protocol**).
  2. Automatically detect which elements are interactive (buttons, input boxes, links, etc.).
  3. Annotate each interactive element with a unique ID and draw bounding boxes on the screenshot.
  4. Simultaneously generate a text list describing the element corresponding to each ID.
- Example annotation list:
```
[1] <input type="text" placeholder="Search" aria-label="Search" />
[2] <button id="submit-btn" aria-label="Submit form" />
[3] <input type="text" placeholder="Enter your name" value="" />
[4] <a href="/docs" aria-label="Documentation" />
```
- The model only outputs an ID; the system automatically clicks the center of the corresponding element. This does **not** save tokens (all annotation data must still be sent to the model), but provides accurate, stable localization and avoids the missed detections and false positives segmentation models can introduce.
- Figure 6-13: simulated webpage screenshot with annotated bounding boxes ([1] Search box, [2] Submit button, [3] "Enter your name" input, [4] Documentation link) + element list (text descriptions). Model output ID → system executes with center coordinates. SoM flow (browser-use implementation): CDP acquisition (DOM/A11y) → interactivity detection → bounding box + ID assignment → draw box annotation on screenshot → text list + screenshot → model. **Applicable boundary: structured interfaces (web/Accessibility API); Games/Canvas fall back to pure visual methods.**
- **Pure Coordinate Prediction** — skips annotation; the model outputs coordinates directly. Systems such as **SeeClick** and Claude's computer use rely on vision models trained on massive datasets of GUI screenshots paired with element positions. These models learn to map natural-language descriptions ("click the submit button") directly to precise screenshot coordinates, relying on visual perception like a human user.
- Coordinate prediction is highly dependent on the resolution used during training (Figure 6-14). **Claude was trained using XGA (1024×768), WXGA (1280×800), and FWXGA (1366×768)**. If the input screenshot resolution does not match, predicted coordinates systematically shift — like measuring distance on a small map and applying it to a large one. Therefore implement a **bidirectional coordinate scaling mechanism at the tool layer**, and select the target resolution by **aspect ratio** to avoid non-uniform stretching that distorts the image.
  - Example: actual screen 2560×1440 (16:9) → best target is FWXGA (1366×768), closest aspect ratio to 16:9. Screenshot scaled to 1366×768, fed to model; model outputs (683, 384); inverse mapping: (683×2560/1366, 384×1440/768) ≈ **(1280, 720)**. Conversely, forcibly stretching a 16:9 image into 4:3 1024×768 horizontally compresses the image, causing systematic coordinate shift.
  - Full workflow: ① select target resolution by aspect ratio → ② downscale screenshot with ImageMagick → ③ model inference generates coordinates → ④ scale up proportionally → ⑤ execute with xdotool.
  - Scaling example: screenshot 2560×1440 → 1366×768 (×0.53); coordinates model (683, 384) → actual (1280, 720) (×1.87).
- Choice among the three routes: when structured information is available, **prioritize DOM/accessibility-tree indexing** (most accurate, stable). When unavailable — native desktop software (Photoshop), canvas/WebGL-rendered interfaces, games — use **visual annotation** (original SoM) or **coordinate prediction**. Visual annotation turns localization into a multiple-choice problem, friendlier to general-purpose models without specialized training; coordinate prediction eliminates the annotation step, more direct for GUI-localization-trained models. **Both still struggle with small elements and dense interfaces.**

### Experiment 6-8 ★: Using browser-use to Implement Automated Browser Operations
Use Playwright (browser-automation framework) together with a multimodal model to implement natural-language-driven browser operations. Enable SoM visualization and save a screenshot with annotated bounding boxes before every decision. Test task "Open Google and query San Francisco weather": after startup the screenshot shows Google Search with numbered interactive elements; the model selects the search box, enters "San Francisco weather today," submits, and extracts the temperature and conditions from the results page.

### 6.4.3 A Computer Use Agent That Can Watch Animations and Hear Sound

- So far Computer Use perception rested on an implicit assumption: **the screen is static** — screenshot, reason one step, click, next screenshot. Real screens play videos, flash short-lived notifications, and carry voices from meetings. An Agent that opens its eyes only once every 3-5 seconds and has no ears cannot see or hear what happens between two frames.
- What needs redesign is **not the action interface but the observation interface**. An **Agent-computer observation interface (AOI)** converts continuous environmental observation into discrete events the model can handle. Key techniques:
  - **Screen keyframe capture**: a small model judges whether the screen has changed meaningfully and only screenshots on a significant change — when changes are frequent, capturing once per second already works well.
  - **Volume-gated speech transcription**: invokes recognition when sound is present and feeds the recognized text into the context so the Agent can hear.
  - **Describing the screen as text**: the model turns each captured screenshot into a one-sentence description that stays in context after the original image leaves it, compressing multimodal interaction history.
- Reference: Li, Bojie and Noah Shi. Agent-Computer Observation Interfaces Enable Dynamic Computer Use. arXiv:2606.29472, 2026.

### 6.4.4 World Models for Computer Use

- The observation interface answers "what happened in between?" — but it does **not remove planning latency**. The Agent still runs a serial "screenshot — think — click" loop, re-observing and reasoning about the next step after every single action. The **OSWorld-Human efficiency study** shows that even when a task eventually succeeds, the Agent takes markedly more steps and waits markedly longer than a person does; **reaching human-level accuracy is not the same as being practical**.
- People do not start thinking about the next step only after clicking. They first **predict what an action will do**: if the actual change matches the expectation they carry on with the existing plan; only when the page state departs from expectation do they stop to observe and plan again. A **world model** lets the Agent predict what the desktop may turn into before it acts — a human-like "speculative execution" that improves efficiency substantially.
- **Desktop state is more than a grid of pixels**: it includes windows, focus, scroll position, input-field contents, loading state, permissions, and network responses; actions include clicking, typing, scrolling, dragging, and waiting. A Computer Use world model must at minimum encode the current state, predict the state change a candidate action would cause, and hand that prediction to the planner:
  - `desktop state + click/type/scroll/wait → representation of the next state`
- This lets the Agent compare candidate actions' consequences before clicking, prepare the next step while a page loads, and recover from a dialog that flashed past by reasoning about the state difference.
- Example: "create a new Python file in VS Code and write hello world" → predict the key state of the file tree and editor on success, then choose the click/type/save actions. "Delete a file" → predict inside an isolated virtual desktop whether an irreversible confirmation dialog will appear, and ask the user to confirm when necessary. The point is **not to generate a photorealistic future screenshot, but to predict the checkable state differences the task requires**.
- **Photon-1 from Induction Labs (July 2026)** demonstrated one implementation: **completed the pretraining of a computer use world model with only 30,000 hours of H200 GPU time**. It compresses each frame into discrete latent tokens and autoregressively predicts the representation of the next state after an action, rather than generating screenshots pixel-by-pixel during pretraining; the attached image generator only visualizes latent representations and is **not a component required for inference**. Given a seed screenshot and the actions that follow, the model can "imagine" desktop states continuously, then learn to output computer-use actions through online training on virtual machines. (David Li and Jonathan Li, Induction Labs, "Scaling Video Pretraining with Imagination Models," 2026-07-23. Parameters, data scale, internal benchmarks, and cost comparisons are company-disclosed figures.)

### 6.4.5 Mobile: Ecosystem Barriers Are Harder Than Technology

- Computer Use is expanding to mobile. Technical differences exist: instead of mouse coordinates and keyboard input, the mobile action space typically uses the system's **accessibility-service API** (e.g., Android's `AccessibilityService`) to read interface elements and issue clicks or enter text. Interaction shifts from a mouse pointer to **touch gestures**, changing the meaning of coordinates — the same (x, y) position may indicate a tap, a long press, or the starting point of a swipe, so the action must also specify a **gesture type**. Mobile benchmarks such as **AndroidWorld** (introduced in Chapter 7) evaluate an Agent's ability to complete tasks in real applications within this action space.
- What truly hinders mobile Computer Use is often **not these technical differences but ecosystem barriers**. Some phone manufacturers attempted to integrate AI assistants into consumer-grade phones to automatically operate everyday apps like WeChat, Taobao, and Alipay, but quickly encountered platform restrictions.
- **Ecosystem barriers**: the fundamental reason is a **conflict of business models**. The core monetization logic of traditional internet applications is traffic and attention — users see ads while scrolling feeds, are guided by recommendation algorithms when searching products, and make impulse purchases while browsing. When an Agent operates on the user's behalf, that monetization chain is bypassed entirely: the AI ignores ads, makes no impulse purchases, heads straight for the goal, finishes the task, and leaves. For platforms that live on advertising and traffic, every Agent operation erodes the foundation of the business model.
- Computer Use therefore faces not only technical countermeasures such as CAPTCHAs, but also a **structural conflict of interest** — difficult to resolve short-term and a greater obstacle to consumer adoption than purely technical problems.

## 6.5 Robot Manipulation: Tidying a Desk with XLeRobot

- **Reading note**: this section uses one task throughout — "put the red cup in the tray, put the yellow scrap paper in the bin, then observe again and confirm the state of the desk." Experiments 6-9 and 9-9 run on real XLeRobot hardware and need an arm, calibration, an emergency stop, and an on-site observer; experiments 9-8, 9-10, and 9-11 are the corresponding local-GPU experiments. Hardware and simulation are reported separately, but the task goal, action semantics, and success conditions stay the same.
- Robot manipulation is much harder than answering questions about a picture: the model must understand the scene and then take actions continuously in the real world, where every action changes what the next moment looks like. XLeRobot makes that difference concrete: the same arm can be teleoperated by a person through a keyboard, a gamepad, or a VR device, or it can hand camera observations and a constrained set of action tools to an Agent. The hardware and task stay fixed; only the operator changes — in the first case a human observes and corrects continuously, in the second the model and the control system must do the same work.
- Five experiments on "tidy the desk": (1) a human teleoperates the real XLeRobot, measuring what the hardware can do under a sufficiently capable operator; (2) a simulator establishes the ideal control ceiling for the same task; (3) an Agent controls the real XLeRobot autonomously, showing how perception, planning, and failure recovery affect the result; (4) the same tool contract goes into the simulator so open-loop execution, step-by-step checking, and world models can be compared in bulk; (5) the background, object appearance, lighting, and visual noise change, to see whether a visual policy learned in simulation adapts to a new environment.
- The bottleneck is usually not one more static question-answering benchmark, but whether the model can keep closing the loop under limited perception and control bandwidth. A usable robot system must answer at least **four questions**:
  1. What task does the person want done?
  2. Which subtask comes next?
  3. What actions does the current skill actually emit?
  4. After the action executes, does reality still match the plan?
- These four questions sit inside one XLeRobot control loop, and each of four techniques is responsible for one: **long-horizon planning** decides whether the cup or the paper is handled first; a **VLA or action primitive** performs the grasp and the placement; a **world model** estimates the consequences of an action; **sim-to-real transfer** handles the differences between training footage and the real camera and actuators. Even when the high-level model already has enough knowledge and planning ability, **losing any one of these feedback links can still leave the task unfinished**.

### 6.5.1 The Division of Labour Between Hardware and Algorithms

- First question XLeRobot is best suited to answer: when autonomous desk tidying fails, is it the **arm** that cannot do it, or the **algorithm** that is not using the arm well? An arm costing only a **few hundred dollars** (like XLeRobot) can already complete the continuous multi-step desk task through teleoperation — a person watches the camera feed, picks up the red cup and puts it in the tray, then puts the yellow scrap paper in the bin, and confirms the state again. This is a clear piece of diagnostic evidence: **for this task the hardware itself is not the bottleneck, the algorithm is.**
- Diagnostic method: keep the camera, arm, gripper, desk layout, and success conditions fixed, and let a human take over the loop. A human continuously corrects object localization, action choice, and timing, and handles failed grasps — the gap between an autonomous system and a person lies precisely in those closed-loop abilities. Scope: this section's desk task — it shows the hardware has cleared the payload, precision, and workspace thresholds the task requires, not that a few-hundred-dollar arm can handle every open environment or harder manipulation.
- XLeRobot supports **keyboard, Xbox controller, Switch Joy-Con, and VR teleoperation**. A human operator naturally does many things an algorithm must implement explicitly: slowing the gripper as it nears the cup, correcting the grasp point when the cup slides, observing again after failing to pinch the paper the first time, and checking the outcome once an object is in the target area. Teleoperation is not only a way to collect demonstrations but also a "**fix the hardware, swap the operator**" diagnostic experiment. (XLeRobot, "Teleop documentation," https://xlerobot.readthedocs.io/en/latest/software/getting_started/XLeRobot_teleop.html)

### Experiment 6-9 ★: Teleoperating a real XLeRobot to tidy a desk
Place a red cup, a tray, yellow scrap paper, and a bin in the real XLeRobot workspace. Using one calibrated teleoperation method, the operator performs the fixed task: "put the red cup in the tray, put the yellow scrap paper in the bin, then observe again and confirm the state of the desk." Repeat for several rounds at minimum, recording the camera feed, operator input, arm state, action timing, failed grasps, retry counts, and the final state.
**Acceptance cannot rest on "the desk looks tidy at the end."** The red cup must be inside the tray, the yellow paper inside the bin, the arm back in a safe pose, with no collision, no out-of-bounds motion, and no unconfirmed manual intervention along the way.
- Teleoperation on real hardware gives the most convincing ceiling for the task, but is not suited to varying object counts and positions in bulk. For a repeatable, statistically meaningful control, the next step moves the same "put objects where they belong" problem into a **2D desktop simulator**, with an ideal controller standing in for a strong operator who never misperceives and never picks the wrong action.

### Experiment 6-10 ★: Measuring the ideal control ceiling for the same task in simulation
In a 2D desktop simulator, randomly place the red cup, the yellow paper, and their target areas, and let an ideal controller approach each object in turn, grasp it, and move it to the right place. It does not need to recognize images and never picks the wrong action, so it represents "what this task can at least achieve when perception and decision-making are both correct."
The experiment tracks **task success rate, number of steps, and path length**, and varies initial object positions and task scale to see whether the ideal ceiling stays stable. It uses the same success conditions as experiment 9-7, but measures a non-actuated simulation and does not imply the real XLeRobot has been run. Together the two establish the reference lines for the autonomous control that follows: experiment 9-7 is a human loop on real hardware, experiment 9-8 an ideal loop in simulation.

### 6.5.2 The Basic Structure of Robot Control

Robot systems usually separate work by timescale:

| Layer | Core question | Output | Typical timescale |
|---|---|---|---|
| Task goal | What does the person want done | "Put the cup and the paper away" | Minutes |
| Long-horizon planning | What comes first, what comes after | Handle the cup, then the paper, then check | Seconds to minutes |
| Basic skills | Which state change to achieve now | `pick(red_cup)`, `place(red_cup, tray)` | About 1-3 s |
| VLA / skill policy | How this skill actually moves | A short motion or continuous trajectory of the XLeRobot gripper | About 1-10 Hz inference |
| Low-level control and safety | How to execute stably and in time | Joint or end-effector commands, speed limits, and emergency stop | About 50-1000 Hz |

- This is a common engineering split, not the only model architecture. A VLA can take on part of the high-level judgement, and the planner can be a rule-based program, a VLM, or an optimiser. Whichever implementation you choose, "task order" and "the action right now" should stay separate; otherwise the high-level model's inference latency drags down low-level control, and high-frequency low-level control forces the high-level model to process a great deal of irrelevant detail. For XLeRobot the model should **not emit arbitrary joint angles directly**; it only selects bounded skills such as `pick`, `place`, `verify_state`, or `stop`, and a calibrated, speed-limited executor with timeouts turns those skills into real arm motion.

### 6.5.3 Long-Horizon Planning and Task Decomposition

- When the user says "tidy up the desk," the system cannot hand that sentence straight to an action model. The planner first lists the objects and goals in the scene, then decides the order, and for each step writes down the start condition, the completion condition, and the risk limits. Example:
  - `handle the red cup → clear the yellow paper → check the desk`
  - "Handle the red cup" decomposes further into two actions and one check: `pick(red_cup) → place(red_cup, tray) → verify_state()`
- Every completed skill yields a **checkable node**. If a grasp fails, only that step is retried; if someone moves an object or the user changes the goal, only the affected later steps need replanning — the old plan does not have to be redone from scratch. Tools given to the agent should be equally simple: **one call does one thing, the range of motion is fixed, there is a timeout, and observation happens again immediately after execution.**

### Experiment 6-11 ★★: Driving XLeRobot to tidy a desk autonomously with Gemini Robotics-ER 1.5
Keep the real XLeRobot, desk layout, task instruction, and success conditions of experiment 9-7 unchanged, and replace the human operator with an Agent. An embodied reasoning model such as **Gemini Robotics-ER 1.5** can handle observation and planning, exposing only **five tools** through a RoboCrew-style agent loop: `observe_scene`, `pick`, `place`, `verify_state`, and `stop`.
The model first observes the desk, decides the order, then calls the calibrated XLeRobot grasp and place actions. After every completed skill it must observe again and check the postcondition; on a failed grasp it may only retry the current skill, and it must call `stop` when the user says stop, when an object leaves the workspace, or when the state cannot be confirmed. **The model cannot emit arbitrary joint angles, nor skip a real check merely because it previously said "done."**
Acceptance criteria are exactly those of experiment 9-7: cup in the tray, paper in the bin, arm back in a safe pose, no collision, no out-of-bounds motion. The difference: in the autonomous experiment the task semantics must come from the model's own observation, the real actions must come from tool calls, and the final state must be confirmed by a fresh observation; the human may only start the run, hit the emergency stop, and supervise safety — never complete an action on the Agent's behalf midway. Only then can experiments 9-7 and 9-9 be compared directly on "same hardware, same task — what is still missing between the human loop and the model loop."
(Google DeepMind, "Gemini Robotics-ER 1.5"; XLeRobot, "LLM Agent control." The upstream XLeRobot example shows orchestration; this section keeps the same orchestration principle but restricts action tools to calibrated desktop grasp, place, check, and stop primitives.)
- Real-hardware experiments expose calibration error, camera occlusion, and gripper failure, but are poorly suited to repeating large numbers of faults safely and controllably. The simulation experiments that follow keep these five tools and exactly the same task state, replacing only the real actuator with a desktop environment into which failures can be injected — to separate what open-loop execution, step-by-step checking, and action prediction each contribute.

### 6.5.4 VLA Control

- **VLA = Vision-Language-Action**. It takes the current frame and one skill instruction, then emits the action the robot should perform next:
  - `current observation + skill instruction → action`
- In the XLeRobot case the high-level planner only submits `pick(red_cup)`; the VLA or skill policy still has to decide, from the current frame, which direction to approach the cup from, when the gripper closes, and along what trajectory the arm lifts. After the execution layer finishes that short motion it photographs the desk again, and only once the cup is confirmed held may the planner submit `place(red_cup, tray)`. **A tool call defines the desired state change; the VLA defines how to realise that change through continuous motion.**
- **RT-2 and OpenVLA** cut continuous actions into discrete tokens and emit them one at a time, like generating text; **π₀** represents the other route, producing continuous, smooth action trajectories directly. Neither is simply better: discrete tokens combine more easily with language models, while continuous trajectories usually express smooth motion better. **The real trade-off is how the action should be represented, not merely model size.** (Moo Jin Kim et al. OpenVLA: An Open-Source Vision-Language-Action Model. arXiv:2406.09246, 2024.)
- A large model usually runs inference only **1-10 times per second**, whereas a traditional controller may update tens to thousands of times per second. A common engineering answer is **action chunking**: the model generates a short segment of future actions at once, a control thread executes that segment at a higher rate, and the model prepares the next segment in the background. This hides part of the inference wait inside the execution time. The cost: **the longer the segment, the smoother the motion but the fewer new frames the model sees during it** — if the cup is knocked while XLeRobot reaches for it, the arm may still be executing actions generated from the old frame. **Action chunking is a trade-off between smoothness and reaction speed, not free acceleration.**

### 6.5.5 The Limits of VLAs

- "Long-horizon planning + VLA" is a practical baseline, but several problems are easy to overlook:
  - **Limited training data**: robot demonstrations are far scarcer than internet text and images. That a model has seen the word "cup" does not mean it has seen cups of every material and friction condition.
  - **Imitation without consequence**: behaviour cloning mainly learns "what the demonstrator did next" and never explicitly requires the model to answer "what will this action cause."
  - **Robots differ**: different degrees of freedom, coordinate frames, grippers, and actuator latencies — the same action does not necessarily transfer to another machine.
  - **Observations go stale**: once an action chunk starts executing, an object may be moved, occluded, or knocked over while the model is still deciding from the previous frame.
- So a language model knowing what a "cup" is does not mean it knows how friction, contact, liquid sloshing, and a power cable will change the future state. **A VLA mainly answers "what should be done now"; another kind of model is needed to judge "what may happen afterwards."**

### 6.5.6 World Models

- A world model is an "**action-outcome predictor**": given the current state and some action, how the next state may change.
  - `current state + candidate action → predict the next state or a future segment → compare candidate outcomes → choose an action, replan, or stop safely`
- A world model usable for robotics must do at least three things well:
  1. understand the current state;
  2. predict the outcomes different actions may bring;
  3. pass those predictions to the planner or controller to help them choose.
- A VLM that can only describe video, or a model that can only generate frames, does not automatically become a reliable robot world model — it must also know what the actions are and predict their effect on objects and the environment. **V-JEPA 2** represents the route of predicting the future in an internal state; **World-Action Models** explicitly learn the "action–future observation" relationship. These models can work alongside a VLA; they need not replace it. (Meta AI, "Introducing the V-JEPA 2 world model and new benchmarks for physical reasoning," 2025-06-11; V-JEPA 2 technical report, arXiv:2506.09985.)
- In practical systems a world model is typically used in three ways:
  1. **Before acting**: compare candidates such as grasping, pushing, or waiting, and prefer the lower-risk option;
  2. **During execution**: compare the real observation against the prediction, and on divergence shorten the action, stop, or replan;
  3. **During training**: learn state transitions from video, simulation data, and failure trajectories, reducing trial and error on real hardware.
- Back to the XLeRobot desk task: if the yellow paper is partly hidden under the red cup, the system can compare candidate skills such as "grab the paper first," "move the cup first," and "approach from another direction." The world model does not need to generate photorealistic robot video — predicting which candidates are more likely to make the paper graspable and which might knock the cup over is already enough to help the planner rank them. **Once an action executes, the real camera observation remains the final truth; prediction can inform the choice but cannot replace acceptance.**
- What a world model gives is not a definite answer but a comparable prediction of "if I do this, what may happen." **The further ahead it predicts, the larger the error usually grows**, and a future frame that looks realistic may still violate real contact and friction. Practical systems therefore still need short-horizon prediction, real-time observation, an estimate of uncertainty, and an independent hardware safety controller. Generative world models can serve interactive simulation or visualisation, but **"can generate video" must not be conflated with "can guide robot action."** (Jack Parker-Holder and Shlomi Fruchter, Google DeepMind, "Genie 3: A new frontier for world models," 2025-08-05; Zachary Lin et al. Cosmos World Foundation Model Platform for Physical AI. arXiv:2501.03575, 2025.)

### Experiment 6-12 ★★: Comparing three autonomous desk-tidying loops in simulation
Put the task, object state, success conditions, and five tools of experiment 9-9 into the desktop simulator unchanged, replacing only the real XLeRobot actuator with a controllable simulated one, and let grasps occasionally suffer recoverable transient failures. This allows three strategies to be compared without changing the problem:
- **Open-loop execution**: generates the full action sequence once and never observes again midway;
- **Step-by-step checking**: re-reads the state after every pick and place and retries only the current skill on failure;
- **Predictive execution**: adds a short-horizon world model, comparing the expected outcomes of candidate skills before choosing the next step.
The experiment compares **task success rate, tool-call overhead, and failure-recovery ability**, and checks that every final success is confirmed by a fresh `verify_state` observation.
The point is not to prove a small simulated world model equals a real robot's physics model, but to verify a more basic relationship: an open-loop plan carries a single local failure all the way to the end of the task; step-by-step checking can recover; action prediction can further help rank candidate skills. **Whether the task is truly finished must still be decided by environment feedback.**

### 6.5.7 From Simulation to a Real Robot

- Even if experiment 9-10 is stable in the simulator, that does not imply the real XLeRobot of experiment 9-9 will succeed the same way. Going from simulation to a real robot is not a matter of swapping in yet another controller, but of handling the differences between two environments. Training may use teleoperation data, video data, or simulated interaction data; in real deployment the same red cup, yellow paper, tray, and bin appear against **different backgrounds, lighting, camera positions, and occlusion relationships**, and the arm additionally meets **different friction, sensor noise, and actuator latency**. Once those differences are large enough, motions learned in simulation may fail in reality.

### Experiment 6-13 ★★★: A cross-environment RGB test on the same desk task
Keep using the basic "move the object to its target" problem in simulation, treating each sample as one local decision within desk tidying: from the RGB frame, judge which direction to approach the object from, or whether it can already be grasped. Train **four visual policies with identical structure**: one sees only a fixed scene, one varies the background, one varies object appearance, and the last varies background, appearance, lighting, and noise together.
All policies are tested in the original environment and in the changed one, comparing **action-decision accuracy** before and after the visual conditions change. The question is not "is the simulator already equal to the real XLeRobot" but a narrower one: does actively widening the range of visual variation during training help the same cup–tray, paper–bin task adapt to a new camera view? Even if the result improves, real deployment still requires real camera calibration, actuator testing, and a complete safety loop. (LeRobot, "Sim2Real tutorial," GitHub: https://github.com/StoneT2000/lerobot-sim2real)

## 6.6 Chapter Summary

- Along the two axes of modality and execution timing: **asynchrony and event-driven execution** expand observation from "the Agent fetches it" to "the world pushes it," and action from "finish within the turn" to "start now and finish through later events." **Voice** compresses the scale to milliseconds, moving from turn-taking toward continuous listening and speaking while dividing realtime foreground interaction from deeper background thought. **Computer Use** moves the loop to the screen, where the bottlenecks include efficiency, continuous visual understanding, and state confirmation after actions. **Robotics** moves it into the physical world, where action chunking trades smoothness against responsiveness and completion must still be judged from a new observation.
- The four sections share one control skeleton: keep perceiving → judge current state and timing → choose a reply or an action → let the output enter the environment → observe the feedback → continue, correct, retry, stop, or replan.
- They also share the same **primitives — wake-up, safe points, cancellation, preemption, and fast/slow separation**.
- This chapter completes the last piece of the "building an Agent" part: the observation and action spaces have now been expanded in all three directions — **content, modality, and timing**. Next: Chapter 7 asks how to determine whether the system was built correctly; Chapter 8 explains how post-training updates model parameters; Chapter 9 organizes runtime trajectories, evaluation, and multiple update carriers into a continual-evolution loop; Chapter 10 moves from this complete single-Agent foundation to multi-Agent collaboration.

## Deep Thinking: Thought Questions

1. ★★ In an asynchronous Agent architecture, the priority strategy for the event queue must be determined at design time. But if priority judgment itself requires semantic understanding (e.g., determining whether a new message is more urgent than the current task), who should make this judgment — a rules engine or another LLM call? What are the costs of each?
2. ★★ In queue-based event processing, models tend to focus only on the last event. This chapter mitigates this through Agent status bar markers and summarization. But if the queue has 20 events backlogged (10 tool results + 5 user messages + 5 system alerts), how would you organize the presentation order and format of these events so that the model does not miss key information?
3. ★★★ When an Agent interacts with the external world on behalf of a user, it essentially faces an identity choice: use an independent virtual identity (dedicated email and phone number) to act as a third party, or directly operate the user's personal accounts as the user? The former allows autonomous background operation, but third parties may not trust a non-human identity; the latter has more complete context and permissions but introduces authorization, trust, and security-boundary issues. In what scenarios do you think each mode should be chosen?
4. ★★ The end-to-end model for voice Agents merges ASR-LLM-TTS into a single model, reducing latency but losing modularity. If the end-to-end model makes an error in a specific stage (e.g., speech recognition), debugging and fixing it is much harder than in a serial pipeline. How would you design an observability system for an end-to-end voice Agent?
5. ★ Step-Audio R1 achieves "thinking while speaking" through the MPS dual-brain architecture. However, humans, when "thinking while speaking," often say things before they have fully thought them through, self-correct, or use filler words. Should an Agent's "thinking while speaking" mimic these human characteristics?
6. ★★ SoM (Set-of-Mark) and its structured variants (DOM element indexing) convert Computer Use's visual localization from open-ended coordinate prediction to closed-set ID selection, but they all require detecting and annotating UI elements first — whether via a segmentation model or the DOM. If the interface contains non-standard controls or dynamically changing elements, the annotations may be incomplete or inaccurate. In such cases, should we fall back to coordinate prediction?
7. ★★ Thousand-dollar robot platforms like XLeRobot make teleoperation data collection inexpensive. However, the quality of teleoperation data depends heavily on the operator's skill. How would low-quality data from an unskilled operator affect the training of a VLA model? How can low-quality data be automatically filtered during the data collection phase?
8. ★★★ This chapter covers three interaction modalities: voice, Computer Use, and robotics. A common trend across these modalities is the evolution from serial pipelines to end-to-end models. If this trend continues, what might the Agent interaction layer look like in five years?
9. ★★ DOM/Accessibility Tree element indexing works well on standard web applications, but an increasing number of software interfaces (Canvas/WebGL rendering, cross-platform custom-drawn controls) do not provide accessible structured information, relying solely on visual annotation or coordinate prediction. Do you think Computer Use should bet on a purely visual approach, or maintain both structured and visual paths? What are the costs and benefits of maintaining both paths?
10. ★★ VLA models use action chunking — as mentioned in the text, π₀'s typical configuration generates 25-50 future actions at 50Hz — to hide inference latency within execution time. However, if the environment changes suddenly during execution (e.g., an object is moved), the pre-generated action sequence becomes invalid. How can we balance the efficiency advantage of action chunking with the need for responsiveness to environmental changes?
11. ★★★ All three scenarios in this chapter (voice, Computer Use, robotics) face the latency problem of the "perceive-think-act" loop and are evolving toward parallelized fast and slow thinking. In voice, this manifests as "correcting after misspeaking"; in Computer Use, as "clicking first, then looking"; in robotics, as "taking a step, then looking." How can we ensure that these actions based on fast thinking do not lead to irreversible consequences?
12. ★★★ The same set of primitives (wake-up, safe point, cancellation, preemption, fast/slow separation) recurs in this chapter at different time scales. Pick one and explain how its implementation differs between event-driven processing (seconds to days) and robot action chunking (milliseconds). What mainly determines that difference — the speed at which the environment changes, the reversibility of the action, or the cost of obtaining an observation?