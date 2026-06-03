---
phase: 08-multi-arch-binary-distribution
plan: 01
subsystem: infra
tags: [librespot, cross-rs, musl, docker, binary-distribution, arm, aarch64, windows]

# Dependency graph
requires:
  - phase: 01-plugin-skeleton-binary-foundation
    provides: librespot-spoton Rust codebase and Cargo.toml with rustls-tls configuration
provides:
  - "x86_64-linux/spoton: musl-static binary replacing prior glibc build (ARCH-01)"
  - "aarch64-linux/spoton: musl-static ARM64 binary for Pi 4/5 and NAS (ARCH-02)"
  - "armhf-linux/spoton: musl-static ARMv7 binary for Pi 2/3 (ARCH-03)"
  - "arm-linux/spoton: musl-static ARMv6 binary for Pi 1/Zero (ARCH-10)"
  - "i386-linux/spoton: musl-static i386 binary for 32-bit Intel LMS hosts (ARCH-04)"
  - "x86_64-win64/spoton.exe: Windows PE32+ binary via MinGW-w64 (ARCH-07)"
  - "librespot-spoton/Cross.toml: cross-rs target configuration for 6 platforms"
affects: [08-02-helper-platform-detection, 10-connect-dstm]

# Tech tracking
tech-stack:
  added: [cross-rs 0.2.5, Docker cross-compilation, musl-tools via cross-rs images]
  patterns: [cross-rs Docker-based cross-compilation with target-specific images, build artifact cleanup between targets]

key-files:
  created:
    - librespot-spoton/Cross.toml
    - Plugins/SpotOn/Bin/aarch64-linux/spoton
    - Plugins/SpotOn/Bin/armhf-linux/spoton
    - Plugins/SpotOn/Bin/arm-linux/spoton
    - Plugins/SpotOn/Bin/i386-linux/spoton
    - Plugins/SpotOn/Bin/x86_64-win64/spoton.exe
  modified:
    - Plugins/SpotOn/Bin/x86_64-linux/spoton (replaced glibc with musl-static)

key-decisions:
  - "Clean target/release/build and target/release/deps between each cross-rs Docker target to prevent GLIBC version conflicts in build script executables"
  - "Use default cross-rs images for musl targets (no image overrides in Cross.toml) — rustls-tls eliminates OpenSSL system dependency"
  - "Windows target: x86_64-pc-windows-gnu (MinGW-w64) not MSVC per P-46"

patterns-established:
  - "Cross-target build artifact cleanup: remove target/release/build/ and target/release/deps/ between targets to prevent glibc ABI conflicts in build scripts compiled by different Docker images"
  - "Verification sequence: file (ELF type + arch) + ldd (static linkage) + --check (functional test for x86_64)"

requirements-completed: [ARCH-01, ARCH-02, ARCH-03, ARCH-04, ARCH-07, ARCH-10]

# Metrics
duration: 28min
completed: 2026-06-03
---

# Phase 08 Plan 01: Multi-Arch Binary Distribution Summary

**Six musl-static Linux binaries + Windows PE32+ binary cross-compiled via cross-rs Docker, replacing the glibc-linked x86_64 binary and enabling SpotOn on all supported LMS platforms**

## Performance

- **Duration:** 28 min
- **Started:** 2026-06-03T16:52:53Z
- **Completed:** 2026-06-03T17:21:37Z
- **Tasks:** 2
- **Files modified:** 11 (6 new binaries + Cross.toml + 4 .gitkeep deletions)

## Accomplishments

- Cross-compiled librespot-spoton for 5 Linux targets (x86_64, aarch64, armv7, ARMv6, i686) as musl-static binaries — zero shared library dependencies
- Cross-compiled for Windows x86_64 via MinGW-w64 GNU toolchain, producing a valid PE32+ executable
- Replaced the prior glibc-linked x86_64 binary with a musl-static build — plugin now works on all glibc versions including very old LMS installs
- Total binary footprint: 117MB in Plugins/SpotOn/Bin/ (6 binaries × 16-36MB, within D-07 budget estimate)
- All Linux binaries verified statically linked; x86_64 binary passes functional `--check` test

