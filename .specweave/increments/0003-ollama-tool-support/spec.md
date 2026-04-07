---
increment: 0003-ollama-tool-support
title: "Enable Ollama Tool Calling Passthrough"
type: feature
priority: P1
status: planned
created: 2026-04-06
structure: user-stories
test_mode: TDD
coverage_target: 80
---

# Feature: Enable Ollama Tool Calling Passthrough

## Overview

The anymodel proxy currently strips ALL tools for the Ollama provider at `proxy.mjs:261-265`. Many Ollama models (Qwen 3 Coder, Llama 3.1+, Mistral, DeepSeek R1, etc.) now support native tool calling since Ollama v0.3.0 (July 2024). This increment replaces the blanket strip with a capability-aware passthrough controlled by the `OLLAMA_TOOLS` environment variable, with per-model caching of tool support status.

### Technical Context

- Ollama's native `/api/chat` endpoint uses the same tool/function format as OpenAI
- `translateRequest()` from `openai.mjs` already converts Anthropic tool format to OpenAI/Ollama format (tools, tool_calls, tool_results)
- `translateResponse()` from `openai.mjs` already handles `tool_calls` in non-streaming responses
- Ollama does **NOT** support `tool_choice` -- must always be stripped for Ollama
- The existing auto-retry without tools at `proxy.mjs:485-514` handles models that reject tools at runtime
- Ollama native response format includes `message.tool_calls` identical to OpenAI's format

## User Stories

### US-001: Tool Capability Configuration Module (P1)
**Project**: anymodel

**As a** developer running local Ollama models
**I want** a configuration module that controls whether tools are sent to Ollama
**So that** I can enable tool calling for capable models or force it off for incompatible ones

**Acceptance Criteria**:
- [x] **AC-US1-01**: New module `providers/ollama-tools.mjs` exports `shouldSendTools(model)` that returns `true`/`false`
- [x] **AC-US1-02**: Environment variable `OLLAMA_TOOLS` controls behavior: `auto` (default) = send tools and cache result; `on` = always send tools; `off` = never send tools (preserves current behavior)
- [x] **AC-US1-03**: Per-model capability cache: after a successful tool-use response, `cacheToolResult(model, true)` marks the model as tool-capable; after a tool-related error, `cacheToolResult(model, false)` marks it as incapable
- [x] **AC-US1-04**: When `OLLAMA_TOOLS=auto`, `shouldSendTools()` returns `true` for uncached models (optimistic), `true` for cached-capable models, `false` for cached-incapable models
- [x] **AC-US1-05**: `clearToolCache()` export allows cache reset (useful for testing and when model versions change)

---

### US-002: Tool Request Passthrough (P1)
**Project**: anymodel

**As a** developer using a tool-capable Ollama model
**I want** the proxy to pass tool definitions through to Ollama
**So that** the model can invoke tools like file editing, search, and bash commands

**Acceptance Criteria**:
- [x] **AC-US2-01**: `ollama.transformRequest()` passes `tools` array from the OpenAI-translated body to the Ollama request body when tools are present
- [x] **AC-US2-02**: `tool_choice` is NEVER included in the Ollama request body (Ollama does not support it)
- [x] **AC-US2-03**: When `shouldSendTools()` returns `false`, `proxy.mjs` strips tools before calling `transformRequest()` (same behavior as current blanket strip)
- [x] **AC-US2-04**: When `shouldSendTools()` returns `true`, tools pass through to Ollama unmodified

---

### US-003: Tool Response Handling -- Non-Streaming (P1)
**Project**: anymodel

**As a** developer using tool calling with Ollama in non-streaming mode
**I want** tool call responses to be correctly translated to Anthropic format
**So that** Claude Code can process tool invocations from Ollama models

**Acceptance Criteria**:
- [x] **AC-US3-01**: `ollamaToAnthropic()` converts `message.tool_calls` array to Anthropic `tool_use` content blocks with correct `id`, `name`, and `input` fields
- [x] **AC-US3-02**: When tool_calls are present, `stop_reason` is set to `"tool_use"` instead of `"end_turn"`
- [x] **AC-US3-03**: Mixed responses (text + tool_calls) produce both `text` and `tool_use` content blocks in the correct order
- [x] **AC-US3-04**: On successful tool-use response, `proxy.mjs` calls `cacheToolResult(model, true)`

---

### US-004: Tool Streaming Handling (P2)
**Project**: anymodel

**As a** developer using tool calling with Ollama in streaming mode
**I want** streamed tool call chunks to be correctly translated to Anthropic SSE events
**So that** Claude Code receives real-time tool invocation events during streaming

