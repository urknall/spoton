---
status: awaiting_human_verify
trigger: "Connect Reconnect: Nach Disconnect → Reconnect empfängt LMS kein start-Event. Kein Ton nach Reconnect, Skip ist 1 Song versetzt."
created: 2026-06-15
updated: 2026-06-15
---

## Symptoms

- **Expected:** Disconnect von Spotify Connect, dann Reconnect → Ton kommt sofort, Track-State synchron
- **Actual:** Nach Reconnect kein Ton. Beim nächsten Skip kommt Ton, aber 1 Song versetzt (LMS hat alten Track-State)
- **Error messages:** Keine Fehler im Log. Problem ist ein fehlendes Event.
- **Timeline:** Beobachtet am 2026-06-15. Früher hat Reconnect funktioniert.
- **Reproduction:** Connect → Play → Disconnect in Spotify App → Reconnect zum selben Player → kein Ton

## Evidence

- timestamp: 2026-06-15T15:38:40 — LMS DIAG: stop event, echo_suppressed (user disconnects)
- timestamp: 2026-06-15T13:38:51Z — Daemon: SessionConnected + TrackChanged(10FPUH8BG5cn6s3ETpJ9sZ) + Playing(position_ms=27978) — daemon re-established session
- timestamp: 2026-06-15T15:38:51-15:39:03 — LMS: 23 Sekunden Lücke, KEIN start/change DIAG event obwohl Daemon Events feuert
- timestamp: 2026-06-15T15:39:03 — LMS DIAG: change event (prev=10FPUH8BG5cn6s3ETpJ9sZ new=597f77kv4RdC9qXb5r9FUB) — Skip erzeugt change, aber Track-State ist 1 Song versetzt

## Current Focus

reasoning_checkpoint:
  hypothesis: "After disconnect, current_track remains Some(id) in the daemon's LMS event dispatcher. On reconnect, TrackChanged fires with the same track_id — hitting the Some(prev)==new_id branch which does nothing (no start, no change). Playing then fires but was_paused=false and needs_position_sync=false, so no LMS notification is sent. Result: LMS never receives any event and stays paused."
  confirming_evidence:
    - "Daemon log at 13:38:40: Stopped fires, current_track=Some(10FPUH8BG5cn6s3ETpJ9sZ). Stopped handler sends notify(stop) but does NOT set current_track=None."
    - "Daemon log at 13:38:51: SessionConnected → TrackChanged(10FPUH8BG5cn6s3ETpJ9sZ) → Playing(27978). TrackChanged hits Some(prev)==new_id branch — no start emitted."
    - "Playing(27978) fires with same track id. was_paused=false (reset by TrackChanged same-id branch). needs_position_sync=false (only set in None->Some branch). No LMS notification emitted."
    - "LMS DIAG confirms: 23-second gap between stop at 15:38:40 and next event (change) at 15:39:03. No start/resume ever received."
    - "The change at 15:39:03 comes from a user-initiated skip (TrackChanged to 597f77kv4RdC9qXb5r9FUB, a different id) which hits the Some(prev)!=new_id branch and does emit change — proving the wire path works."
  falsification_test: "If hypothesis is wrong, the daemon log would show a notify(start/resume) call between 13:38:51 and 13:39:03. It does not."
  fix_rationale: "Stopped must clear current_track to None (or set was_paused=true). Then on reconnect, TrackChanged hits the None branch, sets grace timer, and emits start — which triggers LMS playlist play and resumes audio. Alternatively: detect SessionConnected as a reconnect signal and reset current_track. The cleanest fix is: in the Stopped handler, set current_track=None after sending notify(stop)."
  blind_spots: "Paused event also does not clear current_track — but Paused sets was_paused=true, so next Playing fires resume. Stopped does NOT set was_paused=true (intentionally — Stopped is end-of-session). This means the Stopped handler is specifically broken for reconnect."

- next_action: "Apply fix in connect.rs: in PlayerEvent::Stopped handler, set *current_track = None after notify(stop). Build and verify."

## Evidence

- timestamp: 2026-06-15T13:38:40Z
  checked: daemon log Stopped event handler in handle_player_event
  found: "Stopped: current_track=Some(10FPUH8BG5cn6s3ETpJ9sZ)" — sends notify("stop") but NEVER sets current_track=None
  implication: current_track stays Some(same_id) across the disconnect

- timestamp: 2026-06-15T13:38:51Z
  checked: daemon log after reconnect — TrackChanged(10FPUH8BG5cn6s3ETpJ9sZ) with current_track=Some(same_id)
  found: Hits "Some(prev) if prev == new_id" branch → only resets was_paused, no start or change emitted
  implication: LMS receives nothing. The None->Some branch (which emits start) is never taken.

- timestamp: 2026-06-15T13:38:51Z
  checked: Playing(position_ms=27978) after reconnect
  found: was_paused=false (reset by same-id TrackChanged), needs_position_sync=false (only set in None branch) → no LMS notification
  implication: Fully silent — daemon processes events but sends nothing to LMS

- timestamp: 2026-06-15T15:38:40-15:39:03
  checked: LMS DIAG log
  found: 23-second gap with zero spottyconnect events between stop (15:38:40) and change (15:39:03)
  implication: Confirms no wire event reached LMS during the reconnect window

- timestamp: 2026-06-15T15:39:03
  checked: LMS DIAG — change event at 15:39:03
  found: User-initiated skip fires TrackChanged(597f77kv4RdC9qXb5r9FUB) — different id → hits Some(prev)!=new_id branch → emits change. Wire path is healthy.
  implication: The bug is specifically in same-id TrackChanged after Stopped, not in the wire transport

## Eliminated

- hypothesis: "JSON-RPC notification not reaching LMS (network/port issue)"
  evidence: Change event at 15:39:03 proves wire path works; the issue is upstream — notify() is never called
  timestamp: 2026-06-15

- hypothesis: "LMS-side guard filtering the start event (isSpotifyConnect check, grace period)"
  evidence: No start event is emitted by the daemon at all — LMS never receives anything to filter
  timestamp: 2026-06-15

## Resolution

- root_cause: "In handle_player_event, the Stopped handler sends notify('stop') but does NOT reset current_track to None. On reconnect with the same track (SessionConnected → TrackChanged(same_id)), the Some(prev)==new_id branch is taken, which only resets was_paused — no start or change event is sent to LMS. The None→Some branch (which emits start) is never reached. LMS stays paused."
- fix: "In PlayerEvent::Stopped handler in librespot-spoton/src/connect.rs: after notify('stop'), set *current_track = None. This ensures the next TrackChanged(same_id) after reconnect takes the None→Some branch, emits start, and LMS issues playlist play to resume audio."
- verification: "cargo check passes. Awaiting human test: Connect → Play → Disconnect in Spotify App → Reconnect → verify audio resumes immediately."
- files_changed:
  - librespot-spoton/src/connect.rs (PlayerEvent::Stopped handler, 3 lines added)
