# Phase 12: Protocol Handler Rename - Pattern Map

**Mapped:** 2026-06-05
**Files analyzed:** 8 (7 modified + 1 new test)
**Analogs found:** 8 / 8

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `Plugins/SpotOn/Plugin.pm` | plugin/controller | request-response | self (current state) | exact |
| `Plugins/SpotOn/ProtocolHandler.pm` | protocol-handler | request-response | self (current state) | exact |
| `Plugins/SpotOn/Connect.pm` | event-handler | event-driven | self (current state) | exact |
| `Plugins/SpotOn/DontStopTheMusic.pm` | provider | request-response | self (current state) | exact |
| `librespot-spoton/src/main.rs` | binary/utility | transform | self (current state) | exact |
| `Plugins/SpotOn/API/Client.pm` | service | request-response | self (current state) | exact |
| `Plugins/SpotOn/API/TokenManager.pm` | service | request-response | self (current state) | exact |
| `t/12_protocol_rename.t` | test | — | `t/11_track_history.t` + `t/03_convert_conf.t` | role-match |

---

## Pattern Assignments

### `Plugins/SpotOn/Plugin.pm` (plugin/controller, request-response)

**Analog:** self — current state of the file

**Three change sites:**

**Change 1 — registerHandler (lines 82-85):**
```perl
# BEFORE (current):
Slim::Player::ProtocolHandlers->registerHandler(
    'spotify',
    'Plugins::SpotOn::ProtocolHandler'
);

# AFTER:
Slim::Player::ProtocolHandlers->registerHandler(
    'spoton',
    'Plugins::SpotOn::ProtocolHandler'
);
```

**Change 2 — URL construction in `_trackItem` (line 418):**
```perl
# BEFORE (current):
my $spotify_url = 'spotify://' . $track_path;

# AFTER — variable rename is at implementer discretion; functionally identical:
my $spoton_url = 'spoton://' . $track_path;
```
All downstream uses of `$spotify_url` in the same function (cache key, OPML item `play` field) follow from this single variable.

**Change 3 — URL construction in `_albumTrackItem` (line 1155) — identical pattern to Change 2:**
```perl
# BEFORE (current):
my $spotify_url = 'spotify://' . $track_path;

# AFTER:
my $spoton_url = 'spoton://' . $track_path;
```

**Change 4 — URL regex in `_killOrphanedProcesses` (line 182):**
```perl
# BEFORE (current):
next unless $url =~ m{^spotify://};

# AFTER:
next unless $url =~ m{^spoton://};
```

**Change 5 — Cache initialization (line 24) — shared pattern, see Shared Patterns below.**

**Change 6 — `cacheSchemaVersion` pref and version constant — new addition to `initPlugin`:**
```perl
# Add constant at file top, alongside other constants:
use constant SPOTON_CACHE_VERSION => 2;

# In $prefs->init({...}) block — add new key:
cacheSchemaVersion => 0,   # default: migration not done

# In initPlugin(), after $prefs->init({...}):
if ( ($prefs->get('cacheSchemaVersion') || 0) < SPOTON_CACHE_VERSION ) {
    $log->info("SpotOn cache schema version changed — cache cleared by namespace version bump");
    $prefs->set('cacheSchemaVersion', SPOTON_CACHE_VERSION);
}
```
The `SPOTON_CACHE_VERSION` constant is also used by the cache init pattern (see Shared Patterns).

---

### `Plugins/SpotOn/ProtocolHandler.pm` (protocol-handler, request-response)

**Analog:** self — current state of the file

**Cache initialization (line 17) — shared pattern, see Shared Patterns below.**

**Full regex/string change map (25 occurrences — mechanical substitution):**

All occurrences of `spotify://` in this file must become `spoton://`. The only
exceptions (which must NOT change) are `spotify:track:` Spotify API URI strings
at lines 266, 351, 431 (per D-06). See RESEARCH.md Pattern 3 for the full
line-by-line map.

