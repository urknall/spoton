# Architecture Patterns: LMS Spotify Plugin

**Domain:** Streaming service plugin for Lyrion Music Server
**Researched:** 2026-05-26
**Sources:** Spotty-Plugin (michaelherger/Spotty-Plugin, library-integration branch), LMS slimserver source (LMS-Community/slimserver public/8.3), lyrion.org/reference/music-service-plugin

---

## How LMS Plugin Architecture Works (Enforced Patterns)

### The Single-Thread Constraint

LMS is single-threaded Perl. Every operation in the main server loop must be non-blocking. This is the dominant architectural constraint. All HTTP calls from plugin code must use `Slim::Networking::SimpleAsyncHTTP` (callback-based). Synchronous `LWP::UserAgent` calls are only permitted in scanner/importer context (`main::SCANNER`).

### OPMLBased Base Class

`Slim::Plugin::OPMLBased` is the base class for service plugins. It handles:
- Registration in LMS menus (`menu => 'radios'`, `is_app => 1`)
- CLI query dispatch (the `tag` maps to a CLI command, e.g. `spotty items`)
- Jive home menu registration (for native SB hardware displays)
- Feed function: one entry point `handleFeed(\&cb, $args)` drives the entire browse tree

The plugin calls `$class->SUPER::initPlugin(feed => \&handleFeed, tag => 'spoton', menu => 'radios', is_app => 1, weight => 1)` and the framework does the rest.

### What `initPlugin` Must Do

Every plugin's `initPlugin` is responsible for:
1. Preference defaults (`$prefs->init({...})`)
2. Protocol handler registration (`Slim::Player::ProtocolHandlers->registerHandler('spotify', ...)`)
3. Settings UI setup (if `main::WEBUI`)
4. Importer registration (if `CAN_IMPORTER`, LMS 8.0+)
5. Calling `$class->SUPER::initPlugin(...)` to register the feed

`postinitPlugin` (called after all plugins load) is where cross-plugin integrations go (DSTM, LastMix, OnlineLibrary icons).

### Protocol Handler Pattern

`ProtocolHandler.pm` inherits from `Slim::Formats::RemoteStream`. It is the bridge between Spotify URIs and LMS playback. Key methods:

| Method | Purpose |
|--------|---------|
| `contentType` | Returns `'spt'` — the custom format token |
| `formatOverride` | Called at track start; updates transcoding table, returns `'spt'` |
| `canSeek` / `canTranscodeSeek` | Returns true for LMS 7.9.1+ |
| `getSeekData` | Returns `{ timeOffset => $newtime }` — seek position for `$START$` substitution |
| `canDirectStream` | Returns 0 — always transcodes |
| `isRepeatingStream` | Returns true when in Connect mode — prevents LMS from treating stream end as track end |
| `getNextTrack` | Dispatcher: delegates to Connect if in Connect mode, calls `$successCb->()` otherwise |
| `explodePlaylist` | Resolves `spotify:album:X` or `spotify:playlist:X` URIs to track list |
| `getMetadataFor` | Returns track metadata for the Now Playing display |
| `audioScrobblerSource` | Returns `'P'` (user-chosen) |

`canDirectStream { 0 }` is critical — it tells LMS to always run the transcoding pipeline (`custom-convert.conf`) rather than streaming directly. This is what routes audio through the librespot binary.

### Transcoding Pipeline (custom-convert.conf)

The `spt` format is declared custom. LMS selects the pipeline based on player capability:

```
spt pcm * *    # RT:{START=--start-position %s}
  [binary] --single-track $URL$ --bitrate 320 $START$ ...

spt flc * *    # RT:{START=--start-position %s}
  [binary] --single-track $URL$ ... | [flac] -cs ...

spt mp3 * *    # RB:{BITRATE=--abr %B}T:{START=--start-position %s}
  [binary] --single-track $URL$ ... | [lame] ...
```

`# R` = Remote Streaming (stream mode, `isRepeatingStream`). `# T:` = seek position substitution. `# B:` = bitrate selection. `$CACHE$` is injected at runtime by `updateTranscodingTable()`. `$URL$` is the `spotify:track:ID` URI (without leading `//`).

`updateTranscodingTable()` in Plugin.pm does a regex replace on the live transcoding table — injecting the per-player cache directory and bitrate preference. This must happen before each track starts, not once at init.

---

## Recommended Architecture: Component Map

