---
phase: 29-unified-browse-connect-daemon
plan: "03"
subsystem: perl-integration
tags:
  - unified-daemon
  - perl
  - plugin
  - protocol-handler
  - daemonMode
  - integration
dependency_graph:
  requires:
    - "29-01: unified.rs Rust core (--unified flag, stream_port=N protocol)"
    - "29-02: Unified::DaemonManager + Unified::Daemon (helperForClient, streamPortForClient, helperPids)"
    - "28-03: Browse-HTTP pipeline in ProtocolHandler.pm (Phase 28 legacy behavior)"
  provides:
    - "daemonMode pref: 'unified' (default) or 'legacy' migration gate"
    - "Plugin.pm: Unified::DaemonManager lifecycle (init, diagnosticMode restart, orphan PID exclusion, shutdown)"
    - "ProtocolHandler.pm: unified daemon dispatch in formatOverride, canDirectStream, new()"
  affects:
    - "Phase 30: dead-code removal of Browse::* and Connect::* modules (documented in 29-DEAD-CODE.md)"
tech_stack:
  added: []
  patterns:
    - "daemonMode pref gate: ($prefs->get('daemonMode') || 'unified') eq 'unified' pattern"
    - "$INC guard pattern for lazy-loaded DaemonManager shutdown (same as Browse/Connect)"
    - "unified check before legacy check in all ProtocolHandler dispatch functions"
    - "D-09/D-10 mode transitions delegated to Rust daemon (no Perl-side _stopConnectDaemon in unified)"
key_files:
  created:
    - path: ".planning/phases/29-unified-browse-connect-daemon/29-DEAD-CODE.md"
      description: "Phase 30 dead code manifest: 4 Perl modules, 6 Rust paths, Plugin.pm + ProtocolHandler.pm code paths, 2 prefs"
  modified:
    - path: "Plugins/SpotOn/Plugin.pm"
      description: "daemonMode pref + Unified::DaemonManager lifecycle (timer, diagnosticMode, orphan PIDs, shutdown)"
    - path: "Plugins/SpotOn/ProtocolHandler.pm"
      description: "Unified daemon dispatch in formatOverride, canDirectStream, new() Browse + Connect proxy"
decisions:
  - "daemonMode pref (default 'unified') checked at init time — changing requires LMS restart (acceptable for Phase 29)"
  - "Browse::DM + Connect::DM init guarded by daemonMode=legacy — when daemonMode=unified, neither starts"
  - "D-08 _stopConnectDaemon skipped in unified mode — Rust ActiveMode mutex handles D-09/D-10 transitions internally"
  - "Unified dispatch check placed BEFORE legacy checks in all three ProtocolHandler functions — early return pattern"
  - "new() Browse/Connect sync proxies use else-branch for legacy (explicit separation, no accidental fallthrough)"
metrics:
  duration: "~18 minutes"
  completed_date: "2026-06-22"
  tasks_completed: 2
  files_created: 2
  files_modified: 2
---

# Phase 29 Plan 03: Plugin.pm + ProtocolHandler.pm Unified Integration Summary

**One-liner:** daemonMode pref gates unified vs. legacy daemon startup; ProtocolHandler dispatches all Browse and Connect traffic through Unified::DaemonManager when daemonMode=unified.

## What Was Built

### Plugin.pm Changes

**daemonMode pref** added to `prefs->init()`:
```perl
daemonMode => 'unified',   # Phase 29: 'unified' or 'legacy', default 'unified'
```

**Daemon startup gating** in `initPlugin()` (main::WEBUI block):
- `daemonMode=legacy`: existing Connect (3s) + Browse (3.5s) timers start as before
- `daemonMode=unified`: only Unified::DaemonManager init timer (4s) starts — Browse::DM and Connect::DM are NOT initialized

**`_startUnifiedDaemons()`** timer callback function:
```perl
sub _startUnifiedDaemons {
    require Plugins::SpotOn::Unified::DaemonManager;
    Plugins::SpotOn::Unified::DaemonManager->init();
}
```

**diagnosticMode change handler** extended to restart Unified daemons (2s delay, after Connect 1s + Browse 1.5s), guarded by `$INC` guard so it only fires when Unified::DaemonManager is loaded.

**`_killOrphanedProcesses()`** extended with Unified PID exclusion:
```perl
my %unifiedPids;
if ($INC{'Plugins/SpotOn/Unified/DaemonManager.pm'}) {
    %unifiedPids = map { $_ => 1 }
        Plugins::SpotOn::Unified::DaemonManager->helperPids();
}
# ... next if $unifiedPids{$pid};
```
Mitigates T-29-10 (Denial of Service via orphan cleanup killing live unified daemons).

**`shutdownPlugin()`** extended with Unified shutdown and timer kill:
```perl
if ($INC{'Plugins/SpotOn/Unified/DaemonManager.pm'}) {
    Plugins::SpotOn::Unified::DaemonManager->shutdown();
}
Slim::Utils::Timers::killTimers($class, \&_startUnifiedDaemons);
```

### ProtocolHandler.pm Changes

