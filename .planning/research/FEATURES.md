# Feature Landscape: SpotOn v1.3 — Polish & Publish

**Domain:** LMS Spotify plugin — incremental UX polish + publication preparation
**Researched:** 2026-06-06
**Confidence:** HIGH (based on live codebase, Spotty-Plugin reference implementation, Spotify official docs)

---

## Context: What v1.1 Already Ships

v1.1 is complete. v1.3 adds polish and prepares for wider distribution. None of these features
require architectural changes — they are targeted additions to existing subsystems.

| Area | v1.1 State | v1.3 Goal |
|------|-----------|-----------|
| Credentials | Shared cache dir per account, same dir for Browse and Connect daemons | Separate cache dir for Connect daemon per player to prevent cross-account contamination |
| Like Button | Liked Songs browsable (read-only); no write-back | Add Like/Unlike track from now-playing via `PUT /me/tracks` |
| Connect Volume | Grace period + debounce suppress echoes but initial volume is hardcoded 50% | Fix discrepancy between Connect volume level and Browse/LMS volume level |
| Format dropdown | 5 modes (Auto/OGG/PCM/FLAC/MP3) tested on squeezelite | Verify Auto correctly selects FLAC for B&O/Chromecast (no OGG support) |
| Client-ID | Bundled ncspot Extended Quota ID used for Browse endpoints | Prepare own Spotify app registration for future Extended Quota application |
| Repo | Private custom repo.xml only | Submit to LMS Community plugin repository (include.json PR) |
| macOS | No macOS binary | Universal Binary for Intel + Apple Silicon via CI |
| Community | Zero GitHub infrastructure | Issue Templates, Contributing.md, CI for test runs |

---

## Feature Area 1: Connect Credential Isolation

### The Problem (from Backlog item #1)

The Connect daemon writes credentials to the per-account cache dir (`spoton/<accountId>/`).
This is the same directory used by Browse/API to store the Keymaster token and ZeroConf
credentials for that account. When a different Spotify user connects via the Spotify app
to a player running SpotOn Connect, librespot overwrites `credentials.json` in that dir
with the new user's credentials. The next Browse API call authenticates as the wrong user —
"Browse zeigt fremde Inhalte."

### How Other Implementations Handle It

The standard pattern across Spotify Connect implementations (Spotty-NG, Raspotify, ha-spotifyplus):

- Separate cache directory per daemon instance
- Browse tokens and Connect credentials never share a directory
- Pattern: `spoton/connect-{mac-without-colons}/` for Connect, `spoton/{accountId}/` for Browse

Spotty-NG used a similar separation: its helper binary uses `{cachedir}/spotty/{mac}/` for
Connect credentials, separate from the main account credential store.

The ZeroConf discovery flow intentionally allows any Spotify user to take over a Connect
receiver — this is expected Spotify Connect behavior. What must be isolated is the effect
on the Browse API session.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Connect daemon uses its own cache dir `spoton/connect-{mac}/` | Without isolation, switching Spotify accounts via Connect corrupts Browse session | LOW | One-line change in Daemon.pm `start()` — `catdir($cachedir, 'spoton', 'connect-' . $self->id)` |
| Dir created on daemon start if absent | Binary will fail to start if dir missing | LOW | `mkpath()` already called; new path is consistent |
| Browse cache dir `spoton/{accountId}/` unchanged | Browse session remains stable across Connect handoffs | LOW | No change needed — Browse path already isolated by accountId |
| Old shared cache migration (if credentials exist) | Users upgrading from v1.1 may have existing credentials in wrong location | LOW | On start: if `spoton/connect-{mac}/credentials.json` absent AND `spoton/{accountId}/credentials.json` present, copy credentials to new dir — ZeroConf will re-authenticate on next Connect session anyway, so migration is optional |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Per-player Connect log file already in `spoton/{mac}-connect.log` | Consistent with credential isolation pattern — debug logs are per-daemon | LOW | Already implemented — no change needed |
| Credential cleanup on daemon stop (optional) | Remove `spoton/connect-{mac}/` on full stop to prevent stale credential buildup | LOW | Deliberate NON-feature: ZeroConf re-auth is cheap, and credential removal causes unnecessary re-auth delays. Keep credentials across restarts. |

### Anti-Features

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Removing ALL credentials on Connect session end | Causes re-auth delay on every reconnect; LMS users expect instant re-connect | Keep credentials; isolation is by directory, not deletion |
| Requiring user to manually configure cache dirs | Users don't know or care about cache dirs | Automatic — hardcoded convention `spoton/connect-{mac}/` |
| Shared single cache dir for all players | If multi-player household, Player A's Connect session invalidates Player B's Browse | Per-player isolation by MAC |

### Dependencies

```
Daemon.pm start() — change cacheDir construction
    └── single line change: 'spoton', 'connect-' . $self->id  (instead of 'spoton', $activeAccountId)
    └── mkpath() call at start already covers new dir creation

Helper.pm / Plugin.pm — no changes needed (Browse path stays per-accountId)
```

---

## Feature Area 2: Like Button (Track Favoriting)

### How Comparable LMS Plugins Handle It

Research on Spotty-Plugin (Herger) and Qobuz plugin:

**Spotty-Plugin:** Does NOT implement track liking. It implements:
- `addAlbumToLibrary()` → `PUT /me/albums` (album-level only)
- `followArtist()` → `PUT /me/following` (artist-level)
- No `PUT /me/tracks` — track-level liking is absent from Spotty's OPML.

**Qobuz plugin:** Implements favorites via a conditional menu item pattern:
```perl
push @$items, {
    name => cstring($client, $isFavorite ? 'PLUGIN_QOBUZ_REMOVE_FAVORITE_ARTIST'
                                         : 'PLUGIN_QOBUZ_ADD_FAVORITE_ARTIST', $name),
    url  => $isFavorite ? \&deleteFromFavorites : \&addToFavorites,
    image => 'html/images/favorites.png',
    passthrough => [{ artist_ids => $artist->{id} }],
    nextWindow => 'parent'
};
```
The pattern: check current state, show toggle item, `nextWindow => 'parent'` returns to parent
menu after action, `showBriefly => 1` confirms success inline.

**Spotty's album-add pattern (OPML.pm):**
```perl
push @$items, {
    name => cstring($client, 'PLUGIN_SPOTTY_ADD_ALBUM_TO_LIBRARY'),
    url  => \&addAlbumToLibrary,
    passthrough => [{ id => $album->{id} }],
    nextWindow => 'parent'
};
sub addAlbumToLibrary {
    my ($client, $cb, $params, $args) = @_;
    Plugins::Spotty::Plugin->getAPIHandler($client)->addAlbumToLibrary(sub {
        $cb->({ items => [{ name => cstring($client, 'PLUGIN_SPOTTY_MUSIC_ADDED'), showBriefly => 1 }] });
    }, $params->{ids} || $args->{ids});
}
```

This is the exact pattern to follow for track liking in SpotOn.

### Spotify API — Track Like (Feb 2026 context)

The Feb 2026 API changes introduced a unified library endpoint:
- **New unified:** `PUT /me/library` with `scope: user-library-modify` — saves tracks, albums, shows
- **Legacy:** `PUT /me/tracks` with `user-library-modify` scope — still works, not deprecated
- **Check:** `GET /me/library/contains` or `GET /me/tracks/contains` — both work
- **Remove:** `DELETE /me/tracks` or `DELETE /me/library` — both work

For simplicity, use `PUT /me/tracks` (track-level, well-understood, works in dev mode).

The `user-library-modify` scope must be added to the OAuth flow. Current SpotOn scopes include
`user-library-read` but not `user-library-modify`.

### Where to Surface the Like Button

OPML UI paradigm means no persistent toolbar. The Like action appears as a menu item in:
1. **Track context menu** — when drilling into a track item (the ">" action opens a submenu)
2. **Now-Playing info page** — not a standard LMS pattern for OPMLBased plugins

