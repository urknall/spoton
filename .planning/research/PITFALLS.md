# Domain Pitfalls: SpotOn v1.3 — Polish & Publish

**Domain:** LMS Spotify plugin — v1.3 milestone additions (Perl + Rust binary + CI + repo submission)
**Researched:** 2026-06-06
**Confidence:** HIGH (verified against existing codebase + upstream sources)
**Scope:** NEW pitfalls for v1.3 features ONLY — P-01 through P-46 from v1.0/v1.1 are excluded. Numbering continues from P-47.

---

## Critical Pitfalls

### P-47: Like Button — `PUT /me/tracks` Is Removed; New Endpoint Uses URIs, Not IDs

**What goes wrong:**
The natural implementation of a Like Button calls `PUT /v1/me/tracks?ids=TRACK_ID` (the historical endpoint). This endpoint was removed as of February 2026. Calling it returns 403 Forbidden with no useful error message, even with a valid token and `user-library-modify` scope. The client code silently fails — the user taps Like, no feedback, nothing saved.

**Why it happens:**
The February 2026 API overhaul retired all old per-type library write endpoints (`PUT /me/tracks`, `PUT /me/albums`, `DELETE /me/tracks`, etc.) in favor of a single unified `PUT /me/library` / `DELETE /me/library` endpoint that accepts Spotify URIs instead of IDs.

**The correct endpoint:**
```
PUT /v1/me/library
Body: {"uris": ["spotify:track:TRACK_ID"]}
Scope: user-library-modify
```
The body must contain full `spotify:track:` URIs — bare IDs are rejected. The query-string `ids=` pattern from the old endpoint does not work on the new one.

**Consequences:**
A 403 that looks like a scope or auth error, but is actually an endpoint removal. If the Like Button calls the old path and swallows errors, the UI shows no feedback and the bug is invisible in testing unless the developer checks the response body.

**Prevention:**
Only implement `PUT /me/library` with URI payloads. Never use `PUT /me/tracks`. Verify in Client.pm that `getSavedTracks` (`GET /me/tracks`) is kept distinct from the write endpoint — `GET /me/tracks` still works for reading; only the write path changed.

**Detection warning signs:**
API Client receives 403 on the like action. Check `$cleanPath` in logs — if it says `me/tracks` on a PUT, the wrong endpoint is being used. Confirmed issue: Spotify developer community report of `403 Forbidden on PUT /v1/me/tracks` even with correct scope (community.spotify.com, 2026).

**Phase to address:** Like Button implementation (Phase N: Like Button).

---

### P-48: Like Button — `user-library-modify` Scope Is Not in the Current Token

**What goes wrong:**
The current ZeroConf/Keymaster token acquisition flow (`spoton --get-token`) retrieves a token from Keymaster. Keymaster tokens include a fixed scope set determined by the client ID's registration. Adding `PUT /me/library` (Like Button) requires `user-library-modify` scope. If the existing token was cached without that scope, the Like request returns 403 Insufficient Scope — not a transport error but a scope error, indistinguishable from P-47 in the log without reading the response body.

**Why it happens:**
The `own` token flavor is obtained via the user's own Client-ID through Keymaster. The scopes returned depend on what Keymaster grants for that client ID. The `bundled` ncspot token cannot be used for `me/*` write endpoints (hard guard D-05 in Client.pm). Cached tokens do not re-negotiate scope; the user must re-auth to get a new token with the expanded scope list.

**Consequences:**
Users who set up SpotOn before v1.3 have cached tokens without `user-library-modify`. The Like Button returns 403. The fix is a forced token refresh (cache invalidation), but this requires the user to re-trigger ZeroConf discovery. Without a clear in-plugin message, the user sees a silent failure.

**Prevention:**
1. When adding the Like Button, bump the `cacheSchemaVersion` constant in `Plugin.pm` to force a one-time token cache flush on upgrade.
2. In `Settings.pm`, document that users may need to re-authenticate after the v1.3 upgrade.
3. In the Like Button handler, detect 403 response bodies that contain `Insufficient client scope` and surface an actionable error message in the LMS UI.

**Detection warning signs:**
API Client log shows 403 on `me/library` (PUT). Response body contains `error.message = "Insufficient client scope"`. Confirm the token is the `own` flavor (log line includes `[flavor=own]`).

**Phase to address:** Like Button implementation.

---

### P-49: Connect Credential Isolation — Race Between Daemon Restart and Browse Token

**What goes wrong:**
When a second Spotify user connects via Spotify Connect to a player, librespot overwrites `credentials.json` in the shared cache dir (`spoton/{accountId}/`) with the new user's Spotify blob. The next `--get-token` call by the Browse/API path reads this overwritten `credentials.json` and retrieves a token for the wrong user — Browse menus show the connecting user's library, not the authenticated SpotOn account's library.