**`formatOverride()`** — Unified check inserted BEFORE legacy Browse/Connect checks:
- Connect URL + daemonMode=unified: checks Unified::DaemonManager for alive daemon, returns 'soc' (dead history URL still returns 'son')
- Browse URL + daemonMode=unified: checks Unified::DaemonManager for alive daemon, returns 'soc'
- Legacy checks (Connect::DM, Browse::DM) unchanged — only reached when daemonMode=legacy

**`canDirectStream()`** — Unified check inserted BEFORE legacy Browse/Connect checks:
- Browse URL + daemonMode=unified: returns `http://host:streamPort/track/{id}` if daemon alive and not synced
- Connect URL + daemonMode=unified: returns `http://host:streamPort/stream` if daemon alive and not synced (also handles dead history URL + streamFormat force-transcode)
- Legacy checks unchanged — only reached when daemonMode=legacy

**`new()`** — Three unified additions:
1. **D-08 mutual exclusion**: Browse URL in unified mode skips `_stopConnectDaemon` — Rust daemon handles D-09/D-10 mode transitions internally via ActiveMode mutex
2. **Browse sync proxy**: `daemonMode=unified` uses `Unified::DaemonManager->helperForClient()` + `_streamPort` for URL substitution (seek offset support included); legacy path uses Browse::DM
3. **Connect sync proxy**: `daemonMode=unified` uses `Unified::DaemonManager->helperForClient()` for `/stream` URL substitution; returns `undef` if no alive unified helper; legacy path uses Connect::DM

All DIAG logging uses `[DIAG]` prefix with `unified_browse`, `unified_connect`, `unified_browse_sync_proxy`, `unified_connect_sync_proxy` markers.

### 29-DEAD-CODE.md

Dead code manifest for Phase 30: 4 Perl modules (Browse::DaemonManager, Browse::Daemon, Connect::DaemonManager, Connect::Daemon), 6 Rust code paths, Plugin.pm code paths, ProtocolHandler.pm code paths, and 2 preferences (browseMode, daemonMode) to remove.

## Checkpoint Pending: Task 3 (Human Verification)

Task 3 is a `checkpoint:human-verify` gate. The orchestrator will present this to the user for manual UAT validation before Phase 30 proceeds.

**What to verify:**
1. Restart LMS — check server log shows Unified helper daemon init (NOT Browse/Connect separately)
2. `ps aux | grep 'spoton --unified'` — one process per connected player
3. **UA-01 Browse playback**: Play a Browse track through the SpotOn menu — should play through unified daemon
4. **UA-02 Connect playback**: Select LMS device in Spotify app, play a track — should play through unified daemon
5. **UA-05 Pure Browse mode**: Player with Connect disabled should show `--unified` WITHOUT `--enable-connect`; player should NOT appear in Spotify app
6. **UA-07 Legacy fallback**: Set daemonMode=legacy in server prefs, restart LMS — old Browse + Connect daemons should start instead of unified

**Rollback**: Set `daemonMode` pref to `'legacy'` in LMS server preferences and restart. Old behavior is completely unchanged behind the legacy gate.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking Issue] Worktree files were at pre-Phase-28 state**
- **Found during:** Task 1 start
- **Issue:** Worktree branch `worktree-agent-ab541f41048c04df7` has `Plugin.pm` (2479 lines) and `ProtocolHandler.pm` (812 lines) from before Phase 28 changes. Main branch has Phase 28 versions (2519 and 940 lines respectively). Modifying the older worktree files would create merge conflicts on merge-back.
- **Fix:** Loaded current main-branch versions via `git show main:` before applying Phase 29-03 changes. This ensures the worktree commit diffs cleanly against main.
- **Files modified:** Both files re-seeded from main before Phase 29 edits applied
- **Impact:** None — net change on merge will correctly show only Phase 29-03 additions

## Known Stubs

None — all dispatch paths are functional. The unified mode requires a running Unified::DaemonManager (loaded lazily from `_startUnifiedDaemons` timer) to be effective; before it loads, dispatch falls through to the non-unified result (e.g., `son` fallback in formatOverride) which is safe.

## Threat Flags

No new threat surface. All T-29-* mitigations applied:
- T-29-10: Unified PID exclusion in `_killOrphanedProcesses` (Plugin.pm)
- T-29-11: daemonMode check before each DaemonManager query; unified check runs first (ProtocolHandler.pm)
- T-29-12: Unified daemon has single port for all routes — no stale port possible

## Commits

| Hash | Message |
|------|---------|
| c250297 | feat(29-03): add daemonMode pref and unified daemon wiring to Plugin.pm |
| 33ada8e | feat(29-03): add unified daemon dispatch to ProtocolHandler.pm |

## Self-Check: PASSED

- FOUND: Plugins/SpotOn/Plugin.pm — daemonMode occurrences: 7
- FOUND: sub _startUnifiedDaemons in Plugin.pm
- FOUND: unifiedPids exclusion in _killOrphanedProcesses
- FOUND: Unified shutdown in shutdownPlugin
- FOUND: Unified diagnosticMode restart block
- FOUND: Plugins/SpotOn/ProtocolHandler.pm — Unified::DaemonManager occurrences: 14
- FOUND: daemonMode=unified checks in ProtocolHandler: 9
- FOUND: 29-DEAD-CODE.md with 5 tables (Perl modules, Rust paths, Plugin.pm, ProtocolHandler.pm, Preferences)
- FOUND commits c250297 and 33ada8e
