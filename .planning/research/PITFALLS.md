# Domain Pitfalls: SpotOn LMS Spotify Plugin

**Domain:** LMS Spotify plugin (Perl + Rust binary + Spotify Web API)
**Researched:** 2026-05-26
**Scope:** NEW pitfalls ONLY — P-01 through P-20 from REQUIREMENTS.md are excluded

---

## Critical Pitfalls

Mistakes that cause rewrites, broken production installs, or silent data corruption.

---

### P-21: Keymaster 403 — Silent Elimination of the Fallback Strategy

**What goes wrong:** The `hm://keymaster/token/authenticated` endpoint used in librespot for obtaining spclient access tokens started returning 403 "Invalid request" errors in August 2025, breaking all librespot installations for 24+ hours. The librespot team migrated to login5 (HTTP endpoint `https://login5.spotify.com/v3/login`) in PR #1344. If SpotOn ships a binary fork still using keymaster-based token acquisition (pre-0.6.0 approach), any future Spotify backend change could silently break Keymaster-only auth with no fallback.

**Why it happens:** Keymaster is an internal Spotify Mercury protocol endpoint — not documented, not guaranteed. Spotify has already killed it once. The fix is now login5, but login5 itself is also undocumented and subject to change.

**Consequences:** Entire plugin goes dark. No browse, no Connect, no auth. Because this is the Keymaster-only auth path (the core SpotOn architectural decision), there is no PKCE fallback. Users see 403 errors with no actionable error message.

**Prevention:**
- Pin the binary to a librespot version >= 0.6.0 that uses login5 for token acquisition.
- Monitor librespot issues and releases; subscribe to `librespot-org/librespot` releases on GitHub.
- Log the token acquisition method and endpoint in DEBUG output so users can report which step fails.
- Design `Auth.pm` to separate "token obtained from binary" vs "token obtained via PKCE" as distinct code paths, even if only one is used at runtime. This makes adding a PKCE emergency fallback a 1-day task, not a rewrite.

**Detection:** Watch for `403` responses on any Spotify API call immediately after fresh daemon start; if the daemon cannot obtain its own token on first connect, the issue is authentication, not an API endpoint.

**Phase:** Authentication phase (Phase 1 / Core Auth).

---

### P-22: Spotify Web API Endpoint Removals in Development Mode (February 2026)

**What goes wrong:** Spotify's February 2026 changes removed or restricted a large set of endpoints for apps in Development Mode (the default for new apps). SpotOn will be in Development Mode. Removed endpoints that directly affect planned features:

- `GET /artists/{id}/top-tracks` — used in NAV-06 (Artist Detail Page)
- `GET /browse/new-releases` — used in NAV-02 / NAV-05 (Home Feed, Browse)
- `GET /browse/categories` — used in NAV-05 (Browse Categories)
- Batch fetch endpoints `GET /tracks`, `GET /albums`, `GET /artists` — must fetch individually
- `GET /users/{id}` — affects playlist creator attribution
- Search limit reduced from 50 to 10 per request — pagination now required for any substantive search result

**Why it happens:** Spotify has been restricting the Web API to prevent AI scraping and reduce infrastructure load. Development Mode apps (the only path for open-source projects since May 2025) bear the full brunt of these restrictions. Extended Quota Mode requires 250k MAU, commercial viability, and organizational registration — impossible for an open-source LMS plugin.

**Consequences:** Features designed against the full API spec silently fail or return empty results in production. "Artist Top Tracks" as a menu item will 404. Browse Categories will be empty. New Releases will be missing.

**Prevention:**
- Before coding any API call in `API/Browse.pm` or `API/Library.pm`, verify the endpoint works in Development Mode with a live test.
- For removed endpoints, design fallbacks: "Artist Top Tracks" can fall back to the artist's albums list. "New Releases" can fall back to "Featured Playlists". Document which menu items are Development-Mode-degraded vs full-featured.
- Search code must never assume more than 10 results per request — use offset pagination from day one.
- Maintain a compatibility table in `API/Browse.pm` comments: which endpoints are Development-Mode-available vs removed.