**Why it happens:**
Both the Connect daemon and the Browse `--get-token` invocations point to the same `cacheDir` (computed identically in `Daemon.pm:91-93` and `Plugin.pm:1270-1272`). librespot's session startup unconditionally writes the new session's credentials blob to `credentials.json` in whatever `--cache` dir it receives. There is no locking, no versioning, and no "read-only credentials" mode in librespot 0.8.0.

**The isolation fix:**
The Connect daemon should use a separate `--cache` dir scoped to the player MAC address:
```
Browse/Token: spoton/{accountId}/              (unchanged)
Connect:      spoton/connect-{mac_no_colons}/  (new, per-player)
```
This prevents the Connect session blob from overwriting the Browse session's `credentials.json`. Each daemon reads its own credentials sub-directory. The MAC-scoped dir is already the natural key for `Daemon.pm` (`$self->id` is the MAC without colons).

**Consequences:**
Without isolation: after a Connect session from a different account, Browse shows alien library content. Token invalidation does not fix this because the wrong credentials blob is already on disk. The user must re-authenticate the correct account via ZeroConf.

**Race condition detail:**
The race is not a true concurrent write race (LMS is single-threaded). It is a sequential clobber: librespot Connect daemon writes blob on session start, then Browse's `--get-token` reads the now-wrong file. The Perl event loop ordering means the Browse call always loses if it executes after the daemon start event.

**Prevention:**
In `Daemon.pm::start()`: compute `$cacheDir` as `spoton/connect-{$self->id}/` (MAC without colons), NOT `spoton/{accountId}/`. Create this directory if missing. The `--cache` flag passed to librespot in `@helperArgs` must point to the isolated dir. The Browse/Token `--cache` in `Plugin.pm::updateTranscodingTable` and `TokenManager.pm::_fetchKeymasterToken` must remain at `spoton/{accountId}/`.

**Detection warning signs:**
Browse menus show tracks from a different Spotify account after someone else connected via Spotify app. Log shows `--cache spoton/{accountId}` in both the Connect daemon start command AND the `--get-token` command — if both paths show the same dir, isolation is not in place.

**Phase to address:** Connect Credential Isolation (first priority in v1.3).

---

### P-50: Connect Volume Discrepancy — librespot Default Curve Is `log`, LMS Uses Linear

**What goes wrong:**
At 50% volume in the Spotify app, Connect sounds significantly louder than Browse at 50% LMS volume. The Connect volume event comes from Spirc as a 0-100 integer (already converted by SpotOn's binary from the raw 0-65535 Spirc range). However, librespot applies its own internal `--volume-ctrl log` (logarithmic) curve to the output PCM before it hits the HTTP stream. The LMS/squeezelite side applies a second curve (squeezelite's built-in soft-volume), resulting in double-curved attenuation at low end and elevated perceived loudness near midpoints.

**Why it happens:**
librespot's default `--volume-ctrl` is `log`. When SpotOn passes volume=50 to librespot via `/control/volume`, librespot applies the log curve internally on the audio output side. LMS then also applies its own volume curve via the player's mixer. The two curves compound. The pipe backend with `--passthrough` (Ogg) bypasses librespot's software volume entirely — so passthrough mode has a *different* discrepancy than PCM mode.

**The available fix:**
Pass `--volume-ctrl linear` to the Connect daemon. This makes librespot apply a proportional (linear) volume curve to its audio output, which then compounds predictably with squeezelite's linear soft-volume. The result is still not identical to Browse (which has no librespot-side volume application at all in single-track mode), but the perceived discrepancy is reduced significantly.

**Consequences of wrong fix:**
If you set `--volume-ctrl fixed`, librespot ignores all volume commands — Connect volume appears stuck. If you attempt PCM-side volume scaling in the HTTP relay, the FIFO/pipe buffering creates a lag between slider movement and audio change (P-19 inherited pattern). Volume normalisation (`--enable-volume-normalisation`) compounds this further — the normalisation gain interacts with the volume curve.

**Prevention:**
Add `--volume-ctrl linear` to the Connect daemon's `@helperArgs` in `Daemon.pm::start()` unconditionally, next to `--enable-volume-normalisation`. Verify the flag is not conditional on a pref (keep it always on for consistent behavior). Test with squeezelite AND with UPnP/B&O to confirm neither device clips or distorts with linear curve at high volumes.

**Detection warning signs:**
Connect-mode audio at 50% slider is noticeably louder than Browse-mode at 50% slider, on the same player. B&O UPnP players may exhibit different behavior because UPnPBridge applies its own volume scaling. Test each player type separately.

