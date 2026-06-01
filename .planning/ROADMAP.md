# Roadmap: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Milestone:** v1
**Created:** 2026-05-26
**Granularity:** standard
**Requirements:** 62 v1 requirements mapped across 6 phases

## Phases

- [ ] **Phase 1: Plugin Skeleton + Binary Foundation** - Plugin loads in LMS, correct manifest, binary scaffolding in place
- [x] **Phase 2: Auth + API Foundation** - Authenticated Spotify API requests work; token lifecycle managed (completed 2026-05-27)
- [x] **Phase 02.1: OAuth-PKCE Browser Auth** - Replace non-functional Keymaster/login5 auth with OAuth 2.0 PKCE browser flow (completed 2026-05-27)
- [x] **Phase 3: Browse + Navigation** - Users can navigate Home, Search, and Library via LMS menus (completed 2026-05-28)
- [x] **Phase 4: Single-Track Streaming** - Users can play any Spotify track from the Browse menus (completed 2026-05-28)
- [x] **Phase 04.1: Streaming Bug Fixes + Passthrough Binary** - Fix UAT blockers and build passthrough-decoder binary (completed 2026-05-28)
- [x] **Phase 04.2: Credentials + Made For You Fix** - Own librespot credentials + category endpoint for personal mixes (completed 2026-05-29)
- [x] **Phase 04.3: ZeroConf + Keymaster Auth** - Single auth step via Spotify app replaces PKCE browser flow for credential provisioning (completed 2026-05-29)
- [x] **Phase 04.4: Dual-Token API Routing** - Dual-flavor token routing for rate-limit distribution (completed 2026-05-29)
- [ ] **Phase 5: Spotify Connect** - LMS players appear as Spotify Connect receivers; Spotify app controls playback
- [ ] **Phase 6: Polish + DSTM + Settings** - Player-specific preferences, auto-play continuation, and custom binary override functional

## Phase Details

### Phase 1: Plugin Skeleton + Binary Foundation

**Goal**: The plugin loads cleanly under LMS and all LMS integration contracts are in place before any Spotify functionality is added
**Depends on**: Nothing
**Requirements**: LMS-01, LMS-02, LMS-03, LMS-04, LMS-05, LMS-06, LMS-07
**Success Criteria** (what must be TRUE):

  1. LMS recognizes and loads the SpotOn plugin after installing the zip from the repository URL in install.xml
  2. The SpotOn settings page is accessible and renders (even if fields are empty) under LMS Settings
  3. `spotify://` URIs are registered as a protocol; attempting to play one does not crash LMS
  4. librespot binaries for x86_64, aarch64, armhf, and i386 are present; running `binary --check` returns a parseable JSON version response that satisfies the minimum version requirement
  5. All UI strings display in English and German without missing-key placeholders

**Plans**: TBD

### Phase 2: Auth + API Foundation

**Goal**: The plugin can obtain, cache, and refresh a Spotify access token via Keymaster/login5, and all outbound Spotify API calls flow through a single rate-limited, caching HTTP client
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, AUTH-06, API-01, API-02, API-03, API-04, API-05, API-06
**Success Criteria** (what must be TRUE):

  1. Configuring a Spotify account in settings causes the plugin to obtain a valid access token; the token is visible in the debug log
  2. The token is automatically refreshed before expiry with no user interaction; a 50-minute-old Connect daemon is restarted proactively
  3. Credentials file has chmod 600 and parent directory has chmod 700; confirmed via filesystem check
  4. Switching between two configured Spotify accounts causes the active token to change within one menu refresh
  5. Making 50 rapid API calls in a row produces no 429 errors; the central throttle absorbs bursts and respects `Retry-After` headers

**Plans**: TBD

### Phase 02.1: OAuth-PKCE Browser Auth (INSERTED)

**Goal:** Replace non-functional Keymaster/login5 authentication with OAuth 2.0 Authorization Code + PKCE browser flow; users authenticate via their own Spotify Developer App through a guided Setup Wizard in the LMS Settings page
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, AUTH-06, D-01, D-02, D-03, D-04, D-05, D-06, D-07, D-08, D-09, D-10, D-11, D-12
**Depends on:** Phase 02
**Success Criteria** (what must be TRUE):

  1. Entering a Spotify Developer App Client-ID in the Settings page and clicking "Mit Spotify verbinden" redirects the browser to Spotify's auth page
  2. After authenticating on Spotify, the browser returns to LMS and shows "Erfolgreich verbunden!" with the user's display name
  3. The access token is cached with TTL and automatically refreshed via refresh_token before expiry
  4. No username/password fields exist anywhere in the Settings page; the old Keymaster/login5 code is completely removed
  5. All new UI strings display correctly in both English and German

