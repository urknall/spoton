---
phase: quick-260615-jub
plan: 01
subsystem: diagnostics
tags: [diag, logging, observability, debug]
dependency-graph:
  requires: [diagnosticMode pref]
  provides: [comprehensive DIAG event coverage across all modules]
  affects: [Daemon.pm, ProtocolHandler.pm, Connect.pm, TokenManager.pm, Client.pm]
tech-stack:
  patterns: ["[DIAG] prefix with diagnosticMode guard", "warn-level logging for grep-ability"]
key-files:
  modified:
    - Plugins/SpotOn/Connect/Daemon.pm
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/Connect.pm
    - Plugins/SpotOn/API/TokenManager.pm
    - Plugins/SpotOn/API/Client.pm
decisions:
  - "All DIAG lines use $log->warn level (not info/debug) for universal grep visibility"
  - "Account IDs redacted to first 4 chars + **** in all token-related DIAG lines"
  - "api_slow threshold set at 2s with dual guard (duration AND diagnosticMode)"
metrics:
  duration: 6m 48s
  completed: 2026-06-15T12:27:46Z
  tasks: 6/6
  new-diag-lines: 36
  total-diag-lines: 43
  tests: 316 passed (12 files)
---

# Quick Task 260615-jub: Comprehensive DIAG Events Across All Modules Summary

36 new [DIAG] log points across 5 modules for universal remote debugging via diagnosticMode pref toggle

## Task Results

| Task | Name | Commit | Files | DIAG Lines |
|------|------|--------|-------|------------|
| 1 | Daemon.pm Lifecycle DIAG | 16af324 | Plugins/SpotOn/Connect/Daemon.pm | 5 (daemon_start, daemon_port_announce, daemon_stop, crash_loop_disable, crash_loop_reset) |
| 2 | ProtocolHandler.pm Stream Routing DIAG | 7745111 | Plugins/SpotOn/ProtocolHandler.pm | 6 (canDirectStream x2, canEnhanceHTTP, connect_sync_proxy, formatOverride x2) |
| 3 | Connect.pm Extended Events DIAG | 7a5bfd8 | Plugins/SpotOn/Connect.pm | 10 (startOffset_adjust, echo_suppressed x2, volume_from/to_binary, seek_from/to_binary, metadata_fetch/stale/success) |
| 4 | Connect.pm Control Commands DIAG | c02dab7 | Plugins/SpotOn/Connect.pm | 4 (control_cmd_sent/ok/fail, web_api_fallback) |
| 5 | TokenManager.pm Auth DIAG | b158a80 | Plugins/SpotOn/API/TokenManager.pm | 6 (token_refresh_ok/fail, token_parse_fail, discovery_start/credential, account_stored) |
| 6 | Client.pm API DIAG | d834cc1 | Plugins/SpotOn/API/Client.pm | 5 (api_slow, api_429, api_401, api_bundled_fallback, api_error) |

## DIAG Event Coverage Map

| Module | Category | Events |
|--------|----------|--------|
| Daemon.pm | Lifecycle | daemon_start, daemon_port_announce, daemon_stop |
| Daemon.pm | Crash Protection | crash_loop_disable, crash_loop_reset |
| ProtocolHandler.pm | Stream Routing | canDirectStream (synced, single_player), formatOverride (soc, son) |
| ProtocolHandler.pm | Proxy | canEnhanceHTTP, connect_sync_proxy |
| Connect.pm | Position | startOffset_adjust |
| Connect.pm | Echo Suppression | echo_suppressed (1s pause, 3s grace) |
| Connect.pm | Binary Events | volume_from_binary, seek_from_binary |
| Connect.pm | LMS-to-Binary | volume_to_binary, seek_to_binary |
| Connect.pm | Control Commands | control_cmd_sent, control_cmd_ok, control_cmd_fail, web_api_fallback |
| Connect.pm | Metadata | metadata_fetch, metadata_stale, metadata_success |
| TokenManager.pm | Token | token_refresh_ok, token_refresh_fail, token_parse_fail |
| TokenManager.pm | Discovery | discovery_start, discovery_credential |
| TokenManager.pm | Account | account_stored |
| Client.pm | Performance | api_slow (>2s) |
| Client.pm | Rate Limiting | api_429 |
| Client.pm | Auth | api_401 |
| Client.pm | Fallback | api_bundled_fallback |
| Client.pm | Errors | api_error |

## Deviations from Plan

None - plan executed exactly as written.

## Security Verification

- No raw access tokens logged in any DIAG line (verified: 0 matches for accessToken/Bearer in DIAG lines)
- All account IDs redacted to first 4 chars + **** in TokenManager.pm and Client.pm
- web_api_fallback also redacts account ID

## Known Stubs

None.

## Test Results

All 316 tests pass across 12 test files. No regressions.
