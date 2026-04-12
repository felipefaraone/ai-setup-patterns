# ai-setup-patterns

**What I learned shipping a production AI assistant, and where I think it goes from here.**

[→ View the interactive demo](https://felipefaraone.github.io/ai-setup-patterns)

---

## What this is

I was the product manager on a production AI assistant built on MCP (Model Context Protocol) in a B2B SaaS product. I worked with a small team. They built the infrastructure and the interaction layer. I led product direction, contributed to prompt strategy, and owned the decisions around what to change when production data told us the architecture had limits.

The assistant was built with OpenAI models, Adaptive Cards for structured input, and a custom MCP server connected to the product's REST API.

We shipped it. It worked. Then we learned two things, one about the technology and one about the users. Both pointed in the same direction.

## The problem and what we built

The product had a complex setup process. Users needed to configure several interdependent setups during onboarding: business entities, integrations with external systems, data field mappings, targeting and routing rules. Most users couldn't do it alone. That meant support tickets, incomplete setups, and slow time to value, or in most cases early churn of really good prospects.

We built a conversational AI assistant that walked users through the full setup in chat. It used MCP to call backend APIs, collected input through interactive cards, and ran multi-phase flows with context summarization between phases. One entry point, one conversation, everything handled.

It shipped. In testing, the assistant completed a full setup in under 15 minutes with zero schema violations. Users tried it. Early adoption showed real activity.

Then the data started revealing more about usage.

## The technical learning

The production telemetry showed structural problems in the architecture. Over half of session time was the model generating UI components, not doing anything intelligent. The model skipped required steps in 20 to 30% of runs, lost state between phases, and behaved differently between calls.

We spent weeks stabilizing: splitting model usage across phases, running hundreds of test runs, restructuring context management. Testing prompt changes meant running the same flow dozens of times and comparing results manually. A single wording change could fix one step and break another. The feedback loop was slow, expensive, and non-deterministic.

It got better. But the complex steps kept failing at the same rate. The team reached a clear conclusion: the remaining issues couldn't be fixed with better prompts. We had put product logic inside a prompt (sequencing, validation, state management, UI rendering), and that logic needed to move into code.

This was a real and important learning. But even if we had solved every technical issue, it wouldn't have been enough. Because the second learning was about something else entirely.

## The product learning

Users would start with the assistant, hit friction, and either finish manually or ask support for help. We initially assumed the barriers were speed and bugs. Those were real, but they weren't the core reason.

The assistant only covered a slice of the total setup. Users still needed to configure campaigns, data sources, routing logic, and many other things the assistant didn't handle. All of that lived in the product UI they'd need to navigate regardless.

So the assistant wasn't saving users from learning the product. It was just deferring it. And users figured that out quickly: if I'm going to spend time in the product anyway for everything the assistant doesn't cover, I might as well set things up there too. Learning one path is easier than learning two.

The chat became a detour from a journey the user needed to take regardless.

On top of that, the assistant didn't always finish the job. Partial automation without a clear handoff is worse than no automation, because the user loses visibility into what was done and what wasn't.

The assistant proved there was interest in guided setup. But it didn't change behavior. No technical fix would have changed that. This was a product strategy question, not an execution one.

## Where both learnings converge

The technical side said: complex logic in prompts is fragile, untestable, and hits a ceiling. Move it to code.

The product side said: users rejected a separate AI path. The evidence suggests that contextual, embedded assistance has higher adoption than separate help channels, which is why the next step is to test that hypothesis with one helper.

Both point to the same direction: smaller, focused AI capabilities embedded inside the product, not a separate all-in-one assistant alongside it.

This progression has precedent. GitHub Copilot started as an inline code completion helper in 2021 and only expanded into chat, multi-file editing, and autonomous agents as each previous step proved adoption. Notion AI followed a similar arc: writing assistant (2022), then AI capabilities distributed into specific product contexts like database autofill (2023), then autonomous agents built on top of those foundations with Notion 3.0 (2025). Whether each step was deliberately contingent on the previous one is my inference, not something either company has stated, but the pattern is consistent: start embedded, prove value, expand scope.

## The embedded helper pattern

Instead of one big assistant that owns everything, you build focused helpers that each own one task. Not inside a chat. Inside the product, at the point where the user actually needs help.

Each helper takes one input, returns one output, and has zero responsibility for flow sequencing or state. The product handles workflow and validation. The AI does interpretation and suggestion. The product does execution.

```
All-in-one assistant:
User → [AI owns: sequencing + data collection + UI rendering + validation + execution]

Embedded helpers:
User → Product UI → [AI owns: one specific interpretation task]
                  → Product executes, validates, saves
```

Users stay in the product. They navigate, learn, keep control. When they hit a genuinely hard task, the helper is already there. No chat, no detour.

And the hard parts aren't things like choosing a schedule or entering a name. Those are straightforward. The hard parts are mapping fields between two systems with different naming conventions, or figuring out which criteria operators work with which data types. That's not learning. That's grunt work. Nobody gets smarter by manually mapping "first_name" to "contact_first_name." That's where AI helps without getting in the way.

The helpers are backend services the UI calls, like any other API. No frontend rewrite. And if the contracts are clean, the same tools can also be consumed externally via MCP by any agent that speaks the protocol.

## Where this goes from here

The assistant was the right first bet. It proved demand, generated the data that made the next step possible, and the infrastructure carries forward.

The next step is to pick the single hardest task, build one helper, and measure whether users adopt it. If they do, build the next one. If they don't, the broader thesis is weaker than the data suggests, and it's better to know early.

One helper at a time. Prove value, then expand. This is an informed direction, not a proven result. It's not proven until it ships.


### Sources
- [GitHub Copilot technical preview (2021)](https://github.blog/news-insights/product-news/introducing-github-copilot-ai-pair-programmer/)
- [GitHub Copilot agent mode (2025)](https://github.com/newsroom/press-releases/agent-mode)
- [GitHub coding agent (2025)](https://github.com/newsroom/press-releases/coding-agent-for-github-copilot)
- [Notion AI launch (2022)](https://www.notion.com/blog/introducing-notion-ai)
- [Notion AI Autofill (2023)](https://www.notion.com/releases/2023-05-31)
- [Notion 3.0 announcement (2025)](https://www.notion.com/blog/introducing-notion-3-0)


## Author

Felipe Faraone, Senior Product Manager
[felipefaraone.com](https://felipefaraone.com) · [LinkedIn](https://linkedin.com/in/felipefaraone) · [GitHub](https://github.com/felipefaraone)
