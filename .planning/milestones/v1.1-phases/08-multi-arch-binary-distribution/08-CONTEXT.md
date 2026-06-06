# Phase 8: Multi-Arch Binary Distribution - Context

**Gathered:** 2026-06-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Cross-compile the librespot-spoton binary for all supported platform targets and extend Helper.pm to automatically detect the host platform and select the correct binary. The current glibc-linked x86_64 binary is replaced with a musl-static build.

Scope: 5 Linux targets (musl-static) + 1 Windows target (GNU). macOS is deferred to v1.2 (no macOS hardware available for native builds — P-44 prohibits cross-rs for macOS).

</domain>

<decisions>
## Implementation Decisions

### Build Infrastructure
- **D-01:** All Linux targets are built locally using cross-rs (Docker-based cross-compilation). No CI/CD pipeline in Phase 8.
- **D-02:** Windows target (`x86_64-pc-windows-gnu`) is built via cross-rs from Linux. MinGW-w64 GNU target, not MSVC (P-46).
- **D-03:** macOS targets (ARCH-05, ARCH-06) are deferred to v1.2. No Mac available; P-44 prohibits cross-rs for macOS. These requirements move to v1.2+ in REQUIREMENTS.md.
- **D-04:** All Linux binaries use musl-static linking (`*-unknown-linux-musl` targets). No glibc dependency. P-42 confirms: `+crt-static` on GNU targets breaks proc-macro — musl only.
- **D-05:** rustls-tls is already configured in Cargo.toml — no system OpenSSL dependency, which is required for musl-static builds.

### Binary Distribution
- **D-06:** All binaries are committed directly to the Git repo under `Plugins/SpotOn/Bin/`. No separate release download mechanism. This matches Spotty-NG convention and keeps install.xml simple.
- **D-07:** Expected total repo size: ~85-120MB (6 binaries × ~15-20MB each). Acceptable for an LMS plugin.

### Directory Naming
- **D-08:** Existing directory names are kept. The mapping from Rust target triple to Bin/ subdirectory:

| Rust Target Triple | Bin/ Directory | Platform |
|---|---|---|
| `x86_64-unknown-linux-musl` | `x86_64-linux/` | x86_64 Linux (existing, replace glibc binary) |
| `aarch64-unknown-linux-musl` | `aarch64-linux/` | Pi 4/5, NAS (existing dir, empty) |
| `armv7-unknown-linux-musleabihf` | `armhf-linux/` | Pi 2/3 32-bit (existing dir, empty) |
| `arm-unknown-linux-musleabihf` | `arm-linux/` | Pi 1/Zero ARMv6 (existing dir, empty) |
| `i686-unknown-linux-musl` | `i386-linux/` | 32-bit Intel (existing dir, empty) |
| `x86_64-pc-windows-gnu` | `x86_64-win64/` | Windows 64-bit (NEW directory) |

- **D-09:** Naming follows Herger/Spotty-NG convention, not Rust triple names. Consistent with LMS plugin ecosystem.

### Helper.pm Platform Detection
- **D-10:** Helper.pm uses `addFindBinPaths()` with platform-specific directories. LMS `findbin()` then locates the binary automatically.
- **D-11:** A new `_detectArch()` function maps the host platform to the correct Bin/ subdirectory name using `main::ISWINDOWS`, `main::ISMAC`, and `Slim::Utils::OSDetect::details()->{osArch}` plus `$Config::Config{'archname'}`.
- **D-12:** Fallback chain for ARM: aarch64 → armhf (32-bit compat), armv7 → arm (armv6 compat). This ensures graceful degradation when the optimal binary is missing.
- **D-13:** `spoton-custom` override remains first in the candidate list (LMS-10 preparation).

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/PROJECT.md` — Pitfalls P-42 (musl only), P-44 (no macOS cross-rs), P-46 (Windows GNU target)
- `.planning/REQUIREMENTS.md` — ARCH-01 through ARCH-10 requirement definitions
- `CLAUDE.md` §librespot — Build flags, audio backends, target triples, CLI flags

### Source Files
- `Plugins/SpotOn/Helper.pm` — Current binary detection logic (162 LOC). Primary file to modify for platform detection.
- `librespot-spoton/Cargo.toml` — Build configuration, features, dependencies. rustls-tls already configured.

### Reference Implementation
- Herger's Spotty-Plugin Helper.pm: `https://github.com/michaelherger/Spotty-Plugin` — Binary detection patterns for multi-arch LMS plugins

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Helper.pm::_findBin()` — existing binary finder with candidate list pattern. Extend candidate list for multi-arch.
- `Helper.pm::helperCheck()` — binary validation via `--check` flag. Works for any platform binary.
- `Helper.pm::init()` — already has aarch64→armhf fallback via `addFindBinPaths()`. Generalize this pattern.
- `addFindBinPaths()` from `Slim::Utils::Misc` — LMS utility to add search paths for findbin().

### Established Patterns
- Binary naming: `spoton` (no platform suffix in filename). Each platform gets its own subdirectory.
- Custom override: `spoton-custom` is always searched first. Must remain at top of candidate list.
- `--check` output: `ok spoton vX.Y.Z\n{json_capabilities}` — parsed by `helperCheck()`. All new binaries must emit this.
- Binary path stored in `$prefs->get('binary')` — user can override path via Settings.

### Integration Points
- `Helper.pm::init()` is called from `Plugin.pm::initPlugin()` — platform detection runs at plugin startup.
- `DaemonManager.pm` and `ProtocolHandler.pm` call `Helper->get()` for the binary path — no changes needed there, they're agnostic to which platform binary is returned.
- `Settings.pm` displays binary path and version — works with any binary, no changes needed.

</code_context>

<specifics>
## Specific Ideas

No specific requirements — standard cross-compilation and platform detection approach.

</specifics>

<deferred>
## Deferred Ideas

- **macOS Universal Binary (ARCH-F01):** Fat binary combining x86_64 + aarch64 macOS. Deferred to v1.2 with ARCH-05 and ARCH-06.
- **GitHub Actions CI:** Automated builds on release tag. Useful once project is stable and public. v1.2+.
- **Binary size optimization:** `strip` and UPX compression to reduce repo size. Not needed for v1 — correctness first.

</deferred>

---

*Phase: 8-Multi-Arch Binary Distribution*
*Context gathered: 2026-06-03*
