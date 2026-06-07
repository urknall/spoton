---
phase: 14-connect-fixes
verified: 2026-06-07T13:30:00Z
status: human_needed
score: 5/5
overrides_applied: 0
human_verification:
  - test: "Connect two different Spotify accounts to two different LMS players simultaneously. Verify the first player's Browse session continues showing their own library."
    expected: "Each player shows its own Spotify library. No credential overwrite occurs."
    why_human: "Requires two Spotify Premium accounts and two LMS players. Cannot verify multi-player credential isolation programmatically."
  - test: "Start a Spotify Connect session on a player. Check the Spotify app volume indicator immediately after connection."
    expected: "The Spotify app shows the same volume as the LMS player (within 3 seconds of session start). No volume jump from a hardcoded 50% default."
    why_human: "Requires live Spotify Connect session and visual confirmation of volume in Spotify app."
  - test: "During an active Connect session, change volume in the Spotify app. Measure how quickly LMS reflects the change."
    expected: "LMS volume updates within 3 seconds (down from previous 20-second grace period)."
    why_human: "Requires timing measurement of runtime volume sync behavior across Spotify app and LMS."
---

# Phase 14: Connect Fixes Verification Report

**Phase Goal:** Connect sessions start with correct volume and each player's Spotify credentials are isolated from other players and other users
**Verified:** 2026-06-07T13:30:00Z
**Status:** human_needed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Each Connect daemon uses a per-player cache directory `spoton/connect-{mac}/` preventing credential overwrite | VERIFIED | Daemon.pm:90 `catdir($serverPrefs->get('cachedir'), 'spoton', 'connect-' . $self->id)`. `activeAccount` reference count = 0 in Daemon.pm. TokenManager.pm browse path unchanged (still uses `activeAccount`). |
| 2 | TokenManager.pm Browse-Token path remains unchanged | VERIFIED | TokenManager.pm lines 86, 109, 111: `activeAccount` still used for browse token cache dir. No changes to TokenManager.pm in Phase 14 commits. |
| 3 | Connect daemon starts with `--volume-ctrl linear` and `--initial-volume` matching LMS player volume | VERIFIED | Daemon.pm:124 `push @helperArgs, '--volume-ctrl', 'linear'`. Daemon.pm:125 `push @helperArgs, '--initial-volume', int($client->volume // 50)`. main.rs:201-205 parses `--initial-volume` as u8 with `.min(100)` clamp. main.rs:264 converts to u16 via `(v as u32 * 65535 / 100) as u16`. connect.rs:852-856 dispatches `volume_ctrl_str` to `VolumeCtrl::Linear/Fixed/Log`. connect.rs:860 applies to primary MixerConfig. connect.rs:979 applies initial_volume to primary ConnectConfig. connect.rs:1074 applies to reconnect ConnectConfig. connect.rs:1084 applies to reconnect MixerConfig. |
| 4 | Volume events from Spotify are processed within 3 seconds of daemon start (not 20) | VERIFIED | Connect.pm:27 `use constant VOLUME_GRACE_PERIOD => 3;`. Connect.pm:524 used in comparison `uptime($client->id) < VOLUME_GRACE_PERIOD`. |
| 5 | Perl syntax checks pass for Connect.pm and Connect/Daemon.pm in CI | VERIFIED | `prove t/05_perl_syntax.t` exits 0 with 8/8 tests ok (9 entries in `@pm_files`, 8 found on disk -- Callback.pm not yet created, filtered by existing `grep { -f $_ }` pattern). Test 7 = Connect.pm, Test 8 = Daemon.pm. Full suite: 232/232 pass. |

**Score:** 5/5 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `librespot-spoton/src/main.rs` | `--initial-volume` and `--volume-ctrl` CLI flag parsing + `run_connect()` call update | VERIFIED | Lines 112-113: variable declarations. Lines 201-211: match arms. Line 264: u16 conversion. Lines 339-350: `run_connect()` call with `initial_volume_u16` and `&volume_ctrl_str`. |
| `librespot-spoton/src/connect.rs` | `run_connect()` signature with `initial_volume` and `volume_ctrl_str`, VolumeCtrl import, MixerConfig/ConnectConfig updates | VERIFIED | Line 44: `VolumeCtrl` import. Lines 830-831: signature params. Lines 851-856: enum dispatch. Line 860: primary MixerConfig. Line 979: primary ConnectConfig. Line 1074: reconnect ConnectConfig. Line 1084: reconnect MixerConfig. |
| `Plugins/SpotOn/Connect/Daemon.pm` | Per-player cache dir isolation + `--volume-ctrl linear` + `--initial-volume` args | VERIFIED | Line 90: `connect-{id}` cache dir. Line 124: `--volume-ctrl linear`. Line 125: `--initial-volume` with `int($client->volume // 50)`. |
| `Plugins/SpotOn/Connect.pm` | Reduced grace period constant | VERIFIED | Line 27: `use constant VOLUME_GRACE_PERIOD => 3;` (was 20). |
| `t/05_perl_syntax.t` | Syntax coverage for Connect.pm and Daemon.pm | VERIFIED | Lines 22-23: both entries in `@pm_files`. Lines 268-325: 6 stubs (Accessor, Client, Sync, Request, Music::Info, Player::Source). 8/8 found files pass. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `main.rs` | `connect.rs` | `run_connect()` call passing `initial_volume_u16` and `&volume_ctrl_str` | WIRED | main.rs:348-349 passes both as trailing args. connect.rs:830-831 receives them in matching signature positions. |
| `Daemon.pm` | librespot binary | `@helperArgs --volume-ctrl linear --initial-volume` | WIRED | Daemon.pm:124-125 pushes both flags. main.rs:201-211 parses them. |
| `Daemon.pm` | filesystem | `catdir` for per-player cache dir | WIRED | Daemon.pm:90 constructs path with `connect-{id}`. Line 91: stored via `$self->cache($cacheDir)`. Line 101: passed as `-c` arg. |
| `Connect.pm` | volume handler | `VOLUME_GRACE_PERIOD` constant consumed by volume handler | WIRED | Connect.pm:27 defines constant = 3. Connect.pm:524 uses it in `uptime() < VOLUME_GRACE_PERIOD` comparison. |

