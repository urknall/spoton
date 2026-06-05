---
status: resolved
trigger: "Connect history replay bypasses getNextTrack when phone is connected — _onPause forwards unpause to Connect daemon before ProtocolHandler can translate URL"
created: 2026-06-05
updated: 2026-06-05
---

# Debug: connect-history-replay

## Symptoms

- expected: Connect history tracks in "What Was That Tune" replay via Browse pipeline (getNextTrack translates spotify://connect-TIMESTAMP to spotify://track:ID)
- actual: When phone is paired with Connect, _onPause intercepts the LMS unpause event and forwards /control/play to the Connect daemon. Daemon sends resume event. _connectEvent re-enters Connect mode via new playlist play. getNextTrack is never called.
- errors: Either 503 from idle daemon (no active session) or daemon starts new Connect stream instead of Browse playback
- timeline: Discovered 2026-06-05 during Phase 11 verification
- reproduction: Connect phone → play track via Connect → stop → open "What Was That Tune" → click play on old Connect track

## Key Evidence (from live debugging session)

- _onPause (Connect.pm:328) catches unpause event and sends /control/play to daemon
- _onPause fires because isSpotifyConnect returns true while phone is still connected
- _connectEvent (Connect.pm:560) sees "Resume while not on Connect stream" and creates new connect URL
- getNextTrack (ProtocolHandler.pm) is never reached — Connect event chain handles playback first
- Without phone connected: getNextTrack IS called, but canDirectStream still finds the discovery daemon and returns HTTP URL → 503
- Cache key mismatch was also a bug: Connect.pm used streamUrl (HTTP proxy) instead of track->url for cache key — FIXED in ca9ca6e

## Current Focus

- hypothesis: "RESOLVED"
- next_action: "Verify in live environment"
- reasoning_checkpoint: |
    Root cause confirmed and fixed. See Resolution section.

## Evidence

- _onPause checks isSpotifyConnect() which returns true as long as $_activeConnectPlayer is set (even after phone pauses)
- History URL spotify://connect-TIMESTAMP has same pattern as live Connect URL — no distinction previously
- _isDeadHistoryUrl() helper added: checks cache for spotifyUri field — present only on previously-played Connect tracks
- Cache entry with spotifyUri = reliable signal that URL is a dead history record, not live session

## Eliminated

- Cache key bug (ca9ca6e): already fixed, not the cause here
- Without phone path: getNextTrack IS reached but canDirectStream returns HTTP URL for daemon → 503 (separate issue, canDirectStream already has _translatedConnectUrls guard — was already correct)

## Resolution

- root_cause: "_onPause forwards LMS unpause to Connect daemon for ANY connect- URL when isSpotifyConnect is true. History replay URLs (spotify://connect-TIMESTAMP with cached spotifyUri) are indistinguishable from live URLs by URL pattern alone. The forwarded /control/play causes a spurious resume event. _connectEvent's resume handler sees the connect- URL as actuallyInConnect=true and re-enters Connect mode instead of letting Browse/getNextTrack handle translation."
- fix: "Added _isDeadHistoryUrl() helper that detects history URLs by checking cache for spotifyUri presence. Applied in two places: (1) _onPause: skip daemon forwarding on unpause when current song URL is a dead history URL; (2) _connectEvent resume handler: extend $actuallyInConnect check to exclude dead history URLs, and explicitly drop spurious resume events for history replay URLs."
- verification: "Manual test: Connect phone → play via Connect → stop → open What Was That Tune → click history track. Expected: Browse pipeline plays the track via spotify://track:ID translation. isSpotifyConnect still true but _onPause now suppresses daemon forward. No spurious resume event reaches _connectEvent."
- files_changed:
  - Plugins/SpotOn/Connect.pm: added _isDeadHistoryUrl(), modified _onPause (history-replay guard before daemon forward), modified _connectEvent resume handler (extended actuallyInConnect check + explicit drop of spurious resumes)
