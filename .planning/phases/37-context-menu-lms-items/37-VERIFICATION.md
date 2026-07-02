---
phase: 37-context-menu-lms-items
verified: 2026-06-30T16:00:00Z
status: human_needed
score: 2/3 must-haves verified
behavior_unverified: 1
overrides_applied: 0
re_verification: false
behavior_unverified_items:
  - truth: "Standard LMS actions (Add to Favorites, play controls, More Info) appear in the More menu for SpotOn tracks"
    test: "On a running LMS with SpotOn: navigate to a SpotOn track, press More. Observe menu contents."
    expected: "Standard LMS items (Add to Favorites, Add to Playlist, More Info) appear alongside SpotOn-specific items (Artist View, Album View, Like, Add to Playlist)"
    why_human: "Removal of trackInfoURL is verified; LMS menu assembly is a framework runtime behavior (Slim::Menu::TrackInfo::menu()) that cannot be exercised without a running LMS instance"
human_verification:
  - test: "Standard LMS items appear in SpotOn More menu for a track"
    expected: "Add to Favorites, play controls, and More Info appear alongside SpotOn items in the More menu for a spoton:// track"
    why_human: "LMS menu assembly is a framework runtime behavior — trackInfoURL removal is verified, but the actual rendering of Slim::Menu::TrackInfo items requires a live LMS instance"
  - test: "Add to Favorites creates a working favorite from SpotOn track"
    expected: "LMS Favorites entry for a SpotOn track navigates back to the track; pressing Play works"
    why_human: "End-to-end LMS Favorites integration with spoton:// URIs requires a live LMS instance with SpotOn running"
  - test: "Episode More menu: MANAGE_FOLLOW item is selectable (pre-existing CR-01 defect)"
    expected: "PLUGIN_SPOTON_MANAGE_FOLLOW renders as a navigable link in the episode context menu, not as a non-interactive text node"
    why_human: "CR-01 (37-REVIEW.md): PLUGIN_SPOTON_MANAGE_FOLLOW in Plugin.pm line 603-608 is missing type => 'link'. All other items in trackInfoMenu carry type => 'link'. The test suite checks only name fields and does not catch this defect. This is pre-existing in Plugin.pm (not introduced by Phase 37) but was discovered during this phase's code review and has not been fixed."
---

# Phase 37: Context Menu LMS Items — Verification Report