**Phase to address:** Connect-Mode Volume Fix.

---

## Moderate Pitfalls

### P-51: Like Button — Rate Limit on `PUT /me/library` Is Unexpectedly Aggressive

**What goes wrong:**
Unlike read endpoints, library write endpoints hit rate limits with very low request counts. Community reports document `PUT /me/tracks` (now `PUT /me/library`) rate-limiting after a single successful request, with `Retry-After` values of 6 to 24 hours. This is an app-wide rate limit (not per-user), so heavy testing by the developer during implementation exhausts the quota for all users of the same Client-ID.

**Why it happens:**
Spotify's rolling 30-second window applies app-wide. Write endpoints appear to have a lower token bucket than read endpoints, but Spotify does not document per-endpoint limits. A `Retry-After: 86400` (24 hours) has been observed in the wild for a single Like action.

**Prevention:**
1. The Like Button action must pass through the existing `API/Client.pm` `_request` path (not a direct HTTP call). The existing rate-limit guard (`spoton_rate_limit_own` cache key) already handles 429 responses.
2. Do not implement a "sync liked songs" batch operation at Like-button time. One Like = one request.
3. During implementation testing, use a separate Dev Mode app (different Client-ID) to avoid exhausting the production Client-ID's write quota.
4. Check the `user-library-modify` scope token uses the `own` flavor (D-05 guard ensures this — `me/*` always routes to own token, which is correct).

**Detection warning signs:**
First Like action works; subsequent ones within the same session return 429 with multi-hour `Retry-After`. This is a real rate-limit hit, not a bug. Log the `Retry-After` value and surface it to the user.

**Phase to address:** Like Button implementation.

---

### P-52: macOS Binaries — Gatekeeper Blocks Unsigned `spoton` Command-Line Binary on macOS Sequoia

**What goes wrong:**
macOS Sequoia (15.x) removed the option to Control-click override Gatekeeper for unsigned software. Users downloading the SpotOn plugin zip (which contains the `spoton` binary for macOS) cannot run it by clicking through a dialog — they must go to System Settings > Privacy & Security to manually allow it. Without guidance, the binary simply fails silently (LMS logs "SpotOn helper application not found" or similar).

**Why it happens:**
Notarization for standalone command-line binaries (Mach-O executables) is constrained: you cannot staple a notarization ticket directly to a raw binary — stapling only works on `.dmg`, `.pkg`, and `.app` bundles. A signed but unstapled binary that is distributed as a raw file inside a `.zip` acquires the quarantine attribute (`com.apple.quarantine`) when downloaded by Safari or a browser. Gatekeeper then requires the notarization ticket to be fetched from Apple's OCSP servers, which requires the binary to be notarized (signed with a Developer ID + submitted to Apple). This requires a paid Apple Developer Program membership ($99/year).

**The realistic options:**

| Approach | Cost | User friction | Outcome |
|----------|------|--------------|---------|
| Code-sign + notarize via Developer ID | $99/year + CI setup | Low (works on first run) | Best UX |
| Ship unsigned; document `xattr -d com.apple.quarantine` workaround | $0 | High (CLI for non-technical users) | Acceptable for now |
| Wrap binary in `.pkg` installer + notarize | $99/year | Medium (double-click PKG) | Better than raw binary but same cost |
| Don't ship macOS binaries initially | $0 | N/A (feature unavailable) | Pragmatic deferral |

**Consequences of ignoring this:**
macOS users who download the plugin find the binary silently blocked. LMS Server.log shows no binary found despite it being present. The fix is a user-visible README/setup-guide instruction: "macOS users: run `xattr -d com.apple.quarantine Plugins/SpotOn/Bin/mac/spoton` after first install."

**Prevention for the macOS binary phase:**
1. Build macOS universal binary on `macos-latest` runner (native Apple Silicon + Intel via `lipo`).
2. Ship unsigned initially with prominent documentation in `setup-guide.html` and GitHub README.
3. In the Setup Guide, add a macOS-specific note about the quarantine removal command. This is the same pattern used by other open-source CLI tools distributed outside the App Store.
4. Separately: investigate Apple Developer Program enrollment for future notarization — this is a milestone gate, not a v1.3 blocker.

**Detection warning signs:**
LMS log on macOS: "Didn't find SpotOn helper application!" despite binary being present in `Bin/mac/`. `file /path/to/spoton` shows a valid Mach-O binary. Running `spoton --check` from Terminal works after quarantine removal. Running without quarantine removal fails.

**Phase to address:** macOS Binary build + documentation.

---

### P-53: macOS Universal Binary — `lipo` Must Combine Two Separate Architecture Builds

