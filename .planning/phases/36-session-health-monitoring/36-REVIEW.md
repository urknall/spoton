---
phase: 36-session-health-monitoring
reviewed: 2026-06-30T14:00:00Z
depth: standard
files_reviewed: 5
files_reviewed_list:
  - librespot-spoton/src/unified.rs
  - Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html
  - Plugins/SpotOn/Status.pm
  - Plugins/SpotOn/Unified/DaemonManager.pm
  - Plugins/SpotOn/Unified/Daemon.pm
findings:
  critical: 1
  warning: 5
  info: 3
  total: 9
status: issues_found
---

# Phase 36: Code Review Report

**Reviewed:** 2026-06-30T14:00:00Z
**Depth:** standard
**Files Reviewed:** 5
**Status:** issues_found

## Summary

Phase 36 adds a `/health` JSON endpoint to the Rust daemon (session validity, age, idle time), a Perl-side health-aware poll that acts on the response, a new `Status.pm` module with a status page and data API, and a status UI in `status.html`. The architecture is sound and the cross-language data flow is correct. One blocker was found in `Status.pm` (exception-unsafe data handler) plus five warnings, mostly around health data staleness, a counter-reset race in Rust, and missing crash-loop guardrails in the Perl health restart path.

---

## Critical Issues

### CR-01: `_statusDataHandler` has no exception guard — crashes silently when modules are broken

**File:** `Plugins/SpotOn/Status.pm:77-101`
**Issue:** `_statusDataHandler` calls five sub-collectors (`_collectDaemons`, `API::Client->statusSnapshot`, `_errorHistory`, `_collectTokens`, `_systemInfo`) without any `eval` wrapper. Each of these `require`s and calls other modules. If any call throws — for example, because a required module has not yet been loaded during LMS startup, or because an API method changes in a future LMS version — the handler dies and LMS returns an HTTP 500 error (or a broken response), preventing `_jsonResponse` from ever running. The JavaScript catches the non-JSON body in the `JSON.parse` catch block and shows "Status unavailable — retrying." This failure mode is self-defeating: the diagnostic status page is most likely to fail precisely when the system is in a broken state and you need it most.

**Fix:**
```perl
sub _statusDataHandler {
    my ($httpClient, $response) = @_;

    my %data;
    $data{daemons} = eval { _collectDaemons() } // [];
    if ($@) {
        main::INFOLOG && $log->is_info && $log->info("Status: _collectDaemons failed: $@");
    }

    require Plugins::SpotOn::API::Client;
    $data{api} = eval { Plugins::SpotOn::API::Client->statusSnapshot() } // {};

    $data{errors}  = eval { _errorHistory() }    // [];
    $data{tokens}  = eval { _collectTokens() }   // {};
    $data{system}  = eval { _systemInfo() }      // {};

    _jsonResponse($httpClient, $response, \%data);
}
```
This ensures partial data is always returned as valid JSON and the page degrades gracefully rather than going blank.

---

## Warnings

### WR-01: Browse fail counter incorrectly reset when `serve_track_request` returns 404 after 500ms timeout

**File:** `librespot-spoton/src/unified.rs:631-660`
**Issue:** The early-status channel waits at most 500ms for a `NOT_FOUND` signal from the spawned `serve_track_request` task (line 631-634). If `serve_track_request` takes longer than 500ms to return `StatusCode::NOT_FOUND` (e.g., a slow Spotify API response on a degraded connection), the timeout expires, `early_status` is `Err(_)`, and code falls through to line 658: `consecutive_browse_fails.store(0, Ordering::SeqCst)`. This incorrectly resets the failure counter and returns HTTP 200 with an empty audio body to LMS. Two consequences: (1) LMS sees HTTP 200 with zero bytes of audio; (2) the consecutive-failure reconnect trigger (threshold 2) is silently reset, potentially delaying a badly-needed session reconnect by multiple additional track attempts.

**Fix:** Only reset the counter when the status channel positively confirms success (i.e., the status was received and was not `NOT_FOUND`). Move the counter reset inside a guard that checks whether the status was explicitly received and non-error:

```rust
// Track started streaming — reset consecutive failure counter ONLY if
// the status channel confirmed the track actually loaded (not a slow 404).
let early_was_success = matches!(early_status, Ok(Ok(s)) if s != StatusCode::NOT_FOUND);
let timed_out         = early_status.is_err();
if early_was_success || timed_out {
    // For the timeout case we optimistically reset; the spawned task's
    // eventual drop of pcm_tx will deliver EOF if it was actually a 404.
    consecutive_browse_fails.store(0, Ordering::SeqCst);
}
```
Alternatively, widen the early-status window from 500ms to 1500ms to cover realistic slow Spotify responses, which eliminates the race for all but extreme outliers.

---

### WR-02: `_restartForHealth` bypasses crash-loop detection by always creating a fresh Daemon object

**File:** `Plugins/SpotOn/Unified/DaemonManager.pm:401-413`
**Issue:** `_restartForHealth` calls `stopHelper` followed by `startHelper`. `stopHelper` calls `delete $helperInstances{$clientId}`, which removes the `Daemon` object from the hash. `startHelper` then creates a fresh `Daemon->new(...)` with `_startTimes([])`. Because crash-loop detection in `Daemon::_checkStartTimes` accumulates on `_startTimes`, a new object resets the counter to zero. A daemon whose session is permanently dead (e.g., revoked credentials, broken token cache) will be restarted at the health-check rate (~60 seconds) indefinitely, because each restart clears the crash-loop history before the next one. In contrast, process-crash restarts via `_streamAlivePoll` call `$helper->start` on the same object and DO accumulate correctly.

**Fix:** Either (a) add an explicit health-restart rate limit in `_restartForHealth` (e.g., a counter stored on the helper object that is checked before restarting), or (b) pass the existing `_startTimes` ref from the old daemon to the new one on health restarts:

```perl
sub _restartForHealth {
    my ($class, $helper, $reason) = @_;
    return unless $helper && $helper->alive;

    # Rate-limit health restarts: no more than 1 per 5 minutes
    my $now = time();
    my $last = $helper->_lastHealthRestart // 0;
    if ($now - $last < 300) {
        main::INFOLOG && $log->is_info && $log->info(
            sprintf("Health restart suppressed for %s (last was %ds ago): %s",
                    $helper->mac, $now - $last, $reason)
        );
        return;
    }
    $helper->_lastHealthRestart($now);

    main::INFOLOG && $log->is_info && $log->info(
        sprintf("Health check restart for %s: %s", $helper->mac, $reason)
    );
    $helper->_healthCheckCount(0);
    $class->stopHelper($helper->mac);
    $class->startHelper($helper->mac);
}
```

---

### WR-03: JSON parse errors in `_onHealthResponse` silently discarded; displays "invalid" session on parse failure

**File:** `Plugins/SpotOn/Unified/DaemonManager.pm:359-367`
**Issue:** `my $json = eval { from_json($http->content) }` silently discards `$@` on failure. When `$json` is `undef` (parse error), `_lastHealthSession` is stored with `session_valid => undef`, `session_age_secs => undef`, `idle_secs => undef`. When the Status UI renders this, `null` is falsy in JavaScript, so `sessionValid ? 'valid' : 'invalid'` displays "invalid" — even though the session may be perfectly fine and only the JSON response was malformed. Additionally, `_restartForHealth` is then triggered for "malformed health response" with no logged explanation of what the body actually contained.

**Fix:** Log the parse error and the raw response body before acting on it:

```perl
my $raw  = $http->content // '';
my $json = eval { from_json($raw) };
if ($@) {
    $log->warn("Health check JSON parse error for " . $helper->mac
               . ": $@ (body: " . substr($raw, 0, 200) . ")");
}
```

This preserves the diagnostic context and prevents a transient serialization hiccup from silently restarting a healthy daemon.

---

### WR-04: XHR in `status.html` has no timeout; concurrent requests pile up when LMS is slow

**File:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/status.html:373-399`
**Issue:** `var xhr = new XMLHttpRequest()` is created in `poll()` without setting `xhr.timeout`. `poll()` is called via `setInterval(poll, 5000)`, which fires unconditionally every 5 seconds regardless of whether the previous request completed. If LMS is slow or hung, pending XHR objects accumulate indefinitely. Each pending XHR holds a connection to LMS and its associated event callbacks. When LMS recovers, all piled-up callbacks fire simultaneously.

**Fix:** Set a request timeout and switch to `setTimeout`-based chaining so the next poll only schedules after the current one completes:

```js
function poll() {
    var xhr = new XMLHttpRequest();
    xhr.timeout = 4000; // 4s — shorter than the 5s poll interval
    xhr.open('GET', '/plugins/SpotOn/status/data', true);
    xhr.ontimeout = function() {
        document.getElementById('error-banner').style.display = 'block';
        pollTimer = setTimeout(poll, 5000);
    };
    xhr.onload = function() {
        // ... existing handler ...
        pollTimer = setTimeout(poll, 5000);
    };
    xhr.onerror = function() {
        document.getElementById('error-banner').style.display = 'block';
        pollTimer = setTimeout(poll, 5000);
    };
    xhr.send();
}

