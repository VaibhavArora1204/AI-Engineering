# DeepSeek V4 Flash: Architecture and Systems Guide for AI Engineers

> **Audience:** AI engineers who build systems on top of LLMs.
> **Goal:** Understand *what* DeepSeek V4 Flash does, *why* each design choice exists, and *which ideas transfer* to the systems you build.
> **Non-goals:** Deriving attention equations, implementing CUDA kernels, or learning inference-engine internals.

---

## Table of Contents

1. [Executive Overview](#1-executive-overview)
2. [Architectural Map](#2-deepseek-v4-flash-architectural-map)
3. [Mixture of Experts](#3-mixture-of-experts)
4. [Sparse Activation](#4-sparse-activation)
5. [Expert Routing](#5-expert-routing)
6. [Efficient Attention and Long Context](#6-efficient-attention-and-long-context)
7. [DeepSeek V4's Long-Context Architecture](#7-deepseek-v4s-long-context-architecture)
8. [Information Compression](#8-information-compression)
9. [FP4 / FP8 and Low Precision](#9-fp4--fp8-and-low-precision)
10. [Multi-Token Prediction](#10-multi-token-prediction)
11. [Architecture × System Design](#11-architecture--system-design)
12. [Ideas You Can Steal](#12-ideas-you-can-steal)
13. [What NOT to Copy](#13-what-not-to-copy)
14. [Conventional vs Modern Architecture Thinking](#14-conventional-vs-modern-architecture-thinking)
15. [A Practical AI Engineer's Mental Model](#15-a-practical-ai-engineers-mental-model)
16. [Glossary](#16-glossary)
17. [Recommended Reading](#17-recommended-reading)

---

## 1. Executive Overview

DeepSeek V4 Flash is a **sparse Mixture-of-Experts (MoE) language model** released in April 2026. Here is what you need to know in one breath:

| Dimension | Value |
|---|---|
| **Total parameters** | 284 billion |
| **Active parameters per token** | 13 billion |
| **Context window** | 1 million tokens |
| **Expert architecture** | DeepSeekMoE (256 routed + 1 shared) |
| **Attention** | Hybrid CSA + HCA + local sliding window |
| **Weight precision** | FP4 (experts) / FP8 (attention, norms, routers) |
| **Training data** | 32 T+ tokens |
| **Optimizer** | Muon |
| **Reasoning modes** | Non-think, Think High, Think Max |

### What is this model actually trying to optimize?

DeepSeek V4 Flash pursues five goals simultaneously, and understanding how they interact is the key mental model:

```
┌──────────────────────────────────────────────────────┐
│              DeepSeek V4 Flash Goals                  │
│                                                      │
│  1. Massive capacity, tiny active compute            │
│     284B params → only 13B fire per token            │
│                                                      │
│  2. Million-token context without linear cost growth │
│     Hybrid attention compresses KV cache ~90%        │
│                                                      │
│  3. Extreme memory efficiency                        │
│     FP4 experts + FP8 everything else                │
│                                                      │
│  4. Faster generation                                │
│     Multi-token prediction → speculative decoding    │
│                                                      │
│  5. Training stability at scale                      │
│     Manifold-constrained hyper-connections (mHC)     │
└──────────────────────────────────────────────────────┘
```

These goals **interact**:

- MoE enables massive capacity cheaply, but MoE layers are memory-hungry → **FP4 quantization** solves memory.
- Million-token context generates enormous KV caches → **hybrid CSA/HCA attention** compresses them.
- Deep MoE stacks are unstable to train → **mHC residual connections** keep gradients healthy.
- Sequential token generation is the latency bottleneck → **MTP** enables generating multiple tokens per step.

**Engineering takeaway:**
- DeepSeek V4 Flash is *not* just a bigger model. It is a *system of co-designed solutions* where each component enables or compensates for another.
- The core philosophy: **maximize useful capacity per unit of compute, memory, and latency.**

---

## 2. DeepSeek V4 Flash Architectural Map

```
                        Input Tokens
                             │
                             ▼
                    ┌─────────────────┐
                    │    Embedding     │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │     Transformer Block (×N)   │
              │                              │
              │  ┌────────────────────────┐  │
              │  │   Hybrid Attention      │  │
              │  │  ┌──────┬──────┬─────┐ │  │
              │  │  │ CSA  │ HCA  │Local│ │  │
              │  │  │(4:1) │(128:1)│ win │ │  │
              │  │  └──────┴──────┴─────┘ │  │
              │  └───────────┬────────────┘  │
              │              │               │
              │      mHC Residual Stream     │
              │      (4 parallel paths)      │
              │              │               │
              │  ┌───────────┴────────────┐  │
              │  │    DeepSeekMoE Layer    │  │
              │  │                        │  │
              │  │  ┌──────────────────┐  │  │
              │  │  │  Shared Expert   │  │  │  ← Always active
              │  │  │  (1 expert)      │  │  │
              │  │  └──────────────────┘  │  │
              │  │           +            │  │
              │  │  ┌──────────────────┐  │  │
              │  │  │  Router → Top-8  │  │  │  ← Selects 8 of 256
              │  │  │  Routed Experts  │  │  │
              │  │  └──────────────────┘  │  │
              │  └───────────┬────────────┘  │
              │              │               │
              │      mHC Residual Stream     │
              │              │               │
              └──────────────┬──────────────┘
                             │
                             ▼ (repeat N blocks)
                    ┌─────────────────┐
                    │   Output Head    │
                    │  + MTP Heads     │
                    └─────────────────┘
```

**Precision map:**

```
Component              Precision    Why
─────────────────────  ─────────    ─────────────────────────────
MoE expert weights     FP4          Largest memory consumer
Attention weights      FP8          Needs more fidelity
Router weights         FP8          Routing decisions are critical
Normalization          FP8          Small footprint anyway
```

---

## 3. Mixture of Experts

### What is an expert?

An expert is a feed-forward network (FFN)—the same type of component that already exists inside every Transformer block. In a dense Transformer, there is one FFN per block. In an MoE Transformer, that single FFN is replaced by *many* FFNs (the experts), and a router decides which ones to use for each token.

### How MoE works

```
                        Token
                          │
                          ▼
                    ┌───────────┐
                    │  Router   │  Scores all 256 experts
                    └─────┬─────┘
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
     Expert 12       Expert 87      Expert 203    ← Top-8 selected
     (coding?)       (math?)        (language?)
            │             │             │
            └─────────────┼─────────────┘
                          ▼
                  Weighted Sum → Output
```

Plus: the **shared expert** (always active, handles common knowledge).

### DeepSeek V4 Flash specifics

| Component | Detail |
|---|---|
| Routed experts | 256 fine-grained experts |
| Shared experts | 1 (always active) |
| Experts selected per token | 8 |
| Total MoE parameters | ~271B (bulk of 284B total) |
| Active MoE parameters per token | ~13B |

### Why fine-grained experts?

DeepSeek deliberately uses **many small experts** instead of fewer large ones. This gives the router more precise control over what computation to apply. Think of it as the difference between choosing from 8 large departments vs. choosing from 256 specialized consultants.

### Total parameters vs active parameters

```
DeepSeek V4 Flash:
┌─────────────────────────────────────────┐
│ Total capacity: 284B parameters         │
│ ████████████████████████████████████████ │
│                                         │
│ Active per token: 13B parameters        │
│ █████                                   │
│                                         │
│ Utilization ratio: ~4.6%                │
└─────────────────────────────────────────┘
```

This is not waste. The 284B parameters represent the model's total *knowledge*. The 13B active parameters represent the *computation applied to a specific token*. Different tokens activate different experts, so the full 284B is used across the entire workload.

### Why should an AI engineer care?

MoE is a concrete implementation of **conditional computation**: the idea that not every input requires the same processing. This principle generalizes far beyond LLM internals:

| MoE Concept | Application-Level Analog |
|---|---|
| Expert = specialized FFN | Specialist model or service |
| Router = learned gating | Intent classifier, complexity scorer |
| Sparse activation | Only invoke what you need |
| Shared expert | Base model handling common cases |
| Load balancing | Request distribution across services |

**Concrete examples:**

1. **Agent routing:** A customer-service agent routes HR questions to an HR-specialist LLM and refund questions to a refund-specialist LLM. Same principle as expert routing.
2. **Model cascade:** A cheap model handles simple queries; a powerful model handles hard ones. Same principle as sparse activation.
3. **RAG pipeline selection:** Different retrieval strategies for different query types. Same principle as conditional computation.

**Engineering takeaway:**
- MoE proves that you can scale capacity without scaling per-request compute.
- The router is the critical component. A good router makes sparse capacity useful; a bad router wastes it.
- Fine-grained specialization (many small experts) often outperforms coarse specialization (few large experts).
- The shared-expert pattern (always-on baseline + specialized routing) is directly applicable to multi-model systems.

---

## 4. Sparse Activation

### The key idea

**Capacity and computation do not have to scale together.**

```
Model A (Dense):                    Model B (Sparse MoE):
┌──────────────────────┐            ┌──────────────────────────────────┐
│ 100B parameters      │            │ 300B parameters                  │
│ 100B active per token│            │ 30B active per token             │
│                      │            │                                  │
│ Compute: ████████████│            │ Compute: ███                     │
│ Capacity: ██████████ │            │ Capacity: ██████████████████████ │
└──────────────────────┘            └──────────────────────────────────┘

Model B has 3× the knowledge with 0.3× the compute per token.
```

### Why this matters

In a dense model, every parameter participates in every computation. If you want more knowledge, you need more parameters, which means more compute, more memory, more latency, more cost. They are all coupled.

Sparse activation **decouples** them:

```
Dense model scaling:
Knowledge ↑  →  Compute ↑  →  Latency ↑  →  Cost ↑
   (all coupled)

Sparse model scaling:
Knowledge ↑  →  Compute ≈  →  Latency ≈  →  Cost ≈
   (decoupled via routing)
```

### DeepSeek V4 Flash in context

```
Model                Total Params    Active Params    Ratio
───────────────────  ────────────    ─────────────    ─────
GPT-4 (est.)        ~1.8T           ~280B            ~15%
DeepSeek V4 Pro     1.6T            49B              ~3%
DeepSeek V4 Flash   284B            13B              ~4.6%
Llama 3.1 405B      405B            405B             100% (dense)
```

### System design implications

The sparse activation principle transfers directly to how you design AI systems:

**Instead of:**
```
Every request → Same large model → Same cost
```

**Think:**
```
Request → Complexity assessment
              │
              ├── Simple → Small/cheap model
              ├── Medium → Medium model
              └── Hard   → Large/expensive model
```

You are implementing sparse activation at the *system* level. Most requests are simple. If 80% of your traffic can be handled by a model that costs 10× less, you have achieved a massive efficiency gain—the same principle that makes MoE work.

**Engineering takeaway:**
- Sparse activation is the single most important scaling idea in modern LLMs.
- It decouples knowledge from per-request compute cost.
- You can apply this principle at the application level by routing requests to different models based on complexity.
- The efficiency comes from the observation that most tokens/requests don't need the full capacity of the system.

---

## 5. Expert Routing

### Why routing exists

Without routing, you have two options: run everything (dense, expensive) or pick randomly (cheap, useless). Routing is the mechanism that makes sparse activation *intelligent*. It decides which experts to activate for which tokens.

### How specialization emerges

Experts are **not** pre-assigned to topics. They specialize *during training* through a feedback loop:

```
Training step 1000:
  Token "def" → Router sends to Expert 42 → Expert 42 gets good at code

Training step 10000:
  Token "def" → Router reliably sends to Expert 42
  Expert 42 is now a code specialist
  Expert 42 also handles "class", "return", "import"

Training step 100000:
  Expert 42: Python syntax specialist
  Expert 87: Mathematical reasoning
  Expert 203: Multilingual text
  Expert 15: General knowledge (activated frequently)
```

### Why routing quality matters

```
Good routing:                        Bad routing:
Token → Best expert                  Token → Random expert
Full capacity utilized               Capacity wasted
Specialists stay specialized         Experts become generic duplicates
```

### Routing failure modes

| Problem | What happens | System-level analog |
|---|---|---|
| **Expert collapse** | Most tokens route to a few experts; rest are dead weight | 90% of traffic hitting one microservice |
| **Load imbalance** | Some experts overloaded, others idle | Uneven load balancing |
| **Redundancy** | Multiple experts learn the same thing | Duplicate services |
| **Communication overhead** | Routing adds latency | Extra network hop for service discovery |

### Why shared experts help

The shared expert solves a specific problem: **redundant knowledge across routed experts.**

```
Without shared expert:
  Expert 1: grammar + specialty_A
  Expert 2: grammar + specialty_B
  Expert 3: grammar + specialty_C
  ← grammar knowledge duplicated 256 times

With shared expert:
  Shared:   grammar (always active)
  Expert 1: specialty_A (pure specialization)
  Expert 2: specialty_B (pure specialization)
  Expert 3: specialty_C (pure specialization)
  ← grammar knowledge stored once, more room for specialization
```

### What can I steal from this idea?

The routing concept maps directly to application architecture:

```
          User request
               │
               ▼
        ┌──────────────┐
        │ Intent Router │  (classifier, LLM judge, rule engine)
        └──────┬───────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
   HR        Policy     Refund
  Agent      Agent      Agent
    │          │          │
  HR LLM   Policy LLM  Refund LLM
```

**MoE concept → Application pattern:**

| MoE Routing | Application Routing |
|---|---|
| Token → router → experts | Request → classifier → specialist models |
| Shared expert (always on) | Base model handling all requests, augmented by specialists |
| Load balancing across experts | Traffic management across model endpoints |
| Fine-grained experts | Narrow, focused agents or tools |
| Expert collapse | One model endpoint receiving all traffic |

**Application-level routing examples:**

1. **RAG routing:** Route factual questions to a retrieval pipeline, creative questions to a generative pipeline, code questions to a code-search pipeline.
2. **Model routing:** Simple queries → small fast model. Complex reasoning → large reasoning model. Code → code-specialized model.
3. **Tool routing:** Agent decides which tool to call based on the task—this is literally a learned router.
4. **Cost-aware routing:** High-value customers → premium model. Batch processing → cheap model.

**Important distinction:** Architectural MoE routing happens *inside* the model, at the token level, in microseconds. Application-level routing happens *outside* the model, at the request level, in milliseconds. They are the same *principle* but different implementations. Don't confuse them.

**Engineering takeaway:**
- Routing is what makes specialization work. Without intelligent routing, specialists are wasted.
- Shared + routed is a powerful pattern: always-on baseline + conditional specialization.
- Monitor for routing failure modes: collapse (everything goes to one endpoint), imbalance, redundancy.
- Keep your router simple and fast. A slow router defeats the purpose of conditional computation.
- The router's quality is often more important than any individual specialist's quality.

---

## 6. Efficient Attention and Long Context

### The practical problem

Standard attention has a fundamental scaling problem:

```
Context length    KV cache memory     Attention compute
───────────────   ──────────────────  ──────────────────
4K tokens         Manageable          Fast
32K tokens        Significant         Noticeable
128K tokens       Expensive           Slow
1M tokens         Impossible*         Prohibitive*

* without architectural intervention
```

**Why 1M tokens is an engineering crisis:**

1. **Memory:** The KV cache stores key-value pairs for every token at every layer. At 1M tokens, this can exceed hundreds of GB.
2. **Compute:** Standard attention is O(n²) in sequence length. 1M tokens means 1 trillion attention operations per layer.
3. **Latency:** Every new token must attend to all previous tokens. At 1M tokens, each generation step is slow.
4. **Cost:** All of the above translates to dollars per request.

### The evolution of attention efficiency

```
Original Multi-Head Attention (MHA)
  │  Every head has its own K and V
  │  Full memory cost
  ▼
Multi-Query Attention (MQA)
  │  All heads share one K and one V
  │  Dramatic memory reduction, some quality loss
  ▼
Grouped-Query Attention (GQA)
  │  Groups of heads share K and V
  │  Balance between MHA and MQA
  ▼
Multi-Head Latent Attention (MLA)  ← DeepSeek V2/V3
  │  Compress K,V into low-dimensional latent vectors
  │  Store compressed representation, expand when needed
  │  Better memory than GQA, better quality than MQA
  ▼
Hybrid CSA + HCA  ← DeepSeek V4
     Compress at multiple granularities
     Sparse selection of relevant context
     ~90% KV cache reduction vs MLA alone at 1M tokens
```

### The key questions for long context

Rather than understanding the math of attention, an AI engineer should think about information management:

| Question | Why it matters |
|---|---|
| What information *must* be retained? | Recent context, key facts, instructions |
| What can be *compressed*? | Repetitive patterns, background context |
| What can be *discarded*? | Irrelevant earlier conversation, padding |
| How is *relevant* information identified? | Learned routing, similarity search, recency |

Every modern long-context architecture answers these questions differently. DeepSeek V4 answers them with a multi-resolution compression strategy.

**Engineering takeaway:**
- Long context is fundamentally an information management problem, not just a compute problem.
- The evolution from MHA → MQA → GQA → MLA → CSA/HCA is a progression of increasingly aggressive compression.
- Each step trades some representational fidelity for massive efficiency gains.
- The same trade-off exists in application design: do you pass the entire conversation history, or summarize it?

---

## 7. DeepSeek V4's Long-Context Architecture

### The three-layer approach

DeepSeek V4 uses three complementary mechanisms to handle 1M tokens:

```
┌─────────────────────────────────────────────────────────┐
│                  Token Position                          │
│  ◄── recent ────────────── medium ────────────── old ──► │
│                                                          │
│  Layer 1: Local Sliding Window (128 tokens)              │
│  ████████                                                │
│  Full resolution, no compression                         │
│  Handles: immediate local dependencies                   │
│                                                          │
│  Layer 2: Compressed Sparse Attention (CSA)              │
│           ░░░░░░░░░░░░░░░░░░░░░░░░░░░░                  │
│           4 tokens → 1 compressed entry                  │
│           + Lightning Indexer selects relevant ones       │
│           Handles: medium-range retrieval                 │
│                                                          │
│  Layer 3: Heavily Compressed Attention (HCA)             │
│                    ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒  │
│                    128 tokens → 1 compressed entry        │
│                    Handles: global "bird's-eye" context   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Component details

#### Local Sliding Window

- **What:** Retains the most recent ~128 tokens at full, uncompressed resolution.
- **Why:** Local dependencies (the current sentence, recent instructions) are the most important and should not be degraded by compression.
- **Trade-off:** Limited range, but perfect fidelity.

**Status:** ✅ Confirmed in public documentation.

#### Compressed Sparse Attention (CSA)

- **What:** Compresses groups of 4 adjacent tokens into a single KV cache entry. Then uses a learned "Lightning Indexer" to select only the *relevant* compressed entries for each query.
- **Why:** Reduces memory by ~4× and makes attention sub-quadratic by skipping irrelevant context.
- **Trade-off:** Some information is lost in the 4:1 compression. The indexer might miss relevant context.

**Status:** ✅ Confirmed architecture. Specific compression ratio (4:1) confirmed in multiple sources.

#### Heavily Compressed Attention (HCA)

- **What:** Compresses groups of 128 adjacent tokens into a single KV cache entry. Provides a coarse, global view of the entire context.
- **Why:** At 1M tokens, even CSA's 4:1 compression produces 250K entries. HCA reduces this to ~8K entries for a fast global scan.
- **Trade-off:** Very aggressive compression loses fine-grained detail. This layer provides a "summary" of distant context, not a precise representation.

**Status:** ✅ Confirmed architecture. Compression ratio (128:1) confirmed.

#### How they work together

The model interleaves CSA and HCA layers throughout the network. This means information flows through multiple compression/expansion cycles:

```
Block 1:  CSA layer  → sees medium-resolution past
Block 2:  HCA layer  → sees low-resolution global context
Block 3:  CSA layer  → re-examines medium-resolution details
Block 4:  HCA layer  → updates global understanding
...

This interleaving prevents the uniform information loss
that would occur if only one compression level were used.
```

**Status:** ✅ Confirmed (interleaving of heterogeneous attention types). Exact interleaving pattern: 🟡 Reasonable interpretation based on published descriptions.

#### KV cache reduction

| Architecture | KV cache at 1M tokens (relative) |
|---|---|
| Standard MHA | 100% (baseline) |
| GQA | ~25% |
| MLA (DeepSeek V3) | ~10% |
| CSA + HCA (DeepSeek V4) | ~1% of MHA, ~10% of MLA |

**Status:** ✅ ~90% reduction vs predecessor confirmed by HuggingFace model card.

#### Lightning Indexer

The Lightning Indexer is a learned routing mechanism within CSA. Rather than attending to all compressed entries (which would still be expensive at 1M tokens), it uses the current query to identify which compressed entries are likely to contain relevant information.

**Status:** ✅ Confirmed mechanism. Internal implementation details: 🔴 Unknown.

Think of it as a learned retrieval system embedded inside the attention mechanism. The model doesn't scan everything; it first identifies *where to look*, then looks there at higher resolution.

### AI Engineering Implications

DeepSeek V4's long-context design teaches several lessons for building systems:

**1. Multi-resolution context is better than flat context.**

```
Naive approach:                     DeepSeek's approach:
┌───────────────────────┐           ┌───────────────────────┐
│ Everything at full     │          │ Recent: full detail    │
│ resolution             │          │ Medium: compressed     │
│ → runs out of memory   │          │ Distant: heavily       │
│                        │          │   compressed           │
│                        │          │ → fits in memory       │
└───────────────────────┘           └───────────────────────┘
```

**Applies to:** Long-context RAG, conversation memory, document processing.

**2. "Just increase context window" is not a complete strategy.**

Simply cramming more tokens into a flat context is like storing every email you've ever received in RAM. The architecture needs to decide what to keep at full resolution, what to compress, and what to summarize.

**Applies to:** Agent memory systems, large codebase assistants, knowledge systems.

**3. Retrieval inside attention is a form of RAG.**

The Lightning Indexer inside CSA is conceptually similar to a retrieval step: given a query, find the most relevant information from a large corpus. DeepSeek has baked RAG-like behavior *into the attention mechanism itself*.

**Applies to:** Thinking about when to use external RAG vs. relying on long context.

**4. Compression ratios can vary by distance.**

Recent information deserves full fidelity. Distant information can tolerate heavy compression. This graduated approach is more efficient than uniform treatment.

**Applies to:** Conversation memory (recent turns verbatim, older turns summarized), document processing (current section full detail, other sections compressed).

**Engineering takeaway:**
- Multi-resolution context management is the key idea. Not everything needs full detail.
- Learned retrieval inside the model is philosophically identical to external retrieval (RAG).
- Design your context like DeepSeek designs its attention: recent=full, medium=compressed, distant=summarized.
- Simply increasing context window size doesn't solve the problem; you need a strategy for *what goes in it*.

---

## 8. Information Compression

This is one of the most important conceptual sections for AI engineers.

### The core principle

> **Modern AI systems increasingly try to avoid carrying every piece of information forward at full resolution.**

This idea appears everywhere in DeepSeek V4 Flash:

| Where | What gets compressed | How |
|---|---|---|
| MLA | Key-value pairs | Low-rank projection to latent vectors |
| CSA | Groups of 4 tokens | Learned compression |
| HCA | Groups of 128 tokens | Aggressive learned compression |
| FP4 weights | Expert parameters | Numerical precision reduction |
| MoE routing | Computation itself | Only activate relevant experts |

### Compression vs Retrieval vs Summarization vs Memory

These four concepts are related but distinct:

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  COMPRESSION                                         │
│  Take X, produce a smaller X' that preserves         │
│  the most important information.                     │
│  Example: MLA compresses KV pairs to latent vectors  │
│  Reversible? Partially (lossy compression)           │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  RETRIEVAL                                           │
│  Given a query, find the most relevant items         │
│  from a large collection.                            │
│  Example: Lightning Indexer selecting relevant       │
│  compressed entries; RAG finding relevant docs       │
│  Reversible? N/A (it's selection, not transformation)│
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  SUMMARIZATION                                       │
│  Take X, produce a natural-language summary          │
│  that captures key points.                           │
│  Example: Summarizing 50 conversation turns          │
│  into 3 sentences for context                        │
│  Reversible? No (destructive by design)              │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  MEMORY                                              │
│  Persistent storage of information across            │
│  interactions, in structured or unstructured form.   │
│  Example: Agent memory storing key facts,            │
│  user preferences, past decisions                    │
│  Reversible? Yes (it's storage)                      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Why DeepSeek's approach matters

DeepSeek V4 uses compression **at multiple levels simultaneously**:

```
Raw token sequence (1M tokens)
         │
         ▼
    ┌─────────┐
    │ Layer 1  │  Local window: no compression
    │          │  CSA: 4:1 compression + sparse retrieval
    │          │  HCA: 128:1 compression
    └────┬────┘
         │
    ┌─────────┐
    │ Layer 2  │  Different compression at each layer
    │          │  Information re-expanded and re-compressed
    └────┬────┘
         │
         ▼
    (repeat through all layers)
```

The critical insight: **compression and retrieval are complementary.** CSA compresses first (reducing memory), then retrieves selectively (reducing compute). This is the same pattern you should use in application design.

### Application-level compression patterns

```
Pattern 1: Progressive summarization
──────────────────────────────────────
Turn 1:     Full text
Turns 2-10: Recent summary + full text of last 3 turns
Turns 11+:  Compressed memory + recent summary + current turn

Pattern 2: Hierarchical context
──────────────────────────────────────
System prompt          → always present, full resolution
Retrieved context      → relevant chunks, medium resolution
Conversation summary   → compressed background
Current user message   → full resolution

Pattern 3: Semantic caching
──────────────────────────────────────
Query → Check semantic cache → Hit? Return cached result
                              → Miss? Full computation → Cache result
```

**Engineering takeaway:**
- Information compression is not optional at scale. It's an architectural necessity.
- Compression + retrieval is more powerful than either alone. Compress to reduce cost, retrieve to maintain relevance.
- Design your AI system's context pipeline as a compression pipeline: raw → filtered → compressed → relevant.
- Different information deserves different compression ratios. Recent and critical data stays full resolution.
- Every piece of context you pass to a model has a cost. Compression reduces that cost. But compression can destroy information—be intentional about what you compress.

---

## 9. FP4 / FP8 and Low Precision

### Why lower precision?

Every parameter in a model occupies memory. Lower numerical precision means each parameter takes fewer bits:

```
Precision   Bits per param   Memory for 284B params
─────────   ──────────────   ──────────────────────
FP32        32 bits          ~1,136 GB
BF16        16 bits          ~568 GB
FP8         8 bits           ~284 GB
FP4         4 bits           ~142 GB
```

DeepSeek V4 Flash stores expert weights in FP4 and everything else in FP8. This is why a 284B-parameter model can fit on hardware that would choke on a 70B dense model in FP16.

### What DeepSeek V4 Flash does

```
┌─────────────────────────────────────────────┐
│         DeepSeek V4 Flash Precision Map     │
│                                             │
│  MoE Expert Weights ────── FP4  (4 bits)    │
│  │                                          │
│  │  These are the bulk of the model.        │
│  │  256 routed + 1 shared expert.           │
│  │  ~95% of total parameters.              │
│  │  FP4 saves enormous memory.              │
│                                             │
│  Attention Weights ─────── FP8  (8 bits)    │
│  Router Weights ────────── FP8  (8 bits)    │
│  Normalization ─────────── FP8  (8 bits)    │
│  │                                          │
│  │  These components need more precision.   │
│  │  Routing decisions must be accurate.     │
│  │  Attention needs fine-grained control.   │
│  │  But they're a small fraction of params. │
│                                             │
└─────────────────────────────────────────────┘
```

### Why can't everything be FP4?

| Component | Can tolerate FP4? | Why / why not |
|---|---|---|
| Expert FFN weights | ✅ Yes | Large number of parameters provides redundancy; individual weight precision matters less |
| Router weights | ❌ No | Small weight changes can redirect tokens to completely different experts; needs precision |
| Attention weights | ❌ No | Attention scores control what information flows forward; precision matters for retrieval accuracy |
| Normalization | ❌ No | Small parameters that stabilize the entire computation; numerical stability critical |

### Why training matters

DeepSeek uses **Quantization-Aware Training (QAT)**, meaning the model is trained *knowing* it will run in FP4/FP8. This is fundamentally different from post-training quantization:

```
Post-training quantization:
  Train in FP16 → Quantize to FP4 → Hope it still works
  Result: Unpredictable quality loss

Quantization-Aware Training (QAT):
  Train with FP4 simulation → Model learns to be robust to low precision
  Result: Designed-in tolerance, ~99.7% recall maintained
```

### The precision roadmap

```
2020: BF16 is the standard
         │
2023: FP8 training emerges (H100 hardware)
         │
2025: FP4 for inference becomes practical (Blackwell hardware)
         │
2026: FP4 for MoE experts is native (DeepSeek V4)
         │
Future: Hardware-software co-design continues
        Precision matched to component role
```

### What this means for AI engineers

**1. Model selection is now a precision decision.**

```
Workload                     Recommendation
────────────────────────     ──────────────────────
Production, quality-critical  FP8 or BF16 model
High-volume, cost-sensitive   FP4/GPTQ quantized model
Local development             4-bit quantized (GGUF)
Experimentation               Whatever runs on your GPU
```

**2. Quantization is not free, but it's increasingly cheap.**

Models designed for low precision (like DeepSeek V4 Flash) lose very little quality. Models quantized *after* training can lose more. When choosing quantized models, prefer those trained with QAT.

**3. The cost implications are enormous.**

```
Same model, different precision:
FP16: needs 8× H100 GPUs      → $X/hour
FP8:  needs 4× H100 GPUs      → $X/2 per hour
FP4:  needs 2× H100 GPUs      → $X/4 per hour
      (or native on Blackwell)
```

**Engineering takeaway:**
- Lower precision is a first-class architectural decision, not a post-hoc optimization.
- Not all model components are equal: bulk parameters (experts) tolerate aggressive quantization; control-flow parameters (routers, attention) need more precision.
- QAT > post-training quantization for quality preservation.
- When selecting models for production, factor in precision: a QAT FP4 model may outperform a post-hoc quantized FP8 model.
- The memory savings from lower precision enable everything else: larger models, longer contexts, cheaper serving.

---

## 10. Multi-Token Prediction

### The basic problem

Standard LLMs generate one token at a time:

```
Standard autoregressive generation:

Step 1: "The"        → predict → "cat"
Step 2: "The cat"    → predict → "sat"
Step 3: "The cat sat"→ predict → "on"
Step 4: ...          → predict → ...

Each step requires a full forward pass.
For 1000 tokens, that's 1000 forward passes.
```

This is the fundamental latency bottleneck. Each step must wait for the previous one to complete.

### What Multi-Token Prediction (MTP) does

During **training**, the model learns to predict not just the next token, but also tokens at positions t+2, t+3, etc.:

```
Training with MTP:

Input: "The cat sat on the"

Main head:  predict position t+1 → "mat"      (primary loss)
MTP head 1: predict position t+2 → "and"      (auxiliary loss)
MTP head 2: predict position t+3 → "purred"   (auxiliary loss)

Total loss = main_loss + 0.1 × average(auxiliary_losses)
```

### Why MTP matters

MTP provides two distinct benefits:

**Benefit 1: Richer training signal.**

Standard next-token prediction gives the model a narrow view: "What comes immediately next?" MTP forces the model to develop a deeper understanding of the text because predicting token t+3 requires understanding the overall trajectory, not just local patterns.

```
Without MTP: Model learns local transitions
  "cat" → "sat"  (what word follows "cat"?)

With MTP: Model learns trajectory
  "The cat" → "sat on the mat and purred"
  (what is the story arc? what happens next?)
```

This makes the model better even when generating one token at a time.

**Benefit 2: Speculative decoding at inference.**

The MTP heads, trained to predict future tokens, serve as a built-in "draft model" for speculative decoding:

```
Standard generation:        Speculative generation (with MTP):

Step 1: predict token 1     Step 1: predict tokens 1, 2, 3 (draft)
Step 2: predict token 2     Step 2: verify all 3 in one forward pass
Step 3: predict token 3              Accept 2, reject 1
                             Step 3: predict tokens 3, 4, 5 (draft)
                             Step 4: verify all 3
                                      Accept all 3

3 steps → 3 tokens           2 verification steps → 5 tokens
```

Speculative decoding generates *candidate* tokens quickly (using the small MTP heads), then *verifies* them in a single forward pass of the full model. Verification is cheap because the model can check multiple tokens in parallel. This can increase generation throughput by 1.5–2×.

### DeepSeek's MTP design

DeepSeek's MTP implementation has a specific design choice: **sequential** prediction rather than parallel.

```
Parallel MTP (other approaches):
  Main model → Head 1 predicts t+1
             → Head 2 predicts t+2 (independently)
             → Head 3 predicts t+3 (independently)
  Problem: Each head ignores what the others predicted.

Sequential MTP (DeepSeek):
  Main model → Head 1 predicts t+1
                 ↓ (feeds into)
               Head 2 predicts t+2
                 ↓ (feeds into)
               Head 3 predicts t+3
  Benefit: Each prediction is conditioned on previous predictions.
           Maintains causal chain integrity.
```

**Status:** ✅ Sequential MTP design confirmed. Established in DeepSeek V3, carried forward to V4.

### System-level implications

| Aspect | Impact |
|---|---|
| **Latency** | MTP enables speculative decoding → faster generation |
| **Quality** | Richer training signal → better model even without speculative decoding |
| **Cost** | More tokens per forward pass → lower cost per output token |
| **Throughput** | Higher tokens/second → more requests per GPU |

**Engineering takeaway:**
- MTP is both a training technique (better model) and an inference technique (faster generation).
- The idea of "predict, then verify" is broadly useful: speculative execution in agents, optimistic concurrency, parallel candidate generation.
- When evaluating models, check if they support speculative decoding. It can significantly reduce latency without sacrificing quality.
- Sequential prediction (DeepSeek's approach) maintains coherence better than parallel prediction.
- MTP doesn't change what *you* do with the model's API. But it changes the cost and latency profile, which affects your system design.

---

## 11. Architecture × System Design

### Why architecture matters for system builders

Choosing a model is not just a benchmark comparison. The model's architecture determines what systems you can build on top of it.

```
Model Architecture
       │
       ├──→ Context capabilities ──→ RAG strategy
       │
       ├──→ Active compute ────────→ Cost per request
       │
       ├──→ Memory footprint ──────→ Deployment options
       │
       ├──→ Generation speed ──────→ Latency budget
       │
       ├──→ Expert structure ──────→ Task specialization
       │
       └──→ Precision ─────────────→ Hardware requirements
```

### Architecture-aware model selection

Stop asking: *"Which model has the highest benchmark score?"*

Start asking:

| Question | Why it matters | DeepSeek V4 Flash answer |
|---|---|---|
| What is the active compute? | Determines cost and latency | 13B (very low for its capability) |
| How does it handle long context? | Determines RAG vs. long-context strategy | 1M tokens with 90% KV compression |
| What memory does it require? | Determines deployment infrastructure | FP4/FP8 → fits on fewer GPUs |
| Does it benefit from routing? | Determines if you need a model router | Yes—it's already doing internal routing |
| Does it work with quantization? | Determines cost optimization options | Natively FP4—designed for it |
| What workloads fit its architecture? | Determines where to use it | High-volume, fast tasks, agentic workloads |

### How architecture changes system design

**Example 1: RAG decision**

```
Model with 4K context:
  You MUST use RAG. No choice.
  System design dominated by retrieval pipeline.

Model with 128K context:
  RAG is optional for moderate documents.
  System design can be simpler.

Model with 1M context (V4 Flash):
  Entire codebases, long documents fit in context.
  RAG may still help for cost optimization, but is
  architecturally optional for many workloads.
  
  New question: Is it cheaper to retrieve and pass 2K tokens,
  or to pass 100K tokens and let the model's internal
  attention indexing find what's relevant?
```

**Example 2: Cost optimization**

```
Dense 70B model (all params active):
  Cost = f(70B) per token
  Every request costs the same

MoE 284B model (13B active):
  Cost = f(13B) per token
  Same capability class, much lower cost
  
  System implication: You can afford to make more model calls.
  This enables architectures like:
  - Multi-agent systems (multiple cheap calls)
  - Chain-of-thought with verification
  - Iterative refinement
```

**Example 3: Agent architecture**

```
Slow, expensive model:
  Agent architecture = minimize model calls
  Plan carefully, execute once

Fast, cheap model (V4 Flash):
  Agent architecture = iterate freely
  Try, evaluate, retry
  Multiple parallel tool calls
  
  V4 Flash's "Think High" and "Think Max" modes
  let the agent itself decide how much compute to use
  per step—another form of conditional computation.
```

**Engineering takeaway:**
- Model architecture should inform system architecture, not the other way around.
- Active parameters (not total parameters) determine your cost.
- Long context changes the RAG calculus: sometimes stuffing context is cheaper than building a retrieval pipeline.
- Cheap, fast models enable architecturally different (and often better) systems than expensive, slow models.
- The model's precision determines your hardware requirements. FP4 models need dramatically less infrastructure.
- Match your system design to the model's strengths: V4 Flash is designed for high-volume, agentic, long-context workloads.

---

## 12. Ideas You Can Steal

### A. Conditional Computation

**The principle:** Not every request needs the same processing.

```
Instead of:
  Every request → Same pipeline → Same cost

Do:
  Request → Complexity/Intent Assessment
                   │
                   ├── Simple query      → Small model (fast, cheap)
                   ├── Code generation   → Code-specialized model
                   ├── Deep reasoning    → Reasoning model (Think Max)
                   ├── Multilingual      → Multilingual-optimized model
                   └── Domain-specific   → Fine-tuned specialist
```

**When this works:**
- Workload has diverse request types.
- Cost matters (most workloads).
- Different request types have genuinely different difficulty levels.
- You can classify requests cheaply and accurately.

**When this doesn't work:**
- All requests are equally complex.
- Classification overhead exceeds savings.
- You only have one model anyway.
- Quality requirements are uniformly maximum.

**Implementation pattern:**

```python
# Pseudocode for request routing
def route_request(request):
    complexity = classify_complexity(request)  # cheap classifier
    intent = classify_intent(request)          # cheap classifier
    
    if complexity == "simple" and intent == "factual":
        return small_fast_model(request)
    elif intent == "code":
        return code_model(request)
    elif complexity == "hard" or intent == "reasoning":
        return reasoning_model(request, mode="think_max")
    else:
        return default_model(request)
```

---

### B. Information Compression

**The principle:** Don't pass everything forward at full resolution.

```
Instead of:
  Entire conversation history (50 turns, 100K tokens)
       ↓
  Every model call

Do:
  System prompt (always, full)
       +
  Retrieved context (relevant chunks only)
       +
  Conversation summary (compressed)
       +
  Recent turns (last 3-5, full resolution)
       ↓
  Model call (maybe 4K tokens instead of 100K)
```

**Connection to DeepSeek's architecture:**

| DeepSeek V4 internal | Your application |
|---|---|
| Local window (128 tokens, full) | Recent conversation turns (full) |
| CSA (4:1 compressed + selective) | Retrieved relevant context (filtered) |
| HCA (128:1 compressed, global) | Conversation summary (heavily compressed) |
| Shared expert (always active) | System prompt (always present) |

**Implementation patterns:**

1. **Progressive summarization:** After every N turns, summarize the oldest turns and replace them with the summary.
2. **Semantic filtering:** Before each model call, embed the current query and retrieve only relevant past context.
3. **Tiered context:** Divide context into tiers (critical/relevant/background) with different retention policies.

---

### C. Hierarchical Memory

**The principle:** Different information needs different storage strategies.

```
┌─────────────────────────────────────────┐
│  Tier 1: Working Memory                 │
│  Current conversation, recent context   │
│  Storage: In-context (full resolution)  │
│  Lifetime: Current session              │
│                                         │
├─────────────────────────────────────────┤
│  Tier 2: Retrieved Context              │
│  Relevant documents, past interactions  │
│  Storage: Vector DB + retrieval         │
│  Lifetime: Persistent, query-driven     │
│                                         │
├─────────────────────────────────────────┤
│  Tier 3: Compressed Long-Term Memory    │
│  User preferences, learned patterns     │
│  Storage: Structured summaries, KV store│
│  Lifetime: Persistent, periodically     │
│            refreshed                    │
│                                         │
└─────────────────────────────────────────┘
```

This mirrors DeepSeek's multi-resolution attention:
- Tier 1 = Local sliding window (full fidelity, limited range)
- Tier 2 = CSA (compressed but retrievable)
- Tier 3 = HCA (heavily compressed global context)

**Implementation example for a coding assistant:**

```
Working Memory:
  Current file being edited
  Recent conversation (last 5 messages)

Retrieved Context:
  Relevant code files (found via embeddings)
  Related documentation
  Similar past conversations

Long-Term Memory:
  Project architecture summary
  User coding preferences
  Common patterns in this codebase
```

---

### D. Model Routing

**The principle:** Choose the right model for each task.

```
┌─────────────────────────────────────────────┐
│            Model Router                      │
│                                              │
│  Input: request + metadata                   │
│                                              │
│  Routing dimensions:                         │
│  ├── Intent: code / chat / analysis / search │
│  ├── Complexity: simple / medium / hard      │
│  ├── Latency SLA: <1s / <5s / <30s          │
│  ├── Cost tier: free / standard / premium    │
│  └── Context: short / medium / long          │
│                                              │
│  Output: model + configuration               │
│                                              │
│  Examples:                                   │
│  Simple chat     → V4 Flash (non-think)      │
│  Complex code    → V4 Pro (think max)        │
│  Bulk processing → V4 Flash FP4              │
│  Quick lookup    → Small fine-tuned model     │
└─────────────────────────────────────────────┘
```

**Routing strategies:**

| Strategy | How it works | Best for |
|---|---|---|
| **Intent-based** | Classify the request type, route to specialist | Multi-domain applications |
| **Complexity-based** | Estimate difficulty, route to appropriately sized model | Cost optimization |
| **Latency-based** | Route to fastest model that meets quality threshold | Real-time applications |
| **Cost-aware** | Route based on budget allocation | High-volume systems |
| **Fallback** | Try cheap model first, escalate if quality is poor | General-purpose systems |

---

### E. Precision-Aware System Design

**The principle:** Match precision to the workload.

```
┌──────────────────────────────────────────┐
│  Workload Precision Mapping              │
│                                          │
│  High-value / quality-critical           │
│  → Full-precision model (BF16/FP8)      │
│  → Example: Legal document analysis      │
│                                          │
│  High-volume / cost-sensitive            │
│  → Aggressively quantized (FP4/INT4)    │
│  → Example: Content moderation at scale  │
│                                          │
│  Simple / low-stakes                     │
│  → Smallest possible model              │
│  → Example: Auto-complete, suggestions   │
│                                          │
│  Local / offline                         │
│  → Whatever fits on the hardware        │
│  → Example: On-device assistants        │
└──────────────────────────────────────────┘
```

This is the same principle as DeepSeek's mixed precision: expert weights (bulk) get FP4, control-flow weights (critical) get FP8. In your system: bulk workloads get cheap models, critical workloads get premium models.

**Engineering takeaway (all sections):**
- Conditional computation, information compression, hierarchical memory, model routing, and precision-aware design are all variations of the same meta-principle: **allocate resources proportional to need.**
- DeepSeek V4 Flash applies this principle at every level of its architecture. You can apply the same principle at every level of your system.

---

## 13. What NOT to Copy

### Architectural MoE ≠ Application-level routing

You cannot build a "Mixture of Experts" by having three different API endpoints and a classifier. Architectural MoE operates at the token level with microsecond-scale routing decisions, shared representations, and joint training. Application-level routing is a useful pattern, but calling it "MoE" is misleading and will lead to incorrect engineering expectations.

### You probably should not train your own frontier model

DeepSeek V4 was trained on 32T+ tokens using custom hardware, custom optimizers, and a team of researchers. The lessons from their architecture are valuable. The suggestion that you should replicate their training is not.

### Increasing context length is not automatically better

DeepSeek V4 supports 1M tokens. That does not mean you should stuff 1M tokens into every request. Longer context = higher cost, higher latency, potential context dilution. Use the minimum context that gives you the information you need.

### Quantization is not free

DeepSeek V4 Flash maintains quality at FP4 because it was *designed and trained* for FP4 (QAT). If you take an arbitrary model and quantize it to 4 bits post-hoc, expect quality degradation. Treat quantization as a design choice, not a free lunch.

### Compression can destroy information

DeepSeek's HCA layer compresses 128 tokens into 1 entry. This is architecturally sound because the model was trained with this compression. If you aggressively summarize conversation history, you may lose critical details. Compression should be intentional, tested, and monitored.

### More models do not automatically produce better systems

Adding models to a pipeline adds latency, failure modes, and complexity. Each model adds a distribution shift risk, a potential hallucination source, and an integration point. The conditional computation principle says: use what you need, not more.

### Don't confuse model capability with model architecture

DeepSeek V4 Flash's performance comes from the combination of architecture, training data, and post-training. You cannot get the same results by copying only the architecture. The architecture is a necessary condition, not a sufficient one.

---

## 14. Conventional vs Modern Architecture Thinking

| Conventional Thinking | Modern Architecture Thinking | Why the shift |
|---|---|---|
| **Make model larger** | **Increase capacity efficiently** (MoE) | Scaling parameters without scaling compute. 284B params, 13B active. |
| **Put everything in context** | **Compress/retrieve information** (CSA/HCA) | 1M tokens of raw context is unmanageable. Multi-resolution compression makes it tractable. |
| **One model handles everything** | **Conditional specialization** (routed experts) | Different tokens need different knowledge. 256 experts specialize without redundancy. |
| **Full precision everywhere** | **Precision matched to workload** (FP4/FP8) | Expert weights tolerate FP4. Router weights need FP8. Match precision to sensitivity. |
| **More context = better** | **Better information management** | Cramming more tokens in a flat context dilutes attention. Structured compression + retrieval outperforms raw stuffing. |
| **Sequential generation** | **Predict/verify multiple tokens** (MTP) | Speculative decoding generates 1.5-2× faster. MTP training produces better models even without speculation. |
| **Optimize model independently** | **Co-design architecture and systems** | FP4 weights are designed for Blackwell GPUs. CSA is designed for the KV cache hardware budget. Architecture and hardware are co-designed. |
| **Residual connections are solved** | **Residual connections need scaling** (mHC) | Standard residual connections break down in very deep MoE networks. mHC constrains signal propagation to prevent training collapse. |

---

## 15. A Practical AI Engineer's Mental Model

When evaluating any modern LLM, ask these ten questions:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. How much total capacity does it have?                   │
│     → Total parameter count (284B for V4 Flash)             │
│                                                             │
│  2. How much capacity is actually active?                   │
│     → Active parameters per token (13B for V4 Flash)        │
│     → This determines your real compute cost                │
│                                                             │
│  3. How does it decide what computation to perform?         │
│     → Router architecture, expert selection strategy        │
│     → Quality of routing = quality of sparse activation     │
│                                                             │
│  4. What information does it retain at full resolution?     │
│     → Local window (128 tokens), critical parameters (FP8)  │
│     → What's privileged in the architecture?                │
│                                                             │
│  5. What information does it compress?                      │
│     → KV cache (CSA 4:1, HCA 128:1), weights (FP4)         │
│     → What might be lost?                                   │
│                                                             │
│  6. How does it handle long context?                        │
│     → Hybrid attention, multi-resolution compression        │
│     → Can I actually use the full context window reliably?  │
│                                                             │
│  7. What precision does it use?                             │
│     → FP4 experts, FP8 control. QAT or post-hoc?           │
│     → Determines memory footprint and hardware requirements │
│                                                             │
│  8. How does it reduce unnecessary computation?             │
│     → Sparse activation, sparse attention, speculative      │
│       decoding. Every "skip" is a cost saving.              │
│                                                             │
│  9. What does this architecture imply for system design?    │
│     → Cost profile, latency profile, context strategy,      │
│       routing strategy, deployment requirements             │
│                                                             │
│  10. Which of these ideas can I apply without building      │
│      a frontier model?                                      │
│      → Conditional computation, compression, hierarchical   │
│        memory, model routing, precision-aware design        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**The summary in one sentence:**

> DeepSeek V4 Flash is an existence proof that **intelligent resource allocation**—using routing, compression, mixed precision, and speculative execution—can deliver frontier-class capability at a fraction of the expected cost. Every one of those ideas applies to the AI systems you build.

---

## 16. Glossary

| Term | Definition |
|---|---|
| **MoE** | Mixture of Experts. Replace one large FFN with many smaller FFNs and a router. |
| **Sparse Activation** | Only a subset of the model's parameters are active for any given input. |
| **Router** | A learned gating mechanism that decides which experts process each token. |
| **Shared Expert** | An expert that is always active for every token, capturing universal knowledge. |
| **Routed Expert** | An expert that is conditionally activated by the router. |
| **MLA** | Multi-Head Latent Attention. Compresses KV pairs into low-dimensional latent vectors. |
| **CSA** | Compressed Sparse Attention. Compresses groups of 4 tokens into 1 KV entry, then selectively attends. |
| **HCA** | Heavily Compressed Attention. Compresses groups of 128 tokens into 1 KV entry for global context. |
| **Lightning Indexer** | A learned query-conditioned mechanism inside CSA that selects relevant compressed entries. |
| **mHC** | Manifold-Constrained Hyper-Connections. Stabilized residual connections using doubly stochastic matrices. |
| **KV Cache** | Storage of key and value tensors from previous tokens, enabling incremental generation. |
| **FP4 / FP8** | 4-bit and 8-bit floating-point formats for model weights. |
| **QAT** | Quantization-Aware Training. Training the model with simulated low-precision to preserve quality. |
| **MTP** | Multi-Token Prediction. Training objective that predicts multiple future tokens. |
| **Speculative Decoding** | Generate draft tokens quickly, then verify them in parallel to speed up generation. |
| **GRPO** | Group Relative Policy Optimization. RL algorithm used in DeepSeek's post-training. |
| **Muon Optimizer** | Optimizer used by DeepSeek V4 for training stability and convergence. |
| **Context Dilution** | Degradation of model performance when relevant information is buried in a very long context. |
| **Expert Collapse** | Failure mode where most tokens route to a few experts, leaving others unused. |

---

## 17. Recommended Reading

### Primary sources

| Source | What you learn | Priority |
|---|---|---|
| **DeepSeek V4 Technical Report** (arxiv.org) | Full architecture details, training methodology, CSA/HCA design | 🔴 Essential |
| **DeepSeek V3 Technical Report** (arxiv.org) | MLA, MTP, DeepSeekMoE foundations that V4 builds on | 🔴 Essential |
| **DeepSeek V4 HuggingFace Model Card** | Confirmed specs, benchmark results, deployment details | 🟡 Important |
| **DeepSeekMoE Paper** (arxiv.org) | Original fine-grained MoE + shared expert design | 🟡 Important |

### Conceptual background

| Source | What you learn |
|---|---|
| **"Outrageously Large Neural Networks"** (Shazeer et al., 2017) | Original MoE paper for Transformers |
| **GQA Paper** (Ainslie et al., 2023) | Grouped-Query Attention, the predecessor to MLA |
| **"Better & Faster Large Language Models via Multi-Token Prediction"** (Meta, 2024) | MTP concept and training benefits |
| **"Fast Inference from Transformers via Speculative Decoding"** (Leviathan et al., 2023) | How speculative decoding works |

### For AI engineers specifically

| Source | What you learn |
|---|---|
| **Sebastian Raschka's blog posts on MLA and DeepSeek** | Clear, engineer-friendly explanations of attention innovations |
| **DeepSeek's API documentation** | Practical integration: reasoning modes, context limits, pricing |
| **"The Illustrated Transformer"** (Jay Alammar) | If you need a refresher on Transformer basics (not math, concepts) |

---

*Document created August 2026. Architecture details based on publicly available DeepSeek V4 documentation, technical reports, and confirmed model specifications. Unconfirmed details are explicitly labeled.*