**What goes wrong:**
GitHub Actions `macos-latest` runner is ARM64 (Apple Silicon). Cross-compiling to `x86_64-apple-darwin` from an ARM runner works natively (Rosetta-based cross-compile toolchain), but produces a single-arch binary. Creating a Universal Binary (`fat binary` with both `aarch64` and `x86_64` slices) requires running `lipo -create` to merge the two outputs. Forgetting `lipo` means the `mac` binary only works on one architecture.

**Why it happens:**
Rust's `cargo build --target x86_64-apple-darwin` on an `aarch64-apple-darwin` host (the default `macos-latest` runner since 2024) compiles via Rosetta-compatible toolchain. Both targets can be built on the same macOS runner by installing both Rust targets and building twice. `lipo -create aarch64_binary x86_64_binary -output universal_binary` is the merge step.

**Prevention:**
CI matrix entry for macOS must:
1. Install both `aarch64-apple-darwin` and `x86_64-apple-darwin` targets.
2. Build for each target separately.
3. Run `lipo -create ... -output spoton` to produce the Universal Binary.
4. Verify with `file spoton` — output must say `Mach-O universal binary with 2 architectures`.

**Detection warning signs:**
`file Plugins/SpotOn/Bin/mac/spoton` shows `Mach-O 64-bit executable arm64` (single arch). LMS running on Intel Mac cannot find a working binary.

**Phase to address:** macOS Binary build CI matrix.

---

### P-54: Format Dropdown B&O Verification — UPnPBridge Players Report Wrong Format Capabilities

**What goes wrong:**
The Auto format mode uses LMS player capability detection to choose OGG (for players that support it) or FLAC/PCM (for others). B&O players appear in LMS as UPnP players via UPnPBridge. UPnPBridge reports codec capabilities based on the UPnP `ProtocolInfo` response from the device. Some B&O models report OGG capability in ProtocolInfo but cannot actually decode Ogg Vorbis — they silently play silence or produce glitched audio.

**Why it happens:**
UPnP `ProtocolInfo` is self-reported by the device. Older B&O firmware sometimes advertises OGG MIME types inherited from generic UPnP profiles without actually implementing the codec. LMS trusts `ProtocolInfo` and may select OGG in Auto mode. The result is silent failure at the device level, with no error propagated back to LMS.

**The existing architecture:**
`custom-types.conf` defines `son-ogg` as a type requiring OGG support. LMS's format selection logic queries the player's supported types. B&O via UPnPBridge: the UPnP bridge translates between LMS and the device's native protocol — if the device advertises OGG, LMS sends OGG. If it doesn't transcode to the device's actual preferred format, the device gets a stream it cannot decode.

**Prevention:**
1. In the Format Dropdown verification phase, test Auto mode explicitly on B&O players by checking the transcoding log to confirm which format (`son-ogg`, `son-pcm`, `son-flac`) was selected.
2. If Auto incorrectly chooses OGG for a B&O player: verify UPnPBridge's `ProtocolInfo` config for those players and force `son-flac` or `son-pcm` in the player's per-player format preference.
3. This is a user-environment verification step, not a code change — the per-player format override (Format Dropdown) already exists as the escape hatch.

**Detection warning signs:**
B&O player in Auto mode: LMS selects `son-ogg` pipeline but no audio is heard. Transcoding log shows OGG stream initiated. Switching player's Format pref to FLAC or PCM resolves audio. This is not a SpotOn bug — it is a UPnPBridge capability reporting issue.

**Phase to address:** Format Dropdown B&O/Chromecast Verification.

---

### P-55: LMS Community Repo Submission — `include.json` PR Requires Active Review; Not Instant

**What goes wrong:**
Submitting to the LMS community repo requires a Pull Request to `LMS-Community/lms-plugin-repository` that adds SpotOn's `repo.xml` URL to `include.json`. This is not automated — a human maintainer (LMS-Community member) must review and merge the PR. Timeline is unpredictable. If the PR is submitted during a period of low maintainer activity, it may wait weeks.

**Additionally:** The automated `buildrepo.pl` script validates each included repository URL by fetching it. If the repo.xml URL is unreachable or malformed at PR review time, the CI build fails and the PR is blocked.

**Why it happens:**
The LMS community repo uses a human-reviewed PR model for new plugin additions (not a fully automated open submission). Existing plugins are updated automatically when their `repo.xml` URLs are already in `include.json` — but first-time inclusion requires the PR merge.

