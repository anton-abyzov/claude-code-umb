# Tasks: anymodel Production Readiness

## Task Notation

- `[T-NNN]`: Task ID
- `[P]`: Parallelizable
- `[ ]`: Not started
- `[x]`: Completed

## DevOps Agent Tasks

### T-001: Strip Embedded PAT from Git Remote [P]

**Description**: Fix anymodel git remote URL to remove embedded GitHub token.

**References**: AC-US4-01

**Implementation Details**:
- Run `git remote set-url origin https://github.com/antonoly/anymodel.git`
- Verify with `git remote get-url origin`

**Test Plan**:
- **TC-001**: Git remote URL contains no token
  - Given the anymodel repository
  - When `git remote get-url origin` is run
  - Then the URL is `https://github.com/antonoly/anymodel.git`

**Dependencies**: None
**Status**: [x] Completed

---

### T-002: Move vercel.json to Repo Root [P]

**Description**: Create `vercel.json` at repo root with `outputDirectory: "site"`, delete `site/vercel.json`.

**References**: AC-US1-01, AC-US1-03

**Implementation Details**:
- Create `vercel.json` at repo root: `{"outputDirectory": "site", "cleanUrls": true, "trailingSlash": false}`
- Delete `site/vercel.json`

**Test Plan**:
- **TC-002**: Root vercel.json is valid JSON with correct fields
  - Given the repo root
  - When vercel.json is parsed
  - Then it contains `outputDirectory`, `cleanUrls`, and `trailingSlash`
- **TC-003**: site/vercel.json does not exist
  - Given the site directory
  - When checking for vercel.json
  - Then the file is absent

**Dependencies**: None
**Status**: [x] Completed

---

### T-003: Create CI Test Workflow [P]

**Description**: Create GitHub Actions CI workflow with Node.js 18/20/22 matrix.

**References**: AC-US2-01

**Implementation Details**:
- Create `.github/workflows/ci.yml`
- Trigger on push to main and pull_request to main
- Matrix: node-version [18, 20, 22]
- Steps: checkout, setup-node, run `node --test test/*.test.mjs`
- No npm install needed (zero dependencies)

**Test Plan**:
- **TC-004**: ci.yml is valid YAML
  - Given the workflow file
  - When parsed as YAML
  - Then it parses without errors
- **TC-005**: Workflow triggers on correct events
  - Given the workflow
  - When examining the `on` key
  - Then it includes push to main and pull_request to main

**Dependencies**: None
**Status**: [x] Completed

---

### T-004: Create npm Publish Workflow [P]

**Description**: Create GitHub Actions workflow to publish to npm on release.

**References**: AC-US2-02, AC-US2-03

**Implementation Details**:
- Create `.github/workflows/publish.yml`
- Trigger on release (type: published)
- Steps: checkout, setup-node with registry-url, run tests, npm publish
- Uses `NPM_TOKEN` secret for authentication

**Test Plan**:
- **TC-006**: publish.yml is valid YAML
  - Given the workflow file
  - When parsed as YAML
  - Then it parses without errors
- **TC-007**: Workflow runs tests before publish
  - Given the workflow steps
  - When examining step order
  - Then test step precedes publish step

**Dependencies**: None
**Status**: [x] Completed

---

### T-005: Verify All DevOps Artifacts

**Description**: Validate all created DevOps files are correct.

**References**: AC-US1-01, AC-US2-01, AC-US2-02, AC-US4-01

**Implementation Details**:
- Validate ci.yml and publish.yml as YAML
- Validate vercel.json as JSON
- Confirm git remote URL is clean

**Dependencies**: T-001, T-002, T-003, T-004
**Status**: [x] Completed

---

## Backend Agent Tasks

### T-006: Add GET /health Endpoint [P]

**Description**: Add health check endpoint to proxy.mjs inside createProxy() server callback.

**References**: AC-US3-01

**Implementation Details**:
- Insert health handler BEFORE existing route branches in createProxy()
- Only respond to `GET /health` (other methods fall through)
- Return JSON: `{status: "ok", version: pkg.version, provider: provider.name, model: model || null, uptime: process.uptime(), timestamp: new Date().toISOString()}`
- Set `content-type: application/json` header

**Test Plan**:
- **TC-008**: GET /health returns 200
  - Given a running proxy server
  - When GET /health is requested
  - Then response status is 200
- **TC-009**: Response has correct JSON shape
  - Given a running proxy server
  - When GET /health is requested
  - Then response body contains status, version, provider, model, uptime, timestamp
- **TC-010**: Health response has correct values
  - Given a proxy with openrouter provider
  - When GET /health is requested
  - Then provider is "openrouter" and status is "ok"

**Dependencies**: None
**Status**: [x] Completed

---

### T-007: Create Health Endpoint Tests

**Description**: Create `test/health.test.mjs` with integration tests for the health endpoint.

**References**: AC-US3-02

**Implementation Details**:
- Import `createProxy` from proxy.mjs
- Create mock provider: `{name: "test", buildRequest: () => ({}), displayInfo: () => "test"}`
- Start server on port 0 (random available), read actual port from `server.address().port`
- Make HTTP GET request using `node:http`
- Assert response status, content-type, JSON body fields
- Clean up: `server.close()` in afterEach/after

**Dependencies**: T-006
**Status**: [x] Completed

---

### T-008: Run Full Test Suite

**Description**: Run all tests to verify no regressions.

**References**: AC-US3-03

**Implementation Details**:
- Run `node --test test/*.test.mjs`
- All 40+ existing tests plus new health tests must pass

**Dependencies**: T-006, T-007
**Status**: [x] Completed
