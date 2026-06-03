# Domain Pitfalls: SpotOn v1.1 — Connect-DSTM, Multi-Arch Binaries, Code Cleanup

**Domain:** LMS Spotify plugin — v1.1 milestone additions (Perl + Rust binary)
**Researched:** 2026-06-03
**Confidence:** HIGH (verified against existing codebase + librespot upstream issues)
**Scope:** NEW pitfalls for v1.1 features ONLY — P-01 through P-39 from v1.0 are excluded

---

## Critical Pitfalls

### P-40: Connect-DSTM — `Spirc::next()` vs LMS DSTM Callback Architecture Mismatch

**What goes wrong:**
The existing DSTM in `DontStopTheMusic.pm` is driven by the LMS DSTM framework: LMS calls `dontStopTheMusic($client, $cb)` when the LMS playlist reaches end-of-queue. In Connect mode, LMS never owns the queue — Spirc does. When the Spotify context ends, the librespot Spirc emits `EndOfTrack` / `Stopped` internally and handles next-track selection itself. There is no LMS DSTM callback invoked because no LMS playlist event fires. Plugging the existing `dontStopTheMusic` sub into Connect mode would require LMS to observe a playlist-end event that never happens.

**Why it happens:**
Browse-mode DSTM and Connect-mode DSTM are architecturally different. Browse mode: LMS owns the playlist, LMS detects end. Connect mode: Spirc owns the context, Spirc detects end. They cannot share the same entry point.

**How to avoid:**
Connect-DSTM requires one of two approaches, which is the spike decision:

1. **Binary-side autoplay:** Hook librespot's `EndOfTrack` event in `connect.rs`. When Spirc's queue is empty, send an LMS JSON-RPC `spottyconnect endofqueue` event. Perl catches it in `_connectEvent`, calls the existing `dontStopTheMusic` logic, and issues `spirc.next_uri(uri)` via a new `/control/queue` HTTP endpoint on the daemon. Requires binary changes.

2. **LMS playlist injection:** When `Stopped` fires from an empty Spirc context, detect the empty-queue condition on the Perl side (no `change` follows within 2s of `stop`), then invoke the DSTM callback and inject the results into the Connect playlist via `PUT /me/player/play` (Spotify Web API) with `uris` parameter. No binary changes needed, but requires active Web API access at end-of-track time.

Approach 1 is more reliable (no 2s heuristic). Approach 2 is simpler (no binary fork needed).

**Warning signs:**
If implementing approach 2: a 2-second heuristic fires false positives when the user legitimately pauses Connect at end of queue. Add a queue-emptiness check via `GET /me/player/queue` before injecting.

**Phase to address:** Connect-DSTM Spike (Phase 1 of v1.1)

---

### P-41: Connect-DSTM — `recommendations` Endpoint Removed in Dev Mode

**What goes wrong:**
The existing `_getRecommendations()` in `DontStopTheMusic.pm` calls `Plugins::SpotOn::API::Client->recommendations()`. The `/recommendations` endpoint was removed for Development Mode apps on 2024-11-27. Any Connect-DSTM implementation that reuses this code path will silently return empty results — the callback fires with no tracks, DSTM yields no playback, and Connect mode goes silent at end of queue.

**Why it happens:**
The existing Browse-mode DSTM already has a `_searchFallback` that handles the `recommendations` removal, but the fallback is tied to the `_getRecommendations` flow. If Connect-DSTM skips that flow and calls `recommendations` directly, it hits the removed endpoint.

**How to avoid:**
Connect-DSTM must reuse the complete `_searchFallback` chain from `DontStopTheMusic.pm`, not call `recommendations` directly. Specifically: ensure the code path always falls through to `_searchFallback($client, $accountId, $seedArtist, $cb)` when recommendations returns 404/empty. The seed artist extraction from `$seedData->{_firstArtistName}` already handles this — do not bypass it.

**Warning signs:**
DSTM appears to work in testing (search fallback fires) but silently fails in a fresh test that calls `recommendations` directly. Check API::Client logs for 404 on the `/recommendations` endpoint.

**Phase to address:** Connect-DSTM implementation

---

### P-42: Cross-Compilation — `+crt-static` on `x86_64-unknown-linux-gnu` Breaks Proc-Macro Crates

