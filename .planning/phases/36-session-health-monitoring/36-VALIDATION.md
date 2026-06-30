---
phase: 36
slug: session-health-monitoring
status: complete
nyquist_compliant: false
wave_0_complete: false
created: 2026-06-30
---

# Phase 36 — Validation Strategy

> Per-phase validation contract for Session Health Monitoring.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None — Perl+Rust project without test harness |
| **Config file** | none |
| **Quick run command** | `cargo check` (Rust syntax only) |
| **Full suite command** | N/A |
| **Estimated runtime** | ~8s (cargo check) |

---

## Per-Task Verification Map

| Task ID | Plan | Requirement | Test Type | Status |
|---------|------|-------------|-----------|--------|
| 36-01-01 | 01 | P36-GOAL | manual | ✅ verified (VERIFICATION.md) |
| 36-01-02 | 01 | P36-GOAL | manual | ✅ verified (VERIFICATION.md) |
| 36-02-01 | 02 | P36-GOAL | manual | ✅ verified (VERIFICATION.md + UAT) |
| 36-02-02 | 02 | P36-GOAL | manual | ✅ verified (UAT) |

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Verified |
|----------|-------------|------------|----------|
| `/health` returns JSON with session_valid, session_age_secs, idle_secs | P36-GOAL | No test harness; verified via curl + VERIFICATION.md | ✅ |
| session_age_secs reflects session connect time | P36-GOAL | Runtime behavior, requires active daemon | ✅ |
| idle_secs reflects last activity time | P36-GOAL | Runtime behavior, requires active daemon | ✅ |
| session_valid reflects Session::is_invalid() | P36-GOAL | Runtime behavior, requires Spotify session | ✅ |
| Perl polls /health every 60s | P36-GOAL | Runtime timing, requires LMS observation | ✅ |
| session_valid=false triggers restart | P36-GOAL | Requires simulating dead Spotify session | ⬜ |
| Age >4h + idle >5min triggers restart | P36-GOAL | Requires 4h idle wait | ⬜ |
| Active Connect not disrupted by restart | P36-GOAL | Runtime behavior, verified via UAT | ✅ |
| Health restart logged at INFO | P36-GOAL | Log observation during restart event | ⬜ |
| Watchdog cycle produces no INFO output | P36-GOAL | Code review confirmed (DEBUGLOG); runtime verified | ✅ |
| Status page shows session health | P36-GOAL | Visual inspection, verified via UAT | ✅ |

---

## Validation Audit 2026-06-30

| Metric | Count |
|--------|-------|
| Total requirements | 11 |
| Automated tests | 0 |
| Manual verified | 8 |
| Manual pending | 3 |
| Escalated | 0 |

**Note:** 3 pending items (session_valid=false restart, stale-session restart, restart logging) require specific failure conditions that cannot be reliably triggered in a dev environment. They are structurally verified via code review (CR-01 through WR-05 in 36-REVIEW.md).
