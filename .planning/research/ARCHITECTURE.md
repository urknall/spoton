# Architecture Patterns — SpotOn v1.3 Polish & Publish

**Domain:** LMS Spotify Plugin (subsequent milestone — adding features to shipped v1.1)
**Researched:** 2026-06-06
**Confidence:** HIGH — based on direct codebase inspection of all .pm files

---

## Recommended Architecture

v1.3 adds no new components. Every feature integrates into existing module boundaries. The architecture stays flat; the work is surgical modifications and additive code within existing files.

### Existing Component Map

```
Plugin.pm               — OPML menu tree, initPlugin, updateTranscodingTable, _trackItem
API/Client.pm           — HTTP client, dual-token routing, rate limiting, all Spotify endpoints
API/TokenManager.pm     — token cache, Keymaster --get-token, ZeroConf discovery, DISCOVER_DIR
Connect.pm              — spottyconnect event dispatch (binary→LMS + LMS→binary), volume/seek/pause sync
Connect/Daemon.pm       — single librespot process lifecycle, per-account cache dir, port capture
Connect/DaemonManager.pm — multi-player daemon registry, watchdog, sync-group election
ProtocolHandler.pm      — spoton:// URI scheme, formatOverride, canDirectStream, getMetadataFor
Helper.pm               — binary discovery, --check capability detection, multi-arch path registration
Settings.pm             — web UI prefs, per-player toggles, daemon restart on save
strings.txt             — i18n keys (11 languages)
```

---

## Feature Integration Analysis

### 1. Connect Credential Isolation

**Status:** Largely already implemented — may need only verification.

`Connect/Daemon.pm` lines 87–94 already compute a per-account cache dir and pass it via `-c $cacheDir` to the librespot process:

```perl
my $activeAccountId = $prefs->client($client)->get('activeAccount')
                   || $prefs->get('activeAccount')
                   || '';
my $cacheDir = $activeAccountId
    ? catdir($serverPrefs->get('cachedir'), 'spoton', $activeAccountId)
    : catdir($serverPrefs->get('cachedir'), 'spoton');
```

`Plugin.pm::updateTranscodingTable` (called before each Browse-mode track) does the identical computation for the single-track `--single-track` invocations. Both code paths independently arrive at the same `spoton/<accountId>/` path. No collision between Connect daemon and Browse-mode streaming for the same account.

**Actual gap to investigate:** The ZeroConf discovery process uses `DISCOVER_DIR = '__DISCOVER__'` as a temporary credential directory in `TokenManager.pm`. When discovery completes, credentials migrate to the per-account dir. The question: if two accounts complete ZeroConf concurrently (edge case: multi-user household doing simultaneous setup), does the `__DISCOVER__` dir get clobbered?

The stderr log in Daemon.pm uses a per-player (MAC-based) path, not per-account, so no collision there.

**Probable finding:** The isolation is complete for the common case. The phase work is verification + documentation, with a possible small fix in `TokenManager.pm` if the concurrent-ZeroConf edge case is real.

**Files to potentially modify:** `API/TokenManager.pm` only (DISCOVER_DIR migration path). `Daemon.pm` is already correct.

**No new components needed.**

---

### 2. Like Button (Liked Songs via PUT /me/tracks)

Three integration points across two existing files.

**A. API/Client.pm — new endpoint methods**

The write-operation pattern already exists for player control (`playerPause`, `playerPlay`, `playerVolume` all use `_request('put', ...)`). Adding Like follows the exact same shape:

```perl
sub saveTracks {
    my ($class, $accountId, $trackIds, $cb) = @_;
    $class->_request('put', 'me/tracks', {
        _accountId => $accountId,
        _noCache   => 1,
        ids        => join(',', ref $trackIds ? @{$trackIds} : ($trackIds)),
    }, $cb);
}

sub removeTracks { ... }   # DELETE /me/tracks
sub checkTracks  { ... }   # GET /me/tracks/contains
```

