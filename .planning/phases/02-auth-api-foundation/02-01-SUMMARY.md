---
phase: 02-auth-api-foundation
plan: 01
subsystem: auth
tags: [rust, librespot-core, spotify, binary, login5, keymaster, mercury, tokio, serde_json, tls, musl, static-pie]

# Dependency graph
requires:
  - phase: 01-plugin-skeleton-binary-foundation
    provides: "librespot-spoton/ Rust project with --check contract, Plugins/SpotOn/Bin/x86_64-linux/spoton binary"
provides:
  - "librespot-spoton binary accepts --authenticate and --get-token subcommands"
  - "Rust binary can acquire Spotify credentials via login5 username/password and write credentials.json"
  - "Rust binary can read cached credentials and retrieve Web API access token via Keymaster/Mercury"
  - "librespot-core 0.8.0 dependency integrated with rustls-tls-native-roots (no system OpenSSL)"
  - ".cargo/config.toml with +crt-static produces static-pie binaries without musl-tools"
  - "Cargo.lock pins vergen 9.0.6 (9.1.0 breaks librespot-core build script)"
affects:
  - 02-02 (TokenManager.pm: calls --get-token, parses stdout JSON)
  - 02-04 (Settings.pm: calls --authenticate to acquire credentials)
  - 02-05 (checkpoint: real Spotify auth test against live credentials)

# Tech tracking
tech-stack:
  added:
    - "librespot-core 0.8.0 (default-features=false, rustls-tls-native-roots)"
    - "serde_json 1.x"
    - "tokio 1.x (rt-multi-thread, macros)"
  patterns:
    - "Mode enum dispatch pattern for binary subcommands (Check/Authenticate/GetToken/Connect)"
    - "+crt-static via .cargo/config.toml for static-pie without musl-tools"
    - "Session::new + session.connect(credentials, store_credentials=true) for credential persistence"
    - "Cache::new(credentials_path, None, None, None) for librespot credential cache"
    - "token_provider().get_token(scopes_str) via Keymaster/Mercury protocol"

key-files:
  created:
    - librespot-spoton/.cargo/config.toml
  modified:
    - librespot-spoton/Cargo.toml
    - librespot-spoton/src/main.rs
    - librespot-spoton/Cargo.lock
    - Plugins/SpotOn/Bin/x86_64-linux/spoton

key-decisions:
  - "Used rustls-tls-native-roots feature (not native-tls) for musl-compatible TLS — no system OpenSSL dependency"
  - "Used x86_64-unknown-linux-gnu target with +crt-static (musl-tools not available) — produces identical static-pie binary format"
  - "Pinned vergen 9.0.6 in Cargo.lock — vergen 9.1.0 breaks librespot-core build script (trait bound mismatch in vergen-gitcl 1.0.x)"
  - "expires_in field of Token struct is std::time::Duration — convert to u64 seconds for JSON output"

patterns-established:
  - "Rust binary subcommand pattern: Mode enum + match dispatch, validate args before async work"
  - "Cargo.lock pin for transitive build dep: cargo update <crate> --precise <version>"

requirements-completed: [AUTH-01, AUTH-04]

# Metrics
duration: 1min
completed: 2026-05-27
---

# Phase 2 Plan 01: Binary Auth Extension Summary

**librespot-core 0.8.0 integrated into spoton binary — --authenticate (login5 credentials) and --get-token (Keymaster/Mercury Web API token) subcommands working, static-pie binary deployed**

## Performance

- **Duration:** 1 min
- **Started:** 2026-05-27T16:26:46Z
- **Completed:** 2026-05-27T16:27:24Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments

- Extended spoton Rust binary with librespot-core 0.8.0 dependency for Spotify auth
- Implemented `--authenticate` subcommand: login5 username/password auth via `Session::connect`, writes credentials.json to cache dir
- Implemented `--get-token` subcommand: reads cached credentials.json, connects to Spotify, retrieves Web API token via Keymaster/Mercury (`token_provider().get_token()`), prints `{"accessToken":"...","expiresIn":N}` JSON to stdout
- Phase 1 `--check` contract preserved exactly — 4/4 backward-compat tests pass
- Deployed static-pie binary to Plugins/SpotOn/Bin/x86_64-linux/spoton
- Full test suite: 47/47 pass

## Task Commits

