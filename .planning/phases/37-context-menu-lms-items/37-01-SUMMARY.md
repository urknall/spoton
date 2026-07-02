---
phase: 37-context-menu-lms-items
plan: 01
subsystem: ui
tags: [lms, context-menu, protocol-handler, track-info, opml]

requires: []

provides:
  - ProtocolHandler.pm without trackInfoURL override — LMS framework handles menu assembly
  - t/14_context_menu.t regression test preventing trackInfoURL from re-appearing
  - JSON::XS stub added to t/10 and t/11 (unblocks full suite)

affects: []

tech-stack:
  added: []
  patterns:
    - "Use symbol table check (!defined &Package::method) instead of ->can() when base class stubs override UNIVERSAL::can"

key-files:
  created:
    - t/14_context_menu.t
  modified:
    - Plugins/SpotOn/ProtocolHandler.pm
    - t/10_stream_metadata.t
    - t/11_track_history.t

key-decisions:
  - "trackInfoURL removed rather than left as a thin passthrough — Plugin.pm trackInfoMenu already provides identical items, duplication removed"
  - "Regression test uses !defined(&Plugins::SpotOn::ProtocolHandler::trackInfoURL) not ->can() — stub base class overrides can() returning 1 always"

patterns-established:
  - "Symbol table check pattern: !defined(&Package::method) for absence assertions in tests where base stubs override can()"

requirements-completed:
  - CTX-01

duration: 20min
completed: 2026-06-30
---

# Phase 37 Plan 01: Context Menu LMS Items Summary

**Removed 80-line trackInfoURL method from ProtocolHandler.pm so LMS framework can assemble the standard More menu with both native items (Favorites, play controls) and SpotOn items via registerInfoProvider**

## Performance

- **Duration:** ~20 min
- **Started:** 2026-06-30T15:00:00Z
- **Completed:** 2026-06-30T15:20:17Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments

- Deleted `sub trackInfoURL` (lines 758-837) from ProtocolHandler.pm — the method intercepted LMS's TrackInfo menu assembly and bypassed all standard items
- Created `t/14_context_menu.t` with 16 tests covering CTX-01 regression gate plus trackInfoMenu behavioral coverage (non-spoton URLs, no accountId, track URL with 4 items, episode URL with 3 items)
- Fixed pre-existing full-suite failures in t/10_stream_metadata.t and t/11_track_history.t by adding JSON::XS stub (all 416 tests now pass)

## Task Commits

1. **Task 1: Create regression test** — `7994a53` (test)
2. **Task 2: Remove trackInfoURL + fix suite** — `ee4a428` (feat)

## Files Created/Modified

- `t/14_context_menu.t` — 16-test regression suite for context menu behavior; CTX-01 permanently gates trackInfoURL absence
- `Plugins/SpotOn/ProtocolHandler.pm` — Deleted sub trackInfoURL (80 lines); getMetadataFor and _cacheExplodedTrack untouched
- `t/10_stream_metadata.t` — Added JSON::XS stub (pre-existing failure fix)
- `t/11_track_history.t` — Added JSON::XS stub (pre-existing failure fix)

## Decisions Made

- Use `!defined(&Plugins::SpotOn::ProtocolHandler::trackInfoURL)` rather than `->can('trackInfoURL')` — the `Slim::Formats::RemoteStream` stub has `sub can { 1 }` which returns true for any method, making `->can()` unreliable for absence checks in test stubs
- No replacement code needed — Plugin.pm's `trackInfoMenu` registered via `registerInfoProvider(spotonTrackInfo)` already provides identical items; the override was pure duplication

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] CTX-01 test used wrong check method**
- **Found during:** Task 2 verification
- **Issue:** `Plugins::SpotOn::ProtocolHandler->can('trackInfoURL')` returned true even after deletion because the base class stub `Slim::Formats::RemoteStream` defines `sub can { 1 }` which is inherited and always returns truthy
- **Fix:** Changed test to `!defined(&Plugins::SpotOn::ProtocolHandler::trackInfoURL)` — direct symbol table lookup bypasses inheritance and AUTOLOAD
- **Files modified:** t/14_context_menu.t
- **Verification:** Test 3 (CTX-01) now correctly passes after deletion
- **Committed in:** ee4a428 (Task 2 commit)

**2. [Rule 1 - Bug] Pre-existing JSON::XS failure in t/10 and t/11**
- **Found during:** Task 2 (full suite verification)
- **Issue:** t/10_stream_metadata.t and t/11_track_history.t failed with "Can't locate JSON/XS.pm" because Plugin.pm uses `use JSON::XS qw(encode_json)` which is an LMS-bundled XS module not available in the test environment
- **Fix:** Added `JSON::XS` stub (delegates to `JSON::PP`) to both test files, mirroring the existing `JSON::XS::VersionOneAndTwo` stub pattern
- **Files modified:** t/10_stream_metadata.t, t/11_track_history.t
- **Verification:** prove t/ exits 0 (416 tests pass)
- **Committed in:** ee4a428 (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (2 Rule 1 bugs)
**Impact on plan:** Both fixes necessary for correct test behavior. No scope creep. Full test suite now passes.

## Issues Encountered

None — the taskInfoURL removal was a clean deletion. The two deviations were discovered during verification and fixed immediately.

## Threat Surface Scan

No new trust boundaries introduced. This change removes code; it does not add input handling, authentication, or data flow.

## Known Stubs

None — no stub data or placeholders in production code.

## Next Phase Readiness

- Phase 37 complete. LMS standard menu items (Add to Favorites, play controls, More Info) will appear alongside SpotOn-specific items in the More menu for SpotOn tracks.
- Phase 38 (Importer Foundation) can proceed immediately.

## Self-Check

Created files:
- t/14_context_menu.t: exists (created in this plan)

Commits:
- 7994a53: test(37-01): add regression test
- ee4a428: feat(37-01): remove trackInfoURL

---
*Phase: 37-context-menu-lms-items*
*Completed: 2026-06-30*
