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
- Phase 1: Not started
- Next step: 1.1 — Build Docker image locally

## [2026-03-24] Initial Setup

### Added
- Forked steel-dev/steel-browser to nextbysam/steel-browser
- Created `orb-cloud-integration` branch
