---
phase: 18-podcast-api-foundation
reviewed: 2026-06-14T14:32:00Z
depth: standard
files_reviewed: 5
files_reviewed_list:
  - librespot-spoton/Cargo.toml
  - librespot-spoton/src/main.rs
  - Plugins/SpotOn/API/Client.pm
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/API/TokenManager.pm
findings:
  critical: 1
  warning: 3
  info: 1
  total: 5
status: issues_found
---

# Phase 18: Code Review Report

**Reviewed:** 2026-06-14T14:32:00Z
**Depth:** standard
**Files Reviewed:** 5
**Status:** issues_found

## Summary

Phase 18 adds podcast API foundation: four new Client.pm methods (getSavedShows, getShow, getShowEpisodes, getEpisode), extended _cacheTTL rules, the user-read-playback-position OAuth scope in the librespot binary, and a synchronized cache version bump across three modules. The implementation is well-structured and follows established patterns. The new API methods correctly include input validation guards (ID regex) that the older methods lack. The _cacheTTL ordering is correct (specific episode-list rule before the general shows/ rule), and the cache version is synchronized across Plugin.pm, TokenManager.pm, and Client.pm.

One critical issue exists in the pre-existing code (start_position overflow in main.rs), and several warnings affect correctness and maintainability across the reviewed files.

## Narrative Findings (AI reviewer)

## Critical Issues

### CR-01: Floating-point to u32 cast overflow for start_position_ms

**File:** `librespot-spoton/src/main.rs:692`
**Issue:** The expression `(start_position_secs * 1000.0) as u32` performs an unchecked cast from f64 to u32. For `start_position_secs` values exceeding approximately 4,294,967 seconds (~49.7 days), the multiplication result exceeds `u32::MAX` (4,294,967,295). In Rust (since 1.45), `as u32` saturates to `u32::MAX` for out-of-range positive values, and saturates to 0 for negative values. While extreme values are unlikely during normal playback, `start_position_secs` is parsed from user-supplied CLI input with `unwrap_or(0.0)` (line 418), and a malicious or malformed caller providing `--start-position 99999999` would silently saturate. More critically, a negative value like `--start-position -5` would wrap to 0 silently (starting from the beginning instead of failing), masking a caller bug. This is reachable from the LMS transcoding pipeline via `custom-convert.conf`.
**Fix:**
```rust
// Clamp and validate before cast
if start_position_secs < 0.0 {
    return Err("start-position must be non-negative".into());
}
let start_position_ms = (start_position_secs * 1000.0).min(u32::MAX as f64) as u32;
```

## Warnings

### WR-01: _resolveStartFlavor D-03 degraded-mode path is unreachable

**File:** `Plugins/SpotOn/API/Client.pm:511`
**Issue:** The expression `($prefs->get('clientId') || SPOTON_DEFAULT_CLIENT_ID)` always evaluates to a truthy value because `SPOTON_DEFAULT_CLIENT_ID` is the hardcoded constant `'d420a117a32841c2b3474932e49fb54b'`. This means `$hasOwnId` is always 1, and the `return 'bundled'` branch on line 512 is dead code. The comment says "D-03: Degraded mode -- no own Client-ID configured -> fall back to bundled" but this condition can never be reached. When no user clientId is configured, both 'own' and 'bundled' flavors end up using `SPOTON_DEFAULT_CLIENT_ID` in TokenManager.pm (lines 383-388), making them functionally identical and defeating the dual-token routing purpose.
**Fix:** The intent of D-03 was to detect "no user-configured clientId". The fallback should be removed from the boolean check:
```perl
# D-03: Degraded mode -- no own Client-ID configured -> fall back to bundled
my $hasOwnId = $prefs->get('clientId') ? 1 : 0;
return $hasOwnId ? 'own' : 'bundled';
```

### WR-02: Pre-existing API methods lack input validation on interpolated IDs

**File:** `Plugins/SpotOn/API/Client.pm:293-294, 316-317, 385-386, 373`
**Issue:** The Phase 18 methods (getShow, getShowEpisodes, getEpisode) correctly validate their IDs with `/^[A-Za-z0-9]{1,40}$/` before URL interpolation. However, the pre-existing methods -- getArtist (line 294), getAlbum (line 317), getTrack (line 386), getArtistAlbums (line 310), getAlbumTracks (line 326), and getPlaylistItems (line 373) -- interpolate `$artistId`, `$albumId`, `$trackId`, and `$playlistId` directly into URL paths without any validation. While the Spotify API will reject malformed IDs with a 400/404 response, path traversal characters (e.g., `../`) or query string injection (e.g., `?foo=bar`) in the ID string could alter the request URL in unintended ways.
**Fix:** Apply the same validation guard pattern to all methods that interpolate IDs into URL paths:
```perl
return $cb->(undef, { error => 'invalid_id' })
    unless $artistId && $artistId =~ /^[A-Za-z0-9]{1,40}$/;
```

### WR-03: TokenManager _fetchKeymasterToken blocks event loop via backtick command

**File:** `Plugins/SpotOn/API/TokenManager.pm:394`
**Issue:** Despite the comment "WR-01: Defer via Timer -- prevents event-loop block", the 0.1-second timer deferral on line 393 only delays the blocking call; when the timer fires, `my $output = backtick($cmd)` on line 394 still blocks the LMS single-threaded event loop for the duration of the shell command execution (the comment acknowledges "~100-500ms Mercury/AP connection"). During this time, all LMS UI responses, playback events, and other plugin callbacks are stalled. This affects user experience on every token refresh cycle (every 45 minutes per account, plus on-demand fetches).
**Fix:** Use `Proc::Background` or `open()` with a non-blocking IO pattern and a timer-based poll, or use SimpleAsyncHTTP to communicate with an intermediary. The current approach is a known limitation documented in the code, but it remains a real event-loop blocking issue.

## Info

### IN-01: Inconsistent comment numbering in _request pipeline

**File:** `Plugins/SpotOn/API/Client.pm:445-493`
**Issue:** The pipeline comments reference "Step 0" through "Step 6" but skip "Step 5" entirely (jumps from Step 4 on line 482 to Step 6 on line 492). Step 5 was previously the flavor resolution step (now moved to Step 1 on line 460-462), but the comments were not renumbered after the restructuring. This creates confusion when cross-referencing the pipeline documentation.
**Fix:** Renumber Step 6 to Step 5, or add Step 5 as a no-op comment noting it was merged into Step 1.

---

_Reviewed: 2026-06-14T14:32:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
