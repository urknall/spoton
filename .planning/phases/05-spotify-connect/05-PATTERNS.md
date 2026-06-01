# Phase 5: Spotify Connect - Pattern Map

**Mapped:** 2026-06-01
**Files analyzed:** 12 (7 new, 5 modified)
**Analogs found:** 12 / 12

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|---|---|---|---|---|
| `Plugins/SpotOn/Connect.pm` | service + event-dispatcher | event-driven | `/home/sti/spotty-ng/Spotty-Plugin/Connect.pm` | exact |
| `Plugins/SpotOn/Connect/DaemonManager.pm` | service + lifecycle | event-driven | `/home/sti/spotty-ng/Spotty-Plugin/Connect/DaemonManager.pm` | exact |
| `Plugins/SpotOn/Connect/Daemon.pm` | service + process-wrapper | event-driven + request-response | `/home/sti/spotty-ng/Spotty-Plugin/Connect/Daemon.pm` | exact |
| `librespot-spoton/src/connect.rs` | service + sink + HTTP-server | streaming + event-driven | `/home/sti/spotty-ng/librespot/src/spotty.rs` (lms_connect module) | exact |
| `Plugins/SpotOn/Plugin.pm` (modify) | plugin entrypoint | request-response | self (existing) | self |
| `Plugins/SpotOn/ProtocolHandler.pm` (modify) | protocol-handler | streaming | `/home/sti/spotty-ng/Spotty-Plugin/ProtocolHandler.pm` | exact |
| `librespot-spoton/src/main.rs` (modify) | binary entrypoint | request-response | self (existing) | self |
| `librespot-spoton/Cargo.toml` (modify) | config | N/A | self (existing) | self |
| `Plugins/SpotOn/Settings.pm` (modify) | settings | request-response | self (existing) | self |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (modify) | settings UI | request-response | self (existing) | self |
| `Plugins/SpotOn/custom-convert.conf` (modify) | config | streaming | `/home/sti/spotty-ng/Spotty-Plugin/custom-convert.conf` | exact |
| `Plugins/SpotOn/custom-types.conf` (modify) | config | N/A | `/home/sti/spotty-ng/Spotty-Plugin/custom-types.conf` | exact |

---

## Pattern Assignments

### `Plugins/SpotOn/Connect.pm` (service, event-driven)

**Analog:** `/home/sti/spotty-ng/Spotty-Plugin/Connect.pm`

**Namespace and imports pattern** (lines 1-17, adapt `Spotty` → `SpotOn`):
```perl
package Plugins::SpotOn::Connect;

use strict;

use File::Path qw(mkpath);
use File::Spec::Functions qw(catdir catfile);
use JSON::XS::VersionOneAndTwo;
use Scalar::Util qw(blessed);
use Time::HiRes;

use Slim::Utils::Log;
use Slim::Utils::Cache;
use Slim::Utils::Prefs;
use Slim::Utils::Timers;

use Plugins::SpotOn::API::Client;  # SpotOn uses API::Client, not API directly
```

**Constants pattern** (lines 19-31):
```perl
use constant SEEK_THRESHOLD   => 3;
use constant IMG_TRACK        => '/html/images/cover.png';
use constant VOLUME_GRACE_PERIOD => 20;    # CON-11
use constant CONNECT_START_GRACE => 12;

my $prefs       = preferences('plugin.spoton');   # spoton, not spotty
my $serverPrefs = preferences('server');
my $log         = logger('plugin.spoton');

my $initialized;
my $_activeConnectPlayer;
```

**Dispatch registration — init() pattern** (lines 47-82):
```perl
sub init {
    my ($class) = @_;
    return if $initialized;

    # C=1(requires Client), Q=0(not a query), T=1(has Tags), F=handler ref
    Slim::Control::Request::addDispatch(
        ['spottyconnect', '_cmd'],
        [1, 0, 1, \&_connectEvent]
    );

    Slim::Control::Request::subscribe(\&_onNewSong,      [['playlist'], ['newsong']]);
    Slim::Control::Request::subscribe(\&_onPause,        [['playlist'], ['pause', 'stop']]);
    Slim::Control::Request::subscribe(\&_onVolume,       [['mixer'],    ['volume']]);
    Slim::Control::Request::subscribe(\&_onSeek,         [['time']]);
    Slim::Control::Request::subscribe(\&_onPlaylistJump, [['playlist'], ['jump', 'index']]);

    require Plugins::SpotOn::Connect::DaemonManager;
    Plugins::SpotOn::Connect::DaemonManager->init();

    $initialized = 1;
}
```

**Source-marking pattern — loop prevention** (lines 451, 552-554, used throughout):
```perl
# ALL outbound commands MUST be source-marked to break feedback loops (Pitfall 6)
return if $request->source && $request->source eq __PACKAGE__;

# When issuing commands back to LMS:
my $req = Slim::Control::Request->new($client->id, ['mixer', 'volume', $volume]);
$req->source(__PACKAGE__);
$req->execute();
```