**Prevention:**
1. Submit the PR to `include.json` early in the v1.3 milestone, not as the last step. Timeline to merge is unknown.
2. Ensure the raw GitHub URL to `repo.xml` is stable and public: `https://raw.githubusercontent.com/stiefenm/spoton/main/repo.xml`. This URL is already in use for the manual repo install — it can be submitted as-is.
3. Verify the repo.xml passes the `buildrepo.pl` validation locally before submitting the PR: fetch the extensions.xml build output and check that SpotOn appears without errors.
4. The existing `repo.xml` in the project root already has the correct structure (`name`, `version`, `sha`, `url`, `category`, `target`, `minTarget`). No format changes needed for community submission.

**Common rejection or delay triggers:**
- Unreachable `repo.xml` URL at review time (GitHub outage, wrong branch).
- Mismatched SHA1 in repo.xml (SHA must be of the zip, not the plugin directory).
- Missing or incorrect `category` field (must be one of: `hardware`, `information`, `misc`, `musicservices`, `playlists`, `radio`, `scanning`, `skin`, `tools`). SpotOn is `musicservices`.

**Detection warning signs:**
CI build on the `lms-plugin-repository` fork fails at the "fetch and validate" step. Error message: URL not reachable or XML validation failed.

**Phase to address:** LMS Community Repo Submission.

---

### P-56: Spotify Extended Quota — Requirement Is 250k MAU + Legal Business Entity (Individual Developers Cannot Apply)

**What goes wrong:**
The Spotify Extended Quota Mode application (needed for the plugin to be used by more than 5 test users) requires: a legally registered business, 250,000 monthly active users, availability in key Spotify markets, and submission via a company email address. As of May 2025, Spotify no longer accepts applications from individuals. The v1.3 goal of "preparing the Extended Quota application" must account for the reality that SpotOn as a personal open-source project cannot currently meet the eligibility requirements.

**Why it happens:**
Spotify tightened Extended Quota criteria in March 2025 and made organization-only applications mandatory in May 2025. The 250k MAU threshold is a hard requirement. The review process takes up to 6 weeks. The application is evaluated by Spotify's app review team for Developer Policy compliance.

**What "prepare the application" realistically means for v1.3:**
1. Document what would be needed to apply (this pitfall is the documentation).
2. Identify if there is an organizational entity that could sponsor the application (LMS-Community GitHub org, or a foundation).
3. Do NOT submit an application under an individual name — it will be rejected immediately.
4. Instead: focus the Extended Quota phase on the ncspot bundled-token situation — confirm the ncspot/Extended Quota ID is still working, and document the contingency plan if it gets revoked.

**Consequences of wrong expectation:**
If a phase plan assumes the Extended Quota application is submitted and processed within v1.3, it will fail — the eligibility criteria alone prevent individual submission. The phase should be scoped as "research + documentation + contingency planning", not "submit and wait for approval."

**Detection warning signs:**
Spotify Developer Dashboard application form: first field asks for organizational email. Individual `sti@posteo.de` will be rejected at form submission stage.

**Phase to address:** Extended Quota preparation phase (scope to research + documentation only).

---

### P-57: Perl CI — Stub Modules Must Cover All LMS Dependencies Loaded at Compile Time

**What goes wrong:**
Adding a new Perl module (e.g., `API/Library.pm` for the Like Button) may import LMS modules at compile time that are not covered by the existing stub set in the test suite. The CI run fails with "Can't locate Slim/Utils/Foo.pm in @INC" — even on a module that is syntactically correct. The existing tests in `t/` use a sophisticated stub pattern (see `t/07_token_manager.t`) to mock LMS internals. A new module that pulls in an unstubbed dependency breaks `05_perl_syntax.t` and all downstream tests.

**Why it happens:**
The LMS Perl environment is not present in CI. The test suite maintains hand-written stubs for every LMS module that any plugin module imports. Adding a feature that uses a new LMS API (e.g., `Slim::Control::Jive` for button overlays, `Slim::Web::XMLBrowser` for inline actions) requires a corresponding stub. The compile-time check catches missing stubs before runtime.

