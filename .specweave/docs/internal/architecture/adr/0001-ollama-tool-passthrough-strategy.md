# 0001: Ollama Tool Passthrough Strategy

**Date**: 2026-04-06
**Status**: Accepted
**Increment**: 0003-ollama-tool-support

## Context

The anymodel proxy strips ALL tools for Ollama at `proxy.mjs:258-265`. This was correct when Ollama models couldn't use tools, but since Ollama v0.3.0 (July 2024), many models support native tool calling (Qwen 3 Coder, Llama 3.1+, Mistral, etc.). Claude Code sends 86+ MCP tool definitions with every request, and capable models should be able to invoke them.

The proxy already has `translateRequest()` in `openai.mjs` that converts Anthropic tool format to OpenAI/Ollama format (they're identical). The existing auto-retry logic at `proxy.mjs:485-514` already handles models that reject tools at runtime.

## Decision

### 1. Capability-aware passthrough with optimistic caching (not pre-detection)

**Chosen**: Optimistic send + runtime cache. Send tools on first request; if the model rejects them, cache the failure and skip tools on subsequent requests.

**Rejected alternative -- Ollama `/api/show` pre-detection**: Query Ollama's model metadata API before first request to check if a model lists tool support. Problems:
- Adds latency to first request (extra HTTP round-trip)
- Model metadata may not accurately reflect tool support
- Creates a dependency on Ollama's metadata API format
- The auto-retry path already handles failures gracefully

**Rejected alternative -- Hardcoded model list**: Maintain a list of known tool-capable models. Problems:
- Requires constant maintenance as new models are released
- Users with custom/fine-tuned models would be excluded
- Ollama model names have variants (`:latest`, `:q4_0`, etc.)

### 2. Reuse translateRequest() but NOT translateResponse()

**Chosen**: Reuse `translateRequest()` from openai.mjs for request translation. Write Ollama-native response translation in `ollamaToAnthropic()`.

**Rationale**: Ollama's request format (tools, messages) is identical to OpenAI's -- `translateRequest()` produces the correct output directly. However, Ollama's response format is fundamentally different:
- Ollama: `{ message: { content, tool_calls }, done, done_reason, eval_count }`
- OpenAI: `{ choices: [{ message: { content, tool_calls }, finish_reason }], usage: {...} }`

Wrapping Ollama responses in OpenAI shape just to reuse `translateResponse()` would add a shim layer with no benefit. The Ollama provider already has its own `ollamaToAnthropic()` -- extending it with tool_calls handling is the natural path.

### 3. Environment variable control with three modes

**Chosen**: `OLLAMA_TOOLS` env var with `auto` (default), `on`, `off`.

**Rationale**:
- `auto`: Best UX for most users. Tools work when possible, fail gracefully when not.
- `on`: Override for users who know their model supports tools and want to bypass cache (e.g., after model update).
- `off`: Preserves pre-increment behavior. Zero risk for users who don't want tool overhead.

### 4. Always strip tool_choice for Ollama

**Decision**: `tool_choice` is stripped from ALL Ollama requests, independent of `OLLAMA_TOOLS` setting.

**Rationale**: Ollama's API does not support `tool_choice`. Sending it causes errors. This is a hard constraint of the API, not a configuration preference. Even when `OLLAMA_TOOLS=on`, tool_choice must be removed.

### 5. In-memory cache only (no persistence)

**Decision**: Cache is a plain `Map` that resets on proxy restart.

**Rationale**: Users update models via `ollama pull`, which can change capabilities. A persistent cache would require invalidation logic on model changes. An in-memory cache automatically resets when the proxy restarts, which is the natural workflow after pulling new models. The cost of one extra failed request per model per proxy session is negligible.

## Consequences

- Tool-capable Ollama models can now invoke Claude Code tools through the proxy
- First request to an incapable model incurs one retry (existing auto-retry path), then subsequent requests are fast
- No new dependencies or external API calls
- Users can opt out entirely with `OLLAMA_TOOLS=off`
- Cache does not persist across restarts (by design)
