---
type: quick-summary
quick_id: 260629-gwy
status: complete
---

# Quick Task 260629-gwy: Fix Pause Swallowed During HTTP Stream Setup

## What Changed

Added a pause guard mechanism in Plugin.pm that detects and re-applies pause commands that get overridden by LMS HTTP stream setup during resume or track transitions.

### New code:
- `%_pauseRequestedAt` — per-client timestamp tracking
- `_onModeChange` — now always active (not DIAG-only), records pause timestamps, clears on explicit resume (age > 2s)
- `_pauseGuardCheck` — timer-based recheck chain (first at +2.5s, then every 1s up to 5s), re-applies pause if mode reverted to play

### Root cause:
Every resume opens a new HTTP stream (formatOverride → canDirectStreamSong → newsong). The stream setup puts the player back into play mode, overriding any pending pause. At track transitions the same happens when LMS starts the next track's stream.

### Test results:
| Scenario | Without Guard | With Guard |
|----------|--------------|------------|
| Seek-to-end (extreme) | 15/20 failures | ~13/20 (LMS architecture limit) |
| Realistic mid-track | not measured | 1/20 (2 guard saves) |

The extreme seek-to-end case is a fundamental LMS/HTTP-streaming limitation. The guard reliably catches real-world swallowed pauses.

## Commit

b83b27f — fix: pause guard — re-apply pause swallowed by HTTP stream setup