**Detection:** Any `404` on a Browse endpoint in development should be checked against the February 2026 migration guide before debugging the plugin code.

**Phase:** API / Browse phase (Phase 2 / Navigation).

---

### P-23: OAuth Redirect URI Breakage — `localhost` is Prohibited

**What goes wrong:** As of November 27, 2025, Spotify removed support for `localhost` as a redirect URI hostname. Apps using `http://localhost:PORT/callback` as their redirect URI will receive `INVALID_CLIENT: Insecure redirect URI` errors. The `lms-community` OAuth relay (documented in P-07) already had a `state` parameter stripping bug. If SpotOn's PKCE fallback or any auth setup uses `localhost` in the redirect URI, auth will silently break on a Spotify-side enforcement date.

**Why it happens:** Spotify's security hardening considers `localhost` an alias that can be hijacked; they still allow the explicit loopback IP `http://127.0.0.1:PORT` under the loopback exception.

**Consequences:** Any user who hasn't yet authenticated gets a cryptic Spotify error page. No migration path without code change.

**Prevention:**
- Use `http://127.0.0.1:PORT/callback` exclusively in all auth flows. Never construct redirect URIs with `localhost`.
- If relying on the lms-community relay, verify the relay itself uses `127.0.0.1` internally and does not inject `localhost`.
- In `API/Auth.pm`, derive the redirect URI from the LMS server's bound address, not from a hardcoded `localhost` string.

**Detection:** `INVALID_CLIENT` error in auth flow = check the redirect URI construction first.

**Phase:** Authentication phase.

---

### P-24: HTTP Audio Transport — LMS Client Disconnect on Seek Kills the Stream

**What goes wrong:** When LMS receives a seek request on an HTTP internet-radio style stream (the proposed HTTP Connect transport), it closes and re-opens the HTTP connection to the audio source. For a standard internet radio URL this is fine — the remote server accepts new connections. For the embedded HTTP server in the librespot binary, this reconnect hits the same localhost server that was serving the previous connection. If the server implementation does not handle rapid close/reopen cleanly (e.g., `TIME_WAIT` on the port, or the server only accepts one client at a time), the new LMS connection fails and audio stops.

**Why it happens:** LMS treats HTTP audio streams like internet radio and its seek/reconnect model assumes a stateless server. The librespot HTTP server is stateful — it has a specific audio position and session. The server must explicitly handle "new client = resume from current position" or "new client replaces previous client."

**Consequences:** Seek in Spotify app → LMS drops audio → Connect session appears to break → User sees the track position jump but hears silence.

**Prevention:**
- The HTTP server in the binary must be designed for single-client-at-a-time with explicit client replacement: when a new GET arrives while audio is streaming, close the previous connection cleanly before accepting the new one.
- Add a small listen backlog (e.g., 2) so the new LMS connection is accepted even if the old connection is still in `TIME_WAIT`.
- Use different port per daemon (see P-25) so there is no port reuse contention between multiple players' HTTP servers.
- Validate with a seek stress test (rapid seek 5+ times in 10 seconds) before declaring HTTP transport stable.

**Detection:** Silence after seek but ProgBar showing correct position = HTTP transport reconnect failure.

**Phase:** Connect / HTTP Transport phase.

---

### P-25: Multiple HTTP Server Instances — Port Collision Between Player Daemons

**What goes wrong:** SpotOn runs one librespot daemon per LMS player. If each daemon starts an embedded HTTP server on a hardcoded port (e.g., 24879), the second daemon to start will fail to bind — `Address already in use` — and silently fall back to no audio output or crash. With 4+ players, port allocation becomes a coordination problem.

**Why it happens:** Each librespot daemon is an independent process. Without explicit port assignment from the plugin, each daemon will either pick the same default port or pick a random port — either way the Perl plugin does not know which port to tell LMS to connect to.

**Consequences:** Only the first Connect daemon works. Other players appear in Spotify but produce no audio. Errors are logged at the Rust binary level, not visible in LMS logs unless the plugin reads the daemon's stderr.

