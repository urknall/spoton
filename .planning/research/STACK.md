# Technology Stack: SpotOn (LMS Spotify Plugin)

**Project:** SpotOn — Spotify plugin for Lyrion Music Server
**Researched:** 2026-05-26
**Mode:** Ecosystem

---

## Recommended Stack

### Core Framework

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Perl | >= 5.10 (LMS floor); LMS 9.x ships ~5.38 | Plugin language | LMS plugins are Perl modules under `Slim::Plugin::*`. No alternative. |
| LMS Plugin API | LMS 8.0+ (floor), 8.5.1+ (full), 9.1.1 (latest stable) | Plugin framework | OPMLBased, SimpleAsyncHTTP, Cache, Prefs — this IS the framework. |
| librespot | 0.8.0 (Nov 2024, latest stable) | Spotify streaming + Connect receiver | Only open-source Spotify streaming implementation; Shannon protocol for lossy. |
| Spotify Web API v1 | Current — see endpoint status table below | Browse, Search, Library, Player State | No alternative; OAuth 2.0 required. |

### LMS Plugin API Modules

All bundled in LMS. No CPAN installation required.

#### `Slim::Plugin::OPMLBased` — Menu Framework
**Confidence: HIGH** (verified against LMS source)

The base class for all menu-driven plugins. Handles CLI dispatch, web pages, and Jive menu registration.

```perl
# Plugin.pm — Registration
sub initPlugin {
    my $class = shift;
    $class->SUPER::initPlugin(
        feed    => \&toplevel,      # callback returning OPML items
        tag     => 'spoton',        # unique tag for CLI dispatch
        menu    => 'radios',        # or 'apps'
        is_app  => 1,
        weight  => 100,
        icon    => 'html/images/icon.png',
    );
}

# OPML item structure
sub toplevel {
    my ($client, $callback, $args) = @_;
    $callback->([
        { name => 'Home',    type => 'link', url => \&homeMenu },
        { name => 'Search',  type => 'search', url => \&searchMenu },
        { name => 'Library', type => 'link', url => \&libraryMenu },
    ]);
}
```

OPML item types: `link` (submenu), `audio` (playable track), `search` (text input), `text` (informational). No grid, no tabs — strictly hierarchical tree.

`condition()` method controls dynamic visibility. `weight` controls sort order (lower = higher in list).

#### `Slim::Networking::SimpleAsyncHTTP` — Non-Blocking HTTP
**Confidence: HIGH** (verified against LMS source)

**CRITICAL RULE:** All HTTP calls in the plugin server context MUST use SimpleAsyncHTTP. LMS is single-threaded; any blocking call freezes audio playback.

```perl
use Slim::Networking::SimpleAsyncHTTP;

sub fetchTrack {
    my ($uri, $successCb, $errorCb) = @_;
    my $http = Slim::Networking::SimpleAsyncHTTP->new(
        sub {
            my $http = shift;
            my $data = $http->content();
            # parse JSON, invoke $successCb
        },
        sub {
            my $http = shift;
            # handle error, invoke $errorCb
        },
        {
            cache   => 1,
            expires => 300,         # seconds; built-in response caching
            timeout => 30,
        }
    );
    $http->get('https://api.spotify.com/v1/tracks/...',
        'Authorization', "Bearer $token",
        'Accept', 'application/json',
    );
}

# POST with body and headers:
$http->post($url,
    'Content-Type', 'application/json',
    'Authorization', "Bearer $token",
    encode_json($body_hashref),
);
```

Key methods on the response object: `content()`, `code()`, `headers()`, `params($key)`.

The `cache`/`expires` options are built-in. Use them for metadata (300s), browse results (300s), and static data (3600s). **Do NOT rely on this for the API throttle** — that must be a separate component (see Architecture).

**Note on DELETE/PUT:** SimpleAsyncHTTP supports GET and POST natively. For DELETE/PUT needed by some Spotify API endpoints (library management), use the underlying `Slim::Networking::Async::HTTP` class directly, or construct the request manually via `$http->_createHTTPRequest`. In practice, all SpotOn read operations use GET; write operations (save/delete tracks) can use POST with `_method` override if necessary, or the new unified `PUT /me/library` endpoint.

