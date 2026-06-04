---
phase: "09"
plan: "01"
subsystem: metadata-display
tags: [songinfo, type-string, stream-metadata, tdd]
dependency_graph:
  requires: []
  provides:
    - "_typeString helper sub in Plugin.pm"
    - "Dynamic stream metadata in Songinfo for Browse and Connect"
  affects:
    - "Plugins/SpotOn/Plugin.pm"
    - "Plugins/SpotOn/Connect.pm"
    - "Plugins/SpotOn/DontStopTheMusic.pm"
tech_stack:
  added: []
  patterns:
    - "Class method helper (_typeString) for shared display string construction"
    - "Pre-load stubs in tests to control on-demand require modules"
key_files:
  created:
    - "t/10_stream_metadata.t"
  modified:
    - "Plugins/SpotOn/Plugin.pm"
    - "Plugins/SpotOn/Connect.pm"
    - "Plugins/SpotOn/DontStopTheMusic.pm"
decisions:
  - "D-04 bitrate guard: _typeString reads raw pref value (no || 320 fallback) so absent bitrate produces format-only string"
  - "D-05 auto resolution: passthrough capability check via Helper->getCapability('passthrough') — OGG if true, PCM if false"
  - "Helper stub pre-loaded before Plugin.pm in tests to ensure require finds stub, not real module"
metrics:
  duration: "6 minutes"
  completed: "2026-06-04T08:51:00Z"
  tasks_completed: 2
  tasks_total: 2
  test_count: 16
  files_changed: 4
---

# Phase 09 Plan 01: Stream Metadata Display Summary

Dynamic type string for Songinfo -- `_typeString` helper in Plugin.pm builds "{bitrate}k, {format} (Spotify {mode})" from per-player prefs and passthrough capability, replacing all 4 hardcoded metadata call sites.

## What Was Done

### Task 1: RED -- Failing tests for _typeString and grep gates (TDD)
- Created `t/10_stream_metadata.t` with 16 test cases:
  - 12 unit tests covering all format prefs (ogg/flac/mp3/pcm), auto+passthrough, auto-passthrough, bitrateOverride, undef client, D-04 bitrate-absent guard, D-02 mode label presence
  - 3 grep-gate tests scanning Plugin.pm, DontStopTheMusic.pm, Connect.pm for stale hardcoded type literals
  - 1 require_ok gate
- Test infrastructure adapted from t/09_settings.t with controllable Helper stub
- All 15 behavioral tests failed (RED state confirmed)
- **Commit:** 4be6f9e

### Task 2: GREEN -- Implement _typeString and update all call sites
- Added `_typeString` class method to Plugin.pm (40 lines):
  - Sync-group normalization (`$client->master` guard)
  - Bitrate chain: raw pref value with per-player bitrateOverride
  - streamFormat chain with connectOggOverride migration fallback
  - Auto resolution via `Helper->getCapability('passthrough')`
  - D-03 short format labels (OGG/FLAC/MP3/PCM)
  - D-01/D-04 assembly with bitrate-absent guard
- Updated 4 call sites:
  1. `Plugin.pm _trackItem` (line 405): `type => __PACKAGE__->_typeString($client, 'Browse')`
  2. `Plugin.pm _albumTrackItem` (line 1143): same pattern
  3. `Connect.pm _fetchTrackMetadata` (line 844): `type/originalType => Plugin->_typeString($client, 'Connect')`
  4. `DontStopTheMusic.pm _cacheAndExtractUris` (line 262): `type => Plugin->_typeString(undef, 'Browse')`
- Fixed test: pre-load Helper stub before Plugin.pm to ensure `require` finds stub
- All 16 tests pass (GREEN state confirmed)
- **Commit:** 65b4e8d

## Verification Results

| Check | Result |
|-------|--------|
| `perl t/10_stream_metadata.t` | 16/16 pass |
| `grep type.*'Spotify' Plugin.pm DontStopTheMusic.pm` | 0 matches |
| `grep 'Ogg Vorbis (Spotify)' Connect.pm` | 0 matches |
| Pre-existing test failures (t/07, t/08, t/09) | Unchanged, not caused by this plan |

## TDD Gate Compliance

- RED gate: `test(09-01)` commit 4be6f9e -- 15 tests fail, _typeString not yet implemented
- GREEN gate: `feat(09-01)` commit 65b4e8d -- all 16 tests pass
- REFACTOR gate: skipped -- code already clean, no refactoring needed

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Missing DEBUGLOG constant in test stubs**
- **Found during:** Task 1
- **Issue:** Plugin.pm uses `main::DEBUGLOG` at line 161/196; test stubs only defined INFOLOG
- **Fix:** Added `*main::DEBUGLOG = sub () { 0 }` to test BEGIN block
- **Files modified:** t/10_stream_metadata.t
- **Commit:** 4be6f9e (part of RED commit)

**2. [Rule 3 - Blocking] Helper stub not loaded before Plugin.pm require**
- **Found during:** Task 2
- **Issue:** `_typeString` does `require Plugins::SpotOn::Helper` which loaded real Helper.pm (lexical `$helperCapabilities`) instead of stub (package variable `$helperCapabilities`), making passthrough tests uncontrollable
- **Fix:** Added explicit `require Plugins::SpotOn::Helper` before `require_ok('Plugins::SpotOn::Plugin')` so the stub wins the `%INC` race
- **Files modified:** t/10_stream_metadata.t
- **Commit:** 65b4e8d (part of GREEN commit)

**3. [Rule 1 - Bug] Bitrate D-04 guard prevented by || 320 fallback**
- **Found during:** Task 2
- **Issue:** Plan specified `$prefs->get('bitrate') || 320` (mirrors updateTranscodingTable), but this makes D-04 (absent bitrate guard) unreachable since `0 || 320 = 320`
- **Fix:** Changed to `$prefs->get('bitrate')` (no `|| 320` fallback) so the ternary guard in assembly handles absent bitrate correctly
- **Files modified:** Plugins/SpotOn/Plugin.pm
- **Commit:** 65b4e8d

## Pending Verification

Task 3 (checkpoint:human-verify) requires live LMS testing to confirm Songinfo displays dynamic metadata for both Browse and Connect tracks.

## Known Stubs

None -- all data paths are fully wired.

## Threat Flags

None -- no new trust boundaries, network endpoints, or auth paths introduced.

## Self-Check: PASSED

- All 5 key files exist on disk
- Both task commits (4be6f9e, 65b4e8d) found in git history
- `_typeString` sub present in Plugin.pm (1 match)
- Zero stale hardcoded type literals in all 3 production files
