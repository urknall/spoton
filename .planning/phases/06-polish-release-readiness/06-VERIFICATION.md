---
phase: 06-polish-release-readiness
verified: 2026-06-03T14:00:30Z
status: human_needed
score: 10/12 must-haves verified
overrides_applied: 0
gaps:
  - truth: "SC-10: librespot binaries for x86_64, aarch64, armhf, and i386 are present and pass --check version verification"
    status: failed
    reason: "Only x86_64-linux has an actual binary. aarch64-linux, armhf-linux, arm-linux, i386-linux directories contain only .gitkeep placeholder files. Plan 05 explicitly defers this to Phase 6.1 but Phase 6.1 does not exist in ROADMAP.md."
    artifacts:
      - path: "Plugins/SpotOn/Bin/aarch64-linux/"
        issue: "Contains only .gitkeep, no binary"
      - path: "Plugins/SpotOn/Bin/armhf-linux/"
        issue: "Contains only .gitkeep, no binary"
      - path: "Plugins/SpotOn/Bin/i386-linux/"
        issue: "Contains only .gitkeep, no binary"
    missing:
      - "Cross-compiled binaries for aarch64, armhf, i386"
      - "Phase 6.1 formally defined in ROADMAP.md with success criteria"
  - truth: "SC-12: Plugin is installable via LMS custom repository URL"
    status: partial
    reason: "repo.xml exists with correct structure but contains PLACEHOLDER_SHA1_PHASE_6_1 and PLACEHOLDER_URL_PHASE_6_1. Plugin cannot actually be installed until these are populated. Deferred to a Phase 6.1 that does not yet exist in ROADMAP.md."
    artifacts:
      - path: "repo.xml"
        issue: "SHA1 and URL are placeholders -- not functional for actual installation"
    missing:
      - "Actual SHA1 checksum and download URL"
      - "Phase 6.1 formally defined in ROADMAP.md"
deferred:
  - truth: "SC-10: Multi-architecture binaries present and passing --check"
    addressed_in: "Phase 6.1 (planned but not yet in ROADMAP)"
    evidence: "Plan 05 deferred_to_phase_6_1 section explicitly: 'SC-10 requires cross-compilation for aarch64, armhf, i386. Phase 6.1 will build all binaries and finalize repo.xml SHA1 + download URL.'"
  - truth: "SC-12: Plugin installable via LMS custom repository URL (fully functional)"
    addressed_in: "Phase 6.1 (planned but not yet in ROADMAP)"
    evidence: "repo.xml template ready; SHA1 and URL to be finalized when binaries are built"
human_verification:
  - test: "SC-1: Set bitrate to 96 on Player A and 320 on Player B, play tracks on each"
    expected: "Log shows --bitrate 96 for Player A and --bitrate 320 for Player B in updateTranscodingTable"
    why_human: "Requires running LMS with two configured players and Spotify streaming"
  - test: "SC-2: Enable DSTM SpotOn Empfehlungen, play short playlist, let it end"
    expected: "DSTM queues additional Spotify tracks automatically, playback continues"
    why_human: "Requires running LMS with active Spotify account, real playlist, and DSTM timing"
  - test: "SC-3: Copy binary to Bin/x86_64-linux/spoton-custom, restart LMS"
    expected: "Settings page shows spoton-custom path as active binary"
    why_human: "Requires LMS restart and binary filesystem interaction"
  - test: "SC-4: Open Settings, select a player, verify format dropdown and bitrate override"
    expected: "5-option Format-Dropdown and 4-option Bitrate Override visible in per-player section"
    why_human: "Visual UI verification in LMS web interface"
  - test: "SC-5: Set Format to FLAC for a player, play a track"
    expected: "canDirectStream returns 0 in logs, transcoding pipeline used"
    why_human: "Requires running LMS with active streaming and log inspection"
  - test: "SC-6: Enable Normalization, start Connect playback"
    expected: "--enable-volume-normalisation flag present in daemon args"
    why_human: "Requires Connect playback session and daemon log inspection"
  - test: "SC-8: Remove all accounts (or fresh install), open Settings"
    expected: "Setup Guide visible at top with 3 numbered steps and developer.spotify.com link"
    why_human: "Visual UI verification in LMS web interface"
  - test: "SC-9: Set LMS language to FR/NL/IT/ES, open Settings"
    expected: "All strings display in selected language, no raw PLUGIN_SPOTON_* key names"
    why_human: "Visual UI verification across multiple languages in LMS"
  - test: "WR-04: Set Format to FLAC, start Connect playback"
    expected: "Connect should use PCM DirectStream regardless (per STREAM_FORMAT_DESC string). Currently code blocks DirectStream for Connect when format is pcm/flac/mp3 -- verify if this causes functional issues"
    why_human: "Requires Connect playback session to test if blocking DirectStream for Connect breaks audio"