**Prevention:**
- `Connect/Manager.pm` must assign each daemon a unique port from a configurable pool (e.g., 24880 + player index).
- Pass the assigned port as a CLI argument to the binary and record the mapping `player_id → port` in the Manager.
- Expose the port mapping via `pluginData` so `ProtocolHandler.pm` can construct the correct streaming URL per player.
- On daemon start, read the daemon's stderr for "port already in use" errors explicitly and retry with the next available port.

**Detection:** Second player shows as Connect device in Spotify app but plays silence; daemon stderr contains "AddrInUse" or similar OS error.

**Phase:** Connect / Daemon lifecycle phase.

---

### P-26: librespot Long-Running Daemon Token Expiry — Session Dies After 1 Hour

**What goes wrong:** When librespot authenticates via an OAuth access token (the mechanism Keymaster-only auth uses), it stores only the access token — not the refresh token or expiry timestamp. After ~1 hour, the access token expires. The session becomes unusable for API calls (even though the Connect audio stream itself may continue). The daemon continues to run but cannot fetch track metadata, update playback state, or respond to Spirc commands correctly. This was filed as librespot issue #1377, opened October 2024, with no confirmed fix as of research date.

**Why it happens:** The librespot session initialization path for OAuth discards the `TokenResponse` object after extracting only the access token. There is no proactive refresh logic in the session layer for this case.

**Consequences:** After ~1 hour of uptime: "Now Playing" metadata stops updating in Spotify app, volume commands may stop working, track transitions become unreliable. The daemon does not crash — it silently degrades. Restart of the daemon fixes it, but that interrupts the Connect session.

**Prevention:**
- Track daemon uptime in `Connect/Daemon.pm`. If uptime approaches 50 minutes, proactively restart the daemon with a fresh token before the session expires.
- Alternatively, implement a "token renew" signal from the Perl plugin to the daemon (requires binary-side support).
- Log a WARN at 45 minutes of uptime so the issue is visible.
- The restart strategy must preserve the Connect session gracefully: notify LMS before restarting so it can reconnect, not drop audio mid-track.

**Detection:** Connect session works for first hour then becomes unresponsive to Spirc commands without daemon crash.

**Phase:** Connect / Daemon lifecycle phase.

---

## Moderate Pitfalls

---

### P-27: SimpleAsyncHTTP HTTPS Blocking — SSL Handshake Freezes LMS

**What goes wrong:** `Slim::Networking::SimpleAsyncHTTP` is documented as async but has a known blocking path during SSL/TLS handshakes, particularly on systems with older or incompatible SSL libraries. A slow or failing HTTPS server (Spotify API, CDN for artwork) can freeze the entire LMS process — blocking the web UI, IR remote, player control — for 30+ seconds. This is filed as LMS issue #261 and is an architectural limitation of the networking layer.

**Why it happens:** The async networking implementation blocks the main Perl event loop during the TLS negotiation phase. Since LMS runs single-threaded, everything stalls.

**Consequences:** LMS becomes completely unresponsive when a Spotify API call hangs. Other plugins, players, and the web UI are all affected.

**Prevention:**
- Always set explicit timeouts on `SimpleAsyncHTTP` requests. The timeout must be set before the request fires, not after.
- In `API/Client.pm`, wrap every outgoing request with a `Slim::Utils::Timers::setTimer` failsafe that fires if the request callback is not called within N seconds (recommend 10s for API, 20s for binary spawn).
- For artwork fetching (which hits CDNs that may be slow), use shorter timeouts and degrade gracefully to a missing-artwork state rather than blocking.
- Test on the oldest supported LMS version with SSL to verify no regression.

**Detection:** All LMS activity freezes at the exact moment a Spotify API call fires.

**Phase:** API / Client layer phase.

---

### P-28: PCM Frame Size Mismatch in HTTP Audio Stream — White Noise on FLAC Wrapping

**What goes wrong:** The HTTP audio transport delivers raw PCM (S16LE, 44100 Hz, 2ch = 4 bytes/frame). If the HTTP server sends a partial final chunk that is not aligned to a frame boundary (not a multiple of 4 bytes), and LMS pipes that into `flac -cs`, FLAC will wrap a misaligned PCM stream. The FLAC decoder on the player side will produce white noise or click artifacts — identical in symptom to P-20 (FIFO reconnect white noise) but from a different cause.