The most natural placement is the track-level submenu, similar to how Spotty surfaces album-add.
The check for current liked status requires an additional API call (`GET /me/tracks/contains`),
which adds latency — this can be omitted for v1.3 (add-only, no toggle) to keep it simple.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| "Add to Liked Songs" menu item on track context | Core Spotify UX; users expect to save tracks they hear | MEDIUM | Requires: new scope in OAuth, new API::Client method, new OPML handler in Plugin.pm |
| `user-library-modify` OAuth scope | API call rejected without it; ZeroConf re-auth needed on scope change | LOW | Add to scope string in TokenManager.pm; users will re-auth once |
| Confirmation feedback via `showBriefly` | Without feedback, users don't know the action succeeded | LOW | Return `{ showBriefly => 1, name => 'Added to Liked Songs' }` in callback |
| `PUT /me/tracks` call in API::Client.pm | New method `saveTrack($accountId, $trackId, $cb)` | LOW | Follow existing pattern: `_request('put', 'me/tracks', { ids => [$trackId] }, $cb)` |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Toggle (Like/Unlike) based on current state | Shows "Unlike" if already liked — prevents duplicate saves | MEDIUM | Requires `GET /me/tracks/contains` call before rendering menu item. Adds 1 extra API call per track view. Worth it for v1.3 |
| Like from Connect now-playing | Like while listening via Spotify app — surfaces in Browse | HIGH | Connect provides track ID via `start` event; could expose "Like current track" via CLI command — complex, defer |
| Like album / Follow artist | Broader library management | MEDIUM | Add as separate items in album/artist submenu; follow Spotty's pattern |

### Anti-Features

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Heart/star icon overlay on track items in list | OPML has no overlay/badge support; items are text lists | Text menu item only |
| Automatic like on play | Users don't expect LMS to like every played track | Explicit menu item only |
| Like via new unified `PUT /me/library` endpoint | Adds complexity (body format changed); `PUT /me/tracks` with query param is simpler and still works | Keep `PUT /me/tracks?ids=` pattern |
| Toggle-on-first-render without caching | Each browse re-renders the menu → 1 `contains` call per view → burns API quota | Cache the `contains` result for 60s |

### Dependencies

```
API::Client.pm
    └── new: saveTrack($accountId, $trackId, $cb)      → PUT /me/tracks?ids={id}
    └── new: removeTrack($accountId, $trackId, $cb)    → DELETE /me/tracks?ids={id}
    └── new: isTrackSaved($accountId, $trackId, $cb)   → GET /me/tracks/contains?ids={id}

API::TokenManager.pm
    └── Add 'user-library-modify' to scope string
    └── Scope change triggers re-auth for existing users (ZeroConf re-authentication required)

Plugin.pm (Browse OPML)
    └── Track submenu: add "Add to Liked Songs" / "Remove from Liked Songs" conditional item
    └── Handler sub: _likeTrack($client, $cb, $params, $args)
    └── Handler sub: _unlikeTrack($client, $cb, $params, $args)

i18n strings (language/*.strings)
    └── PLUGIN_SPOTON_LIKED, PLUGIN_SPOTON_UNLIKE, PLUGIN_SPOTON_ADDED_TO_LIKED_SONGS
    └── 11 languages; EN only for v1.3, others as empty strings or EN fallback
```

---

## Feature Area 3: Connect Volume Discrepancy Fix

### The Problem (from Backlog item #4)

"Bei gleichem %-Setting ist Connect deutlich lauter als Browse."

Root cause analysis from codebase:
- **Browse mode:** Audio decoded by librespot to S16LE PCM → piped through LMS transcoding
  pipeline → squeezelite applies LMS volume curve (hardware or software attenuation).
  Volume knob at 50% = 50% of squeezelite's output amplitude.
- **Connect mode (OGG/DirectStream):** Audio decoded by librespot's Spirc loop → HTTP stream
  directly to squeezelite bypassing LMS transcoding. Volume in librespot is controlled by
  Spirc which stores its own volume (0–65535 range, logarithmic by SoftMixer). When Connect
  starts, `initial_volume: u16::MAX / 2` = 32767 ≈ 50% of librespot's range. But librespot's
  `SoftMixer` applies the logarithmic curve: 50% SoftMixer volume ≠ 50% LMS volume.
