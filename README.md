# StackEval

**Measuring LLM Coding Agents Across Real-World Technology Stacks**

*Whitepaper Draft v0.5 — February 2026*

---

## The Problem

Every major LLM coding benchmark — SWE-bench, AgentBench, OSWorld, LiveCodeBench — evaluates agents inside a single, fixed, resource-rich environment. Usually an Ubuntu container on x86 with ample RAM and a known Python version.

The benchmark holds the stack constant to isolate model capability. This makes sense for controlled science. It does **not** make sense for evaluating production readiness.

In production there is no fixed stack. The same agent, same model, same prompt will land on Alpine Linux with musl libc and 2 GB RAM on Railway, or a Mac Mini M4 with 64 GB, or an AWS Graviton instance behind ECS with different CPU caps and no GPU. The code that works perfectly in the benchmark sandbox fails, times out, OOMs, or produces subtly wrong results on the stacks where software actually ships.

Current benchmarks cannot see this. They are measuring model intelligence in a vacuum. **StackEval measures it where it matters.**

### How Current Benchmarks See the World

```mermaid
flowchart TB
    subgraph bench [" "]
        direction TB
        M[/"Model A, B, C"/]
        S["Ubuntu x86 · 64 GB · Python 3.11<br/>(one fixed stack)"]
        R["✅ Pass / ❌ Fail"]
        M --> S --> R
    end

    style bench fill:#1a1a2e,stroke:#e94560,color:#eee
    style M fill:#0f3460,stroke:#e94560,color:#eee
    style S fill:#16213e,stroke:#e94560,color:#eee
    style R fill:#16213e,stroke:#e94560,color:#eee
```

> One model, one stack, one score. That is all existing benchmarks measure.

---

## The Core Observation

*This has not been systematically proven at scale. That is the point of this project. But the pattern is well-known to anyone who deploys.*

Consider giving the exact same coding agent the exact same task:

**Task**: Build a FastAPI service that ingests 8 GB of IoT sensor JSON, computes rolling statistics and anomaly detection, stays under 4 GB peak RAM, uses only pinned dependencies, and starts cleanly in under 60 seconds.

Now run it on three setups that thousands of developers actually use in 2026:

| Stack | Configuration | What Probably Happens |
|:---|:---|:---|
| **Local Mac Mini M4** | 64 GB, Apple Silicon, Python 3.12, no caps, NVMe | Agent succeeds quickly. Naive pandas works. Zero friction. **This is what every existing benchmark tests.** |
| **OpenRouter + Railway** | Same model via API, Alpine 3.20 musl, ARM64, 2 GB RAM cap | Wheels missing. OOM on naive pandas. Agent must probe, rewrite, adapt — or fail entirely. Same model, same prompt, radically different outcome. |
| **AWS Graviton + Ubuntu** | Ubuntu 24.04, ARM64, 8 GB RAM, ECS/Fargate CPU limits, no GPU | Agent succeeds but never probes fork-vs-spawn, uses too much RAM, starts slowly. Passes checks but would be expensive and fragile at scale. |

These are not exotic configurations. These are the three most common deployment patterns used by startups and enterprises today. Current benchmarks only ever test the first row.

### How StackEval Sees the World

```mermaid
flowchart TB
    subgraph se [" "]
        direction TB
        M[/"Model A"/]
        S1["Mac Mini M4<br/>64 GB · macOS · Py 3.12"]
        S2["Railway<br/>Alpine musl · ARM64 · 2 GB"]
        S3["AWS Graviton<br/>Ubuntu · ARM64 · 8 GB"]
        S4["RPi5 Edge<br/>Fedora · ARM64 · 8 GB"]
        R1["✅ 4 min · 1.2 GB peak"]
        R2["⚠️ 28 min · OOM x2 then adapt"]
        R3["✅ 9 min · 6.8 GB peak"]
        R4["❌ timeout"]
        M --> S1 --> R1
        M --> S2 --> R2
        M --> S3 --> R3
        M --> S4 --> R4
    end

    style se fill:#1a1a2e,stroke:#00b4d8,color:#eee
    style M fill:#0f3460,stroke:#00b4d8,color:#eee
    style S1 fill:#16213e,stroke:#00b4d8,color:#eee
    style S2 fill:#16213e,stroke:#00b4d8,color:#eee
    style S3 fill:#16213e,stroke:#00b4d8,color:#eee
    style S4 fill:#16213e,stroke:#00b4d8,color:#eee
    style R1 fill:#1b4332,stroke:#52b788,color:#eee
    style R2 fill:#5a3e1b,stroke:#e9c46a,color:#eee
    style R3 fill:#1b4332,stroke:#52b788,color:#eee
    style R4 fill:#4a1525,stroke:#e94560,color:#eee
```

