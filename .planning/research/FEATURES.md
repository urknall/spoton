# Feature Landscape: LMS Spotify Plugin (SpotOn)

**Domain:** Spotify integration plugin for Lyrion Music Server
**Researched:** 2026-05-26
**Sources:** Herger's Spotty-Plugin source (OPML.pm, API.pm, Plugin.pm, DontStopTheMusic.pm, Importer.pm), GitHub issues #35 #62 #88 #97 #115 #125 #182 #223 #224, Spotify Web API docs, February 2026 API migration guide

---

## Table Stakes

Features users expect. Missing = product feels incomplete or users stay on Spotty.

### Browse & Navigation

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Search (text query → tracks/albums/artists/playlists) | Core Spotify action; first thing users try | Low | API: `search?q=...&type=track,album,artist,playlist`. Feb 2026: limit now 10 per request (was 50), implement pagination |
| Liked Songs list | Repeatedly requested in Spotty issues (#80, #92); users expect their collection | Medium | API: `me/tracks`. Requires own Client ID (dev-mode scope: `user-library-read`). Herger gated this behind "advanced features" toggle — SpotOn should expose it unconditionally |
| Saved Albums | Part of "Your Library" experience | Medium | API: `me/albums`. Sorting: recency default, alphabetical option |
| Followed Artists | "Your Library" → Artists | Medium | API: `me/following?type=artist` |
| User Playlists | Core Spotify collection feature | Medium | API: `me/playlists`. Must handle pagination (users can have 200+ playlists) |
| Album detail page | Track list, release year, artist links | Low | API: `albums/{id}` |
| Artist detail page | Top tracks, Albums, Singles, Compilations | Medium | Artists endpoint. **CRITICAL:** Feb 2026 removed `artists/{id}/top-tracks` for dev-mode apps — only Extended Quota mode retains it. SpotOn must handle graceful absence |
| Playlist detail page | Paginated track list with metadata | Medium | API: `playlists/{id}/tracks`. Large playlists (1000+ tracks) need sequential pagination |
| Basic playback (play track, play album, play playlist) | Fundamental music playing | Low | Via `spotify://` URI + ProtocolHandler |

### Spotify Connect

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Connect receiver visibility in Spotify app | The #1 reason LMS users want a Spotify plugin; Spotty issue #224 shows how critical it is | High | One librespot daemon per LMS player. mDNS/ZeroConf announces it |
| Play/Pause/Skip from Spotify app | Expected of any Connect endpoint | High | librespot event dispatch → LMS `playlist` commands |
| Volume control from Spotify app | Expected; Spotty issue #59 shows volume-jump bugs when missing | Medium | Volume suppression window on Connect start (P-05) |
| Transfer to Connect device from Spotify app | "Tap the speaker icon and it plays" — the core Connect UX promise | High | `_doTransferPlaylist` equivalent: pull current context from `me/player`, inject into LMS queue |
| Connect with sync groups (multiroom) | LMS multiroom is a core feature; users expect Spotify to respect it | High | One daemon on master, name = concat of player names. P-17 differential restart is required |

### Authentication

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Login without browser (Keymaster/login5) | Decided in PROJECT.md; any auth flow must Just Work | High | login5 flow via librespot binary. Zero user-visible OAuth redirect |
| Token persistence across LMS restarts | Users don't want to log in after every restart | Low | Cache credentials to disk, auto-refresh |
| Multi-account support | Families/households share one LMS with multiple Spotify accounts | Medium | Per-player account assignment |

### Audio Streaming

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Reliable playback (no random stops) | Spotty issues #82, #49, #69, #76, #103 are all "playback stops" — the #1 complaint category | Medium | Correct daemon lifecycle, rate-limit guard, no parallel pagination |
| Bitrate selection (96/160/320 kbps) | Advanced users tune for bandwidth/quality; Spotty issue #27 shows users notice when ignored | Low | `--quality` flag to librespot, per-player pref |
| FLAC output for network efficiency | Squeezebox/piCorePlayer users expect FLAC; raw PCM wastes bandwidth | Low | `custom-convert.conf` FLAC pipeline via `flac -cs` |
| Seeking | Users expect scrubbing; Spotty issue #81/#102 show seek bugs cause frustration | Medium | P-13: NEVER use `['time', N]` in stream mode. Use `startOffset` |

---

## Differentiators

Features that set SpotOn apart from Herger's Spotty. These are where SpotOn wins or loses users who already know Spotty.

### Home Feed (Personalized, Not Editorial)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| "Made For You" mixes (Daily Mix 1–6) | Users love Daily Mixes — Spotty issue #1644426 shows users engaging daily with them | Medium | Accessible via category ID `0JQ5DAt0tbjZptfcdMSKl3` — Herger's trick. These are real playlists on the user account, discoverable via `me/playlists` or the category endpoint. HIGH confidence this works |
| Recently Played | Explicitly requested in Spotty issue #88; 13 comments, obvious user demand | Low | API: `me/player/recently-played`. Scope: `user-read-recently-played`. Simple |
| Top Tracks (personal, not chart) | Users want "what I actually listen to" not "what's popular in Germany" | Low | API: `me/top/tracks?time_range=medium_term`. Scope: `user-top-read` |
| Release Radar + Discover Weekly | Named directly in REQUIREMENTS.md; these are real user playlists | Low | These are normal playlists findable in `me/playlists` — filter by Spotify's known names or surface via "Made For You" category |
| Sorted home feed (Daily Mixes first, then Release Radar, then Discover Weekly) | Mirrors Spotify app priority | Low | Herger has `sortHomeItems` with this logic — replicate it |

**API note:** Herger's `home()` calls `categoryPlaylists` with the "Made For You" category ID. This still works as of 2026. Featured playlists (`browse/featured-playlists`) returns 404 intermittently (P-12) and is not personalized — deprioritize it.

### Connect Quality Improvements

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| HTTP-streaming audio transport (vs FIFO) | Eliminates seek latency (P-19), white noise on reconnect (P-20), SIGPIPE risk (P-14) | Very High | This is the architecture decision from AD-06. Requires HTTP server in librespot binary. FIFO as fallback during development |
| No seek latency in Connect mode | Spotty issue #81 shows incorrect time after pause/seek is a real complaint | Very High | Depends on HTTP transport. With FIFO: startOffset workaround only |
| Position-accurate progress bar during Connect | "Seek bar on Spotify incorrectly shows time" — Spotty issue #102 | High | P-13 + P-15 pattern: store startOffset BEFORE `playlist play` call |
| No volume jumps on Connect start | Spotty issue #59, #33 — volume resets are disruptive | Medium | P-05: suppress first N volume events after Connect start |
| Gapless Connect playback | Users notice gaps between tracks in playlists | High | P-16: sink-level rate limiting instead of EndOfTrack suppression hack |

### Library Management Write-Back

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| "Like" track from LMS context menu | Spotty issues #62, #97 — two separate requests, explicitly asked for | Low | API: `PUT /me/library` (new unified endpoint post-Feb 2026). Scope: `user-library-modify` |
| Save album from album detail page | Herger has it; users expect it | Low | Same API, album URI |
| Follow artist from artist page | Herger has it; completes the "bookmark" feature set | Low | `PUT /me/library` with artist URI (new endpoint) |
| Add track to playlist from context menu | Herger has it; power users use it constantly | Medium | API: `POST /playlists/{id}/tracks` |

### Browse Depth

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Genre/Mood browse categories | Users want to explore by feel, not just library | Medium | API: `categories` endpoint + playlists per category. **CRITICAL:** Feb 2026 removed category endpoints for dev-mode apps. Extended Quota Mode required to keep this. In dev-mode, gracefully hide this menu |
| Artist Radio | "Play more like this artist" — logical action after finding a new artist | Medium | Herger uses `recommendations` endpoint (still available). Requires `seed_artists` parameter |
| Track Radio | "Play more like this song" | Medium | Same `recommendations` endpoint with `seed_tracks` |
| Related Artists | Explore the genre graph | Low | Herger has `relatedArtists`. **CRITICAL:** Feb 2026 removed `artists/{id}/related-artists` for dev-mode. Extended Quota required |
| Recent searches (search history) | Convenience; Herger has it | Low | Client-side cache, no API call needed |
| Spotify URI / URL paste as search | Power-user shortcut; Herger has `parseUri` | Low | Parse `spotify:album:xxx` or `open.spotify.com/...` format |

### Rate-Limit Robustness

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Central throttle (no 429 bursts) | Spotty issues #55, #196, #200, #213 — 429 is the #2 complaint category after playback stops | Medium | Single `API::Client.pm` through which ALL requests flow. Sequential pagination with backpressure (P-01) |
| Graceful degradation when API unavailable | Users don't want a blank screen when Spotify is temporarily unreachable | Medium | Show cached data with staleness indicator. Continue playing current track |

### Online Library Importer

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Import Spotify library into LMS database | Enables LMS-native search of Spotify collection; Herger's `Importer.pm` is heavily used | High | `Slim::Plugin::OnlineLibraryBase`. Scans albums, artists, playlists. LMS 8.0+ only (already our floor) |

### Don't Stop The Music (DSTM)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| DSTM integration via `recommendations` | Auto-play continues after queue ends; Herger's `DontStopTheMusic.pm`; mentioned in forums | Medium | Uses `recommendations` endpoint (seed_tracks/seed_artists from current queue). This endpoint is still available in dev-mode post-Feb 2026 |

### Settings Quality of Life

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Per-player settings (bitrate, Connect on/off, normalization) | Power users tune each room differently | Low | `$prefs->client($client)` pattern |
| Sort options for library (recency / alphabetical) | Herger has it; users with large libraries need it | Low | Pref flags + sort logic in API response handling |
| Configurable pagination limit | Spotty issue #115 — users with large libraries explicitly requested this | Low | `SPOTIFY_LIMIT` constant → settings-configurable |

---

## Anti-Features

Features to explicitly NOT build in v1.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Podcast / Show support | Scope exclusion in PROJECT.md. Adds significant complexity (different playback rules, seek behavior, episode ordering). Herger has it but it's a common source of bugs. | Defer to v2. Add capability detection so it can be added cleanly later |
| Lossless / HiFi audio | PlayPlay DRM blocks it for librespot. Attempting it risks the entire project (DMCA exposure). Herger issue #180 is open because it cannot be done | Architecture must not hardcode OGG/Vorbis limitations, but do not expose UI knobs for "lossless" |
| PKCE / Browser OAuth flow | Decided in PROJECT.md. Keymaster-only is simpler, requires no Developer App registration per user, and aligns with librespot's own auth | Keymaster via librespot binary only |
| Spotify charts / "Top 50 Germany" hardcoded playlists | Herger has a hardcoded `%topuri` hash of country chart playlist URIs. Fragile (URIs change), editorial (not personalized), and duplicates what Browse Categories already provide | Surface charts via Browse Categories if Extended Quota available, else omit |
| Concurrent Browse + Connect streaming (dual-session) | Spotify enforces one active session per account. Two simultaneous sessions cause credential invalidation (P-04, CON-08) | Implement explicit state machine: Connect takes priority, Browse-streaming pauses during Connect sessions |
| Playlist folder hierarchy | Herger has `PlaylistFolders.pm` — complex, requires scraping internal Spotify desktop client data, not a public API. Fragile | Flat playlist list. Folder support if Spotify ever exposes it via official API |
| Spotify Free tier support | Plugin requires Premium for streaming (hard Spotify requirement). Free tier has skip/seek restrictions that would complicate every UX decision | Document "Premium required" clearly, fail gracefully with a message if non-Premium account detected |

---

## Feature Dependencies

```
Keymaster Auth → everything (no auth = no API calls, no Connect)
librespot binary → Connect, Streaming
Connect daemon lifecycle → Connect receiver, Transfer Playback, Sync-group handling
HTTP-streaming transport → Seek accuracy, No white noise (optional; FIFO fallback)
Extended Quota Mode → Genre/Mood categories, Related Artists, Artist Top Tracks
  (dev-mode: these three features must gracefully degrade to hidden/empty)
Liked Songs → requires user-library-read scope (unconditional in SpotOn; Herger gated it)
Like/Save write-back → requires user-library-modify scope
DSTM → requires recommendations endpoint + DSTM plugin enabled in LMS
Online Library Importer → LMS 8.0+, OnlineLibraryBase framework
Sync-group Connect → differential daemon restart (P-17), master detection
```

---

## MVP Feature Prioritization

### Ship First (table stakes, no SpotOn without these)

1. Keymaster authentication + token persistence
2. Search (tracks, albums, artists, playlists)
3. Liked Songs + Saved Albums + Followed Artists + User Playlists
4. Album/Artist/Playlist detail pages
5. Reliable playback via FLAC pipeline with bitrate selection
6. Seeking (startOffset method, not `['time', N]`)
7. Central API throttle (sequential pagination, no 429 bursts)

### Ship in Connect phase (core differentiator)

8. Connect daemon per player with mDNS
9. Play/Pause/Skip/Volume from Spotify app
10. Transfer playback to LMS player
11. Sync-group Connect (one daemon on master)
12. Volume suppression on Connect start
13. Progress bar accuracy in Connect mode

### Ship in Polish phase (differentiation over Spotty)

14. Home feed: Recently Played, Made For You mixes, sorted home items
15. Like/Save/Follow write-back from context menu
16. Artist Radio + Track Radio (via recommendations)
17. DSTM integration
18. Per-player settings UI
19. Graceful degradation when Extended Quota endpoints unavailable

### Defer to v2

- Podcast support
- Online Library Importer (high complexity, not core)
- HTTP-streaming transport (FIFO for v1 with documented limitations)
- OGG-direct passthrough (optimization, not correctness)

---

## Critical API Constraints for Features (post-Feb 2026)

These affect which features are available in development mode vs require Extended Quota:

| Feature | Dev Mode | Extended Quota | Notes |
|---------|----------|---------------|-------|
| Search | Yes, limit 10/request | Full limit | Must paginate more aggressively |
| Liked Songs / Library | Yes | Yes | New unified `PUT/DELETE /me/library` |
| Artist Top Tracks | NO | Yes | Removed for dev-mode |
| Browse Categories | NO | Yes | Removed for dev-mode |
| Related Artists | NO | Yes | Removed for dev-mode |
| New Releases | NO | Yes | Removed for dev-mode |
| recommendations | Yes | Yes | Still available in dev-mode |
| Recently Played | Yes | Yes | `me/player/recently-played` |
| Made For You mixes | Yes (via category ID) | Yes | Category endpoint + hardcoded PERSONAL_MIX_CATEGORY ID |
| me/playlists | Yes | Yes | Library access intact |
| me/top/tracks, me/top/artists | Yes | Yes | Personal top items intact |

**Architectural implication:** SpotOn must test quota mode on startup (or on first 403/404 from a browse endpoint) and hide/show features accordingly. Artist page degrades gracefully (no top tracks section) in dev-mode. Browse menu omits "Genres & Moods" in dev-mode.

---

## Sources

- Spotty-Plugin source: https://github.com/michaelherger/Spotty-Plugin
- Spotty GitHub issues (feature requests and complaints): https://github.com/michaelherger/Spotty-Plugin/issues
- Spotify Web API February 2026 changes: https://developer.spotify.com/documentation/web-api/references/changes/february-2026
- Spotify Feb 2026 migration guide: https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide
- Spotify recently played endpoint: https://developer.spotify.com/documentation/web-api/reference/get-recently-played
- Lyrion forums Spotty discussions: https://forums.lyrion.org/forum/user-forums/3rd-party-software/
