---
phase: 06-polish-release-readiness
plan: "02"
subsystem: settings-transcoding
tags: [per-player-prefs, bitrate-override, stream-format, transcoding, protocol-handler]
dependency_graph:
  requires: []
  provides:
    - per-player bitrateOverride pref (96/160/320 or empty for global)
    - per-player streamFormat pref (auto/ogg/pcm/flac/mp3)
    - updateTranscodingTable with per-player bitrate injection
    - canDirectStream returns 0 for flac/mp3/pcm format selection
    - formatOverride returns 'ogg' for Browse OGG passthrough mode
  affects:
    - Plugins/SpotOn/Settings.pm
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
    - Plugins/SpotOn/strings.txt
tech_stack:
  added: []
  patterns:
    - per-player pref read with migration fallback (streamFormat || connectOggOverride || 'auto')
    - input validation before pref storage (T-06-03, T-06-04)
    - re-validation in updateTranscodingTable before bitrate regex injection (T-06-05)
    - canDirectStream force-transcoding gate for pcm/flac/mp3 formats
key_files:
  created: []
  modified:
    - Plugins/SpotOn/Settings.pm
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
    - Plugins/SpotOn/strings.txt
decisions:
  - streamFormat replaces connectOggOverride as active pref; connectOggOverride retained for backward compatibility and migration fallback
  - per-player prefs use client() namespace; no global $prefs->init() needed (lazy initialization)
  - canDirectStream block-scopes the streamFormat read to avoid variable leakage into existing Connect logic
metrics:
  duration: "~25 minutes"
  completed: "2026-06-03"
  tasks_completed: 2
  files_modified: 5
---

# Phase 06 Plan 02: Per-Player Preferences and Transcoding Engine Summary

Per-player bitrate override and unified 5-option Format-Dropdown (auto/ogg/pcm/flac/mp3) for both Connect and Browse modes. FLAC/MP3 selections force transcoding via custom-convert.conf pipeline.

## Tasks Completed

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 | Per-player pref handling (Settings.pm) + UI (basic.html) + strings | 629bc4b | Settings.pm, basic.html, strings.txt |
| 2 | Transcoding engine + ProtocolHandler per-player format support | 4bba7ad | Plugin.pm, ProtocolHandler.pm |

## What Was Built

### Task 1: Settings UI + Pref Handling

**Settings.pm:**
- Added `streamFormat` pref save with validation regex `^(?:auto|ogg|pcm|flac|mp3)$` (T-06-04)
- Added `bitrateOverride` pref save with validation regex `^(?:96|160|320)$` or empty string (T-06-03)
- Retained existing `connectOggOverride` save for backward compatibility
- Template vars: `$paramRef->{streamFormat}` reads new key with `connectOggOverride` fallback for migration
- Template vars: `$paramRef->{bitrateOverride}` reads per-player override (empty = global)

**basic.html:**
- Replaced 3-option `connectOggOverride` dropdown with 5-option `streamFormat` dropdown (auto/ogg/pcm/flac/mp3)
- Added `bitrateOverride` dropdown with 4 options: Global (default), 320, 160, 96 kbps

**strings.txt:**
- Added 10 new keys (EN+DE): `PLUGIN_SPOTON_STREAM_FORMAT`, `_DESC`, `_AUTO`, `_OGG`, `_PCM`, `_FLAC`, `_MP3`, `PLUGIN_SPOTON_BITRATE_OVERRIDE`, `_DESC`, `PLUGIN_SPOTON_BITRATE_GLOBAL`

### Task 2: Transcoding Engine + Protocol Handler

**Plugin.pm (updateTranscodingTable):**
- Per-player bitrate override: reads `bitrateOverride` pref, re-validates with `^(?:96|160|320)$` before `--bitrate` regex injection (T-06-05 threat mitigation)
- Replaced `connectOggOverride` block with `streamFormat` logic: deletes `son-ogg-*-*` and `soc-ogg-*-*` when `streamFormat != 'ogg'`; migration fallback reads `connectOggOverride` if `streamFormat` is empty

**ProtocolHandler.pm (formatOverride):**
- Reads per-player `streamFormat` pref with `connectOggOverride` fallback
- Browse mode: returns `'ogg'` if `streamFormat eq 'ogg'` (OGG passthrough), `'son'` otherwise
- Connect mode: always returns `'soc'` regardless of `streamFormat` (D-12)

**ProtocolHandler.pm (canDirectStream):**
- Added streamFormat gate: returns 0 when `streamFormat` is `pcm`, `flac`, or `mp3` — forces LMS to use custom-convert.conf pipeline (D-11)
- Migration fallback: `streamFormat || connectOggOverride || 'auto'`

**D-02 verified:** Daemon.pm line 120 already passes `--enable-volume-normalisation` to Connect daemons when `normalization` pref is set. No change needed.

## Must-Have Verification

| Truth | Status |
|-------|--------|
| D-01: Bitrate 96 on Player A / 320 on Player B streams at configured bitrate | Code-verified: bitrateOverride in updateTranscodingTable applies per-player |
| D-11: Format-Dropdown with 5 options visible per player | Implemented: basic.html pref_streamFormat with auto/ogg/pcm/flac/mp3 |
| D-03: Per-player settings in single section with player dropdown | Existing structure maintained; new dropdowns added to player section |
| FLAC/MP3 forces transcoding | canDirectStream returns 0 for flac/mp3/pcm |
| D-12: Format-Dropdown applies to Connect and Browse | formatOverride reads streamFormat; updateTranscodingTable applies to both son-* and soc-ogg entries |
| D-02: Volume normalisation for Connect daemons | Daemon.pm line 120 verified — no change needed |

## Deviations from Plan

None — plan executed exactly as written.

## Threat Model Compliance

| Threat | Mitigation | Status |
|--------|-----------|--------|
| T-06-03: bitrateOverride tampering | Validated with `^(?:96|160|320)$/` in Settings.pm; re-validated in Plugin.pm before regex injection | Implemented |
| T-06-04: streamFormat tampering | Validated with `^(?:auto|ogg|pcm|flac|mp3)$/` before storage | Implemented |
| T-06-05: bitrate injection in commandTable | bitrateOverride re-validated in updateTranscodingTable before `s/--bitrate \d+/--bitrate $bitrate/` | Implemented |

## Known Stubs

None. All prefs are wired to real data sources.

## Threat Flags

None. No new network endpoints, auth paths, or schema changes introduced.

## Self-Check: PASSED

- `Plugins/SpotOn/Settings.pm` — modified, contains bitrateOverride + streamFormat handling
- `Plugins/SpotOn/Plugin.pm` — modified, contains bitrateOverride in updateTranscodingTable
- `Plugins/SpotOn/ProtocolHandler.pm` — modified, contains streamFormat in canDirectStream + formatOverride
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — modified, contains pref_streamFormat + pref_bitrateOverride
- `Plugins/SpotOn/strings.txt` — modified, contains 10 new keys
- Commit 629bc4b — Task 1
- Commit 4bba7ad — Task 2
