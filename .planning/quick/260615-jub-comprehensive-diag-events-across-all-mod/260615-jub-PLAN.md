---
phase: quick-260615-jub
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - Plugins/SpotOn/Connect/Daemon.pm
  - Plugins/SpotOn/ProtocolHandler.pm
  - Plugins/SpotOn/Connect.pm
  - Plugins/SpotOn/API/TokenManager.pm
  - Plugins/SpotOn/API/Client.pm
autonomous: true
---

<objective>
Add comprehensive [DIAG] diagnostic logging to all SpotOn modules for universal remote debugging.

Purpose: Enable users to capture detailed timing and decision data across the entire plugin stack by toggling a single `diagnosticMode` pref, without touching existing log levels. When a user reports a bug, asking them to enable diagnosticMode and grep for [DIAG] gives a complete picture of daemon lifecycle, stream routing, Connect events, auth, and API calls.

Output: Five modified .pm files, each with new DIAG log points guarded by `$prefs->get('diagnosticMode')`.
</objective>

<context>
@Plugins/SpotOn/Plugin.pm (lines 43-55: prefs init with diagnosticMode default)

Existing DIAG pattern (Connect.pm _connectEvent, lines 493-494):
```perl
my $diagMode = $prefs->get('diagnosticMode');
my $diagTs = $diagMode ? sprintf('%.3f', Time::HiRes::time()) : '';
```
Then at each log point:
```perl
$log->warn("[DIAG] [$diagTs] event_name: key=value key2=value2") if $diagMode;
```

All five modules already have `my $prefs = preferences('plugin.spoton');` and `my $log = logger('plugin.spoton');` at file level. All five already `use Time::HiRes;`. No new `use` statements needed in any module.

CRITICAL: Connect.pm already has DIAG logs for start/change/stop/resume inside _connectEvent (lines 608, 652, 687, 759, 806, 829, 849). DO NOT modify those existing lines.
</context>

<tasks>

<task type="auto">
  <name>Task 1: Daemon.pm -- Daemon Lifecycle DIAG Events</name>
  <files>Plugins/SpotOn/Connect/Daemon.pm</files>
  <action>
Add DIAG logging to Daemon.pm for daemon lifecycle visibility. All guards use `$prefs->get('diagnosticMode')` (module already has `$prefs` at line 42 and `Time::HiRes` at line 14).

1. **Daemon start** (in `start()`, after line 211 where stream port is confirmed):
   Log PID, stream_port, MAC, daemon name, binary path, and whether discovery is disabled.
   Format: `$log->warn("[DIAG] daemon_start: mac=MAC pid=PID stream_port=PORT name=NAME binary=PATH discovery_disabled=0|1") if $prefs->get('diagnosticMode');`
   Place AFTER line 211 (`main::INFOLOG && $log->is_info && $log->info("SpotOn Connect daemon started..."`).

2. **Port announcement timing** (in `start()`, around lines 194-208):
   Capture `Time::HiRes::time()` BEFORE the IO::Select->can_read(5) call (before line 195). After the port is successfully read (after line 208), log the elapsed time.
   Add `my $portWaitStart = Time::HiRes::time();` before the IO::Select block (before line 194).
   Then after line 208 (`$self->_streamPort($1 + 0);`):
   `$log->warn(sprintf("[DIAG] daemon_port_announce: mac=%s port=%d wait_ms=%.0f", $self->mac, $self->_streamPort, (Time::HiRes::time() - $portWaitStart) * 1000)) if $prefs->get('diagnosticMode');`

3. **Daemon stop/crash** (in `stop()`, inside the alive branch, before `$self->_proc->die` at line 318):
   `$log->warn("[DIAG] daemon_stop: mac=" . $self->mac . " pid=" . ($self->pid || 'unknown') . " uptime=" . sprintf('%.1f', $self->uptime) . "s") if $prefs->get('diagnosticMode');`

4. **Crash-loop trigger** (in `_checkStartTimes()`, inside the `unless ($already)` block, after the existing $log->warn at line 236, before the `if ($client)` at line 243):
   `$log->warn(sprintf("[DIAG] crash_loop_disable: mac=%s crash_count=%d interval=%ds cooldown=%ds", $self->mac, MAX_FAILURES_BEFORE_DISABLE_DISCOVERY, time() - $self->_startTimes->[0], DISCOVERY_COOLDOWN_SECONDS)) if $prefs->get('diagnosticMode');`

