# Phase 10: Connect-DSTM - Context

**Gathered:** 2026-06-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Enable auto-play continuation in Spotify Connect mode. When the Spotify queue runs out during a Connect session, playback continues automatically with related tracks — matching the behavior already present in Browse mode via LMS's DontStopTheMusic framework.

**Key insight (from Spotty-NG analysis):** Connect-DSTM is NOT a plugin-side feature — it is a **Spirc-native capability** built into librespot 0.8. Spirc's `add_autoplay_resolving_when_required()` asks the Spotify server for an autoplay context when the queue is exhausted. The binary continues streaming without any API calls, EndOfTrack event handling, grace timers, or queue injection from the Perl side.

**What Phase 10 actually delivers:** Wire the existing Spirc autoplay capability through to the user via a `--autoplay on/off` CLI flag, per-player Settings toggle, and bidirectional sync with the LMS DSTM provider.

</domain>

<decisions>
## Implementation Decisions

### Architecture: Spirc-Native Autoplay (NOT API Queue Injection)
- **D-01:** Connect-DSTM uses Spirc's built-in autoplay context resolution (`add_autoplay_resolving_when_required()` in librespot-connect 0.8 spirc.rs). NO EndOfTrack event handling, NO grace timer, NO `POST /me/player/queue` injection, NO new DSTM code on the Perl side.
- **D-02:** `SessionConfig.autoplay` controls the behavior. `None` = use Spotify user setting; `Some(true)` = force on; `Some(false)` = force off. Default in SessionConfig is `None`.
- **D-03:** This mirrors the Spotty-NG approach exactly: binary receives `--autoplay on/off` flag → sets `session_config.autoplay = Some(true/false)`.

### Binary Changes
- **D-04:** Add `--autoplay on/off` CLI flag to `librespot-spoton/src/main.rs`. Parse in the argument loop, pass to `run_connect()`.
- **D-05:** In `connect.rs::run_connect()`, set `session_config.autoplay = Some(true/false)` based on the flag value.
- **D-06:** Add `"autoplay": true` to the `--check` JSON capability manifest so Helper.pm can detect the feature.
- **D-07:** Binary rebuild required for all 8 platform targets (Phase 8 Bin/ directories).

### Per-Player Autoplay Toggle
- **D-08:** New per-player pref `enableAutoplay`, default `1` (on). Controls both Connect-Autoplay AND Browse-DSTM.
- **D-09:** DaemonManager passes `--autoplay on/off` to the Connect daemon based on `$prefs->client($client)->get('enableAutoplay')`.
- **D-10:** Settings UI shows the toggle only when `Helper->getCapability('autoplay')` is true.

### Bidirectional DSTM Sync
- **D-11:** When user turns Autoplay OFF via SpotOn toggle → LMS DSTM dropdown is programmatically set to "Off" for that player. Connect daemon gets `--autoplay off`.
- **D-12:** When user turns Autoplay ON via SpotOn toggle → LMS DSTM dropdown is set to "SpotOn Empfehlungen". Connect daemon gets `--autoplay on`.
- **D-13:** When user manually selects "SpotOn Empfehlungen" in the LMS DSTM dropdown → SpotOn Autoplay toggle syncs to ON.
- **D-14:** When user manually sets LMS DSTM dropdown to "Off" or another provider → SpotOn Autoplay toggle syncs to OFF.
- **D-15:** The LMS DSTM provider ("SpotOn Empfehlungen") stays registered regardless of toggle state — only the dropdown selection changes.

