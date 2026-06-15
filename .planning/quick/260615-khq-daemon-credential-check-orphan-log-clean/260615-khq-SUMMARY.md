---
phase: quick-260615-khq
plan: 01
subsystem: connect-daemon
tags: [daemon-hygiene, crash-loop-prevention, log-cleanup]
dependency_graph:
  requires: []
  provides: [credential-precheck, orphan-log-cleanup]
  affects: [Connect/DaemonManager.pm]
tech_stack:
  added: []
  patterns: [pre-condition-guard, delayed-timer-cleanup]
key_files:
  modified:
    - Plugins/SpotOn/Connect/DaemonManager.pm
decisions:
  - "Credential path matches Daemon.pm account-scoped cache dir (CON-01), not per-MAC subdirectory"
  - "Orphan cleanup uses 30s delay timer to allow players to reconnect after restart"
  - "Only log files deleted, never credential directories"
metrics:
  duration: 180s
  completed: 2026-06-15T12:51:12Z
  tasks_completed: 2
  tasks_total: 2
  files_modified: 1
---

# Quick 260615-khq Plan 01: Daemon Credential Check + Orphan Log Cleanup Summary

Credential pre-check prevents crash-loop-detection cycle for unconfigured players; delayed orphan log cleanup removes stale connect logs from disconnected players at startup.

## Task Results

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Credential pre-check before daemon start | 700932e | DaemonManager.pm |
| 2 | Delayed orphaned log file cleanup at startup | 765b38c | DaemonManager.pm |

## Changes Made

### Task 1: Credential Pre-check

Added a credential existence check in `startHelper()` BEFORE any `Daemon->new()` or `$helper->start()` call. The check constructs the credential path using the same account-scoped cache directory logic as `Daemon.pm::start()` (CON-01):

- `$serverPrefs->get('cachedir')/spoton/$activeAccountId/credentials.json`
- If no credentials.json exists, logs at INFO level and returns immediately
- Eliminates the crash-loop-detection -> 30min disable -> retry cycle for players without cached credentials

New imports: `File::Spec::Functions qw(catdir catfile)`, `$serverPrefs = preferences('server')`.

### Task 2: Orphaned Log Cleanup

Added `_cleanupOrphanedLogs()` subroutine triggered by a 30-second delayed timer from `init()`:

- Scans `{cachedir}/spoton/*-connect.log` files
- Extracts 12-hex-char MAC from filename, converts to colon-separated format
- Checks `Slim::Player::Client::getClient($mac)` to determine if player is connected
- Deletes log file only if player is NOT connected; logs each deletion at INFO level
- 30s delay ensures players have had time to reconnect after LMS restart

New imports: `File::Basename qw(basename)`, constant `ORPHAN_LOG_CLEANUP_DELAY => 30`.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Credential path corrected to match actual binary behavior**
- **Found during:** Task 1
- **Issue:** Plan initially specified `connect-{MAC}/credentials.json` path which does not match the actual cache directory structure used by Daemon.pm. The binary receives `-c $cacheDir` where `$cacheDir` is the account-scoped directory, and stores `credentials.json` at the root of that directory -- not in a per-MAC subdirectory. Using the plan's path would cause the check to always fail (file never exists there), preventing ALL daemons from starting.
- **Fix:** Used the same account-scoped cache dir logic as Daemon.pm::start() lines 89-92: `catfile($serverPrefs->get('cachedir'), 'spoton', $activeAccountId, 'credentials.json')`
- **Files modified:** DaemonManager.pm
- **Commit:** 700932e

## Verification

1. `perl -c` syntax check: PASS (with LMS stubs)
2. Credential check positioned at line 261, before `Daemon->new()` at line 275: PASS
3. Orphaned log cleanup uses 30s delayed timer (not immediate): PASS
4. Only `*-connect.log` files deleted, never credential dirs: PASS
5. Both features log at INFO level: PASS
6. `prove t/` -- all 316 tests pass across 12 files: PASS

## Self-Check: PASSED