```
Plugin.pm (OPMLBased)
  |-- initPlugin: register handlers, prefs, importer, kill-hanging timer
  |-- postinitPlugin: cross-plugin integrations  
  |-- updateTranscodingTable(client): inject $CACHE$ + bitrate + normalization
  |-- killHangingProcesses(): hourly cleanup of single-track binary instances
  |-- getAPIHandler(client): factory returning per-client API instance
  
ProtocolHandler.pm (Slim::Formats::RemoteStream)
  |-- contentType/formatOverride: declares 'spt', triggers transcoding setup
  |-- getNextTrack: dispatcher (Connect mode vs Browse mode)
  |-- isRepeatingStream: true in Connect mode (FIFO/HTTP stream stays open)
  |-- getMetadataFor: track info for Now Playing display
  |-- explodePlaylist: spotify:album/playlist URI → track list
  
Settings.pm (Slim::Web::Settings)
  |-- Global settings (bitrate, normalization, Connect enable)
  |-- Per-player settings via $prefs->client($client)
  
API/
  |-- Client.pm: ALL HTTP exits. getToken → build request → AsyncRequest → callback
  |    |-- Rate limit state: 'spoton_rate_limit_exceeded' cache key
  |    |-- Cache key: md5_hex(url + token-for-personal-endpoints)
  |    |-- TTL: from Cache-Control header, fallback 60s, playlist special-cases
  |-- Auth.pm: Token storage/refresh. Keymaster mode: get token from binary.
  |-- Browse.pm: browse/*, search/*, new-releases, categories
  |-- Library.pm: me/tracks, me/albums, me/playlists, me/artists, me/player/recently-played
  |-- Player.pm: me/player, me/devices
  |-- Cache.pm: Persistent cache (Slim::Utils::DbCache subtype), stores normalized items
  
Connect/
  |-- Manager.pm: DaemonManager
  |    |-- Subscribes to client new/disconnect events → debounced initHelpers()
  |    |-- Subscribes to sync events → shutdown + deferred initHelpers()
  |    |-- One Daemon instance per player (or sync-group master)
  |    |-- %helperInstances: { mac => Daemon object }
  |-- Daemon.pm: One librespot process
  |    |-- start(): Proc::Background->new(binary, -c cache, -n name, --player-mac, --lms)
  |    |-- _checkStartTimes(): disable discovery after 3 crashes in 5 minutes
  |    |-- alive()/pid()/uptime(): health check
  |-- EventHandler.pm (Connect.pm in Spotty, merged):
  |    |-- addDispatch ['spottyconnect', '_cmd'] → _connectEvent(request)
  |    |-- subscribe newsong → _onNewSong
  |    |-- subscribe pause/stop → _onPause  
  |    |-- subscribe mixer/volume → _onVolume
  |    |-- _connectEvent: handles start/stop/change/volume events from binary
  |    |-- getNextTrack: intercepts ProtocolHandler->getNextTrack in Connect mode
  
Helper.pm:
  |-- Binary discovery: arch-specific Bin/ paths + custom override
  |-- helperCheck($candidate): runs --check, parses "ok spoton vX.Y.Z\n{JSON}"
  |-- getCapability($key): returns capability from --check JSON, with defaults
  
AccountHelper.pm:
  |-- Credential storage: cacheFolder(account_id) → per-account temp dir
  |-- getAccount(client) → account ID for player
  |-- hasCredentials() / getAllCredentials()
  |-- purgeAudioCache() / purgeAudioCacheAfterXTracks()
  
OPML.pm (Browse.pm equivalent):
  |-- handleFeed(cb, args): entry point registered in SUPER::initPlugin
  |-- Top-level menu: Home / Search / Library
  |-- All API calls async via API/Client.pm
  |-- Converts API responses to OPML item arrays
  
Importer.pm:
  |-- startScan() (sync): populates LMS online library database
  |-- needsUpdate() (async): checks if rescan needed
  
install.xml: Plugin manifest (name, version, minVersion, GUID, repo URL)
strings.txt: i18n (EN + DE minimum)
custom-convert.conf: spt→pcm, spt→flc, spt→mp3 pipelines
custom-types.conf: declares 'spt' as valid LMS source format
Bin/: Pre-compiled librespot binaries per architecture
```

---

## Component Boundaries