**Why it happens:** TCP is a stream protocol. When the HTTP server flushes audio chunks, there is no guarantee that chunk boundaries align to PCM frame boundaries. This is a correctness requirement on the server side that is easy to overlook.

**Consequences:** Intermittent white noise at track boundaries or on reconnect. Extremely difficult to diagnose without PCM-level inspection.

**Prevention:**
- The HTTP server in the binary must flush only complete frames. Maintain a 4-byte alignment buffer: if the last chunk would leave the stream at a non-multiple-of-4 byte count, pad with silence or delay the flush.
- In the LMS integration, declare the audio source as FLAC (not PCM) only if the HTTP server guarantees aligned output; otherwise use PCM passthrough and let LMS handle transcoding from a known-good byte stream.
- Add a PCM frame alignment validation step in the binary's HTTP sink before any flush.

**Detection:** Periodic single-frame clicks or bursts of white noise at track transitions, not reproducible consistently.

**Phase:** Connect / HTTP Transport phase.

---

### P-29: librespot Binary Version Skew — `--check` Capabilities Schema Changes Between Versions

**What goes wrong:** SpotOn (like Spotty-NG, per P-10) relies on `--check` JSON output from the binary to detect capabilities. librespot v0.8.0 introduced a new `SpotifyUri` type and removed `SpotifyItemType` enum — internal Rust API changes. Between major versions, the binary's command-line interface has also evolved (new flags added, options renamed). If the plugin ships with a binary but expects a `--check` schema from an older version, capability detection silently fails and features are misconfigured.

**Why it happens:** The `--check` or capability-probe mechanism is not part of any stable API contract. It is a convention established by Herger's fork. librespot upstream has no such interface; it must be implemented in the custom fork.

**Consequences:** Wrong bitrate, wrong format negotiation, or Connect accidentally disabled because a capability flag changed name.

**Prevention:**
- `Helper.pm`'s capability detection must use `defined($caps->{key}) ? $caps->{key} : $default` — never die on missing keys (already noted in P-10, but the specific trigger here is cross-version schema drift, not just "key might be missing").
- Tag every binary release with the exact librespot base version in its `--version` output, and log it at INFO on plugin init.
- Maintain a minimum required binary version in `Plugin.pm`. If the binary's reported version is below the minimum, log a WARN and disable Connect rather than proceeding with misconfigured capabilities.

**Detection:** Features silently not working after binary update; DEBUG log shows missing capability keys.

**Phase:** Binary integration phase.

---

### P-30: mDNS ZeroConf Port Binding — Default Port 0 Collisions Across Daemons

**What goes wrong:** librespot's ZeroConf discovery server binds to `0.0.0.0` by default. When multiple daemons run on the same host without explicit `--zeroconf-port` assignment, whichever daemon starts second gets an `EADDRINUSE` error on the ZeroConf port. The daemon may still start (depending on librespot version), but the Connect device will not be discoverable via mDNS — it will only be reachable via the Spotify Web API's device list, which is a slower and less reliable path.

**Why it happens:** librespot's default ZeroConf port is either hardcoded or randomly selected. Without coordination between instances, collisions are inevitable in a multi-player LMS setup.

**Consequences:** Some Connect devices are discoverable via the Spotify app (via mDNS), others are not. Inconsistent UX: Player A shows in Spotify app immediately, Player B requires switching to "web" device list.

**Prevention:**
- `Connect/Manager.pm` must assign both the audio HTTP port (P-25) and a unique ZeroConf port per daemon. Allocate from separate port pools.
- Pass `--zeroconf-port XXXX` to every daemon invocation.
- Verify the ZeroConf port is not already in use before passing it to the daemon (use a brief `IO::Socket::INET` bind test in Perl).

**Detection:** Spotify app shows some LMS players but not others; log shows `EADDRINUSE` in daemon stderr.

**Phase:** Connect / Daemon lifecycle phase.

---

### P-31: Prefs System — `$prefs->client($client)` Persists Detached-Client Data