- **The discrepancy:** `initial_volume: u16::MAX / 2` hardcoded in `connect.rs` line 970 means
  Connect always starts at 50% librespot-volume regardless of LMS player volume setting.
  When the Spotify app connects and pushes its stored volume (e.g. 80%), the suppress mechanism
  (`suppress_next_volume` AtomicBool) drops the first VolumeChanged, but subsequent app volume
  changes still come through at librespot's scale. The CON-11 grace period and debounce work
  correctly for echo suppression but don't address the initial mismatch.

### Possible Approaches

1. **Read LMS player volume at daemon start → set `initial_volume` accordingly**
   - Read `$client->volume` in Daemon.pm start() before spawning
   - Convert LMS 0–100 to librespot 0–65535: `int($lms_vol * 65535 / 100)`
   - Pass as `--initial-volume N` to binary (if binary supports it; add arg parse if not)
   - Complexity: LOW. Addresses startup discrepancy only.

2. **Send current LMS volume via `/control/volume` immediately after Spirc becomes active**
   - After daemon start + Spirc active, Daemon.pm fires a one-shot volume sync
   - Complexity: LOW. Addresses startup discrepancy only, uses existing control endpoint.

3. **Linear volume mapping (`--volume-ctrl linear`)**
   - librespot's `--volume-ctrl` (or SoftMixer config) supports linear vs. log scaling
   - Linear makes percentage-to-dB relationship predictable
   - May make the output "too loud" at high volume compared to Browse mode (different curve)
   - Complexity: MEDIUM (binary arg change + user testing)

4. **Accept as known limitation**
   - Document that Connect volume is controlled by the Spotify app, not LMS
   - Inform users to match volumes manually
   - Complexity: ZERO. Pragmatic for v1.3 given other priorities.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Initial volume sync on daemon start | Connect shouldn't suddenly be louder than LMS setting when Spotify app connects | LOW | Read `$client->volume` in Daemon.pm, pass as `--initial-volume` arg to binary. Binary already suppresses first echo (CON-11). |
| Document volume discrepancy if not fully fixed | Users need to know this is a known issue with workaround | LOW | Update Setup Guide or Settings description |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Bi-directional volume sync: Connect→LMS already works; add LMS→Connect on LMS volume change | Full sync: when user moves LMS volume knob during Connect session, librespot volume follows | MEDIUM | Already have `_onVolume` which fires on LMS volume change; extend to also call `/control/volume`. Current code only goes LMS→Web API, not LMS→binary. |
| Linear volume mapping flag | Predictable 1:1 volume mapping for audiophiles | MEDIUM | Binary change (add `--volume-ctrl` arg parse), test carefully |

### Anti-Features

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| PCM-level volume scaling in the relay | Complicates the relay significantly; different result per format | Use Spirc volume control instead |
| Polling Web API for current device volume | `GET /me/player` is expensive (rate limit), doesn't reflect real-time | Read from binary's event stream or LMS pref |

### Dependencies

```
Daemon.pm start()
    └── Read: $client->volume (before spawn)
    └── Add: --initial-volume arg to @helperArgs (convert 0-100 → 0-65535)

connect.rs (binary)
    └── Add: --initial-volume N arg parse (if not already present)
    └── Pass N to ConnectConfig.initial_volume

Alternative (post-start sync):
Connect.pm
    └── After 'start' event received: fire one-shot /control/volume with current LMS volume
    └── Source-mark the LMS request to prevent echo-back via _onVolume
```

---

## Feature Area 4: Format Dropdown B&O/Chromecast Verification

### The Problem (from Backlog item #3)

"Auto-Modus mit B&O/Chromecast verifizieren (kein OGG-Support → Auto sollte FLAC wählen). Bisher nur mit squeezelite getestet."

### How the Format Selection Works (current codebase)

`Plugin.pm::updateTranscodingTable()` removes unused transcoding entries from
LMS's in-memory table based on the `streamFormat` pref:
- `auto` → remove PCM, FLAC, MP3 entries; leave OGG (preferred for native capability)
- `ogg`  → remove everything except OGG passthrough
- `pcm`  → remove everything except PCM
- `flac` → remove everything except FLAC
- `mp3`  → remove everything except MP3

