---
phase: 09-stream-metadata
reviewed: 2026-06-04T15:22:00Z
depth: standard
files_reviewed: 5
files_reviewed_list:
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/Connect.pm
  - Plugins/SpotOn/DontStopTheMusic.pm
  - Plugins/SpotOn/ProtocolHandler.pm
  - t/10_stream_metadata.t
findings:
  critical: 0
  warning: 3
  info: 2
  total: 5
status: issues_found
---

# Phase 09: Code Review Report

**Reviewed:** 2026-06-04T15:22:00Z
**Depth:** standard
**Files Reviewed:** 5
**Status:** issues_found

## Summary

Phase 09-01 adds two class methods (`_typeString`, `_bitrateForClient`) to Plugin.pm and updates four metadata call sites (Plugin.pm `_trackItem`, Plugin.pm `_albumTrackItem`, Connect.pm `_fetchTrackMetadata`, DontStopTheMusic.pm `_cacheAndExtractUris`) plus the `getMetadataFor` overlay in ProtocolHandler.pm. The implementation is clean and focused, with no security issues introduced (no new trust boundaries, no external input interpolation). The `_typeString` auto-resolution via `Helper->getCapability('passthrough')` correctly mirrors the `formatOverride` chain in ProtocolHandler.pm. All call sites consistently use `'Browse'` or `'Connect'` string literals for the mode parameter, and the `getMetadataFor` overlay correctly applies dynamic type/bitrate at read-time to fix the stale-after-pref-change issue.

Three warnings and two informational findings follow. No critical/blocker issues were found.

## Warnings

### WR-01: Doc comment above `_bitrateForClient` describes `_typeString`

**File:** `Plugins/SpotOn/Plugin.pm:1337-1339`
**Issue:** The three-line comment block starting at line 1337 reads `# _typeString($client, $mode)` and describes the `_typeString` function. However, it sits directly above `sub _bitrateForClient` (line 1340). The actual `_typeString` sub at line 1354 has no doc comment. This misleads readers into thinking `_bitrateForClient` returns a display string, and leaves `_typeString` undocumented.
**Fix:** Move the doc comment to above `sub _typeString` and add a separate comment for `_bitrateForClient`:
```perl
# _bitrateForClient($client)
# Returns the effective bitrate (96|160|320) for the given client,
# respecting per-player bitrateOverride over the global bitrate pref.
sub _bitrateForClient {
```

### WR-02: DontStopTheMusic.pm bitrate field not updated to use `_bitrateForClient`

**File:** `Plugins/SpotOn/DontStopTheMusic.pm:272`
**Issue:** The `type` field was updated to use `_typeString(undef, 'Browse')` but the `bitrate` field on the same cache entry still uses the inline expression `($prefs->get('bitrate') || 320) . 'k'`. All other call sites (Plugin.pm `_trackItem` line 404, `_albumTrackItem` line 1142, Connect.pm line 854) were updated to use `_bitrateForClient`. This inconsistency means that if `_bitrateForClient` evolves (e.g., adding new validation tiers or a different default), DontStopTheMusic would diverge silently.

The inconsistency is partially mitigated by `getMetadataFor` in ProtocolHandler.pm (lines 283-288), which overlays the bitrate from `_bitrateForClient` at read-time when `$client` is available. However, when `getMetadataFor` is called without a `$client` (e.g., from scanner context), the stale inline value is returned as-is. Additionally, `_cacheAndExtractUris` has no `$client` available, so calling `_bitrateForClient(undef)` would produce the same result -- but using the shared helper function is preferable for maintainability.
**Fix:** Replace the inline bitrate expression with the helper:
```perl
bitrate  => Plugins::SpotOn::Plugin->_bitrateForClient(undef) . 'k',
```

### WR-03: Test prefs state leaks between tests (no global reset)

**File:** `t/10_stream_metadata.t:269-288`
**Issue:** The `setup_prefs` helper writes to a shared package-level `%_store` in the Prefs stub, and there is no teardown/reset between tests. Per-player prefs are isolated because each test uses a unique client ID string. However, global prefs (e.g., `bitrate`) persist across tests. For example, Test 10 sets `bitrate => 96`, and this value remains for all subsequent tests. If a future test is added that relies on a clean default bitrate without explicitly setting it, it will inherit the value from Test 10.

Currently all tests either set their own `bitrate` or do not depend on it, so this does not cause incorrect results today. But it makes the test file fragile for future additions.
**Fix:** Add a `reset_prefs` helper and call it at the start of each test block, or add a `clear` method to the Prefs stub:
```perl
sub reset_prefs {
    Slim::Utils::Prefs::_reset_store();  # add to stub: sub _reset_store { %_store = () }
}
```

## Info

### IN-01: Missing test for `connectOggOverride` migration fallback in `_typeString`

**File:** `t/10_stream_metadata.t`
**Issue:** The `_typeString` implementation reads `connectOggOverride` as a migration fallback when `streamFormat` is unset (Plugin.pm lines 1360-1362). This fallback path is not covered by any test. All tests either set `streamFormat` explicitly or leave both unset (triggering `'auto'`). A test that sets `connectOggOverride => 'ogg'` without setting `streamFormat` would verify the migration path works.
**Fix:** Add a test case:
```perl
{
    my $client = MockClient->new('player_migration');
    setup_prefs(
        client              => $client,
        connectOggOverride  => 'ogg',
        # streamFormat intentionally not set
    );
    my $result = Plugins::SpotOn::Plugin->_typeString($client, 'Browse');
    is($result, 'OGG (Spotify Browse)',
        'Migration: connectOggOverride=ogg used when streamFormat absent');
}
```

### IN-02: Perl "used only once" warning from test file

**File:** `t/10_stream_metadata.t:286`
**Issue:** Running the test produces the warning: `Name "Plugins::SpotOn::Helper::helperCapabilities" used only once: possible typo at t/10_stream_metadata.t line 286`. This is because the package variable is only assigned in the test file at that one location; the stub declaration is in a separate file loaded via `require`. The warning is harmless but noisy.
**Fix:** Suppress with `no warnings 'once'` around the assignment, or reference the variable twice:
```perl
if (exists $opts{passthrough}) {
    no warnings 'once';
    $Plugins::SpotOn::Helper::helperCapabilities = { passthrough => $opts{passthrough} };
}
```

---

_Reviewed: 2026-06-04T15:22:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
