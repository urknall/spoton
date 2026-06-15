---
phase: 21-podcast-ux-polish-i18n
plan: "02"
subsystem: Plugin.pm / i18n / strings
tags: [ux, i18n, explicit, cstring, podcast, strings]
dependency_graph:
  requires: [21-01]
  provides: [explicit marker in _episodeItem, $client-aware _formatEpisodeLine2/_formatRelativeDate, 27 new string keys in 11 languages]
  affects: [Plugins/SpotOn/Plugin.pm, Plugins/SpotOn/strings.txt, t/02_strings.t]
tech_stack:
  added: []
  patterns: [cstring() for all user-visible strings, sprintf(cstring(...)) for parameterized strings]
key_files:
  created: []
  modified:
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/strings.txt
    - t/02_strings.t
decisions:
  - "UX-01 (episodeOrder setting) dropped — Spotify API default (newest first) is sufficient, no setting needed"
  - "PLUGIN_SPOTON_SHOW_VIEW added to strings.txt as well (was referenced in Plugin.pm from Plan 21-01 without a strings entry)"
metrics:
  duration: "~10 minutes"
  completed: "2026-06-15"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 3
---

# Phase 21 Plan 02: UX-03 Explicit Marker + I18N-01 Full Podcast i18n Summary

**One-liner:** Explicit content marker in episode list + full cstring() i18n refactor of duration/date helpers with 27 new string keys across 11 languages.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | UX-03 Explicit-Marker in _episodeItem | e8ff257 | Plugins/SpotOn/Plugin.pm |
| 2 | I18N-01 cstring refactor + 27 new string keys + t/02_strings.t update | 8fb3a02 | Plugins/SpotOn/Plugin.pm, Plugins/SpotOn/strings.txt, t/02_strings.t |

## Changes Made

### Task 1: UX-03 Explicit Content Marker

**_episodeItem changes:**
- Added `$explicit_tag = ($episode->{explicit}) ? ' [' . cstring($client, 'PLUGIN_SPOTON_EXPLICIT') . ']' : ''` after title extraction
- Applied to `name` and `line1` fields in `%item` hash
- `$explicit_tag` evaluates to locale-specific string (e.g., " [E]" in English, " [Explizit]" in German)
- The `PLUGIN_SPOTON_EXPLICIT` key was added in Task 2; cstring() returns key name as fallback during Task 1 execution

### Task 2: I18N-01 cstring Refactor + String Keys

**_formatEpisodeLine2 changes (I18N-01):**
- Signature changed from `($duration_sec, $release_date)` to `($client, $duration_sec, $release_date)`
- `"$hours Std $mins Min"` replaced with `sprintf(cstring($client, 'PLUGIN_SPOTON_DURATION_HM'), $hours, $mins)`
- `"$mins Min"` replaced with `sprintf(cstring($client, 'PLUGIN_SPOTON_DURATION_M'), $mins)`
- `_formatRelativeDate($release_date)` call updated to `_formatRelativeDate($client, $release_date)`
- Comment updated: removed "German-only until Phase 21" note

**_formatRelativeDate changes (I18N-01):**
- Signature changed from `($iso_date)` to `($client, $iso_date)`
- `'Heute'` replaced with `cstring($client, 'PLUGIN_SPOTON_DATE_TODAY')`
- `'Gestern'` replaced with `cstring($client, 'PLUGIN_SPOTON_DATE_YESTERDAY')`
- `"Vor $delta_days Tagen"` replaced with `sprintf(cstring($client, 'PLUGIN_SPOTON_DATE_N_DAYS_AGO'), $delta_days)`
- German months array removed; replaced with `cstring($client, "PLUGIN_SPOTON_MONTH_$month")`

**Call site update:** `_formatEpisodeLine2($duration, $date)` in `_episodeItem` updated to `_formatEpisodeLine2($client, $duration, $date)`

**strings.txt additions (27 new keys, 11 languages each):**
- `PLUGIN_SPOTON_SHOW_VIEW` — "View show" label for lazy-load sub-item (was missing, referenced by Plan 21-01)
- `PLUGIN_SPOTON_DURATION_HM` — "%s hr %s min" format
- `PLUGIN_SPOTON_DURATION_M` — "%s min" format
- `PLUGIN_SPOTON_DATE_TODAY`, `PLUGIN_SPOTON_DATE_YESTERDAY`, `PLUGIN_SPOTON_DATE_N_DAYS_AGO`
- `PLUGIN_SPOTON_MONTH_1` through `PLUGIN_SPOTON_MONTH_12` (12 keys)
- `PLUGIN_SPOTON_EXPLICIT` — "E" in EN, "Explizit" in DE, localized in all 11 languages
- `PLUGIN_SPOTON_RESUME_UNPLAYED`, `PLUGIN_SPOTON_RESUME_IN_PROGRESS`, `PLUGIN_SPOTON_RESUME_FINISHED`

**t/02_strings.t update:**
- `@bilingual_keys` extended with all 27 new keys
- Test count: 132 → 199 tests (67 new test cases)

## Verification

```
perl t/05_perl_syntax.t         # PASS (all 8 modules)
perl t/02_strings.t             # PASS (199 tests, all ok)
explicit_tag occurrences        # 3 (definition, name field, line1 field)
cstring.*PLUGIN_SPOTON_DURATION # 2 (HM and M variants)
cstring.*PLUGIN_SPOTON_DATE_TODAY # 1 (in _formatRelativeDate)
PLUGIN_SPOTON_MONTH_* in strings.txt # 12 key headers × 11 languages = 132 translation lines
```

## Deviations from Plan

### UX-01 Dropped (Pre-existing decision)

UX-01 (episodeOrder setting) was excluded before execution began, per user decision documented in the `<important_context>` section. The plan's objective line still references UX-01 but all tasks were already updated to exclude it. The API default (newest-first) is kept as-is. No Settings.pm or basic.html changes were made.

### PLUGIN_SPOTON_SHOW_VIEW Added to strings.txt

The plan specified 26 new keys for Task 2. PLUGIN_SPOTON_SHOW_VIEW was referenced in Plugin.pm (added in Plan 21-01) but had no strings.txt entry. Added it as the 27th key (Rule 2: missing critical functionality — cstring() falling back to the raw key name in UI is not acceptable for a released plugin). The test was updated accordingly (199 tests instead of ~198).

## Known Stubs

None. All strings used in Plugin.pm now have entries in strings.txt with DE and EN translations verified by t/02_strings.t.

## Threat Flags

None. T-21-02-02 (explicit flag from Spotify API) is accepted per plan — no PII, no risk from displaying it. T-21-02-03 (sprintf format strings from codebase) is accepted — no user input flows to sprintf format strings.

## Self-Check

- [x] `Plugins/SpotOn/Plugin.pm` exists and modified
- [x] `Plugins/SpotOn/strings.txt` exists and modified  
- [x] `t/02_strings.t` exists and modified
- [x] Commit `e8ff257` exists (Task 1)
- [x] Commit `8fb3a02` exists (Task 2)
- [x] `perl t/05_perl_syntax.t` passes (8/8)
- [x] `perl t/02_strings.t` passes (199/199)
- [x] `explicit_tag` count >= 2 in Plugin.pm

## Self-Check: PASSED
