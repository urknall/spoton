---
phase: 32-status-page
plan: 01
subsystem: status-page-backend
tags: [status, telemetry, ring-buffer, json-endpoint]
dependency_graph:
  requires: []
  provides: [status-json-endpoint, api-telemetry-counters, error-ring-buffer]
  affects: [API/Client.pm, Plugin.pm]
tech_stack:
  added: []
  patterns: [ring-buffer, INC-guard, addRawFunction, statusSnapshot]
key_files:
  created:
    - Plugins/SpotOn/Status.pm
    - t/13_status_page.t
  modified:
    - Plugins/SpotOn/API/Client.pm
    - Plugins/SpotOn/Plugin.pm
    - t/05_perl_syntax.t
decisions:
  - "Used undef instead of JSON::null for streamPort null values (cross-platform compat with JSON::PP)"
  - "Ring-buffer size set to 30 entries per Claude's discretion range"
  - "Pre-loaded Plugin stub in test to avoid require ordering issues"
metrics:
  duration: 6m49s
  completed: 2026-06-25T15:45:43Z
  tasks_completed: 2
  tasks_total: 2
  test_count: 13
  files_changed: 5
---

# Phase 32 Plan 01: Status Page Backend Summary

Status.pm JSON endpoint with ring-buffer error history, API telemetry counters in Client.pm, and 13-test coverage file.

## Commits

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Create Status.pm module with JSON endpoint and ring-buffer | 6410a35 | Plugins/SpotOn/Status.pm |
| 2 | Add API telemetry counters, Status registration, and tests | b1f89e7 | Client.pm, Plugin.pm, t/05_perl_syntax.t, t/13_status_page.t |

## What Was Built

**Status.pm** (161 lines): New module providing the `/plugins/SpotOn/status/data` JSON endpoint. Aggregates five data categories:
- **daemons**: Per-player daemon health from DaemonManager (mac, name, alive, pid, uptime, connectEnabled, streamPort)
- **api**: Telemetry snapshot from Client.pm (inflightCount, apiRequestCount, api429Count, rateLimitedOwn, rateLimitedBundled)
- **errors**: Ring-buffer of last 30 errors in reverse chronological order
- **tokens**: Account count and discovery status from TokenManager
- **system**: Plugin/binary/LMS/Perl versions and OS (cached in module variable, loaded once)

**Client.pm extensions**: Two new module-level counters (`$apiRequestCount`, `$api429Count`) alongside existing `$inflightCount`. New `statusSnapshot()` method returns 5-key hashref. `reset()` clears all three counters. 429 handler increments counter and calls INC-guarded `Status->recordError()`.

**Plugin.pm**: Added `require Plugins::SpotOn::Status; Plugins::SpotOn::Status->new()` in the WEBUI block, between Settings::Player and the auto-start timers.

**t/13_status_page.t** (13 tests): Covers recordError push behavior, ring-buffer trimming at 30, correct field storage (ts/level/module/message), reverse chronological ordering, _systemInfo caching, statusSnapshot key completeness, and reset counter zeroing.

**t/05_perl_syntax.t**: Added Status.pm to the syntax-checked file list. Added `addRawFunction` to Slim::Web::Pages stub and new Slim::Web::HTTP stub for `addHTTPResponse`.

## Verification Results

- `prove t/05_perl_syntax.t` -- PASS (8/8 tests, Status.pm syntax check green)
- `prove t/13_status_page.t` -- PASS (13/13 tests)
- `prove t/` -- PASS (404/404 tests across 13 files, zero regressions)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] JSON::null not available under test stubs**
- **Found during:** Task 2 (test execution)
- **Issue:** `JSON::null` bareword caused strict-subs compilation failure because the test stub for JSON::XS::VersionOneAndTwo doesn't export `JSON::null`.
- **Fix:** Changed `JSON::null` to `undef` in Status.pm -- `to_json()` encodes undef as JSON null correctly.
- **Files modified:** Plugins/SpotOn/Status.pm
- **Commit:** b1f89e7

**2. [Rule 3 - Blocking] Missing stubs for cross-module calls in test**
- **Found during:** Task 2 (test execution)
- **Issue:** `_systemInfo()` calls Helper->get() and Plugin->_pluginDataFor() which require deep LMS infrastructure stubs. Test crashed on `Slim::Utils::Misc::findbin` call inside Helper.pm.
- **Fix:** Added lightweight stubs for Plugins::SpotOn::Helper, Plugins::SpotOn::Plugin, Plugins::SpotOn::Unified::DaemonManager, and Plugins::SpotOn::API::TokenManager in test stub_dir. Pre-loaded Plugin stub before test execution.
- **Files modified:** t/13_status_page.t
- **Commit:** b1f89e7

**3. [Rule 3 - Blocking] Missing Slim::Web::HTTP stub for _jsonResponse**
- **Found during:** Task 2 (syntax test)
- **Issue:** Status.pm's _jsonResponse calls `Slim::Web::HTTP::addHTTPResponse()` but the syntax test had no stub for Slim::Web::HTTP (only Slim::Web::HTTP::CSRF existed).
- **Fix:** Added Slim::Web::HTTP stub with `addHTTPResponse` in t/05_perl_syntax.t.
- **Files modified:** t/05_perl_syntax.t
- **Commit:** b1f89e7

## Self-Check: PASSED

- Plugins/SpotOn/Status.pm: FOUND
- t/13_status_page.t: FOUND
- 32-01-SUMMARY.md: FOUND
- Commit 6410a35: FOUND
- Commit b1f89e7: FOUND
