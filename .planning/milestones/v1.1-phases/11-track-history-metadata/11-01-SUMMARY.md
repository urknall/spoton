---
phase: 11-track-history-metadata
plan: "01"
subsystem: cache
tags:
  - cache-ttl
  - connect-metadata
  - history
dependency_graph:
  requires:
    - "Phase 5 — Connect.pm _fetchTrackMetadata established"
    - "Phase 9 — Plugin.pm _trackItem/_albumTrackItem cache pattern established"
  provides:
    - "7-day cache TTL for all spoton_meta_ entries"
    - "Connect track Spotify URI persisted to cache (spotifyUri field)"
    - "t/11_track_history.t test scaffold with TTL and cache assertions"
  affects:
    - "Plugins/SpotOn/Plugin.pm"
    - "Plugins/SpotOn/DontStopTheMusic.pm"
    - "Plugins/SpotOn/Connect.pm"
    - "t/11_track_history.t"
tech_stack:
  added: []
  patterns:
    - "Slim::Utils::Cache->new()->set(..., 604800) — 7-day TTL for all spoton_meta_ entries"
    - "spotifyUri field in Connect cache entries for future Browse URL translation"
key_files:
  created:
    - path: t/11_track_history.t
      purpose: Test scaffold — TTL grep gates, spotifyUri assertion, cache lookup test
  modified:
    - path: Plugins/SpotOn/Plugin.pm
      change: _trackItem and _albumTrackItem cache TTL 3600 -> 604800
    - path: Plugins/SpotOn/DontStopTheMusic.pm
      change: _cacheAndExtractUris cache TTL 3600 -> 604800
    - path: Plugins/SpotOn/Connect.pm
      change: Add Digest::MD5 import; add persistent cache write with spotifyUri in _fetchTrackMetadata
decisions:
  - "Kept KILL_PROCESS_INTERVAL constant at 3600 (hourly orphan cleanup) — unrelated to cache TTL"
  - "Used Slim::Utils::Cache->new() in Connect.pm (no package-level $cache var)"
  - "spotifyUri stores full 'spotify:track:ID' URI — Plan 02 will parse it for translation"
metrics:
  duration: "3 minutes"
  completed: "2026-06-04T15:55:35Z"
  tasks_completed: 2
  files_modified: 3
  files_created: 1
---

# Phase 11 Plan 01: Track History Metadata — Cache TTL Bump + Connect Persistence Summary

**One-liner:** Bumped all spoton_meta_ cache TTLs from 1 hour to 7 days and added Connect mode cache persistence with spotifyUri field for future Browse URL translation.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create t/11_track_history.t test scaffold | bd08192 | t/11_track_history.t |
| 2 | TTL bump + Connect cache persistence | 37fc1df | Plugin.pm, DontStopTheMusic.pm, Connect.pm |

## What Was Built

**Task 1 — Test Scaffold (bd08192)**

Created `t/11_track_history.t` following the exact structural pattern from `t/10_stream_metadata.t`:
- Full LMS module stub suite (Log, Prefs, Cache with ttl(), Timers, Strings, etc.)
- Additional stubs needed for ProtocolHandler.pm: Slim::Utils::Network, Slim::Utils::Versions, Slim::Utils::Misc, Slim::Control::Request, Slim::Music::Info, Slim::Player::Protocols::HTTP
- Plugins::SpotOn::API::Client stub with controllable $mock_track
- 8 subtest blocks: 5 asserting current state, 2 TODO-marked for Plan 02

Test groups:
- A: TTL grep gate — Plugin.pm (no 3600 cache sets)
- B: TTL grep gate — DontStopTheMusic.pm (no 3600 cache sets)
- C: TTL grep gate — Connect.pm cache sets use 604800
- D: Connect.pm has spotifyUri in non-comment code
- E: getMetadataFor returns cached Browse metadata (live test)
- F: Cache miss returns placeholder (TODO — Plan 02)
- G: Connect URL with spotifyUri returns metadata (TODO — Plan 02)

**Task 2 — TTL Bump + Cache Persistence (37fc1df)**

Three files modified:

- `Plugin.pm`: Two cache-set sites (_trackItem line ~421, _albumTrackItem line ~1159) changed from 3600 to 604800
- `DontStopTheMusic.pm`: One cache-set site (_cacheAndExtractUris line ~265) changed from 3600 to 604800
- `Connect.pm`: Added `use Digest::MD5 qw(md5_hex)` import; added persistent cache write block in _fetchTrackMetadata after the $song->pluginData(info => {...}) call with spotifyUri field

## Verification Results

```
perl t/05_perl_syntax.t       — 6/6 PASS
perl t/11_track_history.t     — 8/8 subtests PASS (F and G are TODO-marked)
grep -c '604800' Plugin.pm    — 4 (2 cache-set + 2 comment lines)
grep -c '604800' DSTM.pm      — 2 (1 cache-set + 1 comment line)
grep -c 'spotifyUri' Connect.pm — 2 (1 in cache write + 1 in comment)
```

## Deviations from Plan

### Note: KILL_PROCESS_INTERVAL constant retained at 3600

The plan's acceptance criteria stated `grep -v '^\s*#' Plugins/SpotOn/Plugin.pm | grep -c '3600' returns 0`, but `Plugin.pm` contains `use constant KILL_PROCESS_INTERVAL => 3600` (hourly orphan cleanup — STR-10). This constant is semantically correct and must not be changed to 604800. The TTL grep test in t/11_track_history.t correctly uses the cache-set pattern (`},\s*3600\s*\)`) which excludes this constant. The test passes with 0 matches. The raw grep count returns 1, which is an acknowledged deviation from the literal acceptance criteria text.

Otherwise, plan executed exactly as written.

## Known Stubs

None — no UI rendering stubs or placeholder data introduced.

## Threat Flags

None — no new network endpoints, auth paths, or trust boundary changes introduced.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| t/11_track_history.t | FOUND |
| Plugins/SpotOn/Plugin.pm (modified) | FOUND |
| Plugins/SpotOn/DontStopTheMusic.pm (modified) | FOUND |
| Plugins/SpotOn/Connect.pm (modified) | FOUND |
| .planning/phases/11-track-history-metadata/11-01-SUMMARY.md | FOUND |
| Commit bd08192 (test scaffold) | FOUND |
| Commit 37fc1df (TTL bump + Connect persistence) | FOUND |