### Data-Flow Trace (Level 4)

Not applicable -- phase modifies daemon startup configuration and constants, not UI-rendered dynamic data.

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Perl syntax test passes | `prove t/05_perl_syntax.t` | 8/8 ok, exit 0 | PASS |
| Full test suite passes | `prove -l t/` | 232/232 ok, exit 0 | PASS |
| Per-player cache dir pattern in Daemon.pm | `grep "connect-" Daemon.pm` | Line 90: `'connect-' . $self->id` | PASS |
| Volume flags in Daemon.pm | `grep "volume-ctrl" Daemon.pm` | Line 124: `'--volume-ctrl', 'linear'` | PASS |
| Initial volume in Daemon.pm | `grep "initial-volume" Daemon.pm` | Line 125: `'--initial-volume', int(...)` | PASS |
| Grace period updated | `grep "VOLUME_GRACE_PERIOD => 3" Connect.pm` | Line 27 matches | PASS |
| activeAccount removed from Daemon.pm | `grep -c "activeAccount" Daemon.pm` | 0 | PASS |
| VolumeCtrl import in connect.rs | `grep "VolumeCtrl" connect.rs` | Line 44: imported from librespot_playback::config | PASS |
| Both ConnectConfig blocks use dynamic volume | `grep "initial_volume" connect.rs` | Lines 979, 1074: both use `unwrap_or(u16::MAX / 2)` | PASS |
| Both MixerConfig blocks use volume_ctrl_enum | `grep "volume_ctrl_enum" connect.rs` | Lines 860, 1084: both set `volume_ctrl: volume_ctrl_enum` | PASS |
| All 4 commits exist | `git log --oneline` | de9c67c, a620093, e47c2c1, f2be02f all present | PASS |

### Probe Execution

No probes found for this phase. Step 7c: SKIPPED (no probe scripts).

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| CON-01 | 14-02 | Connect daemon uses separate cache directory per player MAC (`spoton/connect-{mac}/`), preventing credential overwrite | SATISFIED | Daemon.pm:90 uses `connect-{id}` path. `activeAccount` removed (count=0). TokenManager.pm browse path unchanged. |
| CON-02 | 14-01, 14-02 | Connect volume matches LMS player volume at session start | SATISFIED | Binary: `--initial-volume` parsed (main.rs:201), converted to u16 (main.rs:264), applied to both ConnectConfig blocks (connect.rs:979, 1074). `--volume-ctrl linear` applied to both MixerConfig blocks (connect.rs:860, 1084). Perl: Daemon.pm:124-125 passes both flags. |
| CON-03 | 14-02 | Connect volume changes sync within 3 seconds of user action | SATISFIED | Connect.pm:27 `VOLUME_GRACE_PERIOD => 3` (was 20). Used at Connect.pm:524. |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| (none) | - | No TBD/FIXME/XXX/TODO/HACK/PLACEHOLDER markers found | - | Clean |

### Human Verification Required

### 1. Multi-Player Credential Isolation (CON-01)

**Test:** Connect two different Spotify accounts to two different LMS players simultaneously. After both are connected, navigate Browse > Library on the first player.
**Expected:** The first player's Browse session shows their own Spotify library (playlists, liked songs). The second connection does not overwrite the first player's credentials.
**Why human:** Requires two Spotify Premium accounts and two LMS players. Multi-player credential isolation cannot be verified programmatically.

### 2. Volume Match at Connect Start (CON-02)

**Test:** Set LMS player volume to a specific value (e.g., 75%). Start a Spotify Connect session on that player. Immediately check the Spotify app volume indicator.
**Expected:** The Spotify app shows approximately 75% volume within 3 seconds of connection. No jarring volume jump from the old hardcoded 50% default.
**Why human:** Requires live Spotify Connect session and visual confirmation of volume level in the Spotify app.

### 3. Volume Sync Speed (CON-03)

**Test:** During an active Connect session, change volume in the Spotify app. Observe how quickly LMS reflects the change.
**Expected:** LMS volume updates within 3 seconds (reduced from the previous 20-second grace period).
**Why human:** Requires timing measurement of runtime volume sync behavior across Spotify app and LMS interface.

### Gaps Summary

No code-level gaps found. All 5 observable truths are verified in the codebase. All 3 requirements (CON-01, CON-02, CON-03) have supporting implementation evidence. All 4 commits exist and match the documented changes.

Three items require human verification -- all involve live Spotify Connect sessions with real hardware that cannot be simulated programmatically. The code paths are fully verified and correctly wired.

---

_Verified: 2026-06-07T13:30:00Z_
_Verifier: Claude (gsd-verifier)_