The "Auto" intent: OGG passthrough if player supports it (squeezelite does), else LMS transcodes
to whatever the player requests. B&O/Chromecast via UPnPBridge will negotiate a format through
UPnP DLNA protocol — if they don't accept `audio/ogg`, LMS should fall back via transcoding.

### Verification Scope

This is a **test/verification task**, not a code change task. The question is:

1. Does Auto mode correctly fall back to FLAC or PCM when B&O requests non-OGG?
2. Does the UPnPBridge correctly translate the content-type negotiation?
3. Is the `custom-convert.conf` transcoding entry for FLAC correct for B&O?

No code changes are expected until verification identifies a bug.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Auto mode works on B&O/Chromecast (UPnP) players | Plugin should work on all LMS player types, not just squeezelite | LOW (test) | Test: connect B&O player, set Auto, play a track. Check Songinfo for format. |
| FLAC mode works on B&O | FLAC is the expected fallback for non-OGG players | LOW (test) | Already implemented; just needs hardware verification |
| Format Songinfo shows correct format on B&O | Debugging aid — verify format negotiation succeeded | LOW (already built) | META-02 already implemented; just verify it reports correctly on B&O |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Explicit "FLAC for non-OGG players" detection | Auto-detect OGG capability and default to FLAC for UPnP players | HIGH | Would require querying UPnPBridge capabilities; overkill — let user set FLAC explicitly |
| Per-player-model default format | Pre-configure B&O to use FLAC by default | MEDIUM | Fragile (model names vary); manual setting is safer |

### Anti-Features

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Silent FLAC-without-verification | Writing FLAC-forcing code without real hardware test may break squeezelite | Test first, code only if test fails |
| Documenting as "known issue — use FLAC setting" without testing | Lazy; B&O is one of two test player types in the dev environment | Must test |

### Dependencies

```
No code changes until verification finds a bug.

Test environment: B&O devices via UPnPBridge (already in dev env per .planning/PROJECT.md)

If a bug is found:
    custom-convert.conf — may need additional or corrected FLAC/MP3 entries for soc-* pipeline
    Plugin.pm::updateTranscodingTable() — may need logic adjustment for UPnP player detection
```

---

## Feature Area 5: Spotify Extended Quota Client-ID Preparation

### Current Situation

SpotOn uses a bundled ncspot Extended Quota App ID for Browse endpoints (Dual-Token routing).
This works but is legally fragile — using a third party's client ID without permission.

### Extended Quota Requirements (2026)

From Spotify's official documentation (as of May 2025 update):

| Requirement | Status for SpotOn |
|-------------|------------------|
| Established business entity (legally registered) | **FAIL** — individual open source project |
| Operating an active, launched service | PASS — plugin is public and working |
| Minimum 250,000 monthly active users | **FAIL** — far below threshold |
| Commercial viability | FAIL |

**Conclusion:** SpotOn does not qualify for Extended Quota under Spotify's May 2025 criteria.
The Extended Quota application path is blocked for individual developers / open source projects.

### What "Preparation" Actually Means

Given the quota application is impossible under current criteria, "preparation" in v1.3 means:

1. **Register SpotOn's own Spotify app** — create a Developer App with SpotOn's own credentials
   (Development Mode, no Extended Quota). Gives SpotOn its own rate limit bucket separate
   from the bundled ncspot ID. Reduces legal risk of using ncspot's ID.

2. **Make Client-ID configurable** — users who have their own Spotify Developer App can enter
   their Client-ID in settings. Already implemented as `pref_clientId` in Settings.pm.

3. **Document the situation** — explain in Setup Guide why Extended Quota isn't available,
   what Development Mode limitations apply, and how to register your own app as a workaround.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| SpotOn registers its own Spotify Developer App | Reduces legal/rate-limit risk of bundled ncspot ID | LOW | Out-of-band action by maintainer, not code change. Creates new `client_id` to hardcode as default. |
| Document Dev Mode limitations in Setup Guide | Users need to understand the 5-user limit and search limit | LOW | Update existing Setup Guide strings |
| Verify bundled client ID still works | ncspot's ID may be revoked at any time | LOW | Ongoing monitoring |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Dual-Token architecture already supports own Client-ID | When/if Extended Quota granted, just swap the bundled ID | ZERO | Already built — no code change needed |
| Client-ID field in Settings already exists | Users who have their own app can already plug it in | ZERO | Already built in Settings.pm |

