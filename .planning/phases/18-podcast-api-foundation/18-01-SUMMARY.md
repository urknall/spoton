---
phase: 18-podcast-api-foundation
plan: 01
subsystem: api
tags: [spotify, podcasts, oauth, cache, librespot, perl]

# Dependency graph
requires:
  - phase: 13-browse-search-library
    provides: Client.pm API pattern (getSavedTracks, getAlbumTracks, getArtist)
  - phase: 04-dual-token-auth
    provides: Dual-token routing, _cacheTTL framework, cache version pattern
provides:
  - getSavedShows method (me/shows, offset-paginated)
  - getShow method (shows/{id}, with ID validation)
  - getShowEpisodes method (shows/{id}/episodes, offset-paginated, with ID validation)
  - getEpisode method (episodes/{id}, with ID validation)
  - _cacheTTL extended with podcast TTLs (60s/300s/3600s per path)
  - user-read-playback-position OAuth scope in librespot binary
  - SPOTON_CACHE_VERSION 4 for transparent token cache flush
affects:
  - phase: 19-podcast-browse (direct consumer of all 4 API methods)
  - future phases using podcast resume_point field

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "ID validation guard before URL interpolation: unless $id && $id =~ /^[A-Za-z0-9]{1,40}$/"
    - "_cacheTTL rule ordering: more-specific paths first (shows/{id}/episodes before shows/)"

key-files:
  created: []
  modified:
    - librespot-spoton/Cargo.toml
    - librespot-spoton/src/main.rs
    - Plugins/SpotOn/API/Client.pm
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/API/TokenManager.pm

key-decisions:
  - "D-01: Episode list TTL is 60s (shows/{id}/episodes) — balance between resume freshness and API call reduction"
  - "D-02: shows/{id}/episodes rule inserted BEFORE general shows/ rule to prevent shadowing by 3600s rule"
  - "SPOTON_CACHE_VERSION bumped to 4 across all three cache consumers simultaneously (Plugin.pm, Client.pm, TokenManager.pm)"

patterns-established:
  - "ID validation: all methods interpolating IDs into URL paths use /^[A-Za-z0-9]{1,40}$/ guard"
  - "Cache version sync: all three files (Plugin.pm, Client.pm, TokenManager.pm) must be bumped together"

requirements-completed:
  - API-01
  - API-02

# Metrics
duration: 12min
completed: 2026-06-14
---

# Phase 18 Plan 01: Podcast API Foundation Summary

**Four podcast API methods (getSavedShows, getShow, getShowEpisodes, getEpisode) added to Client.pm with ID validation, _cacheTTL extended with podcast-specific TTLs (60s/300s/3600s), user-read-playback-position added to librespot OAuth scope, and SPOTON_CACHE_VERSION bumped to 4**

## Performance

- **Duration:** 12 min
- **Started:** 2026-06-14T08:50:00Z
- **Completed:** 2026-06-14T09:02:00Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments
- Extended librespot binary with user-read-playback-position OAuth scope (required for episode resume_point fields) and bumped binary version to 1.1.0
- Implemented 4 podcast API methods in Client.pm following established getSavedTracks/getAlbumTracks/getArtist patterns
- Extended _cacheTTL with correct ordering: shows/{id}/episodes -> 60s (D-01, before general shows/), me/shows -> 60s, episodes/{id} -> 300s, shows/ -> 3600s
- Bumped SPOTON_CACHE_VERSION to 4 in all three cache consumers for transparent token flush on upgrade

## Task Commits

Each task was committed atomically:

1. **Task 1: Add user-read-playback-position scope to librespot binary** - `c61b552` (feat)
2. **Task 2: Add podcast API methods, extend _cacheTTL, bump cache version** - `8109cfe` (feat)

**Plan metadata:** (docs commit follows)

## Files Created/Modified
- `librespot-spoton/Cargo.toml` - Bumped version from 1.0.0 to 1.1.0
- `librespot-spoton/src/main.rs` - Added user-read-playback-position to default OAuth scope string
- `Plugins/SpotOn/API/Client.pm` - Added 4 podcast API methods, extended _cacheTTL, bumped cache version to 4
- `Plugins/SpotOn/Plugin.pm` - Bumped SPOTON_CACHE_VERSION constant from 3 to 4
- `Plugins/SpotOn/API/TokenManager.pm` - Bumped cache namespace version from 3 to 4

## Decisions Made
- D-01: 60s TTL for episode lists (shows/{id}/episodes) — compromise between resume freshness and API call reduction, per plan specification
- D-02: Rule ordering in _cacheTTL is critical: shows/{id}/episodes must precede general shows/ (3600s) to prevent episode lists from being cached for 1 hour

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None — `perl -c` checks produce expected LMS-module-not-found errors in the dev environment (JSON::XS::VersionOneAndTwo, Slim::* modules are bundled with LMS, not available in standalone Perl). All structural verification (sub existence, grep checks for cache versions and ID validation guards) passed.

## User Setup Required

None - no external service configuration required. CI will build the new librespot binary (version 1.1.0 with expanded OAuth scope) on next tag push.

## Next Phase Readiness
- Phase 19 (Podcast Browse) can consume all 4 API methods immediately
- user-read-playback-position scope will be active after next librespot binary release
- SPOTON_CACHE_VERSION bump to 4 ensures clean token flush on upgrade (no stale v3 tokens)
- ID validation guards on getShow/getShowEpisodes/getEpisode satisfy T-18-01 threat mitigation

---
*Phase: 18-podcast-api-foundation*
*Completed: 2026-06-14*
