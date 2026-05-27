# SpotOn

## What This Is

A from-scratch Spotify plugin for Lyrion Music Server (LMS), built on lessons learned from the Spotty-NG project. It provides Spotify Browse, Search, Library access and Spotify Connect integration through librespot, with a clean architecture that avoids the historical debt of Herger's Spotty plugin.

## Core Value

Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## Requirements

### Validated

- [x] OAuth 2.0 PKCE browser authentication via per-user Spotify Developer App — Validated in Phase 02.1

### Active

- [ ] Spotify navigation (Home, Search, Library) via OPML menu trees
- [ ] Audio streaming via librespot single-track mode with FLAC/PCM/MP3 transcoding
- [ ] Spotify Connect with per-player daemons and HTTP-streaming audio transport
- [ ] OAuth 2.0 PKCE authentication with guided Setup Wizard
- [ ] Central API throttle preventing 429 bursts
- [ ] OGG-Direct passthrough for capable players
- [ ] Sync-group handling for Connect (one daemon on master)
- [ ] Player-specific preferences (bitrate, normalization, connect on/off)

### Out of Scope

- Lossless/HiFi streaming — blocked by PlayPlay DRM, architecturally prepared but not implementable (HIF-04)
- PlayPlay DRM reverse engineering — explicit prohibition, legal + ethical
- Podcast support — defer to v2, not core to music playback use case
- Mobile app — LMS plugin only

## Context

**Prior art:** The Spotty-NG project (v1 through v3.1) provided deep experience with LMS plugin development, librespot integration, and Spotify API behavior. 20 documented pitfalls (P-01 through P-20) inform the architecture. Key lessons: never parallelize pagination (P-01), guard Connect daemons from LMS zombie-killer (P-03), never use `['time', N]` in stream-mode (P-13), rate-limit audio in the sink (P-16).

**Herger's Spotty:** The established Spotify plugin for LMS. SpotOn is an independent alternative with its own namespace (`Slim::Plugin::SpotOn`), own preferences, own GUID. Can coexist with Spotty but parallel operation is not a design goal.

**Test environment:** squeezelite (software player) as primary, plus 2x Bang & Olufsen devices via UPnPBridge (DLNA/UPnP players appearing as LMS players).

**Spotify ecosystem:** Shannon protocol (used by librespot) remains active due to certified Connect hardware (Sony, Bose etc.). Medium risk of future deprecation. Web API v1 stable. `recommendations` endpoint deprecated since 2024-11-27.

## Constraints

- **Language**: Perl — LMS plugins are Perl modules under `Slim::Plugin::*`
- **Framework**: LMS Plugin API (`OPMLBased`, `SimpleAsyncHTTP`, `Cache`, `Prefs`)
- **Playback engine**: librespot — only open-source Spotify streaming implementation
- **Perl version**: >= 5.10 (LMS floor)
- **No external CPAN deps**: Everything with LMS bundled modules only
- **Spotify Premium**: Required for streaming
- **LMS version**: 8.0+ minimum, full features from 8.5.1
- **UI paradigm**: OPML menu trees — no grid layout, no tabs, no horizontal scrolling
- **Branding**: Pragmatic compliance with Spotify Design Guidelines — correct metadata, attribution where possible, no over-compliance

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| OAuth 2.0 PKCE auth (replaced Keymaster) | Keymaster/login5 auth proved non-functional. PKCE browser flow via per-user Spotify Developer App provides reliable auth. Setup Wizard guides users through Client-ID entry. | Phase 02.1 — Complete |
| HTTP-streaming for Connect audio | FIFO has architectural limitations (seek lag, white noise on reconnect, no safe flush from Perl). HTTP gives clean connection semantics. FIFO as fallback until HTTP is ready. | — Pending |
| LMS 8.0+ floor | Only 1-2 test environments available. Higher floor = less legacy code. Most active installations are 8.0+. | — Pending |
| Sliding Window or Adaptive pipeline | Research phase to evaluate. Central throttle is MUST regardless. | — Pending |
| OGG-Direct as option | Saves CPU on weak devices. Player capability decides automatically. FLAC as default pipeline. | — Pending |
| Binary strategy deferred | Start with Perl plugin only. librespot binary sourcing (monorepo vs separate) decided when Connect phase begins. | — Pending |
| Separate namespace, coexistence possible | `Slim::Plugin::SpotOn` — own GUID, own prefs. Can exist alongside Spotty but not tested together. | — Pending |
| DSTM is SHOULD | Don't Stop The Music is nice-to-have, not critical for v1. | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-27 after Phase 02.1 completion — auth switched from Keymaster to OAuth PKCE*
