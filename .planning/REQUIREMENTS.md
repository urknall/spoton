# Requirements: SpotOn v2.3

**Defined:** 2026-06-30
**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## v2.3 Requirements

Requirements for Library Integration milestone. Each maps to roadmap phases.

### Library Import

- [ ] **LIB-01**: User's Liked Songs erscheinen in LMS My Music > Tracks und sind über globale Suche findbar
- [ ] **LIB-02**: User's Saved Albums erscheinen in LMS My Music > Albums mit Cover und Tracklisting
- [ ] **LIB-03**: User's Followed Artists erscheinen in LMS My Music > Artists
- [ ] **LIB-04**: Inkrementeller Sync — nur neue/geänderte Items werden importiert (added_at Early-Exit)
- [ ] **LIB-05**: Change Detection via needsUpdate() — leichtgewichtige Prüfung (3 API-Calls) ob Rescan nötig
- [ ] **LIB-06**: Scanner nutzt SimpleSyncHTTP mit 1 req/3s Throttle und 429-Handling (sleep + retry)
- [ ] **LIB-07**: Library-Icon Badge für Spotify-Items in LMS Library View
- [ ] **LIB-08**: Library Stats auf Status Page (Tracks/Albums/Artists importiert)
- [ ] **LIB-09**: Progress-Anzeige während Scan (Slim::Utils::Progress)

### Playlist Import

- [ ] **PL-01**: Spotify-Playlists als LMS-Playlists importierbar (opt-in via Preference)
- [ ] **PL-02**: Playlist-Change-Detection via snapshot_id — nur geänderte Playlists werden reimportiert

### Token Routing

- [ ] **TOK-01**: Library-Import nutzt Own-ID-Token (Keymaster), Fallback auf Bundled bei 403
- [ ] **TOK-02**: Cross-Process Rate-Limit-Signal zwischen Scanner und Hauptprozess via Cache-Key

### Context Menu

- [x] **CTX-01**: Standard-LMS-Menüeinträge (Add to Favorites, etc.) erscheinen im SpotOn More-Menü neben den SpotOn-Einträgen

### Configuration

- [ ] **CFG-01**: Preference "Import Library" (global toggle, default: off)
- [ ] **CFG-02**: Preference "Import Playlists" (global toggle, default: off, nur aktiv wenn LIB enabled)

## Future Requirements

Deferred to future release. Tracked but not in current roadmap.

### Library Enhancements

- **LIB-F01**: Tag Cleanup — Strip "(Remastered)", "(Deluxe Edition)" from imported album/track names (opt-in)
- **LIB-F02**: Multi-Account Support — Import from multiple Spotify accounts into separate LMS library namespaces

## Out of Scope

| Feature | Reason |
|---------|--------|
| Full Catalog Search via Importer | Dev Mode limit=10 macht Search-basierte Anreicherung unwirtschaftlich |
| PKCE OAuth für Library Import | Würde zusätzliche Setup-Schritte erfordern, ZeroConf-Simplizität hat Priorität |
| Background Polling kürzer als 1h | LMS OnlineLibrary pollt stündlich, kürzere Intervalle nicht vorgesehen |
| Extended Quota | Spotify requires 250k MAU + legally registered business |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| CTX-01 | Phase 37 | Complete |
| LIB-06 | Phase 38 | Pending |
| TOK-01 | Phase 38 | Pending |
| TOK-02 | Phase 38 | Pending |
| CFG-01 | Phase 38 | Pending |
| LIB-02 | Phase 39 | Pending |
| LIB-03 | Phase 39 | Pending |
| LIB-07 | Phase 39 | Pending |
| LIB-09 | Phase 39 | Pending |
| LIB-01 | Phase 40 | Pending |
| LIB-04 | Phase 40 | Pending |
| LIB-05 | Phase 40 | Pending |
| LIB-08 | Phase 40 | Pending |
| PL-01 | Phase 41 | Pending |
| PL-02 | Phase 41 | Pending |
| CFG-02 | Phase 41 | Pending |

**Coverage:**
- v2.3 requirements: 16 total
- Mapped to phases: 16
- Unmapped: 0

---
*Requirements defined: 2026-06-30*
*Last updated: 2026-06-30 — traceability updated with phase mappings*