### Anti-Features

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Applying for Extended Quota as an individual | Will be rejected; wastes time | Wait until criteria change or project gains org status |
| Switching to PKCE flow for own Client-ID | PKCE requires browser redirect — breaks ZeroConf single-click auth | Stay with Keymaster/ZeroConf |
| Removing the bundled Client-ID safety net | Breaks Browse for users without own app | Keep bundled ID as fallback; own ID goes first if configured |

---

## Feature Area 6: Repo Maintenance Framework

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| GitHub Issue Templates (Bug Report, Feature Request) | Prevents low-quality issues; guides users to provide version, log, steps | LOW | `.github/ISSUE_TEMPLATE/bug_report.yml` + `feature_request.yml`. Standard GitHub feature. |
| CONTRIBUTING.md | Contributors need guidance on how to submit PRs, run tests, build binary | LOW | One file; outline test commands, build commands, code style |
| CI for Perl tests (GitHub Actions) | Tests should run on every PR/push; catches regressions | MEDIUM | Existing `t/` directory with 230 tests; LMS test harness. Need to find if LMS provides a test runner CI action or mock setup. |
| Security policy (SECURITY.md) | LMS plugin repo submission may require it | LOW | Template: report vulnerabilities via email, not public issues |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Automated test run on PR | Catch regressions before merge | MEDIUM | Requires LMS test environment in CI; Spotty-Plugin uses no CI — SpotOn would be first LMS Spotify plugin with CI |
| Changelog (CHANGELOG.md) | Users can see what changed between versions | LOW | Retroactively document v1.0/v1.1/v1.3 changes |
| Code of Conduct | Standard open source hygiene | LOW | One boilerplate file |

### Anti-Features

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Complex CI with hardware-in-loop tests | CI can't test actual audio output or B&O hardware | Unit/integration tests in CI; hardware tests manual |
| Automated release pipeline | Release procedure is partially manual (SHA, repo.xml update); automate in future | Document the procedure in CONTRIBUTING.md |

---

## Feature Area 7: macOS Binaries (Universal Binary)

### Current State

`Helper.pm` init() has:
```perl
if ( !main::ISWINDOWS && !main::ISMAC
     && Slim::Utils::OSDetect::details()->{osArch} =~ /^aarch64/i ) { ... }
if ( main::ISWINDOWS ) { ... }
```

There is no `ISMAC` branch adding a binary path. The `_findBin()` method relies entirely on
`Slim::Utils::Misc::findbin()` which searches LMS's configured bin paths. Without an explicit
`addFindBinPaths` call for macOS, the binary won't be found unless it happens to be in LMS's
default bin path.

### macOS Binary Naming Convention

LMS on macOS uses `Perl->$Config::Config{archname}` which produces `darwin-thread-multi-2level`
(Intel) or `darwin-thread-multi-2level` with arm64 (Apple Silicon under Rosetta) or
`darwin-2level` in some builds. Spotty-Plugin avoids this complexity by using a single `darwin`
binary name (Universal Binary with lipo) placed in a `darwin` subdirectory — LMS finds it via
`findbin()` which searches platform-specific subdirs of the plugin Bin/ dir.

### Universal Binary vs. Separate Binaries

A Universal Binary (`lipo -create intel aarch64 -output universal`) is preferred because:
1. Single download, single binary to distribute
2. macOS automatically runs the correct slice
3. Spotty uses this approach — consistent expectation for LMS plugin ecosystem

Build approach in GitHub Actions:
1. `runs-on: macos-latest` (Apple Silicon runner — natively builds aarch64)
2. `rustup target add x86_64-apple-darwin` → `cargo build --target x86_64-apple-darwin`
3. `cargo build --target aarch64-apple-darwin` (native)
4. `lipo -create x86_64/spoton aarch64/spoton -output universal/spoton`
5. Place in `Plugins/SpotOn/Bin/darwin/spoton`

### Helper.pm Change Required

