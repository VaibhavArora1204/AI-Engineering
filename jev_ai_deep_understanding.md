# Jev AI: The Decision Layer for Agents

> **Objective:** Reach high-leverage understanding of Jev quickly.  
> **Scope:** Only Jev. No broad AI survey. No hype verdicts.  
> **Format:** Progressive — each section builds on the last. Don't skip ahead until the current section clicks.

---

## Prerequisite Check

Before reading this document you should already know:

- What an LLM is (a model that generates text token-by-token)
- What an AI agent is (software that uses an LLM in a loop to accomplish tasks)
- What an API call looks like (send JSON, get JSON back)

If yes → proceed. If no → learn those three things first, then come back.

---

## 1. The Mental Model

### The one-liner

```
Code computes.   LLMs create.   Jev decides.   Fresh state proves.
```

### What each piece does

| Layer | Job | Output shape | Example |
|---|---|---|---|
| **Code** | Deterministic computation | Exact values | `price * quantity = total` |
| **LLM** | Open-ended generation & reasoning | Prose, code, plans | "Write a summary of this document" |
| **Jev** | Bounded decision-making | Typed probability + label | "Which of these 4 teams should handle this ticket?" → `ops` at 0.91 confidence |
| **Fresh state** | Verification | Observable evidence | Re-read the database row after the write |

### Why this split matters

An LLM generates text **token by token**. When you ask it "should I route this to ops or engineering?" it still:

1. Processes your full prompt through attention layers
2. Generates an answer **autoregressively** — one token at a time: `"` → `I` → ` would` → ` recommend` → ` routing` → ` this` → ` to` → ` ops` → `"`
3. Returns prose you then have to parse

This is like hiring a novelist to fill in a checkbox.

**Jev eliminates autoregressive generation entirely.** It evaluates all predefined options in a **single parallel forward pass** and returns structured data — a label, a probability distribution, and a confidence score. No prose. No parsing. No hallucinated tool names.

### The agent loop with Jev

```
Current State (goal + evidence + live options)
        │
        ▼
      ┌─────┐
      │ JEV │ ← evaluates bounded questions against state
      └──┬──┘
         │
         ▼
   Choose: tool / model / agent / escalation
         │
         ▼
      Execute (one observable action)
         │
         ▼
   Read fresh state from the external system
         │
         ▼
      ┌─────┐
      │ JEV │ ← next decision on new evidence
      └──┬──┘
         │
         ▼
       ...repeat
```

**Key insight:** Jev sits at every fork in the agent's graph. It does not replace the agent — it replaces the slow, expensive LLM calls that were being used *just to make a choice*.

### ✅ Checkpoint: What to understand before moving on

- Jev is a **decision model**, not a generation model
- It returns **typed structured data** (labels + probabilities), not prose
- It evaluates options in **parallel** (non-autoregressive), not token-by-token
- It sits **between** states in an agent loop — it does not own execution

---

## 2. The Three Primitives

Jev exposes exactly three question types. Every decision you send to Jev is expressed as one of these. Nothing else.

### 2.1 Choice — "Pick one"

**What it does:** Selects the single best option from a closed set.

**When to use:** Routing, classification, tool selection — any time exactly one option should win.

**Agent example:** An inbox agent needs to decide who handles an incoming request.

```python
"route": Choice(
    instructions="Which team should own this?",
    criteria={
        "ops":         "Operational work — deployments, infra, incidents",
        "engineering": "Software defect — bugs, regressions, missing features",
        "support":     "Customer-facing — account issues, billing, how-to",
        "human":       "Unclear, sensitive, or requires judgment"
    }
)
```

**What Jev returns:**

```json
{
  "choice": "engineering",
  "probabilities": {
    "ops": 0.05,
    "engineering": 0.82,
    "support": 0.11,
    "human": 0.02
  },
  "confidence": 0.82
}
```

**What you do with it:** Your code reads `choice` and `confidence`. If confidence < your threshold → escalate to human. Otherwise → route to the winning team.

---

### 2.2 Score — "Rate it on a scale"

**What it does:** Returns a numeric position on a defined rubric. Can return fractions.

**When to use:** Complexity assessment, relevance ranking, quality scoring — any ordered scale.

**Agent example:** Rate how complex the next step in a task is.

```python
"complexity": Score(
    instructions="How complex is the next step?",
    criteria=[
        "Mechanical — single, obvious action",
        "Multi-step — requires sequencing several actions",
        "Ambiguous or high-risk — needs human judgment"
    ]
)
```

**What Jev returns:**

```json
{
  "score": 1.7,
  "distribution": [0.05, 0.55, 0.40],
  "confidence": 0.62
}
```

**What you do with it:** `score >= 2.0` → escalate. `score < 1.0` → auto-execute. In between → add review step.

---

### 2.3 Noul — "Probability of yes"

**What it does:** Returns a single value from 0 to 1 — the probability that one statement is true.

**When to use:** Approval gates, completion checks, binary relevance — any yes/no question.

**Agent example:** Should this agent action require human sign-off?

```python
"needs_human": Noul(
    instructions="Does the next step require human approval?"
)
```

**What Jev returns:**

```json
{
  "noul": 0.87
}
```

**What you do with it:** `noul >= 0.70` → require human approval. Below → auto-proceed.

> **Why "Noul"?** It's TypeSafe's name. Think of it as "the probability of yes" and move on. No separate confidence field — the number *is* the confidence.

---

### Combining primitives

