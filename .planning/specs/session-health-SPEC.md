# SPEC: Session Health Monitoring + Log Cleanup

**Status:** Draft
**Created:** 2026-06-30
**Trigger:** Cold-start playback failure after overnight idle — first track plays 2-4s then stops.

## Problem

After hours of daemon idle, the librespot Spotify session (Shannon TCP connection) goes stale. The daemon process stays alive and its HTTP server responds to alive polls, but the Spotify data channel is dead.

When the user plays a track, librespot delivers a few seconds of buffered/partially-fetched audio before the dead channel causes EOF. LMS receives 2-4 seconds of PCM, then the stream ends. The user perceives this as "track starts, then silence."

### Evidence (server.log 2026-06-30)

```
06:31:22  Track 1 requested → daemon responds (pid=203615, >6h uptime)
06:31:25  Track 2 requested — 2.3s after Track 1 → Track 1 already dead
06:32:13  Different playlist → same pattern (1.5s between tracks)
06:33:25  User toggles Diagnose Mode → daemon full restart → pid=228561
06:33+    All playback works
```

### Why existing safeguards miss this

**Alive poll (`_streamAlivePoll`, 5s cycle):** Checks `Proc::Background->alive` — process-level only. The HTTP server can be alive with a dead Spotify session.

**Browse reconnect (`consecutive_browse_fails`):** Only triggers on `StatusCode::NOT_FOUND` from `serve_track_request` within the first 500ms. Mid-stream failures (track starts then dies at 2-4s) return `StatusCode::OK` because the `Unavailable` window has already passed. Counter never increments.

**60s watchdog (`initHelpers`):** Calls `startHelper()` which checks daemon alive + config unchanged → falls through without restart. Additionally, logs misleading `"Starting Unified daemon for player: ..."` message 800 times/day.

## Solution

Two changes: (1) enhanced `/health` endpoint with Spotify session health reporting, (2) Perl-side health monitoring that acts on it. Plus a log cleanup.

---

## Part 1: Rust — Enhanced `/health` Endpoint

### Current (unified.rs:287-298)

```rust
GET /health → 200 OK "ok"
```

No session information. Indistinguishable from a daemon with a dead session.

### New

```rust
GET /health → 200 OK application/json
```

Response body:

```json
{
  "status": "ok",
  "session_valid": true,
  "session_age_secs": 21600,
  "idle_secs": 3600
}
```

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `status` | string | always "ok" | Daemon HTTP server is alive |
| `session_valid` | bool | `!session.is_invalid()` | librespot Session object state |
| `session_age_secs` | u64 | `Instant::elapsed()` since last `session.connect()` or Spirc reconnect | Seconds since session was established |
| `idle_secs` | u64 | `Instant::elapsed()` since last successful Browse track or Connect activity | Seconds since last audio streaming activity |

### Implementation Details

**New shared state (passed to `unified_http_server`):**

```rust
session_created_at: Arc<std::sync::Mutex<Instant>>
last_activity: Arc<std::sync::Mutex<Instant>>
```

Both are `Mutex<Instant>` (not `AtomicU64`) because `Instant` is not atomic-compatible and we need wall-clock-independent monotonic time.

**Update points for `last_activity`:**
- `serve_track_request` returns `StatusCode::OK` → Browse track streamed successfully (unified.rs ~line 634, after `consecutive_browse_fails.store(0)`)
- `/stream` relay sends data → Connect audio active (unified.rs ~line 433, inside `Some(bytes)` match arm)

**Update points for `session_created_at`:**
- Initial `session.connect()` in `run_unified` (line 842)
- After Spirc reconnect succeeds (line 1282, after `"Spirc reconnected"`)
- After Browse-only reconnect succeeds (line 1437, after `"Browse-only session reconnected"`)
- After ZeroConf credential rotation reconnect (same as Spirc reconnect path)

**`/health` handler changes (replacing lines 287-298):**

```rust
if method == Method::GET && path == "/health" {
    let session_valid = {
        let s = session.lock().await;
        !s.is_invalid()
    };
    let age_secs = {
        let t = session_created_at.lock().unwrap_or_else(|e| e.into_inner());
        t.elapsed().as_secs()
    };
    let idle_secs = {
        let t = last_activity.lock().unwrap_or_else(|e| e.into_inner());
        t.elapsed().as_secs()
    };

    let json = format!(
        r#"{{"status":"ok","session_valid":{},"session_age_secs":{},"idle_secs":{}}}"#,
        session_valid, age_secs, idle_secs
    );

    let body = Full::new(Bytes::from(json))
        .map_err(|e| match e {})
        .boxed();
    let resp = Response::builder()
        .status(StatusCode::OK)
        .header("Content-Type", "application/json")
        .body(body)
        .expect("health response builder");
    return Ok(resp);
}
```

