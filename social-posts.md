# AnyModel Social Media Posts

## X (Twitter)

I was burning through my $200 Claude Max plan in a single afternoon.

So I built AnyModel — an open-source proxy that lets you run Claude Code with any AI model.

DeepSeek R1. GPT-5.4. Gemini. 300+ models. Including free ones.

Two commands:

npx anymodel proxy deepseek
npx anymodel

DeepSeek R1's full reasoning shows up live in your terminal. You watch it think through your bug in real time.

$0 cost? Qwen3 Coder and Nemotron — free, no credit card.
Full privacy? Ollama runs everything offline.

Open source. MIT licensed.

anymodel.dev
github.com/anton-abyzov/anymodel
Demo: youtu.be/k0RI_M6lIsg

## Threads

I was burning through my $200 Claude Max plan in a single afternoon. So I built something about it.

AnyModel — an open-source proxy that lets you run Claude Code with any AI model. DeepSeek R1, GPT-5.4, Gemini, Llama, 300+ models. Including completely free ones.

Two terminal commands and you're coding:

npx anymodel proxy deepseek
npx anymodel

The part that surprised even me: DeepSeek R1's full chain-of-thought reasoning shows up live in your terminal. You watch it think through your bug in real time.

Want $0 cost? Qwen3 Coder and Nemotron are free. No credit card needed.
Want full privacy? Ollama runs everything offline on your machine.

Open source. MIT licensed.

anymodel.dev
github.com/anton-abyzov/anymodel

---

## X (Twitter) — Reply to Theo (Claude Code blocks analysis)

Huh, NOT for AnyModel!

They blocked it in the latest version, but the source was already leaked. The entire Claude Code core — hooks system, 37 event types, trust validation — analyzed in one command:

npx anymodel claude

Run Claude Code's agentic framework with ANY model. Local. OpenRouter. Whatever works for you.

---

## Threads — Claude Code blocks analysis (standalone)

Claude Code now blocks you from analyzing its own source code.

So I built a tool that does it anyway.

One command. Any model. Local or cloud.

npx anymodel claude

The source was already leaked before they locked it down. AnyModel reverse-engineers the full system — 37 hook events, trust validation, permission chains — everything Anthropic doesn't want you to see.

Open source. Works right now.

---

## LinkedIn — Claude Code blocks analysis

Anthropic just blocked Claude Code from analyzing its own source. Here's why that matters.

Last week, Anthropic pushed an update that prevents Claude Code from inspecting its own codebase. Run the analysis — you get a usage policy violation.

But the source was already out there.

I've spent months studying Claude Code's internals. Here's what most people don't realize: Claude Code isn't magic. It's an agentic framework — a carefully engineered harness of system prompts, tool execution pipelines, and event-driven hooks that make the model useful.

The architecture is fascinating:
• 37 hook event types covering every stage of execution
• A full trust validation and permission chain system
• Pattern matching for tool routing, result processing, context management
• MCP (Model Context Protocol) server integration for extensibility

None of this is model-specific. The harness works. The model is interchangeable.

So I built AnyModel.

One command:

npx anymodel claude

It lets you run Claude Code's agentic framework with any model — local models, OpenRouter, whatever fits your use case and budget.

The takeaway for engineering leaders: the competitive moat in AI tooling isn't the prompt engineering. It's the ecosystem lock-in. Anthropic blocking self-analysis isn't about security — it's about preventing exactly this kind of interoperability.

The source is open. The tool works today.

Curious to hear from others building in this space — is model-agnostic tooling the future, or does tight model-framework coupling win long-term?

anymodel.dev
github.com/anton-abyzov/anymodel

First comment: #AItools #DeveloperTools #OpenSource #LLM #ClaudeCode #AnyModel
