---
phase: 5
slug: spotify-connect
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-06-01
---

# Phase 5 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual verification (LMS plugin + librespot binary) |
| **Config file** | none — no automated test framework for Perl LMS plugins |
| **Quick run command** | `perl -c Plugins/SpotOn/Plugin.pm && perl -c Plugins/SpotOn/Connect.pm` |
| **Full suite command** | `cd librespot-spoton && cargo build --features connect && cargo test` |
| **Estimated runtime** | ~30 seconds (Rust), ~2 seconds (Perl syntax) |

---

## Sampling Rate

- **After every task commit:** Run `perl -c Plugins/SpotOn/Plugin.pm && perl -c Plugins/SpotOn/Connect.pm`
- **After every plan wave:** Run `cd librespot-spoton && cargo build --features connect && cargo test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| TBD | TBD | TBD | CON-01..CON-17 | — | N/A | TBD | TBD | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

*Will be populated after PLAN.md files are generated.*

---

## Wave 0 Requirements

- [ ] `librespot-spoton/Cargo.toml` — add `librespot-connect`, `hyper`, `hyper-util`, `http-body-util`, `tokio-stream`, `bytes` dependencies
- [ ] `cargo build --features connect` — verify Connect mode compiles
- [ ] `perl -c Plugins/SpotOn/Connect.pm` — verify new Connect module syntax

*Existing Rust test infrastructure (cargo test) covers binary-side requirements. Perl-side relies on syntax checks and manual UAT.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Player appears in Spotify app device list | CON-01 | Requires Spotify app + mDNS discovery | Start LMS with SpotOn, check Spotify app → Devices |
| Audio plays within 3 seconds of transfer | CON-02 | End-to-end timing across apps | Transfer playback in Spotify app, measure time to audio |
| Transport controls work bidirectionally | CON-03, CON-04 | Requires Spotify app interaction | Play/Pause/Skip from Spotify app, verify LMS state |
| Sync group appears as single device | CON-06 | Requires multiple physical players | Sync 2 players, check Spotify device list |
| Volume no-jump on transfer | CON-05 | Audio level measurement | Transfer playback, listen for volume spike |
| Connect/Browse mutual exclusion | CON-08, CON-17 | Requires active playback in both modes | Play Browse track, then transfer Connect — verify clean switch |
| Crash recovery with exponential backoff | CON-07 | Requires daemon kill + timing | Kill daemon process, verify restart timing |

*Most Connect behaviors require real Spotify app interaction and physical LMS players.*

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
