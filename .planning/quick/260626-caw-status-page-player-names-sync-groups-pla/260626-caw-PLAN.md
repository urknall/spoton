---
phase: quick
plan: 260626-caw
type: execute
wave: 1
depends_on: []
files_modified:
  - Plugins/SpotOn/Status.pm
  - Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html
  - Plugins/SpotOn/API/TokenManager.pm
  - Plugins/SpotOn/API/Client.pm
  - Plugins/SpotOn/Unified/Daemon.pm
  - Plugins/SpotOn/ProtocolHandler.pm
autonomous: true

must_haves:
  truths:
    - "Status page shows player name prominently with MAC as secondary text"
    - "Status page shows sync group members when player is synced"
    - "Status page shows current track title when playing a SpotOn track"
    - "Error history captures token failures, API errors, daemon errors, and browse retries"
  artifacts:
    - path: "Plugins/SpotOn/Status.pm"
      provides: "playing, currentTrack, syncGroup fields in daemon data"
      contains: "isPlaying"
    - path: "Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html"
      provides: "Enhanced daemon rendering with sync groups and playback"
      contains: "syncGroup"
    - path: "Plugins/SpotOn/API/TokenManager.pm"
      provides: "recordError calls at 4 error sites"
      contains: "recordError"
    - path: "Plugins/SpotOn/API/Client.pm"
      provides: "recordError calls at 2 error sites"
      contains: "recordError.*JSON parse"
    - path: "Plugins/SpotOn/Unified/Daemon.pm"
      provides: "recordError call for tempfile failure"
      contains: "recordError"
    - path: "Plugins/SpotOn/ProtocolHandler.pm"
      provides: "recordError call for browse 404 exhaustion"
      contains: "recordError"
  key_links:
    - from: "Plugins/SpotOn/Status.pm"
      to: "status.html"
      via: "JSON data endpoint /status/data"
      pattern: "playing.*currentTrack.*syncGroup"
---

<objective>
Enhance the SpotOn status page with player playback state, sync group visibility, and error recording across all major modules.

Purpose: Give users immediate visibility into which players are playing SpotOn tracks, which are synced together, and capture errors from TokenManager, Client, Daemon, and ProtocolHandler into the status page error history.
Output: Enhanced Status.pm data collector, richer status.html rendering, recordError calls at 8 error sites across 4 modules.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@Plugins/SpotOn/Status.pm
@Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html
@Plugins/SpotOn/API/TokenManager.pm
@Plugins/SpotOn/API/Client.pm
@Plugins/SpotOn/Unified/Daemon.pm
@Plugins/SpotOn/ProtocolHandler.pm
</context>

<tasks>

<task type="auto">
  <name>Task 1: Backend — Add playing, currentTrack, syncGroup to _collectDaemons</name>
  <files>Plugins/SpotOn/Status.pm</files>
  <action>
In _collectDaemons (lines 107-132), after resolving the $client object (line 117), add three new fields to the daemon hash pushed at line 120:

1. `playing` — boolean (JSON 1/0). Check via `$client->isPlaying` (returns true if player is actively playing). Only set to 1 if the player is playing AND the current URL is a SpotOn URL. To get the URL: `my $song = $client->playingSong(); my $url = ($song && $song->currentTrack()) ? $song->currentTrack()->url : '';` then check `$url =~ /^spoton:\/\//`.

2. `currentTrack` — string or JSON::null. When `playing` is true, get the track title via `Slim::Music::Info::getCurrentTitle($client, $url)` (this is the standard LMS way to get the display title for the current URL). If that returns empty, fall back to undef. Require `Slim::Music::Info` at top of the sub (same pattern as the existing `require Slim::Player::Client`).

3. `syncGroup` — array of player name strings, or JSON::null. Check `$client->isSynced()`. If true, get sync group members via `my @syncedClients = $client->syncedWith();` which returns a list of Client objects. Map each to `$_->name` to get display names. Include the player itself in the list for a complete group view. If not synced, set to undef.

Guard all three with `if ($client)` — if no client object resolved, set all three to their null/false defaults.

The hash entry additions (inside the existing push):
- `playing => $isPlaying ? 1 : 0,`
- `currentTrack => $currentTrack,`
- `syncGroup => $syncGroup,`
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/Status.pm 2>&1 | grep -q "syntax OK" && echo "PASS" || echo "FAIL"</automated>
  </verify>
  <done>_collectDaemons returns playing (bool), currentTrack (string/null), syncGroup (array/null) for each daemon. Perl compiles cleanly.</done>
</task>

<task type="auto">
  <name>Task 2: Frontend — Render MAC, sync groups, and playback in status.html</name>
  <files>Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html</files>
  <action>
In the renderDaemon function (lines 196-236), enhance the per-daemon rendering block (inside the for loop starting line 208):

After the existing nameRow block (lines 214-229 — the row with dot + name + alive/dead), add three new visual elements:

