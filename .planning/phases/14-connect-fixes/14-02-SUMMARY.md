---
phase: 14-connect-fixes
plan: 02
subsystem: connect
tags: [connect, credential-isolation, volume-sync, grace-period, syntax-tests]
dependency_graph:
  requires: []
  provides: [per-player-cache-dir, volume-ctrl-linear, initial-volume, reduced-grace-period]
  affects: [Plugins/SpotOn/Connect/Daemon.pm, Plugins/SpotOn/Connect.pm, t/05_perl_syntax.t]
tech_stack:
  added: []
  patterns: [per-player-cache-dir-isolation, linear-volume-mapping]
key_files:
  modified:
    - Plugins/SpotOn/Connect/Daemon.pm
    - Plugins/SpotOn/Connect.pm
    - t/05_perl_syntax.t
decisions:
  - "No event-delta-filter needed: 3s grace + suppress_next_volume AtomicBool + --initial-volume covers all echo scenarios"
  - "Callback.pm not yet on disk: test array has 9 entries but 8 found files; existing skip_all pattern handles this"
metrics:
  duration_seconds: 174
  completed: "2026-06-07T12:38:30Z"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 3
---

# Phase 14 Plan 02: Connect Credential Isolation, Volume Sync & Grace Period Summary

Per-player cache dir isolation (connect-{mac}/), --volume-ctrl linear + --initial-volume CLI flags, and VOLUME_GRACE_PERIOD reduced from 20s to 3s.

## Commits

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Credential isolation + volume flags + grace period | e47c2c1 | Daemon.pm, Connect.pm |
| 2 | Extend syntax tests for Connect modules | f2be02f | t/05_perl_syntax.t |

## Changes Made

### Task 1: Credential isolation, volume flags, grace period

**Daemon.pm - CON-01 (P-49): Cache dir isolation**
- Replaced `$activeAccountId`-based cache dir calculation with per-player path: `catdir($serverPrefs->get('cachedir'), 'spoton', 'connect-' . $self->id)`
- Each Connect daemon now gets its own credential cache keyed by MAC (without colons)
- Prevents credential overwrite when different Spotify users connect to different players

**Daemon.pm - CON-02 (P-50): Volume synchronization flags**
- Added `push @helperArgs, '--volume-ctrl', 'linear'` (unconditional) -- matches squeezelite's SoftMixer linear curve
- Added `push @helperArgs, '--initial-volume', int($client->volume // 50)` -- seeds librespot with current LMS volume
- Both flags placed after `--enable-volume-normalisation` and before `getCapability('autoplay')` block

**Connect.pm - CON-03: Grace period reduction**
- Changed `VOLUME_GRACE_PERIOD` from 20 to 3 seconds
- Comment block above remains accurate (lines 24-26 describe the purpose)

### Task 2: Syntax test expansion

- Added `Connect.pm` and `Connect/Daemon.pm` to `@pm_files` array (9 entries total)
- Added 6 stubs for compile-time dependencies: `Slim::Utils::Accessor`, `Slim::Player::Client`, `Slim::Player::Sync`, `Slim::Control::Request`, `Slim::Music::Info`, `Slim::Player::Source`
- No `Proc::Background` stub needed (require'd inside sub body, not at compile time)
- All 8 found modules pass `perl -c` (Callback.pm not on disk, filtered by existing grep)
- Full test suite: 232/232 tests pass

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing stub] Added Slim::Player::Source stub**
- **Found during:** Task 2
- **Issue:** Connect.pm calls `Slim::Player::Source::songTime()` at module level scope (compile-time resolution needed)
- **Fix:** Added stub with `songTime` returning 0
- **Files modified:** t/05_perl_syntax.t
- **Commit:** f2be02f

## Verification Results

- `grep "connect-" Daemon.pm` -- returns per-player cache dir line
- `grep "volume-ctrl" Daemon.pm` -- returns --volume-ctrl linear push
- `grep "initial-volume" Daemon.pm` -- returns --initial-volume push with int() and // 50
- `grep "VOLUME_GRACE_PERIOD => 3" Connect.pm` -- returns updated constant
- `grep -c "activeAccount" Daemon.pm` -- returns 0 (removed from cache-dir block)
- `prove -l t/` -- 232/232 tests pass, exit 0

## Self-Check: PASSED
