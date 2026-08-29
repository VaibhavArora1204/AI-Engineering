# Introduction

## The Book's Origins: AI Agent Bootcamp & Whisper Coding

- Aug–Oct 2025: author delivered technical lectures at the "AI Agent Bootcamp" run by **Turing**, the Chinese tech publisher.
- Goal: move AI Agent design from **intuition-driven** to **principle-driven** — not just getting a demo running, but understanding *why* an Agent is designed the way it is and *what trade-offs* lie behind each architectural decision.
- Book was compiled and expanded from the notes and experiments of those lectures.
- The book itself, from first idea to finished manuscript, was created through **whisper coding** (dictation-based collaboration). Dictation tool: **Pine**, the author's own voice Agent.
- Workflow per lecture: dictate a rough outline → have Pine survey the topic → assemble a first draft. After the lecture, share Bootcamp students' feedback with Pine, discuss and polish over many rounds; iteration by iteration, lecture notes grew into the book.
- Throughout, author rarely typed — spoke thoughts aloud instead. Speech has far higher bandwidth than typing (normal speech runs about **four times typing speed**), so the "dictate—survey—discuss—revise" loop moved quickly.
- The book is not only *about* Agents; it is also a work an **Agent helped create**.

## The DeepSeek R1 Era: Model & Product Layer Progress

- Since the release of **DeepSeek R1** in early 2025, the AI field has moved beyond the pure foundation model stage (i.e., general-purpose large language model backbones) into the deep end of **engineering deployment**.
- **Progress at the model layer** — visible in two directions:
  1. Through **Agentic Reinforcement Learning**, models have trained tool-calling capabilities into their parameters, mastering general abilities in areas like coding, mathematics, and computer use.
  2. Model iteration keeps accelerating — GPT-5.2 to GPT-5.5, Claude Opus 4.5 to 4.8, each leap taking just **half a year**.
- **Progress at the product layer**: general Agents like **Manus, Claude Code, and OpenClaw** have redefined human-computer interaction, pushing the architectural paradigm of **"code generation + file system"** into the mainstream.
- Agent architecture principles summarized in the course nearly a year ago have not gone stale — they have become *more canonical*.
- New terms — **Skill, harness, loop engineering** — have swept the Agent industry, but the real sequence runs the other way: companies like **Anthropic** did not invent these concepts and watch the industry adopt them; a great many Agents were already doing these things, and Anthropic then distilled the practice into named design principles.

## Practice Comes First, Naming Comes Later

- The confidence behind these principles comes from pushing Agents into **long-horizon, high-stakes** work in the real world.
- As **Chief Scientist of Pine AI**, the author built Pine with his team. To his knowledge, Pine is the **first general Agent** that can autonomously interact with real people and reliably handle sensitive, complex, long-horizon tasks involving **money** on its own:
  - Calls carriers on behalf of users to negotiate bills
  - Handles refund requests and complaints with merchants
  - Cancels subscriptions
  - All without a human stepping in
- Such tasks routinely run to **dozens of rounds of negotiation**, and a single mistake means real financial loss. This unforgiving demand for reliability hammered out, one by one, the architectural principles the book keeps returning to.
- Examples from that practice:
  - **Long before Skill became a buzzword**: already using **dynamic prompt loading** to rein in runaway prompt growth, **command-line execution tools** to rein in runaway tool lists, and a **system status bar** to give the Agent awareness of its execution environment, the user's time, and its own work status.
  - **Long before harness became a buzzword**: already using Claude Code-style methods to handle unstable tool calls, hallucinations, dangerous operations, unauthorized operations, and ignored instructions.
  - **Long before loop engineering became a buzzword**: already using what this book calls the **proposer-reviewer method** to keep the model from declaring a task finished too early.
- None of this is exclusive invention: most leading model and Agent companies have worked out similar methods on their own.
- That's why the author launched the "AI Agent Bootcamp" course at Turing in **August 2025**, and taught a hands-on AI Agent course at the **University of Chinese Academy of Sciences** continuously from **2024 to 2026** — and why he chose to release this book **open source** rather than keep it closed and collect royalties: to let this knowledge reach more practitioners.

### Practical Lesson for Enterprise Agent Development

- If you wait for an Agent buzzword to catch on before putting it into practice, you are already a step behind. By the time a term is trending, the leading companies have usually worked through the problem it names.
- **Two things are key** to knowing what to do before the name arrives:

1. **Have a real business that makes extreme demands on the Agent's capability ceiling, and keep drawing genuine feedback from it.** Example (Pine): a single task often takes hours or even weeks and can mean repeated back-and-forth with multiple stakeholders — hours on the phone, pages of complex forms filled out on a computer, several rounds of email. Through it all, not one number can be wrong, and every exchange must stay careful enough to protect the user's interests. Only immersed in a scenario this complex does practice naturally push you to build harnesses — to cover what the model cannot yet do on its own but the business requires. Conversely, if the business asks little of the capability ceiling and a modest model upgrade is all it takes, you will never have reason to hone these architectural principles.

2. **Build an Evaluation mechanism.** The book returns to this point again and again: **without evaluation, there is no progress**. Evaluation lets you tell whether a change is genuinely better or merely lucky, so the Agent's direction of iteration no longer rests on gut feeling. At bottom, what is advocated is using **the scientific method to do engineering and build Agents** — evaluation is that method's foundation. Chapter 7 develops it in full.

- However the underlying models advance and however product forms shift, almost all successful Agent systems follow the same architectural patterns. This is no coincidence: good design principles should outlast model iteration cycles, because they describe not the usage of a specific model but the **fundamental patterns by which intelligent systems interact with the world**.

## Richard Sutton's Four Stages of Cosmic Evolution

- **Richard Sutton**, Turing Award winner and father of reinforcement learning, once said the evolution of the universe has gone through four stages: **dust, stars, life, and agents** (originally termed *designed entities*).
- Biological evolution is blind: random mutation, natural selection. Most organisms do not understand their own working principles and cannot design or remake living things on their own.
- Agents are something entirely new in the history of cosmic evolution: by generating code they can **bootstrap and evolve themselves**, like a programmer who writes another programmer, who in turn writes the next.
- That is, an Agent can understand its own operating mechanism and, given a goal, create entirely new Agents — even improve itself.
- The mission of this book: help you understand and master the principles of this creation.

## The Core Formula: Agent = LLM + Context + Tools

- The core formula of the book is one sentence: **Agent = LLM + Context + Tools**. All three are indispensable.
- More intuitively: **Brain + Eyes + Hands and Feet**.
  - **Brain (LLM)**: responsible for thinking and decision-making
  - **Eyes (Context)**: determines what information the Agent can see
  - **Hands and Feet (Tools)**: determines what the Agent can do
- Caveat: strictly speaking, "eyes" is a rough analogy — Context includes not only environmental information and conversation history but also **tool definitions**, meaning the information the Agent "sees" also includes "what hands and feet are available." The metaphor conveys the core intuition: **Context is all the information the model can perceive**.
- **RL mapping** (for readers familiar with reinforcement learning): LLM corresponds to **Policy**; Context corresponds to **Observation Space**; Tools correspond to **Action Space**. The three expressions refer to the same object, just at different levels of abstraction.

### Figure 0-1: Agent = LLM + Context + Tools (diagram breakdown)

- **Agent** (Autonomous Decision System) = LLM + Context + Tools
  - **LLM: Brain** — Understanding · Thinking · Planning · Decision
  - **Context: Eyes** — Instructions · Memory · Knowledge · Trajectory
  - **Tools: Hands & Feet** — Perception · Execution · Collaboration · Code
- Chapter mapping around the diagram:
  - **Ch. 6 Evaluation · Ch. 7 Post-Training** — Model as Agent · SFT · Reinforcement Learning. Evaluation Runs Through the Entire Process.
  - **Ch. 2 Context Engineering · Ch. 3 Knowledge Base** — Prompt Engineering · KV Cache · Compression · Memory; RAG · Structured Indexing · Agentic RAG
  - **Ch. 4 Tools · Ch. 5 Code Generation** — MCP · Asynchronous Event Architecture · Tool Security; Code as Thinking · Agent Bootstrapping
  - **Ch. 8 Self-Evolution · Ch. 9 Multimodal · Ch. 10 Multi-Agent** — Learning Paradigms · Tool Creation · Voice · Robotics · Collaboration

## Book Structure and Chapter Summaries

- Ten chapters organized across **four levels** (Figure 0-2).
  - **Chapter 1**: establishes the foundational framework for Agents.
  - **Chapters 2–6**: how to build Agents — covering context, knowledge, tools, code generation, and interaction in sequence.
  - **Chapters 7–9**: how to evaluate and continuously improve Agents — progressing from measurement systems and model post-training to continuous evolution driven by operational experience.
  - **Chapter 10**: broadens the scope to multi-agent collaboration.

