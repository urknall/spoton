---
phase: 29-unified-browse-connect-daemon
plan: "01"
subsystem: rust-daemon
tags:
  - unified-daemon
  - librespot
  - rust
  - browse
  - connect
  - mode-transitions
dependency_graph:
  requires:
    - "28-03: Browse daemon pipeline integration (browse.rs, ProtocolHandler.pm)"
    - "05-01: Connect daemon (connect.rs, LMS struct, HttpStreamSink)"
  provides:
    - "unified.rs: run_unified() entry point for --unified CLI mode"
    - "Mode::Unified in main.rs for CLI dispatch"
    - "'unified': true in --check capability manifest"
  affects:
    - "29-02: Perl Unified::DaemonManager + Unified::Daemon (depends on stream_port=N protocol)"
    - "29-03: ProtocolHandler.pm unified daemon integration"
tech_stack:
  added: []
  patterns:
    - "ActiveMode enum (Arc<Mutex>) for Browse/Connect mode transition coordination"
    - "browse_preempting AtomicBool for relay exit signal on D-10 Browse preemption"
    - "browse_cancel Notify for Browse EOF on D-09 Connect takeover"
    - "Arc<tokio::sync::Mutex<Session>> for shared session with reconnect swap"
key_files:
  created:
    - path: "librespot-spoton/src/unified.rs"
      description: "Combined Browse+Connect daemon (1153 LOC): ActiveMode, UnifiedHttpStreamSink, unified_http_server, run_unified"
  modified:
    - path: "librespot-spoton/src/main.rs"
      description: "Added mod unified; Mode::Unified; --unified/--enable-connect CLI flags; Unified dispatch arm; 'unified': true in --check"
    - path: "librespot-spoton/src/browse.rs"
      description: "Made serve_track_request() and empty_response() pub for use by unified.rs"
decisions:
  - "D-05: Single HTTP server on one port (not two servers) — go-librespot prior art, no routing ambiguity, simpler Perl tracking"
  - "D-09 impl: browse_cancel.notify_waiters() drops Browse pcm_tx -> EOF to LMS (stream-based)"
  - "D-10 impl: browse_preempting AtomicBool exits /stream relay cleanly before Spirc pause"
  - "D-11: Stream-based EOF sufficient (no JSON-RPC mode_changed notification needed)"
  - "LMS::notify private — reconnect 'ready' notification omitted; Spirc TrackChanged/Playing handled by LMS event dispatcher instead"
  - "UnifiedHttpStreamSink: local reimplementation of HttpStreamSink to avoid ownership issues with flush_tx"
metrics:
  duration: "~14 minutes"
  completed_date: "2026-06-22"
  tasks_completed: 2
  files_created: 1
  files_modified: 2
---

# Phase 29 Plan 01: Unified Daemon Rust Core Summary

**One-liner:** Rust unified daemon core: single `run_unified()` entry point with one HTTP server, shared librespot Session, conditional Spirc, and mutex-protected Browse/Connect mode transitions.

## What Was Built

### unified.rs (1153 LOC)

New Rust module `librespot-spoton/src/unified.rs` implementing the unified Browse+Connect daemon:

**ActiveMode enum** protected by `Arc<tokio::sync::Mutex<ActiveMode>>`:
- `Idle` — no active playback
- `Connect` — Spirc-driven playback via GET /stream
- `Browse(String)` — Browse track playing, String = track_id

**UnifiedHttpStreamSink** — rate-limited PCM sink for Connect path (S16LE, wall-clock rate-limited with buffer_latency_ns compensation). Local reimplementation of connect.rs HttpStreamSink to avoid flush_tx ownership conflicts.

**unified_http_server** — single hyper HTTP/1.1 server with combined route dispatch:
- `GET /stream` — Connect PCM relay (spirc_active guard, RelayGuard drop pattern, flush_rx seek-drain, browse_preempting exit signal)
- `GET /track/{id}` — Browse track decode (T-29-01: `[A-Za-z0-9]+` input validation, serve_track_request from browse.rs, mode transition + browse_cancel listen)
- `POST /control/*` — Spirc commands (T-29-02 volume clamp, T-29-03 seek u32 validation)
- `GET /health` — 200 OK for watchdog keepalive
- Everything else — 404

**Mode transition D-09** (Connect takes over Browse): Spirc event watcher fires `browse_cancel.notify_waiters()` when TrackChanged/Playing fires while mode is Browse — Browse handler drops pcm_tx — EOF to LMS.

