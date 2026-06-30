---
gsd_state_version: 1.0
milestone: v2.1
milestone_name: Context Menu
status: milestone_complete
stopped_at: "Issue triage — #91 PKCE vs Keymaster erklärt, #92 go-librespot beantwortet, ohne-Client-ID lokal verifiziert."
last_updated: "2026-06-30T06:26:55.637Z"
progress:
  total_phases: 1
  completed_phases: 0
  total_plans: 2
  completed_plans: 0
  percent: 0
---

# Project State: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Initialized:** 2026-05-26
**Mode:** yolo

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-25)

**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

**Current Focus:** Phase 36 — session-health-monitoring

## Current Position

Phase: 36 (session-health-monitoring) — EXECUTING
Plan: 1 of 2
All milestones shipped (v1.0, v1.1, v1.3, v1.5, v2.0, v2.1).
Current version: v2.1.8
No active milestone.

## Progress Bar

```
v2.0 Browse Daemon Migration: [████████████████████] 9/9 phases (SHIPPED 2026-06-25)
Phase 22: [x] Seek + Favorites Bugfixes
Phase 25: [x] Play-All Full Pagination
Phase 26: [x] Browse Error Recovery
Phase 27: [x] Pipeline Failure Recovery
Phase 28: [x] Persistent Browse Daemon
Phase 29: [x] Unified Daemon
Phase 30: [x] Legacy Pipe Cleanup
Phase 31: [x] Code Review Hardening
Phase 32: [x] Status Page

v2.1 Context Menu: [████████████████████] 2/2 phases (SHIPPED 2026-06-26)
Phase 33: [x] More Context Menu (GH #29)
Phase 34: [x] Add to Playlist

Standalone fixes:
Phase 35: [x] Liked Songs Play-All Throttle (GH #51) → v2.1.2
```

## Performance Metrics

**v2.0 velocity:**

- 9 phases, 16 plans in 9 days (2026-06-17 → 2026-06-25)
- Avg ~1.8 plans/day (architecture-heavy phases, Rust + Perl dual-stack)
- 197 commits, 25 files changed, +3020/-989 lines

**v1.5 velocity:**

- 4 phases, 6 plans in 2 days (2026-06-14 → 2026-06-15)
- Avg ~3 plans/day

**v1.3 velocity:**

- 5 phases, 9 plans in 7 days (2026-06-07 → 2026-06-13)
- Avg ~1.3 plans/day

**v1.1 velocity:**

- 7 phases, 13 plans in 3 days (2026-06-03 → 2026-06-06)
- Avg ~4 plans/day

**v1.0 reference velocity:**

- 15 phases, 50 plans in 9 days
- Avg ~5-6 plans/day

## Deferred Items

Items acknowledged at v2.0 milestone close (2026-06-25):

| Category | Item | Status |
|----------|------|--------|
| debug | connect-reconnect-no-audio | awaiting_human_verify |
| debug | sync-group-audio | fixed |
| uat | Phase 16 macOS Binary (3 scenarios) | deferred (no macOS test env) |
| verification | Phase 25, 26, 30 human_needed | functionally verified via UAT |

## Accumulated Context

### Decisions

- [v1.3]: Like endpoint: `PUT /me/library` (unified, Feb 2026 API)
- [v1.3]: Extended Quota blocked — Spotify requires 250k MAU + org
- [v1.3]: macOS ad-hoc codesign sufficient for LMS plugin manager
- [v1.3]: Conditional Rust build via tag-diff change detection
- [v1.3]: Phase 17 removed — MozartBridge project preferred
- [v1.5]: "Podcasts" is a top-level menu item (not nested under Bibliothek)
- [v1.5]: Episode order is a GLOBAL setting (not per-player)
- [v2.0]: Unified Daemon Architecture — one librespot process per player (Browse + Connect)
- [v2.0]: HTTP Track Streaming replaces FIFO pipe architecture
- [v2.0]: Status Page uses addPageFunction (standalone, not inside Settings framework)
- [v2.0]: Browse 404 retry: 3 attempts with 2s delay before skip (audio-key throttle resilience)
- [v2.0]: CSRF guard checks csrfProtectionLevel before enforcing
- [33]: trackInfoMenu returns arrayref (multi-item), after => 'top' positioning
- [33]: Cache-driven context menu: store IDs at cache-write, read at menu-build — no live API call
- [34]: Spotify playlists (not LMS) for Add to Playlist — LMS handles its own natively
- [34]: Query param for uris in addToPlaylist — works with existing _request infrastructure

### Blockers/Concerns

None.

### Quick Tasks Completed