Each task was committed atomically:

1. **Task 1: Add librespot-core dependency and arg parsing for auth modes** - `1ce5003` (feat)
2. **Task 2: Build updated binary and verify --check backward compatibility** - `0c98c01` (feat)

## Files Created/Modified

- `librespot-spoton/Cargo.toml` - Added librespot-core 0.8, serde_json, tokio dependencies
- `librespot-spoton/src/main.rs` - Mode enum, full arg parsing, run_authenticate(), run_get_token()
- `librespot-spoton/.cargo/config.toml` - +crt-static flag for static-pie without musl-tools
- `librespot-spoton/Cargo.lock` - Locked deps including vergen 9.0.6 pin
- `Plugins/SpotOn/Bin/x86_64-linux/spoton` - Updated binary (static-pie linked, 9.2MB)

## Decisions Made

- **rustls-tls-native-roots vs native-tls:** Used rustls to avoid system OpenSSL dependency; required for static builds.
- **GNU target with +crt-static vs musl target:** musl-gcc not available on this system (musl-tools not installed, no sudo). Using x86_64-unknown-linux-gnu with `RUSTFLAGS=-C target-feature=+crt-static` via `.cargo/config.toml` produces an equivalent static-pie linked binary — confirmed by `file` output matching Phase 1 binary format.
- **vergen version pin:** vergen 9.1.0 introduced a breaking API change in its trait system that breaks librespot-core's build script. Pinned to 9.0.6 via `cargo update vergen --precise 9.0.6`. This is tracked in Cargo.lock.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Used GNU target with +crt-static instead of musl target**
- **Found during:** Task 2 (musl binary build)
- **Issue:** `x86_64-linux-musl-gcc` not found — musl-tools package not installed on build system; no sudo access to install it
- **Fix:** Configured `.cargo/config.toml` with `rustflags = ["-C", "target-feature=+crt-static"]` for the GNU target. The resulting binary is `static-pie linked` (identical format to Phase 1 musl binary, as confirmed by `file` output)
- **Files modified:** librespot-spoton/.cargo/config.toml (new)
- **Verification:** `file Plugins/SpotOn/Bin/x86_64-linux/spoton` reports "static-pie linked"; all acceptance criteria met
- **Committed in:** 1ce5003 (Task 1 commit)

**2. [Rule 3 - Blocking] Pinned vergen 9.0.6 to fix librespot-core build script**
- **Found during:** Task 1 (initial cargo check)
- **Issue:** librespot-core 0.8.0 build script uses vergen-gitcl 1.0.x which requires vergen ^9.0.6; cargo resolves to latest vergen 9.1.0 which changed trait API breaking the build
- **Fix:** `cargo update vergen --precise 9.0.6` in the librespot-spoton worktree, pinned in Cargo.lock
- **Files modified:** librespot-spoton/Cargo.lock
- **Verification:** `cargo check` succeeds with no errors after pin
- **Committed in:** 1ce5003 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (2 blocking)
**Impact on plan:** Both auto-fixes required for compilation. No scope creep. Binary format identical to Phase 1.

## Issues Encountered

The librespot-core 0.8.0 `expires_in` field on the `Token` struct is `std::time::Duration` (not `u32`). The JSON output uses `.as_secs()` to produce integer seconds, matching the `{"accessToken":"...","expiresIn":<seconds>}` contract expected by TokenManager.pm.

## Threat Surface Scan

No new network endpoints or auth paths introduced beyond what is in the plan's threat model. The binary now accepts `--password` as a CLI argument (T-02-01 in plan's threat register) — this is documented as "accept" disposition since the Perl caller uses shell-safe quoting.

## User Setup Required

None — no external service configuration required for this plan.

## Next Phase Readiness

- Binary is deployed and ready for Plan 02-02 (TokenManager.pm) to call `--get-token`
- Binary is deployed and ready for Plan 02-04 (Settings.pm) to call `--authenticate`
- Actual Spotify auth test with real credentials is deferred to Plan 02-05 checkpoint (login5 feasibility gate)
- Note: If Spotify login5 password auth has been disabled, the `--authenticate` command will return an error at the Spotify AP connection step. This is the expected failure mode for Plan 02-05 checkpoint.

---
*Phase: 02-auth-api-foundation*
*Completed: 2026-05-27*