The real power: you ask all three in a **single API call** against the **same state**.

```python
questions = {
    "route":       Choice(...),   # who handles this?
    "complexity":  Score(...),    # how hard is it?
    "needs_human": Noul(...)      # does it need approval?
}

response = client.system_one(
    state=current_state,
    questions=questions
)
```

One request. Three independent decisions. Three typed outputs. No parsing.

### Design rules for questions

| Rule | Why |
|---|---|
| Keep each question **atomic** | One judgment a person could make quickly from the evidence |
| If a question mixes urgency + risk + relevance → **split into 3 questions** | Combine their outputs in code with explicit weights |
| Add `"human"` or `"other"` option when the list might be incomplete | So the model can abstain instead of forcing a bad pick |
| Criteria should describe **observable situations**, not repeat labels | `"Software defect"` is better than `"engineering stuff"` |

### ✅ Checkpoint: What to understand before moving on

- Every Jev question is **Choice**, **Score**, or **Noul** — nothing else
- Each returns **typed structured data** with probabilities
- Multiple questions can share the same state in **one API call**
- Your **code** reads the outputs and applies thresholds — Jev proposes, code disposes

---

## 3. The Key Advantage: Batch Decisions

This is the single most important practical insight about Jev.

### The problem with serial LLM decisions

A typical agent at a decision fork:

```
Need to classify a GDPR document against 13 compliance checks.

Traditional approach:
  LLM call 1: "Is personal data collected?"      → wait → parse → yes
  LLM call 2: "Is consent mechanism present?"     → wait → parse → yes
  LLM call 3: "Is data retention specified?"      → wait → parse → no
  ...
  LLM call 13: "Is breach notification covered?"  → wait → parse → yes

Total: 13 sequential API calls
```

Each call independently tokenizes the full document, runs inference, generates text, and returns. You pay for the document tokens **13 times**.

### Jev's batching

```
Jev approach:
  1 API call with 13 Noul questions against the same state

Total: 1 call
```

### Measured result (TypeSafe's public GDPR experiment)

| Metric | 13 questions batched (1 call) | 13 questions serial (13 calls) | Advantage |
|---|---|---|---|
| **Requests** | 1 | 13 | 13× fewer |
| **Latency** | 0.27 s | 2.71 s | **10× faster** |
| **Cost** | $0.000497 | $0.006090 | **12.2× cheaper** |

### Why this works technically

Jev processes the state **once** through its model. The state representation is computed and cached. Each question is then evaluated against that cached representation **in parallel** — not sequentially.

In a traditional LLM:
- Each call = full forward pass + autoregressive generation
- State tokens are re-processed every time

In Jev:
- State processed once → internal representation
- All questions evaluated simultaneously against that representation
- Output is structured data, not generated text → no sequential token prediction

### When batching works and when it doesn't

| ✅ Works | ❌ Doesn't work |
|---|---|
| Questions are **independently answerable** from the same state | Question B **depends on** Question A's answer |
| Same evidence applies to all questions | Each question needs different external data |
| Questions are bounded (Choice/Score/Noul) | Questions require open-ended reasoning |

**Rule:** If question B needs the answer to question A, that's a second decision step. Ask A, execute or transform state, then ask B against fresh evidence.

### The compounding effect

When each decision costs ~$0.00004 and takes ~0.02s, you can afford to:

- Check **every** email, not a sample
- Score **every** candidate, not just the top 10
- Validate **every** agent action, not just the risky ones
- Review **every** draft, not just the final version

> **The moat is decision density.** When decisions become cheap enough, you evaluate everything instead of sampling.

### ✅ Checkpoint: What to understand before moving on

- Independent questions against the same state can be **batched into one call**
- This eliminates redundant state processing and autoregressive overhead
- The advantage compounds: 10× per call × many calls per task = massive workflow savings
- Dependent questions must be separate calls with fresh state between them

---

## 4. Where Jev Sits (and Where It Doesn't)

### The architecture

```
┌────────────────────────────────────────────────────────────┐
│                    AGENT CONTROL LOOP                       │
│                                                            │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│  │ OBSERVE  │───▶│  DECIDE  │───▶│   GATE   │             │
│  │          │    │  (Jev)   │    │  (Code)  │             │
│  │ Read the │    │          │    │ Validate │             │
│  │ current  │    │ Choice   │    │ threshold│             │
│  │ state    │    │ Score    │    │ permission             │
│  │ from the │    │ Noul     │    │ budget   │             │
│  │ external │    │          │    │ rate limit│             │
│  │ system   │    │          │    │          │             │
│  └──────────┘    └──────────┘    └────┬─────┘             │
│                                       │                    │
│                    ┌──────────────────┐│                    │
│                    │                  ▼│                    │
│              ┌──────────┐    ┌──────────┐                  │
│              │  VERIFY  │◀───│ EXECUTE  │                  │
│              │          │    │          │                  │
│              │ Re-read  │    │ One      │                  │
│              │ external │    │ observable                  │
│              │ system   │    │ action   │                  │
│              │          │    │          │                  │
│              └────┬─────┘    └──────────┘                  │
│                   │                                        │
│                   └──── new state ──── back to OBSERVE     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### What Jev controls

| Decision point | Questions | Example |
|---|---|---|
| **Agent routing** | Choice: which worker? Score: complexity? Noul: need approval? | Chief of Staff assigns tasks to specialized agents |
| **Tool selection** | Choice: which tool? Noul: is this safe? | Only offer currently authorized tools |
| **Model routing** | Choice: fast/powerful/human? Score: reasoning depth? | Use cheap model for simple queries, expensive for hard ones |
| **Browser actions** | Choice: click/type/scroll? Choice: which element? Noul: done? | Browser agent selects from visible page elements |
| **Email triage** | Choice: reply/wait/junk? Noul: urgent? Noul: impersonation? | Classify 500 emails in seconds |
| **Quality checks** | Noul × 21 checks per document | Run all editorial checks in parallel |
| **Safety gates** | Noul: allowed? Score: harm level? Choice: run/block/ask? | Review every tool call before execution |

### What Jev must NOT replace

| Task | Why not Jev | What to use instead |
|---|---|---|
| **Writing and reasoning** | Output must be *created*, not selected | LLM (GPT, Claude, etc.) |
| **Exact math & dates** | Answer can be *calculated* deterministically | Code |
| **Irreversible execution** | Sending a payment, deleting data, publishing | Deterministic policy + human approval |
| **Multi-step planning** | Requires chain-of-thought reasoning | LLM with structured planning |
| **Open-ended creativity** | No bounded answer space | LLM |

### The selection rule

```
Is the answer space bounded?
  ├─ YES → Can it be calculated exactly?
  │          ├─ YES → Use CODE
  │          └─ NO  → Use JEV
  └─ NO  → Does it require creation/reasoning?
             ├─ YES → Use LLM
             └─ NO  → Rethink the question (probably needs restructuring)
