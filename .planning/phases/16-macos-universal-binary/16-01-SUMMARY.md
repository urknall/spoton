---
phase: 16-macos-universal-binary
plan: 01
subsystem: infra
tags: [github-actions, macos, universal-binary, lipo, codesign, librespot, ci-cd]

# Dependency graph
requires:
  - phase: ci-pipeline
    provides: existing build-librespot.yml with Linux/Windows jobs
provides:
  - macOS Universal Binary CI pipeline (build-macos + lipo jobs)
  - Bin/darwin/ placeholder directory for LMS plugin manager binary detection
  - Ad-hoc code-signed macOS binary as spoton-darwin release artifact
affects: [Phase 17 — macOS testing, future releases]

# Tech tracking
tech-stack:
  added: [macos-15 GitHub Actions runner, macos-15-intel GitHub Actions runner, lipo, codesign]
  patterns: [matrix with runs-on from matrix variable for different-runner builds, lipo Universal Binary merge job]

key-files:
  created:
    - Plugins/SpotOn/Bin/darwin/.gitkeep
  modified:
    - .github/workflows/build-librespot.yml

key-decisions:
  - "build-macos uses runs-on from matrix (not hardcoded) to support different runners per arch (D-01/RESEARCH Pitfall 1)"
  - "Native runners (macos-15 + macos-15-intel) preferred over cross-compilation for reliability"
  - "release job needs changed to [build, lipo] to prevent macOS artifact missing from GitHub Release (Pitfall 7)"
  - "Ad-hoc codesign sufficient — LMS plugin manager does not set quarantine xattr (D-06)"

patterns-established:
  - "Pattern: macOS Universal Binary via lipo -create from separate arch artifacts"
  - "Pattern: build-macos matrix with os in matrix include for different runners"

requirements-completed: [PLT-01, PLT-02]

# Metrics
duration: 15min
completed: 2026-06-11
---

# Phase 16 Plan 01: macOS Universal Binary — CI Pipeline Summary

**GitHub Actions CI extended with native macos-15/macos-15-intel build matrix, lipo Universal Binary merge with ad-hoc codesign, and Bin/darwin/ placeholder for LMS plugin manager binary detection**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-06-11T17:29:00Z
- **Completed:** 2026-06-11T17:44:00Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments

- Created `Plugins/SpotOn/Bin/darwin/.gitkeep` placeholder so LMS plugin manager finds the darwin binary directory on macOS hosts
- Extended `build-librespot.yml` with `build-macos` job (parallel to Linux/Windows) using native runners: ARM64 on `macos-15`, Intel on `macos-15-intel` — no cross-compilation
- Added `lipo` job that merges both arch artifacts into a Universal Binary, ad-hoc signs it with `codesign --force --sign -`, verifies with `lipo -info` + `file` + `codesign -dv`, and uploads as `spoton-darwin` artifact
- Updated `release` job `needs` from `[build]` to `[build, lipo]` so macOS binary is always included in GitHub Releases

## Task Commits

Each task was committed atomically:

1. **Task 1: Create Bin/darwin/.gitkeep placeholder** - `2afad00` (chore)
2. **Task 2: Extend build-librespot.yml with macOS build, lipo, and release integration** - `ba497cd` (feat)

**Plan metadata:** (see final commit)

## Files Created/Modified

- `Plugins/SpotOn/Bin/darwin/.gitkeep` - Empty placeholder so darwin/ directory is git-tracked; removed by CI when real binary is placed
- `.github/workflows/build-librespot.yml` - Added build-macos (matrix: aarch64+x86_64), lipo (merge+sign+verify), updated release needs

## Decisions Made

- Used `runs-on: ${{ matrix.os }}` in build-macos (not hardcoded) per RESEARCH Pitfall 1: `macos-latest` returns ARM64, not Intel. Explicit labels `macos-15` and `macos-15-intel` required.
- Native builds on separate runners vs. cross-compilation: native is more reliable, especially with C-linked dependencies like `rustls-native-certs`.
- Trigger unchanged: `v*` tags + `workflow_dispatch` only (D-04). No PR/push builds — avoids 10x macOS CI minute consumption.
- `lipo` job uploads artifact as `spoton-darwin` (not `spoton-aarch64-darwin` or `spoton-x86_64-darwin`) to avoid release job confusion (Pitfall 5).

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None.

## Threat Surface Scan

No new network endpoints or auth paths introduced. The release job already generates SHA256SUMS.txt (T-16-01 mitigated). All actions pinned to @v4/@v2 (T-16-02 mitigated). No new security surface beyond what was planned in the threat model.

## Known Stubs

None — no stubs introduced. The `Bin/darwin/.gitkeep` placeholder is intentional and documented; it is replaced by the actual binary during CI execution.

## User Setup Required

None — CI changes only. macOS Universal Binary will be produced automatically on the next `v*` tag push or `workflow_dispatch` trigger.

## Next Phase Readiness

- CI pipeline ready for macOS Universal Binary production on next tag push
- Phase 16 Plan 02 (Helper.pm ISMAC block + Gatekeeper UI) can proceed independently
- Note: `macos-15-intel` runner is the last Intel image (available until Aug 2027 per GitHub changelog)

---
*Phase: 16-macos-universal-binary*
*Completed: 2026-06-11*
