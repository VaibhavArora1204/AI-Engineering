# Chapter 8 Model Post-Training

Core formula of book: Agent = LLM + Context + Tools. This chapter turns to the LLM itself — the "brain."
- Use **Mid-training** to fill gaps in domain knowledge and foundational capabilities.
- Use **post-training** methods such as **SFT** and **RL** to shape how the model uses context and tools.
- Chapter 7 ended noting that the evaluation system and simulation environment are the two cornerstones of post-training: the evaluation environment gives training its practice ground; evaluation metrics give it its target.
- This chapter discusses how to actually change model weights — how to bake capability into the parameters.
- Assumes no background in RL or model training (no gradients/policy optimization required). Starts from: how a model gets trained at all.
- By chapter end, reader should answer: At what stages are model capabilities formed? What does each stage do? How are stages commonly combined, and when can order differ? Where to focus effort in your own projects?

Most important map: modern model capability development divides into **four parts**:
1. **Pre-training**: Training on massive internet text to "predict the next token." Teaches language rules, world knowledge, basic reasoning. Like a person who read all books in a library — erudite, but not yet good at answering questions. Most expensive step (often tens of millions of dollars); foundation of all capabilities.
2. **Mid-training** (intermediate training or continued pre-training): From an existing base model, continue language modeling on target-language data, domain documents, code, long contexts, or deliberately designed capability data. Does not rebuild the foundation from scratch; fills in "textbook chapters" general pre-training covered poorly. Uses less data and compute than full pre-training; better suited than SFT to absorbing large bodies of knowledge and forming basic representations a task requires. Some teams treat Mid-training as the latter part of pre-training; others call it Continued Pre-training (CPT), Domain-Adaptive Pre-training (DAPT), or Task-Adaptive Pre-training (TAPT).
3. **Supervised Fine-Tuning (SFT)**: Training on labeled input-output pairs, like a teacher giving a student standard answers to imitate. Thousands to tens of thousands of Q&A demonstrations teach format, style, process. Transforms a knowledgeable/capable model into an assistant that understands instructions and produces well-structured outputs. Cheap, fast, stable; almost all deployed models undergo it.
4. **Reinforcement Learning (RL)**: Letting the model try repeatedly and improve from rewards and penalties, like reviewing exercises by their scores. Instead of directly imitating tokens of a standard response, RL lets the model try on its own, increasing probability of good behavior and decreasing poor behavior. When base model occasionally succeeds, and rewards/data/environment are well designed, can improve decisions in unseen situations — also the step that takes up most space in this chapter and requires the most engineering.

Analogy: Pre-training = general education; Mid-training = intensive study of specialist textbooks; SFT = teacher demonstrating solution and communication conventions; RL = working problems yourself and refining approach from outcomes.

**Two main threads** running throughout (all subsequent content serves them):
- **Thread One**: In this chapter's controlled experiments, SFT tends to memorize demonstrations while RL generalizes better. Under same task, model, budget in GeneralPoints and V-IRL, SFT overfits training answers while RL more often learns a transferable strategy under tested distribution shifts. This is a measured result under those conditions, NOT a universal property: SFT can generalize with diverse data + appropriate regularization; RL can overfit when reward/environment is biased. Chapter uses "SFT memorizes, RL generalizes" as shorthand. Section "From Pre-training to RL: A Four-Part Panorama" explains why the two objectives can produce that difference.
- **Thread Two**: Data and environment matter more than algorithms. Industry's most counterintuitive and most valuable lesson. With off-the-shelf RL (PPO, GRPO), knowing how to use them is enough. What determines success: (a) whether Mid-training corpus repairs the foundation; (b) whether demonstration data establishes a behavioral protocol; (c) whether simulation environment and reward provide reliable trial-and-error feedback. In many scenarios, if first two kinds of data are good enough, RL is not needed at all. Redirects attention from "which algorithm should I tune?" to "have the data and environment been set up correctly?"

**Reading Guide** — two paths:
- Agent Application Developers (don't need to train models): Read the opening panorama for global understanding; skip the two [Optional Reading] sections on classic RL and pre-training background; continue from standalone Mid-training section. Focus on decision framework for choosing Mid-training/SFT/RL and judgment "data and environment more important than algorithms."
- Model Training Engineers: Read sequentially from the beginning. The two [Optional Reading] sections provide complete background on RL and pre-training; subsequent experiments provide reproducible training schemes.

## 8.1 From Pre-training to RL: A Four-Part Panorama

The four parts differ in data, optimization objectives, and costs. Understanding those differences is key to the entire chapter. Table 8-1 gives overview.

### Table 8-1 The Four Parts of Model Capability Development
| Stage | Data Used | Optimization Objective | What Is Learned | Typical Cost |
|---|---|---|---|---|
| Pre-training | Massive raw internet text | Predict the next token | Language rules, world knowledge, basic reasoning | Very High (millions to tens of millions USD) |
| Mid-training | Target-language/domain/capability corpora plus retention data | Continue next-token prediction (usually with loss on every token) | Fill gaps in domain knowledge, language, and foundational capabilities | Medium to high, depending on token volume and whether all parameters are trained |
| SFT | Thousands to tens of thousands of "input-output" demonstration pairs | Predict the next token (loss calculated only on the response) | Instruction following, output format, style, process protocol | Low (hours to days) |
| RL | Task and environment + reward signal (reference answers optional) | Maximize expected reward | Transferable decision-making strategy, newly discovered solutions | High (often tens to hundreds of times that of SFT) |

### 8.1.1 What Pre-training Does: Predicting the Next Token
- All "intelligence" of modern large models built on **Next Token Prediction (NTP)** — a task so simple it's surprising.
- Show model the first part of text, have it guess next token. E.g., input "The capital of China is" → high probability on "Beijing." Each guess compared to actual next token; larger difference (the **loss**) → more parameter adjustment to guess more accurately in similar contexts next time.
- Do this on trillions of internet tokens → model forced to learn grammar, facts, logic, even basic reasoning — no shortcut; it must "digest" patterns in text.
- Key point (carries through to Mid-training, SFT, RL): **The model's output is essentially a probability distribution.** Given preceding text, model assigns probability to every possible token in its vocabulary. "Training" at core is adjusting this probability distribution — making desired tokens higher, undesired lower. The four parts differ only in "what is desired" and "what signal defines desired."
- After pre-training, model is erudite but not user-friendly: asked a question it might keep generating questions — because in internet text, a question is often followed by another question. Hasn't learned the protocol "when asked a question, you should answer."

### 8.1.2 The Essence of Mid-training: Continue Learning on the Target Distribution
- General pre-training cannot cover every language, domain, and capability. If a model can barely read Korean documents, doesn't understand enterprise internal protocols, or never formed code/long-context representations required by the target task, it is too late to teach only "how to answer" or reward only success/failure.
- Mid-training retains pre-training's next-token objective but narrows the data distribution to the target domain and mixes in general retention data to control forgetting.
- Asks whether the model possesses the knowledge and foundational capabilities needed to complete the task — not what the response should look like or which policy earns highest reward.
- Mid-training vs SFT: may appear to use similar loss functions, but data organization and supervision density differ. Mid-training usually treats whole documents/code/derivations as learning targets, computes loss over many tokens. SFT organizes data as input-output demonstrations, usually computes loss only on response tokens.
- Technically possible to make a model memorize some facts through a small QA SFT set, but this repeatedly reinforces only a few access paths; model may memorize the questions without forming broadly accessible knowledge.
- Prefer Mid-training when absorbing large, interconnected bodies of domain knowledge; prefer **RAG** when knowledge must remain updateable and traceable.

### 8.1.3 The Essence of SFT: "Predict the Next Token" with Different Data
- **First key insight**: Mathematically, SFT and pre-training are the same task — both predict next token and minimize the same loss function. Difference lies in just two things:
  1. **Different Data**: Pre-training uses raw internet text (unstructured, containing everything); SFT uses carefully prepared "input-output" pairs, uniformly formatted as "user question → ideal answer." Model continues "predicting the next token" on these demonstrations, learning the protocol of "how to structure a response when asked a question."
  2. **Loss calculated only on the "response" (loss masking)**: An SFT sample = question + labeled response. We don't want the model to learn "how to ask a question," only "how to answer." So when calculating loss, question tokens are masked, gradients backpropagated only through the response portion. This is the **only substantive engineering difference** between SFT and pre-training.
- Why SFT can exhibit memorization on limited demonstrations: its optimization goal is to maximize probability of every token in the labeled response — reproducing the demonstration as closely as possible. For tasks with clear goals and fixed formats, extremely efficient (few thousand examples suffice). But when coverage/diversity insufficient, model may overfit surface patterns or shortcuts in the demonstrations and lose performance under distribution shift.
- SFT uses extremely high sample efficiency to encode a stable input-to-output mapping and protocol in parameters. It encodes **protocol knowledge** — how to say or do something, including format, style, and process — rather than large amounts of **factual knowledge** (what the model knows). The latter relies on pre-training or RAG.

**Training Cost: LoRA Parameter-Efficient Fine-Tuning**
- Both SFT and RL require updating model parameters; full-parameter fine-tuning has high VRAM requirements (storing gradients + optimizer states for billions of parameters).
- **LoRA (Low-Rank Adaptation)** is the most common cost-saving method: instead of modifying large original weight matrices, attach a small "patch" (low-rank matrix) to learn the task. Parameter count only 1%–5% of original, yet can approach full fine-tuning performance. Original weights frozen → less perturbation to base model's existing capabilities → reduces risk of catastrophic forgetting.
- Validated rules of thumb (Schulman, John and Thinking Machines Lab, "LoRA Without Regret", 2025):
  - Apply LoRA to **all** major weight matrices (especially MLP layers, which have the largest parameter count); applying only to attention layers costs accuracy.
  - Optimal learning rate ≈ **10 times** that of full fine-tuning (true for both SFT and RL — a very practical transfer rule).
  - Use medium-to-high rank (**64–256**) for SFT; since information per round is small for RL, a small rank (**8–32**) or even rank=1 is sufficient.
  - During deployment, a single inference server can load multiple LoRA adapters simultaneously for multi-tenant service.
  - This book treats LoRA as the default engineering choice for all post-training methods; will not elaborate separately.

### 8.1.4 When to Repair the Foundation Before Applying SFT or RL
- An RL policy does not directly imitate the tokens of a reference response; it uses rewards to evaluate responses the model generates itself (though reference answers or preference data may still be used to calculate that reward).
- Learning from this signal requires at least **two preconditions**: output must be verifiable, and current policy must occasionally explore valuable behavior.
- **Precondition 1 — format support**: If task requires JSON or a tool call and model emits unparseable text, reward function cannot even tell success from failure. SFT can first make the model articulate itself properly: a small number of demonstrations stabilizes format/basic procedure so reward can be computed, after which RL can optimize the policy. This is the familiar "SFT first, RL second" pattern.
- **Precondition 2 — capability support**: Sample held-out tasks at temperature close to training setup, measure pass@1 and pass@k. If probability of success on one sample is *p*, then under approximately independent sampling probability of at least one success in *k* samples is:
  - **pass@k = 1 − (1 − p)^k**
- If pass@1 is low but pass@k rises clearly with *k*: correct policy already in model's distribution but has too little probability mass; RL, rejection sampling, or distillation has something to amplify.
- If empirical pass@k remains near zero at reasonable *k*, sampling temperature, task coverage: base model can hardly generate a successful trajectory. With only a terminal 0/1 reward, a GRPO rollout group will likely be all zero, eliminating within-group advantage; PPO likewise sees no positive example showing where to move. Increasing sample count merely waits roughly 1/p trials for accidental success — quickly impractical.
- At that point, ask what is missing:
  - If domain language, facts, code patterns, or foundational long-context capability → use **Mid-training** to repair the foundation.
  - If capability exists but cannot be expressed through the interface → use **SFT**.
  - If model makes partial progress but cannot reach the endpoint → add verifiable partial rewards or curriculum learning.
- RL is good at raising the probability of existing-but-unlikely successful behavior; poor at creating knowledge/capabilities the model never learned from an all-zero reward.
- Boundary: "SFT must come first" is true only when output format or basic behavior has not yet been established. Experiment 8-11 shows Llama-3.2-Vision-11B fails under strict structured-output requirements when trained directly with RL. A sufficiently strong base model with nonzero success can skip SFT; **DeepSeek-R1-Zero** is one example — its later cold-start SFT primarily improved readability and language consistency rather than injecting task knowledge for RL.

### 8.1.5 The Essential Difference Between SFT and RL (The Most Important Table in This Chapter)
Why "SFT memorizes, RL generalizes" tendency can appear — key is the different optimization objectives:
- **SFT maximizes probability of the labeled response** (Maximum Likelihood): pushes model to reproduce the demonstration for each training sample. Diverse, representative demonstrations can teach generalizable features, but limited demonstrations/prompts can also produce overfitting to surface patterns or shortcuts. In GeneralPoints, limited demonstrations treated J/Q/K as 10, and performance dropped when those values changed at test time.
- **RL maximizes expected reward**: model explores paths and raises probability of those earning high reward. When reward faithfully represents objective and exploration is sufficient, can discover transferable strategies absent from demonstrations. In GeneralPoints, recomputing the answer when values changed produced better out-of-distribution performance. Conversely, a biased reward/environment can make RL overfit to shortcuts too.

### Table 8-2 Essential Comparison of SFT and RL
| Dimension | SFT (Supervised Fine-Tuning) | RL (Reinforcement Learning) |
|---|---|---|
| Optimization Objective | Maximize probability of labeled answer (Maximum Likelihood) | Maximize expected reward |
| Training Signal | Token-level supervision on a labeled response | Policy-generated responses or trajectories + outcome- or step-level scalar rewards |
| Data Form | "Input-Output" demonstration pairs | Task and environment + reward signal (reference answers optional) |
| Direct Optimization Pressure | Imitate mappings and protocols in the demonstrations | Reinforce behaviors and strategies that earn reward |
| Under Distribution Shift | Depends on demonstration coverage and regularization; limited demonstrations overfit in this chapter's experiments | Depends on reward, environment, and exploration; transfer was better in this chapter's experiments |
| Sample Efficiency | High (thousands of examples are effective) | Low (often tens to hundreds of times that of SFT) |
| Training Stability | High, converges quickly | Low, prone to oscillation, requires careful tuning |
| Best Suited For | Solidifying format/style/process, high-quality demonstrations, stable environment | Needing generalization to new scenarios, exploring optimal strategies, high annotation cost |

**Mode-seeking vs mass-covering distinction**: A question usually admits several families of reasonable answers, each a "mode" in the distribution.
- Maximum-likelihood SFT learns demonstrations one by one → often **mass-covering tendency**: tries to cover several modes that appear in training data.
- RL redistributes probability according to reward and, combined with common reverse-KL constraint, more readily exhibits **mode-seeking tendency**: concentrates probability on a few high-reward modes rather than reproducing every demonstration evenly.
- This explains characteristic strengths: SFT good at covering many known ways of phrasing something; RL good at searching among candidate behaviors for a high-reward strategy.
- Whether end result preserves diversity or contracts to a few modes depends on: demonstration distribution, reward function, KL direction and coefficient, entropy regularization, and sampling temperature.

**Post-training shapes when a model acts** (coding model example):
- GPT-family and Claude-family models often exhibit different default action thresholds. Former may read more of a repository before editing; latter may localize from fewer files, implement first, then use test feedback to correct course.
- Not anthropomorphizing ("cautious" vs "instinctive") — it is a policy in the parameters estimating whether expected value of reading one more file still exceeds expected value of submitting and validating the current patch.
- If SFT demonstrations repeatedly investigate broadly before editing, model imitates a higher action threshold. If process or outcome rewards repeatedly validate rapid localization and early verifiable loop, probability mass shifts toward earlier action.
- Experiment 7-9 (Chapter 7) swaps models inside an identical neutral Coding harness and measures behavior changing with the model: harness need not enforce a workflow for the model to carry a stable tool-use policy of its own. Harness can modify the policy, but its primary source can reside in post-trained parameters. Because vendors don't publish complete data/reward recipes, experiment establishes a model-side behavioral difference, not the particular proprietary algorithm.
- **Online feedback** creates opportunity to explore strategies beyond demonstrations. Three opportunities created by online feedback:
  1. **Evaluate candidates beyond a fixed demonstration set**: RL can reinforce new behaviors the reward function can score. The "push-cut" action in Experiment 8-13 (SimpleVLA-RL) never appeared in human demonstrations — possibility of discovering strategy outside the data. But model cannot learn quality the reward cannot recognize, or discover a strategy it never explores.
  2. **Exploit tasks where verification is easier than generation**: SFT needs a correct answer/good trajectory written first; RL needs a reliable way to judge answer quality. Math answers can be checked, code tested, proofs verified. This asymmetry is a strength of **RLVR**, but an incomplete verifier can also produce reward hacking.
  3. **Train on states visited by current policy**: Offline imitation has the classic problem of **covariate shift** — after policy leaves demonstrations and enters unseen states, recovery signals may be absent. In specific sequential imitation-learning settings, worst-case error can accumulate roughly as **T²** with trajectory length T; online data aggregation can reduce it to about **T**. On-Policy Distillation (see "Distillation: Improving Sample Efficiency") combines this online matching with SFT's dense supervision.
- Analogy: SFT studies an existing map in detail; RL can use reward as a compass to explore candidate routes beyond it. An inaccurate map or compass can lead the model astray. Many systems use SFT to establish a stable starting point, then add RL when reward/environment are trustworthy.

Next two sections, both [Optional Reading] — "From Classic RL Agents to Modern Agents" and "Model Pre-training Basics" — fill in RL and pre-training background.

## 8.2 From Classic RL Agents to Modern Agents [Optional Reading]

### 8.2.1 Agent-Environment Interaction
- **Reinforcement Learning (RL)**: learning how to select actions based on the current situation to maximize cumulative reward. Chess example: each move is an action; winning positive reward, losing negative; cumulative reward is total gain from the entire game.
- Agent and environment interact continuously: at each step, Agent observes current state, chooses an action, environment produces a new state and gives a reward.

**Figure 8-1: Reinforcement Learning Agent-Environment Interaction Loop**
- Agent (Explorer): Policy π: Observe state → Select action (move/pick up/attack); ε-greedy: 90% optimal, 10% random.
- Environment (Maze World): 5 rooms · Key/Door/Guard; Transition: P(Storage|Hall,East)=1.0; R(Defeat Guard)=+50, R(Move)=0.
- Loop: Action A(t) = "Move East" → State S(t+1) + Reward R(t+1).
- Trajectory: S(0), A(0), R(1), S(1), A(1), R(2), ...
  - S(0): Entrance Hall; A(0): East; R(1): 0; S(1): Storage Room; A(1): Take Key; R(2): +5; S(2): Storage Room; A(2): North; R(3): 0; S(3): Guard Room; A(3): Craft Silver Sword; R(4): +10; S(4): Guard Room; A(4): Attack; R(5): +50.
  - Goal: Maximize cumulative reward **G = R(1) + γR(2) + γ²R(3) + ... (γ=0.99)**

- This interaction produces a **trajectory** — complete record of "state → action → reward → new state → action → reward…". Quality of policy ultimately reflected in quality of trajectories.
- **Value function** answers: "If I am in this state now and continue acting according to the current policy, how much total reward will I eventually accumulate?" Like an experienced chess player intuitively estimating winning probability without calculating to the end. (When "current policy" replaced by "optimal policy," get the optimal value function — used later for Bellman optimality equation.)
- Boundary between Agent and environment: anything the Agent cannot arbitrarily change belongs to the environment.
- **Two unique features** distinguishing RL from supervised (labeled correct answers) and unsupervised (discover hidden patterns):
  - **Trial-and-error search**: Agent must figure out which actions are good on its own, without a teacher directly providing the correct answer.
  - **Delayed reward**: effect of an action may only become apparent many steps later (e.g., good chess move's value only evident at game end).
  - This brings the **exploration-exploitation tradeoff**: always taking familiar paths learns nothing new; always trying randomly never reaches the goal.
- **Five core elements** of an RL system:
  - **Action Space**: set of all possible actions. Discrete (e.g., "which move in chess") or continuous (e.g., "how many degrees to rotate a joint").
  - **Policy**: Agent's behavioral rule — what to do in a state. Can be simple (lookup table) or complex (deep neural network).
  - **Reward Signal**: immediate feedback from environment. Goal is to maximize long-term, not immediate, reward (like investment judged by long-term returns, not today's gains).
  - **Value Function**: estimates total cumulative reward obtainable from a state in the future, helping make wise decisions even without immediate feedback. One of the most important insights from sixty years of RL research: central role of value estimation.
  - **Environment Model (optional)**: predicts environment's response to actions. Model-based methods (learn to predict how environment changes, then plan); model-free methods (don't predict environment, learn directly from experience).

**Table 8-3 Comparison of Key Elements in Different Agent Systems**
| Agent Type | Environment | Action Space | Reward Signal |
|---|---|---|---|
| Newborn Gazelle | Terrain, gravity, body posture | Continuous high-dimensional (muscle group contractions) | Balance (+), Falling (−) |
| Vacuum Robot | Room layout, battery level | Discrete (direction, vacuum, charge) | Cleaned area (+), Battery depleted (−) |
| Chess Grandmaster | Board state, time limit | Discrete finite (legal moves) | Win (+1), Loss (−1) |
| Customer Service Agent | Conversation history, knowledge base | Variable-length compositional (think, speak, API call) | Problem solved (+), Handling time (−) |
| Code Assistant Agent | Requirements document, codebase | Variable-length compositional (think, search, edit, execute) | Test passed (+), Bug introduced (−) |

- Key distinction revealed: board-game/Atari environments use predefined finite discrete primitive actions; robot control uses continuous actions with fixed dimensions and physical bounds. Modern LLM-based customer-service and coding Agents compose finite tokens and tool calls into **variable-length action sequences** — making possible sequences difficult to enumerate at once. They can also use internal thinking to improve capabilities.

### 8.2.2 Two Action Representations: Classic RL Settings and Variable-Length LLM Policies
- Most visible difference: how actions are represented. An **MDP** itself can represent finite/infinite, discrete/continuous action spaces. Board-game/Atari use finite discrete primitive actions; robot control uses bounded continuous actions; an LLM policy composes a finite token vocabulary and tool schemas into variable-length sequences. This compositional representation has major consequences for algorithm design, sample efficiency, and generalization.

**Foundational Example: MDP and Tabular Q-learning.**
- **MDP (Markov Decision Process)**: the mathematical framework for RL, defining states, actions, rewards. Core assumption is the **Markov property**: the future depends only on the current state, which must contain all history relevant to the decision. In chess, state includes piece placement, side to move, castling and en passant rights, information for the fifty-move and repetition rules. With a sufficient state definition, entire game record need not be reread for each transition. If an observation omits necessary history, that history must be added to the state or handled with a partially observable model.

**Figure 8-2: Markov Decision Process (MDP) Diagram**
- States: Entrance Hall, Storage Room, Northern Corridor, Guard Room, Treasure Room.
- Actions: East/Take Key; North (with key); North→West; East; Attack (Silver Sword); R=+100.
- MDP Quintuple: (S, A, P, R, γ); S = 5 states; A = {East, West, South, North, Take, Attack, Combine}; P(s'|s,a) = deterministic; γ = 0.99.

- Representative RL environments use predefined action spaces. Go: 361 move positions (large but finite); chess actions enumerable; Atari exposes a few to a dozen discrete primitive actions. Robotic agents: continuous but bounded action spaces (joint angles, velocities, grip forces have physical bounds and dimensions fixed by degrees of freedom).
- Finite discrete actions make individual candidates easier to evaluate. Small state/action spaces → tabular Q-learning stores values directly; larger Atari/board-game spaces combine function approximation with search. Continuous-action MDPs cannot enumerate every action → policy gradients and actor-critic approximate policy and value function. Classic example also differs from an LLM policy because it starts trial-and-error learning without pretrained knowledge.
- **Q-learning**: maintains a value estimate for each "state-action" pair: if you take action *a* in state *s* and then act optimally thereafter, how much total reward can you expect? Whether an action is good depends on the immediate reward plus "how good the next state it leads to is."
- **Bellman equation** (core recursive relationship): the true value of an action = immediate reward obtained at this step + the maximum future value obtainable from the next state:
  - **Q\*(s, a) = r + γ max_{a'} Q\*(s', a')**
  - where *r* = immediate reward, *s'* = next state after executing action (written deterministic for intuition; in stochastic environment, an expectation over next state *s'* is needed), and γ ∈ [0, 1) = **discount factor** — how much the Agent values the future: closer to 1 → values long-term returns; closer to 0 → focuses on the immediate.
  - "Cumulative reward" = sum of per-step rewards discounted by γ: **Σ_t γ^t r_t**.
  - After each action, algorithm slightly adjusts old estimate toward the "actually observed outcome" — this paradigm ("correcting an old estimate with a one-step actual result") is **Temporal-Difference Learning (TD learning)**. After thousands of trials, estimate gradually approaches true value.

**Figure 8-3: Q-learning Grid World** (exploration process + Q-value convergence)
- Q-values per cell approximate: 0.12 → 0.25 → 0.41 → 0.63 → (↓)0.85 → (↑)0.08 ... wall ... 0.38 ... wall ... 0.90/0.05, 0.10/0.30/0.55/0.93/0.02, 0.06/0.20/0.70/0.96/0.00, 0.03/0.15/0.80, treasure V=1.00.
- Q-learning parameters: α = 0.2 (learning rate); γ = 0.99 (discount factor); ε: 1.0 → 0.1 (exploration rate).
- Training progress: 0–1K ep = blind exploration; 1–5K ep = path discovery; 7–8K ep = win rate → 96%; 10K ep = optimal solution 11 steps.
- Shading: dark gray = high value; light = low value; black = obstacle wall.

**Q-update rule**: Q(s,a) ← Q(s,a) + α [ r + γ max_{a'} Q(s',a') − Q(s,a) ] (TD Target / Current Estimate).

**Figure 8-4: Q-value Update Visualization** — Concrete example: Agent picks up red key in storage room.
- Current state s: Storage room, Holding: empty; Action a: Pick up red key; Reward r: +5; New state s': Storage room, Holding: red key; max Q(s',a'): 0.63 (Open door north).
- Calculation: Q(storage room, pick up key) ← 0.25 + 0.2 × [5 + 0.99×0.63 − 0.25] = 0.25 + 0.2 × [5.624 − 0.25] = 0.25 + 1.075 = 1.325. TD error = 5.374 → Positive update: Q value increases significantly (Reward propagation).

- Q-learning is an **off-policy** method: can learn an optimal policy from data generated by an exploratory policy different from the target policy. Still requires adequate coverage of relevant state-action pairs and appropriate learning-rate/convergence conditions; does not automatically converge on arbitrary data distribution. Strict definitions of on-policy/off-policy and mapping to LLM post-training discussed later in "RL Algorithms: From 16 Rollouts to One Parameter Update."

**Experiment 8-1 ★: Q-learning Performance in a Treasure Hunt Game**
- Treasure hunt environment challenges: hidden mechanisms (discover correspondence between keys and doors, weapon effects, item crafting rules on its own); multi-step dependencies (optimal solution: 11 steps); sparse rewards (only key actions and final victory yield significant rewards, most intermediate steps no feedback).
- Q-learning Agent: standard Q-learning parameters and ε-greedy exploration (usually selects currently optimal action, occasionally random; proportion of random exploration gradually decreases during training).
- Learning curve (an episode = one complete game from start to completion/failure):
  - First 1000 episodes: 0% win rate; Q-table has only 124 states; Agent blindly exploring.
  - First 5000 episodes: still no stable victories; Q-table has 133 states.
  - 7,000–8,000 episodes: win rate gradually rises from 34% to 96%.
  - 10,000 episodes: 100% win rate; Q-table has 145 states; found 11-step optimal solution.
- Entire training takes less than 10 seconds (very efficient simulation) but requires nearly 10,000 complete attempts. Demonstrates prior-free, ε-greedy tabular Q-learning behavior: needs substantial random exploration to complete path by chance, and value signals propagate slowly enough to require repeated reinforcement.
- In a game simulator, 10,000 trials take only 10 seconds (negligible). But in real-world Agent scenarios — each phone call has cost, each browser operation delay, each wrong decision can have irreversible consequences — 10,000 trials are completely unacceptable. One reason to use a pretrained LLM policy: accumulated knowledge supports effective decisions with far fewer environmental interactions.
- Three limitations of prior-free tabular Q-learning: even a simple task needs extensive interaction; values learned in one environment don't transfer directly to another; each new task must be explored again. Not limitations of MDP framework itself — function approximation, transfer learning, and model-based RL can handle richer states and knowledge transfer, though may still require substantial environmental interaction compared with a pretrained LLM.

**Agents Based on Pretrained LLM Policies.**
- LLMs bring an important practical change to how Agent actions are represented and initialized.
- Classic RL can also model internal computation or information gathering as states and actions. The practical change introduced by LLMs is not that thinking became possible for the first time, but that a pretrained language policy can represent internal computation as variable-length token sequences and generate it within the same policy as external actions.
- Thinking tokens do not directly change the external world, but can improve the final action. Action representation now includes not only "what to do," but also "how long to think and what to think about."
- **Most important practical innovation**: incorporating thinking tokens as special actions in the policy output space. Traditional RL environments emphasize primitive actions (moving, attacking, picking up), though internal computation can be modeled in an MDP or hierarchical policy. In LLM Agents, internal thinking becomes a core part of the learned language action space. It doesn't directly change external environment or receive immediate environmental reward, but can express many computational paths within token costs and context limits.
- Variable-length compositional actions create a much larger search space than primitive actions and are difficult to learn from scratch without prior knowledge (like searching for treasure in a desert blindfolded). LLMs instead learn human problem-solving patterns from massive text pre-training: math solutions follow "identify conditions → recall formulas → calculate step by step"; coding follows "understand requirements → design structure → implement details."
- The pretrained policy gives structured paths higher prior probability, greatly compressing the search space. Even without additional RL, a pretrained LLM can generate a basic logical **Chain of Thought (CoT)**, learned through next-token prediction over math solutions, code comments, discussions, and other human-written reasoning traces.
- RL post-training then uses external rewards to teach the LLM to apply these patterns more effectively to a specific task. Language structure is not a separate "internal reward"; it acts as a **prior distribution** in the pretrained policy. A pattern consistently present in training data (e.g., "we need to convert currency, so first look up the exchange rate") may start with higher generation probability than an unrelated path (e.g., checking the weather). RL uses the actual task reward to reshape path probabilities from that starting distribution.

**Figure 8-5: Comparison of Classic RL and Modern LLM Agent**
| Feature | Classic RL Agent | LLM Agent |
|---|---|---|
| Action Space | Closed finite set (6 actions) | Open: natural language + tools |
| Prior Knowledge | Zero prior, from random policy | Pretrained knowledge + priors |
| State Representation | Hash encoding: state_id=0x3A7F | Semantic: "red key in storage room" |
| Learning Signal | Scalar reward: R ∈ {−1, 0, +5, +50} | Reward + feedback + self-reflection |
| Thinking Ability | No internal thinking (purely reactive) | Thinking as a special action (CoT) |
| Generalization Ability | New rules → full retraining | Cross-task zero/few-shot transfer |
| Sample Efficiency | ~10000 episodes / 10 seconds | ~1 episode / 2 minutes |

- Pretrained language policy enables LLM Agents to understand unseen instructions (**zero-shot generalization**) and adapt to new tasks from a few demonstrations (**few-shot adaptation**), in sharp contrast with the prior-free tabular Q-learning setting.
- Expanding from predefined primitive actions to variable-length compositional actions is an important shift in the AI Agent paradigm. LLM actions still defined by finite token vocabulary and tool schemas, but internal thinking, natural-language queries, program code, complex JSON, and multimodal content combine into an explosive number of variable-length sequences. Code interpreters and search tools connect that representation to a wide range of real-world tasks.
- Opportunities and challenges: Agents can combine basic tools to handle unseen tasks, but reward design and efficient exploration must operate over an enormous compositional space.
- Models like **Kimi K3** (optimized for tool use and long-chain reasoning) illustrate the typical LLM+RL direction: large-scale language pre-training provides the foundation; post-training strengthens problem decomposition, tool use, self-correction.
- **OpenVLA** (detailed in Chapter 6) showcase the VLA (Vision-Language-Action) architecture paradigm: a vision encoder processes environmental observations, a language model understands instructions and reasons, an action decoder generates control signals — enabling language-conditioned control and cross-task generalization. Note: OpenVLA itself is trained through imitation learning on nearly one million robot demonstration trajectories — SFT in nature, not RL. SimpleVLA-RL (Experiment 8-13) is the representative example of bringing RL into robotics by using rewards to further optimize this kind of VLA architecture.

**Figure 8-6: Evolution of OpenAI Training Paradigms**
| Period | Theme | Detail |
|---|---|---|
| 2015-2016 | algorithm-centrism | DQN · Atari; better algorithm = key; new environment requires training from scratch |
| 2016-2018 | importance of environment | Gym · Universe; superhuman performance in Dota 2; web navigation remains unsolved |
| 2018–present | awakening of prior knowledge | GPT-2/3 · WebGPT; ChatGPT · Agent; prior > environment > algorithm; key insight: prior knowledge can be obtained in ways completely unrelated to RL (language pretraining) |

- OpenAI's Exploration Path (chronicled by Shunyu Yao, Princeton, author of ReAct paper, in "The Second Half"):
  - Phase 1 (2015-2016), Algorithm-Centric: better algorithms were the key; progress in standard environments like Atari, but every new environment required retraining from scratch.
  - Phase 2 (2016-2018), Importance of Environment: Gym standardized a range of tasks; Universe and World of Bits tried to turn the entire internet into an RL training environment; Dota 2 pursued superhuman performance in a specific complex environment. The idea was clear, but general computer use and web navigation remained out of reach.
  - Phase 3 (2018-present), Awakening of Priors: GPT-2/GPT-3 demonstrated power of language pre-training; WebGPT and ChatGPT proved those priors could be turned into practical Agents. Most important discovery: **priors can be acquired in ways that have nothing to do with RL**. Counterintuitive truth — for decades, RL researchers may have had their priorities exactly backwards. Real order is not algorithm > environment > prior, but **prior > environment > algorithm**.
- References: OpenVLA (Kim, Moo Jin, et al., 2024, arXiv:2406.09246); Yao, Shunyu, "The Second Half", April 10, 2025, https://ysymyth.github.io/The-Second-Half/

**Experiment 8-2 ★★: Comparative Study of Traditional RL and LLM Agent**
- Compare Q-learning Agent vs LLM Agent (Kimi K3, maintaining a buffer of up to 50 experiences) in the same treasure hunt game.
- Q-learning Agent: State = hash code 0x3A7F; room=2, items=[key], doors=[N]; Q-table lookup (145 states × 7 actions = 1015 entries); ε-greedy selection (ε=0.1 → 90% optimal / 10% random); Action = integer ID (a=3 → "move north", no semantic understanding).
- LLM Agent: State = natural language description ("The storage room has a red key, the north door is locked"); LLM reasoning + experience buffer; Reasoning: "Door locked → need key → get key"; Semantic action selection (understand conceptual relationships, make purposeful decisions); Action = structured instruction {"action":"pick_up","target":"red_key"}.
- Performance comparison:
  - First completion: Q-learning ~7000 episodes; LLM Agent Episode 1 (17 steps).
  - Final win rate: Q-learning 100% (after 10K ep); LLM Agent ~90% (instant).
  - Computation cost: Q-learning 10 sec/10K ep; LLM Agent 1-2 min/episode.
- Reference shown: "LLM Agent（Kimi K2）" (label) — actual experiment uses Kimi K3.
- Results: The LLM Agent completed the game in 18 steps on its first try.
  - **Early Stage (Purposeful Exploration)**: picks up a rusty sword ("A weapon is better than bare hands"), systematically explores the map, deduces "need to find a key" after finding the north gate locked, explores storeroom, acquires the red key and magic crystal.
  - **Middle Stage (Mechanism Understanding and Proactive Synthesis)**: understands "key auto-use" rule, anticipates rusty sword insufficient against the guard, proactively synthesizes a silver sword on step 8.
  - **Late Stage (Execution and Error Correction)**: heads north with the silver sword and defeats the powerful guard at step 13. Along the way makes one or two ineffective attempts — repeatedly swinging the sword or backtracking — and finally obtains the dragon's treasure at step 18.
- Demonstrates fundamental difference between semantic understanding and symbolic mapping. LLM Agent understood conceptual structure of the game; every step had purpose and logical support. For Q-learning, "door," "key," "sword" are just meaningless symbol combinations; it can only slowly discover their relationships through extensive statistical learning.
- Computational cost paradox: Q-learning runs 10,000 games in 10 seconds while LLM Agent takes 1-2 minutes per game. But in real-world tasks, time/money/risk cost per interaction far outweighs pure computational cost; judging solely by GPU time is unfair.
- More critical insight: LLM Agent's success isn't due to a better "learning algorithm," but because it carries vast prior knowledge. When game rules change, Q-learning needs complete retraining while the LLM Agent can adapt directly through reasoning.
- **Practical design principle**: Traditional RL remains valuable in scenarios with low simulation cost and high repeatability; in real-world scenarios with high interaction costs and a need for rapid adaptation, the sample efficiency of LLM Agents is more valuable in practice.
- Chapter 1 provided a conceptual map of how contextual adaptation, updates to external artifacts, and parameter updates work together; the section "Post-Training Practical Takeaways" returns to the topic. This chapter's main thread is post-training: writing into model parameters capabilities that cannot be fully expressed through external rules.

## 8.3 Model Pre-training Basics [Optional Reading]

- To understand why post-training techniques are effective, first understand what pre-training establishes. Post-training (SFT and RL) essentially optimizes within the representation space established by pre-training — the knowledge structure laid down by pre-training determines the **ceiling** of post-training.
- Examine core aspects of pre-training through three experiments: training a small-scale language model from scratch, extending visual capabilities, and injecting new language knowledge. The three experiments are supplementary, building intuition about pre-training.
- Three-step pipeline: "tokenization — pre-training — post-training." Tokenization segments text into discrete units. E.g., "I like programming" might be tokenized into "I," "like," "program," "ming" — the smallest textual units processed by the model.
- Pre-training task is conceptually simple: show model the first part of a text segment, have it predict the next token; compare prediction to correct answer (difference = loss; smaller loss = more accurate prediction); model continuously adjusts parameters. After repeated training on massive text, model gradually learns language rules, world knowledge, and basic reasoning abilities.
- After pre-training, model can generate fluent text but output lacks structure and struggles to follow instructions. Post-training transforms model into a practical assistant through SFT and preference optimization, such as DPO, which teaches the model to generate responses humans prefer.

**Figure 8-8: Pre-training Next Token Prediction**
- probability distribution P(x_t | x_{<t}); example: Chinese 42%, li 28%, down 15%, ... 15%.
- Loss = −Σ log P(x_t | x_1...x_{t−1}) → minimize cross-entropy.
- Seemingly simple goal-driven model learning: language rules → world knowledge → basic reasoning.

**Experiment 8-3 ★★: Training an LLM from Scratch—The Power of Algorithm Improvement**
- Uses MiniMind 2, a 100-million-parameter model; completes entire training process on a consumer-grade GPU.
- Two algorithmic optimizations — **QK Norm** and the **Muon optimizer** — triple the convergence speed and significantly improve generation quality, all at very low cost: approximately 14 hours of training and $34 total.
- Effects of each training stage: after pre-training, model can answer factual questions ("What is the highest mountain in the world?") but format is non-standard; after SFT, instruction following and output formatting improve significantly; preference optimization further reduces factual errors and unnatural expressions.
- 100M-parameter model still has obvious limitations (prone to errors on complex problems), but the lesson: with a fixed, small budget, algorithmic improvements offer better value than simply scaling up size.

**Experiment 8-4 ★★: Training Your Own VLM**
- **Figure 8-9: Vision-Language Model (VLM) Architecture** — three components:
  - Vision encoder (CLIP ViT): parameters frozen (not trained); input 224×224 image; output 196 visual tokens; dimension 768.
  - Projection layer (fully connected): the only part trained from scratch; Linear(768, 512); aligns vision → language space; parameter count ~400K.
  - Language model (MiniMind 100M): unfrozen during SFT stage; hidden_dim=512; autoregressive description generation. Muon optimizer accelerates 3×.
  - Flow: input image (waterfall/city/nature) → vision tokens → language embedding → output description ("a huge waterfall with a rainbow hanging beside it").
  - Strategy: freeze LLM + train projection layer → unfreeze LLM during SFT (avoid catastrophic forgetting).
- VLMs unify visual perception and language understanding within a single model. Core challenge is **cross-modal alignment** — making "what is seen" correspond to "what is said."
- Training uses "freeze LLM + train only projection layer" strategy to avoid **catastrophic forgetting** (forgetting old skills after learning new ones); after alignment pre-training stage, LLM is unfrozen and SFT is performed on high-quality image-description pairs, significantly improving detail and accuracy of descriptions.
- Reveals the basic paradigm for multimodal model training: reusing unimodal pre-training results and achieving cross-modal alignment by training a lightweight projection layer — efficient and scalable, but projection layer's limited expressiveness can become a bottleneck for deep cross-modal understanding.
- Extending the same "vision encoder + projection layer + LLM" architecture by having the model output actions produces the VLA (Vision-Language-Action) model detailed in Chapter 6.
- Together, the two pre-training experiments reveal a pattern: under a limited budget, algorithmic and architectural improvements often offer better value than scale alone. More importantly, pre-training supplies descriptive knowledge and language-modeling capability but not structured instruction following or task-oriented behavior. Yet SFT and RL cannot bypass a target language or domain that general pre-training never covered — that is the gap Mid-training addresses.

## 8.4 Mid-training: Filling Knowledge and Foundational Capability Gaps

- In this chapter, Mid-training = taking an existing base model and continuing language-model training on a target data distribution. It usually retains pre-training's next-token objective and computes loss over every token in a document, code sample, or derivation.
- Classic DAPT/TAPT research shows a second pre-training stage on domain or task-related unlabeled corpora can continue improving downstream performance (Gururangan et al., "Don't Stop Pretraining: Adapt Language Models to Domains and Tasks", ACL 2020).
- "Mid" describes its place in the capability-development pipeline; its data format and loss remain those of pre-training.
- Mid-training mainly addresses **two kinds of gap**:
  - **Knowledge gaps**: general pre-training didn't adequately cover a target language, finance, medicine, law, internal enterprise documents, or a class of codebases — model cannot even understand the concepts and terminology.
  - **Foundational capability gaps**: target task requires long-context, coding, mathematical-derivation, or multimodal representations the base model has not formed. The problem is not merely the response format: even after many samples, the model almost never reaches a correct solution.
- Why SFT should NOT be the main vehicle for knowledge injection: SFT can memorize a small number of facts and often follows Mid-training to teach the model how to answer domain questions. But a small QA set covers only a limited set of phrasings; it is better at training how to access and express knowledge than at carrying a large, interconnected body of raw knowledge.
- Conversely, reducing language-model loss on domain text does not ensure the model will retrieve that knowledge in response to a question. Research shows the order and organization of continued pre-training and instruction tuning materially affect whether knowledge can be accessed in QA form (Jiang et al., "Instruction-tuned Language Models are Better Knowledge Learners", ACL 2024).
- Robust recipe: **Mid-training absorbs knowledge and capabilities → small-scale SFT establishes access and output protocols → RL is added if needed once success is nonzero.**

### 8.4.1 Constructing Mid-training Data
1. **Infer data needs from the failure distribution.** Slice evaluations by topic, language, document type, code pattern, and context length. Determine which low-pass@k cases come from a base-model gap, and add data only for knowledge and capability gaps rather than misdiagnosing output-format errors as missing knowledge.
2. **Build high-density target corpora.** Raw documents establish terminology and factual associations; repositories teach structure and dependencies; textbook-style derivations, synthetic explanations, and cross-document association samples make implicit relationships explicit. Deduplicate, filter for quality, and check for evaluation-set contamination.
3. **Mix the data by capability.** The mixture needs: natural long text (books, long documents, code repositories); **chain-of-thought data** embodying atomic long-text capabilities — long-text retrieval, multi-hop reasoning, instruction following, information aggregation and statistics; and **Agent execution trajectories** embodying the capabilities an Agent cannot do without — planning, tool selection and invocation, long-range state tracking, and recovery from errors. The chain-of-thought and Agent-trajectory data can be distilled from a stronger open-source model or taken from existing datasets.
4. **Use two forms of replay at every stage.** First: original short text and general data — preserves language, knowledge, and short-context capability. Second: "old tasks lifted to the new length" — place short tasks the model already handles into a context of the current length, scattering relevant information and distractors across different positions, and check whether the same capability still holds in the wider window. General data best drawn from the base model's original pre-training set; when unavailable, an open pre-training corpus such as **FineWeb-2** can stand in.
5. **Decide when to stop with multidimensional gates.** Besides training loss, track pass@1/pass@k on held-out domain tasks, general capabilities, prior instruction following, and the target task. If domain metrics rise while the general retention set drops → mixture or learning rate is too aggressive; if loss falls while pass@k does not move → check whether data truly covers the required capability, and whether the SFT that makes knowledge accessible is missing downstream.
- After Mid-training, evaluation sets such as **LongBench v2**, **IFEval**, and end-to-end Agent benchmarks (Ch 7) verify the model's foundational long-context capability across different context lengths hasn't been lost. Long-context capability underpins long chain-of-thought and instruction following; those underpin higher-order Agent capabilities such as tool calling.
- Long-context capability categories:
  - **Position and retrieval**: single-needle and multi-needle extraction, key information at different positions, retrieval under distractors.
  - **Relations and reasoning**: cross-paragraph, cross-document, and multi-hop relation tracking, contradiction resolution, evidence composition.
  - **Aggregation and statistics**: counting, grouping, sorting, comparison, trend summaries, aggregation over long tables or logs.
  - **Instruction following**: following complex instructions, including multiple instructions at once, contradiction resolution, adherence to a prescribed thinking procedure, output-format compliance.
  - **Long-chain thinking**: solving hard mathematics, logical-reasoning, and code-generation problems.
  - **Agent primitives**: basic task decomposition, planning, tool selection, argument construction, state memory, recovery from failure.
- If facts change frequently or must be cited to primary sources, **RAG** is still preferable to writing them into weights. Mid-training is better suited to stable, large-scale domain knowledge and capabilities that need internal representations.
- Full-parameter Mid-training on a large model costs more and risks more forgetting than small-scale SFT, so validate the mixture in a small pilot before scaling the training budget.

**Experiment 8-5 ★★: Continued Pre-training to Learn a New Language**
- Base model: **Mistral 7B v0.3** — primarily pre-trained on English, almost no understanding of Korean. Introduces Korean capability through continued language-model training on Korean Wikipedia. Model already has general representations and only needs to adapt to a new data distribution — much cheaper than training from scratch.
- Uses approximately **80% Korean and 20% English** to mitigate catastrophic forgetting; that ratio is an experimental choice, not a universal default.
- Korean instruction data is then used for **SFT** to obtain practical conversational ability. Division of responsibility clear: Mid-training first supplies Korean knowledge and language capability; then SFT teaches the model how to receive instructions and organize answers in Korean.
- Demonstrates the catastrophic forgetting continued pre-training can cause: blind ratings improved for Korean in the final stage while English capability declined. Continued pre-training can write the target distribution into parameters, but it does not remove the need for retention sets, factual evaluation, and data-quality audits.
- Once the model has enough knowledge and foundational capability, the next step is to turn it into a practical Agent that works according to a protocol.

## 8.5 SFT (Supervised Fine-Tuning)

**Figure 8-10: Supervised Fine-Tuning (SFT) Pipeline**
- Data Preparation: Input-Output Pairs (x, y); Speech: Reference Audio → Style Token; Distillation: Teacher Output → Student Target; Multilingual: Thinking Template Demonstration.
- Training Optimization: L = −Σ log P(yᵢ|xᵢ); Maximize Conditional Log-Likelihood; Thousands to Tens of Thousands of Examples; Convergence in Hours.
- Evaluation: Format Compliance ✓; Instruction Understanding ✓; Style Consistency ✓; Out-of-Distribution Generalization ✗.
- SFT Fixed "Protocol" Types:
  - Speech SFT: Style Control Protocol — Timbre + Paralinguistic Markers.
  - Multilingual SFT: Thinking Organization Template — "Analysis → Hypothesis → Verification → Summary".
  - Prompt Distillation: Teacher Output Mapping — Long Prompt → Direct Output Without Prompt.
  - Tool Calling SFT: Interface Format Protocol — JSON Schema + Error Handling.

- Section "From Pre-training to RL: A Four-Part Panorama" explained SFT's essence ("predict the next token" with different data and loss on response only). This section uses four experiments to show what this mechanism — writing stable mappings and protocols into parameters — solidifies across different tasks.
- **Core value of SFT is not injecting new knowledge but solidifying protocols**: writing mappings, interaction formats, and style norms into parameters so the model can produce compliant outputs at inference time without lengthy prompts. Typically only a few thousand to tens of thousands of high-quality examples needed to establish basic conversational ability and instruction following.
- Efficiency can come with dependence on the training distribution. In tasks requiring exploring diverse correct strategies, or where deployment shifts away from demonstrations, SFT may favor reproducing demonstrated patterns and lose performance in new situations. The following experiments show this process of "solidifying protocols" from different angles; they do not establish a universal ranking of SFT and RL.
- **Where does SFT data come from?** Three routes (often combined):
  - **Human expert demonstrations** — highest quality ceiling, but expensive and slow; best used as the "seed data" that defines format and style.
  - **Teacher-model generation** (synthetic data): have a strong model mass-produce "input-output" pairs, filter them, distill into the student (see Experiments 8-8 and 8-9).
  - **Rejection sampling**: the model samples several candidates for the same problem itself, a verifier picks out the correct ones, and it trains on those (see Experiment 8-9).
- Construction pipeline (whichever route): define the task distribution and output schema → generate candidates in bulk → filter for quality with rule-based validation, format checks, human spot checks → deduplicate, balance the mixture, ensure diversity.
- No need to chase volume — a few thousand to a few tens of thousands of high-quality samples is usually enough to solidify the output format. Rather than piling up a hundred thousand dirty samples, refine ten thousand clean ones: every bit of noise in the data is something SFT may faithfully write into the parameters.

**Experiment 8-6 ★★★: Voice SFT—From "Voice Cloning" to "Paralinguistic Modeling" [Extended Experiment]**
- Case studies: **Orpheus** (contextual-prompt voice cloning) and **Sesame** (paralinguistic token modeling). Shows how "voice style and expression habits" get written into parameters. Two different routes:
  - Orpheus: compresses the voice waveform into a token sequence. Concatenating reference audio from the same speaker → model learns to "speak in this person's voice," achieving cross-sentence timbre consistency.
  - Sesame: abstracts paralinguistic phenomena like laughter and sighs into special tokens like `<laugh>`, `<sigh>`. Model learns to "produce the corresponding sound when seeing the token."
- In expressive tasks, SFT solidifies **style control protocols and structured expression habits**, not factual knowledge or complex reasoning. Key lies in diversity and annotation quality of training data.
- Common failure modes: too few speakers in training data → everyone sounds the same; and **token overfitting** (model memorizes training sample details and performs worse on new situations) → "mechanical laughter."

**Experiment 8-7 ★★★: Multilingual Thinking—Enabling the Model to Think in Any Language [Extended Experiment]**
- Most thinking models only "think" in English: regardless of the question language, the internal chain of thought is almost always English, because high-quality thinking demonstrations in training data are mostly English. Goal: enable the model to think in a specified language.
- Approach: perform SFT on **gpt-oss-20b**: add a line `reasoning language: German` (or another language) to the system instruction, then train with reasoning examples in English, Spanish, French, etc. The training data contains **no Chinese at all**, but after training, simply setting the reasoning language to Chinese enables the model to perform complete chain-of-thought reasoning in Chinese — this **zero-shot cross-lingual generalization** is the most interesting finding.
- Note: this is NOT the generalization capability of SFT itself. Multilingual pre-training has already established a shared cross-lingual representation space in the model; SFT merely activates this pre-existing cross-lingual ability.

**Experiment 8-8 ★★: Prompt Distillation—Replicating Usable Capabilities at Lower Cost**
- Problem: to make a model perform complex tasks, lengthy system prompts (thousands or even tens of thousands of tokens) are often required, increasing latency and cost per call. When using reasoning LLMs, internal thinking tokens further amplify cost.
- Idea: compress the behavior of a "long prompt + thinking teacher" into a "short prompt/no prompt + non-thinking student." The teacher generates high-quality answers under full prompt and thinking mode; training data retains only the user input and final conclusion, discarding the lengthy prompt and intermediate thinking process. Student learns to "directly give the conclusion."
- After distillation, student's output quality on same inputs approaches the teacher, while latency and cost significantly reduced (no need to process lengthy prompts and thinking tokens).
- Two dimensions of distillation (not mutually exclusive; often used together in production):
  - "**large to small**" — replacing a large model with a medium or small one to balance cost and quality.
  - "**thinking to non-thinking**" — folding explicit CoT into implicit parametric knowledge at the same scale, achieving a **20-30x improvement in response speed**.
- Important: distillation **inherits the teacher's boundaries** — if teacher has systematic errors on the long tail of the distribution, student further hard-codes these errors; if teacher relies on tools for correctness, simple output distillation loses the robustness provided by tools.
- Engineering takeaway: when product design is stable, input distribution predictable, and cost constraints significant → prompt distillation is an excellent optimization. During exploration or before the task has stabilized → retaining explicit thinking and editable prompts remains central to rapid iteration.

**Experiment 8-9 ★★★: Chain of Thought (CoT) Distillation**
- Prompt distillation discards the thinking process; CoT distillation does the opposite: transfers the complete thinking trajectory of a strong teacher model to the student.
- Distilling CoT from a capable teacher can enable a student with the same parameter count to recover **70%-80% of the teacher's capabilities**. For teams that don't aim to push the frontier but want models they can control themselves, this is the most pragmatic follower strategy. The series of distilled small models open-sourced by DeepSeek-R1 (using R1's thinking trajectories to perform SFT on Qwen and Llama series) is a representative example.
- **Background: The "Thinking Wall" Phenomenon.** Some closed-source reasoning models (e.g., OpenAI o-series, Gemini series) generate internal chain-of-thought during reasoning, but what users see is not the original thinking process — for reasons including distillation prevention, safety, and product experience, providers often rewrite or summarize the CoT before outputting it, hiding the most valuable original thinking behind the API. This is why the experiment chooses **open-source reasoning models as teachers**: models like **DeepSeek V4, Kimi K3, GLM 5.2** directly expose their complete chain-of-thought, making distillation feasible technically and under the license (still confirm the license's terms regarding distilled products before use).
- **From the lab**: a model that can write code may still refuse to help distill another model. The author first used OpenAI Codex powered by GPT-5.6-Sol to write the experimental code; once the task explicitly involved model distillation, Codex refused. Switched to Claude Code powered by Claude Opus 5, encountered the same refusal. **Kimi K3** ultimately completed the experimental code and subsequent run.
  - Neither refusal concerned ordinary mathematical reasoning or merely asking a model to reveal internal chain-of-thought. The request was to implement a complete distillation experiment using data from a strong teacher to train a student. Model distillation is technically very similar to ordinary supervised fine-tuning, but vendor safety and product policies may associate it with model extraction, capability replication, and IP protection — a sensitive category.
  - This event should not be simplified to "Claude does not provide chain-of-thought," nor does it prove "Kimi has weaker guardrails." Whether the Claude API returns summarized thinking, whether a Coding Agent will implement a distillation pipeline, and whether service terms permit model outputs to be used for training are three different questions. This experiment did not attempt to bypass any model's hidden reasoning or safety mechanisms.
  - Practical judgment: for the vast majority doing post-training, there is no need to distill closed-source models' chain-of-thought at all. The gap between today's best open-source models and SOTA closed-source models is not as large as one might imagine; a teacher only needs to be "clearly stronger than the student," not "the best in the world." If the model being post-trained is 200B parameters or smaller, an open-source SOTA model is entirely sufficient as the teacher.
- **Experiment Design: a three-step process.**
  - Step 1, Collect Trajectories: sample problems from the target task distribution (e.g., math, code); use the open-source teacher model to generate complete "thinking + answer" trajectories; filter out trajectories with incorrect final answers using a rule-based validator — otherwise the student imitates the erroneous thinking process. This step — "generate candidates, verify and filter, keep only correct trajectories" — is **rejection sampling**; performing SFT on data constructed this way is **rejection sampling fine-tuning (RFT)**. RFT sits between pure SFT and RL: no reward model to train, no policy gradients — just "sample many, reject the wrong ones, keep the right ones" to improve data quality; an extremely cost-effective way to construct data for verifiable tasks.
  - Step 2, SFT Training: use "problem → thinking trajectory + final answer" as training pairs to perform standard SFT on a small model (e.g., 7B scale).
  - Step 3, Comparative Evaluation: compare student model before and after distillation, and the teacher, on the same benchmark to measure the proportion of capability recovered.
- **Acceptance Criteria**: the distilled student model shows significant improvement on math and code benchmarks relative to pre-distillation performance, and its thinking trajectories exhibit teacher-like behaviors such as reflection, backtracking, and verification.
- Cost of distillation: the student will inherit the teacher's systematic errors and verbose thinking habits (the latter can be further optimized using the AdaptThink approach from Experiment 8-10).
- These four experiments share a common feature — "writing stable mappings and protocols into parameters": voice SFT solidifies style-control protocols, multilingual SFT solidifies thinking-organization templates, distillation SFT solidifies direct input→output mapping. The clearer the objective, the cleaner the format, and the more stable the evaluation criteria, the more sample-efficiently SFT can improve performance.

## 8.6 SFT Data Synthesis: From Demonstrations to Trainable Trajectories

- The ceiling of SFT is set first by its data. Real projects rarely hand-write enough demonstrations one at a time; they usually combine a **small human seed set, teacher-model generation, and verifier filtering**: human demonstrations define format and boundaries; teacher model scales them up; rule-based verification or human spot checks hold the quality line.
- When the model bootstraps itself, you can sample several candidates for the same problem and keep only trajectories that pass verification — this is **rejection sampling fine-tuning (RFT)**.
- Goal of synthetic data: not to replay production logs, but to **distill from them a reusable task structure**: user intent, initial state, available tools, business constraints, common failure modes, and success conditions. Once identifying information is stripped, regenerate fictional people, orders, files, and states for each task type and place them in a resettable, isolated environment. This preserves genuine difficulties while keeping the model from memorizing customer data or internal credentials.
- **Dependable pipeline**: production data → task blueprint → synthetic task → multiple candidate trajectories → task verification and trajectory verification → SFT data.
  - **Task verification** checks whether the problem itself is solvable, whether its difficulty is appropriate, and whether the reference result is correct.
  - **Trajectory verification** checks the final state, tool calls, and business constraints.
  - Conditions writable as unit tests, database assertions, or state-diff checks should use **deterministic code first**; open-ended qualities such as communication quality are then supplemented by a **model evaluator** and calibrated by human sampling.
  - Skill graphs, executable environments, and independent verifiers can further widen task coverage and filter out invalid trajectories (refs: Autodata arXiv:2606.25996; SKT arXiv:2608.02287; skill taxonomy arXiv:2601.03676; TermiGen arXiv:2602.07274; CLI-Universe arXiv:2606.22883).
- The same task and verification infrastructure can later be turned into an RL environment, but the two stages use it differently: **SFT keeps only the successful trajectories that passed verification**, learning stable formats, procedures, basic actions; **RL has the current policy roll out again and uses environment rewards to explore paths beyond demonstrations**.
- Failed trajectories should NOT be fed in directly as correct demonstrations — they can be used to construct **preference pairs**, to reveal gaps in task coverage, or added to training after a diagnosis and fix has been appended.
- In data synthesis, what matters is not volume but **coverage, diversity, and accuracy**. Training set should also be deduplicated and split by task template, customer, or time period; the **evaluation set must come from non-overlapping task types**; reference solutions, hidden tests, and verifier feedback must not leak to the model.
- Bad cases from Chapter 7 can also become training data. Take the Coding Agent's "premature completion": cut out the trajectory prefix up to the point where it's about to declare completion; treat that premature declaration as the **rejected** sample and "run the tests first, check the acceptance conditions one by one, and only then conclude" as the **chosen** sample. Data like this suits **DPO or decision-boundary demonstrations** rather than being used directly as correct SFT trajectories; store the failure reason, applicable conditions, and verifier with the sample so it can be traced and re-examined.
- `build_preference_data.py` in Experiment 8-17 offers two construction paths — a deterministic template and a teacher model — and keeps training data separate from the evaluation set.
- The two Bad Case experiments added in this chapter demonstrate two different supervision targets. **Chinese curly-quote case**: first distills feedback into a scope-sensitive documentation Skill, then runs SFT on structured synthetic data. **Special-string case**: turns old_string mismatches into a byte-exact copying task, training token-by-token fidelity. Both share Chapter 7's failure-attribution and train/eval isolation protocols, but they do not share a total score: the former tests "change what should change, leave what should be left," the latter tests "copy verbatim."

## 8.7 When to Choose Mid-training, SFT, and RL

- Section "From Pre-training to RL: A Four-Part Panorama" explained mechanics of all three training methods. This section gives a practical diagnosis: first decide whether the missing piece is the **foundation, the protocol, or the policy**; do not treat every model failure as a need for RL.

**Figure 8-11: SFT→RL Two-Stage Training Pipeline; Mid-training Precedes These Two Behavioral-Alignment Stages**
- Phase 1: SFT Formatting. Goal: Output parseable (JSON/tool call). Data: Thousands of high-quality demonstrations. Stopping condition: Format stable, basic capability achieved. ⚠ Overtraining → Model collapses to training distribution.
- Phase 2: RL Shaping Strategy. Goal: Maximize task reward (accuracy/success rate). Prerequisite: Output format stable → Reward computable. Breakthrough: Discover new strategies beyond SFT demonstrations. ✓ Stable format + Strategy generalization = Deployment ready.
- Why can't we skip SFT and go directly to RL? Base model output is unstructured → Cannot parse JSON → Reward function returns NaN → Gradients all zero → Training completely fails.
- SFT: max Σ log P(y|x) (fit training distribution); RL: max E[R(τ)] (optimize task objective).
- "SFT memorizes distribution → RL generalizes strategy."
- When "no matter how many demonstrations are added, new scenarios still perform poorly" → Tipping point to switch to RL.

**Table 8-4 Criteria for Choosing Mid-training, SFT, and RL**
| Observed behavior | Main gap | Preferred method | Gate for moving on |
|---|---|---|---|
| The model does not know domain concepts, the language, or basic operations; pass@k stays near zero under reasonable sampling | Knowledge and capability are outside the base model's effective support | Mid-training; use RAG for dynamic facts | Held-out domain results improve, general retention remains acceptable, and the target task begins to yield verifiably correct or partially correct trajectories |
| The model is occasionally correct, but format, tool schema, tone, or fixed procedure is unstable | Behavioral protocol has not been solidified | SFT or constrained decoding | Parse success stabilizes, and a verifier can reliably score key actions and output protocols |
| Success is nonzero and rewards are reliable, but good policies have low probability or long-horizon decisions and OOD generalization remain weak | Probability allocation and policy optimization | RL | Reward agrees with the real objective, rollout groups have enough reward variation, and independent test performance improves during training |
| Only a few stable demonstrations exist and no interactive environment is available | Imitable data exists, online feedback does not | SFT/RFT/offline preference optimization | Establish a baseline and evaluation first, then decide whether building an RL environment is worthwhile |

**Make the decision in this order:**
1. **First rule out solutions that do not modify weights.** If prompts, tools, code constraints, or context management solve the behavior problem, do not train. Prefer RAG for facts that need frequent updates, citations, or deletion.
2. **Measure capability support on a target held-out set.** Don't look only at greedy pass@1; under fixed sampling setup, also measure pass@k, partial-progress rate, parse rate, and manually audit failure causes. If pass@k remains near zero and failures cluster around knowledge or foundational capability, use Mid-training first and remeasure before choosing a later stage.
3. **Use SFT to establish protocols, not to stuff in a knowledge base.** When the model can do the task but cannot do it as required, use high-quality demonstrations to solidify JSON schemas, tool calls, terminology, procedures, and style. A few facts may enter parameters with demonstrations, but a handful of QA pairs should not carry a large knowledge base.
4. **Use RL only when there is something to explore.** RL is appropriate when the current policy already produces scoreable, occasionally successful rollouts and the reward faithfully represents deployment goals. If pass@k is near zero, first use Mid-training/SFT or design a reachable curriculum and partial rewards; applying PPO or GRPO directly to all-zero rollouts usually only burns sampling budget.
- This flow does not require every project to run all three methods in order. A strong base model may enter RL directly, a format-only task may need only SFT, and stable domain knowledge may need Mid-training followed by reuse of the model's existing alignment. Key: **every transition has a measurable entry condition** rather than treating "Mid-training → SFT → RL" as a ritual pipeline.

## 8.8 Single-Turn Reinforcement Learning: A Comparison of Memory and Generalization

- "Single-turn" = task completed in one interaction: model receives input, produces output, receives a reward, without maintaining state across steps. Simplified setting focuses on fundamental differences in learning mechanisms between SFT and RL, without multi-turn complexity. Same task, same base model, same computational budget; only variable is training method.
- First experiment (AdaptThink) demonstrates how RL learns the meta-strategy of "when to think"; second (GeneralPoints) uses an arithmetic reasoning card game to systematically quantify "SFT memorizes, RL generalizes."
- **Minimal RL intuition** (enough to follow the experiments): RL training in this chapter mostly rests on the **policy gradient**: the model generates several responses to the same problem, increasing probability of high-reward responses and decreasing low-reward ones. To discourage a single large update from derailing the model, mainstream **PPO clips** additional gains in its surrogate objective when a probability ratio falls outside a specified range — discourages large changes but does not impose a hard constraint on policy movement. Later experiments use "PPO with a value network," whose value network estimates a baseline for finer-grained advantages. **GRPO** trains no value network; instead it compares multiple responses to the same problem against one another to judge each one's relative quality.

**GRPO-style Python pseudocode** (omits sampling parallelism, KL regularization, optimizer details; marks only causal chain from one rollout to a parameter update):
```
for prompt in batch:
    group = [rollout(policy, env.reset(prompt)) for _ in range(G)]
    rewards = [verify(trajectory) for trajectory in group]
    advantages = normalize_within_group(rewards)   # GRPO baseline
    update(policy, group, advantages)
```

**PPO's value network and clipped objective pseudocode**:
```
for trajectory in rollouts:
    returns = discounted_returns(trajectory.rewards)
    values = value_model(trajectory.states)
    advantages = returns - stop_gradient(values)
    ratio = exp(policy.log_prob(trajectory.actions)
                - old_policy.log_prob(trajectory.actions))
    policy_loss = -mean(min(
        ratio * advantages,
        clip(ratio, 1 - epsilon, 1 + epsilon) * advantages
    ))
    value_loss = mean((value_model(trajectory.states) - returns) ** 2)
    update(policy, value_model, policy_loss + value_coef * value_loss)
```
- The "relative" in GRPO comes from comparing rollouts within a group for the same prompt. The `old_policy` in PPO is the frozen policy snapshot that generated this batch of rollouts; the probability ratio measures how far the current policy has already moved from it. Clipping discourages large steps but is not a hard constraint on policy movement. Both still depend on a reliable environment and reward.

**Experiment 8-10 ★★: AdaptThink—Learning "When Not to Think"**
- Problem: large reasoning models (e.g., OpenAI o1, DeepSeek-R1) generate lengthy chain-of-thought for all problems, causing unnecessary overhead on simple problems.
- Validates an intuition: **No-Thinking mode** (skipping thinking via `[thinking]`/response) performs comparably or even better on simple problems; only for difficult problems does Thinking mode's advantage become apparent.
- **AdaptThink** uses RL to train the model to adaptively choose the mode. Two core components:
  - **Constrained Optimization Objective**: encourages NoThinking while ensuring overall performance does not degrade.
  - **Importance Sampling Strategy**: balances Thinking and NoThinking samples to solve the **cold-start problem** (here: initial model almost always chooses Thinking, leaving the NoThinking branch with too few samples to learn effectively; this differs from the earlier use of "cold-start SFT" for DeepSeek-R1, which involves a small number of demonstration examples).
  - Importance sampling: a common statistical method — when the sampling distribution is biased toward a certain class of samples, weights are applied to correct the distribution, ensuring the learning signal fairly covers all classes. Repeatedly used in RL algorithms like PPO and DAPO later in this book.
- Canonical record: the checkpoint-free training report. Public W&B main run `wubbn5tj` used 8×NVIDIA H100 80GB GPUs.
  - Step 0→300: MATH500 accuracy 0.8100→0.8180 (+0.80 pp) while response length 4911.46→1576.62 (−67.90%).
  - GSM8K: 0.796816→0.818802 (+2.20 pp) and 1025.24→477.33 (−53.44%).
  - AIME mean16: 0.314583→0.310417 (−0.42 pp) and 12119.51→6402.23 (−47.17%).
  - Corresponding NoThinking ratios: 83.80%, 84.15%, and 56.25%.
  - Results show a routing signal aligned with difficulty at the aggregate dataset level, but do not justify calling it "perfect difficulty awareness" on every problem or claiming accuracy improved universally.
- After the report's selected measurement point, the run continued to step 410 and 36.92 cumulative hours before W&B marked it as crashed; the configured 10 epochs / 3,140 steps were not completed. Although step 300 contains a checkpoint-timing event, the checkpoint is not distributed with the book, and there is no independent receipt proving it was successfully evaluated with `run_eval_verl_hf.sh` or used to rerun MMLU. Historical source commit `9e588202…`; future reproductions pinned to its direct child commit `0033ad172…`. The three entry-point files are unchanged, but the `-fl-` path generated by the training script is incompatible with the `-fl4096` path hard-coded in the evaluation script and must be corrected manually.
- AdaptThink complements prompt distillation to form a "**fast-slow dual system**": distillation reduces the proportion of tasks that require thinking; AdaptThink optimizes the triggering strategy for the remaining tasks, jointly improving thinking efficiency.

**Experiment 8-11 ★★: GeneralPoints—A "Memory and Generalization" Comparison in Single-Turn RL**
- **GP-L (Plain Text)**: Training Set J/Q/K = 10; Test Set (OOD) J=11, Q=12, K=13. Input: "Use [7, J, 3, 5] to compute 24"; Output: "(J-7)×(5+3) = 3×8 = 24". *OOD when J=11: "(11-7)×(5+3)=32≠24".
- **GP-VL (Vision-Language)**: Training Set ♠♣ Black Suits; Test Set (OOD) ♥♦ Red Suits. Input: Playing card image; Visual Encoder → Digit Recognition → Reasoning. *OOD: suit appearance variation tests visual robustness.
- Model: Llama-3.2-Vision-11B. SFT Path: max Σ log P(y|x) | Memorize Training Distribution. RL Path (PPO): max E[R(τ)] | Explore Generalization Strategies.
- **OOD Test Results (Relative to SFT Init Baseline)**:
  | | Rule OOD (GP-L) | Rule OOD (GP-VL) | Visual OOD (GP-VL) |
  |---|---|---|---|
  | SFT Extension | −8.1% (11.5→3.4) | −5.6% | −9.9% (23.6→13.7) |
  | RL Extension | +3.5% (11.5→15.0) | +3.0% | +17.6% (23.6→41.2) |
- Conclusion: SFT consistently decreases out-of-distribution, RL consistently improves — "SFT memorizes, RL generalizes."
- Key premise: End-to-end RL without SFT completely fails (output is unstructured, reward cannot be computed).
- GeneralPoints is an arithmetic reasoning card game proposed by Chu et al. (Tianzhe Chu et al., "SFT Memorizes, RL Generalizes: A Comparative Study of Foundation Model Post-training", 2025, arXiv:2501.17161), designed specifically to evaluate model generalization. Objective resembles the "24 Game": use each of the four numbers on the cards exactly once, combining with addition, subtraction, multiplication, and division to reach the target number 24. Two variants: text-only GP-L and image-based GP-VL — examining rule generalization and visual generalization in the same framework.
  - Rule Variant: during training J/Q/K counted as 10; during testing 11/12/13 respectively — test set contains unseen number combinations (operations involving 11, 12, 13) to strictly evaluate generalization.
  - Visual Variant: training uses black suits (♠♣), testing uses red suits (♥♦) — robustness to changes in visual appearance.
- Standard post-training pipeline with Llama-3.2-Vision-11B: first SFT initialization gives basic instruction-following ability; then, under same computational budget, the model undergoes additional SFT and RL training in separate branches, with PPO and a value network for RL. Both branches trained on single rule J/Q/K=10; evaluated on in-distribution (ID) and out-of-distribution (OOD) test sets.
- Results:
  - Rule OOD: RL +3.5 pp on GP-L (11.5%→15.0%); SFT −8.1 pp (11.5%→3.4%). On GP-VL: RL +3.0 pp; SFT −5.6 pp.
  - Visual OOD: RL +17.6 pp on GP-VL (23.6%→41.2%); SFT −9.9 pp (23.6%→13.7%).
- Tracking visual recognition accuracy: RL improves the underlying visual encoder through outcome-oriented optimization, highly correlated with overall performance gains; SFT overfits to token patterns in the thinking process, neglecting learning of visual tokens, leading to decreased recognition accuracy.
- RL required SFT initialization in this setting: with a Llama-3.2-Vision-11B-scale base model and strict structured-output requirements, end-to-end RL without SFT failed completely because base model could not produce scoreable structured outputs. Specific to the setting, not universal; a sufficiently strong base model can skip SFT and succeed with direct RL (see DeepSeek-R1-Zero discussion).
- Another finding: more verification iterations produced better measured generalization — **10 iterations yielded +5.99% versus +0.48% for one iteration** — test-time computation an important factor in the observed gain.
- Why SFT degraded while RL performed better: limited SFT data reinforced fixed pattern "treat J/Q/K as 10," which remained active when J changed to 11. The outcome-trained RL branch was more likely to reinforce a strategy of recalculating until reaching the correct result, allowing the same procedure to apply after the rule changed. This explains the memorization-vs-generalization contrast; it does not imply SFT can only memorize or RL must learn a general algorithm.
- Core contribution: systematic quantification, within the limited GeneralPoints setting, of SFT's overfitting tendency and RL's better OOD performance, with same pattern in both text-only and vision-language variants. In this setting, **SFT stabilized the format and RL explored strategies on that foundation — the two methods complementary.**

## 8.9 RL Algorithms: From 16 Rollouts to One Parameter Update

- **GRPO (Group Relative Policy Optimization)**, proposed by DeepSeek, is one of the most widely used RL training algorithms today.
- Concrete example (SWE-bench task): `parser.py` in some Python project raises an IndexError on empty input; Agent must fix the code without modifying the tests. Four steps:

**Step 1: Let the policy model try repeatedly.** The policy model is the language model currently being trained. Copy the same initial code and problem description into **16 mutually isolated sandboxes**; let the model solve it 16 times independently. Each attempt covers the full "read the code → edit the files → run the tests → submit the result"; the entire process is one rollout. Problem and initial environment identical, but sampling is stochastic, so the 16 attempts may take different paths: some correctly add the boundary check, some merely catch the exception and paper over the problem, some edit the wrong file, some try to modify the tests.

**Step 2: Compute the reward.** After each rollout ends, a verifier applies the patch in a clean environment and runs the tests. Suppose 4 of the 16 attempts pass all tests without touching test files, and the other 12 fail; then the first 4 receive reward 1 and the other 12 reward 0. "Computing the reward" is nothing mysterious — using tests and rules to judge whether the fix is correct. Only for open-ended tasks with no definitive test do you need human preference or a reward model to do the judging.

**Step 3: Compute the relative advantage.** A reward only says whether a single trajectory succeeded or failed; relative advantage says how good it is compared with other attempts in the same group. This group's average success rate is 4/16: the 4 that passed are above the group average (positive advantage); the 12 that failed are below (negative advantage). **This within-group comparison is the core of GRPO.** If all 16 fail, or all 16 succeed, every reward is identical, no way to tell which is better, and the relative advantage vanishes. RLVP's path signals, process rewards, and partial-progress rewards exist precisely to restore meaningful differences within such groups.

**Step 4: Update the policy by gradient descent.** Turn the relative advantages into a training loss, compute gradients, have an optimizer (AdamW, Muon, etc.) perform gradient descent — raising the probability of choices in positive-advantage trajectories, lowering it in negative-advantage ones. It does not memorize some successful patch verbatim; it adjusts gradually across many tasks and rollouts, so that when a similar bug appears later, "reproduce the problem, check the boundary condition, change the implementation, and run the tests" is more likely to occur, while "swallow the exception, edit the tests, submit without verifying" is less likely.

**Figure 8-13: 16 Rollouts, Verification, and Relative Advantage on the Same SWE-bench Task**
- Input Prompt x; Policy model πθ parallel sampling: y₁: (8+6)×3-4 = 38 ✗; y₂: 3×4×(8-6) = 24 ✓; y₃: 8×4-6-3 = 23 ✗; y₄: (8+4)×3÷6 = 6 ✗.
- Reward calculation: R₁ = 0 (incorrect); R₂ = 1 (correct); R₃ = 0; R₄ = 0.
- Relative advantage (no value network needed): A(yi) = (Ri - mean(R)) / std(R) → A1=-0.58, A2=+1.73, A3=-0.58, A4=-0.58.
- Policy update (symmetric clipping): L = -E[ min(r(θ)·A, clip(r(θ), 1-ε, 1+ε)·A) ]; Symmetric clipping range [1-ε, 1+ε], ε=0.2; Advantage from intra-group relative comparison, no value network needed.
- Core advantages of GRPO: No reward model/value network · Intra-group relative comparison · Retains online exploration capability · Suitable for verifiable reward scenarios.

- These four steps make up one training iteration = one step: step k generates a batch of rollouts with the current policy, completes reward, advantage, and gradient computations, updates parameters; step k+1 rolls out again with the updated policy. Training for 100 steps = repeating this loop ~100 times. A given RL training framework may count internal minibatch updates separately, so confirm how it defines a step when reading logs.
- **Rough time estimate**: a complex Agent rollout generates dozens of tool-calling turns; even with 16 in parallel, wall-clock of rollout stage set by the slowest one. Suppose slowest rollout ≈ 2,000 seconds and gradient descent + optimizer update ≈ 600 seconds; one step ≈ 2,000 + 600 = 2,600 seconds ≈ 43 minutes; 100 consecutive steps ≈ nearly 72 hours.
- **PPO vs GRPO**: both follow this loop; differ mainly in what they compare against. GRPO directly compares multiple rollouts of the same problem, no separate value model. PPO trains a value model estimating "how well one typically does" at each step of a trajectory, then judges whether the current action beats that expectation — suits long trajectories needing fine-grained credit assignment. Both limit the size of a single update so a small batch cannot suddenly change the model too much.
- **DPO is different**: learns directly from pre-collected "better response—worse response" preference pairs, never has the current policy generate this group of rollouts online.
- Among this chapter's cases: AdaptThink uses a custom constrained objective; GeneralPoints and V-IRL use PPO with a value model; SimpleVLA-RL and RLVP use GRPO; ReTool uses PPO. **The algorithm decides how trajectories are compared and parameters updated; the reward decides what counts as success; the environment and data decide which problems the model gets to experience.**

### 8.9.1 Why LLM RL Usually Prefers On-Policy Data
- Separate two easily conflated terms:
  - **Online**: data is continually produced through interaction with an environment during training.
  - **On-policy**: requires the behavior policy μ that generates rollouts to be identical, or sufficiently close, to the policy π_θ currently being optimized.
- An asynchronous cluster may generate data continuously yet still become off-policy in the statistical sense if rollout workers lag several checkpoints behind. Replaying old trajectories or using trajectories generated entirely by an older model or a teacher is more clearly off-policy.
- The PPO/GRPO recipes in this chapter generally aim to roll out again from the latest policy at every step. When PPO performs several minibatch epochs on the same batch, later epochs already drift away from the old_policy that generated the data — precisely why PPO uses a probability ratio and clipping.
- **Policy gradients** seek to estimate expected reward under the current policy π_θ. If data was sampled from another policy μ, the correction uses an **importance ratio**:
  - **ρ_t = π_θ(a_t|s_t) / μ(a_t|s_t) = exp( log π_θ(a_t|s_t) − log μ(a_t|s_t) )**
- Genuinely fresh on-policy rollouts satisfy π_θ = μ before any parameter update, so ρ_t = 1. This focuses training on "the states the current model actually enters" and avoids paying a high-variance correction for distribution mismatch.
- Off-policy data advantages: old data can be reused, sampling and training can run asynchronously, higher throughput. But the staler the policy, the heavier the tail of the ρ_t distribution. For long autoregressive sequences, a strict prefix or trajectory correction also multiplies many per-token ratios together, so a small bias can accumulate into an enormous or vanishing weight.
- PPO's clipping limits outlier updates but cannot losslessly restore lost distribution coverage: clip too much → gradients discarded; clip too little → a handful of samples dominate.
- "On-policy is better" is NOT a universal theorem; in today's LLM policy gradients it usually means lower distribution bias and more stable optimization. Empirical work on stabilizing large-model RL finds that reducing policy staleness and training–inference discrepancy is an important condition for the surrogate objective to remain valid (Zheng, Chujie et al., "Stabilizing Reinforcement Learning with LLMs: Formulation and Practices", 2025, arXiv:2512.01374).

### 8.9.1.1 Why Training Is Sensitive to Sampler/Trainer Numerical Mismatch
- Large-scale LLM RL typically generates rollouts with an inference engine such as **vLLM/SGLang**, then recomputes log probabilities and gradients with a training engine such as **FSDP/Megatron**. Even when both sides load the same weights, differences in floating-point precision, reduction order, tensor-parallel layout, batch size, KV cache, and fused kernels can make the log probability of the same token differ slightly. As a result, ρ_t — which should equal 1 before the update — has already drifted away from 1: the system nominally synchronized weights, but numerically turned on-policy training into off-policy training. Controlled experiments show a tiny token-level training–inference discrepancy on its own is enough to trigger training collapse (Zhong, Tianle et al., "Diagnosing Training Inference Mismatch in LLM Reinforcement Learning", 2026, arXiv:2605.14220).
- The sensitivity comes from an **amplification chain**: a small log-probability error → an exponentiated probability-ratio deviation → accumulation over a long prefix → changed clipping/advantage weighting → a changed gradient direction and effective sample count.
  - Example: if the log ratios of 4,000 tokens all deviate by 10⁻³ in the same direction, the trajectory-level ratio accumulates to e⁴ ≈ 54.6; real errors are not necessarily same sign, but the example shows why long sequences amplify errors that "look tiny on every single token."
  - A minute probability difference on an early token can also change which token is actually sampled, causing the entire subsequent state trajectory to diverge.
  - End result: not merely "the same prompt occasionally produces a different answer" — importance ratios can spike, large numbers of tokens can be clipped, gradients or response lengths can jump, after which reward and entropy collapse together.
  - Changing the batch size alters how computation is reduced and breaks numerical batch invariance; directly observed to turn nominally on-policy RL into implicit off-policy RL. Either matching sampling/training numerics or applying an explicit off-policy correction improves stability (He, Horace and Thinking Machines Lab, "Defeating Nondeterminism in LLM Inference", 2025).
- **Engineering should treat this as a core problem, not ordinary floating-point noise**:
  - Before any parameter update, compare the sampler's and trainer's token log probabilities on the same batch of trajectories; monitor mean, quantiles, maximum, approximate KL, and clipped fraction of ρ_t — the most direct on-policy unit test.
  - What must be synchronized is not only weights, but also the LoRA adapter, tokenizer, chat template, model revision, and positional-encoding configuration; the rollout should store the behavior log probability from generation time, rather than passing off the current model's numbers after the fact.
  - Align precision, parallel layout, and key compute kernels of sampling and training as far as possible; where impossible, treat the difference explicitly as off-policy, apply importance correction, monitor effective sample size, instead of assuming PPO clipping will automatically compensate.
  - Keep rollouts fresh; limit both the number of update epochs per batch of data and the asynchronous staleness. Reusing old data buys throughput, but it should be a measured bias–efficiency trade-off, not a free speedup.

## 8.10 RL Environments: From Evaluation to Simulation

- The bottleneck in RL training is often **not the algorithm but whether the environment is realistic, resettable, and parallelizable enough**. A real Agent's phone calls, payments, or file modifications can be expensive and irreversible; one mistake cannot be made good by unlimited retries. Chapter 7's evaluation environment can supply the verifier, but training additionally requires the Agent to fail repeatedly, absorb the side effects of its actions, and stay stable across millions of interactions. Environment engineering is therefore a **precondition for RL**, not an afterthought.

### 8.10.1 Environment: The Training Ground for the Model
- RL is fundamentally "learning by trial and error," and trial and error needs somewhere to happen — the **simulation environment**. The model runs tasks over and over, collects feedback, adjusts policy. The environment's **fidelity** (how closely it resembles real deployment) directly determines whether the resulting policy is usable:
  - **A distorted environment guarantees a useless policy.** If the simulated customer always answers from a fixed script and error messages don't match production, the model learns a test-taking strategy that only works in simulation and falls apart on first real deployment. This is the most common way RL projects fail — not a bad algorithm, but a practice ground that is not the same as the exam hall.
  - **Building a high-fidelity environment is often more expensive and harder than the training itself.** An environment massively parallel, reproducible, and realistic in feedback usually takes far more engineering than tuning the model. The tool-calling experiments later (AWorld's MCP sandbox, ReTool's code-interpreter sandbox) invest heavily in the environment precisely because real APIs have rate limits, ban accounts, and have side effects — unusable for direct training; you must build a stable, controllable, replayable "shadow world" first.
  - **The other half of the environment is the reward function.** The environment must not only simulate how the world changes but also judge how well the Agent did — the input to the reward design discussed later.
- In a nutshell: before tuning algorithms, ask — does my simulation environment truly resemble the real world? The answer matters far more than choosing between PPO and GRPO.

### 8.10.2 What If You Can't Build an Environment? Let the Model Play the Environment
- More fundamental problem: in many scenarios a high-fidelity environment is not merely expensive, it **cannot be built at all** — real APIs have side effects and cannot be called at random, real users cannot be experimented on, the physical world cannot be fast-forwarded. If you cannot stand up a usable "shadow world," is RL off the table? An increasingly mainstream idea: **use a model to simulate the environment** — have an LLM play the environment and generate the feedback the Agent's interactions require. Two levels:
  - **Level one: the model synthesizes the return values of tool calls.** Take **ZeroSearch** (Sun, Hao et al., "ZeroSearch: Incentivize the Search Capability of LLMs without Searching", 2025, arXiv:2505.04588): training a model that "knows how to search" normally requires a real search engine, but search APIs cost money, have rate limits, return uncontrollable results. ZeroSearch simply has an LLM play the search engine: the student issues a search query, the "simulated engine" generates the retrieval results it returns. Better still, it uses a **curriculum design** — early in training the simulated engine returns high-quality, highly relevant documents; as training proceeds it progressively mixes in noise and lowers quality, forcing the student to learn to extract useful information from the kind of imperfect results a real search engine gives. In the end, a model that never saw a real search engine during training still performs well when connected to one.
  - **Level two: the model simulates the whole environment's dynamics.** Not just a single tool's return value, but "what the world looks like after an action is taken" can also be handed to a model. **DreamGym** ("DreamGym: Scaling Agent Learning via Experience Synthesis", 2025, arXiv:2511.01824) distills environment dynamics into a reasoning-style "experience model": given the current state and Agent's action, it reasons step by step to the state transition and feedback signal, synthesizing rollouts in bulk for online RL without touching the real environment. Training customer-service and sales Agents commonly uses an LLM to play the user (a **user simulator**), and the **τ-bench family of evaluations** is built on this idea — the same model simulator can serve as both exam hall and practice ground.
- **Risk of this route must be stated plainly**: the simulator's knowledge of the world is the **ceiling** on training, and the simulator's systematic biases will be adopted wholesale by the policy. If the simulated customer is more patient than real users, or the simulated search engine never returns junk, the student learns a strategy that only holds in "the world as the model imagines it"; worse, RL will actively seek out and exploit the simulator's flaws, which is **reward hacking**.
- **Prudent engineering answer: a hybrid** — let model simulation carry most of the interaction volume, supplement it with interactions in the real environment, and use those real interactions to periodically calibrate the simulator's bias.

### 8.10.3 Environments, Task Distribution, and Evaluation Isolation
- The environment itself determines what RL can learn: it must be **resettable, parallelizable, reproducible**, and return a trustworthy verification result after each state transition. Training tasks come from the same source as SFT data synthesis — distill task blueprints from real business logs, then strip identifying information and regenerate fictional people, orders, files, and states.
- Isolation requirements same, with one addition specific to RL: **the training and evaluation environments may share the task generator and verification code, but must not share the same set of tasks**. SWE-Gym, τ²-bench, and AndroidWorld all illustrate this (Pan, Jiayi et al., "Training Software Engineering Agents and Verifiers with SWE-Gym", 2024, arXiv:2412.21139; Barres, Victor et al., "τ2-Bench: Evaluating Conversational Agents in a Dual-Control Environment", 2025, arXiv:2506.07982; Rawles, Christopher et al., "AndroidWorld: A Dynamic Benchmarking Environment for Autonomous Agents", 2024, arXiv:2405.14573): test cases, hidden state, and reference solutions belong on the verifier's side.
- Use a small number of rollouts first to check "is the task completable, and can the verifier tell right from wrong," then scale up sampling; if the verifier itself has a systematic bias, RL will only exploit it faster.
- **Order for environment engineering**: task blueprint → resettable simulator → deterministic verifier → training/evaluation isolation → calibration with a small amount of real interaction.
- SFT data synthesis constructs stable demonstrations; the environment here serves RL, letting the current policy fail repeatedly and explore paths beyond the demonstrations.
- A deterministic verifier being "cheap" is not the same as being free. A Lean kernel, test runner, or container execution can make CPU verification far slower than GPU generation; **throughput is then set by the number of parallel verifier workers, not by adding more GPUs**.

## 8.11 From Single-Turn to Multi-Turn: Task Scenarios and Credit Assignment

### 8.11.1 The Core Challenge of Multi-Turn Tasks
**Figure 8-14: Comparison of Single-Turn RL and Multi-Turn RL**
- Single-turn (GeneralPoints / AdaptThink): Prompt: "Use [7,J,3,5] to calculate 24" → LLM generates thinking + answer in one go → Verify answer → R ∈ {0, 1} → Strategy update. No state maintenance · Immediate reward · Simple credit assignment.
- Multi-turn (V-IRL navigation / AWorld tool): Step 1: Observe street view → Decide to turn left; Step 2: Identify landmark → Correct direction; Step 3: Wrong turn → R=-1; ... Step 20: Reach target → R=+1. Cross-step state · Delayed reward · Difficult credit assignment.
- Credit assignment: immediate per-step feedback (single) vs delayed; which step earns it? (multi).
- State management: no hidden state (single) vs partial obs.; keep history (multi).
- Reward design: outcome reward (0/1) (single) vs process vs outcome reward (multi).
- Exploration cost: single generation (single) vs full trajectory (10-50 steps) (multi).

**Figure 8-15: Credit Assignment in Multi-Turn Interactions** (V-IRL navigation example)
- Step 1: Observe ahead, Choose left turn, R=+1.
- Step 2: Identify sign, Confirm direction, R=+1.
- Step 3: Miss the turn, Continue straight, R=−1.
- Step 4: Detect error, Turn around and go back, R=0.
- Step 5: Reach target, Task complete, R=+10.
- **Key question**: Which step should the final reward +10 be attributed to? Step 3 makes a mistake but step 4 corrects → step 4's correction is more critical than steps 1-2's correctness.
- Process reward (V-IRL navigation): immediate feedback per step, correct +1 / wrong −1. ✓ Reduces credit assignment difficulty; ✗ May limit exploration.
- Outcome reward (SimpleVLA-RL): only terminal feedback, success 1 / failure 0. ✓ Maximum exploration freedom; ✗ More difficult training.

- Going from single-turn to multi-turn is a **qualitative jump** in complexity. The policy must not only choose the best action now but also consider the value of future states; must handle not only immediate feedback but also **credit assignment under delayed rewards** — deciding which step in a multi-step sequence contributed most to the final outcome. Customer-service example: 10 turns of dialogue resolve a user's problem and finally earn a positive rating — should credit go to the precise question asked in turn 2, or to the patient explanation in turn 7?
- The multi-turn interaction here is exactly the **ReAct loop** described in Chapters 1 and 4 — each turn is one think → act → observe iteration, and the delayed reward comes from the structural constraint that "how good the final outcome is can only be judged several turns later."

**Experiment 8-12 ★★★: V-IRL-VL—Multi-Turn Visual Navigation**
- V-IRL (Yang, Jihan et al., "V-IRL: Grounding Virtual Intelligence in Real Life", 2024, arXiv:2402.03310) has the Agent navigate continuously through real urban street scenes: training uses New York routes, while testing transfers to different cities and changes both the phrasing of directions and the visual appearance.
- RL clearly outperforms SFT on both rule OOD and visual OOD, showing that in multi-turn tasks the policy must learn to **re-plan from the current observation** rather than reproduce training trajectories.
- Uses **PPO with a value network**; step-by-step feedback observed to ease long-horizon credit assignment.

**Experiment 8-13 ★★★: SimpleVLA-RL—Open Exploration Under Outcome Rewards [Extended Experiment]**
- Uses only success/failure outcome rewards on **LIBERO robotics tasks**. Each task gets just one demonstration trajectory for SFT cold start; RL then lifts the success rate from **17.3% to 91.7%** and discovers a **"pushcut" action** that never appeared in the demonstrations.
- Contrasts with V-IRL: when process signals are easy to define they accelerate learning, but when the optimal path is unknown a sparse outcome reward preserves far more room for exploration.

### 8.11.2 Tool Calling: Bringing the Environment Into the Agent
- Once a multi-turn task connects to external tools, actions are no longer just "move or answer" but searching, executing code, editing files, querying databases, and composing several APIs. Tool calling pushes **credit assignment, environment engineering, and safety constraints** to the foreground all at once.

**Figure 8-16: Tool Calling RL Reward Loop**
- policy model πθ generate action: think/search/code/answer → tool execution: API call / code sandbox → result parsing: success/failure/error message → reward calculation: correct +1 / incorrect −1 → policy gradient update → loop.
- Three levels of challenges in tool-use RL:
  - Level 1: Single tool mastery — when to call + how to call + error handling.
  - Level 2: Multi-tool selection — search / code / document parsing: choose one of three.
  - Level 3: Tool chain orchestration — dependency management + mutual exclusion constraints + cost optimization.

- **Search-R1** (Jin, Bowen et al., "Search-R1: Training LLMs to Reason and Leverage Search Engines with Reinforcement Learning", 2025, arXiv:2503.09516) represents the retrieval-augmented route: the model decides on its own when to search and what to search for, and uses returned results to continue reasoning.
- **ReTool** instead embeds a code interpreter into the thinking loop, so the model must learn when to execute code, how to read the feedback, and how to correct itself from error messages.
- **AWorld-train** provides an MCP multi-tool sandbox, further introducing tool selection, dependency management, state reset, and replayability.
- **Crucial implementation detail for tool trajectories**: the tokens returned by the environment are NOT generated by the policy, so when computing the policy gradient those feedback tokens should be **masked**, and gradients propagated only through the model's own thinking and its tool-call arguments. Otherwise the model is trained to predict sandbox output instead of learning how to use tools.

**Experiment 8-14 ★★★: ReTool—Code Interpreter Enhanced Math Problem Solving**
- **Figure 8-17: ReTool Interleaving Text-Code Thinking and Sandbox Execution Feedback Loop**
  - Text Thinking₁: "Need to solve the equation x²+2x-8=0. First solve with sympy..."
  - Code Generation₁: `<code> from sympy import *; x=symbols("x"); solve(x**2+2*x-8,x) </code>`
  - Sandbox Feedback₁ (feedback tokens shown in the figure as `<interpreter> [-4, 2] </interpreter>`)
  - Text Thinking₂: "Solution: x=-4 or x=2; Verification: 2²+2×2-8=0 ✓"
  - Final Answer: \boxed{-4, 2}
- SandboxFusion: 128 parallel workers; Python sandbox execution; sympy/numpy/scipy; Timeout control + memory limit.
- PPO training loop: Base model Qwen2.5-32B; SFT cold start ~1 hour; RL training ~9 days / 400 steps; Per step: 32 questions x 16 candidates = 512; Average 7-9 rounds of interaction/response.
- AIME 2024 accuracy: SFT baseline 25%; After 110 steps 52%; After 400 steps 67%.
- After an SFT warm-up, ReTool trains with PPO on interleaved text reasoning, code execution, and interpreter feedback. Shows how tool feedback changes the thinking strategy: the model gradually learns to execute proactively, read errors, and correct itself.
- Training data comes from **DAPO-Math-17k**, but the optimization algorithm is still standard PPO (Feng, Jiazhan et al., "ReTool: Reinforcement Learning for Strategic Tool Use in LLMs", 2025, arXiv:2504.11536; Yu, Qiying et al., "DAPO: An Open-Source LLM Reinforcement Learning System at Scale", 2025, arXiv:2503.14476).
- On AIME 2024, training raised accuracy from about 25% to 67.0%; compared with pure-text RL, code feedback let the model learn precise calculation and error correction faster. Detailed training dynamics and sandbox configuration are in the experiment's companion notes.

**Experiment 8-15 ★★★: AWorld-train—Learning to Use Tools in a Sandbox**
- **Figure 8-18: AWorld-train MCP Sandbox Training Architecture and Tool Ecosystem**
  - (1) Environment Setup: MCP Server Sandbox, 26 Servers, 126 Tool Functions.
  - (2) Agent Construction: AgentLoop Decision, System Prompt + Tools, Multi-turn Dialogue (<=32 turns).
  - (3) Adapter Layer: VeRL Interface Unification, Rollout Collection, Reward Calculation.
  - (4) Training Framework: GRPO / PPO, Policy Gradient Update, 131K Context.
  - MCP Tool Ecosystem (GAIA Evaluation Environment):
    - Web Interaction (3): Google Search, Smart Browser, Playwright.
    - Document Processing (4): CSV/DOCX, PPTX/PDF, Structured Extraction.
    - Multimedia (3): Audio Transcription, OCR Recognition, Video Summarization.
    - Code Execution (3): Terminal Commands, E2B Sandbox, File Management.
    - Excel (1): 29 Operations, Formulas/Charts, Pivot Table.
    - Knowledge Retrieval (3): Wikipedia, ArXiv Papers, Wayback.
  - Distributed Rollout Architecture: Data Collection, 14.6x Speedup (7 days -> 12 hours).
  - Model: Qwen3-4B (This Experiment), 8xA100 Cluster; Training Configuration: batch=32, 16 resp/prompt; GRPO, lr=1e-6.
  - Evaluation: GAIA benchmark; Reference ~32% for LLMs.
- AWorld-train uses an MCP server sandbox providing web, document, multimedia, code, and knowledge-retrieval tools. The point of this open-ended experiment is not to push GAIA numbers but to get a resettable, replayable multi-tool training loop running end to end, and to observe whether tool-call success rates and composition strategies improve with training.
- These scenarios together make the same point: the difficulty in training multi-turn Agents is NOT "whether there is a fancier optimizer," but whether environment feedback is reliable, whether the action chain is verifiable, and how the final reward should be attributed to intermediate decisions.

## 8.12 Reward Design: Turning Task Goals into Learning Signals
- The single-turn, multi-turn and tool-calling scenarios established what to train; this section answers how the environment should tell the model whether it did well. Reward design unfolds along three complementary dimensions: **where the reward comes from, when it is given, and how much information it must express.** A fourth question follows: when the outcome is correct, was the path also acceptable?

### 8.12.1 Where the Reward Comes From: Rules, Human Preference and Model Judgment
- Most reliable source: a **verifiable reward (RLVR)** — judge the result directly with test cases, database assertions, state diffs or format checks. Mathematical answers, code tests, and structured tool calls are all good starting points for a binary outcome reward. The more deterministic the rule, the cheaper and more reproducible the reward, and the harder it is for the model to game.
- **RLHF** is background here. The basic **InstructGPT** pipeline (Ouyang, Long et al., "Training Language Models to Follow Instructions with Human Feedback", OpenAI, 2022): humans compare responses, a reward model is trained, PPO then optimizes the policy. The reward model is only a proxy for preference, and over-optimizing it leads to **reward hacking** (Gao, Leo, John Schulman, and Jacob Hilton, "Scaling Laws for Reward Model Overoptimization", OpenAI, 2023), which is why a **KL penalty** is normally used to anchor the policy near the SFT reference. **DPO** (Rafailov, Rafael et al., "Direct Preference Optimization: Your Language Model is Secretly a Reward Model", 2023) skips the explicit reward model and optimizes offline from preference pairs directly. These methods are not the main line of Agent RL in this chapter.
- When the goal cannot be fully reduced to rules, **model judgment** is an option. A **generative reward model (GRM)** emits not just a score but a diagnosis of what went well and what needs to change; it can serve as a reward source, and its diagnoses can be turned into distillation or preference data. The core idea of **DeepSeek-GRM** (Liu, Zijun et al., "Inference-Time Scaling for Generalist Reward Modeling", 2025, arXiv:2504.02495) is to have the model first induce evaluation principles for the task, then evaluate the trajectory against those principles, and finally check the evaluation itself against verifiable facts. The resulting feedback is more transparent, but still needs **sampled human calibration** so the judge does not develop biases of its own.
- Two easily confused notions:
  - **Reward hacking**: exploiting a rule or an implementation hole to score highly.
  - **Reward seeking**: the model first builds an internal picture of what the grader will look at, then adjusts its behavior to that guess. The latter need not tamper with tests or fabricate results, yet on long-horizon tasks it can lead the model to set itself a very shallow check, stop as soon as it passes, and deliver something satisfying the proxy metric but not the real intent (storm, "Long-horizon agent self-checking and early stopping: the reward-seeking phenomenon and its mitigations", Qingke Community, 6 Aug 2026).
  - So "it passed the grader" cannot be equated with "the task is done": the grader is a proxy for intent, and the harder you train, the more likely the model is to treat the proxy as the goal itself.

### 8.12.2 When the Reward Is Given: Outcome or Process
- An **outcome reward (ORM)** judges only at the end of the episode whether the task was completed. Simplest; gives the policy the most freedom to explore. When there is no agreed standard for the intermediate path and the optimal solution has not yet been found by humans, SimpleVLA-RL's sparse success/failure reward is the right starting point. Sparse feedback makes it hard for the model to localize a specific mistake in a multi-step trajectory — one long-standing reason RL sample efficiency is limited (Silver, David and Richard S. Sutton, "Welcome to the Era of Experience", 2025).
- On long-horizon coding or cowork tasks, the "is it done" judgment should be handed to **hidden tests, state assertions, or an external termination hook** that the model cannot write — never to the model's own claim of completion.
- **"Premature completion"** is a concrete example: when the model says the task is done, the harness runs acceptance tests the model cannot see, in an isolated workspace. Passing earns positive reward, failing earns negative reward. Those tests must read real files or environment state rather than checking whether the model said "done," or the model will learn to promise verification without performing it. During evaluation, keep a **boundary set of unfinished tasks separate from a held-out set of genuinely finished ones**: the former shows the premature-stop rate, the latter shows whether the model can still close out normally — otherwise you train a model that never dares to finish.
- A **process reward (PRM)** gives feedback at intermediate steps, checking things like authentication, tool arguments, the number of passing tests or navigation actions. OpenAI's "Let's Verify Step by Step" (Lightman, Hunter et al., 2023) showed the value of step-by-step verification in mathematical reasoning. Process rewards ease long-horizon credit assignment, but they can confine the model to the path the designer had in mind, and they cost more to label and validate.
- V-IRL-VL (Experiment 8-12) uses step-by-step navigation feedback while SimpleVLA-RL (Experiment 8-13) keeps only the endpoint reward — together they form a controlled contrast: **dense feedback buys convergence speed; sparse feedback buys exploration space**.
- In practice: establish a reliable baseline with outcome rewards first, and only then add process signals for intermediate events that are genuinely verifiable. Multi-turn LLM RL usually sets the **discount factor gamma = 1**; PPO's value network or turn-level advantage attributes endpoint feedback back to earlier actions, while GRPO spreads a trajectory-level advantage across the generated tokens, so **signal dilution** deserves particular care on long trajectories.

### 8.12.3 How Much Information the Reward Must Express: Scalar, Vector, Generative Diagnosis
- The density of a reward and its representation are two different things.
  - A **scalar** answers only "how good overall."
  - A **semi-scalar** gives a brief reason and then a score.
  - A **vector** scores separately along dimensions such as accuracy, completeness, cost and safety.
  - A **generative reward** produces a natural-language diagnosis that can be sampled several times and aggregated.
- Selection rule:
  - A definite answer or test exists: prefer a **binary scalar**.
  - Several mutually independent quality goals: use a **vector**, or weight the dimensions into a scalar.
  - Open-ended and hard to enumerate as rules: use **generative diagnosis**, but pair it with fact-checking and sampled human review.
- Do NOT stack unverifiable dimensions in the name of a "richer" reward. Every additional evaluation dimension adds one more way for the policy to game it. Confirm first that the signal produces meaningful within-group variation across a handful of rollouts, and only then decide whether it belongs in training.

### 8.12.4 A Correct Outcome Is Not Enough: Path Constraints and RLVP
- An outcome reward settles whether the job got done, but it cannot express whether it was done the way it was supposed to be. A real Agent may achieve surface success by editing the test file, skipping authentication, or running a destructive command.
- The principle behind **RLVP (Reinforcement Learning with Verified Penalty)** (Li, Bojie and Noah Shi, "RLVP: Penalize the Path, Reward the Outcome", 2026, arXiv:2607.07435): **reward the outcome, penalize the path.** It targets machine-decidable, outcome-neutral constraints that have no bearing on final success or failure, and it is not a substitute for independent checks on semantic intent, delivery completeness, and early-stopping behavior.
- Real environments are typically **asymmetric verifiers**: detecting "a bad action was taken" is cheap and reliable, whereas proving "this step made meaningful progress toward the goal" is hard.
- Write the total reward as **R = O + beta*Phi**, where **O** is the task outcome and **Phi** is a path signal computed per action by deterministic rules. Deduct points for verifiable violations; give a small partial reward for verifiable compliant actions or reachable sub-goals; normalize the two channels before combining them so the path signal cannot drown out the main objective. None of this changes PPO or GRPO — it changes only the reward seen at each step.
- Implementation (split verifier output into two channels, hand to existing policy optimizer):
  ```
  outcome = verify_final_state(trajectory)   # result, not self-report
  path_signal = 0
  for step in trajectory:
      path_signal += deterministic_path_signal(step)   # penalty or reachable progress
  reward = normalize(outcome) + beta * normalize(path_signal)
  ```
- Which actions are permitted, which sub-goals are reachable, what the hidden tests are, and how evidence is recorded all depend on the specific environment. The text only explains how the outcome reward and path constraint merge — one environment's rules should not be mistaken for a general algorithm.
- The point of RLVP is NOT that "denser rewards are better" but whether **within-group variation can be restored**. A pure outcome reward produces zero variance and no gradient in both all-fail and all-succeed groups. Violating actions are usually easy to detect, so a penalty almost always restores the variance; a progress reward only works when partial progress is actually reachable.
- **Four design rules**:
  1. Penalize specific actions, never "insufficient effort."
  2. Always keep the outcome reward so the model does not learn to do nothing.
  3. Pair every penalty with a reachable compliant path where possible.
  4. Make the rules deterministic and hard to game.
- If the base policy would never sample the compliant action at all, seed that path with a few demonstrations first, and taper the path shaping once compliant behavior is stable. Put differently: **the penalty is the half that is usually reachable, and the progress reward is the half gated by reachability.**

**Experiment 8-16 ★★★: RLVP — Reward the Outcome, Penalize the Path**
- Add an outcome reward O and a path signal Phi on top of GRPO and compare against a pure outcome reward.
- On **TerminalBench**: violations drop from **3.71 to 0.66** while the success rate is essentially unchanged.
- On **miniF2F**: a reachable partial reward cuts the iterations needed to reach a 0.9 success rate from **7.0 to 4.4**.
- In software repair, where no rollout passes any test, the progress signal is unreachable and adding it brings no benefit.
- **Lesson: test whether the signal is reachable before deciding to add a reward dimension.**
- These numbers come from controlled proxy environments and cannot be extrapolated directly into equivalent gains for a production Agent. The safer conclusion is mechanistic: as long as the path signal distinguishes behaviors within the same group of rollouts and the rules are hard for the policy to game, it fills in exactly the information the endpoint reward cannot see. Real deployments additionally need hidden verification, trajectory monitoring, and external termination conditions built into the harness.

## 8.13 Distillation: Improving Sample Efficiency
- The experiments above systematically showed RL's core value in Agent training, but every one paid a steep sample cost. "Sample efficiency" here means: how many effective parameter updates each expensive environment interaction buys, not merely training steps or GPU hours. ReTool's RL training took more than **200 times as long** as its SFT (9 days versus 1 hour), making reducing environment sampling especially valuable.
- RL's low sample efficiency comes from high variance and the difficulty of reusing on-policy data, but the more fundamental cause is that **feedback is too sparse**. Mainstream model-free RL typically yields a single success/failure scalar at the end of one rollout; the reason for an intermediate mistake, a missing field, or a hint about the procedure carries no direct learning signal. When a customer-service script says "I need the last four digits of the credit card," the model can only trial-and-error its way there from a final 0/1 outcome, perhaps taking hundreds of interactions to stumble onto that step — whereas a human remembers it after hearing it once.
- **Distillation turns one rollout into a dense supervisory signal**, letting a single trajectory contribute a large number of gradients without exploring any additional environment trajectories. That is the key to how distillation improves sample efficiency.

### 8.13.1 On-Policy Distillation: Making One Rollout Produce Dense Supervision
- **On-Policy Distillation** was systematically organized and popularized by **Thinking Machines Lab in 2025** (Thinking Machines Lab, "On-Policy Distillation", 2025). Here, "policy" refers to who generates the state prefixes on which the student learns, not who supplies the supervision.
- Comparison table (who samples the trajectory/state; main supervision per trajectory):
  | Method | Who samples the trajectory/state? | Main supervision per trajectory |
  |---|---|---|
  | SFT / off-policy distillation | Human or teacher | Dense token-level supervision from labeled answers |
  | On-policy RL | Current student | Usually sparse outcome or process rewards |
  | On-Policy Distillation | Current student | Dense teacher token distributions on student prefixes |
- SFT supervision is dense but mainly covers states that a teacher would visit. If the deployed student makes an early mistake that the teacher would not make, it enters a prefix absent from the training data; every subsequent prediction is then made in an unfamiliar state, and errors can compound along a long sequence.
- On-policy RL trains directly on the student's own state distribution (more relevant), but often receives only a success/failure signal at the end of the trajectory. **On-Policy Distillation combines the two**: the student decides where it goes, and the teacher supplies the full next-token distribution at the state the student has actually reached.
- A rollout of length **T** therefore no longer produces only one 0/1 signal but roughly **T sets of token-level supervision**. It follows the student's real errors more closely than off-policy SFT and supplies denser, lower-variance feedback than pure RL. Teacher inference adds compute but does not require a second set of environment trajectories.
- It still cannot create capability from nothing: the student must at least enter meaningful states the teacher can correct, and the teacher's policy cannot lie too far outside the student's effective support. If the base model lacks even the target language, domain concepts, or basic actions, first use Mid-training or off-policy demonstrations for a cold start, then switch to on-policy distillation.
- Why the numerical issue matters: On-Policy Distillation optimizes the teacher KL on states visited by the student's current policy. If the rollout engine actually samples from mu while the trainer computes another pi_theta, the training states are already off-policy even though no PPO ratio is used explicitly. Implementations should still verify sampler/trainer log-probability agreement before an update; otherwise nominal On-Policy Distillation degenerates into training with a distribution mismatch.
- Concretely, the student's predicted distribution is pulled toward the teacher's, usually by minimizing the **KL divergence** between them. Example: when the student generates "first query the API, then parse the return value...", the teacher can give a distribution at the current position of 80% "query," 15% "call," and 5% for everything else. Compared with a binary end-of-task reward, token-level alignment provides a far denser, lower-variance learning signal; the cost is the teacher's inference, which pays off especially well when environment interaction is expensive.
- **Basic pseudocode for on-policy distillation**:
  ```
  student_trajectory = rollout(student, task)
  loss = 0
  for state in student_trajectory:
      teacher_logits = teacher(state)
      loss += KL(student_logits(state), teacher_logits)
  update_student(loss)
  ```
- On tasks such as mathematics, reaching comparable performance takes roughly **one tenth the training steps** of pure RL. In multi-turn Agents, where the success signal arrives later and more sparsely, the teacher's token-level distribution can guide intermediate decisions directly — but only if the simulation environment is realistic enough that the states the student explores stay close to the deployment distribution; otherwise the teacher's scores on unfamiliar, off-distribution states are unreliable too.
- The principle "dense signals beat sparse signals" has also been verified in a pure Agent setting. The author and collaborators once compared DPO, four RL variants, and On-Policy Distillation on a "sense of time" task: the first group was limited by sparse rewards, objective mismatch, rollout-shape mismatch, and policy collapse, respectively. Switching to a frozen **Qwen3-32B** teacher and aligning token by token on the student's own multi-turn trajectories, training converged smoothly, and pass rates across the four conditions were **23 to 47 percentage points above the same-source SFT baseline** (Li, Bojie and Noah Shi, "Agents That Sense Physical Time: Urgency, Persistence, and Vigilance as Missing Controls for LLM Agents", 2026). This suggests the bottleneck is often not that the reward function is insufficiently sophisticated, but that each interaction supplies too little signal.

### 8.13.2 What If There Is No Stronger Teacher? On-Policy Self-Distillation
- On-Policy Distillation's power comes from the teacher, saddling it with a hard prerequisite: there must be a teacher model clearly stronger than the student. In many settings that does not hold (e.g., training a vertical-domain model where every existing model falls short). Without a stronger teacher, is the dividend of dense signals out of reach?
- **On-Policy Self-Distillation (OPSD)** (Zhao, Siyan, et al. "Self-Distilled Reasoner: On-Policy Self-Distillation for Large Language Models", 2026, arXiv:2601.18734): the same model plays both teacher and student, but sees different context. The teacher version sees **"privileged information"** — a reference answer or a verified correct solution; the student version sees only the problem, yet aligns to the teacher version's token-level distribution on trajectories it sampled itself. Explaining a path the student just walked while holding the answer is usually easier than exploring independently, so one rollout still produces dense supervision.
- **OPSD as a constrained variant of the pseudocode above**:
  ```
  student_trajectory = rollout(model, task_without_answer)
  loss = 0
  for state in student_trajectory:
      privileged_state = add_verified_answer(state)
      teacher_logits = stop_gradient(model(privileged_state))
      loss += KL(model(state), teacher_logits)
  update(model, loss + retention_regularizer)
  ```
  - `privileged_state` may only be constructed on the training side and must not leak to the deployed Agent.
  - `retention_regularizer` stands for a retention set or style constraint, not some fixed hyperparameter.
  - The training pipeline must also check data permissions, answer masking, and the risk of forgetting.
- Compared with RLVR, OPSD does not require the reward to be automatically verifiable: the privileged information can be a reference answer, a human demonstration, or domain documentation. It uses that information in place of a stronger external teacher while keeping the sample-efficiency advantage of "on-policy sampling plus token-level supervision."
- But it does not create new knowledge out of nothing — if the model still cannot explain the process even while holding the answer, self-distillation yields no extra signal; naive OPSD can also make the model lose its original reasoning style, requiring additional regularization to stabilize (Shen, Ziqi, et al. "Purified OPSD: On-Policy Self-Distillation Without Losing How to Think", 2026, arXiv:2607.02234).

## 8.14 From Bad Cases to Post-Training
- Returns to the question left open in Chapter 7: how an evaluation dataset built from production bad cases actually becomes an input to post-training.
- Failure-attribution records, end-to-end regression tasks, trajectory-prefix regression tasks, and rubric scores each map to a different training use:

**Table 8-5. Mapping Chapter 7 evaluation data to Chapter 8 training uses**
| Chapter 7 evaluation data | Chapter 8 training use |
|---|---|
| End-to-end regression task with a verifier | RL rollout tasks and verifiable rewards (RLVR); the sampling pool for rejection-sampling fine-tuning (RFT) |
| Trajectory-prefix regression task | DPO preference pairs, SFT demonstrations for decision boundaries, and teacher states for On-Policy Distillation |
| Failure-attribution record (first erroneous step and error category) | Negative labels for process supervision (PRM); rules for RLVP path penalties |
| Multi-dimensional rubric scores and human gold set | Dimensions of vector rewards; training and calibration data for generative reward models (GRM) |

### 8.14.1 Case 1: Coding Agent premature completion
- **From bad case to attribution.** One of the most common and most stubborn Coding Agent failures is premature completion: declaring "done" before the tests have run; wrapping up after fixing two of the three features the user asked for; announcing "this task is impossible" after two failures. In Chapter 7's error taxonomy this belongs to "task completeness and logical judgment," and all three production signals catch it: user corrections ("you never ran the tests"), thumbs-down, and post-hoc audits (a trajectory claiming completion with no test tool call anywhere in it). The attribution record places the first error at the decision boundary where the Agent was "about to declare completion" — up to that point, reading and editing code may all have been fine; what was wrong was the step of "concluding without evidence." The reward seeking discussed earlier (setting up a shallow check that just barely passes, then finishing early) describes exactly this behavior.
- **Constructing the training data.**
  - **End-to-end regression task**: write "acceptance tests must pass before completion is declared" as a verifiable reward. Tests invisible to the model, run only when it claims to be done; passing scores +1, failing -1. This is the direct application of "leave the judgment to hidden tests the model cannot write"; it is this case's optional RL branch.
  - **Trajectory-prefix regression task**: cut at the "about to declare completion" decision boundary to build preference pairs — the rejected sample is the premature-completion behavior, the chosen sample is the desired "run the tests first, check the acceptance conditions one by one, and only then conclude." Chosen samples generated by a teacher model then filtered by a rule-based verifier (rejection sampling), yielding a batch of DPO training pairs. If there are too few bad cases, data augmentation (varying the task type, the missing verification item, the completion phrasing) can produce hundreds of preference pairs. Mix them into general task data at a small ratio for LoRA fine-tuning, so "always verify before wrapping up" does not become a new overfit and catastrophic forgetting risk stays low.
- **Evaluation: the boundary set and the retention set are both indispensable.** Post-training validation uses Chapter 7's evaluation datasets: the trajectory-prefix boundary set checks "when the task is not finished, does the model choose to keep verifying rather than declare completion"; equally important is the retention set — when the task really is finished, the model should declare completion normally. Watching only the first metric trains the model into an over-corrected state that never dares to finish: every task verifies forever, and latency and cost collapse. This is the parameter-level version of Chapter 7's principle that "a change must not break existing behavior"; evaluation should also spot-check general capability to confirm the LoRA patch has not damaged anything else.

**Experiment 8-17 ★★: From a "Premature Completion" Bad Case to a DPO Fix**
- Goal: run the complete chain from a production bad case to a parameter update — failure attribution -> trajectory-prefix regression task -> DPO preference pairs -> LoRA training of a 7B model -> dual validation on a boundary set and a retention set.
- Data construction: the companion repository provides **24 realistic premature-completion bad cases** covering four failure types (claiming completion without running tests, completing only part of a multi-goal request, unmet acceptance conditions, and giving up after errors by declaring the task impossible, including nastier reward-hacking variants such as deleting the failing test), plus a held-out evaluation set strictly isolated from the training data (12 boundary cases + 8 retention cases).
- This is a teaching experiment. In production, the preference pairs must cover more task families, the retention set must cover more "normal wrap-up" scenarios, and you must watch for new forms of reward hacking: the model may learn to say it verified without actually verifying. That is precisely why the end-to-end dataset's reward must rely on hidden tests the model cannot write, rather than on the model's own claims.

### 8.14.2 Case 2: Chinese quotation marks
- A user reports that "straight quotes in Chinese articles should be normalized to curly quotes." That sentence describes an expectation but gives no directly trainable rule: the same quotation mark plays completely different roles in Chinese prose, quoted English, Markdown inline code, code blocks, code comments, JSON, and paths.
- The correct fix is a **scope-sensitive minimal edit**: quotations in Chinese prose may be converted to curly quotes, with nested quotations following Chinese punctuation rules; quoted English, executable code, JSON/schemas, paths, identifiers, and anything inside Markdown backticks must be preserved verbatim; and when the scope cannot be determined, the original text should be left alone.
- **Constructing the training data.** Write the quotation rules as a **Skill**. Positive examples cover Chinese paragraphs, nested quotations, and Chinese prose inside code comments; negative examples cover quoted English, string and character literals, JSON, paths, inline code, and whole code blocks. What this teaches the model is "determine the scope first, then make the minimal edit," not "replace every straight quote you see."

**Experiment 8-18 ★★: Scope-Sensitive Chinese Curly-Quote SFT**
- Goal: verify whether LoRA SFT can make the model accurately "curl the quotes that should be curled and leave protected quotes untouched" in documents mixing Chinese, English, Markdown, code, and JSON, and hold that boundary on unseen context combinations.
- Setup: **Qwen/Qwen3-8B** as the base, trained with bf16 LoRA for 2 epochs (256 updates). The scope rules in SKILL.md serve simultaneously as the label-generation spec, the quality gate, and the regression specification; the model is only responsible for choosing the scope and producing the minimal edit, and the production-side parser and syntax checks are not removed.
- Data construction: **1,024 training samples, 256 held-out samples, and 256 boundary samples** rendered across 16 fragment categories, 10 article genres, and 9 programming languages. Samples store source and target text in pairs; Chinese prose and Chinese code comments provide the positive examples needing conversion, while quoted English, string literals, JSON, paths, inline code, code blocks, and nested structures provide the negative examples that must be protected.

### 8.14.3 Case 3: Frequent file-edit failures
- As described in Chapter 5, Coding Agents commonly use a tool like `edit_file(path, old_string, new_string)`: the model transcribes the old_string it wants replaced into the tool arguments. Edit tools usually match by exact string, so a single difference in a space, a newline, a backslash, a Unicode combining character, or a low-frequency token returns a failure.
- **From bad case to attribution.** Compare failed trajectories layer by layer along this chain: original file bytes -> tool return -> Harness serialization -> model context -> model token output -> decoded string -> JSON/tool-call parsing -> tool matching.
  - If the file read or tool return already altered the bytes -> attribute to the tool.
  - If serialization, escaping, or prompt assembly changed the content -> attribute to the Harness.
  - If encoding then decoding with the tokenizer changes it -> attribute to the tokenizer.
  - Only when the context the model received matches the original string exactly and the model's output is the first place in the chain where a difference appears can it be labeled a **model precise-copying problem** and become a post-training candidate.
- **Constructing the training data.** Abstract the copying task into three verifiable tasks: (1) verbatim restatement; (2) selecting the exactly identical target among several similar strings of equal length; (3) transcribing a given string in full into the old_string JSON argument of a tool call. Samples deliberately include the spaces, real newlines, backslashes, and Unicode characters that most often corrupt real edits.

**Experiment 8-19 ★★: Exact-Copy SFT for Special Strings**
- Goal: given that the difference has been confirmed to come from the model's transcription error, test whether LoRA SFT improves the model's exact transcription of random strings, and use an independent tokenizer audit to rule out artifacts caused by tokenization.
- Setup: **Qwen/Qwen3-8B** as the base, trained with bf16 LoRA for 2 epochs. The training script supplies token-level supervision only on the target string or the old_string JSON field.
- Results: **byte-exact accuracy on the model's held-out set rose from the base model's 37.5% to 78.9%**, with 80.1% on an independent boundary set; the mean position of the first diverging byte was 54.0 and 54.2 respectively. Separately, **512 probes** drawn from the held-out and boundary sets were used to compare three open-source tokenizers, and the lossless round-trip rate for both Qwen3 and Qwen2.5 was 80.1%. The 80.1% therefore reflects both the model's copying ability and the tokenizer ceiling.

## 8.15 Post-Training Practical Takeaways
- This chapter has come a long way from pre-training's "predict the next token": Mid-training fills knowledge and foundational capability gaps on the target distribution; SFT learns formats and protocols efficiently; outcome-oriented RL improved OOD generalization in this chapter's controlled experiments. Multi-turn tasks introduce the credit-assignment problem, reward design extends from outcome rewards to path signals that "reward the outcome and constrain the process," and tool use brings combinatorial explosion.
- A single thread runs through all of it — what the model learns depends on what the training signal taught it, and the quality of that signal is determined mainly by the **data and the environment, not by the algorithm**.
- **Common pitfalls** (recognizing them saves more wasted resources than mastering technical details):
  1. **Stuffing a knowledge base into SFT, or handing all knowledge to parameters** — large bodies of stable domain knowledge and foundational capabilities can be written into parameters with Mid-training, after which SFT teaches the model how to access and express them. Facts needing updates, citations, access control, or deletion belong in RAG.
  2. **Introducing RL before the format is stable** — if the model cannot reliably produce the JSON the reward computation needs, the training signal becomes sparse or distorted. The acceptable parse-failure rate depends on the task and reward design; no fixed threshold is universal. Set a format-stability bar with a small-scale evaluation first, and stabilize output with SFT or constrained decoding before applying RL if needed.
  3. **Treating a nominal context window as an effective one** — allowing 128K input through positional encoding does not mean the model can still retrieve, reason, and plan at 128K. Complete the current-length capability gates before expanding, retain short data and earlier-stage replay at every stage, and check degradation with a capability x length matrix.
  4. **Applying RL while pass@k is still near zero** — all-failure rollouts contain no positive trajectory, and GRPO also loses within-group advantage. First use Mid-training to add capability, SFT or distillation to widen effective support, or a reachable curriculum and partial rewards aligned with the final goal.
  5. **Poorly designed reward functions leading to reward hacking** — the model learns to exploit loopholes in the reward for a high score instead of actually completing the task. Evaluate the final goal, not an intermediate proxy.
  6. **Ignoring simulation fidelity** — if the simulation is too simplistic or the environment's responses are unrealistic, the resulting policy fails in real scenarios. Building a high-fidelity simulation can cost more than the training itself.
  7. **Over-training that degrades generalization** — falling training loss with worsening validation means the model is memorizing details. Mid-training can forget general capabilities, SFT can overfit demonstrations, and RL can overfit the current reward and task distribution; all three require independent retention sets and early stopping.
  8. **Value-function collapse and insufficient exploration** — inaccurate value estimates in PPO bias the advantage computation, showing up as violently oscillating training curves. Too low a temperature or too little randomness traps the Agent in a local optimum.
  9. **Treating training-inference numerical mismatch as harmless noise** — if the sampler/trainer probability ratio already differs from 1 before an update, nominal on-policy training has silently become off-policy. Monitor log-probability differences, approximate KL, clipping fraction, and policy staleness.
  10. **Underestimating RL's compute cost** — a task that works well with SFT may need 10-100 times the training time under RL. If the test distribution closely matches training, SFT may already be enough.
  11. **Low-quality training data** — Mid-training absorbs incorrect associations from the corpus, SFT learns demonstration noise directly, and a systematically biased RL reward amplifies the policy in the wrong direction.
- **Core principle**: validate the key assumptions with small-scale experiments before committing large-scale resources — a small Mid-training corpus to inspect knowledge, capability, and forgetting curves; a small SFT set to test format stability; a small rollout batch to inspect pass@k, reward variation, and sampler/trainer numerical agreement. Failing fast is more acceptable than failing at scale.
- **Synergy with RAG and ICL (in-context learning)**: the three are not mutually exclusive alternatives but act in different places.
  - **ICL**: uses examples, rules, and current state for zero-parameter, immediate adaptation, though latency and cost rise as context grows.
  - **RAG**: puts facts and evidence in external knowledge that can be updated dynamically and traced.
  - **Post-training**: writes high-dimensional perception, generation style, and implicit decision policies into parameters.
  - The choice depends not only on whether the task is stable over the long term but, more importantly, on whether the capability can be adequately expressed in external symbols. Capabilities such as medical image recognition or a natural tone of voice often still require parameter updates even in a continuously changing domain; conversely, a long-stable transfer-approval rule should be guaranteed deterministically by code rather than left to the model's memory.
- Robust systems generally combine these methods: manage dynamic facts and evidence with RAG, experiment quickly with language-describable strategies via ICL, encode deterministic processes and hard constraints in program code, absorb stable domain knowledge and foundational capabilities with Mid-training, and shape behavior that external rules cannot fully express with SFT and RL. Distillation can also transfer the behavior of a capable large model into a cheaper small one.

## 8.16 Chapter Summary
- Mid-training, SFT, and RL are not interchangeable strengths of "fine-tuning"; they address the **foundation, protocol, and policy**, respectively.
- Mid-training should also turn a nominal context extension into an effective context that retains short-range capabilities through a **length curriculum, mixed data, and staged gates**.
- If pass@k remains near zero under reasonable sampling, use **Mid-training** to add knowledge and capability. If the model occasionally succeeds but produces unparseable output, use **SFT** to stabilize the format. Only when the current policy generates scoreable trajectories with reward variation can **RL** efficiently reallocate probability and explore strategies.
- "SFT memorizes, RL generalizes" summarizes a tendency observed in this chapter's controlled experiments, not a law independent of the data, model, reward, and environment.
- **Two further judgments** run through the whole chapter and are worth remembering more than any algorithm:
  - **Data and environment matter more than algorithms**: the Mid-training corpus determines what gaps are repaired in the foundation; SFT demonstrations determine whether the protocol is stable; the environment and reward determine what RL can explore and reinforce. When a real environment cannot be built, using a model to simulate it is viable, but the simulator's bias remains the ceiling on training. In many scenarios, once the foundation and demonstration data are good enough, RL is unnecessary.
  - **RL's main bottlenecks today are sample efficiency and distribution consistency**. On-Policy Distillation expands one rollout's terminal scalar into token-level supervision on states the student actually visits, while RLVP turns wasted environment feedback into a learnable signal. Truly on-policy rollouts also reduce the bias and variance of importance correction. Training-inference numerical mismatch breaks that premise, so sampler/trainer consistency deserves the same attention as the reward curve.
- This chapter answers how updating parameters can enable continuous Agent evolution. In the next chapter, parameters are only one of four carriers of Agent self-evolution: **knowledge, instructions, programs, and parameters**.

## Deep Thinking

## Thought Questions
1. ★★ **Catastrophic forgetting** — where fine-tuning for a specific task destroys the model's original general capabilities, such as general tool calling — is particularly troublesome in Agent scenarios. Compared with full-parameter fine-tuning, LoRA freezes the base weights and carries a lower risk of forgetting, but it is not immune. What strategies can further mitigate capability forgetting during fine-tuning?
2. ★★ Post-training solidifies capabilities into model weights, or "muscle memory," while in-context learning places knowledge in the input at inference time. Some capabilities, such as domain knowledge, can be learned through post-training or supplied through few-shot examples. What criteria would you use to decide which path a capability should take?
3. ★★ Model distillation allows a small model to learn the behavior of a large model. By capability level, the models being distilled can be divided roughly into three tiers — Chat models (single-turn dialogue and direct answers), Reasoning models (long chains of thought before answering), and Agentic models (multi-turn tool calls and interaction with the environment). What different challenges arise in distilling each type? (Hint: Begin with "what exactly is being distilled" — the style of the output, the complete reasoning trajectory, or the policy for interacting with the environment; which tokens in the trajectory should be learned and which environmental returns should not; and how delayed and sparse the success/failure signals are.)
4. ★★★ In multi-turn Agent interactions, the credit-assignment problem is more severe than in single-turn scenarios — a final success or failure is difficult to attribute to a decision made in turn 3 rather than turn 7. How would you design a reward-allocation strategy?
5. ★★★ If you had a fixed budget, such as $10,000, to improve a customer-service Agent, how would you allocate it among context and knowledge, Prompt/Skills, programmatic constraints, and parameter training? What factors would determine your decision?
6. ★★★ Autonomous model learning under scarce samples and without a clear reward function is regarded by some as the ultimate goal of post-training. How far are current RL training methods from this goal? Where is the next breakthrough most likely to come from?
7. ★★ This chapter notes that LoRA fine-tuning is not expensive. Could a dedicated LoRA therefore be trained for every user or client company, writing user memory or enterprise knowledge into parameters rather than storing it in an external knowledge base as in Chapter 3? When would "writing memory into parameters" have an advantage over "storing memory in a knowledge base," and when would it be counterproductive?
8. ★★★ On-Policy Distillation relies on a stronger teacher model to supervise the student. OpenAI's Weak-to-Strong Generalization research, however, offered a counterintuitive finding: supervision from a weak model can sometimes unlock capabilities latent but inactive in a stronger model. If applied to Agent training, could this enable reverse distillation in which "a small model teaches a large model"?
9. ★★ A Process Reward Model (PRM) evaluates each reasoning step, whereas an Outcome Reward Model (ORM) considers only the final result. Which deserves more reward: "a correct process that leads to a wrong result," or "a wrong process that happens to produce the correct result"? How would you balance the two in multi-step Agent tool-calling scenarios?
10. ★★★ The evaluation datasets discussed in this chapter, such as SWE-Bench Verified, tau-squared-bench, and AndroidWorld, can be used both for evaluation and post-training. But once an evaluation set is used for training, it is no longer independent. Does this violate the fundamental principle that training and test sets must remain separate? Dynamic parameter generation in tau-squared-bench and parameterized templates in AndroidWorld mitigate the problem to some extent, but their template structures remain fixed. How can the training value of evaluation data be fully exploited while preserving evaluation independence?
11. ★★★ For a target task, the base model has a very low pass@1. How would you combine pass@k, parse success, partial-progress rate, and failure attribution to decide whether to start with Mid-training or SFT, or move directly to RL? What conditions should these metrics satisfy before switching stages?
12. ★★★ ReTool's training dynamics show (see Experiment 8-14) that a few extremely long responses can significantly extend the entire training cycle — most rollouts in a batch have already been generated, but the system must wait for the longest responses to finish, leaving cluster GPU utilization low. How can resource utilization be improved in training clusters under such long-tail response conditions?
13. ★★★ When training an Agent against LLM-simulated environments — such as a simulated search engine or simulated users — the target of the Agent's exploitation shifts from "the rules of the real environment" to "the biases and loopholes of the simulator itself." What concrete reward hacking behaviors can arise in this kind of training, and how should they be prevented?

