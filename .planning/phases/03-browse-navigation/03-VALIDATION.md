---
phase: 03
slug: browse-navigation
status: draft
nyquist_compliant: true
wave_0_complete: true
created: 2026-05-28
---

# Phase 03 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Perl syntax validation (`perl -c`) + manual LMS integration |
| **Config file** | none — LMS plugins use no separate test framework |
| **Quick run command** | `perl -c Plugins/SpotOn/API/Client.pm && perl -c Plugins/SpotOn/Plugin.pm` |
| **Full suite command** | `perl -c Plugins/SpotOn/API/Client.pm && perl -c Plugins/SpotOn/API/TokenManager.pm && perl -c Plugins/SpotOn/Plugin.pm` |
| **Estimated runtime** | ~2 seconds |

**Note:** Phase 03 is a pure Perl plugin extension with no unit test framework. `perl -c` validates syntax, compilation, and module resolution. Functional verification requires a running LMS instance and is covered by the checkpoint:human-verify task in Plan 03.

---

## Sampling Rate

- **After every task commit:** `perl -c <modified_file>.pm` — syntax check
- **After every plan wave:** Full suite syntax check (all 3 .pm files)
- **Before `/gsd:verify-work`:** Full suite + human verification of all NAV requirements
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 03-01-T1 | 01 | 1 | NAV-01..NAV-11 (API layer) | T-03-01, T-03-02 | uri_escape in _request(); no token logging | syntax | `perl -c Plugins/SpotOn/API/Client.pm` | Yes | pending |
| 03-01-T2 | 01 | 1 | NAV-03 (scope), all (i18n) | T-03-03 | user-consented scope expansion via PKCE | syntax+grep | `perl -c Plugins/SpotOn/API/TokenManager.pm && grep -c 'user-follow-read' Plugins/SpotOn/API/TokenManager.pm` | Yes | pending |
| 03-02-T1 | 02 | 2 | NAV-01..NAV-03, NAV-08..NAV-10 | T-03-05..T-03-07 | LMS escapes HTML in OPML rendering | syntax | `perl -c Plugins/SpotOn/Plugin.pm` | Yes | pending |
| 03-03-T1 | 03 | 3 | NAV-02..NAV-07, NAV-10, NAV-11 | T-03-08..T-03-11 | uri_escape in Client.pm; null track check | syntax+grep | `perl -c Plugins/SpotOn/Plugin.pm && grep -c 'PLUGIN_SPOTON_ARTIST_VIEW' Plugins/SpotOn/strings.txt` | Yes | pending |
| 03-03-T2 | 03 | 3 | NAV-01..NAV-11 | — | — | manual | Human verify in LMS (checkpoint:human-verify) | N/A | pending |

*Status: pending · green · red · flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements.

- `perl -c` is available system-wide (Perl >= 5.10 guaranteed by LMS floor)
- No test framework setup needed — syntax validation suffices for automated checks
- Functional validation deferred to human-verify checkpoint (Plan 03, Task 2)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Top-level menu shows Home, Search, Library | NAV-01 | Requires running LMS instance | Navigate to SpotOn in LMS, verify 3 top-level entries |
| Home feed shows 3 items | NAV-02 | Requires Spotify auth + API access | Open Home, verify Recently Played / Made For You / Top Tracks |
| Library shows 4 items | NAV-03 | Requires Spotify auth + API access | Open Library, verify Liked Songs / Albums / Artists / Playlists |
| Search returns grouped results | NAV-04 | Requires Spotify auth + API access | Search "Radiohead", verify Tracks/Albums/Artists/Playlists sections |
| Artist detail shows 4 discography sections | NAV-05 | Requires Spotify auth + API access | Navigate to any artist, verify Albums/Singles/Compilations/Appears On |
| Album tracklist with numbers and duration | NAV-06 | Requires running LMS + Spotify auth | Navigate to an album, verify numbered tracklist |
| Playlist shows paginated tracks | NAV-07 | Requires running LMS + Spotify auth | Open a playlist, verify tracks with pagination |
| Liked Songs unconditional access | NAV-08 | Requires Spotify auth | Open Library > Liked Songs, verify no gating |
| Library items sorted by recently added | NAV-09 | Requires Spotify auth | Verify sort order matches Spotify app |
| Dev Mode removed endpoints hidden | NAV-10 | Requires Dev Mode API restrictions | Verify no errors for missing endpoints, no empty placeholders |
| Search pagination at limit=10 | NAV-11 | Requires Spotify auth | Drill into search category, scroll to verify pagination |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references (none needed)
- [x] No watch-mode flags
- [x] Feedback latency < 10s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved
