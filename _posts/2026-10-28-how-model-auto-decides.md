---
layout: post
title: "How model: \"auto\" decides: judge-based effort routing in 250 ms"
date: 2026-10-28 07:00:00 +0000
reading_time: 6
tags:
  - routing
  - ktms
  - judge
  - llm
description: "Inside Keystone's auto-routing: how a decision model rates a prompt's difficulty, why it gets 250 ms, what happens when it is slow, and how to keep control with pinned efforts."
---

![How hard is this prompt?](/assets/posts/how-model-auto-decides/cover.jpg)

**"What time is it?" and "prove that √2 is irrational" arrive in the same chat box, a minute
apart.** Send both to your best model and you overpay for the first. Send both to the cheap one and
the second gets a confident wrong answer.

In Keystone. The Model Switchboard. (KTMS), a service never names a model. It asks for an effort,
`LOW`, `MEDIUM` or `HIGH`, and routing rules map that to a chain of models. That works when the
caller knows how hard its task is. A summarizer does. A chat box where users ask anything does not.

For that case there is `model: "auto"`: a judge model rates the prompt in under 250 ms, and the
gateway routes on its answer. This post walks through what KTMS does with it, as of 1.3.

---

## The order of precedence

Before any judge is called, Keystone checks whether the caller already decided:

1. **An effort pinned in the model name**: `auto:low`, `auto:medium`, `auto:high`, or combined with
   a placement such as `auto:nodes-only:high`.
2. **The `reasoning_effort` field** of the OpenAI request (`low`, `medium`, `high`). Over gRPC, the
   same thing is `auto_route` with `effort_specified`.
3. **Only then a judge.**

So an existing client that already sends `reasoning_effort` keeps full control, and the judge only
fills the gap.

## What the judge is asked

The judge is a decision model, not a chat model. It doesn't write text; it answers typed questions
with probabilities. KTMS supports TypeSafe's Jev (`jev-latest`) and OpenAI Decisions
(`gpt-6-luna`) as judges.

KTMS builds the judge's input from the request instruction and the user messages, including text
parts of multimodal messages, trimmed to 4,000 characters. If guardrails masked personal data
earlier in the request, the judge sees the masked text. The state is one JSON field:

```json
{"prompt": "Explain the difference between MVCC in Postgres and Oracle"}
```

and the question is a single choice:

| Option | Description given to the judge |
|---|---|
| `low` | Simple, short, greeting, basic formatting, translation, or straightforward retrieval |
| `medium` | Standard instruction following, summary, general conversational chat, creative writing |
| `high` | Complex multi-step reasoning, mathematical proof or calculation, advanced programming/coding, deep technical analysis |

The judge returns the chosen option with its confidence. KTMS maps it to an effort and then routes
exactly as if the caller had sent that effort: the same rules, the same fallback chain.

## Why 250 ms, and what happens when it's slow

The judge call has a hard deadline of 250 ms. A decision model typically answers well within that,
and anything longer starts to eat into the user's time to first token, which is the whole point of
routing easy prompts to fast models.

When the judge misses the deadline, isn't configured, or fails, KTMS does not drop the request and
does not return an error. It routes at `MEDIUM` and marks the response as `auto-default`. An empty
prompt goes to `MEDIUM` without asking anyone.

The judge call itself goes through the normal machinery: it is routed by a rule for the `judge`
modality, respects the judge model's rate limit, and has its own circuit breaker, so a judge vendor
outage costs at most one deadline per request until the breaker opens, and then nothing.

## Seeing what happened

Every response says how it was routed:

```bash
curl -si localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer ktms_..." -H "Content-Type: application/json" \
  -d '{"model":"auto","messages":[{"role":"user","content":"Prove that sqrt(2) is irrational"}]}' \
  | grep -i '^x-ktms'
```

```text
X-KTMS-Route-Mode: auto-judge
X-KTMS-Judge-Score: 0.93
X-KTMS-Judge-Model: jev-latest
X-KTMS-Rule: high-default
X-KTMS-Vendor: anthropic
```

*(Example output: the score, rule and vendor depend on your models and rules.)*

`X-KTMS-Route-Mode` is one of `direct`, `override`, `auto-effort` (effort pinned), `auto-judge`
(the judge chose) and `auto-default` (the judge didn't answer, `MEDIUM` was used). Over gRPC the
same values are in `route_mode`, `judge_score` and `judge_model` on the response.

## Setting it up

You need a judge model and a rule for the `judge` modality:

```bash
grpcurl -plaintext -H "authorization: Bearer $ADMIN_KEY" -d '{"model":{"id":"jev-latest",
  "vendor":"typesafe","capabilities":["judge"],"efforts":["EFFORT_LOW"],"enabled":true}}' \
  localhost:9090 ktms.v1.AdminService/UpsertModel
```

```json
{"name":"judge", "priority":100, "attributes":{"modality":"judge"},
 "use":["jev-latest"], "enabled":true}
```

Add it to your rules with `ReplaceRules` (the whole set is replaced at once) and give the client
key a TypeSafe vendor key with `SetVendorKey`. Without a judge, `model: "auto"` still works: every
unpinned request simply goes out at `MEDIUM`.

## Is it worth it?

Measure it on your traffic rather than trusting a blog post. The usage events KTMS sends carry the
rule and model of every call, so you can compare cost and failure rates per effort before and after
turning auto-routing on. Things to watch:

* **Judge cost.** TypeSafe bills input tokens only; OpenAI Decisions bills input and output.
* **Misroutes.** A hard prompt routed `LOW` is worse than an easy prompt routed `HIGH`. Shadow
  traffic with `shadow_eval` can compare a cheaper model's answers with the primary's in the
  background, without touching the user's response.
* **Latency.** The judge's time is added before the model call. For streaming chat, check time to
  first token.

## The honest part

The judge's effort choice is marked **Experimental** in 1.x. It works and is supported, but the
question, the options and how the answer is mapped may change in a minor release, and the change
will be listed in the changelog. Auto-routing with a pinned effort (`auto:high`), placement policies
and `model_override` are stable.

* Code: [`internal/grpcapi/judge_auto.go`](https://github.com/ai-process/ktms/blob/main/internal/grpcapi/judge_auto.go)
* Docs: [Choosing a model and placement](https://github.com/ai-process/ktms#choosing-a-model-and-placement)
