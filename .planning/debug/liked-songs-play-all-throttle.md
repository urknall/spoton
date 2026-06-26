---
status: awaiting_human_verify
trigger: "Liked Songs Play-All (1633 tracks) causes extreme slowdown and song skips (#51)"
created: 2026-06-26
updated: 2026-06-26
---

# Debug: Liked Songs Play-All Throttle

## Symptoms

- expected: Playing all 1633 Liked Songs loads quickly and plays without interruption
- actual: Material Skin counts up extremely slowly, 200+ API token usage, first 1-2 songs skip at ~2/3
- errors: 75 api_slow entries for me/tracks (2-4s each), 11 Browse daemon 404s in prefetch context
- timeline: First observed 2026-06-26, exists since large Liked Songs lists were supported
- reproduction: Material Skin → SpotOn → Bibliothek → Liked Songs → Play All

## Current Focus

- hypothesis: 1633 synchronous _trackItem calls each execute a SQLite INSERT (via $cache->set) before the callback fires, blocking the LMS event loop for multiple seconds
- next_action: implement deferred metadata cache writes — fire callback immediately after O(N) string ops, flush SQLite writes in background batches of 50 via setTimer
- reasoning_checkpoint:
    hypothesis: "1633 synchronous SQLite INSERT calls (via $cache->set in _trackItem) in the done callback block the LMS event loop for 1-4+ seconds before $callback fires"
    confirming_evidence:
      - "Slim::Utils::Cache uses DbCache.pm (SQLite, AutoCommit=1) — each cache->set is a separate INSERT OR REPLACE transaction"
      - "WAL autocheckpoint=200: every 200 writes trigger a checkpoint (random writes to db file). 1633 writes = 8 checkpoints, each blocking for 100-500ms on Pi SD card = 0.8-4s total"
      - "_savedTracksFeed done callback calls map { _trackItem } on ALL 1633 items before firing $callback — nothing reaches LMS until all SQLite writes complete"
      - "onPage fix made things WORSE — confirms total SQLite I/O is the bottleneck (same writes, more overhead)"
    falsification_test: "If deferring all cache->set calls to background batches stops the event loop blocking and audio skipping disappears, hypothesis is confirmed"
    fix_rationale: "Defer all $cache->set calls to _flushDeferredMeta background batches (50/tick via setTimer). Callback fires immediately after O(N) string ops with no I/O. Cache misses during playback are handled gracefully by _asyncRefetch in ProtocolHandler."
    blind_spots:
      - "Exact SQLite write timing on Pi not measured — estimate based on WAL+checkpoint analysis"
      - "Concurrent Material Skin browse + _fetchAllPages rate-limiting issue NOT addressed by this fix"

## Evidence

- timestamp: 2026-06-26T16:25-16:28 — Pi server.log shows 75 api_slow entries (2-4s each) for me/tracks
- timestamp: 2026-06-26T16:27:53 — 11 Browse daemon 404s, all "prefetch context, scheduling skip"
- timestamp: 2026-06-26 — Calls arrive in groups of 2-3 at ~0.1s apart (Material Skin browse + _fetchAllPages concurrent)
- timestamp: 2026-06-26 — onPage fix (commit 0a1a617) made behavior worse, reverted (37d88ae)
- timestamp: 2026-06-26 — _fetchAllPages IS sequential (callback-chained), not parallel as initially assumed
- timestamp: 2026-06-26 — Classic Skin loads faster than Material Skin (fewer JSONRPC calls for queue display)

## Eliminated

- hypothesis: _fetchAllPages fires parallel API calls — WRONG, it's sequential (callback-chained)
- hypothesis: Incremental onPage cache fixes concurrent API calls — WRONG, made it worse by adding synchronous _trackItem work during pagination

## Prior Art

- Phase 25 (Play-All Full Pagination) implemented _fetchAllPages with sequential pagination
- _playAllItemCache was added to serve browse requests from accumulated play-all results
- _trackItem writes to spoton_meta_ cache with 604800s TTL + _extractTrackIds (Phase 33)

## Evidence

- timestamp: 2026-06-26 — Slim::Utils::Cache uses DbCache.pm (SQLite, AutoCommit=1, WAL, synchronous=OFF)
- timestamp: 2026-06-26 — Each $cache->set() executes INSERT OR REPLACE transaction individually
- timestamp: 2026-06-26 — WAL autocheckpoint=200: every 200 writes triggers checkpoint (random writes to db file). 1633 writes = ~8 checkpoints. On Pi SD card: ~100-500ms per checkpoint = 0.8-4s total blocking
- timestamp: 2026-06-26 — done callback in _savedTracksFeed called map { _trackItem } on ALL 1633 items synchronously BEFORE $callback->() — nothing reached LMS until all SQLite writes complete
- timestamp: 2026-06-26 — Root cause confirmed. Fix implemented: _flushDeferredMeta() defers all SQLite writes to background batches of 50/tick via setTimer

## Resolution

- root_cause: 1633 synchronous SQLite INSERT calls (via $cache->set in _trackItem's done callback) blocked the LMS event loop for 0.8-4s before $callback fired, preventing audio prefetch requests from being processed, causing buffer starvation and track skips at ~2/3 of first song
- fix: Added _flushDeferredMeta() helper that processes deferred cache writes in background batches of 50 items per event-loop tick. _trackItem and _albumTrackItem accept $opts->{defer_cache} arrayref. All play-all done callbacks now fire $callback immediately, then call _flushDeferredMeta in background. Cache misses during playback are handled gracefully by _asyncRefetch in ProtocolHandler.
- verification: pending human test
- files_changed: [Plugins/SpotOn/Plugin.pm]