No additional crate dependencies — `format!()` is sufficient for this fixed-schema JSON.

### Thread parameters

`session_created_at` and `last_activity` are passed through `unified_http_server` as additional parameters. They must also be cloned into each `service_fn` closure (same pattern as existing `session`, `spirc_active`, etc.).

---

## Part 2: Perl — Health-Aware Daemon Monitoring

### Design

A new periodic health check in `_streamAlivePoll` that calls `GET /health` on each daemon and evaluates session health. Uses `SimpleAsyncHTTP` (non-blocking, fits LMS event loop).

### Health check frequency

Every 12th `_streamAlivePoll` cycle = every 60 seconds.

Rationale: 5s is too frequent for HTTP requests. 60s balances detection speed vs overhead. A health check is a single GET to localhost — negligible load.

### New accessor in Daemon.pm

```perl
__PACKAGE__->mk_accessor( rw => qw(
    ...existing...
    _healthCheckCount
) );
```

Initialized to 0 in `new()`.

### Implementation in DaemonManager.pm

Add health check logic inside the `_streamAlivePoll` alive branch:

```perl
# In _streamAlivePoll, when $helper->alive:
my $count = ($helper->_healthCheckCount || 0) + 1;
$helper->_healthCheckCount($count);

if ($count % 12 == 0) {  # every 60s
    my $port = $helper->_streamPort;
    if ($port) {
        Slim::Networking::SimpleAsyncHTTP->new(
            sub { $class->_onHealthResponse($helper, @_) },
            sub { $class->_onHealthError($helper, @_) },
            { timeout => 5 }
        )->get("http://127.0.0.1:$port/health");
    }
}
```

### Restart decision logic (`_onHealthResponse`)

```perl
sub _onHealthResponse {
    my ($class, $helper, $http) = @_;

    my $json = eval { JSON::XS::decode_json($http->content) };
    unless ($json && $json->{status} eq 'ok') {
        # Malformed response — daemon is confused, restart
        $class->_restartForHealth($helper, 'malformed health response');
        return;
    }

    # Signal 1: librespot reports session invalid
    if (!$json->{session_valid}) {
        $class->_restartForHealth($helper, 'session_valid=false');
        return;
    }

    # Signal 2: session stale (old + idle)
    #   session_age > 4h AND idle > 5min → proactive restart
    #   Rationale: TCP keepalive (Linux default 2h) should kill dead connections,
    #   but kernel settings vary. 4h is a conservative safety net.
    #   Idle guard prevents restarting during active use.
    if ($json->{session_age_secs} > 14400 && $json->{idle_secs} > 300) {
        $class->_restartForHealth($helper,
            sprintf('stale session (age=%ds, idle=%ds)', $json->{session_age_secs}, $json->{idle_secs}));
        return;
    }
}
```

### Restart execution (`_restartForHealth`)

```perl
sub _restartForHealth {
    my ($class, $helper, $reason) = @_;

    main::INFOLOG && $log->is_info && $log->info(
        sprintf("Health check restart for %s: %s", $helper->mac, $reason)
    );

    $class->stopHelper($helper->mac);
    $class->startHelper($helper->mac);
}
```

### Error handler (`_onHealthError`)

```perl
sub _onHealthError {
    my ($class, $helper, $http) = @_;

    # HTTP error to localhost health endpoint while daemon process is alive
    # = daemon HTTP server not responding. This is unusual but not critical
    # because _streamAlivePoll already handles process death.
    # Log but don't restart (avoid double-restart race with alive poll).
    $log->warn("Health check failed for " . $helper->mac . ": " . ($http->error || 'unknown'));
}
```

### Thresholds

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Health check interval | 60s (12 × 5s) | Balance: detection speed vs HTTP overhead |
| `session_valid=false` | Immediate restart | librespot explicitly reports dead session |
| Session age threshold | 14400s (4h) | Safety net for silently-dead TCP connections. Linux TCP keepalive default is 7200s (2h), so most dead connections are caught by `session_valid=false` within 2-3h. 4h catches edge cases. |
| Idle threshold | 300s (5 min) | Guard against restarting during active playback. User playing music → idle_secs ≈ 0. |
| HTTP timeout | 5s | Generous for localhost request |

### Connect safety

