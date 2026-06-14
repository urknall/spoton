---
phase: 19-podcast-browse
plan: 02
subsystem: ui
tags: [perl, lms, opml, podcast, spotify, browse, search]

# Dependency graph
requires:
  - phase: 19-01
    provides: strings.txt podcast keys and test suite updates for podcast strings
  - phase: 18-podcast-api-foundation
    provides: getSavedShows, getShowEpisodes, search API methods in Client.pm
provides:
  - Podcasts top-level menu entry in _mainMenuFeed (NAV-01)
  - _podcastsFeed: static menu container with Meine Podcasts + Podcast-Suche
  - _savedShowsFeed: paginated saved shows browse (POD-01, NAV-02)
  - _showItem: show link item builder with publisher on line2 (D-01/D-02/D-04)
  - _showFeed: paginated episode list per show (POD-02)
  - _episodeItem: audio item with spoton://episode:ID URI and metadata cache (POD-03)
  - _formatEpisodeLine2: German duration + relative/absolute date formatting (D-05/D-06/D-07)
  - _formatRelativeDate: Heute/Gestern/Vor N Tagen/absolute date helper
  - _podcastSearchFeed: podcast text search entry with type=search LMS input (NAV-03, SRC-01/02)
  - _podcastSearchTypeFeed: typed show/episode result list with pagination (SRC-03)
affects:
  - 19-polish
  - 20-library-actions
  - 21-ux-polish

# Tech tracking
tech-stack:
  added:
    - Time::Local (Perl core module — imported at file level for _formatRelativeDate)
  patterns:
    - Episode URI pattern: spoton://episode:ID (same prefix-strip as tracks via regex /^spotify:((?:track|episode):.+)/)
    - Show artwork fallback chain: episode.images -> showImages -> episode.show.images
    - Relative date: Heute/Gestern/Vor N Tagen (0-6 days), then absolute DD. Mon [YYYY]
    - Podcast search: single combined API call with type=show,episode, separate result sections

key-files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm

key-decisions:
  - "Episode items omit playall=1 — podcast episodes are not queued like music album tracks (D-09)"
  - "_podcastSearchFeed uses type=search so LMS renders text input box (Pitfall 3)"
  - "No Top-Result in podcast search — simpler two-section layout (Shows/Episoden)"
  - "Time::Local imported at file level (not inside sub) for clarity and Perl::Critic compliance"
  - "_showFeed always uses getShowEpisodes — no embedded-episodes shortcut (Pitfall 2)"

patterns-established:
  - "Episode audio item: spoton://episode:ID URI with 7-day metadata cache (same as track)"
  - "Show item: type=link with showImages in passthrough for episode artwork fallback"
  - "_formatRelativeDate: eval-guarded timelocal, German month abbreviations array (no POSIX locale)"

requirements-completed:
  - POD-01
  - POD-02
  - POD-03
  - NAV-01
  - NAV-02
  - NAV-03
  - SRC-01
  - SRC-02
  - SRC-03

# Metrics
duration: 18min
completed: 2026-06-14
---

# Phase 19 Plan 02: Podcast Browse + Search Summary

**9 new Plugin.pm subs wiring getSavedShows/getShowEpisodes/search into OPMLBased menus with spoton://episode:ID playback and German relative-date formatting**

## Performance

- **Duration:** 18 min
- **Started:** 2026-06-14T17:30:00Z
- **Completed:** 2026-06-14T17:48:00Z
- **Tasks:** 2 (implemented together in one section, committed atomically)
- **Files modified:** 1

## Accomplishments

- All 9 podcast subs implemented in Plugin.pm: _podcastsFeed, _savedShowsFeed, _showItem, _showFeed, _episodeItem, _formatEpisodeLine2, _formatRelativeDate, _podcastSearchFeed, _podcastSearchTypeFeed
- Podcasts top-level menu entry added after Bibliothek in _mainMenuFeed
- Episode playback via spoton://episode:ID confirmed compatible with existing ProtocolHandler and librespot binary (prefix-agnostic spoton:// -> spotify: conversion)
- Full test suite 278/278 green

## Task Commits

Each task was committed atomically:

1. **Task 1 + Task 2: Core podcast navigation and search** - `3f0f26a` (feat)

**Plan metadata:** (committed below)

## Files Created/Modified

- `Plugins/SpotOn/Plugin.pm` — 346 lines added: 9 new subs + Podcasts menu entry + Time::Local import

## Decisions Made

- Episode items omit `playall => 1` — podcast episodes are not queued like music album tracks (D-09)
- `_podcastSearchFeed` uses `type => 'search'` in its menu item so LMS renders a text input box (Pitfall 3)
- No Top-Result inline in podcast search — simpler than global search (Claude's Discretion)
- `Time::Local` imported at file level with `use Time::Local qw(timelocal)` rather than inside the sub — avoids repeated redeclaration warnings
- `_showFeed` always calls `getShowEpisodes` — no embedded-episodes shortcut (Pitfall 2)
- `_formatRelativeDate` uses explicit `@months_de` array to avoid POSIX locale dependency (Anti-Pattern)

## Deviations from Plan

None — plan executed exactly as written. All patterns followed from PATTERNS.md and RESEARCH.md. Both tasks implemented in a single pass since they both insert into the same new section in Plugin.pm.

## Issues Encountered

None. The plan was well-researched with direct Perl code patterns provided in RESEARCH.md. Time::Local is a Perl core module (Assumption A1 confirmed).

## Known Stubs

None — all API calls are real, all item builders produce live data from Spotify API responses.

## Threat Flags

No new threat surface beyond what is documented in the plan's threat model. All new endpoints use existing API::Client (which handles URL encoding), and showId flows only from API response data (validated in Client.pm T-18-01).

## Next Phase Readiness

- Phase 19 complete: all 9 requirements (POD-01/02/03, NAV-01/02/03, SRC-01/02/03) implemented
- Phase 20 (Library Actions): Follow/Unfollow for shows can reuse existing `PUT/DELETE /me/library` unified endpoint pattern
- Phase 21 (UX Polish + i18n): `_formatEpisodeLine2` and `_formatRelativeDate` use hardcoded German — ready for i18n parameterization

---
*Phase: 19-podcast-browse*
*Completed: 2026-06-14*