**_connectEvent dispatcher — volume + seek handlers** (lines 488-594):
```perl
sub _connectEvent {
    my $request = shift;
    my $client  = $request->client()->master;
    my $cmd     = $request->getParam('_cmd');

    # Claim active Connect ownership on 'start' (must be synchronous)
    if ($cmd eq 'start') {
        $client->pluginData(pendingConnect => 1);
        $_activeConnectPlayer = $client->id;
    }

    # Volume handler with grace period (CON-11)
    if ($cmd eq 'volume' && !($request->source && $request->source eq __PACKAGE__)) {
        my $volume = $request->getParam('_p2');
        return unless defined $volume && $volume ne '';
        if (Plugins::SpotOn::Connect::DaemonManager->uptime($client->id) < VOLUME_GRACE_PERIOD) {
            main::INFOLOG && $log->is_info && $log->info("Ignoring initial volume during grace period");
            return;
        }
        my $volReq = Slim::Control::Request->new($client->id, ['mixer', 'volume', $volume]);
        $volReq->source(__PACKAGE__);
        $volReq->execute();
        return;
    }

    # Seek handler (CON-13: startOffset, NOT ['time', N] in stream mode)
    if ($cmd eq 'seek') {
        my $position = $request->getParam('_p2');
        if (defined $position && $position ne '') {
            if ($client->pluginData('pendingConnect')) {
                $client->pluginData(progress => $position);
                $client->pluginData(pendingConnect => 0);
            } else {
                my $song = $client->playingSong();
                if ($song) {
                    my $elapsed = $client->songElapsedSeconds() || 0;
                    $song->startOffset(int($position) - $elapsed);  # CON-13 critical
                }
            }
        }
        return;
    }
}
```

**CON-17 race prevention — progress before play** (lines 787-792):
```perl
# Store progress BEFORE issuing playlist play (CON-17)
if ($result->{progress} && $result->{progress} > 10) {
    $client->pluginData(progress => $result->{progress});
}
my $ts  = int(Time::HiRes::time() * 1000);
my $req = $client->execute(['playlist', 'play', "spotify://connect-$ts"]);
$req->source(__PACKAGE__);
```

**Stale-API fallback on change events** (lines 645-656):
```perl
# On 'change': binary's event p2 (new_track_id) has priority over API response
if ($cmd eq 'change' && (my $eventTrackId = $request->getParam('_p2'))) {
    my $eventUri = "spotify:track:$eventTrackId";
    if (ref $result->{track} && $result->{track}->{uri}
        && $result->{track}->{uri} eq $streamUrl
        && $eventUri ne $streamUrl) {
        # Trust the binary, not the stale API
        $result->{track} = { uri => $eventUri };
    }
}
```

**SpotOn-specific deviations from Spotty-NG:**
- Use `Plugins::SpotOn::API::Client` instead of `Plugins::Spotty::API`
- HTTP control endpoints (D-14): `_onPause`, `_onVolume`, `_onSeek` POST to `http://127.0.0.1:$port/control/{cmd}` via `Slim::Networking::SimpleAsyncHTTP` instead of Spotify Web API
- Spotify Web API (`API::Client`) is only the fallback (D-15) when binary endpoint unreachable
- Content type is `soc` (not `spc`)
- prefs namespace: `plugin.spoton`

---

### `Plugins/SpotOn/Connect/DaemonManager.pm` (service, event-driven)

**Analog:** `/home/sti/spotty-ng/Spotty-Plugin/Connect/DaemonManager.pm`

**Imports and constants pattern** (lines 1-25):
```perl
package Plugins::SpotOn::Connect::DaemonManager;

use strict;
use Scalar::Util qw(blessed);
use Slim::Utils::Log;
use Slim::Utils::Prefs;
use Slim::Utils::Timers;
use Plugins::SpotOn::Plugin;
use Plugins::SpotOn::Connect::Daemon;

use constant DAEMON_INIT_DELAY        => 2;
use constant DAEMON_WATCHDOG_INTERVAL => 60;
use constant STREAM_WATCHDOG_INTERVAL => 5;   # fast poll for streaming daemons

my $prefs = preferences('plugin.spoton');
my $log   = logger('plugin.spoton');
my %helperInstances;
```

**init() — subscribe + debounced initHelpers** (lines 26-106):
```perl
sub init {
    my $class = shift;

    # Debounced init on client connect/disconnect
    Slim::Control::Request::subscribe(sub {
        Slim::Utils::Timers::killTimers($class, \&initHelpers);
        Slim::Utils::Timers::setTimer($class, Time::HiRes::time() + DAEMON_INIT_DELAY, \&initHelpers);
    }, [['client'], ['new', 'disconnect']]);

    # CON-15: Differential restart on sync changes (0.1s micro-delay, not 2s)
    Slim::Control::Request::subscribe(sub {
        my $request = shift;
        return if $request->isNotCommand([['sync']]);
        Slim::Utils::Timers::killTimers($class, \&initHelpers);
        foreach my $clientId (keys %helperInstances) {
            $helperInstances{$clientId}->stopForSync();
        }
        Slim::Utils::Timers::setTimer($class, Time::HiRes::time() + 0.1, \&initHelpers);
    }, [['sync']]);

    # React to per-player Connect toggle (D-10, CON-10)
    $prefs->setChange(\&initHelpers, 'enableSpotifyConnect');
}
```

