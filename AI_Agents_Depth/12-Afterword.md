# Afterword: Back to Agent = LLM + Context + Tools

## The Formula and the Book's Arc

- The book opened with the formula `Agent = LLM + Context + Tools`; all ten chapters unfold within these three words.
- Chapter 1 establishes a **three-layer understanding of the formula** — the **implementation layer**, the **intuitive layer**, and the **academic layer** — and presents the **orchestration spectrum** from workflows to autonomous Agents.
- The following chapters unfold progressively along the sequence **"Building — Evaluation and Evolution — Collaboration."**

### Building Agents (Chapters 2–6)
- **Context engineering** determines what an Agent sees within a task; **memory and knowledge bases** extend information across sessions.
- **Tools** define what it can do; **code generation** provides the meta-capability to create new tools and systems.
- The **interaction chapter** pushes the observation and action spaces out of text turn-taking into voice, GUIs, and the physical world.

### Evaluation and Evolution (Chapters 7–9)
- **Evaluation** turns performance into trustworthy signals.
- **Post-training** writes high-dimensional capabilities into model parameters.
- **Continuous evolution** transforms production experience into controlled updates to knowledge, instructions, programs, or parameters.

### Collaboration (Chapter 10)
- **Multi-agent collaboration** further changes how context, tools, and responsibility are organized.

### Levels Are Not Independent Shelves
- Chapter 9 depends on all earlier foundations:
  - Without trajectories and knowledge systems, experience has nowhere to be stored.
  - Without code capability, an Agent cannot modify tools and Harnesses.
  - Without evaluation, the system cannot determine whether a modification is progress or regression.
- Chapter 9 is therefore the **convergence point** at which the book shifts from "how to build an Agent" to **"how to make an Agent improve over the long term."**

## Two Clouds

- In 1900, **Lord Kelvin** said two clouds still hung over the clear sky of physics — one later became **relativity**, the other **quantum mechanics**. The sky over Agents is hardly clear either; the author sees two clouds.

### Cloud 1: Streaming, Real-Time Interaction
- Today the vast majority of Agents operate in **turn-by-turn "request-response" mode**: you finish a sentence, it thinks through an entire paragraph, then spits out the result all at once.
- But the real world doesn't stop and wait — speech gets interrupted, the scene keeps changing, emails keep arriving.
- A truly **"living" Agent** should:
  - Listen while thinking and speak while thinking.
  - Start planning while you're still mid-sentence.
  - Notice on its own "this email needs handling" even when no one has asked.
