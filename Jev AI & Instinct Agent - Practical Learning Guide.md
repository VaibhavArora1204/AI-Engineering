# Jev AI & Instinct Agent: Practical Learning Guide

> **Goal:** Understand Jev deeply enough to know what it changes in agent architecture, without drowning in AI news.
> **Scope:** Tightly scoped to Jev and Instinct Agent only.

---

## Table of Contents

1. [Mental Model](#1-mental-model)
2. [The Three Primitives](#2-the-three-primitives)
3. [Key Advantage: Batching Decisions](#3-key-advantage-batching-decisions)
4. [Where Jev Sits in Architecture](#4-where-jev-sits-in-architecture)
5. [Is Jev Genuinely New?](#5-is-jev-genuinely-new)
6. [Technical Internals](#6-technical-internals)
7. [Limitations & Failure Modes](#7-limitations--failure-modes)
8. [Practical Implementation](#8-practical-implementation)
9. [Real-World Builds](#9-real-world-builds)
10. [Quick Reference](#10-quick-reference)

---

## 1. Mental Model

### The Core Idea

```
LLM generates → Agent acts → Jev decides → Code executes
```

**Breakdown:**
- **LLM generates:** Creates text, code, ideas, reasoning
- **Agent acts:** Orchestrates tools, takes multi-step actions
- **Jev decides:** Picks the next move from bounded options
- **Code executes:** Carries out deterministic operations

### What Jev Adds to an Agent Loop

```
Current State → Jev Decision → Action → New State → Jev Decision → ...
```

### Why a Normal LLM Call Is Inefficient Here

When the required output is only a **bounded decision** (pick one option, score something, say yes/no), a full LLM call is wasteful because:

1. **Serializes decisions:** 13 questions = 13 sequential LLM calls
2. **Generates prose:** You need structured output, not paragraphs
3. **Costs more:** Full generation tokens for a simple pick-one
4. **Slower:** Each call has latency overhead

**Jev's insight:** Split intelligence from execution. Let Jev handle bounded decisions; let LLMs handle generation.

---

## 2. The Three Primitives

Jev works through exactly three typed decisions:

### Primitive 1: Choice — Pick One Option

**What it does:** Selects exactly one winner from a list of options.

**Concrete example: Task Router**
```
State: {goal: "Fix login bug", available_routes: ["llm", "agent", "tool", "human"]}

Questions: {
  route: {
    type: "choice",
    instructions: "Who should handle this next?",
    criteria: {
      llm: "Writing, synthesis, open reasoning",
      agent: "Several tools and changing state",
      tool: "One deterministic operation",
      human: "Sensitive, irreversible, unclear"
    }
  }
}

Response: {
  route: {
    choice: "tool",
    probabilities: {llm: 0.05, agent: 0.12, tool: 0.78, human: 0.05},
    confidence: 0.78
  }
}
```

**When to use:** Exactly one option must win.

---

### Primitive 2: Score — Evaluate on a Defined Scale

**What it does:** Returns a numeric score on a rubric you define.

**Concrete example: Complexity Assessment**
```
State: {goal: "Fix login bug", evidence: ["Error in auth.py line 42"]}

Questions: {
  complexity: {
    type: "score",
    instructions: "How complex is the next step?",
    criteria: ["Mechanical", "Multi-step", "Ambiguous or high-risk"]
  }
}

Response: {
  complexity: {
    score: 1,  // "Multi-step"
    distribution: [0.15, 0.72, 0.13],
    confidence: 0.72
  }
}
```

**When to use:** You need to rank or position something on an ordered scale.

---

### Primitive 3: Noul — Yes/No Probability

**What it does:** Returns a probability (0 to 1) that a statement is true.

**Concrete example: Human Approval Required?**
```
State: {goal: "Fix login bug", action: "Delete user table"}

Questions: {
  needs_human: {
    type: "noul",
    instructions: "Does the next step require human approval?"
  }
}

Response: {
  needs_human: {
    noul: 0.92  // 92% probability yes, human approval needed
  }
}
```

**When to use:** You need the probability of one specific condition.

---

### Rule: Keep Questions Atomic

If a question mixes urgency, risk, and relevance, **split it into three questions** and combine their outputs in code.

```python
priority = (
    0.50 * relevance +
    0.30 * urgency +
    0.20 * source_quality
)
if priority < 1.20:
    route = 'defer'
```

---

## 3. Key Advantage: Batching Decisions

### The Core Insight

**13 LLM calls → 13 sequential decisions**

**1 Jev call → 13 decisions**

When multiple questions share the same state, they can be evaluated in parallel in a single call.

### The Numbers (from TypeSafe's GDPR experiment)

| Metric | 13 Separate Calls | 13 Questions in 1 Call | Advantage |
|--------|-------------------|------------------------|-----------|
| API Calls | 13 | 1 | 13x fewer |
| Latency | 2.71s | 0.27s | 10.0x faster |
| Cost | $0.00609 | $0.000497 | 12.2x cheaper |

**Test conditions:** 53,777-character GDPR article, 13 independent questions evaluated in parallel.

### Why This Works

1. **Questions share state:** All questions see the same evidence
2. **Questions are independent:** No question depends on another's answer
3. **One model pass:** Jev evaluates all questions simultaneously
4. **Structured output:** No prose generation overhead

### Important Caveat

If question B needs the answer to question A, it is a **second decision step**. Ask A, execute or transform the state, then ask B against fresh evidence.

---

## 4. Where Jev Sits in Architecture

### The Practical Architecture

```
Current State
      ↓
     Jev
      ↓
choose tool / model / agent / escalation
      ↓
   execute
      ↓
 New State
      ↓
     Jev
```

### The Decision Loop

```
OBSERVE → DECIDE → GATE → EXECUTE → VERIFY → REPEAT
```

### What Jev Should NOT Replace

| Keep in Code | Keep in LLMs | Keep in Humans |
|-------------|--------------|----------------|
| Math/computation | Writing/reasoning | Irreversible decisions |
| Deterministic logic | Creative generation | Policy exceptions |
| Schema validation | Open-ended analysis | Complex tradeoffs |
| Permission checks | Summarization | Final approval |

### The Rule

> **Jev decides among allowed options. Code validates schema, threshold, permission and budget.**

---

## 5. Is Jev Genuinely New?

### Separating Evidence from Hype

#### Demonstrated (publicly verified)

1. **Batching works:** 13 questions in 1 call = 10x faster, 12.2x cheaper
2. **Browser optimization:** Browser Use hit Google Flights in 7.1s (1,092 → 101 browser calls)
3. **Writing quality:** Every ran 777 checks in under 0.7s at ~$0.0025
4. **Skill routing:** Hermes wrong skill loads dropped from 16.8% → 7.3%
5. **Legal retrieval:** Top-10 accuracy improved from 38% → 62%
6. **Mobile:** Uber flow reached payment in ~21s / 9 actions

#### Vendor-reported (not independently verified)

1. **Speed claims:** "193x faster than Claude Fable 5.1"
2. **Cost claims:** "444x cheaper than GPT-6 Astra"
3. **Latency:** Median 0.35s per passage vs 8.83s for Fable 5.1

#### Still speculative

1. **Universal applicability:** Works best for bounded, repeated decisions
2. **Quality parity:** Fable caught 7/7 defects; Jev caught 6/7
3. **Total workflow cost:** Retriever AI saw cost rise 38-51% despite faster decisions

### The Honest Assessment

**Jev is not a new kind of AI.** It is a **decision layer** optimized for:

- Bounded choice spaces
- Repeated judgments
- Structured output consumption

**The real value:** Making decisions cheap enough to run on *every* message, *every* candidate, *every* draft, *every* tool call — not just sampled expensive cases.

---

## 6. Technical Internals

### How Jev Produces Decisions

1. **Typed questions** are sent as structured JSON (not prose)
2. **State** is application snapshot: text, object, or array
3. **Response** contains:
   - `answers` object keyed by question names
   - Choice: `choice` + `probabilities` + `confidence`
   - Score: `score` + `distribution` + `confidence`
   - Noul: `noul` (probability 0-1)

### Why Multiple Questions Are Efficient

```
POST /v1/systemone
{
  "model": "jev-latest",
  "state": { ... },
  "questions": {
    "route": {"type": "choice", ...},
    "risk": {"type": "score", ...},
    "approve": {"type": "noul", ...}
  }
}
```

- One model forward pass evaluates all questions
- Questions share the same context window
- No prose generation — only structured probability distributions

### Calibration

1. **Split labeled examples** into development and holdout sets
2. **Calculate quality and automation share** at several thresholds on development data
3. **Select the lowest threshold** that meets required error rate
4. **Freeze it**, evaluate on holdout, store result beside policy version
5. **Repeating threshold tuning on holdout turns it into training data** — don't do this

### Important Failure Modes

| Failure Mode | Cause | Solution |
|-------------|-------|----------|
| Wrong but confident | Confidence ≠ accuracy | Measure thresholds on labeled tasks |
| Stale action menus | Elements disappear after state changes | Rebuild options from fresh state |
| Choice used as multi-label | Choice forces one winner | Use independent Nouls, combine in code |
| DONE without proof | Model says "done" but action failed | Verify external state, not model claim |
| Too much state | Large irrelevant context | Retrieve, filter, then decide |

---

## 7. Limitations & Failure Modes

### What Breaks Jev

1. **Counting, math, date arithmetic:** Compute exact values in code
2. **Too much or untrusted state:** Retrieve and filter before deciding
3. **Wrong but confident:** Confidence is not accuracy
4. **Stale action menus:** Rebuild after every state change
5. **Choice as multi-label:** Use independent Nouls instead
6. **DONE without proof:** Verify external result, not model claim

### The Safety Boundary

```python
# The model may propose a branch
# Code decides whether that branch is allowed to execute
if route.confidence < minimum[route.choice]:
    destination = "human"
```

**Never let a probability bypass:**
- Permissions
- Budgets
- Rate limits
- Irreversible-action controls

### Deployment Checklist

- [ ] Define one repeated bounded decision
- [ ] Collect labeled examples (including adversarial cases)
- [ ] Specify state schema, remove irrelevant context
- [ ] Write atomic questions with explicit criteria
- [ ] Offer only live, authorized actions + abstain/human
- [ ] Measure quality, calibration, latency, cost, escalation share
- [ ] Set different thresholds for different consequences
- [ ] Pin the model, log complete decision trace
- [ ] Execute one reversible action, then read fresh state
- [ ] Verify observable result, test recovery

---

## 8. Practical Implementation

### Step 1: Setup

```bash
# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
.\.venv\Scripts\Activate.ps1  # Windows

# Install SDK
pip install typesafe-sdk

# Set API key
export TYPESAFE_API_KEY=your_key  # Linux/Mac
$env:TYPESAFE_API_KEY='your_key'  # Windows
```

### Step 2: First Jev Call

```python
from typesafe_sdk import Choice, Noul, TypeSafeClient

client = TypeSafeClient()
r = client.system_one(
    state={"message": "Need this fixed today"},
    questions={
        "owner": Choice(
            instructions="Which team should own this?",
            criteria={
                "ops": "Operational work",
                "engineering": "Software defect"
            }
        ),
        "urgent": Noul(
            instructions="Is same-day action required?"
        )
    }
)
print(r.answers["owner"].choice)
print(r.answers["urgent"].noul)
```

### Step 3: Build Before Jev Access (Adapter)

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

r = client.system_one(
    state=state,
    questions=questions,
    provider="openai",
    model="gpt-4o-mini"
)
```

### Step 4: Swap to Jev

```python
# Keep questions and state construction behind one function
# Swap only the client layer

# Before (adapter)
client = SystemOneAdapterClient(...)
r = client.system_one(state, questions, provider="openai", model="gpt-4o-mini")

# After (Jev)
client = TypeSafeClient()
r = client.system_one(state, questions, model="jev-latest")
```

**Important:** Recalibrate thresholds — probability distributions from adapter and Jev are not interchangeable.

---

## 9. Real-World Builds

### Build 1: Chief of Staff Router

```
State: Request, active projects, live workers, permissions, last artifact
Questions: Choice for owner; Score for urgency; Noul for ask-user
Action: Enqueue one handoff
Verify: Destination accepted job AND artifact path exists
```

### Build 2: Model Router

```
State: Task shape, budget, latency target, available models
Questions: Choice among fast/powerful/human; Score for complexity
Action: Call selected model for full run
Verify: Validate schema + task-specific quality checks
```

### Build 3: Email Firewall

```
State: Message, sender history, thread, communication policy
Questions: Choice reply/wait/junk; Nouls for urgency, impersonation, approval
Action: Tag, research, or draft
Verify: Human approves any send or sensitive action
```

### Build 4: Browser Action Selection

```
State: Goal, visible page elements, current URL, recent actions
Questions: Choice for operation + compatible target; Noul for completion
Action: Execute one validated action ID
Verify: Re-observe page, independently test success
```

### Build 5: Safety Gate

```
State: Proposed command, caller, permissions, policy
Questions: Noul for allowed; Score for harm; Choice run/block/ask
Action: Enforce policy branch in code
Verify: Record effect + immutable audit event
```

---

## 10. Quick Reference

### The Decision Table

| Action | Starting Gate | Fallback |
|--------|--------------|----------|
| Internal label | 0.70 | Accept |
| Agent handoff | 0.80 | Re-evaluate |
| Browser/tool action | 0.90 | Ask human |
| Publish/pay/delete | Policy + human | Block |

### What to Automate First vs. Later

| Automate First | Automate After Evaluation | Keep Out of Jev |
|----------------|---------------------------|-----------------|
| Routing, tagging, ranking | Tool gates, browser actions | Exact math |
| Relevance, repeated quality | Agent handoffs, completion | Date arithmetic |
| Low consequence, high frequency | Fresh state + thresholds required | Free-form writing |

### The Primitives Cheat Sheet

| Primitive | Returns | Use When |
|-----------|---------|----------|
| Choice | One option + probabilities + confidence | Exactly one option must win |
| Score | Numeric score + distribution + confidence | Rank on ordered rubric |
| Noul | Probability 0-1 | Yes/no probability needed |

### The Execution Boundary

```
Jev decides among allowed options
  ↓
Code validates schema, threshold, permission, budget
  ↓
Executor performs exactly one observable action
  ↓
Verifier reads external system again
  ↓
New observation becomes next state
```

### The Formula

> **Code computes. LLMs create. Jev decides. Fresh state proves the result.**

---

## Appendix: Key Sources

- TypeSafe SDK: `pip install typesafe-sdk`
- System One Adapter: `pip install 'system-one-adapter[openai]'`
- Agent Skill: `npx skills add typesafe-ai/skills --skill typesafe-ai`
- Browser Use Jev: Open-source browser agent with Jev integration
- Mobile Jev: Android agent with screen hierarchy parsing

---

## 11. System Design Example: Customer Support Triage Agent

### The Problem

A SaaS company receives 500 support tickets per day. Each ticket needs:

- Classification (bug, billing, feature request, how-to, security)
- Urgency scoring (critical, high, normal, low)
- Owner assignment (engineering, billing, support, security team)
- Escalation check (needs human review?)
- Response draft decision (auto-reply vs. human writes)

### The Traditional Approach (Without Jev)

```
Ticket arrives
  ↓
LLM Call 1: Classify ticket type
  ↓
LLM Call 2: Score urgency
  ↓
LLM Call 3: Assign owner
  ↓
LLM Call 4: Check escalation
  ↓
LLM Call 5: Decide response strategy
  ↓
Total: 5 sequential LLM calls per ticket
500 tickets × 5 calls = 2,500 LLM calls/day
```

**Bottleneck:** Each ticket serializes 5 decisions. Latency compounds. Cost scales linearly.

---

### The Jev Architecture

```
                    ┌─────────────────────────────┐
                    │       TICKET ARRIVES         │
                    │  (text, sender, metadata)    │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │      BUILD STATE             │
                    │  - ticket content            │
                    │  - sender history            │
                    │  - account tier              │
                    │  - available teams           │
                    │  - business hours flag       │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │     ONE JEVS CALL            │
                    │  5 parallel questions:       │
                    │                              │
                    │  Choice: classification      │
                    │  Score: urgency              │
                    │  Choice: owner assignment    │
                    │  Noul: escalation needed?    │
                    │  Noul: auto-reply safe?      │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │    GATE + VALIDATE           │
                    │  - confidence thresholds     │
                    │  - permission checks         │
                    │  - budget limits             │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼─────────┐ ┌───────▼───────┐ ┌─────────▼─────────┐
    │   AUTO-REPLY      │ │  ROUTE TO     │ │   ESCALATE TO     │
    │   (low urgency,   │ │  TEAM QUEUE   │ │   HUMAN           │
    │    high confidence)│ │  (confirmed)  │ │   (low confidence │
    │                   │ │               │ │    or high risk)  │
    └─────────┬─────────┘ └───────┬───────┘ └─────────┬─────────┘
              │                    │                    │
              └────────────────────┼────────────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   VERIFY + LOG              │
                    │  - ticket routed correctly?  │
                    │  - response sent?            │
                    │  - audit trail complete      │
                    └─────────────────────────────┘
```

---

### Where Jev Shines in This System

#### Decision 1: Classification (Choice)

**Without Jev:** LLM generates "This is a billing question" as prose, then code parses it.

**With Jev:** Returns `{choice: "billing", probabilities: {bug: 0.05, billing: 0.82, feature: 0.08, howto: 0.03, security: 0.02}, confidence: 0.82}`

**Why it's better:** Structured output. No parsing. Full distribution visible. Can see if it's ambiguous (close probabilities between options).

---

#### Decision 2: Urgency Scoring (Score)

**Without Jev:** LLM writes "This seems urgent because..." — code has to extract a number.

**With Jev:** Returns `{score: 2, distribution: [0.05, 0.12, 0.68, 0.15], confidence: 0.68}` where scale is [critical, high, normal, low].

**Why it's better:** Numeric score ready for threshold logic. Distribution shows certainty. No text parsing.

---

#### Decision 3: Owner Assignment (Choice)

**Without Jev:** Separate LLM call to decide routing.

**With Jev:** Evaluated in parallel with classification and urgency. Same state. One call.

**Why it's better:** The classification result doesn't need to be "read" by a separate model. Jev evaluates all questions against the same evidence simultaneously.

---

#### Decision 4: Escalation Check (Noul)

**Without LLM:** Binary rule: "If urgency > 2, escalate" — too rigid, misses nuance.

**Without Jev (LLM):** "I think this might need human review because..." — prose, slow.

**With Jev:** Returns `{noul: 0.23}` — 23% probability needs human. Code applies threshold: if > 0.70, escalate.

**Why it's better:** Probabilistic, not binary. Threshold tunable by consequence. Fast.

---

#### Decision 5: Auto-Reply Safety (Noul)

**Without Jev:** Either always auto-reply (risky) or never auto-reply (wastes human time).

**With Jev:** Returns `{noul: 0.91}` — 91% safe to auto-reply. Threshold: if > 0.85, auto-reply; else, human writes.

**Why it's better:** Reduces human workload on safe tickets. Escalates genuinely uncertain ones.

---

### The Batching Win

```
Traditional: 5 LLM calls × 2s each = 10s per ticket
Jev:         1 call × 0.3s = 0.3s per ticket

500 tickets/day:
  Traditional: 500 × 10s = 83 minutes of LLM time
  Jev:         500 × 0.3s = 2.5 minutes of LLM time

Cost comparison (estimated):
  Traditional: 2,500 calls × $0.002 = $5.00/day
  Jev:         500 calls × $0.0005 = $0.25/day
```

---

### The State Design

```json
{
  "ticket": {
    "id": "T-4521",
    "content": "Login fails with 403 after password reset",
    "sender": "user@company.com",
    "account_tier": "enterprise",
    "created_at": "2026-09-19T14:30:00Z"
  },
  "sender_history": {
    "tickets_last_30d": 2,
    "avg_satisfaction": 4.2,
    "previous_escalations": 0
  },
  "available_teams": ["engineering", "billing", "support", "security"],
  "business_hours": true,
  "current_queue_depths": {
    "engineering": 12,
    "billing": 3,
    "support": 8,
    "security": 1
  }
}
```

**Key design choices:**
- Include live queue depths so Jev can balance load
- Include sender history so Jev can weight urgency
- Include business hours so escalation logic has context
- Keep it small — only evidence that changes the decision

---

### The Confidence Gates

```
Classification confidence < 0.60 → route to human reviewer
Urgency score = critical (0)     → skip queue, immediate engineering
Escalation noul > 0.70           → human writes response
Auto-reply noul < 0.85           → human writes response
Owner confidence < 0.70          → default to support team
```

**The rule:** Different thresholds for different consequences. Low-risk auto-route at 0.70. High-risk require 0.90+. Irreversible actions require human.

---

### What Stays Outside Jev

| Component | Owner | Why |
|-----------|-------|-----|
| Ticket parsing | Code | Deterministic extraction |
| Response drafting | LLM | Creative generation needed |
| Send email | Code | Irreversible, needs permission |
| SLA tracking | Code | Math/dates |
| Satisfaction survey | Code | Deterministic trigger |
| Audit logging | Code | Deterministic write |

---

### The Verification Loop

After Jev routes a ticket:

1. **Check the queue:** Did the ticket actually appear in the target queue?
2. **Check the response:** If auto-replied, was it sent successfully?
3. **Check the state:** Read fresh state to confirm the action took effect
4. **If verification fails:** Retry once, then escalate to human

**Never trust the model's "done" signal. Verify the external system.**

---

### Why This Design Works

1. **One state, five decisions:** All questions see the same evidence
2. **Parallel evaluation:** No sequential dependency between questions
3. **Structured output:** Code reads numbers, not parses prose
4. **Tunable thresholds:** Business rules live in code, not prompts
5. **Auditable:** Full probability distribution logged for every decision
6. **Graceful degradation:** Low confidence → human fallback
7. **Fresh state:** After every action, re-observe before next decision

---

### When to Apply This Pattern

This same architecture works for:

- **Email triage:** Classify, prioritize, route, draft decision
- **Content moderation:** Classify, severity, action, escalation
- **Lead scoring:** Fit score, timing, intent, owner assignment
- **Code review routing:** Complexity, risk, reviewer assignment
- **Document processing:** Type extraction, validation, routing

**The signal:** If you're making 3+ bounded decisions on the same state, batch them with Jev.

---

*Last updated: September 2026*
*Keep this guide narrow. If a piece of information does not improve your understanding of what Jev is, why it matters, how it works, or where it fits, leave it out.*
