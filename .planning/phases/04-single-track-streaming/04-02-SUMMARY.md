---
phase: 04-single-track-streaming
plan: "02"
subsystem: context-queueing-settings-ui
tags:
  - playall
  - context-queueing
  - orphaned-process-cleanup
  - settings-ui
  - normalization
  - i18n
dependency_graph:
  requires:
    - "04-01: Plugin.pm with updateTranscodingTable, normalization pref initialized"
  provides:
    - "Context queueing via playall flag in _trackItem and _albumTrackItem"
    - "Orphaned librespot process cleanup every 3600s when no player is active"
    - "Settings page Streaming section with normalization checkbox"
    - "i18n strings for streaming settings in DE and EN"
  affects:
    - "04-03 and beyond: normalization pref now persisted via Settings UI"
tech_stack:
  added:
    - "Slim::Utils::Timers (killOrphanedProcesses timer pattern)"
    - "Slim::Player::Client (autoloaded — not explicitly imported)"
  patterns:
    - "playall => 1 in OPML audio item hash for XMLBrowser context queueing"
    - "Timer pattern with killTimers + setTimer for cleanup cycle"
    - "Settings checkbox ternary: $paramRef->{'pref_normalization'} ? 1 : 0"
key_files:
  created: []
  modified:
    - "Plugins/SpotOn/Plugin.pm"
    - "Plugins/SpotOn/Settings.pm"
    - "Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html"
    - "Plugins/SpotOn/strings.txt"
decisions:
  - "D-09/D-10: playall => 1 added to both _trackItem and _albumTrackItem for full context queueing"
  - "STR-10: cleanup timer fires every 3600s; skips kill when any player is isPlaying()"
  - "Pitfall 6 / CON-09: PHASE-5-NOTE comment added to _killOrphanedProcesses for Phase 5 Connect PID exclusion"
  - "T-04-05: normalization pref ternary forces 0 or 1, no arbitrary string accepted"
  - "ReplayGain in NORMALIZATION_LABEL (user-facing) instead of technical --enable-volume-normalisation flag name"
metrics:
  duration: "~8 minutes"
  completed_date: "2026-05-28"
  tasks_completed: 3
  tasks_total: 3
  files_changed: 4
---

# Phase 04 Plan 02: Context Queueing, Orphaned Process Cleanup, Settings UI Summary

playall context queueing for album/playlist/library track taps, hourly orphaned librespot process cleanup with isPlaying guard, and normalization checkbox in Settings UI.

## What Was Built

### Task 1: playall flag + orphaned process cleanup in Plugin.pm (commit a1f5c16)

Three changes to Plugin.pm:

**KILL_PROCESS_INTERVAL constant:**
- Added `use constant KILL_PROCESS_INTERVAL => 3600;` after imports

**playall flag in _trackItem and _albumTrackItem:**
- `_trackItem` (line 327): `playall => 1` — enables context queueing for search results, recently played, top tracks, liked songs, and playlist tracks (D-09, D-10)
- `_albumTrackItem` (line 975): `playall => 1` — enables context queueing for album track views (D-09)
- XMLBrowser reads the flag and issues `playlist loadtracks listref @urls undef $playIndex` to queue the entire visible feed starting at the tapped track

**_killOrphanedProcesses sub + timer:**
- initPlugin registers `killTimers + setTimer` for `\&_killOrphanedProcesses` inside `!main::SCANNER` block, after token refresh timer
- `_killOrphanedProcesses` sub: kills duplicate timers, checks all clients via `Slim::Player::Client::clients()` and `isPlaying()`, conditionally kills via `pkill -f` (Unix) or `taskkill /IM` (Windows), always reschedules
- PHASE-5-NOTE comment included per Pitfall 6 / CON-09 requirement
- `eval {}` wraps the kill command for error containment (T-04-06)

### Task 2: Settings UI normalization toggle + pref handling (commit 2e0df1b)

**Settings.pm:**
- `prefs()` method extended: now returns `($prefs, 'bitrate', 'binary', 'clientId', 'normalization')`
- `handler()` saveSettings block: `my $norm = $paramRef->{'pref_normalization'} ? 1 : 0; $prefs->set('normalization', $norm);` — added after bitrate validation, before OAuth block (T-04-05)