5. **Cooldown reset** (in `_resetDiscoveryCooldown()`, after line 282 where discoveryDisabledByCrashLoop is set to 0):
   `$log->warn("[DIAG] crash_loop_reset: mac=$mac discovery_re_enabled=1") if $prefs->get('diagnosticMode');`
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/Connect/Daemon.pm 2>&1 | tail -1 && grep -c '\[DIAG\]' Plugins/SpotOn/Connect/Daemon.pm</automated>
  </verify>
  <done>Daemon.pm compiles cleanly. grep shows 5 new [DIAG] log lines (daemon_start, daemon_port_announce, daemon_stop, crash_loop_disable, crash_loop_reset).</done>
</task>

<task type="auto">
  <name>Task 2: ProtocolHandler.pm -- Stream Delivery DIAG Events</name>
  <files>Plugins/SpotOn/ProtocolHandler.pm</files>
  <action>
Add DIAG logging to ProtocolHandler.pm for stream routing decisions. Module already has `$prefs` (line 17) and `$log` (line 16). No new imports needed.

1. **canDirectStream decision** (in `canDirectStream()`, at the final return on line 142 where the DirectStream URL is returned):
   BEFORE the existing `return $ds_url;` at line 142, add:
   `$log->warn("[DIAG] canDirectStream: mac=" . $client->id . " url=$ds_url single_player=1") if $prefs->get('diagnosticMode');`

2. **canDirectStream rejection for sync group** (at line 131-134, inside the `if ($client->isSynced())` block):
   After line 131 (`if ($client->isSynced()) {`), before the existing INFOLOG:
   `$log->warn("[DIAG] canDirectStream: mac=" . $client->id . " result=0 reason=synced") if $prefs->get('diagnosticMode');`

3. **canEnhanceHTTP decision for Connect proxy** (in `canEnhanceHTTP()`, inside the stream-URL branch at line 185):
   After line 185 (`if ($url && $url =~ m{:\d+/stream\b}) {`):
   `$log->warn("[DIAG] canEnhanceHTTP: url=$url result=0 reason=connect_proxy_infinite_stream") if $prefs->get('diagnosticMode');`

4. **Sync group stream proxy** (in `new()`, at line 233-237 where the HTTP URL substitution happens):
   After line 233 (`my $httpUrl = 'http://' ...`), before the existing INFOLOG at line 234:
   `$log->warn("[DIAG] connect_sync_proxy: mac=" . $client->id . " http_url=$httpUrl port=" . $helper->_streamPort) if $prefs->get('diagnosticMode');`

5. **formatOverride decision** (in `formatOverride()`, before the final `return 'son';` at line 82):
   `$log->warn("[DIAG] formatOverride: mac=" . ($client ? $client->id : 'none') . " url=$url fmt=$fmt result=son") if $prefs->get('diagnosticMode');`
   And inside the `soc` return path at line 76 (before `return 'soc';`):
   `$log->warn("[DIAG] formatOverride: mac=" . ($client ? $client->id : 'none') . " url=$url result=soc (connect_stream_mode)") if $prefs->get('diagnosticMode');`
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/ProtocolHandler.pm 2>&1 | tail -1 && grep -c '\[DIAG\]' Plugins/SpotOn/ProtocolHandler.pm</automated>
  </verify>
  <done>ProtocolHandler.pm compiles cleanly. grep shows 6 new [DIAG] log lines (canDirectStream x2, canEnhanceHTTP, connect_sync_proxy, formatOverride x2).</done>
</task>

<task type="auto">
  <name>Task 3: Connect.pm -- Extended Events DIAG (New Points Only)</name>
  <files>Plugins/SpotOn/Connect.pm</files>
  <action>
Add NEW DIAG logging to Connect.pm for extended events. DO NOT modify the existing 7 DIAG log lines (lines 608, 652, 687, 759, 806, 829, 849). Module already has `$prefs` (line 38) and `Time::HiRes` (line 11).

