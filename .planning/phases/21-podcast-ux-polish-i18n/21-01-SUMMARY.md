---
phase: 21-podcast-ux-polish-i18n
plan: "01"
subsystem: Plugin.pm / UX
tags: [ux, songinfo, favorites, podcast, lazy-load, search]
dependency_graph:
  requires: []
  provides: [favorites_url on track/episode items, label-bearing sub-items for songinfo, _episodeInfoFeed lazy-load function]
  affects: [Plugins/SpotOn/Plugin.pm]
tech_stack:
  added: []
  patterns: [XMLBrowser songinfo label mechanism, OPML lazy-load sub-item pattern, LRU cache for show context]
key_files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm
decisions:
  - "UX-05: Added separate label=ARTIST/ALBUM text sub-items before nav links — NOT labels on existing link items (XMLBrowser hides labeled link items with ignore=1, destroying navigation)"
  - "UX-04: Lazy-load via separate link sub-item (PLUGIN_SPOTON_SHOW_VIEW) rather than inline API call — keeps _episodeItem synchronous and composable"
  - "Resume point (UX-02) surfaced from _episodeInfoFeed getEpisode response when scope available"
  - "Cache key spoton_ep_show_{episodeId} with 300s TTL — prevents repeat API calls for same episode across page views"
metrics:
  duration: "~15 minutes"
  completed: "2026-06-15"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 1
---

# Phase 21 Plan 01: Songinfo Labels + Episode Lazy-Load Summary

**One-liner:** Added XMLBrowser songinfo label sub-items and favorites_url to track/episode items (UX-05), plus _episodeInfoFeed lazy-load for search result episodes (UX-04).

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add songinfo labels + favorites_url to _trackItem/_episodeItem | 1c79f31 | Plugins/SpotOn/Plugin.pm |
| 2 | Implement _episodeInfoFeed lazy-load for search result episodes | 54eb394 | Plugins/SpotOn/Plugin.pm |

## Changes Made

### Task 1: Songinfo Labels + favorites_url (UX-05)

**Root cause of missing Play/Queue/Favorites buttons in Default skin:**
XMLBrowser's songinfo mechanism requires `$details` to be non-empty. `$details` is only populated when sub-items carry known `label` fields (ARTIST, ALBUM, etc.). Without this, the entire `if (scalar keys %$details)` block is skipped, and no Play/Queue/Favorites buttons are rendered — even though `favorites_url` and `playUrl` are set.

**_trackItem changes:**
- Added `{ name => $artist, type => 'text', label => 'ARTIST' }` sub-item before nav links
- Added `{ name => $album, type => 'text', label => 'ALBUM' }` sub-item before nav links
- Added `favorites_url => $track->{uri}` to `%item` hash

**_episodeItem changes:**
- Added `{ name => $showName || cstring(..., 'PLUGIN_SPOTON_PODCASTS'), type => 'text', label => 'ARTIST' }` at top of contextItems
- Added `{ name => $showName, type => 'text', label => 'ALBUM' }` when showName present
- Added `favorites_url => $episode->{uri}` to `%item` hash

**Key design constraint:** Labels must NOT be added to existing `type => 'link'` items. XMLBrowser sets `ignore => 1` on labeled items, hiding them and converting them to redirect-type entries that lose navigation. Separate text sub-items are the correct pattern (matching `Slim::Menu::TrackInfo.pm` line 400).

### Task 2: _episodeInfoFeed Lazy-Load (UX-04)

**Problem:** Search result episodes arrive as SimplifiedEpisodeObject (no `show` field in Dev Mode). `_episodeItem` was called with `undef` showContext, so no show link or Follow action appeared.

**Solution:** New `_episodeInfoFeed` function that:
1. Validates `$episodeId` against `^[A-Za-z0-9]{1,40}$` (T-21-01 mitigation)
2. Checks cache (`spoton_ep_show_{episodeId}`, 300s TTL)
3. On cache miss: calls `Client->getEpisode($accountId, $episodeId, ...)` to fetch full episode
4. Extracts `$ep->{show}` context and caches it
5. Builds show link + Follow action sub-items from show context
6. Surfaces `resume_point` status (UX-02): `fully_played` → finished message, `resume_position_ms > 0` → remaining minutes
7. Falls back to textarea with line2 if no show data available

**_episodeItem modification:** When `!$showId && $episode->{id}`, adds lazy-load link sub-item:
```
{ name => cstring($client, 'PLUGIN_SPOTON_SHOW_VIEW'), url => \&_episodeInfoFeed, passthrough => [...], type => 'link' }
```

The `label => 'ARTIST'` text sub-item (Task 1) is always present in `_episodeItem`, even for search results without showContext — ensuring Play/Queue/Favorites buttons appear before the lazy-load completes.

## Verification

```
perl t/05_perl_syntax.t         # PASS (all 8 modules)
label=ARTIST count              # 2 (track + episode)
favorites_url => $*->{uri}      # 5 occurrences (track, album, playlist, show, episode)
_episodeInfoFeed references     # 3 (sub def, url ref in _episodeItem, sub ref in lazy-load)
```

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. The `PLUGIN_SPOTON_SHOW_VIEW` and `PLUGIN_SPOTON_RESUME_*` string keys are referenced in code but not yet defined in the strings file. Per plan specification, `cstring()` returns the key name as fallback — acceptable for now. Plan 21-03 adds the actual string entries.

## Threat Flags

None — no new network endpoints, auth paths, or file access patterns beyond what the plan's threat model covers. T-21-01 (episodeId validation) is implemented in `_episodeInfoFeed` line 1.

## Self-Check

- [x] `Plugins/SpotOn/Plugin.pm` exists and modified
- [x] Commit `1c79f31` exists (Task 1)
- [x] Commit `54eb394` exists (Task 2)
- [x] `perl t/05_perl_syntax.t` passes
- [x] `label.*ARTIST` count >= 2
- [x] `favorites_url.*uri` count >= 4
- [x] `_episodeInfoFeed` count >= 3

## Self-Check: PASSED