**initHelpers() — sync-group master selection pattern** (lines 108-157):
```perl
sub initHelpers {
    my $class = __PACKAGE__;
    Slim::Utils::Timers::killTimers($class, \&initHelpers);
    $class->shutdown('inactive-only');

    for my $client (Slim::Player::Client::clients()) {
        my $syncMaster;
        if (Slim::Player::Sync::isSlave($client) && (my $master = $client->master)) {
            if ($prefs->client($master)->get('enableSpotifyConnect')) {
                $syncMaster = $master->id;
            }
        }

        if ($syncMaster && $syncMaster eq $client->id) {
            $class->startHelper($client);
        } elsif ($syncMaster) {
            $class->stopHelper($client);
        } elsif (!$syncMaster && $prefs->client($client)->get('enableSpotifyConnect')) {
            $class->startHelper($client);
        } else {
            $class->stopHelper($client);
        }
    }

    # 60s watchdog
    Slim::Utils::Timers::setTimer($class, Time::HiRes::time() + DAEMON_WATCHDOG_INTERVAL, \&initHelpers);
}
```

**helperPids() — CON-09 exclusion support** (lines 292-295):
```perl
# Called by Plugin.pm::_killOrphanedProcesses to exclude Connect daemon PIDs
sub helperPids {
    my $class = shift;
    return map { $_->pid } grep { $_->alive } values %helperInstances;
}
```

**helperForClient() and streamPortForClient()** (lines 271-285):
```perl
sub helperForClient {
    my ($class, $clientId) = @_;
    $clientId = $clientId->id if $clientId && blessed $clientId;
    return unless $clientId;
    return $helperInstances{$clientId};
}

sub streamPortForClient {
    my ($class, $clientId) = @_;
    $clientId = $clientId->id if $clientId && blessed $clientId;
    my $helper = $helperInstances{$clientId} || return;
    return $helper->_streamPort;
}
```

**SpotOn-specific deviations:**
- Prefs key `enableSpotifyConnect` under `plugin.spoton` namespace
- `checkDaemonConnected` block is NOT present (explicitly disabled per REQUIREMENTS.md — was 429 source)
- `forceFallbackAP` pref watch is NOT needed (SpotOn does not use AP-port fallback)
- Device name pattern: `Slim::Player::Sync::syncname($client)` (same as Spotty-NG, D-11)

---

### `Plugins/SpotOn/Connect/Daemon.pm` (service, event-driven + request-response)

**Analog:** `/home/sti/spotty-ng/Spotty-Plugin/Connect/Daemon.pm`

**Accessor setup and constants pattern** (lines 1-38):
```perl
package Plugins::SpotOn::Connect::Daemon;

use strict;
use base qw(Slim::Utils::Accessor);

use File::Spec::Functions qw(catdir);
use IO::Select;
use MIME::Base64 qw(encode_base64);
use Proc::Background;

use Slim::Utils::Log;
use Slim::Utils::Prefs;

use constant MAX_FAILURES_BEFORE_DISABLE_DISCOVERY => 3;
use constant MAX_INTERVAL_BEFORE_DISABLE_DISCOVERY => 5 * 60;
use constant MAX_STREAM_FAILURES  => 5;
use constant MAX_STREAM_INTERVAL  => 2 * 60;

__PACKAGE__->mk_accessor(rw => qw(
    id mac name cache
    _lastSeen _spotifyId _proc
    _startTimes _streamStartTimes
    _streamMode _streamPort
));
```

**new() constructor pattern** (lines 44-57):
```perl
sub new {
    my ($class, $id) = @_;
    my $self = $class->SUPER::new();
    $self->mac($id);
    $id =~ s/://g;
    $self->id($id);
    $self->_startTimes([]);
    $self->_streamStartTimes([]);
    $self->start();
    return $self;
}
```

**start() — device name, cache dir, flag args pattern** (lines 59-100):
```perl
sub start {
    my $self = shift;
    my $helperPath = Plugins::SpotOn::Helper->get();
    my $client     = Slim::Player::Client::getClient($self->mac);

    # D-11: truncate to 60 chars; use syncname for sync groups
    $self->name(substr(
        ($client->isSynced() && $client->model ne 'group')
            ? Slim::Player::Sync::syncname($client)
            : $client->name,
        0, 60
    ));

    # Per-account cache dir (same pattern as Plugin.pm::updateTranscodingTable)
    my $serverPrefs     = preferences('server');
    my $prefs           = preferences('plugin.spoton');
    my $activeAccountId = $prefs->client($client)->get('activeAccount')
                       || $prefs->get('activeAccount')
                       || '';
    my $cacheDir = $activeAccountId
        ? catdir($serverPrefs->get('cachedir'), 'spoton', $activeAccountId)
        : catdir($serverPrefs->get('cachedir'), 'spoton');
    $self->cache($cacheDir);

    my @helperArgs = (
        '-c', $self->cache,
        '-n', $self->name,
        '--disable-audio-cache',
        '--player-mac', $self->mac,
        '--lms', '127.0.0.1:' . $serverPrefs->get('httpport'),
        '--connect',        # SpotOn flag (Spotty-NG used '--connect-stream')
    );

    push @helperArgs, '--disable-discovery' if $prefs->get('disableDiscovery');
}
```

