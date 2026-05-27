---
phase: 02-auth-api-foundation
plan: "00"
subsystem: testing
tags: [perl, test-more, tdd, lms-stubs, mock-binary]

# Dependency graph
requires:
  - phase: 01-skeleton
    provides: existing test infrastructure (t/05_perl_syntax.t, t/06_binary_check.t), write_stub helper, LMS stub patterns

provides:
  - t/07_token_manager.t with AUTH-01..05 behavioral contracts and mock binary infrastructure
  - t/08_api_client.t with API-01..06 behavioral contracts and SimpleAsyncHTTP mock
  - t/09_settings.t with AUTH-04/05 immediate filesystem tests and AUTH-06/i18n skip-guarded contracts
  - Reusable Slim::Utils::Cache stub that records TTL for inspection
  - Reusable Slim::Utils::Timers stub that records setTimer/killTimers calls
  - Mock spoton binary (shell script) that handles --get-token, --authenticate, --check

affects: [02-02-token-manager, 02-03-api-client, 02-04-settings-account-crud, 02-05-integration-gate]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "SKIP-guarded test blocks: skip N unless -f module_path; guards all module-dependent assertions"
    - "Sentinel-based skip: check for presence of one key (PLUGIN_SPOTON_ACTIVE_ACCOUNT) to activate entire i18n block"
    - "Slim::Utils::Cache stub with ttl() accessor for per-key TTL inspection"
    - "Slim::Utils::Timers stub with reset_calls() and package-global arrays for call capture"
    - "SimpleAsyncHTTP mock with auto_mode flag (success/error_429/error_generic/none) for controlled callback invocation"
    - "Mock binary as shell script in tempdir — handles --get-token/--authenticate/--check sub-commands"

key-files:
  created:
    - t/07_token_manager.t
    - t/08_api_client.t
    - t/09_settings.t
  modified: []

key-decisions:
  - "i18n tests use sentinel-based skip (PLUGIN_SPOTON_ACTIVE_ACCOUNT presence) rather than immediate assertions to avoid false failures before Plan 02-03 adds strings"
  - "RATE_LIMIT_CACHE_KEY accessed via method call (->RATE_LIMIT_CACHE_KEY()) not as bareword to avoid strict-subs compile error before Client.pm exists"
  - "AUTH-04/05 filesystem chmod tests run immediately (no skip guard) since they test OS-level behavior, not module behavior"

patterns-established:
  - "Pattern: write_stub helper copied verbatim from t/05_perl_syntax.t — canonical LMS stub creation"
  - "Pattern: tempdir(CLEANUP => 1) for all stub and mock directories — no filesystem leaks"
  - "Pattern: Prefs stub interpolates $cache_dir at stub-write time via heredoc with double-quotes"

requirements-completed: [AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, AUTH-06, API-01, API-02, API-03, API-04, API-06]

# Metrics
duration: 6min
completed: 2026-05-27
---

# Phase 2 Plan 00: Test Infrastructure Summary

**Three test stub files establishing behavioral contracts for Auth+API via LMS module stubs, mock binary, and skip-guarded assertions that activate as production modules land**

## Performance

- **Duration:** 6 min
- **Started:** 2026-05-27T16:17:33Z
- **Completed:** 2026-05-27T16:23:28Z
- **Tasks:** 1
- **Files created:** 3

## Accomplishments

- Created t/07_token_manager.t: 12 SKIP-guarded tests for AUTH-01..05 with full LMS stub set and mock binary that handles --get-token/--authenticate/--check
- Created t/08_api_client.t: 12 tests (10 SKIP-guarded for API-01..04, 1 for API-05, 1 immediate for API-06) with controllable SimpleAsyncHTTP mock and TokenManager stub
- Created t/09_settings.t: 20 tests (5 immediate filesystem/chmod tests pass now, rest skip-guarded for Plan 02-03/02-04 deliverables)
- Full test suite (prove -v t/) passes with no failures — 91 tests across 9 files

## Task Commits

1. **Task 1: Create test stub files with mock infrastructure** - `fc4c251` (feat)

**Plan metadata:** (docs commit follows)

