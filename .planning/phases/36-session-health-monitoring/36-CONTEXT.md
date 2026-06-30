# Phase 36: Session Health Monitoring - Context

**Gathered:** 2026-06-30
**Status:** Ready for planning
**Source:** Spec (`.planning/specs/session-health-SPEC.md`)

<domain>
## Phase Boundary

Prevent cold-start playback failure after overnight daemon idle. The librespot daemon's Spotify session (Shannon TCP) goes stale after hours of idle. The daemon process stays alive (HTTP server responds to alive polls), but the Spotify data channel is dead. First track plays 2-4 seconds of buffered audio then stops.

This phase enhances the `/health` endpoint with session health reporting and adds Perl-side monitoring that acts on it, plus cleans up misleading watchdog log noise.

</domain>

<decisions>
## Implementation Decisions

### Rust: `/health` Endpoint
- Return JSON: `{"status":"ok","session_valid":bool,"session_age_secs":u64,"idle_secs":u64}`
- `session_valid` = `!session.is_invalid()` from librespot Session object
- `session_age_secs` = seconds since last session connect/reconnect (monotonic Instant)
- `idle_secs` = seconds since last successful Browse track or Connect audio activity
- No additional crate dependencies — use `format!()` for fixed-schema JSON

### Rust: Shared State
- Add `session_created_at: Arc<Mutex<Instant>>` — updated on initial connect, Spirc reconnect, Browse-only reconnect, ZeroConf rotation
- Add `last_activity: Arc<Mutex<Instant>>` — updated on successful Browse track completion (after `consecutive_browse_fails.store(0)`) and Connect relay data send
- Both passed through `unified_http_server` function signature and cloned into service closures

### Perl: Health-Aware Alive Poll
- New `_healthCheckCount` accessor on `Daemon.pm`
- Every 12th `_streamAlivePoll` cycle (~60s): async `GET http://127.0.0.1:{port}/health`
- Parse JSON response, restart on: `session_valid=false` OR (`session_age_secs > 14400` AND `idle_secs > 300`)
- Connect safety: if Connect is active, `idle_secs` stays low → age threshold never met
- Race condition guard: check `$helper->alive` before acting on async response

### Perl: Restart Method
- New `_restartForHealth($class, $helper, $reason)` method
- Calls `stopHelper` → `startHelper` (clean daemon lifecycle)
- Logs at INFO: `"Health check restart for {mac}: {reason}"`

### Log Cleanup
- `initHelpers` line 127 ("Checking SpotOn Unified helper daemons..."): INFO → DEBUG
- `initHelpers` line 163 ("Starting Unified daemon for sync group master"): INFO → DEBUG, rename to "Evaluating"
- `initHelpers` line 173 ("Starting Unified daemon for player"): INFO → DEBUG, rename to "Evaluating"
- `startHelper` lines 317, 321 ("Need to create/re-start"): remain at INFO — these are actual actions

### Claude's Discretion
- Error handling for malformed health JSON responses
- HTTP timeout value for health check requests (spec suggests 5s)
- Whether to reset `_healthCheckCount` on daemon restart

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Spec
- `.planning/specs/session-health-SPEC.md` — Complete specification with thresholds, code examples, edge cases

### Rust Source
- `librespot-spoton/src/unified.rs` — Unified daemon: `/health` endpoint (lines 287-298), shared state, `run_unified()`, session reconnect logic
- `librespot-spoton/src/browse.rs` — `serve_track_request()` for Browse activity tracking

### Perl Source
- `Plugins/SpotOn/Unified/DaemonManager.pm` — `_streamAlivePoll()` (lines 201-228), `initHelpers()` (lines 105-199), `startHelper()` (lines 254-337)
- `Plugins/SpotOn/Unified/Daemon.pm` — Accessor definitions (line 26-38), `alive()` (line 411-413)

</canonical_refs>

<specifics>
## Specific Ideas

### Thresholds (from Spec)
| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Health check interval | 60s (12 × 5s) | Balance detection speed vs overhead |
| `session_valid=false` | Immediate restart | Explicit dead session |
| Session age threshold | 14400s (4h) | Safety net for silent TCP death |
| Idle threshold | 300s (5 min) | Guard against restart during active use |
| HTTP timeout | 5s | Generous for localhost |

</specifics>

<deferred>
## Deferred Ideas

- Active session probe via Mercury ping (more accurate but adds latency to health checks)
- Mid-stream failure detection (track plays <10s → count as failure, increment `consecutive_browse_fails`)
- Rust-side periodic session keepalive (daemon self-heals without Perl involvement)

</deferred>

---

*Phase: 36-session-health-monitoring*
*Context gathered: 2026-06-30 via Spec*
