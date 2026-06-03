---
phase: 08-multi-arch-binary-distribution
verified: 2026-06-03T21:00:00Z
status: human_needed
score: 8/10 must-haves verified
overrides_applied: 0
gaps:
  - truth: "Windows binary check succeeds — helperCheck() validates spoton.exe on Windows"
    status: failed
    reason: "helperCheck() uses POSIX single-quote shell quoting unconditionally. On Windows, cmd.exe treats single-quotes as literal characters, not shell delimiters. The binary path is passed as \"'C:\\...\\spoton.exe' -n 'SpotOn' --check\" which cmd.exe cannot execute — it looks for a file named \"'spoton.exe'\". CR-01 (code review) documented this. The fix was not applied."
    artifacts:
      - path: "Plugins/SpotOn/Helper.pm"
        issue: "helperCheck() lines 72-74: single-quote quoting is invalid on Windows cmd.exe. main::ISWINDOWS branch with double-quote quoting is absent."
    missing:
      - "Add main::ISWINDOWS branch in helperCheck() that uses double-quote cmd.exe quoting convention instead of POSIX single-quotes"
  - truth: "REQUIREMENTS.md ARCH-08 checkbox reflects delivered state"
    status: failed
    reason: "ARCH-08 checkbox in REQUIREMENTS.md is unchecked ([ ]) and traceability row shows 'Pending', but the implementation (Windows addFindBinPaths in init(), LMS initSearchPath delegation for Linux platforms) is present in Helper.pm. The requirements document was not updated to reflect delivered work. ARCH-08 text also references '8 platforms' but phase scope is 6 (macOS deferred to v1.2)."
    artifacts:
      - path: ".planning/REQUIREMENTS.md"
        issue: "ARCH-08 row: checkbox [ ] and status 'Pending' — should be [x] / Complete given that Windows path and Linux delegation are implemented"
    missing:
      - "Update ARCH-08 checkbox to [x] in REQUIREMENTS.md"
      - "Update ARCH-08 traceability row to 'Complete'"
      - "Optionally clarify ARCH-08 description: '6 in-scope platforms' instead of '8 platforms' (ARCH-05/06 macOS excluded from Phase 8)"
human_verification:
  - test: "Verify binary detection on x86_64 Linux with LMS loaded"
    expected: "LMS Settings -> Advanced -> SpotOn shows binary path pointing to Bin/x86_64-linux/spoton and version 1.0.0"
    why_human: "LMS plugin loading, findbin() path resolution, and settings UI cannot be grep-verified. Need running LMS instance."
  - test: "Verify aarch64 hardware streaming (ARCH-09 — deferred by user)"
    expected: "On aarch64 Linux (Pi 4 or NAS): binary path shows Bin/aarch64-linux/spoton, Connect device appears in Spotify app, audio plays without errors"
    why_human: "Requires physical aarch64 hardware and running Spotify session. Deferred pending plugin repo setup for remote installation."
---

# Phase 08: Multi-Arch Binary Distribution Verification Report

