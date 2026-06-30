# Roadmap: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Created:** 2026-05-26
**Granularity:** standard

## Milestones

- ✅ **v1.0 Foundation** — Phases 1-6 (shipped 2026-06-03)
- ✅ **v1.1 Hardening & Reach** — Phases 7-12 (shipped 2026-06-06)
- ✅ **v1.3 Polish & Publish** — Phases 13-16.1 (shipped 2026-06-13)
- ✅ **v1.5 Podcasts** — Phases 18-21 (shipped 2026-06-15)
- ✅ **v2.0 Browse Daemon Migration** — Phases 22, 25-32 (shipped 2026-06-25)

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
- ✅ **v2.0 Browse Daemon Migration** — Phases 22, 25-32 (shipped 2026-06-25) → [archive](milestones/v2.0-ROADMAP.md)
- ✅ **v2.1 Context Menu** — Phases 33-34 (shipped 2026-06-26)

## Progress Table

| Phase | Milestone | Plans | Status | Completed |
|-------|-----------|-------|--------|-----------|
| 1-6 (15 phases) | v1.0 | 50/50 | Complete | 2026-06-03 |
| 7-12 (7 phases) | v1.1 | 13/13 | Complete | 2026-06-06 |
| 13-16.1 (5 phases) | v1.3 | 9/9 | Complete | 2026-06-13 |
| 18-21 (4 phases) | v1.5 | 6/6 | Complete | 2026-06-15 |
| 22. Seek + Favorites Bugfixes | v2.0 | 1/1 | Complete | 2026-06-17 |
| 25. Play-All Full Pagination | v2.0 | 1/1 | Complete | 2026-06-18 |
| 26. Browse Error Recovery | v2.0 | 2/2 | Complete | 2026-06-21 |
| 27. Pipeline Failure Recovery | v2.0 | 1/1 | Complete | 2026-06-22 |
| 28. Persistent Browse Daemon | v2.0 | 3/3 | Complete | 2026-06-22 |
| 29. Unified Daemon | v2.0 | 3/3 | Complete | 2026-06-22 |
| 30. Legacy Pipe Cleanup | v2.0 | 2/2 | Complete | 2026-06-22 |
| 31. Code Review Hardening | v2.0 | 2/2 | Complete | 2026-06-24 |
| 32. Status Page | v2.0 | 2/2 | Complete | 2026-06-25 |

| 33. More Context Menu | v2.1 | 1/1 | Complete | 2026-06-26 |
| 34. Add to Playlist | v2.1 | 1/1 | Complete | 2026-06-26 |
| 35. Liked Songs Play-All Throttle | v2.1.2 | 1/1 | Complete | 2026-06-26 |

| 36. Session Health Monitoring | — | 2/2 | Complete   | 2026-06-30 |

### Phase 36: Session Health Monitoring
**Goal:** Prevent cold-start playback failure after overnight daemon idle by enhancing the `/health` endpoint with Spotify session health reporting and adding Perl-side health-aware daemon monitoring.

**Spec:** `.planning/specs/session-health-SPEC.md`

**Scope:**
- Rust: Enhance `/health` to return JSON with `session_valid`, `session_age_secs`, `idle_secs`
- Perl: Add periodic health check in `_streamAlivePoll` that calls `/health` and restarts daemon on stale session
- Log cleanup: Downgrade misleading 60s watchdog logs from INFO to DEBUG

**Constraints:**
- Rust binary rebuild required (CI tag push)
- Binary + Perl must ship together
- Must not disrupt active Connect sessions

**Plans:** 2/2 plans complete

Plans:
- [x] 36-01-PLAN.md — Rust: Enhanced /health endpoint with session health JSON + shared state
- [x] 36-02-PLAN.md — Perl: Health-aware monitoring, Status Page display, log cleanup

## Backlog

Items discovered during development — not assigned to a milestone.

1. **Eigene SpotOn Client-ID bei Spotify registrieren** — Blocked: Spotify requires 250k MAU + legally registered business. Extended Quota documentation deferred to future milestone.
2. **~~Online-Musiksammlung (Importer.pm / OnlineLibraryBase)~~** — Evaluiert und bewusst abgelehnt. API-Quota im Dev Mode macht Library-Scan extrem teuer; Browse > Library deckt den Use Case on-demand ab.
3. ~~**LMS Community Repo Submission**~~ — Erledigt: Plugin im Community Repo veröffentlicht.
4. ~~**ZeroConf Auth UX: "Connected" an Spotify App melden**~~ — Verworfen: Setup Guide erklärt das Verhalten, kein technischer Fix möglich ohne Playback-Session.
5. ~~**Diagnostics: "Clear Logs" Button in Settings**~~ — Implementiert in v1.7.4 (truncate on daemon restart + Clear Logs button).
6. **Spotty Favorites Migration** — Settings-Button der `spotify://` Einträge in LMS Favorites und Playlists als `spoton://` Duplikate anlegt. Originale bleiben erhalten, User kann Spotty danach deinstallieren. URI-Schema nach Prefix ist identisch (`track:ID`, `album:ID`, etc.). Idee von Paul Webster (Forum #32, 2026-06-19).
7. **Search Pagination** — Suchergebnisse über offset-Pagination nachladen (aktuell max 50 pro Typ, Spotify hat oft 100+). LMS OPMLBased-Pagination nutzen.
8. **Forum Monitor + Draft Generation** — GitHub Action (cron) die den Lyrion-Forum-Thread pollt, neue Posts erkennt, via Claude API Draft-Replies generiert und als GitHub Issues zur Review erstellt. (Ehemals Phase 23, bereits als Self-Hosted Runner implementiert.)
9. **Forum Auto-Post** — Label-getriggerter GitHub Action Workflow der approved Draft-Replies automatisch im vBulletin-Forum postet. (Ehemals Phase 24.)

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-06-27 — Phase 35 complete, v2.1.2 released*
