---
phase: 11-track-history-metadata
plan: "02"
subsystem: protocol-handler
tags:
  - async-refetch
  - debounce
  - connect-to-browse
  - placeholder
  - history
dependency_graph:
  requires:
    - "Phase 11 Plan 01 — spotifyUri in Connect cache, 7-day TTL"
    - "API::Client->getTrack callback pattern"
  provides:
    - "Async re-fetch on getMetadataFor cache miss (D-03)"
    - "Placeholder metadata instead of empty hashref (D-03)"
    - "Connect-to-Browse URL translation in history (D-06)"
    - "Browse mode label for translated Connect tracks (D-07)"
    - "Per-URL debounce via %_pendingRefetch (D-05)"
  affects:
    - "Plugins/SpotOn/ProtocolHandler.pm"
    - "t/11_track_history.t"
tech_stack:
  added: []
  patterns:
    - "our %_pendingRefetch — package-scoped debounce hash for one in-flight re-fetch per URL"
    - "_asyncRefetch: require API::Client + getTrack callback + cache set + notifyFromArray"
    - "_placeholderMeta: cover=/html/images/cover.png, title='Loading...' for track URLs"
    - "Connect-to-Browse translation: spotifyUri -> spotify://track:ID + _typeString Browse"
key_files:
  created: []
  modified:
    - path: Plugins/SpotOn/ProtocolHandler.pm
      change: "Add %_pendingRefetch, Connect history translation block, async re-fetch + placeholder, _placeholderMeta/_asyncRefetch/_largestImage subs"
    - path: t/11_track_history.t
      change: "Remove TODO blocks F/G, add real tests H/I/J for debounce, async re-fetch, D-07 label"
decisions:
  - "Used 'our' (not 'my') for %_pendingRefetch so test can access it as package variable"
  - "Added 'require Slim::Control::Request' before notifyFromArray call to handle lazy load"
  - "Pre-load Plugins::SpotOn::API::Client and Slim::Control::Request in test setup to prevent require() overriding local typeglob redefinitions"
  - "_largestImage defined locally in ProtocolHandler.pm — avoids cross-module import from Connect.pm (Pitfall 1)"
  - "Connect re-fetch stores under Browse URL key (spotify://track:ID) not Connect URL key (Pitfall 3)"
  - "delete $_pendingRefetch{$url} is first action in callback — always fires even on error (Pitfall 4)"
metrics:
  duration: "4 minutes"
  completed: "2026-06-04T16:03:11Z"
  tasks_completed: 1
  files_modified: 2
  files_created: 0
---

# Phase 11 Plan 02: Track History Metadata — Async Re-fetch, Debounce, Connect-to-Browse Summary

**One-liner:** Replaced empty-hashref cache miss with async API re-fetch + placeholder, and added Connect-to-Browse URL translation with Browse mode label for history tracks.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Async re-fetch, debounce, placeholder, Connect-to-Browse translation | 09d668d | ProtocolHandler.pm, t/11_track_history.t |
| 2 | Live LMS verification (checkpoint:human-verify, gate=non-blocking) | — | — |

## What Was Built

**Task 1 — ProtocolHandler.pm + test upgrades (09d668d)**

Four additions to `ProtocolHandler.pm`:

**A. Debounce hash (D-05)**
Added `our %_pendingRefetch` after the `$cache` declaration. `our` (not `my`) so test code can access it as `$Plugins::SpotOn::ProtocolHandler::_pendingRefetch{...}`.

**B. Connect history URL translation (D-06, D-07)**
New block in `getMetadataFor` between the pluginData block and canonical normalization:
- Detects `spotify://connect-` URLs (history items — no active playingSong)
- Reads `spotifyUri` from cache entry, validates as `spotify:track:[A-Za-z0-9]+`
- Constructs Browse URL: `spotify://track:$trackId`
- Returns metadata with `_typeString($client, 'Browse')` — not Connect (D-07)
- Returns early on cache hit, falls through to async re-fetch on miss

**C. Cache miss → placeholder + async re-fetch (D-03, D-04, D-05)**
Replaced `return {} unless $meta;` with:
```perl
unless ($meta) {
    _asyncRefetch($class, $client, $url, $canonical);
    return _placeholderMeta($url);
}
```

**D. Three new private subs**