### Figure 0-2: Book Structure — Building, Evaluation and Evolution, Interaction and Collaboration

- **Chapter 1: Agent Fundamentals** — Three Pillars of Agent · Design Patterns · ReAct
- **Chapter 2: Context Engineering** (Context = Core) — Prompt Engineering · KV Cache · Compression · Skills · Status Bar
- **Chapter 3: User Memory and Knowledge Base** — User Memory · RAG · Structured Index · Agentic RAG (under Tools)
- **Chapter 4: Tools** — MCP · Tool Safety · Asynchronous Event Architecture (under Tools + Model)
- **Chapter 5: Coding Agent and Code Generation** — Coding Agent · Code as Thinking · Agent Bootstrapping (under Model)
- **Chapter 6: Evaluation** — Benchmark · LLM-as-Judge · Model Selection (under Model / core)
- **Chapter 7: Model Post-Training** — SFT · Reinforcement Learning · LoRA · Tool Calling RL (Advanced Topics and Applications)
- **Chapter 8: Agent Self-Evolution** — Learning Paradigms · Experience Learning · Tool Discovery and Creation
- **Chapter 9: Multimodal and Real-Time Interaction** — Voice · Computer Use · VLA Robotics
- **Chapter 10: Multi-Agent Collaboration** — Shared Context · Manager · Decentralization

### Chapter-by-Chapter Description (2–4 lines each)

- **Chapter 1 (Agent Fundamentals)**: Begins with several real Agent products to build an intuitive understanding of Agents. Examines the Agent's core formula in depth: from the implementation layer of LLM + Context + Tools, to the intuitive layer of Brain + Eyes + Hands and Feet, and finally to the academic layer of Policy, Observation Space, and Action Space. Experiments dissect how the ReAct loop operates — the iterative process of "Think → Act → Observe" — and distinguish among within-task contextual adaptation, cross-task updates to external artifacts, and parameter updates during training cycles. Concludes with orchestration design patterns ranging from workflows to autonomous Agents, establishing a unified conceptual framework for the chapters that follow.

- **Chapter 2 (Context Engineering)**: The most critical chapter in the book, systematically explaining Context, the Agent's "eyes." Begins with the API message structure and the Agent's core loop, establishing the foundation that "Context is a list of messages," then examines the underlying principles of KV Cache — the mechanism for reusing historical computation results during LLM inference. Subsequently covers Prompt Engineering (including process-oriented design, tool descriptions, and refinement of business rules), Prompt Injection attacks and defenses, the on-demand loading mechanism of Agent Skills, Agent status bar technology, and Context Compression strategies. Complete definitions of these terms are provided when they first appear.

- **Chapter 3 (User Memory and Knowledge Base)**: Extends context management into a persistent knowledge system spanning sessions, allowing the Agent not only to remember the current conversation but also to accumulate and retrieve knowledge across multiple conversations. Covers four progressive strategies for user memory; the complete technical stack of RAG (Retrieval-Augmented Generation, in which relevant documents are retrieved before the model generates an answer), including different text-search methods and ranking optimization; multimodal information extraction; more advanced methods of knowledge organization; and Agentic RAG, in which the Agent autonomously decides when and what to retrieve.

- **Chapter 4 (Tools)**: Examines the bridge between Agents and the external world: tools function as the Agent's "hands and feet," enabling it to search the web, call APIs, operate databases, and more. Introduces the MCP tool-interoperability standard and design principles for five categories of tools — Perception, Execution, Collaboration, Event Triggering, and User Communication — covers three approaches to multimodal perception (native multimodal processing, conversion to text, and tool-based multimodal analysis), and emphasizes security mechanisms for execution tools. Event triggering, user communication, and the asynchronous runtime are covered in Chapter 6.

- **Chapter 5 (Coding Agent and Code Generation)**: Argues that a Coding Agent plus a file system constitutes the core technical foundation of every general-purpose Agent. Using the OpenClaw architecture as its organizing thread, it analyzes Coding Agent workflows and implementation techniques, while demonstrating the broad value of code generation beyond programming: from assisting thought and building knowledge bases to dynamically creating new tools and Agent bootstrapping.

