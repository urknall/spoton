# Phase 37: Context Menu LMS Items - Research

**Researched:** 2026-06-30
**Domain:** LMS TrackInfo menu framework, OPML info provider system
**Confidence:** HIGH

## Summary

The root cause of GH #55 is identified with HIGH confidence: SpotOn's `ProtocolHandler.pm` defines a `trackInfoURL` method (line 758) which **completely replaces** the standard LMS TrackInfo menu. When `Slim::Menu::TrackInfo::menu()` detects that a protocol handler implements `trackInfoURL`, it returns that feed immediately (line 243 of TrackInfo.pm: `return $feed if $feed;`) and never iterates through registered info providers. This prevents all standard LMS items (Add to Favorites, play controls, More Info, etc.) and all other plugin providers from appearing.

Neither Spotty nor Qobuz define `trackInfoURL` in their ProtocolHandlers. Both use `registerInfoProvider` exclusively, which merges their items alongside standard LMS items. SpotOn already has an equivalent `trackInfoMenu` function registered via `registerInfoProvider` in Plugin.pm (line 227), but it never executes because `trackInfoURL` preempts it.

The fix is straightforward: remove `trackInfoURL` from `ProtocolHandler.pm`. The existing registered `trackInfoMenu` in Plugin.pm will then participate in the standard menu assembly, and all standard LMS items will appear alongside SpotOn entries.

**Primary recommendation:** Remove `ProtocolHandler.pm::trackInfoURL` (lines 758-837). The existing `Plugin.pm::trackInfoMenu` (registered via `registerInfoProvider` at line 227) already provides identical functionality and will work correctly once the override is removed.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CTX-01 | Standard-LMS-Menueeintraege (Add to Favorites, etc.) erscheinen im SpotOn More-Menue neben den SpotOn-Eintraegen | Root cause identified: `trackInfoURL` in ProtocolHandler.pm completely overrides standard menu. Fix: remove it, let `registerInfoProvider`-based `trackInfoMenu` in Plugin.pm handle SpotOn items alongside standard LMS items. |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Track context menu items | LMS Plugin (Plugin.pm) | -- | `registerInfoProvider` is the standard LMS extension point for adding items to TrackInfo menus |
| Standard LMS menu items | LMS Framework (TrackInfo.pm) | Favorites Plugin | LMS assembles standard items (play, add, contributors, etc.) plus plugin-registered items; Favorites plugin registers its own provider |
| Menu override / bypass | Protocol Handler (ProtocolHandler.pm) | -- | `trackInfoURL` is the override mechanism that MUST be removed |

## Standard Stack

No new libraries or packages are required for this phase. This is a code-only change within the existing SpotOn codebase.

### Core Components Involved
| Component | File | Purpose | Change |
|-----------|------|---------|--------|
| `ProtocolHandler.pm` | `Plugins/SpotOn/ProtocolHandler.pm` | Spotify protocol handler | Remove `trackInfoURL` method |
| `Plugin.pm` | `Plugins/SpotOn/Plugin.pm` | Plugin entry point, info provider registration | Already correct -- no changes needed |
| `Slim::Menu::TrackInfo` | LMS core (not ours) | Menu assembly framework | No changes -- just need to stop overriding it |
| `Slim::Plugin::Favorites` | LMS core (not ours) | "Add to Favorites" provider | Will automatically appear once override is removed |

## Architecture Patterns

### System Architecture Diagram

```
User presses "More" on a SpotOn track
         |
         v
Slim::Menu::TrackInfo::menu($client, $url, $track, $tags)
         |
         |--- (1) Check: does protocol handler define trackInfoURL?
         |         |
         |    [CURRENT: YES] --> trackInfoURL returns {name, type=opml, items}
         |         |               --> menu() returns EARLY <-- PROBLEM: standard items skipped
         |         |
         |    [AFTER FIX: NO] --> Falls through to step (2)
         |
         |--- (2) Get info ordering from registered providers
         |         |
         |         +-- "addtrack" (menuMode=1, after top) --> Play controls
         |         +-- "addtracknext" (menuMode=1)        --> Play next
         |         +-- "playitem" (menuMode=1)            --> Play item
         |         +-- "favorites" (after playitem)       --> Add to/Remove from Favorites
         |         +-- "spotonTrackInfo" (after top)       --> SpotOn: Artist/Album/Like/Playlist
         |         +-- "contributors" (after top)          --> Artist info (if schema data)
         |         +-- "album" (after contributors)        --> Album info (if schema data)
         |         +-- "remotetitle" (after album)         --> Remote title
         |         +-- "moreinfo" (after comment)          --> Bitrate, duration, URL, etc.
         |         +-- ... other standard providers ...
         |
         v
    All items merged into single OPML menu
```