**Port-capture pipe pattern** (lines 122-165 — CON-03, CON-16):
```perl
# CRITICAL: close write-end in parent or readline blocks forever
pipe(my $port_r, my $port_w)
    or do { $log->error("pipe() failed for port capture: $!"); return; };

eval {
    $self->_proc(Proc::Background->new(
        { 'die_upon_destroy' => 1, stdout => $port_w },
        $helperPath,
        @helperArgs,
    ));
};
close($port_w);  # MUST close write-end in parent

if ($@ || !$self->_proc) {
    $log->warn("Failed to launch SpotOn Connect daemon: $@");
    close($port_r);
    $self->_streamPort(undef);
    return;
}

# IO::Select with 5s timeout (avoids SIGALRM in LMS event loop)
my $port_line;
my $sel = IO::Select->new($port_r);
if ($sel->can_read(5)) {
    $port_line = readline($port_r);
}
close($port_r);

if (!defined $port_line || $port_line !~ /^stream_port=(\d+)/) {
    my $reason = defined $port_line ? "unexpected output: $port_line" : "timeout";
    $log->warn("SpotOn daemon did not announce HTTP stream port ($reason) - aborting");
    $self->_proc->die if $self->_proc && $self->_proc->alive;
    $self->_streamPort(undef);
    return;
}
$self->_streamPort($1 + 0);
$self->_streamMode(1);
```

**_checkStartTimes() — crash-backoff pattern** (lines 206-228):
```perl
sub _checkStartTimes {
    my $self = shift;
    if (scalar @{$self->_startTimes} > MAX_FAILURES_BEFORE_DISABLE_DISCOVERY) {
        splice @{$self->_startTimes}, 0,
               @{$self->_startTimes} - MAX_FAILURES_BEFORE_DISABLE_DISCOVERY;
        if (time() - $self->_startTimes->[0] < MAX_INTERVAL_BEFORE_DISABLE_DISCOVERY
            && !preferences('plugin.spoton')->get('disableDiscovery'))
        {
            $log->warn(sprintf(
                'SpotOn daemon crashed %s times within less than %s minutes - disabling discovery.',
                MAX_FAILURES_BEFORE_DISABLE_DISCOVERY,
                MAX_INTERVAL_BEFORE_DISABLE_DISCOVERY / 60
            ));
            preferences('plugin.spoton')->set('disableDiscovery', 1);
        }
    }
    push @{$self->_startTimes}, time();
}
```

**stop() / stopForSync() / alive() / uptime() / pid()** (lines 256-321 — copy verbatim, change log strings):
```perl
sub stop {
    my $self = shift;
    if ($self->alive) {
        $self->_proc->die;
        $self->_streamPort(undef);
    }
}

sub stopForSync {
    my $self = shift;
    if ($self->alive) {
        $self->_proc->die;
        $self->_streamPort(undef);    # clear stale port; start() will set new one
        $self->_streamStartTimes([]);
        $self->_startTimes([]);
    }
}

sub pid    { my $self = shift; return $self->_proc && $self->_proc->pid }
sub alive  { my $self = shift; return 1 if $self->_proc && $self->_proc->alive }
sub uptime { my $self = shift; return Time::HiRes::time() - ($self->_startTimes->[-1] || time()) }
```

**SpotOn-specific deviations:**
- CLI flag is `--connect` (not `--connect-stream` as in Spotty-NG)
- Cache dir uses `plugin.spoton` prefs and `spoton` subdir (not `spotty`)
- No `rmtree` of cache on `stop()` — SpotOn keeps credentials across restarts
- `lms-auth` encoding identical: `encode_base64("$username:$password", '')` but log BEFORE adding `--lms-auth`
- `_checkStreamStartTimes()` same logic, disables stream mode (not discovery) after `MAX_STREAM_FAILURES`

---

### `librespot-spoton/src/connect.rs` (service + sink + HTTP-server, streaming + event-driven)

**Analog:** `/home/sti/spotty-ng/librespot/src/spotty.rs` (lms_connect module, lines ~215-1034)

**Module structure — copy the entire lms_connect module as connect.rs:**

The Spotty-NG `lms_connect` feature-gated module in `spotty.rs` (lines 215-1034) contains:
- `LMS` struct + `new()` + `is_configured()` + `Clone` impl
- `LMS::handle_player_event()` — PlayerEvent dispatcher
- `LMS::notify()` — JSON-RPC POST via TcpStream
- `HttpStreamSink` struct + `Sink` impl (`open()`, `start()`, `stop()`, `write()`)
- `http_stream_server()` — hyper HTTP/1.1 server for `/stream` endpoint