The `_meFamilyRegex` in Client.pm (`qr{^me(?:$|/|\?)}`) already routes all `me/*` paths through the own-token flavor automatically. No routing changes needed.

The Feb 2026 API introduced `PUT /me/library` as a unified endpoint. Both `PUT /me/tracks` and the new library endpoint are listed as working. Use `PUT /me/tracks` — it is the specific endpoint for tracks, has a clear scope requirement (`user-library-modify`), and avoids ambiguity with the newer unified endpoint.

**B. Plugin.pm — context action in _trackItem**

`_trackItem` (line 394) already builds a `@contextItems` array for each track and conditionally appends Artist View and Album View links. Adding a Like action extends this pattern:

```perl
push @contextItems, {
    name        => cstring($client, 'PLUGIN_SPOTON_LIKE_TRACK'),
    url         => \&_likeTrackAction,
    passthrough => [{ trackId => $track->{id} }],
    type        => 'link',
    nextWindow  => 'parent',
};
```

The `_likeTrackAction` handler calls `API::Client->saveTracks(...)` and returns a brief confirmation item. No new menu levels introduced.

**C. UI constraint — no liked/unliked state display**

OPML has no inline toggle. Showing current liked status per track requires one `GET /me/tracks/contains` call per track at feed-render time. With Dev Mode limiting search to 10 results per type, this would mean up to 10 additional API calls per feed render — acceptable in isolation but stack-risky if multiple feeds open simultaneously (concurrency cap is 3 in Client.pm). The safe design: Like is a write-only action, no pre-check of existing liked state shown in the UI. Spotify's SPD-11 requires the write signal to reach Spotify — it does not require displaying the current state.

**Files to modify:** `API/Client.pm` (3 new methods), `Plugin.pm` (context item in `_trackItem` + new `_likeTrackAction` sub), `strings.txt` (PLUGIN_SPOTON_LIKE_TRACK + related keys).

**No new files needed.**

---

### 3. Connect Volume Discrepancy Fix

**Where the bug lives:** `Connect.pm`, specifically the interaction between `_connectEvent` (binary→LMS volume) and `_onVolume` / `_bufferedSetVolume` (LMS→binary volume).

**Existing suppressions:**
- `VOLUME_GRACE_PERIOD = 20` seconds: `_connectEvent` ignores volume events from the binary within 20s of daemon start
- Source-marking (`request->source eq __PACKAGE__`): prevents echo loops between the two directions
- 0.5s debounce (`VOLUME_DEBOUNCE`) on `_bufferedSetVolume`

**Root cause of discrepancy:** The binary's `suppress_next_volume` AtomicBool (in Rust) already suppresses the very first `VolumeChanged` event after `SessionConnected`. The 20-second Perl-side grace period then suppresses all subsequent volume events from the binary for another ~20 seconds. During this window, if Spotify's device volume differs from LMS's current volume, they diverge and stay diverged until the user manually changes volume in either app.

Example: LMS player at 80%, Spotify had this device at 60% in a previous session. When Connect starts: binary's first volume event is suppressed by the Rust AtomicBool, further events suppressed by the 20s Perl grace period. The 60% vs 80% discrepancy persists.

**Fix options (two approaches):**

Option A — Reduce grace period: The Rust-side `suppress_next_volume` already handles the dangerous first event. The Perl-side 20s grace period is defense-in-depth that has become the primary source of the bug. Reducing to 2–3s (enough to cover the session-setup sequence) would allow the binary to report its actual volume within seconds of connect start.

Option B — Post-start sync: On `start` event in `_connectEvent`, after the session is established, schedule a timer (~3s) to call `API::Client->getPlayerState(...)` → read `device.volume_percent` → apply to LMS mixer (source-marked). This actively pulls ground truth from Spotify rather than relying on the binary's event delivery.

Option A is simpler and has lower API cost. Option B is more robust if the binary's volume events remain unreliable.

