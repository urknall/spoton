---
phase: 33-more-context-menu
reviewed: 2026-06-26T13:45:00Z
depth: standard
files_reviewed: 3
files_reviewed_list:
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/ProtocolHandler.pm
  - Plugins/SpotOn/Connect.pm
findings:
  critical: 1
  warning: 2
  info: 2
  total: 5
status: issues_found
fixes_applied: true
---

# Phase 33: Code Review Report

**Reviewed:** 2026-06-26T13:45:00Z
**Depth:** standard
**Files Reviewed:** 3
**Status:** issues_found (all Critical + Warning findings fixed)

## Summary

Phase 33 extends the "More" context menu for tracks and episodes across two entry points: `trackInfoMenu` (Plugin.pm, registered via `registerInfoProvider`) and `trackInfoURL` (ProtocolHandler.pm, protocol handler info view). Both now emit Artist View, Album View, Like/Unlike for tracks and View Show, Follow/Unfollow for episodes. Cache write sites in Plugin.pm, ProtocolHandler.pm, and Connect.pm are extended to persist the required IDs (`artistId`, `artistIds`, `albumId`, `showId`, `showName`).

The implementation is structurally sound and consistent across both menu entry points. String keys exist, feed handlers accept the correct passthrough shapes, and guards prevent navigation items from appearing when IDs are unavailable. One critical cache-key bug and two code quality issues were found and fixed.

## Critical Issues

### CR-01: [FIXED] trackInfoMenu cache lookup does not normalize URL format -- Artist/Album/Show items silently vanish

**File:** `Plugins/SpotOn/Plugin.pm:477`
**Issue:** `trackInfoMenu` constructs the cache key as `md5_hex($url)` using the raw URL passed by LMS. However, `getMetadataFor` in ProtocolHandler.pm (lines 722-726) explicitly normalizes `spoton:track:ID` to `spoton://track:ID` before cache lookup, with the comment "Normalize: cache is keyed on spoton://track:ID but LMS may pass spoton:track:ID". All cache write sites (`_trackItem`, `_cacheExplodedTrack`, `_cacheExplodedEpisode`, `_asyncRefetch`, `_fetchTrackMetadata`) store under `md5_hex("spoton://...")`. If LMS passes the non-double-slash form to `trackInfoMenu`, the cache lookup produces a different MD5 hash, the `$meta` hash is empty, and all new Phase 33 items (Artist View, Album View, View Show, Follow) silently do not appear. The Like/Unlike item still appears (it does not depend on cache), so the regression is partial and hard to notice.

The defensive regex at line 465 (`^spoton:(?://)?track:`) explicitly handles both URL forms, confirming the developer expected both. The cache lookup at line 477 does not.

**Fix applied:** Added URL normalization before cache lookup in `trackInfoMenu`, matching the pattern used in `getMetadataFor`.

## Warnings

### WR-01: [FIXED] Artist/album ID extraction logic duplicated 4 times across 3 files

**File:** `Plugins/SpotOn/Plugin.pm:777-781`, `Plugins/SpotOn/ProtocolHandler.pm:827-830`, `Plugins/SpotOn/ProtocolHandler.pm:957-961`, `Plugins/SpotOn/Connect.pm:957-960`
**Issue:** The same ID extraction pattern (artistId from first artist, artistIds as JSON-encoded array of {id, name}, albumId from album hash) is copy-pasted verbatim in four locations. A parallel pattern for showId/showName exists in three locations. If the schema changes (e.g., Spotify removes the `id` field from simplified artist objects, or a future phase needs additional fields), all sites must be updated in lockstep. Any drift creates silent cache inconsistency where some paths populate fields and others do not.

**Fix applied:** Extracted `_extractTrackIds($track)` helper in Plugin.pm (root module, already imported by all callers). All 4 inline copies replaced with calls to this single source of truth. Removed unused `use JSON::XS qw(encode_json)` from ProtocolHandler.pm (no longer needed after refactor). Also resolves IN-02 (leading-underscore variable names eliminated).

### WR-02: [FIXED] Inconsistent type/favorites attributes between trackInfoMenu and trackInfoURL Like item

**File:** `Plugins/SpotOn/Plugin.pm:497-502` vs `Plugins/SpotOn/ProtocolHandler.pm:795-800`
**Issue:** The Like/Unlike item in `trackInfoMenu` sets `favorites => 0` but omits `type => 'link'`. The same item in `trackInfoURL` sets `type => 'link'` but omits `favorites => 0`. This pre-dates Phase 33 but is amplified by the new code: the new Artist View and Album View items are consistent between both entry points (both have `type => 'link'`, neither has `favorites => 0`), making the Like item's divergence more visible. Without `favorites => 0` in `trackInfoURL`, the Like action may appear as a Favorites-eligible item on some skins, which is confusing since "Like" is a Spotify-side action, not an LMS favorite.

**Fix applied:** Both Like items now include both `type => 'link'` and `favorites => 0`.

## Info

### IN-01: artistIds stored in cache but never consumed

**File:** `Plugins/SpotOn/Plugin.pm` (`_extractTrackIds`), all cache write sites via helper
**Issue:** The `artistIds` field (JSON-encoded array of `{id, name}` pairs for all track artists) is written to cache at all track metadata sites but is never read by any consumer. `trackInfoMenu` and `trackInfoURL` only use `artistId` (the first artist's ID). This is presumably forward-looking for multi-artist navigation, but currently it is dead data that costs a `JSON::XS::encode_json` call on every cache write.
**Fix:** Either add a comment documenting the intended future use (e.g., `# Phase N: multi-artist menu`), or remove the field until it is needed.

### IN-02: [FIXED by WR-01] Leading-underscore local variable names in _trackItem

**File:** `Plugins/SpotOn/Plugin.pm:777-781`
**Issue:** Variables `$_artists`, `$_artistId`, `@_namedA`, `$_artistIds` used leading underscore prefix, clashing with Perl's `$_` convention and differing from equivalent code in other files.
**Fix:** Resolved by WR-01 refactor -- inline code replaced with `_extractTrackIds` helper call, eliminating these variable names entirely.

---

_Reviewed: 2026-06-26T13:45:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