**LMS struct and constructor** (spotty.rs lines ~220-283):
```rust
pub struct LMS {
    host_port: Option<String>,
    player_mac: Option<String>,
    auth: Option<String>,
    suppress_next_volume: Arc<AtomicBool>,
    flush_tx: Option<watch::Sender<u64>>,
    seek_gen: Arc<AtomicU64>,
    needs_position_sync: Arc<AtomicBool>,
}

impl LMS {
    pub fn new(
        host_port: Option<String>,
        player_mac: Option<String>,
        auth: Option<String>,
        flush_tx: Option<watch::Sender<u64>>,
    ) -> Self { ... }
}
```

**handle_player_event — wire vocabulary** (spotty.rs lines 310-440):
```rust
// Five commands: "start", "change", "stop", "volume", "seek"
// "pause" is NOT emitted — Paused and Stopped both collapse to "stop"
PlayerEvent::Playing { track_id, position_ms, .. } => { /* start/change/seek */ }
PlayerEvent::Paused { .. } | PlayerEvent::Stopped { .. } => { notify("stop", "", "") }
PlayerEvent::VolumeChanged { volume } => {
    if self.suppress_next_volume.swap(false, Relaxed) { return; }
    let pct = u32::from(*volume) * 100 / 65535;
    notify("volume", &pct.to_string(), "")
}
PlayerEvent::Seeked { position_ms, .. } => {
    let secs = f64::from(*position_ms) / 1000.0;
    notify("seek", &format!("{secs:.3}"), "")
}
PlayerEvent::SessionConnected { .. } => {
    self.suppress_next_volume.store(true, Relaxed);
}
```

**notify() — JSON-RPC POST pattern** (spotty.rs lines 448-525):
```rust
async fn notify(&self, cmd: &str, p1: &str, p2: &str) {
    let body = json!({
        "id": 1,
        "method": "slim.request",
        "params": [player_mac, ["spottyconnect", cmd, p1?, p2?]],
    }).to_string();
    // HTTP/1.0 POST via TcpStream::connect(host_port)
    // Authorization: Basic header if auth is set
}
```

**HttpStreamSink — CRITICAL: no exit() in stop()** (spotty.rs lines 607-762):
```rust
impl Sink for HttpStreamSink {
    fn stop(&mut self) -> SinkResult<()> {
        // CRITICAL: do NOT exit() here (Pitfall 1 — StdoutSink does exit(0))
        self.frames_consumed = 0;
        self.began_at = Instant::now();
        Ok(())
    }
    fn write(&mut self, packet: AudioPacket, converter: &mut Converter) -> SinkResult<()> {
        // Rate-limiter: expected_ns = frames_consumed * 1e9 / SAMPLE_RATE + buffer_latency_ns
        // std::thread::sleep(Duration::from_nanos(park_ns)) when expected_ns > elapsed_ns
    }
}
```

**http_stream_server() — key patterns** (spotty.rs lines 793-~1034):
```rust
// spirc_active guard: 503 + Retry-After: 2 when not yet connected (Pitfall 2)
if !spirc_active.load(Ordering::Acquire) { return 503 }
// relay_active AtomicBool: 503 when concurrent connection attempts (CR-01)
// Drain stale PCM on seek: flush_rx watch-channel
// Content-Type: audio/L16;rate=44100;channels=2
```

**SpotOn additions vs. Spotty-NG (D-14 HTTP control endpoints):**
```rust
// Additional routes handled in http_stream_server service_fn:
// POST /control/pause  -> spirc.pause()
// POST /control/play   -> spirc.play()
// POST /control/volume -> spirc.set_volume(vol_u16)   // JSON body: {"volume": 0-100}
// POST /control/seek   -> spirc.seek(position_ms)     // JSON body: {"position_ms": N}
// POST /control/next   -> spirc.next()
// POST /control/prev   -> spirc.prev()
// Spirc stored in: Arc<Mutex<Option<SpiHandle>>>  where SpiHandle = (Spirc, task::JoinHandle)
```

**run_connect() skeleton** (from RESEARCH.md Code Examples, verified against spotty.rs main.rs lines 2340-2600):
```rust
pub async fn run_connect(cache_dir: &str, device_name: &str, player_mac: Option<&str>,
                          lms_host_port: Option<&str>, lms_auth: Option<&str>,
                          disable_discovery: bool, buffer_latency_ms: u64)
    -> Result<(), Box<dyn std::error::Error>>
{
    // 1. println!("stream_port={}", port); std::io::stdout().flush()  ← Pitfall 3
    // 2. spirc_active = Arc::new(AtomicBool::new(false))
    // 3. spirc_active.store(true) ONLY on PlayerEvent::SessionConnected  ← Pitfall 2
    // 4. SoftMixer::open() for volume control  ← Pitfall 8
    // 5. Spirc::new(connect_config, session, credentials, player, mixer).await
}
```

---

### `Plugins/SpotOn/Plugin.pm` (modify — add Connect daemon startup + _killOrphanedProcesses fix)

**Analog:** self (existing `/home/sti/spoton/Plugins/SpotOn/Plugin.pm`)