**If `getPlayerState` method doesn't exist in Client.pm:** Add it. The endpoint `GET /me/player` is listed as working in dev mode and is a me/* endpoint (own-token routing automatic).

**Files to modify:** `Connect.pm` (adjust `VOLUME_GRACE_PERIOD` constant and/or add post-start sync timer). Possibly `API/Client.pm` if `getPlayerState` method needs to be added.

**No new files needed.**

---

### 4. Format Dropdown B&O/Chromecast Verification

**Not a code change — QA verification task.**

The format dropdown (auto/ogg/pcm/flac/mp3) is per-player and controls which pipelines remain active in `updateTranscodingTable` (Plugin.pm lines 1344–1368). The pipeline deletion logic is implemented and works for squeezelite.

**What needs verification:** B&O devices (appearing via UPnPBridge as DLNA/UPnP players) and Chromecast (via Cast Bridge) report player capabilities to LMS differently than native Squeezebox players. LMS's `TranscodingHelper` selects the transcoding pipeline based on these capability bits. The plugin's role is only to leave the correct pipelines in the commandTable — LMS makes the final selection.

**Possible issue:** UPnP players often report different codec capabilities than Squeezebox hardware. If a B&O device reports FLAC support but actually needs PCM for the specific sample rate/bit depth that SpotOn outputs (S16LE 44.1kHz stereo), the auto format selection might choose an incompatible pipeline.

**Expected outcome:** Either (a) it works fine and the verification documents that B&O + Chromecast work with `auto` or a specific format setting, or (b) a bug is found in pipeline selection for UPnP players, and a targeted fix is made in `updateTranscodingTable`.

**Files to potentially modify:** `Plugin.pm::updateTranscodingTable` if a bug is found in UPnP player format negotiation. `custom-convert.conf` if new pipeline variants are needed.

---

### 5. Extended Quota Client-ID Preparation

**Not a code change — external process task.**

The Settings.pm `clientId` pref (lines 74–83) already handles input with proper validation (alphanumeric, max 32 chars) and the dual-token routing in Client.pm already supports the own Client-ID for all `me/*` endpoints. The infrastructure for using a custom Client-ID is fully in place.

**What the phase produces:** A completed Spotify Extended Quota application, not code. The application requires: app description, use case justification, expected number of users, demonstration of Spotify design guideline compliance, OAuth redirect URIs.

**One auth consideration:** Extended quota approval typically requires a `redirect_uri` registered in the Spotify Developer Dashboard. SpotOn's current auth uses ZeroConf (no browser redirect). If Spotify's review process requires a working PKCE/browser flow demonstration, `TokenManager.pm` may need a PKCE-path for the application demo — but this does not affect production users who use ZeroConf.

**Files to potentially modify:** `API/TokenManager.pm` if a PKCE demo flow is needed for the application. This is not a production change.

---

### 6. Repo Maintenance (Issue Templates, Contributing, CI)

**CI — the only code-adjacent part.**

**Existing test infrastructure:** 12 test files in `t/`, all runnable with `prove t/`. The canonical pattern (established in `t/05_perl_syntax.t`): write LMS module stubs to a temp dir, then load plugin modules against those stubs. Tests cover: install.xml validity, strings.txt completeness, custom-convert.conf pipeline correctness, custom-types.conf, perl -c syntax for all .pm files, binary --check protocol, token manager behavior, API client routing, Settings.pm save/restore, stream metadata, track history, protocol rename.

