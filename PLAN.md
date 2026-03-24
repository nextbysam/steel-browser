# Plan: Steel Browser Multi-Session Orchestrator (Powered by Orb Cloud)

## The Problem Nobody Is Solving

Steel Browser OSS has 6.7K stars but THREE critical gaps that are driving
users away. Steel deliberately gatekeeps these behind their Cloud product.
No open-source tool fills this gap.

### Gap 1: 1 Session Per Container (Users Are Leaving)

Steel OSS is **hardcoded** to one browser session per container.
Issue #263 — user migrated to raw Puppeteer because of this.
Issue #72 — "Why does launching a new session shut down the previous one?"
Maintainer response: "documentation on orchestration coming soon" (April 2025).
It's March 2026. Still nothing.

### Gap 2: No Production Self-Hosting Story

- No API key auth (issue #235 — anyone can control your browser)
- No versioned Docker images (issue #245 — :latest broke session release)
- No cluster support (issue #144 — "we have our own custom implementation for Cloud")
- Broken live viewer UI on self-hosted (issues #222, #128, #249)
- No health checks, no monitoring, no auto-scaling

### Gap 3: No Session Persistence

- Cookies, localStorage, credentials don't survive container restarts
- Persistent profiles = Cloud-only feature
- Users want "infinite long-running sessions" and "user-data-dir persistence"
- browser-use users can't download files from remote Steel sessions (issue #3136)

## What We're Building

An **open-source multi-session orchestrator** for Steel Browser that runs on
Orb Cloud. It solves all three gaps:

```
                   ┌─────────────────────────────────────┐
                   │  Steel Orchestrator (open source)    │
User ─────────────▶│                                     │
                   │  ✓ Multi-session (unlimited)        │
                   │  ✓ API key auth                     │
                   │  ✓ Session persistence (S3/disk)    │
                   │  ✓ Auto-scaling via Orb Cloud       │
                   │  ✓ Health monitoring                │
                   │  ✓ Session affinity routing         │
                   │  ✓ Works with Steel SDK (baseUrl)   │
                   └────────┬──────────┬─────────────────┘
                            │          │
                   ┌────────▼──┐  ┌────▼───────┐
                   │ Orb VM    │  │ Orb VM     │  ... N VMs
                   │ session-A │  │ session-B  │
                   │ Chrome    │  │ Chrome     │
                   │ isolated  │  │ isolated   │
                   └───────────┘  └────────────┘
```

## Why This Can't Be Replicated Easily

1. **Orb Cloud's per-VM pricing** makes 1-VM-per-session economically viable
   ($0.03-0.04/hr vs $3,500/mo for self-hosted K8s at 50 sessions)
2. **Scale-to-zero** — VMs only run when sessions are active. Competitors
   charge for idle time.
3. **Public URLs per VM** — each session gets its own URL. No session
   affinity hacking needed.
4. **Persistent filesystem** — session state survives. E2B kills at 24hrs.

## Value Proposition

| For Steel OSS Users | What They Get |
|---------------------|---------------|
| "I need concurrent sessions" (issue #263) | Unlimited sessions, each isolated |
| "I need auth" (issue #235) | API key auth on all endpoints |
| "I need scaling" (issue #144) | Auto-scaling via Orb, zero K8s |
| "I need persistent sessions" | Context saved to S3, restore anytime |
| "browser-use is slow on Steel Cloud" (issue #97) | Dedicated VM per session = no sharing |
| "Session release is broken" (issue #245) | VM destruction = guaranteed cleanup |

| For Steel The Company | What They Get |
|------------------------|---------------|
| Users who can't afford Cloud | Now on Orb instead of leaving entirely |
| Community goodwill | Open-source orchestration they promised but never shipped |
| Lower support burden | Fewer "how do I scale" issues |

---

## Phases

### Phase 1: Verify Steel Docker Image Works (Baseline)
**Goal:** Build, run, test every endpoint. Establish what works and what doesn't.
**Time:** 1-2 days

- [ ] 1.1 Build Docker image
  - `docker build -t steel-browser-orb .`
  - **Verify:** builds without errors
  - **Commit:** "verify: Docker image builds successfully"

- [ ] 1.2 Run and test health
  - `docker run -d -p 3000:3000 -p 9223:9223 steel-browser-orb`
  - **Verify:** `curl localhost:3000/v1/health` → 200
  - **Commit:** "verify: health endpoint works"

- [ ] 1.3 Test session lifecycle
  - Create session → scrape → screenshot → context → release
  - **Verify:** all return expected responses
  - **Commit:** "verify: full session lifecycle works locally"

- [ ] 1.4 Test CDP WebSocket
  - Write `tests/cdp-test.ts` — Puppeteer connects, navigates, screenshots
  - **Verify:** screenshot saved, page title matches
  - **Commit:** "verify: CDP WebSocket works via Puppeteer"

- [ ] 1.5 Test what breaks with 2 sessions
  - Create session A → create session B → check if A is dead
  - **Verify:** document the exact failure mode (session A killed)
  - **Commit:** "verify: confirmed 1-session-per-container limitation"

- [ ] 1.6 Document baseline metrics
  - Startup time, session create time, scrape latency, memory usage
  - **Verify:** numbers written to RESULTS.md
  - **Commit:** "docs: baseline performance metrics"

---

### Phase 2: Deploy on Orb Cloud (Single VM Proof)
**Goal:** Prove Steel runs inside Orb with full functionality.
**Time:** 1-2 days

- [ ] 2.1 Create orb.toml
  - **Verify:** Orb CLI accepts the config
  - **Commit:** "feat: add orb.toml for Steel Browser deployment"

- [ ] 2.2 Deploy to Orb, get public URL
  - **Verify:** `curl https://{id}.orbcloud.dev/v1/health` → 200
  - **Commit:** "verify: Steel runs on Orb Cloud"

- [ ] 2.3 Test all endpoints via public URL
  - Session create, scrape, screenshot, CDP, context, release
  - **Verify:** every endpoint works identically to local
  - **Commit:** "verify: all Steel endpoints work through Orb public URL"

- [ ] 2.4 Test CDP WebSocket over public URL
  - Puppeteer connects to `wss://{id}.orbcloud.dev`
  - **Verify:** full browser control works remotely
  - **Commit:** "verify: CDP WebSocket works over Orb public URL"

- [ ] 2.5 Benchmark Orb vs local
  - Compare: startup, latency, throughput
  - **Verify:** numbers added to RESULTS.md
  - **Commit:** "docs: Orb vs local performance comparison"

---

### Phase 3: Build the Orchestrator
**Goal:** Multi-session management with auth, routing, and persistence.
**Time:** 1 week

This is the core value. Each sub-step solves a specific user pain.

#### 3A: Session Router (Solves issue #263 — multi-session)

- [ ] 3.1 Scaffold orchestrator in `orchestrator/`
  - Fastify + @fastify/websocket
  - **Verify:** `npm run dev` starts
  - **Commit:** "feat: scaffold orchestrator project"

- [ ] 3.2 OrbClient — VM lifecycle management
  - createVM(), destroyVM(), waitForReady(), getVM()
  - **Verify:** integration test creates and destroys a VM
  - **Commit:** "feat: OrbClient for VM lifecycle"

- [ ] 3.3 SessionRouter — maps sessions to VMs
  - create() → provisions VM, stores mapping
  - route(sessionId) → returns VM URL
  - release(sessionId) → destroys VM, cleans mapping
  - **Verify:** create 3 sessions, each on different VM, all accessible
  - **Commit:** "feat: SessionRouter — multi-session on separate VMs"

- [ ] 3.4 Wire up POST /v1/sessions
  - Gateway creates session on a new VM, returns rewritten URLs
  - **Verify:** Steel SDK with custom baseUrl creates session successfully
  - **Commit:** "feat: session creation through orchestrator"

- [ ] 3.5 Wire up all proxy routes
  - GET/POST/DELETE session endpoints → route to correct VM
  - **Verify:** full session lifecycle works through orchestrator
  - **Commit:** "feat: proxy all session routes to correct VM"

- [ ] 3.6 Wire up CDP WebSocket proxy
  - /cdp/:sessionId → correct VM's CDP endpoint
  - **Verify:** Puppeteer connects through orchestrator, navigates
  - **Commit:** "feat: CDP WebSocket proxy for multi-session"

- [ ] 3.7 **THE KEY TEST**: Run 5 concurrent sessions
  - Create 5 sessions, each on its own VM
  - Navigate different sites on each
  - Screenshot all 5
  - Release all 5
  - **Verify:** 5 screenshots, 5 different sites, 0 VMs left
  - **Commit:** "verify: 5 concurrent sessions work independently"

#### 3B: Authentication (Solves issue #235)

- [ ] 3.8 API key middleware
  - Bearer token on all routes. Configurable via env or config file.
  - **Verify:** no key → 401, wrong key → 401, right key → 200
  - **Commit:** "feat: API key authentication"

- [ ] 3.9 Per-key session tracking
  - List sessions by API key, enforce per-key limits
  - **Verify:** key A can't see key B's sessions
  - **Commit:** "feat: per-key session isolation"

#### 3C: Session Persistence (Solves the persistence gap)

- [ ] 3.10 Context save on release
  - Before destroying VM, extract cookies + localStorage + sessionStorage
  - Save to disk (or S3 if configured)
  - **Verify:** release session → context JSON file exists
  - **Commit:** "feat: save session context on release"

- [ ] 3.11 Context restore on create
  - POST /v1/sessions with `restoreSessionId` field
  - Load saved context, inject into new session via sessionContext
  - **Verify:** create → visit site → release → restore → cookies still there
  - **Commit:** "feat: restore session context from saved state"

#### 3D: Warm Pool (Solves startup latency)

- [ ] 3.12 Warm pool manager
  - Pre-provision N idle VMs. Take from pool on session create.
  - Refill in background when pool drops below min.
  - **Verify:** session create with warm pool < 2s vs cold provision time
  - **Commit:** "feat: warm pool for fast session starts"

#### 3E: Health & Cleanup (Production readiness)

- [ ] 3.13 Health endpoint with stats
  - Active sessions, pool size, per-VM memory, uptime
  - **Verify:** GET /v1/health returns all stats
  - **Commit:** "feat: health endpoint with orchestrator stats"

- [ ] 3.14 Session timeout & auto-cleanup
  - Configurable inactivity timeout. Auto-destroy idle VMs.
  - **Verify:** idle session auto-destroyed after timeout
  - **Commit:** "feat: session timeout and auto-cleanup"

- [ ] 3.15 VM health monitoring
  - Periodic health checks on all active VMs
  - Dead VMs → clean up session mapping, notify via logs
  - **Verify:** kill a VM manually → orchestrator detects and cleans up
  - **Commit:** "feat: VM health monitoring and dead session cleanup"

---

### Phase 4: Integration Testing (Prove It Works for Real Users)
**Goal:** Test with actual tools that Steel users use.
**Time:** 2-3 days

- [ ] 4.1 Deploy orchestrator on Orb Cloud
  - Orchestrator itself runs on an always-on Orb VM
  - **Verify:** accessible at public URL
  - **Commit:** "deploy: orchestrator running on Orb Cloud"

- [ ] 4.2 Test: Steel Node SDK compatibility
  ```javascript
  const steel = new Steel({
    baseUrl: 'https://orchestrator.orbcloud.dev',
    steelAPIKey: 'test_key'
  });
  const session = await steel.sessions.create();
  // full workflow...
  ```
  - **Verify:** zero code changes needed in Steel SDK
  - **Commit:** "verify: Steel SDK works with orchestrator"

- [ ] 4.3 Test: Playwright via CDP
  - **Verify:** full browser control through orchestrator
  - **Commit:** "verify: Playwright CDP works through orchestrator"

- [ ] 4.4 Test: browser-use agent
  - Run a multi-step browser-use agent through orchestrator
  - **Verify:** agent completes task, no timeouts
  - **Commit:** "verify: browser-use works through orchestrator"

- [ ] 4.5 Test: 10 concurrent sessions
  - **Verify:** all 10 work, no cross-contamination, clean release
  - **Commit:** "verify: 10 concurrent sessions"

- [ ] 4.6 Test: session persistence round-trip
  - Login to a site → release → restore → still logged in
  - **Verify:** cookies persist across session boundaries
  - **Commit:** "verify: session persistence works end-to-end"

- [ ] 4.7 Test: error scenarios
  - VM dies mid-session, network timeout, invalid session ID
  - **Verify:** graceful error handling, no zombie VMs
  - **Commit:** "verify: error handling and recovery"

- [ ] 4.8 Stress test: 20 sequential sessions
  - Create and release 20 sessions one by one
  - **Verify:** 0 VM leaks, stable memory in orchestrator
  - **Commit:** "verify: no resource leaks under load"

- [ ] 4.9 Cost measurement
  - Track actual Orb costs for all tests
  - Compare vs Steel Cloud pricing for same workload
  - **Verify:** numbers in RESULTS.md
  - **Commit:** "docs: cost comparison with real numbers"

---

### Phase 5: Ship It
**Goal:** Make it usable by the community.
**Time:** 3-5 days

- [ ] 5.1 Write README for orchestrator
  - Quick start, configuration, architecture, FAQ
  - **Commit:** "docs: orchestrator README"

- [ ] 5.2 Add docker-compose.yml for local dev
  - Orchestrator + Steel containers for testing without Orb
  - **Commit:** "feat: docker-compose for local development"

- [ ] 5.3 Write deployment guide for Orb Cloud
  - Step-by-step: sign up → deploy → first session
  - **Commit:** "docs: Orb Cloud deployment guide"

- [ ] 5.4 Record demo video
  - 3 minutes: create 5 concurrent sessions, browse, persist, restore
  - **Commit:** "docs: add demo video link"

- [ ] 5.5 Open issue on steel-dev/steel-browser
  - Title: "Open-source multi-session orchestrator for Steel Browser"
  - Reference issues #263, #144, #235, #72
  - Link to the orchestrator repo
  - **Commit:** "outreach: opened issue on upstream"

- [ ] 5.6 Post Show HN
  - "Show HN: Open-source orchestrator for Steel Browser — concurrent sessions, auth, persistence"
  - **Commit:** "outreach: Show HN posted"

- [ ] 5.7 Contact Steel team
  - DM with demo, propose integration or partnership
  - **Commit:** "outreach: contacted Steel team"

- [ ] 5.8 Contact browser-use team
  - "browser-use + Steel Orchestrator = hosted browser agents with zero config"
  - **Commit:** "outreach: contacted browser-use team"

---

## Decision Log

| Date | Decision | Reasoning |
|------|----------|-----------|
| 2026-03-24 | Build an orchestrator, not just "deploy on Orb" | Hosting alone isn't unique value. Solving multi-session + auth + persistence is. |
| 2026-03-24 | 1 VM per session | Steel's architecture is 1 session per container. Instead of fighting it, we make it a feature (perfect isolation). Orb's pricing makes this economically viable. |
| 2026-03-24 | Keep Steel untouched (orchestrator is separate) | Don't fork Steel's code. Orchestrator is a proxy layer. Easier to maintain, works with any Steel version. |
| 2026-03-24 | Open source the orchestrator | Steel gatekeeps multi-session behind Cloud. Opening this up creates goodwill and adoption. |
| 2026-03-24 | Session persistence via context extraction | Steel already has GET /sessions/:id/context. We just save the output and re-inject on create. No Steel code changes needed. |
| 2026-03-24 | Fastify for orchestrator | Same framework as Steel. WebSocket support. Fast. |

## What Makes This Different From "Just Use K8s"

Users on issue #144 were told to build their own Kubernetes orchestrator.
Here's why this is better:

| K8s DIY | Steel Orchestrator |
|---------|-------------------|
| Need K8s expertise | `npm start` or deploy to Orb |
| Build session affinity routing yourself | Built-in session→VM routing |
| Build auth yourself | API key auth included |
| Build session persistence yourself | Context save/restore included |
| Build health checks yourself | VM monitoring included |
| Build auto-scaling yourself | Orb scales automatically |
| $3,500/mo for 50 sessions | ~$200/mo on Orb |
| Weeks to set up | 5 minutes to deploy |
