---
phase: 05-spotify-connect
plan: "02"
subsystem: connect
tags: [daemon, process-wrapper, librespot, transcoding, crash-backoff]
dependency_graph:
  requires:
    - "05-01 (Helper.pm binary path)"
  provides:
    - "Plugins::SpotOn::Connect::Daemon — process wrapper"
    - "soc content type and transcoding profiles"
  affects:
    - "Plugins/SpotOn/ProtocolHandler.pm (soc format routing — Phase 5 plan 04)"
    - "Plugins/SpotOn/Plugin.pm (_killOrphanedProcesses — Phase 5 plan 04)"
tech_stack:
  added:
    - "Proc::Background (LMS-bundled) — child process lifecycle management"
    - "IO::Select — non-blocking port read with 5s timeout"
    - "MIME::Base64 — encode_base64 for LMS auth credentials"
  patterns:
    - "pipe() + IO::Select for synchronous port capture without SIGALRM"
    - "Slim::Utils::Accessor mk_accessor for OO state management"
    - "Per-account cache dir: cachedir/spoton/<accountId>"
key_files:
  created:
    - "Plugins/SpotOn/Connect/Daemon.pm"
  modified:
    - "Plugins/SpotOn/custom-convert.conf"
    - "Plugins/SpotOn/custom-types.conf"
decisions:
  - "Use require Proc::Background inside start() (on-demand) rather than top-level use, consistent with LMS lazy-load pattern"
  - "No rmtree on stop() — SpotOn keeps credentials across restarts (contrast: Spotty-NG deletes cache on stop)"
  - "Single code path in start() — always uses --connect + port capture (Spotty-NG has two paths: stream vs legacy)"
metrics:
  duration: "~15 minutes"
  completed: "2026-06-01"
  tasks_completed: 2
  files_created: 1
  files_modified: 2
---

# Phase 05 Plan 02: Connect Daemon.pm Process Wrapper Summary

**One-liner:** Daemon.pm wraps librespot `--connect` via Proc::Background with pipe/IO::Select port capture, crash backoff, and sync-group device naming; soc PCM/OGG passthrough profiles registered for LMS transcoding pipeline.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Config files — soc transcoding profiles and content type | 1f3f10b | custom-convert.conf, custom-types.conf |
| 2 | Create Daemon.pm process wrapper | 9a9ec0d | Connect/Daemon.pm (new) |

## What Was Built

### Task 1: Config Files

`custom-convert.conf` now has two `soc` profiles appended after the `son ogg` entry:

- `soc pcm * *` with `# I` flag (LMS input passthrough — no binary invocation, LMS fetches via canDirectStream or new() proxy)
- `soc ogg * *` with `# I` flag (same passthrough semantics, OGG format)

All 4 existing `son-*` profiles (pcm, flc, mp3, ogg) remain untouched.

`custom-types.conf` now has the soc content type:
- `soc    soc    audio/x-sb-spoton-connect    audio`

### Task 2: Daemon.pm

`Plugins/SpotOn/Connect/Daemon.pm` implements the full process wrapper:

- **Package:** `Plugins::SpotOn::Connect::Daemon` extending `Slim::Utils::Accessor`
- **Accessors:** id, mac, name, cache, _lastSeen, _spotifyId, _proc, _startTimes, _streamStartTimes, _streamMode, _streamPort
- **new():** Strips colons from MAC for id, initializes start time arrays, calls start()
- **start():** Lazy-loads Proc::Background; builds device name via syncname() or client->name, truncated to 60 chars; constructs per-account cache dir; spawns binary with `--connect` flag; pipe + IO::Select 5s timeout for port capture; sets _streamPort and _streamMode(1)
- **_checkStartTimes():** Crash backoff — disables discovery after MAX_FAILURES_BEFORE_DISABLE_DISCOVERY (3) crashes within MAX_INTERVAL_BEFORE_DISABLE_DISCOVERY (300s)
- **stop():** Kills process, clears _streamPort (no cache cleanup — credentials kept)
- **stopForSync():** Kills process, clears _streamPort, resets both start-time arrays
- **pid/alive/uptime:** Thin wrappers over Proc::Background state

## Security (Threat Model)

| Threat | Mitigation Applied |
|--------|-------------------|
| T-05-07: Device name tampering | substr() at 60 chars; syncname() returns LMS-controlled value |
| T-05-08: Credentials in logs | Binary args logged BEFORE --lms-auth added to @helperArgs |
| T-05-09: Cache dir injection | Cache dir = serverPrefs cachedir + fixed 'spoton' suffix, no user input |
| T-05-SC: Proc::Background integrity | LMS-bundled module, no external install needed |

## Deviations from Plan

None — plan executed exactly as written.

The SpotOn deviations from Spotty-NG specified in PATTERNS.md were applied correctly:
- CLI flag `--connect` (not `--connect-stream`)
- No `rmtree` in `stop()`
- `plugin.spoton` prefs namespace and `spoton` cache subdir
- `require Proc::Background` inside `start()` (on-demand loading)
- Single code path (no stream-backoff branch in this plan — _checkStreamStartTimes is present for future use)

## Known Stubs

None. Daemon.pm is a complete process wrapper. The _spotifyId accessor and spotifyId()/spotifyIdIsRecent() methods from the Spotty-NG reference were intentionally omitted — they are used by Connect.pm (Plan 03) and can be added there or in a later plan when the Connect event dispatcher is wired.

## Self-Check: PASSED