| # | Description | Date | Commit | Directory |
|---|-------------|------|--------|-----------|
| 260614-t8m | Dark theme fix + discovery hint i18n | 2026-06-14 | 7445a85 | [260614-t8m-dark-theme-fix-and-discovery-hint](./quick/260614-t8m-dark-theme-fix-and-discovery-hint/) |
| 260615-ij7 | Settings-Sichtbarkeit + Diagnostik-System | 2026-06-15 | fd9a855 | [260615-ij7-settings-sichtbarkeit-diagnostik-system](./quick/260615-ij7-settings-sichtbarkeit-diagnostik-system/) |
| 260615-jub | Comprehensive DIAG events across all modules | 2026-06-15 | d834cc1 | [260615-jub-comprehensive-diag-events-across-all-mod](./quick/260615-jub-comprehensive-diag-events-across-all-mod/) |
| 260615-khq | Daemon credential check + orphan log cleanup | 2026-06-15 | 765b38c | [260615-khq-daemon-credential-check-orphan-log-clean](./quick/260615-khq-daemon-credential-check-orphan-log-clean/) |
| 260617-i9i | Forum bug hotfix: parseDirectHeaders, md5_hex, shutdownPlugin | 2026-06-17 | 4802712 | [260617-i9i-fix-three-forum-bugs-parsedirectheaders-](./quick/260617-i9i-fix-three-forum-bugs-parsedirectheaders-/) |
| 260618-gt7 | Connect daemon log truncate on restart + clear logs button | 2026-06-18 | 398a722 | [260618-gt7-connect-daemon-log-truncate-on-restart-c](./quick/260618-gt7-connect-daemon-log-truncate-on-restart-c/) |
| 260618-mi1 | IPv6 discovery fallback — vendor librespot-discovery, patch dual-stack bind | 2026-06-18 | 784556b | [260618-mi1-ipv6-discovery-fallback-vendor-librespot](./quick/260618-mi1-ipv6-discovery-fallback-vendor-librespot/) |
| 260618-zc1 | Fix ZeroConf discovery auth race condition + setup guide rewrite + AJAX discovery | 2026-06-18 | 9de1891..42aca77 | [260618-zc1-fix-zeroconf-discovery-auth-race-cond](./quick/260618-zc1-fix-zeroconf-discovery-auth-race-cond/) |
| 260626-caw | Status page: player names, sync groups, playback status, error recording | 2026-06-26 | 20c5381 | [260626-caw-status-page-player-names-sync-groups-pla](./quick/260626-caw-status-page-player-names-sync-groups-pla/) |
| 260629-gwy | Pause guard — re-apply pause swallowed by HTTP stream setup | 2026-06-29 | b83b27f | [260629-gwy-fix-pause-swallowed-during-http-stream-s](./quick/260629-gwy-fix-pause-swallowed-during-http-stream-s/) |

## Session Continuity

**Last session:** 2026-06-29
**Stopped at:** Issue triage — #91 PKCE vs Keymaster erklärt, #92 go-librespot beantwortet, ohne-Client-ID lokal verifiziert.
**Next action:**

- Issue #91 (woorszt): Custom Client ID 403 — PKCE vs Keymaster erklärt, v2.1.8 + ohne-ID Test angefragt
- Issue #92 (urknall): go-librespot Feature Request — abgelehnt (2.700 LOC Rust Neuentwicklung)
- Issue #85 (urknall): Tracks in Playlists — v2.1.8 fix kommentiert, warten auf Bestätigung
- Issue #60 (warminskimarcin): Playlist-Bug (Docker) — Cache-Clear empfohlen, warten auf Rückmeldung
- Issue #50 (JesseHoekema): RPi 3 — awaiting `--check` output (4 Tage)
- Issue #42 (jmhunter): SqueezeDSP conflict — waiting-upstream
- Issue #20 (lmsc): StatusCode(500) — waiting-user
- Issue #82 (Artist/Title Radio): enhancement — beantwortet, kein Handlungsbedarf
- Issue #74 (alex-aust): Connect Volume — beantwortet
- Issue #32 (Library Integration): +1 von akoirium — Backlog
- Upstream PRs: librespot #1724 (IPv6, offen), SqueezeDSP #19 (offen) — kein Handlungsbedarf
- Backlog: #82 Artist Radio, #55 Context Menu LMS Items, #32 Library Integration, #7 Search Pagination, #6 Spotty Migration

## Key Findings This Session

- **Spotty vs SpotOn Auth:** Herger's Spotty nutzt PKCE OAuth (accounts.spotify.com/api/token), SpotOn nutzt Keymaster (hm://keymaster/token/authenticated). Komplett verschiedene Auth-Flows — Spotty-Vergleich bei Client-ID-Problemen nicht aussagekräftig.
- **Ohne Custom Client ID:** Lokal verifiziert — SpotOn funktioniert vollständig ohne eigene Client ID. Alle Endpoints (Browse, Search, Library, Connect) nutzen sauber den Bundled Token.

---
*State initialized: 2026-05-26*
*Last updated: 2026-06-29 — Issue triage, PKCE vs Keymaster Analyse*