**Phase Goal:** A librespot binary is available for every supported platform target; the plugin selects the correct binary at runtime without user configuration
**Verified:** 2026-06-03T21:00:00Z
**Status:** human_needed (with 2 gaps)
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Six binaries exist in their respective Bin/ subdirectories | VERIFIED | `ls` confirms: x86_64-linux/spoton, aarch64-linux/spoton, armhf-linux/spoton, arm-linux/spoton, i386-linux/spoton, x86_64-win64/spoton.exe — all present, 16-36MB each |
| 2 | All five Linux binaries are musl-statically linked | VERIFIED | `file` reports "statically linked" for all 5; `ldd` reports "not a dynamic executable" / "Das Programm ist nicht dynamisch gelinkt" for all 5 |
| 3 | The x86_64-linux binary replaces the previous glibc-linked binary | VERIFIED | `file` output: "ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), static-pie linked" — no glibc |
| 4 | The x86_64 binary passes --check with correct output | VERIFIED | `Plugins/SpotOn/Bin/x86_64-linux/spoton -n Test --check` outputs "ok spoton v1.0.0" followed by JSON capabilities |
| 5 | The Windows binary is a valid PE executable | VERIFIED | `file` reports: "PE32+ executable (console) x86-64 (stripped to external PDB), for MS Windows" |
| 6 | No .gitkeep files remain in populated directories | VERIFIED | `find Plugins/SpotOn/Bin -name .gitkeep` returns 0 results |
| 7 | Helper.pm init() registers Windows binary path | VERIFIED | Lines 28-32: `if (main::ISWINDOWS)` block adds `Bin/x86_64-win64` via `addFindBinPaths` |
| 8 | Dead code removed from _findBin() | VERIFIED | `grep "HELPER.*x86_64\|use Config"` returns no matches — both the spoton-x86_64 candidate block and `use Config` import are absent |
| 9 | Windows binary check succeeds via helperCheck() | FAILED | `helperCheck()` lines 72-74 use POSIX single-quote quoting unconditionally. No `main::ISWINDOWS` branch. Windows cmd.exe treats single-quotes as literal characters — the binary will never be validated. Documented as CR-01 in 08-REVIEW.md; fix not applied. |
| 10 | REQUIREMENTS.md accurately reflects delivered state | FAILED | ARCH-08 checkbox is unchecked and traceability shows "Pending" despite the Windows path registration and Linux initSearchPath delegation being present in Helper.pm. Document not updated after delivery. |

**Score:** 8/10 truths verified

---

### Deferred Items

Items not yet met but explicitly addressed in later milestone phases.

| # | Item | Addressed In | Evidence |
|---|------|-------------|----------|
| 1 | macOS x86_64 binary (ARCH-05) | v1.2+ | REQUIREMENTS.md v1.2+ section; D-03 in CONTEXT.md: "macOS targets deferred to v1.2. No Mac available; P-44 prohibits cross-rs for macOS." |
| 2 | macOS aarch64 binary (ARCH-06) | v1.2+ | Same as ARCH-05 — grouped decision D-03 |

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `librespot-spoton/Cross.toml` | cross-rs target configuration | VERIFIED | Present, 39 lines, 6 target stanzas with correct Rust triples, directory mapping comments, references P-42/P-44/P-46 |
| `Plugins/SpotOn/Bin/x86_64-linux/spoton` | x86_64 musl-static binary (ARCH-01) | VERIFIED | 17MB, ELF 64-bit, static-pie linked, passes --check |
| `Plugins/SpotOn/Bin/aarch64-linux/spoton` | aarch64 musl-static binary (ARCH-02) | VERIFIED | 17MB, ELF 64-bit ARM aarch64, statically linked |
| `Plugins/SpotOn/Bin/armhf-linux/spoton` | armv7 musl-static binary (ARCH-03) | VERIFIED | 16MB, ELF 32-bit ARM EABI5, statically linked |
| `Plugins/SpotOn/Bin/arm-linux/spoton` | ARMv6 musl-static binary (ARCH-10) | VERIFIED | 16MB, ELF 32-bit ARM EABI5, statically linked |
| `Plugins/SpotOn/Bin/i386-linux/spoton` | i386 musl-static binary (ARCH-04) | VERIFIED | 16MB, ELF 32-bit Intel 80386, statically linked |
| `Plugins/SpotOn/Bin/x86_64-win64/spoton.exe` | Windows x86_64 PE binary (ARCH-07) | VERIFIED | 36MB, PE32+ executable (console) x86-64, for MS Windows |
| `Plugins/SpotOn/Helper.pm` | Multi-platform detection with Windows support | PARTIAL | Windows addFindBinPaths present; dead code removed; but helperCheck() lacks Windows quoting fix (CR-01) |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `Helper.pm::init()` | `Slim::Utils::Misc::addFindBinPaths` | Windows clause lines 28-32 | VERIFIED | `grep "addFindBinPaths"` finds 2 matches: aarch64 fallback (line 23) + Windows path (line 29) |
| `Helper.pm::_findBin()` | `Slim::Utils::Misc::findbin` | `findbin($name)` line 131 | VERIFIED | findbin called for each candidate name; LMS initSearchPath resolves platform directory |
| `Cross.toml` | `cross build --release --target` | cross-rs reads Cross.toml for target stanzas | VERIFIED | 6 `[target.*]` stanzas match the 6 build commands documented in file header |
| `helperCheck()` | Windows binary validation | `main::ISWINDOWS` branch with cmd.exe quoting | FAILED | No ISWINDOWS branch; single-quote quoting fails under cmd.exe (CR-01) |

