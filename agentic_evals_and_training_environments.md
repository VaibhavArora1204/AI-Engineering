# Training and Evaluating AI Agents — The Complete Playbook

This covers the full Hugging Face "Training Agents" series (4 sessions), the RL for Agents Workshop (5 talks + panel), and the Agentic Evaluations Workshop (4 talks). Seven videos, rearranged into the order you should actually learn this. Everything traces back to what was said. Nothing invented.

The progression: SFT teaches your model to imitate → Distillation teaches it to learn from a better model's judgment → RL (GRPO) teaches it to discover strategies on its own → RL Environments give it a real world to act in → the RL Workshop shows you the broader ecosystem → the Evals Workshop shows you how to measure whether any of this worked.

---

# Session 1 — SFT on Agent Traces: Teaching a Model to Imitate

**Speakers: Ben (Hugging Face) and Sergio. "Training Agents: Live tutorial on how to fine-tune a coding agent for continual learning."**

## What SFT Actually Is

SFT — supervised fine-tuning — is a continuation of pre-training. In pre-training, the model predicts the next token in arbitrary text scraped from the internet: "the cat sat on the ___" → "mat." SFT does the same thing but on instruction-completion pairs instead of raw text. "What is the capital of France?" → "Paris." Instead of completing a sentence, the model learns to *respond* to a prompt.

For agents specifically, SFT teaches the model to use tools. The training data contains traces where an agent made tool calls — bash commands, file edits, API calls — structured inside a chat template. The model learns how to format those tool calls and when to use them. This is what makes a base model start acting like an agent instead of a text generator.

## The Data Format: Traces, Masks, and Why Masking Is the Core of SFT

A trace is a recorded conversation: user prompt in, model completion out. In the case of agent training, these are real agent sessions — someone used a coding agent like Pi with Claude Opus 4.5 to solve tasks, and the full sequence of prompts, tool calls, and responses was recorded.

The critical mechanic: you don't want the model to learn to generate the *user's* messages. You want it to learn the *assistant's* completions. So you mask the user portions.

Here's how it works concretely. You have a trace:

```
User: "Be concise"
Assistant: "Concise"
```

When you tokenize this, you get a sequence of token IDs. You create a labels array that's a copy of those IDs, but you set every token that belongs to the user's portion to `-100`. That `-100` is a convention — it tells PyTorch's cross-entropy loss to ignore those positions. The model only computes loss on the assistant's tokens.

In practice, Transformers' `apply_chat_template` method handles the formatting. You pass in the conversation in the standard messages format (`[{"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]`), and it applies the model's chat template. Then you tokenize it, copy the input IDs to labels, and mask out everything that isn't from the assistant.

```python
# Pseudocode — the educational version without TRL abstraction
encoding = tokenizer.apply_chat_template(messages, return_tensors="pt")
labels = encoding.input_ids.clone()
# Mask user tokens with -100
labels[user_token_positions] = -100

# Forward pass
outputs = model(encoding.input_ids)
logits = outputs.logits

# Shift and align (standard next-token prediction)
shift_logits = logits[..., :-1, :].contiguous()
shift_labels = labels[..., 1:].contiguous()

# Cross-entropy loss (ignores -100 positions)
loss = F.cross_entropy(shift_logits.view(-1, vocab_size), shift_labels.view(-1))
loss.backward()
optimizer.step()
```

This is the same next-token-prediction loop as pre-training. The only difference is the masking. That's what makes SFT a fine-tuning approach rather than pre-training from scratch. The model already knows language — you're just steering it toward generating responses in a specific format.

## What You Can and Cannot Expect from SFT

SFT is emulation. You're teaching one model to copy the behavior of whatever produced the training traces. If those traces came from Claude Opus 4.5 solving coding tasks, your small Gemma model learns to emulate Opus's behavior on those specific tasks.

**The emulation ceiling**: your trained model cannot surpass the quality of the traces. If Opus made a mistake in a trace and the fix wasn't recorded, your model learns the mistake too. If Opus was only mediocre at a certain task category, your model will be mediocre there at best.

**What SFT is good for**: getting a small model to learn the *format* and *structure* of agentic behavior. Chat templates, tool call syntax, multi-turn interaction patterns, when to call which tool. Think of it as bootstrapping — you're not trying to make the model smarter than its teacher. You're making it competent enough to operate in the harness.

**The realistic goal**: take a large model's traces on a narrow use case (say, PR triage), train a small model (Gemma 4 2B) on those traces, and get a small model that handles that narrow use case well. It won't be generally better. Performance might dip on unrelated tasks. But on *your* task, it'll be better and far cheaper to run.

## The Practical Setup

- **Model**: Small model (Gemma 4 2B in the demo)
- **Data**: Traces from real coding agent sessions (Pi + Claude Opus 4.5). Available on HF Hub as a dataset.
- **Framework**: TRL's SFT Trainer (in production). The educational version uses raw PyTorch + Transformers to show what's happening under the hood.
- **Compute**: HF Jobs for GPU access. The agent orchestrating the training runs on a laptop — it dispatches training jobs to Hugging Face Hub for the actual GPU work.

## Skills — Guiding the Agent That Does Your Training

The repo includes "skills" — instruction files that tell the agent how to:
- Use TRL correctly
- Use HF Jobs to get GPU compute
- Use Tracelo for observability
- Structure training scripts

Skills are guardrails. They encode patterns that work and mistakes to avoid. The agent reads them during execution. As your experiments mature, update the skills — ideally get the agent to update them based on what it learned. You can also record common mistakes so the agent stops making them.

## Tracking Metrics with Tracelo

Tracelo is a free observability tool. Deploy on HF Spaces or run locally, no account needed.

SFT metrics to watch:

- **Loss** — should decrease. The gap between what the model generates and the training target is shrinking.
- **Entropy** — should decrease. The model's predictions across its vocabulary are getting less random, more confident.
- **Token accuracy** — should increase. Correlates with entropy decrease when things are going right.
- **Learning rate** — should follow a schedule (high early, decreasing). If it's too low from the start, training will be sluggish.
- **Eval metrics on holdout set** — should track training metrics roughly. If they diverge, you're overfitting.

## Evals During SFT Training

Incorporate both task-specific and general benchmarks:

- **Task-specific**: your target task. Are scores going up?
- **General** (HumanEval, MBPP): is the model still generally capable? You don't want to catastrophically degrade base abilities while improving on your narrow task.

The combination tells you: am I improving where I care, without destroying what I started with?

## What Comes After SFT

SFT gives you a model that can operate in a harness and imitate good behavior. But it can't surpass its teacher. For that, you need distillation (break the ceiling with a teacher's judgment) or RL (discover novel strategies with no teacher at all). SFT is almost always step one in a training pipeline — the bootstrap that makes everything else work.

---

# Session 2 — Distillation: Learning from a Teacher's Judgment

**Speakers: Ben (Hugging Face) and Sergio. "Training Agents 2: Live tutorial on model distillation for training custom agents."**

## Where SFT Breaks Down

In SFT, you copied traces. The model learned to reproduce what the teacher did. But the teacher's traces might not be relevant to what the student model actually does during inference. If the student is weaker than the teacher, it'll take completely different actions — it might drive straight into the wall while the teacher elegantly navigated the warehouse. The teacher's trace is too far from the student's reality to be useful.

