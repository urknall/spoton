# Roadmap: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Created:** 2026-05-26
**Granularity:** standard

## Milestones

- ✅ **v1.0 Foundation** — Phases 1-6 (shipped 2026-06-03)
- ✅ **v1.1 Hardening & Reach** — Phases 7-12 (shipped 2026-06-06)
- ✅ **v1.3 Polish & Publish** — Phases 13-16.1 (shipped 2026-06-13)
- ✅ **v1.5 Podcasts** — Phases 18-21 (shipped 2026-06-15)
- ✅ **v2.0 Browse Daemon Migration** — Phases 22, 25-32 (shipped 2026-06-25)
- ✅ **v2.1 Context Menu** — Phases 33-34 (shipped 2026-06-26)
- ✅ **v2.2 Session Health** — Phase 36 (shipped 2026-06-30)
- 🚧 **v2.3 Library Integration** — Phases 37-41
- 📋 **v3.0 Auth Overhaul** — Phases 49-53 (PKCE + sp_dc/Pathfinder, spike-validated)

## Phases

<details>
<summary>✅ v1.0 Foundation (Phases 1-6) — SHIPPED 2026-06-03</summary>

- [x] **Phase 1: Plugin Skeleton + Binary Foundation** — completed 2026-05-26
- [x] **Phase 2: Auth + API Foundation** (6/6 plans) — completed 2026-05-27
- [x] **Phase 02.1: OAuth-PKCE Browser Auth** (4/4 plans) — completed 2026-05-27
- [x] **Phase 3: Browse + Navigation** (3/3 plans) — completed 2026-05-28
- [x] **Phase 4: Single-Track Streaming** (2/2 plans) — completed 2026-05-28
- [x] **Phase 04.1: Streaming Bug Fixes + Passthrough Binary** (2/2 plans) — completed 2026-05-28
- [x] **Phase 04.2: Credentials + Made For You Fix** (2/2 plans) — completed 2026-05-29
- [x] **Phase 04.3: ZeroConf + Keymaster Auth** (4/4 plans) — completed 2026-05-29
- [x] **Phase 04.4: Dual-Token API Routing** (2/2 plans) — completed 2026-05-29
- [x] **Phase 5: Spotify Connect** (5/5 plans) — completed 2026-06-01
- [x] **Phase 05.1: Connect Audio Streaming Bugfix** (3/3 plans) — completed 2026-06-01
- [x] **Phase 05.2: Connect Controls & Resume** (2/2 plans) — completed 2026-06-01
- [x] **Phase 05.3: Sync Groups + Connect Robustness** (3/3 plans) — completed 2026-06-02
- [x] **Phase 05.4: mDNS Connect Discovery Fix** (3/3 plans) — completed 2026-06-02
- [x] **Phase 6: Polish + DSTM + Settings** (5/5 plans) — completed 2026-06-03

</details>

<details>
<summary>✅ v1.1 Hardening & Reach (Phases 7-12) — SHIPPED 2026-06-06</summary>

- [x] **Phase 7: DE→EN Code Cleanup** (1/1 plan) — completed 2026-06-03
- [x] **Phase 8: Multi-Arch Binary Distribution** (2/2 plans) — completed 2026-06-03
- [x] **Phase 9: Stream Metadata** (1/1 plan) — completed 2026-06-04
- [x] **Phase 9.5: Prod Deployment & Monitoring** (2/2 plans) — completed 2026-06-04
- [x] **Phase 10: Connect-DSTM** (3/3 plans) — completed 2026-06-04
- [x] **Phase 11: Track History Metadata** (2/2 plans) — completed 2026-06-05
- [x] **Phase 12: Protocol Handler Rename** (2/2 plans) — completed 2026-06-05

</details>

<details>
<summary>✅ v1.3 Polish & Publish (Phases 13-16.1) — SHIPPED 2026-06-13</summary>

- [x] **Phase 13: Repo Maintenance** (2/2 plans) — completed 2026-06-07
- [x] **Phase 14: Connect Fixes** (2/2 plans) — completed 2026-06-07
- [x] **Phase 15: Like Button** (2/2 plans) — completed 2026-06-11
- [x] **Phase 16: macOS Universal Binary** (2/2 plans) — completed 2026-06-11
- [x] **Phase 16.1: CI Conditional Build** (1/1 plan) — completed 2026-06-12

</details>

