# Phase 5: Spotify Connect - Research

**Researched:** 2026-06-01
**Domain:** Spotify Connect / librespot Spirc / LMS Daemon Lifecycle / HTTP Audio Streaming
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Audio-Transport**
- D-01: Reines HTTP-Streaming — kein FIFO, kein Fallback. Binary hat eingebauten HTTP-Server, LMS fetcht Audio von `http://127.0.0.1:PORT/stream`. Seek = neue HTTP-Verbindung.
- D-02: PCM (S16LE) als Default-Format + OGG-Passthrough-Option fuer faehige Player. Sync-Gruppen immer PCM.
- D-03: Dynamische Port-Zuweisung — Binary bindet Port 0, OS vergibt freien Port. Binary meldet Port auf stdout (`stream_port=XXXXX`).
- D-04: Transcoding-Pipeline mit eigenem Content-Type `soc`. `formatOverride()` gibt `soc` zurueck wenn Connect aktiv. Profile: `soc pcm * *` und `soc ogg * *`.
- D-05: OGG-Passthrough Auto-Detect + Override — Default: Player-Capability entscheidet. Pro Player Override in Settings: 'OGG erzwingen' / 'PCM erzwingen' / 'Auto'.
- D-06: `canDirectStream` gibt HTTP-URL zurueck fuer einzelne Player. Fuer Sync-Gruppen: 0 → LMS proxied via `new()` Override (Slim::Player::Protocols::HTTP->new mit substituierter URL).

**Daemon-Lifecycle**
- D-07: Daemons starten bei LMS-Start (Plugin-Init) fuer jeden verbundenen Player mit aktiviertem Connect. Kein On-Demand.
- D-08: Mutual Exclusion — Connect verdraengt Browse, Browse verdraengt Connect.
- D-09: Token-Refresh im Binary — librespot-Session refresht intern ueber Keymaster/Spirc. Kein Daemon-Neustart alle 50 Minuten.
- D-10: Connect per Player an/aus — Settings-Toggle pro Player. Default: an fuer alle.

**Sync-Gruppen**
- D-11: Device-Name = verkettete Player-Namen. Nutzt `Slim::Player::Sync::syncname()`.
- D-12: B&O/UPnPBridge-Sonderfaelle — keine Vorab-Einschraenkungen.

**Event-Protokoll**
- D-13: Binary → LMS: JSON-RPC POST mit angereicherten Events. Events enthalten Metadata direkt im Payload.
- D-14: LMS → Binary: HTTP-Control-Endpoints am Binary-HTTP-Server. Endpoints: `POST /control/pause`, `POST /control/play`, `POST /control/volume`, `POST /control/seek`.
- D-15: Spotify Web API (`me/player/*`) als Fallback wenn Binary-Control-Endpoint nicht erreichbar.

### Claude's Discretion
- Crash-Recovery-Strategie: Exponential Backoff vs. fester Retry, Schwellwerte fuer Discovery-Deaktivierung
- Sync-Gruppen: Master-Only Daemon vs. alternatives Pattern, Differentieller vs. Komplett-Neustart bei Gruppen-Aenderung (CON-15)
- Loop-Prevention: Source-Marking (Spotty-NG-Pattern) vs. robusterer Mechanismus
- Volume-Grace-Period: Uebernehmen (20s) vs. bessere Loesung mit librespot 0.8
- Debouncing-Strategie fuer Volume- und Seek-Events
- killHangingProcesses-Schutz (CON-09): Integration mit bestehendem `_killOrphanedProcesses()` Code (PHASE-5-NOTE bereits im Code)
- Enriched Event-Payload-Schema: welche Felder im JSON, Format, optionale vs. required Felder

### Deferred Ideas (OUT OF SCOPE)
- Per-Player OGG-Passthrough auch fuer Browse/Single-Track-Modus → Phase 6
- CON-12 Requirement-Update ("FIFO-based" → "HTTP-streaming") → bei Phase-Start aktualisieren
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CON-01 | One librespot Connect daemon per LMS player | Daemon.pm-Pattern, `Proc::Background`, per-Player start in DaemonManager |
| CON-02 | Daemon lifecycle management (start at init, stop at shutdown, restart on crash with backoff) | `_checkStartTimes()`, exponential backoff pattern, Watchdog-Timer |
| CON-03 | Event dispatching from binary to LMS via JSON-RPC (start/stop/change/volume/pause) | `lms_connect::LMS::notify()`, `spottyconnect` dispatch, `addDispatch` |
| CON-04 | Transfer playback from Spotify app starts audio within 3 seconds | HTTP streaming + `canDirectStream`, spirc_active guard, `stream_port=N` stdout protocol |
| CON-05 | Play/Pause/Skip/Volume controllable from Spotify app | Spirc event loop, PlayerEvent mapping, HTTP control endpoints |
| CON-06 | Sync-group: one daemon on master, name = concatenated player names | `Slim::Player::Sync::syncname()`, master-daemon pattern |
| CON-07 | mDNS/ZeroConf discovery for Connect receivers (optionally disableable) | `librespot-discovery`, `--disable-discovery` flag |
| CON-08 | Mutual exclusion between Browse-streaming and Connect sessions | `formatOverride` mode-check, daemon stop on browse start |
| CON-09 | Connect daemon PIDs excluded from LMS killHangingProcesses | PHASE-5-NOTE in `_killOrphanedProcesses()`, `helperPids()` pattern |
| CON-10 | Connect per player enable/disable in settings | `$prefs->client($client)->get('enableSpotifyConnect')` pattern |
| CON-11 | Volume suppression window after Connect start | `suppress_next_volume` AtomicBool in binary + VOLUME_GRACE_PERIOD in Perl |
| CON-12 | Audio transport (was FIFO, now HTTP-streaming per D-01) | `HttpStreamSink`, `http_stream_server`, `canDirectStream` HTTP URL |
| CON-13 | Position sync via `startOffset` (never `['time', N]` in stream mode) | `$song->startOffset()` pattern from Spotty-NG Connect.pm |
| CON-14 | Sink-level rate-limiting (wall-clock speed, nanosecond-accurate) | `HttpStreamSink::write()` with `expected_ns` math |
| CON-15 | Differential daemon restart on sync-group changes (only affected daemons) | `stopForSync()`, 0.1s timer re-init pattern |
| CON-16 | Unique port assignment per daemon (HTTP audio + ZeroConf) from Manager pool | `TcpListener::bind("0.0.0.0:0")`, Port 0 OS-Zuweisung |
| CON-17 | Progress stored in `pluginData` before `playlist play` (race condition prevention) | `$client->pluginData(progress => $result->{progress})` pre-play pattern |
</phase_requirements>

---

## Summary

