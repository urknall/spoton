---
phase: 05-spotify-connect
reviewed: 2026-06-01T00:00:00Z
depth: standard
files_reviewed: 9
files_reviewed_list:
  - librespot-spoton/src/connect.rs
  - librespot-spoton/src/main.rs
  - librespot-spoton/Cargo.toml
  - Plugins/SpotOn/Connect.pm
  - Plugins/SpotOn/Connect/Daemon.pm
  - Plugins/SpotOn/Connect/DaemonManager.pm
  - Plugins/SpotOn/ProtocolHandler.pm
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/Settings.pm
  - Plugins/SpotOn/custom-convert.conf
findings:
  critical: 3
  warning: 6
  info: 4
  total: 13
status: issues_found
---

# Phase 05: Code Review Report

**Reviewed:** 2026-06-01
**Depth:** standard
**Files Reviewed:** 10 (including custom-convert.conf)
**Status:** issues_found

## Summary

This phase implements the full Spotify Connect subsystem: a Rust binary providing Spirc event
handling, HTTP PCM stream server, and HTTP control endpoints; plus Perl modules wiring LMS
player events bidirectionally to/from the binary. The architecture is sound and the FIFO
pitfalls are correctly avoided. Three blockers are present: a `newTrack` flag that gets
permanently stuck when the Spotify Web API returns a stale metadata response (silently
suppresses all subsequent pause events on the active player), an unguarded `->client()->master`
call that can crash LMS under edge-case connection timing, and a `DefaultHasher`-derived
device ID that is non-deterministic across Rust version upgrades (causes Spotify to register
a duplicate device entry after a binary update). Six warnings cover a mis-scoped prefs change
listener, an off-by-one in the crash-backoff counter, silent seek-failure masking at the
control endpoint, stale `_streamMode` state after failed restarts, SystemTime use in a
grace timer, and dead code in the API client's player control methods.

---

## Critical Issues

### CR-01: `newTrack` flag never cleared on stale API response — pause events permanently suppressed

**File:** `Plugins/SpotOn/Connect.pm:739-749, 583-588, 793`

**Issue:** `_fetchTrackMetadata` is called on every `start` and `change` event. The `start`
handler sets `$client->pluginData(newTrack => 1)` at line 613. The only place that clears
it back to `0` is inside `_fetchTrackMetadata`'s success callback at line 793. However,
at lines 740-748 there is a stale-API detection block that does:

```perl
$trackInfo = { uri => $eventUri };
return;   # <-- early return, newTrack stays 1 forever
```

When this path is taken (API returns metadata for a different track than the binary just
signalled), the callback returns without executing line 793. The `newTrack` flag stays `1`
indefinitely. The guard at line 583 silently drops ALL subsequent `stop` events while
`newTrack` is set:

```perl
if ($cmd eq 'stop' && $client->pluginData('newTrack')) {
    # returns without pause
    return;
}
```

This means: after one stale API response on session start, the user can never pause
Spotify-Connect playback via the Spotify app until the binary session is restarted. No
error is logged.

**Fix:** Clear the `newTrack` flag unconditionally before returning from the stale-response
branch:

```perl
if ($trackInfo && $trackInfo->{uri} && $eventUri
    && $eventUri ne $trackInfo->{uri})
{
    $log->info("Stale API response ...");
    $client->pluginData(newTrack => 0);   # ADD THIS
    return;
}
```

---

### CR-02: Unguarded `->client()->master` call in `_connectEvent` — crashes on edge-case disconnect

**File:** `Plugins/SpotOn/Connect.pm:452`

**Issue:** The `_connectEvent` dispatcher begins with:

```perl
my $client = $request->client()->master;
```

There is no `defined` guard before calling `->master`. All other LMS event subscriber
functions in this file use the defensive pattern:

```perl
my $client = $request->client();
return if !defined $client;
$client = $client->master;
```

Although the dispatch is registered with `requiresClient=1` (preventing purely server-side
invocation), the binary sends `slim.request` with a player MAC. If the MAC resolves to a
player that disconnects between the JSON-RPC arrival and the dispatch resolution, LMS may
call the handler with a `$client` whose `->master` call triggers an undef dereference,
causing a Perl exception that propagates up the LMS request handler. Under LMS's default
error handling this can abort the current client session.

**Fix:**

```perl
sub _connectEvent {
    my $request = shift;
    my $client  = $request->client();
    return unless defined $client;
    $client = $client->master;
    # ... rest of handler
```