```

### Confidence-based thresholds

Not all decisions have the same risk. Set thresholds by consequence:

| Action type | Starting threshold | If below threshold |
|---|---|---|
| Internal label/tag | 0.70 | Accept anyway |
| Agent handoff | 0.80 | Re-evaluate with more context |
| Browser/tool action | 0.90 | Ask human |
| Publish / pay / delete | Policy + human approval | Block |

> **Critical rule:** The model may propose a branch. **Code decides** whether that branch is allowed to execute. Never let a probability bypass permissions, budgets, rate limits, or irreversible-action controls.

### ✅ Checkpoint: What to understand before moving on

- Jev is the **decision layer** at every fork in the agent graph
- Code owns **validation, permissions, and execution** — Jev never acts directly
- After every action, **read fresh state** — never reuse cached observations
- Match tool to job: Code computes, LLMs create, Jev decides

---

## 5. Evidence Assessment

What has actually been demonstrated vs. what is claimed vs. what is speculative.

### 🟢 Demonstrated (public, reproducible builds)

| Build | Result | Scope & caveats |
|---|---|---|
| **Browser Use** | Google Flights in 7.07s. Protocol calls 1,092 → 101. Median task time 9.45 → 7.09s | Six alternating runs of one task. Flight search only, no booking |
| **Every** | 777 judgments across 37 documents + 21 writing checks in < 0.7s at ~$0.0025 | Writing-quality linting — fixed checks, not open-ended editing |
| **Mobile Jev** | Uber flow reached payment selection in ~21s and 9 actions | Demo stops before booking. Verifies actual screen state |
| **GDPR batching** | 13 questions in 1 call: 10× faster, 12.2× cheaper than 13 sequential | Single document, controlled experiment |
| **Hermes skill routing** | Wrong skill loads: 16.8% → 7.3%. Needless loads: 9.8% → 4.0% | 488 requests against Claude Haiku 4.5. 37 corrected picks but 7 newly broken picks |
| **Legal re-ranking** | Top-1: 5% → 18%. Top-10: 38% → 62%. 1,200 calls cost $0.0645 | CLERC benchmark with BM25 shortlists of 30 passages |
| **Stagehand** | Task cost ~$0.001 per browser action | Single reported demo |
| **Cua (2048)** | 44.9s / $0.00108 vs Astra 294.9s / $1.45 | Run-specific numbers, not averaged across many tasks |

### 🟡 Vendor-reported (published by TypeSafe or close partners)

| Claim | Context |
|---|---|
| Up to **193× faster** than frontier LLMs | On System One-shaped queries (bounded decisions), not general tasks |
| Up to **444× cheaper** | Same qualification — comparison is LLM doing a job it wasn't designed for |
| **580× cheaper** than Fable 5.1 | But Fable caught 7/7 planted defects vs Jev's 6/7 |
| **RLCD training** produces calibrated probabilities | Claimed methodology, limited independent verification available |

### 🔴 Still speculative / unresolved

| Question | Status |
|---|---|
| Does calibration hold across all domains? | Unknown — legal and editorial tested, but not medical, financial, etc. |
| How does Jev handle adversarial/injection inputs? | "Typed output does not make prompt injection impossible" (their own docs) |
| Will the speed advantage survive at massive scale? | Single-user benchmarks — no public data on production load |
| Is the architecture truly novel or a well-optimized classifier? | Open question (see Section 6 below) |
| Can Jev replace all routing LLM calls in a complex agent? | Retriever AI showed cost *increased* 38-51% in some workflows |

### The honest Retriever AI counterexample

Retriever AI reported LinkedIn and Amazon browser tasks running **31-43% faster** with Jev, but **total cost rose 38-51%** because large-page scoring added work. Faster decisions ≠ cheaper workflow when candidate preparation expands.

> **Takeaway:** Benchmark the **entire workflow** cost, not just the decision-model cost.

### ✅ Checkpoint: What to understand before moving on

- The speed/cost claims are real but **workload-specific** — they apply to bounded decisions
- Quality can be slightly lower than frontier LLMs — the value is running checks you couldn't afford before
- Faster decisions don't always mean cheaper workflows (Retriever AI counterexample)
- Treat demo results as "leads to reproduce, not universal constants"

---

## 6. The Real Question: New Building Block or Fast Classifier?

### What Jev demonstrably is

1. **A non-autoregressive decision model** — it does not generate text token-by-token
2. **A typed structured API** — Choice/Score/Noul with probability distributions
3. **A batch-capable evaluation engine** — many questions, one state, one call
4. **Trained with RLCD** for calibrated probability output

### Is that genuinely new?

| Aspect | Assessment |
|---|---|
| **The interface** (typed questions with probabilities) | New as a **product** — not previously available as a managed API with this shape. The concept of structured classification is not new in ML |
| **Non-autoregressive architecture** | Novel application to agent decision-making. Traditional classifiers exist, but Jev handles arbitrary unstructured state as input |
| **Batching independent questions** | Significant engineering contribution. The idea of shared-state evaluation is not new, but packaging it for real-time agent loops is |
| **The "System One" framing** | Useful mental model borrowed from Kahneman. The insight — "stop using a generative model for non-generative tasks" — is genuine |
| **Calibrated probabilities (RLCD)** | A real technical contribution if the calibration holds. Enables programmatic threshold-based routing without per-use-case fine-tuning |

### The honest answer

Jev is **both**. It is a fast, specialized classifier/router **and** a genuinely new building block — because it packages classification into a form that is:

- **Developer-friendly** (typed SDK, not custom ML pipeline)
- **Agent-native** (designed for the observe → decide → execute loop)
- **Batch-aware** (multiple decisions per call, not one model per decision)
- **Calibration-aware** (probabilities meant to be used as thresholds, not just confidence)

The "new building block" claim is valid **if** you accept that developer experience and integration design matter as much as algorithmic novelty. The underlying techniques (parallel evaluation, structured output, calibrated probabilities) exist in ML research. The product contribution is making them available as a single API call that fits into agent architectures.

---

## 7. Technical Depth

### 7.1 How does Jev technically produce decisions?

**Traditional LLM:**
```
Input tokens → Transformer layers → Next token prediction → Generate token
→ Append → Next token prediction → Generate token → ... → Parse output
```

**Jev:**
```
Input tokens (state + question schema) → Transformer layers
→ Parallel evaluation of all defined options → Probability distribution
→ Structured output (no generation step)
```

The key difference: **no autoregressive generation loop**. Jev's architecture cuts the KV-cache overhead and the sequential token prediction that makes LLMs slow for simple decisions.

Think of it as:
- LLM = **generative model** that happens to be used for classification (wasteful)
- Jev = **evaluation model** purpose-built for bounded decisions (efficient)

### 7.2 Why can multiple questions be evaluated in one call?

The state (your JSON/text input) is processed through the model **once** to create an internal representation. This is the expensive part — attention computation over all state tokens.

Each question is then evaluated against this **cached representation**. Since questions are independent (by design constraint), they don't need sequential processing. They can be evaluated in parallel within the same forward pass.

```
                    ┌─── Question 1 (Choice) ──→ distribution₁
