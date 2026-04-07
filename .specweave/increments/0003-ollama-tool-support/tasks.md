# Tasks: Enable Ollama Tool Calling Passthrough

## Task Notation

- `[T###]`: Task ID
- `[P]`: Parallelizable
- `[ ]`: Not started
- `[x]`: Completed
- Model hints: haiku (simple), opus (default)

## Phase 1: Tool Capability Module

### US-001: Tool Capability Configuration Module (P1)

#### T-001: Create providers/ollama-tools.mjs

**User Story**: US-001 | **Satisfies ACs**: AC-US1-01, AC-US1-02, AC-US1-03, AC-US1-04, AC-US1-05 | **Status**: [x] completed

**Description**: Create new module `providers/ollama-tools.mjs` with tool capability decision logic and per-model cache.

**AC**: AC-US1-01, AC-US1-02, AC-US1-03, AC-US1-04, AC-US1-05

**Implementation Details**:
- Create `providers/ollama-tools.mjs` with `shouldSendTools(model)`, `cacheToolResult(model, success)`, `clearToolCache()`
- Use in-memory `Map<string, boolean>` for cache
- Read `OLLAMA_TOOLS` env var: `off` -> false, `on` -> true, `auto`/default -> check cache, optimistic true for uncached

**Test Plan**:
- **File**: `test/ollama-tools.test.mjs`
- **Tests**:
  - **TC-001**: shouldSendTools returns true for uncached model in auto mode
    - Given OLLAMA_TOOLS is unset (defaults to auto)
    - When shouldSendTools("qwen3-coder") is called
    - Then returns true (optimistic default)
  - **TC-002**: shouldSendTools returns false when OLLAMA_TOOLS=off
    - Given OLLAMA_TOOLS="off"
    - When shouldSendTools("any-model") is called
    - Then returns false regardless of cache
  - **TC-003**: shouldSendTools returns true when OLLAMA_TOOLS=on
    - Given OLLAMA_TOOLS="on" and model is cached as incapable
    - When shouldSendTools(model) is called
    - Then returns true (override cache)
  - **TC-004**: cacheToolResult stores capability and shouldSendTools respects it
    - Given OLLAMA_TOOLS=auto
    - When cacheToolResult("model-a", false) is called
    - Then shouldSendTools("model-a") returns false
  - **TC-005**: cacheToolResult(model, true) marks model as capable
    - Given OLLAMA_TOOLS=auto
    - When cacheToolResult("model-b", true) is called
    - Then shouldSendTools("model-b") returns true
  - **TC-006**: clearToolCache resets all entries
    - Given cache has entries
    - When clearToolCache() is called
    - Then shouldSendTools returns true for previously cached-false models

**Dependencies**: None

---

## Phase 2: Ollama Request Passthrough

### US-002: Tool Request Passthrough (P1)

#### T-002: Modify ollama.mjs transformRequest to pass tools

**User Story**: US-002 | **Satisfies ACs**: AC-US2-01, AC-US2-02 | **Status**: [x] completed

**Description**: Modify `ollama.mjs` `transformRequest()` to include `tools` from the translated OpenAI body in the Ollama request. Always exclude `tool_choice`.

**AC**: AC-US2-01, AC-US2-02

**Implementation Details**:
- In `transformRequest()`, after building `ollamaBody`, copy `openaiBody.tools` to `ollamaBody.tools` if present
- Never copy `tool_choice` -- Ollama does not support it
- The `translateRequest()` from openai.mjs already converts Anthropic tools to the correct format

**Test Plan**:
- **File**: `test/ollama.test.mjs`
- **Tests**:
  - **TC-007**: transformRequest includes tools when present
    - Given Anthropic body with tools array
    - When transformRequest is called
    - Then Ollama body contains tools array with OpenAI function format
  - **TC-008**: transformRequest never includes tool_choice
    - Given Anthropic body with tool_choice
    - When transformRequest is called
    - Then Ollama body does NOT contain tool_choice
  - **TC-009**: transformRequest works without tools
    - Given Anthropic body without tools
    - When transformRequest is called
    - Then Ollama body has no tools field (no regression)

**Dependencies**: None

---

## Phase 3: Non-Streaming Response Handling

### US-003: Tool Response Handling -- Non-Streaming (P1)

#### T-003: Modify ollamaToAnthropic to handle tool_calls

**User Story**: US-003 | **Satisfies ACs**: AC-US3-01, AC-US3-02, AC-US3-03 | **Status**: [x] completed

**Description**: Modify `ollamaToAnthropic()` in `ollama.mjs` to translate `message.tool_calls` to Anthropic `tool_use` content blocks.

**AC**: AC-US3-01, AC-US3-02, AC-US3-03

**Implementation Details**:
- Check `msg.tool_calls` array in Ollama response
- For each tool_call: create `{ type: 'tool_use', id: tc.function.name + '_' + index (or tc.id if present), name: tc.function.name, input: JSON.parse(tc.function.arguments) }`
- If tool_calls present, set `stop_reason: 'tool_use'` instead of mapping from `done_reason`
- Handle mixed content + tool_calls: text block first, then tool_use blocks

**Test Plan**:
- **File**: `test/ollama.test.mjs`
- **Tests**:
  - **TC-010**: ollamaToAnthropic translates tool_calls to tool_use blocks
    - Given Ollama response with `message.tool_calls: [{function: {name: "Read", arguments: '{"file":"/a.ts"}'}}]`
    - When ollamaToAnthropic is called
    - Then content includes `{ type: 'tool_use', name: 'Read', input: { file: '/a.ts' } }`
  - **TC-011**: stop_reason is tool_use when tool_calls present
    - Given Ollama response with tool_calls
    - When ollamaToAnthropic is called
    - Then stop_reason is "tool_use"
  - **TC-012**: mixed text + tool_calls produces both content types
    - Given Ollama response with content "Let me check" AND tool_calls
    - When ollamaToAnthropic is called
    - Then content has text block first, then tool_use block(s)
  - **TC-013**: text-only response unchanged (regression)
    - Given Ollama response with only text content
    - When ollamaToAnthropic is called
    - Then content is `[{ type: 'text', text: '...' }]` and stop_reason is 'end_turn'

