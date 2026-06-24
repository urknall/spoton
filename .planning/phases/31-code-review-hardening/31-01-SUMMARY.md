---
phase: 31-code-review-hardening
plan: 01
subsystem: connect
tags: [rust, librespot, spirc, event-dispatcher, reconnect, zeroconf]

requires:
  - phase: 29-unified-daemon
    provides: "Unified daemon with Spirc, ZeroConf reconnect, event dispatcher"
provides:
  - "Event dispatcher respawn on Spirc reconnect (R-WR-07)"
  - "JoinHandle tracking for event dispatcher lifecycle"
affects: [connect, unified-daemon]

tech-stack:
  added: []
  patterns: ["Inline spawn duplication for closures capturing many locals (same as current_spirc_task pattern)"]

key-files:
  created: []
  modified:
    - librespot-spoton/src/unified.rs

key-decisions:
  - "All 3 reconnect paths (ZeroConf, Browse-triggered, Spirc-died) flow through the same reconnect-success handler — single respawn site covers all cases"
  - "lms.clone() (flush_tx=None) is correct for dispatcher respawn — seek-flush goes through UnifiedHttpStreamSink's own flush_tx"

patterns-established:
  - "Event dispatcher lifecycle: track JoinHandle, abort before respawn, abort on shutdown"

requirements-completed: [R-WR-07]

duration: 3min
completed: 2026-06-24
---

# Phase 31 Plan 01: Event Dispatcher Respawn Summary

**Spirc event dispatcher now respawns after ZeroConf reconnect, preventing permanent loss of LMS Connect notifications (start/change/stop/volume/seek)**

## Performance

- **Duration:** 3 min
- **Started:** 2026-06-24T14:18:42Z
- **Completed:** 2026-06-24T14:21:58Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments
- Event dispatcher JoinHandle tracked in `event_dispatcher_handle` variable throughout the Connect event loop
- Old dispatcher aborted and new one spawned with fresh player event channel after every successful Spirc reconnect
- All three reconnect paths covered: ZeroConf credential rotation, Browse-triggered session reconnect, and Spirc-task-died reconnect (all flow through the same unified handler)
- Graceful shutdown aborts the dispatcher to prevent orphaned tokio tasks

## Task Commits

Each task was committed atomically:

1. **Task 1: Respawn event dispatcher after Spirc reconnect** - `f851ebf` (fix)

## Files Created/Modified
- `librespot-spoton/src/unified.rs` - Added event_dispatcher_handle tracking, abort+respawn in reconnect handler, abort in shutdown

## Decisions Made
- All 3 reconnect paths converge into the same "Reconnect with new credentials" handler, so a single respawn site covers all cases (no need for 3 separate respawn blocks)
- Used `lms_for_dispatcher` (a clone of the original `lms`) for reconnect dispatcher spawns. `flush_tx=None` in the clone is correct because the seek-flush mechanism goes through UnifiedHttpStreamSink's own `flush_tx`, and the Seeked event uses `notify()` which is unaffected by `flush_tx`
- Inline duplication of the spawn block (not refactored to a function) because the closure captures many local variables — same pattern as `current_spirc_task`

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- R-WR-07 is resolved; event dispatcher survives ZeroConf reconnects
- Ready for Plan 31-02 or any other phase

---
*Phase: 31-code-review-hardening*
*Completed: 2026-06-24*
