---
phase: 19-podcast-browse
reviewed: 2026-06-14T15:30:00Z
depth: standard
files_reviewed: 4
files_reviewed_list:
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/strings.txt
  - t/02_strings.t
  - t/08_api_client.t
findings:
  critical: 1
  warning: 5
  info: 3
  total: 9
status: issues_found
---

# Phase 19: Code Review Report

**Reviewed:** 2026-06-14T15:30:00Z
**Depth:** standard
**Files Reviewed:** 4
**Status:** issues_found

## Summary

Phase 19 adds podcast browse and search to the SpotOn plugin: saved shows listing, episode feeds, podcast-specific search, and episode metadata/playback item construction. The implementation follows established patterns from the existing music browse/search code (offset pagination, _largestImage, OPML item builders). The new Client.pm API methods (getSavedShows, getShowEpisodes) are properly structured with ID validation.

Key concerns: (1) hardcoded German and English UI strings that bypass the i18n system, (2) a missing null guard on show items that mirrors an existing null-track guard for playlists, and (3) the string test not covering a significant number of keys actively used by Plugin.pm.

## Critical Issues

### CR-01: Missing null-show guard in _savedShowsFeed produces broken menu items

**File:** `Plugins/SpotOn/Plugin.pm:1031`
**Issue:** `_savedShowsFeed` maps API response items directly through `_showItem($client, $_->{show})` without filtering out null show entries. The Spotify API can return null objects in paginated lists (the same pattern that motivated the `grep { defined $_->{track} }` guard at line 1720 for playlist tracks). When `$_->{show}` is `undef`, `_showItem` receives `undef`, producing a menu entry with empty name, no showId, and no image. Clicking such an entry sends an empty showId to `getShowEpisodes`, which Client.pm rejects with an error callback, but the user sees a phantom entry they can tap.
**Fix:**
```perl
# Line 1031: add null guard, same pattern as _playlistFeed (line 1719-1721)
my @items = map  { _showItem($client, $_->{show}) }
            grep { defined $_->{show} }
            @{ $data->{items} || [] };
```

## Warnings

### WR-01: Hardcoded German strings in podcast search results bypass i18n

**File:** `Plugins/SpotOn/Plugin.pm:1249,1258`
**Issue:** Podcast search result line2 uses hardcoded German `"$showsTotal Ergebnisse"` and `"$episodesTotal Ergebnisse"`. Meanwhile, the global search at lines 1378-1405 uses hardcoded English `"$tracksTotal results"`. Both bypass `cstring()` entirely, so neither adapts to the user's language. This creates an inconsistent mixed-language UI for multilingual users.
**Fix:** Either add proper string keys to `strings.txt` with a `%s` placeholder (e.g., `PLUGIN_SPOTON_N_RESULTS` with value `%s results` / `%s Ergebnisse`), or use a neutral numeric-only format like just the count. At minimum, make both callsites consistent:
```perl
# Add to strings.txt:
# PLUGIN_SPOTON_N_RESULTS
#     DE  %s Ergebnisse
#     EN  %s results
# Then use:
line2 => cstring($client, 'PLUGIN_SPOTON_N_RESULTS', $showsTotal),
```

### WR-02: Hardcoded German date/duration strings in _formatEpisodeLine2 and _formatRelativeDate

**File:** `Plugins/SpotOn/Plugin.pm:1148-1205`
**Issue:** The episode line2 helper functions hardcode German throughout: "Min", "Std" (line 1154/1156), "Heute", "Gestern", "Vor N Tagen" (lines 1193-1195), and German month abbreviations (line 1198). A French or English LMS user sees German date labels. The codebase uses `cstring()` for all other user-visible text. These functions bypass i18n completely.
**Fix:** Move the German strings to `strings.txt` keys and pass `$client` through to these functions so `cstring()` can resolve the correct language. This is a larger refactor but is necessary for i18n consistency. At minimum, document this as a known limitation if German-only display is intentional for v1.

### WR-03: Leading-zero day in formatted dates deviates from German convention

**File:** `Plugins/SpotOn/Plugin.pm:1199,1202,1204`
**Issue:** `$day` is captured from the regex as a zero-padded string (e.g., "03"). When interpolated in the output at lines 1202/1204, it produces "03. Jun" instead of the conventional German format "3. Jun". The `$day` variable is never coerced to a number before string interpolation.
**Fix:**
```perl
# After line 1178, add numeric coercion:
my ($year, $month, $day) = ($1, $2, $3);
$day   += 0;  # Strip leading zero for display ("03" -> 3)
$month += 0;
```

