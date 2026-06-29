---
phase: quick
plan: 260629-gwy
type: execute
wave: 1
depends_on: []
files_modified:
  - Plugins/SpotOn/Plugin.pm
autonomous: true
requirements: []
must_haves:
  truths:
    - "Pause during HTTP stream setup is re-applied after the newsong event fires"
    - "Pause during track transitions is not swallowed by the next track starting"
    - "Normal pause/resume without race conditions works exactly as before"
    - "Explicit resume clears any stale pause-pending state"
  artifacts:
    - path: "Plugins/SpotOn/Plugin.pm"
      provides: "Pause guard in _onModeChange and _onNewSongWatchdog"
      contains: "%_pausePending"
  key_links:
    - from: "_onModeChange"
      to: "_onNewSongWatchdog"
      via: "%_pausePending hash keyed by client ID"
      pattern: "_pausePending"
---

<objective>
Fix pause commands being swallowed during HTTP stream setup and track transitions.

Purpose: When LMS resumes playback or transitions tracks, it opens a new HTTP stream to the SpotOn daemon. This triggers a `newsong` event that puts the player back into `play` mode, overriding any pause command that arrived during the ~0.5s stream setup window. The fix adds a per-client "pause guard" that detects this race condition and re-applies the pause after the stream settles.

Output: Modified Plugin.pm with pause guard mechanism in the existing watchdog/mode-change handlers.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@Plugins/SpotOn/Plugin.pm
</context>

<tasks>

<task type="auto">
  <name>Task 1: Implement pause guard in Plugin.pm watchdog handlers</name>
  <files>Plugins/SpotOn/Plugin.pm</files>
  <action>
Add a package-level hash `%_pausePending` near the existing `%_watchdogTriggerUrl` declaration (around line 2747).

Modify `_onModeChange` (line 2794) to do more than just DIAG logging. The handler already subscribes to `[['playlist'], ['pause']]` events. Add this logic BEFORE the existing DIAG logging:

1. Get the client mode via `Slim::Player::Source::playmode($client)`. The subscription fires on pause events, but verify the mode is actually `pause` (not `play`).
2. When mode is `pause` and the URL matches `spoton://` (non-connect): set `$_pausePending{$id} = Time::HiRes::time()`.
3. When mode is `play` and the URL matches `spoton://` (non-connect): delete `$_pausePending{$id}` -- explicit resume clears pending state.

Remove the `return unless $prefs->get('diagnosticMode')` guard at the top of `_onModeChange` -- the pause guard logic must run regardless of DIAG mode. Keep the existing DIAG log line but keep it gated behind diagnosticMode. Add a DIAG log line when setting or clearing the pause-pending flag.

Modify `_onNewSongWatchdog` (line 2749) to check for a pending pause AFTER the existing newsong processing. Add this check at two points:

A. In the main path (after the `$log->warn("[DIAG] Watchdog: newsong...")` line around 2783, before the duration guard and timer setup): call a new helper `_checkPauseGuard($client)`.

B. In the deferred path (inside the 0.5s timer callback, after the deferred DIAG log around line 2771): also call `_checkPauseGuard($client)`.

Create the helper sub `_checkPauseGuard`:
- Takes `$client` as argument.
- Gets `$id = $client->id`.
- Returns immediately if `$_pausePending{$id}` is not set.
- Checks if the pending pause is recent: `Time::HiRes::time() - $_pausePending{$id} < 2.0` (2-second window).
- If recent: log a DIAG warning "[DIAG] PauseGuard: re-applying pause swallowed during stream setup", delete `$_pausePending{$id}`, and schedule the pause re-application via `Slim::Utils::Timers::setTimer($client, Time::HiRes::time() + 0.3, \&_reapplyPause)`. The 0.3s delay lets the stream fully initialize before re-pausing.
- If stale (older than 2s): just delete `$_pausePending{$id}` silently.

