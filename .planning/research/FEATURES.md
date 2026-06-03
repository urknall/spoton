# Feature Landscape: SpotOn v1.1 — Connect-DSTM, Multi-Arch Binaries, Code Cleanup

**Domain:** Spotify integration plugin for Lyrion Music Server — v1.1 Hardening & Reach milestone
**Researched:** 2026-06-03
**Confidence:** HIGH (based on live librespot 0.8.0 source, existing SpotOn codebase, GitHub Actions workflow)

---

## Context: What v1.0 Already Ships

v1.0 is complete. The three features in scope for v1.1 are additions and cleanup, not rewrites:

| Area | v1.0 State | v1.1 Goal |
|------|-----------|-----------|
| DSTM | Browse-mode only (LMS framework, `Slim::Plugin::DontStopTheMusic::Plugin`) | Add Connect-mode DSTM (Spirc-native autoplay OR LMS-side fallback) |
| Binaries | x86_64-linux only (1 of 6 targets built) | All 6 targets (+ macOS x86_64/aarch64, Windows x86_64) |
| Code language | Mixed DE/EN comments and log strings (282 occurrences identified) | English only, throughout all Perl and Rust files |

---

## Feature Area 1: Connect-DSTM

### What "Connect-DSTM" Means

When playback happens via Spotify Connect (the user controls LMS from the Spotify app), the
existing Browse-DSTM (`DontStopTheMusic.pm`) does not fire — it is registered as an LMS
framework handler and only triggers when LMS's own playlist queue runs out in Browse mode.
In Connect mode, audio comes from librespot's Spirc loop. The queue end happens inside the
binary, not inside LMS.

### How librespot 0.8.0 Handles Autoplay Natively

librespot-connect 0.8.0 has built-in autoplay support controlled by `SessionConfig.autoplay`:

```rust
// librespot-core 0.8.0 SessionConfig
pub autoplay: Option<bool>
// None = read from Spotify user attribute "autoplay" (account setting)
// Some(true)  = force on
// Some(false) = force off
```

The autoplay resolution is implemented entirely inside `spirc.rs`:

- `add_autoplay_resolving_when_required()` is called whenever Spirc processes an
  `EndOfTrack` event (via `handle_next()`), after context changes, and after context loads
- It calls `spclient.get_autoplay_context()` to fetch Spotify's server-side radio/seed
  context for the currently playing context URI (album, playlist, artist)
- Tracks from this autoplay context are appended to `connect_state.autoplay_context`
- The `Stopped`/`EndOfTrack` → `handle_next()` path advances into the autoplay context
  seamlessly without any LMS or plugin involvement

**Critical finding:** The current `connect.rs` in SpotOn creates `SessionConfig::default()`
which sets `autoplay: None`, meaning autoplay follows the user's Spotify account setting.
If the user has autoplay enabled in their Spotify account, it works now. The only missing
piece is: (1) a flag `--autoplay true/false` to force it regardless of account setting,
and (2) verifying the binary actually handles it correctly in practice.

The `ConnectConfig` struct in librespot-connect 0.8.0 has NO `autoplay` field — this was
confirmed by reading the struct definition directly. Autoplay is controlled via `SessionConfig`,
not `ConnectConfig`. The v1.0 comment in `connect.rs` line 959 ("No `autoplay` field") refers
to the old `ConnectConfig` but misses that `SessionConfig.autoplay` is the correct lever.

### Table Stakes (for Connect-DSTM)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Verify librespot autoplay works with current binary | Users expect queue to continue like the Spotify app does | LOW | `session_config.autoplay = Some(true)` in `run_connect()`. Requires build + test |
| Expose `--autoplay` flag in binary | Perl side needs to control this per-player | LOW | One arg parse addition in `main.rs`, pass through to `run_connect()` |
| Connect daemon restart with autoplay flag | Daemon.pm passes `--autoplay` to binary | LOW | Follow same pattern as `--bitrate`, `--disable-discovery` flags |

### Differentiators (for Connect-DSTM)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| LMS-side fallback when librespot autoplay fails | Robust: if Spirc autoplay returns nothing, LMS can queue via DSTM | MEDIUM | Detect `Stopped` event with empty autoplay context; call `DontStopTheMusic.pm` logic |
| Per-player autoplay toggle | Power users who want different behavior per room | LOW | Per-player pref `connectAutoplay`, passed as `--autoplay 0/1` to binary |
| Seamless transition: Browse DSTM and Connect DSTM use same seeds | Consistent behavior regardless of mode | MEDIUM | Share seed extraction logic |

