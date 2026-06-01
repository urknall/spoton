---
phase: 05-spotify-connect
plan: "03"
subsystem: connect-lifecycle
tags: [connect, daemon-manager, lifecycle, sync-groups, watchdog, pid-exclusion]
dependency_graph:
  requires:
    - "05-01"   # librespot connect.rs binary with HTTP stream server
    - "05-02"   # Connect/Daemon.pm — per-player process wrapper
  provides:
    - "Plugins/SpotOn/Connect/DaemonManager.pm — daemon lifecycle + sync groups"
    - "Plugin.pm — Connect init at boot, CON-09 PID exclusion, soc OGG guard"
  affects:
    - "05-04"   # Connect.pm event dispatcher — uses DaemonManager.helperForClient()
    - "05-05"   # ProtocolHandler.pm — uses DaemonManager.streamPortForClient()
tech_stack:
  added: []
  patterns:
    - "Debounced timer on client connect/disconnect events (DAEMON_INIT_DELAY=2s)"
    - "Differential sync restart: stopForSync() + 0.1s micro-delay (CON-15)"
    - "60s watchdog timer via Slim::Utils::Timers::setTimer"
    - "5s fast-poll _streamAlivePoll for streaming daemon crash recovery"
    - "CON-09 PID exclusion: $INC check before helperPids() call in orphan cleanup"
key_files:
  created:
    - "Plugins/SpotOn/Connect/DaemonManager.pm (267 lines)"
  modified:
    - "Plugins/SpotOn/Plugin.pm (40 insertions, 6 deletions)"
decisions:
  - "Connect daemon startup guarded by main::WEBUI — not SCANNER — per T-05-12"
  - "checkDaemonConnected NOT ported from Spotty-NG (was 429 source, CON-09)"
  - "forceFallbackAP NOT ported (SpotOn uses HTTP stream, no AP-port fallback)"
  - "soc-ogg-*-* guard added alongside son-ogg-*-* — same capability condition"
  - "3s deferred timer for Connect startup allows player list to populate post-LMS-boot"
metrics:
  duration: "~3 minutes"
  completed: "2026-06-01"
  tasks_completed: 2
  tasks_total: 2
  files_created: 1
  files_modified: 1
---

# Phase 05 Plan 03: DaemonManager + Plugin.pm Connect Wiring Summary

**One-liner:** DaemonManager with sync-group master selection, 60s watchdog, and differential sync restart; Plugin.pm wired with deferred Connect init, CON-09 PID exclusion, and soc-ogg passthrough guard.

## What Was Built

### Task 1: Plugins/SpotOn/Connect/DaemonManager.pm (new, 267 lines)

Lifecycle manager for per-player Connect daemons.

**init()** subscribes to three event sources:
- `[['client'], ['new', 'disconnect']]` — debounced initHelpers with 2s delay
- `[['sync']]` — differential restart: stopForSync all helpers then 0.1s initHelpers (CON-15)
- `$prefs->setChange('enableSpotifyConnect')` — react to per-player toggle (CON-10)

**initHelpers()** iterates `Slim::Player::Client::clients()` and determines:
- Sync slave with master having Connect enabled → startHelper for master's clientId
- Sync slave with no master having Connect → pick first alphabetically-sorted slave with Connect enabled as proxy master
- Standalone player with Connect enabled → startHelper
- Otherwise → stopHelper
Ends with 60s watchdog timer.

**startHelper/stopHelper** create/destroy Daemon instances, activating `_streamAlivePoll` (5s fast-poll) when streaming is active.

**helperPids()** returns PIDs of all alive helpers — called by Plugin.pm::_killOrphanedProcesses for CON-09 exclusion.

**uptime()** returns seconds since daemon last started — used by Connect.pm for volume grace period (CON-11).

Explicitly absent: `checkDaemonConnected` (Spotty-NG 429 source), `forceFallbackAP` (not applicable to HTTP streaming architecture).

### Task 2: Plugins/SpotOn/Plugin.pm (modified)

Four independent changes:

1. **prefs->init additions**: `enableSpotifyConnect => 1` (CON-10, default on) and `connectOggOverride => 'auto'` (D-05).

2. **_startConnectDaemons timer**: Inside `if (main::WEBUI)` block, 3s deferred timer calls `_startConnectDaemons` which does `require Plugins::SpotOn::Connect; Plugins::SpotOn::Connect->init()`. The 3s delay (vs 2s for `_autoStartDiscovery`) allows the player list to stabilize. Guarded by `main::WEBUI` satisfies T-05-12 (no daemon init in SCANNER context).

3. **CON-09 PID exclusion in _killOrphanedProcesses**: Replaced `PHASE-5-NOTE` comment with:
   - `$INC{'Plugins/SpotOn/Connect/DaemonManager.pm'}` guard (safe — module may not be loaded)
   - `helperPids()` call builds `%connectPids` hash
   - `next if $connectPids{$pid}` guard in kill loop protects Connect daemon PIDs

4. **soc-ogg-*-* guard in updateTranscodingTable**: When binary lacks passthrough capability, deletes both `son-ogg-*-*` and `soc-ogg-*-*`. Additionally, when per-player `connectOggOverride eq 'pcm'`, deletes `soc-ogg-*-*` (D-05 force-PCM mode).

## CON Requirements Addressed

| Requirement | Addressed By |
|-------------|-------------|
| CON-01 | DaemonManager startHelper creates one daemon per player |
| CON-02 | Sync master selection in initHelpers() |
| CON-06 | Daemon.pm (Plan 02) handles name truncation; DaemonManager passes client object |
| CON-09 | CON-09 PID exclusion in _killOrphanedProcesses |
| CON-10 | enableSpotifyConnect pref default + setChange reaction |
| CON-15 | Differential restart: 0.1s micro-delay after stopForSync |

## Deviations from Plan

None — plan executed exactly as written.

## Threat Mitigations Applied

| Threat ID | Mitigation Applied |
|-----------|-------------------|
| T-05-10 | helperPids() exclusion checked before every kill (CON-09) |
| T-05-11 | startHelper checks alive() before creating new daemon — dead helpers get restart, not duplicate |
| T-05-12 | _startConnectDaemons is inside `if (main::WEBUI)` block in initPlugin |

## Self-Check: PASSED

| Item | Status |
|------|--------|
| Plugins/SpotOn/Connect/DaemonManager.pm | FOUND |
| Plugins/SpotOn/Plugin.pm | FOUND |
| Commit c97f02d (DaemonManager.pm) | FOUND |
| Commit de7e363 (Plugin.pm) | FOUND |
