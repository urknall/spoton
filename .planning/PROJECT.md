# SpotOn

## What This Is

A from-scratch Spotify plugin for Lyrion Music Server (LMS), built on lessons learned from the Spotty-NG project. It provides Spotify Browse, Search, Library access and Spotify Connect integration through librespot, with a clean architecture that avoids the historical debt of Herger's Spotty plugin.

## Core Value

Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## Current State

**v1.3 shipped** (2026-06-13) — 27 phases total (v1.0 + v1.1 + v1.3), 72 plans, 5.796 LOC Perl + 3.887 LOC tests, 268 tests green.

Features:
- Browse (Home, Search, Library) via OPML menus
- Single-track streaming with 5 format modes (Auto/OGG/PCM/FLAC/MP3) per player
- Spotify Connect with bidirectional controls, sync groups, mDNS discovery
- ZeroConf + Dual-Token Auth (one-click setup via Spotify app)
- DSTM auto-play in Browse mode AND Connect mode (Spirc-native autoplay)
- Per-player settings (bitrate override, format dropdown, Connect toggle, Autoplay toggle)
- Stream metadata in Songinfo (mode, format, bitrate)
- Track history with artwork, async re-fetch, Connect-to-Browse URL translation
- Multi-arch binaries for 6 Linux targets + Windows (auto-selected by Helper.pm)
- spoton:// URI scheme for Spotty coexistence
- 11-language i18n, Setup Guide, Credits
- Production deployment on Pi with monitoring, public GitHub repo with repo.xml

## Requirements

### v1.0 (archived)

All 62 v1 requirements complete. See `.planning/milestones/v1.0-REQUIREMENTS.md`.

### v1.1 (archived)

24 of 27 requirements complete, 3 deferred. See `.planning/milestones/v1.1-REQUIREMENTS.md`.

### v1.3 (archived)

17 of 19 requirements complete, 2 dropped (Phase 17 removed). See `.planning/milestones/v1.3-REQUIREMENTS.md`.

### Active — v1.5 Podcasts

**Goal:** Spotify Podcasts in SpotOn — Shows browsen, Episoden abspielen, Podcast-Bibliothek verwalten

**Target features:**
- Gespeicherte Shows (Podcast-Bibliothek)
- Show-Details mit Episodenliste
- Episoden-Wiedergabe über bestehenden ProtocolHandler
- Podcast-Suche (type=show,episode)
- Show zur Bibliothek hinzufügen/entfernen
- Per-Player Episode-Reihenfolge (reversePodcastOrder)
- Menüstruktur: eigener Top-Level-Punkt oder Untermenü in Bibliothek (zu klären)

### Out of Scope

- Lossless/HiFi streaming — blocked by PlayPlay DRM, architecturally prepared but not implementable (HIF-04)
- PlayPlay DRM reverse engineering — explicit prohibition, legal + ethical
- Mobile app — LMS plugin only
- Online-Musiksammlung (Importer.pm) — API-Quota im Dev Mode zu teuer, Browse > Library deckt Use Case ab
- Client-ID Code-Umstellung — erst nach Genehmigung durch Spotify
- Extended Quota — Spotify requires 250k MAU + legally registered business

## Context

**Prior art:** The Spotty-NG project (v1 through v3.1) provided deep experience with LMS plugin development, librespot integration, and Spotify API behavior. 20+ documented pitfalls (P-01 through P-46) inform the architecture.

**Herger's Spotty:** The established Spotify plugin for LMS. SpotOn is an independent alternative with its own namespace (`Slim::Plugin::SpotOn`), own preferences, own GUID. Coexistence verified — spoton:// URI scheme avoids protocol handler conflict.

**Test environment:** squeezelite (software player) as primary, plus 2x Bang & Olufsen devices via UPnPBridge (DLNA/UPnP players appearing as LMS players).

**Spotify ecosystem:** Shannon protocol (used by librespot) remains active due to certified Connect hardware (Sony, Bose etc.). Medium risk of future deprecation. Web API v1 stable but heavily restricted in Dev Mode (Feb 2026 changes).

## Constraints

- **Language**: Perl — LMS plugins are Perl modules under `Slim::Plugin::*`
- **Framework**: LMS Plugin API (`OPMLBased`, `SimpleAsyncHTTP`, `Cache`, `Prefs`)
- **Playback engine**: librespot — only open-source Spotify streaming implementation
- **Perl version**: >= 5.10 (LMS floor)
- **No external CPAN deps**: Everything with LMS bundled modules only
- **Spotify Premium**: Required for streaming
- **LMS version**: 8.0+ minimum, full features from 8.5.1
- **UI paradigm**: OPML menu trees — no grid layout, no tabs, no horizontal scrolling
- **Branding**: Pragmatic compliance with Spotify Design Guidelines

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| OAuth 2.0 PKCE + ZeroConf hybrid auth | Keymaster/login5 non-functional. ZeroConf for librespot credentials, Keymaster --get-token for Web API. | v1.0 Phase 04.3 — ✓ |
| HTTP-streaming for Connect audio | FIFO has architectural limitations (seek lag, white noise). HTTP gives clean connection semantics. | v1.0 Phase 05 — ✓ |
| Dual-Token API routing | Own Client-ID for me/* endpoints, bundled ID for browse/categories. Distributes rate-limit pressure. | v1.0 Phase 04.4 — ✓ |
| 5-option Format-Dropdown | Auto/OGG/PCM/FLAC/MP3 per player. Pipeline deletion forces format; snapshot-restore enables clean switching. | v1.0 Phase 06 — ✓ |
| Spirc-native autoplay for Connect-DSTM | librespot's built-in autoplay resolving avoids LMS-side queue management complexity. Per-player toggle via enableAutoplay pref. | v1.1 Phase 10 — ✓ |
| musl-static linking for all Linux binaries | No glibc dependency — runs on any Linux regardless of distro/version. cross-rs Docker for CI. | v1.1 Phase 08 — ✓ |
| spoton:// URI scheme | Enables coexistence with Spotty (spotify:// handler). Herger's request via Spotty-Plugin#224. | v1.1 Phase 12 — ✓ |
| 7-day metadata cache TTL | Track history needs artwork/format after playback. Async re-fetch on cache miss for expired entries. | v1.1 Phase 11 — ✓ |

## Evolution

This document evolves at phase transitions and milestone boundaries.

---
*Created: 2026-05-26*
*Last updated: 2026-06-14 — Milestone v1.5 started*
