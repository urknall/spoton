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

- [ ] **Phase 37: Context Menu LMS Items** — Standard LMS actions (Add to Favorites, etc.) alongside SpotOn entries in More menu
- [ ] **Phase 38: Importer Foundation** — Importer.pm skeleton, install.xml registration, scanner throttle, token routing, Import Library preference
- [ ] **Phase 39: Album + Artist Import** — Saved albums and followed artists in LMS My Music with icon badge and progress indicator
- [ ] **Phase 40: Liked Songs + Incremental Sync** — Liked songs in LMS with global search, added_at early-exit, needsUpdate(), status page stats
- [ ] **Phase 41: Playlist Import** — Opt-in playlist import with snapshot_id change detection

## Phase Details

### Phase 37: Context Menu LMS Items
**Goal**: Standard LMS menu entries appear alongside SpotOn entries in the More menu (GH #55)
**Depends on**: Nothing (independent of library work)
**Requirements**: CTX-01
**Success Criteria** (what must be TRUE):
  1. User sees standard LMS actions (Add to Favorites, Add to Playlist, More Info) in SpotOn's More menu for tracks, albums, and artists
  2. Standard LMS actions execute correctly -- adding a SpotOn track to LMS Favorites actually creates a working favorite entry
**Plans**: TBD

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
**Requirements**: LIB-02, LIB-03, LIB-07, LIB-09
**Success Criteria** (what must be TRUE):
  1. After library scan, user's saved Spotify albums appear in LMS My Music > Albums with cover art and full track listing
  2. After library scan, user's followed Spotify artists appear in LMS My Music > Artists
  3. Spotify items in LMS library views show a SpotOn icon badge for visual distinction from local music
  4. User sees scan progress indicator during import showing item counts (Slim::Utils::Progress)
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
| 37. Context Menu LMS Items | v2.3 | 0/? | Not started | - |
| 38. Importer Foundation | v2.3 | 0/? | Not started | - |
| 39. Album + Artist Import | v2.3 | 0/? | Not started | - |
| 40. Liked Songs + Incremental Sync | v2.3 | 0/? | Not started | - |
| 41. Playlist Import | v2.3 | 0/? | Not started | - |

## Backlog

Items discovered during development — not assigned to a milestone.

1. **Eigene SpotOn Client-ID bei Spotify registrieren** — Blocked: Spotify requires 250k MAU + legally registered business. Extended Quota documentation deferred to future milestone.
2. **~~Online-Musiksammlung (Importer.pm / OnlineLibraryBase)~~** — ~~Evaluiert und bewusst abgelehnt.~~ Now v2.3 scope (Phases 38-41).
3. ~~**LMS Community Repo Submission**~~ — Erledigt: Plugin im Community Repo veröffentlicht.
4. ~~**ZeroConf Auth UX: "Connected" an Spotify App melden**~~ — Verworfen: Setup Guide erklärt das Verhalten, kein technischer Fix möglich ohne Playback-Session.
5. ~~**Diagnostics: "Clear Logs" Button in Settings**~~ — Implementiert in v1.7.4 (truncate on daemon restart + Clear Logs button).
6. **Spotty Favorites Migration** — Settings-Button der `spotify://` Einträge in LMS Favorites und Playlists als `spoton://` Duplikate anlegt. Originale bleiben erhalten, User kann Spotty danach deinstallieren. URI-Schema nach Prefix ist identisch (`track:ID`, `album:ID`, etc.). Idee von Paul Webster (Forum #32, 2026-06-19).
7. **Search Pagination** — Suchergebnisse über offset-Pagination nachladen (aktuell max 50 pro Typ, Spotify hat oft 100+). LMS OPMLBased-Pagination nutzen.
8. **Forum Monitor + Draft Generation** — GitHub Action (cron) die den Lyrion-Forum-Thread pollt, neue Posts erkennt, via Claude API Draft-Replies generiert und als GitHub Issues zur Review erstellt. (Ehemals Phase 23, bereits als Self-Hosted Runner implementiert.)
9. **Forum Auto-Post** — Label-getriggerter GitHub Action Workflow der approved Draft-Replies automatisch im vBulletin-Forum postet. (Ehemals Phase 24.)

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-06-30 — v2.3 Library Integration roadmap added (Phases 37-41)*
