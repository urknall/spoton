---
phase: 10-connect-dstm
plan: "02"
subsystem: connect
tags: [librespot, autoplay, dstm, spirc, perl, settings, i18n]

# Dependency graph
requires:
  - phase: 10-connect-dstm/01
    provides: binary --autoplay flag + getCapability('autoplay') in --check JSON
provides:
  - enableAutoplay per-player pref initialized with default 1 (Plugin.pm)
  - --autoplay on/off flag passed to Connect daemon gated on binary capability (Daemon.pm)
  - Bidirectional DSTM provider sync (Settings.pm)
  - Capability-gated Autoplay checkbox in Settings UI (basic.html)
  - PLUGIN_SPOTON_AUTOPLAY_ENABLED + _DESC + _LABEL strings in 11 languages (strings.txt)
affects:
  - 10-connect-dstm/03

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Cross-namespace pref write via preferences('plugin.dontstopthemusic') for DSTM provider sync
    - D-13/D-14 reverse sync as page-load read (no pref-change callback, avoids loop risk)
    - stopForSync() before initHelpers() to force daemon restart on pref change

key-files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/Connect/Daemon.pm
    - Plugins/SpotOn/Settings.pm
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
    - Plugins/SpotOn/strings.txt

key-decisions:
  - "D-13/D-14 sync implemented as page-load read of plugin.dontstopthemusic provider pref — no pref-change callback, no loop risk (Pitfall 3)"
  - "stopForSync() before initHelpers() ensures daemon restarts with new --autoplay flag (Pitfall 1)"
  - "Autoplay checkbox placed between Connect and Discovery toggles in Settings UI"

patterns-established:
  - "Cross-namespace pref write: my $dstmPrefs = preferences('plugin.dontstopthemusic'); $dstmPrefs->client($client)->set('provider', ...)"
  - "Capability-gated UI: [% IF canAutoplay %]...[% END %] wrapping Settings HTML blocks"

requirements-completed:
  - DSTM-05
  - DSTM-06

# Metrics
duration: 15min
completed: 2026-06-04
---

# Phase 10 Plan 02: Autoplay Pref, Daemon Flag, Settings UI, and DSTM Sync Summary

**Per-player enableAutoplay pref wired from Plugin.pm init through Daemon.pm --autoplay flag to Settings UI checkbox with bidirectional LMS DSTM provider synchronization**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-06-04T13:00:00Z
- **Completed:** 2026-06-04T13:04:07Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments

- Added `enableAutoplay => 1` per-player pref to Plugin.pm (D-08)
- Daemon.pm passes `--autoplay on/off` flag gated on `getCapability('autoplay')` (D-09)
- Settings.pm save handler: saves enableAutoplay, syncs DSTM provider bidirectionally (D-11/D-12), stops live daemon before initHelpers (Pitfall 1)
- Settings.pm template vars: `canAutoplay` + `autoplayEnabled` with D-13/D-14 reverse sync via page-load read
- basic.html: capability-gated Autoplay checkbox between Connect and Discovery toggles
- strings.txt: PLUGIN_SPOTON_AUTOPLAY_ENABLED + _DESC + _LABEL in all 11 languages

## Task Commits

Each task was committed atomically:

1. **Task 1: Add enableAutoplay pref, daemon flag, and i18n strings** - `32799fb` (feat)
2. **Task 2: Add Settings UI toggle with DSTM bidirectional sync** - `c6f7a30` (feat)

## Files Created/Modified

- `Plugins/SpotOn/Plugin.pm` - Added `enableAutoplay => 1` to `$prefs->init({})` (D-08)
- `Plugins/SpotOn/Connect/Daemon.pm` - Added `--autoplay on/off` in `@helperArgs` gated on `getCapability('autoplay')` (D-09)
- `Plugins/SpotOn/Settings.pm` - Autoplay save handler + DSTM sync + stopForSync restart + canAutoplay/autoplayEnabled template vars with D-13/D-14 reverse sync
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` - Capability-gated Autoplay checkbox (D-10)
- `Plugins/SpotOn/strings.txt` - Three i18n key blocks (PLUGIN_SPOTON_AUTOPLAY_ENABLED + _DESC + _LABEL) in 11 languages

## Decisions Made

- D-13/D-14 sync implemented as page-load read (not pref-change callback) to avoid the DSTM callback loop risk (Pitfall 3). When the Settings page loads, autoplayEnabled is derived from the current `plugin.dontstopthemusic` provider pref value.
- Used `stopForSync()` before `initHelpers()` in the autoplay save block — necessary because `startHelper()` skips alive daemons (Pitfall 1 from RESEARCH.md).
- Autoplay checkbox positioned between Connect and Discovery checkboxes in Settings UI for logical grouping.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None — all patterns were well-documented in RESEARCH.md and PATTERNS.md. No unexpected behavior.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Plan 02 complete: Perl plugin side fully wired for autoplay capability
- Plan 03 (binary rebuild) can proceed: binaries need `--autoplay` flag + `"autoplay": true` in `--check` JSON for the capability gate to activate
- DontStopTheMusic.pm unchanged (DSTM-06 regression-safe)

## Self-Check: PASSED

- `Plugins/SpotOn/Plugin.pm` exists and contains `enableAutoplay`: confirmed (1 match)
- `Plugins/SpotOn/Connect/Daemon.pm` exists and contains `--autoplay`: confirmed
- `Plugins/SpotOn/Settings.pm` exists and contains `enableAutoplay`: confirmed (4 matches)
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` contains `pref_enableAutoplay`: confirmed (3 matches)
- `Plugins/SpotOn/strings.txt` contains `PLUGIN_SPOTON_AUTOPLAY_ENABLED`: confirmed (3 blocks)
- Commits 32799fb and c6f7a30 verified in git log
- `prove -l t/05_perl_syntax.t t/02_strings.t`: PASS

---
*Phase: 10-connect-dstm*
*Completed: 2026-06-04*