Also, SFT gives you a static dataset. The model trains on the same traces forever. It never gets feedback on *its own* attempts. That's the fundamental limitation.

## What Distillation Changes

Distillation introduces a teacher model that *judges the student's own behavior.* Instead of "copy what the teacher did," it's "do the task yourself, and let the teacher tell you how close you are."

The mechanism is KL divergence — it measures how different two probability distributions are. The student generates tokens. The teacher generates tokens for the same input. You compare their probability distributions at each token position and use that difference as the training signal.

This is a *dense* signal — you get information at every single token position in the sequence, not just a single pass/fail at the end. That's a lot more information per training step than RL will give you later.

## Off-Policy vs. On-Policy Distillation

### Off-Policy (Offline)

The teacher generates completions ahead of time. You store them. Then you train the student to match the teacher's token distributions from that stored dataset.

This is basically SFT with extra information. Instead of just having the teacher's text (which tokens were chosen), you also have the teacher's *logits* (the probability the teacher assigned to every token in its vocabulary at each position). That's dramatically more information.

The limitation: the teacher's trajectories might not represent what the student would actually do. The gap between teacher and student behavior means the signal can be less relevant.

### On-Policy (Online)

The *student* generates completions. Then the *teacher* evaluates those completions by providing its own logit distributions for the same sequence. You compare student logits to teacher logits and update the student.

This is more powerful because the training signal is always about what the student is actually doing. The teacher is judging the student's own work, not showing its own. The student gets feedback on the trajectories it would actually take during inference.

The cost: you need to run the teacher during training (inference calls for every batch), which is more expensive than having a pre-computed dataset.

## The Two Key Parameters: Lambda and Beta

TRL's distillation trainer exposes two critical parameters:

### Lambda (policy-ness: 0 = off-policy, 1 = on-policy)

- `lambda=0`: Fully off-policy. Train on pre-generated teacher completions. Cheap, but the data might not match what the student actually does.
- `lambda=1`: Fully on-policy. Student generates, teacher judges. More relevant signal, more expensive.
- Values between 0 and 1: mix off-policy and on-policy data. Useful when you want some exposure to what the teacher does (off-policy) plus feedback on the student's own behavior (on-policy).

### Beta (KL direction: 0 = forward KL, 1 = reverse KL)

This one matters more than it looks.

- **Forward KL (beta=0)**: The student tries to cover *all* the modes of the teacher's distribution. It becomes mean-seeking — it tries to be reasonably good everywhere the teacher is good. The risk: it spreads itself too thin and doesn't excel anywhere.

- **Reverse KL (beta=1)**: The student picks the teacher's *best* modes and focuses on those. It becomes mode-seeking — it gets really good at the teacher's strongest behaviors and ignores the rest. The risk: it misses important parts of the distribution.

The practical starting point: **lambda=1, beta=1** (on-policy, reverse KL). This is the strongest signal — student generates, teacher judges, student focuses on the teacher's best strategies. Start here, experiment from here.

## Self-Distillation

Self-distillation removes the separate teacher entirely. The model teaches itself.

The basic idea: give the model a *hint* — extra information about the task that it wouldn't normally have. Let it solve the task with the hint (it performs better because it has extra information). Then train it to achieve the same performance *without* the hint.

Variants:

- **Hint-based**: The hint is extra context. Maybe you give the model the answer format, or a worked example, or partial solution. It generates a high-quality completion with the hint. Then you compare that to what it would do without the hint, and use the difference to train.
- **On-policy self-play**: The model generates multiple attempts. It ranks them (or a verifier ranks them). It learns from its own best attempts.

Self-distillation is useful when you don't have a better teacher model available, or when you want to improve within a specific domain where no teacher exists.

## The Practical Experiment

The demo setup:
- **Student**: Gemma 0.6B (smallest version)
- **Teacher**: Gemma 4B (4 billion parameters)
- **Data**: Coding traces from a Pi session
- **Method**: On-policy distillation (lambda=1, beta=1)
- **Experiment**: Three learning rate sweeps logged to Tracelo

The agent orchestrates the whole thing — it reads skills, sets up the TRL script, dispatches three HF Jobs (one per learning rate), and logs results to Tracelo. You watch the eval loss curves to pick the best learning rate.

## When to Use Distillation vs. SFT

