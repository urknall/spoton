---
phase: 36-session-health-monitoring
plan: "02"
subsystem: daemon
tags: [librespot, health-check, session-monitoring, perl, async-http, status-page]

# Dependency graph
requires:
  - phase: 36-session-health-monitoring/36-01
    provides: Enhanced /health JSON endpoint from Rust daemon with session_valid, session_age_secs, idle_secs fields

provides:
  - Perl polls each daemon's /health endpoint every 60s via SimpleAsyncHTTP
  - session_valid=false triggers immediate daemon restart with INFO log
  - Stale session (age>4h + idle>5min) triggers proactive restart
  - _onHealthResponse, _onHealthError, _restartForHealth methods in DaemonManager.pm
  - _healthCheckCount and _lastHealthSession accessors in Daemon.pm
  - Status Page shows Session / Session Age / Idle Time per daemon when data available
  - 60s watchdog log noise eliminated (3 initHelpers lines downgraded INFO -> DEBUG)

affects: [status-page, daemon-lifecycle, connect-safety]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Health check counter pattern: increment per alive poll cycle, fire every Nth cycle"
    - "Async health check: SimpleAsyncHTTP GET to 127.0.0.1:{port}/health, result stored on daemon object"
    - "Alive guard pattern: return unless $helper && $helper->alive before acting on async callback"

key-files:
  created: []
  modified:
    - Plugins/SpotOn/Unified/Daemon.pm
    - Plugins/SpotOn/Unified/DaemonManager.pm
    - Plugins/SpotOn/Status.pm
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html

key-decisions:
  - "Health check fires every 12th _streamAlivePoll cycle (every 60s at 5s poll interval)"
  - "session_valid=false triggers immediate restart; stale = age>14400s AND idle>300s"
  - "_onHealthError logs warn but does NOT restart (avoids double-restart race with alive poll)"
  - "sessionHealth is undef until first health check runs (~60s after daemon start)"
  - "formatDuration function formats seconds with sub-minute precision (Xm Ys, Xs)"

patterns-established:
  - "Async HTTP callback pattern: store result on daemon object before acting on it"
  - "Health check cycle counter on daemon object, reset to 0 on restart"

requirements-completed: ["P36-GOAL"]

# Metrics
duration: 20min
completed: 2026-06-30
---

# Phase 36 Plan 02: Perl Health-Aware Daemon Monitoring Summary

**Perl polls each daemon's /health endpoint every 60s, restarts on stale/invalid sessions, surfaces session health on Status Page, and silences 800 misleading watchdog log entries per day**

## Performance

- **Duration:** ~20 min
- **Started:** 2026-06-30T06:12:00Z
- **Completed:** 2026-06-30T06:32:54Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments

- Session health monitoring active: every 60s each daemon's /health is polled via SimpleAsyncHTTP; session_valid=false or stale session triggers restart with INFO log
- Connect safety maintained: idle_secs threshold (>5min) prevents restarting daemons during active Connect sessions
- Log cleanup: 3 initHelpers watchdog lines downgraded from INFOLOG to DEBUGLOG — eliminates ~800 misleading "Starting" entries per day from server.log
- Status Page enhancement: Session, Session Age, and Idle Time rows appear per daemon within 60s of daemon start, using green/red dot classes for visual session validity

## Task Commits

Each task was committed atomically:

1. **Task 1: Add health check accessors, monitoring logic, and log cleanup** - `c638263` (feat)
2. **Task 2: Surface session health data on the Status Page** - `5aee307` (feat)

## Files Created/Modified

- `Plugins/SpotOn/Unified/Daemon.pm` - Added _healthCheckCount and _lastHealthSession accessors; initialize counter to 0 in new()
- `Plugins/SpotOn/Unified/DaemonManager.pm` - Added health check counter logic in _streamAlivePoll, three new methods (_onHealthResponse, _onHealthError, _restartForHealth), log level downgrades
- `Plugins/SpotOn/Status.pm` - Added sessionHealth field to daemon hashref in _collectDaemons
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html` - Added formatDuration helper, session health display block in renderDaemon

## Decisions Made

- Health check fires every 12th _streamAlivePoll cycle (60s total) — balances detection speed vs HTTP overhead to localhost
- _onHealthError logs warn but does NOT restart; _streamAlivePoll alive check handles process death, avoiding double-restart race condition
- sessionHealth in the Status Page is omitted (not rendered) when null (health check hasn't run yet) — no placeholder text shown
- formatDuration shows seconds for sub-minute values (Xs) unlike formatUptime which only shows hours+minutes

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

- `perl -c` without LMS stubs fails (Log::Log4perl missing from system Perl). Used stub-based approach matching t/05_perl_syntax.t methodology for verification. Both files pass syntax check. This is expected behavior — plan's `<verify>` block calls `perl -c` from the repo root where LMS .pm files are present via /usr/share/perl5, but Log4perl is not available standalone. Actual verification confirmed via stub approach.

## Threat Surface Scan

No new network endpoints, auth paths, or trust boundary changes introduced. Health check is a GET to 127.0.0.1 (loopback only). sessionHealth data on the Status Page is read-only diagnostic data — same trust level as existing PID/uptime data (T-36-06 accepted in plan's threat model).

## Known Stubs

None — all data wires from daemon accessor to Status Page JSON to UI rendering are complete.

## User Setup Required

None — no external service configuration required. Changes are hot-reloadable (LMS restart). The Rust binary with enhanced /health endpoint (Plan 01) must ship together with these Perl changes.

## Next Phase Readiness

- Perl health monitoring infrastructure complete, ready for integration testing
- Rust binary (Plan 01) + Perl changes (Plan 02) must ship together in the same release
- Testing: start daemon, wait >4h idle, observe "Health check restart" in server.log

## Self-Check

- [x] Plugins/SpotOn/Unified/Daemon.pm exists with _healthCheckCount and _lastHealthSession
- [x] Plugins/SpotOn/Unified/DaemonManager.pm exists with _onHealthResponse, _onHealthError, _restartForHealth
- [x] Plugins/SpotOn/Status.pm exists with sessionHealth field
- [x] Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html exists with formatDuration and sessionHealth display
- [x] Task 1 commit c638263 exists
- [x] Task 2 commit 5aee307 exists

---
*Phase: 36-session-health-monitoring*
*Completed: 2026-06-30*
