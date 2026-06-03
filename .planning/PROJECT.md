# SpotOn

## What This Is

A from-scratch Spotify plugin for Lyrion Music Server (LMS), built on lessons learned from the Spotty-NG project. It provides Spotify Browse, Search, Library access and Spotify Connect integration through librespot, with a clean architecture that avoids the historical debt of Herger's Spotty plugin.

## Core Value

Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## Current State

**v1.0 shipped** (2026-06-03) — 15 phases, 50 plans, 62 requirements, 6.502 LOC in 9 days.

Features:
- Browse (Home, Search, Library) via OPML menus
- Single-track streaming with 5 format modes (Auto/OGG/PCM/FLAC/MP3) per player
- Spotify Connect with bidirectional controls, sync groups, mDNS discovery
- ZeroConf + Dual-Token Auth (one-click setup via Spotify app)
- DSTM auto-play (Browse mode) with recommendations + search fallback
- Per-player settings (bitrate override, format dropdown, Connect toggle)
- 11-language i18n, Setup Guide, Credits, repo.xml distribution template

Known gaps (Backlog):
- Connect-DSTM (Auto-Play im Connect-Modus)
- Multi-arch binaries (only x86_64 present)
- Format-Dropdown mit B&O/Chromecast verifizieren
- Eigene SpotOn Client-ID bei Spotify registrieren

## Requirements

### v1.0 (archived)

All 62 v1 requirements complete. See `.planning/milestones/v1.0-REQUIREMENTS.md`.

### Out of Scope

- Lossless/HiFi streaming — blocked by PlayPlay DRM, architecturally prepared but not implementable (HIF-04)
- PlayPlay DRM reverse engineering — explicit prohibition, legal + ethical
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
| OAuth 2.0 PKCE + ZeroConf hybrid auth | Keymaster/login5 non-functional. ZeroConf for librespot credentials, Keymaster --get-token for Web API. | Phase 04.3 — Complete |
| HTTP-streaming for Connect audio | FIFO has architectural limitations (seek lag, white noise). HTTP gives clean connection semantics. | Phase 05 — Complete |
| Dual-Token API routing | Own Client-ID for me/* endpoints, bundled ID for browse/categories. Distributes rate-limit pressure. | Phase 04.4 — Complete |
| 5-option Format-Dropdown | Auto/OGG/PCM/FLAC/MP3 per player. Pipeline deletion forces format; snapshot-restore enables clean switching. | Phase 06 — Complete |
| OGG Passthrough as Auto default | Best format for capable players (no transcoding). Passthrough-Guard removes OGG when binary lacks support. | Phase 06 — Complete |
| DSTM via LMS framework (Browse only) | Connect-DSTM needs Spirc event hook — different architecture, deferred to backlog. | Phase 06 — Browse complete |

## Evolution

This document evolves at phase transitions and milestone boundaries.

---
*Created: 2026-05-26*
*Last updated: 2026-06-03 — Milestone v1.0 complete*
