# Roadmap: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Created:** 2026-05-26
**Granularity:** standard

## Milestones

- ✅ **v1.0 Foundation** — Phases 1-6 (shipped 2026-06-03)
- ✅ **v1.1 Hardening & Reach** — Phases 7-12 (shipped 2026-06-06)
- ✅ **v1.3 Polish & Publish** — Phases 13-16.1 (shipped 2026-06-13)
- 🔄 **v1.5 Podcasts** — Phases 18-21 (active)

## Phases

<details>
<summary>✅ v1.0 Foundation (Phases 1-6) — SHIPPED 2026-06-03</summary>

- [x] **Phase 1: Plugin Skeleton + Binary Foundation** — completed 2026-05-26
- [x] **Phase 2: Auth + API Foundation** (6/6 plans) — completed 2026-05-27
- [x] **Phase 02.1: OAuth-PKCE Browser Auth** (4/4 plans) — completed 2026-05-27
- [x] **Phase 3: Browse + Navigation** (3/3 plans) — completed 2026-05-28
- [x] **Phase 4: Single-Track Streaming** (2/2 plans) — completed 2026-05-28
- [x] **Phase 04.1: Streaming Bug Fixes + Passthrough Binary** (2/2 plans) — completed 2026-05-28
- [x] **Phase 04.2: Credentials + Made For You Fix** (2/2 plans) — completed 2026-05-29
- [x] **Phase 04.3: ZeroConf + Keymaster Auth** (4/4 plans) — completed 2026-05-29
- [x] **Phase 04.4: Dual-Token API Routing** (2/2 plans) — completed 2026-05-29
- [x] **Phase 5: Spotify Connect** (5/5 plans) — completed 2026-06-01
- [x] **Phase 05.1: Connect Audio Streaming Bugfix** (3/3 plans) — completed 2026-06-01
- [x] **Phase 05.2: Connect Controls & Resume** (2/2 plans) — completed 2026-06-01
- [x] **Phase 05.3: Sync Groups + Connect Robustness** (3/3 plans) — completed 2026-06-02
- [x] **Phase 05.4: mDNS Connect Discovery Fix** (3/3 plans) — completed 2026-06-02
- [x] **Phase 6: Polish + DSTM + Settings** (5/5 plans) — completed 2026-06-03

</details>

<details>
<summary>✅ v1.1 Hardening & Reach (Phases 7-12) — SHIPPED 2026-06-06</summary>

- [x] **Phase 7: DE→EN Code Cleanup** (1/1 plan) — completed 2026-06-03
- [x] **Phase 8: Multi-Arch Binary Distribution** (2/2 plans) — completed 2026-06-03
- [x] **Phase 9: Stream Metadata** (1/1 plan) — completed 2026-06-04
- [x] **Phase 9.5: Prod Deployment & Monitoring** (2/2 plans) — completed 2026-06-04
- [x] **Phase 10: Connect-DSTM** (3/3 plans) — completed 2026-06-04
- [x] **Phase 11: Track History Metadata** (2/2 plans) — completed 2026-06-05
- [x] **Phase 12: Protocol Handler Rename** (2/2 plans) — completed 2026-06-05

</details>

<details>
<summary>✅ v1.3 Polish & Publish (Phases 13-16.1) — SHIPPED 2026-06-13</summary>

- [x] **Phase 13: Repo Maintenance** (2/2 plans) — completed 2026-06-07
- [x] **Phase 14: Connect Fixes** (2/2 plans) — completed 2026-06-07
- [x] **Phase 15: Like Button** (2/2 plans) — completed 2026-06-11
- [x] **Phase 16: macOS Universal Binary** (2/2 plans) — completed 2026-06-11
- [x] **Phase 16.1: CI Conditional Build** (1/1 plan) — completed 2026-06-12

</details>

- ✅ **v1.5 Podcasts** — Phases 18-21 (shipped 2026-06-15) → [archive](milestones/v1.5-ROADMAP.md)

## Active

- [x] **Phase 22: Seek + Favorites Bugfixes** — completed 2026-06-17
  **Goal**: Fix seeking (duration 0:00 in seek bar) and LMS favorites (spotify: scheme statt spoton://)
  **Plans:** 1 plan

  Plans:
  - [x] 22-01-PLAN.md — Fix seek bar duration + favorites URL scheme + explodePlaylist

- [ ] **Phase 23: Forum Monitor + Draft Generation**
  **Goal**: GitHub Action (cron) die den Lyrion-Forum-Thread pollt, neue Posts erkennt, via Claude API Draft-Replies generiert und als GitHub Issues zur Review erstellt.

- [ ] **Phase 24: Forum Auto-Post**
  **Goal**: Label-getriggerter GitHub Action Workflow der approved Draft-Replies automatisch im vBulletin-Forum postet.

## Progress Table

| Phase | Milestone | Plans | Status | Completed |
|-------|-----------|-------|--------|-----------|
| 1-6 (15 phases) | v1.0 | 50/50 | Complete | 2026-06-03 |
| 7-12 (7 phases) | v1.1 | 13/13 | Complete | 2026-06-06 |
| 13-16.1 (5 phases) | v1.3 | 9/9 | Complete | 2026-06-13 |
| 18. Podcast API Foundation | v1.5 | 1/1 | Complete    | 2026-06-14 |
| 19. Podcast Browse | v1.5 | 2/2 | Complete    | 2026-06-14 |
| 20. Podcast Library Actions | v1.5 | 1/1 | Complete   | 2026-06-15 |
| 21. Podcast UX Polish + i18n | v1.5 | 2/2 | Complete   | 2026-06-15 |
| 22. Seek + Favorites Bugfixes | — | 1/1 | Complete | 2026-06-17 |

## Backlog

Items discovered during development — not assigned to a milestone.

1. **Eigene SpotOn Client-ID bei Spotify registrieren** — Blocked: Spotify requires 250k MAU + legally registered business. Extended Quota documentation deferred to future milestone.
2. **~~Online-Musiksammlung (Importer.pm / OnlineLibraryBase)~~** — Evaluiert und bewusst abgelehnt. API-Quota im Dev Mode macht Library-Scan extrem teuer; Browse > Library deckt den Use Case on-demand ab.
3. **LMS Community Repo Submission** — Deferred: ship stable version first, gather real-world feedback.
4. **ZeroConf Auth UX: "Connected" an Spotify App melden** — Spotify App zeigt endlos "Connecting..." beim ZeroConf-Handshake, weil SpotOn keine Playback-Session startet. User denken Auth ist fehlgeschlagen, obwohl Credentials korrekt übernommen wurden. Fix: nach erfolgreichem Credential-Empfang ein Connect-Session-Signal an die App senden, damit "Connected" angezeigt wird. Entdeckt im Forum (l.e.hauser, CJS — 2026-06-17).
5. **Diagnostics: "Clear Logs" Button in Settings** — Connect-Daemon-Logs (`<cachedir>/spoton/*-connect.log`) wachsen unbegrenzt (append mode). Kein UI zum Zurücksetzen. Fix: Button in SpotOn Settings der alle `*-connect.log` Dateien truncated. Entdeckt im Forum (CJS — 2026-06-17).

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-06-18 — Backlog: Clear Logs Button*
