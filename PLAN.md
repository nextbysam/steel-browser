# Plan: Replace Steel Browser Runtime with Orb Cloud

## End Goal
Steel Browser sessions run on Orb Cloud VMs instead of local Docker / Fly.io.
Each browser session gets its own isolated Orb VM with Chrome, a public URL,
and persistent filesystem. The Steel SDK works with zero changes (just swap baseUrl).

## Architecture

```
User (Steel SDK / Playwright / Puppeteer)
    │
    ▼
┌────────────────────────────────────────────┐
│  Orb Steel Gateway                          │
│  (Fastify, ~500 LOC, runs on Orb VM)       │
│                                             │
│  • API key auth                             │
│  • Session ID → VM URL routing              │
│  • VM lifecycle (create/health/destroy)     │
│  • Warm pool (pre-provisioned idle VMs)     │
│  • WebSocket CDP proxy                      │
│  • Session timeout & cleanup                │
└─────────────┬───────────────────────────────┘
              │ Orb Cloud API
    ┌─────────┼──────────┐
    ▼         ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│ Orb VM │ │ Orb VM │ │ Orb VM │
│ sess-1 │ │ sess-2 │ │ sess-3 │
│        │ │        │ │        │
│ Steel  │ │ Steel  │ │ Steel  │
│ Chrome │ │ Chrome │ │ Chrome │
│ :3000  │ │ :3000  │ │ :3000  │
└────────┘ └────────┘ └────────┘
```

## Verification Strategy

Every phase has explicit verification steps. A phase is not complete
until ALL verifications pass. No assumptions — prove it works.

---

## Phase 1: Build & Verify Steel Docker Image Locally
**Goal:** Confirm the Docker image builds, runs, and the API works.

- [ ] 1.1 Build Docker image from this repo
  - `docker build -t steel-browser-orb .`
  - **Verify:** Image builds without errors

- [ ] 1.2 Run container locally
  - `docker run -p 3000:3000 -p 9223:9223 steel-browser-orb`
  - **Verify:** `curl http://localhost:3000/v1/health` returns 200

- [ ] 1.3 Create a session via API
  - `curl -X POST http://localhost:3000/v1/sessions`
  - **Verify:** Returns session ID, websocketUrl, debugUrl

- [ ] 1.4 Test scrape endpoint
  - `curl -X POST http://localhost:3000/v1/scrape -H 'Content-Type: application/json' -d '{"url":"https://example.com"}'`
  - **Verify:** Returns HTML content of example.com

- [ ] 1.5 Test screenshot endpoint
  - `curl -X POST http://localhost:3000/v1/screenshot -H 'Content-Type: application/json' -d '{"url":"https://example.com"}' --output screenshot.jpg`
  - **Verify:** screenshot.jpg is a valid image

- [ ] 1.6 Test CDP WebSocket connection
  - Write a small script that connects Puppeteer to `ws://localhost:3000`
  - **Verify:** Can navigate to a page, take screenshot, get title

- [ ] 1.7 Test session release
  - `curl -X POST http://localhost:3000/v1/sessions/{id}/release`
  - **Verify:** Returns success, session is cleaned up

- [ ] 1.8 Test context extraction
  - Navigate to a site with cookies, then GET /v1/sessions/{id}/context
  - **Verify:** Returns cookies and localStorage

---

## Phase 2: Deploy Steel on Orb Cloud (Single VM)
**Goal:** Prove Steel Browser works inside an Orb Cloud VM.

- [ ] 2.1 Create orb.toml for Steel Browser template
  - Define the Orb Cloud configuration for running Steel
  - **Verify:** orb.toml validates against Orb schema

- [ ] 2.2 Deploy Steel to a single Orb VM
  - Use Orb API to create a computer, upload config, build, deploy
  - **Verify:** VM gets a public URL like https://{id}.orbcloud.dev

- [ ] 2.3 Test health endpoint via Orb public URL
  - `curl https://{id}.orbcloud.dev/v1/health`
  - **Verify:** Returns 200

- [ ] 2.4 Test session creation via Orb public URL
  - `curl -X POST https://{id}.orbcloud.dev/v1/sessions`
  - **Verify:** Returns session details

- [ ] 2.5 Test scrape via Orb public URL
  - `curl -X POST https://{id}.orbcloud.dev/v1/scrape -d '{"url":"https://example.com"}'`
  - **Verify:** Returns HTML

- [ ] 2.6 Test CDP WebSocket via Orb public URL
  - Connect Puppeteer to `wss://{id}.orbcloud.dev`
  - **Verify:** Can navigate, screenshot, get page title

- [ ] 2.7 Test file upload/download via Orb
  - Upload a file, list files, download it back
  - **Verify:** Round-trip file integrity

- [ ] 2.8 Measure performance
  - Time: VM startup, session creation, page navigation, screenshot
  - **Verify:** Document baseline numbers

---

## Phase 3: Build the Gateway
**Goal:** A thin proxy that creates Orb VMs per session.

- [ ] 3.1 Scaffold gateway project
  - Fastify + WebSocket + http-proxy
  - **Verify:** `npm run dev` starts without errors

- [ ] 3.2 Implement OrbClient
  - createVM(), waitForReady(), destroyVM(), getVM()
  - **Verify:** Unit test — can create and destroy an Orb VM

