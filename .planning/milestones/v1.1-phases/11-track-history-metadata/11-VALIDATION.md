---
phase: 11
slug: track-history-metadata
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-06-04
---

# Phase 11 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Test::More (Perl built-in) |
| **Config file** | none — test files in `t/` run directly |
| **Quick run command** | `perl t/05_perl_syntax.t && perl t/11_track_history.t` |
| **Full suite command** | `prove t/` |
| **Estimated runtime** | ~5 seconds |

---

## Sampling Rate

- **After every task commit:** Run `perl t/05_perl_syntax.t && perl t/11_track_history.t`
- **After every plan wave:** Run `prove t/`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 5 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 11-01-01 | 01 | 1 | HIST-01 | — | N/A | unit | `perl t/11_track_history.t` | ❌ W0 | ⬜ pending |
| 11-01-02 | 01 | 1 | HIST-02 | — | N/A | unit | `perl t/11_track_history.t` | ❌ W0 | ⬜ pending |
| 11-01-03 | 01 | 1 | HIST-03 | — | N/A | unit | `perl t/11_track_history.t` | ❌ W0 | ⬜ pending |
| 11-01-04 | 01 | 1 | HIST-04 | — | N/A | unit | `perl t/11_track_history.t` | ❌ W0 | ⬜ pending |
| 11-01-05 | 01 | 1 | TTL-Check | — | N/A | grep-gate | `perl t/11_track_history.t` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `t/11_track_history.t` — stubs for HIST-01 through HIST-04, TTL-Grep-Gate, Debounce-Check
- [ ] Reuse LMS-Stub-Pattern from `t/10_stream_metadata.t` (Slim::Utils::Cache stub with `ttl()` support)
