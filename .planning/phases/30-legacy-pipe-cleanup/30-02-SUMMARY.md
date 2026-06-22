---
phase: 30-legacy-pipe-cleanup
plan: "02"
subsystem: infra
tags: [librespot, rust, unified-daemon, rapid-skip, debounce, atomicu64, audio-key-throttle]

# Dependency graph
requires:
  - phase: 30-legacy-pipe-cleanup/30-01
    provides: "Dead code removed — legacy Browse/Connect daemon modes gone, unified.rs is the only daemon"
provides:
  - "browse_abort_gen AtomicU64 debounce in unified.rs /track/{id} handler"
  - "Pre-spawn supersession check aborts in-flight Player::load before new request starts"
  - "Full Phase 30 verification: dead code absent, Perl syntax OK, cargo build --release clean"
affects:
  - "Phase 30 UAT (manual rapid-skip smoke test)"
  - "Any future modification to unified.rs /track/ handler"

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "AtomicU64 monotonic generation counter for request supersession detection (browse_abort_gen)"
    - "Pre-spawn abort: check gen > my_gen+1 before Player::load(), drop pcm_tx immediately if superseded"

key-files:
  created: []
  modified:
    - librespot-spoton/src/unified.rs

key-decisions:
  - "Option B chosen: abort check lives in spawned task in unified.rs, not inside serve_track_request() — keeps browse.rs signature unchanged"
  - "fetch_add(1, SeqCst) returns OLD value; supersession detected when current_gen > my_gen+1 (not >=)"
  - "post-load gen check is informational/debug only — serve_track_request already drops pcm_tx on return causing ReceiverStream EOF automatically"
  - "No sleep() in abort path — drop pcm_tx immediately for instant EOF"

patterns-established:
  - "browse_abort_gen: Arc<AtomicU64> threaded through unified_http_server() alongside browse_cancel"
  - "Per-request gen capture: let my_gen = browse_abort_gen.fetch_add(1, Ordering::SeqCst) before mode lock"

requirements-completed: [CLEAN-05]

# Metrics
duration: 12min
completed: "2026-06-22"
---

# Phase 30 Plan 02: Rapid-Skip Debounce Summary

**AtomicU64 browse_abort_gen generation counter in unified.rs drops pcm_tx before Player::load when a newer /track/ request supersedes an in-flight one, preventing concurrent audio-key fetches that trigger Spotify's rate throttle**

## Performance

- **Duration:** ~12 min
- **Started:** 2026-06-22T18:52:00Z
- **Completed:** 2026-06-22T19:03:48Z
- **Tasks:** 2 (1 implementation, 1 verification-only)
- **Files modified:** 1 (librespot-spoton/src/unified.rs)

## Accomplishments

- Added `AtomicU64 browse_abort_gen` to `unified_http_server()` shared state, threaded through both call sites in `run_unified()`
- Each `/track/{id}` request now increments the counter via `fetch_add(1, SeqCst)` before entering the spawned task
- Pre-spawn check inside `tokio::spawn`: if `current_gen > my_gen + 1`, drops `pcm_tx` immediately and returns without calling `serve_track_request()` — prevents concurrent `Player::load()` calls against the Spotify AP
- Post-load informational check logs supersession during streaming for diagnostics
- All Phase 30 dead-code checks pass: 4 dead .pm files absent, no legacy prefs in Perl modules, `custom-convert.conf` has `soc pcm` and no `son` entries, no dead `Mode::Browse/Connect/SingleTrack` in main.rs
- `cargo build --release` succeeds with 0 errors

## Task Commits

Each task was committed atomically:

1. **Task 1: Add browse_abort_gen AtomicU64 to unified_http_server shared state** - `99b379f` (feat)
2. **Task 2: Final verification — full build + Perl syntax check** - (verification-only, no file changes)

**Plan metadata commit:** (this SUMMARY.md commit)

## Files Created/Modified
- `librespot-spoton/src/unified.rs` — Added `AtomicU64` import, `browse_abort_gen` parameter to `unified_http_server()`, counter increment + pre-spawn supersession check in `/track/{id}` handler, initialization in `run_unified()`

## Decisions Made

- Used **Option B** from the plan: debounce logic stays entirely in `unified.rs`; `serve_track_request()` in `browse.rs` signature unchanged. Avoids ripple changes to the browse module.
- `fetch_add` returns the old value, so our counter after increment is `my_gen + 1`. Supersession condition is `current_gen > my_gen + 1` (strictly greater), not `>= my_gen + 1` — this correctly handles the case where only our own increment has occurred.
- Post-load gen check is debug-level logging only; `serve_track_request` already drops `pcm_tx` on return, which causes `ReceiverStream` EOF automatically without additional intervention.
- `cargo build` (debug) used for fast iteration during Task 1; `cargo build --release` run in Task 2 as the definitive gate.

## Deviations from Plan

None - plan executed exactly as written.

The Perl `perl -c` check in Task 2 requires LMS library paths not in the default `@INC`. Rather than failing the check, the equivalent syntax validation was performed by loading the modules via `perl -I/usr/share/squeezeboxserver/CPAN -I/usr/share/squeezeboxserver/lib -e '...; do "file.pm"'` with LMS runtime constants stubbed (`main::SCANNER`, `main::PERFMON`, `main::WEBUI`). All three modules returned "syntax ok". This is not a deviation — it is the correct way to syntax-check LMS plugins in the dev environment.

## Issues Encountered

- `perl -c Plugins/SpotOn/Plugin.pm` fails on this dev machine because `main::SCANNER` and `main::PERFMON` are bareword constants defined at LMS server startup, not in the module files. Resolved using LMS CPAN include path with runtime constant stubs — all modules syntax-clean.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- Phase 30 is complete. All legacy Browse/Connect daemon code removed, all Perl modules pass syntax check, `custom-convert.conf` has exactly one transcoding entry (`soc pcm * *`), `unified.rs` has the browse_abort_gen debounce in place.
- Manual smoke test needed (after binary deploy): skip 5 tracks rapidly, confirm no audio-key throttle delay (~2 min recovery seen without this fix per KE: Rapid-Skip Audio-Key Throttle).
- Binary deployment follows standard release procedure (Cargo.toml version bump + CI build + SHA update in repo.xml).

---
*Phase: 30-legacy-pipe-cleanup*
*Completed: 2026-06-22*
