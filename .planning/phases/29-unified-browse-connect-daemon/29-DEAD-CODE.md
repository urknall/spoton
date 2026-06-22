# Phase 30 Dead Code Manifest

> Phase 29 adds the Unified daemon alongside old modules. After Phase 29 is verified stable, Phase 30 removes the following:

## Perl Modules to Remove

| Module | LOC | Replacement |
|--------|-----|-------------|
| `Plugins::SpotOn::Browse::DaemonManager` | 344 | `Plugins::SpotOn::Unified::DaemonManager` |
| `Plugins::SpotOn::Browse::Daemon` | 243 | `Plugins::SpotOn::Unified::Daemon` |
| `Plugins::SpotOn::Connect::DaemonManager` | 415 | `Plugins::SpotOn::Unified::DaemonManager` |
| `Plugins::SpotOn::Connect::Daemon` | 377 | `Plugins::SpotOn::Unified::Daemon` |

## Rust Code Paths (Phase 30 Candidates)

| Item | Location | Status After Phase 29 |
|------|----------|----------------------|
| `Mode::Browse` | main.rs line 84 | Dead if `--browse` deprecated |
| `Mode::Connect` | main.rs line 80 | Dead if `--connect` deprecated |
| `browse::run_browse()` entry point | browse.rs | Superseded by `unified::run_unified()` |
| `connect::run_connect()` entry point | connect.rs | Superseded by `unified::run_unified()` |
| `browse_http_server()` standalone | browse.rs | Routes moved into unified server |
| `http_stream_server()` standalone | connect.rs | Routes moved into unified server |

## Plugin.pm Code Paths

| Code Path | Location | Change in Phase 30 |
|-----------|----------|---------------------|
| `_startBrowseDaemons()` timer | Plugin.pm line 127 | Remove (replaced by Unified init) |
| Browse PID exclusion in `_killOrphanedProcesses` | Plugin.pm line 296-300 | Remove (Unified exclusion covers it) |
| `diagnosticMode` change handler for Browse DM | Plugin.pm line 141-150 | Remove |

## ProtocolHandler.pm Code Paths

| Code Path | Location | Change in Phase 30 |
|-----------|----------|---------------------|
| Browse::DaemonManager `require` and lookup | ProtocolHandler.pm lines 89-96, 137-161, 324-342 | Replace with Unified::DaemonManager |
| Connect::DaemonManager `require` and lookup | ProtocolHandler.pm lines 74-79, 189-211, 350-367 | Replace with Unified::DaemonManager |

## Preferences

| Key | Current Use | Phase 30 Status |
|-----|-------------|-----------------|
| `browseMode` | Gates Browse daemon vs. pipe fallback | Remove toggle; unified daemon is always HTTP |
| `daemonMode` | Phase 29 migration toggle (legacy/unified) | Remove; unified becomes the only mode |