#### `Slim::Networking::SimpleSyncHTTP` — Blocking HTTP (Scanner only)
**Confidence: HIGH**

```perl
# ONLY for Importer.pm scanner context:
my $http = Slim::Networking::SimpleSyncHTTP->new();
$http->get($url);
if ($http->is_success()) {
    my $content = $http->content();
}
```

**Never use in the main server.** The module itself warns: "DO NOT USE this class in the server. It's supposed to be used in the scanner only."

#### `Slim::Utils::Cache` — Response Caching
**Confidence: HIGH** (verified against LMS source)

SQLite-backed persistent cache. Keys are hashed internally; use MD5 of the full key to avoid collision (P-08 from REQUIREMENTS.md).

```perl
use Slim::Utils::Cache;

my $cache = Slim::Utils::Cache->new('plugin.spoton', 1);

# Set with TTL (seconds):
$cache->set('track:4uLU6hMCjMI75M1A2tKUQC', $track_data, 300);

# Get:
my $data = $cache->get('track:4uLU6hMCjMI75M1A2tKUQC');

# Delete:
$cache->remove('track:4uLU6hMCjMI75M1A2tKUQC');
```

TTL guidance based on data volatility:
- Player/playback state: 0 (never cache, always live)
- Library items (liked songs): 60s
- Artist/album/track metadata: 3600s
- Playlist tracks: 300s (snapshot_id invalidation recommended)
- Browse/category data: 300s

#### `Slim::Utils::Prefs` — Preferences
**Confidence: HIGH** (verified against LMS source)

```perl
use Slim::Utils::Prefs;

my $prefs = preferences('plugin.spoton');

# Initialize with defaults:
$prefs->init({
    bitrate         => 320,
    normalisation   => 0,
    connect_enabled => 1,
    account         => '',
});

# Global read/write:
my $bitrate = $prefs->get('bitrate');
$prefs->set('bitrate', 160);

# Per-player preferences:
my $playerPrefs = $prefs->client($client);
my $playerBitrate = $playerPrefs->get('bitrate') // $prefs->get('bitrate');
$playerPrefs->set('connect_enabled', 0);

# Validation:
$prefs->setValidate({ validator => 'intlimit', low => 96, high => 320 }, 'bitrate');

# Change callback:
$prefs->setChange(\&onBitrateChange, 'bitrate');
```

Prefs are saved as YAML in `.prefs` files. Migration via `$prefs->migrate($version, \&migrationSub)`.

#### `Slim::Plugin::OnlineLibraryBase` — Library Import
**Confidence: MEDIUM** (documented in LMS source, less battle-tested than the above)

Base class for syncing online music service libraries into the LMS local database. Required if implementing NAV-10 (Online Library Importer).

```perl
# Required methods:
sub trackUriPrefix { return 'spotify:track:' }
sub isImportEnabled { return $prefs->get('enable_import') }
```

The base class handles `storeTracks()`, `deleteRemovedTracks()`, and artist name normalization. Works with `Slim::Networking::SimpleSyncHTTP` in scanner context.

#### `Slim::Utils::Log` — Structured Logging
**Confidence: HIGH** (standard LMS pattern, used in all plugins)

```perl
use Slim::Utils::Log;

my $log = Slim::Utils::Log->addLogCategory({
    category     => 'plugin.spoton',
    defaultLevel => 'WARN',
    description  => 'SpotOn Spotify Plugin',
});

# Usage:
$log->error("Token refresh failed: $err");
$log->warn("API returned 429, backing off ${retry}s");
$log->info("Connect daemon started for $player");
if ($log->is_debug) {
    $log->debug("Full API response: " . encode_json($data));
}
```

Levels: `error`, `warn`, `info`, `debug`. Check `$log->is_debug` before expensive serialization.

#### `Slim::Control::Request` — Event Subscription + Command Dispatch
**Confidence: HIGH**

