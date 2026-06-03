# Technology Stack: SpotOn v1.1 (Connect-DSTM, Multi-Arch Binaries, Code Cleanup)

**Project:** SpotOn — Spotify plugin for Lyrion Music Server
**Researched:** 2026-06-03
**Scope:** Stack additions/changes for v1.1 milestone only. v1.0 stack is validated and unchanged.
**Confidence:** HIGH (cross-compilation verified against cross-rs docs; librespot Spirc internals verified against source; code cleanup tooling verified via codebase inspection)

---

## What This Document Covers

Three new capability areas added in v1.1:

1. **Connect-DSTM** — Auto-play in Connect mode when Spotify queue empties (Spirc event hook architecture)
2. **Multi-Arch Binaries** — Cross-compiling the `spoton` binary for all 6 distribution targets
3. **DE→EN Code Cleanup** — Systematic replacement of German comments and log strings with English

The existing v1.0 stack (Perl/LMS API, librespot 0.8.0, Spotify Web API, ZeroConf/Keymaster auth) is unchanged. Do not re-research those areas.

---

## Area 1: Connect-DSTM

### What needs to happen

When the Spotify Connect queue runs empty in Connect mode, librespot's Spirc emits `PlayerEvent::Stopped` (via `handle_stop()`) and `PlayerEvent::Paused`. The existing `spottyconnect stop` event already flows from the binary into `Connect.pm::_connectEvent`. Connect-DSTM hooks into that event path: when LMS receives `stop` in Connect mode and the player has a DSTM handler registered, SpotOn calls `Slim::Plugin::DontStopTheMusic::Plugin->dontStopTheMusic` to get the next track queue, then re-enters Connect via a new `spotify://connect-<ts>` playlist-play.

### Spirc Autoplay Behavior (HIGH confidence — verified against librespot-org/librespot spirc.rs)

Spirc 0.8.0 implements self-contained autoplay: when `add_autoplay_resolving_when_required()` fires (triggered after each `EndOfTrack`), Spirc checks whether `session.autoplay()` is true and whether there are fewer than `CONTEXT_FETCH_THRESHOLD` tracks remaining. If so, it asynchronously fetches an autoplay context from Spotify's spclient and appends to the internal queue.

**Key implication for Connect-DSTM:** Spirc never signals LMS when autoplay fires. It silently loads more Spotify-side tracks internally. From LMS's perspective, the stream continues uninterrupted. LMS-side DSTM only becomes relevant when:
- Spirc's autoplay is disabled (`session.autoplay()` returns false), OR
- The autoplay context fetch fails (Spotify API error, network issue), OR
- The user explicitly stopped the queue from the Spotify app

In those cases, Spirc calls `handle_stop()` → `player.stop()`, which emits `PlayerEvent::Stopped`. That event reaches `LMS::handle_player_event` in `connect.rs` and sends `spottyconnect stop` to LMS.

### Event API Surface (HIGH confidence — verified against librespot-playback 0.8.0 docs.rs)

The `PlayerEvent` variants relevant to Connect-DSTM:

| Event | Variant | Fields | When Fired |
|-------|---------|--------|------------|
| `EndOfTrack` | `EndOfTrack { play_request_id, track_id }` | Base62 track ID | Track finishes; Spirc will call `handle_next` |
| `Stopped` | `Stopped { play_request_id, track_id }` | Base62 track ID | `handle_next` found no more tracks + autoplay unavailable |
| `AutoPlayChanged` | `AutoPlayChanged { auto_play: bool }` | bool | User toggled autoplay in Spotify app |
| `Paused` | `Paused { play_request_id, track_id, position_ms }` | position | Player paused |

The existing `connect.rs` event dispatcher already handles `Stopped` → sends `spottyconnect stop`. No new event types need to be added to the binary for basic Connect-DSTM.

### Implementation Architecture

**Approach A: Perl-side hook (recommended for spike)**

