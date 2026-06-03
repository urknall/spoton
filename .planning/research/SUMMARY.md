# Project Research Summary

**Project:** SpotOn v1.1 — Hardening & Reach
**Domain:** LMS plugin hardening (Connect-DSTM, Multi-Arch, Code Cleanup)
**Researched:** 2026-06-03
**Confidence:** HIGH (cleanup + multi-arch) / MEDIUM (Connect-DSTM, spike needed)

## Executive Summary

v1.1 comprises three independent work areas with ascending risk profiles. Code cleanup (DE→EN) is zero-risk mechanical work (~25 comment/log occurrences in 4 files). Multi-Arch binary distribution is medium-risk toolchain work requiring cross-rs for Linux musl targets, native macOS CI runners, and MinGW-w64 for Windows. Connect-DSTM is the highest-risk feature requiring a spike to validate the `PlayerEvent::EndOfTrack` event path in librespot before committing to implementation.

The critical architectural finding: LMS's DSTM framework never fires in Connect mode (`isRepeatingStream=1`). Connect-DSTM requires a new `endoftrack` binary event, a grace timer in `Connect.pm`, and queue injection via `POST /me/player/queue`. The spike must validate this before any implementation.

## Key Findings

### Stack Additions

- **cross-rs 0.2.5** (already installed): Linux musl cross-compilation for 4 targets
- **MinGW-w64**: Windows GNU cross-compilation from Linux CI (NOT MSVC/cargo-xwin)
- **Native macOS CI runners**: Apple SDK licensing blocks cross-rs for Darwin
- **`SessionConfig.autoplay`**: Correct lever for librespot autoplay (NOT `ConnectConfig`)
- **`POST /me/player/queue`**: Available in dev mode, own-token via existing me/* guard

### Feature Table Stakes

| Feature | Priority | Complexity | Risk |
|---------|----------|------------|------|
| Linux ARM binaries (aarch64/armv7) | P1 — must | Low | Low |
| Linux i386 binary | P1 — must | Low | Low |
| Helper.pm arch detection | P1 — must | Low | Low |
| DE→EN comments + logs | P1 — must | Low | Zero |
| macOS binaries (x86_64/aarch64) | P2 — should | Medium | Medium (CI) |
| Windows binary | P2 — should | Medium | Low |
| Connect-DSTM (endoftrack + grace timer) | P2 — should | High | High (spike) |
| Per-player autoplay toggle | P2 — should | Low | Low |

### Architecture Changes

| Component | Feature | Change |
|-----------|---------|--------|
| `connect.rs` | DSTM | New `EndOfTrack` match arm → emit `spottyconnect endoftrack` |
| `Connect.pm` | DSTM | New `endoftrack` handler + 3-5s grace timer |
| `API/Client.pm` | DSTM | New `addToQueue($accountId, $uri, $cb)` |
| `Helper.pm` | Multi-arch | Extended `addFindBinPaths` for macOS/Windows |
| `Bin/` | Multi-arch | 5 new subdirectories with static binaries |
| CI workflow | Multi-arch | Platform-specific runners in matrix |
| 4 Perl files | Cleanup | ~25 German comment/log lines translated |

### Critical Pitfalls

- **P-40:** DSTM framework never fires in Connect mode — need separate binary event path
- **P-41:** `recommendations` endpoint removed — use `_searchFallback` only
- **P-42:** `+crt-static` on gnu targets breaks proc-macro (Rust >= 1.87) — musl only
- **P-44:** macOS cannot use cross-rs — native CI runners required
- **P-46:** Windows must be GNU target, not MSVC, for Linux CI

## Suggested Phase Structure

| Phase | Name | Risk | Depends On |
|-------|------|------|------------|
| 1 | DE→EN Code Cleanup | Zero | — |
| 2 | Multi-Arch Binary Distribution | Medium | — |
| 3 | Connect-DSTM Spike + Implementation | High (spike-gated) | Phase 2 (binary rebuild) |

## Gaps to Address

- Connect-DSTM: `PlayerEvent::EndOfTrack` behavior in single-track-no-queue scenario unverified
- Grace timer calibration: 3-5s range needs empirical testing
- macOS CI runner availability and Bin/ directory naming convention
- Windows binary utility questionable (no Windows LMS users confirmed)

---
*Research completed: 2026-06-03*
*Ready for requirements: yes*