### Anti-Features (for Connect-DSTM)

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Reimplementing autoplay context resolution in Perl | librespot 0.8.0 already does this via spclient; duplicating it means two implementations to maintain and different result sets | Use librespot native autoplay via `SessionConfig.autoplay`. LMS-side is a fallback only |
| Using `GET /recommendations` for Connect-DSTM | Deprecated since Nov 27, 2024 — returns 404 in dev mode | librespot native autoplay uses `context-resolve/v1/autoplay` which is still active |
| Polling `GET /me/player` to detect queue end | Race-prone, wastes API quota, 30s rate window pressure | React to `Stopped` event from binary (already wired in Connect.pm) |

### Dependencies for Connect-DSTM

```
Existing Connect daemon (DaemonManager, Daemon.pm) — already built
    └── requires: librespot binary with --autoplay flag support (new)
    └── requires: session_config.autoplay set in run_connect() (new)

LMS-side fallback path (optional):
    Existing DontStopTheMusic.pm ──enhances──> Connect-DSTM
    Requires: detect Connect Stopped with empty queue (via connect event)
    Conflicts: Browse-DSTM fires simultaneously if both are active — guard needed
```

---

## Feature Area 2: Multi-Arch Binaries

### What Exists and What's Missing

The GitHub Actions workflow (`build-librespot.yml`) already defines all 5 Linux targets:
- x86_64-linux — BUILT (binary present in `Plugins/SpotOn/Bin/x86_64-linux/spoton`)
- aarch64-linux — directory with `.gitkeep` only (not built)
- armhf-linux — directory with `.gitkeep` only (not built)
- arm-linux — directory with `.gitkeep` only (not built)
- i386-linux — directory with `.gitkeep` only (not built)

Missing entirely:
- macOS x86_64 (`darwin-x86_64`) — no directory, not in workflow
- macOS aarch64 (`darwin-aarch64`) — no directory, not in workflow
- Windows x86_64 (`MSWin32-x86_64`) — no directory, not in workflow

The current `Helper.pm` binary search only handles Linux and `x86_64` arch explicitly.
macOS and Windows paths are handled generically via `Slim::Utils::Misc::findbin()` but with
no platform-specific binary name mapping. The Spotty-Plugin uses directory names like
`darwin-thread-multi-2level` matching Perl's `$Config{archname}`.

### Table Stakes (for Multi-Arch)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| aarch64 binary (Raspberry Pi 4, NAS devices) | Most common LMS-on-ARM platform; Pi 4 is the dominant install target | LOW | CI workflow already configured; `cross build --target aarch64-unknown-linux-musl`. Trigger workflow |
| armhf binary (Pi 2/3, 32-bit ARM) | armhf-linux is the fallback for aarch64 per existing Helper.pm logic | LOW | `armv7-unknown-linux-musleabihf`. Already in workflow matrix |
| arm-linux (ARMv5/ARMv6, older NAS, Synology) | Spotty supports it; users on older hardware expect same | LOW | `arm-unknown-linux-musleabi`. Already in workflow matrix |
| i386 binary (32-bit x86, old NAS, VM) | Spotty supports it; some LMS installations on legacy x86 hardware | LOW | `i686-unknown-linux-musl`. Already in workflow matrix |
| Helper.pm correctly selects arch-appropriate binary | Without correct selection, users get "no helper found" error | LOW | Extend `_findBin` to map more arch patterns; test on real hosts |

### Differentiators (for Multi-Arch)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| macOS binary (x86_64 + aarch64) | LMS runs natively on Mac; plugin useless without binary | MEDIUM | Requires macOS runner in CI (`runs-on: macos-latest`); `cargo build` native (no cross needed). Different binary suffix convention |
| Windows binary (x86_64) | LMS runs on Windows; small but non-zero user segment | MEDIUM | `runs-on: windows-latest`; `x86_64-pc-windows-msvc`; binary is `spoton.exe` |
| `spoton-custom` override path | Power users who build their own binary or want a non-standard path | LOW | Already in `_findBin` candidates list as first entry; just document it |
| Checksum file in release | Security-conscious NAS users verify downloads | LOW | Already in `build-librespot.yml` release job (SHA256SUMS.txt) |

### Anti-Features (for Multi-Arch)

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Shipping binaries in the git repo | Large binary blobs bloat clone time and LMS plugin download; Spotty does this but it was a historical decision | Binaries as GitHub Release assets, downloaded by plugin installer or manually |
| ARMv5 musl static binary for very old Synology (DSM 4.x) | Marginal platform; `ring` crate (TLS) has linker issues with ARMv5 musl | Document as unsupported; cross-compile for ARMv5 is notoriously fragile with Rust TLS deps |
| Universal macOS binary (lipo fat binary) | Extra step, extra CI complexity; LMS runs on either arch natively | Separate x86_64 and aarch64 macOS binaries in separate directories |

