---
phase: 16-macos-universal-binary
plan: 02
subsystem: plugin-perl
completed_at: "2026-06-11T15:32:23Z"
duration_minutes: 12
tags:
  - macos
  - binary-detection
  - gatekeeper
  - i18n
  - settings-ui
dependency_graph:
  requires:
    - 16-01 (Bin/darwin/.gitkeep placeholder, Wave 0 parallel)
  provides:
    - macOS binary discovery via ISMAC block in Helper.pm
    - isMac template variable for conditional UI in Settings.pm
    - Gatekeeper warning UI in basic.html
    - PLUGIN_SPOTON_GATEKEEPER_HINT in all 11 languages
    - macOS documentation in README.md
  affects:
    - Plugins/SpotOn/Helper.pm
    - Plugins/SpotOn/Settings.pm
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
    - Plugins/SpotOn/strings.txt
    - README.md
tech_stack:
  added: []
  patterns:
    - "ISMAC platform guard (addFindBinPaths — identical to ISWINDOWS pattern)"
    - "isMac template variable (Boolean flag, same pattern as degradedMode)"
    - "Conditional orange warning div in TT2 template"
    - "11-language i18n block in strings.txt"
key_files:
  created: []
  modified:
    - Plugins/SpotOn/Helper.pm
    - Plugins/SpotOn/Settings.pm
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
    - Plugins/SpotOn/strings.txt
    - README.md
decisions:
  - "ISMAC block placed after ISWINDOWS block before $prefs->setChange (D-02)"
  - "isMac placed near binaryPath block in handler(), not in per-player section"
  - "Gatekeeper hint uses margin-top:8px (not margin-bottom) to separate from red error above"
  - "README update replaces the entire sentence (not inline edit) for clarity"
requirements:
  - PLT-02
  - PLT-03
---

# Phase 16 Plan 02: macOS Binary Detection + Gatekeeper UI Summary

**One-liner:** macOS binary auto-discovery via ISMAC guard in Helper.pm, dynamic Gatekeeper warning on Settings page in 11 languages, README platform list updated with xattr workaround.

## Tasks Completed

| # | Task | Commit | Files |
|---|------|--------|-------|
| 1 | Add ISMAC block to Helper.pm and isMac to Settings.pm | 79b171c | Helper.pm, Settings.pm |
| 2 | Add Gatekeeper hint to basic.html + strings.txt + README.md | 6429c22 | basic.html, strings.txt, README.md |

## What Was Built

### Task 1: Binary Detection + Template Flag

**Helper.pm** — Inserted ISMAC block in `init()` after the ISWINDOWS block (line 32) and before `$prefs->setChange`:

```perl
if ( main::ISMAC ) {
    Slim::Utils::Misc::addFindBinPaths(
        catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'darwin')
    );
}
```

Ensures LMS finds the Universal Binary at `Bin/darwin/spoton` on macOS systems.

**Settings.pm** — Added `$paramRef->{isMac} = main::ISMAC ? 1 : 0;` near the `binaryPath` block in `handler()`. Passes a Boolean macOS flag to the TT2 template so `basic.html` can conditionally show the Gatekeeper warning.

### Task 2: Gatekeeper UI + i18n + README

**basic.html** — Added `[% IF isMac %]` conditional block inside the binary-missing ELSE section:

```html
[% IF isMac %]
<div style="color: orange; margin-top:8px">
    [% 'PLUGIN_SPOTON_GATEKEEPER_HINT' | string %]
</div>
[% END %]
```

Orange warning (not red error) follows the established `discoveryByCrashLoop` warning pattern.

**strings.txt** — Appended `PLUGIN_SPOTON_GATEKEEPER_HINT` with 11 language translations (CS, DA, DE, EN, ES, FR, IT, NL, NO, PL, SV). English text: "The SpotOn binary is blocked by macOS. Run in Terminal: xattr -d com.apple.quarantine /path/to/spoton". All translations sourced from 16-RESEARCH.md.

**README.md** — Replaced "macOS binaries not yet included" with:
"Supported platforms: ..., macOS (Universal Binary: Intel + Apple Silicon). On macOS, if you download the binary manually (not via LMS plugin manager), you may need to run `xattr -d com.apple.quarantine /path/to/spoton` in Terminal before first use."

## Deviations from Plan

None - plan executed exactly as written.

## Known Stubs

None. All implemented functionality is wired end-to-end:
- `main::ISMAC` is an LMS compile-time constant (no stub)
- `isMac` template variable flows from Settings.pm → basic.html
- String key `PLUGIN_SPOTON_GATEKEEPER_HINT` defined in strings.txt with all 11 translations

## Threat Flags

None. The `isMac` flag is a Boolean (0/1) that reveals OS type already observable through other LMS mechanisms. No new network endpoints, auth paths, or file access patterns introduced.

## Self-Check

Verified:
- Helper.pm ISMAC block: `grep -c 'main::ISMAC' Plugins/SpotOn/Helper.pm` = 2 (existing + new)
- Settings.pm isMac: `grep -c 'isMac' Plugins/SpotOn/Settings.pm` = 1
- basic.html GATEKEEPER_HINT: `grep -c 'PLUGIN_SPOTON_GATEKEEPER_HINT' basic.html` = 1
- strings.txt GATEKEEPER_HINT: `grep -c 'PLUGIN_SPOTON_GATEKEEPER_HINT' strings.txt` = 1
- README macOS: `grep -c 'Universal Binary' README.md` = 1
- README old text removed: `grep -c 'macOS binaries not yet included' README.md` = 0
- Commits: 79b171c and 6429c22 in git log

## Self-Check: PASSED