1. **startOffset adjustments in _onNewSong** (at line 268, after `$song->startOffset(int($progress) - $elapsed);`):
   `$log->warn(sprintf("[DIAG] startOffset_adjust: mac=%s old=0 new=%d progress=%s elapsed=%.1f", $client->id, $song->startOffset(), $progress, $elapsed)) if $prefs->get('diagnosticMode');`

2. **Echo suppression in _onPause** (at line 315-318, inside the 1s echo suppression block, before `return;`):
   `$log->warn("[DIAG] echo_suppressed: mac=" . $client->id . " event=onPause reason=connectPauseTs_within_1s age=" . sprintf('%.3f', Time::HiRes::time() - $lastConnectPause) . "s") if $prefs->get('diagnosticMode');`

3. **Echo suppression in _onPause grace period** (at line 326-329, inside the 3s grace period block, before `return;`):
   `$log->warn("[DIAG] echo_suppressed: mac=" . $client->id . " event=onPause reason=connect_start_grace age=" . sprintf('%.3f', Time::HiRes::time() - $startTime) . "s") if $prefs->get('diagnosticMode');`

4. **Volume event from binary** (at line 535, after the existing INFOLOG `"Binary reported volume change: $volume"`):
   `$log->warn("[DIAG] volume_from_binary: mac=" . $client->id . " volume=$volume uptime=" . sprintf('%.1f', Plugins::SpotOn::Connect::DaemonManager->uptime($client->id)) . "s") if $prefs->get('diagnosticMode');`

5. **Seek event from binary** (at line 551, after the existing INFOLOG `"Binary reported seek to: $position"`):
   `$log->warn("[DIAG] seek_from_binary: mac=" . $client->id . " position=$position") if $prefs->get('diagnosticMode');`

6. **Volume debounce fire** (in `_bufferedSetVolume`, at line 393, after existing INFOLOG):
   `$log->warn("[DIAG] volume_to_binary: mac=" . $client->id . " volume=$volume debounced=" . VOLUME_DEBOUNCE . "s") if $prefs->get('diagnosticMode');`

7. **Seek debounce fire** (in `_bufferedSeek`, at line 425, after existing INFOLOG):
   `$log->warn("[DIAG] seek_to_binary: mac=" . $client->id . " position_ms=$positionMs debounced=" . SEEK_DEBOUNCE . "s") if $prefs->get('diagnosticMode');`

8. **Metadata fetch start** (in `_fetchTrackMetadata`, at line 895, after `return unless $trackId;`):
   `$log->warn("[DIAG] metadata_fetch: mac=" . $client->id . " track=$trackId account=$accountId") if $prefs->get('diagnosticMode');`
   (Note: $accountId is set a few lines later at line 899. Move this DIAG line to after line 901, after accountId is resolved.)

9. **Metadata fetch success** (in `_fetchTrackMetadata` callback, at line 989, after existing INFOLOG `"Track metadata updated..."`):
   `$log->warn("[DIAG] metadata_success: mac=" . $client->id . " track=$trackId title=$title duration=$duration") if $prefs->get('diagnosticMode');`

10. **Metadata fetch failure / stale API** (at line 912-916, inside the stale API protection block, after `return;`):
    Move the DIAG line to BEFORE `return;`:
    `$log->warn("[DIAG] metadata_stale: mac=" . $client->id . " event_uri=$eventUri api_uri=" . ($trackInfo->{uri} || 'none')) if $prefs->get('diagnosticMode');`
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/Connect.pm 2>&1 | tail -1 && grep -c '\[DIAG\]' Plugins/SpotOn/Connect.pm</automated>
  </verify>
  <done>Connect.pm compiles cleanly. grep shows 17 total [DIAG] lines (7 existing + 10 new). The existing 7 lines for start/change/stop/resume are untouched.</done>
</task>

<task type="auto">
  <name>Task 4: Connect.pm -- Control Commands DIAG</name>
  <files>Plugins/SpotOn/Connect.pm</files>
  <action>
Add DIAG logging to Connect.pm for control command visibility. This is a separate task for the _sendControlCommand and _sendControlFallback functions.

1. **Control command sent** (in `_sendControlCommand`, at line 184, alongside the existing INFOLOG `"_sendControlCommand: POST $url ($jsonBody)"`):
   `$log->warn("[DIAG] control_cmd_sent: mac=" . $client->id . " endpoint=$endpoint body=$jsonBody") if $prefs->get('diagnosticMode');`

