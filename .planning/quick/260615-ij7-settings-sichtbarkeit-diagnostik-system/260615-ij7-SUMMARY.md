---
quick_task: 260615-ij7
title: "Settings Visibility Fix + Diagnostic System for Connect Debugging"
issues: ["#2", "#3"]
key-files:
  modified:
    - Plugins/SpotOn/Settings.pm
    - Plugins/SpotOn/Connect.pm
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
    - Plugins/SpotOn/strings.txt
decisions:
  - "Used $log->warn with [DIAG] prefix for diagnostic logs so they appear at default WARN level without requiring log level changes"
  - "Added 2 extra status indicator strings (Active/Inactive) beyond the 7 specified for better UX"
metrics:
  duration: "4m 31s"
  completed: "2026-06-15T11:29:08Z"
  tasks: 3/3
  tests: "316 pass, 0 fail (12 test files)"
---

# Quick Task 260615-ij7: Settings Visibility Fix + Diagnostic System Summary

needsClient(0) fix for Material Skin server-settings visibility plus diagnosticMode toggle, bundle download endpoint, and [DIAG] timing logs in Connect event handler.

## Task Results

| Task | Name | Commit | Key Changes |
|------|------|--------|-------------|
| 1 | Fix needsClient + diagnostic bundle endpoint + diagnosticMode pref | 071c67f | Settings.pm: needsClient=0, _diagnosticBundleHandler; Plugin.pm: diagnosticMode pref |
| 2 | Add diagnostic timing logs to Connect event handler | 082ff67 | Connect.pm: 7 [DIAG] log points across start/change/stop/resume handlers |
| 3 | Add Diagnostic section to Settings UI + i18n strings | fd9a855 | basic.html: diagnostic section; strings.txt: 9 keys x 11 languages; Settings.pm: save/load |

## Changes Made

### Settings Visibility (GitHub #2)
- Changed `needsClient()` return from 1 to 0 in Settings.pm
- SpotOn now appears in Material Skin server-settings dropdown
- Per-player settings remain guarded by existing `[% IF playerid %]` block in template

### Diagnostic System (GitHub #3)
- **diagnosticMode pref**: Global toggle, default off, registered in Plugin.pm init
- **Bundle endpoint**: `/plugins/SpotOn/settings/diagnosticBundle` returns downloadable text file with system info header + daemon logs (403 when disabled)
- **Timing logs**: 7 `[DIAG]` log points in _connectEvent covering start, change, stop (grace + normal), resume (dead-history, re-entering, normal) paths
- **UI**: Diagnostic section in settings template with checkbox toggle, status indicator, and conditional download button
- **i18n**: 9 string keys with all 11 languages (CS, DA, DE, EN, ES, FR, IT, NL, NO, PL, SV)

### Diagnostic Bundle Content
- LMS version, OS, Perl version, SpotOn version
- Active account ID (redacted), Client-ID (redacted)
- Player list (name, MAC, model)
- All *-connect.log files from spoton cache dir (capped at 500KB each)

## Deviations from Plan

### Auto-added Improvements

**1. [Rule 2 - Missing Functionality] Added DIAG_STATUS_ACTIVE and DIAG_STATUS_INACTIVE strings**
- **Found during:** Task 3
- **Issue:** Template uses status indicator text but plan only specified 7 string keys
- **Fix:** Added 2 additional string keys for active/inactive status indicators (9 total)
- **Files modified:** Plugins/SpotOn/strings.txt

## Test Results

All 316 existing tests pass across 12 test files. No regressions.

## Self-Check: PASSED

- [x] 071c67f exists in git log
- [x] 082ff67 exists in git log
- [x] fd9a855 exists in git log
- [x] Settings.pm contains `return 0` in needsClient
- [x] Settings.pm contains diagnosticBundle endpoint
- [x] Plugin.pm contains diagnosticMode pref
- [x] Connect.pm contains 7 [DIAG] log lines
- [x] basic.html contains diagnostic section
- [x] strings.txt contains 9 diagnostic string keys x 11 languages