## Files Created/Modified

- `t/07_token_manager.t` - AUTH-01..05 contracts; mock binary; Slim::Utils::Cache/Timers/Log/Prefs/Unicode stubs
- `t/08_api_client.t` - API-01..06 contracts; SimpleAsyncHTTP mock with auto_mode; TokenManager stub
- `t/09_settings.t` - AUTH-04/05 immediate chmod tests; AUTH-06 and i18n SKIP-guarded

## Decisions Made

- i18n tests use a sentinel key (PLUGIN_SPOTON_ACTIVE_ACCOUNT) rather than being fully immediate: prevents false failures before Plan 02-03 adds Phase 2 strings to strings.txt while still running the tests when the strings land
- RATE_LIMIT_CACHE_KEY is accessed via `->RATE_LIMIT_CACHE_KEY()` method call rather than `::RATE_LIMIT_CACHE_KEY` bareword constant to avoid strict-subs compile error when Client.pm doesn't exist yet
- AUTH-04/05 filesystem tests (chmod verification) run immediately without skip guard — they test POSIX filesystem behavior, not module behavior, so they always pass

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed compile error: bareword constant in strict context**
- **Found during:** Task 1 (initial test run of t/08_api_client.t)
- **Issue:** `Plugins::SpotOn::API::Client::RATE_LIMIT_CACHE_KEY` used as bareword constant in SKIP block scope caused "Bareword not allowed while strict subs in use" at compile time, preventing test file from running at all
- **Fix:** Changed to method call `Plugins::SpotOn::API::Client->RATE_LIMIT_CACHE_KEY()` which only executes at runtime (inside SKIP block, after require_ok)
- **Files modified:** t/08_api_client.t
- **Verification:** prove t/08_api_client.t passes cleanly
- **Committed in:** fc4c251 (Task 1 commit)

**2. [Rule 1 - Bug] Fixed i18n tests failing immediately against pre-Phase-2 strings.txt**
- **Found during:** Task 1 (initial test run of t/09_settings.t)
- **Issue:** Plan specified i18n tests as "immediate" but the strings (PLUGIN_SPOTON_ACTIVE_ACCOUNT etc.) are added in Plan 02-03, not Plan 02-00. Tests failed for 10 keys that don't exist yet.
- **Fix:** Added sentinel-based SKIP: checks PLUGIN_SPOTON_ACTIVE_ACCOUNT presence first; if absent, entire i18n block skips with clear message "Plan 02-03 will add them"
- **Files modified:** t/09_settings.t
- **Verification:** prove t/09_settings.t passes with skips (no failures)
- **Committed in:** fc4c251 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (both Rule 1 bugs)
**Impact on plan:** Both fixes essential for prove to exit cleanly. No scope creep — test coverage and contract definitions unchanged.

## Issues Encountered

- Minor Perl warning "used only once" for `@Slim::Utils::Timers::set_calls` in t/07_token_manager.t — this is a compile-time false positive because the package global is defined inside a write_stub (loaded at runtime) but referenced at test time. The warning does not cause test failures and will disappear once TokenManager.pm is loaded and the stub is properly initialized. Not fixed to avoid overcomplicating the stub.

## Next Phase Readiness

- Test infrastructure is ready for Plans 02-02 (TokenManager.pm), 02-03 (Client.pm), and 02-04 (Settings.pm account CRUD)
- Each plan's executor activates skip-guarded tests by creating the production module — no test file changes needed
- All LMS stub patterns (write_stub, Prefs with cachedir, Cache with TTL inspection, Timers with call recording, SimpleAsyncHTTP with auto_mode) are established and reusable

## Self-Check: PASSED

Files exist:
- t/07_token_manager.t: FOUND
- t/08_api_client.t: FOUND
- t/09_settings.t: FOUND
- .planning/phases/02-auth-api-foundation/02-00-SUMMARY.md: FOUND (this file)

Commits:
- fc4c251: FOUND (feat(02-00): create test stub files with mock infrastructure for Phase 2)

---
*Phase: 02-auth-api-foundation*
*Completed: 2026-05-27*