### WR-04: String test does not verify 18 keys actively used by Plugin.pm

**File:** `t/02_strings.t:25-67`
**Issue:** The `@bilingual_keys` array in the test omits 18 string keys that are actively used via `cstring()` in Plugin.pm. Missing keys include core navigation strings: `PLUGIN_SPOTON_HOME`, `PLUGIN_SPOTON_SEARCH`, `PLUGIN_SPOTON_LIBRARY`, `PLUGIN_SPOTON_ALBUMS`, `PLUGIN_SPOTON_ARTISTS`, `PLUGIN_SPOTON_PLAYLISTS`, `PLUGIN_SPOTON_LIKED_SONGS`, `PLUGIN_SPOTON_RECENTLY_PLAYED`, `PLUGIN_SPOTON_MADE_FOR_YOU`, `PLUGIN_SPOTON_TOP_TRACKS`, `PLUGIN_SPOTON_NO_RESULTS`, `PLUGIN_SPOTON_TRACKS`, `PLUGIN_SPOTON_TOP_RESULT`, `PLUGIN_SPOTON_SINGLES`, `PLUGIN_SPOTON_COMPILATIONS`, `PLUGIN_SPOTON_APPEARS_ON`, `PLUGIN_SPOTON_ARTIST_VIEW`, `PLUGIN_SPOTON_ALBUM_VIEW`. If any of these keys were accidentally removed from `strings.txt`, the test would not detect the regression.
**Fix:** Add all 18 keys to the `@bilingual_keys` array in `t/02_strings.t`.

### WR-05: Inconsistent line2 language between global search (English) and podcast search (German)

**File:** `Plugins/SpotOn/Plugin.pm:1378-1405 vs 1249-1258`
**Issue:** Global search shows `"$tracksTotal results"` (English, line 1378) while podcast search shows `"$showsTotal Ergebnisse"` (German, line 1249). A user navigating both search areas sees mixed languages in the same UI. Even if full i18n is deferred, the two code paths should at least agree on one language.
**Fix:** Unify both to use the same approach -- either both use `cstring()` with a localized key, or both use the same hardcoded language. Given that the codebase is German-developer-oriented and the date functions use German, using German consistently would be the minimal fix until proper i18n is added.

## Info

### IN-01: Autovivification side-effect in search response data access

**File:** `Plugins/SpotOn/Plugin.pm:1240-1241`
**Issue:** `$data->{shows}{total}` autovivifies `$data->{shows}` to `{}` if the shows key is absent from the API response. Same pattern at lines 1367-1370 for global search. This silently mutates the response hash. Not a correctness bug (the `// 0` guard handles the value), but could mask issues in debugging.
**Fix:** Use intermediate variable: `my $showsTotal = ($data->{shows} && $data->{shows}{total}) // 0;`

### IN-02: No test coverage for new podcast API methods

**File:** `t/08_api_client.t`
**Issue:** The test file covers core API client functionality (getMe, concurrency, rate limiting, library CRUD) but has no tests for the new podcast-related API methods: `getSavedShows`, `getShowEpisodes`, `getShow`, `getEpisode`. While the existing tests validate the underlying `_request` infrastructure, endpoint-specific tests would verify correct path construction and parameter mapping.
**Fix:** Add basic smoke tests for `getSavedShows` and `getShowEpisodes` following the same pattern as the existing `getMe` test (verify URL path and method).

### IN-03: Episode show name metadata may be empty for show-list episodes

**File:** `Plugins/SpotOn/Plugin.pm:1118`
**Issue:** In `_episodeItem`, the metadata cache sets `artist => $episode->{show}{name} // ''`. When called from `_showFeed` (line 1079), episodes from `/shows/{id}/episodes` are simplified objects that may lack the nested `show` key entirely. This produces an empty artist field in NowPlaying metadata. The show name is available in the passthrough (`showImages` is passed, but show name is not).
**Fix:** Pass the show name through passthrough and use it as fallback:
```perl
# In _showFeed, pass showName in the map call:
my @items = map { _episodeItem($client, $_, $showImages, $showName) } @{$data->{items} || []};
```

---

_Reviewed: 2026-06-14T15:30:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