**What goes wrong:** `Slim::Utils::Prefs::client($client)` returns a preferences object for the client, including clients that are not currently connected ("stored clients"). If SpotOn reads a player pref at startup for all known players (to start Connect daemons), it will start daemons for players that are powered off, disconnected, or gone forever. The `_startTimes` tracking in `Connect/Manager.pm` will then accumulate restart-counter debt for these ghost daemons.

**Why it happens:** LMS persists client preferences indefinitely. The plugin has no built-in way to distinguish "player that will reconnect" from "player that was replaced three years ago".

**Consequences:** Zombie daemon entries, incorrect sync-group state, port pool exhaustion on LMS instances with many historical players.

**Prevention:**
- Only start Connect daemons for players that are currently registered as `connected` in LMS (`$client->connected()`).
- Subscribe to `client_connected` and `client_disconnected` LMS events to dynamically start/stop daemons rather than initializing all known players at boot.
- Prune the port allocation table of players not seen for > 30 days.

**Detection:** After LMS restart, more daemon processes appear in `ps` than there are connected players.

**Phase:** Connect / Daemon lifecycle phase.

---

### P-32: Transcoding Table Race — Parallel Track Start Overwrites PCM Spec

**What goes wrong:** The LMS transcoding table is global state (per P-09 in REQUIREMENTS.md — do not repeat, but this is the related *race* variant). If two players start Spotify tracks within the same server tick, both will call `updateTranscodingTable` nearly simultaneously. The second write overwrites the first player's cache-path injection before the first player's `StreamingController` has consumed the value. Player 1 starts playing Player 2's audio cache path.

**Why it happens:** LMS is single-threaded (Perl + event loop), but "simultaneous" here means two timer callbacks firing in the same event loop iteration — which is possible. The transcoding table mutation is not atomic at the application level.

**Consequences:** Cross-player audio contamination (Player A plays the track Player B just started) or silent failure on one player.

**Prevention:**
- Do not use global transcoding table mutation at all for per-player state. Instead, encode the player-specific parameters (cache dir, bitrate) in the `spotify://` URI itself, and parse them out in `ProtocolHandler::getURL`. This is the architecturally clean solution that P-09 hints at.
- If table mutation is unavoidable (legacy compatibility), acquire a Perl-level mutex via a plugin-global `$_transcoding_lock` flag and defer the second write.

**Detection:** Player A occasionally plays a different track than the one in its "Now Playing" display.

**Phase:** Protocol Handler / Transcoding phase.

---

### P-33: DLNA/UPnP Players via UPnPBridge — OGG Claimed But Not Played

**What goes wrong:** UPnPBridge (used for Bang & Olufsen devices and other DLNA players in the test environment) negotiates audio formats by reading the player's DLNA profile. Some devices report OGG support in their DLNA capabilities but refuse to play it in practice (documented behavior for Sonos; likely for B&O as well). If SpotOn's OGG-Direct mode is enabled and UPnPBridge routes the OGG stream to such a device, the player goes silent without any error.

**Why it happens:** DLNA capability negotiation is notoriously unreliable. Device firmware reports formats it "supports" in some configuration but not in the actual playback pipeline.

**Consequences:** OGG-Direct mode causes silence on DLNA players that would work fine with FLAC transcoding.

**Prevention:**
- OGG-Direct mode must have a per-player override that defaults to OFF for DLNA/UPnP players.
- In `Helper.pm` capability detection, check if the player type is `Slim::Player::SqueezePlay` vs a UPnPBridge proxy client; disable OGG-Direct for the latter by default.
- Document in Settings UI that OGG-Direct is for native Squeezebox players only and must be manually verified for DLNA players.

**Detection:** Enabling OGG-Direct causes silence on specific players; switching to FLAC restores audio.

**Phase:** Transcoding / Player compatibility phase.

---

## Minor Pitfalls

---

### P-34: Spotify Search Pagination — `offset` Arithmetic at New 10-Item Limit

**What goes wrong:** With the February 2026 reduction of the search `limit` parameter maximum from 50 to 10, any pagination code that was written with `limit=50` as a base must be rewritten. More subtly, any code that uses `total / limit` to calculate page count will produce incorrect results if `limit` is dynamically chosen but the server silently caps it at 10. The server may return 10 items when 50 were requested, and if the code does not re-read the `limit` from the response object (vs the request parameter), pagination terminates prematurely.

