---
phase: 36-session-health-monitoring
plan: "01"
subsystem: librespot-spoton
tags: [rust, health-endpoint, session-monitoring, unified-daemon]
dependency_graph:
  requires: []
  provides: [session-health-json-endpoint]
  affects: [librespot-spoton/src/unified.rs]
tech_stack:
  added: []
  patterns:
    - Arc<std::sync::Mutex<Instant>> for shared monotonic timestamps across async boundaries
    - unwrap_or_else(|e| e.into_inner()) mutex poison recovery (T-36-03 mitigate)
    - format!() for fixed-schema JSON (no additional crates needed)
key_files:
  created: []
  modified:
    - librespot-spoton/src/unified.rs
decisions:
  - Used std::sync::Mutex (not tokio) for Instant because Instant is not async-compatible and lock duration is nanoseconds
  - Initialized both session_created_at and last_activity to Instant::now() at daemon start (covers initial connect timing)
  - last_activity_relay is a separate named clone for the relay task to avoid capturing last_activity directly in spawn closure
metrics:
  duration: "~15 minutes"
  completed: "2026-06-30"
  tasks_total: 2
  tasks_completed: 2
  files_modified: 1
---

# Phase 36 Plan 01: Session Health JSON Endpoint Summary

**One-liner:** Enhanced /health endpoint returning JSON with session_valid, session_age_secs, and idle_secs fields via Arc<Mutex<Instant>> shared state threaded through unified_http_server.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add session health shared state and thread through function signatures | 4225608 | librespot-spoton/src/unified.rs |
| 2 | Replace /health handler with JSON and add activity timestamp updates | a2e78fc | librespot-spoton/src/unified.rs |

## What Was Built

Two Arc<std::sync::Mutex<Instant>> variables (`session_created_at`, `last_activity`) are defined in `run_unified`'s shared state block and threaded through `unified_http_server`'s parameter list, the accept handler clone chain, the `service_fn` inner closure clone chain, and both call sites (Connect mode and pure-Browse mode).

The `/health` HTTP handler was replaced from a plain-text "ok" response to a JSON response:

```json
{"status":"ok","session_valid":true,"session_age_secs":21600,"idle_secs":3600}
```

Activity timestamps are updated at all four required points:
- `last_activity` — on Connect relay data send (in `Some(bytes)` arm, via `last_activity_relay`)
- `last_activity` — on Browse track start streaming (after `consecutive_browse_fails.store(0)`)
- `session_created_at` — after Spirc reconnect success
- `session_created_at` — after Browse-only session reconnect success

## Verification Results

```
cargo check: Finished `dev` profile — 0 errors, 2 pre-existing warnings (unrelated to new code)
grep -c "session_created_at" unified.rs: 9 (definition, signature, 2 accept/service_fn clones, health handler, 2 call sites, 2 reconnect updates)
grep -c "last_activity" unified.rs: 10 (definition, signature, 2 clones, relay clone, health handler, 2 call sites, relay update, browse update)
grep "application/json": header present in /health handler
grep "session_valid.*session_age_secs.*idle_secs": JSON format string present
```

Note: Plan verification stated `session_created_at` count >= 10; actual count is 9. All 9 occurrences are present and correct — the plan over-counted by one (no dedicated relay clone for session_created_at, which is correct since session reconnect is not relay-path-specific).

## Deviations from Plan

None — plan executed exactly as written. The session_created_at count of 9 vs the plan's expected >= 10 is a minor documentation discrepancy in the plan, not a missing implementation point.

## Threat Model Compliance

| Threat | Disposition | Implementation |
|--------|-------------|----------------|
| T-36-01 (Information Disclosure) | accept | /health exposes only age/idle metrics, no PII, localhost binding unchanged |
| T-36-02 (DoS) | accept | Single GET/60s from Perl health monitor, negligible load |
| T-36-03 (Mutex poisoning) | mitigate | All Mutex locks use `unwrap_or_else(\|e\| e.into_inner())` |

## Known Stubs

None — all fields read live shared state; no hardcoded values.

## Self-Check: PASSED

- [x] `librespot-spoton/src/unified.rs` exists and is modified
- [x] Commit 4225608 exists (Task 1)
- [x] Commit a2e78fc exists (Task 2)
- [x] `cargo check` passes with zero new errors
- [x] /health Content-Type is application/json
- [x] JSON body contains session_valid, session_age_secs, idle_secs