State ──→ Encode ──→├─── Question 2 (Score)  ──→ distribution₂
                    ├─── Question 3 (Noul)   ──→ probability₃
                    └─── Question 4 (Noul)   ──→ probability₄
```

**Cost implication:** You pay for state encoding once, not N times. The marginal cost of additional questions is small compared to re-encoding the entire state.

### 7.3 How is calibration handled?

**RLCD — Reinforcement Learning for Calibrated Decisions**

The training objective is not just "pick the right answer" but "when you say 0.85, be right 85% of the time."

This matters because:
- Developers set **thresholds** (e.g., "auto-approve if confidence > 0.90")
- If 0.90 actually means "right 75% of the time," the system silently fails
- Calibration means the probabilities are **actionable** — you can write policy against them

**Practical implications:**
- Threshold tuning on labeled examples is still required per workflow
- Calibration may not transfer between domains (legal ≠ email ≠ browser actions)
- You should split labeled examples into dev/holdout sets and calibrate on dev only
- **Pin the model version** — calibration can shift between model updates

### 7.4 What are the important failure modes?

| Failure mode | What happens | Mitigation |
|---|---|---|
| **Wrong but confident** | Jev returns 0.95 confidence on a wrong answer | Never trust confidence alone. Measure on labeled data. Keep an abstain branch |
| **Stale action menu** | A browser element disappeared after the last click, but it's still in the options | Rebuild option set from fresh state before every decision |
| **Too much / untrusted state** | Injected instructions in email body or web page influence the decision | Filter and sanitize state. Treat web pages/emails as untrusted input |
| **Choice used as multi-label** | Task actually matches two categories equally | Use independent Nouls when multiple conditions can be simultaneously true |
| **DONE without proof** | Agent reports task complete but didn't verify | Always re-read the external system. "Completion is a hypothesis until inspected" |
| **Math and dates** | Jev gets calculation wrong | Never use Jev for math. Compute in code, then let Jev judge the result |
| **Prompt injection via state** | Malicious content in state manipulates the decision | Typed output reduces but doesn't eliminate this risk. Defense in depth required |

### 7.5 How would Jev integrate into a LangGraph-style agent?

In a LangGraph agent, you have **nodes** (actions) and **edges** (transitions). Jev replaces the conditional edge logic:

```
Traditional LangGraph:
  Node A → LLM call to decide next node → parse text → route → Node B

