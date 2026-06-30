---
status: passed
phase: 37-context-menu-lms-items
source: [37-VERIFICATION.md]
started: 2026-06-30T17:30:00Z
updated: 2026-06-30T17:30:00Z
---

## Current Test

[complete]

## Tests

### 1. Standard LMS items appear in More menu
expected: Press More on a SpotOn track — Add to Favorites, play controls, and More Info appear alongside SpotOn items (Artist View, Album View, Like, Add to Playlist)
result: passed — items appear, SpotOn: prefix added for distinction

### 2. Add to Favorites creates a working entry
expected: Adding a spoton:// track to LMS Favorites creates a playable entry that streams correctly
result: passed — favorites work, cover art now shows via getIcon

### 3. Episode MANAGE_FOLLOW is selectable (CR-01, pre-existing)
expected: In episode More menu, the Follow/Unfollow item is clickable (not just text). Note: CR-01 found missing type => 'link' at Plugin.pm line 603 — this is pre-existing, not introduced by Phase 37
result: passed — item is functional

## Summary

total: 3
passed: 3
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps
