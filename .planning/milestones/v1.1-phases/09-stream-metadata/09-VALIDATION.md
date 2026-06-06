---
phase: 09
slug: stream-metadata
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-06-04
---

# Phase 09 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Test::More (Perl, bundled with LMS) |
| **Config file** | `t/` directory |
| **Quick run command** | `prove -l t/` |
| **Full suite command** | `prove -l t/` |
| **Estimated runtime** | ~5 seconds |

---

## Sampling Rate

- **After every task commit:** Run `prove -l t/`
- **After every plan wave:** Run `prove -l t/`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 09-01-01 | 01 | 1 | META-01 | — | N/A | manual | Songinfo check | — | ⬜ pending |
| 09-01-02 | 01 | 1 | META-02 | — | N/A | manual | Songinfo check | — | ⬜ pending |
| 09-01-03 | 01 | 1 | META-03 | — | N/A | manual | Songinfo check | — | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Songinfo shows "(Spotify Browse)" for Browse tracks | META-01 | Requires live LMS + Spotify playback | Play track via Browse menu → check Songinfo type field |
| Songinfo shows "(Spotify Connect)" for Connect tracks | META-01 | Requires live Spotify Connect session | Start Connect session → check Songinfo type field |
| Songinfo shows stream format (OGG/FLAC/MP3/PCM) | META-02 | Requires live playback with format pref | Set streamFormat pref → play track → check Songinfo |
| Songinfo shows bitrate alongside format | META-03 | Requires live playback | Play track → verify "320k, OGG (Spotify Browse)" in Songinfo |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 10s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