**What goes wrong:**
The existing `config.toml` documents this exact pitfall: adding `RUSTFLAGS="-C target-feature=+crt-static"` to `x86_64-unknown-linux-gnu` fails with:

```
cannot produce proc-macro for async-trait as target does not support these crate types
```

Rust >= 1.87 enforces that proc-macro crates must be compiled as dynamic libraries on the build host. `+crt-static` prevents that. This surfaces when building with GitHub Actions runners that have recent Rust toolchain.

**Why it happens:**
`proc-macro` crates are loaded by the compiler at compile time as dynamic libraries. Static CRT linkage (`+crt-static`) conflicts with this requirement on `x86_64-unknown-linux-gnu`. The musl target (`x86_64-unknown-linux-musl`) does not have this conflict because musl targets are statically linked by design without the `gnu` toolchain's proc-macro limitation.

**How to avoid:**
Never use `+crt-static` on `-gnu` targets. Use `-musl` targets exclusively for static binaries. The cross-compilation workflow must target `x86_64-unknown-linux-musl` (not `x86_64-unknown-linux-gnu`) for LMS deployment builds. `config.toml` already documents this — do not regress.

**Warning signs:**
CI step fails with "cannot produce proc-macro" error message. This only appears on `gnu` targets with `+crt-static`, not on `musl` targets.

**Phase to address:** Multi-Arch binary build pipeline

---

### P-43: Cross-Compilation — `native-tls` Requires System OpenSSL; Use `rustls-tls-native-roots`

**What goes wrong:**
If any librespot dependency is switched from `rustls-tls-native-roots` to `native-tls`, cross-compilation for musl targets fails with linker errors: `cannot find -lssl` or `undefined reference to GLIBC_*`. System OpenSSL is a C library built against glibc; musl targets cannot link it without a vendored OpenSSL build. The existing `Cargo.toml` already uses `rustls-tls-native-roots` on all four librespot crates — this is the correct configuration. Any dependency added in v1.1 (e.g., for DSTM HTTP calls, queue injection) must also use rustls, not native-tls.

**Why it happens:**
`cross-rs` provides musl toolchains but does not bundle OpenSSL. `native-tls` resolves to the system OpenSSL, which is glibc-linked. musl cannot link glibc objects without special vendored builds.

**How to avoid:**
All librespot crate features must stay on `rustls-tls-native-roots`. Do not add any crate that transitively pulls in `openssl-sys` with `native-tls`. Audit with `cargo tree --features … | grep openssl` before adding new dependencies. If a crate forces `native-tls`, use its `vendored` feature or find a rustls-compatible alternative.

**Warning signs:**
`cross build --target x86_64-unknown-linux-musl` fails at link step with SSL library errors. On local dev build (glibc), the same `cargo build` succeeds — the difference only appears in cross-compilation.

**Phase to address:** Multi-Arch binary build pipeline

---

### P-44: Cross-Compilation — macOS Targets Require SDK Extraction; Cannot Use Cross-rs Directly

**What goes wrong:**
`cross-rs` does not ship Apple SDK images due to Apple's licensing restrictions. The targets `x86_64-apple-darwin` and `aarch64-apple-darwin` cannot be cross-compiled from a Linux CI runner using the standard `cross build` command — it will fail with missing toolchain errors.

**Why it happens:**
Apple requires developers to accept its EULA to use the macOS SDK. No open-source CI tool can redistribute the SDK. `cross-rs` requires custom Docker images built from `cross-toolchains` + `osxcross`, which in turn requires manually extracting an `.xip` or `.dmg` SDK.

**How to avoid:**
For macOS targets: use native GitHub Actions `macos-latest` or `macos-14` runners. Native runners have Xcode installed and can compile `x86_64-apple-darwin` and `aarch64-apple-darwin` natively (including the `aarch64-apple-darwin` → `x86_64-apple-darwin` cross, which macOS supports natively via Rosetta build tooling). Do not attempt Linux → macOS cross-compilation. Structure the GitHub Actions matrix so Linux musl targets use `cross`, macOS targets use native macOS runners, and Windows uses native Windows runners.

**Warning signs:**
CI step for Darwin target fails with "no such toolchain" or linker not found. The fix is always runner selection, not toolchain configuration.

**Phase to address:** Multi-Arch binary build pipeline — CI matrix design

---

### P-45: Cross-Compilation — armv7 musl Thread-Local Storage Error on Older ARM Devices

**What goes wrong:**
When cross-compiling for `armv7-unknown-linux-musleabihf` targeting older ARM SoCs (pre-ARMv7 compatible kernels), the binary may fail at runtime with:

```
Error getting 1120 bytes thread-local storage: No such file or directory
```

This is a musl libc TLS implementation incompatibility with older kernels that do not properly support the TLS interface.

**Why it happens:**
musl uses a static TLS model that requires kernel support for `__tls_get_addr`. Older kernels (< 3.0 era) or devices using non-standard TLS initialization may not provide this. This primarily affects armv6 (Pi Zero, Pi 1) rather than armv7, but some armv7 builds targeting musl with `+crt-static` exhibit it.

**How to avoid:**
If armv7 musl binaries show this error on specific target devices, use `RUSTFLAGS="-C target-feature=-crt-static"` for that target only. This produces a partially dynamic binary that still statically links musl but dynamically handles TLS. Add this as a conditional override in the GitHub Actions matrix for the `armv7-unknown-linux-musleabihf` target specifically. Verify on actual hardware (not QEMU only).

**Warning signs:**
Binary passes all CI tests in QEMU emulation but crashes on real armv7 hardware with the TLS error on startup.

**Phase to address:** Multi-Arch binary build pipeline — armv7 target validation

---

### P-46: Cross-Compilation — Windows Target Must Use `x86_64-pc-windows-gnu`, Not MSVC

**What goes wrong:**
`x86_64-pc-windows-msvc` cannot be cross-compiled from Linux CI runners. The MSVC linker (`link.exe`) is only available on Windows. Attempting to use it on a Linux runner fails with a missing linker error. The GNU Windows target (`x86_64-pc-windows-gnu`) uses MinGW toolchain which is available on Linux CI and can be installed via `apt`.

**Why it happens:**
MSVC is a proprietary Microsoft toolchain that cannot be redistributed or run on Linux. The GNU ABI Windows target (`-gnu`) produces binaries that run on all Windows versions without requiring MSVC redistributables, using the freely available MinGW-w64 toolchain.

**How to avoid:**
Use `x86_64-pc-windows-gnu` as the Windows target in the CI matrix. Install MinGW-w64 on the Linux runner (`apt-get install mingw-w64`). Set `CARGO_TARGET_X86_64_PC_WINDOWS_GNU_LINKER=x86_64-w64-mingw32-gcc`. LMS does not run on Windows (LMS server is Linux/macOS only), but the binary may be useful for local Spotify testing — `gnu` ABI is sufficient.

**Warning signs:**
Build matrix references `windows-msvc` on a Linux runner. Error message: "linker `link.exe` not found".