---

### CR-03: `DefaultHasher` for `device_id` is non-deterministic across Rust versions

**File:** `librespot-spoton/src/connect.rs:957-961`, `librespot-spoton/src/main.rs:538-542`

**Issue:** Both `run_connect` and `run_discover_once` derive the librespot `device_id` from
the cache directory path using `std::collections::hash_map::DefaultHasher`:

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};
let mut hasher = DefaultHasher::new();
cache_dir.hash(&mut hasher);
let device_id = format!("{:016x}", hasher.finish());
```

The Rust documentation explicitly states that `DefaultHasher` is **not guaranteed to be
stable across Rust versions or builds**. The hash algorithm can change between Rust releases.
When the binary is rebuilt with a newer Rust compiler (e.g., a platform update), the
`device_id` for the same `cache_dir` path changes, causing Spotify's cloud to register
a **new duplicate device** visible in the Spotify app alongside the old entry. Users end up
with multiple dead device entries. This affects both ZeroConf discovery credentials and
the mDNS Connect announcement.

**Fix:** Use a stable hash. MD5 or SHA-1 of the cache_dir bytes is sufficient (not a
security context; stability is the only requirement):

```rust
// stable device_id from cache_dir — use SHA-1 or MD5 (not DefaultHasher which is version-specific)
use std::collections::hash_map::DefaultHasher; // REMOVE
// Replace with:
let device_id = {
    let mut d = [0u8; 16];
    // Simple FNV-1a or any stable algorithm; or use sha2 crate if already a dependency.
    // Simplest stable option: iterate bytes manually with FNV:
    let mut h: u64 = 14695981039346656037u64;
    for b in cache_dir.as_bytes() {
        h ^= *b as u64;
        h = h.wrapping_mul(1099511628211u64);
    }
    format!("{:016x}", h)
};
```

---

## Warnings

### WR-01: `prefs->setChange` listens to global namespace but `enableSpotifyConnect` is stored per-player

**File:** `Plugins/SpotOn/Connect/DaemonManager.pm:61`

**Issue:**

```perl
$prefs->setChange(\&initHelpers, 'enableSpotifyConnect');
```

`$prefs` is the global `plugin.spoton` namespace. `setChange` fires when the global key
`enableSpotifyConnect` changes. However, `Settings.pm` line 147 stores the value as a
**per-player** preference:

```perl
$prefs->client($client)->set('enableSpotifyConnect', $enableConnect);
```

Per-player prefs are stored under `plugin.spoton:player:<mac>`, not the global namespace.
The `setChange` callback will **never fire** when a user toggles the Connect switch in
Settings for a specific player. As a result, the daemon lifecycle does not react to per-player
enable/disable changes until the next 60-second watchdog fires. Users expect the daemon to
start or stop immediately when they toggle the setting.

**Fix:** Remove the dead `setChange` call. Instead, call `initHelpers` at the end of the
`Settings.pm` handler when `pref_enableSpotifyConnect` is saved:

```perl
# In Settings.pm handler(), after saving enableSpotifyConnect:
if ($client) {
    require Plugins::SpotOn::Connect::DaemonManager;
    Plugins::SpotOn::Connect::DaemonManager->initHelpers();
}
```

---

### WR-02: Off-by-one in `_checkStartTimes` allows one extra restart before disabling discovery

**File:** `Plugins/SpotOn/Connect/Daemon.pm:191`

**Issue:**

```perl
if ( scalar @{$self->_startTimes} > MAX_FAILURES_BEFORE_DISABLE_DISCOVERY ) {
```

`MAX_FAILURES_BEFORE_DISABLE_DISCOVERY = 3`. The check fires when the array is **strictly
greater than 3**, i.e., at the 5th call to `_checkStartTimes` (4 previous pushes). After
the splice, the oldest of the retained 3 times is checked against the interval. So
discovery is disabled after **5 total starts** (4 crashes), not 4 starts (3 crashes) as
the constant name implies. The analogous `_checkStreamStartTimes` uses `>=` which is
correct for `MAX_STREAM_FAILURES`. The inconsistency suggests an unintended difference.

**Fix:** Change `>` to `>=`:

```perl
if ( scalar @{$self->_startTimes} >= MAX_FAILURES_BEFORE_DISABLE_DISCOVERY ) {
```

---

### WR-03: Known control commands return 204 even when body parse fails — seek and volume failures silently lost

**File:** `librespot-spoton/src/connect.rs:726-734`

**Issue:**

```rust
let status = if result.is_some() || cmd == "pause" || cmd == "play"
    || cmd == "next" || cmd == "prev" || cmd == "volume" || cmd == "seek" {
    StatusCode::NO_CONTENT
} else {
    StatusCode::NOT_FOUND
};
```

The second condition means: if `cmd` is any known command, return 204 regardless of whether
the command was actually executed. For `volume` and `seek`, `result` is `None` when body
parsing fails (invalid or absent JSON body from the Perl side). The 204 response tells the
Perl caller the command succeeded, but Spirc received no instruction. Seek and volume
changes are silently dropped. This is particularly harmful for seek: the user scrubs in
the Spotify app, the binary receives the notification, sends `seek` to LMS; LMS forwards
back to the binary via `/control/seek`; if the body is malformed at that point, the seek
is lost with no error, causing desync between Spotify and LMS position.

**Fix:** Return 204 only when `result.is_some()`. Return 422 for parse failures on body-requiring commands:

```rust
let status = match (result.is_some(), cmd) {
    (true, _) => StatusCode::NO_CONTENT,
    (false, "pause") | (false, "play") | (false, "next") | (false, "prev") => {
        // these commands have no body; None from Spirc is non-fatal
        StatusCode::NO_CONTENT
    }
    (false, "volume") | (false, "seek") => {
        // body required but parse failed
        StatusCode::UNPROCESSABLE_ENTITY
    }
    _ => StatusCode::NOT_FOUND,
};
```

---

### WR-04: Stale `_streamMode` flag not reset after failed restart — unnecessary fast polling

**File:** `Plugins/SpotOn/Connect/Daemon.pm`, `Plugins/SpotOn/Connect/DaemonManager.pm:135-146`

**Issue:** `Daemon::start()` sets `$self->_streamMode(1)` only on successful port capture
(line 177). However it does not reset `_streamMode` to `0` in the failure paths (port
timeout at line 168, `Proc::Background` failure at line 150). If a daemon had previously
started successfully (`_streamMode=1`), then crashes, and its next `start()` call fails
(e.g., port timeout), `_streamMode` remains `1`. `DaemonManager::_streamAlivePoll` checks
`grep { $_->_streamMode }` and continues to poll at 5-second intervals for this daemon
that can never provide a stream. The fast poll is supposed to self-stop when no streaming
daemons are active, but the stale flag prevents that.

**Fix:** In `Daemon::start()`, reset `_streamMode` to `0` at the beginning, before the
outcome is known:

```perl
sub start {
    my $self = shift;
    $self->_streamMode(0);    # ADD: reset before attempt
    $self->_streamPort(undef); # ADD: reset port too
    ...
```

---

### WR-05: Grace timer in `LMS::handle_player_event` uses `SystemTime` instead of `Instant`

**File:** `librespot-spoton/src/connect.rs:200-211`

**Issue:** The 2-second grace timer that suppresses spurious `Paused`/`Stopped` events after
session start is measured using `std::time::SystemTime::now()`:

```rust
let now_ns = std::time::SystemTime::now()
    .duration_since(std::time::UNIX_EPOCH)
    .unwrap_or_default()
    .as_nanos() as u64;
let elapsed_ns = now_ns.saturating_sub(last);
```

`SystemTime` is subject to NTP adjustments and can jump backwards. When a backwards jump
occurs, `saturating_sub` returns `0`, and `elapsed_ns < grace_ns` is always true until
the clock catches up. In practice this extends the grace period, meaning `stop` events
could be suppressed for longer than 2 seconds (up to the NTP correction amount). The
`last_session_start_ns` stored via `Arc<AtomicU64>` and loaded/stored as nanoseconds-since-UNIX
compounds this: an NTP correction could set `now_ns < last`, permanently keeping the timer
suppressed until the binary restarts. Use `Instant` for duration measurement.

**Fix:**

```rust
// Replace last_session_start_ns: Arc<AtomicU64> with last_session_start: Arc<Mutex<Option<Instant>>>
// Or store last_session_start as AtomicU64 of nanos from a monotonic reference point.
// Simplest fix: store an Instant behind an Arc<Mutex<Option<Instant>>>.
pub last_session_start: Arc<std::sync::Mutex<Option<std::time::Instant>>>,
```

---

### WR-06: `_method => 'PUT'` fields in `API::Client` player control methods are dead code

**File:** `Plugins/SpotOn/API/Client.pm:269, 280, 291, 303`

**Issue:** `playerPause`, `playerPlay`, `playerVolume`, and `playerSeek` all pass
`_method => 'PUT'` in the `$params` hash:

```perl
$class->_request('put', 'me/player/pause', {
    _accountId => $accountId,
    _noCache   => 1,
    _method    => 'PUT',   # dead code
}, $cb);
```

In `_doFlavouredRequest`, the query-string builder skips all keys starting with `_`
(`next if $key =~ /^_/`). The `_method` key is never read anywhere in the pipeline —
the actual HTTP method is the first argument to `_request` (`'put'`). These `_method`
fields have no effect.

**Fix:** Remove the `_method => 'PUT'` lines from all four methods. They add noise and
could mislead future maintainers into thinking they have a function.

---

## Info

### IN-01: `soc-ogg-*-*` delete in `updateTranscodingTable` is dead code

**File:** `Plugins/SpotOn/Plugin.pm:1271`, `Plugins/SpotOn/custom-convert.conf`

**Issue:** `updateTranscodingTable` does `delete $commandTable->{'soc-ogg-*-*'}` but
`custom-convert.conf` contains no `soc ogg * *` entry, so this key never exists in
`commandTable`. The delete is always a no-op. The comment says "same guard for Connect
OGG passthrough" but no such entry was added.

**Fix:** Either add a `soc ogg * *` entry to `custom-convert.conf` (if OGG passthrough
for Connect is desired), or remove the dead delete call. Currently it's just noise.

---

### IN-02: `_onPlaylistJump` treats index `+0` as "previous" without comment explaining why

**File:** `Plugins/SpotOn/Connect.pm:431`

**Issue:**

```perl
elsif ($index eq '-1' || $index eq '+0') {
    _sendControlCommand($client, '/control/prev', undef);
}
```

The `+0` case (restart current track in LMS playlist semantics) is mapped to `/control/prev`
without explanation. This could be intentional (Spotify app's "previous within first 3s
restarts, then goes to previous track" behavior), but the absence of a comment makes it
appear to be a typo of `+1` or an error.

**Fix:** Add a comment:

```perl
# +0 = restart current track (LMS semantics); map to prev so Spotify
# honours its own "restart-within-3s" rule rather than LMS restarting the stream.
elsif ($index eq '-1' || $index eq '+0') {
```

---

### IN-03: `notify()` in `connect.rs` silently swallows all TCP errors including partial writes

**File:** `librespot-spoton/src/connect.rs:340-345`

**Issue:**

```rust
if let Err(_e) = stream.write_all(request.as_bytes()).await {
    // silent
}
```

`write_all` can fail partway through with a partial write that leaves LMS's JSON-RPC
parser in a broken state. While the "never panic on LMS outage" design is correct, a
`log::warn` would help diagnose connectivity issues during development and production
debugging without adding risk.

**Fix:** Add a debug-level log:

```rust
if let Err(e) = stream.write_all(request.as_bytes()).await {
    log::debug!("[spoton] notify({cmd}): write failed: {e}");
}
```

---

### IN-04: Blocking 5-second `IO::Select` read in `Daemon::start()` stalls LMS event loop

**File:** `Plugins/SpotOn/Connect/Daemon.pm:163-166`

**Issue:**

```perl
my $sel = IO::Select->new($port_r);
if ($sel->can_read(5)) {
    $port_line = readline($port_r);
}
```

This is a synchronous 5-second blocking wait in LMS's single-threaded event loop. While
LMS plugins routinely do brief synchronous work, 5 seconds is enough to cause visible
latency in the UI and delay other event processing (player commands, HTTP responses, etc.)
if the binary is slow to start. On a loaded Raspberry Pi this is non-negligible. The code
comment acknowledges this ("Synchronous port read with 5s timeout (avoids SIGALRM in LMS
event loop)") but the justification only addresses SIGALRM, not latency.

**Fix (pragmatic):** Reduce the timeout to 2–3 seconds (the binary announces the port
almost immediately after binding; 5 seconds only matters if the binary hangs on startup,
which is itself a failure case). Alternatively, defer the start to an async timer that
polls for the port announcement without blocking:

```perl
if ($sel->can_read(2)) {  # reduce from 5s to 2s
```

---

_Reviewed: 2026-06-01_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
