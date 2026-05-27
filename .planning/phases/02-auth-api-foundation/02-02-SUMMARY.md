---
phase: 02-auth-api-foundation
plan: "02"
subsystem: auth
tags: [perl, token-manager, auth, librespot, multi-account, security]

# Dependency graph
requires:
  - phase: 02-auth-api-foundation
    plan: "00"
    provides: t/07_token_manager.t with SKIP-guarded AUTH-01..05 contracts, mock binary, LMS stubs

provides:
  - Plugins/SpotOn/API/TokenManager.pm with full token lifecycle management
  - AUTH-01 through AUTH-05 behavioral contracts verified by tests

affects: [02-03-api-client, 02-04-settings-account-crud, 02-05-integration-gate]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "alarm(N) + eval die-handler wrapping every backtick binary call — prevents LMS freeze (T-02-06)"
    - "Shell-safe quoting s/'/'\\'\\''//g applied to ALL user-supplied and path values before backtick"
    - "Token cache key: spoton_token_<accountId>, TTL = expiresIn - TOKEN_EXPIRY_BUFFER (300)"
    - "accountId = substr(md5_hex(Slim::Utils::Unicode::utf8toLatin1Transliterate(username)), 0, 8)"
    - "Stub export pattern: custom import() in test stubs to inject functions into caller namespace"

key-files:
  created:
    - Plugins/SpotOn/API/TokenManager.pm
  modified:
    - t/07_token_manager.t

key-decisions:
  - "require Plugins::SpotOn::Helper in TokenManager.pm allows standalone loading in test context (not just when Plugin.pm loads it)"
  - "File::Spec::Functions stub removed — system module available in Perl core, no stub needed"
  - "Slim::Utils::Log and Slim::Utils::Prefs stubs augmented with custom import() to export logger() and preferences() as functions into caller namespace"
  - "Line 105 logs 'no accessToken in binary response' — logs key name absence not value, T-02-07 compliant"

# Metrics
duration: 3min
completed: 2026-05-27
---

# Phase 2 Plan 02: TokenManager Summary

**Token lifecycle bridge between librespot binary and Perl plugin: binary spawn, caching with pre-expiry TTL, 45-minute proactive refresh timer, secure credential directory management (chmod 0700/0600), and multi-account support with MD5-keyed subdirectories**

## Performance

- **Duration:** 3 min
- **Started:** 2026-05-27T16:31:34Z
- **Completed:** 2026-05-27T16:34:52Z
- **Tasks:** 2
- **Files created:** 1 (TokenManager.pm)
- **Files modified:** 1 (t/07_token_manager.t)

## Accomplishments

- Created `Plugins/SpotOn/API/TokenManager.pm` with 351 lines covering the complete token lifecycle
- All 7 public methods implemented: refreshToken, getToken, addAccount, removeAccount, getAccountIds, getActiveAccountName, refreshAllTokens
- All 5 private helpers implemented: _cacheDir, _newAccountCacheDir, _cacheToken, _setPermissions, _finalizeAccountDir
- Activated all 15 tests in `t/07_token_manager.t` — was 0 active (all SKIP-guarded) before this plan
- Full test suite: 94 tests pass across 9 files (up from 91 before)

## Task Commits

1. **Task 1: Create API/TokenManager.pm** — `e1f9b42` (feat)
2. **Task 2: Activate t/07_token_manager.t tests** — `1eec791` (feat)

## Files Created/Modified

- `Plugins/SpotOn/API/TokenManager.pm` — Full token lifecycle: binary spawn + alarm(10/15), JSON parse, cache with TTL, chmod permissions, multi-account with MD5 accountId, timer re-arm
- `t/07_token_manager.t` — Stubs fixed to export logger()/preferences() as functions; File::Spec::Functions stub removed; 15/15 tests active and passing

## Acceptance Criteria Verification

| Criterion | Status |
|-----------|--------|
| TokenManager.pm exists with package declaration | PASS |
| Contains all 7 public methods | PASS |
| Uses Helper->get() for binary path | PASS |
| Shell-safe quoting on all backtick arguments | PASS |
| alarm(10) wraps --get-token backtick | PASS |
| alarm(15) wraps --authenticate backtick | PASS |
| Token cached with key spoton_token_<accountId> and TTL = expiresIn - 300 | PASS |
| _setPermissions calls chmod 0700 on directory, 0600 on credentials.json | PASS |
| accountId derived from substr(md5_hex(...), 0, 8) | PASS |
| No accessToken value in any log call | PASS (line 105 logs key absence, not value) |
| main::INFOLOG guard on info-level calls | PASS |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Slim::Utils::Log stub missing function export for logger()**
- **Found during:** Task 2 (first test run)
- **Issue:** Test stubs defined `logger()` as a package method but did not export it as a function into caller namespace. `TokenManager.pm` calls bare `logger('plugin.spoton')` after `use Slim::Utils::Log`. Resulted in "Undefined subroutine &Plugins::SpotOn::API::TokenManager::logger" at load time.
- **Fix:** Added custom `import()` to the `Slim::Utils::Log` stub that injects `logger` into the caller namespace. Same fix applied to `Slim::Utils::Prefs` stub for `preferences()`.
- **Files modified:** t/07_token_manager.t
- **Commit:** 1eec791

**2. [Rule 3 - Blocking] File::Spec::Functions stub caused undefined catdir function**
- **Found during:** Task 2 (second test run)
- **Issue:** The existing stub used `*catdir = \&File::Spec::catdir` which tried to reference a non-exported symbol. The module may already be in `%INC` at the time the stub is written.
- **Fix:** Removed the stub entirely — `File::Spec::Functions` is a Perl core module available on the test system. No stub needed.
- **Files modified:** t/07_token_manager.t
- **Commit:** 1eec791

**3. [Rule 1 - Bug] require Plugins::SpotOn::Helper missing from TokenManager.pm**
- **Found during:** Task 2 (first test run after load fix)
- **Issue:** TokenManager.pm called `Plugins::SpotOn::Helper->get()` without requiring the module first. In the LMS runtime, Plugin.pm loads Helper before TokenManager; in the test context, there is no such pre-loading.
- **Fix:** Added `require Plugins::SpotOn::Helper` to TokenManager.pm. This is correct for both runtime (idempotent if already loaded) and test contexts.
- **Files modified:** Plugins/SpotOn/API/TokenManager.pm
- **Commit:** e1f9b42

## Threat Surface Scan

No new network endpoints, auth paths, or schema changes introduced beyond what is specified in the plan's threat model. All T-02-04, T-02-05, T-02-06, T-02-07 mitigations are implemented in TokenManager.pm.

## Self-Check: PASSED

Files exist:
- Plugins/SpotOn/API/TokenManager.pm: FOUND
- t/07_token_manager.t: FOUND
- .planning/phases/02-auth-api-foundation/02-02-SUMMARY.md: FOUND (this file)

Commits:
- e1f9b42: FOUND (feat(02-02): create API/TokenManager.pm)
- 1eec791: FOUND (feat(02-02): activate t/07_token_manager.t)

Test run: 15/15 tests pass in t/07_token_manager.t; 94/94 tests pass full suite.

---
*Phase: 02-auth-api-foundation*
*Completed: 2026-05-27*
