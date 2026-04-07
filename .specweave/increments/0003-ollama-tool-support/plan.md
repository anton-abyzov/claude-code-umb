# Implementation Plan: Enable Ollama Tool Calling Passthrough

## Overview

Replace the blanket tool strip at `proxy.mjs:258-265` with capability-aware passthrough. A new module `providers/ollama-tools.mjs` owns the decision logic and per-model cache. The Ollama provider gains tool passthrough in request building and tool_calls handling in both response translation and stream translation. The proxy wires it all together.

## Architecture

### Components

```
proxy.mjs (orchestrator)
  |
  +-- shouldSendTools(model) --> providers/ollama-tools.mjs (decision + cache)
  |
  +-- providers/ollama.mjs     (request: pass tools; response: translate tool_calls)
  |     +-- reuses translateRequest() from openai.mjs
  |
  +-- cacheToolResult(model, bool) <-- called on success/failure
```

- **`providers/ollama-tools.mjs`** (NEW): Stateless decision function + in-memory Map cache. No side effects beyond the cache. Exports: `shouldSendTools(model)`, `cacheToolResult(model, success)`, `clearToolCache()`.
- **`providers/ollama.mjs`** (MODIFIED): `transformRequest()` conditionally includes `tools` in the Ollama body. `ollamaToAnthropic()` handles `message.tool_calls`. `createOllamaStreamTranslator()` handles streamed tool_calls chunks.
- **`proxy.mjs`** (MODIFIED): Replace lines 258-265 with `shouldSendTools()` call. Always strip `tool_choice` for Ollama. Call `cacheToolResult()` on success/failure paths.

### Data Flow

