# ai-setup-patterns

**Where monolithic AI assistants hit their limits in production — and what to build instead.**

→ [Live demo](https://felipefaraone.github.io/ai-setup-patterns)

---

## Background

MCP (Model Context Protocol) is an open standard that lets AI models call external tools through a structured interface. In a setup flow, this means the AI can create records, fetch data, and execute configuration steps — not just talk about them.

This repository documents what I learned leading the product strategy for a production MCP-based AI assistant in a B2B SaaS environment. The engineering team built the infrastructure; product architecture and prompt design were developed collaboratively with the team. My role was translating product decisions into prompt logic, evaluating what broke, and defining the path forward.

---

## The pattern that seems right but isn't

When you first think about using AI to simplify a complex setup flow, the natural design is a conversational assistant. One entry point. The user types what they want. The AI figures out the steps, asks the right questions, and executes.

It demos well. Leadership approves it. Then you ship it.

Within weeks you have a list of failure modes that share a root cause you didn't see coming: **you put product logic inside a prompt**.

Sequencing. Validation. State management. UI rendering. All of it living in the AI layer, invisible to your backend, untestable in any meaningful way, and sensitive to model behavior that changes without warning.

---

## What shipped and what broke

The assistant guided users through multi-step onboarding: creating entities, configuring delivery channels, mapping fields, setting routing rules. It used MCP tools to call backend APIs, ran multi-phase flows with context summarization between phases, and used Adaptive Cards for structured input.

It worked. Users adopted it. Then the performance data came in.

**~52% of total session time was the LLM generating Adaptive Card JSON.**

Not the backend — the backend was 2–4% of total time. The bottleneck was the model doing work that had nothing to do with intelligence: building UI components, formatting payloads, rendering field lists. The field mapping preview alone took 46–56 seconds per session — ~4,000 output tokens to render a table the product could have generated in milliseconds.

That's the structural problem with the monolithic assistant pattern.

---

## Where it breaks down

**The prompt does work the product should do.**
Sequencing, validation, and state management belong in your application layer. When they live in a prompt, you've replaced deterministic code with probabilistic inference. The assistant might skip a required step, re-ask a question it already answered, or lose state across a context summarization boundary.

**Reliability degrades with complexity.**
Each additional phase or branching path increases the surface area for the model to deviate. A flow with 8 phases and 4 delivery types has far more failure modes than 8 separate, contained interactions. In practice: after 60+ test runs and significant prompt refinement, ~75% of remaining misbehavior cases were explicit instructions being ignored by the model — not ambiguity problems. At that point, you've hit the ceiling of what prompt engineering can do.

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

This pattern is not a novel idea — it's where the industry is converging.

Gartner predicts 40% of enterprise applications will embed task-specific AI agents by the end of 2026, up from less than 5% in 2025.

The products that validated this pattern didn't start with a monolithic assistant. GitHub Copilot launched in 2021 as inline autocomplete — one task, in the right place. In 2023 it added Copilot Chat. In 2025 it shipped agent mode and an autonomous coding agent. Each capability added incrementally, each embedded where it was relevant. Cursor followed the same arc: autocomplete in 2023, Composer for multi-file editing in 2024, agent mode as the default in 2025. The CEO of Cursor described their own evolution in three eras: Tab autocomplete, then synchronous agents, then autonomous cloud agents — "start narrow, prove value, expand deliberately" articulated as a product thesis.

Neither product tried to build a monolithic assistant that did everything from day one. Both started with the narrowest possible task, proved it worked, then layered complexity on top of a foundation that already had user trust.

The monolithic all-in-one assistant is the new monolith — and it has the same problems.

---

## What this looks like in practice

A user wants to set up a webhook integration.

**Monolithic:** The assistant carries the full flow in chat. Eight phases. Seven context summarizations. Forty-five assistant messages. Seven to ten minutes. When it fails, it can fail anywhere.

**Helpers:** The assistant understands intent, collects minimal context, and opens the webhook configuration screen. The product renders the form. When the user reaches field mapping, a helper is already there — embedded in that screen, ready to take their posting instructions and return suggestions. The assistant is a routing layer. The helpers do the work.

---

## Where this goes

The helper pattern is phase one of a longer arc.

**Phase 1 — Helpers prove value individually.**
Each helper solves one specific task. Field mapping. Posting instructions parsing. Schema generation. Small scope, measurable outcome, easy to validate.

**Phase 2 — Helpers become a reusable AI layer.**
The same field mapping helper that lives in the webhook screen gets called from the FTP screen, the email screen, any integration screen. The investment compounds.

**Phase 3 — The assistant becomes a routing layer.**
Instead of carrying the setup itself, the assistant understands intent and routes the user to the right screen and the right helper. It coordinates — it doesn't execute.

**Phase 4 — The system becomes proactive.**
With helpers running and generating data, the product surfaces insights without being asked. Delivery failures flagged before the user notices. Mapping gaps identified before leads are lost. The AI shifts from reactive to ambient.

This is the microservices moment for AI product design. Just as monolithic applications gave way to distributed service architectures, single all-purpose assistants are giving way to orchestrated systems of specialized helpers. The organizational and engineering challenges — state management across boundaries, orchestration logic, governance — are real. But the failure surface of each component stays contained. That's the trade worth making.

---

## When helpers don't work

Helpers are the right pattern when the task has a clear input and output, the user is already in the right context, and the AI is doing interpretation rather than execution.

They're the wrong pattern when:
- The user doesn't know where to start — a conversational entry point still has value here
- The task spans multiple contexts the product doesn't surface in one place
- The volume is too low to justify the UI integration investment

The honest answer is that both patterns have a role. The mistake is using the monolithic assistant for everything because it demos well.

---

## Repository structure

```
ai-setup-patterns/
├── README.md
├── patterns/
│   ├── monolithic/              # reference implementation
│   │   ├── system-prompt.md
│   │   ├── action.md
│   │   └── phases/
│   └── helpers/                 # embedded helper implementations
│       ├── field-mapper/
│       ├── schema-parser/
│       ├── schema-generator/
│       └── criteria-drafter/
├── evaluation/
│   ├── test-protocol.md
│   └── findings/
└── diagrams/
```

---

## On honesty in AI product work

The monolithic pattern shows its limits not because the people who built it made poor decisions — it's because the failure modes only become visible after you ship. The demo works. The first tests pass. The problems emerge at scale, under real usage, with models that behave differently on different days.

The embedded helper pattern has its own limitations. Helpers that are too narrow don't deliver enough value. Helpers in the wrong part of the flow don't get used. But when a helper fails, it fails in one place, on one task, in a way you can observe and fix. When a monolithic assistant fails, it can fail anywhere in an eight-phase flow, in ways that are hard to reproduce and harder to test.

That's the real reason to make the switch.

---

## Author

Felipe Faraone — Senior Product Manager  
[felipefaraone.com](https://felipefaraone.com) · [LinkedIn](https://linkedin.com/in/felipefaraone) · [GitHub](https://github.com/felipefaraone)
