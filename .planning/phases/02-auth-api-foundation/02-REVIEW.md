---
phase: 02-auth-api-foundation
reviewed: 2026-05-27T00:00:00Z
depth: standard
files_reviewed: 12
files_reviewed_list:
  - Plugins/SpotOn/API/TokenManager.pm
  - Plugins/SpotOn/API/Client.pm
  - Plugins/SpotOn/Settings.pm
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
  - Plugins/SpotOn/strings.txt
  - librespot-spoton/src/main.rs
  - librespot-spoton/Cargo.toml
  - t/05_perl_syntax.t
  - t/07_token_manager.t
  - t/08_api_client.t
  - t/09_settings.t
findings:
  critical: 4
  warning: 5
  info: 3
  total: 12
status: issues_found
---

# Phase 02: Code Review Report

**Reviewed:** 2026-05-27
**Depth:** standard
**Files Reviewed:** 12
**Status:** issues_found

## Summary

Phase 02 implements the auth and API foundation: TokenManager (credential store + token refresh), API Client (rate-limited async HTTP egress), Settings handler, and the librespot-spoton Rust binary with `--authenticate` and `--get-token` modes.

The overall structure is sound — CSRF protection is in place, the concurrency cap logic is correct, the HTML template consistently applies `| html` escaping, and the binary-spawn alarm pattern provides at least partial protection against LMS freeze. However, four blockers prevent this code from shipping as-is:

1. Spotify account password is passed as a CLI argument and is visible in `ps` output on every authentication attempt.
2. API response cache keys omit query parameters, guaranteeing stale results when any endpoint with query parameters is called.
3. The active-account re-assignment after `removeAccount` is logically unreachable due to double-write to the same pref by `removeAccount`.
4. `URI::Escape` is called in `Client.pm` without any `use`/`require`, causing a fatal "Undefined subroutine" crash the first time a request carries query parameters.

---

## Critical Issues

### CR-01: Password Exposed in Process List During Authentication

**File:** `Plugins/SpotOn/API/TokenManager.pm:164-165`
**Issue:** The Spotify password is passed as a shell command-line argument (`--password '%s'`). On any Unix system, all users with access to `ps aux` (or `/proc/<pid>/cmdline`) can read the plaintext password for the duration of the backtick call (~10-15 seconds). This applies every time `addAccount` is called.

```perl
# Current — password visible in ps output
my $cmd = sprintf(
    "'%s' -n 'SpotOn' --username '%s' --password '%s' --authenticate --cache '%s' 2>&1",
    $safeBin, $safeUser, $safePass, $safeCache
);
```

**Fix:** Pass credentials via a pipe to the binary's stdin. Both the Perl side and the Rust binary must change together:

```perl
# Perl: open a pipe, write "username\npassword\n" to stdin
open(my $fh, '|-', $safeBin, '-n', 'SpotOn', '--authenticate',
     '--cache', $safeCache, '--credentials-stdin')
    or die "Cannot spawn binary: $!";
print $fh "$username\n$password\n";
close($fh);
```

```rust
// Rust: read credentials from stdin when --credentials-stdin flag is present
use std::io::{self, BufRead};
let stdin = io::stdin();
let mut lines = stdin.lock().lines();
let username = lines.next().ok_or("no username")??;
let password  = lines.next().ok_or("no password")??;
```

---

### CR-02: API Response Cache Key Ignores Query Parameters

**File:** `Plugins/SpotOn/API/Client.pm:77` and `Plugins/SpotOn/API/Client.pm:155`
**Issue:** The response cache key is built from `$path` alone, before query parameters are appended to the URL. Two calls that differ only in their query parameters (e.g., `search?q=Beatles` vs `search?q=Radiohead`) share the same cache key `spoton_resp_search`. The first response is cached and returned for all subsequent requests regardless of their parameters. This bug is dormant in Phase 2 (only `getMe` is implemented, and it sets `_noCache=1`), but it will produce wrong data the moment Phase 3 adds search or browse endpoints.

```perl
# Current — wrong: key omits query params
my $cacheKey = "spoton_resp_$path";
```

**Fix:** Include the sorted, serialized query parameters in the cache key:

```perl
# After building @queryParts:
my $queryStr  = join('&', sort @queryParts);
my $cacheKey  = $queryStr
    ? "spoton_resp_${path}?${queryStr}"
    : "spoton_resp_${path}";
```

Both the request-side check (line 77) and the `_onSuccess` store (line 155) must use the same key construction. The simplest fix is to build the key once in `_request` and pass it as a new `_cacheKey` entry in `$params`.

---

### CR-03: Active-Account Reassignment After removeAccount is Logically Unreachable