- **Two paths toward this real-time capability**, often pursued in parallel:
  1. **Architectural separation of fast and slow** — real-time responsiveness and intelligence are nearly orthogonal axes a single model struggles to span; a **fast frontend model** keeps the conversational rhythm while a **slow backend model** does the deep thinking.
  2. **Making inference itself faster** — when decode speed is high enough, the turn-by-turn wait becomes so short it nearly disappears, blurring the line between turn-by-turn and "real-time." Rapidly advanced by chips and inference engines:
     - **Xiaomi MiMo** pushed a 1T-parameter model past **1000 token/s on a single 8-GPU node** (footnote: Xiaomi MiMo-V2.5-Pro-UltraSpeed, via model-system co-design with FP4 quantization, DFlash parallel speculative decoding, and the TileRT inference system; first time a 1T-parameter model passed 1000 token/s on a single general-purpose 8-GPU node. See Xiaomi MiMo official blog, "Pushing 1T-Parameter Model Generation Speed to 1000 TPS," 2026, https://mimo.xiaomi.com/blog/mimo-tilert-1000tps).
     - **Taalas HC1** — hardcodes the entire Llama 3.1 8B model onto a 6nm chip, reaching ~**17000 token/s** with a response time under **100 milliseconds** (footnote: the trade-off is the chip can only run the hardcoded model, and model updates require a new chip tape-out. See Karl Freund, "Taalas Launches Hardcore Chip With 'Insane' AI Inference Performance," Forbes, 2026).
  - When a model can spit out thousands of words per second, the experiential gap between "think, then speak" and "think while speaking" is **simply erased**.

### Cloud 2: Continuous Accumulation of Experience
- Can Agents, like humans, keep accumulating experience from the successes and failures of their interactions with the environment?
- Today's models are like **"a genius with a superb memory who can't learn anything new"**: during training they memorize human knowledge down to the last detail, but once on the job they barely grow — after each task, the pitfalls they've stepped in and the tricks they've figured out are **mostly discarded along with the context**.
- Whether this is a real problem depends on **two opposing hypotheses**:

  - **"Small World Hypothesis"**: a sufficiently large model — say, trillions of parameters — already contains almost all the important general knowledge of the physical world; **learning once is enough**. Holders (including researchers at OpenAI and Anthropic) argue:
    - Programming is the one domain where AI is strongest today — not because code is special to models, but because programming is humanity's **most open field**: vast amounts of open-source code sit ready to be learned, while most industries have no public information or data at all.
    - So frontier labs are going **industry by industry**, partnering with each to "distill" its professional capability into the same large model.
    - The bottleneck is neither the model's capacity nor its ability to learn, but **whether there's enough data** — feed the data in, train once, and the problem is solved.

  - **"Big World Hypothesis"**: there is a layer that cannot be supplied by "training once" — **knowledge specific to a particular user or company**:
    - A company's coding standards and presentation style, along with a particular client's distinctive temperament, are **absent from every training corpus and change constantly**.
    - To fit this "big world" of countless specific situations, a model **must continue learning after deployment**; it cannot arrive from the factory with everything configured.
    - This is precisely the direction explored by **memory in Chapter 3** and **continuous evolution in Chapter 9**: should experience be written into knowledge documents, instructions, or programs, or should selected experience be used to update model parameters?
    - More broadly, both **"RSI (recursive self-improvement)"** and **"AI for Science"** push Agents toward frontiers where no ready-made answers exist; there an Agent can only learn autonomously from repeated experimental successes and failures rather than turning back to humans for every decision.
    - The model's strongest capability will ultimately be **not memorization, but learning and adaptation**.

- Neither cloud will be blown away by any single model upgrade. To understand how they will eventually be dispelled, one thing must be seen clearly: **models and Agents have never been upstream and downstream of each other; they move forward together.**

## The Co-Evolution of Models and Agents

- Look back at the layers of fallback logic in the harnesses — multi-level context compression, retry logic that trips a circuit breaker only after thousands of failures, permission checks that pessimistically default to "unsafe" — every stretch of seemingly ugly **"spaghetti code" records a place where the model is still shaky**.
- When the next generation of models internalizes these constraints, the corresponding code can be deleted; and the reason models can internalize them is precisely that **Agents have already stumbled through those pits on the model's behalf in real business**, condensing the lessons into signals for the next round of training.
- The loop: **users pose real challenges → the application layer uses harnesses to patch over what the model can't yet do well → those patches become training signals for the model's next iteration.** This is a **self-reinforcing flywheel**.

### Will Models Eventually Eat the Harness? (answers the question left hanging in Chapter 1)
- The book's view: **yes — but not all at once**. Models will eat it **layer by layer, and the process will never be complete**.
- Every capability a model stably internalizes lets the corresponding Harness layer be deleted — **Chapter 6's interaction model** is one such case: behaviors like **interruption and interjection** that once had to be assembled with an external harness are now built directly into the model.
- Why the "eating" will never be finished:
  1. **Training takes months** — the model can wait, but the business cannot.
  2. A model cannot internalize every constraint and preference of real business; there is always a **newest boundary** that needs external logic as a backstop.
  3. **Every generation of models opens a new capability frontier**, and the frontier is exactly where the model is least reliable.
- So the Harness will **not disappear**; it simply keeps migrating, together with the model, toward each new frontier.
- This is how the **Bitter Lesson** reads in the Agent era: *general methods will win in the end — but every stretch of road inside that "in the end" is paved by the Harness.*

### The Flywheel as a Moat
- The flywheel spins fastest in the hands of those who hold **both ends**.
- What Anthropic is doing with **Claude Code** is exactly this: letting its own model and its own harness feed each other and co-evolve. The model knows how the harness will call it; the harness knows where the model's boundaries lie; every change on either end feeds back to the other immediately.
- In one experiment, changing nothing but the harness — same model — lifted task accuracy from **52.8% to 66.5%**. That shows how much leverage the harness has today, but also a reminder: the harness has that much leverage **precisely because the model hasn't gotten there yet**.
- For the same reason, **this flywheel itself is the deepest moat of this era**: the tighter real business, feedback data, and model iteration mesh together, the harder it is for anyone to catch up from the outside.

### What This Means for You
- **If you're building models**: the moat is to get this flywheel spinning — let feedback from real-world scenarios flow back into training as quickly as possible.
- **If you're building applications on top of models**: the harness is your sharpest short-term technical lever, but be clear-eyed — each time the model internalizes a layer of constraints, it will (almost in passing) **wipe out a batch of advantages built on the harness alone**.
- The truly lasting moats at the application layer usually lie **outside technology**: exclusive data, solid distribution channels, user trust, network effects, and the physical-world scenarios that require humans and Agents to work together.
- The prudent play: **use the harness to buy time, and use that time to build barriers beyond technology.**

## Closing

- Don't fret over whether the framework in your hands will become obsolete. Models iterate every few months; specific APIs, products, and leaderboards will all turn over.
- But **the three questions — what it sees, what it can do, and how to verify it's doing things right — will not become obsolete**. They describe not the usage of a particular model but the **fundamental way an intelligent system interacts with the world**. Master them, and whatever new capability the next generation brings, you'll know where it fits in the formula — and you'll see at a glance how far it still is from blowing those two clouds away.
- Agent technology is still evolving at full speed; no single book can keep up with every change. But if this book leaves you not the specific usage of an API but **the judgment to stay clear-headed amid the waves of technology**, it has fulfilled its mission.
- All of the book's text, illustrations, and companion experiment code are **open source** — visit the repository, run the experiments yourself, and submit issues and PRs.
- The most fascinating thing about Agents: they **can create new capabilities by writing code, and even improve themselves**. Having read this far, you already hold the principles of "creation." Now, go create something.