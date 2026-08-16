# Production RAG Chatbot — System Design Field Guide

How Claude, ChatGPT, and OSS chatbot wrappers actually handle chat history, context, retrieval, and generation for live users — taught as system design: architecture choices, tradeoffs, cost, latency, and the decisions that separate demos from production.

---

## 0. TL;DR — the request lifecycle

A single user message in a production RAG chatbot travels through these stages:

```
Client (browser/app)
  │  sends: { message, conversation_id, parent_message_id }   ← NOT the history
  ▼
API Gateway (auth, rate limit, abuse filter)
  ▼
Chat Orchestrator (stateless worker)
  ├─ 1. Load thread from DB (messages as linked tree)
  ├─ 2. Budget context (token count, truncate or summarize)
  ├─ 3. Query understanding (extract filters: product, task, doc version)
  ├─ 4. Retrieve (hard metadata filters → hybrid BM25+vector → rerank)
  ├─ 5. Assemble context (small-to-big, walk the chunk graph)
  ├─ 6. Semantic cache check (same meaning, different wording?)
  ├─ 7. Agent loop (LLM call → tool call → LLM call, up to N times)
  ├─ 8. Stream tokens back via SSE
  └─ 9. Persist the new messages
  ▼
Model (Claude / GPT / OSS), Vector DB, Reranker, Embedding service
```

The model is only stage 7. Everything else is what makes the *conversation* work. The **data plane** (ingestion pipeline, chunk graph, index versions — §5.7–5.8) runs alongside: it's what stages 3–5 read from, and the quality system (§6.6) is what keeps it honest.

---

## 1. Conversation state: the thread data model

### 1.1 The client does NOT send the history

The web client only sends:

```json
POST /conversations/{conv_id}/responses
{
  "message": "How do I recalibrate after replacing the print head?",
  "conversation_id": "c_9f2a...",
  "message_id": "m_8ab1...",
  "parent_message_id": "m_3cd2..."     // what this message responds to
}
```

The server **rebuilds the full thread from its database**. This is why:
- You can switch devices and the chat persists
- Refreshing the page doesn't lose context
- The client holds rendered markdown; the server holds source-of-truth messages

### 1.2 Messages are stored as a tree (linked list), not a flat array

```sql
CREATE TABLE messages (
  id            TEXT PRIMARY KEY,       -- "m_8ab1..."
  conversation_id TEXT NOT NULL,
  role          TEXT NOT NULL,          -- 'user' | 'assistant' | 'system'
  content       TEXT NOT NULL,          -- plain text; markdown is client-side
  parent_id     TEXT REFERENCES messages(id),   -- ← the message it responds to
  status        TEXT NOT NULL,          -- 'streaming' | 'completed' | 'failed'
  model         TEXT,                   -- which model answered this
  created_at    TIMESTAMP NOT NULL,
  version       INT DEFAULT 1           -- regeneration bumps this
);

CREATE TABLE conversations (
  id          TEXT PRIMARY KEY,
  user_id     TEXT NOT NULL,
  title       TEXT,
  system_prompt_id TEXT,                -- which instructions variant was active
  created_at  TIMESTAMP NOT NULL,
  archived_at TIMESTAMP
);
```

Why a tree? Regeneration and editing:

- **Regenerate** → delete the assistant child, re-run the pipeline, bump `version`.
- **Edit a user message** → delete everything *after* it, create a new branch.
- The API history is rebuilt by walking `parent_id` pointers from the current node to the root.

### 1.3 Concurrency and ordering

