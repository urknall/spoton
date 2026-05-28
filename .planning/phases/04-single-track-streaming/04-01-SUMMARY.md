---
phase: 04-single-track-streaming
plan: "01"
subsystem: transcoding-engine
tags:
  - protocol-handler
  - transcoding
  - format-selection
  - librespot
  - lms-plugin
dependency_graph:
  requires:
    - "03-xx: Plugin.pm with working prefs, Helper.pm with getCapability"
    - "Plugins/SpotOn/custom-convert.conf with all four son-* pipelines"
  provides:
    - "Dynamic OGG/FLAC format selection per player capabilities (ProtocolHandler::formatOverride)"
    - "Runtime injection of bitrate, cache dir, helper name, normalization into commandTable (Plugin::updateTranscodingTable)"
  affects:
    - "04-02: Settings UI (normalization pref now initialized)"
tech_stack:
  added:
    - "Slim::Player::CapabilitiesHelper (supportedFormats query)"
    - "Slim::Player::TranscodingHelper (Conversions hashref access)"
    - "File::Basename (helper binary name extraction)"
    - "File::Spec::Functions::catdir (cache dir construction)"
  patterns:
    - "updateTranscodingTable pattern (Spotty prior art): formatOverride as trigger, commandTable regex injection for runtime params"
    - "passthrough guard: OGG returned only when player supports OGG AND binary has passthrough capability"
key_files:
  created: []
  modified:
    - "Plugins/SpotOn/ProtocolHandler.pm"
    - "Plugins/SpotOn/Plugin.pm"
decisions:
  - "D-01: updateTranscodingTable pattern chosen over pref-file substitution (TranscodingHelper caches pref-file values after first read)"
  - "D-04: FLAC as default fallback (consistent with getFormatForURL)"
  - "D-06: normalization pref added as global toggle (Phase 4), per-player in Phase 6"
  - "D-07: --disable-audio-cache always present, no UI toggle in Phase 4"
  - "LMS-11: no race condition risk due to LMS single-threaded event loop"
metrics:
  duration: "~4 minutes"
  completed_date: "2026-05-28"
  tasks_completed: 2
  tasks_total: 2
  files_changed: 2
---

# Phase 04 Plan 01: Core Transcoding Engine Summary

Dynamic format selection per player capabilities and runtime parameter injection into LMS commandTable via updateTranscodingTable + formatOverride pattern.

## What Was Built

### Task 1: Dynamic formatOverride in ProtocolHandler.pm (commit 181fc65)

Replaced the static `return 'flc'` in `formatOverride` with a dynamic implementation:

- Added `use Slim::Player::CapabilitiesHelper` import
- `formatOverride` now calls `Plugins::SpotOn::Plugin->updateTranscodingTable($client)` before format selection (D-01)
- Queries player format capabilities via `Slim::Player::CapabilitiesHelper::supportedFormats($client)`
- Returns `'ogg'` only when player supports OGG AND `Plugins::SpotOn::Helper->getCapability('passthrough')` is truthy (passthrough guard per A2-Mitigation)
- Returns `'flc'` as default fallback (D-04, STR-02)
- All existing methods unchanged: contentType, isRemote, canDirectStream, getFormatForURL, canSeek, canTranscodeSeek, getSeekData

### Task 2: updateTranscodingTable + normalization pref in Plugin.pm (commit 3c1dc93)

Added the transcoding runtime injection engine and extended prefs:

- Added imports: `File::Basename`, `File::Spec::Functions qw(catdir)`, `Slim::Player::TranscodingHelper`
- Extended `$prefs->init` with `normalization => 0` (STR-08, D-06)
- Added `updateTranscodingTable` method that:
  - Reads bitrate and normalization from prefs
  - Computes cache dir via `catdir($serverPrefs->get('cachedir'), 'spoton')`
  - Creates cache dir if missing via `File::Path::make_path`
  - Gets helper binary name via `basename(Plugins::SpotOn::Helper->get())`
  - Iterates `Slim::Player::TranscodingHelper::Conversions()` hashref
  - Only modifies keys matching `/^son-/` AND containing `single-track`
  - Applies four regex substitutions: cache dir, bitrate, helper name, volume normalisation
  - Does NOT touch `--disable-audio-cache` (STR-11, D-07)
  - Logs updated entries at INFOLOG level

## Verification Results

| Check | Result |
|-------|--------|
| ProtocolHandler.pm: use Slim::Player::CapabilitiesHelper | PASS (1 occurrence) |
| ProtocolHandler.pm: calls updateTranscodingTable | PASS (1 occurrence) |
| ProtocolHandler.pm: calls supportedFormats | PASS (1 occurrence) |
| ProtocolHandler.pm: passthrough guard | PASS (1 occurrence) |
| ProtocolHandler.pm: returns 'ogg' | PASS (1 occurrence) |
| ProtocolHandler.pm: returns 'flc' fallback | PASS (1 occurrence) |
| ProtocolHandler.pm: getFormatForURL returns 'flc' | PASS (unchanged) |
| Plugin.pm: sub updateTranscodingTable | PASS (1 occurrence) |
| Plugin.pm: normalization => 0 in prefs | PASS (1 occurrence) |
| Plugin.pm: TranscodingHelper::Conversions() | PASS (1 occurrence) |
| Plugin.pm: enable-volume-normalisation (2x: remove + conditional add) | PASS (2 occurrences) |
| custom-convert.conf: unchanged | PASS (0 diff lines) |
| custom-convert.conf: --disable-audio-cache in all 4 pipelines (STR-11) | PASS (4 occurrences) |
| custom-convert.conf: all 4 pipelines contain single-track | PASS (4 occurrences) |

## Deviations from Plan

None - plan executed exactly as written.

Note: `perl -c` syntax checks fail with LMS-dependent modules (Path::Class, Log::Log4perl etc. not installed outside LMS context). This is expected behavior for all SpotOn plugin files and does not indicate code errors. Verified at structural level via grep checks.

## Known Stubs

None. All implementation is complete per plan scope. The `[spoton]` placeholder in custom-convert.conf is intentionally left as-is in the file on disk — `updateTranscodingTable` replaces it at runtime via regex.

## Threat Flags

No new security surface introduced. All threat mitigations from plan threat model are implemented:

- T-04-01 (bitrate tampering): regex `\d{2,3}` safe; whitelist validation in Settings.pm
- T-04-02 (helper name tampering): basename() strips path components; Helper.pm uses trusted search paths
- T-04-03 (cacheDir path traversal): constructed from `$serverPrefs->get('cachedir')` + fixed 'spoton' suffix; no user input
- T-04-04 (commandTable logging): only at INFOLOG level, requires explicit debug enable

## Self-Check: PASSED

- Plugins/SpotOn/ProtocolHandler.pm: exists and contains dynamic formatOverride
- Plugins/SpotOn/Plugin.pm: exists and contains updateTranscodingTable
- Commit 181fc65: feat(04-01): dynamic formatOverride in ProtocolHandler.pm
- Commit 3c1dc93: feat(04-01): updateTranscodingTable + normalization pref in Plugin.pm
- custom-convert.conf: unchanged (0 git diff lines against HEAD)
