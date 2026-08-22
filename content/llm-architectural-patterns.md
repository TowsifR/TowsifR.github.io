---
title: Core Architectural Patterns for LLM Systems
source: https://www.amazon.com/System-Design-LLM-Era-production-grade/dp/1807789934
author: Sampriti Mitra
date: 2026-06-29
---

Integrating an LLM into a production system introduces a new class of dependency: one that's non-deterministic, high-latency, and carries a variable, sometimes unbounded, operational cost. None of that calls for reinventing how we build reliable systems — it calls for adapting the patterns we already have to a component that behaves nothing like a normal database or microservice call. Four concerns dominate: keeping the system stable when the model provider isn't, keeping response times tolerable when inference itself is slow, keeping cost from spiraling now that every architectural decision is also a financial one, and keeping answers grounded in real data rather than whatever the model happens to hallucinate. Part of [[ai-engineering|AI engineering]].

## Resilience: decouple the system from the provider

The instinct when you first build with LLMs is to treat them like any other third-party API — install the SDK, generate a key, call it directly from application code. It ships fast, but it creates a high-coupling architecture: switching providers means rewriting call sites, reliability becomes inconsistent across services, spend is scattered across consoles with no central view, and API keys end up duplicated across environment variables, widening the surface area for a leak.

The fix borrows directly from microservices: stop treating LLMs as external vendors and put a single **LLM gateway** in front of them — one internal service that's the only thing allowed to call OpenAI, Anthropic, or Google, exposed to the rest of the system through one unified API format. Every other resilience, cost, and monitoring pattern in this note gets implemented once, in this one place, instead of being duplicated per service. Swapping GPT-5 for Claude becomes a config change instead of a redeploy, and if a provider goes down, the gateway can retry against a different one without the calling service ever finding out.

On top of the gateway sits a **circuit breaker with tiered fallbacks**. The gateway monitors each provider's latency and error rate; when a primary model breaches a failure threshold, the circuit trips and traffic reroutes to a backup automatically, rather than retrying blindly against a provider that's already struggling.

| Tier | Model | Profile |
|---|---|---|
| 1 (primary) | GPT-5 | high-cost, high-reasoning |
| 2 (fallback) | Claude 3 Haiku | medium-cost, fast |
| 3 (fallback) | Llama 3 8B, locally hosted | free, less capable |
| 4 (final fallback) | cached response, or a graceful error to the UI | — |

A circuit can't stay open forever, so recovery needs its own logic: sleep for a defined cooldown (say 30 seconds), then let a single canary request probe the primary provider. If it succeeds, close the circuit and resume full traffic; if it fails, restart the cooldown.

![The LLM gateway sits between internal services and model providers; a circuit breaker trips from the primary tier down through progressively cheaper fallbacks, with a canary probe on cooldown to test recovery](assets/llm-architectural-patterns/gateway-circuit-breaker.svg)

Retry strategy should also split by whether a user is waiting. In the **interactive** path, skip aggressive exponential backoff — if a user is staring at the screen, a 60-second retry is effectively downtime, so use capped backoff (start at 500ms, cap at 1s) or fail fast to a fallback model. In the **asynchronous** path, nobody's waiting, so exponential backoff (1 min, 2 min, 4 min) is affordable and lets the system ride out a multi-minute provider outage instead of giving up early.

## Latency: hide the wait, since you can't eliminate it

A user expects a search result in under 500ms; an LLM can take 5–10 seconds to generate a full answer. Inference speed isn't something the application layer can fix, so the architecture has to work around it instead.

The first move is a **hybrid sync/async split**. The synchronous path (under ~2 seconds) is for things that must be fast — code completion, a real-time search box — and should lean on fast, cheap models or caching rather than a slow, capable model. The asynchronous path (over ~10 seconds) is for long-running work — report generation, agentic workflows, offline content generation — decoupled from the request via a message queue (Kafka, SQS). The client gets an immediate `202 Accepted` and polls or receives a callback/WebSocket push when the result is ready.

Even a genuinely fast 3-second response feels slow if the user is staring at a spinner the whole time, which is why **response streaming** is close to mandatory for anything conversational. Server-sent events (SSE) run unidirectionally over standard HTTP/2 — firewall-friendly, easy to implement — with the server returning a generator instead of a single JSON object and setting `content-type: text/event-stream`. Streaming the first token as soon as it's generated, rather than waiting for the full response, is what actually moves the needle on perceived latency (time-to-first-token, TTFT).

Caching is the other lever, and it's worth layering rather than treating as a single on/off switch:

| Level | Scenario | Mechanism |
|---|---|---|
| L1 — exact match | 10,000 users ask "when is shipping?" during a viral launch | Hash the prompt string, check Redis; 9,999 users get a ~5ms response |
| L2 — semantic match | User A asks "how do I reset password?", user B asks "forgot password, help" | L1 misses on the string; embed the query, search a vector DB for similar past questions, and reuse the cached answer above a similarity threshold |
| L3 — proactive | A personalized daily report for 100,000 users | Don't wait for the user to open the app — run a batch job hours earlier and pre-populate the cache, moving the latency from online (user waiting) to offline |

![A query checks the L1 exact-hash cache first, falls through to L2 semantic similarity search, and only reaches the LLM on a double miss; a separate proactive batch job pre-warms an L3 cache ahead of expected demand](assets/llm-architectural-patterns/cache-tiers.svg)

During a real traffic spike, thousands of users can ask the same question at the same moment, and a standard cache misses for all of them the first time — a stampede of identical requests hitting the provider at once. **Coalesce caching** (request collapsing) fixes this at the middleware layer: identify identical in-flight requests, pause the duplicates, wait for the first one to finish, then serve that single response to everyone who was waiting on it. This reduces load on the provider by orders of magnitude during exactly the moments it matters most.

## Cost: every architectural decision is now a financial one

LLM calls introduce a new, variable, and potentially unbounded cost of goods sold — a single complex query can cost real money, which means decisions that used to be purely technical now have a budget line attached.

The **model router** pattern avoids paying premium-model prices for tasks that don't need them — using a frontier model for a grammar check is booking a helicopter to beat traffic. A rule engine inside the gateway routes each request to the cheapest model that's good enough for the task:

```
IF task_type == 'simple_grammar_check' THEN route_to 'local_llama_8b'
IF task_type == 'complex_reasoning' AND user_tier == 'premium' THEN route_to 'GPT-5'
```

Static rules like that can bottleneck under a traffic spike, so a production-grade router also does **dynamic, utilization-based routing**: if the primary model's latency breaches a P99 SLA (say, over 2 seconds) due to provider congestion, the router proactively load-sheds onto a faster or cheaper model — even for requests that would normally warrant the expensive one. A hard cap on prompt size (e.g. force-routing anything over 8k tokens to a cheaper model) acts as a cost ceiling so a single oversized query can't cause a cost regression on its own.

Because cost scales with token count, treat prompt context itself as something to optimize. Before sending 50 pages of chat history into a large model, use a cheaper, faster model to **summarize or compress it first**.

## Grounding: fighting hallucination and stale knowledge

LLMs invent facts and their training data has a cutoff — neither is fixable by prompting harder. The reliable fix is **retrieval-augmented generation (RAG)**: instead of asking the model a question and hoping it knows the answer, retrieve the actual data and tell it the answer. Given "what's the status of order #123?", first query the database for the order details (*retrieve*), inject those details into the prompt (*augment*), then instruct the model to answer using only that context (*generate*).

Building the retrieval side means standing up an **ingestion pipeline** — typically an asynchronous, scalable job (Spark is common) that chunks source documents into semantically meaningful pieces, embeds each chunk via an embedding model, and stores the chunk plus its vector in a vector database (OpenSearch, Pinecone, `pg_vector`).

Vector search alone is good at finding similar content but bad at traversing relationships between entities, which is where **hybrid RAG** — GraphRAG being the best-known form — comes in: the ingestion pipeline populates both a vector database *and* a knowledge graph (Neptune, Neo4j), and the RAG orchestrator queries both to assemble richer, more accurate context for interconnected data. (Worth being precise on terms here: this is the *knowledge-graph* sense of "graph" — modeling data and entity relationships — not the [[graph-engineering|workflow-graph]] sense of controlling what an agent runs next. Same word, different concept.)

The other grounding tool is **function calling**: since LLMs are generally bad at math and can't touch the outside world on their own, give them a schema of available tools (`get_weather(city)`) and let the model emit a structured request to call one. The application layer executes the actual code and feeds the result back in.

These four concerns — resilience, latency, cost, grounding — form the base layer of the architect's playbook. The other two pillars, evaluating whether the system is actually any good and defending it against a new class of attack, are covered in [[evaluating-llm-systems|Evaluating LLM Systems]] and [[llm-security-patterns|LLM Security Patterns]].

## Sources

- Sampriti Mitra, *System Design for the LLM Era: Patterns and Principles for Production-Grade AI Architecture* (Packt, 2026), Chapter 2
