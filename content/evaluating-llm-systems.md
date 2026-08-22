---
title: Evaluating LLM Systems
source: https://www.amazon.com/System-Design-LLM-Era-production-grade/dp/1807789934
author: Sampriti Mitra
date: 2026-06-29
---

Testing a non-deterministic system breaks the usual unit-test assumption. `assert(response) == "expected_string"` fails constantly against an LLM, not because the system is broken, but because there was never a single correct string to begin with. The fix isn't to give up on regression testing — it's to swap exact-match assertions for a measurable quality score, checked against a fixed reference set on every change. Part of [[ai-engineering|AI engineering]]; the same pattern set reuses across every case study built on top of it.

## Golden datasets and LLM-as-a-Judge

A **golden dataset** is 50–100 representative inputs paired with their ideal outputs. Running the system against this set in CI/CD on every prompt change or model upgrade catches regressions that would otherwise ship silently — the set doesn't need to be huge, it needs to be representative and stable.

The harder question is how to score quality at that scale without a human reading every response by hand. The answer is to use a second, more powerful model as the grader: feed it the original prompt, the golden answer, and the system's actual response, and have it score the response from 1 to 5 on axes like accuracy, groundedness, and tone. This **LLM-as-a-Judge** pattern turns a subjective "does this look right" into a quantifiable metric you can track over time and gate a deployment on.

![A golden dataset feeds both the system under test and an LLM judge; the judge scores the system's actual response against the ideal answer, and the aggregate score gates whether the change deploys](assets/evaluating-llm-systems/eval-pipeline.svg)

Standard APM (CPU, RAM, 5xx rates) doesn't capture what's actually going wrong in an LLM system, so observability needs an added layer specific to the model itself:

- **Cost** — `Cost_Per_Query` and `Total_Cost_Per_User`, with alerts on spikes
- **Performance** — P99 time-to-first-token (TTFT) and tokens-per-second (TPS)
- **Provider health** — `429_Rate_Limit_Errors` and `5xx_Server_Errors` per provider, feeding directly into the [[llm-architectural-patterns|circuit breakers]] that react to them
- **Quality** — LLM-as-a-Judge scores over time, plus the **escalation rate**: the percentage of conversations the system fails to resolve and has to hand off to a human. A spike here is usually the earliest sign of a model-quality regression, ahead of any complaint reaching a support queue.

## Quantitative evaluation: turning "mostly right" into a number

Golden-set scoring works well for single-turn responses, but agentic workflows are multi-step and non-deterministic in a different way — a task can be mostly correct but slightly off in tone, style, or one sub-step, and a pass/fail unit test can't capture that nuance.

The fix is to score every agentic run against **weighted criteria** that reflect what actually matters for the task — for example, logic worth 50% of the score, syntax 30%, documentation 20% — then run the agent across a golden dataset of at least 50 representative tasks and compute a single **correctness percentage** from the weighted results. That number becomes a real CI/CD gate: if a prompt change or model upgrade drops it below a defined threshold (90%, say), the deployment blocks automatically instead of shipping a quality regression that nobody would have caught by eyeballing a few examples.

## Building the test data itself

Golden datasets and evaluation criteria are only as good as the data behind them, and how you source that data depends on what's being tested:

- **Code completion**: curate a diverse set of open-source repos across languages, sizes, and domains, refreshed regularly to track current style and practice. Randomly strip code blocks, functions, and classes, start typing the signature of what's missing, and compare the system's completions against the actual removed code to measure accuracy.
- **Chat responses**: clone a repo with comments and docs stripped out, ask the system to explain code blocks and functions, and compare its explanations against the original README/docstrings/comments.
- **Agentic workflows**: assign real tasks — add documentation, write tests, refactor a module — often combined into multi-step scenarios, and check the resulting code, tests, and docs for correctness and style automatically.

Two adversarial variants round this out. **Mutation testing** deliberately introduces logic and syntax errors into code and checks whether the system catches and fixes them. **Negative prompt testing** feeds the system confusing or risky instructions — "delete all databases" — and checks that it flags the risk, refuses, or asks for clarification rather than complying. Wiring all of this into CI/CD is what actually keeps prompts and models in sync with an evolving codebase, instead of drifting quietly out of date between manual reviews.

## Sources

- Sampriti Mitra, *System Design for the LLM Era: Patterns and Principles for Production-Grade AI Architecture* (Packt, 2026), Chapter 2
