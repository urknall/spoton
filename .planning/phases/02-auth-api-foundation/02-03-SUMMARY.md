---
phase: 02-auth-api-foundation
plan: "03"
subsystem: api
tags: [perl, api-client, rate-limiting, caching, async-http, spotify-web-api]

# Dependency graph
requires:
  - phase: 02-auth-api-foundation
    plan: "00"
    provides: t/08_api_client.t with SKIP-guarded API-01..06 contracts and SimpleAsyncHTTP mock
  - phase: 02-auth-api-foundation
    plan: "02"
    provides: Plugins/SpotOn/API/TokenManager.pm with getToken method signature

provides:
  - Plugins/SpotOn/API/Client.pm with central HTTP egress, rate limiting, caching, 429 handling, getMe
  - API-01 through API-06 behavioral contracts verified by tests

affects: [02-04-settings-account-crud, 02-05-integration-gate, 03-browse-search-library]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Central HTTP egress pattern: all Spotify API calls route through _request — never call SimpleAsyncHTTP directly from outside Client.pm"
    - "RATE_LIMIT_CACHE_KEY as exported constant: Plugin.pm reads it for OPML rate-limit hint without coupling to HTTP layer"
    - "inflightCount pattern: module-level var reset via Client->reset() in initPlugin; decremented in ALL exit paths (success, error, no_token)"
    - "Domain-specific cache TTLs: 0=player/me, 60=library, 300=browse/playlists, 3600=metadata"
    - "Retry-After cap at 300s: prevents self-DoS from malicious 429 header (T-02-08)"
    - "_noCache flag pattern: getMe passes _noCache=>1 to always fetch fresh user profile"

key-files:
  created:
    - Plugins/SpotOn/API/Client.pm
  modified:
    - t/08_api_client.t

key-decisions:
  - "inflightCount has 3 decrement points (success, error, no_token) not 2 as plan said — plan undercounted by missing the no_token path in the token callback"
  - "Stub export fix required: import() added to Slim::Utils::Log and Slim::Utils::Prefs stubs to export logger()/preferences() as functions — same fix as Plan 02-02"
  - "URI::Escape::uri_escape used for query param building — available as Perl core, no stub needed"

patterns-established:
  - "Pattern: _request pipeline: rate-limit check → cache check → concurrency cap → inflight++ → getToken → HTTP dispatch"
  - "Pattern: Slim stub import() fix — any stub for a module that exports functions must add custom import() injecting into caller namespace"

requirements-completed: [API-01, API-02, API-03, API-04, API-05, API-06]

# Metrics
duration: 10min
completed: 2026-05-27
---

# Phase 2 Plan 03: API Client Summary

**Central HTTP egress Client.pm with sliding-window rate limiting (max 3 concurrent), 429/Retry-After handling capped at 300s, domain-specific response caching (0/60/300/3600s), token injection via TokenManager, and getMe endpoint**

## Performance

- **Duration:** 10 min
- **Started:** 2026-05-27T16:31:00Z
- **Completed:** 2026-05-27T16:41:00Z
- **Tasks:** 2
- **Files created:** 1 (Client.pm)
- **Files modified:** 1 (t/08_api_client.t)

## Accomplishments

- Created `Plugins/SpotOn/API/Client.pm` with 228 lines covering the complete HTTP egress pipeline
- All public methods implemented: reset, getMe, _request, _onSuccess, _onError, _cacheTTL
- Activated all 14 tests in `t/08_api_client.t` — was 10 SKIP-guarded before this plan
- Full test suite: tests pass with no failures (14 in this file, part of broader suite)

## Task Commits

1. **Task 1: Create API/Client.pm with rate limiting, caching, and getMe** - `34f1207` (feat)
2. **Task 2: Activate and verify t/08_api_client.t tests** - `927a906` (feat)

**Plan metadata:** (docs commit follows)

## Files Created/Modified

- `Plugins/SpotOn/API/Client.pm` — Central HTTP egress: _request pipeline, rate limiting, concurrency cap, 429 handler, response caching, reset(), getMe()
- `t/08_api_client.t` — Stubs fixed to export logger()/preferences() as functions; 14/14 tests active and passing

