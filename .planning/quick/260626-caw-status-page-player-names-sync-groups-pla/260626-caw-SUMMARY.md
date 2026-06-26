---
status: complete
---

# Quick Task 260626-caw: Status page enhancements — Summary

## Changes

### 1. Player Daemon Health — Enhanced display
- **Player name** shown prominently with MAC address in muted monospace beside it
- **Sync group members** displayed when player is synced (e.g., "Group: coffee")
- **Playback status** with track title in Spotify green (e.g., "▶ Time")
- Fixed `getClient()` call — was invoked as method (`->`) instead of function (`::`)
- Track title resolved via `ProtocolHandler->getMetadataFor()` (works for both Browse and Connect)

### 2. Error recording — Extended to 8 error sites
Added `recordError()` calls alongside existing `$log->error()` at:
- **TokenManager.pm**: token refresh failure, binary not found, discovery failed, get-token failed (4 sites)
- **API/Client.pm**: JSON parse error, HTTP error responses (2 sites)
- **Unified/Daemon.pm**: tempfile failure (1 site)
- **ProtocolHandler.pm**: Browse 404 retries exhausted (1 site)

All use `$INC{'Plugins/SpotOn/Status.pm'}` guard to avoid circular dependency.

## Files Modified
- `Plugins/SpotOn/Status.pm` — _collectDaemons: added playing, currentTrack, syncGroup fields; fixed getClient call
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html` — renderDaemon: MAC display, sync group, playback indicator
- `Plugins/SpotOn/API/TokenManager.pm` — 4 recordError calls added
- `Plugins/SpotOn/API/Client.pm` — 2 recordError calls added
- `Plugins/SpotOn/Unified/Daemon.pm` — 1 recordError call added
- `Plugins/SpotOn/ProtocolHandler.pm` — 1 recordError call added

## Verified
- Deployed and tested on Raspi (192.168.13.5), LMS 9.1.1
- Player names "chilly" and "coffee" displayed correctly
- Sync group display verified
- Track title "Time" (Pink Floyd) displayed during playback
- All 13 unit tests pass