With Jev:
  Node A → Jev Choice(options=[B, C, D, human]) → code reads label → route → Node B
```

**Concrete integration pattern:**

```python
# In your LangGraph conditional edge function
def route_decision(state: AgentState) -> str:
    """Replace LLM routing with Jev decision."""
    jev_state = {
        "goal": state.goal,
        "evidence": state.gathered_evidence,
        "available_routes": ["research", "write", "review", "done", "human"],
        "previous_actions": state.action_history[-3:],  # last 3 actions
    }
    
    response = jev_client.system_one(
        state=jev_state,
        questions={
            "next_step": Choice(
                instructions="What should the agent do next?",
                criteria={
                    "research": "Task requires finding or validating external evidence",
                    "write":    "Evidence gathered, ready to produce output",
                    "review":   "Draft exists, needs quality check",
                    "done":     "All acceptance criteria verifiably met",
                    "human":    "Unclear, blocked, or high-risk situation",
                }
            ),
            "is_complete": Noul(
                instructions="Has the original goal been fully achieved?"
            )
        }
    )
    
    route = response.answers["next_step"]
    completion = response.answers["is_complete"].noul
    
    # Code applies policy
    if completion > 0.90 and route.choice == "done":
        return "done"
    if route.confidence < 0.75:
        return "human"
    return route.choice
```

LangChain's public integration already includes experimental `ModelRouterMiddleware` and `AutoModeMiddleware` for this pattern.

---

## 8. Practical Setup Guide

### Path A: You have Jev access

```bash
# Setup
python -m venv .venv
.\.venv\Scripts\Activate.ps1          # Windows PowerShell
pip install typesafe-sdk
$env:TYPESAFE_API_KEY='your_key'
```

```python
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

client = TypeSafeClient()
response = client.system_one(
    state={"message": "Need this fixed today"},
    questions={
        "owner": Choice(
            instructions="Which team should own this?",
            criteria={
                "ops": "Operational work",
                "engineering": "Software defect"
            }
        ),
        "urgent": Noul(instructions="Is same-day action required?")
    }
)
print(response.answers["owner"].choice)       # "engineering"
print(response.answers["urgent"].noul)         # 0.87
```

### Path B: Build before access (use the adapter)

```bash
pip install 'system-one-adapter[openai]'
# or
pip install 'system-one-adapter[anthropic]'
```

```python
from system_one_adapter import SystemOneAdapterClient, Choice, Noul

client = SystemOneAdapterClient(
    structured_outputs=True,
    llm_answer_mode="probabilities",
    normalize_probabilities=True,
)
response = client.system_one(
    state=state,
    questions=questions,
    provider="openai",
    model="gpt-4o-mini"
)
# Same interface, same code — swap to TypeSafeClient later
```

> **Important:** The adapter is NOT Jev. It won't match Jev's latency, cost, or calibration. It reproduces the **interface** so you can build and test your workflow now.

### When switching from adapter to Jev

1. Swap `SystemOneAdapterClient` → `TypeSafeClient`
2. Remove `provider` and `model` arguments (Jev uses `jev-latest`)
3. **Re-run your labeled evaluation set**
4. **Recalibrate thresholds** — probability distributions from a general LLM and Jev are not interchangeable

---

## 9. Ready-to-Steal System Blueprints

Each blueprint follows the same pattern: **State → Questions → Gate → Execute → Verify**

### 9.1 Chief of Staff Router

```
State:  request, active projects, live workers, permissions, last artifact
Ask:    Choice → owner | Score → urgency | Noul → ask-user
Gate:   confidence threshold per worker type
Do:     enqueue one handoff
Verify: destination accepted the job, artifact path exists
```

### 9.2 Model Router

```
State:  task shape, budget, latency target, available models
Ask:    Choice → fast/powerful/human | Score → complexity
Gate:   budget check
Do:     call the selected model
Verify: validate output schema + task-specific quality
```

### 9.3 Email Firewall

```
State:  message, sender history, thread, communication policy
Ask:    Choice → reply/wait/junk | Noul → urgency | Noul → impersonation | Noul → approval
Gate:   impersonation score > 0.5 → block
Do:     tag, research, or draft
Verify: human approves any send or sensitive action
```

### 9.4 Browser Action Controller

```
State:  goal, visible page elements (accessibility tree), current URL, recent actions
Ask:    Choice → operation | Choice → compatible target | Noul → completion
Gate:   confidence > 0.85 for any click/type action
Do:     execute one validated action ID (never invent coordinates!)
Verify: re-observe page, independently test success criteria
```

### 9.5 Safety Gate (pre-execution review)

```
State:  proposed command, caller identity, permissions, policy rules
Ask:    Noul → allowed | Score → harm level | Choice → run/block/ask
Gate:   deterministic policy (never let a probability bypass permissions)
Do:     enforce policy branch
Verify: record effect + immutable audit event
```

### 9.6 Research Feed

```
State:  title, abstract, source, personal research interests
Ask:    Choice → topic | Score → relevance | Noul → include
Gate:   relevance Score > threshold
Do:     store top items, send to writing model for summary
Verify: open source, deduplicate, preserve canonical URL
```

---

## 10. What to Keep Out of Jev

This is as important as knowing what to put in.

| ❌ Keep out of Jev | ✅ Use instead | Why |
|---|---|---|
| Exact math (`7 * 13`) | Code | Jev can hallucinate calculations |
| Date arithmetic | Code | Same — compute then let Jev judge the result |
| Free-form writing | LLM | Jev decides, LLMs create |
| Irreversible execution | Code + human approval | A probability must never bypass delete/pay/send controls |
| Multi-step reasoning | LLM with chain-of-thought | Jev evaluates one bounded question, not reasoning chains |
| Full corpus search | Database/BM25 first, Jev re-ranks the shortlist | Jev should not scan when retrieval indexes exist |

### The priority ladder for your first deployment

| Priority | What to automate | Risk level |
|---|---|---|
| **First** | Routing, tagging, ranking, relevance, repeated quality checks | Low consequence, high frequency |
| **Second (after evaluation)** | Tool gates, browser actions, agent handoffs, completion predictions | Medium — needs fresh state, thresholds, logs, recovery |
| **Last / maybe never** | Payment approval, data deletion, public publishing | High — use deterministic code + human owner |

---

## 11. The Complete Decision Trace

Every production Jev decision should log:

| Trace field | Why | What failure it explains |
|---|---|---|
| State + schema version | Reproduce the exact evidence and contract | Wrong or stale input |
| Full probability distribution | Inspect ambiguity and runner-up routes | Close alternatives hidden by top-1 |
| Threshold + final destination | Separate model output from policy | Why code escalated or blocked |
| Executor receipt | Prove what was attempted | Tool failure after a correct route |
| Fresh-state verification | Prove the external effect | False DONE or partial completion |

### Version the decision contract

Store a schema version with every trace. Changing state fields, criteria, option names, or thresholds creates a **new decision policy** even if the model stays the same. This lets you:

- Reproduce a bad route
- Compare two policies on the same examples
- Roll back without guessing which prompt produced the result

---

## 12. Summary: The One-Page Mental Model

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   CODE COMPUTES   →   exact math, dates, validation     │
│   LLMs CREATE     →   writing, reasoning, planning      │
│   JEV DECIDES     →   routing, scoring, gating          │
│   FRESH STATE     →   proves the result actually worked  │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   3 Primitives:                                         │
│     Choice  →  pick one from a closed set               │
│     Score   →  rate on a defined rubric                  │
│     Noul    →  probability of yes (0 to 1)              │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Key Advantage:                                        │
│     Batch N independent questions in 1 call             │
│     State encoded once, questions evaluated in parallel  │
│     13 questions → 10× faster, 12× cheaper              │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Architecture:                                         │
│     Observe → Jev decides → Code gates → Execute →      │
│     Verify → Fresh state → Jev decides → ...            │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   The Rule:                                             │
│     Jev proposes.  Code disposes.  Fresh state proves.  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Quick Reference: API Shape

```
POST /v1/systemone
Authorization: Bearer $TYPESAFE_API_KEY