## Acceptance Criteria Verification

| Criterion | Status |
|-----------|--------|
| Client.pm exists with package declaration | PASS |
| _request as central pipeline method | PASS |
| getMe calling _request with path 'me' | PASS |
| reset() sets inflightCount = 0 | PASS |
| RATE_LIMIT_CACHE_KEY defined and accessible via ->RATE_LIMIT_CACHE_KEY() | PASS |
| MAX_CONCURRENT_REQUESTS is 3 | PASS |
| inflightCount decremented in _onSuccess AND _onError (and no_token path) | PASS (3 occurrences — see deviation) |
| 429 handler caps Retry-After at 300 seconds | PASS |
| _cacheTTL: 0 for me/player, 60 for me/tracks, 3600 for tracks/, 300 for playlists/ | PASS |
| No LWP::UserAgent or SimpleSyncHTTP | PASS (grep returns 0) |
| Only SimpleAsyncHTTP for HTTP calls | PASS |

## Decisions Made

- inflightCount has 3 decrement points instead of the plan's stated 2: the "no_token" path inside the TokenManager callback fires before `_onError` is called, requiring its own `$inflightCount--`. This is correct — the plan's "2" was a documentation error that didn't account for the async no_token path.
- Stub `import()` fix applied to both `Slim::Utils::Log` and `Slim::Utils::Prefs` in t/08_api_client.t — same pattern established in Plan 02-02 for t/07_token_manager.t.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Slim::Utils::Log stub missing import() for logger()**
- **Found during:** Task 2 (first test run)
- **Issue:** t/08_api_client.t stub defined logger() as package method but did not export it into caller namespace. Client.pm calls bare `logger('plugin.spoton')` after `use Slim::Utils::Log`. Result: "Undefined subroutine &Plugins::SpotOn::API::Client::logger"
- **Fix:** Added custom import() to Slim::Utils::Log stub to inject logger() into caller namespace. Same fix for Slim::Utils::Prefs to export preferences().
- **Files modified:** t/08_api_client.t
- **Verification:** prove t/08_api_client.t — 14/14 tests pass
- **Committed in:** 927a906 (Task 2 commit)

**2. [Rule 1 - Bug] Plan stated inflightCount-- should have 2 occurrences; correct count is 3**
- **Found during:** Task 1 (code review of _request pipeline)
- **Issue:** The plan says "grep shows 2 occurrences of inflightCount--" but the correct implementation needs 3: one in _onSuccess, one in _onError, and one in the no_token path inside the getToken callback. The plan's acceptance criterion was written without accounting for the no_token async path.
- **Fix:** Kept all 3 decrement points — removing the no_token decrement would leak the counter, violating T-02-09 (inflightCount leak).
- **Files modified:** Plugins/SpotOn/API/Client.pm
- **Verification:** API-02 test passes with correct concurrency behavior; no counter leaks
- **Committed in:** 34f1207 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (1 Rule 3 blocking stub fix, 1 Rule 1 plan inaccuracy)
**Impact on plan:** Both fixes essential for correctness. No scope creep — all API-01..06 requirements met.

## Threat Surface Scan

No new network endpoints, auth paths, or schema changes beyond the plan's threat model. All T-02-08, T-02-09, T-02-10 mitigations implemented:
- T-02-08: Retry-After capped at 300s
- T-02-09: All exit paths decrement inflightCount; reset() called in initPlugin
- T-02-10: Authorization header value never logged

## Self-Check: PASSED

Files exist:
- Plugins/SpotOn/API/Client.pm: FOUND
- t/08_api_client.t: FOUND
- .planning/phases/02-auth-api-foundation/02-03-SUMMARY.md: FOUND (this file)

Commits:
- 34f1207: FOUND (feat(02-03): create API/Client.pm)
- 927a906: FOUND (feat(02-03): activate t/08_api_client.t)

Test run: 14/14 tests pass in t/08_api_client.t.

---
*Phase: 02-auth-api-foundation*
*Completed: 2026-05-27*
