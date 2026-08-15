# Docker → Production Systems: The System Design Engineer's Learning Path

> *Compiled: August 2026*
> *Primary source: `ALL_DOCKER_TRANSCRIPTS.txt` (Abhishek's DevOps course Day 23-29 plus standalone Docker deep-dives)*
> *Audience: A system design engineer who must make containers real in production — not just know what a container is.*
> *Philosophy: No size limit. Only understanding matters. Every concept maps back to a production problem.*

---

## Table of Contents

1. [Part 0 — The Frame: Why a System Design Engineer Must Own Containers](#part-0--the-frame)
2. [Part 1 — Foundations: Containers vs Virtual Machines](#part-1--foundations-containers-vs-virtual-machines)
3. [Part 2 — The Docker Platform Architecture](#part-2--the-docker-platform-architecture)
4. [Part 3 — Images & Dockerfile Mastery](#part-3--images--dockerfile-mastery)
5. [Part 4 — Multi-Stage Builds & Distroless Images](#part-4--multi-stage-builds--distroless-images)
6. [Part 5 — Storage: Volumes & Bind Mounts](#part-5--storage-volumes--bind-mounts)
7. [Part 6 — Container Networking](#part-6--container-networking)
8. [Part 7 — Docker Compose: Local Production Simulation](#part-7--docker-compose-local-production-simulation)
9. [Part 8 — Registries & Image Distribution](#part-8--registries--image-distribution)
10. [Part 9 — Multi-Architecture & Platform Builds](#part-9--multi-architecture--platform-builds)
11. [Part 10 — Security & Production Hardening](#part-10--security--production-hardening)
12. [Part 11 — The Modern Toolbox: Docker Init, Model Runner, Docker Offload, the Ecosystem](#part-11--the-modern-toolbox)
13. [Part 12 — System Design Integration: Architecting with Containers](#part-12--system-design-integration)
14. [Part 13 — The Phased Learning Path](#part-13--the-phased-learning-path)
15. [Part 14 — Hands-On Labs (Production-Flavored)](#part-14--hands-on-labs)
16. [Part 15 — Scenario-Based Interview Mastery](#part-15--scenario-based-interview-mastery)
17. [Part 16 — Deep Synthesis: The One Mental Model](#part-16--deep-synthesis)
18. [Part 17 — Production Readiness Checklist](#part-17--production-readiness-checklist)
19. [Appendix A — Command Cheat Sheet](#appendix-a--command-cheat-sheet)
20. [Appendix B — Key Terms Glossary](#appendix-b--key-terms-glossary)

---

# Part 0 — The Frame

## Why you are reading this

A system design engineer is judged on one question: **"Can you design systems that survive contact with the real world?"** Containers are the unit of deployment in the real world. Spotify, Netflix, Amazon — every "tech giant" ships containerized applications. As the transcript says: *"The industry is shifting so fast. Companies are containerizing all the applications or the legacy applications that they can."*

This means the container is not a DevOps trivia topic for you. It is the **atomic unit** around which your architecture diagrams, capacity plans, scaling stories, and failure-mode analysis are built.

## The five production problems (your North Star)

Everything in this document exists to solve one of five problems. If you cannot map a concept back to one of these, you are learning trivia:

| # | Production Problem | Container Concept That Solves It |
|---|---|---|
| 1 | **"It works on my machine"** — dev/release/prod environments drift | Images: build once, run anywhere |
| 2 | **State loss** — containers are ephemeral, data must survive restarts | Volumes, bind mounts, persistent storage |
| 3 | **Communication** — services must talk securely, or be isolated | Networking: bridge/host/overlay, custom networks |
| 4 | **Scale & cost** — run more with less, deploy fast | Multi-stage builds, distroless, multi-arch, layer caching |
| 5 | **Supply chain risk** — code from many sources must be trustworthy | Registries, content trust, scanning, non-root images |

## The reality check from the source

From Day-29 (interview questions), the critical lesson: *"Interviewers are not keen to understand the Docker platform — they will try to understand your knowledge on containers and how you use Docker to work with the containers."* You are not learning commands; you are learning **why** commands exist, and what happens in production when you get them wrong.

A second reality check from the "DevOps & Cloud 2025" video: the craft is future-proof *not because you learn one tool*, but because you learn **how software is packaged, deployed, and operated**. That knowledge outlives any tool. Tools change (Docker → containerd → runc → Podman → Kubernetes runtimes); the *problems* do not.

---

# Part 1 — Foundations: Containers vs Virtual Machines

## 1.1 The shared problem: virtualization

Both VMs and containers answer the same question: *How do I run many isolated workloads on one physical machine?* The mechanism is called **virtualization** — software creating an abstraction layer over hardware.

## 1.2 Virtual machines: hardware virtualization

```
┌───────────────────────────────────────────────────────┐
│                     Physical Host                      │
├──────────┬──────────┬──────────┬──────────────────────┤
│   VM 1   │   VM 2   │   VM 3   │      ...             │
│ ┌──────┐ │ ┌──────┐ │ ┌──────┐ │   Each VM:           │
│ │ App  │ │ │ App  │ │ │ App  │ │   - own guest OS     │
│ ├──────┤ │ ├──────┤ │ ├──────┤ │   - own virtual CPU  │
│ │ Bin/ │ │ │ Bin/ │ │ │ Bin/ │ │   - own virtual mem  │
│ │ OS   │ │ │ OS   │ │ │ OS   │ │   - own virtual disk │
│ └──────┘ │ └──────┘ │ └──────┘ │                      │
├──────────┴──────────┴──────────┤                      │
│          Hypervisor            │  → virtualizes HARDWARE │
├────────────────────────────────┤                      │
│            Hardware            │                      │
└────────────────────────────────┘                      ┘
```

- The abstraction software is the **hypervisor** (VMware ESXi, KVM, Xen, Hyper-V).
- Each VM runs its **own full operating system** (guest OS) on **virtual hardware** (virtual CPU, storage, NICs).
- The hypervisor allocates resources *between* VMs on the single physical host.

**Cost:** You duplicate an entire OS per workload — kernel, drivers, system libraries — and emulate hardware. A VM can be gigabytes before you even install your app.

## 1.3 Containers: OS-level virtualization

```
┌───────────────────────────────────────────────────────┐
│                     Physical Host                      │
├───────────────────────────────────────────────────────┤
│            Host Operating System (Linux kernel)        │
├──────────┬──────────┬──────────┬──────────────────────┤
│  CTN 1   │  CTN 2   │  CTN 3   │      ...             │
│ ┌──────┐ │ ┌──────┐ │ ┌──────┐ │   Each container:    │
│ │ App  │ │ │ App  │ │ │ App  │ │   - application only │
│ ├──────┤ │ ├──────┤ │ ├──────┤ │   - app deps only    │
│ │ deps │ │ │ deps │ │ │ deps │ │   - required sys deps│
│ └──────┘ │ └──────┘ │ └──────┘ │   → shares host kernel│
├──────────┴──────────┴──────────┤                      │
│      Container Runtime         │  → virtualizes the OS │
│  (Docker engine, containerd)   │                      │
├────────────────────────────────┤                      │
│           Hardware             │                      │
└────────────────────────────────┘                      ┘
```

- Instead of virtualizing hardware, containers **virtualize the operating system**.
- Each container contains only the **application + its libraries + its dependencies**.
- All containers **share the host kernel**.

**Benefit:** A container is megabytes, not gigabytes. It starts in milliseconds, not minutes. There is no guest OS to boot, patch, or manage.

## 1.4 The key sentence to memorize

> **A container is not a lightweight VM. A container is an isolated process (or group of processes) running on the host kernel, with its own filesystem, its own view of the network, its own process tree, and constrained resource limits.**

This is why debugging containers feels different: no SSH, no init system, no OS to log into. Your "server" is a process tree.

## 1.5 Under the hood: the Linux primitives

The "magic" of containers is a set of Linux kernel features (on Windows hosts these are exposed via WSL2 or Hyper-V isolation in Docker Desktop):

### a) Namespaces — what you can *see*
Namespaces partition kernel resources so a process sees only its own view:

| Namespace | What it isolates |
|---|---|
| `PID` | Process tree — container sees only its own processes (PID 1 = app) |
| `NET` | Network interfaces, IP tables, routing tables |
| `MNT` | Mount points / filesystem views |
| `UTS` | Hostname and domain name |
| `IPC` | Inter-process communication (shared memory, semaphores) |
| `USER` | User and group ID mapping |
| `CGROUP` | Which cgroup tree a process belongs to |

A container = a process launched inside a fresh set of these namespaces.

### b) Cgroups — how much you can *use*
Cgroups (control groups) limit and account for resource usage: CPU time, memory, I/O bandwidth, PID count, network bandwidth. `docker run --memory 512m --cpus 2` maps directly to cgroup settings. **This is how a single noisy container cannot take down the host or its neighbors** — the kernel enforces the limits.

### c) Union filesystems / OverlayFS — why images stay small
Images are built from **layers**. OverlayFS stacks read-only layers with a thin writable top layer. Containers from the same image share the same read-only layers on disk; each adds only a thin writable layer. Running 50 copies of one image is cheap on disk.

### d) Root filesystem isolation
Each container gets its own root filesystem view — it cannot see the host's `/`; it sees the image's filesystem.

### e) Capabilities, seccomp, AppArmor — what you can *do*
Even inside a container, the process runs against the host kernel. Docker drops dangerous capabilities (`CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, etc.), applies a default seccomp profile, and can enforce AppArmor policies. This is the isolation that stops a container escape from being a trivial full-host compromise.

## 1.6 When to use which — the designer's decision table

| Criteria | VM | Container |
|---|---|---|
| Need different OS kernels (Windows + Linux) | ✅ Required | ❌ Not possible |
| Need hardware emulation / driver access | ✅ | ❌ |
| Strongest isolation boundaries | ✅ (hardware-level) | ⚠️ (kernel shared, process-level) |
| Density on a host | Low (GB each) | High (MB each) |
| Startup time | Minutes | Milliseconds |
| Packaged as code | Large VM images | Small container images |
| Right for microservices | ❌ too heavy | ✅ natural fit |
| Right for untrusted third-party code | ⚠️ lean VM/microVM | ⚠️ only with hardening |

A production-grade architecture uses both: VMs are the **hard boundary** (each Kubernetes node is a VM), containers are the **unit of work** inside them.

---

# Part 2 — The Docker Platform Architecture

## 2.1 What Docker actually is

Docker is a **container platform** developed by Docker Inc. Its core job: **manage the lifecycle of containers** — create, start, stop, run in the background, delete, inspect. It provides the tooling to take an application, package it (image), store it (registry), and run it (container).

The transcript's framing: *"Using the Docker platform you can take a virtual machine, install Docker on top of it, or Docker Desktop, Docker CLI — end of the day you can start your container, stop your container, run it as a background process. Docker as a platform manages the lifecycle of your containers."*

## 2.2 Architecture: client, daemon, runtimes

```
 ┌──────────────────────────────────────────────────┐
 │               Docker Client                       │
 │        (docker CLI / Docker Desktop UI)           │
 └──────────────────────┬───────────────────────────┘
                        │  REST API (unix socket / TCP)
 ┌──────────────────────▼───────────────────────────┐
 │            Docker Daemon (dockerd)                │
 │   - builds images, manages containers/networks    │
 │   - pulls/pushes images to registries             │
 └──────────────────────┬───────────────────────────┘
                        │  containerd (container supervisor)
 ┌──────────────────────▼───────────────────────────┐
 │                  runc (OCI runtime)               │
 │    spawns processes with namespaces + cgroups     │
 └──────────────────────┬───────────────────────────┘
                        ▼
                 Host Linux kernel
```

| Layer | Role | Notes for a designer |
|---|---|---|
| **CLI / Client** | Human interface; sends REST commands | `docker ...` runs here; also IDE/CI integrations |
| **Daemon (`dockerd`)** | Central manager for images, containers, volumes, networks | Single point of control per host; talks to registries |
| **containerd** | Container supervisor — pulls images, manages lifecycle | The layer Kubernetes actually talks to (CRI) |
| **runc** | OCI runtime — creates the actual processes with namespaces/cgroups | The layer that "does the isolation" |

Why this layering matters to you:
- The **OCI (Open Container Initiative) standards** (image spec + runtime spec) mean images built by Docker run on any conformant runtime — Podman, containerd, Kubernetes, Docker Engine on a different OS. Your knowledge of images is portable even if the platform changes.
- On Linux servers (EC2, bare metal, k8s nodes) you install **Docker Engine**, not Docker Desktop. On Windows/macOS developers, **Docker Desktop** provides a Linux VM (WSL2 / Hyper-V) underneath so Linux containers can run.

## 2.3 Images vs containers — the vocabulary

- **Image:** the immutable blueprint — a frozen filesystem snapshot plus metadata (entrypoint, env vars, exposed ports, labels). Images are never modified; you create new versions.
- **Container:** a *running instance* of an image — the image + a writable layer + a process tree + network + resource limits.

Analogy used in the course: an image is a **class / template / ISO**, a container is the **object / running instance**. Many containers can run from one image, each with isolated state.

**Immutability guarantee:** *"Images are immutable. Once built, they cannot be modified, only replaced with new versions. This immutability guarantees that what we test in development runs identically in production."*

## 2.4 The Docker container lifecycle

```
pull image → create container → start → (running)
                                   │
                    ┌──────────────┴──────────────┐
                 stop                         pause
                    │                              │
              (stopped)                        (paused)
                    │                              │
                 start                            │
                    │                              │
                 (running)◄───────────────────────┘
                    │
                 rm (delete)
```

States: **created → running → paused → stopped → removed**. A container that exits is still inspectable (`docker ps -a`) until removed. Data in the writable layer is lost on `rm` — this is exactly why storage (Part 5) exists.

## 2.5 What a container filesystem actually contains

From Day-24: the course inspected a running container and compared it with a VM. A container image typically holds:
- Application code
- Application dependencies (libraries)
- Required system dependencies (glibc, certificates, timezone data — whatever the base image ships)
- Config files / env var defaults

It does **not** hold: a kernel, init system, system daemons, package managers (in slim images), a shell (in distroless), etc. "The remaining everything they use from the kernel or the host operating system."

## 2.6 Installing Docker on a server — the permissions gotcha

A classic production/devops pitfall covered in Day-25:
- After installing Docker on an EC2 instance (Ubuntu), running `docker ps` fails with a **permission denied** error because the current user is not in the `docker` group.
- Fix: `sudo usermod -aG docker $USER`, log out/in (or `newgrp docker`).
- **Security note:** the `docker` group is effectively root — members can mount the host filesystem and escape. In production, prefer **rootless Docker**, or restrict who is in the group, or run via managed runtimes (ECS, EKS, GKE) where the host is not directly touched.

## 2.7 Docker Desktop vs Docker Engine

| | Docker Desktop | Docker Engine |
|---|---|---|
| Where | macOS / Windows dev machines | Linux servers, CI, cloud VMs |
| Under the hood | Linux VM (WSL2 / Hyper-V) | Native daemon on Linux |
| GUI | Dashboard, model runner, offload toggle | None (CLI only) |
| Use in prod | ❌ No | ✅ Yes |

Rule: **Docker Desktop is a developer tool. Docker Engine is the production runtime.** Design your images so they behave identically on both — that is the whole point of "build once, run anywhere."

---

# Part 3 — Images & Dockerfile Mastery

## 3.1 The Dockerfile is a contract

The Dockerfile defines the environment your application needs: base image, runtime, dependencies, and the exact commands to build and run. "We specify our base image, like Node 14 Alpine, carefully selecting what we need and nothing more."

The goal is not to write a working Dockerfile. The goal is to write a Dockerfile that is **small, cached well, reproducible, and safe**. That is the production bar.

## 3.2 Anatomy of a Dockerfile

```dockerfile
# Base image: choose the smallest that fits the runtime needs
FROM node:20-alpine AS base

# Working directory inside the container
WORKDIR /app

# Environment and metadata
ENV NODE_ENV=production
LABEL org.opencontainers.image.source="https://github.com/acme/api"

# Dependencies first (exploits layer caching)
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# Application code
COPY . .

# Drop privileges
USER node

# Declare the port (informational; publishing happens at runtime)
EXPOSE 3000

# Health check the orchestrator can call
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://127.0.0.1:3000/health || exit 1

# The command that runs at container start
CMD ["node", "server.js"]
```

## 3.3 Each instruction = a layer; layers drive caching

Every instruction (`FROM`, `RUN`, `COPY`) creates a **layer**. Layers are cached by the daemon:
- If the base image is unchanged, the `FROM` layer is reused from cache.
- If `package.json`/`lockfile` are unchanged, the dependency install layer is reused.
- Only changed layers and everything after them are rebuilt.

**The ordering rule that saves hours of CI time:** put things that change rarely at the top, things that change often at the bottom.
1. `FROM` (base)
2. System-level installs (rarely change)
3. Manifest/lockfiles (`COPY package*.json` / `requirements.txt`)
4. Dependency install (`RUN npm ci` / `pip install`)
5. Application source (`COPY . .`) — changes on every commit
6. `CMD`

Because `COPY . .` is placed last, application code changes never invalidate the expensive dependency layers. This is one of the highest-leverage production practices and a guaranteed interview topic.

## 3.4 Best practices — the production checklist

| Practice | Why | Failure mode if skipped |
|---|---|---|
| Choose a slim/alpine base | Smaller attack surface, faster pull, less disk | Huge images, more CVEs |
| Combine `RUN` commands with `&&` | Fewer layers, smaller image | Many tiny layers, bloat |
| Clean caches in the same `RUN` | `apt-get clean`, `rm -rf /var/lib/apt/lists/*` | Cached package lists bloat image |
| Keep `.dockerignore` | Exclude `.git`, node_modules, logs, env files | Secrets in image, huge build context |
| Order instructions by change frequency | Layer cache reuse | Slow rebuilds, expensive CI |
| Use lockfiles | Reproducible dependencies | "Works on my machine" via floating versions |
| Run as non-root (`USER` directive) | Least privilege | Container running as root is a host risk |
| Pin image tags (`node:20-alpine` not `latest`) | Reproducibility | Silent breaking base-image changes |
| Add `HEALTHCHECK` | Orchestrator knows liveness | Orphaned/dead containers keep traffic |
| Avoid secrets in image (`ARG`/`ENV` leaks, `COPY .env`) | Supply chain hygiene | Leaked credentials in the image history |

## 3.5 Build context and .dockerignore

The **build context** is everything Docker can send to the daemon when building — by default the current directory (or a path you pass). Anything in the context is visible to the builder and (if `COPY . .`) ends up in the image.

`.dockerignore` is the `.gitignore` for images:

```dockerignore
.git
node_modules
*.log
.env
__pycache__
*.md
dist/*.map
```

Production failure this prevents: **secrets committed into images**. `COPY . .` with a `.env` in the build context bakes credentials into every image layer — and they persist in image history. Once an image is pushed, the secret is effectively public. This single class of mistake has caused many real breaches.

## 3.6 Base image selection — a design decision

| Base | Size (approx) | Includes | Use when |
|---|---|---|---|
| `ubuntu` / `debian` | 50-80 MB | Full toolchain | Debugging-heavy, need shell/packages |
| `slim` variants | 20-40 MB | Shell + essentials | Default choice for many apps |
| `alpine` | 2-8 MB | musl, apk | Smallest; watch for glibc-dependent binaries |
| `distroless` | 5-30 MB | Runtime only, no shell/pkg manager | Highest security bar, hardened prod |
| scratch | ~0 MB | Nothing | Static binaries (Go) |

Design principle from the source: *"carefully selecting what we need and nothing more."* Each base choice is a tradeoff among size, debuggability, and security — the transcript's multi-stage video shows reducing image size by up to ~8x (the headline said "800%") with the same application.

## 3.7 Entrypoint vs CMD

- `CMD` — the default command; easily overridden at `docker run ... <new command>`.
- `ENTRYPOINT` — the fixed executable; `CMD` becomes its default arguments.

Pattern: `ENTRYPOINT ["python"]` + `CMD ["app.py"]` lets you run `docker run myimage app2.py` while keeping the runtime fixed. In Kubernetes, `command`/`args` map to these two.

---

# Part 4 — Multi-Stage Builds & Distroless Images

## 4.1 The naive build problem (from Day-26)

To run a calculator (Python) app you might naively:
1. `FROM ubuntu`
2. Install python + pip + compilers + build tools
3. Copy code, install deps
4. Run

Result: a **gigabyte-sized image** containing compilers, package managers, headers, and caches that the runtime never needs. You deploy for 5 minutes what could deploy in 30 seconds; every node must download hundreds of MB; every CVE scanner sees hundreds of vulnerabilities in unused tools.

## 4.2 Multi-stage builds — the fix

Multi-stage lets one Dockerfile define several `FROM` stages; only the final stage is kept. Build tools live in an early stage; the final stage copies only the artifacts it needs.

```dockerfile
# ---- Stage 1: builder ----
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build           # produces dist/ (compiled, minified assets)

# ---- Stage 2: runtime ----
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules   # prod deps only if split
RUN npm prune --omit=dev
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

Python equivalent:

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir -w /wheels -r requirements.txt

FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /wheels /wheels
COPY --from=builder /app .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels
COPY . .
USER appuser
CMD ["python", "app.py"]
```

Go equivalent (scratch + static binary):

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /bin/app .

FROM scratch
COPY --from=build /bin/app /bin/app
EXPOSE 8080
ENTRYPOINT ["/bin/app"]
```

**Why it matters in production:** smaller images → faster pushes/pulls, less disk per node, faster cold starts, smaller CVE surface, cheaper registry storage. This is the answer to "how did you reduce image size" interview questions.

## 4.3 Distroless images

**Distroless** images (by Google) contain *only your application and its runtime* — **no shell, no package manager, no standard Linux utilities**. Everything that could be used to pivot after a compromise is absent.

Benefits:
- Tiny attack surface (nothing to `curl`, `nc`, `apt` with)
- Fewer CVEs, smaller images
- Forced least privilege

Costs (the tradeoff you must know):
- **Harder to debug** — no shell, no `ls`, no `ps`. Debugging pattern: attach with an ephemeral *debug container* (e.g., `kubectl debug` / sidecar), or build a parallel image with tools.
- Some apps need glibc/system certs — distroless images provide `distroless/base`, `distroless/java`, `distroless/python`, `distroless/cc` variants that include those.

## 4.4 The synthesis (the interview-ready statement)

> **Multi-stage builds remove build-time tooling from the final image; distroless images remove even runtime shell utilities. Combined, they produce images that are small, fast to deploy, and hard to attack. The tradeoff is observability — so you compensate with health checks, structured logging, and ephemeral debug containers.**

## 4.5 A production problem story (how to answer "practical problems with Docker")

The Day-26 framing: *"When an interviewer asks what are some practical problems you have faced with Docker/containers and how did you solve them..."* Use this structure — situation → symptom → root cause → fix → measurable result:

- **Problem:** Production image was ~1.2 GB, deploys took minutes, scanner flagged hundreds of CVEs.
- **Symptom:** Slow rollouts, storage pressure on nodes, security review blocked the release.
- **Root cause:** Single-stage build with full Ubuntu base, dev dependencies, and build tools copied into the image.
- **Fix:** Multi-stage build (builder → runtime), alpine/distroless runtime, `.dockerignore`, `npm ci` with prune.
- **Result:** ~120 MB image (~90% reduction), seconds to deploy, CVE count in the low double digits, release unblocked.

This is a template you should be able to fill for any of the topics in this document.

---

# Part 5 — Storage: Volumes & Bind Mounts

## 5.1 The core problem: containers are ephemeral

Day-27 opens with the practical story: an **nginx container** continuously writes user access logs (who logged in, from which IP, when). Those logs matter for security, auditing, and understanding user behavior. Then the container **crashes or is removed** — and the log file is **gone**.

Why? Containers are **ephemeral** — short-lived by design:
- The container's writable layer lives and dies with the container.
- `docker rm` deletes the container and its writable data.
- Restarting, rescheduling (in Kubernetes), or replacing a node destroys all local state.

> **If a container writes state only to its own filesystem, that state has the same lifetime as the container. Persistent storage is the mechanism that decouples data lifetime from container lifetime.**

This is the single most common "data gone" production incident in the container world, and it is exactly the failure mode this Part fixes.

## 5.2 The storage hierarchy in Docker

| Mechanism | Managed by | Use case |
|---|---|---|
| Container writable layer | Docker (per container) | Scratch space, cache; **not** for real data |
| **Bind mount** | You (host path) | Share host dirs: configs, logs, dev code |
| **Volume** | Docker (in `/var/lib/docker/volumes/`) | **Preferred** for production persistent data |
| `tmpfs` | Memory (host RAM) | Sensitive/ephemeral data, nothing persisted |

## 5.3 Volumes

**Volume = a directory on the host filesystem, managed by Docker** (default location `/var/lib/docker/volumes/<name>/_data`). You mount it into the container at a path.

```bash
# create and use a named volume
docker volume create app-data
docker run -d -v app-data:/var/lib/postgresql/data postgres:16

# anonymous volume (auto-created name)
docker run -d -v /var/lib/redis/data redis:7

# inspect
docker volume ls
docker volume inspect app-data
```

Why volumes are the production default:
- Decoupled from container lifecycle — survives `rm`, restart, recreate.
- Easily backed up, migrated, and shared between containers.
- Driver-pluggable: `local` on the host, or cloud/network drivers (NFS, Ceph, EBS-backed CSI in Kubernetes) for distributed systems.
- On macOS/Windows, Docker manages the backing directory so filesystem semantics are consistent.

**Critical design point for system design:** in a multi-node world (Kubernetes, Swarm), a *local* volume lives on one node only. If your design needs data accessible from any replica, you need a **shared/network filesystem** or an external storage service (database-as-a-service, object storage). Local volumes pin workloads to nodes — this is why stateful design is hard and why the 12-factor guidance says prefer stateless apps backed by external state.

## 5.4 Bind mounts

**Bind mount = mount a host path directly into the container.**

```bash
docker run -d -p 8080:80 -v "$PWD/nginx.conf:/etc/nginx/nginx.conf:ro" nginx
docker run -v "$PWD/src:/app:ro" myapp        # dev live-reload
```

Use cases:
- Development: mount source code so edits are live without rebuilding the image.
- Config injection: mount a config file read-only.
- Reading host artifacts: e.g., host log directory.

Production cautions:
- Path-dependent and host-specific — not portable across machines (breaks "build once, run anywhere").
- If the host path doesn't exist, Docker may create it as a directory (surprising bugs).
- Less portable than volumes for orchestrated environments.

## 5.5 tmpfs

```bash
docker run -d --tmpfs /run --tmpfs /var/run/app myapp
```
- Lives in host **RAM** (never on disk). Wiped on stop.
- Use for secrets-in-flight, session tokens, or high-frequency scratch data that must never be persisted.

## 5.6 Practical storage patterns for production

**1. Stateless services (recommended):** no persistent data in the container. All state lives in external systems (database, object store, cache). Containers can be destroyed and replaced at will. This is the 12-factor principle and the reason microservices scale horizontally.

**2. Stateful services:** data in named volumes, plus:
- **Backups:** volume-level backups (`docker run --rm -v app-data:/data -v $PWD:/backup alpine tar czf /backup/app-data.tar.gz /data`), or scheduled snapshots of the host volume directory.
- **Replication:** for databases, prefer the database's own replication (PostgreSQL streaming, Redis Sentinel/Cluster) over copying volume files while the app is running.
- **Read-only where possible:** mount configs as `:ro` to prevent accidental writes.

**3. Logs as data:** even with a volume, in production you usually want logs *streamed out* (stdout) and collected by a log aggregator (ELK, Loki, CloudWatch, Datadog) — not written to a local file that a crashing container can lose. The nginx story is the classic argument: **emit logs to stdout; let the platform collect them.**

## 5.7 The design-answer wrap-up

When a system design interview asks about storage, containers change your answer:
- Default assumption: **stateless containers + external state**.
- If you must be stateful, name the storage tier explicitly (EBS, NFS, managed DB) and explain *replication* and *backup/RPO* for it.
- Explain the lifetime mismatch: container lifetime ≠ data lifetime, and everything you design must respect that.

---

# Part 6 — Container Networking

## 6.1 Why networking exists

Day-28's framing: networking allows containers to **communicate with each other and with the host**. On one host you may run a development app (container A) and a finance/payments service (container B). Sometimes A must talk to B; sometimes a container must be **completely isolated**. Networking is the mechanism that decides who can talk to whom.

For a system design engineer this is the skeleton of your architecture: every arrow between services in your diagram is a networking decision, and getting it wrong produces either broken communication or a security hole.

## 6.2 The Docker network drivers

### Bridge (default on a host)

```
 host ┌──────────────────────────────────────┐
      │   docker0 bridge (172.17.0.0/16)     │
      │       172.17.0.2   172.17.0.3       │
      │     ┌─────────┐ ┌─────────┐         │
      │     │ App A   │ │ App B   │         │
      │     └─────────┘ └─────────┘         │
      └──────────────┬──────────────────────┘
                     │ iptables / NAT (port publish)
                    LAN
```

- Containers get private IPs on an internal virtual network; they reach each other by IP or **container-name DNS**.
- Outbound traffic is NATed to the host. Inbound traffic reaches containers via **port publishing** (`-p 8080:80` → host 8080 → container 80), implemented with `iptables`.
- Isolation is *soft*: all containers on the default bridge can talk to each other.

### Host
- Container shares the **host's network stack** directly — same IP, no NAT.
- Performance: lowest overhead (great for network-bound workloads or load balancers on the host).
- Cost: no isolation; port conflicts possible; everything shares host namespace.

### None
- No network attached to the container (loopback only). For offline batch jobs, isolated agents, or security-sensitive workloads.

### Overlay (multi-host / Swarm / Kubernetes)
- A virtual network **spanning multiple hosts**; containers on any host get addresses in the same overlay network and can communicate directly as if on one L2 domain.
- Underpinned by VXLAN encapsulation between hosts. This is the driver behind Docker Swarm and conceptually what Kubernetes CNI plugins (Calico, Cilium, Flannel) provide.

## 6.3 Custom bridge networks — the security pattern that matters

The most practical lesson of Day-28: **don't use the default bridge for your services; create custom bridge networks per application tier.**

```bash
docker network create --driver bridge payments-net
docker network create --driver bridge frontend-net

docker run -d --network payments-net --name payment-svc payments:v1
docker run -d --network payments-net --name web-svc -p 8080:80 --network-alias api web:v1
```

Why custom networks:
1. **Automatic DNS** — containers resolve each other by *name* (`web` → `web-svc`), not by fragile IP. Containers are recreated with new IPs all the time; DNS is what makes multi-container apps work.
2. **Real isolation** — containers on different custom networks **cannot reach each other by default**. You get network segmentation for free: payment services are invisible to public frontends unless you explicitly connect them (`docker network connect`).

```
          ┌─────────────────────────────────┐
          │  host                          │
          │  ┌── frontend-net ──┐           │
          │  │  web (p 8080)    │           │
          │  └──────────────────┘           │
          │  ┌── payments-net ──┐           │
          │  │  payment-svc     │           │
          │  │  db (internal)   │           │
          │  └──────────────────┘           │
          └─────────────────────────────────┘
```

The interview answer: *"By default, containers on the default bridge can communicate freely. To enforce least-privilege communication, I put each service tier on its own custom bridge network and only connect what must talk — this gives DNS-based service discovery and network segmentation without extra tooling."*

## 6.4 Port publishing — what `-p` actually does

- `-p 8080:80` publishes host port 8080 → container port 80.
- `-p 127.0.0.1:8080:80` binds only to localhost (safer for internal tools).
- **Every published port is an attack surface.** Publish the minimum. Internal services (DBs, caches, brokers) should *not* be published to the host at all — they live on the bridge network and are reached only by other containers.

## 6.5 Security checklist for container networking

| Decision | Production default |
|---|---|
| Service-to-service | Custom bridge network(s), name-based DNS |
| Tier separation | One network per tier/trust domain |
| Expose to host | Only what needs external traffic |
| Bind public interfaces | Only with auth/firewall in front |
| Inter-node traffic | Overlay network or a real CNI in k8s |
| Never | Hardcoded container IPs in configs |

## 6.6 Ingress pattern (bridging to system design)

A complete design for inbound traffic:

```
Clients
   │
   ▼
Load balancer / reverse proxy (host:443)
   │  (published 443:443, TLS termination)
   ▼
web-svc (bridge network) ──→ payment-svc ──→ db-svc
   (only on internal network, never published)
```

The load balancer is the *only* published entry point; everything behind it lives on internal networks. In Kubernetes this maps to Ingress + Services; on a single VM it maps to a proxy container + custom networks.

---

# Part 7 — Docker Compose: Local Production Simulation

## 7.1 What Compose is

Docker Compose defines **multi-container applications as code**: services, networks, volumes, environment, health, and dependencies — all in a single `compose.yaml`. One command (`docker compose up`) brings up the entire application.

For a designer, Compose is the tool that makes your **local machine a miniature of production**: app + database + cache + queue + reverse proxy all running together with the same topology you drew in your diagram. It is also used in CI to spin up ephemeral test environments (e.g., a Postgres for integration tests).

## 7.2 Anatomy of a compose file

```yaml
name: acme-shop

services:
  web:
    build: .
    image: acme/web:1.0
    ports:
      - "8080:3000"          # publish
    environment:
      - DATABASE_URL=postgres://app:pass@db:5432/app
    depends_on:
      db:
        condition: service_healthy
    networks: [frontend, backend]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:?set me in .env}"
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks: [backend]
    volumes_from: []
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    networks: [backend]
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:

networks:
  frontend:
  backend:
```

Key design elements:
- **Services** define containers; `ports` publishes; `networks` assigns isolation tiers (mirrors Part 6).
- **`depends_on`** with `condition: service_healthy` expresses *readiness*, not just start order — the web container starts only after the DB accepts connections. This is the difference between "works sometimes" and "works reliably".
- **Named volumes** survive `docker compose down` and even `down -v` is needed to wipe them — the data-persistence pattern from Part 5.
- **`.env` files** provide local config; `${VAR:?message}` fails fast if required variables are missing (no silent misconfiguration).

## 7.3 Compose commands that matter

```bash
docker compose up -d          # start full stack detached
docker compose ps             # status of services
docker compose logs -f web    # follow logs
docker compose down           # stop + remove containers/networks (keeps volumes)
docker compose down -v        # also remove volumes (data loss — careful)
docker compose config         # validate + render final config
docker compose exec web sh    # shell into a running service
docker compose up --build     # rebuild images
```

## 7.4 Compose vs Kubernetes (from the transcript)

The source explicitly compares them. The design-engineer answer:

| Dimension | Docker Compose | Kubernetes |
|---|---|---|
| Scope | One host, dev/CI/small prod | Multi-node clusters |
| Scaling | Manual (`--scale web=3`, same host) | ReplicaSets / HPA across nodes |
| Self-healing | `restart: unless-stopped` only | Restart, reschedule, node failure handling |
| Rolling updates | `up` replaces services (basic) | Deployments: rolling/blue-green/canary |
| Service discovery | Internal DNS per network | Services + DNS + ingress |
| Config & secrets | env files | ConfigMaps, Secrets |
| Storage | Local volumes | CSI volumes, PV/PVC, cloud-backed |
| Complexity | Low | High — needs a cluster to run |

The mental model: **Compose is the local/CI expression of the same application topology that Kubernetes runs in production.** Design the topology once (services, networks, volumes, health checks, env), and the mapping to k8s manifests is mostly mechanical. That is why learning Compose first — before k8s — is the recommended path in this course.

## 7.5 Compose patterns for production-adjacent work

1. **Full-stack dev environment** — app + db + cache + mailer + proxy; document the diagram in `README`.
2. **Integration test harness in CI** — `docker compose -f docker-compose.test.yml up -d`, run tests, `down -v`.
3. **Single-node production** — a small VM running the whole stack with `restart: always` and named volumes. Acceptable for low traffic; not horizontally scalable.
4. **Golden-path configuration** — secrets via env (never committed), health checks on every service, read-only mounts for configs, resource limits (`mem_limit`, `cpus`).

---

# Part 8 — Registries & Image Distribution

## 8.1 The registry is the single source of truth

From the System Design episode: *"For distribution, we rely on Docker registries. These repositories become the single source of truth for our images. Whether we're using Docker Hub publicly or running our private registry internally, the principle remains the same: build once, run anywhere."*

The pipeline:

```
build image  →  push to registry  →  pull on any host  →  run container
```

The registry decouples *building* from *running*: CI builds and pushes once; every environment (test, staging, prod) pulls exactly the same artifact. This is what kills the "works on my machine" class of bugs forever — the byte-identical image is the deployable unit.

## 8.2 Registry types

| Registry | Use case |
|---|---|
| Docker Hub | Public base images (node, python, postgres); public distribution |
| GitHub Container Registry (GHCR) | Co-located with code; good for OSS and GitHub-native CI |
| Cloud registries (ECR, GCR, ACR) | Private images in your cloud account; IAM-based auth; image scanning |
| Self-hosted (Harbor, Nexus, Docker Registry) | Air-gapped networks, compliance, full control |

## 8.3 Tags vs digests — reproducibility

```bash
docker tag myapp:latest acme/registry:5000/web:1.0.0
docker push acme/registry:5000/web:1.0.0
docker pull acme/registry:5000/web:1.0.0

# immutability: reference by digest (content hash) — never lies
docker pull acme/registry:5000/web@sha256:9f86d081884c7d659a2f...
```

- **Tags are mutable pointers** — `latest` can silently change. Never deploy from `latest`.
- **Digests are content-addressed** — the SHA-256 of the image manifest. Two machines pulling the same digest get byte-identical images. For production deploys and audits, reference digests.
- Production tagging convention: versioned tags (`v1.0.0`, commit SHA, or build number) for humans + digest pinning for machines.

## 8.4 Pushing to Docker Hub / a registry (from the real-time tutorial)

The Google Cloud / Super Mario tutorial showed the full flow:
1. `docker login` (auth to the registry).
2. `docker tag <local-image> <registry>/<repo>:<tag>` — the registry path is part of the image name.
3. `docker push <registry>/<repo>:<tag>`.
4. On the server: `docker pull` then `docker run`.

Names: `docker.io/library/ubuntu` = registry domain (`docker.io`) / repo (`library/ubuntu`). Everything before the first `/` after the domain is the repository; everything after `:` is the tag.

## 8.5 Registry hygiene for production

- **Immutable tags** for releases (registry setting or policy).
- **GC / retention policies** — expired tags are deleted to control storage and cost.
- **Scanning** every pushed image (Trivy, Grype, cloud-native scanners) — block vulnerable images at push or deploy time.
- **Pull-through caching / mirroring** in air-gapped or compliance-heavy environments.
- **Auth + least privilege** — CI has push rights; runtimes have pull-only tokens.
- **Signing** (Notation / cosign) so you can verify image provenance (Part 10).

---

# Part 9 — Multi-Architecture & Platform Builds

## 9.1 The problem: hardware diversity

From the multi-arch video: 10 years ago developers worked on Windows, QE tested on Windows, then admins deployed to Linux — the *same jar* often behaved differently, and every promotion was a manual document of steps ("install Maven", "deploy to Tomcat"). Docker fixed the environment drift, but a new dimension appeared: **CPU architecture**.

- Apple Silicon → `arm64`. Most cloud VMs → `amd64`. Some ARM servers, Raspberry Pi, edge devices → `arm64`/`armv7`.
- A binary built for `amd64` will not run on `arm64`. `node:20-alpine` on an x86 machine and `node:20-alpine` on an ARM Mac are **different images** with the same tag.
- The classic failure: an image built on an M-series Mac runs fine locally but "exec format error" on an x86 cloud VM.

## 9.2 Multi-arch images — one tag, many platforms

A **multi-arch (multi-platform) image** is a *manifest list* (OCI index) that points to per-platform images:

```
 myapp:1.0  (manifest list)
   ├─ myapp:1.0@sha256:...  linux/amd64
   ├─ myapp:1.0@sha256:...  linux/arm64
   └─ myapp:1.0@sha256:...  linux/arm/v7

```

When you `docker pull myapp:1.0`, Docker automatically selects the manifest matching your runtime's `linux/amd64` or `linux/arm64`. One tag serves every machine type.

## 9.3 Building with Buildx

```bash
# create a multi-platform builder (uses QEMU emulation for other archs)
docker buildx create --name mybuilder --use
docker buildx build --platform linux/amd64,linux/arm64 \
    -t acme/web:1.0 --push .
```

Key points:
- **`buildx`** is the modern builder (BuildKit). Enable emulation for foreign architectures (QEMU) or use remote native builders for speed.
- **`--push`** pushes the full manifest list directly to the registry.
- **Verify:** `docker buildx imagetools inspect acme/web:1.0` shows all platforms.
- **Caution:** any architecture-specific step (native binaries, glibc vs musl builds, `CGO_ENABLED`, apt packages) must be done per-platform. Cross-compiling Go with `GOOS/GOARCH` avoids QEMU entirely.

## 9.4 Design implications

- Ship multi-arch from day one if you have any ARM machines or Apple developers; it costs little and prevents "exec format error" surprises in production.
- CI should build for *all* target platforms on every release, not just the one the CI runner happens to be.
- Testing must run on the same architecture as production at least for the primary platform.

---

# Part 10 — Security & Production Hardening

## 10.1 The threat model

A containerized production system has a chain of trust and a chain of attack. Think in layers:

```
Base image supply chain → Dockerfile → image → registry → runtime → network → host
      (from whom?)   (what's inside?) (scanned?) (provenance) (isolation) (published?) (escape?)
```

Every link can be attacked. The hardening practices below close them one by one.

## 10.2 The security stack (bottom-up)

| Layer | Practice | Tool/mechanism |
|---|---|---|
| Supply chain | Pin base image digests; use official/minimal images; sign images | Digest pinning, Notation/cosign |
| Image content | Scan every build; fail on critical/high CVEs; multi-stage + distroless | Trivy, Grype, Snyk |
| Image history | Never bake secrets; `--secret` build mounts, not ARG/ENV secrets | BuildKit secrets |
| Runtime identity | Non-root user; drop capabilities; read-only rootfs | `USER`, `--cap-drop=ALL`, `--read-only` |
| Runtime isolation | seccomp/AppArmor profiles; resource limits | Docker defaults + profiles |
| Network | Custom networks per tier; publish minimum | Part 6 |
| Host | Keep Docker/OS patched; rootless mode; restrict docker group | rootless, group audit |
| Registry | Auth, scanning on push, immutability, pull-through cache | Part 8 |

## 10.3 Supply chain: signing and provenance

- **Image signing** (Notation or cosign): the registry can enforce that only *signed* images are allowed. This proves the image came from your build pipeline, not an imposter.
- **SBOM** (Software Bill of Materials): each image lists its dependencies. Scanning + SBOM = you can answer "what's in this image and what CVE affects it" on demand.
- **Policy enforcement** (e.g., Sigstore policy controller / admission webhooks in k8s): *deny* unsigned images at deploy time, don't just warn.

## 10.4 Secret handling (the top production mistake)

Rules that prevent real breaches:
1. **Never** `COPY .env` or `ARG SECRET` into the image — it lands in layer history and survives forever.
2. Use **BuildKit secret mounts** for build-time secrets:
   ```dockerfile
   RUN --mount=type=secret,id=gh_token npm ci
   ```
   `docker build --secret id=gh_token,src=$HOME/.gh_token .` — secret is never in a layer.
3. At **runtime**, inject secrets via environment, mounted files (Secrets in k8s), or a secrets manager (Vault, AWS Secrets Manager) — never hardcode in the image.

## 10.5 Least privilege runtime

```bash
docker run --rm \
  --user 10001:10001 \
  --cap-drop=ALL \
  --read-only \
  --security-opt no-new-privileges \
  --tmpfs /tmp \
  --network my-net \
  myapp
```

- `--user 10001` → app never runs as root inside the container.
- `--cap-drop=ALL` → even root inside is crippled; add back only what's needed (`--cap-add NET_BIND_SERVICE` for port 80).
- `--read-only` → the writable layer is frozen; only explicit volumes/tmpfs are writable. Kills whole classes of persistence attacks.
- `--no-new-privileges` → blocks privilege escalation.

In Dockerfiles, bake these in: `USER`, and in compose `security_opt`, `read_only`, `cap_drop`.

## 10.6 Scanning in the pipeline

```bash
trivy image myapp:1.0            # scan locally
trivy repo <git-url>             # scan the source dependencies too
trivy fs .                       # scan working dir
```

Integrate into CI: build → scan → fail on high/critical → push → sign → deploy. Treat the scanner output like a compiler error.

## 10.7 Host-level hardening

- **Rootless Docker** where feasible — no daemon running as root.
- **Audit the `docker` group** — membership equals root access.
- Keep **Docker Engine and the host kernel patched**; container escapes target kernel bugs.
- Restrict **sockets/API** access; never expose the Docker socket over the network without TLS + auth.
- On Kubernetes, add **Pod Security Standards / Admission Control** so policy is enforced centrally.

---

# Part 11 — The Modern Toolbox: Docker Init, Model Runner, Docker Offload, the Ecosystem

## 11.1 Docker Init — stop hand-writing Dockerfiles

`docker init` is the answer to *"how do I write a Dockerfile when I'm not confident about the build process?"* It analyzes your project and generates a production-shaped `Dockerfile`, `.dockerignore`, and `compose.yaml` — with sensible base image, health checks, and best practices baked in.

```bash
cd myapp
docker init        # detects language (Python/Node/Go/etc.), asks a few questions
# → creates Dockerfile, .dockerignore, compose.yaml
docker compose up  # run the generated stack
```

Why it matters: the generated output *is* a teaching artifact. Read it, understand it, then customize. It also eliminates the most common onboarding failure — a developer who can't package their own app. For a design engineer, use it to bootstrap, but never ship blind: audit base image, ordering, user, and health check.

## 11.2 Docker Model Runner — running LLMs as containers

Docker Model Runner (built into current Docker Desktop) runs local models with Docker-style commands:

```bash
docker model pull llama3.2      # download model (like docker image pull)
docker model run llama3.2        # run it
docker model rm llama3.2         # remove
```

`docker model --help` shows the command set; model files are stored by the platform. The transcript's framing: *"you can download models, run them locally, and integrate your applications with the local models to build AI agents or assistants."* It competes with Ollama; the advantage is a unified toolchain (same UX as containers) and no separate runtime to manage.

Design relevance: on-prem/edge AI workloads, data-residency requirements, and dev-time agents all need a local inference runtime. Model Runner (and Ollama, vLLM, etc.) are how you deploy model-serving as infrastructure instead of a black-box SaaS call. For a design answer about AI systems, name the inference container as a first-class service with GPU/resource limits, and remember Model Runner implies the *latest* Desktop; on servers you would use the OSS runtime of choice.

## 11.3 Docker Offload — moving containers off your machine

The Offload feature lets you run containers **on Docker-managed cloud infrastructure** without SSHing into a VM: flip a toggle (or a CLI command) in Docker Desktop and the container runs remotely while the local experience stays seamless.

Why it exists (from the transcript): many teams run resource-hungry workloads (three-tier apps with 50+ microservices, LLMs) on cloud VMs because their laptop can't handle them — but SSHing to a VM is a broken developer experience. Offload removes the manual VM chore.

Design relevance: it signals where the industry is going — **local vs remote execution is becoming a toggle**, and the container is the portable unit that makes that possible. Architect your apps so they can run on a laptop or a fleet without code changes; the deployment target becomes an implementation detail.

## 11.4 The beyond-Docker ecosystem

You are not learning "Docker" — you are learning **OCI containers**, of which Docker is one implementation:

| Tool | Role |
|---|---|
| **Podman** | Rootless, daemonless Docker-compatible CLI; drop-in in many setups |
| **Buildah** | Build images without a daemon |
| **containerd** | The runtime Kubernetes actually uses (CRI) |
| **runc** | The OCI runtime that creates processes |
| **Kubernetes** | Orchestration: the next level after this document |
| **K3s / MicroK8s** | Lightweight k8s for edge and small prod |
| **Nerdctl / Skopeo** | containerd-native CLI / image inspection & copying |

The portable asset is the **image** (OCI spec) and your understanding of *isolation, storage, networking, lifecycle* — all of which transfer to every tool above.

---

# Part 12 — System Design Integration: Architecting with Containers

## 12.1 Containers are the deployment unit in your diagrams

When you draw a system design (diagram on whiteboard), every box is a **service**, and in the real world every service ships as a **container**. Your diagram is not complete unless you can answer, for every box:

1. **What image?** — base, tag/digest, size, registry.
2. **What resources?** — CPU, memory limits; request vs limit.
3. **Stateless or stateful?** — if stateful, *where* is the state and how is it backed up/replicated? (Part 5)
4. **Which network?** — what can it reach, what can reach it? (Part 6)
5. **How is it health-checked, logged, and restarted?** — liveness, readiness, stdout logging.
6. **How is it scaled?** — more replicas, and is it safe to run more than one? (shared state?)

## 12.2 The stateless-first principle

The strongest design move in container land:

> **Design services to be stateless. Put all durable state in external systems (managed database, object storage, cache). Then every service becomes a commodity that can be scaled, rescheduled, and replaced at will.**

Consequences:
- Horizontal scaling becomes trivial (replicas are interchangeable).
- Rolling deploys and blue/green are safe (old and new coexist).
- Failures are cheap (kill a bad instance, spawn a new one).
- Cost control is easy (scale to zero off-peak).

Contrast: a stateful service (database inside a container with local volume) pins itself to a node, complicates backup, and makes scaling painful. That's why production databases live outside the container swarm or run with serious orchestration (operators).

## 12.3 The 12-factor mindset

The 12-factor app principles *are* the container-native design rules. The ones that matter most here:

- **Config in environment** — everything that varies between environments comes from env vars, never baked into the image.
- **Stateless processes** — no local persistence, no sticky sessions.
- **Logs as event streams** — write to stdout; the platform collects (Part 5.6).
- **Disposability** — fast startup and clean shutdown; the process may die at any moment.
- **Dev/prod parity** — the same image that passes CI is what runs in prod (partly guaranteed by immutability + registries).
- **Dependencies declared** — lockfiles in the image build.

If your design violates a 12-factor rule, you should be able to say *why* — that awareness is what separates a senior answer from a memorized one.

## 12.4 Service discovery, scaling, and deployment

- **Discovery:** containers get new IPs constantly; resolve by **name/DNS**, not IP (Part 6.3). In k8s this is a Service; in Compose it's the network DNS.
- **Scaling unit:** the container. Scale by replicas; the orchestrator (or a proxy) load-balances. Capacity planning = replicas × per-container request/limit.
- **Deployment strategies:** rolling (recreate pods one at a time), blue-green (two full stacks, switch traffic), canary (gradual %). All require stateless, health-checked containers.
- **Readiness vs liveness:** readiness = "can serve traffic" (for load balancer), liveness = "is the process alive" (for restart). Design endpoints for both.

## 12.5 The container-native reference architecture

```
                  Clients
                     │  HTTPS
                     ▼
        ┌───────────────────────────┐
        │   Load balancer / Ingress  │   (TLS termination, routing)
        └──────────────┬────────────┘
                       │
        ┌──────────────▼──────────────┐
        │  API gateway / BFF (stateless)│  replicas: 3
        └──────┬──────────────┬───────┘
               │              │
        ┌──────▼─────┐  ┌─────▼──────┐
        │  Services  │  │  Services  │   (stateless, N replicas each)
        └──────┬─────┘  └─────┬──────┘
               │              │
        ┌──────▼──────────────▼───────┐
        │   Message queue / cache      │   (Redis, Kafka)
        └──────┬──────────────┬───────┘
               │              │
        ┌──────▼──────────────▼───────┐
        │   Data tier (MANAGED, external)│   (DB-as-a-service / object store)
        └──────────────────────────────┘
```

Rules encoded in this picture:
- Stateless tiers scale horizontally; the **data tier is external and managed**.
- Every tier on its own network segment (Part 6.5).
- Every container has limits, health checks, and stdout logging.
- Deployment is image-based; rollback is "deploy the previous tag".

## 12.6 What changes when you add Kubernetes

The same architecture maps 1:1:
- Services → Deployments + Services + Ingress.
- Networks → namespaces + network policies.
- Volumes → PVCs / CSI.
- Env/config → ConfigMaps/Secrets.
- Health checks → liveness/readiness probes.
- Compose file → Helm/kustomize manifests.

This is *the bridge* this document has been building toward: you have already designed the topology; k8s is the mechanism to run it across many machines with self-healing. The Docker knowledge (images, storage, networking, security) is exactly what you need before learning k8s — which is the explicit sequence of the source course.

---

# Part 13 — The Phased Learning Path

A dependency-ordered path. Do not skip phases — each builds on the last. Typical total: 6–10 weeks of focused practice.

## Phase 0 — Mental model (day 1)

- Read Parts 0 and 1 of this document.
- Write down the five production problems from memory.
- Answer: "What is a container?" in 2 sentences with the words *process, kernel, namespaces, cgroups, filesystem*.

**Exit check:** you can explain to a friend why a container is not a VM.

## Phase 1 — Hands-on fundamentals (days 2–5)

- Install Docker (Desktop on your machine / Engine on a cloud VM).
- `docker run hello-world`; then a CLI app (`docker run -it ubuntu bash`, alpine).
- `docker run -d nginx`, `docker ps`, `docker logs`, `docker exec`, `docker stop/start/rm`.
- Compare `docker inspect` output fields with the concepts in Part 2.
- Hit the **permission denied** wall on Linux; fix it with the docker group (Part 2.6).

**Exit check:** you can run, inspect, log, exec into, and remove containers without googling.

## Phase 2 — Images and the Dockerfile (days 6–10)

- Containerize a simple Python/Flask or Node app by hand first, then with `docker init` (Part 11.1) and compare.
- Learn layer caching empirically: change only source code and watch the rebuild reuse dependency layers.
- Apply Part 3 best practices one by one, measuring image size after each (`docker images`).
- Do the multi-stage + distroless exercise (Part 4) and record the size reduction.
- Write a `.dockerignore`.

**Exit check:** you can take an unfamiliar app and produce a lean, cached, non-root image.

## Phase 3 — Storage and networking (days 11–15)

- Reproduce the nginx log-loss story (Part 5): run nginx, check logs, remove container, show logs gone, then fix with a volume.
- Practice: named volume for a DB, bind mount for configs, `tmpfs` for scratch.
- Create two custom bridge networks; prove containers on different networks can't talk (Part 6.3).
- Publish ports; bind to localhost; explain each `-p` flag.
- Diagram the traffic path for a request reaching a container.

**Exit check:** you can explain and *demonstrate* the ephemeral nature of containers and the network isolation model.

## Phase 4 — Compose and multi-container apps (days 16–21)

- Build a full stack: web app + Postgres + Redis + reverse proxy in `compose.yaml`.
- Add health checks, `depends_on` readiness, named volumes, env files.
- Break things deliberately: stop the DB, watch the web app fail *and* recover with restart policies.
- Compare your compose file to a k8s mental mapping (Part 7.4).
- Push images to a registry (Docker Hub or GHCR) and pull them on another machine (Part 8).

**Exit check:** one command (`docker compose up`) reliably brings up your whole app; you understand each moving part.

## Phase 5 — Security hardening (days 22–26)

- Scan your images with Trivy; fix or document every high CVE.
- Rebuild with distroless/non-root/`--cap-drop=ALL`/`--read-only` and re-scan.
- Set up signing (cosign) and verify provenance.
- Audit your compose file against the Part 10 checklist.
- Implement BuildKit secret mounts for a build-time secret.

**Exit check:** you can defend every image and runtime decision against a security review.

## Phase 6 — Production stories & observability (days 27–30)

- Practice the incident narrative template from Part 4.5 for each topic: storage loss, network misconfig, image bloat, arch mismatch, CVE block.
- Wire logging to stdout; collect with a stack of your choice (Loki/Grafana, ELK, or cloud).
- Add metrics (Prometheus client) and alert on a failing health check.
- Practice rolling a change and rolling back using images as the unit.

**Exit check:** you can run a load test, watch a container die, and describe root cause + fix in production language.

## Phase 7 — Kubernetes bridge (weeks 5–8, optional but expected for a design engineer)

- Learn the mapping: Deployment, Service, Ingress, ConfigMap, Secret, PVC, NetworkPolicy.
- Run a local cluster (kind/k3s/minikube); deploy the compose app as manifests.
- Practice rolling updates, self-healing, and scaling replicas.

**Exit check:** you can deploy the same app you designed in Part 12.5 on a cluster.

## Phase 8 — Interview preparation (ongoing)

- Work through Part 15 scenarios out loud, time-boxed (2 minutes each).
- Rebuild each Part's one-paragraph "interview-ready statement".
- Draw the reference architecture from Part 12.5 from memory and narrate it.

## Weekly rhythm suggestion

- 60–70% hands-on, 30–40% reading/diagramming.
- Keep a running "production problems I have actually caused" list — it becomes interview gold.
- Teach one concept per week to someone (writing this document's style: explain *why*, give the failure mode, give the fix).

---

# Part 14 — Hands-On Labs (Production-Flavored)

Ten labs, ordered. Each has a *goal*, *steps*, and a *gotcha to experience on purpose* — because deliberately breaking things is how this sticks.

## Lab 1 — Hello, process

- Goal: run your first container; understand PID 1.
- Steps: `docker run hello-world`; `docker run -it ubuntu bash`; `ps aux` inside (note: no other system processes — this is the namespace at work); `exit`.
- Gotcha: run `docker run -d ubuntu` and watch it **exit instantly** — no process to keep running, nothing to do. Explains `CMD` and foreground processes.

## Lab 2 — Web server + logs (the ephemerality lesson)

- Goal: experience container state loss.
- Steps: `docker run -d -p 8080:80 nginx`; curl the server; `docker exec` to read `/var/log/nginx/access.log`; `docker rm -f` the container; recreate it; check the log — **gone**.
- Fix: named volume for `/var/log/nginx`; repeat; log survives `rm`.
- Deliverable: a one-paragraph explanation of ephemeral vs persistent, using your own output.

## Lab 3 — Dockerfile from scratch + docker init

- Goal: hand-build a Flask/Node image, then let `docker init` generate one, then diff.
- Steps: write Dockerfile, `.dockerignore`; build; measure size; `docker init`; compare base image, ordering, user, health check.
- Gotcha: leave `COPY . .` before installing deps → change a file → watch the whole dependency layer rebuild (caching loss).

## Lab 4 — Multi-stage size hunt

- Goal: shrink an image ≥ 80%.
- Steps: single-stage full-OS build → measure → multi-stage → measure → distroless → measure. Record each number. Explain each removal (compilers, headers, shell, caches).
- Verify with `docker history <image>` which layers remain.

## Lab 5 — Networking isolation

- Goal: prove custom networks isolate.
- Steps: create `front-net` and `back-net`; run one container per network; `docker exec` ping the other (fails); add second container to *same* network (ping works by **name**, not IP). Then `docker network connect` to bridge the gap intentionally.
- Gotcha: containers on the default bridge CAN ping each other — demonstrate why that's the insecure default.

## Lab 6 — Compose full stack

- Goal: 4-service app, one command.
- Steps: web + postgres + redis + caddy/nginx proxy in `compose.yaml`; health checks; `depends_on` readiness; named volumes; `.env`. Bring down and up; confirm data persists; `down -v` and confirm data is wiped (knowingly).

## Lab 7 — Registry round trip

- Goal: image travels machines.
- Steps: tag → push to Docker Hub/GHCR → pull on a cloud VM → run. Then scan on push.
- Gotcha: push without login; push `latest`; then pull by **digest** and prove immutability.

## Lab 8 — Multi-arch build

- Goal: one tag, two platforms.
- Steps: `docker buildx create --use`; build for `linux/amd64,linux/arm64`; `docker buildx imagetools inspect`; run the amd64 image on an ARM host (with emulation) to see it work, then a native-only image to see it fail.
- Gotcha: "exec format error" — the arch mismatch, now understood.

## Lab 9 — Security hardening

- Goal: make a scanner and a reviewer happy.
- Steps: Trivy scan → fix CVEs → non-root + `--cap-drop=ALL` + `--read-only` → re-scan → sign with cosign → verify.
- Gotcha: an `ARG`/`ENV` secret visible in `docker history` — then redo with BuildKit `--mount=type=secret`.

## Lab 10 — Mini production incident drill

- Goal: incident narrative practice.
- Setup: running compose app. Then (pick one): kill the DB container, delete a volume, break the network, deploy a broken image.
- Playbook: check `docker ps`, `docker logs`, `docker inspect` → root cause → fix → *restore* → write the story in Part 4.5 template (situation/symptom/root cause/fix/result).

Each lab closes with the same question: **"What production problem did this lab just save you from?"** If you can't answer, redo the lab.

---

# Part 15 — Scenario-Based Interview Mastery

The source course (Day-29) deliberately made its 12 questions **scenario-based**, because real interviewers ask "here is a situation" not "what is a command." Below: the question, a strong answer *structure*, and the concept each one tests.

## 15.1 "What is Docker?" (the trap question)

Interviewers use this to test **containers**, not the platform (Part 0 reality check).
- **Answer shape:** Docker is a container platform that manages the lifecycle of containers. A container is an isolated process sharing the host kernel, with its own filesystem/network/process tree (namespaces), resource limits (cgroups), built from immutable image layers. Docker gives you build (Dockerfile), distribute (registry), and run (engine/Desktop) — the *why* is reproducible environments that kill "works on my machine."

## 15.2 "Your image is 2 GB. What do you do?"

- **Answer shape:** measure → root cause → fix.
  - `docker history` to see layer sizes.
  - Likely causes: fat base image, dev dependencies, build tools, caches, whole build context copied.
  - Fix: alpine/slim base, `.dockerignore`, multi-stage build (builder → runtime), prune/clean caches, distroless if possible.
  - Result: target ≤ 80% reduction; cite deploy speed, node storage, CVE surface.

## 15.3 "The container starts and immediately stops."

- **Answer shape:** a container exits when its PID 1 process exits.
  - `docker logs <ctr>` — the first grep.
  - Common causes: wrong `CMD`, app crash on startup (missing env/DB), foreground vs background process, `entrypoint` misconfig.
  - Run a debug shell (`docker run --rm -it --entrypoint sh <image>`) to inspect.
  - With orchestration: understand CrashLoopBackOff = same problem at scale.

## 15.4 "Data is gone after a restart."

- **Answer shape:** containers are ephemeral; writable layer dies with the container.
  - Fix: named volumes for state, bind mounts for configs (Part 5).
  - Design upgrade: make the service stateless; move state to external DB/object store (Part 12.2).
  - Mention: backups (RPO), replication, and that local volumes pin to one node in multi-node setups.

## 15.5 "Containers can't reach each other."

- **Answer shape:** networking diagnosis.
  - Are they on the same network? (`docker inspect --format '{{.NetworkSettings.Networks}}'`).
  - Are they using the default bridge (NAT, no DNS by name) vs a custom network (DNS by name)?
  - Is the target service published/addressable? Is a firewall/security group blocking?
  - Fix pattern: one custom network per tier, resolve by service name, publish only entry points (Part 6).

## 15.6 "How do you reduce image size?" / "multi-stage vs distroless?"

- **Answer shape:** multi-stage removes build-time tooling from the final image; distroless removes shell/runtime utilities. Combine both. Tradeoff: distroless is harder to debug — compensate with health checks, structured logs, ephemeral debug containers (Part 4.4 statement).

## 15.7 "Container is running but the app is unreachable."

- **Answer shape:** separate **publishing** from **exposure**.
  - Is the port published (`-p`/`ports`)? Is the app bound to `0.0.0.0` inside the container, or `127.0.0.1` only?
  - Is the health check passing? Is the host firewall/security group allowing the port?
  - Check `docker port <ctr>` to see the actual mapping.
  - Under orchestration: Service selector labels must match pod labels.

## 15.8 "How do you make configs and secrets safe?"

- **Answer shape:** environment for config; never bake secrets.
  - Build-time secrets via BuildKit `--mount=type=secret`; runtime via env/secrets manager (Vault, cloud secrets, k8s Secrets).
  - `.dockerignore` against `.env`; scanning + SBOM + signing for supply chain (Parts 8–10).
  - Mention the "secret lives in layer history forever" failure mode.

## 15.9 "Containers are running as root — so what?"

- **Answer shape:** inside the container you share the host kernel; root inside + a kernel bug = host compromise.
  - Fix: `USER` non-root in image, `--cap-drop=ALL`, `--no-new-privileges`, seccomp/AppArmor, `--read-only`, rootless Docker. Defend why least privilege is a design requirement, not a nicety (Part 10).

## 15.10 "How do you scale a containerized service?"

- **Answer shape:** the unit is the replica; scaling requires statelessness.
  - Requests/limits per container inform capacity math (replicas × per-container).
  - Discovery by name/DNS, LB in front; rolling deploys + health checks (readiness vs liveness).
  - If the service holds state, scaling is not safe without external state (Part 12.2).

## 15.11 "Compose vs Kubernetes — when do you use which?"

- **Answer shape:** single host / dev / CI / small prod → Compose. Multi-node, self-healing, rolling updates, network policies, CSI storage → Kubernetes. Same topology expressed differently (Part 7.4 table).

## 15.12 "Walk me through how an image goes from code to production."

- **Answer shape:** code → commit → CI builds (multi-stage, scanned, signed, multi-arch) → push to registry by digest → orchestrator pulls → container runs with limits, health checks, secrets from env → logs to stdout → collected/metrics/alerting → rollback = deploy previous tag.
- This is the capstone answer; it touches every part of this document and should be your practiced "elevator" version.

## 15.13 Bonus designer questions (beyond the course)

- "Where would you *not* use containers?" → different kernel needs, bare-metal driver apps, heavy isolation for untrusted code (VM/microVM better).
- "A node dies. What happens to the containers?" → stateless: rescheduled elsewhere; stateful local-volume data is at risk — this is *why* external/managed state wins.
- "Design a platform to run 50 microservices." → start from the Part 12.5 reference architecture; add k8s, ingress, service mesh option, network policies, central logging/metrics, and env-based config.

## 15.14 Answering rules (general)

1. **Scenario first, command second.** Name the problem you're solving, then the mechanism.
2. **Always give the failure mode** — what happens in production if you skip it.
3. **Quantify** where possible (image size, deploy time, CVE count).
4. **Link parts** — storage answers should reference statelessness; networking answers should reference isolation; security answers should reference supply chain.

---

# Part 16 — Deep Synthesis: The One Mental Model

## 16.1 The container is a contract

At the highest level, a container is a **contract between the artifact and the runtime**:

- **The image says:** "Here is my filesystem, my dependencies, my entrypoint, my declared ports and env — this is exactly what I need."
- **The runtime says:** "I give you a kernel, an isolated view (namespaces), fair resource limits (cgroups), storage (volumes), a place on the network, and a promise to restart you if you die."

Every topic in this document is one clause of that contract:
- **Dockerfile** — writing the contract.
- **Multi-stage/distroless** — writing a lean, safe contract.
- **Registries/multi-arch** — distributing the contract to any machine.
- **Volumes** — the contract's promise about state.
- **Networking** — the contract's relationships.
- **Security** — enforcing the contract's boundaries.
- **Compose/k8s** — orchestrating many contracts into a system.

## 16.2 The three loops of production thinking

Everything you do in production reduces to three loops:

```
 1. REPRODUCE     build once → run anywhere (image immutability + registry + multi-arch)
 2. SURVIVE       ephemeral, restartable, health-checked, state external (self-healing)
 3. EVOLVE        change safely: new image, rollback on failure (rolling/canary/blue-green)
```

Any production incident is a break in one of these loops: a "works on my machine" is a *reproduce* break; lost data is a *survive* break; a bad deploy you can't undo is an *evolve* break. When you diagnose, first ask *which loop broke?*

## 16.3 The tradeoff lattice

Senior answers are about tradeoffs, not facts. The core container tradeoffs:

| Tradeoff | Lean | But watch out for |
|---|---|---|
| Size vs debuggability | Small (distroless) | Harder to debug → invest in logs/health |
| Isolation vs density | Isolation (VM for hostile) | Container sharing kernel is the boundary |
| Simplicity vs scale | Compose first | It caps at one host |
| Local vs remote | Local first | Then offload/cloud when resources exceed |
| Built-in vs managed state | External managed state | Cost & latency of remote data |
| Rich image vs minimal | Minimal | May need runtime bits (certs, glibc) |

## 16.4 One paragraph that holds it all

> Containers make the environment a code artifact: an immutable image, built lean (multi-stage, distroless), distributed through a registry (tags and digests, multi-arch), run as isolated kernel processes (namespaces, cgroups) with ephemeral lifecycles. State is external or in volumes; communication is governed by networks and isolation; security is layered from base image to host. Orchestration (Compose → Kubernetes) turns many such containers into a self-healing system where the unit of change is always the image — and that is why learning containers is the prerequisite to designing, scaling, and operating real systems.

---

# Part 17 — Production Readiness Checklist

Use this before shipping any containerized service. Every item links to the Part that justifies it.

## Image
- [ ] Base image pinned (tag or digest), not `latest` (P3, P8)
- [ ] Minimal base chosen (slim/alpine/distroless) with rationale (P3, P4)
- [ ] Multi-stage build; no build tools in runtime image (P4)
- [ ] `.dockerignore` present; no secrets in build context (P3)
- [ ] No secrets in image history; build secrets via BuildKit mounts (P10)
- [ ] Non-root `USER`; capabilities dropped (P10)
- [ ] Lockfiles included for dependencies (P3)
- [ ] Instructions ordered for layer-cache efficiency (P3)
- [ ] Health check defined (`HEALTHCHECK` / probe) (P3)
- [ ] Scanned with Trivy; no unmitigated critical/high CVEs (P10)
- [ ] Signed; provenance verified (P10)
- [ ] Multi-arch manifest built for all target platforms (P9)

## Runtime
- [ ] Resource limits set (CPU, memory) per container (P1)
- [ ] `--read-only` + `tmpfs` for scratch where possible (P10)
- [ ] State in named volumes; stateless design preferred (P5, P12)
- [ ] Backups/RPO defined for any stateful data (P5)
- [ ] Logs to stdout; collector wired (P5, P12)
- [ ] Metrics + alerting on health checks (P13)

## Networking
- [ ] Services on custom networks; tier segmentation (P6)
- [ ] Only entry points published; DBs/caches internal only (P6)
- [ ] Name-based service discovery, no hardcoded IPs (P6)
- [ ] Ingress/LB is the single public path (P6, P12)

## Deployment & Operations
- [ ] Deployable by image tag; rollback = previous tag (P8, P12)
- [ ] Rolling/canary strategy; readiness vs liveness understood (P12)
- [ ] Compose for dev/CI; k8s mapping understood (P7, P12)
- [ ] Incident narrative template rehearsed (P4, P15)

---

# Appendix A — Command Cheat Sheet

## Lifecycle
```bash
docker pull nginx:1.27-alpine     # fetch image
docker run -d --name web -p 8080:80 nginx   # run detached, publish port
docker run --rm -it ubuntu bash   # interactive, remove on exit
docker ps / docker ps -a          # running / all containers
docker logs -f web                # follow logs
docker exec -it web bash          # shell into container
docker inspect web                # full config (networks, mounts, state)
docker stop web && docker start web
docker rm -f web                  # force remove (deletes writable layer!)
docker prune                      # clean dangling images/containers/volumes
```

## Images
```bash
docker build -t myapp:1.0 .                 # build
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 --push .  # multi-arch
docker history myapp:1.0                    # layer sizes
docker image inspect myapp:1.0              # metadata
docker tag myapp:1.0 acme/web:1.0.0         # retag
docker push acme/web:1.0.0 / docker pull acme/web:1.0.0
docker rmi myapp:1.0                        # remove image
```

## Storage
```bash
docker volume create app-data
docker run -v app-data:/var/lib/postgresql/data postgres:16
docker run -v "$PWD/src:/app:ro" myapp      # bind mount, read-only
docker run --tmpfs /tmp myapp               # in-memory
docker run --rm -v app-data:/data -v "$PWD:/backup" alpine tar czf /backup/data.tgz /data  # backup
```

## Networking
```bash
docker network ls
docker network create --driver bridge front-net
docker network connect front-net web        # add container to a network
docker network inspect front-net            # see members
docker run --network front-net --name svc myapp
```

## Compose
```bash
docker compose up -d          # start stack
docker compose ps / logs -f / config
docker compose down           # stop (keeps volumes)
docker compose down -v        # stop + delete volumes (data loss!)
docker compose exec web bash  # shell into service
```

## Security
```bash
docker scan myapp:1.0                 # deprecated; prefer:
trivy image myapp:1.0                 # CVE scan
cosign sign acme/web:1.0.0            # sign
cosign verify acme/web:1.0.0          # verify
docker run --user 10001:10001 --cap-drop=ALL --read-only --security-opt no-new-privileges myapp
```

## Model Runner
```bash
docker model pull llama3.2
docker model run llama3.2
docker model rm llama3.2
docker model --help
```

---

# Appendix B — Key Terms Glossary

| Term | Definition |
|---|---|
| **Container** | An isolated process sharing the host kernel, with own filesystem/network/process tree |
| **Image** | Immutable blueprint: layered filesystem + metadata (entrypoint, env, ports) |
| **Layer** | One instruction's filesystem change; cached and shared across images |
| **Dockerfile** | Declarative recipe for building an image |
| **Docker Engine** | Linux daemon (dockerd) + client; the production runtime |
| **Docker Desktop** | Dev tool for macOS/Windows; runs a Linux VM underneath |
| **containerd / runc** | Container supervisor and OCI runtime under the daemon |
| **Registry** | Storage/distribution service for images (Docker Hub, GHCR, ECR) |
| **Tag / Digest** | Mutable human label vs immutable content hash |
| **Multi-arch image** | Manifest list serving one tag to many CPU platforms |
| **Multi-stage build** | Multiple `FROM` stages; only the final stage ships |
| **Distroless** | Runtime-only images with no shell or package manager |
| **Volume** | Docker-managed persistent directory decoupled from container |
| **Bind mount** | Direct host-path mount into a container |
| **tmpfs** | RAM-backed scratch storage, never persisted |
| **Bridge network** | Default per-host virtual network with NAT + port publish |
| **Host network** | Container shares host network stack directly |
| **Overlay network** | Multi-host virtual network (VXLAN); basis for Swarm/k8s CNI |
| **Namespace** | Kernel mechanism isolating views (PID, net, mount, UTS, IPC, user) |
| **Cgroup** | Kernel mechanism limiting/accounting CPU, memory, I/O |
| **Seccomp / AppArmor / capabilities** | Kernel mechanisms constraining what a process may do |
| **Compose** | Declarative multi-container app definition (`compose.yaml`) |
| **Health check / probe** | Mechanism by which the platform knows a container is alive/ready |
| **Ephemeral** | Short-lived by design; container state dies with the container |
| **SBOM** | Software Bill of Materials — inventory of dependencies in an image |

---

*End of document. Revisit Part 16 whenever something feels disconnected — the five production problems (Part 0) and the three loops (16.2) are the map for everything else.*