{
  "model": "jev-latest",
  "state": {
    "goal": "...",
    "evidence": [...],
    "available_actions": [...],
    "previous_actions": [...]
  },
  "questions": {
    "route":   { "type": "choice", "instructions": "...", "criteria": {...} },
    "risk":    { "type": "score",  "instructions": "...", "criteria": [...] },
    "approve": { "type": "noul",   "instructions": "..." }
  }
}

→ Response:
{
  "answers": {
    "route":   { "choice": "agent", "probabilities": {...}, "confidence": 0.87 },
    "risk":    { "score": 1.3, "distribution": [...], "confidence": 0.74 },
    "approve": { "noul": 0.42 }
  }
}
```

---

## 13. Practical Example: AI-Powered Customer Support Platform

> **Why this example:** Customer support hits every sweet spot for Jev — high volume, bounded decisions, repeating patterns, multiple independent judgments per ticket, confidence-gated escalation, and clear verification. This is where the framework compounds.

### The Scenario

You run **NovaPay**, a fintech startup with 50,000 support tickets/month. You want an AI system that:

- Triages every incoming ticket instantly
- Routes to the right team without human dispatchers
- Auto-resolves simple tickets (password resets, status checks)
- Drafts responses for complex tickets
- Escalates dangerous situations (fraud, compliance, angry VIPs) to humans
- Never takes an irreversible action without approval

Today, 4 human dispatchers read every ticket, decide priority, assign a team, and flag urgency. Average dispatch time: **3.2 minutes per ticket**. You want this under **2 seconds**.

---

### The Decision Graph

Here's the full system — every box is a stage, every diamond is a Jev decision point.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                   TICKET ARRIVES                             │
  │   (email / chat / API — raw text + metadata + customer ID)  │
  └──────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                 STAGE 1: OBSERVE                             │
  │                                                              │
  │  Code enriches the ticket:                                   │
  │  • Pull customer profile from DB (plan, tenure, LTV, flags) │
  │  • Pull last 5 interactions from history                     │
  │  • Check if account has active incidents                     │
  │  • Detect language (deterministic, not Jev)                  │
  │  • Sanitize message body (strip injection attempts)          │
  │                                                              │
  │  Output: enriched_state                                      │
  └──────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │              STAGE 2: TRIAGE (Jev — 1 call, 5 questions)    │
  │                                                              │
  │  ◆ Choice → category                                        │
  │       billing | technical | account | fraud | compliance     │
  │       feedback | other                                       │
  │                                                              │
  │  ◆ Score → urgency (0-3 rubric)                             │
  │       0: informational, no action needed                     │
  │       1: standard, handle within SLA                         │
  │       2: time-sensitive, customer blocked                    │
  │       3: critical, revenue or legal exposure                 │
  │                                                              │
  │  ◆ Noul → is_fraud_signal                                   │
  │       "Does this message indicate unauthorized access,       │
  │        stolen credentials, or financial manipulation?"       │
  │                                                              │
  │  ◆ Noul → needs_human                                       │
  │       "Does this require human judgment due to ambiguity,    │
  │        emotional distress, or regulatory sensitivity?"       │
  │                                                              │
  │  ◆ Noul → is_auto_resolvable                                │
  │       "Can this be fully resolved by an automated action     │
  │        (password reset, status lookup, FAQ link) without     │
  │        any side effects?"                                    │
  │                                                              │
  └──────────────────────┬───────────────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │              STAGE 3: GATE (Code — deterministic)            │
  │                                                              │
  │  Code reads all 5 answers and applies policy:                │
  │                                                              │
  │  IF is_fraud_signal > 0.60                                   │
  │     → FREEZE account + route to Fraud Team (human)           │
  │     → never auto-resolve fraud signals                       │
  │                                                              │
  │  IF needs_human > 0.70 OR category.confidence < 0.75         │
  │     → route to Human Dispatcher queue                        │
  │                                                              │
  │  IF is_auto_resolvable > 0.90 AND urgency.score < 1.5       │
  │     AND customer.plan != "enterprise"                        │
  │     → route to Auto-Resolver                                 │
  │                                                              │
  │  ELSE                                                        │
  │     → route to the team matching category.choice             │
  │                                                              │
  │  ALWAYS:                                                     │
  │     Log full probability distributions + threshold used      │
  │     Enterprise customers: lower auto-resolve threshold       │
  │     Compliance category: always require human review         │
  │                                                              │
  └───────┬──────────────┬──────────────┬────────────────────────┘
          │              │              │
     ┌────▼────┐   ┌─────▼─────┐  ┌────▼──────┐
     │  AUTO   │   │   TEAM    │  │  HUMAN    │
     │ RESOLVE │   │  QUEUE    │  │ DISPATCH  │
     └────┬────┘   └─────┬─────┘  └────┬──────┘
          │              │              │
          ▼              ▼              │
  ┌───────────┐  ┌──────────────┐      │
  │ Execute   │  │ LLM drafts   │      │
  │ automated │  │ response     │      │
  │ action    │  │ (generation, │      │
  │ (code)    │  │  not Jev)    │      │
  └─────┬─────┘  └──────┬───────┘      │
        │               │              │
        ▼               ▼              │
  ┌──────────────────────────────────┐ │
  │     STAGE 4: QUALITY GATE        │ │
  │     (Jev — 1 call, 3 questions)  │ │
  │                                  │ │
  │  ◆ Noul → response_addresses    │ │
  │    _core_issue                   │ │
  │  ◆ Noul → tone_appropriate      │ │
  │  ◆ Noul → contains_accurate     │ │
  │    _information                  │ │
  │                                  │ │
  │  ALL three > 0.85 → send        │ │
  │  ANY below 0.85 → human review  │ │
  └──────────┬───────────────────────┘ │
             │                         │
             ▼                         │
  ┌──────────────────────────────────┐ │
  │     STAGE 5: VERIFY              │ │
  │                                  │ │
  │  • Re-read ticket status from DB │ │
  │  • Confirm message was sent      │ │
  │  • Check customer saw response   │ │
  │  • Log complete decision trace   │ │
  │                                  │ │
  │  If ticket reopened within 1hr → │ │
  │  flag triage as potential miss   │ │
  └──────────────────────────────────┘ │
                                       │
          ◀────────────────────────────┘
          (human handles directly,
           system learns from their
           decisions for threshold
           calibration)
```

