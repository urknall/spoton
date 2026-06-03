# Architecture Research: SpotOn v1.1

**Domain:** Spotify Connect DSTM, Multi-Arch Binary Selection, Code Cleanup
**Researched:** 2026-06-03
**Confidence:** HIGH (all findings based on direct source-code inspection + official API docs)

---

## Existing Architecture (v1.0 Baseline)

The codebase has the following module topology after v1.0:

```
Plugin.pm              — Entry point, OPML menu tree, transcoding engine, orphan cleanup
Connect.pm             — LMS event subscribers + spottyconnect JSON-RPC dispatch handler
Connect/DaemonManager.pm — Daemon lifecycle, watchdog, sync-group election, PID registry
Connect/Daemon.pm      — Per-player process wrapper (Proc::Background, stream_port capture)
API/Client.pm          — Centralized HTTP client, dual-token routing, rate limiting
API/TokenManager.pm    — ZeroConf discovery, Keymaster --get-token, token refresh
ProtocolHandler.pm     — Protocol handler for spotify://, format detection (son/soc)
DontStopTheMusic.pm    — DSTM provider (Browse mode only — Connect gap is the v1.1 target)
Helper.pm              — Binary discovery, --check validation, capability parsing
Settings.pm            — LMS Settings page (HTML + AJAX endpoints)
librespot-spoton/      — Custom Rust binary (connect.rs + main.rs)
  src/connect.rs       — LMS notifier, HttpStreamSink, HTTP control server, run_connect
  src/main.rs          — CLI flag parsing, mode dispatch
Bin/x86_64-linux/spoton — Only populated target (v1.0 shipped one arch)
Bin/{aarch64,armhf,arm,i386}-linux/ — Directories exist with .gitkeep only
custom-convert.conf    — 5 transcoding pipelines: son-pcm, son-flc, son-mp3, son-ogg, soc-pcm
custom-types.conf      — soc type definition
```

### Binary Event Vocabulary (connect.rs LMS notifier)

The binary sends JSON-RPC `spottyconnect` commands to LMS:

| Command | Trigger | LMS handler in Connect.pm |
|---------|---------|--------------------------|
| `start` | TrackChanged (None→Some) | `_connectEvent` → playlist play |
| `change` | TrackChanged (Some→Some) or Playing (diff id) | `_connectEvent` → metadata update |
| `stop` | Paused or Stopped | `_connectEvent` → LMS pause |
| `resume` | Playing after was_paused=true | `_connectEvent` → LMS unpause |
| `seek` | Seeked | `_connectEvent` → startOffset adjust |
| `volume` | VolumeChanged | `_connectEvent` → mixer volume |
| `ready` | Spirc reconnected | `_connectEvent` → re-issue playlist play |

**`EndOfTrack` is currently NOT forwarded** — it falls through to `_ => {}` in `handle_player_event`. This is the architectural gap for Connect-DSTM.

---

## Feature 1: Connect-DSTM

### The Gap

`DontStopTheMusic.pm` registers with `Slim::Plugin::DontStopTheMusic::Plugin` as a Browse-mode DSTM provider. LMS calls the registered handler when the LMS playlist empties. In Connect mode, LMS has a single-track playlist (`spotify://connect-<ts>`) that never empties via LMS — Spotify controls track progression internally via Spirc. LMS's DSTM framework never fires because the Connect pseudo-URL is a perpetual "repeating stream" (`isRepeatingStream` returns 1 for connect URLs).

### Integration Path

The Connect-DSTM feature requires two cooperating pieces:

**Binary side (connect.rs):** Add `EndOfTrack` event handling in `handle_player_event`. When `EndOfTrack` fires and `current_track` is `Some`, emit a new `spottyconnect endoftrack <track_id>` notification to LMS.

```rust
PlayerEvent::EndOfTrack { track_id, .. } => {
    if let Ok(id) = track_id.to_id() {
        if current_track.is_some() {
            self.notify("endoftrack", &id, "").await;
        }
    }
}
```