**Phase to address:** Multi-Arch binary build pipeline — CI matrix design

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Reuse Browse-mode DSTM for Connect via 2s heuristic | No binary changes | False positives on legitimate end-pause; fragile timing | Never — use approach 1 or approach 2 with queue check |
| Use `native-tls` for a new dependency | Easier dependency mgmt | Breaks all musl cross-compilation | Never — rustls only |
| Cross-compile macOS from Linux with osxcross | Single CI platform | Complex SDK extraction; licensing issues; hours of setup | Only if native macOS runners unavailable (they are) |
| Single GitHub Actions job for all targets | Simple workflow | Slow; can't use target-specific runners | Never — use matrix |
| Regex-replace German comments without context check | Fast migration | May corrupt log string patterns, regex literals | Never — review each change |
| Replace all `# Foo` comments across all `.pm` files in one commit | Fast | Hard to review; no way to verify correctness per file | Never — file-by-file commits |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Connect-DSTM + Spotify Web API | Call `PUT /me/player/play` with `uris` without first checking `GET /me/player/queue` | Check queue emptiness before injecting; avoid false-positive DSTM on user-initiated pause |
| Connect-DSTM + binary event | Map `Stopped` event to "end of queue" without distinguishing pause vs context end | Add a new `endofqueue` event type in the binary's event vocabulary; do not reuse `stop` |
| Binary stdout + stderr | Mix `stream_port=N` stdout announce with debug logs | Debug logs go to stderr (via `env_logger`); stdout is exclusively for structured machine-readable output |
| Cross-compile + `cargo-cross` | Use `cross` for macOS and Windows targets | `cross` only works for Linux targets without manual SDK setup; use native runners for macOS/Windows |
| Code cleanup + strings.txt | Replace German strings in `.pm` log calls while leaving `strings.txt` DE translations intact | `strings.txt` German translations are intentional i18n data — never touch DE locale lines |
| Code cleanup + regex patterns | Replace comment text that appears inside a regex: `# für …` becomes `for` but `s/für/for/g` corrupts embedded strings | Use `grep -n` to verify each match is truly a comment before replacing; avoid `sed -i` mass replacements |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Connect-DSTM calls `GET /me/player/queue` on every `stop` event | 429 rate-limit bursts when user pauses repeatedly | Cache queue-empty result for 5s; only check queue once per `stop` event | At any user who pauses/resumes rapidly |
| GitHub Actions matrix with 6 targets as single sequential job | 40+ min CI wall-clock time | Parallelize with `strategy.matrix`; each target is independent | First use of the pipeline |
| Spawning `cross` on macOS runner | `cross` requires Docker; macOS runners have Docker unavailable by default | Use `cargo` directly on macOS runners; `cross` only on Linux | macOS runner without Docker Desktop |
| armv7 binary validated only in QEMU | TLS error on real hardware (P-45) | Always validate armv7 binary on real hardware or a known-compatible Docker image | On actual Raspberry Pi 2/3 32-bit |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Logging `stream_port=N` to stderr (not stdout) | Port announced via wrong channel; Perl IO::Select never reads it; daemon hangs | Strictly: stdout = machine-readable announces; stderr = human-readable logs |
| Binary shipped without version in `--check` output | Plugin cannot enforce minimum binary version for new DSTM feature | Add `dstm_capable: true` to `--check` JSON if Connect-DSTM binary changes land; check in Helper.pm |
| Code cleanup replaces a hardcoded rate-limit error string used in log grep | Ops/monitoring grep breaks silently | Document all log string patterns used in external monitoring before cleanup begins |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Connect-DSTM injects tracks that interrupt an in-progress Spotify app queue | User had queued songs; DSTM replaces them | Only inject when queue is truly empty AND Spirc context has ended; never inject into a non-empty queue |
| Multi-arch binary page in repo.xml lists all 6 targets but only x86_64 is tested | Users on ARM see binary but get broken playback | Mark non-x86_64 targets as "beta" in repo.xml description until validated on each platform |
| Code cleanup changes a translated string that appears in the LMS UI | German-speaking users see broken text | Translation strings in `strings.txt` are out of scope for the DE→EN cleanup; only code comments and log messages |

---

## "Looks Done But Isn't" Checklist