### Hypothetical Results

> **Not real data.** This is the shape of result we expect the pilot to produce.
> The point: a model's benchmark score tells you nothing about *this* axis.

```
Task: FastAPI + 8 GB JSON ingest + anomaly detection (pinned deps, 4 GB RAM target)
Model: [Frontier Agent X]

                        Success    Wall-Clock     Peak RAM    Composite
                        ───────    ──────────     ────────    ─────────
Mac Mini M4 (64 GB)    ██████████  ████            ██          ████████
                         100%       4 min          1.2 GB       0.94

AWS Graviton (8 GB)    ██████████  ████████        ████████    █████
                         100%       9 min          6.8 GB       0.58

Railway Alpine (2 GB)  ██████      ██████████████  ██████████  ██
                          60%       28 min         OOM→adapt    0.22

RPi5 Edge (8 GB)       ██          ████████████    ██████████  █
                          20%       timeout        crash        0.08
```

```
Same model. Same prompt. Same task.
SWE-bench only ever tests Row 1.
```

---

## The Thesis

**The real unit of evaluation is not the Model. It is (Stack + Model).**

Model-capacity benchmarks measure whether an agent can solve a problem in ideal conditions. StackEval measures whether it can solve the same problem across the conditions where software actually runs. These are different capabilities. We expect the delta between them to be large, consistent, and currently invisible to the field.

We do not yet have the data to quantify this precisely. That is the purpose of StackEval: to generate it.

```mermaid
quadrantChart
    title Model Capacity vs Stack Robustness
    x-axis "Low Capacity" --> "High Capacity"
    y-axis "Low Robustness" --> "High Robustness"
    quadrant-1 "Production Ready"
    quadrant-2 "Robust but Weak"
    quadrant-3 "Fragile and Weak"
    quadrant-4 "Lab Hero, Field Zero"
    "Agent A (tested)": [0.82, 0.78]
    "Agent B (untested)": [0.80, 0.28]
    "Agent C (portable)": [0.50, 0.72]
    "Agent D (brittle)": [0.28, 0.22]
```

> The bottom-right quadrant — **"Lab Hero, Field Zero"** — is where we suspect most frontier agents actually live today. StackEval is the only way to find out.

---

## What StackEval Is

A benchmark and open leaderboard that runs identical agent tasks across a controlled matrix of production-representative stacks and measures success, speed, resource use, and adaptation intelligence.

### StackConfig.json

A single lightweight JSON schema (inspired by Railway's [Railpack](https://railpack.io/)) that declares: hardware class, base OS/libc, Python version, RAM/CPU caps, storage type. Any platform — Railway, Vercel, AWS, local Mac, QEMU — can interpret it and run identically. Results normalize to a common format.

See [`stackconfig.schema.json`](./stackconfig.schema.json) and [`schema_examples/`](./schema_examples/) for the full spec and example configs.

### Day-1 Stack Matrix

| Dimension | Options |
|:---|:---|
| **Hardware class** | Edge (RPi5 8 GB ARM) · Workstation (M4 64 GB) · Cloud-x86 · Cloud-ARM (Graviton) |
| **Distro / libc** | Ubuntu 24.04 (glibc) · Alpine 3.20 (musl) · Fedora 41 |
| **Python** | 3.11 · 3.12 · 3.13 |
| **RAM cap** | 2 GB · 16 GB · 64 GB |

> **4 × 3 × 3 × 3 = 108 stack combinations per model.**
> Parallelizable. Under $50 on spot instances + PaaS.

### Where Agents Break

```
                    Ubuntu/glibc    Alpine/musl    Fedora
                    ────────────    ───────────    ──────
Wheel install       ✅ works        🔴 missing     ✅ works
fork vs spawn       ✅ fork ok      ⚠️  musl edge   ✅ fork ok
Case-sensitive FS   ✅ ext4 yes     ✅ ext4 yes     ✅ ext4 yes
asyncpg compile     ✅ prebuilt     🔴 needs gcc   ✅ prebuilt
pandas 2 GB cap     ⚠️  tight       🔴 OOM         ⚠️  tight
GPU detection       ⚠️  CUDA only   🔴 no driver   ⚠️  CUDA only

✅ = likely succeeds    ⚠️  = depends on agent    🔴 = fails without adaptation
```

### Task Design

50 deliberately underspecified tasks across five categories:

| Category | Example Stress Vector |
|:---|:---|
| **Data-heavy processing** | 10 GB JSON → Parquet under RAM cap |
| **Web service deployment** | FastAPI + asyncpg + pinned deps |
| **CLI / embedded tooling** | Concurrent file processor, case-sensitive FS |
| **AI-adjacent** | Local inference wrapper with optional accelerator |
| **Dependency hell** | musl wheels, sdist compilation, platform tags |

Tasks do not hint at the adaptation strategy. The agent must probe the stack and figure it out.

---

## Metrics

**Primary composite:**

```
Composite = Success × (1 / Time_norm) × (1 / Resource_norm)

  Time_norm     = wall_clock / wall_clock_easiest_stack
  Resource_norm = peak_ram   / peak_ram_easiest_stack
```

Normalization is relative to the easiest stack. An agent that works everywhere beats one that is fast only on the easy stack.

**Secondary:** Adaptation steps (retries/probes before success), patch portability (does final code work on other stacks), provisioning time (clean image to first run).

**Energy:** Implied via published industry power models (CodeCarbon tables, RAPL/powermetrics references per hardware class). No physical meters required for core leaderboard. Optional calibrated subset for research.

```
Example scoring:

  Agent A: 100% on Mac, 60% on Alpine
    Mac:    1.0 × 1.0 × 1.0 = 1.00
    Alpine: 0.6 × 0.14 × 0.18 = 0.015
    Mean StackEval score: 0.51

  Agent B: 90% on Mac, 85% on Alpine
    Mac:    0.9 × 0.9 × 0.8 = 0.65
    Alpine: 0.85 × 0.5 × 0.4 = 0.17
    Mean StackEval score: 0.41

  Leaderboard shows BOTH per-stack and aggregate.
  "Works everywhere at 85%" > "Perfect on easy, crashes on hard."
```

---

## Benchmark Gaming Is a Feature

If an agent ships a Dockerfile that runs the task 10× faster than another's on the same stack — and still passes semantic checks — it wins. That is engineering. StackEval rewards it.

---

## Execution Protocol

```mermaid
sequenceDiagram
    participant Dev as Submitter
    participant SC as StackConfig.json
    participant Rig as StackEval Rig
    participant Env as Platform
    participant LB as Leaderboard

    Dev->>SC: Define stack
    Dev->>Rig: stackeval run --config edge-alpine.json --model X
    Rig->>Env: Provision clean image per StackConfig
    Rig->>Env: Enforce resource caps
    loop 50 tasks
        Rig->>Env: Reset image, inject task
        Env->>Env: Agent executes (max 30 turns, 10 min)
        Env-->>Rig: Results + metrics + signed artifacts
    end
    Rig->>LB: Submit results + config + artifacts
    LB->>LB: Human review gate
    LB-->>Dev: Published
```

Pure CLI interface. Max 30 agent turns per task. Clean-image reset between tasks. 10-minute timeout. Open-source StackEval Rig runs locally or on any vendor. Signed artifacts + exact StackConfig.json published with every result. Stack version drift tracked and disclosed — contamination is a feature, not a bug.

---

## What We Need to Prove First

Before public launch, a pilot across **3 frontier models × 3 stacks** must demonstrate that the variance StackEval reveals is large enough to matter and not already captured by existing benchmarks. If the delta is trivial, the benchmark is not worth shipping. We expect it will not be trivial.

Public human baseline (n=20 engineers, same tasks, same stacks) published alongside pilot results.

---

## Path to Adoption

Socialized leaderboard with vetted submissions. Anyone can run the matrix and submit results using a compliant StackConfig.json and published artifacts.

**Leaderboard categories:** Overall · Edge-only · Cost-optimized · Energy-aware

**Day-1 targets:** Railway · Vercel · Cursor · OpenRouter vendors · frontier model labs

Lightweight open governance on GitHub.

---

## What StackEval Is Not

It is not a replacement for model-capacity benchmarks. SWE-bench measures whether an agent can solve hard problems. StackEval measures whether that capability survives contact with the real world. They are complementary. Right now, only one of them exists.

---

## Repo Structure

```
stackconfig.schema.json          # JSON Schema for StackConfig v0.1
schema_examples/
  edge-alpine-arm-2gb.json       # Example: Railway-class container
  workstation-m4-64gb.json       # Example: Mac Mini M4
  cloud-graviton-ubuntu-8gb.json # Example: AWS Graviton
```

---

## License

MIT