1. **MAC as secondary text** — Create a small muted line showing the MAC address below the player name. Use a div with class `metric-row` containing a single span with class `muted` and style `font-size: 0.85em; padding-left: 1.2em;` displaying `d.mac`. Only show this if `d.name !== d.mac` (skip when name IS the mac, to avoid redundancy).

2. **Sync group** — If `d.syncGroup` is a non-empty array, add a metric row: `createMetricRow('Sync Group', d.syncGroup.join(', '))`. Insert after the MAC line and before PID.

3. **Playback status** — If `d.playing` is truthy, add a metric row with the play indicator. Label: `'Now Playing'`. Value: if `d.currentTrack` is truthy, use the unicode play symbol plus the track title (`'▶ ' + d.currentTrack`), otherwise just `'▶ Playing'`. Use no extraClass. Insert after sync group (or after MAC line if no sync group) and before PID.

Keep the existing PID, Uptime, Connect, Stream Port rows unchanged after these new rows.
  </action>
  <verify>
    <automated>cd /home/sti/spoton && grep -c 'syncGroup\|currentTrack\|Now Playing' Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html | xargs test 3 -le && echo "PASS" || echo "FAIL"</automated>
  </verify>
  <done>Status page renders MAC as secondary text, sync group members when synced, and current track title with play indicator when playing a SpotOn track.</done>
</task>

<task type="auto">
  <name>Task 3: Add recordError calls to 8 error sites across 4 modules</name>
  <files>
    Plugins/SpotOn/API/TokenManager.pm
    Plugins/SpotOn/API/Client.pm
    Plugins/SpotOn/Unified/Daemon.pm
    Plugins/SpotOn/ProtocolHandler.pm
  </files>
  <action>
Add `Plugins::SpotOn::Status->recordError(...)` calls alongside existing `$log->error()` or `$log->warn()` calls at these 8 locations. Use the same `$INC` guard pattern already established in Client.pm line 720-722: `if ($INC{'Plugins/SpotOn/Status.pm'}) { Plugins::SpotOn::Status->recordError(...); }`. Place the recordError call immediately AFTER the existing log line.

**TokenManager.pm — 4 sites:**

1. Line 149 (token refresh failure, inside the `unless $token` block):
   `recordError('error', 'Token', "refresh failed for $id")`

2. Line 190 (binary not found for discovery):
   `recordError('error', 'Token', "binary not found for discovery")`

3. Lines 216-218 (discovery process failed to start):
   `recordError('error', 'Token', "discovery process failed to start")`

4. Line 424 (--get-token failed):
   `recordError('error', 'Token', "get-token failed for $accountId ($flavor)")`

**Client.pm — 2 sites:**

5. Line 668 (JSON parse error in success callback):
   `recordError('error', 'API', "JSON parse error for $cleanPath")`

6. Line 754 (HTTP error in error callback — the final catch-all error):
   `recordError('error', 'API', "HTTP $code for $cleanPath")`

**Daemon.pm — 1 site:**

7. Line 173 (tempfile failed):
   `recordError('error', 'Daemon', "tempfile failed: $@")`

**ProtocolHandler.pm — 1 site:**

8. Line 225 (Browse 404 retries exhausted):
   `recordError('warn', 'Browse', "404 retries exhausted for $url")`

Note: Do NOT add `use` or `require` for Status.pm in any of these files. The `$INC` guard handles the case where Status.pm is not loaded (e.g., headless LMS without web interface).
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/API/TokenManager.pm 2>&1 | grep -q "syntax OK" && perl -c Plugins/SpotOn/API/Client.pm 2>&1 | grep -q "syntax OK" && perl -c Plugins/SpotOn/Unified/Daemon.pm 2>&1 | grep -q "syntax OK" && perl -c Plugins/SpotOn/ProtocolHandler.pm 2>&1 | grep -q "syntax OK" && echo "ALL PASS" || echo "FAIL"</automated>
  </verify>
  <done>All 8 error sites have recordError calls with $INC guard. All 4 modules compile cleanly. Error history on status page captures Token, API, Daemon, and Browse errors.</done>
</task>

</tasks>

<verification>
All 6 modified files compile with `perl -c` without errors. The status page JSON data endpoint includes playing, currentTrack, and syncGroup fields in daemon objects. The frontend renders sync groups and playback state. Error recording covers TokenManager (4), Client (2), Daemon (1), ProtocolHandler (1) = 8 total sites.
</verification>

<success_criteria>
- `perl -c` passes for all 6 modified Perl files
- Status.pm _collectDaemons returns playing, currentTrack, syncGroup per daemon
- status.html renders MAC address, sync group, and now-playing indicator
- 8 recordError calls added across 4 modules with $INC guard pattern
- grep confirms: `grep -rc 'recordError' Plugins/SpotOn/` shows 9+ total occurrences (1 definition + 1 existing 429 + 8 new)
</success_criteria>

<output>
Create `.planning/quick/260626-caw-status-page-player-names-sync-groups-pla/260626-caw-SUMMARY.md` when done
</output>
