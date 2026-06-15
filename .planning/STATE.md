---
gsd_state_version: 1.0
milestone: v1.5
milestone_name: Podcasts
status: ready_to_plan
stopped_at: Phase 19 complete (2/2) — ready to discuss Phase 20
last_updated: 2026-06-14T17:34:41.703Z
last_activity: 2026-06-14 -- Completed quick task 260614-t8m: Dark theme fix + discovery hint
progress:
  total_phases: 4
  completed_phases: 1
  total_plans: 3
  completed_plans: 12
  percent: 25
---

# Project State: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Initialized:** 2026-05-26
**Mode:** yolo

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-14)

**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

**Current Focus:** Phase 20 — podcast library actions

## Current Position

Phase: 20
Plan: Not started
Status: Ready to plan
Last activity: 2026-06-15 - Completed quick task 260615-jub: Comprehensive DIAG events (43 points across 5 modules)

## Progress Bar

```
v1.5 Podcasts: [                    ] 0/4 phases
Phase 18: [ ] API Foundation
Phase 19: [ ] Podcast Browse
Phase 20: [ ] Search + Library Actions
Phase 21: [ ] UX Polish + i18n
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

Items acknowledged and carried forward:

| Category | Item | Status |
|----------|------|--------|
| verification | Phase 08 (08-VERIFICATION.md) | human_needed |
| verification | Phase 11 (11-VERIFICATION.md) | human_needed |
| verification | Phase 12 (12-VERIFICATION.md) | human_needed |

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

## Session Continuity

**Last session:** 2026-06-15
**Stopped at:** Quick task 260615-ij7 completed (GitHub #2 + #3 support)
**Next action:** `/gsd:plan-phase 20`

---
*State initialized: 2026-05-26*
*Last updated: 2026-06-15 — Quick task: Settings visibility fix + diagnostic system*
