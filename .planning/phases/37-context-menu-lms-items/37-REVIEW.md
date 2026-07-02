---
phase: 37-context-menu-lms-items
reviewed: 2026-06-30T00:00:00Z
depth: quick
files_reviewed: 2
files_reviewed_list:
  - Plugins/SpotOn/ProtocolHandler.pm
  - Plugins/SpotOn/strings.txt
findings:
  critical: 0
  warning: 1
  info: 2
  total: 3
status: issues_found
---

# Phase 37: Code Review Report

**Reviewed:** 2026-06-30
**Depth:** quick
**Files Reviewed:** 2
**Status:** issues_found

## Summary

Reviewed the new `getIcon` method added to `ProtocolHandler.pm` and the "SpotOn:" prefix additions to six context-menu string keys in `strings.txt`. The `getIcon` implementation has a correctness gap: it omits the URL normalization that the sibling `getMetadataFor` method applies, causing a cache-key mismatch for any URL in the `spoton:track:ID` (no `//`) form — which LMS is known to pass. Two cosmetic string inconsistencies were also found.

## Warnings

### WR-01: `getIcon` skips URL normalization — silent cache-key mismatch

**File:** `Plugins/SpotOn/ProtocolHandler.pm:762`

**Issue:** `getMetadataFor` (lines 723–731) normalizes `spoton:track:ID` → `spoton://track:ID` before computing the `md5_hex` cache key, with an explicit two-step fallback. The comment there explicitly states "LMS may pass `spoton:track:ID`", acknowledging this as a real code path. All metadata is stored under the canonical `spoton://track:…` key. `getIcon`, introduced in this phase, performs no such normalization — it calls `md5_hex($url)` on the raw incoming URL:

```perl
# as written (line 762)
my $meta = $cache->get('spoton_meta_' . md5_hex($url));
```

When LMS passes `spoton:track:ID` to `getIcon`, `md5_hex($url)` produces a different hash than the stored key. The lookup misses silently, and the method falls back to the generic SpotOn plugin icon instead of the album cover. No error is logged; the user just sees the wrong art.

**Fix:** Mirror the same two-step lookup used in `getMetadataFor`:

```perl
sub getIcon {
    my ($class, $url) = @_;

    if ($url) {
        my $canonical = $url;
        $canonical =~ s{^spoton:(?!//)}{spoton://} if $canonical =~ m{^spoton:(?!//)};
        my $meta = $cache->get('spoton_meta_' . md5_hex($canonical));
        if (!$meta && $canonical ne $url) {
            $meta = $cache->get('spoton_meta_' . md5_hex($url));
        }
        return $meta->{cover}
            if $meta && $meta->{cover} && $meta->{cover} ne '/html/images/cover.png';
    }

    return 'plugins/SpotOn/html/images/SpotOn_MTL_svg_spoton.png';
}
```

## Info

### IN-01: `PLUGIN_SPOTON_SHOW_VIEW` English value uses lowercase "show"

**File:** `Plugins/SpotOn/strings.txt:1721`

**Issue:** The EN value is `SpotOn: View show` — "show" is not capitalised. Every other context-menu label in this phase uses title case for the noun: `View Artist`, `View Album`, `Like / Unlike`, `Follow / Unfollow`, `Add to Playlist`. The mismatch is visible to EN users in the LMS context menu.

**Fix:**
```
EN	SpotOn: View Show
```

### IN-02: `PLUGIN_SPOTON_MANAGE_FOLLOW` Czech translation has order reversed

**File:** `Plugins/SpotOn/strings.txt:1614`

**Issue:** The CS (Czech) value is `SpotOn: Odebrat / Sledovat` — "Remove / Follow". Every other locale follows the English order "Follow / Unfollow" (positive action first): DA `Folg / Fjern`, DE `Folgen / Entfolgen`, ES `Seguir / Dejar`, FR `Suivre / Ne plus suivre`, NL `Volgen / Ontvolgen`, NO `Folg / Slutt å følge`, PL `Obserwuj / Przestań`, SV `Följa / Sluta följa`. Only CS is reversed.

**Fix:**
```
CS	SpotOn: Sledovat / Odebrat
```

---

_Reviewed: 2026-06-30_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: quick_
