---
phase: 20-podcast-library-actions
plan: 01
subsystem: api
tags: [spotify, podcast, library, follow, unfollow, perl, lms]

requires:
  - phase: 19-podcast-browse
    provides: "_showItem/_showFeed OPML pattern, episode passthrough structure"
  - phase: 15-like-unlike
    provides: "Like/Unlike action pattern (_doLibraryAction, SpotOnManageLike, _likedCacheKey)"

provides:
  - "saveShows/removeShows/checkShows methods in Client.pm (PUT/DELETE/GET /me/library)"
  - "SpotOnManageFollow sub — dynamic Follow/Unfollow menu item with 60s cache"
  - "SpotOnFollowShow/SpotOnUnfollowShow subs — action handlers"
  - "_followCacheKey sub — spoton_followed_{accountId}_{showId}"
  - "_doShowLibraryAction sub — generic show library mutation with cache invalidation"
  - "_showItem passthrough extended with showUri field"
  - "_showFeed prepends Follow/Unfollow action as first episode-list item"
  - "6 new i18n string keys in 11 languages"

affects: [21-ux-polish, library, podcast-browse]

tech-stack:
  added: []
  patterns:
    - "Show library actions mirror Track library actions exactly (saveShows/removeShows/checkShows = saveTracks/removeTracks/checkTracks)"
    - "Cache invalidation: spoton_resp_me/shows removed on follow/unfollow so list reflects change immediately"
    - "T-20-01 URI validation: ^spotify:show:[A-Za-z0-9]+$ at all entry points (SpotOnManageFollow + _showFeed guard)"

key-files:
  created: []
  modified:
    - Plugins/SpotOn/API/Client.pm
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/strings.txt
    - t/02_strings.t
    - t/08_api_client.t

key-decisions:
  - "Follow/Unfollow uses existing unified PUT/DELETE /me/library (same as Like) — no new endpoint needed"
  - "Cache invalidation key is spoton_resp_me/shows — same pattern as Like invalidates me/tracks"
  - "403 error reuses PLUGIN_SPOTON_LIKE_ERROR_SCOPE string — same OAuth scope (user-library-modify) applies to shows"

patterns-established:
  - "Show library action pattern mirrors track library action pattern exactly"

requirements-completed: [POD-04, POD-05]

duration: 18min
completed: 2026-06-15
---

# Phase 20 Plan 01: Podcast Follow/Unfollow Summary

**Follow/Unfollow action for podcast shows via PUT/DELETE /me/library with cache invalidation, 60s follow-state cache, and URI validation — mirrors the existing Like/Unlike pattern exactly**

## Performance

- **Duration:** 18 min
- **Started:** 2026-06-15T~T (sequential execution)
- **Completed:** 2026-06-15
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments

- 3 new Client.pm methods (saveShows, removeShows, checkShows) mirroring the Track equivalents, all pointing at the unified /me/library endpoint
- 5 new Plugin.pm subs implementing the complete Follow/Unfollow action flow: SpotOnManageFollow (dynamic menu), SpotOnFollowShow, SpotOnUnfollowShow, _followCacheKey, _doShowLibraryAction
- _showItem passthrough extended with `showUri` field; _showFeed prepends "Folgen / Entfolgen" as first item in every episode list
- 6 new i18n string keys across 11 languages; LIB-06..LIB-09 test coverage for show library methods; t/02_strings.t bilingual validation updated

## Task Commits

1. **Task 1: Client.pm show library methods + strings.txt keys + test updates** - `b6fec28` (feat)
2. **Task 2: Plugin.pm Follow/Unfollow subs + _showItem/_showFeed wiring** - `994164a` (feat)

## Files Created/Modified

- `Plugins/SpotOn/API/Client.pm` — saveShows, removeShows, checkShows methods (after checkTracks, before getUserPlaylists)
- `Plugins/SpotOn/Plugin.pm` — SpotOnManageFollow, SpotOnFollowShow, SpotOnUnfollowShow, _followCacheKey, _doShowLibraryAction; _showItem passthrough + _showFeed action prepend
- `Plugins/SpotOn/strings.txt` — 6 new keys (MANAGE_FOLLOW, FOLLOW_SHOW, UNFOLLOW_SHOW, SHOW_FOLLOWED, SHOW_UNFOLLOWED, SHOW_ACTION_ERROR) in 11 languages
- `t/02_strings.t` — 6 new keys added to bilingual_keys validation list
- `t/08_api_client.t` — LIB-06..LIB-09 tests for saveShows/removeShows/checkShows

## Decisions Made

- Follow/Unfollow reuses the unified `PUT/DELETE /me/library` endpoint — no new endpoint needed (same as POD decision from STATE.md)
- 403 error handler reuses `PLUGIN_SPOTON_LIKE_ERROR_SCOPE` string — same OAuth scope (`user-library-modify`) covers shows and tracks
- Cache invalidation key `spoton_resp_me/shows` cleared on follow/unfollow so "Meine Podcasts" reflects changes immediately without waiting for 60s TTL expiry

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- POD-04 and POD-05 complete: users can follow/unfollow podcast shows from within the episode list
- Phase 21 (UX Polish + i18n) can proceed — all functional podcast features are now in place
- Manual verification recommended: LMS -> SpotOn -> Podcasts -> open any show -> confirm "Folgen / Entfolgen" appears as first item, action works, notification shows, "Meine Podcasts" updates on return

## Known Stubs

None.

## Threat Flags

None — no new network endpoints or trust boundaries beyond those in the plan's threat model (T-20-01 URI validation implemented at both SpotOnManageFollow and _showFeed entry points).

## Self-Check: PASSED

- `Plugins/SpotOn/API/Client.pm` — modified, exists
- `Plugins/SpotOn/Plugin.pm` — modified, exists
- `Plugins/SpotOn/strings.txt` — modified, exists
- `t/02_strings.t` — modified, exists
- `t/08_api_client.t` — modified, exists
- Commits b6fec28 and 994164a verified in git log

---
*Phase: 20-podcast-library-actions*
*Completed: 2026-06-15*
