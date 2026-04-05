---
increment: 0001-anymodel-production-readiness
title: anymodel Production Readiness
type: feature
priority: P1
status: completed
created: 2026-04-02T00:00:00.000Z
structure: user-stories
test_mode: TDD
coverage_target: 80
---

# Feature: anymodel Production Readiness

## Overview

Complete anymodel's production readiness: deploy the landing page to Vercel, add CI/CD pipelines via GitHub Actions, implement a health check endpoint in the proxy, and clean up git hygiene (embedded tokens in remotes).

## User Stories

### US-001: Vercel Deployment (P1)
**Project**: anymodel

**As a** visitor to anymodel.dev
**I want** the landing page served via Vercel
**So that** I can learn about and install the tool from a fast, reliable site

**Acceptance Criteria**:
- [x] **AC-US1-01**: Static site from `site/` directory deployed to Vercel with `vercel.json` at repo root using `outputDirectory: "site"`
- [x] **AC-US1-02**: Auto-deploy triggers on push to main branch via Vercel GitHub integration
- [x] **AC-US1-03**: Clean URLs enabled (no `.html` extensions), trailing slashes removed

---

### US-002: CI/CD Pipeline (P1)
**Project**: anymodel

**As a** maintainer
**I want** automated testing and publishing workflows
**So that** every push is validated and releases are published safely to npm

**Acceptance Criteria**:
- [x] **AC-US2-01**: CI workflow at `.github/workflows/ci.yml` runs `node --test test/*.test.mjs` on push to main and PRs, with Node.js 18/20/22 matrix
- [x] **AC-US2-02**: Publish workflow at `.github/workflows/publish.yml` triggers on GitHub release creation, runs tests then `npm publish`
- [x] **AC-US2-03**: Publish workflow requires `NPM_TOKEN` secret and runs tests as a safety gate before publishing

---

### US-003: Health Check Endpoint (P1)
**Project**: anymodel

**As an** operator monitoring the proxy
**I want** a `/health` endpoint
**So that** I can verify the proxy is running and check its configuration

**Acceptance Criteria**:
- [x] **AC-US3-01**: `GET /health` returns HTTP 200 with JSON body containing `status`, `version`, `provider`, `model`, `uptime`, and `timestamp`
- [x] **AC-US3-02**: Health endpoint has dedicated test coverage in `test/health.test.mjs` (server start, request, assert JSON shape, server close)
- [x] **AC-US3-03**: Full test suite (40+ existing tests + new health tests) passes with no regressions

---

### US-004: Git Hygiene (P2)
**Project**: anymodel

**As a** developer
**I want** no credentials in git configuration
**So that** tokens are not leaked through remote URLs

**Acceptance Criteria**:
- [x] **AC-US4-01**: anymodel git remote URL is `https://github.com/antonoly/anymodel.git` with no embedded PAT

## Out of Scope

- Custom domain DNS configuration (Vercel dashboard + registrar step)
- Rate limiting on the proxy
- Additional LLM provider integrations
- npm version bumping strategy

## Dependencies

- Vercel account with GitHub integration
- `NPM_TOKEN` secret configured in GitHub repo settings (for publish workflow)
