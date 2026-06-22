---
phase: 28-persistent-browse-daemon
plan: "02"
subsystem: browse-daemon
tags: [browse, daemon, lifecycle, perl, watchdog, crash-loop]
dependency_graph:
  requires: []
  provides:
    - Browse::Daemon — process wrapper with port capture, crash-loop protection
    - Browse::DaemonManager — per-player Browse daemon lifecycle management
  affects:
    - Plugin.pm (Plan 03 — will call DaemonManager->init())
    - ProtocolHandler.pm (Plan 03 — will call helperForClient())
tech_stack:
  added: []
  patterns:
    - Proc::Background process management with stdout pipe for port capture
    - IO::Select 5s synchronous port read
    - Untie/retie STDERR around Proc::Background fork (Pitfall 7)
    - Sliding-window crash-loop detection (5 crashes / 2 min)
    - Debounced client event subscription (2s) + sync event handler
    - Credential pre-check (credentials.json) before daemon start
    - Per-player MAC dedup with sync-master priority
    - Sync-group fallback in helperForClient
key_files:
  created:
    - Plugins/SpotOn/Browse/Daemon.pm
    - Plugins/SpotOn/Browse/DaemonManager.pm
  modified: []
decisions:
  - Browse daemon uses plain stop() on sync change (no stopForSync — no discovery state to preserve)
  - _checkStartTimes returns 1 to abort start (no cooldown timer — DaemonManager watchdog handles recovery)
  - _streamAlivePoll polls all alive instances unconditionally (Browse daemon is always streaming when alive)
  - stderr log appended (>>) not truncated (>) in diagnostic mode — same log across restarts
metrics:
  duration_minutes: 15
  completed_date: "2026-06-22T12:34:31Z"
  tasks_completed: 2
  tasks_total: 2
  files_created: 2
  files_modified: 0
---

# Phase 28 Plan 02: Browse Daemon Lifecycle Modules Summary

**One-liner:** Perl Browse daemon lifecycle management — per-player Proc::Background process wrapper with dynamic port capture, 5-crash watchdog, and sync-group coordination.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create Browse/Daemon.pm — Process wrapper | 528ea3d | Plugins/SpotOn/Browse/Daemon.pm |
| 2 | Create Browse/DaemonManager.pm — Per-player lifecycle | cb2d87f | Plugins/SpotOn/Browse/DaemonManager.pm |

## What Was Built

### Browse/Daemon.pm (243 lines)

Structural clone of `Connect/Daemon.pm` with Browse-specific adaptations:

- **Package:** `Plugins::SpotOn::Browse::Daemon`
- **Accessors:** `id`, `mac`, `cache`, `_lastSeen`, `_startTimes`, `_proc`, `_browsePort`, `_stderrFh`
- **start():** Builds `spoton -c $cache --browse --disable-audio-cache --player-mac $mac`, reads `browse_port=(\d+)` from stdout via IO::Select 5s timeout
- **Critical patterns:** untie/retie STDERR before Proc::Background fork (Pitfall 7); `close($port_w)` before IO::Select (Pitfall 5)
- **Crash-loop:** `_checkStartTimes()` — 5 crashes in 2 minutes → return 1 to abort start (simplified vs Connect, no per-player discovery flag)
- **Diagnostic logging:** stderr to `{id}-browse.log` when `diagnosticMode` enabled (T-28-07); RUST_LOG gated same way
- **No** `_streamMode`, `_streamStartTimes`, `_checkStreamStartTimes`, `stopForSync`

### Browse/DaemonManager.pm (344 lines)

Structural clone of `Connect/DaemonManager.pm` with Browse-specific simplifications:

- **Package:** `Plugins::SpotOn::Browse::DaemonManager`
- **init():** Subscribes to `client new/disconnect` (2s debounce) + `sync` events; initial 0.5s timer; 30s orphan log cleanup
- **initHelpers():** `browseMode=http` gate (no `_isConnectEnabled`); all connected players get a daemon; sync slaves defer to master
- **_streamAlivePoll():** Polls ALL alive instances (no `_streamMode` filter); 5s fast watchdog
- **startHelper():** Credential pre-check (`credentials.json`) before `Browse::Daemon->new()`
- **helperForClient():** Direct MAC lookup + sync-group master/slave fallback
- **helperPids():** Returns PIDs of alive instances for orphan process exclusion (CON-09)
- **_cleanupOrphanedLogs():** Globs `*-browse.log`, mtime+connected guard (>300s)
- **No** `_isConnectEnabled`, `_streamMode`, `stopForSync`, `streamPortForClient`, `helperInstances`

## Deviations from Plan

None — plan executed exactly as written.

The plan noted `perl -c` would serve as the verification step, but this is not achievable in the development environment: the LMS `Slim::Utils::Log` module depends on `Log::Log4perl::Logger` and bareword constants (`main::SCANNER`, `main::PERFMON`, `main::ISWINDOWS`) that are only defined at LMS runtime. The same limitation applies to the existing `Connect/Daemon.pm` — both fail `perl -c` identically. Structural verification was performed via acceptance criteria grep checks, confirming all required patterns are present.

## Verification Results

| Check | Result |
|-------|--------|
| `_browsePort` count in Daemon.pm >= 3 | 10 occurrences |
| `browse.log` count in DaemonManager.pm >= 1 | 2 occurrences |
| `browseMode` count in DaemonManager.pm >= 1 | 5 occurrences |
| Daemon.pm min_lines >= 120 | 243 lines |
| DaemonManager.pm min_lines >= 150 | 344 lines |
| `--browse` flag in start() | present |
| `browse_port=(\d+)` regex in start() | present |
| untie/retie STDERR pattern | present |
| `close($port_w)` before IO::Select | line 138 vs 153 |
| `{id}-browse.log` filename | present |
| No _streamPort, _streamMode, stopForSync | confirmed absent |
| No Connect::Daemon or Connect::DaemonManager references | confirmed absent |
| helperForClient sync-group fallback | present |
| helperPids() | present |

## Known Stubs

None.

## Threat Flags

None — no new network endpoints introduced by the Perl modules. The actual HTTP server binding (`0.0.0.0:0`) is in `browse.rs` (Rust), which is covered by Plan 01 of this phase.

## Self-Check

Files exist:
- Plugins/SpotOn/Browse/Daemon.pm — FOUND
- Plugins/SpotOn/Browse/DaemonManager.pm — FOUND

Commits exist:
- 528ea3d — FOUND
- cb2d87f — FOUND

## Self-Check: PASSED