### Dependencies for Multi-Arch

```
Existing GitHub Actions workflow (build-librespot.yml)
    └── Add: macOS runner job (separate from Linux matrix)
    └── Add: Windows runner job
    └── Add: darwin-x86_64/ and darwin-aarch64/ and MSWin32-x86_64/ Bin dirs

Helper.pm _findBin
    └── Add: darwin arch detection (ISMAC + arch → select binary dir)
    └── Add: Windows detection (ISWINDOWS → select MSWin32-x86_64 binary)
    └── Note: LMS uses Perl's $Config{archname} which is e.g. "darwin-thread-multi-2level"
             or "MSWin32-x86-multi-thread" — map these to our dir names
```

---

## Feature Area 3: Code Language Cleanup (DE→EN)

### Scope of the Problem

German-language content found in codebase (as of 2026-06-03):

**Helper.pm** (5 German comments):
- Line 21: `# aarch64 kann als Fallback armhf-Binaries verwenden`
- Line 71: `# KRITISCH: 'spoton' nicht 'spotty' im Regex`
- Line 75: `# SpotOn-Erweiterung: Mindestversions-Pruefung`
- Line 115: `# Angepasster Binary-Finder um findbin() von LMS zu nutzen`
- Line 129: `# Custom-Override zuerst (LMS-10 Vorbereitung)`

**Settings.pm** (6 German comments):
- Line 70: checkbox browser behavior explanation in German
- Line 74: `# Client-ID pref speichern (D-02, T-04.4-01)`
- Lines 75-77: input validation comment in German
- Line 81: inline truncation comment in German
- Line 196: `# Client-ID und Degraded-Mode-Status fuer Template`
- Lines 307-309: `_isDegradedMode` function header in German

**Plugin.pm** (1 German comment):
- Line 345: `# nutzen _isMadeForYou — nur diese Funktion aendern reicht (RESEARCH.md Pitfall 4).`

**librespot-spoton Rust source** (not yet searched; likely contains some German inline
comments from rapid development — requires grep check on first pass of cleanup phase):

Total estimate: ~15-25 Perl comment lines, unknown Rust lines. Mechanical task.

### Table Stakes (for Code Cleanup)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| All Perl comment lines in English | Standard for an open-source project; contributors can't read German | LOW | ~25 occurrences, all in Helper.pm, Settings.pm, Plugin.pm, Connect/Daemon.pm |
| All log strings (`$log->info/warn/debug/error`) in English | Log output is the primary debugging tool; German makes it unusable for non-German speakers | LOW | ~13 German log lines, scattered; easily found with grep |
| Rust comment lines in English | Same open-source contributor expectation | LOW | Full grep of librespot-spoton/src/ needed; estimate 5-15 lines |

### Differentiators (for Code Cleanup)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Technical reference codes preserved (D-01, P-13, CON-11, etc.) | These cross-references into RESEARCH.md and REQUIREMENTS.md are valuable — replacing them with prose loses traceability | LOW | Keep all `(D-XX)`, `(P-XX)`, `(CON-XX)`, `(T-XX-YY)` codes in translated comments |
| German-origin RESEARCH.md pitfall labels unchanged | RESEARCH.md is not code; users don't read it; leave it for history | LOW | Only translate `.pm` and `.rs` source files |
| Consistent English style (imperative, present tense) | Matches LMS codebase style and Perl community norms | LOW | "Validates input" not "This validates" or "We validate" |

