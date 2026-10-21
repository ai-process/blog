---
layout: default
title: About
---

<article class="post">
  <header class="post-header">
    <h1 class="post-title">About the KTMS Dev Blog</h1>
    <p class="post-subtitle">Our mission, cadence, and open-source tooling.</p>
  </header>

  <div class="post-content">
    <p>
      Welcome to the official engineering blog for <strong>KTMS</strong>, <strong>light-tokenmeter</strong>, and associated AI process infrastructure tools.
    </p>

    <h2>Why this blog?</h2>
    <p>
      Running LLMs reliably at scale requires moving beyond naive SDK wrappers. Teams face hard production hurdles: multi-provider failover, credential isolation, latency unpredictability, strict cost attribution, and runaway token consumption.
    </p>
    <p>
      We publish an educative post <strong>every 2 days</strong> (whenever there is news, releases, or architectural topics) focusing on:
    </p>
    <ul>
      <li><strong>Educative teardowns:</strong> Deep dives into resilience patterns, load balancing, caching, batch processing, and telemetry.</li>
      <li><strong>Concrete implementations:</strong> Practical guides demonstrating how to solve these challenges using open-source, self-hosted tools.</li>
      <li><strong>Zero-fluff updates:</strong> Release highlights and roadmap previews for our ecosystem.</li>
    </ul>

    <h2>Our Open Source Ecosystem</h2>
    <ul>
      <li>
        <a href="https://github.com/ai-process/ktms" target="_blank" rel="noopener"><strong>KTMS</strong></a>:
        A self-hosted gRPC and HTTP gateway. Provides AES-256 encrypted vendor keys, effort-based runtime routing, multi-provider fallback chains, token/request rate limits, and batch execution.
      </li>
      <li>
        <a href="https://github.com/ai-process/light-tokenmeter" target="_blank" rel="noopener"><strong>light-tokenmeter</strong></a>:
        A lightweight telemetry service (Postgres &amp; ClickHouse) answering <em>"who is spending your tokens, and are they worth it?"</em> with multi-dimensional filtering by user, project, model, and action.
      </li>
      <li>
        <a href="https://github.com/ai-process/ktms-admin" target="_blank" rel="noopener"><strong>ktms-admin</strong></a>:
        A web console to configure models, keys, and routing rules live without redeploying services.
      </li>
    </ul>
  </div>
</article>
