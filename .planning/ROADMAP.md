# Roadmap: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Created:** 2026-05-26
**Granularity:** standard

## Milestones

- ✅ **v1.0 Foundation** — Phases 1-6 (shipped 2026-06-03)
- ✅ **v1.1 Hardening & Reach** — Phases 7-12 (shipped 2026-06-06)
- 🚧 **v1.3 Polish & Publish** — Phases 13-17 (in progress)

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

### 🚧 v1.3 Polish & Publish (In Progress)

**Milestone Goal:** UX gaps closed and plugin ready for broader distribution — from working to publishable.

- [x] **Phase 13: Repo Maintenance** — CI, issue templates, CONTRIBUTING.md (completed 2026-06-07)
- [ ] **Phase 14: Connect Fixes** — Credential isolation + volume sync
- [ ] **Phase 15: Like Button** — Save/remove/check liked state from menus
- [ ] **Phase 16: macOS Universal Binary** — Intel + Apple Silicon via CI
- [ ] **Phase 17: B&O Format Verification** — Hardware QA on UPnPBridge players

## Phase Details

### Phase 13: Repo Maintenance
**Goal**: The GitHub repo has a working CI pipeline and contributor scaffolding that makes it easy to contribute and trust test results
**Depends on**: Nothing (independent of other v1.3 phases)
**Requirements**: REPO-01, REPO-02, REPO-03, REPO-04, REPO-05
**Success Criteria** (what must be TRUE):
  1. A push to `main` triggers GitHub Actions and runs `prove t/` against Perl 5.36 and 5.38 — results visible in commit status checks
  2. A contributor filing a bug sees a structured form with reproduction steps, LMS version, and OS fields
  3. A contributor filing a feature request sees a structured form with problem statement and alternatives fields
  4. A developer new to the project can follow CONTRIBUTING.md to run the test suite locally and submit a PR
**Plans:** 2/2 plans complete

Plans:
- [x] 13-01: Repo hygiene + GitHub Actions CI (perl-tests.yml, prove on 5.36 + 5.38)
- [x] 13-02: Issue templates + CONTRIBUTING.md + LICENSE + README

### Phase 14: Connect Fixes
**Goal**: Connect sessions start with correct volume and each player's Spotify credentials are isolated from other players and other users
**Depends on**: Nothing (independent of other v1.3 phases)
**Requirements**: CON-01, CON-02, CON-03
**Success Criteria** (what must be TRUE):
  1. When a second Spotify user connects to a different LMS player, the first player's Browse session continues showing their own Spotify library — credentials are not overwritten
  2. When Spotify Connect starts on a player, the Spotify volume matches the LMS player volume within 3 seconds — no jarring jump from a hardcoded default
  3. When the user changes volume in the Spotify app, LMS reflects the change within 3 seconds (down from the previous 20-second grace period)
**Plans:** 2 plans

Plans:
- [ ] 14-01: Credential isolation (separate --cache dir per player MAC) + volume fix (grace period + --volume-ctrl linear)

### Phase 15: Like Button
**Goal**: Users can save and remove tracks from Liked Songs directly from browse menus without leaving LMS
**Depends on**: Nothing (independent of other v1.3 phases)
**Requirements**: LIB-01, LIB-02, LIB-03, LIB-04, LIB-05
**Success Criteria** (what must be TRUE):
  1. A track context menu in Browse shows "Like" when the track is not in Liked Songs and "Unlike" when it is
  2. Selecting "Like" saves the track to Spotify Liked Songs — confirmed by the Liked Songs menu showing the track
  3. Selecting "Unlike" removes the track from Spotify Liked Songs — confirmed by the Liked Songs menu no longer showing the track
  4. The liked/unliked state check does not add perceptible delay to opening a track context menu
  5. After upgrading from a prior version, the plugin prompts for re-authorization exactly once to acquire the new library scopes
**Plans:** 2 plans

