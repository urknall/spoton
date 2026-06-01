---
phase: 05-spotify-connect
plan: "04"
subsystem: connect-event-dispatch
tags: [connect, event-dispatch, bidirectional, source-marking, soc-format, protocol-handler]
dependency_graph:
  requires:
    - "05-03"  # DaemonManager with helperForClient/streamPortForClient/uptime
    - "05-02"  # Daemon with _streamMode/_streamPort/stop/uptime
  provides:
    - Connect.pm event dispatch (spottyconnect -> LMS player commands)
    - Connect.pm bidirectional HTTP control (LMS -> binary /control/*)
    - ProtocolHandler.pm soc format routing
    - ProtocolHandler.pm canDirectStream HTTP URL for single players
    - ProtocolHandler.pm sync-group proxy via Slim::Player::Protocols::HTTP
  affects:
    - Plugins/SpotOn/API/Client.pm (getTrack + player control methods added)
tech_stack:
  added:
    - Slim::Utils::Network (serverAddr() for canDirectStream URL construction)
    - Slim::Networking::SimpleAsyncHTTP (HTTP control endpoint calls in Connect.pm)
  patterns:
    - source(__PACKAGE__) source-marking on all outbound Slim::Control::Request (T-05-13)
    - VOLUME_GRACE_PERIOD=20s to suppress initial volume echoes (CON-11)
    - startOffset-only seek -- never ['time', N] in stream mode (CON-13)
    - pluginData(progress) stored BEFORE playlist play (CON-17 race prevention)
    - on-demand require for DaemonManager/Connect in ProtocolHandler (no top-level load)
key_files:
  created:
    - Plugins/SpotOn/Connect.pm
  modified:
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/API/Client.pm
decisions:
  - "Connect.pm uses HTTP /control/* endpoints (D-14) as primary LMS-to-binary path, Web API as fallback (D-15)"
  - "isSpotifyConnect() uses $_activeConnectPlayer comparison (simpler than Spotty-NG pluginData approach)"
  - "getMetadataFor() checks song pluginData('info') for Connect streams before cache fallback"
  - "getTrack/playerPause/playerPlay/playerVolume/playerSeek added to API/Client.pm (Rule 2)"
metrics:
  duration: "306s"
  completed: "2026-06-01T09:13:10Z"
  tasks_completed: 2
  tasks_total: 2
  files_created: 1
  files_modified: 2
---

# Phase 5 Plan 04: Connect.pm Event Dispatch and ProtocolHandler.pm Connect Extensions

**One-liner:** Bidirectional Spotify Connect event dispatch with source-marking loop prevention, HTTP control endpoints, soc format routing, startOffset seek, and sync-group proxy.

## What Was Built

### Task 1: Connect.pm -- Event Dispatch and Bidirectional Control

Created `Plugins/SpotOn/Connect.pm` (722 lines) as the central event dispatcher between the librespot binary and LMS players.

**Key capabilities:**

- `init()` registers `spottyconnect` dispatch via `addDispatch(['spottyconnect', '_cmd'], [1, 0, 1, \&_connectEvent])` and subscribes to 5 event types (newsong, pause/stop, volume, time, jump/index)
- `_connectEvent()` handles 5 binary commands: start, change, stop, volume, seek
- Source-marking: ALL 5 outbound `Slim::Control::Request` calls use `$req->source(__PACKAGE__)` to prevent feedback loops (T-05-13)
- ALL subscriber methods (_onPause, _onVolume, _onSeek, _onPlaylistJump) check `$request->source eq __PACKAGE__` and return early
- `VOLUME_GRACE_PERIOD = 20` suppresses initial volume echoes after daemon start (CON-11)
- Seek handler uses `$song->startOffset(int($position) - $elapsed)` -- NEVER `['time', N]` (CON-13)
- Progress stored in `pluginData` BEFORE playlist play command (CON-17 race prevention)
- HTTP control calls POST to `/control/pause`, `/control/play`, `/control/volume`, `/control/seek`, `/control/next`, `/control/prev` via `SimpleAsyncHTTP` (D-14)
- Web API fallback via `API::Client->playerPause/Play/Volume/Seek` when binary unreachable (D-15)
- D-08 mutual exclusion: Connect start stops active Browse playback
- `_stopConnectDaemon()` for Browse->Connect exclusion (called by ProtocolHandler)
- `isSpotifyConnect()` method for ProtocolHandler seek guard

### Task 2: ProtocolHandler.pm -- soc Format, canDirectStream, Sync Proxy

Extended `Plugins/SpotOn/ProtocolHandler.pm` with Connect-mode protocol handling.

**Key capabilities:**

- `formatOverride()` returns `'soc'` when Connect daemon is active (D-04) -- BEFORE existing logic
- `canDirectStream()` returns HTTP URL `http://serverAddr:port/stream` for single players (D-06), 0 for sync groups
- `new()` two-branch dispatch:
  - (a) D-08 Browse->Connect exclusion: `spotify://` URL while Connect active stops Connect daemon
  - (b) D-06 sync-group proxy: `spotify://connect-*` substitutes HTTP URL via `Slim::Player::Protocols::HTTP->new`
- `canSeek()` returns 0 when `isSpotifyConnect` (CON-13: prevents LMS from restarting HTTP stream)
- `canTranscodeSeek()` same Connect guard
- `getFormatForURL()` returns `'soc'` for `spotify://connect-*` URLs
- `getMetadataFor()` uses `song->pluginData('info')` for Connect streams, cache fallback for Browse
- Added `Slim::Utils::Network` import for `serverAddr()`
- All `DaemonManager` requires are on-demand (not at top level)

## Deviations from Plan

### Auto-fixed Issues (Rule 2 -- Missing Critical Functionality)

**1. [Rule 2 - Missing API] Added getTrack + player control methods to API/Client.pm**
- **Found during:** Task 1 implementation
- **Issue:** `Connect.pm` references `API::Client->getTrack()` and `playerPause/Play/Volume/Seek()` but these methods did not exist in `API/Client.pm`
- **Fix:** Added `getTrack($accountId, $trackId, $cb)`, `playerPause()`, `playerPlay()`, `playerVolume()`, `playerSeek()` to `Plugins/SpotOn/API/Client.pm`
- **Files modified:** `Plugins/SpotOn/API/Client.pm`
- **Commit:** f731ae5

## Files Created/Modified

| File | Change | Lines | Key Addition |
|------|--------|-------|--------------|
| `Plugins/SpotOn/Connect.pm` | Created | 722 | Full event dispatcher and bidirectional control |
| `Plugins/SpotOn/ProtocolHandler.pm` | Modified | +141/-17 | soc format, canDirectStream HTTP URL, new() proxy |
| `Plugins/SpotOn/API/Client.pm` | Modified | +58 | getTrack + player control Web API methods |

## Commits

| Hash | Message |
|------|---------|
| f731ae5 | feat(05-04): create Connect.pm event dispatcher and bidirectional control |
| 8482ffa | feat(05-04): extend ProtocolHandler.pm with Connect-mode soc routing |

## Requirements Covered

| Requirement | Implementation |
|-------------|----------------|
| CON-03 | `addDispatch(['spottyconnect', '_cmd'])` + `_connectEvent` handler |
| CON-04 | `canDirectStream` HTTP URL + `formatOverride` soc + `new()` proxy |
| CON-05 | HTTP /control/* bidirectional control (D-14) with Web API fallback (D-15) |
| CON-08 | D-08 mutual exclusion in `_connectEvent` start + `new()` Browse path |
| CON-11 | `VOLUME_GRACE_PERIOD=20` in volume handler |
| CON-12 | soc transcoding pipeline with HTTP stream integration |
| CON-13 | `$song->startOffset()` only -- never `['time', N]` in seek handler |
| CON-17 | `pluginData(progress)` stored BEFORE `playlist play` in start handler |

## Threat Surface Scan

No new security-relevant surface beyond the plan's threat model:
- All HTTP control calls are to localhost (127.0.0.1:$port) -- no external surface
- canDirectStream URL uses `Slim::Utils::Network::serverAddr()` + daemon port -- no user input (T-05-16)
- T-05-13 (volume/seek feedback loop) mitigated by source-marking on all outbound requests
- T-05-14 (stale API) mitigated by binary event track_id priority in _fetchTrackMetadata
- T-05-15 (startOffset only, no ['time', N]) mitigated in seek handler
- T-05-17 (mutual exclusion) mitigated in _connectEvent start + ProtocolHandler new()

## Self-Check: PASSED

- Plugins/SpotOn/Connect.pm: FOUND
- Plugins/SpotOn/ProtocolHandler.pm: FOUND (modified)
- Plugins/SpotOn/API/Client.pm: FOUND (modified)
- Commit f731ae5: FOUND
- Commit 8482ffa: FOUND