- ✅ **v1.5 Podcasts** — Phases 18-21 (shipped 2026-06-15) → [archive](milestones/v1.5-ROADMAP.md)
- ✅ **v2.0 Browse Daemon Migration** — Phases 22, 25-32 (shipped 2026-06-25) → [archive](milestones/v2.0-ROADMAP.md)
- ✅ **v2.1 Context Menu** — Phases 33-34 (shipped 2026-06-26)

### v2.3 Library Integration

**Milestone Goal:** Spotify library in LMS native library -- Liked Songs, Saved Albums, Followed Artists searchable and browsable in My Music, with incremental sync and optional playlist import.

- [x] **Phase 37: Context Menu LMS Items** — Standard LMS actions (Add to Favorites, etc.) alongside SpotOn entries in More menu (completed 2026-06-30)
- [ ] **Phase 38: Importer Foundation** — Importer.pm skeleton, install.xml registration, scanner throttle, token routing, Import Library preference
- [ ] **Phase 39: Album + Artist Import** — Saved albums and followed artists in LMS My Music with icon badge and progress indicator
- [ ] **Phase 40: Liked Songs + Incremental Sync** — Liked songs in LMS with global search, added_at early-exit, needsUpdate(), status page stats
- [ ] **Phase 41: Playlist Import** — Opt-in playlist import with snapshot_id change detection
- [x] **Phase 42: OGG Vorbis Passthrough** — Rust sinks forward OGG packets to capable players (squeezelite), PCM fallback for hardware players (#96) (completed 2026-07-01)

## Phase Details

### Phase 37: Context Menu LMS Items

**Goal**: Standard LMS menu entries appear alongside SpotOn entries in the More menu (GH #55)
**Depends on**: Nothing (independent of library work)
**Requirements**: CTX-01
**Success Criteria** (what must be TRUE):

  1. User sees standard LMS actions (Add to Favorites, Add to Playlist, More Info) in SpotOn's More menu for tracks, albums, and artists
  2. Standard LMS actions execute correctly -- adding a SpotOn track to LMS Favorites actually creates a working favorite entry

**Plans**: 1 plan
Plans:

- [x] 37-01-PLAN.md -- Remove trackInfoURL override from ProtocolHandler.pm

### Phase 38: Importer Foundation

**Goal**: Importer.pm registered with LMS as online library provider, scanner infrastructure ready for data import phases
**Depends on**: Phase 37
**Requirements**: LIB-06, TOK-01, TOK-02, CFG-01
**Success Criteria** (what must be TRUE):

  1. User can enable/disable library import via "Import Library" preference in SpotOn server settings
  2. LMS recognizes SpotOn as an online library provider (Importer registered via install.xml, addImporter in initPlugin)
  3. Scanner process authenticates with Spotify API using Own-ID token (Keymaster), falling back to bundled token on 403
  4. Scanner throttles API requests to 1 req/3s with proper 429 retry and cross-process rate-limit signaling via cache key

**Plans**: TBD

### Phase 39: Album + Artist Import

**Goal**: User's saved albums and followed artists appear in LMS native library with visual distinction
**Depends on**: Phase 38
**Requirements**: LIB-02, LIB-03, LIB-07, LIB-09, LIB-10
**Success Criteria** (what must be TRUE):

  1. After library scan, user's saved Spotify albums appear in LMS My Music > Albums with cover art and full track listing
  2. After library scan, user's followed Spotify artists appear in LMS My Music > Artists
  3. Spotify items in LMS library views show a SpotOn icon badge for visual distinction from local music
  4. User sees scan progress indicator during import showing item counts (Slim::Utils::Progress)
  5. Imported tracks have separate title/artist/album DB fields so LMS standardTitle() composes the client's titleFormat correctly (current_title compatibility with WiiM Ultra and similar devices, see GH #96)

**Plans**: TBD

### Phase 40: Liked Songs + Incremental Sync

**Goal**: User's liked songs are searchable in LMS and subsequent syncs complete in seconds via incremental update
**Depends on**: Phase 39
**Requirements**: LIB-01, LIB-04, LIB-05, LIB-08
**Success Criteria** (what must be TRUE):

  1. After library scan, user's liked Spotify songs appear in LMS My Music > Tracks and are findable via LMS global search
  2. Subsequent scans only fetch items added since last scan (added_at early-exit) -- a 2000-track library update with 50 new songs makes ~4 API calls instead of 40
  3. LMS hourly poll triggers needsUpdate() which detects library changes via 3 lightweight API calls (me/tracks?limit=1, me/albums?limit=1, me/playlists snapshot_ids)
  4. SpotOn Status Page shows library import statistics (number of tracks, albums, and artists imported)

**Plans**: TBD

### Phase 41: Playlist Import

**Goal**: User can opt-in to import Spotify playlists as LMS playlists with efficient change detection
**Depends on**: Phase 40
**Requirements**: PL-01, PL-02, CFG-02
**Success Criteria** (what must be TRUE):

  1. User can enable playlist import via "Import Playlists" preference (only visible/active when library import is enabled)
  2. After scan with playlist import enabled, user's Spotify playlists appear as LMS playlists with correct track listings
  3. Only playlists whose snapshot_id changed since last scan are reimported -- unchanged playlists are skipped entirely

**Plans**: TBD

## Progress Table

| Phase | Milestone | Plans | Status | Completed |
|-------|-----------|-------|--------|-----------|
| 1-6 (15 phases) | v1.0 | 50/50 | Complete | 2026-06-03 |
| 7-12 (7 phases) | v1.1 | 13/13 | Complete | 2026-06-06 |
| 13-16.1 (5 phases) | v1.3 | 9/9 | Complete | 2026-06-13 |
| 18-21 (4 phases) | v1.5 | 6/6 | Complete | 2026-06-15 |
| 22. Seek + Favorites Bugfixes | v2.0 | 1/1 | Complete | 2026-06-17 |
| 25. Play-All Full Pagination | v2.0 | 1/1 | Complete | 2026-06-18 |
| 26. Browse Error Recovery | v2.0 | 2/2 | Complete | 2026-06-21 |
| 27. Pipeline Failure Recovery | v2.0 | 1/1 | Complete | 2026-06-22 |
| 28. Persistent Browse Daemon | v2.0 | 3/3 | Complete | 2026-06-22 |
| 29. Unified Daemon | v2.0 | 3/3 | Complete | 2026-06-22 |
| 30. Legacy Pipe Cleanup | v2.0 | 2/2 | Complete | 2026-06-22 |
| 31. Code Review Hardening | v2.0 | 2/2 | Complete | 2026-06-24 |
| 32. Status Page | v2.0 | 2/2 | Complete | 2026-06-25 |
| 33. More Context Menu | v2.1 | 1/1 | Complete | 2026-06-26 |
| 34. Add to Playlist | v2.1 | 1/1 | Complete | 2026-06-26 |
| 35. Liked Songs Play-All Throttle | v2.1.2 | 1/1 | Complete | 2026-06-26 |
| 36. Session Health Monitoring | v2.2 | 2/2 | Complete | 2026-06-30 |
| 37. Context Menu LMS Items | v2.3 | 1/1 | Complete    | 2026-06-30 |
| 38. Importer Foundation | v2.3 | 0/? | Not started | - |
| 39. Album + Artist Import | v2.3 | 0/? | Not started | - |
| 40. Liked Songs + Incremental Sync | v2.3 | 0/? | Not started | - |
| 41. Playlist Import | v2.3 | 0/? | Not started | - |
| 42. OGG Vorbis Passthrough | v2.3 | 2/2 | Complete   | 2026-07-01 |
| 43. Connect OGG Passthrough | v2.3 | 1/1 | Complete   | 2026-07-02 |
| 46. Code Review Bugfixes | v2.3 | 1/1 | Complete   | 2026-07-02 |

### ~~Phase 48: login5 Token Retrieval (Bridge)~~ — SUPERSEDED

> Superseded 2026-07-04 by v3.0 Auth Overhaul. Spike session proved login5 rate pool is 429-revoked and PKCE + sp_dc/Pathfinder covers all use cases. Planning artifacts archived in `48-SUPERSEDED-login5-token-retrieval/`.

</details>

### v3.0 Auth Overhaul

> **Decision 2026-07-04:** Complete auth rewrite based on Spike session (7 spikes, all VALIDATED). Three token paths replace the current Keymaster architecture. Spike findings: `.claude/skills/spike-findings-spoton/`. Research: https://gist.github.com/stiefenm/1f8c1231462ec6c41e29832e758f338d
>
> **Revision 2026-07-04 (post urknall forum review #153-158):** ZeroConf stays as guest-discovery feature (not removed). PKCE-first for all auth (not ZeroConf-first). Discovery ON by default. Audit phase added. sp_dc hardened for TOTP rotation. Multi-pool token routing in Client.pm. urknall's 11 success criteria adopted as UAT gates.

**Branch:** `v3.0-auth` (from main)

**Key architectural decisions (post forum review):**
- **PKCE-first (Option A):** PKCE OAuth is the single auth mechanism. One browser click in Settings → Web API tokens + automatic credential derivation. Clean break from Keymaster.
- **ZeroConf stays as feature:** mDNS discovery remains active for guest/party access (household members casting to speaker). It is no longer an auth mechanism — just a Connect discovery feature.
- **Discovery ON by default:** `--disable-discovery` is a per-player option for Docker/owner-only setups, not the default. Existing users' speakers stay visible.
- **Login5 fallback declined:** Login5 tokens get immediate 429 on api.spotify.com (quota revocation, per Herger). librespot handles Login5 internally for spclient/session — no separate SpotOn fallback phase needed.
- **Desktop Client ID OAuth declined:** ToS risk, unnecessary given grandfathered Extended Quota. Documented as conscious decision, not oversight.
- **LMS-community PKCE flow:** Future backlog, not v3.0 scope.
- **Callback URI via GitHub Pages:** Static page at `https://stiefenm.github.io/spoton/auth/` serves as PKCE redirect URI. LMS encodes its local callback URL + nonce in the OAuth `state` parameter. The static page decodes `state`, validates the target is a private IP (RFC1918/loopback/.local), and redirects the browser to `http://<lms-host>:9000/plugins/SpotOn/callback?code=...`. Fallback: page shows the code for manual copy-paste if the redirect fails. No server-side infrastructure needed. Pattern proven by SpotMyBackup (github.io) and similar to Spotty's `api.lms-community.org` relay but fully static.

**Phases (to be broken down via /gsd-plan-phase):**

| Phase | Name | Scope | Spike Basis |
|-------|------|-------|-------------|
| 49-00 | Token Usage Audit + Backend Evaluation | **Part A — Token Audit:** Map every `--get-token` / Keymaster call in Perl + Rust. Classify: Keymaster service call vs. client-ID-as-identity-hint. Answer urknall's 8 audit questions. **Part B — go-librespot Evaluation:** Assess whether go-librespot can replace the SpotOn Rust fork as streaming backend. Evaluate: native token management (Login5, PKCE, /token, /web-api/* proxy), REST API vs CLI+JSON-RPC, audio pipeline quality (OGG passthrough, Connect sinks, rate-limiting), cross-compile story (Go vs Rust). Decision: build token architecture in Perl+Rust-fork, or migrate to go-librespot. Output feeds all subsequent phases. | Pre-requisite |
| 49 | PKCE OAuth Flow | "Login with Spotify" button in Settings, GitHub Pages callback relay (`stiefenm.github.io/spoton/auth/`), `state` parameter carries LMS callback URL + nonce, static page validates private-IP target and redirects browser to LMS, copy-paste fallback if redirect fails, token exchange with stored PKCE verifier, scope management. Redirect URI: `https://stiefenm.github.io/spoton/auth/` (registered in Spotify Developer App). | Spikes 001, 002 |
| 50 | Perl TokenManager Rewrite | OAuth refresh in Perl (no binary spawn), atomic refresh-token persistence, proactive refresh timer. Multi-pool token routing through `API/Client.pm` (pkce \| webplayer), callers never touch tokens directly. go-librespot reference: `login5.go` (AccessToken renewal), `spclient.go` (centralized request wrapper with forced-refresh-on-401) | Spike 003 |
| 51 | Credential Derivation + Connect | `--token-login` integration, credential lifecycle, discovery ON by default with `--disable-discovery` as per-player option, Connect via Spirc cloud. ZeroConf remains active for guest discovery. | Spikes 004, 005 |
| 52 | sp_dc + Pathfinder Integration (best-effort) | sp_dc cookie Settings UI, TOTP secret self-scrape from web-player bundle with re-scrape on token failure, graceful degradation ("Made for You temporarily unavailable" — never breaks Browse/Connect), status page health indicator. Feature documented as "best effort" due to TOTP secret rotation risk. | Spikes 006, 007 |
| 53 | Keymaster Removal + Migration | Remove Keymaster TokenProvider service calls (`hm://keymaster/token/authenticated`) from normal operation. ZeroConf discovery code stays. Migration path for existing users (one-time PKCE setup prompt). Settings flow update. | All spikes |

**Architecture (proven by spikes):**
```
Feature              Token Type          Source                   User Action
───────────────────  ──────────────────  ───────────────────────  ──────────────────────
Browse/Library/      PKCE OAuth          Perl HTTP Refresh        One-time browser login
Search/Player                                                     in LMS Settings

Connect Mode         Stored Credentials  spoton --token-login     Automatic (from PKCE)

Guest Discovery      ZeroConf/mDNS       librespot built-in       None (on by default)

Made for You         Web-Player Token    sp_dc → TOTP → /api/     sp_dc cookie once in
(Daily Mix, DW,      (best-effort)       token (server-side)      LMS Settings (~1yr valid)
Release Radar)
```

**v3.0 Success Criteria (adapted from urknall's 11-point list, Forum #155):**

1. Can SpotOn generate valid credentials.json without mDNS? (via PKCE + --token-login)
2. Can existing ZeroConf-derived credentials continue to work for Connect?
3. Can LMS restart and play without renewed auth?
4. Does Spotify Connect work bidirectionally?
5. Do Browse/Search/Library work with PKCE tokens?
6. **Does `hm://keymaster/token/authenticated` still appear during normal operation? (Must be NO)**
7. Which features truly require official Web API OAuth scopes? (All Browse/Library/Player — answered by audit)
8. Can Library Importer complete a full 2000-track scan on PKCE pool without 429?
9. Where is Keymaster client ID only an identity/platform detail, not a service call? (Answered by audit, those stay)

**Reference implementations:** go-librespot `login5/login5.go`, `spclient/spclient.go`, `daemon/api_server.go` — token ownership and centralized request wrapper patterns.

**Requirements:** AUTH-01 through AUTH-07 (see REQUIREMENTS.md), plus new requirements for sp_dc/Pathfinder TBD

## Backlog

Items discovered during development — not assigned to a milestone.

1. ~~**Eigene SpotOn Client-ID bei Spotify registrieren**~~ — Resolved: existing Extended Quota grandfathered, PKCE with SpotOn ID is the Golden Path.
2. **~~Online-Musiksammlung (Importer.pm / OnlineLibraryBase)~~** — ~~Evaluiert und bewusst abgelehnt.~~ Now v2.3 scope (Phases 38-41).
3. ~~**LMS Community Repo Submission**~~ — Erledigt: Plugin im Community Repo veröffentlicht.
4. ~~**ZeroConf Auth UX: "Connected" an Spotify App melden**~~ — Verworfen: Setup Guide erklärt das Verhalten, kein technischer Fix möglich ohne Playback-Session.
5. ~~**Diagnostics: "Clear Logs" Button in Settings**~~ — Implementiert in v1.7.4 (truncate on daemon restart + Clear Logs button).
6. **Spotty Favorites Migration** — Settings-Button der `spotify://` Einträge in LMS Favorites und Playlists als `spoton://` Duplikate anlegt. Originale bleiben erhalten, User kann Spotty danach deinstallieren. URI-Schema nach Prefix ist identisch (`track:ID`, `album:ID`, etc.). Idee von Paul Webster (Forum #32, 2026-06-19).
7. **Search Pagination** — Suchergebnisse über offset-Pagination nachladen (aktuell max 50 pro Typ, Spotify hat oft 100+). LMS OPMLBased-Pagination nutzen.
8. **Forum Monitor + Draft Generation** — GitHub Action (cron) die den Lyrion-Forum-Thread pollt, neue Posts erkennt, via Claude API Draft-Replies generiert und als GitHub Issues zur Review erstellt. (Ehemals Phase 23, bereits als Self-Hosted Runner implementiert.)
9. **Forum Auto-Post** — Label-getriggerter GitHub Action Workflow der approved Draft-Replies automatisch im vBulletin-Forum postet. (Ehemals Phase 24.)
10. **OGG Passthrough + Normalization** — CJS (Forum #152). Aktuell schaltet SpotOn bei Normalization auf PCM um, weil librespot bei Passthrough nicht normalisieren kann. Lösungsansatz: librespot liest den Spotify Gain-Wert (proprietäres Format, 16 bytes LE floats at OGG offset 144, `NormalisationData::parse_from_ogg()` in player.rs) und meldet ihn via JSON-RPC an SpotOn. SpotOn setzt `Song->replayGain` / `remoteMeta->{replay_gain}` — LMS sendet den Wert im Slim-Protokoll an squeezelite, das ihn nach eigenem OGG-Decode anwendet. Kein LMS-Patch nötig. Aufwand: ~20 Zeilen Rust + ~10 Zeilen Perl.

### Phase 42: OGG Vorbis Passthrough

**Goal**: Wire up prepared OGG passthrough infrastructure — Rust sinks forward AudioPacket::OggData to capable players (squeezelite), PCM fallback for hardware players. Fixes misleading "OGG" display and enables CPU-saving codec offload. (#96)
**Depends on**: None (independent of Phases 38-41)
**Requirements**: OGG-01, OGG-02, OGG-03
**Success Criteria** (what must be TRUE):

  1. squeezelite player receives raw Ogg/Vorbis data from SpotOn daemon (verified via Content-Type and player codec log)
  2. Hardware Squeezebox players continue to receive PCM (no regression)
  3. NowPlaying correctly shows "OGG" only when OGG is actually streamed, "PCM" otherwise
  4. Sync groups with mixed player types work correctly (all players get format they support)

**Plans**: 2/2 plans complete
Plans:

- [x] 42-01-PLAN.md -- Rust sink passthrough and CLI wiring (OGG-01)
- [x] 42-02-PLAN.md -- Perl capability resolution, daemon wiring, NowPlaying fix, and tests (OGG-02, OGG-03)

### Phase 43: Connect OGG Passthrough

**Goal**: Fix Connect mode audio with OGG passthrough — Connect-Sink currently discards AudioPacket::Raw, and /stream relay drains OGG headers on startup. Browse OGG passthrough works, Connect is silent since v2.2.4.
**Depends on**: Phase 42
**Requirements**: OGG-04
**Success Criteria** (what must be TRUE):

  1. Connect mode plays audio with OGG passthrough enabled (squeezelite receives Ogg/Vorbis via /stream)
  2. Track changes in Connect mode produce continuous audio (chained OGG with valid headers per track)
  3. Browse OGG passthrough continues to work (no regression)
  4. Connect PCM fallback works when player doesn't support OGG

**Plans**: 1/1 plans complete
Plans:

- [x] 43-01-PLAN.md — Rust fixes (connect.rs AudioPacket::Raw + unified.rs OGG header buffering) + build verification

### Phase 44: Connect OGG Rate-Limiting

**Goal:** Fix Spirc/audio desync by adding granule_position-based wall-clock rate-limiting to Connect passthrough sink, matching CON-14 PCM pacing
**Requirements**: OGG-05
**Depends on:** Phase 43
**Plans:** 1/1 plans complete

Plans:

- [x] 44-01-PLAN.md -- Granule_position-based CON-14 rate-limiting in both Connect sinks (OGG-05)

### Phase 45: URL Canonicalization

**Goal:** Ensure spoton:// remains the canonical track URL for LMS metadata lookups when canDirectStream() returns daemon HTTP URLs. Evaluate whether URL canonicalization is needed in getMetadataFor(), getIcon(), trackInfoMenu(), parseDirectHeaders(). Reported by @urknall in #96.
**Requirements**: META-01
**Depends on:** Phase 44
**Canonical refs:** GH #96 comment 4857569938 (urknall's design analysis), GH #96 comment 4857885025 (urknall's concession re LMS URL separation)
**Status note:** Quick-fixes from GH #96 comment 4860032342 already applied: currentPlaylistUpdateTime before newmetadata, em dash replaced with dash. Remaining scope is URL canonicalization -- pending urknall's feedback on quick-fixes.
**Plans:** 0 plans

Plans:

- [ ] TBD (run /gsd-plan-phase 45 to break down)

### Phase 46: Code Review Bugfixes

**Goal:** Fix all High and Medium findings from the Fable full-project code review (2026-07-02), plus Windows compatibility fixes.
**Depends on:** None (independent bugfix phase)
**Canonical refs:** Fable review findings documented in session 2026-07-02
**Plans:** 1/1 plans complete

Plans:

- [x] 46-01-PLAN.md — All 30 review fixes grouped by module: API safety (H1-H3, M1-M3), Plugin core (H4, M5-M7, W2), Connect guards (H6, H7, M8, M11), ProtocolHandler (H5, M9), daemon lifecycle (H8-H10, M10, M12, W1), Rust daemon (H11-H13, M15, M16, M18, M19)

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-07-04 — Phase 46 completed (Code Review Bugfixes)*
