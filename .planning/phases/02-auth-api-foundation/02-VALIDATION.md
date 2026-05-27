---
phase: 02
slug: auth-api-foundation
status: draft
nyquist_compliant: true
wave_0_complete: false  # becomes true after 02-00 execution
created: 2026-05-27
---

# Phase 02 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Perl Test::More (LMS bundled) |
| **Config file** | t/ directory |
| **Quick run command** | `prove -v t/` |
| **Full suite command** | `prove -v t/` |
| **Estimated runtime** | ~5 seconds |

---

## Sampling Rate

- **After every task commit:** Run `prove -v t/`
- **After every plan wave:** Run `prove -v t/`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 5 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| *Populated after plans are created* | | | | | | | | | |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Covered by Plan 02-00-PLAN.md (Wave 0):

- [ ] `t/07_token_manager.t` — stubs for AUTH-01 through AUTH-05 (mock binary, cache, timer)
- [ ] `t/08_api_client.t` — stubs for API-01, API-02, API-03, API-04, API-06 (HTTP mock, rate limit)
- [ ] `t/09_settings.t` — stubs for AUTH-04 (chmod), AUTH-05 (multi-account), AUTH-06 (switch), i18n

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Token visible in debug log | AUTH-01 | Requires running LMS with Spotify account | Start LMS, configure account, check debug log for "access_token" |
| Menu refresh after account switch | AUTH-05 | Requires LMS UI interaction | Switch active account, verify OPML menu shows new account data |
| 50 rapid API calls no 429 | API-02 | Requires live Spotify API with valid token | Run bulk API test script against Spotify API |
| Connect daemon restart at 50 min | AUTH-03 | Requires long-running test | Start daemon, wait 50 min, verify restart in log |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 5s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
