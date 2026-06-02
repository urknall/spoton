---
phase: 02-auth-api-foundation
fixed_at: 2026-05-27T17:35:00Z
review_path: .planning/phases/02-auth-api-foundation/02-REVIEW.md
iteration: 1
findings_in_scope: 9
fixed: 7
skipped: 2
status: partial
---

# Phase 02: Code Review Fix Report

**Fixed at:** 2026-05-27
**Source review:** .planning/phases/02-auth-api-foundation/02-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 9 (4 Critical + 5 Warning)
- Fixed: 7
- Skipped: 2 (CR-01 deferred by instruction, WR-01 skipped by instruction)

## Fixed Issues

### CR-02: API Response Cache Key Ignores Query Parameters

**Files modified:** `Plugins/SpotOn/API/Client.pm`
**Commit:** b5c2fed
**Applied fix:** Cache key is now built from path plus sorted query string
(`spoton_resp_${path}?${queryStr}` when query params are present). The key is
computed once inside the token callback (after `@queryParts` is built) and
stored in `$params->{_cacheKey}`. Both the early-return cache check (Step 2)
and the `_onSuccess` store use the same key. An early cache hit after query
params are known also correctly decrements `$inflightCount` before returning.

### CR-04: URI::Escape Used Without Import — Runtime Fatal on Requests with Query Parameters

**Files modified:** `Plugins/SpotOn/API/Client.pm`
**Commit:** b5c2fed (same commit as CR-02)
**Applied fix:** Added `use URI::Escape qw(uri_escape);` to the module imports
and replaced the fully-qualified `URI::Escape::uri_escape()` call with the
imported `uri_escape()` function name.

### CR-03: Active-Account Reassignment After removeAccount is Logically Unreachable

**Files modified:** `Plugins/SpotOn/Settings.pm`
**Commit:** ffa246f
**Applied fix:** The replacement-account selection logic now runs BEFORE
`removeAccount` is called. The candidate `$newActive` is determined from the
accounts pref while it still contains all accounts, then `removeAccount` is
called, and finally `activeAccount` is set to `$newActive` (overwriting the
`''` that `removeAccount` left behind). Uses `sort` on remaining keys for
deterministic selection.

### WR-02: alarm(0) Cancels Any Parent Alarm Set by LMS

**Files modified:** `Plugins/SpotOn/API/TokenManager.pm`
**Commit:** e56e22f
**Applied fix:** Both `refreshToken` and `addAccount` now capture the return
value of `alarm(N)` (which is the remaining time of any prior alarm) into
`$parentAlarm`. The restoring call `alarm($parentAlarm || 0)` appears in both
the normal path (end of eval) and the exception path (after the eval block),
ensuring the parent alarm is always restored regardless of whether the
backtick call succeeded or timed out.

### WR-03: removeAccount accountId Not Validated — Potential Path Traversal

**Files modified:** `Plugins/SpotOn/Settings.pm`
**Commit:** 2944f35
**Applied fix:** Added a guard that checks both `exists $accounts->{$removeId}`
and `$removeId =~ /\A[0-9a-f]{8}\z/` before entering the remove block. The
validation is merged into the CR-03 fix structure: the accounts hash is
fetched once and reused for the membership check and for computing remaining
accounts.

### WR-04: switchAccount accountId Not Validated Against Known Accounts

**Files modified:** `Plugins/SpotOn/Settings.pm`
**Commit:** 8d6c398
**Applied fix:** The switchAccount block now fetches `$prefs->get('accounts')`
and only calls `$prefs->set('activeAccount', $switchId)` if
`exists $accounts->{$switchId}`.

### WR-05: Misleading Comment "addAccount is synchronous, no callback needed"

**Files modified:** `Plugins/SpotOn/Settings.pm`
**Commit:** 948b5b2
**Applied fix:** Replaced the one-line misleading comment with a four-line
accurate description explaining the blocking/synchronous nature and including
an explicit IMPORTANT warning for future maintainers about what breaks if
`addAccount` is ever made truly async.

## Skipped Issues

### CR-01: Password Exposed in Process List During Authentication

**File:** `Plugins/SpotOn/API/TokenManager.pm:164-165`
**Reason:** deferred by instruction — will be obsoleted by OAuth-PKCE in the next phase. The entire `--authenticate` code path (backtick spawn with `--username`/`--password` args) will be replaced by a PKCE-based OAuth flow in Phase 3.
**Original issue:** Spotify account password passed as a CLI argument, visible in `ps aux` and `/proc/<pid>/cmdline` for the duration of the backtick call.

### WR-01: alarm() Has No Effect on Windows

**File:** `Plugins/SpotOn/API/TokenManager.pm:73-79` and `169-177`
**Reason:** skipped by instruction — LMS on Windows is extremely rare, low priority for this phase.
**Original issue:** `alarm()` is a no-op on Windows; blocking backtick calls can hang the LMS event loop indefinitely on Windows.

---

_Fixed: 2026-05-27_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