**Perl side (Connect.pm `_connectEvent`):** Add an `endoftrack` command handler. When received:
1. Check whether Spotify's context still has a next track (librespot handles this transparently — if it fires EndOfTrack and then TrackChanged immediately, no intervention needed). 
2. If no `start`/`change` arrives within a configurable window (~3-5s), invoke the DSTM logic by calling `API::Client->addToQueue($accountId, $uri, $cb)` with a recommended track URI derived from search fallback.

**New API::Client method `addToQueue`:** POST to `POST /me/player/queue?uri=spotify:track:ID`. This endpoint is available in development mode, requires `user-modify-playback-state` scope. Routes via own-token (me/* family, D-05 guard in Client.pm ensures own-token is used).

### Data Flow: Connect-DSTM

```
[Spotify Context Ends]
        ↓
[librespot PlayerEvent::EndOfTrack]
        ↓
[connect.rs handle_player_event]
   → notify("endoftrack", track_id, "")
        ↓ JSON-RPC spottyconnect endoftrack
[Connect.pm _connectEvent]
   → set timer (3-5s grace for Spotify auto-advance)
        ↓ (if no start/change arrives)
[DontStopTheMusic.pm dontStopTheMusic-like logic]
   → API::Client->search() for seed tracks
   → API::Client->addToQueue() → POST /me/player/queue
        ↓
[Spotify app picks up queued track and plays it]
   → binary fires TrackChanged → start/change event → normal flow
```

### Modified Components

| Component | Change |
|-----------|--------|
| `librespot-spoton/src/connect.rs` | Add `EndOfTrack` match arm in `handle_player_event` |
| `Connect.pm` | Add `endoftrack` cmd in `_connectEvent`; add grace timer; import DSTM logic |
| `API/Client.pm` | Add `addToQueue($accountId, $uri, $cb)` method |
| Binary rebuild | Required for all 6 targets (new event) |

### Spike Risk

The critical unknown: does librespot's `EndOfTrack` fire when Spotify's queue/context is exhausted (user played a single track with no queue), or does Spirc always auto-advance to autoplay? In the `spirc.rs` source (upstream librespot), `EndOfTrack` is fired by the player, and Spirc handles it by calling `handle_next()` — which succeeds if the context has a next track, or fires Stopped if not. In the "no next track" case, both `EndOfTrack` and `Stopped` fire in sequence. This means the Perl handler must distinguish between:

- EndOfTrack followed quickly by start/change: Spotify auto-advanced, no DSTM needed
- EndOfTrack followed by stop (no start/change in ~3s): Spotify context exhausted, DSTM needed

The grace timer (3-5s) after EndOfTrack before invoking the DSTM path handles this without binary changes.

### Scope Recommendation

Implement as a spike: add EndOfTrack notification to binary, add minimal Perl handler with grace timer. Defer full DSTM search fallback to a follow-up if the spike reveals that `addToQueue` works reliably.

---

## Feature 2: Multi-Arch Binaries

### Current State

`Helper.pm::_findBin` discovers binaries using LMS's `Slim::Utils::Misc::findbin()`. LMS's findbin searches paths registered via `addFindBinPaths`. The Bin/ subdirectory structure maps to LMS's platform detection. Only x86_64-linux is populated.

### Platform → Directory Mapping (LMS Convention)

LMS plugin binary convention (derived from Spotty-Plugin reference and LMS slimserver source):

| Target | Bin/ subdirectory | LMS Platform String |
|--------|-------------------|---------------------|
| x86_64 Linux | `x86_64-linux/` | `unix` + `archname =~ x86_64` |
| i386/i686 Linux | `i386-linux/` | `unix` (32-bit) |
| aarch64 Linux (Pi 4) | `aarch64-linux/` | `unix` + `osArch =~ aarch64` |
| armv7 Linux (Pi 2/3) | `armhf-linux/` | `unix` (arm fallback path) |
| macOS x86_64 | via `ISMAC + x86_64` | `osx` |
| macOS aarch64 (Apple Silicon) | via `ISMAC + arm64` | `osx` |
| Windows x86_64 | via `ISWINDOWS` | `win` |

**Helper.pm current logic:** Only adds `armhf-linux` as fallback path for aarch64 hosts. Does not handle macOS or Windows with explicit Bin/ subdirectories. LMS's own `findbin` handles OS-specific extension (`.exe` on Windows) and path resolution.

### Architecture of Binary Selection

`Helper.pm::init()` runs once at plugin load:
- Detects aarch64 → adds `armhf-linux/` as fallback via `addFindBinPaths`
- Detects custom pref → validated first

`Helper.pm::_findBin()` tries candidates in order:
1. `spoton-custom` (user override, highest priority)
2. `spoton` (base name, LMS finds in registered paths)
3. `spoton-x86_64` (explicit x86_64 name, x86_64 unix only)

### Changes Needed for Full Multi-Arch

**Helper.pm modifications:**

Add platform branches to `init()`:

```perl
# macOS: explicit arch detection
if (main::ISMAC) {
    my $arch = $Config::Config{'archname'};
    if ($arch =~ /arm|aarch64/i) {
        Slim::Utils::Misc::addFindBinPaths(
            catdir($pluginDir, 'Bin', 'aarch64-macos')
        );
    } else {
        Slim::Utils::Misc::addFindBinPaths(
            catdir($pluginDir, 'Bin', 'x86_64-macos')
        );
    }
}
# Windows: LMS findbin adds .exe automatically
if (main::ISWINDOWS) {
    Slim::Utils::Misc::addFindBinPaths(
        catdir($pluginDir, 'Bin', 'x86_64-win')
    );
}
```

**New Bin/ directory structure:**

```
Bin/
├── x86_64-linux/spoton      (existing, statically linked musl)
├── aarch64-linux/spoton     (new: aarch64-unknown-linux-musl)
├── armhf-linux/spoton       (new: armv7-unknown-linux-musleabihf)
├── arm-linux/spoton         (new: alias/copy of armhf for older Pi)
├── i386-linux/spoton        (new: i686-unknown-linux-musl)
├── x86_64-macos/spoton      (new: x86_64-apple-darwin)
├── aarch64-macos/spoton     (new: aarch64-apple-darwin)
└── x86_64-win/spoton.exe   (new: x86_64-pc-windows-msvc)
```

### Build Toolchain

`cross` is already installed at `/home/sti/.cargo/bin/cross`. Cross-compilation targets:

```bash
# Linux musl targets (via cross — Docker-based)
cross build --release --target x86_64-unknown-linux-musl
cross build --release --target aarch64-unknown-linux-musl
cross build --release --target armv7-unknown-linux-musleabihf
cross build --release --target i686-unknown-linux-musl

# macOS (requires macOS host or osxcross)
# Note: macOS cross-compilation from Linux requires osxcross SDK
cargo build --release --target x86_64-apple-darwin     # macOS host only
cargo build --release --target aarch64-apple-darwin    # macOS host only

# Windows (via cross with mingw toolchain)
cross build --release --target x86_64-pc-windows-gnu  # or msvc with wine
```

**Important:** The current binary is dynamically linked (`ELF 64-bit LSB pie executable, dynamically linked`). Distribution requires static musl builds (`-unknown-linux-musl` targets) so binaries run on any Linux without glibc version matching. macOS and Windows have their own static linking modes.

### Modified Components

| Component | Change |
|-----------|--------|
| `Helper.pm` | Add macOS + Windows `Bin/` path registration in `init()`; add arm64 macOS detection |
| `Bin/` directory | Populate 5 new subdirectories with built binaries |
| Build scripts | New CI/Makefile for cross-compilation of all targets |

### No Changes Required

- `Daemon.pm` — already resolves binary via `Helper->get()`
- `custom-convert.conf` — platform-agnostic (uses `[spoton]` placeholder)
- `install.xml` — no binary paths listed (LMS resolves at runtime)
- `API/Client.pm`, `Connect.pm`, `Plugin.pm` — no binary path dependencies

---

## Feature 3: Code Cleanup (DE→EN)

### Scope Assessment

German text found in Perl sources (exhaustive scan):

**Plugin.pm:**
- Line 21: `# Stundlicher Orphaned-Process-Cleanup (STR-10)` → English
- Lines 342-345: German comment block for `$PERSONAL_MIX_REGEX`
- Lines 415, 1153: `# Kontext-Queueing ...` inline comments

**Helper.pm:**
- Line 21: `# aarch64 kann als Fallback armhf-Binaries verwenden`
- Line 71: `# KRITISCH: 'spoton' nicht 'spotty' im Regex`
- Line 75: `# SpotOn-Erweiterung: Mindestversions-Pruefung`
- Line 115: `# Angepasster Binary-Finder um findbin() von LMS zu nutzen`
- Lines 123, 129: `# auf 64 bit x86 zuerst ...`, `# Custom-Override zuerst ...`

**Connect/Daemon.pm:**
- Line 177: `# CR-02: Whitelist statt Blacklist ...` (in Plugin.pm orphan cleanup — confirmed via grep)

**DontStopTheMusic.pm:**
- None found (already clean English)

**Connect.pm, DaemonManager.pm, API/Client.pm, ProtocolHandler.pm, Settings.pm:**
- No German found in comments (already clean)

**connect.rs (Rust binary):**
- No German found; Rust source is already English throughout

### Implementation Pattern

Pure text replacement — no logic changes. Each file can be processed independently. Verification: grep for common German function words (`kann`, `als`, `Fallback`, `nutzen`, `statt`, `zuerst`, `KRITISCH`, `Angepasst`, `Stundlicher`, `Kontext`, `Muster`, `Aufrufstellen`, `aendern`, `Verwendet`, `Pruefung`, `Erweiterung`, `Vorbereitung`) post-cleanup.

### Modified Components

| Component | German instances | Effort |
|-----------|-----------------|--------|
| `Plugin.pm` | 4 locations | Trivial |
| `Helper.pm` | 6 locations | Trivial |
| `Connect/Daemon.pm` | 1 location (line 177 via Plugin.pm scan) | Trivial |

---

## Component Interaction Map

```
┌─────────────────────────────────────────────────────────┐
│                      LMS Process                         │
│                                                         │
│  Plugin.pm ──────── initPlugin ────────────────────┐    │
│      │                                              │    │
│      ├── ProtocolHandler.pm (spotify:// handler)    │    │
│      ├── DontStopTheMusic.pm (Browse DSTM)         │    │
│      ├── Settings.pm                               │    │
│      └── Helper.pm ─── Binary discovery ────┐      │    │
│                                              │      │    │
│  Connect.pm ─── _connectEvent ──────────────┤      │    │
│      │  (spottyconnect dispatch)             │      │    │
│      │  ┌── [v1.1 NEW] endoftrack handler   │      │    │
│      │  │   → 3s grace timer                │      │    │
│      │  │   → API::Client->addToQueue()     │      │    │
│      │  └────────────────────────────────── │      │    │
│      └── DaemonManager.pm ─── Daemon.pm ────┘      │    │
│                │                                    │    │
│  API/Client.pm ─────────────────────────────────── │    │
│      │  (dual-token, rate-limit, all Spotify calls) │    │
│      └── [v1.1 NEW] addToQueue()                   │    │
│                                                     │    │
│  API/TokenManager.pm (ZeroConf + Keymaster tokens)  │    │
└─────────────────────────────────────────────────────────┘
         │ JSON-RPC spottyconnect             │ HTTP /control/*
         │                                   │
┌────────────────────────────────────────────────────────┐
│              librespot-spoton binary                    │
│                                                         │
│  connect.rs::run_connect()                              │
│      ├── LMS::handle_player_event()                     │
│      │   ├── TrackChanged  → "start"/"change"           │
│      │   ├── Playing       → "resume" / position sync   │
│      │   ├── Paused/Stopped → "stop"                    │
│      │   ├── VolumeChanged → "volume"                   │
│      │   ├── Seeked        → "seek"                     │
│      │   └── [v1.1 NEW] EndOfTrack → "endoftrack"       │
│      └── http_stream_server()                           │
│          ├── GET /stream   → S16LE PCM relay            │
│          └── POST /control/* → Spirc commands           │
└────────────────────────────────────────────────────────┘
```

---

## Data Flows

### Connect-DSTM Flow (New)

```
Spotify context ends
    ↓
PlayerEvent::EndOfTrack fires in librespot-spoton
    ↓
LMS::handle_player_event → notify("endoftrack", track_id, "")
    ↓ (JSON-RPC spottyconnect endoftrack)
Connect.pm::_connectEvent cmd=endoftrack
    ↓
Set grace timer (3-5s) via Slim::Utils::Timers
    ↓ (if "start"/"change" arrives within window → cancel timer, no DSTM)
    ↓ (if timer fires → context really ended)
Invoke _connectDSTM($client)
    ↓
API::Client->search(seed=lastTrackId) → get track suggestions
    ↓
API::Client->addToQueue(accountId, uri) → POST /me/player/queue?uri=spotify:track:ID
    ↓
Spotify plays the queued track → binary fires TrackChanged → normal connect flow resumes
```

### Multi-Arch Binary Selection Flow (Enhanced)

```
Plugin.pm::initPlugin()
    ↓
Helper::init()
    ├── ISWINDOWS? → addFindBinPaths(Bin/x86_64-win/)
    ├── ISMAC + arm? → addFindBinPaths(Bin/aarch64-macos/)
    ├── ISMAC + x86? → addFindBinPaths(Bin/x86_64-macos/)
    └── unix + aarch64? → addFindBinPaths(Bin/armhf-linux/) [existing]
    ↓
Helper::get() → _findBin()
    ├── try "spoton-custom" (user override)
    ├── try "spoton" (LMS findbin resolves via registered paths)
    └── try "spoton-x86_64" (explicit fallback, unix x86_64 only)
    ↓
helperCheck(candidate) → binary --check → parse "ok spoton vX.Y.Z\n{json}"
    ↓
Helper::getCapability('passthrough') → controls son-ogg availability
```

### Code Cleanup Flow

```
Identify all German text (grep) per file
    ↓
Replace in-place (no logic changes)
    ↓
Verify: grep for German function words returns zero matches
    ↓
Run existing tests / LMS startup to confirm no regressions
```

---

## Build Order (Suggested Phase Sequence)

**Rationale:** Connect-DSTM has a spike risk (binary event behavior needs validation). Code cleanup is pure text and has zero risk. Multi-Arch needs the final binary but can be built independently.

### Recommended Order

**Phase 1: Code Cleanup** (lowest risk, pure text, validates CI pipeline)
- Modify Plugin.pm, Helper.pm, Connect/Daemon.pm
- Zero functional impact; easiest verification
- Can run in parallel with binary work

**Phase 2: Multi-Arch Binaries** (independent of Connect-DSTM, high value for users)
- Modify Helper.pm (arch detection for macOS/Windows)
- Build binaries for all 6 targets using cross
- Deploy to Bin/ directories and verify --check on each
- IMPORTANT: all 6 binaries must include the EndOfTrack notification if Phase 3 precedes Phase 2; sequence accordingly

**Phase 3: Connect-DSTM Spike** (highest risk, binary + Perl changes)
- Binary: add EndOfTrack arm in connect.rs, rebuild all 6 targets
- Perl: add addToQueue to Client.pm, add endoftrack handler in Connect.pm
- UAT: verify EndOfTrack fires correctly, grace timer cancels on auto-advance, addToQueue enqueues track in Spotify app

**If spike fails:** Connect-DSTM is scoped out, Multi-Arch + Cleanup ship as v1.1.

**If spike succeeds:** All three features ship together.

### Dependency Table

| Feature | Depends On | Blocks |
|---------|-----------|--------|
| Code Cleanup | Nothing | Nothing |
| Multi-Arch Binaries | Code Cleanup (clean source in builds) | Nothing |
| Connect-DSTM | Code Cleanup, new binary build | Nothing |

---

## Architectural Anti-Patterns to Avoid

### Anti-Pattern 1: Using LMS DSTM Framework for Connect-DSTM

**What people might try:** Register a second DSTM handler that checks `isSpotifyConnect` and pushes tracks.
**Why wrong:** LMS DSTM fires when the LMS playlist empties. Connect mode uses `isRepeatingStream=1` — LMS never considers the playlist empty. The DSTM framework will never fire in Connect mode regardless of handler registration.
**Do this instead:** Detect EndOfTrack at the binary level, add dedicated `endoftrack` spottyconnect command, handle in Connect.pm independently.

### Anti-Pattern 2: Parallelizing Binary Builds

**What people might try:** Build all 6 targets simultaneously.
**Why wrong:** `cross` uses Docker; multiple simultaneous cross builds can exhaust memory on typical development hardware (each musl target pulls ~1GB Docker image). Build sequentially or with 2-parallel max.
**Do this instead:** Sequential builds with `cross build --release --target <triple>` one at a time. Cache Docker images between runs.

### Anti-Pattern 3: Dynamic Binary in x86_64-linux/

**What people might try:** Use the development binary (current dynamically linked x86_64).
**Why wrong:** Current binary is dynamically linked against glibc. Users on older LMS appliances or different distros may have incompatible glibc versions.
**Do this instead:** Build all Linux targets with `*-unknown-linux-musl` for static linking. Replace the existing x86_64-linux/spoton with the musl build.

### Anti-Pattern 4: Clearing `current_track` on EndOfTrack

**What people might try:** In the binary, set `current_track = None` when EndOfTrack fires.
**Why wrong:** If Spotify auto-advances, a TrackChanged fires immediately after EndOfTrack. If `current_track` is cleared, the TrackChanged produces a `start` instead of `change`, causing Connect.pm to re-issue `playlist play` and interrupt streaming.
**Do this instead:** Keep `current_track` unchanged in EndOfTrack handler. Only notify `endoftrack`; let TrackChanged handle state transitions normally.

### Anti-Pattern 5: `addToQueue` Without Own-Token

**What people might try:** Use bundled token for `POST /me/player/queue`.
**Why wrong:** `/me/player/queue` is in the me/* family. The me/* guard in `_request` (`$_meFamilyRegex`) already forces own-token for all me/* endpoints. No change needed in addToQueue — the guard handles it.
**Do this instead:** Use the standard `_request('post', 'me/player/queue', ...)` pattern; me/* guard applies automatically.

---

## Integration Points Summary

| Integration | Direction | Mechanism | Notes |
|-------------|-----------|-----------|-------|
| EndOfTrack → LMS | Binary → Perl | JSON-RPC spottyconnect | New command: `endoftrack` |
| DSTM → Spotify | Perl → Spotify | POST /me/player/queue | New Client.pm method |
| Binary discovery | Perl → Filesystem | Helper.pm::_findBin + addFindBinPaths | Add macOS/Win paths |
| Arch detection | Perl → LMS API | Slim::Utils::OSDetect, main::ISMAC/ISWINDOWS | In Helper::init() |
| German → English | In-file | Direct text replacement | No cross-module impact |

---

## Confidence Assessment

| Area | Confidence | Reason |
|------|-----------|--------|
| Connect-DSTM architecture | MEDIUM | EndOfTrack behavior in upstream librespot confirmed; grace timer approach sound; addToQueue API confirmed available in dev mode. Spike needed to validate librespot EndOfTrack fires in correct scenarios. |
| Multi-Arch binary selection | HIGH | Helper.pm code read directly; LMS OSDetect API confirmed; cross toolchain installed. |
| Code cleanup scope | HIGH | All files scanned directly; German text inventoried precisely. |
| `addToQueue` API availability | HIGH | Endpoint confirmed in CLAUDE.md endpoint status table as "Working in Development Mode". |

---

## Open Questions for Phase Planning

1. **Connect-DSTM spike gate:** Does `PlayerEvent::EndOfTrack` fire when Spotify's queue is exhausted (single track played, no queue), or does librespot always suppress it and fire `Stopped`? This determines whether the endoftrack notification is reliably triggerable.

2. **macOS binary distribution:** macOS cross-compilation from Linux requires `osxcross` or a macOS CI runner. Is a macOS build environment available? If not, macOS binaries must be built separately and committed to the repo.

3. **Windows scope:** Is Windows binary distribution in-scope for v1.1? No Windows LMS users have been reported in the test environment. Deferring Windows to v1.2 is a reasonable option.

4. **Static x86_64 regression:** Replacing the dynamically-linked x86_64 binary with a musl-static build should be regression-free, but requires UAT on the primary squeezelite test environment.

5. **Grace timer duration:** 3-5s window for EndOfTrack grace period — needs empirical calibration based on how quickly Spotify auto-advances in practice. Too short risks false DSTM triggers; too long means slow response when context genuinely ends.

---

*Architecture research for: SpotOn v1.1 — Connect-DSTM, Multi-Arch, Code Cleanup*
*Researched: 2026-06-03*
