# CRIU Checkpoint/Restore Log

Tracking every CRIU test attempt for Steel Browser + Chrome on Orb Cloud.

| Date | Computer | Port | Checkpoint | Restore | Screenshot Before | Screenshot After | Notes |
|------|----------|------|------------|---------|-------------------|------------------|-------|
| 2026-03-26T12:26:41Z | 80427cdd | 10015 | FAIL (demote_failed) | - | 46939B | - | Checkpoint dir: none |
| 2026-03-26T18:46:44Z | 2d086d4e | 10047 | PASS | FAIL (promote_failed) | 46939B | - | CRIU restore failed for agent 10047. Dir: /orb/checkpoints/agent_10047_1774551092 |
| 2026-03-26T19:34:51Z | 457653cb | 10068 | PASS | FAIL (promote_failed) | 46939B | - | CRIU restore failed for agent 10068. Dir: /orb/checkpoints/agent_10068_1774554036 |
| 2026-03-26T21:19:25Z | da36b96b (playwright) | - | - | - | - | - | Build failed |
| 2026-03-26T21:20:03Z | 168b13c4 (playwright) | 10026 | - | - | - | - | Health failed |
| 2026-03-26T21:21:44Z | f2103a57 (playwright) | 10027 | - | - | - | - | Health failed |
| 2026-03-26T21:23:23Z | 3d8f75a4 (playwright) | 10028 | - | - | - | - | Health failed |
| 2026-03-27T00:00:00Z | 61d3bdbc (playwright) | 10007 | PASS | PASS | 121082B (3 cookies) | 121082B | CRIU WORKS! Playwright + Chromium checkpoint/restore verified |
| 2026-03-27T00:30:00Z | 61d3bdbc (playwright) | 10007 | PASS x3 | PASS x3 | Google 121KB, Wiki 51KB, HN 117KB | All match | STRESS TEST: 3 rapid cycles, all pass |