**initPlugin() extension — add Connect daemon startup** (after line 91, inside `if (main::WEBUI)`):
```perl
# Phase 5: Start Connect daemon manager for all players
# Deferred to allow player list to populate after LMS start
require Plugins::SpotOn::Connect;
Plugins::SpotOn::Connect->init();
```

**_killOrphanedProcesses() — PHASE-5-NOTE execution** (lines 155-165):
```perl
# REPLACE the existing PHASE-5-NOTE comment block with:
# Exclude Connect daemon PIDs from orphan cleanup (CON-09)
my %connectPids;
if ($INC{'Plugins/SpotOn/Connect/DaemonManager.pm'}) {
    %connectPids = map { $_ => 1 }
        Plugins::SpotOn::Connect::DaemonManager->helperPids();
}

# In the kill loop, add:
next if $activePids{$pid};
next if $connectPids{$pid};   # CON-09: protect Connect daemon PIDs
kill 'TERM', $pid;
```

**updateTranscodingTable() — soc profile awareness** (lines 1186-1249):
```perl
# The soc-* entries in custom-convert.conf use "# I" (passthrough-input) — they have NO
# [spoton] invocation and do NOT contain "single-track". The existing guard:
#   next unless $key =~ /^son-/ && $commandTable->{$key} =~ /single-track/;
# already correctly skips soc-* entries. No code change needed in the loop body.
# Only add: delete soc-ogg-*-* when player can't do OGG (same guard as son-ogg):
unless (Plugins::SpotOn::Helper->getCapability('passthrough')) {
    delete $commandTable->{'soc-ogg-*-*'};
}
# And conditionally: delete soc-ogg-*-* when per-player pref forces PCM mode
```

---

### `Plugins/SpotOn/ProtocolHandler.pm` (modify — add soc, canDirectStream HTTP-URL, new() sync-proxy)

**Analog:** `/home/sti/spotty-ng/Spotty-Plugin/ProtocolHandler.pm`

**formatOverride() — soc content-type** (Spotty-NG ProtocolHandler.pm lines 60-80):
```perl
sub formatOverride {
    my ($class, $song) = @_;

    require Plugins::SpotOn::Plugin;
    Plugins::SpotOn::Plugin->updateTranscodingTable($song->master);

    # Check if a streaming Connect daemon is active (race-safe: check daemon directly)
    require Plugins::SpotOn::Connect::DaemonManager;
    my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($song->master);
    if ($helper && $helper->_streamMode) {
        return 'soc';   # SpotOn Connect (not 'spc' from Spotty-NG)
    }

    return 'son';   # Single-track (existing behavior)
}
```

**canDirectStream() — HTTP-URL for single players, 0 for sync groups** (Spotty-NG lines 82-101):
```perl
sub canDirectStream {
    my ($class, $client, $url) = @_;
    return 0 unless $client;
    $client = $client->master if $client->can('master');

    require Plugins::SpotOn::Connect::DaemonManager;
    my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client->id);
    return 0 unless $helper && $helper->_streamMode && $helper->_streamPort;

    return 0 if $client->isSynced();   # sync groups: use new() proxy instead (D-06)

    my $host = Slim::Utils::Network::serverAddr();
    return 'http://' . $host . ':' . $helper->_streamPort . '/stream';
}
```

**new() — sync-group proxy via URL substitution** (Spotty-NG lines 115-153):
```perl
sub new {
    my ($class, $args) = @_;
    my $url = $args->{'url'} || '';

    if ($url =~ m{/connect-}) {
        my $client = $args->{'client'};
        if ($client) {
            $client = $client->master if $client->can('master');
            require Plugins::SpotOn::Connect::DaemonManager;
            my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client->id);
            if ($helper && $helper->_streamMode && $helper->_streamPort) {
                my $host    = Slim::Utils::Network::serverAddr();
                my $httpUrl = 'http://' . $host . ':' . $helper->_streamPort . '/stream';
                $args = { %$args, url => $httpUrl };
            } else {
                return undef;
            }
        }
    }
    return Slim::Player::Protocols::HTTP->new($args);
}
```

**Existing methods that need Connect-awareness (copy Spotty-NG pattern):**
```perl
sub canSeek {
    my ($class, $client) = @_;
    return 0 if $client && Plugins::SpotOn::Connect->isSpotifyConnect($client);
    return Slim::Utils::Versions->compareVersions($::VERSION, '7.9.1') >= 0;
}
# same for canTranscodeSeek
```

---

### `librespot-spoton/src/main.rs` (modify — wire Mode::Connect)

**Analog:** self (existing, lines 268-272 = stub to replace)

**Replace stub with connect.rs call** (lines 268-272):
```rust
Mode::Connect => {
    // Phase 5: delegate to connect module
    mod connect;
    if let Err(e) = connect::run_connect(
        &cache_dir, &device_name,
        if player_mac.is_empty() { None } else { Some(&player_mac) },
        if lms_host.is_empty() { None } else { Some(&lms_host) },
        if lms_auth.is_empty() { None } else { Some(&lms_auth) },
        disable_discovery,
        buffer_latency_ms,
    ).await {
        eprintln!("Connect mode error: {}", e);
        process::exit(1);
    }
}
```

