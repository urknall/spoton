---
phase: 10-connect-dstm
plan: 01
subsystem: binary
tags: [librespot, rust, spirc, autoplay, sessionconfig, cli]

# Dependency graph
requires:
  - phase: 05-connect
    provides: connect.rs run_connect() function and main.rs CLI structure
provides:
  - "--autoplay on/off CLI flag parsed in librespot-spoton binary"
  - "SessionConfig.autoplay override set before Session::new() in connect.rs"
  - '"autoplay": true capability in --check JSON manifest'
  - "run_connect() signature accepts autoplay: Option<bool> as 8th parameter"
affects:
  - 10-02  # Daemon.pm passes --autoplay to the binary; binary must support it
  - 10-03  # Settings.pm capability check relies on --check JSON autoplay:true

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Option<bool> flag pattern: CLI 'on'/'off' strings map to Some(true)/Some(false)"
    - "SessionConfig mutation before Session::new() — required for config to take effect"

key-files:
  created: []
  modified:
    - librespot-spoton/src/main.rs
    - librespot-spoton/src/connect.rs

key-decisions:
  - "D-04: --autoplay flag uses 'on'/'off' strings (not 'true'/'false') matching Spotty-NG convention"
  - "D-05: session_config.autoplay set BEFORE Session::new() per Pitfall 4 — SessionConfig is cloned at session creation"
  - "D-06: --check JSON key is 'autoplay' (not 'enable-autoplay') to match getCapability() call in Helper.pm"

patterns-established:
  - "Option<bool> CLI flag: declare None; match arm maps string values; pass as-is to function"

requirements-completed:
  - DSTM-01
  - DSTM-02
  - DSTM-03
  - DSTM-04

# Metrics
duration: 12min
completed: 2026-06-04
---

# Phase 10 Plan 01: Binary Autoplay Support Summary

**Spirc-native autoplay wired to CLI: --autoplay on/off flag sets SessionConfig.autoplay before Session::new(), reported as capability in --check JSON**

## Performance

- **Duration:** ~12 min
- **Started:** 2026-06-04T12:50:00Z
- **Completed:** 2026-06-04T13:02:50Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments
- Added `--autoplay on/off` CLI flag to main.rs with proper `on`/`off` string mapping to `Option<bool>`
- Added `"autoplay": true` to `--check` JSON manifest — enables `Helper->getCapability('autoplay')` in Plan 03
- Extended `run_connect()` signature with `autoplay: Option<bool>` as 8th parameter
- Set `session_config.autoplay` before `Session::new()` per critical Pitfall 4 ordering constraint
- `cargo check` passes with zero errors after both tasks

## Task Commits

Each task was committed atomically:

1. **Task 1: Add --autoplay flag parsing and --check capability to main.rs** - `73c62bc` (feat)
2. **Task 2: Add autoplay parameter to run_connect() in connect.rs** - `a30fd6f` (feat)

**Plan metadata:** (committed after SUMMARY)

## Files Created/Modified
- `librespot-spoton/src/main.rs` - Variable declaration, --autoplay match arm, --check JSON key, run_connect() call site
- `librespot-spoton/src/connect.rs` - run_connect() signature + SessionConfig.autoplay set before Session::new()

## Decisions Made
- Committed Task 1 (main.rs) and Task 2 (connect.rs) as separate commits even though cargo check requires both simultaneously — this is acceptable for git history clarity
- SessionConfig override uses `if let Some(ap) = autoplay` pattern to preserve `None` default (follows Spotify user setting) when flag not provided
- No changes to reconnect path at ~line 1053 — `session_config.clone()` already carries autoplay field (Pitfall 5 is not a problem)

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
- Edit tool rejected absolute paths from shared repo (`/home/sti/spoton/...`) — worktree copy requires worktree-relative paths (`/home/sti/spoton/.claude/worktrees/agent-af1f47b549b627cb7/...`). Resolved immediately by using worktree root.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Binary source changes complete; Plan 02 (DaemonManager + Plugin.pm) can now add `--autoplay on/off` to daemon startup args
- Plan 03 (Settings.pm + UI) can use `Helper->getCapability('autoplay')` once binary is rebuilt
- Binary rebuild (D-07, all 8 platform targets) is required before `getCapability('autoplay')` returns true in production; can be done at end of Phase 10

---
*Phase: 10-connect-dstm*
*Completed: 2026-06-04*