---

### Data-Flow Trace (Level 4)

Not applicable — this phase produces binaries and platform detection code, not UI components with data flows.

---

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| x86_64 binary passes --check | `Plugins/SpotOn/Bin/x86_64-linux/spoton -n Test --check` | "ok spoton v1.0.0\n{...JSON...}" | PASS |
| All Linux binaries statically linked | `ldd` on each of 5 binaries | All: "not a dynamic executable" / "Das Programm ist nicht dynamisch gelinkt" | PASS |
| Windows binary is valid PE | `file Plugins/SpotOn/Bin/x86_64-win64/spoton.exe` | "PE32+ executable (console) x86-64...for MS Windows" | PASS |
| Dead code removed | `grep "HELPER.*x86_64\|use Config" Helper.pm` | 0 matches | PASS |
| Windows path registered | `grep "x86_64-win64\|ISWINDOWS" Helper.pm` | 3 matches at correct lines | PASS |
| No .gitkeep in populated dirs | `find Plugins/SpotOn/Bin -name .gitkeep` | 0 results | PASS |

---

### Probe Execution

No `scripts/*/tests/probe-*.sh` probes declared or found for this phase. Phase builds binaries via cross-rs (not runnable probes). Step 7c: SKIPPED (binary compilation phase, no probe scripts).

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| ARCH-01 | 08-01-PLAN | x86_64 musl-static binary | SATISFIED | Binary confirmed musl-static via ldd + file; replaces glibc build |
| ARCH-02 | 08-01-PLAN | aarch64 musl-static binary | SATISFIED | Binary present, statically linked, 17MB |
| ARCH-03 | 08-01-PLAN | armv7 musl-static binary | SATISFIED | Binary present, statically linked, 16MB |
| ARCH-04 | 08-01-PLAN | i386 musl-static binary | SATISFIED | Binary present, statically linked, 16MB |
| ARCH-05 | — | macOS x86_64 binary | DEFERRED | Explicitly deferred to v1.2 (D-03, P-44) — out of Phase 8 scope |
| ARCH-06 | — | macOS aarch64 binary | DEFERRED | Explicitly deferred to v1.2 (D-03, P-44) — out of Phase 8 scope |
| ARCH-07 | 08-01-PLAN | Windows x86_64 PE binary | SATISFIED | PE32+ binary present, 36MB |
| ARCH-08 | 08-02-PLAN | Helper.pm detects all in-scope platforms | PARTIAL | Windows addFindBinPaths: implemented. Linux platforms via LMS initSearchPath: implemented. But helperCheck() cmd.exe quoting is broken (CR-01) — Windows binary will never validate. REQUIREMENTS.md checkbox not updated. |
| ARCH-09 | 08-02-PLAN | aarch64 streaming on real hardware | NEEDS HUMAN | Deferred by user — plugin repo not set up. Physical aarch64 test pending. |
| ARCH-10 | 08-01-PLAN | ARMv6 musl-static binary | SATISFIED | Binary present, statically linked, 16MB |

