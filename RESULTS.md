# E2E Test Results

**Date:** 2026-03-26T12:19:12Z
**Orb API:** https://api.orbcloud.dev
**Orchestrator:** not tested

## Summary

- **PASS:** 10
- **FAIL:** 1
- **SKIP:** 1
- **Total:** 12

## Results

| Test | Status | Detail |
|------|--------|--------|
| create-vm | PASS | Computer b3490051 created |
| upload-config | PASS | orb.toml accepted |
| build | PASS | All build steps exit 0 |
| deploy | PASS | Agent on port 10014 |
| health | PASS | Steel Browser healthy at https://b3490051.orbcloud.dev |
| create-session | PASS | Session ccb6063b-a2c4-440c-8d50-8184a21aa4dc |
| scrape | PASS | Title: Example Domain |
| screenshot | PASS | 46939 bytes |
| context | PASS | 0 cookies |
| release-session | PASS | Session released |
| criu-checkpoint | FAIL | demote_failed |
| orchestrator-health | SKIP | ORCHESTRATOR_URL not set |

## Notes

- CRIU checkpoint (freeze) consistently works
- CRIU restore (wake) status tracked per run
- Build takes 3-5 minutes (Chrome + npm install)
- Each test run creates and destroys VMs (clean state)