- Message IDs are **client-generated temp IDs** that map to server IDs, so a re-sent request is idempotent (retry doesn't duplicate).
- A streaming assistant message is written to the DB **after** the stream finishes (status `streaming` → `completed`).
- Because workers are stateless, two requests for the same conversation must serialize: take a per-conversation lock (Redis `SETNX` or Postgres advisory lock) around the turn, or the agent loop interleaves and corrupts the thread.

### 1.4 Token counting per message (the budget basis)

Every message is pre-tokenized with the model's tokenizer (tiktoken for OpenAI models, claude tokenizers / `Anthropic().count_tokens()` for Claude, HF tokenizer for OSS). You store `token_count` on the message row at write time — counting at read time costs latency per turn.

---

## 2. The LLM call: exact shape before the latest prompt

The server calls the model with **the entire history, role-wrapped, oldest → newest**, plus the system stack. Using this very conversation as an example, the call into Claude's API looks like:

```json
{
  "model": "claude-sonnet-4-5",
  "system": [
    { "type": "text", "text": "You are a helpful assistant. [platform-level instructions]" },
    { "type": "text", "text": "[custom instructions from the user's profile]" },
    { "type": "text", "text": "<documents>\n<document>\n[retrieved chunks — only if RAG fired this turn]\n</document>\n</documents>" }
  ],
  "messages": [
    { "role": "user",      "content": [{ "type": "text", "text": "How does claude and chatgpt or oss process chat history..." }] },
    { "role": "assistant", "content": [{ "type": "text", "text": "Good question — here's how it works..." }] },
    { "role": "user",      "content": [{ "type": "text", "text": "i dont want to know about models..." }] },
    { "role": "assistant", "content": [{ "type": "text", "text": "Got it — you want the application-layer architecture..." }] },
    { "role": "user",      "content": [{ "type": "text", "text": "i am more interested in learning for a corpus like instructions manual..." }] },
    { "role": "assistant", "content": [{ "type": "text", "text": "This is the right question — flat chunking..." }] },
    { "role": "user",      "content": [{ "type": "text", "text": "how does that thread look like, just before processing starts..." }] }
  ],
  "max_tokens": 4096,
  "stream": true,
  "temperature": 0.7,
  "tools": [{ "name": "retrieve_docs", "description": "...", "input_schema": { "type": "object", ... } }],
  "tool_choice": "auto"
}
```

Key points:

- **System** = instructions + tool schemas + RAG context. Invisible to the user, present in every call.
- **Messages** = the whole history verbatim. Nothing is compressed until it violates the token budget.
- OSS models use the same shape via a **chat template** (`apply_chat_template` in HF Transformers), which converts roles to model-specific special tokens (`<|start_header|>user<|end_header|>` for Llama, `<|im_start|>` for Qwen, etc.).
- The client never sees this. A naive wrapper that just forwards the client's messages array is a security hole (client controls the system prompt) — always rebuild server-side.

---

## 3. Context budgeting (what fits in the window)

### 3.1 The algorithm

```
budget = context_window - max_output_tokens - safety_margin (~10%)
take   = []

for msg in messages_from_newest_to_oldest:
    if sum(tokens of take) + msg.token_count > budget:
        break
    take.append(msg)

if anything was dropped:
    if summarizer enabled:
        summary = llm("Summarize these older messages", truncated_tail)
        prepend summary as a system/user message
    else:
        just drop the oldest turns
```

### 3.2 Truncation strategies — the tradeoff

| Strategy | What it does | Good for | Cost |
|---|---|---|---|
| Drop oldest | Keep system + newest turns | Short chats, cheap | Loses early context (user's stated goal, earlier constraints) |
| Drop middle | Keep head and tail | Long-form reasoning, docs | Weird for conversations; the "lost middle" confuses follow-ups |
| Rolling summary | LLM compresses dropped turns into a paragraph | Long chats, agents | +1 LLM call (cost + latency), lossy |
| Hard cap on turns | e.g. last 20 messages max | Chats, not code | Simple, but arbitrary |
| Summarize + keep last N verbatim | Hybrid | Production | The standard answer |

**Production rule of thumb:** keep the system prompt + the last 1–2 turns verbatim, summarize everything older. Users re-read what they just said; they rarely need turn 40 word-for-word.

### 3.3 Context is expensive — that's the real driver

The dominant cost of a turn is **input tokens**, not output. Every turn re-sends the history. A 20-turn conversation at 5k tokens/turn average is ~100k input tokens per turn. This is *the* reason prompt caching and context trimming exist (see §7).

---

## 4. The agent loop and streaming

### 4.1 One turn = multiple LLM calls

```
turn starts
  loop (max N iterations, e.g. 10):
    call LLM with current messages + tools
    if response has tool_use:
        execute the tool (retrieval, search, code, DB)
        append tool_result as a new message
        loop again
    else:
        stream the text to the client
        done
```

- Each iteration re-sends the growing history — this is why providers cache the prefix (§7.1).
- "Seems smart" is usually 2–3 sequential calls: plan → retrieve → answer.
- **Timeout budget per iteration** is mandatory or a single bad tool call stalls the whole turn. Track cumulative latency; if > P95 target, stop the loop and answer with what you have.

### 4.2 Streaming

- Server-Sent Events (SSE): `data: {"type":"content_block_delta","delta":{"text":"..."}}` relayed to the client.
- The client renders incrementally; the DB write happens after the stream completes.
- **Backpressure:** the client may close the connection (user hits stop). The orchestrator must propagate cancellation to the model API (`abort`/`cancel`) so you stop paying for tokens.

---

## 5. RAG: corpus → index → retrieval → assembly

### 5.1 Ingestion: build a typed chunk tree (the manual case)

Flat chunking + pure vector search breaks on instruction manuals. You need a **structured graph** with **metadata-first retrieval**.

Parse the document into typed chunks by structure:

```
Manual (product_scope: ["XL-2000"], firmware_scope: ">=3.4")
├── Part "Maintenance"                          level 1
│   ├── Section "Print head"                    level 2
│   │   ├── Prereq (tools, warnings)            leaf
│   │   ├── Procedure "Replace print head"      level 3
│   │   │   ├── Step 1
│   │   │   ├── Step 2 ──────┐
│   │   │   └── Step 3 ──────┤ conditional: "if LED red → Recalibrate"
│   │   └── Note "E-04 light"
│   ├── Section "Calibration"                   level 2
│   │   └── Procedure "Recalibrate"
│   │       └── Step 1..6
│   └── Table "Error codes"                     whole-table chunk
```

Rules:
- **Procedures split**: one header chunk (purpose/tools/prereqs) + one chunk per step. Atomic steps = precise retrieval; header = context anchor.
- **Conditionals become links**: "if LED red → go to Recalibrate" is `links: ["proc_recalibrate"]` on Step 3, not buried prose.
- **Tables are chunks**: whole table = embedding unit AND context unit. Steps that say "see Table 3-2" carry `links: ["table_3-2"]`.

### 5.2 The chunk metadata schema (denormalized onto every chunk)

```json
{
  "chunk_id": "sec_maint.calib.proc_recal.step_2",
  "type": "step",
  "title": "Turn adjustment screw 2 clicks clockwise",
  "parent_id": "proc_recalibrate",
  "ancestors": ["man_xl2000", "part_maint", "sec_calibration", "proc_recalibrate"],
  "path": "man_xl2000/part_maint/sec_calibration/proc_recalibrate/step_2",
  "level": 4,
  "order": 2,
  "procedure_id": "proc_recalibrate",
  "step_index": 2,
  "product_scope": ["XL-2000", "XL-2000+"],
  "firmware_scope": ">=3.4",
  "error_codes": ["E-04"],
  "links": ["table_3-2"],
  "page": 42,
  "doc_version": "3.4"
}
```

Two designs for the hierarchy:

| | Ancestors array | Materialized path | parent_id only |
|---|---|---|---|
| Subtree filter | `ancestors CONTAINS "sec_calibration"` | `path LIKE '...%'` prefix | recursive CTE / in-memory walk |
| Cost | Extra index space | Tiny, fast | Query-time graph walk |
| Verdict | Standard in vector DBs | Great for SQL | Only if ingestion can't produce paths |

If your chunker is **agentic and only produces top-down parent_id relations**, use the in-memory adjacency map for assembly (§5.5) and materialized paths for filtering. A manual corpus is ~50k chunks — both fit in RAM trivially.

### 5.3 Index design

Two indexes, queried in parallel and fused (hybrid search):

- **Vector index** (semantic): embedding model → Qdrant/Weaviate/pgvector. Every chunk's payload carries all of §5.2 metadata for filtering.
- **Keyword index** (BM25): catches model numbers, error codes, part IDs ("E-04", "XL-2000") — things embeddings are notoriously bad at.

Fusion via **Reciprocal Rank Fusion (RRF)**:

```
score(chunk) = Σ over each index of 1 / (60 + rank_index(chunk))
```

### 5.4 Retrieval pipeline: filters FIRST, similarity second

```
query: "recalibrate after replacing print head on my XL-2000"

[Stage 1] Query understanding — extract STRUCTURED filters from the text
    product    = "XL-2000"        (entity extraction / small LLM)
    task       = "calibration"
    trigger    = "after-replace"
    type       = procedure|step

[Stage 2] Hard metadata pre-filter (NOT semantic — index-level)
    product_scope CONTAINS "XL-2000"
    AND path LIKE "man_xl2000/part_maint/%"
    AND firmware_scope PASSES ">=3.4"
    AND type IN (procedure, step)
    → 50,000 chunks → ~40 candidates.
    Semantic search never sees the rest of the corpus.

[Stage 3] Search WITHIN the filtered set (hybrid)
    vector_topk = embed(query) → similarity search
    bm25_topk   = keyword search
    candidates  = RRF(vector_topk, bm25_topk)

[Stage 4] Rerank (cross-encoder, e.g. bge-reranker)
    score(query, chunk) for top ~20 → keep top 4–6
```

This is how you "don't pull the whole chain": filters are hard constraints applied *before* similarity, so word matches in the wrong section can never win.

### 5.5 Assembly: small chunks in, complete window out

Retrieval gives small chunks; generation needs complete context. The rules: **match small, walk up for anchor, sideways for neighbors, follow links for cross-refs — never pull every descendant, never pull every sibling.**

```python
# --- in-memory graph loaded once at startup ---
PARENTS = {}      # chunk_id -> parent_id
STEPS   = {}      # chunk_id -> (procedure_id, step_index)
CHUNKS  = {}      # chunk_id -> full chunk row

def ancestors_of(chunk_id, stop_at_level=3):
    out, cur = [], chunk_id
    while cur in PARENTS and len(out) < 4:
        cur = PARENTS[cur]
        if CHUNKS[cur]["level"] <= stop_at_level:
            break                                   # stop at procedure header
        out.append(cur)
    return out

def neighbors_of(chunk_id, radius=1):
    proc, idx = STEPS.get(chunk_id, (None, None))
    if proc is None:
        return []
    return [c for c in db.query(
        "SELECT * FROM chunks WHERE procedure_id=? AND step_index BETWEEN ? AND ?",
        proc, idx - radius, idx + radius)]

def links_of(chunk_id):
    return [CHUNKS[l] for l in (CHUNKS[chunk_id].get("links") or [])]

# --- retrieval ---
hits = hybrid_search(
    query,
    pre_filter={
        "type": ["step", "procedure", "table"],
        "path_prefix": "man_xl2000/part_maint",
    },
    top_k=4,
)

# --- assembly ---
context = {}
for hit in hits:
    context[hit["chunk_id"]] = hit
    for a in ancestors_of(hit["chunk_id"]):
        context[a["chunk_id"]] = a
    for n in neighbors_of(hit["chunk_id"]):
        context[n["chunk_id"]] = n
    for l in links_of(hit["chunk_id"]):
        context[l["chunk_id"]] = l

# don't pull every step of a procedure just because its header matched
for c in list(context.values()):
    if c["type"] == "step" and c["chunk_id"] not in {h["chunk_id"] for h in hits}:
        if c["procedure_id"] and c["chunk_id"] not in context_hits_within_proc:
            pass  # keep only neighbors already added; don't add whole procedure

prompt_context = render_in_order(sorted(context.values(),
                                        key=lambda c: (c["level"], c["order"])))
```

Assembly output goes into the `system` documents block (§2), rendered with steps numbered (`Step 2 of 6: ...`) so the model respects sequence.

### 5.6 Semantic caching (same meaning, different wording)

No LLM call decides "these mean the same." Pipeline:

```
incoming query
  ├─ [fast path] hash(normalized_query) in cache → return cached answer
  ├─ [semantic path] embed(query) → cosine vs cached query embeddings
  │      score ≥ threshold (start 0.92–0.95) → return cached answer
  │      score <  threshold → full pipeline → store result
  └─ cache key includes: user/session, conversation context hash,
                         model params, AND retrieval fingerprint
```

Critical details:

- The **retrieval fingerprint** is the hash of retrieved chunk IDs + doc versions. If the corpus changes (new manual version), the same query must NOT return the stale answer — key the cache on what you retrieved, not just the query.
- Threshold tuning: 0.85 → wrong answers on near-misses; 0.98 → effectively exact matching. Start at 0.93, measure against your eval set.
- Cache per user — "my printer is stuck" means different things to different people.
- Implementation: Redis + an embedding similarity search (or GPTCache). Cost: one embedding call (~5–20ms) instead of a full generation (~2–4s).

### 5.7 The ingestion pipeline (corpus → chunks)

Ingestion is a **batch system, never part of the request path**. It runs on a schedule/queue (Airflow, Prefect, Dagster, or a plain worker) and produces artifacts the request path only *reads*.

```
source sync → parse → structure → clean → embed → quality gate → publish

[1] Source sync      watch doc sources (SFTP/S3/DRM); new manual → (doc_id, version)
                    dedupe by content hash; only changed docs flow downstream

[2] Parse            PDF → layout analysis (Docling / Marker / PyMuPDF / pdfplumber),
                    OCR fallback for scans. Extract headings, tables, page numbers.
                    QUALITY GATE: token yield vs pages, table detection rate — a parser
                    that silently loses tables poisons the index.

[3] Structure        build the typed tree (§5.1). Agentic chunking: an LLM proposes
                    section/step boundaries + writes header summaries.
                    DETERMINISTIC VALIDATION (no LLM): chunk ids unique, no text
                    gaps or overlaps, order monotonic, links resolve, token caps.

[4] Clean            PII scrubbing, boilerplate removal, near-duplicate chunk detection.

[5] Embed            batch embed with the embedding model; store vectors + payload.

[6] Quality gate     sample chunks → run golden-set queries (§6.6) against the NEW
                    index: retrievability + faithfulness must pass, else reject version.

[7] Publish          atomic index swap (blue/green). Bump the retrieval-fingerprint
                    epoch so the semantic cache (§5.6) invalidates. Keep the old index
                    for instant rollback.

[8] Re-embed         when the embedding model changes → full rebuild, not in-place update.
```

Rules that keep ingestion safe:

- **Idempotency**: every stage is keyed by `(doc_id, version, chunk_id)`; re-running a stage is a no-op. This is what makes retries and backfills safe.
- **The index is a cache of the truth** — it must be rebuildable from parsed docs at any time. Never let the index be the only copy.
- **Ingestion is where RAG quality is won or lost** — more than retrieval tuning. Chunk boundaries and metadata are the highest-leverage knobs; this is where you spend your eval effort (§6.6), not on prompt tweaks.
- Log every stage with counts (docs in, chunks out, tokens, rejects) — you need to see a parser regression before users do.

### 5.8 Storage architecture (the data plane)

Production RAG uses **tiered storage**, because tiers have different rebuild time, retention, and access patterns:

| Tier | Stores | Technology | Characteristics |
|---|---|---|---|
| Source files | PDFs, scans, original docs | Object storage (S3), versioned, immutable | Source of truth; keep forever (compliance) |
| Parsed documents | structured tree JSON per doc version | Object storage or SQL, immutable | Rebuildable from sources in minutes |
| Chunks + metadata | chunk rows + payloads | Vector DB (Qdrant/Weaviate) **+ SQL mirror** | SQL mirror exists for assembly joins (`procedure_id`, `step_index` between-queries) — don't do those in the vector DB |
| Conversations / messages | thread tree | Postgres | §1; retention policy applies |
| Caches | semantic cache, conversation locks, hot threads | Redis | §5.6; must be invalidated by fingerprint epoch |
| Traces / evals / feedback | per-turn telemetry | OLAP/warehouse or Postgres + Langfuse | Short retention, PII-scrubbed |

Design rules:

- **Never query the warehouse at request time** — the hot path reads only the vector DB + SQL + Redis. Analytics are batch.
- **Version everything**: `doc_version`, `index_version`, `embedding_model`, `prompt_id` must land in every trace — without them you cannot answer "why did the answer change after the manual update?"
- **ACLs enforced in two places**: the index payload filter (a user must never *retrieve* a chunk they can't see — post-filtering leaks) and the object store (per-tenant prefixes/buckets).
- **Backups and restore drills per tier**: conversations (minutes RTO), source files (hours), chunk index (rebuildable in hours — it's the cheapest tier precisely because it's derived data).
- Encryption at rest and in transit everywhere; retention jobs run on schedule (§6.8).

---

## 6. Production systems thinking

### 6.1 Latency budget

A turn is a chain of dependent calls. Budget the p95 end-to-end:

| Stage | Typical | Notes |
|---|---|---|
| Auth, rate limit, load thread | 10–40 ms | DB reads, cached |
| Query understanding (small LLM) | 100–400 ms | Optional; a classifier or heuristics is faster |
| Embed query | 5–20 ms | Batched, GPU service |
| Vector + BM25 + RRF | 10–60 ms | Pre-filtered candidate set keeps this flat |
| Rerank (cross-encoder) | 20–80 ms | GPU |
| Assembly + cache check | < 5 ms | In-memory |
| LLM first token (TTFB) | 300–1500 ms | Depends on model + prefix cache hit |
| Streaming to completion | 30–100 tok/s | Output length × speed |

**Total p95 target: 2–4 s to first meaningful token, 5–10 s to full answer.** Budget breakdown:
- If retrieval > 200 ms, you're over-fetching (raise filter selectivity) or your index is undersized.
- If TTFB dominates, you're paying for the wrong thing: prefix cache misses (§7.1), big context, or cold model.
- Always return *something* in < 1 s: stream partial state, show "searching your docs..." — perceived latency beats real latency.

### 6.2 Cost model

Cost per turn ≈ input tokens × input price + output tokens × output price + retrieval + rerank + cache misses.

Illustrative prices (check current pricing pages; these move):

| Item | Approx price | Per typical turn |
|---|---|---|
| Embedding (text-embedding-3-small) | $0.02 / 1M tok | ~$0.00001 |
| Vector search + BM25 (self-hosted) | infra | ~$0 |
| Rerank (bge-reranker-v2-m3, self-hosted) | infra | ~$0 |
| Claude Sonnet 4.x | ~$3 in / $15 out per 1M | 40k in + 500 out ≈ $0.13 |
| GPT-4o mini | ~$0.15 / $0.60 per 1M | ≈ $0.007 |
| DeepSeek V3-class OSS | ~$0.27 / $1.10 per 1M | ≈ $0.012 |

The three levers that actually move the bill:

1. **Context length** — 20 turns × 5k tokens = 100k input tokens/turn. Trimming to 30k cuts input cost 3×. This is the #1 lever.
2. **Prompt/prefix caching** — cached tokens cost ~10% of uncached (Anthropic/OpenAI both discount cached input by ~90%). A stable system prompt + history prefix turns $0.13 into ~$0.02. Cache misses (first turn, or any change to the system prompt) are the expensive ones.
3. **Model routing** — small model for query understanding and summarization, big model only for the final answer. Frontier model per turn is the "wasteful default" nearly every production team reverses.

Typical production shape: 90% of turns are cheap (cached prefix + small model + 1 call); 10% are expensive (agents, long tool loops). Budget on the 10%.

### 6.3 Scaling and concurrency

- **Workers are stateless** — the thread lives in the DB, so any worker can serve any conversation. Horizontal scaling = add pods.
- **The model API is the bottleneck** — providers rate-limit; a sync fan-out to the model can exhaust quota. Use a per-tenant queue (Redis Streams/Kafka) with worker pools sized to the provider rate limit, not to your request rate.
- **Backpressure**: if queues exceed a threshold, reject fast with 429 + retry-after rather than queueing for minutes. Users refresh instead.
- **Per-conversation serialization**: one lock per conversation (§1.3) — with stateless workers this is Redis `SETNX conv_id`.
- **Multi-model failover**: primary model 429/5xx → fallback model, then fallback provider. Measure and route by latency/error rate. This is standard at scale.
- **Cold starts**: embedding + reranker services must be prewarmed (they're the interactive-stage calls, not batch).

### 6.4 Observability: minimum telemetry

You cannot iterate on RAG without knowing *where* it fails. Minimum instrumentation per turn (deep-dives: evaluation system → §6.6, deployment & lifecycle → §6.7, governance → §6.8):

```
turn_id, user_id, conv_id,
query, filters_extracted,
candidate_counts (pre-filter → top-k → reranked),
chunk_ids_retrieved, assembly tree (which parents/neighbors/links added),
cache_hit_or_miss,
model, tokens_in/out, cost,
latency breakdown (retrieve/rerank/llm/stream),
user_feedback (thumbs up/down, regenerate count)
```

Offline evaluation (before shipping changes):
- Golden set: 100–500 real user queries with human-verified answers.
- Metrics: retrieval recall@k, answer faithfulness (RAGAS or LLM-as-judge), latency, cost.
- Run every index/embedding/prompt change against the golden set; no golden set = no changes.

Online evaluation (after shipping):
- Thumbs up/down, regenerate rate, explicit "this didn't answer my question" clicks.
- Sampling for LLM-judge quality scoring on real traffic.
- The regenerate button is a free eval signal: it means the first answer failed.

### 6.5 Security and reliability

- **Prompt injection via retrieved docs**: the corpus is untrusted input. Never let retrieved text override the system prompt. Common defenses: delimit retrieved content with tags, instruct the model to treat it as data, run an injection-classifier on retrieval results (small model), never auto-execute tool calls derived from doc content.
- **Access control**: documents have ACLs; filters must enforce them *in the index* (a user must not retrieve chunks they can't see — post-filtering leaks).
- **Rate limiting and abuse**: per-user quotas, content moderation on input, cost caps per tenant (one runaway agent loop = one big invoice).
- **PII**: logs must redact; don't ship raw conversation text to your observability vendor.
- **Graceful degradation**: no retrieval hits → answer honestly ("I don't have that doc") instead of hallucinating; model outage → queue the turn or offer email follow-up; retrieval latency spike → skip rerank, answer with top-k.
- **Corpus updates**: version chunks; the semantic cache fingerprint (§5.6) is how stale answers get invalidated. Re-ingest on a schedule, verify embeddings of new chunks, and run a diff against the eval set.

### 6.6 Evaluation: the quality system

Two loops — **offline gates** (before shipping) and **online signals** (after shipping) — feeding each other in a flywheel.

**Offline (in CI, blocks deploys):**

- Golden set: 100–500 real user queries with human-verified expected chunks + answers, tagged by domain (procedures, error codes, tables, cross-refs, follow-ups, ambiguous queries).
- Metrics you track:
  - *Retrieval*: recall@k, MRR — did the right chunks get retrieved?
  - *Answer*: faithfulness (every claim supported by retrieved chunks), answer relevance, coverage (did it answer all parts of the question?) — via RAGAS or LLM-judge, with human spot-checks on a sample.
  - *System*: cost and latency per query. Quality without cost/latency numbers is not production-ready.
- **Regression gates**: every change — chunk boundaries, embeddings, filters, prompts, reranker, assembly rules — runs the golden set; block on faithfulness or recall regression beyond a threshold (e.g. >2pp). This is the only thing standing between "it feels better" and "it is better."
- **Failure mining**: every production failure (negative feedback, regenerate click, low judge score) becomes a new golden case. The set grows with the product — this is the flywheel.

**Online (on live traffic):**

- Implicit signals: thumbs up/down, regenerate rate, follow-up questions right after an answer (= incomplete answer), abandonment.
- Judge-on-sample: 5–10% of turns scored by an LLM-judge daily; watch trends, not single points.
- A/B on 5–10% traffic: index A vs B, prompt A vs B — judged on quality + cost + latency.
- Human review queue for low-confidence/low-score answers; the queue feeds the golden set.
- Alerts: faithfulness drop >2pp, retrieval recall drop, latency p95 breach, semantic cache hit-rate drop, per-turn cost drift.

```
production failures → golden set → CI regression gates → better ingestion/retrieval → fewer failures
```

### 6.7 Deployment, testing & lifecycle

- **Everything is a versioned artifact**: prompts (git-tracked, `prompt_id` referenced in traces), model versions, embedding models, chunk schema versions, index versions. If a trace doesn't record the versions it used, you can't debug it.
- **Index deploys are blue/green**: build the new index in parallel, point traffic at it, keep the old for rollback. Never mutate the live index in place for a big change — a failed in-place rebuild is a silent quality regression.
- **Prompt/model deploys are canaries**: 5–10% traffic, compare judge score + cost + latency vs baseline, auto-rollback on alert breach. The model and the prompt are a single artifact pair — changing one without re-evaluating the other is the classic regression.
- Test pyramid:
  - *Unit*: chunking rules, metadata extraction, filter logic, assembly logic — deterministic, assert exact outputs (these are plain functions; test them like any other).
  - *Integration*: fixture manual → parser → chunks → index → retrieval returns the expected chunk IDs.
  - *Load*: concurrent turns vs latency SLO; queue depth; provider rate-limit behavior; cache hit ratio under load.
  - *Chaos*: provider 5xx/429, vector DB down, cache down, reranker down — verify graceful degradation (§6.5) actually degrades gracefully.
- **Runbooks**: a documented failure mode + action per component (index corrupted, embedding model deprecated, provider outage, cache thrashed). First incident in LLM systems is always a cost, cache, or prompt surprise — the runbook is what saves you.
- Ingestion scheduling: nightly/weekly re-ingest, index rebuild on embedding change, retention jobs — all with the quality gates from §5.7.

### 6.8 The other things that are crucial (governance & operations)

- **Cost governance**: per-tenant budgets, cost dashboards by feature/model, alerts on per-turn cost drift, kill-switch per feature. An agent loop is bounded by max iterations *and* max tokens *and* a cumulative latency cap — three independent limits.
- **Data lineage**: which doc version answered which turn. Required for "the answer changed after the manual update" debugging; a `chunk_id → doc_version` lookup in the trace is enough.
- **Compliance**: retention policies per data class, GDPR deletion (user deletes → conversations, semantic cache entries, traces), regional data residency, audit logs for admin actions.
- **Prompt-injection red-teaming**: an adversarial corpus in CI — docs that try to override instructions, exfiltrate, or force tool calls. Run it on every ingestion change.
- **Multi-tenant isolation**: tenant metadata in every chunk payload; cross-tenant leakage tests in CI (tenant A's query must never return tenant B's chunks).
- **Feedback capture is architecture, not a feature**: the thumbs/feedback UI feeds §6.6; if it doesn't exist, the quality loop doesn't exist.
- **Human-in-the-loop**: confidence-based escalation — low retrieval score or low judge score → offer "talk to a human" or route to a review queue instead of confidently hallucinating.

---

## 7. Tradeoff cheat sheet

| Decision | Cheap/simple | Production | Why |
|---|---|---|---|
| Chunking | Fixed 500-token splits | Typed, hierarchical, procedural | Manuals need structure; flat chunks lose flow |
| Retrieval | Vector-only | Hybrid BM25 + vector + filters | Part numbers and error codes defeat embeddings alone |
| Filtering | Post-filter results | Pre-filter at index | Post-filter can return nothing while docs exist |
| Context | Full parent chunk | Small-to-big assembly | Big chunks waste context; small lose context |
| Top-k | 10 | 4–6 + rerank | More chunks ≠ better; rerank is the quality lever |
| History | All messages | Budget + summarize + cache | Input tokens are the cost center |
| Cache | None | Exact + semantic + prefix | 3× cost reduction, faster TTFB |
| Latency | Serial everything | Pipeline + budgets + streaming | Perceived latency = streaming |
| Model | One frontier model | Route: small (understanding) → big (answer) | Cost 10×, quality same |
| Scaling | Synchronous | Queues + per-conv locks + failover | Provider rate limits are real |
| Eval | Vibes | Golden set + online signals | You can't tune what you don't measure |
| Ingestion | Manual scripts | Versioned pipeline + quality gates | The index must be rebuildable; parse quality IS retrieval quality |
| Storage | One DB for everything | Tiered: object + SQL + vector + cache | Tiers differ in rebuild time, retention, and access pattern |
| Index deploy | Mutate in place | Blue/green + canary + rollback | In-place rebuild failure = silent quality regression |
| Versioning | None | Every artifact versioned, traced | Can't debug "it worked yesterday" without versions |
| Observability | Log print statements | Traces + dashboards + alerts | Failures are distributed across 6+ services |

---

## 8. Resource library

### Architecture patterns (the theory)

- Anthropic, **"Building effective agents"** — the agent loop formalized (workflows vs agents, tool loop, guardrails).
  https://www.anthropic.com/engineering/building-effective-agents
- OpenAI, **"A practical guide to building agents"** (PDF) — model selection, tools, orchestration, guardrails.
  https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf
- OpenAI, **"A practical guide to building RAG agents"** (PDF) — RAG-specific agent design from real deployments.
  https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-rag-agents.pdf
- Anthropic, **"Introducing Contextual Retrieval"** — prepend chunk context before embedding; big retrieval-quality win, fully explained.
  https://www.anthropic.com/news/contextual-retrieval
- Anthropic Engineering blog index — "How we built our multi-agent research system", "Harness design", "Scaling Managed Agents".
  https://www.anthropic.com/engineering
- Gergely Orosz (Pragmatic Engineer), **"How Claude Code is built"** — a real agentic chatbot's architecture, from the team that built it.
  https://newsletter.pragmaticengineer.com/p/how-claude-code-is-built
- Anthropic, **"The Making of Claude Code"** — inside story, tool loop, permissions design.
  https://www.anthropic.com/features/making-of-claude-code

### Production codebases to actually read (OSS)

- **Microsoft azure-search-openai-demo** ("Chat with your data") — the most complete production chatbot + RAG: approach planning, filters, citations, token budgeting. Start here.
  https://github.com/Azure-Samples/azure-search-openai-demo
- **LibreChat** — full-stack ChatGPT clone: thread trees, editing/regeneration, streaming, multi-provider. The closest thing to ChatGPT's actual wrapper, open source.
  https://github.com/danny-avila/LibreChat
- **Open WebUI** — another production-grade wrapper (conversation store, RAG, tools).
  https://github.com/open-webui/open-webui
- **OpenAI chatgpt-retrieval-plugin** — the retrieval architecture behind ChatGPT plugins (archived, still the canonical read).
  https://github.com/openai/chatgpt-retrieval-plugin

### RAG frameworks — learn from, don't marry

- **LlamaIndex** — read `ChatEngine`, `CondenseQuestionChatEngine`, `DocumentHierarchy` source; it's where the history-condensing + retrieval ideas live in code.
  https://github.com/run-llama/llama_index
- **LangChain** — `ConversationSummaryBufferMemory` docs for summarization-vs-truncation; hybrid search how-tos.
  https://python.langchain.com/docs/how_to/chatbots_memory/
- **Microsoft AI Agents for Beginners** — free course, has an agentic-RAG lesson.
  https://microsoft.github.io/ai-agents-for-beginners/05-agentic-rag

### Retrieval, chunking, hierarchy

- **RAPTOR** (arXiv:2401.18059) — recursive tree-structured retrieval; summarize up, retrieve down. The paper behind tree-hierarchical RAG.
  https://arxiv.org/abs/2401.18059
- **Qdrant filtering docs** — payload filters = the hard pre-filter layer in practice.
  https://qdrant.tech/documentation/concepts/filtering/
- **pgvector** — if you want one database (Postgres) for chunks + filters + vectors.
  https://github.com/pgvector/pgvector
- **Pinecone hybrid search explainer** — RRF fusion, when keyword beats vector.
  https://www.pinecone.io/learn/hybrid-search/
- **GPTCache** — semantic caching, runnable implementation (also the paper arXiv:2310.03014).
  https://github.com/zilliztech/GPTCache
- **tiktoken** — token counting; the basis of every context budget.
  https://github.com/openai/tiktoken

### Caching (the cost lever)

- Anthropic prompt caching docs (90% discount on cached tokens).
  https://docs.claude.com/en/docs/build-with-claude/prompt-caching
- OpenAI prompt caching docs.
  https://platform.openai.com/docs/guides/prompt-caching
- Anthropic token counting docs (budgeting foundation).
  https://docs.claude.com/en/docs/build-with-claude/token-counting

### Evaluation

- **RAGAS** — the standard open-source RAG evaluation framework (faithfulness, answer relevancy, context precision).
  https://github.com/explodinggradients/ragas
- **DSPy** — programmatic prompt/retrieval optimization with eval loops.
  https://github.com/stanfordnlp/dspy
- Anthropic, **"Demystifying evals for AI agents"** — how a lab actually evaluates agentic systems.
  https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- **DeepEval** — unit-test-style assertions for LLM apps (Pytest-style evals in CI).
  https://github.com/confident-ai/deepeval
- **promptfoo** — eval + red-teaming with CI integration.
  https://github.com/promptfoo/promptfoo
- **TruLens** — feedback and evaluation framework for RAG.
  https://github.com/truera/trulens

### Ingestion & parsing

- **IBM Docling** — production-grade document parsing: layout, tables, PDF → structured output. The standard for manual-style corpora.
  https://github.com/IBM/docling
- **Marker** — PDF → markdown with table handling.
  https://github.com/VikParuchuri/marker
- **PyMuPDF / pdfplumber** — low-level extraction when you need fine control.
  https://github.com/pymupdf/PyMuPDF
- **Apache Tika** — parsing toolkit in the JVM ecosystem.
  https://tika.apache.org
- **Orchestrators** for the ingestion pipeline: Airflow (https://airflow.apache.org), Prefect (https://github.com/PrefectHQ/prefect), Dagster (https://github.com/dagster-io/dagster)

### Observability

- **Langfuse** — open-source LLM observability: traces, tokens, cost per turn.
  https://github.com/langfuse/langfuse
- **OpenTelemetry GenAI semantic conventions** — the standard trace fields for LLM applications; implement your traces against these.
  https://opentelemetry.io/docs/specs/semconv/gen-ai/
- **Arize Phoenix** — open-source tracing + evals.
  https://phoenix.arize.com
- **LangSmith** — tracing/eval platform (LangChain ecosystem).
  https://www.langchain.com/langsmith
- **Datadog LLM Observability** — vendor option for the same telemetry.
  https://www.datadoghq.com/product/llm-observability/

### Streaming

- MDN Server-Sent Events — the transport every chat UI uses.
  https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events

### Case studies (real production numbers)

- **DoorDash, "How we built a RAG-powered knowledge assistant"** — production RAG for internal support at scale.
  https://careersatdoordash.com/blog/how-doordash-built-a-rag-powered-knowledge-assistant/
- **Shopify Engineering** — "How we built Shopify Sidekick" (production AI assistant), plus ongoing agent/RAG posts.
  https://shopify.engineering/how-we-built-shopify-sidekick
  https://shopify.engineering/latest
- **Business Compass, "RAG at Scale: Architecture, Bottlenecks, and Optimization Strategies"** — enterprise RAG bottlenecks and scaling.
  https://blogs.businesscompassllc.com/2026/03/rag-at-scale-architecture-bottlenecks.html

---

## 9. The mental model in one diagram

```
                    ┌──────────────────────────────────────────┐
                    │            CHAT ORCHESTRATOR             │
                    │  (stateless, horizontally scalable)      │
  Client ─────────► │                                          │
  {msg, conv,      │  load thread ──► budget ──► understand    │──► embedding svc
   parent}         │  retrieve ──► assemble ──► cache check    │──► vector DB + BM25
                    │  agent loop ──► stream via SSE           │──► reranker
                    │                                          │──► model API
                    └──────────┬───────────────────────────────┘
                               │
                    ┌──────────▼──────────┐        ┌─────────────────────┐
                    │ conversations +     │        │ chunk graph +       │
                    │ messages (tree)     │        │ metadata + cache    │
                    │ in Postgres/Redis   │        │ in vector DB/SQL    │
                    └─────────────────────┘        └─────────────────────┘
```

Three rules that govern everything in this document:

1. **The client never owns state** — threads, system prompts, and history are rebuilt server-side every turn.
2. **Retrieval is filters-first** — hard metadata before soft similarity, small chunks for precision, graph assembly for context.
3. **The model is the last resort** — cache, trim, route, and stream before you pay for generation.
4. **The index is a cache of the truth** — rebuildable from sources, versioned, quality-gated before every publish (§5.7–5.8, §6.6).
5. **Every turn is an eval data point** — versioned traces plus feedback feed the golden set; the flywheel (§6.6) is what makes the system improve instead of rot.