**Additional CLI args to parse** (insert alongside existing arg matching):
```rust
"--player-mac" => { player_mac = args[i+1].clone(); i += 1; }
"--lms"        => { lms_host   = args[i+1].clone(); i += 1; }
"--lms-auth"   => { lms_auth   = args[i+1].clone(); i += 1; }
"--buffer-latency-ms" => { buffer_latency_ms = args[i+1].parse().unwrap_or(2000); i += 1; }
"--disable-discovery" => { disable_discovery = true; }
```

---

### `librespot-spoton/Cargo.toml` (modify — add connect deps)

**Analog:** self (existing) + RESEARCH.md Standard Stack

**Add to [dependencies]** (existing file lines 19-25, extend):
```toml
librespot-connect = { version = "0.8", default-features = false, features = ["rustls-tls-native-roots"] }
bytes             = "1"
hyper             = { version = "1", features = ["http1", "server"] }
hyper-util        = { version = "0.1", features = ["server-auto", "server-graceful", "service", "tokio"] }
http-body-util    = "0.1"
tokio-stream      = "0.1"
# Extend existing tokio entry to add net + io-util + sync:
tokio             = { version = "1", features = ["rt-multi-thread", "macros", "net", "io-util", "sync"] }
```

---

### `Plugins/SpotOn/Settings.pm` (modify — per-player Connect toggle + OGG override)

**Analog:** self (existing `/home/sti/spoton/Plugins/SpotOn/Settings.pm`)

**prefs() — add per-player prefs** (lines 42-47):
```perl
sub prefs {
    # Global prefs handled by Slim::Web::Settings base class
    return ($prefs, 'bitrate', 'binary', 'normalization');
    # Per-player prefs (enableSpotifyConnect, connectOggOverride) are handled
    # manually in handler() via $prefs->client($client) — same pattern as
    # existing activeAccount per-player handling
}
```

**handler() — save per-player Connect toggle** (after existing saveSettings block, same pattern as `activeAccount`):
```perl
# Per-player Connect toggle (D-10, CON-10)
# Use $prefs->client($client) — same pattern as existing activeAccount per-player pref
if (defined $paramRef->{saveSettings} && $client) {
    my $enableConnect = $paramRef->{'pref_enableSpotifyConnect'} ? 1 : 0;
    $prefs->client($client)->set('enableSpotifyConnect', $enableConnect);

    # OGG-passthrough override (D-05): 'auto' | 'ogg' | 'pcm'
    if (defined $paramRef->{'pref_connectOggOverride'}) {
        my $override = $paramRef->{'pref_connectOggOverride'};
        $override = 'auto' unless $override =~ /^(?:auto|ogg|pcm)$/;
        $prefs->client($client)->set('connectOggOverride', $override);
    }
}

# Pass to template:
$paramRef->{connectEnabled}    = $prefs->client($client)->get('enableSpotifyConnect') // 1;
$paramRef->{connectOggOverride} = $prefs->client($client)->get('connectOggOverride') || 'auto';
```

---

### `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (modify — Connect settings UI)

**Analog:** self (existing, copy checkbox pattern from existing normalization block lines 20-24)

**Per-player Connect toggle** (copy normalization checkbox pattern):
```html
[% WRAPPER setting title="PLUGIN_SPOTON_CONNECT_ENABLED" desc="PLUGIN_SPOTON_CONNECT_ENABLED_DESC" %]
    <input type="checkbox" class="stdedit" name="pref_enableSpotifyConnect"
           id="pref_enableSpotifyConnect"
           value="1" [% IF connectEnabled %]checked[% END %]/>
    <label for="pref_enableSpotifyConnect">[% 'PLUGIN_SPOTON_CONNECT_ENABLED_LABEL' | string %]</label>
[% END %]
```

**OGG-passthrough override select** (copy bitrate select pattern from lines 12-18):
```html
[% WRAPPER setting title="PLUGIN_SPOTON_CONNECT_OGG_OVERRIDE" desc="" %]
    <select class="stdedit" name="pref_connectOggOverride" id="pref_connectOggOverride">
        <option value="auto" [% IF connectOggOverride == 'auto' %]selected[% END %]>Auto</option>
        <option value="ogg"  [% IF connectOggOverride == 'ogg'  %]selected[% END %]>OGG Passthrough</option>
        <option value="pcm"  [% IF connectOggOverride == 'pcm'  %]selected[% END %]>PCM (force)</option>
    </select>
[% END %]
```

---

### `Plugins/SpotOn/custom-convert.conf` (modify — add soc profiles)

**Analog:** `/home/sti/spotty-ng/Spotty-Plugin/custom-convert.conf` lines 15-18 (`spc pcm * *`)

**Add soc profiles** (append to existing file after son-ogg entry):
```
soc pcm * *
    # I
    -

soc ogg * *
    # I
    -
```

Key: `# I` means "LMS fetches the stream directly" (input passthrough). No binary invocation — LMS connects via `canDirectStream` URL or `new()` proxy. Identical to Spotty-NG `spc pcm * *` pattern.