**Note on ARCH-08 scope:** REQUIREMENTS.md text says "alle 8 Plattformen" (all 8 platforms) but phase scope is 6 (ARCH-05 and ARCH-06 macOS are deferred). The implementation covers the 6 in-scope platforms correctly. The "8" in the requirement text is inaccurate given the macOS deferral.

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `Plugins/SpotOn/Helper.pm` | 72-74 | Single-quote POSIX shell quoting without Windows branch | BLOCKER | Windows binary validation always fails via cmd.exe; plugin reports no binary on Windows despite correct binary being present and path registered |
| `Plugins/SpotOn/Helper.pm` | 122, 55 | Dead parameter `$customFirst` accepted but never evaluated | WARNING | `unshift` is unconditional regardless of parameter value; misleading code; from CR-01 WR-01 |
| `.planning/REQUIREMENTS.md` | 25, 78 | ARCH-08 checkbox unchecked + traceability "Pending" | WARNING | Requirements document does not reflect delivered code state; creates tracking confusion |

---

### Human Verification Required

#### 1. LMS Runtime Binary Detection

**Test:** Restart LMS with the updated plugin. Open LMS Settings -> Advanced -> SpotOn.
**Expected:** Binary path field shows `.../Plugins/SpotOn/Bin/x86_64-linux/spoton` with version `1.0.0`. Play a Spotify track to confirm end-to-end streaming on the development x86_64 machine.
**Why human:** LMS plugin loading, `findbin()` path resolution, and settings UI rendering cannot be verified without a running LMS instance.

#### 2. aarch64 Hardware Streaming (ARCH-09 — deferred by user)

**Test:** On an aarch64 Linux system (Pi 4 or NAS): copy the updated `Plugins/SpotOn/` directory to LMS plugins folder, restart LMS. Check Settings -> Advanced -> SpotOn for the binary path. Enable Spotify Connect for a player. Open Spotify app, confirm the LMS player appears as a Connect device. Play a track.
**Expected:** Binary path shows `Bin/aarch64-linux/spoton`, version `1.0.0`. Audio plays without errors. `ldd /path/to/aarch64-linux/spoton` on the target reports "not a dynamic executable".
**Why human:** Requires physical aarch64 hardware, a configured Spotify Premium account, and a plugin repository installation mechanism not yet in place. Explicitly deferred by user pending plugin repo setup (08-02-SUMMARY.md).

---

### Gaps Summary

**Two gaps blocking full goal achievement, one as BLOCKER:**

**Gap 1 (BLOCKER): Windows binary validation broken (CR-01 not fixed)**

`helperCheck()` applies POSIX single-quote shell quoting to the binary path before dispatching via backtick. On Windows, Perl's backtick uses `cmd.exe` which treats single-quotes as ordinary characters. The command `'C:\...\spoton.exe' -n 'SpotOn' --check` fails because `cmd.exe` cannot find a file named literally `'spoton.exe'`. This means `$helper` is never populated on Windows — the plugin reports "Didn't find SpotOn helper application!" despite Phase 08 correctly placing the binary and registering the path. The Phase 08 code review (08-REVIEW.md) identified this as CR-01 and provided an exact fix, but the fix was not applied.

**Fix required:** Add a `main::ISWINDOWS` branch in `helperCheck()` that wraps the path in double-quotes and escapes embedded double-quotes (`s/"/\"\"/g`), or replace the backtick with list-form `IPC::Open3` to bypass the shell entirely.

**Gap 2 (WARNING): REQUIREMENTS.md tracking not updated**

ARCH-08 checkbox remains unchecked and traceability row shows "Pending" in REQUIREMENTS.md despite the Windows path registration and Linux platform delegation being implemented in Helper.pm. This is a documentation gap, not a code gap. The implementation is present; the tracking document is stale.

**Fix required:** Set ARCH-08 checkbox to `[x]` and traceability status to "Complete".

---

_Verified: 2026-06-03T21:00:00Z_
_Verifier: Claude (gsd-verifier)_
