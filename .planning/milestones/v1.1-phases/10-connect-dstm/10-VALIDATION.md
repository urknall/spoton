---
phase: 10
slug: connect-dstm
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-06-04
---

# Phase 10 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual verification (Perl LMS plugin + Rust binary — no automated test framework) |
| **Config file** | none |
| **Quick run command** | `librespot-spoton/target/release/librespot --check` |
| **Full suite command** | Manual: start Connect daemon, verify autoplay via Spotify app |
| **Estimated runtime** | ~30 seconds (binary check), ~120 seconds (full manual) |

---

## Sampling Rate

- **After every task commit:** Run `librespot-spoton/target/release/librespot --check` for binary changes
- **After every plan wave:** Full manual Connect autoplay verification
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds (binary), 120 seconds (manual)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| TBD | 01 | 1 | DSTM-01 | — | N/A | manual | `librespot --check \| jq .autoplay` | ❌ W0 | ⬜ pending |
| TBD | 01 | 1 | DSTM-02 | — | N/A | manual | Verify SessionConfig.autoplay set | ❌ W0 | ⬜ pending |
| TBD | 02 | 2 | DSTM-05 | — | N/A | manual | Settings page shows toggle | ❌ W0 | ⬜ pending |
| TBD | 02 | 2 | DSTM-06 | — | N/A | manual | Browse-DSTM still works | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements — no new test framework needed.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Spirc autoplay continues after queue exhaustion | DSTM-01, DSTM-04 | Requires live Spotify Premium session + Connect handover | 1. Start SpotOn Connect daemon 2. Play a single track via Spotify app 3. Wait for track to end 4. Verify autoplay context loads next track |
| Per-player toggle disables autoplay | DSTM-05 | Requires LMS Settings UI interaction | 1. Open LMS Settings > SpotOn player settings 2. Disable Autoplay toggle 3. Restart Connect daemon 4. Verify no autoplay after track ends |
| Browse-DSTM regression | DSTM-06 | Requires LMS Browse playback + DSTM trigger | 1. Play track via Browse menu 2. Wait for track to end 3. Verify DSTM provider fires and next track plays |
| Bidirectional DSTM sync | DSTM-03, DSTM-04 | Requires LMS DSTM dropdown interaction | 1. Toggle SpotOn Autoplay ON → verify DSTM dropdown shows "SpotOn" 2. Set DSTM dropdown to "Off" → verify SpotOn toggle shows OFF |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 120s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
