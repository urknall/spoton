---
phase: 01-plugin-skeleton-binary-foundation
plan: "04"
subsystem: infra
tags: [lms, rust, binary, librespot, musl, github-actions, cross-compilation, ci]

# Dependency graph
requires:
  - 01-03 (Bin/x86_64-linux/ directory, t/06_binary_check.t test contract)
  - 01-02 (Helper.pm --check contract: "ok spoton v{VERSION}" + JSON, MIN_BINARY_VERSION 1.0.0)
provides:
  - Executable x86_64-unknown-linux-musl static binary v1.0.0 in Plugins/SpotOn/Bin/x86_64-linux/spoton
  - Minimal Rust project (librespot-spoton/) implementing --check contract for LMS Helper.pm
  - GitHub Actions CI workflow for ARM cross-compilation (aarch64, armhf, arm, i386) via cross-rs
  - .gitignore for Rust build artifacts
affects:
  - t/06_binary_check.t (now runs fully — 4/4 tests pass, no longer skip_all)
  - Helper.pm helperCheck() (binary satisfies MIN_BINARY_VERSION 1.0.0 constraint)
  - Phase 2 (Connect mode + streaming): librespot-spoton/ is the Rust source foundation

# Tech tracking
tech-stack:
  added:
    - Rust (cargo 1.95.0 + rustup 1.29.0) — binary source language
    - x86_64-unknown-linux-musl target — static PIE binary for LMS deployment
    - cross-rs/cross — ARM cross-compilation via Docker (GitHub Actions only)
    - GitHub Actions — CI workflow for ARM binary builds + GitHub Release
    - dtolnay/rust-toolchain@stable — pinned Rust toolchain in CI
    - actions/checkout@v4, actions/upload-artifact@v4, actions/download-artifact@v4 — pinned CI actions
    - softprops/action-gh-release@v2 — GitHub Release creation
  patterns:
    - Minimal Rust binary with env!("CARGO_PKG_VERSION") for version embedding
    - --check output contract: "ok spoton v{VERSION}\n{JSON}" matching Helper.pm regex
    - Static musl PIE binary: no system dependencies, portable across Linux versions
    - GitHub Actions matrix strategy: 4 ARM targets, 1 job per target
    - cross build for ARM targets (requires Docker — GitHub Actions runner provides Docker)
    - Binary artifacts uploaded per target, aggregated in release job

key-files:
  created:
    - librespot-spoton/Cargo.toml
    - librespot-spoton/Cargo.lock
    - librespot-spoton/src/main.rs
    - Plugins/SpotOn/Bin/x86_64-linux/spoton
    - .github/workflows/build-librespot.yml
    - .gitignore
  modified:
    - Plugins/SpotOn/Bin/x86_64-linux/.gitkeep (deleted — replaced by binary)

key-decisions:
  - "Binary version set to 1.0.0 (not 0.1.0) to satisfy Helper.pm MIN_BINARY_VERSION 1.0.0 constraint"
  - "x86_64-unknown-linux-musl build succeeded without musl-tools — Rust bundles musl libc for pure-Rust programs without C FFI"
  - "librespot-spoton/ is Phase 1 minimal binary; full librespot integration deferred to Phase 2+"
  - "Cargo.lock committed to repo for reproducible builds (binary package, not library)"
  - "GitHub Actions workflow uses pinned action versions (@v4) per threat model T-01-10"
  - "No secrets exposed in workflow — cross build requires no Spotify credentials"

requirements-completed:
  - LMS-06
  - LMS-07

# Metrics
duration: 15min
completed: 2026-05-27
---

# Phase 01 Plan 04: x86_64 Binary Build + GitHub Actions CI Workflow Summary

**Static x86_64-musl spoton binary v1.0.0 implementing the --check contract; GitHub Actions CI matrix for ARM cross-compilation via cross-rs**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-05-27T09:55:00Z
- **Completed:** 2026-05-27T10:10:00Z
- **Tasks:** 1 complete (Task 1 auto), 1 pending (Task 2 checkpoint)
- **Files modified:** 6 created, 1 deleted

## Accomplishments

- Created minimal Rust project `librespot-spoton/` implementing the --check contract required by Helper.pm
- Built static PIE binary (x86_64-unknown-linux-musl, v1.0.0) — no system dependencies
- Binary satisfies `ok spoton v[\d\.]+` regex and MIN_BINARY_VERSION 1.0.0 enforced by helperCheck()
- Binary `--check` output: `ok spoton v1.0.0\n{"version":"1.0.0","lms-auth":false,"ogg-direct":false,"passthrough":true}`
- t/06_binary_check.t now runs fully (4/4 tests pass) — no longer skip_all
- Full test suite: 47/47 tests pass across 6 test files
- Created `.github/workflows/build-librespot.yml` with matrix for 4 ARM targets
- Added `.gitignore` for Rust build artifacts

## Task Commits

1. **Task 1: x86_64 Binary + GitHub Actions Workflow** - `289d231` (feat)

**Task 2 (checkpoint:human-verify): awaiting manual verification**

## Files Created/Modified

- `librespot-spoton/Cargo.toml` — Rust package manifest: name=spoton, version=1.0.0, edition=2021
- `librespot-spoton/Cargo.lock` — Pinned dependencies (no external crates — standard library only)
- `librespot-spoton/src/main.rs` — Binary implementation: --check contract, -n flag, usage info
- `Plugins/SpotOn/Bin/x86_64-linux/spoton` — Compiled static x86_64-musl binary, 755 permissions
- `.github/workflows/build-librespot.yml` — CI workflow: 4 ARM targets + release job
- `.gitignore` — Excludes librespot-spoton/target/ from version control
- `Plugins/SpotOn/Bin/x86_64-linux/.gitkeep` — Deleted (replaced by binary)