## Task Commits

Each task was committed atomically:

1. **Task 1 + 2: Build all 6 binaries and commit** - `91e755c` (feat)

## Files Created/Modified

- `librespot-spoton/Cross.toml` - cross-rs target configuration for 6 platforms (no image overrides needed)
- `Plugins/SpotOn/Bin/x86_64-linux/spoton` - musl-static x86_64 binary (replaced glibc build), 17MB
- `Plugins/SpotOn/Bin/aarch64-linux/spoton` - musl-static ARM64 binary, 17MB
- `Plugins/SpotOn/Bin/armhf-linux/spoton` - musl-static ARMv7 binary, 16MB
- `Plugins/SpotOn/Bin/arm-linux/spoton` - musl-static ARMv6 binary, 16MB
- `Plugins/SpotOn/Bin/i386-linux/spoton` - musl-static i386 binary, 16MB
- `Plugins/SpotOn/Bin/x86_64-win64/spoton.exe` - Windows PE32+ binary, 36MB
- `.gitkeep` files removed from 4 Bin/ directories now containing binaries

## Decisions Made

- **Build artifact cleanup required between cross targets:** The first build (x86_64-musl) compiled build script executables inside a Docker image with a newer glibc. The second build (aarch64) attempted to reuse these via `target/release/build/`, but the glibc requirement (`GLIBC_2.28`, `2.29`, `2.30`) was incompatible with the aarch64 container's host toolchain. Fix: delete `target/release/build/` and `target/release/deps/` before each subsequent cross build. This forces the new container to recompile build scripts natively.

- **Windows target uses GNU not MSVC:** Per P-46, `x86_64-pc-windows-gnu` (MinGW-w64) was used, not `x86_64-pc-windows-msvc`, as MSVC requires a Windows host for the linker.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Build script GLIBC conflict between cross-rs Docker images**
- **Found during:** Task 1 (aarch64 build after x86_64 build)
- **Issue:** The aarch64 cross-rs container's host toolchain had older glibc. Build scripts compiled in the x86_64-musl container (which has newer glibc) were reused via `target/release/build/` but couldn't execute: `GLIBC_2.28`, `GLIBC_2.29`, `GLIBC_2.30` not found.
- **Fix:** Added `rm -rf target/release/build/ target/release/deps/` before each cross build (targets 2-5). This forces each container to recompile build scripts natively inside its own glibc environment.
- **Files modified:** None (build step adjustment, no source changes)
- **Verification:** All 4 subsequent builds succeeded after cleanup. Build scripts were recompiled inside each target's container.
- **Committed in:** 91e755c (included in task commit — no source changes, operational fix only)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Build environment issue specific to sequential cross-compilation using different Docker images. Fix is the established pattern for cross-rs multi-target builds. No scope creep.

## Issues Encountered

- Docker image pull failure on first `x86_64-unknown-linux-musl` build (network reset during layer download). Retry succeeded immediately — transient network issue, no lasting impact.

## User Setup Required

None — no external service configuration required. All binaries are committed to the Git repository.

## Next Phase Readiness

- All 6 binaries are in place in `Plugins/SpotOn/Bin/` subdirectories
- Plan 08-02 (Helper.pm platform detection) can now proceed — it needs these binaries to be present for the `addFindBinPaths()` detection logic
- The x86_64-linux binary passes `--check` and is functional for immediate use
- ARM/i386 binaries cannot be locally executed but are verified as statically linked ELF/PE binaries of the correct architecture
- macOS targets (ARCH-05, ARCH-06) remain deferred to v1.2 (P-44: no cross-rs for macOS, no native Mac hardware)

---
*Phase: 08-multi-arch-binary-distribution*
*Completed: 2026-06-03*
