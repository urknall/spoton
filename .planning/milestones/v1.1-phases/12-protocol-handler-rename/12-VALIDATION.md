---
phase: 12
slug: protocol-handler-rename
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-06-05
---

# Phase 12 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Test::More (Perl) |
| **Config file** | none — direct `prove` invocation |
| **Quick run command** | `prove t/05_perl_syntax.t t/03_convert_conf.t t/11_track_history.t` |
| **Full suite command** | `prove t/` |
| **Estimated runtime** | ~5 seconds |

---

## Sampling Rate

- **After every task commit:** Run `prove t/05_perl_syntax.t`
- **After every plan wave:** Run `prove t/`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 5 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 12-01-01 | 01 | 1 | PROTO-01 | — | N/A | unit (grep) | `prove t/12_protocol_rename.t` | ❌ W0 | ⬜ pending |
| 12-01-02 | 01 | 1 | PROTO-02 | — | N/A | unit (grep) | `prove t/12_protocol_rename.t` | ❌ W0 | ⬜ pending |
| 12-01-03 | 01 | 1 | PROTO-03 | — | N/A | unit | `prove t/03_convert_conf.t` | ✅ | ⬜ pending |
| 12-01-04 | 01 | 1 | PROTO-04 | — | N/A | unit (grep) | `prove t/12_protocol_rename.t` | ❌ W0 | ⬜ pending |
| 12-01-05 | 01 | 1 | PROTO-05 | — | N/A | unit (grep) | `prove t/12_protocol_rename.t` | ❌ W0 | ⬜ pending |
| 12-01-06 | 01 | 1 | PROTO-06 | — | N/A | unit (grep) | `prove t/12_protocol_rename.t` | ❌ W0 | ⬜ pending |
| 12-01-07 | 01 | 1 | PROTO-01 | — | N/A | unit | `prove t/05_perl_syntax.t` | ✅ | ⬜ pending |
| 12-01-08 | 01 | 1 | All | — | N/A | regression | `prove t/11_track_history.t` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `t/12_protocol_rename.t` — covers PROTO-01 through PROTO-06 via grep assertions on source files and binary --check

*Existing infrastructure covers PROTO-01 syntax (t/05_perl_syntax.t), PROTO-03 (t/03_convert_conf.t), and regression (t/11_track_history.t).*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Spotty + SpotOn coexistence | PROTO-05, D-07 | Requires two LMS plugins running simultaneously on real hardware | Activate both plugins on raspi, browse in each, verify no handler conflict |
| Browse playback after rename | PROTO-01 | End-to-end audio requires librespot binary + Spotify Premium | Play a track via Browse on dev/raspi after deploy, verify audio output |
| Connect playback after rename | PROTO-04 | Requires Spotify app Connect handoff | Start playback from Spotify app, verify Connect URL uses `spoton://connect-` prefix in LMS log |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 5s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