| Component | Owns | Does NOT Touch |
|-----------|------|----------------|
| Plugin.pm | Initialization, transcoding table, API factory | HTTP requests, audio pipeline |
| ProtocolHandler.pm | URI→stream routing, metadata, seek data | API calls (delegates to API/), daemon management |
| OPML.pm | Browse tree construction, UI strings | Daemon management, transcoding table |
| API/Client.pm | All outbound HTTP, token injection, rate limit, response cache | LMS player state, Connect state |
| API/Auth.pm | Token storage/refresh only | Browse logic, player control |
| API/Browse.pm | Spotify browse/search endpoints | Library, player endpoints |
| API/Library.pm | me/* personal library endpoints | Browse, player endpoints |
| API/Player.pm | me/player state, devices | Browse, library, streaming |
| Connect/Manager.pm | Daemon lifecycle, sync group logic | Event handling, API calls |
| Connect/Daemon.pm | One process: start/stop/monitor | Anything outside its process lifecycle |
| Connect/EventHandler.pm | LMS event subscriptions, event→action dispatch | Daemon lifecycle (delegates to Manager) |
| Helper.pm | Binary path resolution, capability detection | Everything else |
| AccountHelper.pm | Credential/cache directory management | API calls, UI |

**The critical boundary:** `API/Client.pm` is the only module that calls `Slim::Networking::SimpleAsyncHTTP`. No other module makes HTTP requests. Rate limiting state lives here.

---

## Data Flow: Browse

```
User navigates in LMS app
  → OPMLBased framework calls handleFeed(cb, {client, ...})
    → OPML.pm builds top-level menu or fetches sub-menu
      → API/Browse.pm (or Library.pm) called with callback
        → API/Client.pm checks cache (Slim::Utils::Cache)
          → Cache hit: cb->($cached) immediately
          → Cache miss: getToken from Auth.pm
            → AsyncRequest->get(API_URL, headers=[Bearer token])
              → LMS event loop handles HTTP response
                → _gotResponse: parse JSON, cache result, cb->($result)
                  → API/Browse.pm transforms result to OPML items
                    → OPML.pm returns items array to handleFeed cb
                      → LMS renders menu to user
```

Key properties:
- Every arrow is a callback, never a blocking call
- Cache sits at Client.pm level — entire response objects cached
- TTL strategy: 60s default, 3600s for artist/album metadata, playlist-specific logic

---

## Data Flow: Browse Streaming (Single Track)

```
User selects a track in LMS
  → ProtocolHandler::getNextTrack(song, successCb, errorCb)
    → Not in Connect mode → successCb->() immediately
      → LMS calls formatOverride(song)
        → Plugin::updateTranscodingTable(client) [injects $CACHE$, bitrate]
        → returns 'spt'
          → LMS looks up transcoding rule: spt→flc (or pcm, mp3)
            → Spawns: [binary] --single-track spotify:track:ID --start-position N
              → binary writes PCM to stdout
                → [flac] reads stdin, writes FLAC to stdout
                  → LMS reads stdout, distributes to player(s)
```

Seeking:
```
User seeks to position N
  → ProtocolHandler::getSeekData returns { timeOffset => N }
    → LMS substitutes $START$=N in convert.conf command
      → binary respawned with --start-position N
```

Multiroom: LMS's `Slim::Player::Source::nextChunk` reads from the sync master's stream and pushes chunks to sync slaves natively. No plugin action needed.

---

## Data Flow: Spotify Connect

### Subsystem Initialization
```
Plugin.pm::initPlugin (at server start)
  → Connect/EventHandler.pm::init()
    → addDispatch(['spottyconnect','_cmd'], \&_connectEvent)
    → subscribe newsong, pause/stop, mixer/volume
    → Connect/Manager.pm::init()
      → subscribe client new/disconnect → debounced initHelpers()
      → subscribe sync events → shutdown + deferred initHelpers()
        → for each player with Connect enabled:
            Connect/Daemon.pm::new(mac) → start() → Proc::Background
              → binary running: -n name -c cache --player-mac mac --lms host:port
```

### Connect Playback Event
```
User presses Play in Spotify app
  → binary receives Spirc message from Spotify cloud
    → binary sends HTTP POST to LMS: 
        {"method":"slim.request","params":["mac",["spottyconnect","start","trackID"]]}
          → LMS dispatches to _connectEvent(request)
            → _connectEvent calls API/Player.pm::player(cb) to get current state
              → API returns current track + position
                → if new track: 
                    client->pluginData(newTrack => 1)
                    Queue spotify:track:ID for playback:
                      Slim::Control::Request(['playlist','play','spotify://track:ID'])
                        → ProtocolHandler::getNextTrack called
                          → isSpotifyConnect == true
                            → Connect::getNextTrack called
                              → pluginData(newTrack) == 1:
                                  song->streamUrl(track_uri)
                                  setSpotifyConnect(client, state)
                                  successCb->()
                → isRepeatingStream returns true (FIFO stays open)
                → LMS starts playback via transcoding pipeline
                  → [binary running in daemon mode] writes audio
```

### Connect Volume Event
```
Spotify app changes volume
  → binary sends: ["spottyconnect","volume","N"]
    → _connectEvent: cmd eq 'volume'
      → suppress if within VOLUME_GRACE_PERIOD (5s) of daemon start
      → Slim::Control::Request ['mixer','volume',N]
        → request source = Connect (prevents echo-back loop)
```

### Connect Stop Event
```
User pauses in Spotify app
  → binary sends: ["spottyconnect","stop"]
    → _connectEvent → cmd eq 'stop'
      → API::player(cb) confirms stopped
        → Slim::Control::Request ['playlist','pause']
```

### LMS Pause → Spotify Sync
```
User pauses in LMS
  → _onPause subscription fires
    → isSpotifyConnect? → API/Player.pm::pause()
```

### Sync Group Connect
```
Two players synced, user plays on Spotify
  → Manager::initHelpers called
    → isSlave($client)? → find sync master
      → only ONE daemon for the group master
        → name = concat of all player names: "player1 & player2"
          → single binary announces group to Spotify
```

---

## Data Flow: HTTP Streaming (Connect Audio Transport)

SpotOn targets HTTP streaming over FIFO. The architecture difference:

**FIFO (Spotty current approach, SpotOn fallback):**
```
binary daemon --fifo /tmp/spoton-stream-MAC.pcm
  → Shell redirect: binary stdout > FIFO file
    → LMS reads FIFO via custom-convert.conf: [cat] $FIFO$
      → LMS treats as repeating internet-radio stream
```

**HTTP Streaming (SpotOn primary target):**
```
binary daemon --http-port 57xxx
  → Binary serves http://127.0.0.1:57xxx/stream.pcm (chunked)
    → LMS fetches via standard HTTP radio infrastructure
      → ProtocolHandler for Connect mode returns http://127.0.0.1:57xxx/stream.pcm
        → LMS's existing HTTP stream handling takes over
          → Content-Range headers enable position sync
          → Clean connection semantics on track change (no FIFO flush problem)
```

The HTTP approach requires the binary to embed an HTTP server. Until that is built, FIFO is the fallback with `isRepeatingStream(1)` and `startOffset` position sync.

---

## Suggested Build Order

Dependencies flow upward: lower layers must exist before higher layers can be tested.

### Layer 0: Skeleton (prerequisite for everything)
- `install.xml`, `strings.txt`, `Plugin.pm` (minimal initPlugin)
- `Helper.pm` (binary discovery + `--check`)
- `AccountHelper.pm` (credential + cache dir management)
- `custom-types.conf` (declares `spt`)
- Result: Plugin loads and appears in LMS menu

### Layer 1: Auth + API Foundation
- `API/Auth.pm` (Keymaster token: get token from binary, cache it)
- `API/Client.pm` (central HTTP outlet: token injection, rate limit guard, cache, async)
- `API/Cache.pm` (persistent cache for normalized items)
- Result: Can make authenticated Spotify API calls

### Layer 2: Browse
- `API/Browse.pm`, `API/Library.pm`
- `OPML.pm` (handleFeed, browse tree)
- `Settings.pm` (basic global prefs)
- Result: Full navigation tree works — Home, Search, Library

### Layer 3: Single-Track Streaming
- `ProtocolHandler.pm` (contentType, formatOverride, getSeekData, getMetadataFor)
- `custom-convert.conf` (spt pipelines)
- `Plugin.pm::updateTranscodingTable` (runtime injection)
- `Plugin.pm::killHangingProcesses` (hourly cleanup)
- Result: Tracks play, metadata shows, seeking works

### Layer 4: Connect Core
- `Connect/Daemon.pm` (Proc::Background wrapper, start/stop/alive)
- `Connect/Manager.pm` (lifecycle management, sync groups)
- `Connect/EventHandler.pm` (addDispatch spottyconnect, subscriptions)
- `ProtocolHandler.pm::getNextTrack` extended (isSpotifyConnect branch)
- `ProtocolHandler.pm::isRepeatingStream` (true in Connect mode)
- `API/Player.pm` (me/player, me/devices — needed by EventHandler)
- Result: Connect works with FIFO audio transport

### Layer 5: HTTP Streaming Transport (Connect audio upgrade)
- Binary HTTP server implementation
- ProtocolHandler Connect mode returns `http://127.0.0.1:PORT/` URL
- Remove FIFO paths (or retain as fallback)
- Result: Clean seek, no white noise, accurate position sync

### Layer 6: Polish
- `Importer.pm` (online library scan)
- `Settings/Player.pm` (per-player prefs)
- `DontStopTheMusic.pm`
- `API/Browse.pm` extended (recommendations, related artists, etc.)
- Result: Full feature set

---

## Spotty vs SpotOn: Key Architectural Differences

| Aspect | Spotty (Herger) | SpotOn (planned) |
|--------|-----------------|-----------------|
| API module | Monolithic API.pm (1488 lines) | Split: Browse/Library/Player/Client |
| Rate limiting | Cache key check, dispersed | Centralized in Client.pm, sliding window |
| Connect code | Connect.pm + Connect/DaemonManager.pm + Connect/Daemon.pm | Connect/Manager.pm + Daemon.pm + EventHandler.pm |
| Auth | PKCE OAuth + keymaster fallback | Keymaster primary only |
| Audio transport | FIFO (repeating stream) | HTTP streaming primary, FIFO fallback |
| Binary namespace | `spotty` | `spoton` (own binary, own name) |
| Config key | `spotty_rate_limit_exceeded` | `spoton_rate_limit_exceeded` |
| Sync restart | Full shutdown+restart | Differential restart (P-17) |
| Binary in repo | Separate (michaelherger/spotty) | Decision deferred, likely Bin/ subdir |

The main architectural insight from studying Spotty: **Connect.pm is the most complex module** — it handles the state machine between Connect mode and Browse-streaming mode. In SpotOn, `Connect/EventHandler.pm` absorbs that responsibility, keeping it separate from daemon lifecycle (Manager.pm).

---

## Critical Invariants (Architecture Enforces)

1. **Only `API/Client.pm` calls `SimpleAsyncHTTP`** — rate limit guard and token injection are impossible to enforce otherwise.

2. **`updateTranscodingTable` runs before each track** — the transcoding table is global and mutable (P-09). The only safe pattern is per-track injection in `formatOverride`.

3. **`isRepeatingStream` must return true in Connect mode** — otherwise LMS treats stream-end as track-end and stops the FIFO/HTTP connection.

4. **Progress stored before `playlist play`** — `_onNewSong` fires synchronously from `playlist play` (P-15). Any state the new-song handler needs must be in `pluginData` before the play command.

5. **`['time', N]` never in stream mode** — triggers `_Stop + _Stream` → audio pipeline restart → white noise (P-13). Use `$song->startOffset` only.

6. **Connect daemon PIDs excluded from `killHangingProcesses`** — single-track binary cleanup must not kill daemon-mode binaries (P-03). Guard by checking daemon PID list before pkill.

7. **Daemon uses `Proc::Background` with `die_upon_destroy`** — ensures daemon stops when the Perl object goes out of scope, preventing zombies.

8. **Sink rate-limits at wall-clock** — binary must not decode faster than realtime. Spirc position reports must be accurate. No end-of-track premature fire.

---

## Where the HTTP Streaming Server Fits

The HTTP streaming server lives **inside the binary** (librespot/Rust side), not in Perl. From Perl's perspective, it appears as a standard HTTP radio stream URL.

Architecture impact:
- `Connect/Daemon.pm::start()` passes an HTTP port to the binary: `--http-port 57xxx`
- Port is deterministic (based on player MAC or daemon index) to survive restarts
- `Connect/EventHandler.pm` constructs the stream URL: `http://127.0.0.1:57xxx/stream`
- In `ProtocolHandler::getNextTrack` (Connect mode): `$song->streamUrl("http://127.0.0.1:57xxx/stream")`
- LMS fetches it as an internet radio stream — no FIFO, no `[cat]` in convert.conf
- `isRepeatingStream` still returns true (stream is continuous across track changes)
- Track changes signaled via `change` event from binary, not by HTTP stream ending

This means Connect mode with HTTP transport bypasses `custom-convert.conf` entirely — no transcoding binary spawned per track. Audio comes pre-decoded from the HTTP server. LMS's existing HTTP radio PCM handling takes over.

---

## Sources

- [Spotty-Plugin source (library-integration branch)](https://github.com/michaelherger/Spotty-Plugin) — Connect.pm, Connect/DaemonManager.pm, Connect/Daemon.pm, API/Pipeline.pm, Plugin.pm, ProtocolHandler.pm
- [LMS slimserver source](https://github.com/LMS-Community/slimserver) — Slim/Plugin/OPMLBased.pm
- [Music Service Plugin reference](https://lyrion.org/reference/music-service-plugin/) — official LMS plugin architecture docs
- [Spotty→LMS communication commit](https://github.com/michaelherger/librespot/commit/9a8214340646dbd270cee3b91361b08abeeaa6d5) — JSON-RPC event format
- REQUIREMENTS.md (project) — P-01 through P-20, AD-01 through AD-07