### Pattern 1: Info Provider Registration (Correct Pattern)
**What:** Plugins add items to TrackInfo/AlbumInfo/ArtistInfo menus by registering info providers via `registerInfoProvider`. The framework calls each provider function and merges results. [VERIFIED: LMS slimserver source TrackInfo.pm line 286-307]
**When to use:** Always -- this is the only way to add items while preserving standard LMS items.
**Example:**
```perl
# Source: Slim::Menu::TrackInfo::menu() lines 275-308 (slimserver public/9.0)
# In initPlugin():
Slim::Menu::TrackInfo->registerInfoProvider( spotonTrackInfo => (
    after => 'top',
    func  => \&trackInfoMenu,
) );

# Provider function:
sub trackInfoMenu {
    my ($client, $url, $track, $remoteMeta, $tags) = @_;
    # Return arrayref of items -- they get flattened into the main menu
    # Return hashref for a single item
    # Return undef to add nothing
    return \@items;
}
```

### Pattern 2: trackInfoURL Override (Anti-Pattern for Plugins)
**What:** Protocol handlers can define `trackInfoURL` to completely replace the TrackInfo menu. LMS checks for this FIRST and returns early if found. [VERIFIED: LMS slimserver source TrackInfo.pm lines 238-245]
**When to use:** Almost never -- only for protocols that truly need to replace the entire menu with a custom feed URL. Not appropriate for plugins that want to add items alongside standard LMS items.
**Example (what NOT to do):**
```perl
# Source: ProtocolHandler.pm line 758 (SpotOn -- to be removed)
# This PREVENTS all standard items from appearing
sub trackInfoURL {
    my ($class, $client, $url) = @_;
    return {
        name  => $meta->{title},
        type  => 'opml',
        items => \@items,  # Only these items appear -- no Favorites, no standard items
    };
}
```

### Anti-Patterns to Avoid
- **Defining `trackInfoURL` in ProtocolHandler:** This completely overrides the standard menu assembly. Neither Spotty nor Qobuz do this. [VERIFIED: checked both repos, no `trackInfoURL` in either ProtocolHandler.pm]
- **Duplicating menu logic between Plugin.pm and ProtocolHandler.pm:** SpotOn currently has identical trackInfoMenu logic in both files. After removing `trackInfoURL`, only the Plugin.pm version should remain.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Track context menu | Custom `trackInfoURL` in ProtocolHandler | `registerInfoProvider` in Plugin.pm | LMS framework handles ordering, merging, and standard items automatically |
| Add to Favorites | Custom favorites implementation | LMS Favorites plugin (auto-registers its provider) | Favorites plugin handles add/remove/check, UI states, and persistence |
| Play controls in menu | Custom play/add/insert items | LMS standard providers (addtrack, addtracknext, playitem) | These are `menuMode=1` items added by LMS for Jive/Material UIs |

## Common Pitfalls

### Pitfall 1: trackInfoURL Replacing Entire Menu
**What goes wrong:** Defining `trackInfoURL` in a ProtocolHandler causes `Slim::Menu::TrackInfo::menu()` to return early at line 243, before any registered info providers (including Favorites) are called.
**Why it happens:** The developer assumes `trackInfoURL` adds items alongside standard ones, but it actually replaces the entire menu.
**How to avoid:** Never define `trackInfoURL` unless you intentionally want to replace the entire TrackInfo menu. Use `registerInfoProvider` instead.
**Warning signs:** No "Add to Favorites" or standard LMS items in the More menu for your plugin's tracks.

### Pitfall 2: Provider Return Value Semantics
**What goes wrong:** A provider returns an arrayref expecting it to create a submenu, but LMS actually flattens it. [VERIFIED: TrackInfo.pm line 294-297: `push @{$items}, @{$item}` for ARRAY ref]
**Why it happens:** Confusion about how LMS processes return values.
**How to avoid:** Know the semantics: arrayref = items are flattened into main list; hashref = single item added to main list; undef = nothing added.
**Warning signs:** Items appearing at wrong nesting level.

### Pitfall 3: Standard Providers Expecting Schema Data on Remote Tracks
**What goes wrong:** Standard providers like `contributors`, `album`, `genres` may return empty results for remote/streaming tracks because the Slim::Schema::Track object doesn't have those relationships populated.
**Why it happens:** These providers use `$track->contributorsOfType()`, `$track->album`, etc. which require database-backed track objects.
**How to avoid:** This is expected behavior -- not all standard items will show meaningful data for remote tracks. The important ones (Favorites, play controls, More Info with bitrate/duration) work fine with remote metadata. Spotty has the same behavior.
**Warning signs:** None -- this is normal.

