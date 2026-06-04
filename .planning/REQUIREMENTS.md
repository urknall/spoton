# Requirements: SpotOn

**Defined:** 2026-06-03
**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## v1.1 Requirements

Requirements for Milestone v1.1: Hardening & Reach.

### Code Cleanup

- [ ] **CLEAN-01**: Alle deutschen Kommentare in Perl-Quellcode durch englische ersetzt
- [ ] **CLEAN-02**: Alle deutschen Log-Strings (DEBUGLOG, INFOLOG etc.) durch englische ersetzt
- [ ] **CLEAN-03**: Verifizierung: `grep -rn` auf deutsche Sonderzeichen in Code-Kommentaren liefert null Treffer (i18n strings.txt ausgenommen)

### Multi-Arch Binary Distribution

- [x] **ARCH-01**: x86_64 Linux Binary ist musl-static gelinkt (ersetzt aktuelles glibc-Binary)
- [x] **ARCH-02**: aarch64 Linux Binary (Pi 4/5, NAS) via cross-rs gebaut und in Bin/ abgelegt
- [x] **ARCH-03**: armv7 Linux Binary (Pi 2/3 32-bit) via cross-rs gebaut und in Bin/ abgelegt
- [x] **ARCH-04**: i386 Linux Binary via cross-rs gebaut und in Bin/ abgelegt
- [ ] **ARCH-05**: macOS x86_64 Binary via native CI Runner gebaut und in Bin/ abgelegt
- [ ] **ARCH-06**: macOS aarch64 Binary (Apple Silicon) via native CI Runner gebaut und in Bin/ abgelegt
- [x] **ARCH-07**: Windows x86_64 Binary via MinGW-w64 GNU gebaut und in Bin/ abgelegt
- [x] **ARCH-08**: Helper.pm erkennt alle 8 Plattformen und wählt korrektes Binary aus Bin/ Unterverzeichnis
- [ ] **ARCH-09**: Plugin startet und streamt erfolgreich auf aarch64 Linux (verifiziert auf realer Hardware oder CI)
- [x] **ARCH-10**: ARMv6 Linux Binary (Pi 1/Zero) via cross-rs gebaut und in Bin/ abgelegt

### Connect-DSTM

- [x] **DSTM-01**: Spike: `PlayerEvent::EndOfTrack` in librespot-spoton emittiert `spottyconnect endoftrack` Event an LMS
- [x] **DSTM-02**: Connect.pm empfängt `endoftrack` Event und startet Grace-Timer (3-5s kalibriert)
- [x] **DSTM-03**: API/Client.pm hat `addToQueue()` Methode für `POST /me/player/queue`
- [x] **DSTM-04**: Bei Queue-Ende im Connect-Modus wird nächster Track via Search-Fallback ermittelt und per addToQueue eingefügt
- [x] **DSTM-05**: Per-Player Autoplay-Toggle in Settings UI (aktiviert/deaktiviert Connect-DSTM)
- [x] **DSTM-06**: Browse-DSTM bleibt unverändert funktional

### Stream-Metadaten

- [ ] **META-01**: Songinfo zeigt "(Spotify Browse)" oder "(Spotify Connect)" je nach aktivem Modus
- [ ] **META-02**: Songinfo zeigt aktuelles Stream-Format (OGG, FLAC, MP3, PCM)
- [ ] **META-03**: Songinfo zeigt Bitrate wenn verfügbar (z.B. "320k, OGG Vorbis (Spotify Connect)")

### Deployment

- [ ] **DEPLOY-01**: GitHub Repo stiefenm/spoton ist öffentlich; interne Dateien (.planning/, .claude/, CLAUDE.md) via .gitignore ausgeschlossen
- [ ] **DEPLOY-02**: install.xml existiert mit korrekter Version, Creator und Modulpfaden
- [ ] **DEPLOY-03**: repo.xml hat echte SHA1 und Download-URL; LMS zeigt SpotOn in der Plugin-Liste wenn die URL als Custom Repo hinterlegt wird
- [ ] **DEPLOY-04**: SpotOn ist auf dem Pi (192.168.13.5) installiert und läuft über den LMS Plugin Manager
- [ ] **DEPLOY-05**: SpotOn-Monitor ist aktiv auf dem Pi (Cron-Job, tägliche Log-Rotation); alte Spotty-Artefakte aufgeräumt

## v1.2+ Requirements

Deferred to future release. Tracked but not in current roadmap.

### Binary Distribution

- **ARCH-F01**: Universal macOS Fat Binary (x86_64 + aarch64 kombiniert)

### Connect-DSTM

- **DSTM-F01**: LMS-side DSTM Fallback falls librespot-native Autoplay versagt

## Out of Scope

| Feature | Reason |
|---------|--------|
| Connect-DSTM ohne Spike | Architektur-Risiko zu hoch ohne EndOfTrack-Validierung |
| `recommendations` Endpoint | Entfernt seit Nov 2024, `_searchFallback` stattdessen |
| MSVC Windows Build | Nicht möglich von Linux CI, GNU-Target stattdessen |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| CLEAN-01 | Phase 7 | Pending |
| CLEAN-02 | Phase 7 | Pending |
| CLEAN-03 | Phase 7 | Pending |
| ARCH-01 | Phase 8 | Complete |
| ARCH-02 | Phase 8 | Complete |
| ARCH-03 | Phase 8 | Complete |
| ARCH-04 | Phase 8 | Complete |
| ARCH-05 | Phase 8 | Pending |
| ARCH-06 | Phase 8 | Pending |
| ARCH-07 | Phase 8 | Complete |
| ARCH-08 | Phase 8 | Complete |
| ARCH-09 | Phase 8 | Pending |
| ARCH-10 | Phase 8 | Complete |
| META-01 | Phase 9 | Pending |
| META-02 | Phase 9 | Pending |
| META-03 | Phase 9 | Pending |
| DSTM-01 | Phase 10 | Complete |
| DSTM-02 | Phase 10 | Complete |
| DSTM-03 | Phase 10 | Complete |
| DSTM-04 | Phase 10 | Complete |
| DSTM-05 | Phase 10 | Complete |
| DSTM-06 | Phase 10 | Complete |
| DEPLOY-01 | Phase 9.5 | Pending |
| DEPLOY-02 | Phase 9.5 | Pending |
| DEPLOY-03 | Phase 9.5 | Pending |
| DEPLOY-04 | Phase 9.5 | Pending |
| DEPLOY-05 | Phase 9.5 | Pending |

**Coverage:**
- v1.1 requirements: 27 total
- Mapped to phases: 27
- Unmapped: 0 ✓

---
*Requirements defined: 2026-06-03*
*Last updated: 2026-06-03 — traceability mapped to phases 7-10*