2. **Control command success** (in `_sendControlCommand` success callback, at line 191):
   `$log->warn("[DIAG] control_cmd_ok: mac=" . $client->id . " endpoint=$endpoint") if $prefs->get('diagnosticMode');`

3. **Control command failure + fallback trigger** (in `_sendControlCommand` error callback, at line 194-196, alongside existing INFOLOG):
   `$log->warn("[DIAG] control_cmd_fail: mac=" . $client->id . " endpoint=$endpoint error=$error fallback=web_api") if $prefs->get('diagnosticMode');`

4. **Web API fallback triggered** (in `_sendControlFallback`, at the top of the function after line 209):
   `$log->warn("[DIAG] web_api_fallback: mac=" . $client->id . " endpoint=$endpoint account=" . substr($accountId, 0, 4) . "****") if $prefs->get('diagnosticMode');`
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/Connect.pm 2>&1 | tail -1 && grep -c 'control_cmd\|web_api_fallback' Plugins/SpotOn/Connect.pm</automated>
  </verify>
  <done>Connect.pm compiles cleanly. grep shows 4 new control-command DIAG lines (control_cmd_sent, control_cmd_ok, control_cmd_fail, web_api_fallback).</done>
</task>

<task type="auto">
  <name>Task 5: TokenManager.pm -- Authentication DIAG Events</name>
  <files>Plugins/SpotOn/API/TokenManager.pm</files>
  <action>
Add DIAG logging to TokenManager.pm for auth lifecycle visibility. Module already has `$prefs` (line 32) and `Time::HiRes` (line 15). NEVER log token values -- always redact account IDs to first 4 chars + ****.

1. **Token refresh success** (in `_fetchKeymasterToken`, inside the timer callback, after `$class->_cacheToken(...)` at line 414):
   `$log->warn("[DIAG] token_refresh_ok: account=" . substr($accountId, 0, 4) . "**** flavor=$flavor ttl=" . ($result->{expiresIn} || 'unknown') . "s") if $prefs->get('diagnosticMode');`

2. **Token refresh failure** (in `_fetchKeymasterToken`, at line 401 where exit != 0):
   Inside the `if ($exit != 0 || !$output)` block, before `$cb->(undef);`:
   `$log->warn("[DIAG] token_refresh_fail: account=" . substr($accountId, 0, 4) . "**** flavor=$flavor exit=$exit") if $prefs->get('diagnosticMode');`

3. **Token JSON parse failure** (at line 407, inside `if ($@ || !$result->{accessToken})`):
   Before `$cb->(undef);`:
   `$log->warn("[DIAG] token_parse_fail: account=" . substr($accountId, 0, 4) . "**** flavor=$flavor") if $prefs->get('diagnosticMode');`

4. **Discovery start** (in `startDiscovery()`, after line 209 where PID is confirmed):
   `$log->warn("[DIAG] discovery_start: pid=" . $discoveryProc->pid() . " device_name=$deviceName cache=$discoverDir") if $prefs->get('diagnosticMode');`

5. **Discovery credential received** (in `_setupAccountFromCredentials`, after line 310 where accountId is derived):
   `$log->warn("[DIAG] discovery_credential: account=" . substr($accountId, 0, 4) . "**** spotify_user=" . substr($spotifyUserId, 0, 4) . "****") if $prefs->get('diagnosticMode');`

6. **Account created** (in `_storeAccountPrefs`, after line 478 where activeAccount is set, alongside existing INFOLOG at line 481):
   `$log->warn("[DIAG] account_stored: account=" . substr($accountId, 0, 4) . "**** display_name=$displayName is_active=" . (($prefs->get('activeAccount') || '') eq $accountId ? 1 : 0)) if $prefs->get('diagnosticMode');`
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/API/TokenManager.pm 2>&1 | tail -1 && grep -c '\[DIAG\]' Plugins/SpotOn/API/TokenManager.pm</automated>
  </verify>
  <done>TokenManager.pm compiles cleanly. grep shows 6 new [DIAG] lines (token_refresh_ok, token_refresh_fail, token_parse_fail, discovery_start, discovery_credential, account_stored).</done>
</task>

