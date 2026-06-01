---
phase: "05-spotify-connect"
plan: "05"
subsystem: "settings-ui"
tags: ["settings", "connect", "per-player", "i18n", "ui"]
dependency_graph:
  requires:
    - "05-03"  # DaemonManager.pm pref listener (enableSpotifyConnect)
  provides:
    - "settings-connect-ui"  # per-player Connect toggle + OGG override UI
  affects:
    - "Plugins/SpotOn/Settings.pm"
    - "Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html"
    - "Plugins/SpotOn/strings.txt"
tech_stack:
  added: []
  patterns:
    - "LMS per-player prefs via $prefs->client($client)->set()"
    - "Template variable guard with [% IF playerid %] for per-player sections"
    - "Whitelist validation for form inputs (auto|ogg|pcm)"
    - "Checkbox-absent-means-zero pattern for boolean toggles"
key_files:
  created: []
  modified:
    - "Plugins/SpotOn/Settings.pm"
    - "Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html"
    - "Plugins/SpotOn/strings.txt"
decisions:
  - "Used [% IF playerid %] template guard (correct LMS idiom) instead of [% IF playercount %] (plan said playercount, but playerid is the correct LMS variable set by Slim::Web::Settings when a client is selected)"
metrics:
  duration: "~15m"
  completed: "2026-06-01T09:20:00Z"
  tasks_completed: 1
  tasks_total: 2
  files_modified: 3
---

# Phase 05 Plan 05: Settings UI Connect Toggle + OGG Override Summary

**One-liner:** Per-player Spotify Connect enable/disable toggle and OGG-passthrough override select (Auto/OGG/PCM) added to Settings UI with full EN+DE i18n strings.

## What Was Built

Settings UI additions for per-player Spotify Connect control:

1. **Settings.pm** — Extended `handler()` with two per-player pref saves:
   - `enableSpotifyConnect` (0 or 1, checkbox coercion, T-05-18)
   - `connectOggOverride` ('auto'|'ogg'|'pcm', whitelist validated, T-05-19)
   - Template variables `connectEnabled` (default 1) and `connectOggOverride` (default 'auto') passed when `$client` is defined
   - Both blocks guarded with `if ($client)` to avoid errors on global settings page

2. **basic.html** — New `[% IF playerid %]` guarded block between Client-ID and ZeroConf sections:
   - Checkbox: `pref_enableSpotifyConnect` — shows "Enable Spotify Connect" with `connectEnabled` checked state
   - Select: `pref_connectOggOverride` — 3 options (Auto/OGG Passthrough/PCM force) with `connectOggOverride` selected state

3. **strings.txt** — 6 new `PLUGIN_SPOTON_CONNECT_*` string keys with DE+EN translations:
   - `PLUGIN_SPOTON_CONNECT_ENABLED`
   - `PLUGIN_SPOTON_CONNECT_ENABLED_DESC`
   - `PLUGIN_SPOTON_CONNECT_ENABLED_LABEL`
   - `PLUGIN_SPOTON_CONNECT_OGG_OVERRIDE`
   - `PLUGIN_SPOTON_CONNECT_OGG_OVERRIDE_DESC`
   - `PLUGIN_SPOTON_CONNECT_OGG_AUTO`

## Task Commits

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Settings.pm + basic.html + strings.txt | 0ffeefa | Settings.pm, basic.html, strings.txt |
| 2 | UAT checkpoint | — | Human verification pending |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Used `[% IF playerid %]` instead of `[% IF playercount %]` as template guard**
- **Found during:** Task 1 (template implementation)
- **Issue:** Plan specified `[% IF playercount %]` as the per-player settings guard. However, `playercount` is a playlist page variable in LMS (not available in settings templates). The correct LMS settings template variable is `playerid`, which is set by `Slim::Web::Settings` base class when `$client` is defined. Using `playercount` would either always be truthy (any players connected) or undefined in settings context — both incorrect behaviors.
- **Fix:** Used `[% IF playerid %]` — the standard LMS idiom verified in `/usr/share/squeezeboxserver/HTML/EN/settings/header.html` and `Slim::Web::Settings` source.
- **Files modified:** `basic.html`

## Checkpoint: Task 2 (UAT) — Awaiting Human Verification

Task 2 is a `checkpoint:human-verify` requiring end-to-end Spotify Connect UAT. See plan for 10-item verification checklist.

## Known Stubs

None — all prefs are wired to real LMS pref storage. Template variables use live values from `$prefs->client($client)->get()`.

## Threat Flags

No new security-relevant surfaces introduced beyond those in the plan's threat model (T-05-18, T-05-19 both mitigated).

## Self-Check

- [x] `Plugins/SpotOn/Settings.pm` modified with enableSpotifyConnect and connectOggOverride
- [x] `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` contains Connect UI block
- [x] `Plugins/SpotOn/strings.txt` has 8 PLUGIN_SPOTON_CONNECT_* keys (includes pre-existing CONNECT_HINT_ALT and CONNECTED_AS)
- [x] Commit 0ffeefa exists with all 3 files

## Self-Check: PASSED
