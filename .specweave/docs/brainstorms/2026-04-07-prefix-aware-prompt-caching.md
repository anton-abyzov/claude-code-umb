# Brainstorm: Prefix-Aware Prompt Caching for AnyModel
**Date**: 2026-04-07 | **Depth**: deep | **Lens**: Default (6 orientations) | **Status**: complete

## Problem Frame
**Statement**: AnyModel proxy claims prompt caching but only passes through cache_control headers. For Ollama, every request reprocesses 100KB+ of system prompt + tools from scratch despite Ollama's implicit KV prefix cache that could deliver 17.7x speedups with byte-stable prefixes.

| Dimension | Answer |
|-----------|--------|
| **Who** | Developers using AnyModel + Ollama for local AI coding |
| **What** | Implement real prefix caching maximizing Ollama's KV reuse + cache metrics |
| **When** | v1.9.0 release |
| **Where** | proxy.mjs, providers/ollama.mjs, new cache module |
| **Why** | 2-30s first-request latency; 17.7x speedup available with stable prefixes |
| **How** | Hybrid: deterministic prefix serialization + synthetic cache metrics |

## Approaches

### Approach A: Deterministic Output Stabilization
**Source**: Conservative | **Effort**: Low
**Summary**: Audit and fix non-determinism in existing condensing functions. No new architecture.
**Strengths**: Zero risk, zero new code, immediate benefit
**Risks**: Limited — only fixes what's broken, no observability

### Approach B: Direct llama.cpp Bypass with Slot Pinning
**Source**: Bold | **Effort**: Medium
**Summary**: Bypass Ollama entirely, talk to llama-server with cache_prompt=true and slot pinning.
**Strengths**: Maximum cache performance, explicit control
**Risks**: Requires separate server, violates zero-dep philosophy, version coupling

### Approach C: Deterministic Prefix Serialization Cache
**Source**: Speed | **Effort**: Low
**Summary**: Cache serialized prefix bytes keyed by hash. Reuse exact bytes on cache hit.
**Strengths**: Simple, direct, works with existing Ollama
**Risks**: Date injection breaks hash, no observability

### Approach D: Provider-Agnostic Cache Hint Layer
**Source**: Extensibility | **Effort**: Medium
**Summary**: CacheStrategy interface per provider. Extensible but over-engineered for current scope.
**Strengths**: Future-proof, supports vLLM/Gemini/etc
**Risks**: Over-engineering, breaks zero-dep simplicity

### Approach E: Prepared Statement Caching
**Source**: Lateral | **Effort**: Low
**Summary**: Pre-warm KV cache on startup with canonical prefix. Normalize all requests to match.
**Strengths**: Zero latency on first real request
**Risks**: Ollama may evict between warmup and use

### Approach F: Hybrid Prefix Stabilization + Synthetic Metrics (SELECTED)
**Source**: Hybrid | **Effort**: Low
**Summary**: Combines A (stabilization) + C (serialization cache) + metrics. Provider-aware routing: stabilize for Ollama, passthrough for OpenRouter/OpenAI. Emit synthetic cache_read_input_tokens.

## Evaluation Matrix

| Criterion | A | B | C | D | E | **F** |
|-----------|:-:|:-:|:-:|:-:|:-:|:---:|
| Simplicity | 5 | 2 | 4 | 3 | 4 | **4** |
| Speed-to-deliver | 5 | 2 | 4 | 3 | 4 | **4** |
| Safety | 5 | 2 | 4 | 4 | 4 | **4** |
| Extensibility | 2 | 3 | 3 | 5 | 2 | **4** |
| Alignment | 5 | 2 | 4 | 2 | 4 | **5** |
| Cache effectiveness | 2 | 5 | 4 | 3 | 3 | **4** |
| **Total** | 24 | 16 | 23 | 20 | 21 | **25** |

## Recommendation
**Selected**: Approach F — Hybrid Prefix Stabilization + Synthetic Metrics
**Rationale**: Best total score. Combines deterministic prefix caching with observability, fits existing zero-dep architecture, ~60-80 lines of new code. Works across all providers.
**Caveats**: Prefix overlap estimation is heuristic. Ollama KV eviction is opaque.

## Deep Analysis

### Abstraction Ladder
- **Zoom OUT**: Making local AI coding feel as fast as cloud
- **Current**: Deterministic prefix + synthetic metrics
- **Zoom IN**: PrefixCache class → wire into proxy.mjs → inject metrics into response

### Analogies
1. HTTP ETag: hash response bytes, return 304 on match
2. DB prepared statements: compile query plan once, reuse
3. CPU L1 cache: tight loops stay hot

### Pre-Mortem
| Failure Mode | Likelihood | Impact | Mitigation |
|---|:---:|:---:|---|
| Date changes invalidate hash | High | Med | Normalize/exclude date from hash |
| Ollama KV evicted silently | Med | High | Metrics show 0 hits → user alerted |
| Synthetic metrics mislead | Med | Med | Log warning on cache drop |
| Tool list changes mid-session | Low | Low | Hash miss → auto-recovers |

## Next Steps
- `sw:increment` with selected approach F
- Implementation: ~60-80 lines new code, PrefixCache class + proxy.mjs wiring + response metrics injection