**What CI needs:** A GitHub Actions workflow that runs `prove t/` on every push and PR. The test suite is pure Perl — no LMS runtime, no Docker, no external services needed.

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: perl -V && prove -v t/
```

All test dependencies (`Test::More`, `File::Temp`, `File::Path`, `Cwd`, `Digest::MD5`, `MIME::Base64`) are in Perl core or standard distribution.

**New files:** `.github/workflows/test.yml`, `.github/ISSUE_TEMPLATE/bug_report.md`, `.github/ISSUE_TEMPLATE/feature_request.md`, `CONTRIBUTING.md`.

**No changes to existing .pm files.**

---

### 7. macOS Binaries (Universal Binary)

**Integration point:** `Helper.pm` only.

**Current state:** `Helper.pm::init()` (lines 19–37) handles two platforms:
- aarch64 Linux: adds `armhf-linux/` as fallback path  
- Windows: adds `x86_64-win64/` path

macOS (`main::ISMAC`) has no explicit path registration — it falls through to `_findBin` which calls `Slim::Utils::Misc::findbin('spoton')` with whatever paths LMS registers by default. This works if macOS LMS adds the plugin's Bin dir to findbin paths, but a `darwin/` subdirectory is the conventional approach.

**Helper.pm change needed:**

```perl
if ( main::ISMAC ) {
    Slim::Utils::Misc::addFindBinPaths(
        catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'darwin')
    );
}
```

A Universal Binary (`lipo -create x86_64_build aarch64_build -output spoton`) contains both Intel and Apple Silicon slices in a single file. macOS automatically executes the correct slice. This means no arch-detection logic is needed in Perl — a single `Bin/darwin/spoton` handles both. The binary's `--check` output confirms identity (`ok spoton vX.Y.Z`).

**Binary build process (Rust/CI task, not Perl):** Requires a macOS build environment (or macOS GitHub Actions runner). Cross-compilation from Linux to macOS requires `osxcross` with the macOS SDK — nontrivial. The practical path is a macOS GitHub Actions runner:

```yaml
- runs-on: macos-latest
  steps:
    - run: cargo build --release --target x86_64-apple-darwin
    - run: cargo build --release --target aarch64-apple-darwin
    - run: lipo -create target/x86_64-apple-darwin/release/spoton
                        target/aarch64-apple-darwin/release/spoton
                 -output Bin/darwin/spoton