- `_placeholderMeta($url)`: Returns `{ cover => '/html/images/cover.png', icon => ..., title => 'Loading...' }` for track URLs, empty title otherwise.
- `_asyncRefetch($class, $client, $url, $canonical)`: Debounce check → extract track ID (Browse regex or Connect cached spotifyUri) → resolve accountId → set flag → `require Plugins::SpotOn::API::Client` → `getTrack` callback → delete flag (first!) → build metadata → `$cache->set` under Browse URL key → `require Slim::Control::Request` + `notifyFromArray`.
- `_largestImage($images_arrayref)`: Copy from Connect.pm — sorts images by width, returns largest URL.

**Test upgrades (t/11_track_history.t)**
- Removed TODO blocks from F and G subtests
- Test F: placeholder cover is `/html/images/cover.png`, hash is non-empty
- Test G: Connect URL with spotifyUri returns `play` field with `spotify://track:XYZ789`
- Test H: debounce — pre-loads `API::Client` stub, sets `%_pendingRefetch` flag, verifies `getTrack` not called
- Test I: async re-fetch — sets `$mock_track`, calls `getMetadataFor`, asserts cache populated and `notifyFromArray` fired
- Test J: D-07 label — type field contains "Browse" not "Connect"

## Verification Results

```
perl t/05_perl_syntax.t           — 6/6 PASS
perl t/11_track_history.t         — 11/11 subtests PASS (all TODO removed)
grep -c '_asyncRefetch' ...       — 3 (definition + call + comment)
grep -c '_placeholderMeta' ...    — 3 (definition + call + comment)
grep -c '%_pendingRefetch' ...    — 2
grep -c '_largestImage' ...       — 3 (definition + 2 calls)
grep -c 'return {} unless \$meta' — 0 (pattern fully replaced)
grep -c 'spotifyUri' ...          — 8
```

## Checkpoint: Human Verification Pending

Task 2 is a `checkpoint:human-verify` (non-blocking). Live LMS verification steps:
1. Deploy updated `Plugins/SpotOn/` to LMS plugin dir, restart LMS
2. Play a track via Browse menu → check Track History shows artwork + metadata
3. Play a track via Spotify Connect → check Track History shows Connect track artwork
4. Wait >1 hour (or clear cache entry), revisit history → verify "Loading..." placeholder then re-fetched metadata
5. Click a former Connect track in history → verify playback via Browse pipeline

## Deviations from Plan

### Auto-fix: `our` instead of `my` for %_pendingRefetch

- **Found during:** Test H implementation
- **Issue:** Plan specified `my %_pendingRefetch` but tests need package-variable access as `$Plugins::SpotOn::ProtocolHandler::_pendingRefetch{...}`. With `my`, this scoped lexical is inaccessible from outside.
- **Fix:** Changed to `our %_pendingRefetch` — semantically equivalent for all production code but accessible for test verification.
- **Files modified:** Plugins/SpotOn/ProtocolHandler.pm (one-word change)

### Auto-fix: `require Slim::Control::Request` before notifyFromArray

- **Found during:** Test I execution
- **Issue:** `notifyFromArray` is called as a package function but the module is never `use`d or `require`d in ProtocolHandler.pm — in tests, the stub isn't loaded until the first `require` in _asyncRefetch.
- **Fix:** Added `require Slim::Control::Request;` immediately before the `notifyFromArray` call in `_asyncRefetch`.
- **Files modified:** Plugins/SpotOn/ProtocolHandler.pm

### Auto-fix: Pre-load stubs in test setup

- **Found during:** Test H (debounce) — `local *...` override was silently overridden by `require` inside `_asyncRefetch`
- **Issue:** `_asyncRefetch` uses `require Plugins::SpotOn::API::Client` which re-executes the stub file and resets the typeglob, defeating `local *Plugins::SpotOn::API::Client::getTrack = sub {...}`.
- **Fix:** Added `require Plugins::SpotOn::API::Client; require Slim::Control::Request;` to test setup before `require_ok('Plugins::SpotOn::ProtocolHandler')`. Pre-loading ensures `require` is a no-op inside production code.
- **Files modified:** t/11_track_history.t

## Known Stubs

None — no UI rendering stubs or placeholder data introduced. `_placeholderMeta` is intentional transient state (displayed briefly during re-fetch), not a stub.

## Threat Flags

None — no new network endpoints, auth paths, or trust boundary changes beyond those covered in the plan's threat model (T-11-03 through T-11-05).

## Self-Check: PASSED

| Item | Status |
|------|--------|
| Plugins/SpotOn/ProtocolHandler.pm (modified) | FOUND |
| t/11_track_history.t (modified) | FOUND |
| .planning/phases/11-track-history-metadata/11-02-SUMMARY.md | FOUND |
| Commit 09d668d (feat(11-02): async re-fetch...) | FOUND |
