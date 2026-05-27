---
phase: 01-plugin-skeleton-binary-foundation
plan: "02"
subsystem: infra
tags: [lms, perl, binary-discovery, settings, helper, tt2-template, i18n]

# Dependency graph
requires:
  - 01-01 (Plugin.pm _pluginDataFor, prefs namespace, strings.txt string keys)
provides:
  - Helper.pm binary discovery via findbin() with spoton/spoton-x86_64/spoton-custom candidates
  - helperCheck() with ok spoton v... regex and MIN_BINARY_VERSION 1.0.0 enforcement
  - aarch64 armhf-linux fallback via addFindBinPaths
  - Settings.pm Slim::Web::Settings subclass with CSRF-safe routing
  - basic.html TT2 template with binary status, bitrate select, disabled account section
affects:
  - 01-03 (binary foundation can now be validated via Helper.pm helperCheck)
  - all subsequent phases (Settings page is the admin entry point for all configuration)

# Tech tracking
tech-stack:
  added:
    - Slim::Web::Settings (settings controller base class)
    - Slim::Web::HTTP::CSRF (CSRF-safe name/page routing)
    - Slim::Utils::Misc::findbin (LMS binary discovery)
    - Slim::Utils::Misc::addFindBinPaths (binary search path extension)
    - Slim::Utils::OSDetect (OS/arch detection for binary selection)
    - JSON::XS::VersionOneAndTwo (JSON parsing for --check capability manifest)
  patterns:
    - Slim::Web::Settings subclass pattern (name/page/prefs/handler)
    - CSRF-safe settings routing via protectName/protectURI
    - Binary discovery: findbin() + decodeExternalHelperPath + -f && -x check
    - helperCheck backtick exec with ok spoton v... regex validation
    - MIN_BINARY_VERSION enforcement via _versionCompare (not in Spotty, SpotOn extension)
    - wantarray return pattern: ($helper, $helperVersion) or scalar $helper
    - prefs->setChange listener for binary cache invalidation
    - TT2 template with WRAPPER setting macro for LMS settings UI
    - Account section disabled via placeholder text per D-06

key-files:
  created:
    - Plugins/SpotOn/Helper.pm
    - Plugins/SpotOn/Settings.pm
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
  modified: []

key-decisions:
  - "PATTERNS.md had PLUGIN_SPOTTY_BITRATE in the bitrate snippet (copy-paste error from Spotty analog) — used PLUGIN_SPOTON_BITRATE as required by plan"
  - "helperCheck regex uses 'ok spoton' not 'ok spotty' — critical correctness requirement for binary validation"
  - "MIN_BINARY_VERSION 1.0.0 added as SpotOn extension to Spotty pattern — enforces version floor on binary discovery"
  - "Settings.pm handler does not call saveSettings path for 'binary' explicitly — Slim::Web::Settings base class handles it via prefs() declaration"
  - "basic.html account section rendered as grey placeholder text (not hidden) — visible disabled state per D-06"

patterns-established:
  - "Binary validation command: spoton -n SpotOn --check (name differs from Spotty's Spotty)"
  - "Capability manifest in second line of --check output, parsed as JSON"
  - "Settings URL: plugins/SpotOn/settings/basic.html (case-sensitive SpotOn)"
  - "Template WRAPPER blocks use LMS [% WRAPPER setting title=... desc=... %] macro"

requirements-completed:
  - LMS-02
  - LMS-07

# Metrics
duration: 3min
completed: 2026-05-27
---

# Phase 01 Plan 02: Helper.pm + Settings.pm Summary

**Binary-Discovery via findbin() with spoton regex, MIN_BINARY_VERSION enforcement, Settings.pm Slim::Web::Settings subclass, and TT2 template with binary status and disabled account section**

## Performance

- **Duration:** ~3 min
- **Started:** 2026-05-27T07:03:35Z
- **Completed:** 2026-05-27T07:06:17Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments

- Helper.pm: Binary discovery via LMS `findbin()` with spoton/spoton-x86_64/spoton-custom candidates
- Helper.pm: `helperCheck()` validates binary with `/^ok spoton v([\d\.]+)/i` regex and MIN_BINARY_VERSION 1.0.0 check
- Helper.pm: aarch64 armhf-linux fallback via `addFindBinPaths`; prefs 'binary' change listener resets cached state
- Helper.pm: `_versionCompare()` for semantic version comparison (not in Spotty — SpotOn extension)
- Settings.pm: `Slim::Web::Settings` subclass with CSRF-safe `name()`/`page()` routing
- Settings.pm: `handler()` passes `helperMissing`, `binaryVersion`, `binaryPath` to template
- basic.html: TT2 template with binary status section, bitrate select (320/160/96), disabled account section
- Zero `PLUGIN_SPOTTY_*` references in template (confirmed by grep)

## Task Commits

1. **Task 1: Helper.pm** - `5dff96f` (feat)
2. **Task 2: Settings.pm + basic.html** - `8efff7c` (feat)

## Files Created/Modified

- `Plugins/SpotOn/Helper.pm` - Binary-lifecycle module: findbin discovery, helperCheck, MIN_BINARY_VERSION, aarch64 fallback
- `Plugins/SpotOn/Settings.pm` - Settings controller: CSRF routing, binary status handler, bitrate save
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` - TT2 settings template: binary status, bitrate select, disabled account section

## Decisions Made

- `PATTERNS.md` contained `PLUGIN_SPOTTY_BITRATE` in the Bitrate snippet — this was a copy-paste error from the Spotty analog. The plan explicitly called this out as a known error; `PLUGIN_SPOTON_BITRATE` is used in all generated files.
- `helperCheck` regex: `/^ok spoton v([\d\.]+)/i` — the binary outputs `ok spoton v...` not `ok spotty v...`. This is the critical correctness distinction between SpotOn and Spotty.
- `MIN_BINARY_VERSION` enforcement (`_versionCompare`) is a SpotOn extension not found in Spotty. Binaries below 1.0.0 are rejected at discovery time.

## Deviations from Plan

None - plan executed exactly as written.

## Known Stubs

| Stub | File | Line | Reason |
|------|------|------|--------|
| Account section disabled (grey placeholder) | `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | 22-24 | Per D-06: Account configuration is Phase 2 scope. Section is visible but non-functional. |

The account stub does not prevent the plan's goal (binary discovery + settings page). It is an intentional Phase-1 design decision.

## Threat Flags

No new threat surface introduced. The `helperCheck()` backtick exec uses binary paths from `findbin()` (LMS-internal path walking), not user input — T-01-05 in the threat register is accepted as per plan.

## Self-Check: PASSED

- `Plugins/SpotOn/Helper.pm` exists: FOUND
- `Plugins/SpotOn/Settings.pm` exists: FOUND
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` exists: FOUND
- Commit `5dff96f` exists: FOUND
- Commit `8efff7c` exists: FOUND
- `grep -c PLUGIN_SPOTTY basic.html` = 0: CONFIRMED

---
*Phase: 01-plugin-skeleton-binary-foundation*
*Completed: 2026-05-27*
