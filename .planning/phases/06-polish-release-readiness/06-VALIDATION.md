---
phase: 06
slug: polish-release-readiness
status: draft
nyquist_compliant: true
wave_0_complete: true
created: 2026-06-03
---

# Phase 06 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual UAT (LMS plugin — no unit test framework) |
| **Config file** | none |
| **Quick run command** | `perl -c Plugins/SpotOn/Plugin.pm 2>&1` |
| **Full suite command** | `find Plugins/SpotOn -name '*.pm' -exec perl -c {} \; 2>&1` |
| **Estimated runtime** | ~5 seconds |

---

## Sampling Rate

- **After every task commit:** Run `perl -c` on modified .pm files
- **After every plan wave:** Run full syntax check on all .pm files
- **Before `/gsd:verify-work`:** Full syntax check + manual UAT must pass
- **Max feedback latency:** 5 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 06-01-01 | 01 | 1 | LMS-10 | — | N/A | syntax | `perl -c Plugins/SpotOn/API/Client.pm` | ✅ | ⬜ pending |
| 06-01-02 | 01 | 1 | LMS-10 | — | N/A | syntax | `perl -c Plugins/SpotOn/API/TokenManager.pm` | ✅ | ⬜ pending |
| 06-02-01 | 02 | 1 | LMS-08 | T-06-01 | Bitrate validated against allowlist | syntax | `perl -c Plugins/SpotOn/Settings.pm` | ✅ | ⬜ pending |
| 06-02-02 | 02 | 1 | LMS-08 | — | N/A | syntax | `perl -c Plugins/SpotOn/ProtocolHandler.pm && perl -c Plugins/SpotOn/Plugin.pm` | ✅ | ⬜ pending |
| 06-03-01 | 03 | 2 | LMS-09 | — | N/A | syntax | `perl -c Plugins/SpotOn/DontStopTheMusic.pm` | ❌ W1 | ⬜ pending |
| 06-03-02 | 03 | 2 | LMS-09 | — | N/A | syntax | `perl -c Plugins/SpotOn/Plugin.pm` | ✅ | ⬜ pending |
| 06-04-01 | 04 | 3 | LMS-03 | — | N/A | syntax+count | `grep -cP '^\t[A-Z]{2}\t' Plugins/SpotOn/strings.txt \| sort -u \| wc -l` | ✅ | ⬜ pending |
| 06-04-02 | 04 | 3 | LMS-03 | — | N/A | grep | `grep -c SETUP_GUIDE Plugins/SpotOn/strings.txt` | ✅ | ⬜ pending |
| 06-05-01 | 05 | 4 | LMS-03 | — | N/A | grep | `test -f repo.xml && grep -q 'plugin name' repo.xml` | ❌ W4 | ⬜ pending |
| 06-05-02 | 05 | 4 | — | — | N/A | manual | `echo "Checkpoint: awaiting human verification"` | — | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements. DontStopTheMusic.pm is created in Wave 2 (Plan 06-03 Task 1). repo.xml is created in Wave 4 (Plan 06-05 Task 1).*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Per-player bitrate streams at configured rate | LMS-08 | Requires two physical players | Set 96kbps on Player A, 320kbps on Player B, play same track, verify audio quality |
| DSTM queues track after playlist ends | LMS-09 | Requires Spotify API + real playback | Play short playlist, wait for end, verify auto-queue |
| Custom binary override works | LMS-10 | Requires binary swap on filesystem | Place spoton-custom in Bin path, verify --check, verify playback |
| Format dropdown forces transcode | LMS-08 | Requires player audio chain | Set FLAC transcode, verify custom-convert.conf pipeline used |
| Setup guide renders correctly | LMS-03 | Visual inspection | Open Settings without account, verify guide shows correct steps |
| All 11 languages render | LMS-03 | Visual per-language | Switch LMS language, verify no missing-key placeholders |
| Volume normalization in Connect mode | LMS-08 | Requires Connect playback | Enable normalization, play via Connect, verify Daemon.pm passes flag |

*Most phase behaviors require manual verification due to LMS plugin + hardware dependency.*

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 5s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-06-03