**Critical non-mechanical change — canonical normalization (lines 368-372):**
```perl
# BEFORE (current):
my $canonical = $url;
if ($canonical && $canonical =~ m{^spotify:(?!//)}) {
    $canonical =~ s{^spotify:}{spotify://};
}

# AFTER:
my $canonical = $url;
if ($canonical && $canonical =~ m{^spoton:(?!//)}) {
    $canonical =~ s{^spoton:}{spoton://};
}
```
This normalization handles LMS passing `spoton:track:ID` (colon notation) instead
of `spoton://track:ID`. The regex `(?!//)` prevents double-slashing an already
normalized URL.

**Placeholder meta title check (line 403):**
```perl
# BEFORE (current):
my $title = ($url && $url =~ m{spotify://track:}) ? 'Loading...' : '';

# AFTER:
my $title = ($url && $url =~ m{spoton://track:}) ? 'Loading...' : '';
```

**Track ID extraction in `_asyncRefetch` (lines 426, 428):**
```perl
# BEFORE (current):
if ($canonical && $canonical =~ m{spotify://track:([A-Za-z0-9]+)}) {
    $trackId = $1;
} elsif ($url && $url =~ m{spotify://connect-}) {

# AFTER:
if ($canonical && $canonical =~ m{spoton://track:([A-Za-z0-9]+)}) {
    $trackId = $1;
} elsif ($url && $url =~ m{spoton://connect-}) {
```

**String construction in `_asyncRefetch` callback (line ~477-478) — from RESEARCH.md:**
```perl
# The callback builds a browse URL from trackId; BEFORE:
"spotify://track:$trackId"

# AFTER:
"spoton://track:$trackId"
```
Note: The `spotifyUri` field stored in cache stays as `spotify:track:$trackId` (no double-slash — this is a Spotify API URI, not an LMS routing URL, per D-06).

---

### `Plugins/SpotOn/Connect.pm` (event-handler, event-driven)

**Analog:** self — current state of the file

**Cache initialization (line 43) — shared pattern, see Shared Patterns below.**

**`_isDeadHistoryUrl` (line 107):**
```perl
# BEFORE (current):
return 0 unless $url && $url =~ m{spotify://connect-};

# AFTER:
return 0 unless $url && $url =~ m{spoton://connect-};
```

**Three `sprintf` connect-URL constructions (lines 630, 722, 828) — identical pattern:**
```perl
# BEFORE (current):
sprintf("spotify://connect-%u", $ts)

# AFTER:
sprintf("spoton://connect-%u", $ts)
```

**Two `^spotify://` + `!~ connect-` guard patterns (lines 617, 695):**
```perl
# BEFORE (current):
if ($currentUrl =~ m{^spotify://} && $currentUrl !~ m{spotify://connect-}) {

# AFTER:
if ($currentUrl =~ m{^spoton://} && $currentUrl !~ m{spoton://connect-}) {
```

**All remaining `m{spotify://connect-}` matches (lines 285, 592, 598) — mechanical:**
```perl
# BEFORE: $url =~ m{spotify://connect-}
# AFTER:  $url =~ m{spoton://connect-}
```

**Lines that must NOT change** (Spotify API URIs per D-06):
- Line 624: `"spotify:track:$trackId"` stored as `eventTrackUri` pluginData
- Line 714: same pattern in 'start' handler
- Line 771: same pattern in another branch
- Line 918: `spotifyUri => $trackInfo->{uri}` — value comes from Spotify API response

---

### `Plugins/SpotOn/DontStopTheMusic.pm` (provider, request-response)

**Analog:** self — current state of the file

**Cache initialization (line 16) — shared pattern, see Shared Patterns below.**

