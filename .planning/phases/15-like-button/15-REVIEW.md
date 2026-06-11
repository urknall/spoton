---
status: findings
phase: 15-like-button
reviewed: 2026-06-11T16:00:00Z
reviewed_files:
  - Plugins/SpotOn/API/Client.pm
  - Plugins/SpotOn/API/TokenManager.pm
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/ProtocolHandler.pm
  - Plugins/SpotOn/strings.txt
  - t/02_strings.t
  - t/08_api_client.t
finding_count: 5
severity_counts:
  critical: 0
  warning: 3
  info: 2
---

# Phase 15: Like-Button — Code Review

**Reviewed:** 2026-06-11
**Depth:** standard + targeted cross-file analysis
**Files Reviewed:** 7
**Status:** findings (0 critical, 3 warnings, 2 info)

## Summary

Phase 15 implements the Spotify Like/Unlike feature through three entry points: OPML Browse via `_trackItem` context items, Material Skin via `ProtocolHandler::trackInfoURL`, and the LMS TrackInfo menu via `registerInfoProvider`. The core API methods (`saveTracks`, `removeTracks`, `checkTracks`) are correctly implemented. The empty-body guard for PUT/DELETE `/me/library` is sound. Cache version bumps cover all modified modules. All 131 tests pass.

Three warnings were found: incorrect skip-counts in new test SKIP blocks, a missing `use Slim::Utils::Strings` in `ProtocolHandler.pm`, and cache version inconsistency in two unmodified modules that creates a future maintenance risk. No logic errors, security vulnerabilities, or data-loss paths were identified.

---

## Warnings

### WR-01: Incorrect SKIP counts in LIB-01, LIB-02, LIB-03 test blocks

**File:** `t/08_api_client.t:518`, `541`, `564`

**Issue:** The skip argument declares fewer tests than are actually present in each block. When `Client.pm` is absent the TAP parser records a plan mismatch, which causes the test suite to appear broken in CI scaffolding or when the module is intentionally removed for testing:

| Block | skip declares | actual assertions |
|-------|--------------|-------------------|
| LIB-01 | 3 | 5 (`is`, `is`, `like`, `like`, `is`) |
| LIB-02 | 3 | 5 (`is`, `is`, `like`, `like`, `is`) |
| LIB-03 | 4 | 6 (`is`, `is`, `like`, `like`, `ok`, `is`) |

LIB-04 (skip=2, actual=2) and LIB-05 (skip=2, actual=2) are correct.

**Fix:**

```perl
# LIB-01 (line 518)
skip "Client.pm not yet created", 5

# LIB-02 (line 541)
skip "Client.pm not yet created", 5

# LIB-03 (line 564)
skip "Client.pm not yet created", 6
```

---

### WR-02: `Slim::Utils::Strings` used without `use` declaration in `ProtocolHandler.pm`

**File:** `Plugins/SpotOn/ProtocolHandler.pm:412`

**Issue:** `trackInfoURL` calls `Slim::Utils::Strings::cstring(...)` via fully-qualified name, but `ProtocolHandler.pm` has no `use Slim::Utils::Strings` statement. The call works at runtime because LMS loads `Slim::Utils::Strings` as a core module before any plugin callbacks execute. However:

1. The dependency is invisible to static analysis and `perlcritic`.
2. The plugin's test scaffold (which stubs LMS modules selectively) does not load it, so any future test that exercises `trackInfoURL` will die with `Undefined subroutine &Slim::Utils::Strings::cstring`.
3. Every other module in the codebase that calls `cstring` uses an explicit `use` (e.g. `Plugin.pm:12`).

**Fix:** Add the import to `ProtocolHandler.pm` alongside the other `use Slim::Utils::*` statements:

```perl
use Slim::Utils::Strings qw(cstring);
```

---

### WR-03: `Connect.pm` and `DontStopTheMusic.pm` still use cache version 2

**File:** `Plugins/SpotOn/Connect.pm:43`, `Plugins/SpotOn/DontStopTheMusic.pm:16`

**Issue:** Phase 15 bumped the shared `'spoton'` cache namespace version to 3 in `Plugin.pm`, `Client.pm`, `TokenManager.pm`, and `ProtocolHandler.pm`, but skipped two modules:

