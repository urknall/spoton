---
phase: 04
slug: single-track-streaming
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-05-28
---

# Phase 04 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual verification + LMS runtime testing |
| **Config file** | none — LMS plugin testing via live server |
| **Quick run command** | `perl -c lib/Slim/Plugin/SpotOn/ProtocolHandler.pm` |
| **Full suite command** | Manual: play track, verify audio, seek, check process cleanup |
| **Estimated runtime** | ~30 seconds (syntax check); manual tests ~5 min |

---

## Sampling Rate

- **After every task commit:** Run `perl -c` syntax check on modified files
- **After every plan wave:** Full manual verification cycle
- **Before `/gsd:verify-work`:** All success criteria must be manually verified
- **Max feedback latency:** 30 seconds (syntax), 300 seconds (manual)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| TBD | TBD | TBD | STR-01..STR-11, LMS-11 | — | N/A | manual | `perl -c {file}` | TBD | pending |

*Status: pending · green · red · flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements — LMS plugin testing is runtime-based.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Track plays audio within 5s | STR-01 | Requires LMS + librespot runtime | Select track from Browse menu, measure time to first audio |
| OGG direct for capable players | STR-02 | Requires player capability detection | Check transcoding table for OGG-capable vs non-OGG player |
| Seeking works correctly | STR-04 | Requires audio playback state | Seek to middle of track, verify position in LMS UI |
| No race condition | LMS-11 | Requires two simultaneous players | Start different tracks on two players, verify correct audio |
| Process cleanup | STR-11 | Requires time-based observation | Run for 2 hours, verify no orphaned librespot processes |

---

## Validation Sign-Off

- [ ] All tasks have verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 300s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