```perl
if ( main::ISMAC ) {
    Slim::Utils::Misc::addFindBinPaths(
        catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'darwin')
    );
}
```

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| macOS Universal Binary (Intel + Apple Silicon) | LMS runs natively on macOS; plugin is useless without binary | MEDIUM | Two native CI builds + lipo merge. `macos-latest` GitHub runner is Apple Silicon. Cross-compile Intel from Apple Silicon requires `x86_64-apple-darwin` target. |
| Helper.pm `ISMAC` branch adds darwin/ bin path | Without explicit path, binary won't be found on macOS | LOW | One `addFindBinPaths` call in `init()` |
| Binary placed in `Plugins/SpotOn/Bin/darwin/spoton` | Consistent with LMS plugin binary naming convention | LOW | Follow Spotty's `darwin` directory name |
| CI workflow adds macOS job | Binary must be built and uploaded as release asset | MEDIUM | Add to `build-librespot.yml`: `runs-on: macos-latest`, two cargo builds, lipo step |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Code-signed binary | macOS Gatekeeper will block unsigned binaries from untrusted developers | HIGH | Requires Apple Developer account ($99/year). Without signing, users must manually approve in System Preferences → Security. Document workaround. Do not block v1.3 for this. |
| macOS-specific Connect discovery note | macOS uses Bonjour natively; libmdns (used by librespot) may conflict | LOW | Note in documentation: if mDNS issues, try `--disable-discovery` and use LAN Connect mode |

### Anti-Features

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Separate Intel + Apple Silicon binaries (no lipo) | Two binaries in different dirs; Helper.pm needs arch detection; doubled download size | Single Universal Binary via lipo |
| Shipping macOS binary in git repo | Large binary blob; increases clone size significantly | GitHub Release asset only |
| Signing requirement blocking release | Apple Developer account is optional; document the Gatekeeper workaround instead | Ship unsigned; document |

### Dependencies

```
GitHub Actions build-librespot.yml
    └── New job: build-macos (runs-on: macos-latest)
    └── Steps: rustup targets x86_64 + aarch64, two cargo builds, lipo merge
    └── Output: Plugins/SpotOn/Bin/darwin/spoton as release asset

Helper.pm init()
    └── Add ISMAC branch with addFindBinPaths to darwin/ dir

Plugins/SpotOn/Bin/darwin/
    └── New directory; add .gitkeep placeholder until CI builds and commits binary
```

---

## Feature Area 8: LMS Community Repo Submission

### How the LMS Community Plugin Repository Works

The LMS Community plugin repository (`lms-plugin-repository` on GitHub) works via aggregation:

1. Each plugin author hosts their own `repo.xml` file (SpotOn already has one at GitHub raw URL)
2. The central `include.json` lists URLs to all participating `repo.xml` files
3. A GitHub Actions job runs every few hours, fetches all repo.xml files, merges into `extensions.xml`
4. LMS fetches `extensions.xml` from lyrion.org and shows all listed plugins in Plugin Manager

**Submission process:** Submit a PR to `LMS-Community/lms-plugin-repository` adding SpotOn's
repo.xml URL to `include.json`. No quality review process documented — it's honor-system.
PRs are reviewed by LMS-Community maintainers before merge.

### Quality Bar Expected

No formal checklist exists, but examining existing plugins in the repo and community norms:

- Plugin must be functional and installable (zip + sha checksum)
- `install.xml` must be correct and version numbers must match
- Plugin must not crash LMS on install
- Must use its own namespace (`Slim::Plugin::SpotOn::*`) — SpotOn already does
- `repo.xml` must be publicly accessible and return valid XML

SpotOn already meets all technical requirements. The main risk is community scrutiny of
Spotify API legality (use of ncspot bundled Client-ID).

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| PR to lms-plugin-repository include.json | Gets SpotOn in the default LMS Plugin Manager — massive discoverability improvement | LOW | One-line PR adding SpotOn's repo.xml URL. SpotOn's existing repo.xml is already correct format. |
| verify install.xml is correct before submission | Broken install.xml = broken plugin install = embarrassing | LOW | Already correct; just verify version numbers match |
| Forum post on lyrion.org forums announcing plugin | LMS community is forum-driven; announcements drive adoption | LOW | Non-code task; write a brief announcement post |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Announcement includes macOS support note | Mac users are underserved in LMS ecosystem; highlighting Mac support drives adoption | LOW | Mention in forum post once macOS binary is ready |
| Readme on GitHub with clear install instructions | Users who find the GitHub repo need to know how to install | LOW | Update README.md with LMS Plugin Manager URL or manual zip install |

