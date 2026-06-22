---
phase: 28-persistent-browse-daemon
plan: 01
subsystem: infra
tags: [rust, librespot, hyper, http, pcm, browse-daemon]

# Dependency graph
requires:
  - phase: 05-connect-mode
    provides: HttpStreamSink pattern, http_stream_server pattern, connect.rs hyper/mpsc architecture

provides:
  - librespot-spoton browse.rs: BrowseHttpSink, run_browse(), browse_http_server(), handle_request(), serve_track_request()
  - librespot-spoton main.rs: Mode::Browse variant, --browse arg parsing, browse::run_browse() dispatch, "browse":true in --check manifest

affects: [28-02, 28-03, Browse::Daemon.pm, Browse::DaemonManager.pm, ProtocolHandler.pm]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Per-request Player with shared Session for concurrent track streaming (D-04)"
    - "browse_port=N stdout announcement with flush (same as stream_port pattern in connect.rs)"
    - "BrowseHttpSink: no rate-limiting (no Spirc position sync needed in Browse mode)"
    - "oneshot channel for early Unavailable detection before sending HTTP response headers"

key-files:
  created:
    - librespot-spoton/src/browse.rs
  modified:
    - librespot-spoton/src/main.rs

key-decisions:
  - "Per-request Player (not shared): each GET /track/{id} creates its own Player; Session is shared for audio key caching across requests (Pitfall 2 / D-04)"
  - "500ms early-unavailable detection window: poll status channel before sending HTTP 200 headers; on Unavailable return 404 immediately"
  - "player_mac stored but unused in Phase 28: available for future diagnostics without breaking API"

patterns-established:
  - "browse.rs follows connect.rs structure exactly but without Spirc, rate-limiting, and flush watch-channel"
  - "Track ID input validation: .chars().all(|c| c.is_ascii_alphanumeric()) instead of regex crate (no new dependency)"

requirements-completed: []

# Metrics
duration: 15min
completed: 2026-06-22
---

# Phase 28 Plan 01: Browse Daemon Rust Implementation Summary

**Persistent Browse daemon in Rust: hyper HTTP/1.1 server on dynamic port streams raw S16LE PCM via GET /track/{spotify_id} with per-request Player instances sharing a single Spotify Session**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-06-22T12:30:00Z
- **Completed:** 2026-06-22T12:45:00Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments

- New `librespot-spoton/src/browse.rs` module (416 lines): BrowseHttpSink, run_browse(), browse_http_server(), handle_request(), serve_track_request()
- `spoton --browse` mode operational: binds :0, announces `browse_port=N` to stdout, serves GET /track/{id}
- `spoton --check` now reports `"browse":true` in capability manifest
- Audio key caching enabled: Cache::new with audio_path=Some(cache_dir) lets librespot reuse audio keys across per-request Players
- Input validation: track ID must match [A-Za-z0-9]+ before SpotifyUri::from_uri() (T-28-01 Tampering mitigation)
- No librespot_connect import: Browse daemon is invisible to Spotify app (Pitfall 1)
- No rate-limiting in BrowseHttpSink: PCM delivered at librespot decode speed (Pitfall 3)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create browse.rs** - `8f1cae0` (feat)
2. **Task 2: Add Mode::Browse to main.rs** - `ab5b2e2` (feat)

## Files Created/Modified

- `librespot-spoton/src/browse.rs` — New file: Browse daemon HTTP server, BrowseHttpSink, run_browse(), serve_track_request()
- `librespot-spoton/src/main.rs` — Added: mod browse, Mode::Browse, --browse arg, dispatch block, "browse":true in --check JSON

## Decisions Made

- **Per-request Player with shared Session**: Each GET /track creates a new Player but shares the Session. This is the recommended approach from RESEARCH.md Open Question 1. Audio keys are cached at Session level so per-request Players still benefit from key caching across consecutive requests.
- **Early Unavailable detection via oneshot channel**: When PlayerEvent::Unavailable fires before any PCM is sent, a 500ms timeout window allows returning HTTP 404 before sending the response headers. If no early status arrives, streaming proceeds with HTTP 200. This avoids sending 200 and then an empty body for unavailable tracks.
- **No regex crate**: Track ID validation uses `.chars().all(|c| c.is_ascii_alphanumeric())` — achieves the same [A-Za-z0-9]+ constraint without adding a dependency.

## Deviations from Plan

None — plan executed exactly as written. All pitfalls from RESEARCH.md were addressed as specified.

## Issues Encountered

None — build succeeded on first attempt. All referenced crates (hyper, tokio, bytes, http-body-util, tokio-stream, librespot-*) were already in Cargo.toml.

## Next Phase Readiness

- Browse daemon Rust binary is complete and compiled
- Perl side (Browse::DaemonManager.pm, Browse::Daemon.pm) to be implemented in Plan 02
- ProtocolHandler.pm and Plugin.pm integration in Plan 03
- Phase 28 Plan 01 provides the binary foundation that plans 02 and 03 depend on

## Self-Check: PASSED

- FOUND: librespot-spoton/src/browse.rs (416 lines)
- FOUND: librespot-spoton/src/main.rs (modified)
- FOUND: .planning/phases/28-persistent-browse-daemon/28-01-SUMMARY.md
- FOUND commit 8f1cae0: feat(28-01): add browse.rs
- FOUND commit ab5b2e2: feat(28-01): add Mode::Browse to main.rs
- cargo build: Finished dev profile (no errors)
- spoton --check output: contains "browse":true

---
*Phase: 28-persistent-browse-daemon*
*Completed: 2026-06-22*