**Single URL construction in `_cacheAndExtractUris` (line 259):**
```perl
# BEFORE (current):
my $uri = "spotify://$1";

# AFTER:
my $uri = "spoton://$1";
```
Context: `$1` captures `track:[a-z0-9]+` from the regex on line 258
(`$track->{uri} =~ /(track:[a-z0-9]+)/i`). The full resulting URL is
`spoton://track:XXXXX`.

---

### `librespot-spoton/src/main.rs` (binary/utility, transform)

**Analog:** self — current state of the file

**Single critical change at line 667:**
```rust
// BEFORE (current):
// Normalize URI: LMS passes spotify://track:ID, librespot needs spotify:track:ID
let normalized_uri = track_uri.replace("spotify://", "spotify:");

// AFTER:
// Normalize URI: LMS passes spoton://track:ID, librespot needs spotify:track:ID
let normalized_uri = track_uri.replace("spoton://", "spotify:");
```
The surrounding context (lines 666-670) is unchanged:
```rust
// Line 666: comment (update text to match)
// Line 668: (blank or next statement)
let track_id = SpotifyUri::from_uri(&normalized_uri)?;
```
After this change, all 6 platform binaries must be rebuilt (5 via CI
`workflow_dispatch` on `.github/workflows/build-librespot.yml`, 1 Windows via
local `cross build --target x86_64-pc-windows-gnu`).

---

### `Plugins/SpotOn/API/Client.pm` (service, request-response)

**Analog:** self — current state of the file

**Only change: cache initialization (line 35) — see Shared Patterns below.**

No URL construction or regex changes in this file. It uses `spoton_bundled_hint_`
key prefix (already `spoton`-prefixed, not `spotify`-prefixed) — no change needed.

---

### `Plugins/SpotOn/API/TokenManager.pm` (service, request-response)

**Analog:** self — current state of the file

**Only change: cache initialization (line 33) — see Shared Patterns below.**

No URL construction or regex changes in this file.

---

### `t/12_protocol_rename.t` (test)

**Analog:** `t/11_track_history.t` (structural template) + `t/03_convert_conf.t` (grep-assertion pattern)

**Test file structure — copy from `t/11_track_history.t` lines 1-320:**
```perl
#!/usr/bin/perl
use strict;
use warnings;
use Test::More;
use File::Basename qw(dirname);
use Cwd qw(abs_path);

my $test_dir    = dirname(abs_path($0));
my $project_dir = dirname($test_dir);
```

This phase's test is primarily grep-based (no LMS stub loading needed — just source
file scanning). Pattern from `t/03_convert_conf.t`:

```perl
# Open source file, scan lines, collect matches, assert count
open(my $fh, '<', "$project_dir/Plugins/SpotOn/Plugin.pm")
    or BAIL_OUT("Cannot read Plugin.pm: $!");
my @matches;
while (my $line = <$fh>) {
    chomp $line;
    next if $line =~ /^\s*#/;          # skip comment lines
    push @matches, "$.: $line" if $line =~ /PATTERN_TO_FIND/;
}
close($fh);
is(scalar @matches, $expected_count, "Description of assertion");
```

**Six required subtests (PROTO-01 through PROTO-06):**

- **PROTO-01:** No `spotify://` (double-slash) in Plugin.pm, ProtocolHandler.pm,
  Connect.pm, DontStopTheMusic.pm non-comment lines. Exception: allow
  `spotify://` only inside comment lines (explanatory text).
- **PROTO-02:** `registerHandler('spoton'` present in Plugin.pm; `registerHandler('spotify'` absent from Plugin.pm.
- **PROTO-04:** All `sprintf(...connect-` constructions use `spoton://connect-`, not `spotify://connect-`.
- **PROTO-05:** No `registerHandler('spotify'` in any SpotOn Perl file.
- **PROTO-06:** `cacheSchemaVersion` present in Plugin.pm (in `$prefs->init` block and in the post-init guard).
- **Rust binary check:** Verify `main.rs` line 667 contains `spoton://` not `spotify://` in the `replace(` call.

