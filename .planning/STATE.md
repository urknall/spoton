---
gsd_state_version: 1.0
milestone: v2.0
milestone_name: Browse Daemon Migration
status: milestone_complete
stopped_at: Phase 33 complete — More Context Menu shipped, pending local UAT
last_updated: "2026-06-26T09:15:00Z"
progress:
  total_phases: 13
  completed_phases: 9
  total_plans: 21
  completed_plans: 17
  percent: 65
---

# Project State: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Initialized:** 2026-05-26
**Mode:** yolo

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-25)

**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

**Current Focus:** Phase 34 complete — Add to Playlist shipped

## Current Position

Phase: 34 (add-to-playlist) — COMPLETE
Plan: 1 of 1 — done
All milestones shipped (v1.0, v1.1, v1.3, v1.5, v2.0).
Current version: v2.0.9
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

Standalone:
Phase 33: [x] More Context Menu (GH #29)
Phase 34: [x] Add to Playlist
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

## Session Continuity

**Last session:** 2026-06-26
**Stopped at:** Phase 34 complete — Add to Playlist shipped as v2.1.1
**Next action:** Backlog priorisieren (Spotty Favorites Migration #6, Search Pagination #7) oder Issue #50 untersuchen.

---
*State initialized: 2026-05-26*
*Last updated: 2026-06-26 — Phase 33 complete*