---

### Why Jev Shines Here: Decision by Decision

#### Stage 2 — Triage: 5 questions, 1 call

This is the **core win**. Without Jev, you'd need:

| Approach | Calls | Latency | Monthly cost (50K tickets) |
|---|---|---|---|
| 5 separate LLM calls per ticket | 250,000 | ~1.5s per ticket (sequential) | ~$1,500+ |
| 5 questions batched in 1 Jev call | 50,000 | ~0.03s per ticket | ~$25-50 |

All 5 questions are **independently answerable** from the same enriched state. Category doesn't depend on urgency. Fraud signal doesn't depend on auto-resolvability. They share the same evidence but answer different questions. This is exactly the shape Jev was built for.

#### The state is evidence, not a prompt

Notice what goes into the triage state — it's **structured evidence**, not a clever instruction:

```
State for triage:
├── goal: "Triage this support ticket"
├── ticket
│   ├── subject: "I can't login and someone changed my email"
│   ├── body: (sanitized message text)
│   ├── channel: "email"
│   └── timestamp: "2026-09-19T10:32:00Z"
├── customer
│   ├── plan: "pro"
│   ├── tenure_months: 14
│   ├── ltv: 840
│   ├── open_incidents: 0
│   └── flags: ["2fa_disabled"]
├── recent_history
│   ├── last_ticket: "billing inquiry, resolved, 12 days ago"
│   └── sentiment_trend: "neutral"
├── available_routes
│   └── ["billing", "technical", "account", "fraud",
│        "compliance", "feedback", "other"]
└── rules
    ├── "Enterprise customers always get human review"
    ├── "Compliance tickets require human approval"
    └── "Fraud signals freeze account before routing"
```

