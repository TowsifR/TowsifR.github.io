---
title: LLM Security Patterns
source: https://www.amazon.com/System-Design-LLM-Era-production-grade/dp/1807789934
author: Sampriti Mitra
date: 2026-06-29
---

Treating an LLM as just another API call is risky — you're inviting a component that doesn't behave predictably inside a system you're supposed to trust, and that opens up a whole category of attack that normal API security was never built to catch. The seven threats below line up closely with the [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) (2025, v2.0) — the book never says "OWASP" out loud, but the overlap is close to one-to-one, which matters because it means these aren't just one author's opinions, they're grounded in an actual industry-standard list. Part of [[ai-engineering|AI engineering]].

| Threat | OWASP category | Mitigation |
|---|---|---|
| **Prompt injection** — someone hides instructions inside their message to trick the model into ignoring its real rules | LLM01 | Keep the model's real instructions clearly separate from whatever the user typed; use a small, fast model to check incoming messages for anything suspicious before they reach the main one; double-check the model's own reply for signs it leaked its instructions |
| **Insecure output handling** — trusting the model's output without checking it first, which can let it run malicious code in someone's browser or worse | LLM05 | Treat everything the model outputs as untrusted: never run it as code, clean up anything shown on a webpage, double-check that JSON actually matches the shape you expect, and if the model helps build a database query, only let it fill in blanks in a query you already wrote — never run a raw query it wrote itself |
| **Excessive agency** — giving the model more power than it needs, so it can take real actions without anyone checking first | LLM06 | Only hand it the specific permissions a task needs, not a full toolkit; have it propose an action and require your code to approve it before anything actually runs; require a human's sign-off for anything high-stakes, like a refund or a delete |
| **Sensitive information disclosure** — the model reveals private data that leaked into its training data or the context it was given | LLM02 | Scrub private information out of data before it's used for training or lookup; for very sensitive data, process it in memory and throw it away immediately rather than storing it; make sure every lookup is limited to that specific user's own data, never the whole database |
| **Model denial of service** — someone floods the model with expensive requests to run up your bill or knock it offline | LLM10 (Unbounded Consumption) | Limit how many requests one person or IP can send; reject obviously oversized input; if someone's spending is unusually high, quietly switch them to a cheaper model instead of shutting them out entirely |
| **Data poisoning** — someone tampers with training or fine-tuning data to corrupt how the model behaves | LLM04 | Only use data from sources you actually trust and can track back to their origin; have a human review any data before it's used to fine-tune a model |
| **Supply chain and insecure plugin vulnerabilities** — a third-party plugin, dataset, or dependency turns out to be the weak point | LLM03 | Give every plugin the smallest amount of access it needs (read email, not delete email); never trust data flowing through a plugin; keep dependencies up to date and scanned; route everything through the [[llm-architectural-patterns|LLM gateway]] so a compromised provider or model can be swapped out fast |

The model-denial-of-service fix deserves a real example, since "just rate limit it" undersells what actually works well: pair a hard per-user limit with a *soft* budget check that quietly downgrades instead of cutting someone off:

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

Someone who's simply an expensive user today still gets an answer — just from a cheaper model — while someone who's actually spamming the system gets cut off outright. That difference between throttling and rejecting is the useful part to remember; the actual numbers are whatever fits your own budget.

## Privacy

Security and privacy are related but different questions: security asks "can an attacker misuse this system," privacy asks "does this system leak someone's data even when nobody's attacking it." For something like a code assistant, the goal is giving useful answers about a codebase without ever putting that codebase at risk of leaking:

- **Ephemeral data handling** — never write code or filenames to disk or a database; anything decrypted for processing lives only in memory and gets thrown away the moment it's no longer needed
- **Search without storing the real thing** — instead of saving the actual code, save a one-way mathematical fingerprint of it that can't be turned back into the original, and replace real file and function names with anonymous IDs
- **Send as little as possible** — only send the small piece of context an answer actually needs, not the whole codebase
- **Encrypt everywhere** — encrypt data on the way in and out, and encrypt anything that does end up stored

These four work well together as one real design, not just a checklist for an audit — the idea running through all of them is that there's simply nothing sensitive left sitting around to leak, instead of hoping access controls keep attackers away from data that's just sitting there unprotected.

## Sources

- Sampriti Mitra, *System Design for the LLM Era: Patterns and Principles for Production-Grade AI Architecture* (Packt, 2026), Chapter 2
- [OWASP Top 10 for LLM Applications (2025, v2.0)](https://genai.owasp.org/llm-top-10/)