```

**New directory:** `Plugins/SpotOn/Bin/darwin/spoton` (single universal binary).

**install.xml:** No change — LMS discovers binaries via `findbin` path registration at runtime, not via manifest.

**Files to modify:** `Helper.pm` (add `main::ISMAC` block). New binary directory added.

---

### 8. LMS Community Repo Submission

**Not a code change — submission process.**

Prerequisites already met: `install.xml` is correct (version 1.2.9, minVersion 8.0, correct GUID), `repo.xml` exists at repo root, plugin handles missing binary gracefully (`handleFeed` shows `PLUGIN_SPOTON_BINARY_MISSING` textarea if `Helper->get()` returns nothing).

The submission process requires: plugin page on LMS Community forums, email to LMS Community repo maintainers, verified `repo.xml` format matching the community repo schema.

**One pre-check:** Confirm the `repo.xml` URL format matches what the LMS Community repo aggregator expects. This is a documentation/process verification, not code.

---

## Component Boundaries — v1.3 Changes Summary

| Component | Change Type | What Changes |
|-----------|-------------|--------------|
| `API/Client.pm` | Additive | New `saveTracks`, `removeTracks`, `checkTracks` methods; possibly `getPlayerState` |
| `Plugin.pm` | Additive | Like context item in `_trackItem`; new `_likeTrackAction` handler sub |
| `Connect.pm` | Modification | Volume grace period reduction or post-start sync timer for volume fix |
| `Connect/Daemon.pm` | Verify only | Per-account cache dir already correct; confirm DISCOVER_DIR edge case |
| `API/TokenManager.pm` | Possibly none | Investigate ZeroConf concurrent-setup edge case only |
| `Helper.pm` | Additive | `main::ISMAC` block for `Bin/darwin/` path registration |
| `Settings.pm` | None | No changes expected |
| `ProtocolHandler.pm` | None | No changes expected |
| `strings.txt` | Additive | PLUGIN_SPOTON_LIKE_TRACK and confirmation keys |
| `.github/` | New | CI workflow, issue templates |
| `Bin/darwin/` | New | Universal binary for macOS |

---

## Data Flow Changes

### Like Button (new flow)

```
User taps "Like" in OPML track context menu
  → _likeTrackAction($client, $cb, $args, $passthrough)
    → API::Client->saveTracks($accountId, [$trackId], sub { ... })
      → _request('put', 'me/tracks', {_accountId => ..., _noCache => 1, ids => $trackId})
        [me/* → own-token flavor automatic via _meFamilyRegex]
        → SimpleAsyncHTTP PUT to https://api.spotify.com/v1/me/tracks?ids=...
          → 200 OK → $cb->({items => [{name => 'Liked', type => 'textarea'}]})
```

No caching involved (write operation). No LMS database writes. Rate-limit applies to own-token bucket (shared with all `me/*` requests — monitor if this creates contention with rapid browsing).

### Volume Fix (modified flow)

**Current (broken):**
```
Connect start
  → 20s grace period begins (Perl-side VOLUME_GRACE_PERIOD)
  → binary sends volume events → all suppressed for 20s
  → LMS and Spotify volume diverge and stay diverged
```

**Fixed (Option A — reduced grace period):**
```
Connect start
  → 2–3s grace period (covers session setup)
  → binary sends current Spotify volume → _connectEvent applies to LMS mixer (source-marked)
  → volumes converge within seconds
```

**Fixed (Option B — post-start sync):**
```
Connect start event received
  → set 3s timer
  → timer fires → API::Client->getPlayerState(accountId, sub { ... })
    → GET /me/player [own-token]
    → extract device.volume_percent
    → apply to LMS mixer (source-marked, skips _onVolume echo)
  → volumes converge
```

---

## Suggested Build Order

Dependencies and risk determine sequence:

**1. CI Setup** — Foundation. Enables regression detection for all subsequent changes. Unblocks macOS binary validation. No .pm changes, low risk. Deliver first.

**2. Connect Volume Fix** — Standalone bug fix, no dependencies, high user-facing impact. Validate on squeezelite + B&O hardware. Deliver early.

**3. Like Button** — Requires Client.pm new methods first, then Plugin.pm integration. Self-contained. No cross-feature dependencies. Medium complexity, well-understood pattern.

**4. Connect Credential Isolation Verification** — Investigation task. Run concurrently with Like Button code work. If gaps found: TokenManager.pm fix. If complete: document and close.

**5. Format Dropdown B&O/Chromecast Verification** — Requires hardware access (B&O + Chromecast). Run as QA alongside any code task, no code dependency.

**6. macOS Binaries** — Rust build task. Helper.pm Perl change is trivial. Blocked on macOS build environment (GitHub Actions macOS runner). Deliver when CI is in place (CI runner can build macOS binaries).

**7. Repo Maintenance** — Issue templates, Contributing doc. Documentation only. Can be done anytime after CI is working.

**8. Extended Quota Preparation** — External Spotify process. Start early (Spotify review can take up to 6 weeks) and run in parallel. No code dependency.

**9. LMS Community Repo Submission** — Final gate. All features complete and tested. Last.

---

## Architecture Anti-Patterns to Avoid

### Pre-fetching liked state per track at render time

Adding a `checkTracks` call inside `_trackItem` to show a heart icon per track multiplies API calls by the number of tracks displayed. 50 Liked Songs page = 50 `GET /me/tracks/contains` calls. Dev Mode rate limits make this unviable. Accept the constraint: Like is a write-only action without pre-fetched state display.

### Third copy of the cache-dir computation

`Daemon.pm` and `Plugin.pm::updateTranscodingTable` both independently compute `catdir($serverPrefs->get('cachedir'), 'spoton', $activeAccountId)`. If a fix is needed in credential isolation, do not add a third copy — refactor to a shared helper function (in Plugin.pm or a new `Util.pm`) and update both call sites.

### Using ['time', N] in the volume fix

The volume fix must not use `['time', N]` or any mechanism that touches the stream position. The fix is a `mixer volume` operation only. The existing pattern (`Slim::Control::Request->new($client->id, ['mixer', 'volume', $volume])->execute()` with source-marking) is correct and already used in `_connectEvent`.

### Like state stored locally

Spotify SPD-11 is explicit: liked content must NOT be stored locally. `saveTracks` writes to Spotify only. Do not cache the liked/unliked state in `Slim::Utils::Cache`.

### Separate Intel and ARM macOS binaries in Bin/

Distributing two separate macOS binaries (`x86_64-darwin/` and `aarch64-darwin/`) requires arch detection in Perl. A Universal Binary eliminates this. Always use `lipo -create` to produce a single `Bin/darwin/spoton` file.

---

## Integration Points Summary

| Integration Point | Direction | Mechanism | File |
|------------------|-----------|-----------|------|
| Like track write | Perl → Spotify | PUT /me/tracks | API/Client.pm (new) + Plugin.pm (new handler) |
| Like track volume sync | Binary → LMS | mixer volume request (source-marked) | Connect.pm (modified) |
| Post-start volume sync | Perl → Spotify | GET /me/player | Connect.pm (timer) + possibly Client.pm (new method) |
| Credential isolation | Daemon → Filesystem | -c $cacheDir arg | Daemon.pm (verify) |
| macOS binary path | Perl → Filesystem | addFindBinPaths(Bin/darwin/) | Helper.pm (new block) |
| CI test execution | GitHub → Perl | `prove t/` | .github/workflows/test.yml (new) |

---

## Confidence Assessment

| Area | Confidence | Reason |
|------|-----------|--------|
| Like Button integration | HIGH | Pattern identical to existing player control methods; me/* routing automatic |
| Volume fix root cause | HIGH | Traced through Connect.pm VOLUME_GRACE_PERIOD constant and _connectEvent handler directly |
| Credential isolation status | HIGH | Daemon.pm and Plugin.pm cache-dir code read directly; isolation already in place for standard case |
| macOS binary architecture | MEDIUM | Helper.pm ISMAC block needed confirmed; Universal Binary approach confirmed correct; build environment not yet verified |
| CI wiring | HIGH | Test suite structure read; all dependencies in Perl core |
| Format dropdown B&O | MEDIUM | Logic in updateTranscodingTable confirmed correct; B&O UPnP capability reporting not directly observable without hardware test |

---

## Open Questions for Phase Planning

1. **Volume fix: Option A vs B?** Reducing VOLUME_GRACE_PERIOD to 2–3s (Option A) may be sufficient if the Rust-side AtomicBool reliably suppresses the echo. Option B (post-start API poll) is more robust but adds an API call per Connect session start. Test Option A first — if the discrepancy recurs, escalate to Option B.

2. **macOS CI runner availability?** GitHub Actions `macos-latest` runners are available on public repos but have billing implications for heavy usage. One-shot build acceptable; continuous CI builds on macOS may be too expensive. Alternative: build macOS binaries locally once, commit to repo, exclude from CI rebuild.

3. **ZeroConf concurrent-setup edge case real?** Multi-user household with two users simultaneously doing ZeroConf is an edge case requiring specific hardware setup to reproduce. If the investigation cannot replicate the scenario, close credential isolation with a documented "already complete" finding.

4. **Like Button scope: only Save, or also Remove?** `saveTracks` (PUT) is the Like action. `removeTracks` (DELETE) is the Unlike action. Building both makes the feature complete, but the UI requires a separate "Unlike" context item that only makes sense when the track is already liked. Without pre-fetching liked state, Unlike cannot be intelligently shown. Decision: implement Save only for v1.3; Unlike can be a v1.4 addition if liked-state display becomes feasible.

---

*Architecture research for: SpotOn v1.3 — Polish & Publish*
*Researched: 2026-06-06*