---

# Phase 6: Polish + Release Readiness Verification Report

**Phase Goal:** Feature polish (per-player prefs, DSTM, normalization), release preparation (full i18n, all binaries, setup guide, security review), and distribution as LMS custom repository for early adopter feedback
**Verified:** 2026-06-03T14:00:30Z
**Status:** human_needed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | SC-1: Setting bitrate to 96 on one player and 320 on another causes each to stream at configured bitrate independently | VERIFIED (code) | Plugin.pm:1226-1228 reads per-player `bitrateOverride` pref, re-validates with `/^(?:96\|160\|320)$/`, injects into `--bitrate` regex. Settings.pm:165-168 saves pref. basic.html:50-57 has 4-option dropdown. |
| 2 | SC-2: When a playlist ends, DSTM automatically queues a related Spotify track and playback continues | VERIFIED (code) | DontStopTheMusic.pm:21-26 registers handler. Plugin.pm:89-92 registers at init with isEnabled guard. Handler at DontStopTheMusic.pm:32-96 gets seeds via getMixableProperties, classifies them, calls Client->recommendations, falls back to search. |
| 3 | SC-3: Custom librespot binary in designated path is used; --check enforcement still applies | VERIFIED (code) | Helper.pm:35 checks `$prefs->get('binary')` first. Helper.pm:130 puts `spoton-custom` as first candidate in `_findBin()`. helperCheck at line 67 validates via `--check` with shell-safe quoting. |
| 4 | SC-4: Per-player Settings shows toggles for Connect on/off, auto-play (DSTM hint), and transcoding fallback | VERIFIED (code) | basic.html:20-25 Connect checkbox, 40-48 Format-Dropdown (5 options), 50-57 Bitrate Override, 59-61 DSTM hint pointing to Player Settings. |
| 5 | SC-5: Player with transcoding fallback receives audio through custom-convert.conf pipeline | VERIFIED (code) | ProtocolHandler.pm:87-98 canDirectStream returns 0 for pcm/flac/mp3. Plugin.pm:1312-1331 deletes competing pipeline entries so LMS uses the desired one. |
| 6 | SC-6: Volume normalisation setting applies to Connect mode | VERIFIED (code) | Daemon.pm:120 `push @helperArgs, '--enable-volume-normalisation' if $prefs->get('normalization')`. |
| 7 | SC-7: Bundled Client-ID defined in exactly one location | VERIFIED | Client.pm:32 defines `SPOTON_DEFAULT_CLIENT_ID`. TokenManager.pm:28 imports via lazy require. `grep -c '93aac68fb06348598c1e67734dfaceee' Client.pm` = 1. `grep -v '^#' TokenManager.pm \| grep -c '93aac68...'` = 0. |
| 8 | SC-8: Setup guide shows correct step order for new users | VERIFIED (code) | basic.html:4-15 Setup Guide conditional on `NOT accounts.keys.size`, 3 numbered steps with developer.spotify.com/dashboard link. strings.txt:1002-1053 has SETUP_GUIDE_TITLE/STEP1/STEP2/STEP3 in 11 languages. |
| 9 | SC-9: All UI strings translated to standard LMS language set with no missing-key placeholders | VERIFIED | strings.txt: 83 keys, all 11 languages (CS, DA, DE, EN, ES, FR, IT, NL, NO, PL, SV), UTF-8 encoded. Alphabetical order within blocks. TAB indentation. |
| 10 | SC-10: librespot binaries for x86_64, aarch64, armhf, i386 present and pass --check | FAILED | Only x86_64-linux/spoton exists (19MB). aarch64-linux, armhf-linux, i386-linux contain only .gitkeep placeholders. |
| 11 | SC-11: Full security review completed, all HIGH/CRITICAL findings resolved | VERIFIED | 06-05-SUMMARY.md documents ASVS L1 review of 6 modules. 06-REVIEW.md documents code review: 1 critical (determined false positive), 5 warnings, 3 info. No HIGH/CRITICAL unresolved. |
| 12 | SC-12: Plugin installable via LMS custom repository URL | FAILED (partial) | repo.xml exists at root with correct XML structure (version=1.0.0, minTarget=8.0, category=musicservices). But sha and url attributes are PLACEHOLDER values. Plugin cannot actually be installed via repo URL until populated. |