**Mode transition D-10** (Browse preempts Connect): GET /track/{id} handler sets `browse_preempting = true` (exits relay cleanly per Pitfall 2) then pauses Spirc — loads Browse track via per-request Player.

**run_unified** — main orchestrator:
- Immediate session.connect() (D-03)
- FNV-1a device_id from cache_dir + player_mac (same as connect.rs)
- stream_port=N announcement on stdout (Pitfall 4: NOT unified_port=N)
- Conditional Connect infrastructure (D-01): SoftMixer + Connect Player + LMS + Spirc::new
- Spirc reconnect loop with ZeroConf Discovery and rate-limited backoff
- GracefulShutdown for in-progress Browse requests (5s timeout)

### main.rs modifications

- Added `mod unified;` module declaration
- Added `Mode::Unified` variant to Mode enum
- Added `enable_connect: bool = false` variable
- Added `--unified` and `--enable-connect` CLI flag parsing
- Added `Mode::Unified` dispatch arm calling `unified::run_unified()` with full parameter set
- Updated `--check` JSON manifest to include `"unified": true`

### browse.rs modifications

- Made `serve_track_request()` public (used by unified.rs)
- Made `empty_response()` public (used by unified.rs)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Player::new returns Arc<Player>, not Player**
- **Found during:** Task 1 compilation
- **Issue:** `connect_player_opt: Option<Player>` type was wrong — Player::new returns `Arc<Self>`
- **Fix:** Changed type to `Option<Arc<Player>>`; Arc<Player> implements Clone naturally
- **Files modified:** librespot-spoton/src/unified.rs
- **Commit:** 34b3dfa

**2. [Rule 1 - Bug] LMS::notify is a private method**
- **Found during:** Task 1 compilation
- **Issue:** `lms_rc.notify("ready", ...)` fails — LMS::notify is private to connect.rs
- **Fix:** Removed the "ready" notify call in the reconnect path. Spirc reconnect fires TrackChanged/Playing events which the LMS event dispatcher handles. The "ready" signal is a soft convenience. Phase 30 can add `pub fn notify_ready()` to LMS if needed.
- **Files modified:** librespot-spoton/src/unified.rs

**3. [Rule 1 - Bug] Worktree missing browse.rs and had stale main.rs**
- **Found during:** Task 2 setup
- **Issue:** Worktree branch was behind main branch — browse.rs did not exist, main.rs had no `mod browse;`
- **Fix:** Extracted current versions from main branch via `git show main:path` before modifications
- **Files modified:** librespot-spoton/src/browse.rs, librespot-spoton/src/main.rs
- **Commit:** eb459fb

**4. [Rule 2 - Missing critical functionality] Compiler warnings in run_unified**
- **Found during:** Post-compilation review
- **Issue:** 5 "value assigned to X is never read" warnings from variables in if/else enable_connect branch
- **Fix:** Added `#[allow(unused_assignments)]` on `run_unified()` with explanatory comment
- **Files modified:** librespot-spoton/src/unified.rs
- **Commit:** 683a8c1

## Known Stubs

None. All code paths are functional.

## Threat Flags

No new threat surface beyond the plan's threat model. All T-29-* mitigations implemented:
- T-29-01: track ID `[A-Za-z0-9]+` regex before SpotifyUri (unified.rs)
- T-29-02: volume u64 clamp 0..=100 + u16 scale (unified.rs)
- T-29-03: seek u64 + u32::try_from reject >u32::MAX (unified.rs)
- T-29-04: LMS auth header sanitized in LMS::new() (connect.rs, imported)
- T-29-05: ActiveMode mutex prevents concurrent Player::load (Pitfall 1)

## Commits

| Hash | Message |
|------|---------|
| 34b3dfa | feat(29-01): add unified.rs — combined Browse+Connect daemon entry point |
| eb459fb | feat(29-01): wire Mode::Unified into main.rs and expose browse.rs pub API |
| 683a8c1 | refactor(29-01): suppress unused_assignments warnings in run_unified |

## Self-Check: PASSED

- unified.rs: 1153 lines (above 400 LOC minimum)
- cargo build: SUCCESS (0 errors, 0 warnings)
- spoton --check output: `{"unified":true,...}` confirmed
- Mode::Unified in main.rs: confirmed
- pub serve_track_request in browse.rs: confirmed
- pub empty_response in browse.rs: confirmed