**Acceptance Criteria**:
- [x] **AC-US4-01**: `createOllamaStreamTranslator()` detects `message.tool_calls` in streamed NDJSON chunks
- [x] **AC-US4-02**: Tool call start emits `content_block_start` SSE event with `type: "tool_use"`, `id`, and `name`
- [x] **AC-US4-03**: Tool call argument chunks emit `content_block_delta` SSE events with `type: "input_json_delta"` and `partial_json`
- [x] **AC-US4-04**: When the final chunk has `done: true` and tool_calls were emitted, `stop_reason` is `"tool_use"`
- [x] **AC-US4-05**: Text content and tool_calls in the same stream are both correctly translated with proper block indexing

---

### US-005: Proxy Capability-Aware Tool Logic (P1)
**Project**: anymodel

**As a** developer using the anymodel proxy with Ollama
**I want** the proxy to intelligently decide whether to send tools per-model
**So that** tool-capable models get tools while incompatible models get clean requests without tools

**Acceptance Criteria**:
- [x] **AC-US5-01**: `proxy.mjs` replaces the blanket tool strip block (lines 258-265) with a call to `shouldSendTools(model)`
- [x] **AC-US5-02**: When `shouldSendTools()` returns `false`, tools and tool_choice are stripped with a log message (preserves current UX)
- [x] **AC-US5-03**: `tool_choice` is always stripped for Ollama regardless of `shouldSendTools()` result
- [x] **AC-US5-04**: On successful response containing tool_calls, `cacheToolResult(model, true)` is called
- [x] **AC-US5-05**: When the existing auto-retry-without-tools path triggers (error contains "support tool use"), `cacheToolResult(model, false)` is called so subsequent requests skip tools immediately
- [x] **AC-US5-06**: Log output clearly indicates the tool decision: `[OLLAMA] Tools: sending N tools to model` or `[OLLAMA] Tools: stripped (model cached as incapable)` or `[OLLAMA] Tools: stripped (OLLAMA_TOOLS=off)`

## Functional Requirements

### FR-001: Environment Variable Configuration
`OLLAMA_TOOLS` accepts three values:
- `auto` (default): Optimistically send tools; cache success/failure per model
- `on`: Always send tools regardless of cache (useful for testing new models)
- `off`: Never send tools (preserves pre-increment behavior)

### FR-002: Tool Format Compatibility
Ollama's native API uses the same tool format as OpenAI. The existing `translateRequest()` from `openai.mjs` already produces the correct format. The Ollama provider must pass `tools` from the translated body to the Ollama request body without additional transformation.

### FR-003: tool_choice Always Stripped
Ollama does not support `tool_choice`. It must be stripped from the request body for ALL Ollama requests, even when tools are being sent. This is a separate concern from the `OLLAMA_TOOLS` setting.

### FR-004: Cache Behavior
The per-model cache is in-memory only (resets on proxy restart). This is intentional -- model capabilities can change when users update models via `ollama pull`, so a fresh start should re-probe. Cache entries are keyed by the exact model name string (e.g., `qwen3-coder:latest`).

## Success Criteria

- Tool-capable Ollama models (Qwen 3 Coder, Llama 3.1+, Mistral) can invoke Claude Code tools through the proxy
- Models that don't support tools fail once then automatically skip tools on subsequent requests
- `OLLAMA_TOOLS=off` preserves exact current behavior for users who prefer no tool overhead
- No regression in non-tool Ollama requests (system prompt condensing, thinking strip, message strip all unchanged)
- Unit test coverage >= 80% for `providers/ollama-tools.mjs` and modified functions in `providers/ollama.mjs`

## Out of Scope

- **Ollama model auto-detection via `/api/show`**: Querying Ollama's model metadata API to pre-check tool support. The auto/cache approach is simpler and works without an extra API call.
- **Tool filtering/limiting**: Sending a subset of tools to reduce token count. This is a separate optimization.
- **Parallel tool calls**: Ollama may return multiple tool_calls; we translate them all. Orchestrating parallel execution is Claude Code's responsibility.
- **tool_choice support**: Ollama does not support this. If it adds support in the future, that's a separate increment.

## Dependencies

- Ollama v0.3.0+ (July 2024) -- required for tool calling support
- Existing `translateRequest()` from `providers/openai.mjs` -- already handles Anthropic-to-OpenAI tool format conversion
- Existing auto-retry logic at `proxy.mjs:485-514` -- will be enhanced with cache writes
