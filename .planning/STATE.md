---
gsd_state_version: 1.0
milestone: v1.5
milestone_name: Podcasts
status: milestone_complete
stopped_at: Phase 29 context gathered
last_updated: "2026-06-24T14:22:41.121Z"
progress:
  total_phases: 10
  completed_phases: 7
  total_plans: 17
  completed_plans: 14
  percent: 70
---

# Project State: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Initialized:** 2026-05-26
**Mode:** yolo

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-14)

**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

**Current Focus:** Phase 30 — legacy-pipe-cleanup

## Current Position

Phase: 30 (legacy-pipe-cleanup) — EXECUTING
Plan: 2 of 2
All milestones shipped (v1.0, v1.1, v1.3, v1.5).
Phase 22 (seek-favorites-bugfixes) complete.
Phase 25 (play-all-full-pagination) complete.
Phases 23/24 (forum monitor) in progress — ad-hoc, no milestone.
Current version: v1.7.8

## Progress Bar

```
v1.5 Podcasts: [████████████████████] 4/4 phases (SHIPPED 2026-06-15)
Phase 18: [x] API Foundation
Phase 19: [x] Podcast Browse
Phase 20: [x] Library Actions
Phase 21: [x] UX Polish + i18n
```

## Performance Metrics

**v1.3 velocity:**

- 5 phases, 9 plans in 7 days (2026-06-07 → 2026-06-13)
- Avg ~1.3 plans/day (lower due to hardware testing, research, and UX iteration)

**v1.1 velocity:**

- 7 phases, 13 plans in 3 days (2026-06-03 → 2026-06-06)
- Avg ~4 plans/day

**v1.0 reference velocity:**

- 15 phases, 50 plans in 9 days
- Avg ~5-6 plans/day

## Deferred Items

Items acknowledged at v1.5 milestone close (2026-06-16):

| Category | Item | Status |
|----------|------|--------|
| debug | connect-reconnect-no-audio | awaiting_human_verify |
| debug | sync-group-audio | fixed |
| uat | Phase 16 macOS Binary (3 scenarios) | deferred (no macOS test env) |
| Phase 31 P02 | 113 | 1 tasks | 3 files |

## Accumulated Context

### Decisions

- [v1.3]: Like endpoint: `PUT /me/library` (unified, Feb 2026 API)
- [v1.3]: Extended Quota blocked — Spotify requires 250k MAU + org
- [v1.3]: macOS ad-hoc codesign sufficient for LMS plugin manager
- [v1.3]: Conditional Rust build via tag-diff change detection
- [v1.3]: Phase 17 removed — MozartBridge project preferred
- [v1.3]: Account switcher: text + link pattern (LMS OPML has no auto-redirect)
- [v1.5]: "Podcasts" is a top-level menu item (not nested under Bibliothek)
- [v1.5]: Episode order is a GLOBAL setting (not per-player like Spotty had it)
- [v1.5]: Batch show/episode endpoints removed in Dev Mode — individual fetch only
- [v1.5]: Search limited to max 10 results per type in Dev Mode
- [v1.5]: Resume point available via episode.resume_point (fully_played + resume_position_ms)
- [v1.5]: Follow/unfollow uses existing unified `PUT/DELETE /me/library` (same as Like button)
- [Phase ?]: Follow/Unfollow uses unified PUT/DELETE /me/library — same endpoint as Like/Unlike, no new Spotify API surface
- [Phase ?]: UX-01 dropped: episodeOrder setting not needed (Spotify API default newest-first is sufficient)
- [Phase ?]: P-CR-03: CSRF guard checks csrfProtectionLevel before enforcing — respects admin security settings

### Pending Todos

None.

### Blockers/Concerns

None for current milestone.

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

## Session Continuity

**Last session:** 2026-06-24T14:22:41.113Z
**Stopped at:** Phase 29 context gathered
**Next action:** Issue #20 Antwort abwarten. Upstream PRs monitoren (librespot#1722 CDN fix, #1724 IPv6, lms-material#1236). Backlog: Spotty Favorites Migration (#6).

---
*State initialized: 2026-05-26*
*Last updated: 2026-06-21 — Forum support @lmsc, Phase 25 complete, Issue #12 relabeled*
