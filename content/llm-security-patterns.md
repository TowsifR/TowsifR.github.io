---
title: LLM Security Patterns
source: https://www.amazon.com/System-Design-LLM-Era-production-grade/dp/1807789934
author: Sampriti Mitra
date: 2026-06-29
---

Treating an LLM as a simple API call is naive and dangerous — it's a non-deterministic component you're inviting inside a trusted system, and it opens up a class of vulnerability that standard API security doesn't cover. The seven threat/mitigation pairs below line up closely with the [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) (2025, v2.0) — the book doesn't name OWASP directly, but the overlap is close to 1:1, which is worth knowing since it means these patterns are grounded in an actual industry-standard taxonomy rather than one author's opinion. Part of [[ai-engineering|AI engineering]].

| Threat | OWASP category | Mitigation |
|---|---|---|
| **Prompt injection** — attackers embed malicious instructions in input to override the system's original behavior | LLM01 | Separate trusted instructions from untrusted input (role-based `system`/`user` structure, clear delimiters); filter suspicious intent with a second, faster model before it reaches the primary one; validate the response itself for leaked system-prompt text |
| **Insecure output handling** — an LLM's output is used without sanitization, opening the door to XSS and similar attacks | LLM05 | Treat all output as untrusted: never `eval()` it, sanitize/encode HTML or text output, parse expected JSON in a try/catch with schema validation, and if the model helps build SQL, have it only generate parameters for a query you control — never execute a raw string it produced |
| **Excessive agency** — the model is given more control than it needs and takes unauthorized actions without oversight | LLM06 | Grant dynamic, scoped permissions rather than a static full toolset; use a plan-approve-execute loop where the model proposes and your code approves before anything executes; require explicit human approval for high-impact actions like deleting data or issuing a refund |
| **Sensitive information disclosure** — the model reveals confidential data or PII that leaked into its training data or context | LLM02 | Scrub PII before training or RAG ingestion; use zero-retention processing for ultra-sensitive data (decrypt in memory, discard immediately); filter every RAG query by `tenant_id`/`user_id` rather than searching the whole index and trusting the model to pick the right rows |
| **Model denial of service** — resource-intensive requests overload the model or make it unavailable to other users | LLM10 (Unbounded Consumption) | Enforce per-user/per-IP rate limits at the gateway; reject obviously abusive input (a 50,000-token prompt); throttle by real-time cost, not just request count, so one expensive user degrades to a cheaper model instead of taking the budget down for everyone |
| **Data poisoning** — training or fine-tuning data is tampered with to corrupt model behavior | LLM04 | Ingest only from trusted, tracked sources with real data lineage; put fine-tuning datasets through human-in-the-loop review before training |
| **Supply chain and insecure plugin vulnerabilities** — third-party plugins, datasets, or dependencies are themselves the weak point | LLM03 | Give every plugin the minimum functionality it needs (`read_email`, not `delete_email`); treat all data passed into a plugin as untrusted; scan dependencies and containers regularly; route everything through the [[llm-architectural-patterns|LLM gateway]] so a compromised provider or model can be swapped out fast |

Model DoS mitigation is worth a concrete example, since "rate limit it" undersells the actual shape of the fix — a real implementation layers a hard per-user rate limit with a *soft* budget check that downgrades rather than blocks:

```python
def route_request(user, prompt):
    # Hard limit: abuse -> stop
    if redis.get(f"rate:{user.id}") > 50:
        raise HTTPException(status_code=429, detail="Rate limit exceeded.")

    # Soft limit: over daily budget -> downgrade, don't fail
    if db.get_spend(user.id) > DAILY_BUDGET:
        return call_llm(model="llama-3-8b", prompt=prompt)

    return call_llm(model="gpt-4", prompt=prompt)
```

A user who's simply expensive today still gets served — just on a cheaper model — while a user who's actually spamming the system gets cut off outright. That distinction (throttle vs. reject) is the part worth keeping; the specific thresholds are whatever your actual cost model demands.

## Privacy

Security and privacy overlap but aren't the same question — security is "can an attacker misuse this system," privacy is "does this system leak the user's own data even when nothing's attacking it." For something like a code-completion or code-review assistant, the goal is serving useful answers about a codebase without that codebase ever being at risk of leaking:

- **Ephemeral data handling** — code snippets and filenames are never written to disk or a database; anything decrypted for processing lives only in memory and is discarded the instant it's no longer needed.
- **Embedding-only search** — the system never stores the actual code, only its vector representation. That conversion is one-directional, so the original code can't be reconstructed from the store even in a breach, and real file/function names get replaced with anonymous hashed IDs in metadata.
- **Minimized code transfer** — only the context actually needed for a given query or completion is sent to the server, not the surrounding codebase.
- **Encryption everywhere** — requests are encrypted in transit, and anything that does land on disk server-side is encrypted at rest.

These four hold up as a genuinely coherent design, not just a compliance checklist — the through-line is that the system is architected so there's nothing sensitive left to leak, rather than relying on access controls to keep attackers away from data that's sitting there in the clear.

## Sources

- Sampriti Mitra, *System Design for the LLM Era: Patterns and Principles for Production-Grade AI Architecture* (Packt, 2026), Chapter 2
- [OWASP Top 10 for LLM Applications (2025, v2.0)](https://genai.owasp.org/llm-top-10/)