---

### `Plugins/SpotOn/custom-types.conf` (modify — register soc type)

**Analog:** `/home/sti/spotty-ng/Spotty-Plugin/custom-types.conf` line 6 (`spc spc audio/x-sb-spotify-connect audio`)

**Append soc line** (after existing `son` line):
```
soc    soc    audio/x-sb-spoton-connect    audio
```

Format: `ID  Suffix  MIME-Content-Type  ServerFileType` — same column layout as existing `son` line.

---

## Shared Patterns

### Timer Pattern (Debouncing, Watchdog, Grace Periods)
**Source:** `Plugins/SpotOn/Plugin.pm` lines 60-73 (existing pattern)
**Apply to:** Connect.pm (_onVolume debounce), DaemonManager.pm (watchdog, sync-change delay)
```perl
# Kill existing timer first, then set new one (prevents duplicate timers)
Slim::Utils::Timers::killTimers($obj, \&_callback);
Slim::Utils::Timers::setTimer($obj, Time::HiRes::time() + $delay, \&_callback);
```

### Per-Player Prefs Pattern
**Source:** `Plugins/SpotOn/Plugin.pm` line 1197 + `Settings.pm` line 128-136 (existing)
**Apply to:** Connect.pm, DaemonManager.pm, Daemon.pm, Settings.pm
```perl
my $val = $prefs->client($client)->get('enableSpotifyConnect')
       // $prefs->get('enableSpotifyConnect')
       // 1;   # default: enabled
```

### Cache Dir Pattern
**Source:** `Plugins/SpotOn/Plugin.pm` lines 1195-1205 (existing)
**Apply to:** Daemon.pm (per-daemon cache dir for credentials)
```perl
my $activeAccountId = $prefs->client($client)->get('activeAccount')
                   || $prefs->get('activeAccount')
                   || '';
my $cacheDir = $activeAccountId
    ? catdir($serverPrefs->get('cachedir'), 'spoton', $activeAccountId)
    : catdir($serverPrefs->get('cachedir'), 'spoton');
```

### Logging Pattern
**Source:** `Plugins/SpotOn/Plugin.pm` lines 26-31 (existing)
**Apply to:** All new Perl modules
```perl
my $log = logger('plugin.spoton');    # NOT Slim::Utils::Log->addLogCategory — that's only for Plugin.pm
# Usage guards (performance):
main::INFOLOG && $log->is_info && $log->info("message");
main::DEBUGLOG && $log->is_debug && $log->debug("message");
```

### Rust Imports for connect.rs
**Source:** `/home/sti/spotty-ng/librespot/src/spotty.rs` lms_connect module + main.rs imports
**Apply to:** `librespot-spoton/src/connect.rs`
```rust
use std::sync::{Arc, atomic::{AtomicBool, AtomicU64, Ordering}};
use std::time::{Duration, Instant};
use tokio::sync::{mpsc, watch};
use tokio::net::TcpListener;
use bytes::Bytes;
use hyper::{Response, StatusCode};
use hyper::body::Frame;
use http_body_util::{Full, StreamBody, BodyExt, combinators::BoxBody};
use hyper_util::rt::TokioIo;
use hyper_util::server::conn::auto::Builder as http1;
use hyper_util::server::graceful::GracefulShutdown;
use tokio_stream::wrappers::ReceiverStream;
use librespot_connect::{spirc::Spirc, config::ConnectConfig};
use librespot_core::session::Session;
use librespot_playback::audio_backend::Sink;
use librespot_playback::audio_backend::{SinkResult, AudioPacket, Converter};
use librespot_playback::config::AudioFormat;
use librespot_playback::player::{Player, PlayerEvent};
```

---

## No Analog Found

All 12 files have close analogs. No files require RESEARCH.md-only patterns.

---

## Critical Pitfalls (from RESEARCH.md, to be called out in PLAN.md)

| Pitfall | File | Guard |
|---------|------|-------|
| `exit(0)` in Sink::stop() | connect.rs HttpStreamSink | `stop()` resets counters only, no exit |
| spirc_active race | connect.rs http_stream_server | set true ONLY on `PlayerEvent::SessionConnected` |
| stdout not flushed after `stream_port=N` | connect.rs run_connect | `std::io::stdout().flush()` immediately after println |
| Connect PIDs killed by orphan cleanup | Plugin.pm + DaemonManager.pm | `helperPids()` exclusion list before pgrep kill loop |
| Stale API on change events | Connect.pm _connectEvent | p2 (eventTrackId) overrides API response |
| Volume feedback loop | Connect.pm _onVolume | `$req->source(__PACKAGE__)` + early return check |
| `['time', N]` in stream mode | Connect.pm _onSeek | `$song->startOffset()` only |

---

## Metadata

**Analog search scope:** `/home/sti/spoton/Plugins/SpotOn/`, `/home/sti/spotty-ng/Spotty-Plugin/`, `/home/sti/spotty-ng/librespot/src/`
**Files scanned:** 12 (7 SpotOn existing, 5 Spotty-NG analogs)
**Pattern extraction date:** 2026-06-01