- [ ] **Connect-DSTM:** The `endofqueue` event fires from the binary — verify the Perl `_connectEvent` handler has a branch for it; the existing switch does not.
- [ ] **Connect-DSTM:** Search fallback uses random offset — verify `_searchFallback` is reachable from Connect-DSTM code path (not just Browse-mode DSTM).
- [ ] **Multi-Arch:** All 6 binaries listed in `install.xml` `Bin/` — verify each target's binary name matches what `Helper.pm`'s OS/arch detection expects.
- [ ] **Multi-Arch:** `rustls-tls-native-roots` on every librespot crate in `Cargo.toml` — verify with `cargo tree --features rustls-tls-native-roots | grep native-tls` (should be empty).
- [ ] **Multi-Arch:** armv7 binary tested on real hardware or known-good Docker container, not only QEMU.
- [ ] **Multi-Arch:** macOS binaries built on native macOS runner — verify CI matrix uses `runs-on: macos-latest` for Darwin targets.
- [ ] **Code Cleanup:** `strings.txt` DE locale lines untouched — diff `strings.txt` shows zero changes to `DE` lines.
- [ ] **Code Cleanup:** No German text in `log->info()/warn()/error()` calls — verify with `grep -rn "log.*[äöüÄÖÜß]"` in `Plugins/`.
- [ ] **Code Cleanup:** No German comments (`# ... [äöüÄÖÜß]`) remain in `.pm` files — verify with `grep -rn "#.*[äöüÄÖÜß]"` in `Plugins/`.
- [ ] **Code Cleanup:** `Settings.pm` German comment blocks at lines 69-77 and 307-310 translated — these are the only confirmed German code comments in the codebase.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| P-40: Wrong Connect-DSTM architecture | HIGH | Revert to spike result; choose approach before implementation; do not mix both approaches |
| P-41: recommendations 404 in Connect-DSTM | LOW | Redirect call through existing `_searchFallback`; one-line fix if code is modular |
| P-42: +crt-static breaks proc-macro | LOW | Remove `+crt-static` from gnu target in `config.toml`; switch to musl target |
| P-43: native-tls breaks musl cross-compile | MEDIUM | Audit `cargo tree`, find offending crate, switch to rustls feature or vendored openssl |
| P-44: macOS cross-compile attempt | LOW | Switch CI matrix to use `macos-latest` native runner; no code changes needed |
| P-45: armv7 TLS error on real hardware | LOW | Add `RUSTFLAGS="-C target-feature=-crt-static"` for armv7 target only in CI matrix |
| P-46: MSVC linker not found on Linux | LOW | Change target to `x86_64-pc-windows-gnu`; install `mingw-w64` on Linux runner |
| Code cleanup regex corruption | MEDIUM | `git diff` to find corrupted lines; restore from git; apply surgical per-line replacements |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| P-40: DSTM architecture mismatch | Connect-DSTM Spike (Phase 1) | Spike decision document; prototype tested end-to-end in Connect mode |
| P-41: recommendations removed | Connect-DSTM implementation | Test DSTM in Connect mode; verify `_searchFallback` is reached via log |
| P-42: +crt-static proc-macro | Binary build pipeline (Phase 2) | CI musl build succeeds; no `+crt-static` on gnu targets |
| P-43: native-tls musl failure | Binary build pipeline (Phase 2) | `cargo tree \| grep openssl` is empty for musl builds |
| P-44: macOS cross-compile | Binary CI matrix (Phase 2) | Darwin targets use `macos-latest` runner in CI; build succeeds |
| P-45: armv7 TLS runtime crash | armv7 validation (Phase 2) | armv7 binary tested on real Pi 2/3 hardware or equivalent |
| P-46: MSVC Windows target | Binary CI matrix (Phase 2) | Windows target uses `windows-gnu`; CI build succeeds on Linux runner |
| Code cleanup regressions | Code cleanup (Phase 3) | `grep -rn "[äöüÄÖÜß]" Plugins/ \| grep -v strings.txt` returns zero results |
| Cleanup breaks log grep patterns | Code cleanup review (Phase 3) | Known log patterns documented in CLAUDE.md before cleanup begins |
| Cleanup corrupts strings.txt DE | Code cleanup review (Phase 3) | `git diff strings.txt` shows zero changes to DE locale lines |

---

## Sources

- librespot `EndOfTrack` / autoplay issues: https://github.com/librespot-org/librespot/issues/1205
- librespot empty queue hang: https://github.com/librespot-org/librespot/issues/1192
- librespot autoplay attribute failure: https://github.com/librespot-org/librespot/issues/1046
- librespot cross-compilation wiki: https://github.com/librespot-org/librespot/wiki/Cross-compiling
- librespot issue #1534 (OpenSSL cross-compile): https://github.com/librespot-org/librespot/issues/1534
- librespot issue #1184 (arm musl cross-compile): https://github.com/librespot-org/librespot/issues/1184
- musl libc functional differences (DNS, TLS): https://wiki.musl-libc.org/functional-differences-from-glibc.html
- cross-rs macOS SDK licensing: https://github.com/cross-rs/cross (Apple target images not provided)
- Rust forum: cross-compile macOS from Linux: https://users.rust-lang.org/t/is-cross-compile-from-linux-to-mac-supported/95105
- houseabsolute/actions-rust-cross (recommended GitHub Action): https://github.com/houseabsolute/actions-rust-cross
- rustls vs native-tls for musl: https://oneuptime.com/blog/post/2026-03-04-compile-rust-musl-static-binaries-rhel/view
- Spotify recommendations endpoint removal: https://developer.spotify.com/blog/2024-11-27-changes-to-the-web-api
- SpotOn existing config.toml noting +crt-static proc-macro failure: /home/sti/spoton/librespot-spoton/.cargo/config.toml

---
*Pitfalls research for: SpotOn v1.1 — Connect-DSTM, Multi-Arch Binaries, Code Cleanup*
*Researched: 2026-06-03*