- **Chapter 6 (Interaction: Expanding the Observation and Action Spaces)**: Removes the turn-taking premise the previous five chapters assumed, and expands the Agent's interaction with the world along two axes — modality and timing: async and event-driven processing lets the world wake the Agent (event-triggered tools, user communication tools, virtual identities and isolated execution environments); voice pushes interaction to the millisecond scale (from cascading pipelines to end-to-end omnimodal and full-duplex models); Computer Use lets an Agent operate graphical interfaces like a human; and robotic manipulation extends action into the physical world (VLA control and Sim2Real transfer). The four scenarios share one set of primitives — wake-up, safe point, cancellation, preemption, and fast/slow separation.

- **Chapter 7 (Agent Evaluation)**: Constructs a scientific evaluation methodology. Covers evaluation environments — two core paradigms, tool calling and human-computer interaction, plus simulation environments discussed separately at the end of the chapter — dataset design principles, LLM-as-a-Judge automated evaluation, evaluation-driven model selection, and the complete closed loop that converts evaluation results into system improvements.

- **Chapter 8 (Model Post-training)**: Examines two post-training techniques in depth: SFT (Supervised Fine-Tuning, using labeled data to teach the model to follow examples) and RL (Reinforcement Learning, allowing the model to improve autonomously through trial and error and reward feedback). Organized around the central arguments that "SFT memorizes, RL generalizes" and "Data and environment matter more than algorithms," it covers the full pre-training/SFT/RL landscape, classical RL theory, reward-signal design — from binary rewards and process rewards to verified path penalties that "reward the outcome and constrain the process" — single-turn and multi-turn reinforcement-learning algorithms, and frontier topics such as sample-efficiency optimization.

- **Chapter 9 (Continuous Agent Evolution)**: Studies how to transform an Agent's operational experience into capabilities for its next version. First establishes learning signals composed of environmental outcomes, process rules, and LLM Rubrics; then compares four update carriers — knowledge documents, Prompts and Skills, programs and Harnesses, and model parameters; and finally discusses candidate-version validation, canary releases, rollback, and long-term consolidation.

- **Chapter 10 (Multi-Agent Collaboration)**: Discusses the ultimate form of AI Agent systems: how multiple Agents divide work and collaborate. Systematically presents a classification framework for multi-agent collaboration — shared/independent context × peer/manager/decentralized structures — uses translation Agents and telephone-plus-computer Agents to demonstrate collaborative architecture design, and surveys frontier directions in Agent societies and Agent economies.

## How to Read This Book

- The chapters are relatively independent. Choose different reading paths based on your needs:
  - **If you are an Agent developer**: read Chapters 1 through 9 in order. The first six chapters present construction methods, Chapter 7 establishes the foundations of evaluation, Chapter 8 explains how to train models, and Chapter 9 integrates parameters and other update carriers into a complete continuous-evolution loop. Chapter 10 can be read selectively according to your needs in multi-agent collaboration.
  - **If you have limited time**: prioritize Chapter 1, for the overall picture, and Chapter 2, for context engineering — the most critical skill. The underlying principles of KV Cache in Chapter 2 are fairly technical; on a first reading, you may skip the mechanics and retain only the three core conclusions stated at the beginning, without affecting your understanding of later material.
  - **If your focus is model training**: go directly to Chapter 8 (Model Post-training). Because the evaluation methods in Chapter 7 are a prerequisite for training, read the two together, after Chapters 1 and 2 have established the overall picture.
- Each chapter contains a large number of experiments and thought questions, numbered in the format "Experiment X-Y" (X = chapter number, Y = sequence number within the chapter).
- **Star ratings** for difficulty on experiment/thought-question titles:
  - ★ = entry-level, suitable for all readers
  - ★★ = medium difficulty, requiring some engineering practice
  - ★★★ = an advanced challenge, usually involving open-ended questions or complex system design
- Most experiments come with complete runnable code, organized in the accompanying open-source repository.

### Companion Code Repository

- **URL**: https://github.com/bojieli/ai-agent-book
- Obtain all companion code with Git:
  - `git clone https://github.com/bojieli/ai-agent-book.git`
  - `cd ai-agent-book`
- Without Git: select **Code → Download ZIP** on the repository page.
- Experiment code is organized by chapter under `chapter1/` through `chapter10/`. To find "Experiment X-Y": open the corresponding `chapterX/README.en.md`, use the experiment number to locate its project directory, then follow that project's README to install dependencies and run it. Some experiments marked as reproduction guides depend on external repositories; their READMEs explain how to obtain those as well.
- Strong recommendation: run these experiments yourself — AI Agents are an intensely hands-on field, and many design intuitions only take shape while you are debugging.

