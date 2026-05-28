---
phase: 03-browse-navigation
plan: 03
subsystem: plugin-ui
tags: [search, artist-detail, album-detail, playlist-detail, context-navigation, opml, lms-feeds]
dependency_graph:
  requires: [03-01, 03-02]
  provides: [search-feed, artist-feed, album-feed, playlist-feed, context-nav]
  affects: [Plugins/SpotOn/Plugin.pm, Plugins/SpotOn/strings.txt]
tech_stack:
  added: []
  patterns: [search-feed-pattern, passthrough-pagination, album-track-context, null-track-skip]
key_files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/strings.txt
decisions:
  - D-07: Track context navigation via items array on OPML audio items
  - D-09: Artist detail shows 4 sections (Albums, Singles, Compilations, Appears On) — no Top Tracks
  - D-10: Top Result prominent above category sections; 0-result categories hidden
  - D-11: Dev-Mode-removed endpoints (Top Tracks, Related Artists) silently omitted
metrics:
  duration: "~20 minutes"
  completed: "2026-05-28T11:05:33Z"
  tasks_completed: 1
  tasks_total: 2
  files_modified: 2
---

# Phase 03 Plan 03: Search + Detail Pages + Context Navigation Summary

**One-liner:** Search with categorized results, artist/album/playlist detail pages, and track context navigation via OPML items array using limit=10 Dev Mode constraint.

## What Was Built

### Task 1: Search Feed, Detail Pages, and Context Navigation (COMPLETE)

**Commit:** `556e705`

Extended `Plugins/SpotOn/Plugin.pm` with 7 new functions and 1 modified function, and added 2 new i18n strings to `strings.txt`.

#### New Functions

**`_searchFeed($client, $callback, $args)`** — NAV-04, D-10
- Entry point for the Search menu item (type=search feeds receive query via `$args->{search}`)
- Calls `Client->search` with `type => 'track,album,artist,playlist'`, `limit => 10` (Dev Mode max)
- Builds Top Result section from `$tracks->[0]` (inline track via `items => [...]` on outline)
- Creates category sub-menu items for Tracks, Albums, Artists, Playlists
- Categories with 0 results are skipped entirely (D-10)
- Empty query returns NO_RESULTS textarea

**`_searchTypeFeed($client, $callback, $args, $passthrough)`** — NAV-11
- Paginated drill-down into one search type (track/album/artist/playlist)
- Maps LMS `index`/`quantity` to Spotify `offset`/`limit`, capped at 10 (Dev Mode)
- Maps singular type names to plural response keys (track->tracks, album->albums, etc.)
- Dispatches items through appropriate builder (`_trackItem`, `_albumItem`, `_artistItem`, `_playlistItem`)
- Returns `total` for LMS built-in pagination

**`_artistFeed($client, $callback, $args, $passthrough)`** — NAV-05, D-09
- Returns 4 section links: Albums, Singles, Compilations, Appears On
- Each section points to `_artistAlbumsFeed` with a single `includeGroups` value
- No Top Tracks (removed in Feb 2026 Dev Mode), no Related Artists (removed Nov 2024)

**`_artistAlbumsFeed($client, $callback, $args, $passthrough)`** — D-09/Pitfall 1
- Fetches paginated albums for one artist with ONE `include_groups` value per call
- Correctly maps LMS pagination to Spotify offset/limit (cap 50)
- Returns total for LMS pagination

**`_albumFeed($client, $callback, $args, $passthrough)`** — NAV-06
- For `index=0`: calls `Client->getAlbum` (album metadata + embedded first-page tracks)
- For `index>0`: calls `Client->getAlbumTracks` with correct offset
- Passes album images to `_albumTrackItem` (simplified track objects lack images)
- Returns total from `album->{tracks}{total}` or `getAlbumTracks total`

**`_albumTrackItem($client, $track, $albumImages, $albumArtist)`** — NAV-06
- `line1`: `"$track_number. $title"` (track number prefix per NAV-06)
- `line2`: featuring artists only when they differ from the album's primary artist
- Uses album images from `getAlbum` response (track objects in album context lack images)
- Adds artist context navigation via `items` array (no album view — already in album context)

**`_playlistFeed($client, $callback, $args, $passthrough)`** — NAV-07
- Maps LMS pagination to Spotify offset/limit (cap 100 per playlist items API limit)
- Skips null track entries (local files in playlists return null track objects — T-03-10)
- Made-For-You 403 fallback: `undef $data` returns NO_RESULTS textarea gracefully

#### Modified Function

**`_trackItem($client, $track)`** — D-07
- Now builds an `items` array with context navigation links when IDs are available:
  - `PLUGIN_SPOTON_ARTIST_VIEW` -> `_artistFeed` (only when `artists[0].id` present)
  - `PLUGIN_SPOTON_ALBUM_VIEW` -> `_albumFeed` (only when `album.id` present)
- The `items` field on an OPML audio item enables LMS context menu actions
- Item still works as before when IDs are missing (simplified track objects)

#### New i18n Strings

Added to `strings.txt` before the `SON` entry:
- `PLUGIN_SPOTON_ARTIST_VIEW` — DE: "Künstler anzeigen", EN: "View Artist"
- `PLUGIN_SPOTON_ALBUM_VIEW` — DE: "Album anzeigen", EN: "View Album"

### Task 2: Human Verify (PENDING — checkpoint reached)

This plan contains a `checkpoint:human-verify` gate. Manual LMS testing is required before the plan is marked complete.

## Deviations from Plan

None — plan executed exactly as written.

The `perl -c` check fails in isolation (LMS runtime dependency on Log4perl is not available outside LMS), but this is expected and consistent with Plans 01 and 02. The code is syntactically correct Perl and follows all established patterns.

## Threat Surface Scan

No new security-relevant surface beyond what the plan's `<threat_model>` identified:
- T-03-08: Search query escaping handled by `Client.pm _request()` URL builder (no additional escaping needed in Plugin.pm)
- T-03-10: Null track entries in playlist items are skipped with `defined $_->{track}` check — implemented

## Known Stubs

None — all feeds are wired to real Spotify API calls. The only conditional rendering is for missing IDs (context navigation links), which is correct behavior when simplified track objects lack album/artist IDs.

## Self-Check

**Files exist:**
- `Plugins/SpotOn/Plugin.pm` — FOUND (948 lines, 28 subs)
- `Plugins/SpotOn/strings.txt` — FOUND (ARTIST_VIEW and ALBUM_VIEW strings present)

**Commits exist:**
- `556e705` — feat(03-03): implement search, detail pages, and context navigation

## Self-Check: PASSED