- **Use SFT when**: you have high-quality traces and the student model is close enough in capability to learn from them directly. Good for format learning, tool-use patterns, and narrow task specialization.
- **Use distillation when**: the gap between teacher and student is large (teacher's traces are too far from what the student would actually do), or you want the student to learn more nuanced behavior than what can be captured in a static trace.
- **Use on-policy distillation when**: you can afford the compute to run the teacher during training and you want the strongest signal.

## Multi-Teacher Distillation

Frontier models now use multiple teachers. You train in stages — one teacher for STEM, another for code, another for instruction following. Then you combine checkpoints. TRL doesn't directly support multi-teacher in a single trainer call, but you can chain stages: distill from teacher A, checkpoint, distill from teacher B, checkpoint, merge.

The Nematron 3 paper has a good visualization of how checkpoints with different capability peaks move through the pipeline and get combined.

## Can You Distill from Closed Models?

Only off-policy. You can't get logits from closed-model APIs (OpenAI, Anthropic). So you can't do on-policy distillation from them. What you *can* do is generate synthetic data from closed models, then use that data for SFT or DPO. Papers like DEITA and WizardLM show pipelines for this — use closed models to generate high-quality samples, use other models to judge quality, train on the filtered dataset.

## What Comes After Distillation

Distillation still has a ceiling — the teacher's judgment ability. If the teacher can't tell good from great, the student can't learn the difference either. To break through this ceiling entirely, you need reinforcement learning — where the signal comes from the task itself (did the code compile? did the tests pass?) rather than from any model's judgment.

---

# Session 3 — Reinforcement Learning with GRPO: Learning from the Game Itself

**Speakers: Ben (Hugging Face) and Sergio. "Training Agents 3: Reinforcement Learning."**

## The Signal Gets Sparse

Here's the progression so far in terms of what signal you use to update model weights:

- **SFT**: Dense signal at every token. You have a target token and you compute cross-entropy loss against it. Lots of information per training step. Ceiling: the quality of the training traces.
- **Distillation**: Dense signal at every token. You have the teacher's full probability distribution at each position. Even more information per step than SFT. Ceiling: the teacher's judgment ability.
- **RL (GRPO)**: Sparse signal. One number — the reward — at the *end* of the entire trajectory. Did the agent solve the task or not? That's it. No ceiling from any teacher. The model can discover strategies no teacher ever demonstrated.

The chess analogy makes this concrete: SFT is studying grandmaster games from books — you'll never be better than the books. Distillation is having a coach watch you play and give feedback move-by-move — you're limited by the coach's skill. RL is just going to tournaments, playing games, winning some, losing most. No books, no coach. You learn entirely from the outcomes. Slow, but the only method with no ceiling.

## The GRPO Loop

GRPO — Group Relative Policy Optimization — is the RL algorithm you're using. Here's the loop:

1. **Sample a prompt** (task) from your dataset.
2. **Generate a group of completions** (rollouts/trajectories) — the model attempts the task multiple times.
3. **Score each completion** with reward functions — did the format match? Did the code compile? Did the tests pass?
4. **Calculate advantages** — each completion's reward relative to the group average. Completions better than average get positive advantage, worse get negative.
5. **Update the policy** — increase the probability of actions that led to above-average outcomes, decrease the probability of actions that led to below-average outcomes.
6. **Repeat.**

The "group relative" part is what matters. You're not comparing against some absolute standard. You're comparing each attempt against the other attempts in the same group. This is what creates the learning signal from a sparse reward.

### TRL GRPO Parameter Reference

Ben walked through every `GRPOConfig` parameter mapped to where it acts in the loop. Here's the reference:

| Parameter | Loop Stage | What It Controls |
|-----------|-----------|------------------|
| `temperature` | Generation | How random/diverse each completion is. Higher = more varied rollouts. |
| `num_generations` | Generation | Group size — how many completions per prompt. More = higher chance of variation, more compute. |
| `max_completion_length` | Generation | Maximum tokens per completion. Some completions will be shorter depending on what the model does. |
| `use_vllm` | Generation | Whether to use vLLM as the inference engine for faster generation. Usually yes. |
| `reward_weights` | Scoring | Scaling weights for each reward function. E.g., you might scale format reward lower than accuracy reward to prevent overfitting to format. |
| `epsilon` / `epsilon_high` | Loss | Clipping — how much the policy can change per update step. Bounds the advantage. |
| `loss_type` | Loss | Usually cross-entropy. Defines how the token-level loss is calculated from the advantages. |
| `beta` | Policy update | KL penalty — distance from the reference policy (model at start of training). 0 to ~0.1. |
| `learning_rate` | Policy update | How much weights change per step. Standard LR scheduling applies. |
| `num_iterations` | Outer loop | How many times to repeat the full generation-scoring-update cycle. |
| `steps_per_generation` | Update frequency | How many gradient steps to take per batch of generations. Controls how "online" the training is. |

**Note on reward signal density**: GRPO gives you one reward at the end of the trajectory by default. You *can* turn this into a process reward (reward at intermediate steps), but the GRPO algorithm itself doesn't natively produce dense signals the way SFT and distillation do. Some tasks support per-tool-call rewards (did this specific action help?), and if yours does, use it — it gives more signal. But many tasks only have a natural reward at completion (did the code pass tests or not?), and GRPO handles that fine through reward stacking and variation.

## The Variation Requirement — If You Miss This, Nothing Works

This is the single most important thing to understand about GRPO.

The advantage signal is *relative to the group.* If every completion in the group gets the same reward — all succeed or all fail — the advantage is zero. There is literally nothing to learn.

Think about it: if four attempts all fail, GRPO can't distinguish "this action was slightly better" from "this action was slightly worse." They all got zero reward. No gradient. If four attempts all succeed, same problem — they all got reward 1, no relative signal.

**You need variation in the group.** Some completions must succeed, some must fail, for there to be any learning signal.

The chess version: playing only grandmasters (lose every game, no signal) or only beginners (win every game, no signal). You need opponents near your level where you win some and lose some.

### Knobs to Maintain Variation

1. **Task difficulty** — if every task is too hard, every attempt fails. If every task is too easy, every attempt succeeds. You need a spread of difficulty. This is the most practical knob.

2. **Reward stacking** — stack rewards from easy to hard. Even if the main task (pass all tests) is too hard for the model right now, easier rewards (correct format, code compiles) still create variation within the group:
   - `format_reward`: Does the output have the correct structure? (regex, tool call format)
   - `compile_reward`: Does the generated code compile?
   - `test_reward`: Does it pass the test suite?
   - `quality_reward`: Among correct solutions, prefer fewer lines of code.

   Each layer provides signal even when harder layers fail. This is what creates variation GRPO can learn from.

3. **Group size** — larger groups (more completions per prompt) increase the probability of getting variation. But they cost more compute.

4. **Temperature** — higher temperature means more diverse completions. Increases variation but also increases noise.

The **"aha moment"** from the DeepSeek papers: there's a point in training where the model finally gets enough variation that learning takes off. Before that, it's struggling to find any signal. After it, progress accelerates.

## Guard Rails: Clipping and KL Penalty

Two mechanisms stop the model from going off the rails during RL:

### Clipping (Epsilon)

Epsilon defines how much the policy can change in a single update step. It clips the advantage — even if a particular trajectory was massively better than the group average, the update is bounded. This prevents the model from making huge jumps based on a single lucky trajectory.

TRL exposes `epsilon_high` and `epsilon_low` for asymmetric clipping.

### KL Penalty (Beta)

Beta controls how far the policy can drift from its initial state (the model at the start of training, usually after SFT).

Think of it as a landscape: the model starts at a position in "behavior space" defined by its SFT training. The KL penalty makes it increasingly expensive to move away from that starting position. A chess move the model learned during SFT is cheap. A completely novel action the model never saw in SFT is expensive.

Beta is typically small (0 to 0.1). Higher beta = more conservative (model stays closer to SFT behavior). Lower beta = more exploratory (model can try novel strategies, but also more prone to collapse).

**KL penalty vs. reward hacking**: KL penalty does help prevent reward hacking because hacks are typically novel behaviors the model never saw in SFT. But it's not a complete solution — you're using the same parameter to control exploration generally. The real defense against reward hacking is a well-designed reward function, not the KL penalty.

## The Reward Contract — Where You'll Actually Spend Your Time

The reward function is the coldface of RL training. This is where you iterate: define functions, do dry runs, smoke tests, evaluate. You'll spend more time on reward functions than on any other part of the setup.

### Designing Reward Functions

For a coding agent, a stacked reward might look like:

```python
def reward_fn(completion):
    score = 0.0
    # Format: is the output structured as a valid tool call?
    if passes_format_check(completion):
        score += 0.2
    # Compilation: does the generated code compile?
    if compiles(completion):
        score += 0.3
    # Tests: does it pass the test suite?
    test_ratio = run_tests(completion)  # fraction of tests passed
    score += 0.5 * test_ratio
    return score
```

The stacking gives partial credit. A completion with correct format and compilable code gets 0.5 even if no tests pass. That's still useful signal for GRPO.

### Preventing Reward Hacking

If your format reward uses an open-ended regex, the model will find a string that matches the regex while being completely useless. If the reward checks for a keyword in the output, the model will learn to just output that keyword.

Practical defenses:
- **Tight verifiers** — make reward functions precise with no exploitable ambiguity.
- **Separate observation metrics** — track metrics *not used for training*. If training reward goes up but tool diversity drops, entropy collapses, or trajectories become repetitive, you've got a hack.
- **Look at the data** — have a dashboard showing live trajectories. If behavior looks too uniform but reward is climbing, something's wrong.
- **Eval reward vs. train reward** — if training reward hits ceiling but eval reward doesn't follow, your reward function has a bug or is being gamed. Sergio's experiment showed exactly this pattern.

## Interpreting Training Curves

What healthy curves look like:
- **Format reward**: rises quickly early. This is easy for the model.
- **Accuracy/completion reward**: starts slower, still trends up.
- **Entropy**: roughly flat, maybe slowly declining. Not diving (collapse) or rising (chaos).
- **Completion length**: stable. Suspicious if it spikes up (potential rambling/hacking) or crashes.
- **Reward spread**: varied. Best-fit line roughly flat but with healthy deviation. The model is still exploring.

What a stalled run looks like:
- No reward signal from any reward function.
- No reward spread.
- The task is too hard — the model never achieves any reward. You need to adjust difficulty or add easier reward layers.

What a collapsing run looks like:
- Entropy falls (model fixating on one strategy).
- Completion length spikes (potential hacking or rambling).
- One reward goes up while others stagnate (hacked that specific reward).
- KL divergence rises sharply (model has deviated far from starting policy).

## Practical Experiment Workflow

1. **Dry run the reward function** — test it on some text without training. Does it give sensible scores? Does it have exploitable edge cases?
2. **Smoke test with 5 steps** — run 5 training steps with small group size (4 generations). You want to see *some* reward variation. If zero variation in 20 completions (5 steps × 4 generations), the task might be too hard or too easy.
3. **Short training run** — 50-200 steps. Watch the curves. Iterate on reward functions and hyperparameters.
4. **Full run** — once curves look healthy, scale up.

Dataset size: a few thousand samples is enough to get meaningful signal. Papers go to hundreds of thousands, but you can change model capabilities with a few thousand. Sergio's experiments used about 200 steps.

## Comparison to Other RL Algorithms

- **DPO (Direct Preference Optimization)**: offline. Uses pre-collected chosen/rejected pairs. Simpler but limited by the quality of the offline data. If the chosen samples are outdated or don't represent current model capability, DPO can make the model worse.
- **Online DPO**: same concept but with online samples. More relevant signal.
- **PPO (Proximal Policy Optimization)**: the original RLHF algorithm (InstructGPT). More complex — adds a separate reward model. More expensive. Useful when you need a learned reward (for stylistic properties or things you can't express as a deterministic function). Not the first thing you'd reach for anymore.
- **GRPO**: the sweet spot for most current work. Python reward functions, no separate reward model, simpler loop. Best starting point for RL with LLMs.

---

# Session 4 — RL Environments: Giving the Agent a World to Act In

**Speakers: Ben (Hugging Face) and Sergio. "Training Agents 4: From reward functions to environments."**

## Why Environments Are the Next Step

In Session 3, the model generated a single completion and received a reward. The reward function was a plain Python function — check the format, check the answer, return a score. The model's "action" was generating text once.

But real agents don't work that way. They take sequences of actions in a changing world. A coding agent reads files, edits code, runs tests, reads error output, edits again. A customer service agent receives messages, queries databases, sends responses, receives follow-ups. The world changes with every action.

An RL environment captures this: a stateful system that the agent interacts with over multiple steps, where each action changes the world and the agent observes the new state.

## What Is an Environment?

Simple definition: the agent acts, the environment updates, the agent observes the new state. Loop until done.

In agent terms:
- **Actions** = tool calls (bash commands, file edits, API calls, code execution)
- **Environment** = world state (code files on disk, chess board position, PR branch code)
- **Observation** = current state + reward signal
- **Reward** = learning signal for GRPO

The critical property: the environment and agent are **completely independent**. The environment is just software that responds to actions. It has no concept of training. It runs exactly the same way whether the agent is being trained or deployed in production.

## Turning Real Tasks Into Environments

### Chess
- World = board
- State = piece positions
- Rules = legal moves (business logic)
- Score = game outcome → reward

### GitHub Pull Request (Repo-to-RL)

This is the practical pattern most coding agent training uses:

- **Task** = the GitHub issue (bug to fix, feature to implement)
- **State** = the code branch the agent acts on
- **Reward** = run the PR's test suite against the agent's implementation

Decompose a real PR into: issue → branch → tests. The issue is the task, the branch is the sandbox, the tests are the reward signal. You get task, implementation context, and reward from one real-world artifact.

## The Three Components of Every Environment

### 1. Task
What the agent has to do. A bug to fix, a Blender representation to generate, a chess game to win.

### 2. Runtime
The compute and software environment:
- Compute resources (CPU, GPU)
- Software packages (Python, Blender, compilers)
- Code from a specific branch
- File system state
- Everything the agent needs to physically perform the task

### 3. Grading
Verifiers, rewards, rubrics that evaluate the observation. Everything that creates the learning signal for GRPO.

### The Sandbox-Verifier Separation

**This is critical and people get it wrong**: the sandbox (where the agent works) must NOT contain anything related to verification or reward.

If the agent can access the verifiers, it will:
- Use them to understand the task (undermining the evaluation)
- Hack the reward (game the verifier code directly)

This is not theoretical. It happens consistently in RL agent training. Any verification data or task description the agent can see *will* be exploited. The sandbox and the grading system must be physically separate — different containers, different file systems, no shared access.

## The Agentic Training Loop

```
Training Framework (TRL) ←→ Agent Harness (contains LLM)
                                    ↕
                              Sandbox (safe compute, task-only, no verifiers)
                                    ↕
                              Verifier (separate, calculates reward)
                                    ↕
                              Reward → back to TRL → GRPO weight update
```

The training framework connects to the harness. The harness contains the model whose weights get updated. The harness interacts with the sandbox as if it were a real software environment. The verifier receives the sandbox state separately and computes the reward.

**The harness should have no concept of training.** No difference between training and production usage.

## White-Box vs. Black-Box Environment Connections

This is the most practical architectural decision you'll face.

### White-Box: Trainer Owns the Loop

The TRL trainer directly controls each step. It sends a prompt, gets a generation, sends it to the environment, gets back a reward, loops.

```python
# Simplified white-box connection
env = CodingEnvironment(task)
for step in range(max_steps):
    action = model.generate(observation)
    observation, reward, done = env.step(action)
    if done:
        break
# reward goes to GRPO
```

This works for simple environments — a single tool call, a single code generation. The trainer knows exactly what's happening at each step.

**Use when**: your environment is simple enough that the training loop can control each action directly.

### Black-Box: Harness Owns the Loop

Real coding harnesses (Open Code, Codex, Pi) are complex. They manage their own loops — making multiple tool calls, spawning sub-agents, compressing context, managing multi-turn conversations. You can't have the trainer step through each individual action because the harness's internal logic is opaque and multi-step.

Solution: **Capture Proxy**.

```
Agent ←→ Capture Proxy ←→ LLM
```

The capture proxy sits between the agent harness and the LLM. It speaks the same protocol as the harness (the harness doesn't know it's there). It captures every call — building the complete rollout graph with full information. This gets sent back to TRL so GRPO can update weights.

The harness owns the loop. The trainer captures rollouts through the proxy. The agent operates exactly as it would in production.

**Use when**: your environment involves a real agent harness that manages its own multi-step interaction.

## OpenM — The Environment Framework

OpenM is a gymnasium-style API for defining RL environments. Three core operations:

```python
from openm import Environment

env = Environment.from_hub("username/my-environment")

# Reset to get initial state
observation = env.reset()

# Loop: act, observe, repeat
while not done:
    action = policy(observation)
    observation, reward, done = env.step(action)
```

### What an Environment Looks Like in Code

Every environment is a self-contained Docker container running a FastAPI application. You define a class with `reset()`, `step()`, and `state()` methods:

```python
class CodingEnvironment:
    def reset(self):
        # Initialize the world state
        # Set up file system, install dependencies
        return initial_observation

    def step(self, action):
        # Execute the agent's action (e.g., run Python code)
        # Update world state
        # Calculate reward
        return observation, reward, done

    def state(self):
        # Return current world state
        return current_state
```

Tools the agent can use are just methods on this class — `run_python()`, `edit_file()`, `run_tests()`, etc.

### Deployment Options

Since it's just a Docker container:
- **HF Spaces** — lighter environments (CPU tasks like running Python)
- **HF Sandboxes (HF Jobs)** — heavier environments needing full machine isolation (coding harnesses)
- **Modal, Daytona** — third-party compute
- **Local, Kubernetes** — wherever Docker runs

~4,000 environments on HF Hub: Atari games, text games, self-driving (CARLA), email triage, coding tasks, terminal tasks.

### OpenM CLI

```bash
openm init          # Scaffold a boilerplate environment
openm generate      # Generate code for your specific task
openm import        # Import from other libraries (verifiers, open-reward)
openm push          # Publish to HF Hub
openm pull          # Pull environments from Hub
openm fork          # Fork existing environment and modify
openm discover      # Search Hub for environments by domain (coming soon)
openm validate      # Validate environment quality for training (coming soon)
```

DeepSeek's recent paper describes a pipeline: agent decides if a task is feasible → generates environment → verifies validity → validates quality → uses in training. The OpenM CLI enables the same workflow.

## Two Practical Demos

### Demo 1: White-Box — Python Problem Solving

- **Model**: Qwen 1.7B (tiny)
- **Task**: Solve basic Python problems from an HF dataset
- **Environment**: HF Space running a Python executor
- **Tool**: `run_python` — sends code to the environment, gets pass/fail
- **Connection**: White-box. GRPO generates rollouts → agent sends code → environment returns result → reward → GRPO updates
- **Result**: ~30 minutes training. Eval reward goes up. Training reward bounces (different problem difficulties), but eval reward trends upward consistently.

The key insight: a 1.7B parameter model becomes meaningfully better at Python problems with a simple RL environment and 30 minutes of training.

### Demo 2: Black-Box — Open Code Harness

- **Model**: Qwen 8B (bigger because the task is harder)
- **Harness**: Open Code sessions, each launched as HF Sandbox
- **Connection**: Black-box. Capture proxy captures rollouts. Harness owns the loop.
- **Algorithm**: DAPO (variant of GRPO, more advanced)
- **Reward**: Fraction of hidden test cases passed
- **Trainer**: `HarnessRolloutWorker` with factory of Open Code sessions

The agent doesn't know it's being trained. It operates exactly as it would in production. The proxy captures everything.

## Environments Are Versatile — Not Just for RL

The same environment works for every training method:

- **Evaluation** — run models through the environment, score them. Best starting point for any project. You might find an existing model already handles your task.
- **RL (GRPO)** — the canonical use case.
- **Distillation** — student and teacher both perform tasks in the environment. Compare their behaviors.
- **Self-distillation** — use extra information from the environment as a "hint."
- **SFT** — get a great model to perform tasks in the environment, record traces, use for supervised fine-tuning.

Environments are reusable infrastructure underneath all training methods. Build the environment once, use it everywhere.

## The Capability Cycle

How capabilities evolve in practice:

1. **Discover** a capability in a model or harness ("it can generate Blender representations")
2. **Integrate** into the harness (prompts, tools)
3. **Benchmark** — build an eval around the task
4. **Build environment** — formalize the benchmark into a training environment
5. **Train** — use the environment with GRPO to bake the capability into model weights
6. **Evaluate** — ship, update harness, remove scaffolding that's now in the weights
7. **Repeat** — discover the next capability gap

## The Recipe Is Not Fixed

Sergio's important point across the series: this pipeline is not a fixed sequence you must follow start to finish. You pick the pieces you need. If SFT alone gets you where you need to be, stop there. If distillation is enough, skip RL. If you already have a strong base model and just need it to handle a specific environment, go straight to RL. The pipeline is a menu, not a conveyor belt. The sessions present the full picture so you understand what each method does — but your actual recipe depends on your starting model, your task, your data, and your compute budget.

---

# Session 5 — RL for Agents Workshop: The Broader Ecosystem

**Multiple speakers. "RL for Agents Workshop - Deep Dive on Training Agents with RL and Open Source."**

This was a workshop with five talks and a panel discussion. It covers the broader landscape — where the community is, what's missing, what's coming next.

## Lewis Tunstall — The State of Open-Source Agent Training

**Lewis Tunstall, Hugging Face. Research engineer who led SmolLM and worked on TRL.**

### The Agentic Moment

Every month, the task horizon that agents can handle is growing. METR tracks this: the time a model can productively spend on a task before needing human help is doubling roughly every 4-7 months. This means the class of tasks agents can handle — in terms of complexity and duration — keeps expanding.

But there's a split: frontier labs are building agents primarily through RL + environments. The open-source community has been mostly doing post-training (SFT, distillation) on traces. The environments-and-RL approach is where the real breakthroughs are happening, and open source is catching up.

### Training Agents, Not Just Models

Key insight from Lewis: you're training the model to work *within* a harness. The model isn't the agent — the model + scaffold + tools = the agent. SFT teaches the model to operate within that scaffold. RL teaches it to solve tasks within that scaffold. Both need the scaffold as context.

This is why traces are important — they capture the full interaction pattern, not just prompts and completions. Multi-turn, tool calls, context management, sub-agent spawning. The model learns the full interaction protocol of the harness it'll operate in.

### What Open Source Is Missing

Three gaps Lewis identified:

1. **Realistic environments** — most open-source environments are toys (games, simplified tasks). Frontier labs probably have full replicas of enterprise tools (Excel, Slack, databases). If you want open-weight agents that solve real tasks, you need environments that simulate real software, not just games.

2. **Open recipes** — we have open recipes for pre-training (AllMo, SmolLM) and basic post-training. We don't have coherent recipes that show: "train model X on environment Y with method Z, get real-world performance improvement W." Chinese tech reports give high-level information but miss implementation details. We need reproducible recipes for agent training on real tasks.

3. **Long-horizon training** — agents that work on tasks spanning hours or days. The RL training loop for long-horizon tasks is an open problem. How do you give reward signal for a task that takes thousands of tool calls?

## Ofir Press — Benchmarks Are the Source of Everything

**Ofir Press, SWE-bench team.**

### The Benchmark Cycle

Ofir's framing: benchmarks drive all of AI progress. The cycle:

1. Build a good benchmark → spotlights missing capabilities
2. Missing capabilities → clarity on what training data/environments you need
3. Training → new capabilities
4. New capabilities → benchmark gets saturated → build a new benchmark

Benchmarks are perishable. Even the toughest benchmarks saturate within 1-2 years. SWE-bench went from 1.5% to 93.9% (Anthropic's Methuselah). The community needs a constant stream of new benchmarks.

### The Evolution of Benchmarks

1. **School exams** (GSM8K — word problems) → saturated
2. **College exams** (MMLU, HumanEval — first programming class) → saturated
3. **Real human work** (SWE-bench — actual GitHub issues that real developers solved) → nearly saturated
4. **Multi-day tasks** (METR's task suite — tasks that took researchers multiple days) → current frontier
5. **Next**: real-world deployment (tasks in production environments with real users and consequences)

### SWE-bench Design Philosophy

SWE-bench collects real-world tasks from GitHub. The task is defined by a GitHub issue. The solution is the actual PR that fixed it. The verification is the test suite that the PR added.

Key design decisions:
- Tasks are *natural* — they come from real development, not synthetic generation.
- Verification is *automated* — run the tests, check if they pass.
- Difficulty varies naturally — some issues are trivial, some took senior developers days.

SWE-bench Multimodal extends this to tasks involving screenshots, diagrams, and visual context. The next frontier is tasks that require understanding video, audio, and multi-modal content.

### Mining Environments from Real Repos (SWE-Smith)

This is the most actionable takeaway from Ofir's talk for anyone building their own training environments:

SWE-Smith automatically generates thousands of RL environments from GitHub repositories. The pipeline:

1. **Find repos** with good test suites.
2. **Find PRs that add tests** — these are the gold. The PR gives you: the issue (task description), the branch before the fix (starting state), and the new tests (verifier/reward signal).
3. **Create a sandbox** for each — the repo at the commit before the fix, with all dependencies installable.
4. **Extract the verifier** — the test suite added by the PR. Run the agent's changes against these tests to get the reward.
5. **Validate** — confirm the tests actually fail before the fix and pass after.

This produced tens of thousands of training environments. Academics with limited budgets used them to train 32B models and showed improvements on SWE-bench. The key insight: you don't need to manually create environments. Real repos with good test coverage are a nearly unlimited source of environment-task-verifier triples.

The same pattern can potentially extend beyond coding — any domain where you have historical task-solution-verification triples (support tickets with resolution validation, accounting entries with audit checks) could be mined similarly.

### The Saturation Problem

When a benchmark saturates, you don't just make it "harder" — you fundamentally change what you're measuring. Going from GSM8K (10 seconds to solve) to SWE-bench (hours) to METR (days) isn't just difficulty scaling — each jump requires qualitatively different capabilities (planning, persistence, error recovery, multi-step reasoning).

## Alex — Recursive Language Models and Long-Horizon RL

### The Long Context Problem for Agents

Agents that work on multi-day tasks generate massive context — potentially millions of tokens of tool calls, observations, intermediate states. No model can hold all of this in its context window simultaneously.

Recursive Language Models (RLMs) address this: instead of one giant context, the model recursively breaks problems into sub-problems, solves each with focused context, and composes the results. The context at any given moment stays manageable, but the model can work on arbitrarily long tasks.

### Training for Long Horizons

Training for long-horizon tasks is hard because:
- The reward is extremely sparse (one signal at the end of a multi-day task)
- The trajectory is enormous (thousands of steps)
- Credit assignment is nearly impossible (which of the 1000 steps mattered?)

Current approaches:
- **Curriculum learning** — start with short tasks, gradually increase horizon
- **Dense intermediate rewards** — break the long task into checkpoints and reward at each
- **Hierarchical RL** — high-level policy decomposes into sub-tasks, low-level policy executes each

This is an active research frontier. No one has a clean solution for RL on truly long-horizon agent tasks.

## Will Brown (Prime Intellect) — Evals and Environments Are the Same Thing

**Will Brown, Prime Intellect. Works on the Verifiers library and the Environments Hub.**

### The Core Claim

Will makes a strong claim: **evaluations and environments are the same thing.** Not similar, not adjacent — the same object.

An evaluation has:
- **Tasks** — what the agent should do
- **Harness** — the tool interface and execution context
- **Metrics** — how to score performance

An environment has:
- **Tasks** — what the agent should do
- **Runtime** — the execution context
- **Rewards** — how to score performance

Same structure, different names. By treating them as the same object, you avoid maintaining two separate implementations — one for evaluation, one for training.

### Why This Matters Practically

If evaluations and environments are different objects, you:
- Build an eval to measure performance
- Separately build an environment to train the model
- Hope they measure/reward the same things
- Maintain two codebases

If they're the same object, you:
- Build one environment
- Use it for evaluation first (benchmark your current models)
- Then use it for training (RL with the same environment)
- Then use it for ablations (swap out models, compare cost/speed/performance)

The environment becomes reusable infrastructure for your entire workflow.

### The Verifiers Library

Will's library `verifiers` provides the infrastructure for creating environments:

```python
from verifiers import Environment

env = Environment(
    tasks=my_task_dataset,
    harness=my_tool_interface,
    metrics=[format_check, compile_check, test_check]
)

# Use for eval
results = env.evaluate(model="claude-4-opus")

# Use for training
trainer = GRPOTrainer(model="qwen-8b", environment=env)
trainer.train()
```

The same environment object handles both. The tasks are the dataset, the harness is the agent interface, the metrics are the reward functions.

### The Environments Hub

~4,000 environments on the Hub. The goal: make creating and sharing environments have as little friction as publishing a dataset on Hugging Face. You should be able to fork an environment, modify it for your domain, validate it, and publish it in minutes.

The current gap: most environments are toys. The community needs environments for real enterprise tasks — email management, project management, accounting, customer service — not just games and coding puzzles.

### Tau-Bench as an Example

Tau-bench is a popular agent benchmark that tests customer service agents. Will uses it to illustrate the environment abstraction:

- **Tasks** = customer service scenarios (return a product, change a flight)
- **Harness** = API calls to simulated airline/retail systems
- **Metrics** = did the agent complete the customer's request correctly?

The same Tau-bench environment can be used for:
1. Evaluating which model handles customer service best
2. Training a smaller model with RL on customer service tasks
3. Generating synthetic training data by recording large model performances
4. Optimizing prompts by running the same model with different system prompts

### The Scale Problem

Building environments for individual tasks is manageable. But agents need to handle thousands of task types. This suggests the need for *automated* environment generation — agents that create their own environments.

SWE-Smith (from Ofir's team) did this for coding: automatically mine GitHub repos for tasks, extract issues, create sandboxes, generate test suites. The result: tens of thousands of training environments generated automatically. The same approach can potentially extend to other domains.

## Panel Discussion — What's Missing in Open Source

Key exchanges from the panel (Lewis, Ofir, Alex, Will):

### What Do Frontier Labs Have That We Don't?

The panel's consensus:
- **Data at scale** — not just internet data, but data from real users interacting with their products. Billions of real conversations, tool calls, failures.
- **Mature reward modeling** — the biggest gap is probably in reward modeling. Labs have scaled up both verifiable rewards (deterministic checks) and RLHF-style learned rewards (trained reward models), and use them together.
- **Environments for real tools** — full simulations of enterprise software. Not toy tasks.
- **Infrastructure** — the combination of all the small advantages (better infra, more compute, faster iteration) compounds.

No single secret sauce. Just a compound advantage from having all pieces at scale simultaneously.

### Can Academics Compete?

Yes, with targeted approaches:
- SWE-Smith showed that academics with limited budgets can generate thousands of environments automatically and train competitive models.
- Small models (1-8B) + targeted environments + RL = meaningful capability improvement on specific tasks.
- The gap between frontier models and fine-tuned small models is closing for specific use cases.

### Who Should Build Environments?

The panel leans toward: domain experts + environment tooling. The people who know accounting should build accounting environments, not ML engineers who don't understand debits and credits. The tooling (OpenM, Verifiers) should make it easy enough that domain experts can create environments with minimal ML knowledge.

---

# Session 6 — Agentic Evaluations Workshop: Measuring Whether Any of This Works

**Multiple speakers. "Agentic Evaluations Workshop - Deep Dive on the Future on Evals for Agents."**

Four talks covering eval transparency, reliability measurement, dynamic simulation, and the evaluation infrastructure gap.

## Balaji Koch — Eval Reporting Is Broken

**Balaji Koch, Technical AI Policy Researcher at Hugging Face. Co-leads the Eval Eval coalition (400+ researchers).**

### The Transparency Problem

Eval reporting for regular LLMs is already a mess. Examples:

- **Hidden fine print**: OpenAI's GPT-5.2 system card reported a high SWE-bench score but omitted 40 of 237 problems in footnotes you'd miss unless you read the full card.
- **Benchmark mixing**: Meta's Llama 4 release used different model versions for different benchmarks — cherry-picking the best score from each variant.
- **Chart manipulation**: OpenAI showed scores of 69.1 and 30.8 as similarly-tall bars on a histogram. No axis starting at zero, no error bars.

### Social Impact Eval Transparency Is Declining

The Eval Eval coalition studied 171 model release documents. Findings:

- Model developers are *less* transparent about eval results over time.
- Environmental cost reporting has dropped below 15%.
- Google and Meta reported much more in 2022-2023. They've pulled back.
- Reason: teams dedicated to documentation and social impact evaluation have been broken up or reassigned. Legal liability concerns push companies toward capability reporting over risk measurement.

Positive news: third-party evaluation by organizations like METR, Apollo, and MARCO has increased in both quality and quantity.

### Every Eval Ever — Centralizing Eval Results

Two parts:

1. **A unified data format** — requires: source provenance, model specification (including quantization, version, prompt format), evaluation library used. Supports both aggregate and instance-level results.
2. **A public dataset** of every possible first- and third-party evaluation on HF Hub.

The product: **Eval Cards**. Go to a website, select a model, see all evaluations organized by category. Hold confounding variables constant (same quantization, same eval library, same prompt format) and compare actual scores. Makes benchmark gaming structurally harder.

### What Agentic Evals Need That LLM Evals Don't

Agentic evals are harder because agents are complex systems. Two agents can both pass an eval but tell completely different stories — one takes fewer steps, costs less, is faster, has zero errors. The other barely scrapes by. A single pass/fail score hides all of this.

What agentic evals must capture:
- **Action sequences** — chain of thought, tool calls, context at each action, environment state
- **Session data** — two runs with identical aggregate scores can have completely different behavior. Without session-level data, you can't reproduce results.
- **Agent identity** — the model is not the agent. You need: model name, sub-agent list, MCP servers, memory config, tool setup.
- **Robustness measurements** — seeds, prompt perturbations, pass@k
- **Cost** — inconsistent or absent across benchmarks
- **Human interaction** — impact of human input noise is under-measured

### Agentic Schema Extensions

Proposed extensions to the Every Eval Ever schema for agents:
- **System compositions** — models, their roles, sub-agents, MCP servers
- **Session semantics** — what constitutes a run
- **Interaction accounting** — all human-agent interaction measurements
- **Eval conditions** — everything needed to reproduce the agent evaluation

## Arvind Narayanan — The Capability-Reliability Gap

**Arvind Narayanan, Princeton. Paper: "Towards a Science of AI Agent Reliability."**

### The Paradox

AI agents crush capability benchmarks. If you believe the numbers, companies should be replacing people everywhere. That's not happening. No measurable GDP impact yet.

The explanation this work explores: **capability benchmarks measure only one component of what makes an agent useful.** There's a capability-reliability gap, and it's large.

### Real Failures

- **Rabbit R1** — could order products online by voice (incredible capability). Delivered food to wrong addresses, messed up tips. Dead on arrival.
- **OpenAI Operator** — incorrect purchases.
- **Agentic coding** — deleting production databases.

The stakes distinction: if Alexa plays the wrong song 10% of the time, that's an annoyance. If an agent uses your credit card wrong 10% of the time, that's dead on arrival.

### Four Dimensions of Reliability

Analyzing failures across domains, four dimensions emerged:

#### 1. Consistency

70% accuracy can mean two very different things:
- 70% of tasks the agent consistently handles, 30% it consistently fails on (manageable — just avoid the 30%)
- On *any* given task, it unpredictably works 70% and fails 30% (catastrophic — you never know if this particular run will work)

Sub-metrics:
- **Outcome consistency** — same pass/fail for the same task each time?
- **Trajectory consistency** — same action sequence each time? (Low consistency is fine for creative tasks, bad for customer service at scale)
- **Cost consistency** — stable token/API/time usage across runs?

#### 2. Robustness

- **Fault robustness** — inject real-world faults (API timeouts, errors) into the environment. Real deployments always have these.
- **Prompt robustness** — reword the prompt preserving semantics but changing style. If informal vs. formal prompts give different answers, that's a reliability failure.

#### 3. Predictability (Calibration)

- **Calibration** — if the agent says "60% confident," is it right 60% of the time? Typically overconfident — says 1.0 when right only half the time. Good news: calibration has been improving (companies got burned by overconfident sycophantic chatbots).
- **Discrimination** — the agent shouldn't always say ~50%. It needs to genuinely separate its successes from its failures. Bad news: discrimination has been getting *worse* even as calibration improves.

#### 4. Severity

Are failures minor (formatting error) or catastrophic (data deletion)? Measured separately.

### The Key Finding

14 frontier models with agentic scaffolds, measured on Gaia and Tau-bench over 18 months:

**While accuracy improved dramatically, reliability improved only gradually.**

There's a linear relationship between accuracy and reliability — reliability goes up as accuracy goes up, but the slope is much shallower. If capability improvement is exponential, reliability improvement is stubbornly linear.

### Insights from Trace Analysis

- **Calibration confusion** — models confuse a clean execution process with a correct answer. If tool calls failed during execution, the model doubts its answer even when the answer is correct.
- **Hallucination under stress** — when faults are injected (data becomes inaccessible), models that normally don't hallucinate start hallucinating to fill the gap.
- **Ambiguous questions** — in real deployment, tasks are always ambiguous. Models don't handle ambiguity well.

### Augmentation vs. Automation

- **Augmentation** (coding agent with human review) — reliability errors are tolerable.
- **Automation** (autonomous customer service) — reliability errors are catastrophic.

For deployment decisions: there should be a reliability threshold, not just a capability threshold, before deploying an agent for autonomous tasks.

### The Metric Saturation Problem

Provocation: maybe benchmarks aren't getting saturated — **the metric is.** If you only measure pass/fail accuracy, you're missing 12+ dimensions of reliability, plus collaboration ability, cost, latency. Agent evaluations need to be multi-dimensional.

## Pierre André — GAIA 2: Dynamic Multi-App Simulation

**Pierre André, Research Engineer at Meta.**

### The Problem GAIA 2 Solves

Previous evals are static: prompt in, answer out. But real-world agents operate in changing environments. New emails arrive, calendars update, colleagues respond, prices change. The outside world sends inputs the agent must react to.

Example: "Organize a wine tasting with colleagues." The agent emails colleagues, *waits for responses*, adjusts plans based on who can attend, books a venue, handles cancellations. The environment is actively changing.

### Multi-App Simulation

Built on Meta's **Agent Research Environment (ARE)** platform. Four core concepts:

1. **Apps** — like phone apps. Email, messenger, calendar, file system. Each maintains state and exposes tools/APIs.
2. **Universe** — the simulated world: initial state of all apps (past emails, calendar events, message conversations with simulated personas). Generated from persona databases using LLMs.
3. **Events** — injected from user, agent, or environment. This is what makes GAIA 2 dynamic.
4. **Scenarios** — not simple prompts. Include: task prompt + expected agent events + environment events that happen during the task.

### Why Simulation Over Real World

- **Reproducible** — replay exact scenarios
- **Safe** — test destructive actions without consequences
- **Cheap** — no external API costs

### Five Capability Categories

1,000 scenarios across 10 universes, ~11 apps each:

1. **Execution** — multi-tool-call tasks in a single turn. "Cancel all meetings," "respond to that email."
2. **Search** — find information across different APIs. "I forgot my Netflix password but shared it with my parents. Can you find it?" (Agent must figure out who parents are, search communication apps.)
3. **Adaptability** — multi-turn execution where the environment disrupts. Agent books meetings → attendees cancel → agent must reschedule.
4. **Time** — events based on time progression. "Book a flight when it drops below $500." Simulates weeks of time (fast-forwarded, not real-time).
5. **Ambiguity** — tasks the agent cannot resolve without asking follow-up questions. Must stop and ask before doing something wrong.

### Stress Testing

Four noise injection mechanisms:
- **Tool failures** — inject failures on any app
- **API variation** — change tool names and signatures to prevent overfitting
- **Environment noise** — external events unrelated to the task
- **Agent-to-agent** — agent can't call tools directly, must delegate to a sub-agent in natural language

### Verification

Moved away from rubric judging (expensive, LLM-dependent):

- **Hard verifiers** — annotated expected action diagrams. Compare expected vs. actual action sequences: right order, right parameters, right recipients. Pure equality checks. Cheap and reproducible.
- **Soft verifiers** — for free-form content (email body text), use an LLM to check semantic match.

### Key Results

Most agents perform well on search and execution. On time tasks: ~0% across all models. Adaptability and ambiguity scores are also poor. These are the frontier capabilities — the ones training needs to target next.

## Mahesh (Bespoke Labs) — Environments Are Critical for Evaluation Too

**Mahesh, Co-founder and CEO at Bespoke Labs.**

### The Eval-First Approach

Bespoke Labs' practical advice: **start with evaluation, not training.** Build the environment first. Run existing models through it. See where you stand. You might discover:
- An off-the-shelf model already handles your task (no training needed)
- The task is much harder than you thought (you need a different approach)
- The reward signal is too sparse (you need to redesign the environment)

Either way, you learn about the problem before burning compute on training.

### The Reward Hacking Problem (Practically)

No silver bullet. The practical approach:

1. **Look at the data** — experienced ML engineers talk about data constantly. Have a window open showing trajectories at all times.
2. **Build separate observation metrics** — measure *how* the agent performs, not just *whether* it succeeds.
3. **Watch for entropy collapse** — behavior becomes repetitive but reward keeps climbing? Go look at the trajectories manually.
4. **Track tool usage patterns** — an agent solving a real task should use varied tools. Same tool in the same way every time = suspicious.
5. **Separate observation from training metrics** — have logic tests: "if this task is done correctly, *these other metrics* should also go up." If they don't, your reward is being gamed.

---

# Synthesis — How This All Fits Together

## The Training Pipeline in One View

```
SFT on traces          → Model learns format, tool calls, harness protocol
                            ↓ (emulation ceiling reached)
Distillation           → Model learns from teacher's judgment on its own attempts
                            ↓ (teacher ceiling reached)
RL with GRPO           → Model discovers novel strategies from task rewards
                            ↓ (static reward functions limit complexity)
RL with Environments   → Model acts in stateful worlds with multi-step tasks
                            ↓ (need to measure if any of this worked)
Evaluation             → Capability + reliability + robustness + calibration
                            ↓ (gaps found)
Back to top            → New traces, new environments, retrain
```

Each step has a ceiling. The ceiling of SFT is the training data quality. The ceiling of distillation is the teacher's judgment. RL has no theoretical ceiling but hits practical limits (reward design, variation, compute). The cycle continues.

## The Complete Toolchain

| Step | Tool | Purpose |
|------|------|---------|
| Trace collection | Pi, Open Code, Claude | Record agent behavior for SFT data |
| SFT training | TRL SFTTrainer | Teach format and structure |
| Distillation | TRL DistillationTrainer | Learn from teacher judgment |
| RL training | TRL GRPOTrainer | Learn from task rewards |
| Environment definition | OpenM | Define stateful environments |
| Environment hosting | HF Spaces / Sandboxes | Run environments at scale |
| Metrics tracking | Tracelo | Observe training curves |
| Evaluation | Inspect AI, GAIA 2, SWE-bench | Measure capability |
| Reliability measurement | Princeton Reliability Index | Measure consistency, robustness, calibration |
| Result comparison | Every Eval Ever / Eval Cards | Standardized comparison across models |

## The Most Expensive Mistakes

1. **Deploying based on capability scores alone.** The reliability gap means your 90% accurate agent will fail catastrophically often enough to kill user trust. Measure reliability dimensions separately. Cost: user churn, production incidents, liability.

2. **Putting verifiers in the agent's sandbox.** The agent will find and exploit verification data. Always separate sandbox from grading. Cost: your trained model looks great on metrics but learned to game the reward, not solve the task.

3. **Static reward for multi-step tasks.** A single pass/fail at the end of a 50-step trajectory gives almost no signal. Stack rewards from easy to hard to maintain variation. Cost: training runs that burn GPU hours and produce nothing.

4. **Treating all failures as equal.** A formatting error and a database deletion are not the same severity. Set different reliability thresholds for augmentation vs. automation. Cost: blocking safe deployments or shipping dangerous ones.

5. **Ignoring prompt robustness.** Works with your test prompts, fails with real user prompts. Test with semantically-preserved prompt perturbations. Cost: works in demos, fails in production.

6. **Starting with training instead of evaluation.** Build the environment first. Run existing models through it. You might not need training at all. Cost: wasted training compute solving a problem that doesn't exist.

## What's Established, What's Predicted

**Already happening:**
- RL environments becoming the default training method for agent capabilities (OpenM + TRL, DeepSeek's pipeline)
- Third-party evaluation growing as first-party reporting declines
- Dynamic simulation environments (GAIA 2) replacing static benchmarks for agents
- Small models (1-8B) achieving meaningful capability improvement on narrow tasks with targeted RL

**Evidence-based predictions:**
- Reliability metrics will become as standard as accuracy within 12 months. The Princeton Reliability Index will drive this.
- Automated environment generation (agents building their own training environments) will become common. The infrastructure is being built now.
- The capture-proxy pattern (black-box harness connection) will win over white-box for production agent training, because real harnesses are too complex for trainer-level control.
- Domain experts (not ML engineers) will become the primary creators of training environments, enabled by tooling like OpenM CLI.

## The Single Most Important Takeaway

The capability-reliability gap is real, measured, and not closing automatically. If you're building agents for production: measure reliability dimensions (consistency, robustness, calibration, failure severity) separately from capability — and set explicit reliability thresholds before deploying for autonomous tasks. Start with evaluation, not training. Build the environment first.

