---
phase: 28-persistent-browse-daemon
plan: "03"
subsystem: protocol-handler
tags: [browse, daemon, http, protocol-handler, lifecycle, perl]
dependency_graph:
  requires:
    - 28-01: librespot Browse daemon HTTP server (GET /track/{id}, browse_port=N)
    - 28-02: Browse::Daemon and Browse::DaemonManager lifecycle modules
  provides:
    - ProtocolHandler.pm: Browse-HTTP branches in formatOverride, canDirectStream, new, requestString, canEnhanceHTTP, getFormatForURL
    - Plugin.pm: browseMode pref, Browse daemon boot timer, shutdown hook, PID exclusion
  affects:
    - LMS playback pipeline: Browse tracks now route to Browse daemon HTTP endpoint
    - Orphan cleanup: Browse PIDs excluded from hourly kill
tech_stack:
  added: []
  patterns:
    - "formatOverride + canDirectStream must agree (Pitfall 8) — same browseMode check, same helperForClient guard"
    - "Browse sync-proxy in new() mirrors Connect sync-proxy pattern exactly"
    - "Browse PID exclusion via %browsePids hash mirrors %connectPids pattern (CON-09)"
    - "All Browse module references use require (on-demand) — not top-level use"
    - "3.5s Browse daemon boot timer — after Connect (3s) to avoid credentials.json read race"
key_files:
  created: []
  modified:
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/Plugin.pm
decisions:
  - "Episode URLs handled identically to track URLs in formatOverride and canDirectStream (both match spoton://(?:track|episode):)"
  - "Browse-HTTP branch in canDirectStream uses client->master before sync check (mirrors Connect pattern)"
  - "Browse sync-proxy in new() is inserted BEFORE Connect sync-proxy block to preserve correct flow order"
  - "browsePids uses $INC guard (same as connectPids) — Browse::DaemonManager may not be loaded if browseMode=pipe"
metrics:
  duration_minutes: 30
  completed_date: "2026-06-22"
  tasks_completed: 2
  tasks_total: 3
  files_modified: 2
---

# Phase 28 Plan 03: Integration — Browse Daemon Playback Pipeline Summary

**One-liner:** ProtocolHandler.pm dispatches Browse tracks/episodes to Browse daemon HTTP endpoint via six targeted changes; Plugin.pm wires daemon boot, shutdown, diagnosticMode restart, and orphan PID exclusion.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add Browse-HTTP branches to ProtocolHandler.pm | 74e4f83 | Plugins/SpotOn/ProtocolHandler.pm |
| 2 | Wire Browse daemon into Plugin.pm lifecycle | 2cd911a | Plugins/SpotOn/Plugin.pm |

## What Was Built

### ProtocolHandler.pm — 6 targeted changes

**1. getFormatForURL** — added `return 'pcm' if $url && $url =~ m{:\d+/track/}` so Browse HTTP URLs get the correct format identified as 'pcm'.

**2. formatOverride** — Browse-HTTP branch inserted BEFORE `return 'son'`:
- Guard: `$url !~ connect- && $url =~ spoton://(?:track|episode):`
- Reads `browseMode` pref (default 'http')
- If browseMode=http: `require Browse::DaemonManager`, get helper, check alive + _browsePort
- Returns 'pcm' — routes to existing `soc pcm * *` pipeline (direct stream passthrough)
- DIAG log: `[DIAG] formatOverride: ... result=pcm (browse_http)`

**3. canDirectStream** — Browse-HTTP branch inserted BEFORE the Connect-only early-return:
- Match: `spoton://(?:track|episode):([A-Za-z0-9]+)$`
- If browseMode=http + daemon alive: synced players return 0 (new() proxy handles), single players return `http://host:port/track/$trackId`
- DIAG log on both paths
- Consistent with formatOverride (Pitfall 8)

**4. new()** — Browse sync-proxy branch inserted BEFORE the Connect sync-proxy block:
- Match: `^spoton://(?:track|episode):([A-Za-z0-9]+)$`
- If browseMode=http + daemon port available: substitutes `$args = { %$args, url => $httpUrl }` with Browse HTTP URL
- DIAG log: `[DIAG] browse_sync_proxy: mac=... http_url=...`

