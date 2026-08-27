# 🚀 Handling 1 Million HTTP Requests Per Second — The Complete Engineering Guide

> **Source**: "Let's Handle 1 Million Requests per Second, It's Scarier Than You Think!" (2.5 hr video)
> **What this is**: A complete, structured breakdown of every concept, benchmark, bottleneck, and solution from the video. Every number, every command, every architecture decision — preserved and organized for learning.

---

## Table of Contents

1. [Context: How Insane is 1M RPS?](#1-context-how-insane-is-1m-rps)
2. [Prerequisites & Mindset](#2-prerequisites--mindset)
3. [CPU Utilization & Threading Deep Dive](#3-cpu-utilization--threading-deep-dive)
4. [Benchmarking Fundamentals with Autocannon](#4-benchmarking-fundamentals-with-autocannon)
5. [Framework Shootout: Express vs Fastify vs Cpeak](#5-framework-shootout-express-vs-fastify-vs-cpeak)
6. [Local Machine Benchmarks](#6-local-machine-benchmarks)
7. [Moving to AWS — The Power Machines](#7-moving-to-aws--the-power-machines)
8. [The Simple Route: 6M RPS (Easy Mode)](#8-the-simple-route-6m-rps-easy-mode)
9. [The Patch Route: Network Bottleneck](#9-the-patch-route-network-bottleneck)
10. [Database Writes: PostgreSQL Bottleneck](#10-database-writes-postgresql-bottleneck)
11. [Database Reads: O(N) vs O(1) — The $33K Lesson](#11-database-reads-on-vs-o1--the-33k-lesson)
12. [Redis: The Game Changer](#12-redis-the-game-changer)
13. [Redis Clustering: Hitting 1M Writes/Second](#13-redis-clustering-hitting-1m-writessecond)
14. [Node.js Hits Its Ceiling](#14-nodejs-hits-its-ceiling)
15. [C++ & Drogon: The Final Push to 1M RPS](#15-c--drogon-the-final-push-to-1m-rps)
16. [The Final Boss: 60 Servers, 2 Billion Requests](#16-the-final-boss-60-servers-2-billion-requests)
17. [AWS Load Balancer Limits (Amazon's Own Reply)](#17-aws-load-balancer-limits-amazons-own-reply)
18. [Cost Breakdown & Real-World Perspective](#18-cost-breakdown--real-world-perspective)
19. [Key Engineering Principles](#19-key-engineering-principles)
20. [Action Items & Practice Projects](#20-action-items--practice-projects)

---

## 1. Context: How Insane is 1M RPS?

### The World's Busiest Route

**AWS IAM** (Identity and Access Management) — the security guard for all AWS services — handled **400 million requests per second** globally a few years ago. That's the current record for a single service.

### What Would 1M RPS Cost at API Pricing?

| API Service | Price per Request | Monthly Cost at 1M RPS | Monthly Cost (full) |
|------------|-------------------|----------------------|---------------------|
| OpenWeatherMap | $0.0015/call | **$3.8 billion** | Absurd |
| Typical API ($0.90/million) | $0.0000009 | **$3.9 million** | Still insane |
| Cloudflare Workers | ~$0.50/million | **~$1 million** | More realistic but still costly |
| Google Maps API | Varies | **Millions** | Not cost-effective |

> [!CAUTION]
> At this scale, launching your own infrastructure is **orders of magnitude cheaper** than using per-request API pricing. This is why companies like Uber and Netflix run their own servers.

### How Real Companies Handle It

They **don't** run one massive supercomputer. They distribute across:
- Multiple servers per region
- Load balancers routing by geography
- Each server handles ~500K RPS
- Scale horizontally by adding servers to hot regions

---

## 2. Prerequisites & Mindset

### What You Need to Know

| Prerequisite | Level Needed |
|-------------|-------------|
| SQL | `SELECT`, `INSERT`, `UPDATE` — basics |
| Backend development | Built at least one API in any language |
| Computer hardware | CPU cores, threads, RAM vs disk, network card |
| Networking | What happens when you SSH somewhere |
| Terminal | `cd`, `mkdir`, `rm`, `cat` — basics |
| Node.js | Know what it is (not React — it's system-level) |
| Bytes | 1 byte = 8 bits, 1 GB = 8 Gb |

### The Mindset Shift

> [!WARNING]
> **"The concept of 'code that just works is good enough' is ridiculous in a high-stakes environment. That mentality could cost a company literally millions of dollars."**

Key principles at this scale:

| Normal Scale | 1M RPS Scale |
|-------------|-------------|
| "It works" | "What's the Big-O complexity?" |
| Bug → fix it | Bug → **unfathomable** (you don't even get close to having one) |
| O(N) vs O(log N)? "Doesn't matter" | The difference = **tens of thousands of dollars** |
| 1-in-a-million chance? Ignore it | **It happens every minute** |
| "I'm just a programmer" | "I'm an engineer who uses math" |

---

## 3. CPU Utilization & Threading Deep Dive

### Core Utilization Formula

```
Core Utilization = ((Total Time - Idle Time) / Total Time) × 100
```

**Example**: In the last hour, core was idle for 30 minutes → `(60 - 30) / 60 × 100 = 50%`

### CPU Utilization: Two Methods

| Method | Formula | Example (4 cores, all at 100%) |
|--------|---------|-------------------------------|
| **Method 1** (Sum) | Add all core utilizations | **400%** |
| **Method 2** (Average) | Sum ÷ Number of cores | **100%** |

> Different tools use different methods. macOS Activity Monitor uses Method 1. Linux `mpstat` typically shows per-core + averages.

### Threading Fundamentals

**Single Thread**:
- One thread = one core utilized
- `while(true)` → that core hits 100%
- Other cores sit idle

**Multi-Thread**:
- Spawn N threads to utilize N cores
- Each thread can run independently on its own core
- **Must handle**: race conditions, semaphores, locks

### Practical Experiment

```javascript
// singlethread.js — uses 1 core
while (true) {} // Check task manager: ~100% on one core

// multithread.js — uses all cores
const { Worker } = require('worker_threads');
const numCores = 12; // Change to YOUR core count
for (let i = 0; i < numCores; i++) {
  new Worker(`while(true){}`, { eval: true });
}
// Check task manager: idle CPU → 0%
```

> [!TIP]
> **Run this yourself.** Open Activity Monitor / Task Manager and watch. Understanding these numbers is the foundation for everything that follows.

---

## 4. Benchmarking Fundamentals with Autocannon

### Installation

```bash
npm i -g autocannon
```

### How Autocannon Works

```
autocannon -c <connections> -d <duration> -p <pipeline> -W <workers> -m <method> <url>
```

| Flag | Meaning | Example |
|------|---------|---------|
| `-c` | Total TCP connections to open | `-c 20` |
| `-d` | Duration in seconds | `-d 20` |
| `-p` | Requests pipelined per connection | `-p 2` |
| `-W` | Worker threads to spawn | `-W 6` |
| `-m` | HTTP method | `-m GET` |
| `-b` | Request body (JSON) | `-b '{"key":"value"}'` |

### Calculating Concurrent Requests

```
Concurrent Requests = connections × pipelining
```

**Example**: `-c 6 -p 2` → **12 requests** being handled simultaneously at any instant.

### How Workers Distribute Connections

```
Connections per worker = total connections / workers
```

**Example**: `-W 2 -c 6` → **3 connections per worker thread**

### Reading Results

| Metric | What It Means |
|--------|---------------|
| **Avg Req/Sec** | The number you care about most |
| **Latency** | How long each request took (p50, p99) |
| **Total Requests** | Sum of all requests during the test |
| **Bytes Read** | Total data transferred |
| **Errors** | Timeouts, non-2xx responses |

---

## 5. Framework Shootout: Express vs Fastify vs Cpeak

### Benchmark Results (Simple JSON Response)

| Framework | Avg RPS | vs Express |
|-----------|---------|-----------|
| **Express** | ~20,000 | 1× (baseline) |
| **Fastify** | ~66,000 | **3.3×** faster |
| **Cpeak** (custom, 500 LOC) | ~73,000 | **3.6×** faster |

### Why Cpeak?

- **Zero dependencies** — only ~500 lines of code
- Performance comparable to Fastify
- Educational — you can read and understand the entire source
- Same API patterns as Express

### Why Not Express at Scale?

Express adds overhead on every request. At 1M RPS, even microseconds per request compound:

```
1,000,000 × 0.00005 seconds overhead = 50 seconds of CPU wasted per second
```

> [!IMPORTANT]
> **Framework choice matters at scale.** Express is fine for 99% of applications. But when you're pushing hardware limits, that 3× difference is the difference between needing 1 server and needing 3.

---

## 6. Local Machine Benchmarks

### Test Machine: Mac Studio

| Spec | Value |
|------|-------|
| CPU Cores | 12 |
| RAM | 32 GB |
| Network | 10 Gbps (1.25 GB/s) |
| Cost | ~$2,000 (one-time) / ~$60/month amortized |

### Route: `/simple` (GET → returns `{message: "hi"}`)

| Setup | RPS | Idle CPU |
|-------|-----|----------|
| Single Node process, Express | ~18,000 | Very high |
| Single Node process, Cpeak | ~73,000 | Still high |
| **12 instances via PM2 (cluster mode)** | **~50,000** | **~0%** |

### Route: `/patch` (PATCH → params + query + 32KB JSON response)

| Setup | RPS | Notes |
|-------|-----|-------|
| Single Node process | ~8,000 | 1 core at 100%, rest idle |
| PM2 cluster (12 instances) | ~42,000 | All cores utilized |
| PM2 (no recording overhead) | ~50,000 | Closer to real max |

### Key Insight: PM2 Cluster Mode

```bash
pm2 start ecosystem.config.js  # Starts N instances (N = CPU cores)
```

How it works:
```
                    ┌─► Node Process 1
                    ├─► Node Process 2
Incoming Traffic ──►│   ...
(via parent proc.)  ├─► Node Process 11
                    └─► Node Process 12
```

The parent Node.js process receives all traffic and **round-robin distributes** to child processes. Each child runs on its own core.

---

## 7. Moving to AWS — The Power Machines

### Architecture

```
┌─────────────────┐         Private Network        ┌──────────────────┐
│  Power Tester    │◄──────────────────────────────►│   Power Server    │
│                  │                                │                  │
│  128 CPU cores   │       autocannon traffic       │  128 CPU cores   │
│  256 GB RAM      │ ────────────────────────────►  │  256 GB RAM      │
│  50 Gbps network │                                │  50 Gbps network │
│                  │                                │                  │
│  $6/hr           │                                │  $6/hr           │
│  C8i.32xlarge    │                                │  C8i.32xlarge    │
└─────────────────┘                                └────────┬─────────┘
                                                            │
                                                    ┌───────▼──────────┐
                                                    │   PostgreSQL DB   │
                                                    │                  │
                                                    │  64 CPU cores    │
                                                    │  256 GB RAM      │
                                                    │  DB.M5.16xlarge  │
                                                    │  $6/hr           │
                                                    └──────────────────┘
```

### Machine Comparison

| Machine | CPU Cores | RAM | Network | Cost |
|---------|-----------|-----|---------|------|
| Mac Studio | 12 | 32 GB | 10 Gbps | $60/mo |
| **Power Server** (C8i.32xlarge) | **128** | **256 GB** | **50 Gbps** | **$5,000/mo** |
| **Beast Server** (C8gn.48xlarge) | **192** | **384 GB** | **600 Gbps** | **$8,000/mo** |

### Why Two Separate Machines?

On the Mac Studio, both **generating** and **handling** traffic ran on the same machine — they competed for CPU. Now:
- **Tester** = dedicated to generating traffic
- **Server** = dedicated to handling traffic
- Results are clean and uncontaminated

> [!CAUTION]
> **Cost warning**: This setup costs **$30+/hour** (~$20,000/month). Every minute of debugging costs money. Know what you're doing before launching cloud infrastructure at this scale.

---

## 8. The Simple Route: 6M RPS (Easy Mode)

### Setup
- 128 Node.js instances via PM2
- autocannon: 1000 connections, pipelining 100, 120 worker threads

### Results

| Metric | Value |
|--------|-------|
| **Avg RPS** | **6,000,000** 🤯 |
| Server CPU (idle) | 0% — fully utilized |
| Tester CPU (idle) | 50% — room to spare |

### Why So High?

- Returning just `{"message": "hi"}` — tiny payload
- No I/O (no database, no disk, no external calls)
- Pure CPU-bound response generation
- 128 cores all working in parallel

> But this is meaningless in the real world — no production API just returns "hi".

---

## 9. The Patch Route: Network Bottleneck

### The Route

```javascript
app.patch('/patch/:id/:name', (req, res) => {
  // Validate ID is a number
  // Generate ~30KB of dummy JSON data (array of 100 objects)
  // Return response
});
```

### First Attempt: 100K RPS (Expected 1M+)

| Metric | Value |
|--------|-------|
| Avg RPS | **100,000** |
| Server CPU (idle) | **50%** ← not fully utilized! |
| Data transferred (20s) | **120 GB** |

### The Bottleneck Discovery

```
120 GB ÷ 20 seconds = 6 GB/s = 48 Gbps
```

The machine's network limit is **50 Gbps (6.25 GB/s)**. **The network card was the bottleneck, not the CPU!**

### Fix Attempt: Reduce Response Size

Changed array length from 100 → 3 objects (~1 KB response instead of ~30 KB):

| Metric | Before | After |
|--------|--------|-------|
| Response Size | ~30 KB | ~1 KB |
| RPS | 100,000 | **3,000,000** |
| Network saturated? | Yes | No — CPU is now the limit |

### Network Speed Tiers Available on AWS

| Network Speed | Cost/month (approx) | Can Handle at 30KB/req |
|---------------|---------------------|----------------------|
| 50 Gbps | $5,000 | ~200K RPS |
| 100 Gbps | $8,000+ | ~400K RPS |
| 200 Gbps | $15,000+ | ~800K RPS |
| 600 Gbps | $8,000 (beast) | **1M+ RPS** ✅ |
| 3,000 Gbps | ~$30,000 | Overkill (near-supercomputer) |

> [!IMPORTANT]
> **The bottleneck changes depending on your workload:**
> - Tiny responses → CPU-bound
> - Large responses → Network-bound
> - Database queries → Disk I/O-bound
> - Heavy computation → CPU-bound
> 
> **Resource monitoring is non-negotiable at this scale.** If you're not monitoring, you're guessing.

---

## 10. Database Writes: PostgreSQL Bottleneck

### The Route

```javascript
app.post('/code', async (req, res) => {
  const code = generateRandomCode(500); // 500 char string
  const result = await db.query(
    'INSERT INTO codes (code) VALUES ($1) RETURNING *', [code]
  );
  res.json(result.rows[0]);
});
```

### Benchmark Progression

| IOPS Config | Writes/sec | Cost/month | Server CPU |
|-------------|-----------|-----------|-----------|
| 3,000 IOPS | **35,000** | $5,000 | Nearly idle |
| 12,000 IOPS (+4×) | **66,000** (+1.9×) | $6,000 (+$1K) | Nearly idle |

### Why Only 2× Improvement for 4× More IOPS?

Because IOPS isn't the only bottleneck. **Database CPU hit 100%** during benchmarks:
- 128 Node instances × 10 connections each = **1,280 concurrent connections**
- PostgreSQL `max_connections` was 5,000 (not the limit)
- But the **64-core DB CPU** was fully saturated

### Cost to Hit 1M Writes/sec with PostgreSQL

| Scaling Approach | Estimated Cost/month |
|-----------------|---------------------|
| Single massive instance | $15,000–$33,000 |
| Aurora Auto-scaling | $20,000–$30,000 |
| Multiple read replicas | $14,000+ (reads only) |

> [!WARNING]
> **Verdict**: Hitting 1M writes/sec directly to PostgreSQL is possible but **insanely expensive**. There's a better way — Redis + batch sync (covered later).

### Connection Pool Math

```
Total DB Connections = Node instances × pool size per instance
                    = 128 × 10
                    = 1,280 connections

PostgreSQL max_connections = 5,000 (can handle ~4× more)
```

---

## 11. Database Reads: O(N) vs O(1) — The $33K Lesson

### Four Versions of the Same Read Query

```sql
-- V1: ORDER BY RANDOM() ← O(N) — scans entire table
SELECT id, code FROM codes ORDER BY RANDOM() LIMIT 1;
-- Result: 43 SECONDS for a single query with 10M records 💀

-- V2: SELECT COUNT(*) then random ID ← O(N) — count scans full table
SELECT COUNT(*) FROM codes;
-- Then: SELECT * FROM codes WHERE id = random(1, count);
-- Result: ~40 seconds — still terrible

-- V3: SELECT MAX(id) then random ID ← O(log N) with index
SELECT id FROM codes ORDER BY id DESC LIMIT 1;
-- Then: SELECT * FROM codes WHERE id = random(1, max_id);
-- Result: 200,000 RPS ✅

-- V4: Pure random ID lookup ← O(1) with primary key index
SELECT * FROM codes WHERE id = $1;  -- random ID between 1 and 10M
-- Result: 400,000 RPS ✅✅
```

### Performance Comparison

| Version | Time Complexity | RPS with 10M rows | Works? |
|---------|-----------------|-------------------|--------|
| V1 (`ORDER BY RANDOM()`) | O(N) | ~0.02 (43 sec/query) | ❌ **Crashed** |
| V2 (`COUNT(*)`) | O(N) | ~0.025 (40 sec/query) | ❌ **Crashed** |
| V3 (`MAX(id)` + random) | O(log N) + O(1) | **200,000** | ✅ |
| V4 (direct ID lookup) | O(1) | **400,000** | ✅✅ |

### The Lesson

> *"This is why you got to know algorithms if you want to move into such a high-stakes environment. You make one simple mistake, it could cost you a whole lot down the line."*

```
V1 at scale = company-killing decision
V4 at scale = 400,000 reads/second on same hardware

Same feature. Same database. Same server. 20,000,000× difference.
```

> [!CAUTION]
> `ORDER BY RANDOM()` is a beloved tutorial pattern that **destroys production databases**. It performs a full table scan, generates a random value for every row, sorts the entire result set, then returns the first row. With 10M rows, this is catastrophic.
>
> **Always use indexed lookups** for high-throughput reads.

---

## 12. Redis: The Game Changer

### Why Redis?

| Storage | Access Speed | Use Case |
|---------|-------------|----------|
| **SSD (disk)** | ~100 μs | Persistent data (PostgreSQL) |
| **RAM (memory)** | ~0.1 μs (~1000× faster) | Hot data (Redis) |

### Architecture: Redis + Background Sync

```
Hot Path (real-time):                    Cold Path (batch):
                                         
Client ──► Node ──► Redis (RAM)          Redis Queue ──► Sync Worker ──► PostgreSQL
              └──► SQS/Queue                              (overnight / background)
                  (save task ID)
```

### Write Benchmark: Redis vs PostgreSQL

| Storage | Writes/sec | Cost to hit 1M | CPU Usage |
|---------|-----------|----------------|-----------|
| PostgreSQL (single) | 35,000–66,000 | $15K–$33K/mo | DB CPU at 100% |
| **Redis (single instance)** | **100,000** | Already running on same server | Server at 80% idle |

### Migration: Move PostgreSQL → Redis

```javascript
// migrate.js — batch move from PostgreSQL to Redis
// 1. Flush Redis
// 2. SELECT id, code, created_at FROM codes (in batches of 2,000)
// 3. HSET each record into Redis
// 4. Track max ID for future writes
```

The entire 16 GB PostgreSQL database moved into RAM (machine had 256 GB — plenty of room).

### Read Benchmark: Redis vs PostgreSQL

| Storage | Reads/sec | Notes |
|---------|-----------|-------|
| PostgreSQL V4 (ID lookup) | 400,000 | Hitting DB CPU limit |
| **Redis (single instance)** | **300,000** | Single-threaded Redis limit |

Wait — Redis reads are *slightly slower*? Yes, because **a single Redis instance is single-threaded** and caps at ~100K–300K ops/sec. The fix: **clustering**.

### The Batch Sync Pattern (Real-World)

> *"This is what Uber and these companies with insane traffic do. Save driver locations to Redis, sync to SQL overnight."*

```
Real-time writes ──► Redis (fast, in-memory)
                         │
                    Background Sync (batch)
                         │
                         ▼
                    PostgreSQL (durable, queryable)
```

---

## 13. Redis Clustering: Hitting 1M Writes/Second

### Why Cluster?

Single Redis instance = single-threaded = ~100K–200K ops/sec max. 
To go beyond, you need **multiple instances** working in parallel.

### Redis Cluster Setup

```bash
redis.sh --setup  # Launches 30 Redis instances
                   # 15 masters + 15 replicas
```

### How Redis Cluster Distributes Data

```
                    ┌─► Redis Instance 1 (Master) ◄─► Replica 1
                    │   Slots: 0–5460
                    │
HSET {shard}:key ──►├─► Redis Instance 2 (Master) ◄─► Replica 2
  (hashed)          │   Slots: 5461–10922
                    │
                    └─► Redis Instance 3 (Master) ◄─► Replica 3
                        Slots: 10923–16383
```

The content inside `{braces}` is hashed to determine which node gets the data. This is called **hash slot routing** (16,384 total slots distributed across masters).

### Code Changes for Cluster Mode

```javascript
// Single Redis:
await redis.hset('code:123', data);

// Cluster Redis — add shard key:
await redis.hset('{shard1}:code:123', data);
// The {shard1} determines which node stores this data
```

### UUID Instead of Sequential IDs

At 1M writes/sec, maintaining a sequential counter requires an extra Redis write per request (checking uniqueness). Solution: **UUID v4 (122-bit random)**:

```javascript
const id = crypto.randomUUID(); // 122-bit random UUID
```

**Collision probability** (birthday paradox):
> At 1 million UUIDs/second, it takes **86,000 years** to reach a 50% probability of a single collision.

### The Moment of Truth: 1M Writes/Second

```
autocannon POST /code-ultra-fast
  → 128 Node instances
  → 30 Redis instances (15 masters + 15 replicas)
  → All CPU cores near 0% idle
```

| Metric | Value |
|--------|-------|
| **Avg RPS** | **1,000,000+** ✅🎉 |
| Server CPU idle | ~0% |
| Memory used | 100 GB / 256 GB |

### Memory Management Warning

At 1M writes/sec, memory fills fast:
- Each test run: ~20M records added
- After 5 runs: ~100M records, 100 GB RAM consumed
- **Must batch-sync to PostgreSQL and clear Redis regularly**

---

## 14. Node.js Hits Its Ceiling

### The Problem with the Patch Route

Even on the **Beast machine** (192 cores, 600 Gbps network), Node.js topped out at:

| Framework | Max RPS (patch route, 30KB response) | CPU Idle |
|-----------|--------------------------------------|----------|
| **Express** | ~500,000 | 0% |
| **Cpeak/Fastify** | ~700,000–800,000 | 0% |

### Why Node.js Can't Reach 1M Here

1. **JavaScript overhead** — V8 engine adds per-request cost for string manipulation, JSON serialization
2. **PM2 parent process bottleneck** — all traffic flows through one parent process before distribution
3. **CPU-intensive operations** — input validation, data generation, JSON serialization are expensive in JS

### Other Languages Tried

| Language | Could hit 1M? | Notes |
|---------|---------------|-------|
| Python | ❌ | Even slower than Node |
| Java + Spring | ❌ | Better but still not enough |
| Go | ❌ | Close but no cigar |
| **C++** | **✅** | Only language that made it |

> [!IMPORTANT]
> **The lesson isn't "don't use Node.js."** Node.js handled 6M RPS on the simple route and 1M writes/sec with Redis. The lesson is: **for CPU-intensive response generation at extreme scale, you need a systems language.**

---

## 15. C++ & Drogon: The Final Push to 1M RPS

### The Framework: Drogon

[Drogon](https://github.com/drogonframework/drogon) — one of the **fastest web frameworks in the world** (C++17). Key features:
- Multi-threaded by default (no PM2 needed)
- Non-blocking I/O
- Built-in JSON, ORM, WebSocket support

### The JSON Parser Problem

| JSON Library | Performance vs Node V8 |
|-------------|----------------------|
| Drogon default (JsonCpp) | **4× slower than Node.js** 😱 |
| **RapidJSON** | **Faster than V8** ✅ |

> Even C++ can be slow with the wrong library. The default JSON parser of Drogon was **four times slower** than Node.js V8 engine's JSON parser.

### Key Optimizations

```cpp
// 1. Use RapidJSON instead of default JsonCpp
// 2. Set thread count to match CPU cores (minus a few for OS)
app.setThreadNum(180); // 192 cores, leave 12 for OS/overhead

// 3. Disable compression (for fair benchmark — response stays 30KB)
// 4. Disable logging (every log = wasted CPU cycles)
```

### Results: C++ Drogon vs Node.js

| Metric | Node.js (Cpeak) | C++ (Drogon) |
|--------|----------------|-------------|
| Avg RPS | ~800,000 | **1,000,000–1,200,000** ✅ |
| CPU Utilization | 100% | **70%** (room to spare!) |
| Memory Usage | Moderate | **Negligible** |
| Data Transferred | ~20 GB/s | **38 GB/s** (304 Gbps) |

### The Numbers in Perspective

```
38 GB/s network throughput = 304 Gbps
                           = 8× faster than a fast SSD's read speed
                           = 2 TB of data transferred per minute
```

> *"It's like copying a 2 TB SSD to another 2 TB SSD in one minute."*

---

## 16. The Final Boss: 60 Servers, 2 Billion Requests

### Why 60 Small Servers Instead of 1 Big One?

The single tester machine couldn't open enough connections to utilize the remaining 30% of the beast server's CPU. Solution: **distribute the testing load**.

### Architecture

```
     ┌─── c8gn.2xlarge (8 cores, 16 GB) ──► autocannon
     ├─── c8gn.2xlarge ──► autocannon
     ├─── c8gn.2xlarge ──► autocannon
     │    ... (60 total)
     ├─── c8gn.2xlarge ──► autocannon           ┌──────────────────┐
     └─── c8gn.2xlarge ──► autocannon ────────► │   Beast Server    │
                                                 │  C8gn.48xlarge   │
         Total: 60 × 400 connections             │  192 CPU cores   │
              = 24,000 connections               │  384 GB RAM      │
         × 5 pipelining = 120,000                │  600 Gbps net    │
         concurrent requests                     │  C++ / Drogon    │
                                                 └──────────────────┘
```

### AWS Limits Encountered

- **Default**: 32 CPU cores per region
- **Requested increase**: 800 CPU cores
- **60 servers × 8 cores = 480 cores** (within limit)
- Tried 100 servers — hit the 800-core cap

### The 30-Minute Test Results

| Metric | Value |
|--------|-------|
| **Total Requests** | **2,000,000,000** (2 billion) |
| **Data Transferred** | **60+ TB** |
| **Duration** | 30 minutes |
| **Avg RPS** | **1,000,000+** consistently |
| **Timeouts** | **40** out of 2 billion (0.000002%) |
| **Server Status** | "Did not break a sweat" |

### Automation with AWS SSM

Instead of SSH-ing into 60 servers individually:

```bash
# Send command to ALL 60 servers simultaneously
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --parameters commands="cd /app && node patch.js 400 1800 20 $HOST" \
  --max-concurrency "100%" \  # All at once, not batched
  --output-s3-bucket-name "my-results" \
  --cloud-watch-output-config ...
```

### Result Aggregation with Bash

```bash
# Download all results from S3
aws s3 sync s3://bucket/command-id ./results

# Concatenate all 60 result files
find ./results -path "*/stdout" -exec cat {} \; > 30min-60-results.txt

# Count test outputs (should be 60)
grep "read" 30min-60-results.txt | wc -l  # → 60 ✅

# Calculate total requests and data
grep "read" 30min-60-results.txt | awk '
  { req += $1; data += $5 }
  END { print "Total requests:", req/1000, "K";
        print "Total data:", data, "TB" }'
# → 2,000,000K requests, 60+ TB
```

### Electricity Perspective

> *"With the electricity used in 1 hour of this test, you could power a Tesla to drive thousands of kilometers."*

---

## 17. AWS Load Balancer Limits (Amazon's Own Reply)

### The Problem

With a **Network Load Balancer (NLB)** in front of 2 beast servers, performance **degraded drastically**:

| Setup | Throughput |
|-------|-----------|
| Direct to server (no LB) | 38 GB/s |
| Through NLB | **5 GB/s** (7.6× worse) |

### Amazon's Response

> *"Based on our metrics, the consumed load balancer capacity reached 165, which was the limit."*

**Solution**: You must **pre-register capacity** with AWS for extreme traffic:

```
AWS Console → Load Balancer → Reserve Capacity
- Expected bandwidth: 600 Gbps
- Expected connections: 100,000
- Availability Zones: 1
→ AWS calculates required LCUs and provisions accordingly
```

> [!NOTE]
> **Even AWS has hidden limits.** At extreme scale, you need to talk to AWS support and pre-provision resources. The "auto-scaling" marketing has real ceilings.

### Even AI Couldn't Help

> *"They ran an AI to solve my problem — gave 10 suggestions, none worked. At this scale, even AI can't help because only very few companies handle 1M RPS, so AIs didn't have enough data to train on."*

---

## 18. Cost Breakdown & Real-World Perspective

### Total Cost for All Video Testing

| Category | Cost |
|----------|------|
| EC2 Compute | ~$1,200 |
| Databases | ~$800 |
| **Total** | **~$2,000** |

### Running Cost at Scale (Monthly)

| Component | Cost/month |
|-----------|-----------|
| Power Server (C8i.32xlarge) | $5,000 |
| Beast Server (C8gn.48xlarge) | $8,000 |
| PostgreSQL (DB.M5.16xlarge) | $5,000–$7,000 |
| 60 Test Servers | $20,000 |
| PostgreSQL scaled for 1M writes | $15,000–$33,000 |

### Real-World Architecture (Not What We Did)

```
                    ┌── Server (NY) ─── 500K RPS ──┐
                    │                               │
Users Worldwide ──► ├── Server (EU) ─── 500K RPS ──┼──► Total: 1M+ RPS
  (via DNS/CDN)     │                               │
                    ├── Server (APAC) ── 300K RPS ──┤
                    │                               │
                    └── Server (SA) ─── 200K RPS ──┘
```

---

## 19. Key Engineering Principles

### The Hierarchy of Bottlenecks

```
1. Code Complexity (O(N) vs O(1))     ← Fix this FIRST
2. Framework Overhead                  ← Choose wisely
3. Language Performance                ← Switch if needed (Node → C++)
4. Database I/O (Disk)                 ← Move to RAM (Redis)
5. Network Bandwidth                   ← Upgrade instance type
6. CPU Cores                           ← Add more machines
7. Cloud Provider Limits               ← Talk to support
```

### Critical Rules at Scale

| Rule | Why |
|------|-----|
| **Monitor everything** | If you're not monitoring CPU, memory, network, disk — you're blind |
| **Know your bottleneck** | Is it CPU? Network? Disk I/O? Database? Don't guess — measure |
| **Big-O matters more than language** | O(N) in C++ is still slower than O(1) in Python at scale |
| **Framework overhead compounds** | 50μs × 1M = 50 seconds of waste per second |
| **Memory is 1000× faster than disk** | Redis (RAM) vs PostgreSQL (disk) — use both strategically |
| **Horizontal scaling isn't free** | Connection overhead, coordination, consistency — all add complexity |
| **Simple mistakes are catastrophic** | `ORDER BY RANDOM()` can crash a $33K/month database |

### When to Use What

| Scenario | Technology |
|----------|-----------|
| Hot read/write path (millions/sec) | **Redis** (in-memory) |
| Durable storage, complex queries | **PostgreSQL** (batch sync from Redis) |
| API at moderate scale | **Node.js / Python / Java** |
| API at extreme scale (CPU-intensive) | **C++ / Rust / Go** |
| Static responses at extreme scale | **Node.js is fine** (6M RPS proven) |

---

## 20. Action Items & Practice Projects

### Immediate (Do This Week)

- [ ] **Run the CPU utilization experiment** — single thread and multi-thread, watch your monitor
- [ ] **Install autocannon** and benchmark a simple Express/Fastify server
- [ ] **Set up PostgreSQL locally**, insert 1M records, try `ORDER BY RANDOM()` and see it crawl
- [ ] **Set up Redis locally**, benchmark SET/GET operations

### Short-Term Projects

- [ ] **Build a URL shortener** that handles high traffic:
  - Express/Fastify for the API
  - Redis for hot lookups (recent URLs)
  - PostgreSQL for durable storage
  - Background sync worker
- [ ] **Benchmark your shortener** with autocannon
  - Single process vs PM2 cluster
  - Redis vs PostgreSQL reads
  - Different connection counts and pipelining values
- [ ] **Set up resource monitoring** — `mpstat`, `htop`, `free`, `iftop`

### Advanced Projects

- [ ] **Rewrite the hot path in C++** using Drogon
  - Same API, compare RPS with Node.js
  - Experiment with RapidJSON vs default
- [ ] **Set up Redis Cluster** (even locally with the provided redis.sh script)
  - Understand hash slot routing
  - Test writes across the cluster
  - Implement batch sync to PostgreSQL
- [ ] **Deploy on AWS** (carefully, with budget alerts!):
  - Launch 2 small EC2 instances (one tester, one server)
  - Benchmark over the private network
  - Compare with local results

### Key Tools to Master

| Tool | Purpose |
|------|---------|
| `autocannon` | HTTP benchmarking |
| `mpstat` | Per-core CPU monitoring |
| `htop` / `top` | Process monitoring |
| `free -h` | Memory usage |
| `iftop` / `nload` | Network bandwidth monitoring |
| `PM2` | Node.js cluster management |
| `redis-cli` | Redis interaction |
| `psql` | PostgreSQL interaction |
| `aws ssm` | Remote command execution at scale |
| `awk` / `grep` | Log parsing and aggregation |

### Repositories from the Video

| Repo | What's In It |
|------|-------------|
| Node 1M RPS | Express, Fastify, Cpeak routes + benchmarks |
| CPP 1M RPS | Drogon C++ server with RapidJSON |
| 1M-RPS Tester | Autocannon Node.js script for distributed testing |
| Cpeak Framework | Zero-dependency Node.js web framework (~500 LOC) |

---

> *"It takes a lot of engineering, a whole lot of effort, and only very few companies in the world would ever get to this scale of 1 million requests per second."*
>
> *"Every single bit that you can save is going to really add up."*
>
> *"A simple mistake here would cost your company tens of thousands of dollars."*

**Now go build something and break it with traffic.** That's how you learn.
