# Changelog

All changes to this project are documented here. Updated after every step.

## [2026-03-24] Plan Rewrite — Focus on Real User Pain

### Changed
- Rewrote PLAN.md from scratch. Old plan was "run Steel on Orb for cheaper"
  (not unique). New plan solves three unsolved problems in Steel's community:
  1. Multi-session orchestration (issue #263, #72 — users literally leaving)
  2. Auth for self-hosted (issue #235 — zero auth in OSS)
  3. Session persistence (Cloud-only feature, no OSS equivalent)
- Renamed from "gateway" to "orchestrator" — it's not just a proxy, it's
  the missing orchestration layer Steel promised in April 2025 and never shipped.

### Research Findings
- Issue #263: user migrated to raw Puppeteer due to 1-session limit
- Issue #72: maintainer confirmed OSS is "designed for one session at a time"
- Issue #144: maintainer said scaling is "our own custom Cloud implementation"
- Issue #235: no auth whatsoever in self-hosted
- Issue #245: :latest Docker images broke session release, no version tags
- Issues #222, #128: live viewer UI broken for self-hosted
- browser-use issues #97, #163: Steel integration is "extremely slow"

### Status
- Phase 1: Skipped (no Docker runtime on machine — will verify on Orb directly)
- Phase 3: Steps 3.1-3.15 COMPLETE — full orchestrator built and compiles

## [2026-03-24] Phase 3: Orchestrator Built (Steps 3.1-3.10)

### Added
- `orchestrator/` — complete multi-session orchestrator for Steel Browser
  - `src/config.ts` — environment-based configuration
  - `src/orb-client.ts` — Orb Cloud VM lifecycle (create/destroy/health)
  - `src/session-router.ts` — session→VM mapping, multi-session support,
    per-key limits, timeout cleanup, health monitoring
  - `src/warm-pool.ts` — pre-provisioned idle VMs for fast session starts
  - `src/context-store.ts` — session persistence (save/restore cookies,
    localStorage across sessions)
  - `src/auth.ts` — API key auth middleware (solves issue #235)
  - `src/routes.ts` — full Steel API proxy (sessions, scrape, screenshot,
    pdf, search, files, CDP WebSocket)
  - `src/index.ts` — entry point with graceful shutdown

### Technical Details
- 0 TypeScript errors, 0 dependencies vulnerabilities
- 98 packages installed
- Compiles with `tsc --noEmit` cleanly
- Same API as Steel — existing SDKs work with just baseUrl swap
- New endpoints: POST with `restoreSessionId`, GET /v1/orchestrator/health,
  GET /v1/orchestrator/contexts
- Graceful shutdown: releases all sessions, destroys VMs, saves contexts

### Decisions Made
- Skipped Phase 1-2 (no Docker runtime available). Will verify on Orb directly.
- Built orchestrator first since it's the core value.
- Used `ws` package for CDP WebSocket proxy (not http-proxy) for better control.

## [2026-03-24] Orchestrator Deployed on Orb Cloud

### Deployed
- Steel Orchestrator running live at **https://0669167d.orbcloud.dev**
- Computer ID: `0669167d-c551-4370-8082-410523ee5653`
- Health endpoint working: `GET /v1/health` → `{"status":"ok"}`
- Extended health: `GET /v1/orchestrator/health` → sessions, warm pool, uptime stats
- Agent stays alive (agent_count: 1) — confirmed fix from Orb team

### Issues Found & Resolved with Orb Team
1. Agent process dying → entry point CWD was wrong, fixed by Orb
2. DNS/npm ci failing → subnet collision on computer, fixed by Orb
3. [agent.env] via exec → env vars are injected into agent process, not exec shell (by design)
4. Exec timeout → 10s by design, use build steps for long commands
5. ${VAR} syntax → resolves from org secrets at deploy, use literal values instead

### Current State
- Orchestrator: LIVE on Orb Cloud, health endpoints working
- Session creation: fails because Steel Browser VMs need Chromium (not available via Orb build steps)
- Next: need a way to run Steel Browser Docker image on Orb, or use an external compute provider for the browser VMs

## [2026-03-24] Orchestrator README

### Added
- `orchestrator/README.md` — full documentation with:
  - Quick start guide
  - Usage examples (Steel SDK, Puppeteer, session persistence, concurrency)
  - API reference (standard Steel endpoints + new orchestrator endpoints)
  - Configuration reference
  - Architecture diagram
  - "Why not K8s" comparison table

## [2026-03-24] Initial Setup

### Added
- Forked steel-dev/steel-browser to nextbysam/steel-browser
- Created `orb-cloud-integration` branch

## [2026-03-25] Checkpoint/Restore Test with Chrome

### Tested
- Steel Browser running with active session, 8 cookies (Google, Wikipedia), localStorage
- Attempted CRIU checkpoint via `POST /v1/computers/{id}/agents/demote`
- **Result: CRIU dump FAILED**
  - Error: "CRIU namespaced dump failed for PID"
  - Chrome's multi-process architecture (browser + renderer + GPU processes)
    uses shared memory, /dev/shm, and complex IPC that CRIU can't snapshot

### Implications
- Chrome does NOT survive checkpoint/restore on Orb (as of today)
- The "browser sessions that sleep for free" pitch needs adjustment
- Orb's checkpoint/restore works for simple Node.js processes, but not Chrome
- This is a known CRIU limitation with multi-process apps

### Steel Browser Still Works
- Steel survived the failed checkpoint attempt (still running, health OK)
- Sessions, scraping, screenshots all still functional

### Revised Strategy
- Checkpoint/restore is NOT the killer feature for browser use cases
- Focus on: multi-session orchestration, auth, persistence, and cost savings
- The 10-60x cost savings from Orb's pricing model is still real value
- Session context can be saved/restored via Steel's context API (our ContextStore)

## [2026-03-25] Multi-Session Proven — Two Steel VMs on Orb

### Verified
- **VM 1** (8bcd9357): Steel Browser + Chrome, health OK, scrape Google OK, sessions working
- **VM 2** (0a4365b7): Steel Browser + Chrome, health OK, independent sessions
- Both running simultaneously with different session IDs
- Complete isolation — separate Chrome instances, separate session state

### Build Workaround Documented
The getcwd bug in Orb's build step runner causes npm to exit 1.
Workaround: build step installs Chrome via apt-get, then use exec endpoint to:
1. `git checkout package.json` (restore if sed damaged it)
2. Node script to replace only `scripts.prepare` (not the husky dep)
3. `npm install --workspace=api` (with native modules)
4. `npm run build --workspace=api`

### CRIU/Checkpoint Result
- Chrome CANNOT be checkpointed by CRIU (multi-process, shared memory)
- The "sleep for free" feature does NOT work for browser sessions
- Session persistence must use Steel's context API (save/restore cookies)
