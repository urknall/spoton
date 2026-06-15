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

### v1.5 Podcasts (Phases 18-21)

- [x] **Phase 18: Podcast API Foundation** - OAuth scope + Client.pm methods for shows/episodes (completed 2026-06-14)
- [x] **Phase 19: Podcast Browse + Search** - Top-level menu, saved shows, episode playback, podcast search (completed 2026-06-14)
- [x] **Phase 20: Podcast Library Actions** - Follow/unfollow shows (completed 2026-06-15)
- [ ] **Phase 21: Podcast UX Polish + i18n** - Episode order setting, resume display, explicit filter, translations

## Phase Details

### Phase 18: Podcast API Foundation

**Goal**: The API layer supports all podcast operations with correct OAuth scope
**Depends on**: Phase 16.1 (existing API client infrastructure)
**Requirements**: API-01, API-02
**Success Criteria** (what must be TRUE):

  1. The `user-read-playback-position` scope is requested during auth and present in the stored token
  2. `getSavedShows` returns a paginated list of the user's saved shows from `GET /me/shows`
  3. `getShow` returns show metadata (name, description, total_episodes, artwork) from `GET /shows/{id}`
  4. `getShowEpisodes` returns paginated episodes for a show from `GET /shows/{id}/episodes`
  5. `getEpisode` returns episode metadata including `resume_point` from `GET /episodes/{id}`

**Plans**: 1 plan
Plans:

- [x] 18-01-PLAN.md — Binary scope update + Client.pm podcast methods + cache TTL extension

### Phase 19: Podcast Browse

**Goal**: Users can navigate to Podcasts, browse their saved shows, open a show, play episodes, and search for shows/episodes
**Depends on**: Phase 18
**Requirements**: POD-01, POD-02, POD-03, NAV-01, NAV-02, NAV-03, SRC-01, SRC-02, SRC-03
**Success Criteria** (what must be TRUE):

  1. A "Podcasts" entry appears in the top-level SpotOn menu alongside Home, Suche, Bibliothek
  2. "Meine Podcasts" lists all saved shows sorted by add date (most recently added first)
  3. Selecting a show opens its episode list with episode title, duration, and release date visible
  4. Selecting an episode begins playback via the existing ProtocolHandler (spoton:// URI)
  5. The Podcasts menu contains a "Podcast-Suche" entry as a distinct submenu item
  6. Entering a query under "Podcast-Suche" returns matching shows and episodes as separate result sections
  7. Show results and episode results each display up to 10 items (Dev Mode limit)

**Plans**: 2 plans
Plans:
**Wave 1**

- [x] 19-01-PLAN.md — Podcast string keys + test fixes

**Wave 2** *(blocked on Wave 1 completion)*

- [x] 19-02-PLAN.md — Plugin.pm podcast navigation, browse, and search feeds

### Phase 20: Podcast Library Actions

**Goal**: Users can follow or unfollow shows from within SpotOn
**Depends on**: Phase 19
**Requirements**: POD-04, POD-05
**Success Criteria** (what must be TRUE):

  1. A "Folgen" action on a show adds it to the user's saved shows via `PUT /me/library`
  2. An "Entfolgen" action on a saved show removes it via `DELETE /me/library`
  3. Following or unfollowing a show is reflected immediately when returning to "Meine Podcasts"

**Plans**: 1 plan
Plans:

- [x] 20-01-PLAN.md — Client.pm show methods + Plugin.pm Follow/Unfollow + strings + tests

### Phase 21: Podcast UX Polish + i18n

**Goal**: Episode lists convey playback state clearly, the episode order is configurable globally, and all strings are translated
**Depends on**: Phase 20
**Requirements**: UX-01, UX-02, UX-03, I18N-01
**Success Criteria** (what must be TRUE):

  1. A global plugin setting controls episode sort order (newest first vs. chronological) and applies to all show episode lists
  2. Episode list entries display a visual resume indicator (unplayed / in-progress / finished) derived from `resume_point`
  3. Episodes with explicit content are visibly marked (or omitted if the filter is active)
  4. All Podcast UI strings (menu labels, action names, setting labels, status indicators) appear correctly in all 11 supported languages

**Plans**: TBD
**UI hint**: yes

## Progress Table

| Phase | Milestone | Plans | Status | Completed |
|-------|-----------|-------|--------|-----------|
| 1-6 (15 phases) | v1.0 | 50/50 | Complete | 2026-06-03 |
| 7-12 (7 phases) | v1.1 | 13/13 | Complete | 2026-06-06 |
| 13-16.1 (5 phases) | v1.3 | 9/9 | Complete | 2026-06-13 |
| 18. Podcast API Foundation | v1.5 | 1/1 | Complete    | 2026-06-14 |
| 19. Podcast Browse | v1.5 | 2/2 | Complete    | 2026-06-14 |
| 20. Podcast Library Actions | v1.5 | 1/1 | Complete   | 2026-06-15 |
| 21. Podcast UX Polish + i18n | v1.5 | 0/? | Not started | - |

## Backlog

Items discovered during development — not assigned to a milestone.

1. **Eigene SpotOn Client-ID bei Spotify registrieren** — Blocked: Spotify requires 250k MAU + legally registered business. Extended Quota documentation deferred to future milestone.
2. **~~Online-Musiksammlung (Importer.pm / OnlineLibraryBase)~~** — Evaluiert und bewusst abgelehnt. API-Quota im Dev Mode macht Library-Scan extrem teuer; Browse > Library deckt den Use Case on-demand ab.
3. **LMS Community Repo Submission** — Deferred: ship stable version first, gather real-world feedback.

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-06-15 — Phase 20 planned (1 plan)*
