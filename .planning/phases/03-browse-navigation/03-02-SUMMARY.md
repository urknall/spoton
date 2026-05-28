---
phase: 03-browse-navigation
plan: "02"
subsystem: plugin-feeds
tags: [plugin, feeds, opml, home, library, browse, navigation]
dependency_graph:
  requires:
    - Plugins::SpotOn::API::Client.getRecentlyPlayed
    - Plugins::SpotOn::API::Client.getTopTracks
    - Plugins::SpotOn::API::Client.getUserPlaylists
    - Plugins::SpotOn::API::Client.getSavedTracks
    - Plugins::SpotOn::API::Client.getSavedAlbums
    - Plugins::SpotOn::API::Client.getFollowedArtists
    - 16 Phase 3 i18n string keys (Plan 01)
  provides:
    - Plugins::SpotOn::Plugin.handleFeed with Home/Search/Library top-level items
    - Plugins::SpotOn::Plugin._homeFeed (3 sub-feeds)
    - Plugins::SpotOn::Plugin._libraryFeed (4 sub-feeds)
    - Plugins::SpotOn::Plugin._recentlyPlayedFeed
    - Plugins::SpotOn::Plugin._madeForYouFeed
    - Plugins::SpotOn::Plugin._topTracksFeed
    - Plugins::SpotOn::Plugin._savedTracksFeed
    - Plugins::SpotOn::Plugin._savedAlbumsFeed
    - Plugins::SpotOn::Plugin._followedArtistsFeed
    - Plugins::SpotOn::Plugin._userPlaylistsFeed
    - Plugins::SpotOn::Plugin._getAccountId (shared helper)
    - Plugins::SpotOn::Plugin._largestImage (shared helper)
    - Plugins::SpotOn::Plugin._isMadeForYou (shared helper)
    - Plugins::SpotOn::Plugin._trackItem (shared item builder)
    - Plugins::SpotOn::Plugin._albumItem (shared item builder, forward-refs Plan 03)
    - Plugins::SpotOn::Plugin._artistItem (shared item builder, forward-refs Plan 03)
    - Plugins::SpotOn::Plugin._playlistItem (shared item builder, forward-refs Plan 03)
  affects:
    - Plugins::SpotOn::Plugin (Plan 03-03 adds _searchFeed, _artistFeed, _albumFeed, _playlistFeed)
tech_stack:
  added: []
  patterns:
    - LMS OPMLBased index/quantity -> Spotify offset/limit pagination mapping (D-12)
    - cursor-based feed callbacks omit offset (Pitfall 4/7 avoidance)
    - Made-For-You detection via owner.id eq 'spotify' (D-04)
    - OPML audio item with line1/line2/play/on_select/image/duration (D-06)
    - OPML link item with passthrough for sub-feed navigation
    - _largestImage() for Spotify CDN artwork selection (D-13, Pitfall 6)
key_files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm
decisions:
  - "Forward references _albumFeed, _artistFeed, _playlistFeed, _searchFeed resolved at runtime (Perl late-binding) — defined in Plan 03-03"
  - "_userPlaylistsFeed total includes Made-For-You playlists in count (API limitation) — displayed total may be slightly off, accepted per plan must_haves"
  - "_followedArtistsFeed is single-page (limit=50) — cursor-based API does not support offset (Pitfall 4)"
metrics:
  duration: "~4 minutes"
  completed: "2026-05-28T10:59:36Z"
  tasks_completed: 1
  tasks_total: 1
  files_modified: 1
---

# Phase 3 Plan 02: Top-Level Menu + Home Feed + Library Feed Summary

**One-liner:** Plugin.pm extended with handleFeed top-level (Home/Search/Library), 6 sub-feed handlers for Home and Library, and 7 shared OPML item builders with correct pagination mapping and Made-For-You filtering.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Extend handleFeed + build Home feed and Library feed with shared helpers | 8402bfe | Plugins/SpotOn/Plugin.pm |

## What Was Built

### Plugin.pm — Feed Handler Extensions (+366 lines)

**handleFeed() extended (D-01):** After the account-switcher block, inside `if ($activeName)`, three new top-level items are pushed:
- Home → `\&_homeFeed`
- Search → `\&_searchFeed` (type: 'search'; _searchFeed defined in Plan 03-03)
- Library → `\&_libraryFeed`

