---
layout: post
title: "Why We Chose Go Over Python for Our LLM Gateway Infrastructure"
date: 2026-10-17 16:00:00 +0000
reading_time: 6
tags:
  - architecture
  - performance
  - golang
  - llm-proxy
  - tokenmeter
  - devops
description: "An in-depth look at memory footprint, sub-60ms readiness, scale-to-zero economics, and why compiled Go binaries outperform Python gateways like LiteLLM in high-concurrency production deployments."
---

Python is the undisputed lingua franca of machine learning. From research notebooks in PyTorch to high-level framework wrappers like LangChain and LlamaIndex, the AI ecosystem was built on Python.

When developers build their first prototype against OpenAI or Anthropic APIs, Python is the natural first choice. But when an engineering team transitions from prototyping to running central platform infrastructure—a unified gateway that routes millions of LLM requests across multiple microservices, handles thousands of concurrent streaming connections, manages cryptographic keys, and meters token budgets—the engineering requirements shift from data science to network systems programming.

In this deep dive, we explore why we built our open-source AI infrastructure stack—[**llm-proxy**](https://github.com/ai-process/llm-proxy) and [**light-tokenmeter**](https://github.com/ai-process/light-tokenmeter)—in **Go** instead of Python, comparing its resource footprint, concurrency model, and serverless economics with popular Python solutions like LiteLLM Proxy.

---

## 1. The Anatomy of an LLM Gateway Workload

An LLM gateway operates under vastly different constraints than an inference engine or a model training script:

* **I/O Bound, Not Compute Bound:** The gateway does not compute tensor multiplications. Its responsibilities are network multiplexing, header transformation, protocol conversion (gRPC to HTTP/REST), rate limiting, and request routing.
* **Long-Lived Streaming Connections:** Generating a 1,000-token completion over Server-Sent Events (SSE) or gRPC streams often takes between 5 to 30 seconds. At high traffic volumes (e.g. 1,000 requests per second), the gateway must sustain tens of thousands of concurrent open TCP connections.
* **Low Gateway Latency Overhead:** Because upstream model latency is inherently high, the gateway itself must add negligible overhead (sub-millisecond routing and queuing).
* **Strict Cost and Density Constraints:** Infrastructure proxies run 24/7 across multiple environments (production, staging, developer preview clusters). In high-density Kubernetes clusters or serverless environments, operational memory overhead translates directly into cloud hosting costs.

In a Python-based gateway (such as LiteLLM running on FastAPI and Uvicorn), handling this workload requires multiple operating system worker processes to utilize multi-core machines, due to the Global Interpreter Lock (GIL). Each worker process imports heavy vendor SDKs (`openai`, `boto3`, `google-genai`), database ORMs (such as Prisma), and serialization frameworks (`pydantic`), replicating hundreds of megabytes of baseline memory per process.

---

## 2. Resource Footprint: The Benchmark Numbers

To quantify the efficiency differences between compiled Go binaries and Python runtimes, we evaluated `llm-proxy` and `light-tokenmeter` against LiteLLM Proxy in a synthetic benchmark scenario.

### Test Environment & Methodology

* **Environment:** Linux x86_64, 4 vCPU, 8 GiB RAM, Docker/containerd.
* **Software Versions:** `llm-proxy` (Go 1.26), `light-tokenmeter` (Go 1.26), LiteLLM Proxy (`ghcr.io/berriai/litellm:main-latest`, 4 Uvicorn workers).
* **Workload:** Synthetic `k6` benchmark running at 1,000 requests per second against a 10 ms mock upstream (measuring pure gateway overhead without upstream model variance). Memory was tracked via Linux cgroup `memory.current`.

### Comparison Table

| Metric | `llm-proxy` (Go) | `light-tokenmeter` (Go) | LiteLLM Proxy (Python) | Difference (`llm-proxy` vs LiteLLM) |
|---|---|---|---|---|
| **Docker image size (uncompressed)** | ~64 MB (slim) / ~172 MB (audio) | ~25–26 MB (distroless) | ~2.2 GB | ~92–97% smaller disk footprint |
| **Idle memory (`memory.current`)** | ~25–35 MiB | ~18–25 MiB | ~400–600 MiB | ~11–24x less idle memory |
| **Active load memory (1k RPS)** | ~45–80 MiB | ~35–50 MiB | ~1.2–2.0 GiB (4 workers) | ~15–46x less memory under load |
| **Time to readiness (`/readyz` 200)** | ~30–60 ms (schema applied) | ~20–40 ms (schema applied) | 8–15 s (`/health/readiness`) | ~130–500x faster readiness |
| **Concurrency model** | Goroutines across all cores | Goroutines across all cores | Multiple OS processes + asyncio | Single process; no per-worker duplication |
| **Recommended container limit** | 256–512 MiB | 128–256 MiB | ~2.5–3 GiB | ~5–12x lower memory reservation |

---

## 3. Concurrency and Memory Architecture: Goroutines vs Multi-Process Workers

Why does Python require 1.5–2 GiB under moderate load, while Go stays comfortably below 80 MiB?

### The Multi-Process Memory Penalty

In Python, achieving high throughput across 4 CPU cores necessitates spawning 4 separate OS worker processes under Uvicorn or Gunicorn. Because each process runs an independent Python interpreter:

1. **Duplicate Module Memory:** The standard library, third-party libraries, and internal caches are loaded into memory 4 times.
2. **Duplicate Connection Pools:** Database connection pools and HTTP keep-alive connection pools are maintained separately per process.
3. **Coarse Garbage Collection:** Python's reference counting and cyclic garbage collector operates independently per process, leading to asynchronous memory spikes.

### Go's Lightweight Runtime

In contrast, `llm-proxy` executes as a **single operating system process**. Go's runtime scheduler multiplexes tens of thousands of goroutines across OS threads:

* **Tiny Goroutine Stacks:** Each goroutine starts with only ~2 KB of stack space, growing dynamically only when needed. Holding 10,000 active streaming requests requires only ~20–30 MiB of stack memory.
* **Unified Connection Pooling:** A single `pgxpool` for PostgreSQL and a single `fasthttp` / `net/http` transport pool serve all incoming requests, eliminating redundant socket allocations.
* **Precise Memory Limits with `GOMEMLIMIT`:** Go's runtime respects cgroup boundaries. By configuring `GOMEMLIMIT=200MiB` on a 256 MiB container, Go dynamically tunes GC frequency during traffic bursts, ensuring zero Out-Of-Memory (OOM) terminations without over-provisioning.

---

## 4. The Scale-to-Zero and Serverless Advantage

In modern cloud architectures, autoscaling workloads down to zero replicas during quiet periods (e.g. nights and weekends) is one of the most effective ways to reduce cloud expenditure. Frameworks like **Knative Serving**, **Google Cloud Run**, and Kubernetes **KEDA** make scale-to-zero accessible.

However, scale-to-zero is only viable if cold start latencies are imperceptible to end users.

### The Python Cold Start Dilemma

When a Python gateway container spins up from zero:
1. **Registry Pull Time:** Downloading an uncompressed ~2.2 GB container image (or ~800 MB compressed) across the network takes several seconds, even on fast cluster backplanes.
2. **Import Latency:** Python must discover, read, and byte-compile hundreds of modules (`import openai`, `import boto3`, `import fastapi`, `import prisma`). This phase alone typically takes 8 to 15 seconds before the HTTP server can begin accepting traffic.

For real-time user-facing chat applications, adding a 10–15 second cold start delay to the initial user prompt is unacceptable. Teams are forced to keep minimum replica counts (`minReplicas: 1` or `2`) running 24/7, defeating the purpose of scale-to-zero.

### Instant Readiness with Go

With `llm-proxy` and `light-tokenmeter`:
* **Rapid Image Pulls:** At ~25 MB for `light-tokenmeter` (distroless) and ~64 MB for `llm-proxy`, container images download in hundreds of milliseconds.
* **Sub-60ms In-Process Readiness:** Go binaries have zero module loading overhead. The binary initializes configuration, sets up routing tables, connects to the database, and exposes healthy readiness endpoints (`/readyz`) within 30 to 60 milliseconds.

While end-to-end serverless cold start also includes cloud scheduling and networking overhead, eliminating container bootstrap delays makes scale-to-zero a practical reality without degrading user experience.

---

## 5. Security & Attack Surface: Minimal Distroless Runtimes

Infrastructure gateways hold sensitive assets—most notably vendor API keys (OpenAI, Anthropic, Gemini, DeepSeek) and internal authentication tokens. A compromised gateway exposes your entire AI budget and upstream credentials.

Python containers inherently ship with a complete operating system userland: Python interpreters, package managers (`pip`), shell environments (`/bin/sh`, `/bin/bash`), and dozens of system libraries. In enterprise vulnerability scans (CVE scanners like Trivy, Grype, or Snyk), large Python base images regularly flag dozens of transitive CVEs in system packages unrelated to the gateway itself.

In our stack:
* **`light-tokenmeter`** is packaged on Google's `gcr.io/distroless/static-debian12:nonroot`. The image contains **only** the static Go binary, SSL certificates, and timezone data. There is no package manager, no shell, and no operating system utilities. The process runs under UID `65532:65532`. Even in the event of an arbitrary code execution vulnerability, an attacker has no shell to spawn and no filesystem tools to leverage.
* **`llm-proxy`** is compiled into a lightweight Alpine container running under non-root user `appuser` (UID `10001`), keeping the attack surface minimal and verifiable.

---

## 6. Summary: Right Tool for the Right Job

| Dimension | Python Gateways (e.g. LiteLLM) | Go Gateways (ai-process Stack) |
|---|---|---|
| **Best suited for** | Rapid prototyping, Python-native SDK experimentation | High-throughput production routing, enterprise gateways |
| **Typical Container Image** | ~800 MB–2.2 GB (Debian / Python / pip) | ~25–64 MB (Distroless / Alpine) |
| **Memory per Node** | ~1.5–3.0 GiB (multi-process) | ~128–512 MiB (single process goroutines) |
| **Readiness Latency** | 8–15 seconds | 30–60 milliseconds |
| **Attack Surface** | Full OS userland & interpreter | Minimal static binary / distroless |

Choosing Go for `llm-proxy` and `light-tokenmeter` was not about language dogma; it was an architectural decision driven by operational economics. By slashing memory requirements by 15–40x and container sizes by over 90%, teams can deploy enterprise-grade AI gateways and token accounting sinks on a fraction of the infrastructure budget.

---

## What's Next?

In upcoming posts, we will explore:
* Configuring zero-downtime multi-vendor fallback chains across OpenAI, Anthropic, and Gemini.
* High-throughput usage accounting with `light-tokenmeter` and ClickHouse OLAP aggregations.
* Securing LLM credentials using AES-256-GCM envelope encryption and scoped API tokens.

Explore the repositories on GitHub:
* [**llm-proxy**](https://github.com/ai-process/llm-proxy) — High-performance self-hosted LLM gateway.
* [**light-tokenmeter**](https://github.com/ai-process/light-tokenmeter) — Lightweight token and spend accounting service.
* [**llm-proxy-admin**](https://github.com/ai-process/llm-proxy-admin) — Web console for models, rules, and keys.