Everything the model needs to decide is **explicit**. No hidden instructions. No asking the model to figure out who the customer is. Code already did that work.

#### Stage 3 — Gate: Where code takes control

This is the part many people get wrong. **Jev proposes, code disposes.** Look at the logic:

```
Fraud signal detected?
  → Code freezes the account (irreversible — code + human, never Jev alone)

Low confidence on category?
  → Code routes to human (Jev is uncertain, so let a person decide)

Auto-resolvable AND low urgency AND not enterprise?
  → Code permits auto-resolution (three independent conditions, all in code)

Compliance category?
  → Code forces human review (hard business rule, not a model judgment)
```

Jev made 5 semantic judgments in milliseconds. Code applied 6+ deterministic business rules on top. Neither could do the other's job efficiently.

#### Stage 4 — Quality gate: Jev checks the LLM's work

After the LLM **generates** a draft response (that's the LLM's job — creation), Jev **evaluates** it with 3 independent Nouls:

- Does the response address the core issue? (not just acknowledge it)
- Is the tone appropriate? (professional, empathetic, not robotic)
- Is the information accurate? (no hallucinated policy details)

This is another parallel batch — 3 questions, 1 call, same state (the ticket + the draft). Total added latency: ~20ms. Without this gate, hallucinated responses reach customers.

---

### The Numbers That Would Compound

For a 50,000 ticket/month operation:

| Metric | Before (human dispatch) | After (Jev triage) |
|---|---|---|
| **Triage time per ticket** | 3.2 minutes | < 2 seconds |
| **Monthly dispatcher hours** | 2,667 hours | ~50 hours (review escalations only) |
| **Triage cost per ticket** | ~$1.60 (labor) | ~$0.001 (Jev call) |
| **Time to first response** | 15-45 minutes | < 30 seconds for auto-resolvable |
| **Fraud detection speed** | When dispatcher notices | Instant (every ticket screened) |
| **Quality check coverage** | Spot-checked (~10%) | 100% of auto-responses checked |

The last two rows are the real insight: **you couldn't afford to screen every ticket for fraud or quality-check every response before.** When decisions cost $0.001, you check everything.

---

### Where Each Piece Does Its Job (Summary)

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  CODE does:                                              │
│  ├── Customer lookup from database                       │
│  ├── History enrichment                                  │
│  ├── Language detection                                  │
│  ├── Input sanitization                                  │
│  ├── Threshold enforcement (business rules)              │
│  ├── Account freezing (irreversible action)              │
│  ├── Message sending (irreversible action)               │
│  └── Logging and audit trail                             │
│                                                          │
│  JEV does:                                               │
│  ├── Category classification (Choice)                    │
│  ├── Urgency scoring (Score)                             │
│  ├── Fraud signal detection (Noul)                       │
│  ├── Human-need assessment (Noul)                        │
│  ├── Auto-resolvability check (Noul)                     │
│  ├── Response quality evaluation (3× Noul)               │
│  └── Total: 8 decisions per ticket, 2 Jev calls          │
│                                                          │
│  LLM does:                                               │
│  ├── Draft the actual response text                      │
│  ├── Summarize ticket history for human reviewers        │
│  └── Generate knowledge base article suggestions         │
│                                                          │
│  HUMAN does:                                             │
│  ├── Handle fraud cases                                  │
│  ├── Handle compliance tickets                           │
│  ├── Review low-confidence triage decisions               │
│  ├── Review failed quality gate responses                │
│  └── Final approval on irreversible actions              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

### The 5 Reasons This System Design Shines with Jev

**1. Decision density is extreme**
8 bounded decisions per ticket × 50,000 tickets/month = 400,000 decisions/month. At LLM prices that's expensive. At Jev prices it's pocket change. You screen everything.

**2. Every decision is independently answerable**
Category, urgency, fraud, human-need, and auto-resolvability are all judgment calls a knowledgeable support lead could make independently from reading the same ticket. Perfect parallel batch shape.

**3. Confidence gates create a natural escalation path**
Low confidence → human reviews → their decision becomes a labeled example → thresholds improve over time. The system gets better by handling the cases it's uncertain about honestly.

**4. The LLM stays where it belongs**
Jev doesn't try to write the response. The LLM doesn't try to triage. Each model does what it's architecturally best at. The glue between them is deterministic code.

**5. Fresh state closes every loop**
After sending a response, you re-read the ticket status. After freezing an account, you verify it's actually frozen. After routing to a team, you confirm they accepted. Jev's "completion" label is always a hypothesis until the external system proves it.

---

### Anti-Patterns This Design Avoids

| Anti-pattern | How this design avoids it |
|---|---|
| Using Jev for generation | LLM writes responses; Jev only evaluates them |
| Trusting confidence as accuracy | Code enforces thresholds calibrated on labeled tickets |
| Stale option menus | Available routes are pulled from live team availability, not hardcoded |
| Skipping verification | Every action is followed by a fresh state read |
| One threshold for everything | Fraud uses 0.60 (catch more), auto-resolve uses 0.90 (be certain), category uses 0.75 (moderate) |
| Letting Jev take irreversible action | Account freeze, message send, refund — all gated by code + human |
| Parsing prose from a model | Jev returns typed data; code reads `.choice`, `.score`, `.noul` directly |

---

*Last updated: September 2026. Source: TypeSafe "Jev Engineering" playbook by @0xCodila + independent analysis.*