```perl
use Slim::Control::Request;

# Subscribe to player events:
Slim::Control::Request::subscribe(
    \&_onNewSong,
    [['playlist'], ['newsong']]
);
Slim::Control::Request::subscribe(
    \&_onPlaylistChange,
    [['playlist'], ['stop', 'clear', 'play', 'pause']]
);

# Unsubscribe:
Slim::Control::Request::unsubscribe(\&_onNewSong);

# Execute a command on a player:
Slim::Control::Request::executeRequest($client, ['playlist', 'play', $url]);

# Register custom CLI command:
Slim::Control::Request::addDispatch(
    ['spoton', '_cmd'],
    [1, 0, 0, \&handleCLI]
);
```

Event callback signature: `sub _onNewSong { my $request = shift; my $client = $request->client(); }`.

**P-13 warning:** Never use `['time', N]` on a stream-mode client — it triggers a full stream restart. Use `$song->startOffset($position)` instead.

**P-15 warning:** `_onNewSong` fires synchronously from `playlist play`. Store `pluginData(progress => $position)` BEFORE issuing `playlist play`.

### Bundled CPAN Modules Available

These are in the LMS `CPAN/` directory — no external installation needed:

| Module | Use Case |
|--------|----------|
| `JSON::XS` | JSON encoding/decoding (fast XS version) |
| `LWP` | HTTP (use SimpleAsyncHTTP instead in plugin context) |
| `URI` | URI construction/parsing |
| `Digest` | MD5 hashing for cache keys |
| `MIME::Base64` | Base64 for OAuth PKCE code_challenge |
| `Crypt::OpenSSL::Random` | Secure random bytes for PKCE code_verifier |
| `HTML::Parser` | HTML parsing if needed for settings |
| `XML::Simple` or `XML::Parser` | XML parsing |
| `DBI` / `DBD::SQLite` | Direct database access (if needed for importer) |
| `Proc::Background` | Background process management (alternative to direct fork) |
| `IO::Socket::SSL` | TLS sockets |

**Note:** `JSON::XS` is available. Use `use JSON::XS qw(encode_json decode_json)` directly. No need for `JSON->new`.

### Protocol Handler Pattern

Register a custom URI scheme for `spotify://` tracks:

```perl
# In Plugin.pm initPlugin():
Slim::Player::ProtocolHandlers->registerHandler('spotify', 'Plugins::SpotOn::ProtocolHandler');

# ProtocolHandler.pm — key methods:
sub new { ... }                     # returns HTTP-based handler for remote
sub getFormatForURL { 'flc' }       # or 'pcm', 'mp3' depending on player
sub canSeek { 1 }
sub getSeekData {
    my ($class, $client, $song, $newtime) = @_;
    return { timeOffset => $newtime };
}
sub isRemote { 1 }
sub isRepeatingStream { 1 }         # Connect mode: treat as radio stream
sub getMetadataFor {
    my ($class, $client, $url) = @_;
    return {
        artist => ..., album => ..., title => ...,
        cover  => ..., duration => ..., bitrate => ...,
    };
}
sub getNextTrack { ... }             # fetch next track URL
```

### Transcoding (custom-convert.conf)

```
# SpotOn single-track mode:
# [binary] [flags] | [transcoder]
# Flags: R = remote, T = can seek to time offset, B = bitrate selectable

spt flc * *
    [spoton] --single-track %I --start-position %s | [flac] --best --silent --endian little --sign signed --channels 2 --bps 16 --sample-rate 44100 -s -

spt pcm * *
    [spoton] --single-track %I --start-position %s

spt mp3 * *
    [spoton] --single-track %I --start-position %s | [lame] --silent -q 0 -r -s 44.1 --preset cbr 320 - -
```

`spt` is the custom source format token. LMS selects the right pipeline based on player capabilities. `%I` = Spotify track URI, `%s` = start position in seconds.

---

## librespot

**Version:** 0.8.0 (released November 10, 2024) — latest stable
**Source:** https://github.com/librespot-org/librespot
**Confidence: HIGH** (official releases page verified)

### Build: Audio Backends

Compiled with `--features` at build time — cannot be changed at runtime:

| Backend | Feature Flag | Use Case | Default |
|---------|-------------|----------|---------|
| Rodio | `rodio-backend` | Cross-platform, uses ALSA on Linux | Yes |
| ALSA | `alsa-backend` | Linux low-latency | No |
| PulseAudio | `pulseaudio-backend` | Linux PulseAudio | No |
| GStreamer | `gstreamer-backend` | Custom pipelines | No |
| **Pipe** | always included | stdout output for LMS integration | N/A |
| **Subprocess** | always included | spawn child with audio | N/A |

For SpotOn: compile with `--features rodio-backend,with-libmdns` for Connect daemon mode. The pipe backend (`--backend pipe`) is always available for single-track streaming.

OGG passthrough: compile with `--features passthrough-decoder` to enable `--passthrough` flag (outputs raw Ogg Vorbis instead of decoded PCM).

### Discovery Backends (mDNS)

| Backend | Feature Flag | Platform |
|---------|-------------|----------|
| libmdns | `with-libmdns` | All (default, pure Rust) |
| avahi | `with-avahi` | Linux only |
| dns-sd | `with-dns-sd` | macOS/iOS, not Windows |

For SpotOn Connect daemons: `with-libmdns` is correct (cross-platform, no system dependency).

### TLS Backends

| Backend | Feature Flag | Notes |
|---------|-------------|-------|
| native-tls | `native-tls` | System OpenSSL/SChannel (default) |
| rustls-tls | `rustls-tls` | Pure Rust, no system TLS dependency |

At least one TLS backend is required. `native-tls` for ARM devices where OpenSSL is present; `rustls-tls` for fully static builds.

### Key CLI Flags (Runtime)

**Identity:**
- `--name <name>` / `-n` — Device name as shown in Spotify app
- `--device-type <type>` — e.g., `speaker`, `avr`, `stb`, `computer`

**Audio:**
- `--backend <backend>` — e.g., `pipe`, `alsa`, `pulseaudio`
- `--device <device>` — Audio device to use
- `--format <format>` — `S16`, `S24`, `S24_3`, `S32`, `F32`, `F64`
- `--bitrate <kbps>` / `-b` — `96`, `160`, `320` (Vorbis quality tiers)
- `--passthrough` — Output raw Ogg Vorbis (requires `passthrough-decoder` feature)
- `--enable-volume-normalisation` — Replay Gain
- `--normalisation-method <method>` — `basic` or `dynamic` (basic = simpler, dynamic = loudness-compensated)
- `--initial-volume <0-100>` — Starting volume

**Cache:**
- `--cache <dir>` / `-c` — Directory for credentials and audio cache
- `--cache-size-limit <bytes>` — Limit audio cache size
- `--disable-audio-cache` — Disable audio caching entirely

**Discovery/Connect:**
- `--disable-discovery` — Disable mDNS/ZeroConf announcement (no Connect visibility)
- `--device-id <id>` — Stable device ID (use player MAC address for stability)