**Prevention:**
Before implementing a new Perl module:
1. Run `perl -c NewModule.pm` locally with the stub directory in `@INC` (the test suite's `$stub_dir` pattern).
2. For each new LMS API used, add a stub to the test setup in the relevant `t/*.t` file.
3. Keep stubs minimal — AUTOLOAD-based stubs that return `1` for all method calls are sufficient for syntax/compile tests.
4. Never `use` an LMS module at file scope inside a new module without verifying the stub coverage. Prefer `require` inside sub bodies (lazy loading) for modules that are only needed at runtime — this defers the dependency to runtime and avoids compile-time failures.

**Detection warning signs:**
`prove t/05_perl_syntax.t` fails with "Can't locate Slim/..." for a module not currently in the stub set. The error pinpoints the exact missing stub.

**Phase to address:** Any phase adding new Perl modules (Like Button, CI setup, settings changes).

---

### P-58: CI for Perl Tests — `prove` Version Mismatch on Old GitHub Actions Ubuntu Runners

**What goes wrong:**
SpotOn's test suite uses `File::Temp`, `File::Path`, `MIME::Base64`, and `Digest::MD5` — all bundled with core Perl since 5.8. However, the GitHub Actions `ubuntu-latest` runner may ship with Perl 5.34 or 5.38 depending on the runner image vintage. Some test patterns that rely on `local $@` in eval blocks behave differently between Perl 5.10 (LMS floor) and Perl 5.38. If CI uses the system Perl (5.38) but the LMS target is Perl 5.10, tests can pass on CI while failing on production LMS deployments running older Perl.

**Why it happens:**
`ubuntu-latest` at time of writing ships Perl 5.38 on Ubuntu 24.04. LMS 8.0 (the floor) may run on systems with Perl 5.10-5.20. Syntax constructs like `say`, `given/when`, and `//=` work on 5.38 but `given/when` is experimental/removed on newer Perls while `//=` is fine. The gap is narrow but real.

**Prevention:**
1. The CI Perl tests workflow (when added) should pin to `perl: 5.38` for syntax checks but add a comment documenting the LMS floor is 5.10.
2. Avoid using features newer than 5.10 in plugin code: no `say` (unless `use feature 'say'` is present), no `given/when`, no `...` Yada-Yada operator.
3. The existing test files already use `use strict; use warnings;` — maintain this in all new modules.

**Detection warning signs:**
Test passes on `ubuntu-latest` (Perl 5.38) but a user reports a syntax error on LMS 8.0 (Perl 5.10). The feature likely uses a post-5.10 construct. Add a `perl -c` check against a 5.10 Perl image as an optional CI step.

**Phase to address:** CI setup (Repo Maintenance phase).

---

## Minor Pitfalls

### P-59: Like Button — Optimistic UI in OPML Has No Native "Liked" State Indicator

**What goes wrong:**
OPML menus in LMS have no native boolean toggle state (no checkbox, no star icon, no ♥ that persists across renders). Implementing a "Like" action as a menu item works, but there is no way to show the current "liked" state in the menu without a fresh API call (`GET /me/library/contains`). Checking liked state for every track in a list view would trigger a separate API call per track — guaranteed 429 at any reasonable library size.

**Why it happens:**
OPML items render from server-fetched JSON on each menu navigation. There is no client-side state cache in the LMS web UI or the Squeezebox controller. Implementing "show ♥ on liked tracks" requires either: a pre-fetched liked-ID set (expensive), or no visual indicator (just an action button that says "Like" and always appears).

**Prevention:**
Implement the Like Button as a context action only ("Add to Liked Songs" appears in the track's action menu), not as a per-track visual indicator. Do not attempt to pre-fetch and cache which tracks are liked — the API cost is prohibitive in Dev Mode. Accept that the button always says "Like" even on already-liked tracks; duplicate saves to Spotify library are idempotent (no error, no duplicate).

**Phase to address:** Like Button UX design.

---

### P-60: Connect Credential Isolation — Existing Users' Connect Dirs Must Be Migrated

**What goes wrong:**
If deployed users have an existing Connect session cached in `spoton/{accountId}/credentials.json` (the current shared path), and the v1.3 update moves Connect to `spoton/connect-{mac}/`, the Connect daemon starts fresh with no credentials in the new directory. The daemon falls back to mDNS discovery (ZeroConf), requiring the user to Connect from the Spotify app once to seed the new credentials. This is a one-time re-authentication, not a breaking failure, but it will surprise users who had persistent Connect sessions.

**Prevention:**
Document in release notes that the first Connect session after the v1.3 upgrade requires re-connecting from the Spotify app. No code migration of credential files is needed — librespot will write fresh credentials to the new directory on first connect. Do not copy or symlink the old `credentials.json` — a stale credential blob from a different account would defeat the purpose of the isolation.

**Phase to address:** Connect Credential Isolation.

---

### P-61: Repo Maintenance — Issue Templates Lock Users Into a Format; Missing "Other" Escape Hatch Causes Friction

**What goes wrong:**
GitHub issue templates with `type: required` in the form schema prevent users from filing issues that don't fit a predefined template. The LMS user population is non-technical (they may not know the difference between a "Bug Report" and a "Feature Request"). Strict templates create friction that reduces bug reports — users give up and post on the Lyrion forums instead.

**Prevention:**
Include a blank "Other / Question" template. Do not mark any template field as required. Keep the bug report template short: LMS version, Spotify account type (Premium/Free), player type, and what happened vs. what was expected. The Contributing guide should link to the Lyrion forum as the preferred discussion venue for non-bug items.

**Phase to address:** Repo Maintenance.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Connect Credential Isolation | Race between Connect daemon start and Browse --get-token reading the same credentials.json (P-49) | Separate --cache dirs: `spoton/{accountId}/` for Browse, `spoton/connect-{mac}/` for daemon |
| Like Button — API call | PUT /me/tracks is removed; wrong endpoint returns 403 (P-47) | Use PUT /me/library with URI body; verify in Client.pm |
| Like Button — auth | user-library-modify scope missing from cached token (P-48) | Bump cacheSchemaVersion; handle "Insufficient client scope" 403 distinctly |
| Like Button — rate limit | Write endpoints rate-limit after single request (P-51) | Route through existing API/Client.pm; no batch likes; test with separate Client-ID |
| Connect Volume Fix | Double-curved volume from librespot log + squeezelite linear (P-50) | Add --volume-ctrl linear to Daemon.pm @helperArgs |
| Format Dropdown verification | B&O UPnPBridge reports OGG capability but device cannot decode it (P-54) | Test Auto mode on B&O; use per-player Format pref as override |
| macOS Binaries | Gatekeeper blocks unsigned binary silently on Sequoia (P-52) | Ship with xattr workaround documented; plan notarization for future milestone |
| macOS Binaries | Single-arch instead of Universal Binary (P-53) | Use lipo on macos runner to combine aarch64 + x86_64 slices |
| Extended Quota prep | Individuals cannot apply; 250k MAU required (P-56) | Scope to documentation + contingency planning only |
| LMS Repo Submission | PR to include.json requires human review; not instant (P-55) | Submit early in milestone; repo.xml URL already correct |
| Perl CI | New module imports unstubbed LMS dependency (P-57) | Run perl -c locally with stub @INC before writing test |
| Perl CI | CI Perl version (5.38) masks 5.10-incompatible constructs (P-58) | Avoid post-5.10 features; add comment about LMS floor |
| Like Button UX | No native OPML ♥ indicator without per-track API calls (P-59) | Context-action only; duplicate saves are idempotent |
| Credential isolation upgrade | Existing cached Connect creds in old path lost on upgrade (P-60) | Document one-time re-connect in release notes |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Like Button + dual-token routing | Calling PUT /me/library via `bundled` flavor | All `me/*` paths hard-route to `own` flavor (D-05 guard). No code change needed if routed through Client.pm::_request. |
| Like Button + API body | Sending `ids=TRACK_ID` query param (old pattern) | New endpoint requires JSON body `{"uris": ["spotify:track:ID"]}`. See D-04 Content-Length requirement — body is non-empty, so Content-Length must be set correctly. |
| Connect isolation + DaemonManager | Two daemons on same player using different cache dirs simultaneously | DaemonManager tracks one Daemon per MAC. On restart, Daemon.pm reads accountId fresh from prefs — the new isolated path is computed at start() time, no stale state. |
| Connect isolation + TokenManager | TokenManager still uses `spoton/{accountId}/` for --get-token | Correct — Browse tokens must NOT use the connect-{mac} dir. Both paths are independent. |
| macOS binary + CI | Attempting to use `cross` for Darwin targets on the existing ubuntu-latest matrix | cross-rs cannot build macOS targets (P-44, v1.1 pitfall). Add a separate macos-latest matrix entry; do not extend the Linux cross job. |
| LMS repo + repo.xml SHA | SHA1 in repo.xml must be of the zip file, not a git commit hash | Use `sha1sum SpotOn-v1.3.0.zip` to compute. The existing release procedure already covers this. |

---

## "Looks Done But Isn't" Checklist

- [ ] **Like Button:** API call uses `PUT /me/library` with `{"uris": ["spotify:track:ID"]}` body — NOT `PUT /me/tracks?ids=...`
- [ ] **Like Button:** Response handles 403 "Insufficient client scope" distinctly from other 403 errors, with user-visible error message
- [ ] **Like Button:** Routes through `API/Client.pm::_request` (not direct SimpleAsyncHTTP) so rate-limit guard applies
- [ ] **Connect isolation:** `Daemon.pm::start()` computes `$cacheDir` as `spoton/connect-{$self->id}/` — NOT `spoton/{accountId}/`
- [ ] **Connect isolation:** `Plugin.pm::updateTranscodingTable` and `TokenManager.pm` still use `spoton/{accountId}/` — no regression
- [ ] **Connect volume:** `--volume-ctrl linear` appears in `@helperArgs` in `Daemon.pm::start()`
- [ ] **macOS binary:** `file Plugins/SpotOn/Bin/mac/spoton` outputs `Mach-O universal binary with 2 architectures`
- [ ] **macOS binary:** Setup guide documents quarantine removal for macOS users
- [ ] **LMS repo submission:** `repo.xml` SHA matches `sha1sum` of the release zip
- [ ] **LMS repo submission:** PR to `include.json` submitted early (not at milestone end)
- [ ] **Extended quota:** Phase scope is documentation + contingency only, no actual Spotify application submitted
- [ ] **Perl CI:** All new `.pm` files covered by `05_perl_syntax.t` compile check
- [ ] **Perl CI:** No post-5.10 Perl features in new code

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| P-47: Wrong Like endpoint | LOW | Change endpoint from `me/tracks` to `me/library`; change params from `ids=` query to `{"uris":[...]}` body |
| P-48: Missing user-library-modify scope | LOW-MEDIUM | Bump cacheSchemaVersion; add 403 scope-error handler with re-auth prompt |
| P-49: Connect credential clobber | MEDIUM | Move Daemon.pm cacheDir to `connect-{mac}/`; no migration code needed |
| P-50: Connect volume curve | LOW | Add `--volume-ctrl linear` to @helperArgs in Daemon.pm; restart daemon |
| P-51: Like write rate limit | LOW | Already handled by existing rate-limit guard; document Retry-After in logs |
| P-52: Gatekeeper blocking | LOW (docs) | Add xattr instruction to setup guide; HIGH if notarization pursued (requires $99/year Apple Developer membership) |
| P-53: Single-arch macOS binary | LOW | Add lipo step to macOS CI job |
| P-54: B&O format detection | LOW | Per-player format preference already exists as escape hatch |
| P-55: Repo PR delay | LOW | Submit PR early; nothing to recover if pending review |
| P-56: Extended quota eligibility | LOW | Re-scope phase to documentation; no application submitted |
| P-57: Missing LMS stub in CI | LOW | Add stub to test file; run prove to verify |
| P-58: Perl version mismatch | LOW | Audit new code for post-5.10 constructs; `perl -c` under 5.10 Docker image |

---

## Sources

- Spotify Feb 2026 API changelog (PUT /me/tracks removal): https://developer.spotify.com/documentation/web-api/references/changes/february-2026
- Spotify Feb 2026 migration guide (PUT /me/library): https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide
- Spotify quota modes (Extended Quota requirements): https://developer.spotify.com/documentation/web-api/concepts/quota-modes
- Spotify Extended Quota criteria update (April 2025): https://developer.spotify.com/blog/2025-04-15-updating-the-criteria-for-web-api-extended-access
- Spotify PUT /me/tracks 403 community report: https://community.spotify.com/t5/Spotify-for-Developers/403-Forbidden-on-PUT-v1-me-tracks-and-v1-me-albums-despite-user/td-p/7381748
- Spotify library write rate limits (community): https://community.spotify.com/t5/Spotify-for-Developers/Web-API-ratelimit/td-p/5330410
- Spotify scopes (user-library-modify): https://developer.spotify.com/documentation/web-api/concepts/scopes
- librespot volume control PR (--volume-ctrl options): https://github.com/librespot-org/librespot/pull/685
- librespot options wiki (--volume-ctrl default = log): https://github.com/librespot-org/librespot/wiki/Options
- librespot credentials.json world-readable issue: https://github.com/librespot-org/librespot/issues/360
- macOS Gatekeeper / notarization for CLI tools: https://dennisbabkin.com/blog/?t=how-to-get-certificate-code-sign-notarize-macos-binaries-outside-apple-app-store
- macOS Sequoia Gatekeeper changes: https://developer.apple.com/news/?id=saqachfa
- LMS plugin repository include.json: https://github.com/LMS-Community/lms-plugin-repository/blob/master/include.json
- LMS repository developer docs: https://lyrion.org/reference/repository-dev/
- SpotOn existing Daemon.pm (cacheDir computation): /home/sti/spoton/Plugins/SpotOn/Connect/Daemon.pm
- SpotOn existing Plugin.pm (updateTranscodingTable cacheDir): /home/sti/spoton/Plugins/SpotOn/Plugin.pm
- SpotOn existing API/Client.pm (D-05 me/* guard): /home/sti/spoton/Plugins/SpotOn/API/Client.pm
- SpotOn existing .github/workflows/build-librespot.yml: /home/sti/spoton/.github/workflows/build-librespot.yml

---

*Pitfalls research for: SpotOn v1.3 — Polish & Publish (Connect Credential Isolation, Like Button, Volume Fix, Format Verification, Extended Quota, Repo Maintenance, macOS Binaries, LMS Repo Submission)*
*Researched: 2026-06-06*