- [ ] 3.3 Implement SessionManager
  - register(), get(), remove(), listByApiKey(), cleanupExpired()
  - **Verify:** Unit test — session CRUD and timeout cleanup

- [ ] 3.4 Implement WarmPool
  - initialize(), take(), returnOrDestroy(), maintain()
  - **Verify:** Unit test — pool fills to min, take reduces, maintain refills

- [ ] 3.5 Implement POST /v1/sessions route
  - Take from pool or provision → forward to VM → return rewritten URLs
  - **Verify:** Create session via gateway, get back valid session

- [ ] 3.6 Implement session proxy routes
  - GET /v1/sessions/:id, GET /v1/sessions/:id/context
  - **Verify:** Can read session details and context through gateway

- [ ] 3.7 Implement POST /v1/sessions/:id/release
  - Forward release → destroy VM → cleanup mapping
  - **Verify:** Session released, VM destroyed, mapping cleaned

- [ ] 3.8 Implement WebSocket CDP proxy
  - Route /cdp/:sessionId → VM's CDP WebSocket
  - **Verify:** Puppeteer connects through gateway, navigates pages

- [ ] 3.9 Implement stateless actions
  - /v1/scrape, /v1/screenshot, /v1/pdf, /v1/search
  - **Verify:** All return correct results through gateway

- [ ] 3.10 Implement API key auth
  - Bearer token validation on all routes
  - **Verify:** Requests without key get 401, with key get through

- [ ] 3.11 Implement health endpoint
  - GET /v1/health returns session stats + pool stats
  - **Verify:** Returns correct counts

---

## Phase 4: End-to-End Integration Testing
**Goal:** Prove the full flow works reliably.

- [ ] 4.1 Deploy gateway on Orb Cloud
  - Gateway itself runs on an Orb VM
  - **Verify:** Gateway accessible at public URL

- [ ] 4.2 Test: Steel Node SDK → Gateway → Orb VM
  - Use `steel-sdk` with `baseUrl` pointed at gateway
  - Create session, navigate, scrape, screenshot, release
  - **Verify:** All operations succeed

- [ ] 4.3 Test: Playwright → Gateway → Orb VM (CDP)
  - Connect Playwright via CDP through gateway
  - Navigate, interact, screenshot
  - **Verify:** Full browser control works

- [ ] 4.4 Test: browser-use → Gateway → Orb VM
  - Run browser-use agent through gateway
  - **Verify:** Agent completes a multi-step web task

- [ ] 4.5 Test: concurrent sessions
  - Create 5 sessions simultaneously
  - Run different tasks on each
  - **Verify:** All 5 sessions work independently, no cross-contamination

- [ ] 4.6 Test: session timeout
  - Create session, wait beyond timeout
  - **Verify:** Session cleaned up, VM destroyed automatically

- [ ] 4.7 Test: warm pool refill
  - Take all VMs from pool, verify pool refills
  - **Verify:** Pool size returns to minIdle within 2 minutes

- [ ] 4.8 Test: error recovery
  - Kill a VM mid-session
  - **Verify:** Gateway detects failure, cleans up mapping, returns error

- [ ] 4.9 Stress test
  - Create 20 sessions sequentially, release each
  - **Verify:** No VM leaks (all VMs destroyed), no memory leaks in gateway

- [ ] 4.10 Document results
  - Performance numbers, cost per session, reliability metrics
  - **Verify:** Results written to RESULTS.md

---

## Phase 5: Production Hardening
**Goal:** Make it production-ready.

- [ ] 5.1 Add request logging with correlation IDs
- [ ] 5.2 Add Prometheus metrics endpoint
- [ ] 5.3 Add rate limiting per API key
- [ ] 5.4 Add concurrent session limit per API key
- [ ] 5.5 Add VM memory watchdog (restart Chrome on high memory)
- [ ] 5.6 Add context persistence (save/restore between sessions)
- [ ] 5.7 Add proxy sidecar support per VM
- [ ] 5.8 Write deployment documentation
- [ ] 5.9 Create orb.toml for the gateway itself

---

## Phase 6: Demo & Outreach
**Goal:** Ship it, show it, get users.

- [ ] 6.1 Record demo video (session create → browse → release)
- [ ] 6.2 Write blog post: "Run Steel Browser on Orb Cloud — 10x cheaper"
- [ ] 6.3 Post on Hacker News (Show HN)
- [ ] 6.4 Open issue on steel-dev/steel-browser proposing integration
- [ ] 6.5 DM Steel team with demo
- [ ] 6.6 Contact browser-use team
- [ ] 6.7 Offer free credits to first 10 users

---

## Decision Log

All architectural decisions will be documented here as they're made.

| Date | Decision | Reasoning |
|------|----------|-----------|
| 2026-03-24 | 1 VM per session (not multi-session per VM) | Steel's architecture is 1 session per container. Working with the grain, not against it. Also provides perfect isolation. |
| 2026-03-24 | Gateway is a separate service, not a Steel fork | Keeps Steel untouched. Gateway is a thin proxy. Easier to maintain. |
| 2026-03-24 | Warm pool with min 2 idle VMs | Balance between startup latency and cost. 2 VMs idle = ~$3/month. |
| 2026-03-24 | Fastify for gateway | Same framework as Steel (consistency). Fast, WebSocket support built-in. |
