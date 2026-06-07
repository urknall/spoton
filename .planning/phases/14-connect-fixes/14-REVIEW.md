---
phase: 14-connect-fixes
reviewed: 2026-06-07T14:30:00Z
depth: standard
files_reviewed: 5
files_reviewed_list:
  - Plugins/SpotOn/Connect.pm
  - Plugins/SpotOn/Connect/Daemon.pm
  - librespot-spoton/src/connect.rs
  - librespot-spoton/src/main.rs
  - t/05_perl_syntax.t
findings:
  critical: 0
  warning: 3
  info: 2
  total: 5
status: issues_found
---

# Phase 14: Code Review Report

**Reviewed:** 2026-06-07T14:30:00Z
**Depth:** standard
**Files Reviewed:** 5
**Status:** issues_found

## Summary

Phase 14 implements three Connect fixes: per-player cache dir isolation (CON-01), volume CLI flags (CON-02), and reduced volume grace period (CON-03). The implementation is structurally correct -- both primary and reconnect paths in connect.rs are consistently updated with dynamic `initial_volume` and `volume_ctrl_enum`, the Perl Daemon.pm correctly passes `--volume-ctrl linear` and `--initial-volume` with proper `int()` + `// 50` fallback, and the cache dir uses `$self->id` (MAC without colons, safe for filesystem paths).

No critical issues found. The codebase has solid loop-prevention (source-marking, debounce, grace periods) and the volume conversion math is correct at each boundary in isolation. Three warnings address a systematic volume precision loss in round-trips, a panic path in the reconnect mixer, and a missing test stub. Two info items cover minor code quality observations.

## Warnings

### WR-01: Volume conversion asymmetry causes systematic -1 drift per user action

**File:** `librespot-spoton/src/connect.rs:228` and `librespot-spoton/src/main.rs:264`
**Issue:** The LMS-to-librespot conversion uses `v * 65535 / 100` (forward) while the librespot-to-LMS conversion uses `vol * 100 / 65535` (reverse). These are not proper inverses under integer division. For 95 out of 101 LMS volume values, the round-trip loses exactly 1: LMS 50 -> u16 32767 -> LMS 49. The same asymmetry exists in `/control/volume` (line 717) which uses the same forward formula.

Source-marking prevents an oscillation loop (the echoed volume event does not bounce back), but each *user-initiated* volume change from the Spotify app will set LMS to `volume - 1`. For example: user sets Spotify volume to what maps to LMS 50, LMS sees 49. If the user then adjusts again from LMS, the Spotify side receives 49, reports back 48, etc. This drift is bounded by user patience, not code guards.

**Fix:** Use rounding in the reverse conversion (connect.rs:228):
```rust
// Before:
let pct = u32::from(*volume) * 100 / 65535;
// After:
let pct = (u32::from(*volume) * 100 + 32767) / 65535;
```
This makes the round-trip stable: LMS 50 -> u16 32767 -> LMS 50 for all values 0-100.

### WR-02: Reconnect mixer creation uses unwrap_or_else(panic) -- crash without diagnostics

**File:** `librespot-spoton/src/connect.rs:1084`
**Issue:** The reconnect path creates a new mixer via `mixer_fn(MixerConfig { ... }).unwrap_or_else(|_| panic!("mixer"))`. If mixer creation fails, this panics with the unhelpful message "mixer" and no error detail. The primary path (line 860) uses `?` for proper error propagation. The reconnect path cannot use `?` directly since it is inside a `tokio::select!` branch, but the error should at least be logged before the panic or the branch should break the loop with an error message (as the existing `process::exit(1)` pattern does on line 1100).

**Fix:** Log the error and exit gracefully instead of panicking:
```rust
// Before:
mixer_fn(MixerConfig { volume_ctrl: volume_ctrl_enum, ..MixerConfig::default() })
    .unwrap_or_else(|_| panic!("mixer")),
// After:
match mixer_fn(MixerConfig { volume_ctrl: volume_ctrl_enum, ..MixerConfig::default() }) {
    Ok(m) => m,
    Err(e) => {
        eprintln!("Mixer creation failed on reconnect: {e}");
        process::exit(1);
    }
}
```

### WR-03: Slim::Player::Source stub missing from t/05_perl_syntax.t but test passes by coincidence

**File:** `t/05_perl_syntax.t`
**Issue:** Connect.pm uses `Slim::Player::Source::songTime($client)` on line 414. The test file includes a stub for `Slim::Player::Source` (lines 317-322), so this is actually handled. However, the test does NOT include `Plugins/SpotOn/Connect/DaemonManager.pm` in the `@pm_files` array -- DaemonManager.pm exists and is heavily referenced by Connect.pm via `require`. Since Connect.pm uses `require` (runtime load), `perl -c` does not fail. But if DaemonManager.pm were ever changed to a compile-time `use`, the test would break with no stub coverage. This is a gap in test completeness rather than a correctness bug.

**Fix:** Add DaemonManager.pm to the `@pm_files` array:
```perl
"$project_dir/Plugins/SpotOn/Connect/DaemonManager.pm",
```

## Info

### IN-01: uptime() mixes Time::HiRes and integer time() granularity

**File:** `Plugins/SpotOn/Connect/Daemon.pm:354`
**Issue:** The `uptime()` method computes `Time::HiRes::time() - ($self->_startTimes->[-1] || time())`. The `_startTimes` array is populated with `time()` (integer seconds, line 262), while the current time uses `Time::HiRes::time()` (microsecond precision). The fallback `time()` is also integer-granularity. This means the returned uptime has up to ~1 second of jitter compared to the actual start time. With a 3-second `VOLUME_GRACE_PERIOD`, this jitter means the effective grace window is 2-4 seconds. This is acceptable but the inconsistency could be cleaned up.

**Fix:** Store `Time::HiRes::time()` instead of `time()` in `_checkStartTimes`:
```perl
push @{$self->_startTimes}, Time::HiRes::time();
```

### IN-02: _name_provided variable set but never read

**File:** `librespot-spoton/src/main.rs:88`
**Issue:** The variable `_name_provided` is set to `true` on line 222 when `--name`/`-n` is parsed, but is never read anywhere in the code. The leading underscore suppresses the Rust unused-variable warning. This is dead code from an earlier iteration that likely checked whether a device name was explicitly provided.

**Fix:** Remove the variable declaration on line 88 and the assignment on line 222, or use it to validate that `--name` is required for `Mode::Connect`.

---

_Reviewed: 2026-06-07T14:30:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
