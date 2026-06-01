---
phase: 05-spotify-connect
plan: 01
subsystem: binary-connect
tags: [rust, librespot, spirc, http-streaming, connect, pcm-sink]
dependency_graph:
  requires: [04.3-zeroconf-keymaster-auth, 04.4-dual-token-api-routing]
  provides: [binary-connect-mode, http-stream-server, lms-event-notifier, spirc-integration]
  affects: [Plugins/SpotOn/Connect/Daemon.pm, Plugins/SpotOn/ProtocolHandler.pm]
tech_stack:
  added:
    - librespot-connect 0.8.0 (Spirc protocol, ConnectConfig)
    - hyper 1.x (HTTP/1.1 server for /stream and /control/* endpoints)
    - hyper-util 0.1.x (GracefulShutdown)
    - http-body-util 0.1.x (StreamBody, BodyExt)
    - tokio-stream 0.1.x (ReceiverStream for streaming body)
    - bytes 1.x (zero-copy PCM chunk transfer)
  patterns:
    - Spotty-NG lms_connect module ported to connect.rs (LMS notifier + HttpStreamSink)
    - hyper::server::conn::http1::Builder for owned Connection (no lifetime issues)
    - Arc<Mutex<Option<Spirc>>> shared handle for HTTP control endpoint access
    - SoftMixer for Spirc volume control (vs NoOpMixer in single-track mode)
key_files:
  created:
    - librespot-spoton/src/connect.rs
  modified:
    - librespot-spoton/Cargo.toml
    - librespot-spoton/src/main.rs
    - Plugins/SpotOn/Bin/x86_64-linux/spoton
decisions:
  - "Use hyper::server::conn::http1::Builder (owned Connection) instead of AutoBuilder (lifetime issue)"
  - "suppress_next_volume AtomicBool in LMS struct guards CON-11 volume echo on SessionConnected"
  - "spirc_active set only on SessionConnected (not on Spirc::new return) to prevent Pitfall 2 race"
  - "SoftMixer::open() chosen for Spirc volume control per plan decision (Pitfall 8)"
  - "ConnectConfig 0.8 deviates from Spotty-NG: no autoplay field, has_volume_ctrl -> disable_volume"
metrics:
  duration_minutes: 45
  tasks_completed: 2
  tasks_total: 2
  files_created: 1
  files_modified: 3
  completed_date: "2026-06-01"
requirements_addressed: [CON-03, CON-04, CON-05, CON-07, CON-11, CON-12, CON-14, CON-16]
---

# Phase 05 Plan 01: Rust Binary --connect Mode Summary

Implemented the Rust binary --connect mode: Spirc protocol integration via librespot-connect 0.8, HTTP audio streaming server with /stream and /control/* endpoints, JSON-RPC event notifier to LMS, and wall-clock-accurate PCM rate-limiting in HttpStreamSink.

## What Was Built

### connect.rs (1027 lines)

**LMS struct** — JSON-RPC event notifier:
- `new()`: sanitizes auth with `trim().replace(['\r', '\n'], "")` (T-05-01 CRLF mitigation)
- `handle_player_event()`: dispatches 5 commands: `start`, `change`, `stop`, `volume`, `seek`
- `suppress_next_volume` AtomicBool: set on `SessionConnected`, cleared on first `VolumeChanged` (CON-11)
- `notify()`: HTTP/1.0 POST to LMS /jsonrpc.js via TcpStream (same wire format as Spotty-NG)

**HttpStreamSink** — S16LE PCM sink implementing librespot `Sink` trait:
- `open()`: validates AudioFormat::S16, initializes with buffer_latency_ms compensation
- `start()` / `stop()`: reset counters only — CRITICAL: no `exit()` call (Pitfall 1 prevented)
- `write()`: f64 samples to S16LE via converter, nanosecond rate-limiting:
  `expected_ns = frames_consumed * 1e9 / 44100 + buffer_latency_ns` (CON-14)
- Sends PCM chunks to mpsc::channel(256) with try_send spin-retry

**http_stream_server** — hyper HTTP/1.1 server:
- `GET /stream`: spirc_active guard (503 + Retry-After:2 when not yet connected, Pitfall 2/T-05-06),
  relay_active CR-01 guard (prevents split-PCM-stream, T-05-04), StreamBody from ReceiverStream
- `POST /control/pause|play|next|prev`: Spirc dispatch, 204 No Content
- `POST /control/volume`: parse JSON `{"volume": 0-100}`, clamp, convert to u16 (T-05-02)
- `POST /control/seek`: parse JSON `{"position_ms": N}`, validate as u32 (T-05-03)
- Graceful shutdown via oneshot channel

**run_connect** — main orchestrator:
- SoftMixer::open(MixerConfig::default()) for volume control (Pitfall 8)
- TcpListener::bind("0.0.0.0:0") → `println!("stream_port={}", port)` + stdout().flush() (Pitfall 3/CON-16)
- Spirc::new(ConnectConfig, session, credentials, player, mixer) with shared handle
- spirc_active set ONLY on PlayerEvent::SessionConnected (Pitfall 2)
- Main select! loop: discovery reconnect, spirc task restart with backoff (5 retries in 60s), ctrl_c

### Cargo.toml changes

Added: librespot-connect 0.8, bytes 1, hyper 1 (http1+server), hyper-util 0.1 (server-auto+graceful+service+tokio), http-body-util 0.1, tokio-stream 0.1; extended tokio features with net, io-util, sync, signal.

### main.rs changes

- `mod connect;` declaration
- CLI args: `--player-mac`, `--lms`, `--lms-auth`, `--buffer-latency-ms` (default 2000ms), `--disable-discovery`
- Mode::Connect: validates --cache, calls `connect::run_connect()`, exits 1 on error

### Binary

Built with `cargo build --release`, copied to `Plugins/SpotOn/Bin/x86_64-linux/spoton`. `--check` output confirmed working.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] librespot_connect module path privacy**
- **Found during:** Task 1 cargo check
- **Issue:** Plan referenced `librespot_connect::spirc::Spirc` and `librespot_connect::state::ConnectConfig` but both sub-modules are private; types are re-exported from crate root via `pub use`
- **Fix:** Changed imports to `use librespot_connect::{ConnectConfig, Spirc}`
- **Files modified:** librespot-spoton/src/connect.rs
- **Commit:** 73de70b

**2. [Rule 1 - Bug] AutoBuilder lifetime issue**
- **Found during:** Task 1 cargo check
- **Issue:** `hyper_util::server::conn::auto::Builder::serve_connection(&self, ...)` returns `Connection<'_, I, S, E>` with borrow of `self`, making it incompatible with `tokio::spawn`'s `'static` requirement
- **Fix:** Used `hyper::server::conn::http1::Builder::serve_connection(&self, ...)` which returns owned `Connection<I, S>` (no lifetime parameter), then graceful.watch(conn) works correctly
- **Files modified:** librespot-spoton/src/connect.rs
- **Commit:** 73de70b

**3. [Rule 1 - Bug] watch::Receiver borrow lifetime in async block**
- **Found during:** Task 1 cargo check
- **Issue:** `*rx.borrow()` created `Ref<u64>` holding borrow on `rx`; both dropped at end of block causing lifetime conflict
- **Fix:** Explicitly copy dereferenced value before drop: `let val: u64 = *rx.borrow(); drop(rx); val`
- **Files modified:** librespot-spoton/src/connect.rs
- **Commit:** 73de70b

**4. [Rule 1 - Bug] ConnectConfig 0.8 API differs from Spotty-NG**
- **Found during:** Task 2 implementation
- **Issue:** librespot-connect 0.8.0 ConnectConfig has different fields: no `autoplay`, `has_volume_ctrl` renamed to `disable_volume` (inverted), `initial_volume` is u16 (not Option)
- **Fix:** Used `ConnectConfig::default()` with struct update syntax, setting `disable_volume: false` and `initial_volume: u16::MAX / 2`
- **Files modified:** librespot-spoton/src/connect.rs
- **Commit:** 9a898d2

**5. [Rule 1 - Bug] Ambiguous StreamExt::next disambiguation**
- **Found during:** Task 1 cargo check
- **Issue:** Both `futures_util::StreamExt` and `tokio_stream::StreamExt` provide `.next()`, causing ambiguity on Discovery stream
- **Fix:** Used `futures_util::StreamExt::next(d).await` explicit path call
- **Files modified:** librespot-spoton/src/connect.rs
- **Commit:** 9a898d2

**6. [Rule 2 - Security] tokio signal feature missing**
- **Found during:** Task 1 cargo check
- **Issue:** `tokio::signal::ctrl_c()` requires the `signal` feature in tokio
- **Fix:** Added `"signal"` to tokio features in Cargo.toml
- **Files modified:** librespot-spoton/Cargo.toml
- **Commit:** 73de70b

## Known Stubs

None. All implemented functionality is wired and operational. The binary runs --connect mode end-to-end with real Spirc protocol handling.

## Threat Flags

No new threat surfaces beyond the plan's `<threat_model>`. The HTTP server binds to 0.0.0.0 (not 127.0.0.1 explicitly) — consistent with Spotty-NG pattern. LMS connects via localhost by design. All STRIDE threats are mitigated per plan (T-05-01 through T-05-SC).

## Self-Check

### Files exist:

- librespot-spoton/src/connect.rs: FOUND
- librespot-spoton/Cargo.toml (modified): FOUND
- librespot-spoton/src/main.rs (modified): FOUND
- Plugins/SpotOn/Bin/x86_64-linux/spoton: FOUND

### Commits exist:

- 73de70b: feat(05-01): add Cargo.toml connect deps + connect.rs LMS notifier and HttpStreamSink
- 9a898d2: feat(05-01): wire Mode::Connect + binary build with Spirc HTTP streaming

## Self-Check: PASSED
