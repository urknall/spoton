---
status: complete
phase: 19-podcast-browse
source: [19-VERIFICATION.md]
started: "2026-06-14T18:30:00Z"
updated: "2026-06-14T20:25:00Z"
---

## Current Test

[testing complete]

## Tests

### 1. Top-Level Menu Placement
expected: Podcasts entry visible after Bibliothek in LMS browser UI
result: pass

### 2. Saved Shows Browse
expected: Saved shows listed with publisher on line2, most-recently-added first (live getSavedShows call)
result: pass (fixed — publisher inline in name via middle-dot separator)

### 3. Episode List with Duration and Date
expected: Episode line2 shows German duration/date format (e.g. "45 Min · Gestern")
result: pass

### 4. Episode Playback
expected: Selecting an episode triggers spoton://episode:ID via ProtocolHandler, audio plays
result: pass

### 5. Podcast Search
expected: type=search renders LMS text input; query returns Shows and Episoden sections with live results
result: pass

## Summary

total: 5
passed: 5
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps

[none]