### Claude's Discretion
- DSTM sync implementation details (pref change callbacks, timing of sync)
- Settings UI layout for the new toggle (placement relative to existing Connect toggle)
- Whether `enableAutoplay` needs a daemon restart or can be applied live
- i18n string keys for the toggle label

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/REQUIREMENTS.md` — DSTM-01 through DSTM-06 requirement definitions (NOTE: requirements were written assuming API injection; actual implementation is simpler via Spirc autoplay)
- `CLAUDE.md` §librespot — CLI flags, audio backends, SessionConfig
- `.planning/PROJECT.md` — P-40: LMS DSTM framework never fires in Connect mode

### Binary Source (modification targets)
- `librespot-spoton/src/main.rs` — CLI flag parsing, `--check` capability manifest
- `librespot-spoton/src/connect.rs` — `run_connect()` function, SessionConfig setup (line 867)

### Plugin Source (modification targets)
- `Plugins/SpotOn/Connect/DaemonManager.pm` — daemon startup args, where `--autoplay` flag is passed
- `Plugins/SpotOn/Settings.pm` — per-player settings page
- `Plugins/SpotOn/Plugin.pm` — pref initialization (`$prefs->init`), DSTM provider registration
- `Plugins/SpotOn/DontStopTheMusic.pm` — Browse-DSTM provider (no changes, but verify DSTM-06)
- `Plugins/SpotOn/Helper.pm` — `getCapability()` for feature detection

### Spotty-NG Reference (proven pattern)
- `/home/sti/spotty-ng/Spotty-Plugin/Connect/Daemon.pm` lines 102-103 — `--autoplay on/off` flag passing
- `/home/sti/spotty-ng/Spotty-Plugin/Settings/Player.pm` lines 27, 38 — `enableAutoplay` pref + capability check
- `/home/sti/spotty-ng/librespot/src/spotty.rs` line 70 — `"autoplay": true` in `--check`

### librespot 0.8 Library (reference, do not modify)
- `~/.cargo/registry/src/*/librespot-core-0.8*/src/config.rs` line 31 — `SessionConfig.autoplay: Option<bool>`
- `~/.cargo/registry/src/*/librespot-connect-0.8*/src/spirc.rs` — `add_autoplay_resolving_when_required()`, autoplay context resolution

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `SessionConfig::default()` already sets `autoplay: None` — only needs override when user sets the pref
- `Helper->getCapability('key')` — existing capability check pattern, add `'autoplay'` to `--check` JSON
- `$prefs->init({ enableSpotifyConnect => 1 })` — per-player pref pattern to follow for `enableAutoplay`
- `DaemonManager.pm` daemon arg construction — add `--autoplay` to the existing flag list
- `Slim::Plugin::DontStopTheMusic::Plugin->registerHandler()` — existing DSTM registration in Plugin.pm

### Established Patterns
- Per-player prefs: `$prefs->client($client)->get('enableAutoplay')` — same pattern as `enableSpotifyConnect`, `streamFormat`, `bitrateOverride`
- Capability gating: Settings.pm checks `Helper->getCapability()` before showing UI elements
- Daemon restart on pref change: DaemonManager restarts the daemon when relevant prefs change — autoplay toggle should trigger this

### Integration Points
- `Plugin.pm::initPlugin()` — add `enableAutoplay => 1` to `$prefs->init({})`
- `DaemonManager.pm` daemon start — append `--autoplay on/off` based on pref
- `Settings.pm` — add toggle to player settings page
- `main.rs` CLI parser — add `--autoplay` flag
- `connect.rs::run_connect()` — accept autoplay param, set `session_config.autoplay`

</code_context>

<specifics>
## Specific Ideas

- Follow Spotty-NG's `Settings/Player.pm` pattern exactly for the UI toggle
- Binary `--check` capability name must be `"autoplay"` (matches Spotty-NG convention and Helper.pm getCapability key)
- The `--autoplay` flag uses `on`/`off` string values (not `true`/`false`), matching Spotty-NG convention

</specifics>

<deferred>
## Deferred Ideas

- **DSTM-F01 (v1.2+):** LMS-side DSTM fallback if Spirc-native autoplay fails — would require EndOfTrack event path. Only needed if Spirc autoplay proves unreliable in production.
- **Autoplay context customization:** Letting users influence what Spotify's autoplay picks (genre preferences, etc.) — not possible via Spirc, would require API-level intervention.

</deferred>

---

*Phase: 10-Connect-DSTM*
*Context gathered: 2026-06-04*
