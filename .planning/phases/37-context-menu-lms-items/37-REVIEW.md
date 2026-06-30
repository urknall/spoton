---
phase: 37-context-menu-lms-items
reviewed: 2026-06-30T00:00:00Z
depth: standard
files_reviewed: 4
files_reviewed_list:
  - t/14_context_menu.t
  - Plugins/SpotOn/ProtocolHandler.pm
  - t/10_stream_metadata.t
  - t/11_track_history.t
findings:
  critical: 1
  warning: 2
  info: 3
  total: 6
status: issues_found
---

# Phase 37: Code Review Report

**Reviewed:** 2026-06-30
**Depth:** standard
**Files Reviewed:** 4
**Status:** issues_found

## Summary

Four files reviewed: the new context-menu test (`t/14_context_menu.t`), the protocol handler (`Plugins/SpotOn/ProtocolHandler.pm`), and two existing test files (`t/10_stream_metadata.t`, `t/11_track_history.t`). The test suite is well-structured and the stub/mock infrastructure is solid.

One critical defect was found in `Plugin.pm`'s `trackInfoMenu`: the `PLUGIN_SPOTON_MANAGE_FOLLOW` item built for the episode branch is missing `type => 'link'`, making it non-functional as a context menu entry. This defect is not caught by the CTX-05 test, which only checks for the presence of `name` fields. Every other `PLUGIN_SPOTON_MANAGE_FOLLOW` item in the codebase consistently carries `type => 'link'`.

Two warnings concern implicit LMS module dependencies in `ProtocolHandler.pm` — modules called by full package path without a preceding `use` or `require`.

---

## Critical Issues

### CR-01: PLUGIN_SPOTON_MANAGE_FOLLOW missing `type => 'link'` in trackInfoMenu

**File:** `Plugins/SpotOn/Plugin.pm:603`

**Issue:** The `PLUGIN_SPOTON_MANAGE_FOLLOW` item added by `trackInfoMenu` (episode branch) is missing the `type => 'link'` key present on every other MANAGE_FOLLOW item in the codebase. In the LMS OPML framework, items without an explicit `type` are treated as non-navigable text nodes. The item renders visibly in the context menu but cannot be selected — the "Follow Show / Unfollow Show" action is silently broken when accessed via the NowPlaying track-info menu on an episode.

Cross-reference: the same `SpotOnManageFollow` function reference is used at lines 1635 and 1712, both with `type => 'link'`. Only the `trackInfoMenu` path is missing it.

The CTX-05 test at `t/14_context_menu.t:471-476` checks only for `name` field values and does not assert that `type` is correct, so the test suite passes despite this defect.

**Fix:**
```perl
# Plugin.pm lines 603-608 — add type => 'link'
push @items, {
    name        => cstring($client, 'PLUGIN_SPOTON_MANAGE_FOLLOW'),
    url         => \&SpotOnManageFollow,
    passthrough => [{ showUri => "spotify:show:$meta->{showId}", accountId => $accountId }],
    type        => 'link',    # add this line — matches every other MANAGE_FOLLOW item
    favorites   => 0,
};
```

---

## Warnings

### WR-01: `Slim::Utils::Misc` called without `use`/`require` in ProtocolHandler.pm

**File:** `Plugins/SpotOn/ProtocolHandler.pm:171`

**Issue:** `requestString` calls `Slim::Utils::Misc::crackURL($url)` via a full package path, but `Slim::Utils::Misc` is never `use`d or `require`d in `ProtocolHandler.pm`. In production LMS this works because the framework loads `Slim::Utils::Misc` before any plugin code runs. However, the dependency is implicit and invisible — loading the module in isolation (e.g., a future test that exercises `requestString`) will fail with "Undefined subroutine &Slim::Utils::Misc::crackURL". The same file has explicit `use` declarations for all its other Slim dependencies.

**Fix:**
```perl
# Add to the use block at the top of ProtocolHandler.pm
use Slim::Utils::Misc;
```

### WR-02: `Slim::Player::Source` called without `use`/`require` in ProtocolHandler.pm

**File:** `Plugins/SpotOn/ProtocolHandler.pm:245`

**Issue:** `_retryStream` calls `Slim::Player::Source::streamingSongIndex($client)` without importing the module. Same implicit dependency issue as WR-01 — relies on LMS's bootstrap load order. `t/11_track_history.t` does not exercise `_retryStream`, so the missing stub goes unnoticed in tests.

**Fix:**
```perl
# Inside _retryStream, add an explicit guard before the call:
require Slim::Player::Source;
my $idx = Slim::Player::Source::streamingSongIndex($client) // 0;
```

---

## Info

### IN-01: Dead code branch in `getFormatForURL`

**File:** `Plugins/SpotOn/ProtocolHandler.pm:44`

**Issue:** The third return statement in `getFormatForURL` matches Browse daemon HTTP URLs and returns `'soc'`, but the unconditional fallback on line 45 also returns `'soc'`. The regex check can never produce a different result from the default, making line 44 unreachable dead code. The docstring at line 37 also states Browse URLs return `'son'`, which is inconsistent with the actual behavior (always returns `'soc'`).

```perl
# Line 44 is dead — both paths return 'soc'
return 'soc' if $url && $url =~ m{:\d+/(?:track|episode)/};  # never reached
return 'soc';
```

**Fix:** Remove line 44 and update the docstring to remove the stale `'son'` reference.

### IN-02: CTX-05 test does not verify `type` field — misses CR-01

**File:** `t/14_context_menu.t:466`

**Issue:** CTX-05 iterates over returned items and checks only `$_->{name}` values. The `type` field that controls LMS item rendering is never asserted, allowing the missing `type => 'link'` defect (CR-01) to pass the test suite undetected.

**Fix:** Add a type assertion after the name checks in CTX-05:
```perl
my @types = map { $_->{type} } @$result;
ok( (grep { ($_ // '') eq 'link' } @types) == scalar @$result,
    'CTX-05: all items have type => link' );
```
This would have caught CR-01.

### IN-03: `$_translatedConnectUrls` cap evicts arbitrary entry, not oldest

**File:** `Plugins/SpotOn/ProtocolHandler.pm:377`

**Issue:** When the hash hits 200 entries, the eviction logic deletes a random key (Perl hash iteration order is undefined):
```perl
delete $_translatedConnectUrls{(keys %_translatedConnectUrls)[0]};
```
The apparent intent is to bound growth, but the "oldest entry" semantics implied by a simple cap are not achieved — a recently-added URL could be evicted while a stale one survives. Under normal use (at most a few hundred URL translations per session) this has no practical impact, but the behavior is worth documenting.

**Fix:** Document the random-eviction intent explicitly, or switch to an ordered structure (array of keys in insertion order) if true FIFO is needed:
```perl
# Comment should say: evicts an arbitrary entry to bound hash size
delete $_translatedConnectUrls{(keys %_translatedConnectUrls)[0]};
```

---

_Reviewed: 2026-06-30_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
