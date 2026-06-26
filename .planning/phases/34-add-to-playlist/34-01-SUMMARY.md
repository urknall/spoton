---
phase: 34-add-to-playlist
plan: 01
subsystem: ui, api
tags: [lms, opml, context-menu, playlist, spotify-api]

requires:
  - phase: 33-more-context-menu
    provides: trackInfoMenu arrayref infrastructure, registerInfoProvider positioning
provides:
  - Add to Playlist menu item in More context menu for tracks and episodes
  - Playlist picker with pagination and Made-For-You filter
  - addToPlaylist API method (POST /playlists/{id}/items)
affects: []

tech-stack:
  added: []
  patterns: [playlist-picker-callback, two-step-action-menu]

key-files:
  created: []
  modified:
    - Plugins/SpotOn/API/Client.pm
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/strings.txt

key-decisions:
  - "Spotify playlists, not LMS playlists — LMS handles its own playlists natively"
  - "Query param for uris (not JSON body) — works with existing _request infrastructure"
  - "Made-For-You playlists filtered from picker — consistent with _userPlaylistsFeed"

patterns-established:
  - "Two-step action menu: picker handler (SpotOnAddToPlaylist) → action callback (_doAddToPlaylist) with showBriefly confirmation"

requirements-completed: [ATP-01, ATP-02, ATP-03]

duration: 20min
completed: 2026-06-26
---

# Phase 34: Add to Playlist Summary

**"Add to Playlist" in More context menu for tracks and episodes — paginated Spotify playlist picker with API integration**

## Performance

- **Duration:** ~20 min
- **Started:** 2026-06-26
- **Completed:** 2026-06-26
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- New `addToPlaylist` API method in Client.pm (POST /playlists/{id}/items)
- `SpotOnAddToPlaylist` playlist picker with pagination and Made-For-You filter
- `_doAddToPlaylist` callback with showBriefly confirmation popup
- Menu item in trackInfoMenu and trackInfoURL for both tracks and episodes
- 4 i18n string blocks in 11 languages

## Task Commits

1. **Task 1: API method + i18n strings** — `74de8ca` (feat)
2. **Task 2: Playlist picker + menu items** — `a6330d3` (feat)
3. **Review fix: Pagination $params** — `47c1fa7` (fix)

**Plan metadata:** `ef39d0c` (docs: create phase plan)

## Files Created/Modified
- `Plugins/SpotOn/API/Client.pm` — addToPlaylist method
- `Plugins/SpotOn/Plugin.pm` — SpotOnAddToPlaylist, _doAddToPlaylist, menu items in trackInfoMenu
- `Plugins/SpotOn/ProtocolHandler.pm` — menu items in trackInfoURL
- `Plugins/SpotOn/strings.txt` — ADD_TO_PLAYLIST, SELECT_PLAYLIST, ADDED_TO_PLAYLIST, ADD_TO_PLAYLIST_ERROR

## Decisions Made
- Spotify playlists (not LMS) — LMS handles its own playlists natively, SpotOn adds Spotify-side organization
- Query param approach for uris — works with existing Client.pm _request infrastructure, confirmed working

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Pagination reads from wrong variable**
- **Found during:** Code review after Task 2
- **Issue:** `$args->{index}` instead of `$params->{index}` — offset always 0, users with 50+ playlists saw same first page
- **Fix:** Changed to `$params->{index}` and `$params->{quantity}`
- **Files modified:** Plugins/SpotOn/Plugin.pm
- **Verification:** Code review confirmed fix matches all other paginated handlers
- **Committed in:** `47c1fa7`

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Essential pagination fix. No scope creep.

## Issues Encountered
None.

## User Setup Required
None — no external service configuration required.

## Next Phase Readiness
- No further phases planned
- Backlog items available: Spotty Favorites Migration (#6), Search Pagination (#7)

---
*Phase: 34-add-to-playlist*
*Completed: 2026-06-26*