### Anti-Features (for Code Cleanup)

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Automated translation without human review | Machine-translated code comments are often awkward or miss technical nuance | Translate manually; the volume is small (~25-40 lines total) |
| Changing variable/function names | `_isDegradedMode`, `_isMadeForYou` etc. are fine in English already; renaming breaks call sites | Only translate comments and string literals, not identifiers |
| Removing comments in favor of "self-documenting code" | The German comments often contain critical context (validation rules, pitfall references) | Translate; never delete unless the comment truly adds nothing |
| Translating i18n string files (language/*.strings) | Those files are intentionally multi-language | Skip all `language/` files entirely |

### Dependencies for Code Cleanup

```
No code dependencies — purely textual changes to comments and log strings.

Risk: none (comments are not executable; log strings have no semantic effect on behavior)

Order: can be done in any phase; does not block or depend on Connect-DSTM or Multi-Arch work.
       Best done as a single atomic pass rather than mixed with feature changes.
```

---

## Feature Prioritization Matrix (v1.1)

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| aarch64/armhf/arm/i386 Linux binaries | HIGH (most LMS users on ARM NAS/Pi) | LOW (CI already configured) | P1 |
| librespot autoplay via SessionConfig | HIGH (Connect queue continues naturally) | LOW (one line change + test) | P1 |
| Helper.pm arch detection (Linux complete) | HIGH (required to use new binaries) | LOW (pattern matching) | P1 |
| Code cleanup: Perl comments/logs | MEDIUM (contributor accessibility) | LOW (mechanical, ~40 lines) | P1 |
| macOS binaries (x86_64 + aarch64) | MEDIUM (Mac LMS installs) | MEDIUM (new CI runner, binary naming) | P2 |
| Windows binary | LOW (Windows LMS rare in practice) | MEDIUM (MSVC toolchain, .exe suffix) | P2 |
| Connect-DSTM via `--autoplay` flag | HIGH (users expect queue to continue) | LOW (flag + daemon restart) | P1 |
| Per-player autoplay toggle (UI) | MEDIUM (power users) | LOW (settings pref + pass-through) | P2 |
| LMS-side fallback for Connect-DSTM | LOW (librespot native handles it) | MEDIUM (event detection logic) | P3 |
| Rust source comment cleanup | MEDIUM (Rust contributors) | LOW (grep + replace) | P2 |

**Priority key:** P1 = must have in v1.1, P2 = should have, P3 = future consideration

---

## Reference Implementation Comparison

### Connect-DSTM: How Spotty-Plugin Handles It

Herger's Spotty-Plugin does NOT implement Connect-DSTM as a distinct feature. The
`DontStopTheMusic.pm` registers a single handler regardless of mode. This means in Connect
mode, when the Spirc loop ends a context and the `recommendations` endpoint returns results,
LMS attempts to re-inject tracks into the Spirc queue — an approach that is architecturally
unsound because LMS does not control the Spirc queue in Connect mode.

The clean approach (SpotOn v1.1) is: let librespot handle it natively via `SessionConfig.autoplay`.

### Multi-Arch: How Spotty-Plugin Distributes Binaries

Spotty uses directory names matching Perl's `$Config{archname}`:
- `MSWin32-x86-multi-thread` — Windows
- `darwin-thread-multi-2level` — macOS
- `aarch64-linux`, `arm-linux`, `i386-linux` — Linux variants

SpotOn's current `Helper.pm` only handles `x86_64` explicitly; all others fall through to
`Slim::Utils::Misc::findbin()` which scans `$serverPrefs->get('binPath')`. The fix: add
explicit arch detection for each platform, mapping to our Bin directory names.

### Code Cleanup: Standard Practice

No comparable reference — this is basic open-source hygiene. The Spotty-Plugin (by a German
speaker) has similar German comments in places; no other LMS plugin has this issue.

---

## Sources

- librespot-connect 0.8.0 source (local): `~/.cargo/registry/src/.../librespot-connect-0.8.0/src/state.rs` — ConnectConfig struct, no autoplay field (HIGH)
- librespot-core 0.8.0 source (local): `~/.cargo/registry/src/.../librespot-core-0.8.0/src/config.rs` — SessionConfig.autoplay: Option<bool> (HIGH)
- librespot-connect 0.8.0 spirc.rs (local): `add_autoplay_resolving_when_required()` implementation (HIGH)
- librespot CHANGELOG: "Add support for `seek_to`, `repeat_track` and `autoplay` for `Spirc` loading" in 0.8.0 (HIGH)
- SpotOn `connect.rs` (local): `SessionConfig::default()` with no autoplay override — confirmed gap (HIGH)
- SpotOn GitHub Actions `build-librespot.yml` (local): 5 Linux targets already configured, macOS/Windows missing (HIGH)
- SpotOn `Helper.pm` (local): only x86_64 Linux explicit; macOS/Windows fallback is generic (HIGH)
- SpotOn `Plugins/SpotOn/Bin/` (local): x86_64-linux/spoton present, others `.gitkeep` only (HIGH)
- Spotty-Plugin `Helper.pm` (WebFetch): platform detection pattern using `$Config{archname}` (MEDIUM)
- Spotty-Plugin `Bin/` directory structure (WebFetch): MSWin32, darwin, aarch64, arm, i386 dirs (MEDIUM)
- German comment grep results (local): 5 in Helper.pm, 6 in Settings.pm, 1 in Plugin.pm (HIGH)

---
*Feature research for: SpotOn v1.1 — Connect-DSTM, Multi-Arch Binaries, Code Cleanup*
*Researched: 2026-06-03*