In `Connect.pm::_connectEvent`, when `cmd eq 'stop'` and the player is in Connect mode and DSTM is configured, call `Slim::Plugin::DontStopTheMusic::Plugin->dontStopTheMusic`. This requires:
- Detecting "Connect mode stop" vs "user pause" — the existing `$_activeConnectPlayer` check already differentiates
- Checking `Slim::Plugin::DontStopTheMusic::Plugin->isActive($client)` before invoking
- Calling DSTM `dontStopTheMusic($client, $cb)` where `$cb` receives a list of Spotify URIs and re-enters Connect mode via `playlist play spotify://connect-<ts>`

No binary changes needed. The `spottyconnect stop` event is already reliably delivered.

**Approach B: Binary-side autoplay flag forwarding (future)**

Add `PlayerEvent::AutoPlayChanged` handling in `connect.rs` to send `spottyconnect autoplay <0|1>` to LMS. Perl side tracks Spirc autoplay state. When autoplay is off and stop arrives, DSTM fires. When autoplay is on, stop events are suppressed (Spirc handles it). Requires one additional event type in the binary's notify vocabulary.

**Recommendation for spike:** Start with Approach A. No binary recompile needed. Validate the event timing and DSTM callback behavior. Move to Approach B only if autoplay-state tracking causes false-positive DSTM triggers.

### No New Rust Dependencies Needed

The existing librespot 0.8.0 `PlayerEvent::Stopped` is sufficient. No new crates, no new binary flags for the Perl-side spike approach.

### Integration Point with Existing Code

```
Binary: PlayerEvent::Stopped → LMS::notify("stop", "", "") → JSON-RPC to LMS
LMS:    Connect.pm::_connectEvent($cmd eq 'stop')
           → [NEW] check DSTM active + Connect mode
           → [NEW] Slim::Plugin::DontStopTheMusic::Plugin->dontStopTheMusic($client, $cb)
           → $cb->($client, [$spotify_uri, ...]) → playlist play spotify://connect-<ts>
```

The DSTM module (`DontStopTheMusic.pm`) already implements `dontStopTheMusic($client, $cb)` for Browse mode. Connect-DSTM reuses the same callback signature and the same search/seed logic. Only the `$cb` body changes: instead of plain `playlist play $uri`, it re-enters Connect mode via the Connect URL scheme.

---

## Area 2: Multi-Arch Cross-Compilation

### Current State

The existing binary at `Plugins/SpotOn/Bin/x86_64-linux/spoton` is dynamically linked (glibc, not musl-static). The other 5 Bin directories are empty. No `Cross.toml` exists. `cross 0.2.5` is installed.

### Target Matrix

| Platform | Rust Target Triple | Directory | Toolchain |
|----------|--------------------|-----------|-----------|
| Linux x86_64 | `x86_64-unknown-linux-musl` | `Bin/x86_64-linux/` | cross-rs (built-in) |
| Linux i386 | `i686-unknown-linux-musl` | `Bin/i386-linux/` | cross-rs (built-in) |
| Linux aarch64 | `aarch64-unknown-linux-musl` | `Bin/aarch64-linux/` | cross-rs (built-in) |
| Linux armv7 | `armv7-unknown-linux-musleabihf` | `Bin/armhf-linux/` | cross-rs (built-in) |
| macOS x86_64 | `x86_64-apple-darwin` | `Bin/darwin-x86_64/` | osxcross via custom cross image |
| macOS aarch64 | `aarch64-apple-darwin` | `Bin/darwin-aarch64/` | osxcross via custom cross image |
| Windows x86_64 | `x86_64-pc-windows-msvc` | `Bin/win32/` | cargo-xwin |

Note: The existing `Bin/arm-linux/` directory is reserved for a potential `arm-unknown-linux-musleabihf` (ARMv6) target if needed. Omit from initial build matrix unless demanded.

### Tool Selection

**Linux musl targets (x86_64, i686, aarch64, armv7): cross-rs 0.2.5**

cross-rs provides pre-built Docker images for all four Linux musl targets. Zero additional setup. The `Cross.toml` config file controls the build:

```toml
# librespot-spoton/Cross.toml
[build]
pre-build = []

[target.x86_64-unknown-linux-musl]
image = "ghcr.io/cross-rs/x86_64-unknown-linux-musl:main"

[target.i686-unknown-linux-musl]
image = "ghcr.io/cross-rs/i686-unknown-linux-musl:main"

[target.aarch64-unknown-linux-musl]
image = "ghcr.io/cross-rs/aarch64-unknown-linux-musl:main"

[target.armv7-unknown-linux-musleabihf]
image = "ghcr.io/cross-rs/armv7-unknown-linux-musleabihf:main"
```

Build command per target:
```bash
cd librespot-spoton
cross build --release --target x86_64-unknown-linux-musl
cross build --release --target i686-unknown-linux-musl
cross build --release --target aarch64-unknown-linux-musl
cross build --release --target armv7-unknown-linux-musleabihf
```

Output: `librespot-spoton/target/<triple>/release/spoton` — copy to `Plugins/SpotOn/Bin/<dir>/spoton`.

**Why musl over glibc for Linux:** musl produces fully static binaries with no glibc version dependency. A glibc binary built on Ubuntu 24.04 will fail on LMS appliances running older glibc versions (Synology NAS, Raspberry Pi OS Buster). musl eliminates this class of failure entirely.

**macOS targets: osxcross + custom cross-rs Docker image**

cross-rs cannot ship pre-built Apple Darwin images due to Apple SDK licensing. The workflow is:
1. Obtain macOS SDK (requires access to a macOS machine or Xcode download)
2. Package SDK via osxcross scripts: `tar czf MacOSX13.sdk.tar.xz MacOSX13.sdk/`
3. Build a custom Docker image using the cross-toolchains Dockerfile that includes osxcross
4. Reference the custom image in `Cross.toml`

This is a one-time setup per developer machine. The resulting image is not distributable due to Apple licensing.

Alternative approach for macOS builds: Build natively on macOS using `cargo build --release --target aarch64-apple-darwin` (on an Apple Silicon Mac) or `--target x86_64-apple-darwin` (on Intel). These are the simplest paths and avoid cross-compilation entirely.

**Recommendation:** Build macOS binaries natively on macOS hardware (CI runner or local Mac). Use cross-rs only for Linux targets where cross-compilation is genuinely needed.

**Windows target: cargo-xwin 0.22.0**

cargo-xwin cross-compiles to `x86_64-pc-windows-msvc` from Linux without requiring a Windows VM. It downloads MSVC CRT and Windows SDK components automatically via xwin.

Installation:
```bash
cargo install cargo-xwin
rustup target add x86_64-pc-windows-msvc
```

Build:
```bash
cd librespot-spoton
cargo xwin build --release --target x86_64-pc-windows-msvc
```

Requires: `clang` installed (`apt install clang`). Accepts Microsoft SDK license (must be confirmed once).

Note: cargo-zigbuild does NOT support Windows MSVC targets (Linux/macOS only). cargo-xwin is the correct tool for MSVC cross-compilation from Linux.

### Cargo Feature Flags for Distribution

The current `Cargo.toml` uses `rustls-tls-native-roots` for all TLS. For musl static builds, this is correct — rustls provides pure-Rust TLS with no OpenSSL system dependency. Do not switch to `native-tls` for musl targets (it would require a static OpenSSL, adding complexity).

For macOS and Windows: `rustls-tls-native-roots` works. macOS native TLS (`security-framework`) is not needed since rustls handles it.

### Binary Naming Convention

The `Helper.pm` binary discovery currently uses `Slim::Utils::Misc::findbin('spoton')` which searches LMS's registered `Bin/` subdirectories. LMS selects the subdirectory based on OS and architecture detection. Current `Bin/` layout:

```
Bin/x86_64-linux/spoton    → x86_64-unknown-linux-musl build
Bin/i386-linux/spoton      → i686-unknown-linux-musl build
Bin/aarch64-linux/spoton   → aarch64-unknown-linux-musl build
Bin/armhf-linux/spoton     → armv7-unknown-linux-musleabihf build
Bin/darwin-x86_64/spoton   → x86_64-apple-darwin build
Bin/darwin-aarch64/spoton  → aarch64-apple-darwin build
Bin/win32/spoton.exe       → x86_64-pc-windows-msvc build
```

