---
status: partial
phase: 37-context-menu-lms-items
source: [37-VERIFICATION.md]
started: 2026-06-30T17:30:00Z
updated: 2026-06-30T17:30:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Standard LMS items appear in More menu
expected: Press More on a SpotOn track — Add to Favorites, play controls, and More Info appear alongside SpotOn items (Artist View, Album View, Like, Add to Playlist)
result: [pending]

### 2. Add to Favorites creates a working entry
expected: Adding a spoton:// track to LMS Favorites creates a playable entry that streams correctly
result: [pending]

### 3. Episode MANAGE_FOLLOW is selectable (CR-01, pre-existing)
expected: In episode More menu, the Follow/Unfollow item is clickable (not just text). Note: CR-01 found missing type => 'link' at Plugin.pm line 603 — this is pre-existing, not introduced by Phase 37
result: [pending]

## Summary

total: 3
passed: 0
issues: 0
pending: 3
skipped: 0
blocked: 0

## Gaps
