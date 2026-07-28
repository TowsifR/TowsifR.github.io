---
title: Graph Engineering
---

Explicitly modeling which steps an agent (or multi-agent system) is allowed to run next — nodes as steps, edges as allowed transitions — instead of letting the model freely decide every time. Useful when there's real branching, parallel work, or recovery paths that need to be inspectable and enforced rather than left to the model's judgment call.

Not to be confused with knowledge graphs (modeling data/entities) — same words, different concept.

Part of [[ai-engineering|AI engineering]]. See [[agent-harness-loop-graph-engineering|Harness vs. Loop vs. Graph Engineering]] for how this fits alongside the other two layers.

Tools: LangGraph, AutoGen's GraphFlow.
