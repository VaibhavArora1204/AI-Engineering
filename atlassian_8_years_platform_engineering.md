# 🏗️ 8 Years Building Atlassian's Edge Infrastructure — From Interview to Layoff

> **Source**: "I was laid off by Atlassian" by an ex-Atlassian platform engineer (8-year tenure)
> **Why this matters**: This is a rare, honest post-mortem of building **production infrastructure at scale** — the proxy layer, control planes, sidecars, and the soft skills nobody talks about. Every section maps to real systems you'll encounter in any cloud-native company.

---

## Table of Contents

1. [The Interview — What Actually Gets You Hired](#1-the-interview--what-actually-gets-you-hired)
2. [Months 1–2: The Open Service Broker](#2-months-12-the-open-service-broker)
3. [The Envoy Control Plane (Sovereign)](#3-the-envoy-control-plane-sovereign)
4. [Infrastructure as Code — CloudFormation + Packer + SaltStack](#4-infrastructure-as-code--cloudformation--packer--saltstack)
5. [Migrating Jira, Confluence, Bitbucket onto the Edge](#5-migrating-jira-confluence-bitbucket-onto-the-edge)
6. [Programmable Proxy with Sidecars](#6-programmable-proxy-with-sidecars)
7. [Soft Skills That Actually Matter](#7-soft-skills-that-actually-matter)
8. [Build vs. Maintain — The Real Cost](#8-build-vs-maintain--the-real-cost)
9. [Mentoring — The Hardest Skill](#9-mentoring--the-hardest-skill)
10. [Key Takeaways & Action Items](#10-key-takeaways--action-items)

---

## 1. The Interview — What Actually Gets You Hired

### The Process
| Stage | What Happened | What It Tests |
|-------|---------------|---------------|
| **HackerRank** | Coding quiz — he aced it with full marks | Basic coding ability (table stakes) |
| **Tech Interview 1** | Given a **Cloudflare white paper on custom domains** to read for 10 min, then questioned | **Can you absorb unfamiliar material fast?** This is the real job. |
| **Tech Interview 2** | Troubleshooting a **real Atlassian incident** (app-level DoS) by prompting the interviewer for info | **Debugging instinct** — do you ask the right questions? |
| **DNS question** | How does latency-based DNS (Route 53) work? His answer was wrong but principled | **First-principles thinking** > memorization |
| **Values interview** | Standard culture fit | Soft skills |

### 💡 The Hiring Moment

He asked the interviewers:
> *"12 months from now, looking back — what would I need to have achieved for you to say hiring me was a good decision?"*

They described a **self-service load balancer platform** (an Open Service Broker). He said he could build it despite not knowing the framework, because he had **confidence in building web apps with Python**.

> [!TIP]
> **Lesson**: The question that got him hired wasn't a coding trick — it was a **product-minded question** that showed ownership. Ask this in every interview.

---

## 2. Months 1–2: The Open Service Broker

### What is an Open Service Broker?

A standardized API that lets platform users **self-service provision resources** (load balancers, databases, DNS records) without bothering the platform team.

- **Spec**: [Open Service Broker API](https://github.com/openservicebrokerapi/servicebroker) — originally designed for Kubernetes
- Endpoints: `/v2/catalog`, `/v2/service_instances/:id` (PUT/PATCH/DELETE), binding endpoints

### Architecture

```
┌─────────┐      HTTP       ┌──────────────┐     SQS      ┌──────────┐
│  Client  │ ──────────────► │  FastAPI App  │ ──────────► │  Worker   │
│ (DevOps) │                 │  (Web Server) │              │ (Async)   │
└─────────┘                  └──────┬───────┘              └─────┬────┘
                                    │                            │
                              polls status                 provisions:
                                    │                     - DNS records
                                    ▼                     - CloudFront
                              ┌──────────┐               - API calls
                              │ DynamoDB  │◄──── writes ──┘
                              └──────────┘
```

### Evolution of the Tech Stack

```
Connexion (OpenAPI → routes) → Flask → FastAPI
```

> [!NOTE]
> **Why this order matters**: He started with **Connexion** (auto-generates Flask routes from an OpenAPI spec — great for spec compliance), then moved to **pure Flask** (more control), then **FastAPI** (async performance + type hints). This is a common maturity pattern for Python services.

### Key Design Decisions

| Decision | Why |
|----------|-----|
| **Async worker via SQS** | Provisioning is slow (DNS propagation, CloudFront setup). Don't block the web request. |
| **Client polls for status** | Simpler than webhooks. The OSB spec supports this natively with `last_operation` endpoint. |
| **DynamoDB as state store** | Managed, no-ops, scales horizontally. Perfect for key-value provisioning state. |

### 🛠️ Practice Project: Build Your Own Mini Service Broker

Build a FastAPI app with these endpoints:

```python
# GET /v2/catalog — return available services
# PUT /v2/service_instances/{id} — provision (async)
# GET /v2/service_instances/{id}/last_operation — poll status
# DELETE /v2/service_instances/{id} — deprovision
```

Use a simple queue (even Python's `asyncio.Queue` to start) and SQLite/DynamoDB Local.

---

## 3. The Envoy Control Plane (Sovereign)

### Why Envoy?

Atlassian replaced **enterprise load balancers** (expensive licensing) with **Envoy Proxy** (open-source, cloud-native). Key advantage: **dynamic configuration via xDS API** — no restarts needed.

### The Control Plane Architecture

```
┌─────────────────────────────────────────────┐
│            Sovereign (FastAPI)                │
│                                               │
│  ┌───────────┐   ┌──────────────────────┐    │
│  │  Context   │   │     Templates        │    │
│  │ (dynamic)  │   │ - clusters.j2        │    │
│  │            │   │ - routes.j2          │    │
│  │ From:      │   │ - listeners.j2       │    │
│  │ • Broker DB│   │                      │    │
│  │ • S3 data  │   │ (Jinja2 / similar)   │    │
│  └─────┬─────┘   └──────────┬───────────┘    │
│        │                     │                │
│        └──────┬──────────────┘                │
│               ▼                               │
│    Rendered Envoy Config (CDS, RDS, LDS)     │
└───────────────────┬──────────────────────────┘
                    │ gRPC / REST (xDS)
                    ▼
            ┌──────────────┐
            │  Envoy Proxy  │  ← dynamically updated
            │  (hundreds)   │
            └──────────────┘
```

### How It Works (Data Flow)

1. **Developer** sends a provisioning request to the **Broker**
2. **Worker** creates DNS records, CloudFront dist, writes config to **DynamoDB**
3. **Sovereign** polls DynamoDB + S3 for latest state (the "context")
4. Context is injected into **templates** (clusters, routes, listeners)
5. Envoy proxies request new config via **xDS protocol**
6. Proxies update **without restart** — zero downtime

### Envoy Resource Types You Need to Know

| Resource | What It Does | Example |
|----------|-------------|---------|
| **Listener** | Binds to a port, accepts connections | `0.0.0.0:443` with TLS |
| **Route** | Matches incoming requests to clusters | `/api/v1/*` → `backend-service` |
| **Cluster** | Group of upstream endpoints | `{10.0.1.1:8080, 10.0.1.2:8080}` |
| **Endpoint** | Individual backend instance | `10.0.1.1:8080` |

### Key Insight: Templates as Product Features

> All features delivered to developers lived as **logic in the templates**. A developer sends a small JSON config, and the template engine produces complex Envoy configuration.

This is the **"platform as product"** pattern — abstract away infrastructure complexity behind a simple API.

> [!IMPORTANT]
> **Sovereign is open-source** — the author published it on Bitbucket. Search for it to study the actual implementation.

---

## 4. Infrastructure as Code — CloudFormation + Packer + SaltStack

### The Provisioning Pipeline

```
                                        ┌─────────────────────┐
                                        │    Packer Build      │
┌─────────────┐    SaltStack states     │                     │
│ Salt States  │ ──────────────────────► │  1. Launch temp EC2  │
│              │                        │  2. Upload config    │
│ • envoy      │                        │  3. Run provisioning │
│ • logging    │                        │  4. Snapshot → AMI   │
│ • security   │                        └──────────┬──────────┘
│ • networking │                                   │
│ • containers │                                   ▼
│ • observab.  │                           ┌──────────────┐
└─────────────┘                            │     AMI       │
                                           └──────┬───────┘
                                                  │ referenced by
                                                  ▼
                                    ┌──────────────────────────┐
                                    │   CloudFormation Template │
                                    │                          │
                                    │  • VPC + Subnets          │
                                    │  • Security Groups        │
                                    │  • IAM Roles              │
                                    │  • Auto Scaling Group     │
                                    │  • NLB (Layer 4)          │
                                    │  • ACM (TLS certs)        │
                                    │  • Route53 records        │
                                    │  • Parameters (secrets)   │
                                    └──────────────────────────┘
                                              │
                                              ▼
                                    ~2,000 proxies across
                                     ~13 AWS regions
```

### SaltStack States on Each Proxy

| State | Purpose |
|-------|---------|
| `envoy/install` | Install Envoy binary |
| `envoy/configure` | Base config (bootstrap, admin interface) |
| `observability` | Logging agents, metrics (Prometheus/StatsD), tracing (Jaeger/Zipkin) |
| `security/hardening` | OS hardening, firewall rules, CVE patches |
| `network_tuning` | Kernel params (`net.core.somaxconn`, `net.ipv4.tcp_tw_reuse`, etc.) |
| `containers` | Docker/containerd for sidecar containers |

> [!NOTE]
> **Modern equivalent**: Today you'd probably use **Terraform** instead of CloudFormation, and **Ansible** instead of SaltStack. Packer is still the standard for AMI building.

---

## 5. Migrating Jira, Confluence, Bitbucket onto the Edge

### The Migration Strategy

**Before**: Platform provided basic load balancing → services could accidentally be public with no protection.

**After**: Services **must** explicitly configure centralized edge infrastructure → signals **intentional** public exposure.

This was a **security-first** migration, not just a performance one.

### Why It Took 2 Years

| Challenge | Why It's Hard |
|-----------|--------------|
| Special cases per product | Jira ≠ Confluence ≠ Bitbucket routing rules |
| Multi-tenant platform | Generic platform must handle every product's unique needs |
| Feature parity | Can't migrate until the new platform supports everything the old one did |
| Zero downtime | Can't break live products during migration |

> [!IMPORTANT]
> **Key principle**: *"A generic multi-tenanted platform supporting larger products and their special cases"* — this tension between generic and specific is the core challenge of platform engineering.

---

## 6. Programmable Proxy with Sidecars

### The Edge Architecture

```
                    ┌──────────────────────────────────────────┐
                    │             Edge Proxy Node               │
                    │                                          │
  Customer ──────►  │  ┌────────────────────────────────────┐  │  ──────► Backend
  Request           │  │          Envoy Proxy                │  │         Service
                    │  │                                      │  │
                    │  │  • DDoS protection (CloudFront)      │  │
                    │  │  • Access logging (native HCM)       │  │
                    │  │  • Header manipulation               │  │
                    │  │  • Routing (domain, path, header)    │  │
                    │  └───────┬────────┬────────┬────────────┘  │
                    │          │        │        │               │
                    │    ┌─────▼──┐ ┌──▼───┐ ┌──▼────────┐     │
                    │    │ AuthN  │ │AuthZ │ │Rate Limit  │     │
                    │    │(Rust)  │ │      │ │            │     │
                    │    │ Team A │ │Team B│ │  Team C    │     │
                    │    └────────┘ └──────┘ └───────────┘     │
                    │         Sidecar Containers                │
                    └──────────────────────────────────────────┘
```

### What Each Concern Handles

| Concern | Where It Runs | How |
|---------|--------------|-----|
| **DDoS protection** | CloudFront (upstream) | AWS Shield, WAF rules |
| **Access logging** | Envoy native | HCM access log filter — configured via dynamic templates |
| **Authentication** | Sidecar container (written in **Rust** 🦀) | Envoy ext_authz filter calls the sidecar |
| **Authorization** | Sidecar (different team) | Envoy ext_authz or ext_proc filter |
| **Rate limiting** | Sidecar (different team) | Envoy rate limit service |

### Why Sidecars?

> *"Can you imagine if a thousand dev teams needed to deal with authentication, authorization, rate limiting, access logs, and DDoS protection on their own service? It would be a tremendous waste of money."*

**Centralized** = consistent, cheaper, faster feature delivery.

### Why Rust for AuthN?

- Hot path — every single request goes through it
- Microsecond latency budget
- Memory safety without garbage collection pauses
- Perfect for security-critical code

---

## 7. Soft Skills That Actually Matter

### What 8 Years at Atlassian Taught Him (Non-Technical)

| Skill | What He Learned |
|-------|----------------|
| **Diplomacy** | Different managers and colleagues have different styles. Personality clashes are inevitable. |
| **Conflict awareness** | *"Have self-awareness of the other person... anticipate the conflict that's going to arise."* |
| **Teaching** | Breaking down complex systems into simple mental models. His bread and butter. |
| **Persuasion** | Proposing ideas, getting buy-in for architectural changes |
| **Performance under stress** | Conflicts affected his performance — he took it seriously and grew |

> [!WARNING]
> *"I experienced conflicts with certain people. Even though I had conflicts, there are still people I respect. It's just something that happens when your personality doesn't mix."*
> 
> This is honest and real. Expect it. Plan for it. Don't take it personally.

---

## 8. Build vs. Maintain — The Real Cost

### The Maintenance Lifecycle

```
Build Phase                     Maintenance Phase (years)
────────────                    ──────────────────────────
• Write docs                    • Re-onboard new hires (people leave)
• Train team                    • New opinions → code churn
• Set up on-call runbooks       • Churn = smell → complexity growing
• Define key metrics            • Coupling sneaks in
• Automate known failures       • Changes in one area break another
                                • "Detangling" becomes the real work
```

### Key Insights

> *"Building something is easy. Changing it and making sure you can still change it over time is difficult."*

> *"As you change things, it slowly becomes harder to change. Things start to get coupled."*

### Codebase Churn as a Smell

Once you notice **repeated churn** in one area of the codebase:
1. That area will **keep growing** in size and complexity
2. Something **must be done** — refactor, split, redesign
3. Ignoring it = exponential maintenance cost

### On-Call Knowledge Checklist

For any system you build, document:

- [ ] What do specific **log messages** mean?
- [ ] What **metrics** to check when something breaks?
- [ ] What do those metrics **allude to**?
- [ ] What if **AWS has an outage** (DynamoDB, SQS down)?
- [ ] What if the proxy receives **bad configuration**?
- [ ] What if config is **valid but destructive** (kills traffic)?
- [ ] How do you **detect** and **rollback**?

---

## 9. Mentoring — The Hardest Skill

### The Challenge

He mentored an intern who received the **highest rating possible** (guaranteed job offer). But he found mentoring **personally difficult**:

> *"Striking the balance between how much time I give and... I don't want to give them answers, but I don't want them to get so stuck they become frustrated."*

### What Worked

| What | Why |
|------|-----|
| Connected intern with **subject matter experts** | Nobody knows everything — use the team |
| Let them **make design decisions** | Ownership creates growth |
| Let them **do the legwork** | Building muscle, not just watching |

### Teaching vs. Mentoring

| Teaching | Mentoring |
|----------|-----------|
| Breaking complex → simple | Guiding career/growth direction |
| Transferring knowledge | Knowing when to help vs. step back |
| Immediate feedback | Long-term relationship |
| He's great at this | He found this harder |

> *"I've never been mentored myself, so I don't really know what to expect."*

---

## 10. Key Takeaways & Action Items

### Architecture Patterns to Study

- [ ] **Open Service Broker API** — self-service provisioning pattern
- [ ] **Envoy xDS Protocol** — dynamic proxy configuration
- [ ] **Control Plane / Data Plane separation** — Sovereign pattern
- [ ] **Sidecar model** — authentication, rate limiting as containers
- [ ] **AMI baking with Packer** — immutable infrastructure
- [ ] **CloudFormation / Terraform** — multi-region infra as code
- [ ] **SaltStack / Ansible** — configuration management

### Technologies to Get Hands-On With

| Technology | Priority | Why |
|-----------|----------|-----|
| **Envoy Proxy** | 🔴 High | The center of modern service mesh and edge infra |
| **FastAPI** | 🔴 High | The go-to for Python APIs (replaced Flask) |
| **Packer** | 🟡 Medium | Still the standard for machine image building |
| **Terraform** | 🔴 High | Modern replacement for CloudFormation |
| **Redis** | 🔴 High | Used everywhere at scale |
| **Rust** | 🟡 Medium | For hot-path, latency-critical code |

### Career Lessons

1. **Ask product questions in interviews** — "What would make hiring me a good decision?"
2. **First-principles thinking > memorization** — his DNS answer was wrong but showed reasoning
3. **Building is the easy part** — maintaining over years with team churn is the real challenge
4. **Conflict is inevitable** — invest in self-awareness and psychology
5. **Teaching ≠ Mentoring** — both are valuable, they're different skills
6. **Document your on-call runbooks** — future you (and your team) will thank you

---

> *"Feedback I got from my colleagues all the time was that I was always available to help and that I could boil down hard topics into something that was understandable."*

**That's the engineering superpower** — not the code you write, but the clarity you bring to complex systems.
