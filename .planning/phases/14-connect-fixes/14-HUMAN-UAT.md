---
status: partial
phase: 14-connect-fixes
source: [14-VERIFICATION.md]
started: "2026-06-07T12:30:00Z"
updated: "2026-06-07T12:30:00Z"
---

## Current Test

[awaiting human testing]

## Tests

### 1. Multi-Player Credential Isolation
expected: Connect two Spotify accounts to two different players simultaneously; each player's Browse session shows its own Spotify library. Cache dirs are separate (spoton/connect-{mac1}/ vs spoton/connect-{mac2}/).
result: [pending]

### 2. Volume Match at Connect Start
expected: Start Connect session — Spotify app shows LMS volume level within 3 seconds, no jump from 50% default.
result: [pending]

### 3. Volume Sync Speed
expected: Change volume in Spotify app during active session — LMS reflects change within 3 seconds (was 20s).
result: [pending]

## Summary

total: 3
passed: 0
issues: 0
pending: 3
skipped: 0
blocked: 0

## Gaps
