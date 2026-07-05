---
gsd_state_version: 1.0
milestone: v2.3
milestone_name: Library Integration
status: ready_to_plan
stopped_at: Phase 46 complete, Phase 48 context gathered — ready to plan Phase 48
last_updated: 2026-07-04T10:00:00.000Z
last_activity: 2026-07-04 -- Phase 46 documented, forum triage partial
progress:
  total_phases: 5
  completed_phases: 0
  total_plans: 1
  completed_plans: 1
  percent: 0
---

# Project State: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Initialized:** 2026-05-26
**Mode:** yolo

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-30)

**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

**Current Focus:** v3.0 Auth Overhaul — spike session complete, milestone planning

## Current Position

Phase: 48 SUPERSEDED → v3.0 planned (Phases 49-53)
Plan: Spike findings packaged, phases defined, branch pending
Status: Ready to create v3.0-auth branch and plan Phase 49
Last activity: 2026-07-04

## Progress Bar

```
v2.3 Library Integration: [░░░░░░░░░░░░░░░░░░░░] 0/5 phases (37-41)
Phase 37: [x] Context Menu LMS Items (CTX-01)
Phase 38: [ ] Importer Foundation (LIB-06, TOK-01, TOK-02, CFG-01)
Phase 39: [ ] Album + Artist Import (LIB-02, LIB-03, LIB-07, LIB-09)
Phase 40: [ ] Liked Songs + Incremental Sync (LIB-01, LIB-04, LIB-05, LIB-08)
Phase 41: [ ] Playlist Import (PL-01, PL-02, CFG-02)

Side phases (independent):
Phase 42: [x] OGG Vorbis Passthrough (OGG-01..03)
Phase 43: [x] Connect OGG Passthrough (OGG-04)
Phase 44: [x] Connect OGG Rate-Limiting (OGG-05)
Phase 46: [x] Code Review Bugfixes (30 findings)
Phase 48: [~] SUPERSEDED by v3.0 Auth Overhaul (2026-07-04)

v3.0 Auth Overhaul (Phases 49-53, branch: v3.0-auth):
Phase 49: [ ] PKCE OAuth Flow (AUTH-01, AUTH-02)
Phase 50: [ ] Perl TokenManager Rewrite (AUTH-03)
Phase 51: [ ] Credential Derivation + Connect (AUTH-04, AUTH-05)
Phase 52: [ ] sp_dc + Pathfinder Integration (Made for You)
Phase 53: [ ] Keymaster Removal + Migration (AUTH-06, AUTH-07)
```

## Performance Metrics

**Historical velocity (reference):**

- v2.0: 9 phases, 16 plans in 9 days (~1.8 plans/day)
- v2.1: 2 phases, 2 plans in 1 day
- v1.5: 4 phases, 6 plans in 2 days (~3 plans/day)
- v1.0: 15 phases, 50 plans in 9 days (~5-6 plans/day)

## Deferred Items

Items carried forward from previous milestones:

| Category | Item | Status |
|----------|------|--------|
| debug | connect-reconnect-no-audio | awaiting_human_verify |
| uat | Phase 16 macOS Binary (3 scenarios) | deferred (no macOS test env) |

## Accumulated Context

### Decisions

- [v2.3]: Importer follows OnlineLibraryBase pattern (Spotty, Qobuz, TIDAL, Deezer)
- [v2.3]: me/tracks returns full objects — no individual entity fetches needed
- [v2.3]: Incremental sync via added_at early-exit (Spotty doesn't have this)
- [v2.3]: Scanner uses SimpleSyncHTTP (blocking OK in scanner process)
- [v2.3]: Token routing: Own ID via Keymaster, fallback to bundled on 403
- [v2.3]: PKCE-only is the Golden Path — replaces ZeroConf/Keymaster as single auth mechanism
- [v2.3]: ~~Phase 48 is a bridge (login5 fallback), not the target architecture~~ SUPERSEDED → v3.0
- [v3.0]: Auth Overhaul — PKCE + sp_dc/Pathfinder. 7 spikes validated 2026-07-04. Phase 48 archived.
- [v3.0]: PKCE-first (Option A) — sauberer Schnitt, PKCE ist der einzige Auth-Mechanismus. ZeroConf bleibt als Guest-Discovery, nicht als Auth.
- [v3.0]: ZeroConf bleibt als Feature (mDNS Guest-Discovery), wird aber nicht mehr als Auth-Mechanismus verwendet.
- [v3.0]: Discovery ON by default — --disable-discovery ist per-Player Option, nicht Default.
- [v3.0]: Callback URI via GitHub Pages static relay (stiefenm.github.io/spoton/auth/), state-Parameter mit LMS-Callback-URL + Nonce.
- [v3.0]: Login5-Fallback-Phase bewusst abgelehnt (Login5 bekommt sofort 429 auf api.spotify.com).
- [v3.0]: Desktop Client ID OAuth bewusst abgelehnt (ToS-Risiko, unnötig mit Extended Quota).
- [v3.0]: Audit-Phase (49-00) vor Implementation — Keymaster-Service vs. Client-ID-as-Identity klassifizieren.
- [v3.0]: sp_dc/Pathfinder als "best effort" — TOTP-Rotation, Graceful Degradation, Re-Scrape on Failure.
- [v3.0]: urknalls 11 Success Criteria als UAT-Gates übernommen (zentral: kein hm://keymaster/token/authenticated in normalen Logs).
- [v2.3]: Keymaster is dying — 403s widespread since Aug 2025
- [v2.3]: login5 rejects Developer App IDs — Dual-Token architecture is dead

### Blockers/Concerns

- Forum #143 (plasticator2): Audio distortion on Pi 4B + HiFiBerry Digi2 Pro SPDIF — log analysis pending
- ~~PR #104 (urknall): Release year metadata — review pending~~ MERGED, shipped in v2.3.8
- Forum #159 (Chezza): New Spotify account (Oct 2025) → NoStoredCredentials, urknall + CJS helping
- Forum #160 (CJS): "Default Adjustment for Remote Streams" stacks with SpotOn ReplayGain (Spotty doesn't) — potential v2.3.x bug

## Session Continuity

**Last session:** 2026-07-04
**Stopped at:** Forum-Analyse mit Fable 5 abgeschlossen, v3.0 ROADMAP überarbeitet (urknall-Feedback eingearbeitet), Callback-URI-Architektur entschieden (GitHub Pages)
**Next action:** Forum-Reply an urknall vorbereiten (Spike-Ergebnis-Response + was wir übernommen/abgelehnt haben), dann `/gsd-plan-phase 49-00` (Token Usage Audit)

---
*State initialized: 2026-05-26*
*Last updated: 2026-07-04 — v3.0 plan revised per urknall forum review, callback URI architecture decided*
*Last updated: 2026-07-04 — Phase 46 closed, forum triage partial*