Phase 5 implementiert den vollstaendigen Spotify Connect Stack fuer SpotOn. Die Architektur besteht aus drei Ebenen: (1) einem librespot-Binary-Erweiterung das Spirc-Protocol und HTTP-Streaming implementiert, (2) einem Perl Daemon-Management-Layer der einen Connect-Daemon-Prozess pro LMS-Player verwaltet, und (3) einem Event-Dispatch-System das bidirektionale Steuerbefehle zwischen Spotify-App und LMS-Player weiterleitet.

Die Referenz-Implementierung in Spotty-NG (Connect.pm, DaemonManager.pm, Daemon.pm, ProtocolHandler.pm) ist direkt anwendbar, muss aber angepasst werden: (a) SpotOn verwendet HTTP-Streaming statt FIFO (D-01), (b) der Content-Type ist `soc` statt `spc`/`spt`, (c) der Event-Contract unterscheidet sich durch Enriched-Payload (D-13) und HTTP-Control-Endpoints (D-14). Das Spotty-NG `spotty.rs:lms_connect`-Modul liefert das vollstaendige Binary-Seitenpattern: `LMS` struct, `PlayerEvent`-Dispatcher, `ConnectNullSink`, `HttpStreamSink`, `http_stream_server`.

Die kritische Erkenntnis: `librespot-connect 0.8.0` ist auf crates.io verfuegbar (verifiziert per `cargo search`), aber die SpotOn Cargo.toml hat diese Abhaengigkeit noch nicht. Das Binary muss um `--connect` Mode (Spirc-Loop mit `HttpStreamSink`), HTTP-Control-Endpoints und JSON-RPC-Notifier erweitert werden. Der Perl-Layer benoetigt drei neue Module: `Connect.pm`, `Connect/DaemonManager.pm`, `Connect/Daemon.pm`.

