---
phase: 35-liked-songs-play-all-throttle
plan: 01
status: complete
completed: 2026-06-26
commits:
  - dea055f
  - 0a1a617
  - 37d88ae
  - ff90482
---

## Summary

Fixed GitHub Issue #51: Playing all Liked Songs (1633 tracks) caused Browse daemon 404s from concurrent API load. Material Skin browse requests + _fetchAllPages play-all pagination were hitting Spotify simultaneously, causing ~42 redundant me/tracks API calls.

## Approach

First attempt used incremental onPage callback in _fetchAllPages to populate $_playAllItemCache after each page. This was reverted due to complexity — the callback pattern didn't compose cleanly with the existing async flow.

Final fix: deferred metadata cache writes to background batches. Instead of writing cache entries synchronously during pagination, the play-all flow now batches metadata writes and flushes them after pagination completes, preventing concurrent browse requests from triggering independent API calls.

## Key Changes

- `Plugin.pm`: Background batch writes for play-all metadata cache
- Eliminated ~42 redundant API calls for large liked songs libraries
- Concurrent Material Skin browse requests now served from cache during play-all pagination

## Released

v2.1.2