## Decisions Made

- **Version 1.0.0:** The plan described a "Phase 1 minimal binary" with version 0.1.0, but Helper.pm's `MIN_BINARY_VERSION` constant is `'1.0.0'` and `helperCheck()` rejects binaries below that floor. Setting version to 1.0.0 is the correct fix.

- **musl build without musl-tools:** `x86_64-unknown-linux-musl` compiled successfully without `musl-tools` installed because the program uses only the Rust standard library and has no C FFI dependencies. Rust bundles its own musl libc for pure-Rust targets.

- **librespot-spoton/ as Phase 1 stub:** The full librespot port (spotty.rs -> spoton.rs) is 1-2 days of Rust work (per RESEARCH.md). The minimal binary satisfies all Phase 1 requirements. Phase 2 will extend this source for real Connect mode integration.

- **Cargo.lock committed:** Binary packages (not library crates) should commit Cargo.lock for reproducible builds. CI and local builds will produce identical output.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Binary version set to 1.0.0 instead of 0.1.0**
- **Found during:** Task 1 verification (binary --check vs. helperCheck MIN_BINARY_VERSION check)
- **Issue:** Plan described v0.1.0 in example output, but Helper.pm `MIN_BINARY_VERSION = '1.0.0'` and `helperCheck()` rejects binaries with version < 1.0.0
- **Fix:** Set Cargo.toml version to 1.0.0; rebuild binary
- **Files modified:** librespot-spoton/Cargo.toml, Plugins/SpotOn/Bin/x86_64-linux/spoton
- **Commit:** 289d231

## Known Stubs

The librespot-spoton binary is a Phase 1 minimal implementation. The following capabilities are stubs:

| Stub | File | Reason |
|------|------|--------|
| Connect mode (--backend, --device, streaming) | `librespot-spoton/src/main.rs` | Phase 2 will add full librespot Connect integration |
| --single-track streaming | `librespot-spoton/src/main.rs` | Phase 4 (streaming) will implement single-track decode via librespot |
| --lms --player-mac notifications | `librespot-spoton/src/main.rs` | Phase 2 will add JSON-RPC notifications to LMS |
| --lms-auth --get-token | `librespot-spoton/src/main.rs` | Phase 2 will add Keymaster/login5 auth integration |

These stubs do not prevent the plan's goal (--check contract for LMS binary discovery). They are planned Phase 2+ deliverables.

## Threat Flags

| Flag | File | Description |
|------|------|-------------|
| Binary execution surface | `Plugins/SpotOn/Bin/x86_64-linux/spoton` | New ELF binary executed by LMS via backtick in helperCheck(); T-01-09 mitigated (permissions 755, built from verified Rust source, no dynamic deps) |

## Self-Check: PASSED

- `librespot-spoton/Cargo.toml` exists: FOUND
- `librespot-spoton/src/main.rs` exists: FOUND
- `Plugins/SpotOn/Bin/x86_64-linux/spoton` exists and is executable (-x): FOUND
- `.github/workflows/build-librespot.yml` exists: FOUND
- `.gitignore` exists: FOUND
- Binary permissions are 755: CONFIRMED
- `spoton --check` first line matches `ok spoton v[\d\.]+`: CONFIRMED
- `spoton --check` second line is parseable JSON with "version" key: CONFIRMED
- `prove -v t/` passes 47/47 tests: CONFIRMED
- Commit `289d231` exists: FOUND

## Checkpoint: Task 2 — Binary --check und vollstaendige Test-Suite verifizieren

**Status:** Automated parts complete. Manual verification pending.

**Automated verification completed:**
- `spoton --check` output: `ok spoton v1.0.0` + `{"version":"1.0.0","lms-auth":false,"ogg-direct":false,"passthrough":true}`
- `spoton -n SpotOn --check`: same output (name flag accepted and handled)
- `prove -v t/06_binary_check.t`: 4/4 tests passed
- `prove -v t/` (all 6 files): 47/47 tests passed, 0 failures

**GitHub Actions workflow targets verified:**
- aarch64-unknown-linux-musl -> Bin/aarch64-linux/spoton
- armv7-unknown-linux-musleabihf -> Bin/armhf-linux/spoton
- arm-unknown-linux-musleabi -> Bin/arm-linux/spoton
- i686-unknown-linux-musl -> Bin/i386-linux/spoton
- cross build command present (1 occurrence)
- Pinned actions: actions/checkout@v4, dtolnay/rust-toolchain@stable, actions/upload-artifact@v4

**Manual verification required (after worktrees merged + LMS symlink):**
```bash
sudo ln -s /home/sti/spoton/Plugins/SpotOn /usr/share/squeezeboxserver/Plugins/SpotOn
sudo systemctl restart lyrionmusicserver
tail -100 /var/log/squeezeboxserver/server.log | grep -i spoton
```
Then verify:
1. No "couldn't load plugin" errors in server.log
2. SpotOn appears in LMS Settings -> Plugins list
3. Settings page shows Binary Status: v1.0.0 (green, not red missing)
4. OPML menu shows SpotOn (no "Binary nicht gefunden" error)
5. GitHub Actions workflow runs correctly on workflow_dispatch

## Next Phase Readiness

- Phase 1 auto-verification complete: 47/47 tests pass
- ARM binaries: will be provided by GitHub Actions CI run (trigger: workflow_dispatch or v1.0.0 tag push)
- Phase 2 (Spotify Connect): `librespot-spoton/` is the Rust source foundation; main.rs will be extended with full librespot integration

---
*Phase: 01-plugin-skeleton-binary-foundation*
*Completed: 2026-05-27*
