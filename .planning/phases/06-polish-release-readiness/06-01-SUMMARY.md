---
phase: 06-polish-release-readiness
plan: "01"
subsystem: api
tags: [consolidation, exporter, recommendations, dstm, custom-binary]
dependency_graph:
  requires: []
  provides: [SPOTON_DEFAULT_CLIENT_ID-export, recommendations-method, LMS-10-verified]
  affects: [TokenManager.pm, Plan-03-DSTM]
tech_stack:
  added: []
  patterns:
    - "Exporter 'import' with @EXPORT_OK for inter-module constant sharing"
    - "lazy `require` + direct sub call to avoid circular compile-time dependency"
key_files:
  created: []
  modified:
    - Plugins/SpotOn/API/Client.pm
    - Plugins/SpotOn/API/TokenManager.pm
decisions:
  - "Use lazy require+direct-call pattern for TokenManager constant import (avoids circular use)"
  - "recommendations() returns $cb->([]) on empty seed — no-op guard prevents spurious API calls"
  - "Helper.pm verified complete for LMS-10; no code changes needed"
metrics:
  duration: "3 minutes"
  completed: "2026-06-03T09:13:59Z"
  tasks_completed: 2
  tasks_total: 2
  files_changed: 2
---

# Phase 6 Plan 01: Client-ID Consolidation + recommendations() + Custom Binary Summary

Client-ID consolidated to single source of truth in Client.pm (D-04/SC-7), recommendations() API method added as DSTM prerequisite, LMS-10 custom binary support verified complete.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Client-ID Exporter + recommendations() in Client.pm | 48fc89a | Plugins/SpotOn/API/Client.pm |
| 2 | TokenManager.pm Client-ID import + Helper.pm verification | 1cea44d | Plugins/SpotOn/API/TokenManager.pm |

## What Was Built

### Task 1: Client-ID Exporter + recommendations()

Added to `Plugins/SpotOn/API/Client.pm`:

- `use Exporter 'import'` and `our @EXPORT_OK = qw(SPOTON_DEFAULT_CLIENT_ID)` — makes the constant importable by other modules
- `sub recommendations($class, $accountId, $params, $cb)` — full API method for DSTM:
  - Accepts `seed_tracks` and `seed_artists` arrayrefs, converts to comma-separated strings
  - `limit` defaults to 25
  - Guard: calls `$cb->([])` immediately when neither seed_tracks nor seed_artists provided
  - Uses `_noCache => 1` (fresh recommendations for Don't Stop The Music)
  - Wraps `_request()` callback to extract `$result->{tracks}` or return `[]` on failure
  - `recommendations` endpoint already in `@KNOWN_DEPRECATED_FAMILIES` — bundled-token routing automatic, no manual token-switch needed

### Task 2: TokenManager.pm Deduplication + Helper.pm Verification

**TokenManager.pm:** Removed the duplicate `use constant SPOTON_DEFAULT_CLIENT_ID => '93aac68fb06348598c1e67734dfaceee'` (was line 22). Replaced with the safe circular-dependency-avoiding pattern:

```perl
use constant SPOTON_DEFAULT_CLIENT_ID => do {
    require Plugins::SpotOn::API::Client;
    Plugins::SpotOn::API::Client::SPOTON_DEFAULT_CLIENT_ID();
};
```

The `require` is lazy (runtime, not compile-time), avoiding the circular load that a `use` would trigger (Client.pm requires TokenManager.pm at runtime via `_doFlavouredRequest`).

**Helper.pm (LMS-10 verification):** No code changes needed. Confirmed:
- `get()` (line 35): checks `$prefs->get('binary')` first — custom pref-path override works
- `_findBin()` (line 130): `unshift @candidates, HELPER . '-custom'` — `spoton-custom` is already the first search candidate
- `helperCheck()` (line 67): shell-safe quoting with `s/'/'\\''/g` prevents injection on user-supplied path

## Verification Results

| Check | Result |
|-------|--------|
| Client.pm: `use Exporter 'import'` present | PASS |
| Client.pm: `@EXPORT_OK = qw(SPOTON_DEFAULT_CLIENT_ID)` present | PASS |
| Client.pm: `sub recommendations` present | PASS |
| Client.pm: SPOTON_DEFAULT_CLIENT_ID defined exactly once | PASS (count=1) |
| TokenManager.pm: no literal 93aac68fb06348598c1e67734dfaceee in non-comment lines | PASS (count=0) |
| TokenManager.pm: `Plugins::SpotOn::API::Client::SPOTON_DEFAULT_CLIENT_ID()` present | PASS |
| TokenManager.pm loads with correct constant value | PASS |
| Helper.pm: `spoton-custom` in `_findBin` candidates | PASS |
| Helper.pm: `$prefs->get('binary')` checked first in `get()` | PASS |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None.

## Threat Flags

None — no new network endpoints, auth paths, or schema changes introduced. T-06-02 (seed param URI-escaping) handled by existing `_request()` pipeline as noted in plan threat model.

## Self-Check: PASSED

- `Plugins/SpotOn/API/Client.pm` modified and committed at 48fc89a
- `Plugins/SpotOn/API/TokenManager.pm` modified and committed at 1cea44d
- Both commits verified in `git log --oneline`