**Regression subtest — existing URL scheme in test data:** The `t/11_track_history.t`
tests reference `spotify://` URLs in mock cache data (lines 432, 484, 496, 498, 506,
557, 568, 587, 590, 599). These must be updated to `spoton://` in the same wave as
the source changes, so the regression test remains green.

---

## Shared Patterns

### Cache Initialization (applies to all 6 Perl modules)

**Source:** `/usr/share/perl5/Slim/Utils/Cache.pm` lines 104-146 (LMS built-in)
**Apply to:** `Plugin.pm`, `ProtocolHandler.pm`, `Connect.pm`, `DontStopTheMusic.pm`, `API/Client.pm`, `API/TokenManager.pm`

```perl
# BEFORE (current — all 6 files have this identical line):
my $cache = Slim::Utils::Cache->new();

# AFTER — all 6 files must use the same namespace + version:
use constant SPOTON_CACHE_VERSION => 2;
my $cache = Slim::Utils::Cache->new('spoton', SPOTON_CACHE_VERSION);
```

**LMS mechanism:** `Slim::Utils::Cache->new($namespace, $version)` creates a
separate `spoton.db` SQLite file. If the stored version in `spoton.db` differs
from `$version`, `$cache->clear()` is called automatically on construction.
Bumping from the implicit default (version 1, `cache.db`) to `('spoton', 2)`
triggers a one-time clear of all SpotOn cache entries. [Source: `/usr/share/perl5/Slim/Utils/Cache.pm` lines 130-138, `/usr/share/perl5/Slim/Utils/DbCache.pm`]

**Critical:** All 6 files must use the identical namespace string `'spoton'` and
the identical `SPOTON_CACHE_VERSION` value. Mismatched namespaces cause silent
cache misses (Pitfall 4 from RESEARCH.md). The constant can be defined once in
Plugin.pm and the same literal `2` used in the other 5 files, or a shared constant
can be exported from Plugin.pm — implementer discretion.

### URL Scheme Substitution Rule (applies to all Perl files)

**Apply to:** All modified `.pm` files
**Rule:** Replace `spotify://` (double-slash, LMS routing URL) with `spoton://`.
**Never replace:** `spotify:` (single colon, Spotify API URI format).

```perl
# CHANGE: LMS routing URLs
'spotify://' . $path    →    'spoton://' . $path
m{spotify://connect-}   →    m{spoton://connect-}
m{^spotify://}          →    m{^spoton://}
s{^spotify:}{spotify://}  →  s{^spoton:}{spoton://}

# DO NOT CHANGE: Spotify API URIs
'spotify:track:' . $id    # stays as-is
m/^spotify:track:([A-Za-z0-9]+)$/  # stays as-is
spotifyUri => $trackInfo->{uri}     # stays as-is (value from API)
```

### Test Stub Pattern (applies to `t/12_protocol_rename.t`)

**Source:** `t/11_track_history.t` lines 36-145 (LMS stubs)
**Apply to:** `t/12_protocol_rename.t` if it needs to load any Perl module (not
needed for pure grep-based tests)

The grep-only subtests (PROTO-01 through PROTO-06 and Rust check) do not require
loading any LMS module — they only open source files as text and scan with regex.
The `t/03_convert_conf.t` pattern (open file, scan, assert) is sufficient.

If any subtest exercises runtime behavior (e.g., loading ProtocolHandler), use
the full stub set from `t/11_track_history.t` lines 36-318 verbatim.

---

## No Analog Found

All modified files have self-analogs (they exist in the codebase and are being
modified, not created). The test file has close analogs in `t/11_track_history.t`
and `t/03_convert_conf.t`.

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| — | — | — | All files have analogs |

---

## Metadata

**Analog search scope:** `Plugins/SpotOn/`, `librespot-spoton/src/`, `t/`
**Files scanned:** 9 source files + 2 test files
**Pattern extraction date:** 2026-06-05