**Score:** 10/12 truths verified

### Deferred Items

Items not yet met but explicitly addressed in planned Phase 6.1 work.

| # | Item | Addressed In | Evidence |
|---|------|-------------|----------|
| 1 | SC-10: Multi-architecture binaries | Phase 6.1 (planned, not yet in ROADMAP) | Plan 05 deferred_to_phase_6_1: "requires cross-compilation for aarch64, armhf, i386" |
| 2 | SC-12: Functional repo.xml with SHA1 and URL | Phase 6.1 (planned, not yet in ROADMAP) | repo.xml template ready; SHA1 and URL await binary builds |

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/API/Client.pm` | Exporter + SPOTON_DEFAULT_CLIENT_ID + recommendations() | VERIFIED | Lines 10-11: Exporter setup. Line 32: constant. Lines 110-141: recommendations(). |
| `Plugins/SpotOn/API/TokenManager.pm` | Imports SPOTON_DEFAULT_CLIENT_ID from Client.pm | VERIFIED | Line 28: lazy require + direct call pattern. No duplicate constant. |
| `Plugins/SpotOn/Helper.pm` | Custom binary override with spoton-custom first | VERIFIED | Line 35: pref binary check. Line 130: spoton-custom first candidate. |
| `Plugins/SpotOn/DontStopTheMusic.pm` | DSTM provider with seed logic + recommendations + search fallback | VERIFIED | 279 lines. registerHandler at line 22. getMixableProperties at line 35. Client->recommendations at line 190. _searchFallback at line 220. |
| `Plugins/SpotOn/Settings.pm` | Per-player bitrateOverride + streamFormat pref handling | VERIFIED | Lines 158-168: streamFormat + bitrateOverride save with validation. Lines 209-211: template vars. |
| `Plugins/SpotOn/ProtocolHandler.pm` | canDirectStream returns 0 for flac/mp3/pcm; formatOverride returns son | VERIFIED | Lines 87-98: streamFormat gate. Line 69: return 'son' for Browse. |
| `Plugins/SpotOn/Plugin.pm` | updateTranscodingTable with per-player bitrate + DSTM registration | VERIFIED | Lines 1226-1228: bitrateOverride. Lines 89-92: DSTM registration. Lines 1304-1334: streamFormat pipeline control. |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | Format-Dropdown + Bitrate-Override + Setup Guide + Credits | VERIFIED | Lines 40-48: 5-option streamFormat. Lines 50-57: bitrateOverride. Lines 4-15: Setup Guide. Lines 189-191: Credits. |
| `Plugins/SpotOn/strings.txt` | 83 keys in 11 languages | VERIFIED | 83 keys (grep -c). 11 languages (CS,DA,DE,EN,ES,FR,IT,NL,NO,PL,SV). UTF-8 encoding. |
| `Plugins/SpotOn/install.xml` | Version 1.0.0 | VERIFIED | Line 19: `<version>1.0.0</version>`. |
| `repo.xml` | LMS custom repository descriptor | VERIFIED (template) | Valid XML structure. version=1.0.0 matches install.xml. SHA1 and URL are placeholders. |
| `Plugins/SpotOn/Bin/x86_64-linux/spoton` | x86_64 binary | VERIFIED | 19MB executable present. |
| `Plugins/SpotOn/Bin/aarch64-linux/spoton` | aarch64 binary | MISSING | Only .gitkeep present. |
| `Plugins/SpotOn/Bin/armhf-linux/spoton` | armhf binary | MISSING | Only .gitkeep present. |
| `Plugins/SpotOn/Bin/i386-linux/spoton` | i386 binary | MISSING | Only .gitkeep present. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| TokenManager.pm | Client.pm | `Plugins::SpotOn::API::Client::SPOTON_DEFAULT_CLIENT_ID()` | WIRED | Line 28: lazy require + direct call. |
| Client.pm | Spotify recommendations API | `_request('get', 'recommendations', ...)` | WIRED | Line 133: recommendations() calls _request. Endpoint in @KNOWN_DEPRECATED_FAMILIES for bundled-token routing. |
| DontStopTheMusic.pm | Client.pm | `Client->recommendations($accountId, ...)` | WIRED | Line 190: calls recommendations(). Line 124: calls search(). |
| Plugin.pm | DontStopTheMusic.pm | `require + init()` in initPlugin() | WIRED | Lines 89-92: require + init() with isEnabled guard. Placed after ProtocolHandlers, before WEBUI. |
| DontStopTheMusic.pm | Slim::Plugin::DontStopTheMusic::Plugin | `registerHandler()` + `getMixableProperties()` | WIRED | Line 22: registerHandler. Line 35: getMixableProperties. |
| Settings.pm | Plugin.pm | bitrateOverride pref read in updateTranscodingTable | WIRED | Plugin.pm:1227 reads `prefs->client($client)->get('bitrateOverride')`. Settings.pm:168 writes it. |
| ProtocolHandler.pm | Settings.pm | streamFormat pref read in canDirectStream/formatOverride | WIRED | ProtocolHandler.pm:54,89 reads `prefs->client($client)->get('streamFormat')`. Settings.pm:161 writes it. |
| basic.html | strings.txt | `[% 'KEY' \| string %]` | WIRED | Setup Guide, Credits, Format-Dropdown, Bitrate Override all use string keys present in strings.txt. |
| repo.xml | install.xml | version attribute match | WIRED | Both contain version 1.0.0. |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|-------------------|--------|
| DontStopTheMusic.pm | seed tracks | getMixableProperties($client, 5) | LMS internal playlist data | FLOWING |
| DontStopTheMusic.pm | recommendations | Client->recommendations() | Spotify API response | FLOWING (requires active account + API access) |
| Settings.pm | bitrateOverride | HTML form POST pref_bitrateOverride | User selection | FLOWING |
| Settings.pm | streamFormat | HTML form POST pref_streamFormat | User selection | FLOWING |
| Plugin.pm | bitrate in commandTable | prefs->client->get('bitrateOverride') | Per-player pref store | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Client.pm exports SPOTON_DEFAULT_CLIENT_ID | `grep -c '93aac68fb06348598c1e67734dfaceee' Client.pm` | 1 (exactly one definition) | PASS |
| TokenManager.pm has no literal Client-ID | `grep -v '^#' TokenManager.pm \| grep -c '93aac68...'` | 0 | PASS |
| strings.txt has 11 languages | `grep -P '^\t[A-Z]{2}\t' \| sort -u \| wc -l` | 11 (CS,DA,DE,EN,ES,FR,IT,NL,NO,PL,SV) | PASS |
| strings.txt has 83+ keys | `grep -c '^PLUGIN_SPOTON'` | 83 | PASS |
| strings.txt is UTF-8 | `file strings.txt` | "Unicode text, UTF-8 text" | PASS |
| install.xml version 1.0.0 | `grep 'version.*1\.0\.0' install.xml` | Found | PASS |
| repo.xml version matches | `grep 'version="1.0.0"' repo.xml` | Found | PASS |
| All 9 plan commits exist | `git log --oneline \| grep` | All 9 found | PASS |

### Probe Execution

Step 7c: SKIPPED (no probes declared for Phase 6, no scripts/*/tests/probe-*.sh found)

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| LMS-03 | 06-04, 06-05 | i18n support via LMS strings mechanism for EN, DE, FR, NL, IT, ES, SV, NO, DA, PL, CS | SATISFIED | 83 keys in 11 languages in strings.txt |
| LMS-06 | 06-05 | Multi-architecture binaries (x86_64, aarch64, armhf, i386) | BLOCKED | Only x86_64 binary present. Others are .gitkeep placeholders. Deferred to Phase 6.1. |
| LMS-08 | 06-02 | Player-specific preferences (bitrate, normalization, Connect on/off) | SATISFIED | bitrateOverride per player in Settings.pm/Plugin.pm. streamFormat per player. Connect enable/disable per player. Normalization global (Daemon.pm:120). |
| LMS-09 | 06-03 | Don't Stop The Music integration for auto-play after playlist end | SATISFIED | DontStopTheMusic.pm registered in Plugin.pm. Seeds via getMixableProperties, recommendations via bundled-token, search fallback. |
| LMS-10 | 06-01 | Custom binary support (user-provided binary override) | SATISFIED | Helper.pm:35 checks pref binary first. _findBin:130 puts spoton-custom first. helperCheck validates via --check. |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| ProtocolHandler.pm | 53-57 | Dead variable: `$fmt` read but never used in formatOverride() | INFO | Pipeline selection works correctly via updateTranscodingTable deletion; $fmt is vestigial code. Not functional issue. |
| Settings.pm | 151-155 | Dead code: connectOggOverride save handler for removed HTML form element | WARNING | Dead code path. HTML form no longer sends pref_connectOggOverride. Migration read (line 211) is still useful. |
| basic.html | 9, 96 | `target="_blank"` without `rel="noopener noreferrer"` | WARNING | Security hardening: external links to developer.spotify.com. Minor risk in LMS context (local web UI). |
| ProtocolHandler.pm | 87-98 | canDirectStream blocks pcm/flac/mp3 for Connect URLs too | WARNING | String says "Affects Browse mode only" but code blocks DirectStream for Connect when format is pcm/flac/mp3. Mismatch between documentation and behavior. |
| Plugin.pm | 1261-1262 | _baseSonPipelines restore uses `unless exists` instead of unconditional overwrite | WARNING | Fragile: modified entries not restored to original. Currently functional because regex substitutions overwrite all variable parts, but brittle for future changes. |
| DontStopTheMusic.pm | 223 | `int(rand(40))` offset may exceed result count in Dev Mode | WARNING | Dev Mode search limit is 10. Offset >30 often produces empty results. Fallback silently fails. Not a blocker but reduces DSTM reliability. |
| repo.xml | 28-29 | PLACEHOLDER_SHA1_PHASE_6_1 and PLACEHOLDER_URL_PHASE_6_1 | INFO | Documented and intentional. Deferred to Phase 6.1. |

### Human Verification Required

### 1. Per-Player Bitrate Override (SC-1)

**Test:** Open LMS Settings > SpotOn. Select Player A, set Bitrate Override to 96 kbps. Select Player B, set Bitrate Override to 320 kbps. Play a track on each player.
**Expected:** Log shows `--bitrate 96` for Player A and `--bitrate 320` for Player B in updateTranscodingTable output.
**Why human:** Requires running LMS with two configured players and active Spotify streaming.

### 2. DSTM Auto-Play (SC-2)

**Test:** LMS Player Settings > Don't Stop The Music > select "SpotOn Empfehlungen". Play a short Spotify playlist (2-3 tracks). Let playlist end.
**Expected:** DSTM queues additional tracks automatically, playback continues without user intervention.
**Why human:** Requires running LMS with active Spotify account, real playlist, and DSTM end-of-queue timing.

### 3. Custom Binary Support (SC-3)

**Test:** Copy the x86_64 binary to `Bin/x86_64-linux/spoton-custom`. Restart LMS.
**Expected:** Settings page shows spoton-custom path as active binary.
**Why human:** Requires LMS restart and binary filesystem interaction.

### 4. Per-Player Settings Page (SC-4)

**Test:** Open Settings > SpotOn, select a player.
**Expected:** Format-Dropdown (Auto/OGG/PCM/FLAC/MP3) and Bitrate Override visible in per-player section.
**Why human:** Visual UI verification in LMS web interface.

### 5. Transcoding Fallback (SC-5)

**Test:** Set Format to FLAC for a player, play a track.
**Expected:** canDirectStream returns 0 in logs, transcoding pipeline used instead of DirectStream.
**Why human:** Requires running LMS with active streaming and log inspection.

### 6. Volume Normalization in Connect (SC-6)

**Test:** Enable Normalization in global settings, start Connect playback.
**Expected:** `--enable-volume-normalisation` flag present in daemon startup arguments.
**Why human:** Requires Connect playback session and daemon log inspection.

### 7. Setup Guide for New Users (SC-8)

**Test:** Remove all accounts (or use fresh install), open Settings > SpotOn.
**Expected:** Setup Guide visible at top with 3 numbered steps and developer.spotify.com link.
**Why human:** Visual UI verification in LMS web interface.

### 8. i18n Across Languages (SC-9)

**Test:** Set LMS language to FR (or NL, IT, ES, etc.), open Settings > SpotOn.
**Expected:** All strings display in selected language, no raw PLUGIN_SPOTON_* key names visible.
**Why human:** Visual UI verification across multiple languages in LMS.

### 9. WR-04: Connect DirectStream vs Format Dropdown

**Test:** Set Format to FLAC, start Connect playback from Spotify app.
**Expected:** Connect should use PCM DirectStream regardless of format selection (per STREAM_FORMAT_DESC string). Verify if blocking DirectStream for Connect when format is pcm/flac/mp3 causes audio issues.
**Why human:** Requires Connect playback session to test functional impact of the code-documentation mismatch.

### Gaps Summary

Two Success Criteria are not met:

**SC-10 (Multi-architecture binaries):** Only x86_64-linux binary exists. aarch64, armhf, and i386 directories have only .gitkeep placeholders. This was explicitly deferred to a "Phase 6.1" in Plan 05, but Phase 6.1 is not yet formally defined in ROADMAP.md with its own success criteria and plan structure.

**SC-12 (Plugin installable via repository):** repo.xml template is structurally correct and ready, but SHA1 and download URL are placeholders. The plugin cannot be installed via LMS custom repository until these are populated. This is coupled to SC-10 (binaries must be built first to compute SHA1).

Both gaps are deferred items with clear documentation in Plan 05. They require a Phase 6.1 to be created in ROADMAP.md for formal tracking.

Additionally, the code review (06-REVIEW.md) identified 5 warnings that remain unresolved:
- WR-01: _baseSonPipelines fragile restore pattern
- WR-02: DSTM search fallback offset may exceed Dev Mode limits
- WR-03: Dead connectOggOverride save code
- WR-04: canDirectStream blocks Connect DirectStream for pcm/flac/mp3 (contradicts UI strings)
- WR-05: Missing rel="noopener" on external links

None of these are blockers for the phase goal. WR-04 is the most functionally significant and should be verified during human UAT.

---

_Verified: 2026-06-03T14:00:30Z_
_Verifier: Claude (gsd-verifier)_