**Dependencies**: None

---

## Phase 4: Streaming Response Handling

### US-004: Tool Streaming Handling (P2)

#### T-004: Modify createOllamaStreamTranslator to handle tool_calls

**User Story**: US-004 | **Satisfies ACs**: AC-US4-01, AC-US4-02, AC-US4-03, AC-US4-04, AC-US4-05 | **Status**: [x] completed

**Description**: Modify `createOllamaStreamTranslator()` in `ollama.mjs` to detect and translate streamed tool_calls in NDJSON chunks to Anthropic SSE events.

**AC**: AC-US4-01, AC-US4-02, AC-US4-03, AC-US4-04, AC-US4-05

**Implementation Details**:
- Track tool call state: `toolCallsEmitted` flag, `toolBlockIndices` map
- When `parsed.message.tool_calls` appears in a chunk:
  - For each tool_call: if new (first time seeing this index), emit `content_block_start` with `type: "tool_use"`, `id`, `name`
  - Emit `content_block_delta` with `type: "input_json_delta"` for function arguments
- When `parsed.done === true`:
  - Emit `content_block_stop` for all open blocks (text and tool)
  - Set `stop_reason: "tool_use"` if any tool_calls were emitted
- Close text block before starting tool blocks (text and tool_calls in same stream)

**Test Plan**:
- **File**: `test/ollama.test.mjs`
- **Tests**:
  - **TC-014**: stream translator emits tool_use content_block_start for tool_calls
    - Given NDJSON chunk with `message.tool_calls: [{function: {name: "Bash"}}]`
    - When translator.transform is called
    - Then output includes content_block_start with type "tool_use" and name "Bash"
  - **TC-015**: stream translator emits input_json_delta for tool arguments
    - Given NDJSON chunk with tool_call arguments
    - When translator.transform is called
    - Then output includes content_block_delta with type "input_json_delta"
  - **TC-016**: stream translator sets stop_reason tool_use on done
    - Given final NDJSON chunk with done:true after tool_calls
    - When translator.transform is called
    - Then message_delta has stop_reason "tool_use"
  - **TC-017**: stream translator handles mixed text + tool_calls
    - Given chunks with text content followed by tool_calls
    - When translator processes all chunks
    - Then text block is properly closed before tool blocks start, block indices are correct
  - **TC-018**: text-only streaming unchanged (regression)
    - Given NDJSON chunks with only text content
    - When translator processes chunks
    - Then output matches existing behavior exactly

**Dependencies**: T-001

---

## Phase 5: Proxy Integration

### US-005: Proxy Capability-Aware Tool Logic (P1)

#### T-005: Replace blanket tool strip with shouldSendTools

**User Story**: US-005 | **Satisfies ACs**: AC-US5-01, AC-US5-02, AC-US5-03, AC-US5-04, AC-US5-05, AC-US5-06 | **Status**: [x] completed

**Description**: Replace `proxy.mjs` lines 258-265 (blanket tool strip) with capability-aware logic using `shouldSendTools()`. Wire up `cacheToolResult()` on success and failure paths.

**AC**: AC-US5-01, AC-US5-02, AC-US5-03, AC-US5-04, AC-US5-05, AC-US5-06

**Implementation Details**:
- Import `shouldSendTools`, `cacheToolResult` from `./providers/ollama-tools.mjs`
- Replace lines 258-265:
  ```javascript
  if (provider.name === 'ollama' && parsed.tools?.length) {
    if (!shouldSendTools(parsed.model)) {
      console.log(`...stripped...`);
      delete parsed.tools;
    } else {
      console.log(`...sending N tools...`);
    }
    // Always strip tool_choice for Ollama
    delete parsed.tool_choice;
  }
  ```
- On success path (200 response): if response contains tool_calls, call `cacheToolResult(model, true)`
- On auto-retry path (error "support tool use"): after retry, call `cacheToolResult(model, false)`
- Log messages: indicate decision reason (cache hit, OLLAMA_TOOLS setting, first-time optimistic)

**Test Plan**:
- **File**: `test/ollama-tools.test.mjs` (additional integration-style tests)
- **Tests**:
  - **TC-019**: proxy strips tools when shouldSendTools returns false
    - Given OLLAMA_TOOLS=off and Ollama provider
    - When request with tools arrives
    - Then tools and tool_choice are deleted from parsed body
  - **TC-020**: proxy passes tools when shouldSendTools returns true
    - Given OLLAMA_TOOLS=auto and uncached model
    - When request with tools arrives
    - Then tools remain in parsed body
  - **TC-021**: tool_choice always stripped even when tools pass through
    - Given OLLAMA_TOOLS=on and request with tools + tool_choice
    - When request is processed
    - Then tools present but tool_choice deleted

**Dependencies**: T-001, T-002, T-003, T-004

---

## Phase 6: Verification

#### T-006: Run full test suite

**User Story**: All | **Satisfies ACs**: All | **Status**: [x] completed

**Description**: Run all existing and new tests to verify no regressions.

**Test Plan**:
- Run `node --test test/ollama-tools.test.mjs test/ollama.test.mjs`
- Run `node --test test/*.test.mjs` (full suite)
- Verify 0 failures

**Dependencies**: T-001, T-002, T-003, T-004, T-005
