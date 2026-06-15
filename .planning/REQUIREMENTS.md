# Requirements: SpotOn v1.5 Podcasts

**Defined:** 2026-06-14
**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## v1.5 Requirements

Requirements for Podcast support. Each maps to roadmap phases.

### Browse (POD)

- [x] **POD-01**: User kann gespeicherte Podcast-Shows unter "Podcasts > Meine Podcasts" browsen
- [x] **POD-02**: User kann eine Show öffnen und deren Episodenliste sehen
- [x] **POD-03**: User kann eine Episode direkt aus der Episodenliste abspielen
- [ ] **POD-04**: User kann eine Show zur Bibliothek hinzufügen (Folgen)
- [ ] **POD-05**: User kann eine Show aus der Bibliothek entfernen (Entfolgen)

### Suche (SRC)

- [x] **SRC-01**: User kann unter "Podcasts > Podcast-Suche" nach Shows suchen
- [x] **SRC-02**: User kann unter "Podcasts > Podcast-Suche" nach Episoden suchen
- [x] **SRC-03**: Suchergebnisse zeigen Shows und Episoden als getrennte Kategorien

### Menüstruktur (NAV)

- [x] **NAV-01**: "Podcasts" erscheint als eigener Top-Level-Menüpunkt neben Home, Suche, Bibliothek
- [x] **NAV-02**: "Meine Podcasts" zeigt gespeicherte Shows alphabetisch sortiert
- [x] **NAV-03**: "Podcast-Suche" ist ein eigener Untermenüpunkt innerhalb Podcasts

### Settings & UX (UX)

- [ ] **UX-01**: Globales Setting für Episoden-Reihenfolge (neueste zuerst vs. chronologisch)
- [x] **UX-02**: Episoden zeigen Resume-Status (ungehört/angefangen/fertig) visuell an
- [x] **UX-03**: Explicit-Episoden werden markiert oder gefiltert
- [x] **UX-04**: Episode-Info-View zeigt Show-Link und Follow-Action (lazy-load via GET /episodes/{id} für Suchergebnisse)
- [x] **UX-05**: Track- und Episode-Info-View zeigen Play/Queue-Buttons im Default-Skin

### Auth & API (API)

- [x] **API-01**: OAuth-Scope `user-read-playback-position` wird zum Auth-Flow hinzugefügt
- [x] **API-02**: API-Methoden für Shows und Episodes in Client.pm (getSavedShows, getShow, getShowEpisodes, getEpisode)

### i18n (I18N)

- [x] **I18N-01**: Alle Podcast-UI-Strings in 11 Sprachen übersetzt

## Future Requirements

Deferred to future milestone. Tracked but not in current roadmap.

### Podcast-Erweiterungen

- **POD-F01**: Podcast-Suche auch in globale Suche integrieren (zusätzlich zur eigenen Suche)
- **POD-F02**: "Kürzlich gehört" Sektion für Podcasts auf Home-Screen

## Out of Scope

| Feature | Reason |
|---------|--------|
| Podcast-Download/Offline | librespot streamt nur, kein Offline-Modus |
| Video-Podcasts | LMS hat keinen Video-Player |
| Podcast-Kapitelmarken | Spotify API liefert keine Kapitelmarken |
| Playlist-Integration | Podcasts und Musik sind getrennte Domains in Spotify |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| API-01 | Phase 18 | Complete |
| API-02 | Phase 18 | Complete |
| POD-01 | Phase 19 | Complete |
| POD-02 | Phase 19 | Complete |
| POD-03 | Phase 19 | Complete |
| NAV-01 | Phase 19 | Complete |
| NAV-02 | Phase 19 | Complete |
| NAV-03 | Phase 19 | Complete |
| POD-04 | Phase 20 | Pending |
| POD-05 | Phase 20 | Pending |
| SRC-01 | Phase 19 | Complete |
| SRC-02 | Phase 19 | Complete |
| SRC-03 | Phase 19 | Complete |
| UX-01 | Phase 21 | Pending |
| UX-02 | Phase 21 | Complete |
| UX-03 | Phase 21 | Complete |
| UX-04 | Phase 21 | Complete |
| UX-05 | Phase 21 | Complete |
| I18N-01 | Phase 21 | Complete |

**Coverage:**
- v1.5 requirements: 19 total
- Mapped to phases: 19/19 ✓
- Unmapped: 0

---
*Requirements defined: 2026-06-14*
*Last updated: 2026-06-14 — SRC-01/02/03 moved from Phase 20 to Phase 19 per D-13*
