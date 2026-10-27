---
layout: post
title: "Go vs Python for an LLM Gateway: What We Measured"
date: 2026-10-26 07:00:00 +0000
reading_time: 10
tags:
  - architecture
  - performance
  - golang
  - ktms
  - benchmarks
description: "A reproducible benchmark of KTMS (Go) against LiteLLM Proxy (Python) and eight other self-hosted gateways on identical cloud hardware: added latency, time to first token, throughput, memory, CPU and startup time."
---

Python is the language of machine learning, so most teams reach for a Python gateway first. We built
[**Keystone. The Model Switchboard. (KTMS)**](https://github.com/ai-process/ktms), our self-hosted
LLM gateway, in Go instead. A
gateway doesn't run models. It moves bytes between your services and model vendors, and that is a
networking problem.

"Go is lighter" is easy to say and hard to check, so we built a reproducible benchmark stand and ran
KTMS against LiteLLM Proxy on identical hardware, then against eight more self-hosted gateways. This
post walks through what we measured, the bug the stand found in our own code, and where the numbers
stop.

---

## 1. What a Gateway Actually Does

An LLM gateway sits on the request path of every model call:

* **It is I/O-bound.** It authenticates, routes, enforces limits, converts protocols (gRPC to
  HTTP), forwards the request, and streams the answer back. None of that is compute-heavy.
* **It holds connections open.** Streaming responses keep a connection busy for as long as the model
  generates, so concurrency matters more than raw request rate.
* **It must stay out of the way.** Model latency is measured in hundreds of milliseconds or more, so
  whatever the gateway adds should be lost in the noise.
* **It runs everywhere, all the time.** Staging, preview and production all run a copy, so its idle
  footprint is paid many times over.

Python gateways such as LiteLLM run on FastAPI and Uvicorn. To use more than one core, they start
several worker processes, because of the Global Interpreter Lock. Each worker loads its own copy of
the interpreter, the vendor SDKs and the data models. A Go service is one process: the runtime
schedules goroutines across all cores and shares one set of connection pools.

---

## 2. The Benchmark Stand

The whole stand is open source in [`ktms/bench`](https://github.com/ai-process/ktms/tree/main/bench)
and reruns with one command. Terraform creates an isolated network and two VMs, the script runs the
suite and then destroys everything.

```
 runner VM                     gateway VM (private network)
 ┌──────────┐   HTTP    ┌────────────────────────────────────────┐
 │   k6     │ ────────▶ │ KTMS ──────┐                           │
 │          │           │ LiteLLM ───┼──▶ mock OpenAI upstream    │
 └──────────┘           │ (direct) ──┘                           │
                        └────────────────────────────────────────┘
```

* **Hardware:** Google Cloud `c3-standard-4` (4 vCPU, 16 GB, x86_64) for the gateway, and a second,
  identical VM for the load generator, so k6 never competes with the gateway for CPU.
* **Versions:** KTMS built from source with the fix described in section 4; LiteLLM `v1.106.0` (released before the
  compromised 1.107.0, which we never ran) with 4 Uvicorn workers (one per core), master key only
  and no database, its leanest mode; k6 2.3.0.
* **Upstream:** a small Go mock of the OpenAI Chat Completions API with fixed responses and
  configurable delays. Real vendors would add hundreds of milliseconds of noise and make runs
  incomparable. The mock counts the requests it receives, which proves that no answer came from a
  gateway cache.
* **Baseline:** every scenario also runs with no gateway at all ("direct"), so "added latency" means
  gateway minus direct.
* **One gateway at a time:** the others are stopped before each run.
* **Memory:** the cgroup RSS of the gateway container only. KTMS also needs Postgres and Redis, which
  are **not** counted. LiteLLM in this mode needs neither.

We ran the full suite twice. Latency, memory, CPU and maximum throughput agreed within about 10%
between the runs for every gateway; LiteLLM's p99 was the exception. The charts show the mean of both.

---

## 3. Results

### Added Latency

Non-streaming requests at a steady 200 RPS against an upstream that answers instantly.

![Latency added by the gateway: KTMS 0.64 ms at p50 and 1.07 ms at p99, LiteLLM 8.5 ms at p50](/assets/posts/go-vs-python-llm-gateway/latency-overhead.svg)

KTMS adds about 0.6 ms at the median and about 1.1 ms at p99. LiteLLM adds about 8.5 ms at the median,
and its p99 swung from 27 ms in one run to 229 ms in the other, so the mean in the chart is only a
rough guide.

### Time to First Token

50 concurrent streams, each with 20 chunks sent 15 ms apart.

![Time to first token: direct 0.16 ms, KTMS 0.89 ms, LiteLLM 161 ms at p50](/assets/posts/go-vs-python-llm-gateway/ttft.svg)

Both gateways delivered every chunk of every stream. KTMS forwards the first token about 0.7 ms after
it arrives from the upstream. LiteLLM takes about 160 ms. It doesn't merge chunks: the whole stream
just starts later, and finishes the same ~160 ms later.

### Throughput

The offered rate climbs from 250 to 4,000 RPS in steps of 20 seconds, with a 10 ms upstream. A step
counts as sustained when at least 95% of the offered rate gets through with fewer than 1% errors.

![Throughput: KTMS follows the offered rate up to 4,000 RPS; LiteLLM plateaus near 570 RPS with about half of the requests failing](/assets/posts/go-vs-python-llm-gateway/throughput.svg)

KTMS kept up with every step, including 4,000 RPS, the highest we tested. Its p99 stayed at 13 ms
up to 2,000 RPS (direct: 11 ms) and grew to 26–43 ms at 4,000 RPS. LiteLLM sustained 250 RPS. At
500 RPS a third of its requests already failed, and from 1,000 RPS up it plateaued at about 570 RPS
while half of the requests hit the 5-second client timeout.

### Memory

![Gateway memory: KTMS 6.9 MiB idle and 20 MiB under load, LiteLLM 2.5 GiB and 2.9 GiB](/assets/posts/go-vs-python-llm-gateway/memory.svg)

At idle, KTMS uses about 7 MiB. LiteLLM uses about 2.5 GiB with 4 workers, roughly 0.6 GiB per
worker. During a 5-minute soak at 200 RPS, KTMS stayed at about 20 MiB and LiteLLM at about 2.9 GiB,
and neither drifted upward. KTMS peaked at 41–46 MiB over the whole run, including 4,000 RPS, well
under the 512 MiB limit in our Helm chart.

The chart uses a linear scale on purpose. The KTMS bars are hard to see, and that is the result.

### CPU

![CPU time per 1,000 requests: KTMS 0.63 s, LiteLLM 9.7 s](/assets/posts/go-vs-python-llm-gateway/cpu.svg)

Over the soak, KTMS spent about 0.63 s of CPU time per 1,000 requests. LiteLLM spent about 9.7 s,
fifteen times more. This is the number that decides how many cores a given traffic level needs.

### Startup

![Time to readiness: KTMS 0.17 s, LiteLLM 32 s](/assets/posts/go-vs-python-llm-gateway/readiness.svg)

From `docker start` to the first request answered through the gateway, KTMS took about 170 ms and
LiteLLM about 32 s. That time includes starting the container itself, and the KTMS database was
already migrated.

The images differ just as much. For linux/amd64, the KTMS slim image is about 21 MB to pull and
about 67 MB unpacked. The LiteLLM image is about 420 MB to pull and about 1.3 GB unpacked.

---

## 4. The Benchmark Found a Bug in KTMS

The first full run did not look like the charts above. KTMS was fine up to 2,000 RPS, then failed
96% of requests at 3,000 RPS, each in under a millisecond. The log said
`connect: cannot assign requested address`: the container had run out of outbound ports.

The cause was one missing line of configuration. The OpenAI, Anthropic and Gemini SDKs fell back to
Go's default HTTP transport, which keeps only two idle connections per host. Every concurrent
request beyond two opened a new connection to the upstream, and the closed ones piled up in
TIME_WAIT until no ports were left. All vendor clients now share one transport that keeps up to
1,024 idle connections per host. The same run then sustained 4,000 RPS without errors, and p99 at
2,000 RPS dropped from 50–60 ms to 13 ms.

Every Go service that talks to a single upstream through an SDK can hit this. Check what your HTTP
client does with `MaxIdleConnsPerHost`.

---

## 5. What About Other Gateways?

Comparing Go with Python only tells half of the story, so we ran the same suite against eight more
self-hosted gateways: Bifrost (Go), Portkey (TypeScript), Kong AI Gateway and Apache APISIX
(Nginx/Lua), Higress and Envoy AI Gateway (Envoy), agentgateway (Rust) and New API (Go). Plano
(Rust/Envoy) could not hold the 200 RPS scenarios in our setup and is left out.

![Latency added at p50: agentgateway 0.24 ms, APISIX 0.28 ms, Kong 0.35 ms, Bifrost 0.63 ms, KTMS 0.64 ms, LiteLLM 8.5 ms](/assets/posts/go-vs-python-llm-gateway/all-latency.svg)

![Memory at 200 RPS: agentgateway 13 MiB, KTMS 20 MiB, LiteLLM 2.9 GiB](/assets/posts/go-vs-python-llm-gateway/all-memory.svg)

![CPU per 1,000 requests: agentgateway 0.27 s, APISIX 0.29 s, Kong 0.40 s, KTMS 0.63 s, LiteLLM 9.7 s](/assets/posts/go-vs-python-llm-gateway/all-cpu.svg)

The results split into two leagues:

* **Compiled and Nginx/Envoy-based proxies** add 0.25–1.3 ms at p50 and all sustained 4,000 RPS.
  KTMS is in this league.
* **Python and Node.js gateways** fall behind: Portkey, which ships as a single Node.js process,
  saturated at 500 RPS, and LiteLLM at 250 RPS. New API (Go) sits in between at 2,000 RPS.

KTMS is not the fastest proxy we measured. agentgateway, APISIX and Kong add 0.25–0.35 ms where
KTMS adds 0.64 ms, and spend less CPU per request. They also do less work per request in this setup
(a static key check, against KTMS's key lookup, rule routing and limit checks), but we have not
profiled what each step costs, so that is the next thing we will look at. On memory, only
agentgateway is as frugal as KTMS; the other compiled proxies need 47–530 MiB.

---

---

## 6. Why the Gap With Python Is So Large

**One process instead of four.** LiteLLM needs a worker per core, and each worker carries its own
interpreter, imported SDKs, data models and connection pools. That is where most of the 2.5 GiB goes
before the first request arrives. KTMS is a single process. Goroutines start with a small stack that
grows on demand, and one PostgreSQL pool and one HTTP transport serve every request.

**No interpreter on the hot path.** Each request in a Python gateway passes through several layers of
interpreted middleware, validation and logging. In Go, the same work is compiled code with no
interpreter dispatch. It shows up twice: as added latency, and as CPU time per request.

**No import phase at startup.** A Python service must import its dependency tree before it can
answer. A Go binary just parses its configuration and connects to its database.

Go is not magic, though: the Rust and Nginx proxies above show that a compiled language is necessary
for this class of performance, not sufficient. What matters is what runs per request.

**Predictable memory limits.** Go's `GOMEMLIMIT` makes the garbage collector work harder as the
process approaches a soft limit, which helps a container ride out traffic spikes without being
OOM-killed.

---

## 7. What We Did Not Measure

These numbers describe this stand, not every deployment:

* **The upstream is a mock.** Real vendors add TLS, network round trips and hundreds of milliseconds
  of generation time, which dwarf either gateway's overhead at low traffic. The differences matter
  most for throughput, memory and cost.
* **We stopped at 4,000 RPS.** The real ceilings of KTMS and the other compiled proxies on 4 vCPU are
  still unknown.
* **Every gateway ran with one minimal configuration,** routing to the mock with retries, caching and
  logs off where possible. LiteLLM ran standalone with no database and no Redis. Other configurations
  will give different numbers, and we'd welcome pull requests that tune any of them.
* **Postgres and Redis are excluded from memory.** KTMS and New API need both.
* **One machine shape, one workload:** short chat completions and streams. We did not test thousands
  of long-lived streams or large payloads.
* **[light-tokenmeter](https://github.com/ai-process/light-tokenmeter)** was not part of this run.

---

## 8. Trade-Offs We Accepted

Go isn't free:

1. **Day-0 provider support.** Vendors ship Python SDKs first. A Python gateway can wrap a new API in
   hours; in Go, we write typed adapters against the raw HTTP contract.
2. **Extensibility.** Python makes it easy to drop in custom middleware or scripting. Go means
   compiled code and structured configuration.
3. **Verbosity.** Explicit error handling and static types mean more lines of code.

For prototypes, or for teams that need the long tail of niche providers, LiteLLM is a reasonable
choice. For a shared production gateway, where latency, density, CPU cost and startup matter, the
measurements favour a compiled gateway. Among those, pick by features: KTMS's are a gRPC API, effort
routing, budgets and a pool of local GPU nodes under the Apache 2.0 licence.

---

## 9. Summary

| Metric (4 vCPU, mock upstream) | KTMS (Go) | LiteLLM (Python, 4 workers) |
|---|---|---|
| Added latency, p50 / p99 | ~0.6 / ~1.1 ms | ~8.5 / 27–229 ms |
| Time to first token, added (p50) | ~0.7 ms | ~160 ms |
| Max sustained throughput | ≥ 4,000 RPS (highest tested) | ~250 RPS |
| Memory, idle / 200 RPS | ~7 MiB / ~20 MiB | ~2.5 GiB / ~2.9 GiB |
| CPU time per 1,000 requests | ~0.63 s | ~9.7 s |
| Time to readiness | ~170 ms | ~32 s |
| Image (pull / unpacked, amd64) | ~21 MB / ~67 MB | ~420 MB / ~1.3 GB |

Every number above comes from two runs of a public stand. Clone
[`ktms/bench`](https://github.com/ai-process/ktms/tree/main/bench), run it in your own cloud
account, and tell us what you get.

---

## What's Next?

Upcoming posts:
* Profiling the KTMS request path: closing the gap to agentgateway and APISIX.
* Configuring multi-vendor fallback chains across OpenAI, Anthropic and Gemini.
* Usage accounting with `light-tokenmeter` and ClickHouse.

Explore the repositories on GitHub:
* [**KTMS**](https://github.com/ai-process/ktms): self-hosted LLM gateway.
* [**light-tokenmeter**](https://github.com/ai-process/light-tokenmeter): token and spend accounting.
* [**ktms-admin**](https://github.com/ai-process/ktms-admin): web console for models, rules and keys.
