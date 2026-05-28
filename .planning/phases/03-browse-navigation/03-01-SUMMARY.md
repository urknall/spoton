---
phase: 03-browse-navigation
plan: "01"
subsystem: api
tags: [api, client, token-manager, i18n, spotify-web-api, cache]
dependency_graph:
  requires: []
  provides:
    - Plugins::SpotOn::API::Client.search
    - Plugins::SpotOn::API::Client.getRecentlyPlayed
    - Plugins::SpotOn::API::Client.getTopTracks
    - Plugins::SpotOn::API::Client.getSavedTracks
    - Plugins::SpotOn::API::Client.getSavedAlbums
    - Plugins::SpotOn::API::Client.getFollowedArtists
    - Plugins::SpotOn::API::Client.getUserPlaylists
    - Plugins::SpotOn::API::Client.getArtist
    - Plugins::SpotOn::API::Client.getArtistAlbums
    - Plugins::SpotOn::API::Client.getAlbum
    - Plugins::SpotOn::API::Client.getAlbumTracks
    - Plugins::SpotOn::API::Client.getPlaylistItems
    - TokenManager.REQUIRED_SCOPES with user-follow-read and playlist-read-collaborative
    - 16 Phase 3 i18n string keys
  affects:
    - Plugins::SpotOn::Plugin (feed handlers in Plans 02/03 consume these methods)
    - TokenManager (scope change forces re-auth for existing sessions)
tech_stack:
  added: []
  patterns:
    - _request() pipeline for all new API methods
    - domain-specific cache TTL dispatch (_cacheTTL)
    - cursor-based vs offset pagination distinction in method signatures
key_files:
  created: []
  modified:
    - Plugins/SpotOn/API/Client.pm
    - Plugins/SpotOn/API/TokenManager.pm
    - Plugins/SpotOn/strings.txt
decisions:
  - "getArtistAlbums accepts single include_groups per call (D-09) — combined values break pagination"
  - "getPlaylistItems uses /items path (Feb 2026 rename, Pitfall 3)"
  - "getRecentlyPlayed sets _noCache=1 — recently-played is live playback state"
  - "getFollowedArtists hardcodes type=artist — only valid type per Spotify docs"
  - "Cache TTL for me/top, me/following, me/playlists set to 60s (Library tier)"
  - "Cache TTL for search set to 300s (Browse tier)"
  - "No methods for removed Dev Mode endpoints: artist top-tracks, related-artists, browse/categories, new-releases"
metrics:
  duration: "~10 minutes"
  completed: "2026-05-28T10:52:00Z"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 3
---

# Phase 3 Plan 01: API Foundation + i18n Strings Summary

**One-liner:** 12 Spotify API methods across Browse/Search/Library added to Client.pm with correct cache TTLs, two OAuth scopes added to TokenManager, and all 16 Phase 3 i18n strings added with UTF-8 umlauts.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add 12 API endpoint methods + extend cache TTL | 5f096aa | Plugins/SpotOn/API/Client.pm |
| 2 | Extend TokenManager scopes + add Phase 3 i18n strings | 71ddc1d | Plugins/SpotOn/API/TokenManager.pm, Plugins/SpotOn/strings.txt |

## What Was Built

### Client.pm — 12 New API Methods

All methods follow the established `_request('get', path, params, $cb)` pattern from `getMe`:

| Method | Path | Pagination | Cache TTL |
|--------|------|-----------|-----------|
| `search` | `/search` | offset | 300s |
| `getRecentlyPlayed` | `/me/player/recently-played` | cursor (no offset) | 0s (live) |
| `getTopTracks` | `/me/top/tracks` | offset | 60s |
| `getSavedTracks` | `/me/tracks` | offset | 60s |
| `getSavedAlbums` | `/me/albums` | offset | 60s |
| `getFollowedArtists` | `/me/following` | cursor (after) | 60s |
| `getUserPlaylists` | `/me/playlists` | offset | 60s |
| `getArtist` | `/artists/{id}` | — | 3600s |
| `getArtistAlbums` | `/artists/{id}/albums` | offset | 3600s |
| `getAlbum` | `/albums/{id}` | — | 3600s |
| `getAlbumTracks` | `/albums/{id}/tracks` | offset | 3600s |
| `getPlaylistItems` | `/playlists/{id}/items` | offset | 300s |

**Key implementation decisions:**
- `getArtistAlbums` accepts `include_groups` as a single value per call (D-09). Combining values breaks Spotify pagination — callers must issue separate calls per type.
- `getPlaylistItems` uses `/items` path, not `/tracks` (Feb 2026 rename, Pitfall 3).
- `getFollowedArtists` hardcodes `type => 'artist'` — the only valid type per Spotify API.
- `getRecentlyPlayed` sets `_noCache => 1` — recently-played is equivalent to live playback state.

### TokenManager.pm — Two New OAuth Scopes

Added alphabetically to `REQUIRED_SCOPES`:
- `playlist-read-collaborative` — access to collaborative playlists
- `user-follow-read` — required for `GET /me/following?type=artist` (NAV-03, Pitfall 2)

This is a breaking change for existing sessions: users must re-authenticate. TokenManager's existing `refreshToken` path handles this automatically — a scope-upgraded token request will fail, triggering the re-auth flow.

### strings.txt — 16 Phase 3 i18n Keys

All keys follow the existing Tab-indented pattern with DE and EN translations. German translations use proper UTF-8 umlauts (Kürzlich, Für, Künstler, Gefällt). SON entry remains last.

New keys: PLUGIN_SPOTON_HOME, PLUGIN_SPOTON_SEARCH, PLUGIN_SPOTON_LIBRARY, PLUGIN_SPOTON_RECENTLY_PLAYED, PLUGIN_SPOTON_MADE_FOR_YOU, PLUGIN_SPOTON_TOP_TRACKS, PLUGIN_SPOTON_LIKED_SONGS, PLUGIN_SPOTON_ALBUMS, PLUGIN_SPOTON_ARTISTS, PLUGIN_SPOTON_PLAYLISTS, PLUGIN_SPOTON_SINGLES, PLUGIN_SPOTON_COMPILATIONS, PLUGIN_SPOTON_APPEARS_ON, PLUGIN_SPOTON_TOP_RESULT, PLUGIN_SPOTON_TRACKS, PLUGIN_SPOTON_NO_RESULTS.

## Deviations from Plan

None — plan executed exactly as written.

## Threat Surface Scan

No new network endpoints, auth paths, or schema changes beyond what the plan's threat model covers:
- T-03-01: `search()` q-parameter escaping handled by existing `_request()` URI builder (uri_escape on all non-_ params)
- T-03-02: New methods follow existing no-token-logging pattern
- T-03-03: Scope expansion is user-consented via PKCE re-auth
- T-03-04: All new methods reuse existing `_onSuccess` JSON parse with eval protection

## Self-Check: PASSED

Files created/modified:
- [x] Plugins/SpotOn/API/Client.pm — 18 subs (6 existing + 12 new), braces balanced
- [x] Plugins/SpotOn/API/TokenManager.pm — user-follow-read and playlist-read-collaborative present
- [x] Plugins/SpotOn/strings.txt — 51 PLUGIN_SPOTON_ entries (35 existing + 16 new), SON last

Commits verified:
- [x] 5f096aa — feat(03-01): add 12 Browse/Search/Library API methods to Client.pm
- [x] 71ddc1d — feat(03-01): extend TokenManager scopes + add Phase 3 i18n strings