**5. requestString** — extended regex from `:\d+/stream\b` to `:\d+/(?:stream\b|track/)` — Browse /track/ URLs also get plain GET without Range header.

**6. canEnhanceHTTP** — extended regex from `:\d+/stream\b` to `:\d+/(?:stream\b|track/)` — Browse /track/ URLs also return 0 (no Enhanced/Persistent HTTP reconnect loop).

### Plugin.pm — 5 targeted changes

**1. prefs->init** — `browseMode => 'http'` added with comment documenting D-06/D-07/D-08.

**2. initPlugin Browse timer** — `_startBrowseDaemons` scheduled at 3.5s (after Connect at 3s to avoid credentials.json read race).

**3. diagnosticMode restart callback** — extended to also restart Browse daemons at 1.5s delay (with `$INC` guard).

**4. _startBrowseDaemons sub** — new callback that requires Browse::DaemonManager and calls init().

**5. shutdownPlugin** — calls Browse::DaemonManager->shutdown() with `$INC` guard.

**6. _killOrphanedProcesses** — added `%browsePids` hash (same pattern as `%connectPids`), populated from Browse::DaemonManager->helperPids() when loaded, with `next if $browsePids{$pid}` in kill loop.

## Verification Results

| Check | Result |
|-------|--------|
| browseMode in ProtocolHandler.pm >= 3 | 8 occurrences |
| Browse::DaemonManager in ProtocolHandler.pm >= 3 | 6 occurrences |
| _startBrowseDaemons in Plugin.pm >= 2 | 4 occurrences |
| browsePids in Plugin.pm >= 2 | 3 occurrences |
| browseMode in Plugin.pm >= 1 | 1 occurrence |
| ProtocolHandler.pm brace balance | net +0 (matches original) |
| Plugin.pm brace balance | net +0 (matches original) |

## Deviations from Plan

### Minor scope extensions (Rule 2 — missing critical functionality)

**1. Episode URLs in canDirectStream/new()**
- **Found during:** Task 1
- **Issue:** Plan acceptance criteria explicitly requires "Browse tracks AND episodes" but the PATTERNS.md canDirectStream example only showed `^spoton://track:` regex.
- **Fix:** Used `^spoton://(?:track|episode):([A-Za-z0-9]+)$` in all Browse branches (formatOverride, canDirectStream, new()) to match both tracks and episodes.
- **Rationale:** Episodes use the same Browse daemon HTTP endpoint (`GET /track/{id}` — the Rust route handles both). Inconsistency would cause episodes to fall through to legacy `son-*` pipeline even when Browse daemon is alive.

**2. Browse client master in canDirectStream**
- **Found during:** Task 1
- **Issue:** Plan example code showed `helperForClient($client)` directly; Connect's pattern uses `$client->master` before the lookup to handle synced group master resolution.
- **Fix:** Added `my $browseClient = $client->can('master') ? $client->master : $client` before helperForClient call, matching Connect's pattern exactly (Pitfall 8 consistency).

## Known Stubs

None — all Browse-HTTP branches wire to live DaemonManager APIs that were implemented in Plan 02.

## Threat Flags

None — no new network endpoints introduced. The Browse HTTP URL construction uses `serverAddr()` (trusted LMS value) + daemon port (trusted local value) + track ID (extracted via `[A-Za-z0-9]+` regex). This is identical to Connect's URL construction and covered by T-28-08 in the plan's threat model.

## Checkpoint: Task 3 UAT Pending

Task 3 (UAT — Full Browse daemon pipeline verification) is a `checkpoint:human-verify` task that requires human testing with deployed binary. It was NOT executed in this plan run. The checkpoint details are returned to the orchestrator.

## Self-Check

Files exist:
- Plugins/SpotOn/ProtocolHandler.pm — FOUND (modified)
- Plugins/SpotOn/Plugin.pm — FOUND (modified)

Commits exist:
- 74e4f83 — FOUND (Task 1: ProtocolHandler.pm)
- 2cd911a — FOUND (Task 2: Plugin.pm)

## Self-Check: PASSED
