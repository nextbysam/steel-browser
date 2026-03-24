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

## [2026-03-24] Initial Setup

### Added
- Forked steel-dev/steel-browser to nextbysam/steel-browser
- Created `orb-cloud-integration` branch
