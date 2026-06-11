---
phase: 15-like-button
plan: "02"
subsystem: plugin-ui
tags: [like-button, track-info-menu, opml-callbacks, cache, lms-hook]
dependency_graph:
  requires: [15-01]
  provides: [trackInfoMenu, SpotOnManageLike, SpotOnLike, SpotOnUnlike, like-unlike-workflow]
  affects: [Plugin.pm]
tech_stack:
  added: []
  patterns: [slim-menu-trackinfo-hook, opml-passthrough-callbacks, cache-first-strategy, showbriefly-feedback]
key_files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm
decisions:
  - "require Slim::Menu::TrackInfo (not use) — deferred load avoids compile-time failure in headless LMS contexts"
  - "Error check: $err->{code} >= 400 (not != 200) — aligns with empty-body guard contract from Plan 01 where $err is undef on success"
  - "URI guard regex /^spotify:track:[A-Za-z0-9]+$/ — rejects injection, non-SpotOn tracks, and malformed $remoteMeta->{uri}"
  - "Four subs placed in new Like/Unlike section after Shared Helper Functions, before _largestImage — preserves plugin structure"
metrics:
  duration_minutes: 12
  completed_date: "2026-06-11"
  tasks_completed: 2
  tasks_total: 2
---

# Phase 15 Plan 02: Like Button UI Wiring Summary

**One-liner:** trackInfoMenu hook registered in initPlugin + four OPML callback subs (ManageLike/Like/Unlike) with 60s cache, URI guard, 403 error handling, and immediate cache invalidation.

## What Was Built

The complete Like/Unlike user-facing workflow wired into Plugin.pm:

**Task 1: registerInfoProvider in initPlugin**
- `require Slim::Menu::TrackInfo` + `Slim::Menu::TrackInfo->registerInfoProvider(spotonTrackInfo => ...)` added after `$class->SUPER::initPlugin(...)` per D-01 Qobuz-Pattern
- `SPOTON_CACHE_VERSION => 3` was already set by Plan 01 (no change needed)

**Task 2: Four new subs in Plugin.pm**

`sub trackInfoMenu` — LMS Track Info menu hook entry point:
- URI guard rejects all non-SpotOn tracks: `$trackUri =~ /^spotify:track:[A-Za-z0-9]+$/` (T-15-01)
- Guard also rejects clients with no active account
- Returns static `PLUGIN_SPOTON_MANAGE_LIKE` item pointing to `SpotOnManageLike`

`sub SpotOnManageLike` — dynamic state resolution with cache-first strategy:
- Cache key: `spoton_liked_{accountId}_{trackId}` (60s TTL)
- Cache hit (D-07): calls `$buildMenu->($cached)` immediately, no API call
- Cache miss (D-06): calls `Client->checkTracks` asynchronously, sets cache on result
- `$buildMenu` closure renders 'Like' or 'Unlike' item with `nextWindow => 'grandparent'`

`sub SpotOnLike` — saves track to Spotify library:
- Calls `Client->saveTracks($accountId, [$trackUri], sub {...})`
- Success: `$err is undef` (empty-body guard contract) — calls `$cache->remove($cacheKey)` then shows `PLUGIN_SPOTON_LIKED` with `showBriefly => 1, nextWindow => 'grandparent'`
- Error: `$err->{code} >= 400` — 403 shows `PLUGIN_SPOTON_LIKE_ERROR_SCOPE`, other shows `PLUGIN_SPOTON_LIKE_ERROR` (no `nextWindow`, user stays in menu per D-10)

`sub SpotOnUnlike` — removes track from Spotify library:
- Calls `Client->removeTracks($accountId, [$trackUri], sub {...})`
- Identical success/error handling pattern to `SpotOnLike`, success shows `PLUGIN_SPOTON_UNLIKED`

## Commits

| Task | Commit | Files |
|------|--------|-------|
| Task 1: registerInfoProvider in initPlugin | 3a4da58 | Plugin.pm |
| Task 2: trackInfoMenu + SpotOnManageLike/Like/Unlike subs | ff1f440 | Plugin.pm |

## Deviations from Plan

None — plan executed exactly as written.

The plan noted that `SPOTON_CACHE_VERSION => 3` was to be bumped in Task 1, but Plan 01 had already done this. No deviation recorded — the plan's acceptance criteria were already satisfied.

## Test Results

- Full suite: 261 tests, 12 files — all green (unchanged from Plan 01)
- No new tests added in Plan 02 (UI callback layer; integration coverage deferred to verify-work phase)

## Threat Surface

No new network endpoints or trust boundaries introduced. `registerInfoProvider` is an LMS-internal event hook — it does not open network sockets or auth paths. T-15-01 URI injection threat is mitigated by the `/^spotify:track:[A-Za-z0-9]+$/` guard in `trackInfoMenu`.

## Known Stubs

None — all four subs are fully implemented with real API calls, cache logic, and error handling.

## Self-Check: PASSED

- `grep "registerInfoProvider.*spotonTrackInfo" Plugins/SpotOn/Plugin.pm` → match found (line 132)
- `grep "SPOTON_CACHE_VERSION.*3" Plugins/SpotOn/Plugin.pm` → match found (line 22)
- `grep -c "sub trackInfoMenu\|sub SpotOnManageLike\|sub SpotOnLike\|sub SpotOnUnlike" Plugins/SpotOn/Plugin.pm` → 4
- `grep "spotify:track:\[A-Za-z0-9\]" Plugins/SpotOn/Plugin.pm` → match found (line 384)
- `prove t/` → 261 tests, all passing
- Commits 3a4da58 and ff1f440 exist in git log