### Pitfall 4: favorites => 0 Property Confusion
**What goes wrong:** Setting `favorites => 0` on an OPML item prevents the "Add to Favorites" UI affordance for that specific item. SpotOn correctly uses this on sub-items like "Like" and "Add to Playlist" to prevent favoriting the action itself.
**Why it happens:** Understanding the property as a global toggle rather than per-item.
**How to avoid:** Only set `favorites => 0` on items that should NOT be favoriteable (like action menus). The track-level favorites handling is done by the Favorites plugin's info provider.
**Warning signs:** "Add to Favorites" disappearing entirely.

## Code Examples

### Current SpotOn Registration (Already Correct)
```perl
# Source: Plugin.pm lines 226-230 (SpotOn codebase)
# This is already correct and will work once trackInfoURL is removed
require Slim::Menu::TrackInfo;
Slim::Menu::TrackInfo->registerInfoProvider( spotonTrackInfo => (
    after => 'top',
    func  => \&trackInfoMenu,
) );
```

### Current SpotOn trackInfoMenu (Already Correct)
```perl
# Source: Plugin.pm lines 539-619 (SpotOn codebase)
sub trackInfoMenu {
    my ($client, $url, $track, $remoteMeta, $tags) = @_;
    # Returns undef for non-spoton URLs (correct -- doesn't interfere with other plugins)
    # Returns \@items for spoton URLs (correct -- items get flattened into main menu)
    return @items ? \@items : undef;
}
```

### How Spotty Registers (Reference)
```perl
# Source: Spotty OPML.pm lines 79-91 (michaelherger/Spotty-Plugin)
Slim::Menu::TrackInfo->registerInfoProvider( spotty => (
    after => 'top',
    func  => \&trackInfoMenu,
) );
Slim::Menu::ArtistInfo->registerInfoProvider( spotty => (
    after => 'top',
    func  => \&artistInfoMenu,
) );
Slim::Menu::AlbumInfo->registerInfoProvider( spotty => (
    after => 'top',
    func  => \&albumInfoMenu,
) );
```

### How LMS Merges Provider Items (Framework Code)
```perl
# Source: Slim::Menu::TrackInfo::menu() lines 275-308 (slimserver public/9.0)
my $addItem = sub {
    my ( $ref, $items ) = @_;
    my $item = eval { $ref->{func}->( $client, $url, $track, $remoteMeta, $tags, $filter ) };
    return unless defined $item;
    if ( ref $item eq 'ARRAY' ) {
        if ( scalar @{$item} ) {
            push @{$items}, @{$item};   # <-- FLATTENS array into main list
        }
    }
    elsif ( ref $item eq 'HASH' ) {
        if ( scalar keys %{$item} ) {
            push @{$items}, $item;       # <-- Adds single item to main list
        }
    }
};
```

### How LMS Favorites Provider Works
```perl
# Source: Slim::Plugin::Favorites::Plugin lines 91-108 (slimserver public/9.0)
# Registers for TrackInfo, AlbumInfo, ArtistInfo, and PlaylistInfo
Slim::Menu::TrackInfo->registerInfoProvider( favorites => (
    after => 'playitem',
    func  => \&trackInfoHandler,
) );
# The handler checks if URL is already in favorites and shows
# "Save to Favorites" or "Remove from Favorites" accordingly.
# Works with any URL including spoton:// protocol URLs.
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| SpotOn `trackInfoURL` override | Should use `registerInfoProvider` only | Phase 37 fix | Standard LMS items will appear |
| Duplicated menu logic in Plugin.pm + ProtocolHandler.pm | Single implementation in Plugin.pm via `registerInfoProvider` | Phase 37 fix | Cleaner codebase, no redundancy |

**Deprecated/outdated:**
- `trackInfoURL` in ProtocolHandler.pm: Not deprecated by LMS, but inappropriate for plugins that want to coexist with standard menu items. Must be removed.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Removing `trackInfoURL` will not break any other functionality that depends on it | Architecture Patterns | LOW risk -- `trackInfoMenu` in Plugin.pm provides identical items; `getMetadataFor` is separate and unaffected |
| A2 | LMS Favorites works correctly with `spoton://` protocol URLs (can add/play back favorites) | Common Pitfalls | LOW risk -- Favorites stores URLs as-is and LMS resolves them via ProtocolHandler on playback; same mechanism used by all streaming plugins |
| A3 | Standard providers returning empty data for remote tracks is acceptable (matches Spotty behavior) | Common Pitfalls | LOW risk -- this is documented LMS behavior for remote tracks |

## Open Questions