**Shared Helper Functions:**

| Sub | Purpose |
|-----|---------|
| `_getAccountId($client)` | Returns per-player activeAccount pref with global fallback |
| `_largestImage($images)` | Selects highest-width image URL from Spotify images array (Pitfall 6) |
| `_isMadeForYou($playlist)` | Returns true if `owner.id eq 'spotify'` (D-04) |
| `_trackItem($client, $track)` | OPML audio item with line1/line2/play/on_select/image/duration (D-06) |
| `_albumItem($client, $album)` | OPML link item with passthrough albumId, forward-refs `\&_albumFeed` |
| `_artistItem($client, $artist)` | OPML link item with passthrough artistId, forward-refs `\&_artistFeed` |
| `_playlistItem($client, $playlist)` | OPML link item with passthrough playlistId, forward-refs `\&_playlistFeed` |

**Home Feed (D-02):**

`_homeFeed()` returns 3 items immediately (no API call): Recently Played, Made For You, Top Tracks.

| Feed | API Method | Pagination | Notes |
|------|-----------|-----------|-------|
| `_recentlyPlayedFeed` | `getRecentlyPlayed` | cursor (limit=50, no offset) | Single page; no total |
| `_madeForYouFeed` | `getUserPlaylists` | fixed offset=0, limit=50 | Filters to `_isMadeForYou()`; no total |
| `_topTracksFeed` | `getTopTracks` | fixed limit=50 | time_range=medium_term (D-05); no total |

**Library Feed (D-03):**

`_libraryFeed()` returns 4 items immediately (no API call): Liked Songs, Albums, Artists, Playlists.

| Feed | API Method | Pagination | Notes |
|------|-----------|-----------|-------|
| `_savedTracksFeed` | `getSavedTracks` | offset/limit from $args (max 50) | total => $data->{total} |
| `_savedAlbumsFeed` | `getSavedAlbums` | offset/limit from $args (max 50) | total => $data->{total} |
| `_followedArtistsFeed` | `getFollowedArtists` | cursor (limit=50, no offset) | $data->{artists}{items}; no total |
| `_userPlaylistsFeed` | `getUserPlaylists` | offset/limit from $args (max 50) | Excludes `_isMadeForYou`; total => $data->{total} |

**Pagination mapping (D-12):** All offset-paginated feeds map LMS `$args->{index}` → Spotify `offset`, `$args->{quantity}` → `limit` (capped at 50). Cursor-based feeds (`_recentlyPlayedFeed`, `_followedArtistsFeed`) ignore `$args->{index}` entirely — single-page, no offset parameter.

**Error handling:** All feed callbacks check `unless ($data)` and return a `PLUGIN_SPOTON_NO_RESULTS` textarea item.

**NAV-10 compliance:** No menu items for Browse Categories, New Releases, Artist Top Tracks, or Related Artists.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

The following forward references are intentional and will be resolved in Plan 03-03:
- `\&_searchFeed` in `handleFeed()` — Search implementation (Plan 03-03)
- `\&_albumFeed` in `_albumItem()` — Album detail page (Plan 03-03)
- `\&_artistFeed` in `_artistItem()` — Artist detail page (Plan 03-03)
- `\&_playlistFeed` in `_playlistItem()` — Playlist detail page (Plan 03-03)

These are forward references, not stubs — Perl resolves `\&sub` at call time. The code will work correctly once Plan 03-03 defines those subs. They do not prevent this plan's goal (Home and Library navigation) from functioning.

## Threat Surface Scan

No new network endpoints, auth paths, or schema changes beyond what the plan's threat model covers:
- T-03-05: Track/artist names from Spotify API displayed verbatim in OPML — no user secrets; LMS escapes HTML in rendering
- T-03-06: Raw Spotify CDN image URLs in OPML items — from trusted API response; no user-supplied URLs
- T-03-07: `_madeForYouFeed` single call with limit=50 — bounded, no amplification

## Self-Check: PASSED

Files modified:
- [x] Plugins/SpotOn/Plugin.pm — 21 subs total (5 existing + 16 new), 546 lines, braces balanced (186 open = 186 close)

Commits verified:
- [x] 8402bfe — feat(03-02): extend Plugin.pm with Home/Library feeds and shared item builders
