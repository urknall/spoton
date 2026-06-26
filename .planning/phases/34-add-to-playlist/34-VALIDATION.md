---
phase: 34
slug: add-to-playlist
status: validated
nyquist_compliant: false
wave_0_complete: false
created: 2026-06-26
---

# Phase 34 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Perl Test::More |
| **Config file** | none |
| **Quick run command** | `prove -v t/08_api_client.t` |
| **Full suite command** | `prove -v t/` |
| **Estimated runtime** | ~5 seconds |

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Status |
|---------|------|------|-------------|-----------|--------|
| 34-01-01 | 01 | 1 | ATP-01 (addToPlaylist API) | manual | ✅ UAT passed |
| 34-01-01 | 01 | 1 | ATP-02 (playlistId validation) | manual | ✅ UAT passed |
| 34-01-02 | 01 | 1 | ATP-03 (menu items) | manual | ✅ UAT passed |

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Add to Playlist shows in More menu for tracks | ATP-03 | Requires live LMS + Spotify account | Play track → More → verify "Zu Playlist hinzufügen" appears |
| Add to Playlist shows in More menu for episodes | ATP-03 | Requires live LMS + Spotify account | Play episode → More → verify "Zu Playlist hinzufügen" appears |
| Playlist picker shows user's playlists | ATP-01 | Requires Spotify API with auth | Tap "Add to Playlist" → verify playlist list loads |
| Selecting playlist adds item | ATP-01 | Requires Spotify API mutation | Select a playlist → verify track appears in Spotify |
| Invalid playlistId rejected | ATP-02 | API validation tested via code review | addToPlaylist validates /^[A-Za-z0-9]{1,40}$/ |

---

## Sign-Off

- [x] All requirements verified via local UAT (2026-06-26)
- [x] Code review passed (1 critical fixed, 1 info)
- [ ] Automated tests not written (manual-only per user decision)

---
*Phase: 34-add-to-playlist*
*Validated: 2026-06-26*
