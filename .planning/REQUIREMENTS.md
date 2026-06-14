# Requirements: SpotOn v1.5 Podcasts

**Defined:** 2026-06-14
**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## v1.5 Requirements

Requirements for Podcast support. Each maps to roadmap phases.

### Browse (POD)

- [ ] **POD-01**: User kann gespeicherte Podcast-Shows unter "Podcasts > Meine Podcasts" browsen
- [ ] **POD-02**: User kann eine Show öffnen und deren Episodenliste sehen
- [ ] **POD-03**: User kann eine Episode direkt aus der Episodenliste abspielen
- [ ] **POD-04**: User kann eine Show zur Bibliothek hinzufügen (Folgen)
- [ ] **POD-05**: User kann eine Show aus der Bibliothek entfernen (Entfolgen)

### Suche (SRC)

- [ ] **SRC-01**: User kann unter "Podcasts > Podcast-Suche" nach Shows suchen
- [ ] **SRC-02**: User kann unter "Podcasts > Podcast-Suche" nach Episoden suchen
- [ ] **SRC-03**: Suchergebnisse zeigen Shows und Episoden als getrennte Kategorien

### Menüstruktur (NAV)

- [ ] **NAV-01**: "Podcasts" erscheint als eigener Top-Level-Menüpunkt neben Home, Suche, Bibliothek
- [ ] **NAV-02**: "Meine Podcasts" zeigt gespeicherte Shows alphabetisch sortiert
- [ ] **NAV-03**: "Podcast-Suche" ist ein eigener Untermenüpunkt innerhalb Podcasts

### Settings & UX (UX)

- [ ] **UX-01**: Globales Setting für Episoden-Reihenfolge (neueste zuerst vs. chronologisch)
- [ ] **UX-02**: Episoden zeigen Resume-Status (ungehört/angefangen/fertig) visuell an
- [ ] **UX-03**: Explicit-Episoden werden markiert oder gefiltert

### Auth & API (API)

- [x] **API-01**: OAuth-Scope `user-read-playback-position` wird zum Auth-Flow hinzugefügt
- [x] **API-02**: API-Methoden für Shows und Episodes in Client.pm (getSavedShows, getShow, getShowEpisodes, getEpisode)

### i18n (I18N)

- [ ] **I18N-01**: Alle Podcast-UI-Strings in 11 Sprachen übersetzt

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
| POD-01 | Phase 19 | Pending |
| POD-02 | Phase 19 | Pending |
| POD-03 | Phase 19 | Pending |
| NAV-01 | Phase 19 | Pending |
| NAV-02 | Phase 19 | Pending |
| NAV-03 | Phase 19 | Pending |
| POD-04 | Phase 20 | Pending |
| POD-05 | Phase 20 | Pending |
| SRC-01 | Phase 20 | Pending |
| SRC-02 | Phase 20 | Pending |
| SRC-03 | Phase 20 | Pending |
| UX-01 | Phase 21 | Pending |
| UX-02 | Phase 21 | Pending |
| UX-03 | Phase 21 | Pending |
| I18N-01 | Phase 21 | Pending |

**Coverage:**
- v1.5 requirements: 17 total
- Mapped to phases: 17/17 ✓
- Unmapped: 0

---
*Requirements defined: 2026-06-14*
*Last updated: 2026-06-14 — traceability table populated after roadmap creation*