1. **AlbumInfo and ArtistInfo providers**
   - What we know: Spotty registers providers for all three (TrackInfo, AlbumInfo, ArtistInfo). SpotOn only registers TrackInfo. The success criteria mentions "tracks, albums, and artists."
   - What's unclear: Whether CTX-01 requires AlbumInfo/ArtistInfo providers. SpotOn's album/artist views are OPML feeds (not schema objects), so AlbumInfo/ArtistInfo providers would only matter for library-imported items (Phase 38+).
   - Recommendation: Out of scope for Phase 37. AlbumInfo/ArtistInfo providers can be added in Phase 38-41 when library import makes schema objects available. The "albums and artists" in the success criteria likely refers to the track info menu showing artist/album navigation (which already works via SpotOn's trackInfoMenu items).

2. **Dead code cleanup scope**
   - What we know: `trackInfoURL` in ProtocolHandler.pm (lines 758-837) is ~80 lines of code that duplicates `trackInfoMenu` in Plugin.pm (lines 539-619).
   - What's unclear: Whether to simply remove `trackInfoURL` or also refactor/consolidate the remaining code.
   - Recommendation: Just remove `trackInfoURL`. The Plugin.pm `trackInfoMenu` is already correct and working. No refactoring needed.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Perl Test::More (bundled with Perl) |
| Config file | none (tests run directly with `prove`) |
| Quick run command | `prove -v t/XX_test_name.t` |
| Full suite command | `prove -v t/` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CTX-01 | `trackInfoURL` removed from ProtocolHandler.pm | unit | `prove -v t/14_context_menu.t` | No -- Wave 0 |
| CTX-01 | `trackInfoMenu` in Plugin.pm returns correct items for spoton:// URLs | unit | `prove -v t/14_context_menu.t` | No -- Wave 0 |
| CTX-01 | `trackInfoMenu` returns undef for non-spoton URLs | unit | `prove -v t/14_context_menu.t` | No -- Wave 0 |
| CTX-01 | `ProtocolHandler.pm` does not define `trackInfoURL` method | unit | `prove -v t/14_context_menu.t` | No -- Wave 0 |
| CTX-01 | Standard LMS items appear in More menu | manual-only | Visual verification on running LMS | N/A |

### Sampling Rate
- **Per task commit:** `prove -v t/14_context_menu.t`
- **Per wave merge:** `prove -v t/`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `t/14_context_menu.t` -- covers CTX-01: verifies `trackInfoURL` is removed, `trackInfoMenu` returns correct items
- [ ] Test::More stubs for `Slim::Menu::TrackInfo`, `Slim::Utils::Cache`, `Slim::Utils::Strings` (reuse existing stub patterns from `t/08_api_client.t`)

## Security Domain

No security implications for this phase. This is a UI menu configuration change within the existing LMS framework. No authentication, input validation, or data flow changes are involved.

## Sources

### Primary (HIGH confidence)
- [LMS slimserver TrackInfo.pm](https://github.com/LMS-Community/slimserver/blob/public/9.0/Slim/Menu/TrackInfo.pm) -- menu() method lines 238-245 (trackInfoURL override), lines 275-308 (provider iteration), lines 294-297 (arrayref flattening)
- [LMS slimserver Menu/Base.pm](https://github.com/LMS-Community/slimserver/blob/public/9.0/Slim/Menu/Base.pm) -- registerInfoProvider API, ordering system
- [LMS Favorites Plugin](https://github.com/LMS-Community/slimserver/blob/public/9.0/Slim/Plugin/Favorites/Plugin.pm) -- lines 91-108 (info provider registration), lines 1018-1090 (_objectInfoHandler)
- [Spotty-Plugin OPML.pm](https://github.com/michaelherger/Spotty-Plugin/blob/master/OPML.pm) -- lines 79-91 (registerInfoProvider), lines 1477-1511 (trackInfoMenu), lines 1620+ (_objInfoMenu)
- [Spotty-Plugin ProtocolHandler.pm](https://github.com/michaelherger/Spotty-Plugin/blob/master/ProtocolHandler.pm) -- confirmed NO trackInfoURL defined
- [Qobuz Plugin.pm](https://github.com/LMS-Community/plugin-Qobuz/blob/master/Plugin.pm) -- registerInfoProvider without `after`/`before`, returns single hashref
- [Qobuz ProtocolHandler.pm](https://github.com/LMS-Community/plugin-Qobuz/blob/master/ProtocolHandler.pm) -- confirmed NO trackInfoURL defined
- SpotOn codebase: Plugin.pm lines 226-230 (registration), 539-619 (trackInfoMenu), ProtocolHandler.pm lines 758-837 (trackInfoURL -- root cause)

### Secondary (MEDIUM confidence)
- [GitHub Issue #55](https://github.com/stiefenm/spoton/issues/55) -- User CJS reported missing standard items with screenshots comparing Spotty vs SpotOn

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new packages needed, pure code change
- Architecture: HIGH -- root cause verified by reading LMS framework source code directly
- Pitfalls: HIGH -- verified against three reference implementations (Spotty, Qobuz, LMS core)

**Research date:** 2026-06-30
**Valid until:** Indefinite -- LMS Menu framework is stable across versions
