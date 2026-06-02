---
slug: sync-group-audio
status: fixed
trigger: manual
created: 2026-06-02
goal: find_and_fix
tdd_mode: false
---

# Debug: Sync-Group Audio — Metadata works, no audio stream

## Symptoms
- timestamp: 2026-06-02 — Connect event `start` fires for sync group master dc:a6:32:2d:0b:66
- timestamp: 2026-06-02 — canDirectStream returns 0 for synced player (correct)
- timestamp: 2026-06-02 — ProtocolHandler::new() substitutes HTTP proxy URL
- timestamp: 2026-06-02 — Transcoding table has soc-pcm, soc-flc, soc-mp3, soc-ogg entries
- timestamp: 2026-06-02 — custom-convert.conf has `soc pcm * * / # I / -` passthrough
- timestamp: 2026-06-02 — Stream endpoint returns HTTP 200 when curled
- timestamp: 2026-06-02 — Volume and metadata work correctly
- timestamp: 2026-06-02 — NO audio reaches players — no stream_s log from LMS core
- timestamp: 2026-06-02 — No errors in log
- timestamp: 2026-06-02 — Server log shows "Invalid response code (503)" and "stream failed to open"

## Current Focus
- hypothesis: CONFIRMED — DaemonManager creates duplicate daemons for synced players, causing wrong daemon lookup
- next_action: Fix applied and verified

## Evidence
- timestamp: 2026-06-02 — initHelpers processes dc:a6:32:2d:0b:66 twice (slave + standalone duplicate), creating two daemons
- timestamp: 2026-06-02 — Spotify session on daemon 6ca1 (port 36215) but new() connects to daemon dca6 (port 34931) which has no audio
- timestamp: 2026-06-02 — Daemon log shows "503 relay already active" on second connection attempt
- timestamp: 2026-06-02 — Server log confirms: "Invalid response code (503) from remote stream" + "stream failed to open"
- timestamp: 2026-06-02 — After fix: only one daemon per sync group, sync master correctly gets startHelper

## Hypotheses
1. ~~formatOverride() returns 'son' instead of 'soc'~~ — RULED OUT: formatOverride correctly returns 'soc'
2. **Sync group daemon mismatch** — CONFIRMED: duplicate daemons created, wrong one used for stream
3. ~~LMS doesn't recognize 'soc' format~~ — RULED OUT: transcoding entry found correctly
4. ~~Playlist play URL doesn't trigger right pipeline~~ — RULED OUT: pipeline is correct
5. ~~new() returns HTTP object but LMS doesn't stream~~ — PARTIALLY: new() connects to wrong daemon (503)

## Investigation Log
- 2026-06-02: Read ProtocolHandler.pm, Connect.pm, DaemonManager.pm, Daemon.pm
- 2026-06-02: Traced Song::open() flow through LMS core (Song.pm, HTTP.pm, RemoteStream.pm)
- 2026-06-02: Analyzed daemon log 6ca1005245ea-connect.log — found relay disconnect pattern
- 2026-06-02: Found "stream failed to open" in server.log at 17:38:57 with 503 response
- 2026-06-02: Traced initHelpers iteration — dc:a6:32:2d:0b:66 processed 3 times (slave + 2x standalone)
- 2026-06-02: Root cause: LMS returns duplicate client objects for same MAC (UPnP bridge/virtual player)
- 2026-06-02: Fix applied: dedup by MAC with synced-first sort, start master daemon from slave branch
- 2026-06-02: Added sync-group-aware helperForClient fallback for robustness
- 2026-06-02: Verified: two daemons (one per group/standalone), no duplicates, 60s watchdog stable

## Resolution
- root_cause: DaemonManager.initHelpers creates duplicate Connect daemons for the same player MAC when LMS returns multiple client objects (e.g. UPnP bridge + squeezelite sharing a MAC). The Spotify session runs on one daemon but ProtocolHandler.new() connects to the other (empty) daemon, which returns 503 or serves no audio.
- fix: (1) Sort clients synced-first + dedup by MAC to prevent duplicate daemons. (2) Start sync master daemon eagerly from slave branch. (3) Add sync-group-aware fallback to helperForClient() for robustness.
- verified: true (LMS restart + 60s watchdog cycle confirms one daemon per sync group, no duplicates)