### Terminology Note: Reasoning vs Inference

- The book is careful to distinguish two terms casual usage often conflates:
  - **Reasoning**: the model's step-by-step thinking — as in reasoning models, chain-of-thought, and reasoning tokens.
  - **Inference**: the model's forward computation at deployment time — as in inference cost, inference stack, and inference-time scaling.
- The Chinese edition renders them as two distinct words — **思考** for reasoning, **推理** for inference — precisely to keep the two ideas apart; Chapter 10's translation case study refers back to this convention.
- Other key terms are defined at their first occurrence in the text.

## Prerequisites

- Aimed at readers with some technical background, but does not require being an expert in a specific field. Listed in two levels: "Required" and "Recommended", to help assess readiness.

### Required: Foundation for reading the entire book

- **Python Programming**: Almost all experiments in the book are based on Python. Need familiarity with Python's basic syntax, common data structures, package management (pip), and other fundamental concepts. Proficiency is not required, but you should be able to read and modify Python code of moderate complexity.
- **Basic Experience with LLMs**: Should have used ChatGPT, Claude, or similar products, and understand the basic interaction pattern of "Prompt → Model Response".
- **An AI-Assisted Programming Tool**: Install and get comfortable with at least one AI-assisted programming tool, such as Claude Code, Codex, Cursor, or TRAE. These tools greatly speed up the experiments (which involve extensive code writing and debugging), and they are themselves mature Coding Agents — using one gives firsthand experience of this book's core mechanisms: the ReAct loop, tool calls, context management. That firsthand experience is invaluable for understanding Agent design principles.
- **General Software Engineering Knowledge**: Familiar with basic concepts such as command-line operations, Git version control, JSON data format, and REST APIs. These are the foundation for running experiments and understanding the Agent's tool-calling mechanism.

### Recommended: Enhancing the reading experience for specific chapters

- **Machine Learning Basics (Chapter 8)**: Understanding basic concepts like training vs. inference, loss functions, gradient descent, and overfitting will help you understand model post-training.
- **Basic Mathematics (Chapters 2-3, 8)**: An intuitive understanding of linear algebra (e.g., vectors can represent direction and magnitude, matrices can perform batch operations) helps you understand embeddings and attention mechanisms. Basic knowledge of probability and statistics helps with evaluation metrics and expected rewards in reinforcement learning. The mathematics in the book does not involve complex derivations and focuses on intuitive explanations.
- **Web Development Basics (Chapter 6)**: Understanding concepts like HTTP, WebSocket, and front-end/back-end separation architecture will help you understand event-driven asynchronous Agent architectures and real-time communication experiments for voice agents.
- **Basic Understanding of the Transformer Architecture (Chapters 2, 8)**: Transformer is the underlying architecture of almost all current large language models. Readers who want to systematically build foundational knowledge of large models may enjoy **Illustrating Large Models** (图解大模型, published in Chinese by Turing), which uses intuitive diagrams to explain core concepts such as the Transformer architecture, pre-training, and fine-tuning — a good complement to this book's Agent engineering perspective.
- If you are missing some prerequisites, don't be discouraged. The core value of this book lies in architectural design principles and engineering methodology, not in any specific algorithm or technique. Outside of Chapter 8 on post-training, the book requires very little mathematics or machine learning, and it works perfectly well as a starting point.
- Agent technology is still evolving rapidly, but good architectural design principles have the power to endure through time. Master the "why" behind the designs, and you can keep your judgment clear as the waves of technology roll through. The book aims to be a reliable guide as you build AI Agents.

## Acknowledgments

- Thanks to editors **Meng Ge** and **Liu Meiying** at Turing for their diligent editing and for organizing the Turing "AI Agent Bootcamp" course.
- Thanks to **Professor Liu Junming** for offering the hands-on AI Agent course at the University of Chinese Academy of Sciences.
- Special thanks to all students of the Turing "AI Agent Bootcamp" and of the UCAS AI Agent course — teaching these courses brought valuable feedback and advice, and sharpened the author's own grasp of these concepts.
- Thanks to all colleagues at Pine AI — without a product as excellent as Pine AI and the many challenges it brought, the author could never have gone so deep into the Agent field, and colleagues contributed a wealth of valuable insights.
- Thanks to many friends across the AI industry (unnamed) who gave honest feedback, corrected misjudgments, and raised understanding of models and Agents.
- Most of all, thanks to family, especially wife **Meng Jiaying**, who supported the writing and offered many valuable suggestions.