The age+idle restart happens regardless of Connect state. This is intentional:
- If Connect is active, `idle_secs` will be low (< 300s) → threshold not met → no restart
- If Connect was active but stopped >5 min ago AND session is >4h old → safe to restart (Connect is idle too)
- The Spirc reconnect path (ZeroConf) will re-establish Connect after the daemon restart

### Edge case: health check response arrives after daemon was already restarted

`_onHealthResponse` receives `$helper` by reference. If `stopHelper` was called between the HTTP request and the callback (e.g., by a concurrent initHelpers cycle), `$helper->alive` returns false. Check alive before acting:

```perl
return unless $helper && $helper->alive;
```

---

## Part 3: Log Cleanup

### Problem

`initHelpers` runs every 60s. Lines 163 and 173 log `"Starting Unified daemon for ..."` unconditionally, BEFORE calling `startHelper()`. Since `startHelper()` returns without action when the daemon is alive and unchanged, these messages are misleading. 800 entries/day of "Starting" that don't start anything.

### Fix

Downgrade lines 163 and 173 from INFOLOG to DEBUGLOG:

**Line 163 (sync master):**
```perl
# Before:
main::INFOLOG && $log->is_info && $log->info(
    "Starting Unified daemon for sync group master: $syncMasterId"
);

# After:
main::DEBUGLOG && $log->is_debug && $log->debug(
    "Evaluating Unified daemon for sync group master: $syncMasterId"
);
```

**Line 173 (standalone player):**
```perl
# Before:
main::INFOLOG && $log->is_info && $log->info(
    "Starting Unified daemon for player: " . $client->id
);

# After:
main::DEBUGLOG && $log->is_debug && $log->debug(
    "Evaluating Unified daemon for player: " . $client->id
);
```

`startHelper()` itself already logs at INFO when it actually creates or restarts a daemon (lines 317, 321). Those messages remain at INFO — they are the authoritative "daemon was started" signal.

### Additionally

Line 127 (`"Checking SpotOn Unified helper daemons..."`) also fires every 60s. Downgrade to DEBUGLOG:

```perl
# Before:
main::INFOLOG && $log->is_info && $log->info("Checking SpotOn Unified helper daemons...");

# After:
main::DEBUGLOG && $log->is_debug && $log->debug("Checking SpotOn Unified helper daemons...");
```

**Net effect:** Silent 60s watchdog cycles. Only actual daemon lifecycle events (create, restart, crash, health restart) appear in the INFO log.

---

## Files Changed

| File | Changes |
|------|---------|
| `librespot-spoton/src/unified.rs` | Add `session_created_at`, `last_activity` shared state; enhance `/health` to return JSON; pass new state through `unified_http_server` + service closures; update activity timestamps in Browse success + Connect relay |
| `Plugins/SpotOn/Unified/DaemonManager.pm` | Add `_onHealthResponse`, `_onHealthError`, `_restartForHealth` methods; health check logic in `_streamAlivePoll`; log level downgrades in `initHelpers` |
| `Plugins/SpotOn/Unified/Daemon.pm` | Add `_healthCheckCount` accessor |

## What This Does NOT Change

- **`serve_track_request` (browse.rs):** No changes to Browse track handling. Mid-stream failures are detected indirectly via session health monitoring, not by counting short tracks.
- **Connect reconnect logic:** Existing ZeroConf + Spirc reconnect paths untouched.
- **`consecutive_browse_fails` mechanism:** Still active — catches rapid 404 failures. Health monitoring is a complementary, not replacement, mechanism.
- **Diagnose Mode toggle behavior:** Still triggers full daemon restart (Plugin.pm:190-203). Unchanged.

## Build & Deploy

- Rust binary rebuild required (CI tag push)
- Perl changes are hot-reloadable (LMS restart)
- Binary + Perl must ship together (health check JSON depends on new binary)
- Version bump needed (discussed with user before tagging)

## Testing

1. **Start daemon, wait >4h idle, verify health check triggers restart** — observe "Health check restart" in server.log, followed by fresh daemon start with new PID
2. **Start daemon, play continuously for >4h, verify NO restart** — idle_secs stays low, health check passes silently
3. **Kill Spotify session manually (network disconnect), verify health check detects `session_valid=false`** — restart within 60s
4. **Verify log cleanup** — watchdog cycle produces no INFO output; only actual starts/restarts logged
5. **Verify Connect survives health restart** — idle Connect session (>5 min no activity, >4h session age) → daemon restarts → ZeroConf re-announces → Spotify app sees device within ~10s