**Request path (Anthropic -> Ollama)**:
1. `proxy.mjs` receives Anthropic-format request with `tools[]` and `tool_choice`
2. `shouldSendTools(model)` decides whether to keep tools
3. `tool_choice` is always deleted (Ollama doesn't support it)
4. If tools kept: `ollama.transformRequest()` includes `tools` in the Ollama body
5. `translateRequest()` (from openai.mjs) converts Anthropic tools to OpenAI/Ollama format

**Response path (Ollama -> Anthropic, non-streaming)**:
1. Ollama returns `{ message: { content, tool_calls: [{function: {name, arguments}}] } }`
2. `ollamaToAnthropic()` maps `tool_calls` -> Anthropic `tool_use` content blocks
3. Sets `stop_reason: "tool_use"` when tool_calls present
4. `proxy.mjs` calls `cacheToolResult(model, true)` on success

**Response path (Ollama -> Anthropic, streaming)**:
1. Ollama streams NDJSON with `message.tool_calls` chunks
2. `createOllamaStreamTranslator()` emits:
   - `content_block_start` with `type: "tool_use"` on first tool_call chunk
   - `content_block_delta` with `type: "input_json_delta"` for argument fragments
   - `content_block_stop` when done
3. Final `message_delta` has `stop_reason: "tool_use"` if any tool_calls were emitted

**Error/retry path**:
1. If Ollama returns error containing "support tool use" -> existing auto-retry fires
2. After retry, `cacheToolResult(model, false)` marks model as incapable
3. Subsequent requests for same model skip tools immediately

### Cache Design

```javascript
// providers/ollama-tools.mjs
const cache = new Map();  // Map<string, boolean>

// OLLAMA_TOOLS env var: 'auto' (default) | 'on' | 'off'
export function shouldSendTools(model) {
  const mode = (process.env.OLLAMA_TOOLS || 'auto').toLowerCase();
  if (mode === 'off') return false;
  if (mode === 'on') return true;
  // auto: check cache, default to true (optimistic)
  if (cache.has(model)) return cache.get(model);
  return true;
}

export function cacheToolResult(model, success) {
  cache.set(model, success);
}

export function clearToolCache() {
  cache.clear();
}
```

- In-memory only (resets on proxy restart) -- intentional for model version changes
- Keyed by exact model string (e.g., `qwen3-coder:latest`)
- Optimistic default: uncached models get tools on first request

## Technology Stack

- **Language**: JavaScript ESM (matches existing codebase)
- **Test Framework**: `node:test` + `node:assert/strict` (matches existing tests)
- **No new dependencies**

**Architecture Decisions**:
- **ADR-0001: Reuse translateRequest, not translateResponse**: Ollama's native response shape (`message.tool_calls` at top level, `done`/`done_reason` fields) differs from OpenAI (`choices[0].message.tool_calls`, `finish_reason`). Reusing `translateRequest()` is safe (both use identical tool format). Reusing `translateResponse()` would require wrapping Ollama responses in OpenAI shape, adding complexity for no benefit.
- **Optimistic cache default**: New models get tools on first request. If they fail, the existing auto-retry path handles it gracefully and caches the failure. This minimizes configuration burden.
- **tool_choice always stripped**: Separate concern from OLLAMA_TOOLS. Even `OLLAMA_TOOLS=on` strips tool_choice because Ollama API rejects it.

## Implementation Phases

### Phase 1: Tool Capability Module (US-001)
- Create `providers/ollama-tools.mjs` with `shouldSendTools()`, `cacheToolResult()`, `clearToolCache()`
- Pure functions + Map cache, fully testable in isolation
- Test file: `test/ollama-tools.test.mjs`

### Phase 2: Ollama Request Passthrough (US-002)
- Modify `ollama.mjs` `transformRequest()` to conditionally include `tools`
- Always exclude `tool_choice` from Ollama body
- Test: verify tools pass through, tool_choice never passes through

### Phase 3: Non-Streaming Response Handling (US-003)
- Modify `ollamaToAnthropic()` to handle `message.tool_calls`
- Map `tool_calls[].function.{name, arguments}` -> `tool_use` blocks
- Set `stop_reason: "tool_use"` when tool_calls present
- Test: text-only, tool-only, mixed text+tool responses

### Phase 4: Streaming Response Handling (US-004)
- Modify `createOllamaStreamTranslator()` to detect and translate tool_calls in NDJSON
- Handle tool_call start (name+id), argument deltas, and completion
- Proper block indexing when mixing text and tool_calls
- Test: streamed tool_calls, mixed text+tool streams

### Phase 5: Proxy Integration (US-005)
- Replace `proxy.mjs` lines 258-265 with `shouldSendTools()` call
- Always strip `tool_choice` for Ollama
- Call `cacheToolResult()` on success path (200 with tool_calls) and failure path (auto-retry "support tool use")
- Descriptive log messages for tool decision
- Test: integration tests for the proxy tool decision flow

## Testing Strategy

- **Unit tests** for `ollama-tools.mjs`: cache behavior, env var modes, edge cases
- **Unit tests** for `ollama.mjs`: request with/without tools, response translation with tool_calls, stream translation with tool_calls
- **All tests use `node:test`** (existing framework, no vitest needed)
- **Coverage target**: 80% for new/modified code
- **TDD**: Write failing tests first per each phase

## Technical Challenges

### Challenge 1: Ollama Streaming Tool Call Format
**Problem**: Ollama streams tool_calls differently from OpenAI. In Ollama NDJSON, each streamed chunk is a full JSON line with `message.tool_calls` array. Unlike OpenAI SSE where tool_calls arrive as deltas with index-based accumulation, Ollama sends complete tool_call objects in each chunk.
**Solution**: The stream translator checks for `message.tool_calls` in each parsed NDJSON line. On first appearance of a tool_call with a new function name, emit `content_block_start`. Arguments are accumulated and emitted as `input_json_delta`. The `done: true` final line triggers `content_block_stop` for all open blocks.
**Risk**: Low -- Ollama's format is simpler than OpenAI's streaming tool format.

### Challenge 2: Cache Correctness Across Model Aliases
**Problem**: Users may reference the same model with different names (e.g., `qwen3-coder` vs `qwen3-coder:latest`). Cache is keyed by exact string.
**Solution**: Accept this limitation. The proxy uses whatever model string the client sends. If a model is called by two names, it gets two cache entries. This is a minor inefficiency, not a correctness issue -- both will converge to the right cache value after first use.
**Risk**: Negligible.

### Challenge 3: Interaction with Existing Ollama Optimizations
**Problem**: The proxy has multiple Ollama-specific optimizations (system prompt condensing, XML stripping, message condensing, thinking strip). Tool passthrough must not interfere.
**Solution**: Tool decision happens at lines 258-265, before all other Ollama optimizations. The optimizations operate on `parsed.system` and `parsed.messages`, not on `parsed.tools`. No interaction.
**Risk**: None -- these are orthogonal concerns operating on different fields.
