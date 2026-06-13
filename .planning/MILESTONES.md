# Milestones

## v1.0 Foundation (Shipped: 2026-06-03)

**Phases completed:** 15 phases, 50 plans
**Archive:** [v1.0-ROADMAP.md](milestones/v1.0-ROADMAP.md) | [v1.0-REQUIREMENTS.md](milestones/v1.0-REQUIREMENTS.md)

**Key accomplishments:**

- Browse (Home, Search, Library) via OPML menus with Spotify Web API
- Single-track streaming with 5 format modes (Auto/OGG/PCM/FLAC/MP3) per player
- Spotify Connect with bidirectional controls, sync groups, mDNS discovery
- ZeroConf + Dual-Token Auth (one-click setup via Spotify app)
- DSTM auto-play (Browse mode) with recommendations + search fallback
- Per-player settings, 11-language i18n, Setup Guide, repo.xml distribution

---

## v1.1 Hardening & Reach (Shipped: 2026-06-06)

**Phases completed:** 7 phases, 13 plans, 16 tasks
**Archive:** [v1.1-ROADMAP.md](milestones/v1.1-ROADMAP.md) | [v1.1-REQUIREMENTS.md](milestones/v1.1-REQUIREMENTS.md)

**Key accomplishments:**

- DE→EN code cleanup — all German comments and log strings replaced with English
- Multi-arch binary distribution — 6 musl-static Linux + Windows PE32+ binaries via cross-rs, Helper.pm auto-selects correct binary
- Stream metadata in Songinfo — shows active mode (Browse/Connect), format, and bitrate
- Production deployment — GitHub repo public, LMS custom repo XML, Pi deployment with monitoring
- Connect-DSTM — Spirc-native autoplay when Spotify queue is exhausted, per-player toggle
- Track history metadata — 7-day TTL cache, Connect-to-Browse URL translation, async re-fetch
- Protocol handler rename — spoton:// URI scheme for Spotty coexistence (Spotty-Plugin#224)

**Known gaps at close:** 3 deferred requirements (ARCH-05/06 macOS binaries, ARCH-09 aarch64 hardware verification)
**Deferred items:** 6 (1 debug session fixed, 3 verification gaps human_needed, 2 todos)

---

## v1.3 Polish & Publish (Shipped: 2026-06-13)

**Phases completed:** 5 phases, 9 plans
**Timeline:** 7 days (2026-06-07 → 2026-06-13)
**Archive:** [v1.3-ROADMAP.md](milestones/v1.3-ROADMAP.md) | [v1.3-REQUIREMENTS.md](milestones/v1.3-REQUIREMENTS.md)

**Key accomplishments:**

- Connect credential isolation — per-player cache directory, multi-user households work without credential overwrite
- Like/Unlike from browse menus — Unified Library API integration, async liked-state check, 11-language i18n
- macOS Universal Binary — Intel + Apple Silicon via CI lipo, ad-hoc codesign, Gatekeeper hint in settings
- CI conditional build — tag-diff detects Rust changes, plugin-only releases skip compilation (8min → <1min)
- Account switcher UX — confirmation + navigation back to root, discovery instructions for add-account flow
- Code quality — TT2 BLOCK deduplication, JSON.parse hardening, bilingual test coverage for all strings

**Requirements:** 17/19 complete, 2 dropped (QA-01/QA-02 — Phase 17 removed, MozartBridge approach preferred)
**Releases:** v1.4.0, v1.4.1, v1.4.2, v1.4.3

---