**Phase Goal:** Standard LMS actions (Add to Favorites, etc.) alongside SpotOn entries in More menu (GH #55)
**Verified:** 2026-06-30
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Standard LMS actions (Add to Favorites, play controls, More Info) appear in the More menu for SpotOn tracks | ⚠️ PRESENT_BEHAVIOR_UNVERIFIED | trackInfoURL confirmed absent (0 grep hits); registerInfoProvider wired at Plugin.pm:227; LMS runtime menu assembly cannot be exercised without a live LMS instance |
| 2 | SpotOn-specific entries (Artist View, Album View, Like, Add to Playlist) still appear in the More menu | ✓ VERIFIED | All 4 track items confirmed by CTX-04 tests (ok 6-11); trackInfoMenu returns 4-item arrayref for spoton://track URLs with artistId/albumId |
| 3 | ProtocolHandler.pm no longer defines trackInfoURL — LMS framework handles menu assembly | ✓ VERIFIED | `grep -c 'trackInfoURL' Plugins/SpotOn/ProtocolHandler.pm` returns 0; sub _cacheExplodedTrack now starts at line 758 (was 839), confirming 80 lines deleted; sub getMetadataFor intact at line 675 |

**Score:** 2/3 truths verified (1 present, behavior-unverified)

### Roadmap Success Criteria

| # | Success Criterion | Status | Notes |
|---|-------------------|--------|-------|
| SC1 | User sees standard LMS actions (Add to Favorites, Add to Playlist, More Info) in SpotOn's More menu for tracks, albums, and artists | ⚠️ PRESENT_BEHAVIOR_UNVERIFIED | Mechanism in place (trackInfoURL removed); runtime rendering requires live LMS. For albums and artists (browse pages), LMS menu assembly was already correct before this phase — only track/episode more menu was broken. |
| SC2 | Standard LMS actions execute correctly — adding a SpotOn track to LMS Favorites actually creates a working favorite entry | ⚠️ PRESENT_BEHAVIOR_UNVERIFIED | Cannot verify without live LMS + Favorites infrastructure; out of scope for automated test |

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/ProtocolHandler.pm` | Protocol handler without trackInfoURL override; contains sub getMetadataFor | ✓ VERIFIED | 915 lines; 0 occurrences of trackInfoURL; getMetadataFor at line 675; _cacheExplodedTrack at line 758 |
| `t/14_context_menu.t` | Regression test confirming trackInfoURL is removed and trackInfoMenu behaves correctly; contains "trackInfoURL" | ✓ VERIFIED | 479 lines; 16 tests; all pass (prove -v t/14_context_menu.t: 16/16 ok); CTX-01 gate at line 364 uses !defined(&Plugins::SpotOn::ProtocolHandler::trackInfoURL) |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `Plugins/SpotOn/Plugin.pm` | `Slim::Menu::TrackInfo` | `registerInfoProvider(spotonTrackInfo)` | ✓ VERIFIED | Line 227: `Slim::Menu::TrackInfo->registerInfoProvider( spotonTrackInfo => ( after => 'top', func => \&trackInfoMenu, ) )` — exact pattern match; trackInfoMenu defined at line 539 |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| CTX-01: trackInfoURL not defined | `prove -v t/14_context_menu.t` (test 3) | ok 3 - CTX-01: trackInfoURL not defined in ProtocolHandler (regression gate) | ✓ PASS |
| CTX-04: trackInfoMenu returns 4-item arrayref for track URL | `prove -v t/14_context_menu.t` (tests 6-11) | ok 6-11 — arrayref with ARTIST_VIEW, ALBUM_VIEW, MANAGE_LIKE, ADD_TO_PLAYLIST | ✓ PASS |
| CTX-05: trackInfoMenu returns 3-item arrayref for episode URL | `prove -v t/14_context_menu.t` (tests 12-16) | ok 12-16 — arrayref with SHOW_VIEW, MANAGE_FOLLOW, ADD_TO_PLAYLIST | ✓ PASS |
| Full test suite | `prove -v t/` | Files=14, Tests=419 — Result: PASS | ✓ PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| CTX-01 | 37-01-PLAN.md | Standard-LMS-Menüeinträge (Add to Favorites, etc.) erscheinen im SpotOn More-Menü neben den SpotOn-Einträgen | ✓ SATISFIED | trackInfoURL removed (mechanism); CTX-01 regression gate passes; registerInfoProvider wired; runtime appearance needs human verification |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `Plugins/SpotOn/Plugin.pm` | 603-608 | Missing `type => 'link'` on PLUGIN_SPOTON_MANAGE_FOLLOW in trackInfoMenu episode branch | ⚠️ Warning | Pre-existing defect (Plugin.pm not modified by Phase 37); episode MANAGE_FOLLOW item renders as non-selectable text node in LMS OPML. All other items in trackInfoMenu carry `type => 'link'`. Discovered during Phase 37 code review (CR-01 in 37-REVIEW.md); not introduced by this phase; fix deferred. Recommend addressing in next phase touching Plugin.pm. |

No TBD/FIXME/XXX markers found in files modified by this phase (ProtocolHandler.pm, t/14_context_menu.t, t/10_stream_metadata.t, t/11_track_history.t).

### Human Verification Required

### 1. Standard LMS Items in SpotOn More Menu

**Test:** On a running LMS with SpotOn installed, navigate to any SpotOn track via Browse or Search, then press the More (arrow) button. Observe the context menu.
**Expected:** Standard LMS items appear — at minimum: Add to Favorites, More Info. SpotOn items (Artist View, Album View, Like Track, Add to Playlist) appear alongside them in the same menu.
**Why human:** trackInfoURL removal is verified by grep and the CTX-01 regression test. The LMS framework's Slim::Menu::TrackInfo::menu() assembly — which merges registered provider items with standard items when no trackInfoURL override is present — is a runtime behavior that cannot be exercised in the Perl unit test environment.

### 2. Add to Favorites Creates a Working Favorite

**Test:** From the SpotOn More menu on a track, select Add to Favorites. Navigate to LMS Favorites. Find the newly added SpotOn track entry and press Play.
**Expected:** The track plays correctly. The Favorites entry persists across LMS restarts.
**Why human:** ROADMAP SC2 requires end-to-end LMS Favorites integration with spoton:// URIs. This depends on LMS framework handling of the URI scheme and is not exercised by any unit test.

### 3. Episode MANAGE_FOLLOW Is Selectable (CR-01 Follow-up)

**Test:** Navigate to a podcast episode via SpotOn Browse, press More. Attempt to select the "Follow Show / Unfollow Show" item.
**Expected:** The item is selectable and navigates to the follow/unfollow action.
**Why human:** CR-01 from 37-REVIEW.md: `PLUGIN_SPOTON_MANAGE_FOLLOW` at Plugin.pm line 603-608 is missing `type => 'link'`. Without this field, LMS renders the item as a non-navigable text node. The defect is pre-existing (Plugin.pm was not modified in Phase 37) and the test suite does not catch it (CTX-05 checks only `name` fields, not `type`). Manual verification is needed to confirm impact; fix should be applied in the next phase that touches Plugin.pm.

---

_Verified: 2026-06-30T16:00:00Z_
_Verifier: Claude (gsd-verifier)_