**Plans:** 4/4 plans complete

Plans:

- [x] 02.1-01-PLAN.md — TokenManager.pm PKCE rewrite + tests
- [x] 02.1-02-PLAN.md — OAuth callback route (Callback.pm) + tests
- [x] 02.1-03-PLAN.md — Settings UI integration (Setup Wizard, strings, Plugin.pm wiring)
- [x] 02.1-04-PLAN.md — Gap closure: display name in callback + REQUIREMENTS.md update

### Phase 3: Browse + Navigation

**Goal**: Users can navigate the full Spotify content hierarchy — Home, Search, Library — via LMS OPML menus
**Depends on**: Phase 2
**Requirements**: NAV-01, NAV-02, NAV-03, NAV-04, NAV-05, NAV-06, NAV-07, NAV-08, NAV-09, NAV-10, NAV-11
**Success Criteria** (what must be TRUE):

  1. The top-level SpotOn menu shows Home, Search, and Library; all three are navigable
  2. Home displays Recently Played items and at least one Made For You mix; Liked Songs appears in Library without any special configuration
  3. Searching "Radiohead" returns results grouped into Tracks, Albums, Artists, and Playlists sections
  4. Navigating into an artist shows Albums/Singles/Compilations; into an album shows the paginated tracklist with track number, duration, and featuring artists
  5. Endpoints unavailable in Dev Mode (Artist Top Tracks, Related Artists, Browse Categories, New Releases) are silently hidden rather than showing an error

**Plans:** 3/3 plans complete

Plans:
**Wave 1**

- [x] 03-01-PLAN.md — API endpoint methods (Client.pm) + TokenManager scope extension + i18n strings

**Wave 2** *(blocked on Wave 1 completion)*

- [x] 03-02-PLAN.md — Top-level menu + Home feed + Library feed + shared item builders (Plugin.pm)

**Wave 3** *(blocked on Wave 2 completion)*

- [x] 03-03-PLAN.md — Search feed + Detail pages (Artist/Album/Playlist) + context navigation + human verify

### Phase 4: Single-Track Streaming

**Goal**: Users can play any Spotify track found via Browse, with correct transcoding pipeline selection and seeking support
**Depends on**: Phase 2, Phase 3
**Requirements**: STR-01, STR-02, STR-03, STR-04, STR-05, STR-06, STR-07, STR-08, STR-09, STR-10, STR-11, LMS-11
**Success Criteria** (what must be TRUE):

  1. Selecting a track from a Browse menu plays audio through LMS within 5 seconds; audio is FLAC by default
  2. Players that support OGG receive the OGG stream directly; players that do not receive FLAC or MP3 based on capability
  3. Seeking to the middle of a track (via LMS remote or app) resumes from the correct position, not from the start
  4. Two players simultaneously starting different tracks each play their correct track (no transcoding-table race condition)
  5. Hourly cleanup runs with no orphaned librespot processes accumulating after 2 hours of normal use

**Plans:** 2/2 plans complete

Plans:
**Wave 1**

- [x] 04-01-PLAN.md — Core transcoding engine: dynamic formatOverride + updateTranscodingTable + normalization pref

**Wave 2** *(blocked on Wave 1 completion)*

- [x] 04-02-PLAN.md — Context queueing (playall), orphaned process cleanup, streaming settings UI + i18n

### Phase 04.1: Streaming Bug Fixes + Passthrough Binary (INSERTED)

**Goal:** Fix all Phase 4 UAT blockers: URL double-prefix, ProtocolHandler transcoding chain, getMetadataFor for artwork, playall feed structure, strings.txt parse errors, clientId pref prefix, and passthrough-decoder binary build
**Depends on:** Phase 4
**Requirements**: STR-01, STR-02, STR-03, STR-05, STR-06
**Success Criteria** (what must be TRUE):

  1. Selecting a track from SpotOn menus plays audio through LMS without Spotty active; no "Couldn't resolve IP address" errors in logs
  2. Artwork displays correctly for playing and queued tracks
  3. "Alle Titel" play button in album/playlist queues all tracks with working audio and artwork
  4. strings.txt produces no parse errors in LMS log on startup
  5. librespot binary includes passthrough-decoder; OGG-capable players receive OGG stream directly

**Plans:** 2/2 plans complete

Plans:
**Wave 1** *(parallel — no file overlap)*

- [x] 04.1-01-PLAN.md — Perl bug fixes: URL prefix, formatOverride, getMetadataFor, passthrough-guard, strings.txt, pref_ prefix
- [x] 04.1-02-PLAN.md — librespot-spoton binary: --single-track mode with passthrough-decoder

### Phase 04.2: Credentials + Made For You Fix (INSERTED)

**Goal:** SpotOn manages its own librespot credentials independently from Spotty, and the Made For You feed uses the correct Spotify category endpoint instead of the broken me/playlists owner filter
**Depends on:** Phase 04.1
**Requirements**: AUTH-01, AUTH-02, NAV-02
**Success Criteria** (what must be TRUE):

  1. SpotOn provisions librespot credentials in its own cache directory during OAuth setup — no manual copy from Spotty's cache needed
  2. "Für dich gemacht" shows Daily Mix, Discover Weekly, Release Radar and other personal playlists using the browse/categories endpoint
  3. Streaming works after a fresh SpotOn setup without Spotty ever being installed

**Plans:** 2/2 plans complete

Plans:
**Wave 1** *(parallel — no file overlap)*

- [x] 04.2-01-PLAN.md — Token-Login binary mode (main.rs) + credential provisioning in TokenManager.pm
- [x] 04.2-02-PLAN.md — Made For You feed: category endpoint + name-matching fallback (Client.pm + Plugin.pm)

### Phase 04.3: ZeroConf + Keymaster Auth (INSERTED)

**Goal:** Replace PKCE token-based credential provisioning with ZeroConf discovery for librespot credentials and Keymaster --get-token for Web API access — single auth step via Spotify app, no browser OAuth required for streaming
**Depends on:** Phase 04.2
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-05
**Success Criteria** (what must be TRUE):

  1. SpotOn settings show "Connect with Spotify app" instruction that starts ZeroConf discovery mode
  2. Connecting via Spotify app creates credentials.json in SpotOn cache — verified by ls -la
  3. After ZeroConf auth, all Web API calls (library, playlists, search, browse) use Keymaster tokens from --get-token
  4. Token refresh happens automatically via --get-token without user interaction
  5. PKCE OAuth flow removed or disabled — no browser redirect in auth flow

**Plans:** 4/4 plans complete

Plans:
**Wave 1**

- [x] 04.3-01-PLAN.md — Binary: --discover-once mode (librespot-discovery + ZeroConf mDNS)

**Wave 2** *(parallel — no file overlap)*

- [x] 04.3-02-PLAN.md — TokenManager.pm rewrite (Keymaster --get-token + Discovery lifecycle) + Plugin.pm wiring
- [x] 04.3-03-PLAN.md — Settings UI (ZeroConf discovery page + AJAX status endpoint + strings)

**Wave 3** *(blocked on Wave 2, requires UAT)*

- [x] 04.3-04-PLAN.md — PKCE cleanup: UAT verification + Callback.pm deletion (D-06)

### Phase 04.4: Dual-Token API Routing (INSERTED)

**Goal:** Implement dual-flavor token routing in Client.pm — own-token (eigene Client-ID) for me/* and search, bundled-token (librespot-Default-ID) for browse/categories and curated playlists — with 403/410 fallback, hint-cache, and me/* guard based on Spotty-NG reference architecture
**Requirements**: API-01, API-02, API-03
**Depends on:** Phase 04.3
**Success Criteria** (what must be TRUE):

  1. Web API calls to `me/*` endpoints always use own-token (eigene Client-ID) — never bundled
  2. Web API calls to `browse/categories` and curated playlists (`37i9*`) use bundled-token (librespot-Default-ID)
  3. A 403/410 on own-token triggers automatic retry with bundled-token and caches the hint for 24h
  4. No 429 rate-limit errors under normal Browse + Library usage patterns (dual-ID pressure distribution)
  5. Both token flavors refresh independently without user interaction

**Plans:** 2/2 plans complete

Plans:
**Wave 1** *(parallel — no file overlap)*

- [x] 04.4-01-PLAN.md — TokenManager.pm flavor-aware interface + Client.pm dual-token routing pipeline
- [x] 04.4-02-PLAN.md — Settings UI: Client-ID field, degraded-mode warning, i18n strings

### Phase 5: Spotify Connect

**Goal**: Every LMS player appears as a Spotify Connect receiver; transferring playback from the Spotify app to any LMS player starts audio within 3 seconds, and Spotify app transport controls work
**Depends on**: Phase 4
**Requirements**: CON-01, CON-02, CON-03, CON-04, CON-05, CON-06, CON-07, CON-08, CON-09, CON-10, CON-11, CON-12, CON-13, CON-14, CON-15, CON-16, CON-17
**Success Criteria** (what must be TRUE):

  1. Each LMS player appears as a separate device in the Spotify app's device list; a sync group appears as a single merged device named after the grouped players
  2. Transferring playback from the Spotify app to an LMS player starts audio within 3 seconds with correct position (no volume jump in the first second)
  3. Play, Pause, Skip Next, Skip Previous, and Volume from the Spotify app all take effect on the LMS player within 2 seconds
  4. Starting a Browse-streaming session while Connect is active stops the Connect daemon cleanly; starting Connect while Browse-streaming is active stops the local playback cleanly
  5. A Connect daemon that crashes is automatically restarted with exponential backoff; Connect daemons are never killed by LMS's `killHangingProcesses`

**Plans:** 4/5 plans executed

Plans:
**Wave 1** *(parallel — no file overlap)*

- [x] 05-01-PLAN.md — Rust binary: Cargo.toml deps + connect.rs (LMS notifier, HttpStreamSink, http_stream_server, run_connect) + main.rs wiring + binary build
- [x] 05-02-PLAN.md — Config files (custom-convert.conf soc profiles, custom-types.conf soc type) + Daemon.pm process wrapper

**Wave 2** *(depends on Wave 1)*

- [x] 05-03-PLAN.md — DaemonManager.pm lifecycle + Plugin.pm integration (Connect init at boot, CON-09 PID exclusion, updateTranscodingTable soc)

**Wave 3** *(depends on Wave 2)*

- [x] 05-04-PLAN.md — Connect.pm event dispatch + ProtocolHandler.pm Connect extensions (soc format, canDirectStream, sync proxy)

**Wave 4** *(depends on Wave 3)*

- [ ] 05-05-PLAN.md — Settings UI (per-player Connect toggle, OGG override) + i18n strings + UAT checkpoint

### Phase 6: Polish + DSTM + Settings

**Goal**: Player-specific preferences are applied per player, auto-play continues music after a queue ends, and power users can supply their own librespot binary
**Depends on**: Phase 3, Phase 4, Phase 5
**Requirements**: LMS-08, LMS-09, LMS-10
**Success Criteria** (what must be TRUE):

  1. Setting bitrate to 96 kbps on one player and 320 kbps on another causes each player to stream at its configured bitrate independently
  2. When a playlist ends, Don't Stop The Music automatically queues a related Spotify track and playback continues without user intervention
  3. Placing a custom librespot binary in the designated path causes the plugin to use that binary instead of the bundled one; the `--check` version enforcement still applies
  4. Tapping a single track in an album or playlist queues all visible tracks and starts at the tapped track (playall fix from Phase 04.1 UAT backlog)

**Plans**: TBD

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Plugin Skeleton + Binary Foundation | 0/? | Not started | - |
| 2. Auth + API Foundation | 6/6 | Complete   | 2026-05-27 |
| 02.1. OAuth-PKCE Browser Auth | 4/4 | Complete    | 2026-05-27 |
| 3. Browse + Navigation | 3/3 | Complete   | 2026-05-28 |
| 4. Single-Track Streaming | 2/2 | Complete   | 2026-05-28 |
| 04.1. Streaming Bug Fixes + Passthrough Binary | 2/2 | Complete   | 2026-05-28 |
| 04.2. Credentials + Made For You Fix | 2/2 | Complete   | 2026-05-29 |
| 04.3. ZeroConf + Keymaster Auth | 4/4 | Complete   | 2026-05-29 |
| 04.4. Dual-Token API Routing | 2/2 | Complete   | 2026-05-29 |
| 5. Spotify Connect | 4/5 | In Progress|  |
| 6. Polish + DSTM + Settings | 0/? | Not started | - |

## Backlog

Items discovered during UAT — not blocking, schedule into future phases.

1. **Dead Code Cleanup (Phase 04.4)** — Remove `_isMadeForYou` No-Op filter in Plugin.pm:689, dead `_onSuccess`/`_onError` subs in Client.pm:530-604, unused `RATE_LIMIT_CACHE_KEY` constant. ~80 lines. Candidates for Phase 6 or standalone `/gsd-quick`.
2. **Eigene SpotOn Client-ID bei Spotify registrieren** — Aktuell nutzt bundled-Token Hergers Spotty-NG App-ID (`93aac68...`). Langfristig braucht SpotOn eine eigene registrierte App mit Extended Quota Mode für browse/categories-Zugriff.
3. **playall auf Track-Items** — XMLBrowser ignoriert playall; muss in Phase 6 oder eigener Phase gefixt werden (aus Phase 04.1 UAT).

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-06-01 after Phase 05 planning*
