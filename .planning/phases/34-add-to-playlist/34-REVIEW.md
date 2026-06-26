---
phase: 34-add-to-playlist
reviewed: 2026-06-26T15:42:00Z
depth: standard
files_reviewed: 4
files_reviewed_list:
  - Plugins/SpotOn/API/Client.pm
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/ProtocolHandler.pm
  - Plugins/SpotOn/strings.txt
findings:
  critical: 1
  warning: 0
  info: 1
  total: 2
status: issues_found
---

# Phase 34: Code Review Report

**Reviewed:** 2026-06-26T15:42:00Z
**Depth:** standard
**Files Reviewed:** 4
**Status:** issues_found

## Summary

Phase 34 adds "Add to Playlist" functionality to the track/episode context menus. The implementation spans four files: a new `addToPlaylist` API method in Client.pm, playlist picker and callback handlers in Plugin.pm, menu wiring in both Plugin.pm (trackInfoMenu) and ProtocolHandler.pm (trackInfoURL), and 4 new i18n string blocks in strings.txt.

The API method has solid input validation (playlist ID regex guard). The menu wiring in both trackInfoMenu and trackInfoURL is consistent and correctly placed inside the `$accountId` guard. The i18n strings are correct across all 11 languages.

One critical pagination bug was found: `SpotOnAddToPlaylist` reads `index`/`quantity` from the passthrough data instead of the LMS params hash, breaking pagination for users with more than ~50 playlists.

## Critical Issues

### CR-01: SpotOnAddToPlaylist reads pagination params from wrong variable

**File:** `Plugins/SpotOn/Plugin.pm:728-729`
**Issue:** The handler reads `$args->{index}` and `$args->{quantity}` to determine pagination offset and page size. However, `$args` is the 4th argument (passthrough data containing `spotifyUri` and `accountId`), not the 3rd argument (`$params` -- the LMS params hash containing `index` and `quantity`).

Every other paginated handler with passthrough in the codebase follows the convention of reading pagination from the 3rd argument. For example:
- `_artistAlbumsFeed` (line 2132): `($client, $callback, $args, $passthrough)` -- reads `$args->{index}`
- `_showFeed` (line 1479): `($client, $callback, $args, $passthrough)` -- reads `$args->{index}`

In `SpotOnAddToPlaylist`, the 3rd argument is named `$params` and the 4th is named `$args`, but pagination is read from `$args` (4th). Since the passthrough hash `{spotifyUri => ..., accountId => ...}` never contains `index` or `quantity`, these always evaluate to `undef`, falling back to `0` and `200` respectively.

**Impact:** Offset is always 0. Users with more than ~50 playlists will see the same first page of playlists repeated on every "page" of results. Pagination is effectively broken.

**Fix:**
```perl
    my $offset = $params->{index} || 0;
    my $qty    = $params->{quantity} || 200;
```

## Info

### IN-01: Unused i18n string PLUGIN_SPOTON_SELECT_PLAYLIST

**File:** `Plugins/SpotOn/strings.txt:2042`
**Issue:** The `PLUGIN_SPOTON_SELECT_PLAYLIST` string block (11 languages) is defined but never referenced in any Perl source file. It was likely intended as a header for the playlist picker menu but not wired up. Dead i18n strings add translation maintenance burden.
**Fix:** Either remove the string block, or use it as the menu title in `SpotOnAddToPlaylist` (e.g., as a header item or window title) if intended.

---

_Reviewed: 2026-06-26T15:42:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
