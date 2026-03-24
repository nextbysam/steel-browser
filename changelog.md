# Changelog

All changes to this project are documented here. Updated after every step.

## [2026-03-24] Initial Setup

### Added
- Forked steel-dev/steel-browser to nextbysam/steel-browser
- Created `orb-cloud-integration` branch
- Created PLAN.md with full end-to-end 6-phase plan
  - Phase 1: Build & verify Steel Docker image locally
  - Phase 2: Deploy Steel on Orb Cloud (single VM)
  - Phase 3: Build the gateway (~500 LOC Fastify proxy)
  - Phase 4: End-to-end integration testing
  - Phase 5: Production hardening
  - Phase 6: Demo & outreach
- Created changelog.md (this file)

### Status
- Phase 1: Not started
- Next step: 1.1 — Build Docker image locally