<task type="auto">
  <name>Task 6: Client.pm -- API and Rate Limiting DIAG Events</name>
  <files>Plugins/SpotOn/API/Client.pm</files>
  <action>
Add DIAG logging to Client.pm for API request visibility and rate limit debugging. Module already has `$prefs` (line 35) and `Time::HiRes` (line 19).

1. **429 hit** (in `_doFlavouredRequest` error callback, inside the 429 block around line 631-644, after the existing `$log->warn` at line 644):
   `$log->warn("[DIAG] api_429: endpoint=$cleanPath flavor=$flavor retry_after=${retryAfter}s") if $prefs->get('diagnosticMode');`

2. **Request failure (non-429, non-401)** (at line 672, alongside the existing `$log->error`):
   `$log->warn("[DIAG] api_error: endpoint=$cleanPath flavor=$flavor code=$code error=$error") if $prefs->get('diagnosticMode');`

3. **Slow response detection** (in the success callback around line 583-617). Add a timing mechanism:
   BEFORE the `my $http = Slim::Networking::SimpleAsyncHTTP->new(` at line 581, add:
   `my $reqStartTime = Time::HiRes::time();`
   Then in the success callback, after `my $http = shift;` at line 583:
   ```
   my $reqDuration = Time::HiRes::time() - $reqStartTime;
   if ($reqDuration > 2 && $prefs->get('diagnosticMode')) {
       $log->warn(sprintf("[DIAG] api_slow: endpoint=%s flavor=%s duration=%.1fs", $cleanPath, $flavor, $reqDuration));
   }
   ```
   Note: the slow-response DIAG only fires when BOTH conditions are true (duration > 2s AND diagnosticMode on), to avoid unnecessary time computation.

4. **401 unauthorized** (in the 401 block at line 651-655, after the existing `$log->warn`):
   `$log->warn("[DIAG] api_401: endpoint=$cleanPath flavor=$flavor account=" . substr($accountId || '', 0, 4) . "****") if $prefs->get('diagnosticMode');`

5. **Bundled fallback triggered** (at line 664-668, alongside existing INFOLOG for 403/410 retry):
   `$log->warn("[DIAG] api_bundled_fallback: endpoint=$cleanPath trigger_code=$code") if $prefs->get('diagnosticMode');`
  </action>
  <verify>
    <automated>cd /home/sti/spoton && perl -c Plugins/SpotOn/API/Client.pm 2>&1 | tail -1 && grep -c '\[DIAG\]' Plugins/SpotOn/API/Client.pm</automated>
  </verify>
  <done>Client.pm compiles cleanly. grep shows 5 new [DIAG] lines (api_429, api_error, api_slow, api_401, api_bundled_fallback).</done>
</task>

</tasks>

<verification>
After all 6 tasks:
1. All five .pm files compile: `perl -c Plugins/SpotOn/Connect/Daemon.pm && perl -c Plugins/SpotOn/ProtocolHandler.pm && perl -c Plugins/SpotOn/Connect.pm && perl -c Plugins/SpotOn/API/TokenManager.pm && perl -c Plugins/SpotOn/API/Client.pm`
2. Total new DIAG count: `grep -rn '\[DIAG\]' Plugins/SpotOn/ --include='*.pm' | wc -l` should show 7 (existing) + 31 (new) = 38 total
3. No DIAG line logs raw tokens: `grep -n '\[DIAG\].*accessToken\|Bearer' Plugins/SpotOn/ -r` should return 0 matches
4. All DIAG lines are guarded: every `[DIAG]` line ends with `if $prefs->get('diagnosticMode');`
</verification>

<success_criteria>
- 31 new [DIAG] log points across 5 modules (5 Daemon + 6 ProtocolHandler + 14 Connect + 6 TokenManager + 5 Client = 36... accounting overlap from shared Connect tasks: 10 + 4 = 14 Connect, total = 5+6+14+6+5 = 36 new, but some may consolidate)
- All guarded by `$prefs->get('diagnosticMode')`
- All use `$log->warn` level with `[DIAG]` prefix
- No credentials/tokens logged (redacted to first 4 chars + ****)
- Existing 7 DIAG lines in Connect.pm _connectEvent remain untouched
- All 5 .pm files compile cleanly with `perl -c`
</success_criteria>