**File:** `Plugins/SpotOn/Settings.pm:72-80`
**Issue:** `TokenManager->removeAccount()` already clears `activeAccount` to `''` before returning (TokenManager.pm lines 239-241). Settings.pm then checks `if (($prefs->get('activeAccount') || '') eq $removeId)`, but `activeAccount` is already `''` at this point — the condition is never true. As a result, when the user removes the active account, `activeAccount` stays `''` even when other accounts remain. The user sees "No account configured" until they manually switch.

```perl
# Settings.pm lines 72-80 — condition at line 75 is dead
Plugins::SpotOn::API::TokenManager->removeAccount($removeId);
if (($prefs->get('activeAccount') || '') eq $removeId) {  # always false
    my @remaining = grep { $_ ne $removeId } keys %{$accounts};
    $prefs->set('activeAccount', @remaining ? $remaining[0] : '');
}
```

**Fix (option A):** Move the replacement logic into `removeAccount` in TokenManager so it can run before clearing the pref:

```perl
# TokenManager.pm removeAccount — before clearing activeAccount:
my $active = $prefs->get('activeAccount') || '';
if ($active eq $accountId) {
    my $accounts  = $prefs->get('accounts') || {};
    delete $accounts->{$accountId};         # remove first
    my @remaining = sort keys %{$accounts};
    $prefs->set('activeAccount', @remaining ? $remaining[0] : '');
} else {
    # not active — just remove
    my $accounts = $prefs->get('accounts') || {};
    delete $accounts->{$accountId};
    $prefs->set('accounts', $accounts);
}
```

**Fix (option B):** Pass the intent to Settings.pm by having `removeAccount` return the old active ID (or a boolean) so Settings.pm can decide. Either approach is acceptable; the code must not read `activeAccount` after `removeAccount` has already overwritten it.

---

### CR-04: URI::Escape Used Without import — Runtime Fatal on Requests with Query Parameters

**File:** `Plugins/SpotOn/API/Client.pm:116`
**Issue:** `URI::Escape::uri_escape()` is called via the fully-qualified name without any `use URI::Escape` or `require URI::Escape` at the top of the file. When the URI::Escape module has not yet been loaded by LMS (it is not guaranteed to be pre-loaded just because `URI` is listed as available), Perl raises a fatal `Undefined subroutine &URI::Escape::uri_escape`. This code path is only exercised when `$params` contains keys that don't start with `_`, which doesn't happen in Phase 2's single `getMe` call — the bug is latent but certain to fire in Phase 3.

```perl
# Current — crashes if URI::Escape not already loaded
push @queryParts, "$key=" . URI::Escape::uri_escape($params->{$key});
```

**Fix:** Add `use URI::Escape qw(uri_escape);` to the module's imports and use the imported name:

```perl
# At top of Client.pm:
use URI::Escape qw(uri_escape);

# At line 116:
push @queryParts, "$key=" . uri_escape($params->{$key});
```

---

## Warnings

### WR-01: alarm() Has No Effect on Windows — Blocking Backtick Hangs LMS Indefinitely

