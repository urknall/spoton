---
phase: 33-more-context-menu
plan: 01
subsystem: ui
tags: [lms, opml, track-info, context-menu, cache]

requires:
  - phase: 15-like-button
    provides: Like/Unlike infrastructure (SpotOnManageLike, registerInfoProvider)
  - phase: 18-21 (v1.5 Podcasts)
    provides: Episode/Show feed functions (_showFeed, SpotOnManageFollow)
provides:
  - Full More context menu with Artist View, Album View, Like/Unlike for tracks
  - View Show and Follow/Unfollow for episodes
  - Extended metadata cache with artistId/albumId/showId across all cache sites
affects: []

tech-stack:
  added: []
  patterns: [trackInfoMenu arrayref return, cache-driven context navigation]

key-files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/Connect.pm

key-decisions:
  - "trackInfoMenu returns arrayref (multi-item), consistent with Spotty and LMS convention"
  - "registerInfoProvider after => 'top' so SpotOn items appear after standard LMS items"
  - "artistIds stored as JSON-encoded string for future multi-artist display"
  - "Episode Follow/View Show guarded behind showId presence check"

patterns-established:
  - "Cache-driven context menu: store IDs at cache-write time, read at menu-build time — no live API call needed"

requirements-completed: [GH-29]

duration: 25min
completed: 2026-06-26
---

# Phase 33: More Context Menu Summary

**Full More context menu with Artist View, Album View, Like/Unlike for tracks and View Show, Follow for episodes — resolves GH #29**

## Performance

- **Duration:** ~25 min (manual, after failed executor agent)
- **Started:** 2026-06-26T08:41:35Z (initial attempt); resumed inline
- **Completed:** 2026-06-26
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Extended metadata cache with artistId/albumId/showId across all 6 cache-write sites (Plugin _trackItem + _episodeItem, ProtocolHandler x3, Connect)
- Rewrote trackInfoMenu to return arrayref: Artist View, Album View, Like/Unlike for tracks; View Show, Follow for episodes
- Updated trackInfoURL in ProtocolHandler to mirror the same items
- Fixed registerInfoProvider positioning with `after => 'top'`

## Task Commits

1. **Task 1+2: Cache extension + Menu rewrite** — `13025a4` (feat)
2. **Review fix: Episode Follow showId guard** — `8be296e` (fix)
3. **UAT fix: _episodeItem cache missing showId/showName** — `a276e22` (fix)

**Plan metadata:** `ef39d0c` (docs: create phase plan)

## Files Created/Modified
- `Plugins/SpotOn/Plugin.pm` — trackInfoMenu rewrite (arrayref, track+episode), _trackItem + _episodeItem cache extension, registerInfoProvider after=>'top', JSON::XS import
- `Plugins/SpotOn/ProtocolHandler.pm` — _cacheExplodedTrack (+albumId param), _cacheExplodedEpisode (+showId/showName), _asyncRefetch (+ID fields), trackInfoURL (episode support + Artist/Album View), JSON::XS import
- `Plugins/SpotOn/Connect.pm` — _fetchTrackMetadata cache extension (+artistId/artistIds/albumId)

## Decisions Made
- Execute inline instead of subagent — Sonnet executor agent ran but failed to commit or complete changes
- artistIds stored as JSON-encoded string, not decoded on read — future-proofing for multi-artist display

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Episode Follow without showId guard**
- **Found during:** Code review after Task 2
- **Issue:** Follow/Unfollow item pushed without checking showId, producing invalid URI `spotify:show:` on cache miss
- **Fix:** Moved Follow item inside `if ($meta->{showId})` block, matching trackInfoURL behavior
- **Files modified:** Plugins/SpotOn/Plugin.pm
- **Verification:** Brace balance check passed
- **Committed in:** `8be296e`

**2. [Rule 2 - Missing Critical] _episodeItem cache missing showId/showName**
- **Found during:** Local UAT — episode More menu was empty
- **Issue:** `_episodeItem` in Plugin.pm (6th cache-write site) was missed by plan — no showId/showName stored, so trackInfoMenu returned no items for episodes
- **Fix:** Added showId and showName to the cache hash in _episodeItem
- **Files modified:** Plugins/SpotOn/Plugin.pm
- **Verification:** Local UAT — View Show and Follow/Unfollow now appear for episodes
- **Committed in:** `a276e22`

---

**Total deviations:** 2 auto-fixed (2 missing critical)
**Impact on plan:** Essential correctness fix. No scope creep.

## Issues Encountered
- Initial Sonnet executor agent failed silently — modified Plugin.pm _trackItem but never committed. Resumed with inline execution.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- No further phases planned — Phase 33 is standalone (not part of a milestone)
- Backlog items available: Spotty Favorites Migration (#6), Search Pagination (#7)

---
*Phase: 33-more-context-menu*
*Completed: 2026-06-26*
