# Implementation Plan: anymodel Production Readiness

## Overview

Surgical additions to the existing anymodel project: move Vercel config, create 2 GitHub Actions workflows, add a health endpoint to the proxy server, and fix git remote hygiene. Zero new dependencies. All work happens in the anymodel repo at `repositories/antonoly/anymodel/`.

## Architecture Decisions

### ADR-001: Vercel Config at Repo Root
Move `vercel.json` from `site/` to repo root with `"outputDirectory": "site"`. This keeps deployment config in version control rather than relying on Vercel dashboard settings. Delete `site/vercel.json` to avoid confusion.

### ADR-002: Health Endpoint Placement
Insert `GET /health` handler in `proxy.mjs` inside the `createProxy()` server callback, BEFORE the existing route branches (provider routing and Anthropic passthrough). This ensures health checks never touch external services. Returns JSON with `{status, version, provider, model, uptime, timestamp}`.

### ADR-003: Separate CI Workflows
Two workflow files: `ci.yml` (test gate on push/PR) and `publish.yml` (npm publish on release). Separate triggers and separate secrets requirements justify separate files. No `npm install` step needed since the project has zero dependencies.

## Technology Stack

- **Runtime**: Node.js >=18.0.0 (built-in test runner, ES modules)
- **CI**: GitHub Actions (ubuntu-latest, Node 18/20/22 matrix)
- **Hosting**: Vercel (static site deployment)
- **Testing**: `node:test` + `node:assert/strict` (zero dependencies)

## Implementation Phases

### Phase 1: DevOps + Backend (Parallel)
**DevOps agent**: Git hygiene, Vercel config, CI/CD workflows
**Backend agent**: Health endpoint + tests

No cross-dependencies between agents. All file ownership is non-overlapping.

### Phase 2: Verification
Run full test suite. Validate YAML/JSON. Verify git remote URLs clean.

## File Ownership

| Agent | WRITE Patterns |
|-------|---------------|
| DevOps | `.github/**`, `vercel.json` (root), `site/vercel.json` (delete) |
| Backend | `proxy.mjs`, `test/**` |

## Testing Strategy

- Health endpoint: Integration test using `node:http` client against real server on port 0 (random available)
- `createProxy()` returns the server object, enabling `server.close()` cleanup in tests
- All 40+ existing tests must continue passing