**Prevention:**
- Always use the `limit` field from the response object (what Spotify actually returned) for `offset` calculations, not the limit that was requested.
- Set `limit=10` explicitly in all search calls to Development Mode apps; do not rely on defaults.

**Phase:** API / Browse phase.

---

### P-35: install.xml `maxVersion` — Plugin Silently Disappears After LMS Upgrade

**What goes wrong:** If `install.xml` includes a `maxVersion` field with a specific LMS version ceiling (e.g., `maxVersion="9.0.0"`), the plugin will vanish from the plugin list after users upgrade to LMS 9.x — without any error, just absent from the UI.

**Prevention:**
- Use `maxVersion="*"` for all `install.xml` declarations unless there is a known API incompatibility.
- Test install.xml parsing against current LMS `Slim::Utils::PluginManager` to verify the plugin is loaded.

**Phase:** Plugin scaffold phase.

---

### P-36: Binary SHA256 Checksum Verification — Missing or Mismatch Silently Uses Stale Binary

**What goes wrong:** Multi-architecture binary distribution puts pre-built binaries in `Bin/`. If a user's LMS auto-update mechanism partially updates the plugin (downloads new `.pm` files but not new binaries, or vice versa), the plugin runs with a new Perl layer expecting new binary capabilities against an old binary. The mismatch may not produce an error — the old binary just returns unknown-key JSON from `--check` and the plugin degrades silently.

**Prevention:**
- Ship a `sha256sums.txt` alongside every binary. `Helper.pm` must verify the SHA256 of the binary before executing it.
- If checksum mismatch is detected, log an ERROR and refuse to start Connect (do not silently run with potentially broken binary).
- Use `Digest::MD5` (available in LMS) as a fallback if `Digest::SHA` is not available.

**Phase:** Binary distribution phase.

---

### P-37: librespot Session Reconnect After Network Change / Sleep

