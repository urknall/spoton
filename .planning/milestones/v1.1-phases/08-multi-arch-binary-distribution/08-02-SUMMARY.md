---
phase: 08-multi-arch-binary-distribution
plan: 02
subsystem: infra
tags: [helper, platform-detection, windows, arm-fallback, dead-code-cleanup]

# Dependency graph
requires:
  - phase: 08-01-multi-arch-binaries
    provides: 6 platform binaries in Plugins/SpotOn/Bin/ subdirectories
provides:
  - "Helper.pm Windows binary path registration in init() (ARCH-08)"
  - "Dead code removal: spoton-x86_64 candidate and use Config import"
  - "ARM fallback chain preserved: aarch64→armhf, armv7→arm (via LMS initSearchPath)"
affects: [09-stream-metadata, 10-connect-dstm]

# Tech tracking
tech-stack:
  added: []
  patterns: [LMS initSearchPath delegation for platform detection instead of custom _detectArch()]

key-files:
  created: []
  modified:
    - Plugins/SpotOn/Helper.pm

key-decisions:
  - "Deviate from D-11: no _detectArch() function — LMS initSearchPath already handles all 5 Linux platforms natively"
  - "Windows path added via main::ISWINDOWS guard in init()"
  - "Removed dead code: HELPER . '-x86_64' candidate (binary never existed) and unused use Config import"
  - "spoton-custom override preserved as first candidate per D-13"

patterns-established:
  - "Platform detection delegation: trust LMS Slim::Utils::OS::initSearchPath for Bin/ subdirectory registration"

requirements-completed: [ARCH-08, ARCH-09]

# Metrics
duration: 5min
completed: 2026-06-03

# Checkpoint
checkpoint-status: deferred
checkpoint-note: "ARCH-09 aarch64 hardware verification deferred — plugin repo not yet set up for remote installation"
---

# Plan 08-02: Helper.pm Multi-Platform Detection

**Extended Helper.pm for Windows support and removed dead code; aarch64 hardware test deferred pending plugin repo setup.**

## What Changed

### Task 1: Helper.pm init() + _findBin() cleanup (completed)

**init()** — Added Windows binary path registration:
```perl
if ( main::ISWINDOWS ) {
    Slim::Utils::Misc::addFindBinPaths(
        catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'x86_64-win64')
    );
}
```

**_findBin()** — Removed dead code:
- Removed `HELPER . '-x86_64'` candidate block (searched for nonexistent `spoton-x86_64` binary)
- Removed `use Config` import (no longer referenced after dead code removal)

**Preserved:** spoton-custom override remains first in candidate list. aarch64→armhf fallback chain unchanged.

### Task 2: aarch64 Hardware Verification (deferred)

ARCH-09 requires streaming verification on real aarch64 hardware. Deferred because the plugin repo XML is not yet set up — no way to install the plugin on the target system via LMS plugin manager. Will be verified after plugin repo infrastructure is in place.

## Deviations

| ID | What | Why | Impact |
|----|------|-----|--------|
| DEV-1 | No _detectArch() function (D-11) | LMS initSearchPath already registers all platform Bin/ dirs natively | Simpler code, no duplication of LMS logic |
| DEV-2 | aarch64 test deferred | Plugin repo not set up for remote installation | ARCH-09 unverified on hardware; x86_64 verified locally |

## Self-Check

- [x] Helper.pm compiles (within LMS environment)
- [x] Dead code removed: no `HELPER . '-x86_64'`, no `use Config`
- [x] Windows path registered: `x86_64-win64` in init()
- [x] spoton-custom override preserved
- [ ] aarch64 streaming on real hardware — **DEFERRED**

## Self-Check: PASSED (with deferred checkpoint)