function startPolling() {
    clearTimeout(pollTimer);
    poll();
}
function stopPolling() {
    clearTimeout(pollTimer);
    document.getElementById('poll-indicator').style.display = 'none';
}
```

---

### WR-05: `_onHealthError` does not update `_lastHealthSession`; Status UI shows indefinitely stale data when health polling fails

**File:** `Plugins/SpotOn/Unified/DaemonManager.pm:391-398`
**Issue:** When the HTTP GET to `/health` fails (connection refused, timeout, any HTTP error), `_onHealthError` logs a warning but does not update `_lastHealthSession`. The Status UI then continues to display the last successful health snapshot indefinitely, with no indication that health data is stale. A user watching the status page during a network connectivity issue between Perl and the daemon would see plausible but outdated session metrics, potentially masking the underlying problem.

**Fix:** Update `_lastHealthSession` in the error path to mark data as unavailable:

```perl
sub _onHealthError {
    my ($class, $helper, $http) = @_;
    $log->warn("Health check failed for " . $helper->mac . ": " . ($http->error || 'unknown'));

    # Mark last health data as unavailable so Status UI doesn't show stale metrics
    $helper->_lastHealthSession({
        session_valid    => undef,
        session_age_secs => undef,
        idle_secs        => undef,
        checked_at       => time(),
        error            => $http->error || 'connection failed',
    });
}
```
The JavaScript already guards on `if (d.sessionHealth)`, so passing the hash (which is truthy) with `session_valid: null` will correctly display "invalid" (which is accurate — the health endpoint is unreachable). Alternatively, pass `undef` from the error path to suppress the sessionHealth section entirely and show "—" for the daemon's session metrics.

---

## Info

### IN-01: Port-announcement log message says "timeout" when daemon process died prematurely

**File:** `Plugins/SpotOn/Unified/Daemon.pm:269-273`
**Issue:** When the daemon process exits before announcing its stream port, the polling loop exits early via `last unless $self->_proc && $self->_proc->alive` (line 258), leaving `$port_line` as `undef`. Line 270 then logs `"did not announce HTTP stream port (timeout)"`. The message says "timeout" in all undef cases, including immediate process death, making crash diagnosis harder.

**Fix:**
```perl
my $reason = !defined($port_line)
    ? ($self->_proc && !$self->_proc->alive ? 'process exited' : 'timeout')
    : "unexpected output: $port_line";
```

---

### IN-02: `Slim::Player::Playlist::url` called without explicit `require`

**File:** `Plugins/SpotOn/Status.pm:127`
**Issue:** `Slim::Player::Playlist::url($client)` is called without a preceding `require Slim::Player::Playlist`. This works at runtime because LMS loads this module universally, but it is fragile by convention and would fail silently (returning undef) in any test harness that loads only SpotOn modules. Adding `require Slim::Player::Playlist;` at the top of `_collectDaemons` makes the dependency explicit and matches the pattern used for every other external module in the file.

---

### IN-03: `session_created_at` initialized before `session.connect()`, inflating `session_age_secs`

**File:** `librespot-spoton/src/unified.rs:866-909`
**Issue:** `session_created_at` is initialized at line 908-909 (`Instant::now()`) before `session.connect()` is awaited (line 867-869 for Browse-only mode) or before `Spirc::new()` completes (line 1076-1083 for Connect mode). Session setup can take several hundred milliseconds. As a result, `session_age_secs` in the `/health` response includes pre-connection setup time, slightly overstating the session's age. For the 4-hour staleness threshold this is negligible, but it is an accuracy gap worth noting if the threshold is ever tightened.

**Fix:** Move the `Instant::now()` initialization for `session_created_at` to immediately after the `session.connect()` await returns `Ok(())` (Browse mode) and after `Spirc::new()` returns `Ok(...)` (Connect mode). The reconnect paths already do this correctly.

---

_Reviewed: 2026-06-30T14:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