### Anti-Features

| Anti-Feature | Why Avoid | Alternative |
|--------------|-----------|-------------|
| Submitting before macOS binary is ready | Mac users will install and find it broken | Submit after macOS binary ships in same release |
| Submitting before Credential Isolation is fixed | Multi-account households would report Browse-shows-wrong-content bug immediately | Fix credential isolation first |

---

## MVP Recommendation for v1.3

### P1 — Must Ship (unblock distribution)

1. **Connect Credential Isolation** — one-line fix; blocks submission (multi-account bug)
2. **macOS Universal Binary** — CI job + Helper.pm change; blocks LMS Community submission
3. **LMS Community Repo Submission** — PR to include.json (after credential fix + macOS binary)

### P2 — Should Ship (polish)

4. **Like Button** — adds write-back for Liked Songs; meaningful UX improvement; medium complexity
5. **Connect Volume Fix** — initial volume sync; low complexity; addresses known discrepancy
6. **Repo Maintenance** — Issue Templates + CONTRIBUTING.md; low complexity; needed for community

### P3 — Verify/Investigate Only

7. **Format Dropdown B&O Verification** — no code change expected; verify existing Auto mode works
8. **Extended Quota Preparation** — out-of-band (register Spotify app); minimal code change

### Defer

- Like from Connect mode — high complexity, low demand
- Code-signed macOS binary — requires Apple Developer account
- Automated CI test runs — LMS test harness in CI is non-trivial

---

## Feature Dependency Graph

```
macOS Binary ─────────────────────────────────────┐
Credential Isolation ─────────────────────────────┤
                                                   └→ LMS Community Repo Submission

Like Button → user-library-modify scope → re-auth (ZeroConf one-time)
           → new API::Client methods
           → new OPML items in Plugin.pm

Volume Fix → optional binary change (--initial-volume arg)
           → Daemon.pm reads $client->volume at spawn time

Format Verification → test-only; code only if bug found

Repo Maintenance → independent (no deps on other features)

Extended Quota → out-of-band maintainer action; no code deps
```

---

## Sources

- SpotOn codebase (local): Daemon.pm, Connect.pm, Helper.pm, API/Client.pm, Plugin.pm (HIGH)
- SpotOn ROADMAP.md backlog items #1-4 (local): credential isolation + volume + format + quota context (HIGH)
- Spotty-Plugin OPML.pm: addAlbumToLibrary, followArtist patterns — no track liking (HIGH via gh api)
- Spotty-Plugin API.pm: addAlbumToLibrary, no addTracksToLibrary method confirmed (HIGH via gh api)
- Spotty-Plugin Helper.pm: macOS ISMAC handling absent — falls through to findbin() (HIGH via gh api)
- Qobuz plugin Plugin.pm: conditional favorite menu item pattern with nextWindow + showBriefly (MEDIUM via WebFetch)
- Spotify API quota modes: https://developer.spotify.com/documentation/web-api/concepts/quota-modes (HIGH)
- Spotify Extended Access criteria update April 2025: https://developer.spotify.com/blog/2025-04-15-updating-the-criteria-for-web-api-extended-access (HIGH)
- LMS plugin repository README: https://github.com/LMS-Community/lms-plugin-repository (HIGH)
- LMS repository dev docs: https://lyrion.org/reference/repository-dev/ (HIGH)
- Apple Universal Binary docs: https://developer.apple.com/documentation/apple-silicon/building-a-universal-macos-binary (HIGH)
- librespot initial volume issue: https://github.com/librespot-org/librespot/issues/1554 (MEDIUM)

---
*Feature research for: SpotOn v1.3 — Polish & Publish*
*Researched: 2026-06-06*