**File:** `Plugins/SpotOn/API/TokenManager.pm:73-79` and `169-177`
**Issue:** `alarm()` is a no-op on Windows (Perl's `d_alarm` is defined but the function returns 0 and does nothing). The CLAUDE.md target platform list includes Windows (`x86_64-pc-windows-msvc`). On Windows, both the `--get-token` (10s timeout) and `--authenticate` (15s timeout) calls can block indefinitely, freezing the entire LMS event loop including all players.

**Fix:** Guard alarm usage with a platform check, and add a Windows-specific alternative (e.g., threads or a separate watchdog mechanism):

```perl
my $has_alarm = $^O ne 'MSWin32';

local $SIG{ALRM} = sub { die "timeout\n" } if $has_alarm;
eval {
    alarm(10) if $has_alarm;
    $output = `$cmd`;
    alarm(0) if $has_alarm;
};
alarm(0) if $has_alarm;
```

For a complete fix on Windows, consider using `IPC::Open3` with a poll loop or `Proc::Background` (listed as available in CLAUDE.md).

---

### WR-02: alarm(0) Cancels Any Parent Alarm Set by LMS

**File:** `Plugins/SpotOn/API/TokenManager.pm:79` and `177`
**Issue:** The unconditional `alarm(0)` after each `eval` block cancels any pending alarm that was set by the calling code before `refreshToken`/`addAccount` was invoked. Perl's `alarm()` is process-global. If LMS or a scanner plugin has its own alarm running, these lines silently cancel it. `local $SIG{ALRM}` correctly restores the handler, but `alarm(0)` resets the countdown.

**Fix:** Capture the remaining alarm time and restore it:

```perl
my $remaining = alarm(10);       # also returns remaining time from any previous alarm
$output = `$cmd`;
alarm($remaining || 0);          # restore original alarm, or cancel if none
```

---

### WR-03: removeAccount accountId Not Validated — Potential Path Traversal

**File:** `Plugins/SpotOn/Settings.pm:72-73`, `Plugins/SpotOn/API/TokenManager.pm:222-235`
**Issue:** The `removeAccount` value comes directly from an HTTP POST parameter and is passed to `_cacheDir`, which constructs a filesystem path via `catdir`. `File::Spec::Functions::catdir` does not normalize `..` components: `catdir('/var/cache/lms', 'spoton', '../../etc')` produces `/var/cache/lms/spoton/../../etc`, which resolves to `/var/cache/etc`. A subsequent `rmtree` on that path could delete directories outside the SpotOn data directory. The LMS settings page is CSRF-protected and requires authentication, but this is still a real vulnerability for any attacker with a valid LMS session.

**Fix:** Validate that the account ID exists in `$prefs->get('accounts')` before acting on it, and/or enforce that it matches the expected hex format:

```perl
# Settings.pm — validate before calling removeAccount:
if (my $removeId = $paramRef->{removeAccount}) {
    my $accounts = $prefs->get('accounts') || {};
    if (exists $accounts->{$removeId} && $removeId =~ /\A[0-9a-f]{8}\z/) {
        Plugins::SpotOn::API::TokenManager->removeAccount($removeId);
        # ... active account logic ...
    }
}
```

---

### WR-04: switchAccount accountId Not Validated Against Known Accounts

**File:** `Plugins/SpotOn/Settings.pm:84-86`
**Issue:** The `switchAccount` POST parameter is stored directly to `$prefs->set('activeAccount', $switchId)` without checking that `$switchId` is a known account in `$prefs->get('accounts')`. An attacker with a valid LMS session can set `activeAccount` to an arbitrary string, which could confuse other code that reads this preference.

**Fix:** Check membership before setting:

```perl
if (my $switchId = $paramRef->{switchAccount}) {
    my $accounts = $prefs->get('accounts') || {};
    if (exists $accounts->{$switchId}) {
        $prefs->set('activeAccount', $switchId);
    }
}
```

---

### WR-05: Settings.pm Comment "addAccount is synchronous, no callback needed" is Wrong

**File:** `Plugins/SpotOn/Settings.pm:50`
**Issue:** The comment says "no callback needed" but `addAccount` is called with a callback that captures `$accountId` and `$err` into outer-scope variables. The code works because `addAccount` uses a blocking backtick call and invokes the callback synchronously. However, the comment misrepresents the design — if `addAccount` is ever made truly async (non-blocking), the code at lines 62-68 breaks silently with `$accountId` and `$err` both `undef`. The incorrect comment actively increases the chance of a future maintainer making that change without realizing the downstream impact.

**Fix:** Replace the comment with an accurate description:

```perl
# addAccount uses a blocking backtick spawn and invokes $cb synchronously.
# $accountId and $err are captured from the callback before the if-blocks below.
# IMPORTANT: if addAccount is ever made truly async, the code below must move
# into the callback body.
```

---

## Info

### IN-01: Dead Method _finalizeAccountDir in TokenManager.pm

**File:** `Plugins/SpotOn/API/TokenManager.pm:342-349`
**Issue:** `_finalizeAccountDir` is defined but never called. The same rename/move logic is inlined in `addAccount` (lines 199-203). The private method is unreachable dead code.

**Fix:** Remove the `_finalizeAccountDir` sub entirely, or replace the inline logic in `addAccount` with a call to it for consistency.

---

### IN-02: Stray "SON" String Key in strings.txt

**File:** `Plugins/SpotOn/strings.txt:73-75`
**Issue:** The file ends with a string block for the key `SON` with a single `EN SpotOn` translation. This appears to be a truncated or corrupt `PLUGIN_SPOTON` key. LMS will parse and register `SON` as a real string key, potentially conflicting with keys from other plugins.

**Fix:** Remove the three lines:
```
SON
	EN	SpotOn
```

---

### IN-03: Cargo.toml tokio rt-multi-thread Feature Unnecessary for Phase 2

**File:** `librespot-spoton/Cargo.toml:17`
**Issue:** The `rt-multi-thread` tokio feature pulls in the full multi-threaded runtime. Phase 2 only performs sequential async work in a single async `main`. `rt-current-thread` (or the `current_thread` attribute on `#[tokio::main]`) is sufficient and produces a smaller binary.

**Fix:**
```toml
tokio = { version = "1", features = ["rt", "macros"] }
```
and change `#[tokio::main]` to `#[tokio::main(flavor = "current_thread")]`.

---

_Reviewed: 2026-05-27_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