`Helper.pm` currently has a German comment (`# aarch64 kann als Fallback armhf-Binaries verwenden`) about aarch64 falling back to armhf. This fallback logic — adding `armhf-linux` to `findBinPaths` when arch is aarch64 — must be preserved while the comment gets translated (Area 3).

### Build Script Pattern

A shell script `scripts/build-all.sh` in the repository root is the right artifact:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/../librespot-spoton"

# Linux musl (via cross-rs)
cross build --release --target x86_64-unknown-linux-musl
cross build --release --target i686-unknown-linux-musl
cross build --release --target aarch64-unknown-linux-musl
cross build --release --target armv7-unknown-linux-musleabihf

# Windows (via cargo-xwin, requires clang)
cargo xwin build --release --target x86_64-pc-windows-msvc

# macOS: build on macOS host (or CI runner with macOS)
# cargo build --release --target x86_64-apple-darwin
# cargo build --release --target aarch64-apple-darwin
```

Then copy outputs to `Plugins/SpotOn/Bin/`. The script documents what must be run on macOS separately.

---

## Area 3: DE→EN Code Cleanup

### Scope Assessment

The German-language occurrences in the codebase are **comments and inline code comments only** — no user-facing German strings (those are in `strings.txt` and are correctly multi-language via LMS i18n). Log messages are already in English throughout.

**German occurrences found (by file):**

| File | Line | German text | Action |
|------|------|-------------|--------|
| `Helper.pm` | 21 | `# aarch64 kann als Fallback armhf-Binaries verwenden` | Translate |
| `Helper.pm` | 71 | `# KRITISCH: 'spoton' nicht 'spotty' im Regex` | Translate |
| `Helper.pm` | 115 | `# Angepasster Binary-Finder um findbin() von LMS zu nutzen` | Translate |
| `Helper.pm` | 123 | `# auf 64 bit x86 zuerst x86_64-Build versuchen` | Translate |
| `Settings.pm` | 58 | `# Binary-Status an Template uebergeben` | Translate |
| `Settings.pm` | 70 | `# Checkbox: wenn nicht angehakt, sendet Browser keinen Wert — undef/leer wird zu 0` | Translate |
| `Settings.pm` | 75-76 | `# T-04.4-01: Input-Validierung — nur alphanumerisch...` | Translate |
| `Settings.pm` | 80 | `# T-04.4-01: nur alphanumerisch (Injection-Schutz)` | Translate |
| `Settings.pm` | 196 | `# Client-ID und Degraded-Mode-Status fuer Template (D-02, D-03)` | Translate |
| `Settings.pm` | 307-310 | German block comment for `_isDegradedMode` | Translate |
| `Plugin.pm` | 342-344 | `# Regex-Muster fuer Spotify-generierte...` | Translate |
| `Plugin.pm` | 415 | `# Kontext-Queueing (D-09/D-10) — XMLBrowser reiht alle Items des Feeds ein` | Translate |

**Rust files:** Zero German occurrences found in `connect.rs` and `main.rs`.

Total: ~12-15 comment lines across 4 Perl files. All are in `Helper.pm`, `Settings.pm`, and `Plugin.pm`.

### Tooling

**No special tooling needed.** This is direct text editing, not a scripted migration.

Recommended workflow:
1. Read each affected file completely before editing (required by GSD protocol)
2. Make targeted `Edit` tool replacements comment by comment
3. Verify context is preserved (reference IDs like `D-02`, `T-04.4-01` must be retained)
4. Run a final check: `grep -rn --include="*.pm" -E '#.*[äöüÄÖÜß]|# .*(KRITISCH|Angepasst|Aufrufstell|Zeigt|Binary-Status|Checkbox.*nicht|uebergeben|Laengencheck|Muster.*fuer)' Plugins/`

The grep pattern above is the verification command — zero output = cleanup complete.

**What NOT to do:**
- Do not use `sed -i` bulk replacement — too error-prone for context-sensitive comment translation
- Do not translate `strings.txt` — those are intentionally multi-language (LMS i18n)
- Do not change `$PERSONAL_MIX_REGEX = qr/...Mix der Woche.../` — that is a Spotify playlist name match string (data, not a comment)

