# ai-setup-patterns

**Where monolithic AI assistants hit their limits in production — and what to build instead.**

**[→ View the interactive demo](https://felipefaraone.github.io/ai-setup-patterns/)**

---

## Background

MCP (Model Context Protocol) is an open standard that lets AI models call external tools through a structured interface. In a setup flow, this means the AI can create records, fetch data, and execute configuration steps — not just talk about them.

This repository documents what I learned leading the product strategy for a production MCP-based AI assistant in a B2B SaaS environment. I worked closely with a product designer and the engineering team — they built the infrastructure and interaction design, while I owned the product architecture, prompt strategy, and the decision framework for what to do when production data told us to change direction.

---

## Status

The monolithic assistant is what shipped. It works, users adopted it, and it delivered real results — onboarding time went from days to under 15 minutes, with zero schema violations in production.

The embedded helper pattern is the architectural direction that emerged from production telemetry. It's the natural next step — not because the MVP failed, but because real usage data revealed where the architecture needs to evolve to scale. Companies like Intercom, HubSpot, and Salesforce have faced similar inflection points when moving AI from single-assistant prototypes to distributed, task-specific models. This repository documents that evolution: what we shipped (the MVP), what we measured, what the data told us, and what the next version looks like.

---

## The pattern that seems right but isn't

When you first think about using AI to simplify a complex setup flow, the natural design is a conversational assistant. One entry point. The user types what they want. The AI figures out the steps, asks the right questions, and executes.

It demos well. Leadership approves it. Then you ship it.

Within weeks you have a list of failure modes that share a root cause you didn't see coming: **you put product logic inside a prompt**.

Sequencing. Validation. State management. UI rendering. All of it living in the AI layer, invisible to your backend, untestable in any meaningful way, and sensitive to model behavior that changes without warning.

---

## What shipped and what we learned

The assistant guided users through multi-step onboarding: creating entities, configuring delivery channels, mapping fields, setting routing rules. It used MCP tools to call backend APIs, ran multi-phase flows with context summarization between phases, and used Adaptive Cards for structured input.

It worked. Users adopted it. Onboarding time dropped from days to under 15 minutes, with zero schema violations in production. As usage scaled, the performance telemetry revealed where the architecture needed to evolve.

**~52% of total session time was the LLM generating Adaptive Card JSON.**

Not the backend — the backend was 2–4% of total time. The bottleneck was the model doing work that had nothing to do with intelligence: building UI components, formatting payloads, rendering field lists. The field mapping preview alone took 46–56 seconds per session — ~4,000 output tokens to render a table the product could have generated in milliseconds.

That's the structural problem with the monolithic assistant pattern.

---

## Where the architecture reaches its ceiling

**The prompt does work the product should do.**
Sequencing, validation, and state management belong in your application layer. When they live in a prompt, you've replaced deterministic code with probabilistic inference. The assistant might skip a required step, re-ask a question it already answered, or lose state across a context summarization boundary.

**Reliability degrades with complexity.**
Each additional phase or branching path increases the surface area for the model to deviate. A flow with 8 phases and 4 delivery types has far more failure modes than 8 separate, contained interactions.

**Testing is expensive and non-deterministic.**
You can't unit test a prompt. You run it, observe what happens, patch the wording, run it again. At scale this becomes dozens of manual runs per change, with results that vary between model versions and between calls to the same model.

**Performance scales with output tokens, not backend complexity.**
The slowest operations are almost always UI generation — the model building structured JSON for cards, tables, and choice sets. Formatting work the LLM is slow at and bad at.

---

## The embedded helper pattern

Instead of one assistant that owns everything, you build focused helpers embedded directly in the product UI — at the point where that specific task is relevant.

Each helper has a narrow contract:
- One input type (a JSON payload, a text description, a field list)
- One output type (a suggested mapping, a parsed config, a set of field proposals)
- Zero responsibility for flow sequencing, state, or validation

The product stays responsible for everything else. The AI does interpretation and suggestion. The product does execution.

```
Monolithic:
User → [AI owns: sequencing + data collection + UI rendering + validation + execution]

Helpers:
User → Product UI → [AI owns: one specific interpretation task]
                  → Product executes, validates, saves
```

**Reliability improves** because each helper has fewer things to get wrong.

**Performance improves** because the AI does less per call — no UI generation, no card rendering, no payload formatting.

**Testability improves** because each helper can be evaluated independently: given input X, does the output meet expectations?

---

## What this looks like in practice

A user wants to set up a webhook integration.

**Monolithic:** The assistant carries the full flow in chat. Eight phases. Seven context summarizations. Forty-five assistant messages. Seven to ten minutes. When it fails, it can fail anywhere.

**Helpers:** The assistant understands intent, collects minimal context, and opens the webhook configuration screen. The product renders the form. When the user reaches field mapping, a helper is already there — embedded in that screen, ready to take their posting instructions and return suggestions. The assistant is a routing layer. The helpers do the work.

---


## Why this matters for the industry

The monolithic assistant was the right first step. It validated the core premise — AI can meaningfully accelerate complex setup flows — and delivered measurable results in production. The architectural evolution toward embedded helpers isn't a correction; it's the natural next phase that production telemetry made possible.

This mirrors a broader pattern across B2B SaaS: companies like Notion, Linear, and Figma have moved from single AI features to distributed, context-aware helpers embedded at the point of need. The insight is the same — narrow scope, high precision, and product-layer orchestration outperform monolithic AI ownership as complexity grows.

The embedded helper pattern has its own design challenges. Helpers that are too narrow don't deliver enough value. Helpers in the wrong part of the flow don't get used. But each helper can be evaluated, measured, and improved independently — which is what makes the architecture scalable.

---

## Author

Felipe Faraone — Senior Product Manager  
[felipefaraone.com](https://felipefaraone.com) · [LinkedIn](https://linkedin.com/in/felipefaraone) · [GitHub](https://github.com/felipefaraone)



