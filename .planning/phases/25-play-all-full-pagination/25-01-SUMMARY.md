---
phase: 25-play-all-full-pagination
plan: 01
subsystem: api
tags: [spotify, pagination, play-all, liked-songs, playlists, albums, podcasts]

# Dependency graph
requires:
  - phase: 19-podcast-browse
    provides: _showFeed, _episodeItem, getShowEpisodes — show browse infrastructure this extends
  - phase: 03-playlist-browse
    provides: _playlistFeed, getPlaylistItems — playlist browse infrastructure this extends
  - phase: 07-library-browse
    provides: _savedTracksFeed, getSavedTracks — library browse infrastructure this extends
  - phase: 08-album-browse
    provides: _albumFeed, getAlbum, getAlbumTracks — album browse infrastructure this extends
provides:
  - _fetchAllPages: reusable async offset paginator helper in Plugin.pm
  - Full-pagination play-all for liked songs (all liked tracks, no 50-track cap)
  - Full-pagination play-all for playlists (all playlist tracks, no 100-track cap)
  - Full-pagination play-all for albums (all album tracks, seeded from getAlbum response)
  - Full-pagination play-all for shows (all episodes, no 50-episode cap)
  - Full recursive pagination for ProtocolHandler show-explode path (spoton://show: URIs)
affects:
  - any-future-browse-additions: pattern for full-pagination play-all is now established

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "_fetchAllPages reusable async paginator: $apiFn/$extractItems/$done hashref API, recursive $fetchPage closure"
    - "Play-all detection: qty > pageLimit && offset == 0 triggers full pagination, else single-page browse"
    - "Album play-all: seed accumulator from getAlbum response first page, fetch remaining via getAlbumTracks"
    - "T-25-01 empty-page guard: stop recursion if current page returns 0 items regardless of total field"

key-files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/ProtocolHandler.pm

key-decisions:
  - "play-all detection: qty > pageLimit AND offset == 0 (LMS XMLBrowser sends qty=999999 for play-all)"
  - "album play-all: seed accumulator from getAlbum embedded tracks to avoid redundant API call"
  - "show play-all: Follow button excluded (not a playable item)"
  - "_fetchAllPages generic helper used for savedTracks/playlists/shows; albumFeed uses inline $fetchPage (needs getAlbum first)"
  - "T-25-01 mitigated: empty-page guard prevents infinite loop if Spotify returns total > actual items"

patterns-established:
  - "Full-pagination pattern: detect play-all by qty > pageLimit AND offset == 0, else fall through to single-page"

requirements-completed: []

# Metrics
duration: 20min
completed: 2026-06-20
---

# Phase 25 Plan 01: Play-All Full Pagination Summary

**Reusable `_fetchAllPages` async paginator integrated into all four feed functions and ProtocolHandler show-explode path, fixing GitHub Issue #16 (play-all truncated at first API page)**

## Performance

- **Duration:** ~20 min
- **Started:** 2026-06-20T11:02:00Z
- **Completed:** 2026-06-20T11:22:35Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments

- Created `_fetchAllPages` reusable async paginator with recursive `$fetchPage` closure, T-25-01 empty-page guard, and graceful error degradation
- Integrated play-all detection into `_savedTracksFeed` (>50 triggers full pagination over getSavedTracks)
- Integrated play-all detection into `_playlistFeed` (>100 triggers full pagination over getPlaylistItems)
- Integrated play-all detection into `_albumFeed` (>50 triggers getAlbum seed + full pagination over getAlbumTracks)
- Integrated play-all detection into `_showFeed` (>50 triggers full pagination over getShowEpisodes, Follow button excluded)
- Replaced single-page show-explode in ProtocolHandler with recursive `$fetchPage` closure matching album/playlist pattern

## Task Commits

1. **Task 1: _fetchAllPages helper + all four feed integrations** - `79d3ae0` (feat)
2. **Task 2: ProtocolHandler show-explode full pagination** - `9674fb4` (feat)

## Files Created/Modified

- `Plugins/SpotOn/Plugin.pm` - New `_fetchAllPages` helper; updated `_savedTracksFeed`, `_playlistFeed`, `_albumFeed`, `_showFeed` with play-all branches
- `Plugins/SpotOn/ProtocolHandler.pm` - Show-explode block replaced with recursive `$fetchPage` closure

## Decisions Made

- Play-all detection threshold: `qty > pageLimit AND offset == 0`. LMS XMLBrowser sends a very large quantity (typically 999999) and index=0 for play-all. Normal browse requests use modest quantities and may have non-zero offsets.
- Album play-all seeds accumulator from `getAlbum` embedded first-page tracks to avoid a redundant `getAlbumTracks` call for the first 50 tracks.
- Show play-all excludes the Follow button (not a playable item). The existing browse code includes it only in normal browse mode.
- `_fetchAllPages` generic helper used for savedTracks, playlists, and shows. Album play-all uses an inline `$fetchPage` (because it needs `getAlbum` first for metadata).

## Deviations from Plan

None - plan executed exactly as written. The `_albumFeed` play-all was implemented inline rather than via the generic `_fetchAllPages` helper due to the required `getAlbum` prefetch, exactly as the plan specified ("Note: getAlbum response includes first page of tracks — seed the accumulator with those").

## Issues Encountered

None. `perl -c` syntax check ran successfully via the stub-based `t/05_perl_syntax.t` test (the LMS runtime environment requires stubs for standalone syntax checking).

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Play-all now works for all track/episode feed types with arbitrarily large libraries
- Ready for UAT: test play-all on a liked-songs library >50, a playlist >100 tracks, an album >50 tracks, and a show >50 episodes
- No blockers

---
*Phase: 25-play-all-full-pagination*
*Completed: 2026-06-20*