**Single-track mode (Herger's fork additions, NOT in upstream librespot-org):**

These flags exist in Herger's `michaelherger/spotty` fork and will need to be re-implemented in SpotOn's own librespot fork:

- `--single-track <uri>` — Decode one track to stdout and exit (for `custom-convert.conf` use)
- `--start-position <seconds>` — Start from offset (seeking support)
- `--lms <host:port>` — Notify LMS of state changes via JSON-RPC
- `--player-mac <mac>` — Identify which LMS player to notify
- `--get-token` — Retrieve Web API token (for OAuth flow integration)
- `--lms-auth` — Use LMS-provided credentials (Keymaster/login5 integration)
- `--check` — Print JSON capability manifest and exit

**Note:** Upstream librespot 0.8.0 does NOT have `--single-track` or `--lms`. SpotOn needs its own librespot fork. Per AD-04 decision in REQUIREMENTS.md, SpotOn follows Plan B: fork from librespot-org and add LMS glue code.

**`--check` output (from Herger's spotty):**
```json
{
  "version": "0.9.x",
  "lmsAuthToken": true,
  "hasGetToken": true,
  "hasPassthrough": true,
  "hasSingleTrack": true,
  "hasConnectModePlayback": true
}
```
P-10 warning: Capability keys can change between versions. Always provide defaults when reading `--check` output.

### Audio Transport: Pipe Backend

The pipe backend outputs:
- **Default:** Raw stereo S16LE PCM, 44.1 kHz, 2 channels (standard CD format)
- **With `--passthrough`:** Raw Ogg Vorbis container (skip decoding, save CPU)
- **Bit depth:** Configurable via `--format` (S16 default, S24/S32 available)

For LMS integration: `--backend pipe` writes to stdout, LMS reads via `custom-convert.conf` transcoding pipeline.

**FIFO for Connect mode:** Binary stdout redirected to named pipe (`> /tmp/spoton-$player.pcm`), LMS reads via `[cat] $FIFO$` in convert.conf with `# R` flag (remote streaming). Known limitations: ~5-10s seek latency (P-19), occasional white noise on reconnect (P-20). HTTP-streaming is the architectural target.

**Rate limiting (P-16):** The binary's audio sink MUST implement nanosecond-precision wall-clock rate limiting: `expected_ns = frames_consumed × 10^9 / SAMPLE_RATE`. Without this, Spirc reports incorrect position to Spotify cloud.

### Connect Mode Operation

librespot in Connect mode:
1. Announces device via mDNS (ZeroConf/Spotify Connect protocol)
2. Receives Spirc (Spotify Remote Procedure Call) commands from Spotify app
3. Decodes audio and writes to selected backend
4. Fires `PlayerEvent` callbacks: `Playing`, `Paused`, `Stopped`, `Changed`, `VolumeSet`, `EndOfTrack`, `PositionChanged` (added in 0.7.0)

**Playback transfer (v0.8.0):** Added `transfer()` method on Spirc to initiate playback transfer to local device. Relevant for mid-song Connect handoff.

### Supported Platforms for Binary Distribution

| Platform | Target Triple |
|----------|--------------|
| x86_64 Linux | `x86_64-unknown-linux-musl` (static) |
| i386 Linux | `i686-unknown-linux-musl` |
| aarch64 Linux (Pi 4, etc.) | `aarch64-unknown-linux-musl` |
| armv7 Linux (Pi 2/3 32-bit) | `armv7-unknown-linux-musleabihf` |
| macOS | `x86_64-apple-darwin` / `aarch64-apple-darwin` |
| Windows | `x86_64-pc-windows-msvc` |

Use `cross-rs/cross` with Podman/Docker for ARM cross-compilation (P-11). Native Pi builds are too slow.

musl targets recommended for distribution (static linking, no glibc version dependency).

---

## Spotify Web API v1

**Base URL:** `https://api.spotify.com/v1/`
**Auth:** OAuth 2.0 — Bearer token in Authorization header
**Confidence: HIGH** (verified against official developer docs, February+March 2026 changelogs)

### Critical 2026 Context: Development Mode Restrictions

As of February 11, 2026 (new apps) and March 9, 2026 (existing apps), Development Mode has major restrictions:

| Aspect | Old Behavior | New (Feb 2026) |
|--------|-------------|----------------|
| Max test users | 25 | **5** |
| Premium required for app owner | No | **Yes** |
| Batch endpoints (GET /tracks, /albums, etc.) | Available | **Removed** |
| Browse category endpoints | Available | **Removed** |
| Search limit max | 50 | **10** |
| Search default | 20 | **5** |
| `popularity`, `followers` fields | Available | **Removed** |
| `available_markets` on tracks/albums | Available | **Removed** |

**Extended Quota Mode** (250k+ MAU, registered business): **Not affected** — all old endpoints remain.

**SpotOn implication:** SpotOn operates in Development Mode (each user registers their own Spotify Developer App, or uses Keymaster which bypasses this entirely). The Keymaster approach (librespot's `--lms-auth`) generates tokens directly from the user's logged-in Spotify session, bypassing the Developer App requirement entirely and avoiding quota limits. This is the primary reason the PROJECT.md chose Keymaster-only auth.

### Endpoint Status Table (May 2026)

#### Working in Development Mode

| Endpoint | Path | Scope Required | Notes |
|----------|------|---------------|-------|
| Search | `GET /search` | none | Max limit=10 per type in dev mode |
| Get Track | `GET /tracks/{id}` | none | Batch `GET /tracks` removed in dev mode |
| Get Album | `GET /albums/{id}` | none | Batch `GET /albums` removed in dev mode |
| Get Artist | `GET /artists/{id}` | none | Batch `GET /artists` removed in dev mode |
| Get Playlist | `GET /playlists/{id}` | `playlist-read-private` | |
| Get Playlist Items | `GET /playlists/{id}/items` | `playlist-read-private` | Renamed from `/tracks` |
| Get User's Playlists | `GET /me/playlists` | `playlist-read-private` | |
| Get Saved Tracks | `GET /me/tracks` | `user-library-read` | Max limit=50, offset pagination |
| Get Saved Albums | `GET /me/albums` | `user-library-read` | |
| Get Top Items | `GET /me/top/{type}` | `user-top-read` | type=tracks or artists, time_range param |
| Get Recently Played | `GET /me/player/recently-played` | `user-read-recently-played` | Cursor-based pagination, max 50 |
| Get Playback State | `GET /me/player` | `user-read-playback-state` | |
| Transfer Playback | `PUT /me/player` | `user-modify-playback-state` | |
| Get Devices | `GET /me/player/devices` | `user-read-playback-state` | |
| Play/Resume | `PUT /me/player/play` | `user-modify-playback-state` | |
| Pause | `PUT /me/player/pause` | `user-modify-playback-state` | |
| Skip Next | `POST /me/player/next` | `user-modify-playback-state` | |
| Skip Previous | `POST /me/player/previous` | `user-modify-playback-state` | |
| Seek | `PUT /me/player/seek` | `user-modify-playback-state` | |
| Set Volume | `PUT /me/player/volume` | `user-modify-playback-state` | |
| Get Queue | `GET /me/player/queue` | `user-read-currently-playing` | |
| Add to Queue | `POST /me/player/queue` | `user-modify-playback-state` | |
| Save to Library | `PUT /me/library` | `user-library-modify` | New unified endpoint (Feb 2026) |
| Remove from Library | `DELETE /me/library` | `user-library-modify` | New unified endpoint |
| Check Library | `GET /me/library/contains` | `user-library-read` | New unified endpoint |
| Follow Artists | `PUT /me/following` | `user-follow-modify` | Via new unified endpoint |
| Create Playlist | `POST /me/playlists` | `playlist-modify-public/private` | |

#### Deprecated (available but marked for removal)

| Endpoint | Path | Status | Notes |
|----------|------|--------|-------|
| Get Featured Playlists | `GET /browse/featured-playlists` | DEPRECATED | Still works but marked deprecated |
| Implicit Grant Auth | Auth flow | DEPRECATED | Use PKCE instead |
| Playlist Items (old) | `GET/POST/PUT/DELETE /playlists/{id}/tracks` | DEPRECATED | Use `/items` path |

#### Removed (Development Mode) — Do Not Use

| Endpoint | Path | Removed Since | Replacement |
|----------|------|--------------|-------------|
| Recommendations | `GET /recommendations` | Nov 27, 2024 | None |
| Audio Features | `GET /audio-features/{id}` | Nov 27, 2024 | None |
| Audio Analysis | `GET /audio-analysis/{id}` | Nov 27, 2024 | None |
| Related Artists | `GET /artists/{id}/related-artists` | Nov 27, 2024 | None |
| Artist Top Tracks | `GET /artists/{id}/top-tracks` | Feb 2026 | None |
| New Releases | `GET /browse/new-releases` | Feb 2026 | None |
| Browse Categories | `GET /browse/categories` | Feb 2026 | None |
| Single Category | `GET /browse/categories/{id}` | Feb 2026 | None |
| Several Tracks | `GET /tracks` | Feb 2026 (dev mode) | `GET /tracks/{id}` individually |
| Several Albums | `GET /albums` | Feb 2026 (dev mode) | `GET /albums/{id}` individually |
| Several Artists | `GET /artists` | Feb 2026 (dev mode) | `GET /artists/{id}` individually |
| Get Users Profile | `GET /users/{id}` | Feb 2026 (dev mode) | N/A |
| Get User's Playlists (other) | `GET /users/{id}/playlists` | Feb 2026 (dev mode) | N/A |
| Available Markets | `GET /markets` | Feb 2026 (dev mode) | N/A |

#### Field Removals (Development Mode)

These response fields are no longer returned in dev mode:

| Object Type | Removed Fields |
|-------------|---------------|
| Track | `available_markets`, `linked_from`, `popularity` |
| Album | `album_group`, `available_markets`, `label`, `popularity` |
| Artist | `followers`, `popularity` |
| User | `country`, `email`, `explicit_content`, `followers`, `product` |
| Show | `available_markets`, `publisher` |
| Playlist | `tracks` → renamed to `items` with structure changes |

**Do not hardcode access to removed fields.** Use optional chaining or `// undef` guards.

### OAuth Scopes for SpotOn

Minimal scope set needed:

```
user-library-read          # Saved tracks, albums
user-library-modify        # Save/remove from library
user-read-recently-played  # Recently played
user-top-read              # Top tracks/artists
playlist-read-private      # Private playlists
playlist-modify-private    # Modify private playlists
user-read-playback-state   # Player state
user-modify-playback-state # Playback control
streaming                  # Required for Connect (not directly used but needed for token)
user-follow-read           # Check followed artists
user-follow-modify         # Follow/unfollow artists
```

### Rate Limits

- Rolling 30-second window, app-wide (not per-user or per-endpoint)
- Respond to `429` + `Retry-After` header (value in seconds)
- Central throttle component is mandatory (P-01, NFL-03): one `API/Client.pm` through which ALL requests flow
- Proactive burst prevention is preferred over reactive 429 handling
- Batch APIs (e.g., multiple track IDs per request) were available but are removed in dev mode — fetch individually and throttle accordingly

### Keymaster / login5 Authentication

**Confidence: MEDIUM** — Internal Spotify protocol; not in public docs

Keymaster is Spotify's internal credential exchange mechanism used by librespot since v0.6.0. The librespot binary can obtain a Web API access token directly from the user's logged-in Spotify credentials without requiring a Developer App or browser redirect. This is done via the `login5` protocol (HTTP-based, replaced the old Mercury-based credential exchange in librespot 0.5/0.6).

In librespot (Herger's fork): `--get-token --scope <scopes>` returns a JSON object with `accessToken` and `expiresIn`. The plugin passes this to the Web API.

This is why PROJECT.md chose Keymaster-only: the user only needs to authenticate with librespot (username/password or stored credentials), and all Web API tokens are derived from that. No Spotify Developer App registration required per user. Risk: if Spotify deprecates the Shannon protocol entirely, the entire plugin stack fails simultaneously (streaming + auth + Connect all break at once).

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Async HTTP | SimpleAsyncHTTP | LWP::UserAgent | LWP blocks; LMS is single-threaded |
| Async HTTP | SimpleAsyncHTTP | AnyEvent::HTTP | Available in CPAN bundle but overkill; SimpleAsyncHTTP is the LMS idiom |
| JSON | JSON::XS | Mojo::JSON | Mojo not bundled |
| Spotify transport | librespot | Official Spotify SDK | eSDK requires approved partner, NDA, Certomato certification |
| Spotify transport | librespot | Spotify Web Playback SDK | JavaScript only, browser-based |
| Auth | Keymaster/login5 | PKCE flow | PKCE requires browser redirect + Developer App ID per user |
| Auth | Keymaster/login5 | Client Credentials | No user context, can't access `me/*` endpoints |
| Connect audio | HTTP-streaming | FIFO | FIFO has architectural seek latency (P-19) and white noise (P-20) issues |
| mDNS | libmdns (built-in) | Avahi | Avahi is Linux-only; libmdns is cross-platform pure Rust |
| Cross-compile | cross-rs/cross | Native compile on ARM | Native Pi builds too slow (P-11) |
| Featured Playlists | `me/playlists` filtered | `browse/featured-playlists` | Deprecated; Daily Mixes are user-owned playlists accessible via `me/playlists` |

---

## LMS Version Targeting

| Version | Status | Notes |
|---------|--------|-------|
| LMS 7.x | Herger supports; SpotOn does not | Too much legacy code to maintain |
| LMS 8.0 | Minimum floor | Significant adoption, OPMLBased stable |
| LMS 8.5.1 | Full features | Recommended for primary development |
| LMS 9.0 (Nov 2024) | Active | First "Lyrion" branded release |
| LMS 9.1.1 (Feb 2026) | Current stable | WebSocket client for plugins, artwork hooks |
| LMS 9.2.0-dev | Development | Not stable |

Decision from PROJECT.md: LMS 8.0+ floor, full features from 8.5.1. LMS 9.x compatibility is required since that is what most active installations will be running by the time SpotOn ships.

---

## Installation Skeleton

```bash
# No external dependencies required.
# All modules are bundled with LMS.

# Plugin directory structure:
Plugins/SpotOn/
├── Plugin.pm
├── ProtocolHandler.pm
├── Settings.pm
├── strings.txt
├── install.xml
├── custom-convert.conf
├── custom-types.conf          # declares 'spt' as a known type
├── API/
│   ├── Client.pm              # Central throttle + SimpleAsyncHTTP wrapper
│   ├── Auth.pm                # Keymaster token + refresh
│   ├── Browse.pm
│   ├── Library.pm
│   ├── Player.pm
│   └── Cache.pm               # TTL-aware Slim::Utils::Cache wrapper
├── Connect/
│   ├── Manager.pm
│   ├── Daemon.pm
│   └── EventHandler.pm
├── Helper.pm                  # Binary discovery, --check, capabilities
└── Bin/
    ├── x86_64-linux/spoton
    ├── i386-linux/spoton
    ├── aarch64-linux/spoton
    ├── arm-linux/spoton
    ├── darwin/spoton
    └── win32/spoton.exe

# install.xml minimum:
<plugin name="SpotOn" version="0.0.1" minVersion="8.0.0">
  <creator>...</creator>
  <email>...</email>
  <url>https://github.com/...</url>
</plugin>
```

---

## Sources

- LMS Plugin API: https://lyrion.org/reference/music-service-plugin/ (HIGH)
- LMS slimserver source: https://github.com/LMS-Community/slimserver (HIGH)
- SimpleAsyncHTTP: https://github.com/LMS-Community/slimserver/blob/49ad8a29d3fc0cac5792509d1887e0ff8585a81d/Slim/Networking/SimpleAsyncHTTP.pm (HIGH)
- Slim::Utils::Prefs: https://github.com/LMS-Community/slimserver/blob/public/7.9/Slim/Utils/Prefs/Namespace.pm (HIGH)
- librespot releases: https://github.com/librespot-org/librespot/releases (HIGH)
- librespot audio backends: https://github.com/librespot-org/librespot/wiki/Audio-Backends (HIGH)
- librespot CHANGELOG: https://docs.rs/crate/librespot/latest/source/CHANGELOG.md (HIGH)
- Spotify Nov 2024 API changes: https://developer.spotify.com/blog/2024-11-27-changes-to-the-web-api (HIGH)
- Spotify Feb 2026 API changelog: https://developer.spotify.com/documentation/web-api/references/changes/february-2026 (HIGH)
- Spotify Feb 2026 migration guide: https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide (HIGH)
- Spotify Mar 2026 changelog: https://developer.spotify.com/documentation/web-api/references/changes/march-2026 (HIGH)
- TechCrunch on Feb 2026 dev mode changes: https://techcrunch.com/2026/02/06/spotify-changes-developer-mode-api-to-require-premium-accounts-limits-test-users/ (MEDIUM)
- Herger's spotty (archived): https://github.com/michaelherger/spotty (MEDIUM — archived 2022)
- Herger's Spotty-Plugin: https://github.com/michaelherger/Spotty-Plugin (HIGH)
- LMS 9 changelog: https://lyrion.org/getting-started/changelog-lms9/ (HIGH)
- Qobuz ProtocolHandler (reference implementation): https://github.com/LMS-Community/plugin-Qobuz/blob/master/ProtocolHandler.pm (HIGH)