**basic.html:**
- New WRAPPER setting block inserted between bitrate block and Setup Wizard / Account Settings section
- Contains: `<input type="checkbox" name="pref_normalization" value="1" [% IF prefs.pref_normalization %]checked[% END %]/>` and label with NORMALIZATION_LABEL string
- No "Streaming" umbrella wrapper needed — bitrate + normalization adjacent is sufficient for Phase 4 (D-08)

### Task 3: i18n strings for streaming settings (commit 84c0b47)

Four new string blocks added to strings.txt BEFORE the final SON entry:

| Key | DE | EN |
|-----|----|----|
| PLUGIN_SPOTON_STREAMING_SETTINGS | Streaming-Einstellungen | Streaming Settings |
| PLUGIN_SPOTON_NORMALIZATION | Lautstärkenormalisierung | Volume Normalization |
| PLUGIN_SPOTON_NORMALIZATION_DESC | (empty) | (empty) |
| PLUGIN_SPOTON_NORMALIZATION_LABEL | Lautstärke normalisieren (ReplayGain) | Normalize volume (ReplayGain) |

SON entry remains last in file.

## Verification Results

| Check | Result |
|-------|--------|
| Plugin.pm: KILL_PROCESS_INTERVAL = 3600 | PASS (4 occurrences) |
| Plugin.pm: playall in _trackItem | PASS (line 327) |
| Plugin.pm: playall in _albumTrackItem | PASS (line 975) |
| Plugin.pm: grep -c 'playall' returns 2 | PASS |
| Plugin.pm: _killOrphanedProcesses registered in initPlugin | PASS |
| Plugin.pm: _killOrphanedProcesses sub exists (6 references) | PASS |
| Plugin.pm: pkill present | PASS (1 occurrence) |
| Plugin.pm: PHASE-5-NOTE comment | PASS (1 occurrence) |
| Settings.pm: prefs() includes 'normalization' | PASS |
| Settings.pm: set('normalization', $norm) with ternary | PASS |
| basic.html: WRAPPER with PLUGIN_SPOTON_NORMALIZATION | PASS |
| basic.html: checkbox name=pref_normalization value=1 | PASS |
| basic.html: checked condition [% IF prefs.pref_normalization %] | PASS |
| basic.html: normalization block between bitrate and accounts section | PASS |
| strings.txt: PLUGIN_SPOTON_STREAMING_SETTINGS (DE+EN) | PASS |
| strings.txt: PLUGIN_SPOTON_NORMALIZATION (DE+EN) | PASS |
| strings.txt: PLUGIN_SPOTON_NORMALIZATION_DESC (DE+EN empty) | PASS |
| strings.txt: PLUGIN_SPOTON_NORMALIZATION_LABEL (DE+EN) | PASS |
| strings.txt: SON entry remains last | PASS |

## Deviations from Plan

None - plan executed exactly as written.

Note: `perl -c` syntax checks fail with LMS-dependent modules (Log::Log4perl, Slim::* not installed outside LMS context). This is expected behavior for all SpotOn plugin files and does not indicate code errors. All acceptance criteria verified via grep and structural checks.

## Known Stubs

None. All implementation is complete per plan scope.

## Threat Flags

No new security surface introduced. All threat mitigations from plan threat model are implemented:

- T-04-05 (normalization pref tampering): ternary `? 1 : 0` enforces only 0 or 1, no arbitrary string
- T-04-06 (pkill path tampering): `$helper` path comes from Helper.pm::_findBin (trusted search paths); `eval {}` wraps for error containment
- T-04-07 (cleanup kills active processes): `isPlaying()` check on all clients before killing; skip if any player busy
- T-04-08 (playall flag manipulation): flag set server-side in Plugin.pm; not user-controllable

## Self-Check: PASSED

- Plugins/SpotOn/Plugin.pm: exists and contains playall (2x), KILL_PROCESS_INTERVAL (4x), _killOrphanedProcesses (6x), PHASE-5-NOTE (1x)
- Plugins/SpotOn/Settings.pm: exists and contains normalization in prefs() and handler()
- Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html: exists and contains pref_normalization (3x), PLUGIN_SPOTON_NORMALIZATION (2x)
- Plugins/SpotOn/strings.txt: exists and contains all 4 new string keys, SON is last entry
- Commit a1f5c16: feat(04-02): add playall context queueing + orphaned process cleanup
- Commit 2e0df1b: feat(04-02): settings UI normalization toggle + pref handling
- Commit 84c0b47: feat(04-02): i18n strings for streaming settings UI