```
Plugins/SpotOn/Connect.pm:           Slim::Utils::Cache->new('spoton', 2)
Plugins/SpotOn/DontStopTheMusic.pm:  Slim::Utils::Cache->new('spoton', 2)
```

`Slim::Utils::Cache` returns a **singleton per namespace** (`/usr/share/perl5/Slim/Utils/Cache.pm:109`: `return $caches{$namespace} if $caches{$namespace}`). The first `new()` call for a namespace creates the instance and stores the version; all subsequent calls return the cached instance regardless of the version argument. Since `Plugin.pm` always loads before `Connect.pm` in the LMS lifecycle, the singleton is initialized at v3 and the v2 argument from `Connect.pm` is silently ignored — no cache-clearing loop occurs at runtime.

The risk is future loading-order changes: a test scaffold, direct `Connect.pm` invocation, or future plugin restructuring could cause `Connect.pm` to be first-to-initialize. In that scenario: v2 creates instance + stores v2 → Plugin.pm sees stored=2, expected=3 → clears + stores v3 → Connect.pm next call returns existing v3 instance. This is survivable but produces an unnecessary cache flush on every reload.

Additionally, the inconsistency makes the codebase misleading to read: a maintainer inspecting `Connect.pm` would believe it uses a v2-era cache.

**Fix:** Update both files to match the canonical version:

```perl
# Plugins/SpotOn/Connect.pm line 43
my $cache = Slim::Utils::Cache->new('spoton', 3);

# Plugins/SpotOn/DontStopTheMusic.pm line 16
my $cache = Slim::Utils::Cache->new('spoton', 3);
```

---

## Info

### IN-01: `trackId` extraction and `cacheKey` construction duplicated in `SpotOnManageLike` and `_toggleLike`

**File:** `Plugins/SpotOn/Plugin.pm:411-412` and `455-456`

**Issue:** Both functions contain identical code:

```perl
(my $trackId) = $trackUri =~ /^spotify:track:(.+)$/;
my $cacheKey = "spoton_liked_${accountId}_${trackId}";
```

The duplication means any future change to the cache key format requires editing two places. The helpers are close enough in logic that a divergence could go unnoticed.

**Suggested fix:** Extract a private helper, or consolidate the key construction into a single call site:

```perl
sub _likedCacheKey {
    my ($accountId, $trackUri) = @_;
    my ($trackId) = $trackUri =~ /^spotify:track:(.+)$/;
    return "spoton_liked_${accountId}_${trackId}";
}
```

---

### IN-02: `_toggleLike` and `SpotOnManageLike` have no guard on `$args->{trackUri}` being undef

**File:** `Plugins/SpotOn/Plugin.pm:452` and `408`

**Issue:** Both functions extract `$trackUri = $args->{trackUri}` and then immediately apply a regex without a defined-check. If `$trackUri` is undef, the regex produces an uninitialized-value warning, `$trackId` is undef, and the resulting cache key becomes `"spoton_liked_${accountId}_"` (empty suffix) — a malformed key shared across all tracks for that account. The API call would then send `uris=` with an empty value.

All current callers validate the URI before constructing the passthrough (`_trackItem` at line 563 guards with `/^spotify:track:[A-Za-z0-9]+$/`; `trackInfoURL` validates at extraction), so this path is unreachable today. It is a missing defense-in-depth layer.

**Suggested fix:** Add an early return guard in each function:

```perl
sub _toggleLike {
    my ($client, $cb, $params, $args) = @_;
    my $trackUri  = $args->{trackUri} // '';
    return unless $trackUri =~ /^spotify:track:[A-Za-z0-9]+$/;
    ...
```

```perl
sub SpotOnManageLike {
    my ($client, $cb, $params, $args) = @_;
    my $trackUri  = $args->{trackUri} // '';
    return unless $trackUri =~ /^spotify:track:[A-Za-z0-9]+$/;
    ...
```

---

_Reviewed: 2026-06-11_
_Reviewer: Claude Sonnet 4.6 (adversarial code review)_
_Depth: standard + cross-file_