Create the callback sub `_reapplyPause`:
- Takes `$client` as argument.
- Returns unless `$client`.
- Gets current mode via `Slim::Player::Source::playmode($client)`.
- Only re-applies if mode is still `play` (user may have done something else in the 0.3s).
- Execute: `$client->execute(['pause', '1'])`.
- Log: `$log->warn("[DIAG] PauseGuard: pause re-applied for " . $client->id)` if diagnosticMode.

Also in `shutdownPlugin`: add `%_pausePending = ()` alongside the existing unsubscribe calls for hygiene.

Important: Do NOT change the subscription filters. `_onModeChange` already subscribes to `[['playlist'], ['pause']]` which fires for pause events. The `playmode()` check inside the handler distinguishes the actual state.

Important: Do NOT touch `_onNewSongWatchdog`'s existing prefetch watchdog logic (the timer setup for `_prefetchWatchdog`). The pause guard check is an independent concern that runs alongside it.

Important: Do NOT modify ProtocolHandler.pm or any other file. This is a Plugin.pm-only fix.
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/Plugin.pm 2>&amp;1 | grep -q 'syntax OK' && echo "PASS: syntax OK" || echo "FAIL: syntax error"</automated>
    <automated>cd /home/sti/spoton && grep -c '_pausePending' Plugins/SpotOn/Plugin.pm | xargs -I{} test {} -ge 5 && echo "PASS: pausePending references found" || echo "FAIL: insufficient pausePending references"</automated>
    <automated>cd /home/sti/spoton && grep -c '_reapplyPause\|_checkPauseGuard' Plugins/SpotOn/Plugin.pm | xargs -I{} test {} -ge 3 && echo "PASS: helper subs found" || echo "FAIL: missing helper subs"</automated>
  </verify>
  <done>
    - %_pausePending hash declared at package level
    - _onModeChange sets/clears pause-pending flag based on actual playmode (no longer DIAG-gated)
    - _onNewSongWatchdog calls _checkPauseGuard in both main and deferred paths
    - _checkPauseGuard helper detects recent pause within 2s window
    - _reapplyPause callback re-issues pause command with 0.3s delay
    - All new code paths have DIAG logging
    - shutdownPlugin clears %_pausePending
    - perl -c passes with no syntax errors
  </done>
</task>

<task type="checkpoint:human-verify" gate="blocking">
  <what-built>Pause guard mechanism that detects and re-applies pause commands swallowed during HTTP stream setup and track transitions.</what-built>
  <how-to-verify>
    1. Deploy to local dev LMS: copy Plugin.pm to plugin directory, restart LMS
    2. Enable diagnostic mode in SpotOn settings
    3. Test 1 -- Resume race condition:
       - Start SpotOn playback of any track
       - Let it play for 10+ seconds
       - Pause, wait 1s, resume, then immediately pause again (within 0.5s)
       - Expected: player stays paused (not overridden by stream setup)
       - Check LMS log for "[DIAG] PauseGuard: re-applying pause" messages
    4. Test 2 -- Track transition race:
       - Start playback, seek to near end of track (last 5 seconds)
       - As track transitions, hit pause during the transition window
       - Expected: player pauses on the new track (not swallowed)
    5. Test 3 -- Normal operation:
       - Normal pause/resume cycles with 2+ seconds between actions
       - Expected: no PauseGuard messages, normal behavior unchanged
    6. Check LMS logs: grep PauseGuard /path/to/server.log
  </how-to-verify>
  <resume-signal>Type "approved" or describe issues observed during testing</resume-signal>
</task>

</tasks>

<verification>
- perl -c Plugins/SpotOn/Plugin.pm passes
- grep confirms _pausePending, _checkPauseGuard, _reapplyPause all present
- No changes to ProtocolHandler.pm or any other file
- Existing prefetch watchdog logic unchanged
- All new paths gated behind DIAG logging
</verification>

<success_criteria>
Pause commands issued during HTTP stream setup or track transitions are detected via the %_pausePending flag and re-applied after a 0.3s delay, preventing the stream completion from overriding user intent. Normal pause/resume without race conditions is unaffected.
</success_criteria>

<output>
Create `.planning/quick/260629-gwy-fix-pause-swallowed-during-http-stream-s/260629-gwy-SUMMARY.md` when done
</output>
