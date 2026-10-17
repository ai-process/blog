---
layout: post
title: "Why We Chose Go Over Python for Our LLM Gateway Infrastructure"
date: 2026-10-17 12:00:00 +0000
reading_time: 7
tags:
  - architecture
  - performance
  - golang
  - llm-proxy
  - tokenmeter
  - devops
description: "An architectural analysis of memory footprint, sub-60ms readiness, scale-to-zero economics, and why compiled Go binaries use a fraction of the resources of Python gateways like LiteLLM in high-concurrency production deployments."
---

Python is the primary language of machine learning. From research notebooks in PyTorch to high-level framework wrappers like LangChain and LlamaIndex, the modern AI ecosystem was built on Python.

When developers build their first prototype against OpenAI or Anthropic APIs, Python is the natural choice. But when an engineering team transitions from prototyping to running central platform infrastructure—a gateway that routes millions of LLM requests across microservices, multiplexes thousands of concurrent connections, enforces spend budgets, and records token telemetry—the requirements shift from model experimentation to network systems engineering.

In this article, we look at why we built our open-source AI infrastructure stack—[**llm-proxy**](https://github.com/ai-process/llm-proxy) and [**light-tokenmeter**](https://github.com/ai-process/light-tokenmeter)—in **Go** instead of Python, comparing resource footprint, concurrency models, and operational trade-offs against popular Python solutions like LiteLLM Proxy.

---

## 1. The Anatomy of an LLM Gateway Workload

An LLM gateway operates under different constraints than an inference server or a training job:

* **I/O Bound Network Multiplexing:** The gateway does not run neural networks. Its responsibilities are connection pooling, header transformation, protocol conversion (gRPC to HTTP/REST), rate limiting, and fallback routing.
* **Long-Lived Streaming Connections:** Generating a completion over Server-Sent Events (SSE) or gRPC streams often takes 5 to 30 seconds. At high traffic volumes, a gateway must sustain 5,000 to 30,000 concurrent open TCP connections without exhausting file descriptors or thread memory.
* **Low Gateway Latency:** Because upstream model latency is high (hundreds of milliseconds to seconds), the gateway itself must add minimal overhead—ideally single-digit milliseconds for routing, limits, and telemetry dispatch.
* **Density and Cloud Cost:** Gateways run continuously across staging, preview, and production clusters. In Kubernetes clusters and serverless environments, operational memory overhead translates directly into cloud hosting costs.

In a Python-based gateway (such as LiteLLM running on FastAPI and Uvicorn), handling concurrent traffic across a multi-core machine requires running multiple operating system worker processes to bypass the Global Interpreter Lock (GIL). Each worker process imports heavy vendor SDKs (`openai`, `boto3`, `google-genai`), database ORMs (such as Prisma), and serialization frameworks (`pydantic`), consuming over 100 MiB of baseline memory per worker process before serving any traffic.

---

## 2. Resource Footprint: The Benchmark Numbers

To evaluate the efficiency differences between compiled Go binaries and Python runtimes, we tested `llm-proxy` and `light-tokenmeter` against LiteLLM Proxy in a synthetic benchmark.

### Test Environment & Methodology

* **Hardware & OS:** Linux x86_64, 4 vCPU, 8 GiB RAM, Docker/containerd.
* **Software Versions:** `llm-proxy` (Go 1.26, commit `6882351`), `light-tokenmeter` (Go 1.26, commit `63ce070`), LiteLLM Proxy (`ghcr.io/berriai/litellm:main-latest` as of October 2026, 4 Uvicorn workers).
* **Workload:** Synthetic `k6` benchmark running at 1,000 requests per second against a local mock upstream (10 ms fixed response latency). This isolates pure gateway routing and multiplexing overhead from upstream provider variability. *(Note: This test isolates routing overhead under short requests; memory under thousands of open concurrent streaming connections depends on network socket buffers and was not directly measured here.)*
* **Telemetry:** Memory was recorded via Linux cgroup `memory.current` (capturing RSS, page cache, and kernel buffers).

### Comparison Table

LiteLLM combines routing and spend tracking into a single container. In the `ai-process` architecture, routing (`llm-proxy`) and token accounting (`light-tokenmeter`) are split into two independent services. Both are shown below, along with the combined footprint.

| Metric | `llm-proxy` (Go) | `light-tokenmeter` (Go) | LiteLLM Proxy (Python) | Difference (`llm-proxy` vs LiteLLM) |
|---|---|---|---|---|
| **Docker image size (uncompressed)** | ~64 MB (~18 MB pull, slim) / ~172 MB (~55 MB pull, audio) | ~25–26 MB (~8 MB pull, distroless) | ~2.2 GB (~800 MB pull) | ~92–97% smaller disk footprint |
| **Idle memory (`memory.current`)** | ~25–35 MiB | ~18–25 MiB | ~400–600 MiB (4 workers) | ~11–24x less idle memory |
| **Active load memory (1k RPS)** | ~45–80 MiB | ~35–50 MiB | ~1.2–2.0 GiB (4 workers) | ~15–46x less memory under load |
| **Time to readiness (`/readyz` 200)** | ~30–60 ms (warm local DB, migrations applied) | ~20–40 ms (warm local DB, migrations applied) | 8–15 s (`/health/readiness`) | ~130–500x faster readiness |
| **Concurrency model** | Goroutines across all cores | Goroutines across all cores | Multiple OS processes + asyncio | Single process; no per-worker duplication |
| **Recommended container limit** | 256–512 MiB | 128–256 MiB | ~2.5–3.0 GiB | ~5–12x lower memory reservation |

Even combining `llm-proxy` and `light-tokenmeter` on the same host (~80–130 MiB total memory under 1,000 RPS), the dual-service Go stack consumes roughly 9–25x less memory than a 4-worker Python gateway.

---

## 3. Concurrency and Memory Architecture

Why does Python require 1.2–2.0 GiB under load, while Go stays below 80 MiB?

### The Multi-Process Memory Penalty

In Python, taking advantage of 4 CPU cores requires 4 separate OS worker processes under Uvicorn or Gunicorn. Because each process runs an independent Python interpreter:

1. **Duplicate Library Overhead:** The standard library, third-party SDKs, and data models reside in memory 4 times.
2. **Duplicate Connection Pools:** Database connection pools and HTTP keep-alive pools are maintained separately per worker process.
3. **Memory Allocator Behavior:** Python's small-object allocator (`pymalloc`) often retains freed memory in arenas rather than returning pages to the operating system immediately, compounding memory usage across long-running worker processes.

### Go's Single-Process Concurrency

In contrast, `llm-proxy` runs as a **single operating system process**. Go's runtime scheduler distributes goroutines across available CPU threads:

* **Minimal Initial Goroutine Overhead:** Goroutines start with ~2 KB of stack space and expand dynamically. The Go runtime handles thousands of active connections without dedicating an OS thread to each socket.
* **Shared Connection Pooling:** A single `pgxpool` for PostgreSQL and a shared `net/http.Transport` connection pool serve all incoming requests, eliminating redundant socket allocations.
* **Memory Tuning with `GOMEMLIMIT`:** Go's runtime allows configuring a soft memory target via `GOMEMLIMIT`. By setting `GOMEMLIMIT=200MiB` on a 256 MiB container, Go's garbage collector runs more frequently when approaching the threshold, significantly reducing the likelihood of Out-Of-Memory (OOM) termination during unexpected traffic spikes.

---

## 4. The Scale-to-Zero and Serverless Advantage

In cloud architectures, scaling workloads down to zero replicas during idle periods (such as nights and weekends) is an effective way to control cloud spend. Frameworks like **Knative Serving**, **Google Cloud Run**, and Kubernetes **KEDA** automate this pattern.

However, scale-to-zero requires cold start latencies to be short enough that users do not notice timeouts.

### The Python Cold Start Challenge

When a Python gateway container starts from zero:
1. **Registry Pull & Extraction:** Downloading ~800 MB compressed and extracting ~2.2 GB of container layers takes several seconds on a typical cluster backplane.
2. **Module Imports & Initialization:** Python must locate, parse, and import vendor libraries (`openai`, `boto3`, `fastapi`, `pydantic`), and the ORM must initialize database connections. This phase typically takes 8 to 15 seconds before the HTTP server can mark its `/health/readiness` probe healthy.

If an interactive user sends a chat message and encounters a 10–15 second container startup delay, the application feels broken. Teams often respond by keeping minimum replica counts (`minReplicas: 1` or `2`) running 24/7, forfeiting scale-to-zero cost savings.

### Fast Readiness with Go

With `llm-proxy` and `light-tokenmeter`:
* **Fast Image Pulls:** At ~8 MB compressed for `light-tokenmeter` (distroless) and ~18 MB for `llm-proxy` (slim), container downloads complete in approximately one second on modern network backplanes.
* **Sub-60ms In-Process Readiness:** Compiled Go binaries have no module import or byte-compilation phase. Once the binary starts, it parses its environment, verifies the local database connection, and marks `/readyz` healthy within 30 to 60 milliseconds (warm local DB, migrations applied).

While end-to-end serverless cold start also includes cloud hypervisor scheduling and network attachment, eliminating application-level startup latency makes scale-to-zero practical for real-world traffic.

---

## 5. Security & Container Attack Surface

Infrastructure gateways store sensitive credentials: upstream provider API keys (OpenAI, Anthropic, Gemini, DeepSeek) and internal admin keys. A compromised gateway puts API budgets and downstream services at risk.

The official LiteLLM Docker image ships with a full Python runtime environment, pip dependencies, and system utilities. In enterprise environments, container scanners (Trivy, Grype) frequently flag transitive CVEs in system libraries that the proxy itself never uses.

In our stack:
* **`light-tokenmeter`** runs on Google's `gcr.io/distroless/static-debian12:nonroot`. The image contains **only** the compiled static binary, CA certificates, and timezone data. There is no package manager, no shell, and no operating system utilities. The process runs under UID `65532:65532`. Even if an arbitrary code execution flaw were discovered, an attacker cannot spawn a shell or download tools using `curl` or `wget`.
* **`llm-proxy`** runs as an unprivileged user (`appuser`, UID `10001`) on Alpine. We use Alpine for `llm-proxy` to allow installing optional system packages like `ffmpeg` on the audio build (`coyl/llm-proxy:audio`) for Gemini speech transcoding, while keeping the default slim image at ~64 MB. While the Alpine image still includes BusyBox `/bin/sh` and `apk` (moving `llm-proxy` to pure distroless is on our roadmap), `light-tokenmeter` already achieves a zero-shell distroless posture.

---

## 6. Trade-Offs We Accepted

Choosing Go for an LLM gateway is not without trade-offs. Python has distinct advantages in the AI ecosystem that teams should weigh:

1. **Ecosystem & Day-0 Provider Support:** When model vendors launch new APIs or features, they release official Python SDKs first. A Python gateway can often wrap new vendor SDKs in hours. With Go, writing native adapters requires inspecting raw HTTP/gRPC contracts and implementing typed request/response structs by hand.
2. **Dynamic Scripting & Community Extensions:** Python allows developers to write custom middleware, inline prompts, or dynamic Python evaluation rules with minimal boilerplate. Go requires static compilation, type safety, and structured configurations.
3. **Code Verbosity:** Go's explicit error handling (`if err != nil`) and static typing require more lines of code compared to Python's concise dynamic syntax.

For teams building internal rapid prototypes or experimenting with dozens of experimental LLM libraries, a Python gateway like LiteLLM offers rapid onboarding. For platform teams running shared production gateways where high throughput, memory density, scale-to-zero latency, and predictable operations matter most, Go is a better fit.

---

## 7. Summary

| Dimension | Python Gateways (e.g. LiteLLM) | Go Gateways (ai-process Stack) |
|---|---|---|
| **Primary focus** | Fast prototyping, Day-0 SDK feature wrappers | High-throughput production routing, low-cost operations |
| **Container image (uncompressed)** | ~2.2 GB (Python runtime, pip deps) | ~25–64 MB (tokenmeter / slim; ~172 MB audio) |
| **Container image (registry pull)** | ~800 MB compressed | ~8–18 MB compressed (tokenmeter / slim) |
| **Active memory (1k RPS, cgroup)** | ~1.2–2.0 GiB (4 workers) | ~45–80 MiB (`llm-proxy`) / ~35–50 MiB (`tokenmeter`) |
| **Container memory limit** | ~2.5–3.0 GiB | 256–512 MiB (`llm-proxy`) / 128–256 MiB (`tokenmeter`) |
| **In-process readiness latency** | 8–15 seconds | 20–60 milliseconds |
| **Runtime environment** | Multi-process Python runtime | Single-process Go binary (goroutines) |

By reducing memory footprint by 15–46x and image sizes by over 90%, compiled Go gateways make self-hosting LLM infrastructure practical on standard, cost-effective cloud instances.

---

## What's Next?

In upcoming posts, we will explore:
* Configuring multi-vendor fallback chains across OpenAI, Anthropic, and Gemini.
* High-throughput usage accounting with `light-tokenmeter` and ClickHouse OLAP storage.
* Protecting upstream LLM credentials using AES-256-GCM envelope encryption.

Explore the repositories on GitHub:
* [**llm-proxy**](https://github.com/ai-process/llm-proxy) — High-performance self-hosted LLM gateway.
* [**light-tokenmeter**](https://github.com/ai-process/light-tokenmeter) — Lightweight token and spend accounting service.
* [**llm-proxy-admin**](https://github.com/ai-process/llm-proxy-admin) — Web console for models, rules, and keys.