**What goes wrong:** librespot does not handle network interface changes gracefully (confirmed issue #1627). After a host sleeps, resumes, or switches networks, the librespot daemon continues to run but loses its connection to Spotify's backend. The daemon does not exit — it loops in a reconnect state. The LMS plugin sees a running process (PID alive) and assumes the daemon is healthy. Connect devices disappear from the Spotify app but the plugin reports them as active.

**Prevention:**
- `Connect/Daemon.pm` must monitor daemon health beyond PID presence. Poll the daemon's health via its HTTP endpoint (if HTTP transport is implemented) or via a dedicated heartbeat.
- Alternatively, set a maximum daemon lifetime (e.g., 6 hours) and proactively restart with fresh credentials, accepting the brief Connect session interruption.
- Subscribe to LMS system events for network change notifications if available (`Slim::Networking::Discovery`).

**Detection:** Daemons are running but devices disappear from Spotify app; daemon stderr shows reconnect loop entries.

**Phase:** Connect / Daemon lifecycle phase.

---

### P-38: Spotify API Rate Limit Is App-Wide, Not Per-User

**What goes wrong:** The Spotify Web API rate limit is a rolling 30-second window scoped to the entire app (Client ID), not per-user or per-endpoint. SpotOn's central throttle (NFL-03) is correct in principle, but if multi-account support is implemented naively — each account using the same Client ID — concurrent users on the same LMS instance share one rate limit bucket. Five users each triggering a Library paginate could combine into a 429 burst even if each individual user's request rate looks safe.

**Prevention:**
- The central throttle in `API/Client.pm` must apply globally per Client ID, not per `$userId`.
- If multi-account support is implemented, consider whether each account should use a separate Client ID (and separate rate bucket) or share one with a proportionally lower per-account limit.

**Detection:** 429 errors occur under multi-user load but not under single-user testing.

**Phase:** API / Client layer phase.

---

### P-39: credentials.json World-Readable File Permission

**What goes wrong:** librespot creates `$cache/credentials.json` with world-readable permissions (644) by default (librespot issue #360). On a shared LMS host, other system users can read the stored Spotify authentication token. The file contains a reusable session credential, not a password — but it grants Spotify account access until rotated.

**Prevention:**
- `Connect/Daemon.pm` must `chmod 600` the credentials file after the daemon writes it.
- Configure each daemon's `--cache` to a per-player subdirectory under a plugin-private directory (e.g., `$LMS_CACHE/SpotOn/daemons/$playerid/`) with `0700` permissions on the directory.
- Document in the README that the cache directory should not be world-readable.

**Phase:** Connect / Daemon lifecycle phase.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|---|---|---|
| Authentication (Keymaster-only) | P-21: Keymaster 403 | Pin to librespot >= 0.6.0 (login5); monitor librespot releases |
| Authentication (PKCE fallback) | P-23: `localhost` redirect URI | Use `http://127.0.0.1` only |
| Browse / API endpoints | P-22: Removed endpoints (Dev Mode) | Verify every endpoint against Dev Mode restrictions before implementing |
| API client layer | P-27: SimpleAsyncHTTP HTTPS blocking | Explicit timeouts on every request; timer failsafe |
| API client layer | P-38: Rate limit is app-wide | Central throttle = global per Client-ID, not per user |
| Connect daemon lifecycle | P-25: Port collision (HTTP + ZeroConf) | Manager assigns unique ports; P-30 for ZeroConf |
| Connect daemon lifecycle | P-26: Token expiry after 1 hour | Proactive daemon restart at 50 min uptime |
| Connect daemon lifecycle | P-31: Ghost clients | Only start daemons for `$client->connected()` players |
| Connect daemon lifecycle | P-37: Network change / sleep reconnect | Health check beyond PID presence |
| Connect daemon lifecycle | P-39: credentials.json permissions | chmod 600 + private cache dir |
| HTTP audio transport | P-24: LMS disconnect on seek | Binary HTTP server must handle single-client-replace semantics |
| HTTP audio transport | P-28: PCM frame alignment | Enforce 4-byte flush alignment in HTTP sink |
| Transcoding / Format | P-32: Transcoding table race | Encode player-specific params in URI, not global state |
| Transcoding / Format | P-33: DLNA OGG mismatch | OGG-Direct OFF by default for UPnPBridge players |
| Binary distribution | P-29: `--check` schema drift | Version-gated capability parsing; min binary version check |
| Binary distribution | P-36: SHA256 mismatch | Verify checksum before binary execution |
| Search / Pagination | P-34: Offset arithmetic at limit=10 | Read `limit` from response, not request |
| Plugin manifest | P-35: maxVersion ceiling | Use `maxVersion="*"` |

---

## Sources

- librespot keymaster 403: https://github.com/librespot-org/librespot/issues/1532
- librespot login5 migration: https://github.com/librespot-org/librespot/pull/1220
- librespot token expiry issue: https://github.com/librespot-org/librespot/issues/1377
- librespot v0.8.0 release: https://github.com/librespot-org/librespot/releases/tag/v0.8.0
- Spotify Web API February 2026 changes: https://developer.spotify.com/documentation/web-api/references/changes/february-2026
- Spotify February 2026 migration guide: https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide
- Spotify redirect URI requirements: https://developer.spotify.com/documentation/web-api/concepts/redirect_uri
- Spotify OAuth security changes 2025: https://developer.spotify.com/blog/2025-02-12-increasing-the-security-requirements-for-integrating-with-spotify
- Spotify quota modes: https://developer.spotify.com/documentation/web-api/concepts/quota-modes
- Spotify extended access criteria: https://developer.spotify.com/blog/2025-04-15-updating-the-criteria-for-web-api-extended-access
- LMS SimpleAsyncHTTP HTTPS blocking: https://github.com/LMS-Community/slimserver/issues/261
- librespot ZeroConf port binding: https://github.com/librespot-org/librespot/issues/887
- librespot credentials world-readable: https://github.com/librespot-org/librespot/issues/360
- librespot network change reconnect: https://github.com/librespot-org/librespot/issues/1627
- PCM transcoding metadata mismatch: https://github.com/LMS-Community/slimserver/issues/347
- librespot client disconnect handling: https://github.com/librespot-org/librespot/issues/345