Plans:
- [ ] 15-01: API/Client.pm saveTracks / removeTracks / checkTracks methods + scope upgrade (cacheSchemaVersion bump)
- [ ] 15-02: Plugin.pm context menu item with liked state display + handler wiring

### Phase 16: macOS Universal Binary
**Goal**: macOS users can install SpotOn via the LMS plugin manager and have a working librespot binary without manual steps beyond a one-time Gatekeeper workaround
**Depends on**: Phase 13 (CI infrastructure for macOS build runners)
**Requirements**: PLT-01, PLT-02, PLT-03
**Success Criteria** (what must be TRUE):
  1. The `Bin/darwin/` directory contains a Universal Binary that runs natively on both Intel and Apple Silicon Macs
  2. Installing SpotOn via the LMS plugin manager on macOS downloads and runs the binary without quarantine errors blocking startup
  3. The Setup Guide documents the `xattr -d com.apple.quarantine` command for users who download the binary manually
**Plans:** 2 plans

Plans:
- [ ] 16-01: macOS CI job (macos-15-intel + macos-latest, lipo Universal Binary) + ad-hoc codesign
- [ ] 16-02: Helper.pm ISMAC block + Gatekeeper docs in Setup Guide

### Phase 17: B&O Format Verification
**Goal**: The format dropdown is confirmed working on B&O players via UPnPBridge, with all five format modes producing correct audio output
**Depends on**: Nothing (hardware verification, independent of code changes)
**Requirements**: QA-01, QA-02
**Success Criteria** (what must be TRUE):
  1. With a B&O player selected in LMS, each of the five format modes (Auto, OGG, PCM, FLAC, MP3) produces audible audio output without errors or silence
  2. In Auto mode, a B&O player (which does not support OGG) receives a non-OGG format — the auto-selection logic correctly detects capability
**Plans:** 2 plans

Plans:
- [ ] 17-01: B&O UPnPBridge format matrix test (5 modes × hardware player)

## Progress Table

| Phase | Milestone | Plans | Status | Completed |
|-------|-----------|-------|--------|-----------|
| 1-6 (15 phases) | v1.0 | 50/50 | Complete | 2026-06-03 |
| 7-12 (7 phases) | v1.1 | 13/13 | Complete | 2026-06-06 |
| 13. Repo Maintenance | v1.3 | 2/2 | Complete   | 2026-06-07 |
| 14. Connect Fixes | v1.3 | 0/1 | Not started | - |
| 15. Like Button | v1.3 | 0/2 | Not started | - |
| 16. macOS Universal Binary | v1.3 | 0/2 | Not started | - |
| 17. B&O Format Verification | v1.3 | 0/1 | Not started | - |

## Backlog

Items discovered during UAT — not blocking current milestone.

1. **~~Connect Credential Isolation~~** — moved to Phase 14
2. **Eigene SpotOn Client-ID bei Spotify registrieren** — Blocked: Spotify requires 250k MAU + legally registered business. Extended Quota documentation deferred to future milestone.
3. **~~Format-Dropdown mit Nicht-OGG-Playern testen~~** — moved to Phase 17
4. **~~Connect-Mode Lautstärke-Diskrepanz~~** — moved to Phase 14
5. **~~Online-Musiksammlung (Importer.pm / OnlineLibraryBase)~~** — Evaluiert und bewusst abgelehnt. Spotty-NG importiert Spotify-Playlists/Alben in die LMS-Bibliothek via `Slim::Plugin::OnlineLibraryBase`. Für SpotOn abgelehnt wegen: (a) API-Quota im Dev Mode macht Library-Scan extrem teuer, (b) Browse > Library deckt den Use Case on-demand ab, (c) hohe Wartungslast für fraglichen Mehrwert, (d) Sync-Drift. Kann bei eigener App mit Extended Quota neu evaluiert werden.

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-06-06 — v1.3 milestone roadmap created*