### No New Dependencies

Pure text editing. No linting tools, no Perl formatters, no translation APIs.

---

## What NOT to Add

| Category | Avoid | Reason |
|----------|-------|--------|
| Cross-compilation | cargo-zigbuild for Windows | Does not support MSVC targets; use cargo-xwin |
| Cross-compilation | Native ARM compile on Pi | Too slow; cross-rs is the right tool |
| Cross-compilation | cross-rs for macOS | Cannot ship Apple SDK images; use native macOS build |
| Connect-DSTM | New binary flags for DSTM | Not needed for Perl-side Approach A spike |
| Connect-DSTM | librespot 0.9.x upgrade | 0.8.0 is latest stable; 0.9.x does not exist |
| Code cleanup | Automated translation API | Overkill for 12-15 lines; manual edit is correct |
| Code cleanup | Perl linter (Perl::Critic) | Not bundled with LMS; no CPAN in constraints |
| Binary | ALSA/PulseAudio backends | Increases binary size; pipe + HTTP streaming covers all cases |
| Binary | arm-linux (ARMv6) | Not in target matrix; `armhf-linux` covers Pi 2/3 |

---

## Integration with Existing Build System

The `librespot-spoton/.cargo/config.toml` documents the musl approach:

> "LMS deployment binaries: use cross-rs with musl target for fully static binaries: `cross build --release --target x86_64-unknown-linux-musl`"

This is already the documented intent. v1.1 execution task is to:
1. Create `Cross.toml` with the 4 Linux musl targets
2. Run the cross-rs builds
3. Run cargo-xwin for Windows
4. Obtain macOS binaries (native build)
5. Copy all outputs to `Plugins/SpotOn/Bin/<dir>/spoton`

No changes to `Cargo.toml` or `Cargo.lock` are needed. The dependency set is unchanged.

---

## Version Compatibility

| Tool | Version | Notes |
|------|---------|-------|
| cross-rs | 0.2.5 (installed) | Supports all 4 Linux musl targets built-in |
| cargo-xwin | 0.22.0 (latest Apr 2026) | Windows MSVC cross-compile from Linux |
| librespot | 0.8.0 (locked in Cargo.lock) | No upgrade needed; sufficient for DSTM |
| Rust toolchain | stable 1.96.0 (installed) | Supports all 6 targets with rustup target add |
| clang | system | Required by cargo-xwin for MSVC cross-compile |

---

## Sources

- librespot-playback PlayerEvent enum: https://docs.rs/librespot-playback/0.8.0/librespot_playback/player/enum.PlayerEvent.html (HIGH — official docs.rs)
- librespot Spirc source (spirc.rs, dev branch): https://github.com/librespot-org/librespot/blob/dev/connect/src/spirc.rs (HIGH — verified handle_next, add_autoplay_resolving_when_required, handle_stop)
- librespot CHANGELOG: https://github.com/librespot-org/librespot/blob/dev/CHANGELOG.md (HIGH — autoplay spclient migration, 0.8.0 features)
- cross-rs supported targets: https://github.com/cross-rs/cross (HIGH — confirmed built-in musl targets, cross-toolchains for Darwin/MSVC)
- cross-toolchains (Apple Darwin images): https://github.com/cross-rs/cross-toolchains (HIGH — confirms licensing reason for no pre-built Darwin images)
- cargo-xwin: https://github.com/rust-cross/cargo-xwin (HIGH — Windows MSVC cross-compile, v0.22.0 Apr 2026)
- cargo-zigbuild: https://github.com/rust-cross/cargo-zigbuild (HIGH — confirmed no Windows MSVC support)
- SpotOn codebase: /home/sti/spoton (HIGH — direct inspection of German occurrences, existing Cargo.toml, .cargo/config.toml, Bin/ structure, connect.rs Spirc event handling)

---

*Stack research for: SpotOn v1.1 — Connect-DSTM, Multi-Arch Binaries, DE→EN Cleanup*
*Researched: 2026-06-03*
