---
phase: 12-protocol-handler-rename
plan: "01"
subsystem: protocol-handler
tags: [protocol, rename, cache, rust, tdd]
dependency_graph:
  requires: []
  provides: [spoton-protocol-handler, named-cache-namespace, rust-normalization]
  affects: [Plugin.pm, ProtocolHandler.pm, Connect.pm, DontStopTheMusic.pm, API/Client.pm, API/TokenManager.pm, librespot-spoton/src/main.rs]
tech_stack:
  added: []
  patterns: [named-cache-namespace, cacheSchemaVersion-pref-guard]
key_files:
  created:
    - t/12_protocol_rename.t
  modified:
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/Connect.pm
    - Plugins/SpotOn/DontStopTheMusic.pm
    - Plugins/SpotOn/API/Client.pm
    - Plugins/SpotOn/API/TokenManager.pm
    - librespot-spoton/src/main.rs
    - t/11_track_history.t
decisions:
  - "Named cache namespace 'spoton' with version 2 across all 6 modules (D-01/D-02)"
  - "SPOTON_CACHE_VERSION constant in Plugin.pm, literal 2 in sub-modules"
  - "cacheSchemaVersion pref guard logs migration but actual flush is by namespace version bump"
  - "Multi-line registerHandler() pattern requires content-level grep in tests (not line-by-line)"
metrics:
  duration: "10m"
  completed: "2026-06-05"
  tasks_completed: 3
  tasks_total: 3
  files_modified: 8
  files_created: 1
---

# Phase 12 Plan 01: Protocol Handler Rename Summary

**One-liner:** Renamed LMS routing URLs from `spotify://` to `spoton://` across all Perl sources and Rust binary, with named cache namespace `spoton` version 2 and cacheSchemaVersion pref guard.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Create t/12_protocol_rename.t test scaffold (RED) | 1d722f4 | t/12_protocol_rename.t |
| 2 | Rename spotify:// to spoton:// in all Perl + Rust + cache namespace | 512b22f | Plugin.pm, ProtocolHandler.pm, Connect.pm, DontStopTheMusic.pm, API/Client.pm, API/TokenManager.pm, librespot-spoton/src/main.rs, t/12_protocol_rename.t |
| 3 | Update t/11_track_history.t mock URLs | d957612 | t/11_track_history.t |

## Changes Summary

### Plugin.pm
- Added `use constant SPOTON_CACHE_VERSION => 2`
- Changed `Slim::Utils::Cache->new()` to `Slim::Utils::Cache->new('spoton', SPOTON_CACHE_VERSION)`
- Added `cacheSchemaVersion => 0` to `$prefs->init({})` block
- Added version guard after prefs init: if `cacheSchemaVersion < SPOTON_CACHE_VERSION`, log migration and update pref
- Changed `registerHandler('spotify', ...)` to `registerHandler('spoton', ...)`
- Changed `m{^spotify://}` to `m{^spoton://}` in `_killOrphanedProcesses`
- Renamed `$spotify_url` to `$spoton_url` (both occurrences in `_trackItem` and `_albumTrackItem`)

### ProtocolHandler.pm
- Named cache namespace: `Cache->new('spoton', 2)`
- All 25 `spotify://` occurrences in regex/string contexts replaced with `spoton://`
- Critical: canonical normalization (Pitfall 3) updated: `s{^spoton:}{spoton://}`
- D-06 respected: lines 266, 351, 431 (`spotify:track:` API URIs) unchanged

### Connect.pm
- Named cache namespace: `Cache->new('spoton', 2)`
- All `spotify://` in URL constructions and regex patterns replaced with `spoton://`
- 3 `sprintf("spoton://connect-%u", $ts)` constructions (lines ~630, ~722, ~828)
- D-06 respected: `spotify:track:$trackId` API URI assignments (lines ~624, ~714, ~771) unchanged

### DontStopTheMusic.pm
- Named cache namespace: `Cache->new('spoton', 2)`
- `"spoton://$1"` URL construction

### API/Client.pm, API/TokenManager.pm
- Named cache namespace: `Cache->new('spoton', 2)` (no URL pattern changes needed)

### librespot-spoton/src/main.rs
- Normalization: `track_uri.replace("spoton://", "spotify:")` (was `"spotify://"`)

### t/12_protocol_rename.t (new file)
- Grep-based validation for PROTO-01 through PROTO-06
- Multi-line content matching for `registerHandler` (PROTO-02, PROTO-05)
- Rust binary normalization check
- Cache namespace check (6 named, 0 no-args)

### t/11_track_history.t
- 9 mock URLs updated from `spotify://` to `spoton://`
- `spotify:track:` API URIs left unchanged

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] PROTO-02/05 test regex failed on multi-line registerHandler**
- **Found during:** Task 2 verification
- **Issue:** Test used line-by-line grep `registerHandler\s*\(\s*['"]spoton['"]` but the actual code spans two lines (`registerHandler(` on one line, `'spoton'` on the next)
- **Fix:** Updated PROTO-02 and PROTO-05 test sections to read whole file content and use multi-line regex with `\n?` between `(` and scheme string
- **Files modified:** t/12_protocol_rename.t
- **Commit:** 512b22f (part of Task 2 commit — test was already staged)

## Deferred Issues

The following test failures were pre-existing before this phase (verified via `git stash` to baseline):
- `t/07_token_manager.t`: 4 failing tests (AUTH-02, getToken cache) — related to a `::own` subroutine issue in a temp stub
- `t/08_api_client.t`: parse error (no plan) — stub loading issue
- `t/09_settings.t`: 1 failing test (clientId reference)

These failures are out of scope for Phase 12. They were present on the baseline commit `ac0993a` and are unaffected by this plan's changes.

## Verification Results

```
prove t/12_protocol_rename.t     # 16/16 PASS (PROTO-01 through PROTO-06 + Rust + cache)
prove t/11_track_history.t       # 10/10 subtests PASS
prove t/05_perl_syntax.t         # PASS (no Perl syntax errors)
prove t/03_convert_conf.t        # PASS (custom-convert.conf unchanged, PROTO-03)
```

### Final acceptance checks
- `grep -rn 'spotify://' Plugins/SpotOn/*.pm API/*.pm | grep -v '#'` — 0 matches
- `grep -c 'replace("spoton://"' librespot-spoton/src/main.rs` — 1
- `grep -c 'Cache->new()' all 6 modules` — 0
- `grep -c "Cache->new('spoton'" all 6 modules` — 6
- `grep -c 'SPOTON_CACHE_VERSION' Plugin.pm` — 4
- `grep -c 'cacheSchemaVersion' Plugin.pm` — 4
- Spotify API URIs (`spotify:track:ID`) unchanged in ProtocolHandler.pm lines 266, 351, 431 and Connect.pm

## Self-Check: PASSED

Files exist:
- FOUND: .planning/phases/12-protocol-handler-rename/12-01-SUMMARY.md
- FOUND: t/12_protocol_rename.t
- FOUND: Plugins/SpotOn/Plugin.pm (modified)
- FOUND: librespot-spoton/src/main.rs (modified)

Commits exist:
- FOUND: 1d722f4 (test scaffold RED state)
- FOUND: 512b22f (rename + GREEN state)
- FOUND: d957612 (t/11 mock URLs updated)
