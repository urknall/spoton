---
phase: 35
slug: liked-songs-play-all-throttle
status: complete
nyquist_compliant: true
wave_0_complete: true
created: 2026-06-27
---

# Phase 35 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None — Perl LMS plugin, no test harness |
| **Config file** | none |
| **Quick run command** | `perl -I Plugins -c Plugins/SpotOn/Plugin.pm` |
| **Full suite command** | Manual UAT via LMS + Material Skin |
| **Estimated runtime** | ~5 minutes (manual) |

---

## Sampling Rate

- **After every task commit:** `perl -c` compile check
- **After every plan wave:** Manual UAT
- **Before `/gsd:verify-work`:** Full manual UAT must pass
- **Max feedback latency:** N/A (manual-only)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 35-01-01 | 01 | 1 | GH-51 | T-35-01 | Cache eviction prevents memory growth | manual | N/A | N/A | ✅ green |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No test framework applicable — Perl LMS plugin with async event loop, no unit-testable boundaries without mocking the full LMS runtime.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Play-all keine Browse 404s | GH-51 | Requires live LMS + Spotify API + Material Skin concurrent browse | Play All on 1633 Liked Songs, observe server.log for 404s |
| API call reduction (~33 statt ~75) | GH-51 | Requires live Spotify API interaction | grep 'me/tracks' in server.log during play-all, count calls |
| Concurrent browse from cache | GH-51 | Requires Material Skin sending parallel browse requests | Browse Liked Songs while play-all is loading, verify no additional API calls |
| No regression in normal browse | GH-51 | Requires live browse navigation | Browse Liked Songs with offset pagination (< 500 items), verify normal behavior |

All manual verifications passed during UAT before v2.1.2 release (2026-06-26).

---

## Sign-Off

- [x] All manual verifications passed
- [x] perl -c compiles without errors
- [x] Released as v2.1.2
- [x] No regressions reported post-release