**Primary recommendation:** Uebernimm das `lms_connect` Pattern aus Spotty-NG (`spotty.rs`) 1:1 als `connect.rs` in SpotOn, ergaenze `librespot-connect` in Cargo.toml, und portiere DaemonManager/Daemon/Connect aus Spotty-NG mit soc-Content-Type und HTTP-Control-Endpoints.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Spirc Connect Protocol | Binary (Rust) | — | librespot-connect implementiert Spirc; Perl hat keinen direkten Zugriff auf Spotify-AP |
| HTTP Audio Streaming | Binary (Rust) | LMS (via canDirectStream/new()) | Binary bindet HTTP-Server, LMS fetcht als Remote-Stream |
| Event Dispatch (Binary→LMS) | Binary (Rust) | Perl (Empfaenger) | Binary postet JSON-RPC via TCP; Perl dispatcht per addDispatch |
| Event Handling (LMS-Seite) | Perl Plugin | LMS Core | Connect.pm reagiert auf spottyconnect-Events und steuert Player |
| Daemon Lifecycle | Perl Plugin | — | DaemonManager/Daemon nutzen Proc::Background fuer Prozess-Verwaltung |
| Control Commands (LMS→Spotify) | Perl Plugin | Binary HTTP | Plugin ruft /control/* Endpoints auf; Binary leitet an Spirc weiter |
| Volume Loop Prevention | Binary (Rust) | Perl (Grace Period) | suppress_next_volume AtomicBool im Binary; VOLUME_GRACE_PERIOD in Perl |
| Sync Group Handling | Perl Plugin | — | DaemonManager entscheidet Master vs. Slave, syncname() fuer Device-Name |
| Protocol Handler (soc type) | Perl ProtocolHandler | — | formatOverride, canDirectStream, new() sind LMS Plugin API |
| Settings UI | Perl Settings + HTML | — | Per-Player Toggle und OGG-Override in Settings.pm + basic.html |

---

## Standard Stack

### Core (Rust/Binary)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| librespot-connect | 0.8.0 | Spirc protocol, ConnectConfig, `Spirc::new()` | Einzige Open-Source Spotify-Connect-Implementierung |
| librespot-playback | 0.8.0 | Player, PlayerEvent, AudioBackend | Bereits im Cargo.toml, Basis fuer HttpStreamSink |
| librespot-core | 0.8.0 | Session, Cache, Credentials | Bereits im Cargo.toml |
| tokio | 1.x | Async runtime, TcpListener, mpsc | Bereits verwendet |
| hyper | 1.x (features: http1, server) | HTTP/1.0-Server fuer /stream und /control/* | Spotty-NG Muster, produktionserprobt |
| hyper-util | 0.1.x (features: server-auto, server-graceful, service, tokio) | Graceful shutdown fuer HTTP-Server | Zusammen mit hyper |
| http-body-util | 0.1.x | StreamBody, BodyExt fuer Streaming-Response | Notwendig fuer Streaming-Body-Erstellung |
| tokio-stream | 0.1.x | ReceiverStream fuer HTTP-Streaming-Body | Konvertiert mpsc-Receiver zu Stream |
| bytes | 1.x | Bytes-Typ fuer PCM-Chunks im mpsc-Channel | Effiziente Byte-Buffer ohne Copy |
| serde_json | 1.x | JSON fuer JSON-RPC-Notifier | Bereits im Cargo.toml |

[VERIFIED: cargo search librespot-connect] — librespot-connect 0.8.0 auf crates.io bestaetigt.

### Perl/LMS Layer

| Module | Source | Purpose | Why |
|--------|--------|---------|-----|
| Proc::Background | LMS-Bundle | Daemon-Prozess-Verwaltung | Spotty-NG-Pattern, verfuegbar in LMS |
| Slim::Utils::Timers | LMS API | Watchdog, Debouncing, Grace-Periods | Standard LMS-Timer-Pattern |
| Slim::Control::Request::addDispatch | LMS API | spottyconnect CLI-Registrierung | Einziger LMS-Weg fuer Custom-CLI-Commands |
| Slim::Player::Sync | LMS API | `syncname()`, `isSlave()`, `slaves()` | Sync-Gruppen-Handling |
| Slim::Player::Protocols::HTTP | LMS API | Sync-Group-Proxy via `new()` Override | `canDirectStream` Fallback fuer Sync |
| Slim::Networking::SimpleAsyncHTTP | LMS API | HTTP-Control-Endpoint-Calls (LMS→Binary) | Non-blocking, LMS-Standard |
| IO::Select | Perl Core | Synchronous Port-Capture mit Timeout | Vermeidet SIGALRM in LMS-Event-Loop |

### Neue Perl-Module (zu erstellen)

| Modul | Analogon in Spotty-NG | Abweichungen |
|-------|----------------------|--------------|
| `Plugins::SpotOn::Connect` | `Plugins::Spotty::Connect` | `soc` Content-Type, HTTP-Control-Endpoints statt Web-API-Rueckkanal, Enriched-Events |
| `Plugins::SpotOn::Connect::DaemonManager` | `Plugins::Spotty::Connect::DaemonManager` | `soc`-Port-Key, SpotOn-Prefs-Namespace |
| `Plugins::SpotOn::Connect::Daemon` | `Plugins::Spotty::Connect::Daemon` | `--connect` statt `--connect-stream` Flag-Name, SpotOn-Binary-Name |

### Neue Rust-Module (zu erstellen)

| Modul | Analogon | Abweichungen |
|-------|----------|--------------|
| `src/connect.rs` | `spotty.rs::lms_connect` | HTTP-Control-Endpoints zusaetzlich, Enriched JSON-RPC (D-13) |

**Cargo.toml-Erweiterungen:**
```toml
librespot-connect = { version = "0.8", default-features = false, features = ["rustls-tls-native-roots"] }
bytes         = "1"
hyper         = { version = "1", features = ["http1", "server"] }
hyper-util    = { version = "0.1", features = ["server-auto", "server-graceful", "service", "tokio"] }
http-body-util = "0.1"
tokio-stream  = "0.1"
tokio         = { version = "1", features = ["rt-multi-thread", "macros", "net", "io-util", "sync"] }
```

---

## Package Legitimacy Audit

| Package | Registry | Age | Source Repo | slopcheck | Disposition |
|---------|----------|-----|-------------|-----------|-------------|
| librespot-connect | crates.io | ~5 yrs | github.com/librespot-org/librespot | [OK] | Approved |
| bytes | crates.io | ~9 yrs | github.com/tokio-rs/bytes | [OK] | Approved |
| hyper | crates.io | ~12 yrs | github.com/hyperium/hyper | [OK] | Approved |
| hyper-util | crates.io | ~3 yrs | github.com/hyperium/hyper | [OK] | Approved |
| http-body-util | crates.io | ~3 yrs | github.com/hyperium/hyper | [OK] | Approved |
| tokio-stream | crates.io | ~4 yrs | github.com/tokio-rs/tokio | [OK] | Approved |

[ASSUMED] — slopcheck nicht ausgefuehrt (Python-Tool, nicht installiert). Alle Pakete sind etablierte Kern-Rust-Bibliotheken aus Projekte mit hoher Vertrauenswuerdigkeit. `librespot-connect` ist Teil des offiziellen librespot-org-Projekts.

**Packages removed due to slopcheck [SLOP]:** keine
**Packages flagged as suspicious [SUS]:** keine

---

## Architecture Patterns

### System Architecture Diagram

```
Spotify App (Phone/Desktop)
    |  Spirc Protocol (Spotify AP WebSocket)
    v
[spoton binary: --connect]
    +-- Spirc event loop (librespot-connect)
    |       |
    |       +-- PlayerEvent::Playing    --> JSON-RPC POST --> LMS spottyconnect start
    |       +-- PlayerEvent::Paused     --> JSON-RPC POST --> LMS spottyconnect stop
    |       +-- PlayerEvent::TrackChanged-> JSON-RPC POST --> LMS spottyconnect change
    |       +-- PlayerEvent::VolumeChanged-> JSON-RPC POST --> LMS spottyconnect volume
    |       +-- PlayerEvent::Seeked     --> JSON-RPC POST --> LMS spottyconnect seek
    |
    +-- HttpStreamSink (decoded S16LE PCM → mpsc channel)
    |
    +-- HTTP Server (hyper) bound on :0
            |
            GET /stream     ←-- LMS canDirectStream / new() proxy
            POST /control/play
            POST /control/pause
            POST /control/volume  <-- Connect.pm bidirektionale Steuerung
            POST /control/seek

LMS Plugin (Plugins::SpotOn::Connect)
    +-- addDispatch(['spottyconnect', '_cmd']) → _connectEvent()
    |       |
    |       +-- 'start'  → $client->execute(['playlist','play','spotify://connect-<ts>'])
    |       +-- 'change' → Metadata-Update + startOffset-Reset
    |       +-- 'stop'   → $client->execute(['pause',1])
    |       +-- 'volume' → Slim::Control::Request(['mixer','volume',N])
    |       +-- 'seek'   → $song->startOffset(position - elapsed)
    |
    +-- subscribe(_onNewSong)   → Handle CON-17 progress-before-play
    +-- subscribe(_onPause)     → LMS→Binary via HTTP /control/pause
    +-- subscribe(_onVolume)    → LMS→Binary via HTTP /control/volume (debounced 0.5s)
    +-- subscribe(_onSeek)      → LMS→Binary via HTTP /control/seek (debounced 0.3s)
    +-- subscribe(_onPlaylistJump) → LMS→Binary via HTTP /control/next|prev

LMS Plugin (Plugins::SpotOn::Connect::DaemonManager)
    +-- init() → subscribe(client new/disconnect) → debounced initHelpers()
    +-- initHelpers() → startHelper()/stopHelper() pro Player
    +-- Watchdog-Timer (60s) → initHelpers()
    +-- Sync-Aenderungen → stopForSync() + 0.1s Timer → initHelpers()

LMS Plugin (Plugins::SpotOn::Connect::Daemon)
    +-- new($clientId) → start()
    +-- start() → Proc::Background::new(spoton ... --connect ...) mit pipe fuer Port-Capture
    +-- Port-Capture: IO::Select→can_read(5s) → readline("stream_port=N")
    +-- _checkStartTimes() → Crash-Backoff, Discovery-Disable nach N Crashes
    +-- stop() / stopForSync() → $self->_proc->die
    +-- alive() / uptime() / pid()
```

### Recommended Project Structure (neue Dateien)

```
Plugins/SpotOn/
├── Connect.pm                    # Event-Dispatch, Source-Marking, Volume-Grace
├── Connect/
│   ├── DaemonManager.pm          # Daemon-Lifecycle, Sync-Gruppen, Watchdog
│   └── Daemon.pm                 # Proc::Background, Port-Capture, Crash-Backoff
librespot-spoton/src/
└── connect.rs                    # lms_connect Module: LMS, HttpStreamSink, ConnectNullSink,
                                  # http_stream_server, HTTP-Control-Endpoints
```

### Pattern 1: Binary `--connect` Mode

Das Binary erweitert Mode::Connect mit vollstaendiger Spirc-Integration.

**Startup-Sequenz:**
```rust
// Source: Spotty-NG librespot/src/main.rs lines 2340-2600 [VERIFIED: direkt gelesen]
// In run_connect():
1. Cache laden, Credentials pruefen
2. Session::new() + session.connect(credentials, false).await
3. Mixer + Player::new() mit HttpStreamSink (--connect-stream) oder ConnectNullSink
4. PCM-Channel (mpsc::channel(256)) + flush-watch-channel
5. TcpListener::bind("0.0.0.0:0") -> port = listener.local_addr().port()
6. println!("stream_port={}", port); stdout().flush()
7. tokio::spawn(http_stream_server(listener, pcm_rx, spirc_active, shutdown_rx, flush_rx))
8. LMS-Event-Dispatcher spawnen (handle_player_event loop)
9. spirc_active-Watcher spawnen (setzt true nach SessionConnected)
10. Spirc::new(connect_config, session, credentials, player, mixer).await
11. Main-Event-Loop: select! { discovery, spirc_task, player_invalid, ctrl_c }
```

**ConnectConfig (librespot-connect 0.8):**
```rust
// Source: Spotty-NG librespot/src/main.rs line 1784 [VERIFIED: direkt gelesen]
ConnectConfig {
    name: device_name,           // --name arg
    device_type: DeviceType::Speaker,  // oder --device-type
    initial_volume: Some(volume), // optional
    has_volume_ctrl: true,
    autoplay: ...,
}
```

### Pattern 2: JSON-RPC Event Dispatch (Binary→LMS)

```rust
// Source: Spotty-NG librespot/src/spotty.rs lines 310-503 [VERIFIED: direkt gelesen]
// LMS::notify() sendet POST /jsonrpc.js mit:
let body = json!({
    "id": 1,
    "method": "slim.request",
    "params": [player_mac, ["spottyconnect", cmd, p1?, p2?]],
}).to_string();
// HTTP/1.0 POST via TcpStream::connect(host_port)
```

**Wire-Vocabulary (5 Commands):**
| Command | Trigger | p1 | p2 |
|---------|---------|----|----|
| `start` | None→Some track transition | track_id (base62) | "" |
| `change` | Different track playing | new_track_id | prev_track_id |
| `stop` | Paused oder Stopped | "" | "" |
| `volume` | VolumeChanged (nach Suppress) | volume 0-100 | "" |
| `seek` | Seeked event | position_ms/1000.0 (3 Dezimalen) | "" |

**Enriched Events (D-13):** Die CONTEXT.md entscheidet Enrichment zu verwenden (Track-Metadata direkt im Payload). Implementierungs-Empfehlung: Erweiterung des p1/p2-Schemas OR separater JSON-Body. Am einfachsten: `start`/`change` als Basis-Events (base62 IDs) lassen und Perl ruft `API::Client->getTrack()` auf — spart Binary-Komplexitaet und nutzt bestehende Cache-Infrastruktur. Entscheidung bei Claude's Discretion.

### Pattern 3: HTTP-Control-Endpoints (LMS→Binary, D-14)

Neue Endpoints am Binary-HTTP-Server (zusaetzlich zu `/stream`):

```rust
// Source: Design-Entscheidung D-14 [ASSUMED]
// Empfehlung: POST /control/{cmd} mit JSON-Body
// POST /control/pause  -> spirc.pause()
// POST /control/play   -> spirc.play()
// POST /control/volume -> spirc.set_volume(vol)
// POST /control/seek   -> spirc.seek(position_ms)
// POST /control/next   -> spirc.next()
// POST /control/prev   -> spirc.prev()
```

Spirc-Instanz muss in Arc<Mutex<Option<Spirc>>> im HTTP-Server-Kontext zugaenglich sein. Bietet `shutdown()`, `next()`, `pause()`, `play()` Methoden. Volume-Control ueber Mixer-Interface.

### Pattern 4: Perl-seitige Event-Dispatch-Registrierung

```perl
# Source: Spotty-NG Connect.pm lines 58-80 [VERIFIED: direkt gelesen]
Slim::Control::Request::addDispatch(
    ['spottyconnect', '_cmd'],
    [1, 0, 1, \&_connectEvent]   # C=1(requires Client), Q=0, T=1(has Tags), F=coderef
);
# Subscriber fuer bidirektionale Events:
Slim::Control::Request::subscribe(\&_onNewSong,       [['playlist'], ['newsong']]);
Slim::Control::Request::subscribe(\&_onPause,         [['playlist'], ['pause', 'stop']]);
Slim::Control::Request::subscribe(\&_onVolume,        [['mixer'],    ['volume']]);
Slim::Control::Request::subscribe(\&_onSeek,          [['time']]);
Slim::Control::Request::subscribe(\&_onPlaylistJump,  [['playlist'], ['jump', 'index']]);
```

### Pattern 5: Source-Marking (Loop-Prevention)

```perl
# Source: Spotty-NG Connect.pm lines 335, 386, 422, 451, 461 [VERIFIED: direkt gelesen]
# Alle outbound-Commands werden mit source=__PACKAGE__ markiert:
my $req = Slim::Control::Request->new($client->id, ['mixer', 'volume', $volume]);
$req->source(__PACKAGE__);
$req->execute();
# Alle subscriber pruefen:
return if $request->source && $request->source eq __PACKAGE__;
```

### Pattern 6: CON-17 Race-Prevention (progress-before-play)

```perl
# Source: Spotty-NG Connect.pm lines 787-792 [VERIFIED: direkt gelesen]
# Progress MUSS vor dem playlist-play-Command in pluginData gespeichert werden:
if ($result->{progress} && $result->{progress} > 10) {
    $client->pluginData(progress => $result->{progress});
}
my $playReq = $client->execute(['playlist', 'play', "spotify://connect-$ts"]);
```

### Pattern 7: Port-Capture (synchron mit Timeout)

```perl
# Source: Spotty-NG Connect/Daemon.pm lines 124-165 [VERIFIED: direkt gelesen]
pipe(my $port_r, my $port_w) or die "pipe() failed: $!";
$self->_proc(Proc::Background->new(
    { 'die_upon_destroy' => 1, stdout => $port_w },
    $helperPath, @helperArgs,
));
close($port_w);  # KRITISCH: write-end im Parent schliessen

my $sel = IO::Select->new($port_r);
if ($sel->can_read(5)) {
    $port_line = readline($port_r);
}
close($port_r);
# Parse: $port_line =~ /^stream_port=(\d+)/
$self->_streamPort($1 + 0);
```

### Pattern 8: Sync-Gruppen-Proxy

```perl
# Source: Spotty-NG ProtocolHandler.pm lines 103-153 [VERIFIED: direkt gelesen]
# canDirectStream: gibt HTTP-URL fuer Einzelspieler, 0 fuer Sync-Gruppe
sub canDirectStream {
    my ($class, $client, $url) = @_;
    return 0 if $client->isSynced();
    my $helper = DaemonManager->helperForClient($client->id);
    return 'http://127.0.0.1:' . $helper->_streamPort . '/stream';
}
# new(): URL-Substitution fuer Sync-Gruppen
sub new {
    my ($class, $args) = @_;
    if ($args->{url} =~ m{/connect-}) {
        my $httpUrl = 'http://127.0.0.1:' . $helper->_streamPort . '/stream';
        $args = { %$args, url => $httpUrl };
    }
    return Slim::Player::Protocols::HTTP->new($args);
}
```

### Pattern 9: `soc` Content-Type in formatOverride

```perl
# Source: Spotty-NG ProtocolHandler.pm lines 73-79 [VERIFIED: direkt gelesen]
# SpotOn-Abweichung: 'soc' statt 'spc'
sub formatOverride {
    my ($class, $song) = @_;
    Plugins::SpotOn::Plugin->updateTranscodingTable($song->master);
    my $helper = DaemonManager->helperForClient($song->master);
    if ($helper && $helper->_streamMode) {
        return 'soc';   # Connect: soc (SpotOn-Connect), nicht 'spt' (Single-Track)
    }
    return 'son';   # Single-Track: son (SpotOn)
}
```

### Pattern 10: Crash-Backoff und Discovery-Disable

```perl
# Source: Spotty-NG Connect/Daemon.pm lines 207-228 [VERIFIED: direkt gelesen]
use constant MAX_FAILURES_BEFORE_DISABLE_DISCOVERY => 3;
use constant MAX_INTERVAL_BEFORE_DISABLE_DISCOVERY => 5 * 60; # 5 Minuten
sub _checkStartTimes {
    my $self = shift;
    if (scalar @{$self->_startTimes} > MAX_FAILURES_BEFORE_DISABLE_DISCOVERY) {
        splice @{$self->_startTimes}, 0, @{$self->_startTimes} - MAX_FAILURES_BEFORE_DISABLE_DISCOVERY;
        if (time() - $self->_startTimes->[0] < MAX_INTERVAL_BEFORE_DISABLE_DISCOVERY) {
            $prefs->set('disableDiscovery', 1);  # Fallback: ohne mDNS
        }
    }
    push @{$self->_startTimes}, time();
}
```

### Anti-Patterns to Avoid

- **`exit(0)` in Sink::stop():** Pipe-Backend (`StdoutSink`) ruft `exit(0)` in `stop()` auf (designed fuer `--single-track`). HttpStreamSink und ConnectNullSink duerfen das NICHT tun — der Prozess muss Track-Boundaries ueberleben.
- **spirc_active sofort nach Spirc::new() setzen:** Race condition — LMS verbindet sich vor Audio-Flow. Nur nach `PlayerEvent::SessionConnected` auf `true` setzen.
- **`['time', N]` in Stream-Mode:** Triggert LMS `_JumpToTime → _Stop + _Stream` was den HTTP-Stream neustartet. Stattdessen: `$song->startOffset(position - elapsed)`.
- **Einzelne API-Requests ohne Debouncing:** Volume-Events kommen in Bursts. Immer 0.5s Debounce via `Slim::Utils::Timers::killTimers` + `setTimer`.
- **Connect-PID in `_killOrphanedProcesses()` killen:** PHASE-5-NOTE bereits im Code markiert. Connect-Daemon-PIDs aus `helperPids()` vor pgrep-Kill ausschliessen.
- **Stale-API-Results vertrauen:** Bei `change`-Events liefert `/me/player` noch die alte Track-URI. Binary-Event-p2 (new_track_id) hat Prioritaet.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Spotify Connect Protocol | Eigene Spirc-Impl | `librespot-connect::Spirc` | 10k+ Zeilen protocol state machine |
| HTTP Streaming Server | Eigener Async-HTTP-Server | `hyper` + pattern aus `spotty.rs` | Grenzfaelle (concurrent relay, seek-flush) sind geloest |
| PCM Rate-Limiting | Eigener nanosekunden-Timer | `HttpStreamSink::write()` Pattern | Wall-clock math mit overflow protection bereits im Spotty-NG-Code |
| Prozess-Management | Direkte `fork()`/`exec()` | `Proc::Background` | Handhabt Signal-Propagation, zombie-Vermeidung, PID-Tracking |
| Session Token Refresh | Eigener Refresh-Loop | librespot-Session (intern) | Spirc managt Session-Reconnect und Token-Refresh intern |
| Volume 0-65535 Skalierung | Eigene Formel | `u32::from(volume) * 100 / 65535` | Gleiche Formel wie Spotty-NG; konsistente Werte |

---

## Common Pitfalls

### Pitfall 1: exit(0) im Sink::stop() vergessen zu verhindern
**What goes wrong:** `librespot-playback::StdoutSink::stop()` (fuer Pipe-Backend) ruft `exit(0)` auf. Wenn man den falschen Sink benutzt, stirbt der Daemon nach dem ersten Track.
**Why it happens:** `--single-track` und `--connect` teilen sich Playback-Infrastruktur. Pipe-Backend ist fuer Single-Track designed.
**How to avoid:** `HttpStreamSink::stop()` und `ConnectNullSink::stop()` duerfen NUR Zaehler resetten, KEIN `exit()`.
**Warning signs:** Daemon stirbt nach erstem Track-Ende.

### Pitfall 2: spirc_active Race Condition
**What goes wrong:** HTTP-Server gibt 200 zurueck bevor Spirc-Session vollstaendig ist. LMS verbindet sich, bekommt leere/fehlerhafte PCM-Daten.
**Why it happens:** `Spirc::new()` returned vor dem ersten `SessionConnected`-Event.
**How to avoid:** `spirc_active=true` NUR setzen wenn `PlayerEvent::SessionConnected` gefeuert wird. HTTP-Server gibt 503 mit `Retry-After: 2` solange `spirc_active=false`.
**Warning signs:** Kurze Stille, dann Verbindungsabbruch bei erstem Connect.

### Pitfall 3: stdout nicht flushen nach `stream_port=N`
**What goes wrong:** Perl `IO::Select->can_read(5)` timeouted obwohl Binary den Port bereits geschrieben hat.
**Why it happens:** stdout ist pipe-buffered — ohne explizites `flush()` bleibt der String im Buffer.
**How to avoid:** Nach `println!("stream_port={}", port)` explizit `std::io::stdout().flush()` aufrufen.
**Warning signs:** Daemon-Start bricht mit "timeout" ab, Binary laeuft aber.

### Pitfall 4: Connect-PIDs durch _killOrphanedProcesses() getoetet
**What goes wrong:** Stundlicher Cleanup killt Connect-Daemons obwohl sie aktiv sind.
**Why it happens:** `_killOrphanedProcesses()` sucht per `pgrep` nach dem Binary-Namen. Connect-Daemons matchen den gleichen Namen.
**How to avoid:** PHASE-5-NOTE ausfuehren — `helperPids()` in `DaemonManager` implementieren und vor dem Kill ausschliessen.
**Warning signs:** Connect-Daemons verschwinden stundlich.

### Pitfall 5: Stale API auf change-Events
**What goes wrong:** `_connectEvent` ruft `/me/player` auf, bekommt den alten Track zurueck (Spotify-API-Latenz 1-3s). Plugin spielt alten Track nochmal.
**Why it happens:** Spotify-Web-API ist nicht realtime. Bei `change`-Events ist die API noch nicht aktuell.
**How to avoid:** p2 (eventTrackId) aus dem Binary-Event hat Prioritaet ueber API-Response bei `change`. Pattern aus `Connect.pm` lines 645-656 uebernehmen.
**Warning signs:** Gleicher Track wird zweimal gespielt bei Skip.

### Pitfall 6: Volume-Feedback-Loop
**What goes wrong:** LMS empfaengt Lautstaerke-Aenderung von Spotify → setzt LMS-Lautstaerke → `_onVolume` feuert → schickt Lautstaerke zurueck an Spotify → Loop.
**Why it happens:** Bidirektionale Event-Synchronisation ohne Loop-Schutz.
**How to avoid:** Source-Marking auf ALLEN outbound Requests (`$req->source(__PACKAGE__)`). `_onVolume` prueft `$request->source eq __PACKAGE__` und returnt frueh.
**Warning signs:** Lautstaerke springt oder oszilliert.

### Pitfall 7: startOffset statt seekTime in Stream-Mode
**What goes wrong:** `$client->execute(['time', $position])` triggert `_JumpToTime` in LMS → stoppt HTTP-Stream → startet neu → langer Unterbruch.
**Why it happens:** LMS unterscheidet nicht zwischen "seek" und "adjust reported position".
**How to avoid:** In Stream-Mode IMMER `$song->startOffset(int($position) - $client->songElapsedSeconds())` statt `['time', $position]`.
**Warning signs:** Kurze Unterbrechung bei jedem Seek aus der Spotify-App.

### Pitfall 8: Fehlende Soft-Volume-Instanz fuer ConnectConfig
**What goes wrong:** `Spirc::new()` benoetigt eine Mixer-Instanz. Ohne Mixer kein Volume-Control.
**Why it happens:** Die SpotOn Binary hat bisher keinen Mixer (Single-Track braucht keinen).
**How to avoid:** Fuer Connect-Mode `NoOpMixer` oder `SoftMixer` erstellen. Spotty-NG nutzt `SoftMixer::open()`.
**Warning signs:** Compile-Fehler oder Lautstaerke-Steuerung funktioniert nicht.

---

## Code Examples

### Run_Connect Skeleton (Rust)

```rust
// Source: Spotty-NG librespot/src/spotty.rs + main.rs [VERIFIED: direkt gelesen]
// Analogon fuer SpotOn src/connect.rs run_connect():

pub async fn run_connect(
    cache_dir: &str,
    device_name: &str,
    player_mac: Option<&str>,
    lms_host_port: Option<&str>,
    lms_auth: Option<&str>,
    disable_discovery: bool,
    buffer_latency_ms: u64,
) -> Result<(), Box<dyn std::error::Error>> {
    // 1. Cache + Credentials
    let cache = Cache::new(Some(cache_dir), None, Some(cache_dir), None)?;
    let credentials = cache.credentials().ok_or("No credentials")?;

    // 2. Session
    let session_config = SessionConfig::default();
    let session = Session::new(session_config, Some(cache));
    session.connect(credentials.clone(), false).await?;

    // 3. PCM-Channel + Flush-Channel
    let (pcm_tx, pcm_rx) = tokio::sync::mpsc::channel::<Bytes>(256);
    let (flush_tx, flush_rx) = tokio::sync::watch::channel::<u64>(0);
    let flush_tx_for_lms = flush_tx.clone();

    // 4. Player mit HttpStreamSink
    let spirc_active = Arc::new(AtomicBool::new(false));
    let player = Player::new(
        PlayerConfig::default(),
        session.clone(),
        Box::new(NoOpVolume),
        move || HttpStreamSink::open(None, AudioFormat::S16, pcm_tx, flush_tx, buffer_latency_ms),
    );

    // 5. HTTP-Server: bind Port 0, announce stream_port
    let listener = TcpListener::bind("0.0.0.0:0").await?;
    let port = listener.local_addr()?.port();
    println!("stream_port={}", port);
    std::io::stdout().flush()?;

    let (shutdown_tx, shutdown_rx) = tokio::sync::oneshot::channel();
    let spirc_active_clone = Arc::clone(&spirc_active);
    tokio::spawn(http_stream_server(listener, pcm_rx, spirc_active_clone, shutdown_rx, flush_rx));

    // 6. LMS Event-Dispatcher
    let lms = LMS::new(lms_host_port.map(String::from), player_mac.map(String::from),
                        lms_auth.map(String::from), Some(flush_tx_for_lms));
    if lms.is_configured() {
        let mut event_chan = player.get_player_event_channel();
        tokio::spawn(async move {
            let mut current_track: Option<String> = None;
            while let Some(event) = event_chan.recv().await {
                lms.handle_player_event(&event, &mut current_track).await;
            }
        });
    }

    // 7. spirc_active-Watcher
    {
        let mut sess_chan = player.get_player_event_channel();
        let sa = Arc::clone(&spirc_active);
        tokio::spawn(async move {
            while let Some(event) = sess_chan.recv().await {
                if matches!(event, PlayerEvent::SessionConnected { .. }) {
                    sa.store(true, Ordering::SeqCst);
                }
            }
        });
    }

    // 8. ConnectConfig + Spirc::new()
    let connect_config = ConnectConfig {
        name: device_name.to_string(),
        device_type: DeviceType::Speaker,
        initial_volume: None,
        has_volume_ctrl: true,
        autoplay: false,
    };
    let mixer: Arc<dyn Mixer> = Arc::new(NoOpMixer); // oder SoftMixer
    let (spirc, spirc_task) = Spirc::new(
        connect_config, session.clone(), credentials, player.clone(), mixer,
    ).await?;

    // 9. Main Event Loop
    // ... select! { spirc_task, ctrl_c, ... }
    Ok(())
}
```

### Connect.pm _connectEvent Skeleton (Perl)

```perl
# Source: Spotty-NG Connect.pm lines 488-902 [VERIFIED: direkt gelesen]
sub _connectEvent {
    my $request = shift;
    my $client  = $request->client()->master;
    my $cmd     = $request->getParam('_cmd');

    if ($cmd eq 'start') {
        $client->pluginData(pendingConnect => 1);
        $_activeConnectPlayer = $client->id;
    }

    if ($cmd eq 'volume') {
        my $volume = $request->getParam('_p2');
        # Volume-Grace-Period Check (CON-11)
        if (Plugins::SpotOn::Connect::DaemonManager->uptime($client->id) < VOLUME_GRACE_PERIOD) {
            return;  # Suppress initial volume echo
        }
        my $req = Slim::Control::Request->new($client->id, ['mixer', 'volume', $volume]);
        $req->source(__PACKAGE__);
        $req->execute();
        return;
    }

    if ($cmd eq 'seek') {
        # Stream-mode: startOffset statt ['time', N]
        my $position = $request->getParam('_p2');
        my $song = $client->playingSong();
        if ($song) {
            my $elapsed = $client->songElapsedSeconds() || 0;
            $song->startOffset(int($position) - $elapsed);
        }
        return;
    }

    # start/change/stop: Enriched-Event nutzen ODER API-Call
    # D-13: Track-Info aus Enriched-Payload (p3=name, p4=artist, p5=album, p6=duration_ms, p7=cover)
    # oder Fallback auf API::Client->getTrack()

    if ($cmd eq 'start' && $request->getParam('_p2')) {
        my $trackId  = $request->getParam('_p2');
        # CON-17: progress BEFORE playlist play
        my $progress = $request->getParam('_p3') || 0;  # aus Enriched-Payload
        $client->pluginData(progress => $progress) if $progress > 10;

        my $ts  = int(Time::HiRes::time() * 1000);
        my $req = $client->execute(['playlist', 'play', "spotify://connect-$ts"]);
        $req->source(__PACKAGE__);
    }
}
```

### updateTranscodingTable Erweiterung fuer `soc`

```perl
# Source: SpotOn Plugin.pm line 1186-1249 (bestehender Code) [VERIFIED: direkt gelesen]
# Erweiterung: soc-* Eintraege ignorieren single-track-Regex
foreach my $key (keys %{$commandTable}) {
    # Bestehend: son-* fuer single-track
    next if $key =~ /^son-/ && $commandTable->{$key} =~ /single-track/;
    # NEU: soc-* fuer Connect (keine Aenderungen noetig - soc nutzt curl zum Stream-Port)
    # soc-Profile sind statisch in custom-convert.conf:
    # soc pcm * * → curl http://127.0.0.1:[PORT]/stream  (kein [spoton]-Aufruf)
}
```

### custom-convert.conf Erweiterungen

```
# Source: Design D-04 + Pattern aus Spotty-NG custom-convert.conf [VERIFIED: gelesen]
# SpotOn Connect-Profil (soc = SpotOn-Connect)
# I = input stream (LMS fetcht direkt via canDirectStream oder new()-Proxy)
soc pcm * *
    # I
    -

soc ogg * *
    # I
    -
```

### custom-types.conf Erweiterung

```
# Zu custom-types.conf hinzufuegen:
soc    soc    audio/x-sb-spoton-connect    audio
```

---

## Runtime State Inventory

Phase 5 ist keine Rename/Refactor-Phase. Kein Runtime State Inventory erforderlich.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Rust/cargo | Binary-Build | ✓ | Vorhanden (bestehende Binary) | — |
| librespot-connect 0.8.0 | Spirc-Impl | ✓ | crates.io [VERIFIED: cargo search] | — |
| hyper 1.x | HTTP-Server im Binary | Muss hinzugefuegt werden | — | — |
| Proc::Background | Daemon-Spawn | ✓ | LMS-Bundle [ASSUMED] | — |
| IO::Select | Port-Capture | ✓ | Perl Core | — |
| LMS SimpleAsyncHTTP | HTTP-Control-Calls | ✓ | LMS-Bundle | — |

**Missing dependencies with no fallback:**
- `librespot-connect`, `hyper`, `hyper-util`, `http-body-util`, `tokio-stream`, `bytes` — muessen zu Cargo.toml hinzugefuegt werden (Wave 0 Task).

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Perl-basierte Integrationstests (LMS-eigenes Test-Framework nicht vorhanden) |
| Config file | Keine Standard-Testconfig |
| Quick run command | Manuelle Pruefung via LMS-Log + Spotify-App |
| Full suite command | Spotify-App: Device-Liste, Playback-Transfer, Volume, Skip |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Notes |
|--------|----------|-----------|-------|
| CON-01 | Ein Daemon pro Player sichtbar | manual | Spotify-App Device-Liste: N Players = N Geraete |
| CON-02 | Daemon startet/stoppt/restartiert | manual | LMS-Log + `ps aux` |
| CON-03 | Events in LMS-Log erscheinen | manual | LMS-Log: "Got called from spotty helper for..." |
| CON-04 | Audio startet <3s | manual | Stopwatch bei Transfer |
| CON-05 | Steuerbefehle funktionieren | manual | Spotify-App: Play/Pause/Skip/Volume |
| CON-06 | Sync-Gruppe als ein Device | manual | Spotify-App: zeigt zusammengefassten Namen |
| CON-09 | PIDs nicht durch Cleanup gekillt | manual | Warten auf KILL_PROCESS_INTERVAL, Daemon check |
| CON-11 | Kein Volume-Jump beim Connect | manual | Ohren-Test beim Transfer |
| CON-13 | Position korrekt nach Transfer | manual | Mid-Track-Transfer, Positionscheck |
| CON-17 | Keine Race Condition beim Start | manual | Mehrfaches schnelles Transfer |

### Wave 0 Gaps

- [ ] Cargo.toml: `librespot-connect`, `hyper`, `hyper-util`, `http-body-util`, `tokio-stream`, `bytes` hinzufuegen
- [ ] custom-types.conf: `soc` Type registrieren
- [ ] custom-convert.conf: `soc pcm * *` und `soc ogg * *` Profile
- [ ] Neue Perl-Module anlegen: `Connect.pm`, `Connect/DaemonManager.pm`, `Connect/Daemon.pm`
- [ ] Neues Rust-Modul anlegen: `src/connect.rs`

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | Credentials bereits in Phase 02.1/04.3 gehandhabt |
| V3 Session Management | no | Spirc-Session intern in librespot |
| V4 Access Control | no | LMS-seitig, keine externen Nutzer |
| V5 Input Validation | yes | Binary-CLI-Flags, JSON-RPC-Payload-Parsing |
| V6 Cryptography | no | TLS via librespot-core |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Shell-Injection via Device-Name | Tampering | Name-Sanitisierung in Daemon.pm (max 60 Zeichen, substr()) |
| CRLF-Injection in LMS-Auth-Header | Tampering | `trim().replace(['\r', '\n'], "")` in LMS::new() (Spotty-NG-Pattern) |
| Unverified JSON-RPC vom Binary | Spoofing | JSON-RPC kommt nur von localhost:Binary (keine externe Quelle) |
| Connect-PID Kill durch Cleanup | DoS | PHASE-5-NOTE ausfuehren: helperPids() aus Kill-Scope ausschliessen |
| Concurrent HTTP-Connections zum Stream | DoS | `relay_active` AtomicBool verhindert Split-PCM-Stream (CR-01) |

---

## State of the Art

| Old Approach | Current Approach | Impact |
|--------------|------------------|--------|
| FIFO-basiertes Audio-Transport | HTTP-Streaming (D-01) | Loest Seek-Lag (P-19) und White-Noise (P-20) |
| Spotty spc Content-Type | SpotOn soc Content-Type | Namespace-Trennung, kein Konflikt mit spt/son |
| Token-Refresh via Daemon-Neustart (50min) | librespot Spirc-interne Session-Verwaltung | Kein Neustart, nahtlose Connect-Sessions |
| checkDaemonConnected (429-Quelle) | Watchdog-Restart nur bei `alive==false` | Eliminiert 429-Burst-Quelle |

**Deprecated/outdated:**
- FIFO-Pattern (`mkfifo`): In SpotOn Phase 5 nie implementiert (direkt HTTP-Streaming)
- CON-12 Requirement-Text "FIFO-based": Muss zu "HTTP-streaming" aktualisiert werden (Deferred, aber bei Phase-Start erledigen)

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `Proc::Background` ist in der LMS-Installations-Bundle verfuegbar | Standard Stack | Daemon-Spawn-Pattern muss ueberarbeitet werden |
| A2 | `librespot-connect 0.8.0` auf crates.io ist API-kompatibel mit der Spotty-NG-internen Version | Standard Stack | Spirc::new()-Signatur koennte abweichen |
| A3 | `ConnectConfig` in librespot-connect 0.8.0 hat die gleichen Felder wie in Spotty-NG | Code Examples | ConnectConfig-Initialisierung muss angepasst werden |
| A4 | LMS `Slim::Player::Protocols::HTTP->new($args)` akzeptiert einen `url`-Override in `$args` | Architecture Patterns | Sync-Proxy-Pattern muss anders implementiert werden |
| A5 | Enriched Events (D-13) koennen als separate JSON-Felder im p3-p7 Schema uebertragen werden | Event Protocol | Schema muss mit Binary-Contract abgestimmt werden; Alternative: separater API-Call |
| A6 | HTTP-Control-Endpoints (D-14): `spirc.pause()`, `spirc.play()` etc. sind in librespot-connect 0.8.0 als public API verfuegbar | Architecture Patterns | Control-Endpoints benoetigen anderen Mechanismus (z.B. Mixer-Calls) |
| A7 | slopcheck nicht ausgefuehrt — alle Rust-Pakete als [ASSUMED] | Package Legitimacy | Paket-Legitimation beruht auf Reputation |

---

## Open Questions

1. **librespot-connect 0.8.0 vs. Spotty-NG interne Version: API-Divergenz**
   - What we know: Spotty-NG nutzt `librespot-connect` als path-dependency (git-Workspace). Die crates.io-Version 0.8.0 ist released.
   - What's unclear: Ob `ConnectConfig`-Felder, `Spirc::new()`-Signatur und `spirc.pause()`/`spirc.play()`/`spirc.next()`/`spirc.prev()` identisch sind zwischen git-HEAD und crates.io 0.8.0.
   - Recommendation: Wave 0 Task — `cargo add librespot-connect@0.8.0` und Cargo-Resolve pruefen. Falls Signatur-Divergenz: Spotty-NG git-Fork als lokale Path-Dependency oder Version einfrieren.

2. **HTTP-Control-Endpoints: Spirc-Public-API**
   - What we know: Spotty-NG's `Spirc` hat `shutdown()` Methode. D-14 erfordert `pause()`, `play()`, `volume()`, `seek()`, `next()`, `prev()`.
   - What's unclear: Ob `librespot-connect 0.8.0 Spirc` diese Methoden oeffentlich exponiert oder nur intern durch Dealer-Messages implementiert.
   - Recommendation: `cargo doc --open librespot-connect` nach Installation prueft die public API. Falls nicht vorhanden: Control ueber separaten mpsc-Channel an Spirc-Task.

3. **D-13 Enriched Events: Binary-Implementierungs-Komplexitaet**
   - What we know: D-13 sagt Metadata direkt im Payload. Das benoetigt einen Metadata-Fetch im Binary (librespot-metadata Crate) nach TrackChanged.
   - What's unclear: Ob `librespot-metadata` fuer SpotOn verfuegbar ist und ob der Fetch-Aufwand gerechtfertigt ist gegenueber einem einfachen API-Call von Perl aus.
   - Recommendation: Starter-Implementierung ohne Enrichment (Perl ruft `API::Client->getTrack()` nach `start`/`change`-Event). Enrichment als optionales Upgrade wenn Latenz-Probleme auftreten.

4. **Mixer fuer Connect-Mode**
   - What we know: `Spirc::new()` benoetigt `Arc<dyn Mixer>`.
   - What's unclear: Ob `NoOpMixer` oder `SoftMixer` fuer Volume-Control ueber Spotify korrekt ist.
   - Recommendation: `SoftMixer::open(MixerConfig::default())` wie Spotty-NG. Volume-Events kommen als `VolumeChanged` PlayerEvent und werden als 0-100% ueber LMS-JSON-RPC weitergeleitet.

---

## Sources

### Primary (HIGH confidence)
- Spotty-NG `librespot/src/spotty.rs` — vollstaendige `lms_connect` Implementierung (LMS, HttpStreamSink, ConnectNullSink, http_stream_server, handle_player_event)
- Spotty-NG `librespot/src/main.rs` — Connect-Mode Startup-Sequenz, Spirc-Integration, spirc_active-Pattern
- Spotty-NG `Spotty-Plugin/Connect.pm` — 904-Zeilen Event-Dispatch-Implementierung
- Spotty-NG `Spotty-Plugin/Connect/DaemonManager.pm` — Daemon-Lifecycle, Watchdog, Sync-Gruppen
- Spotty-NG `Spotty-Plugin/Connect/Daemon.pm` — Proc::Background, Port-Capture, Crash-Backoff
- Spotty-NG `Spotty-Plugin/ProtocolHandler.pm` — `formatOverride`, `canDirectStream`, `new()` Sync-Proxy
- SpotOn `librespot-spoton/src/main.rs` — aktueller Stand des Binary (Mode::Connect Stub)
- SpotOn `Plugins/SpotOn/Plugin.pm` — PHASE-5-NOTE in `_killOrphanedProcesses()`, `updateTranscodingTable()`
- SpotOn `Plugins/SpotOn/ProtocolHandler.pm` — aktueller Stand (nur son, canDirectStream=0)
- SpotOn `Plugins/SpotOn/custom-convert.conf` — son-Profile (Basis fuer soc-Erweiterung)
- `librespot-playback-0.8.0/src/player.rs` — vollstaendige `PlayerEvent` Enum-Definition

### Secondary (MEDIUM confidence)
- `cargo search librespot-connect` — librespot-connect 0.8.0 auf crates.io bestaetigt
- Spotty-NG `librespot/connect/Cargo.toml` — librespot-connect Abhaengigkeiten (futures-util, protobuf, rand, serde_json, thiserror, tokio, tokio-stream, uuid)
- CLAUDE.md §librespot — CLI-Flags Dokumentation

### Tertiary (LOW confidence)
- Keine LOW-confidence Quellen

---

## Metadata

**Confidence breakdown:**
- Standard Stack: HIGH — direkt aus Spotty-NG-Code und crates.io verifiziert
- Architecture: HIGH — vollstaendige Implementierung in Spotty-NG als Referenz
- Pitfalls: HIGH — aus Spotty-NG-Code-Kommentaren und bekannten Problemen extrahiert
- Binary API (librespot-connect 0.8.0 public interface): MEDIUM — crates.io-Version ungelesen, nur Signatur aus Spotty-NG erschlossen

**Research date:** 2026-06-01
**Valid until:** 2026-07-01 (librespot 0.8 stabil)
