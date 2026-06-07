---
phase: 14-connect-fixes
plan: 01
subsystem: librespot-spoton
tags: [connect, volume, cli-flags, rust]
dependency_graph:
  requires: []
  provides: [initial-volume-flag, volume-ctrl-flag]
  affects: [Daemon.pm]
tech_stack:
  added: []
  patterns: [cli-flag-parsing, lms-volume-scale-conversion, volumectrl-enum-dispatch]
key_files:
  created: []
  modified:
    - librespot-spoton/src/main.rs
    - librespot-spoton/src/connect.rs
decisions:
  - "D-07: LMS sends 0-100 scale, Rust converts to u16 0-65535 internally"
  - "D-08: volume_ctrl defaults to 'log' (librespot default); Perl passes 'linear'"
metrics:
  duration: 4m 11s
  completed: 2026-06-07
  tasks: 2/2
  files_modified: 2
---

# Phase 14 Plan 01: Connect Volume CLI Flags Summary

Extend librespot-spoton binary with --initial-volume (LMS 0-100 to u16 0-65535 conversion) and --volume-ctrl (linear/log/fixed VolumeCtrl dispatch) CLI flags for Connect daemon startup volume sync.

## Changes Made

### Task 1: Add --initial-volume and --volume-ctrl flag parsing to main.rs (de9c67c)

- Added `initial_volume_lms: Option<u8>` and `volume_ctrl_str: String` Connect mode variables
- Added `--initial-volume` match arm: parses u8, clamps to `.min(100)` (T-14-01 mitigation)
- Added `--volume-ctrl` match arm: clones string value for downstream parsing
- Added LMS-to-librespot scale conversion: `(v as u32 * 65535 / 100) as u16`
- Updated `run_connect()` call with `initial_volume_u16` and `&volume_ctrl_str` trailing args

### Task 2: Extend run_connect() with VolumeCtrl and initial_volume support (a620093)

- Imported `VolumeCtrl` from `librespot_playback::config`
- Extended `run_connect()` signature with `initial_volume: Option<u16>` and `volume_ctrl_str: &str`
- Added `volume_ctrl_enum` match dispatch: "linear" -> `VolumeCtrl::Linear`, "fixed" -> `VolumeCtrl::Fixed`, default -> `VolumeCtrl::Log(DEFAULT_DB_RANGE)` (T-14-02 mitigation)
- Updated primary `MixerConfig` with `volume_ctrl: volume_ctrl_enum`
- Updated primary `ConnectConfig` with `initial_volume: initial_volume.unwrap_or(u16::MAX / 2)`
- Updated reconnect `ConnectConfig` with same dynamic initial_volume (Pitfall 2)
- Updated reconnect `MixerConfig` with same `volume_ctrl_enum` (Pitfall 2, VolumeCtrl is Copy)
- `cargo build --release` exits 0

## Verification Results

| Check | Expected | Actual | Status |
|-------|----------|--------|--------|
| `grep -c "initial_volume" main.rs` | >= 3 | 4 | PASS |
| `grep -c "volume_ctrl" connect.rs` | >= 4 | 6 | PASS |
| `grep -c "unwrap_or" connect.rs` | >= 2 | 9 | PASS |
| `cargo build --release` | exit 0 | exit 0 | PASS |

## Deviations from Plan

None - plan executed exactly as written.

## Commits

| Task | Commit | Message |
|------|--------|---------|
| 1 | de9c67c | feat(14-01): add --initial-volume and --volume-ctrl CLI flag parsing to main.rs |
| 2 | a620093 | feat(14-01): extend run_connect() with VolumeCtrl and initial_volume support |

## Self-Check: PASSED
