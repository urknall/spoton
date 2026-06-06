# Phase 11: Track History Metadata - Pattern Map

**Mapped:** 2026-06-04
**Files analyzed:** 5 (4 modified + 1 new test)
**Analogs found:** 5 / 5

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `Plugins/SpotOn/ProtocolHandler.pm` | protocol-handler | request-response + async callback | `Plugins/SpotOn/Connect.pm` `_fetchTrackMetadata` | role-match (same async getTrack + notify pattern) |
| `Plugins/SpotOn/Connect.pm` | service | event-driven + cache-write | `Plugins/SpotOn/Plugin.pm` `_trackItem` | exact (same cache set structure, same fields) |
| `Plugins/SpotOn/Plugin.pm` | service/menu | CRUD + cache-write | self (`_albumTrackItem`) | exact (TTL literal in two sibling functions) |
| `Plugins/SpotOn/DontStopTheMusic.pm` | service | batch + cache-write | `Plugins/SpotOn/Plugin.pm` `_trackItem` | exact (identical cache set structure) |
| `t/11_track_history.t` | test | unit | `t/10_stream_metadata.t` | exact (stub pattern, MockClient, grep-gate pattern) |

---

## Pattern Assignments

### `Plugins/SpotOn/ProtocolHandler.pm` (protocol-handler, request-response + async callback)

**Analog:** `Plugins/SpotOn/Connect.pm` — `_fetchTrackMetadata` (lines 798-878)

**Imports pattern** (lines 1-17 of ProtocolHandler.pm — already present, no new imports needed):
```perl
use Slim::Utils::Cache;
use Digest::MD5 qw(md5_hex);
# On-demand via require (already the pattern for Plugin and Helper):
require Plugins::SpotOn::API::Client;
require Plugins::SpotOn::Plugin;
```

**Package-level debounce hash** — new pattern, place after `my $cache` declaration (line 17):
```perl
my $cache = Slim::Utils::Cache->new();
# Phase 11 D-05: debounce hash — one in-flight refetch per URL
my %_pendingRefetch;
```

**Existing getMetadataFor core pattern** (lines 268-304 — the entire sub to be extended):
```perl
sub getMetadataFor {
    my ($class, $client, $url) = @_;

    # For Connect streams: try pluginData info first (set by Connect.pm _fetchTrackMetadata)
    if ($url && $url =~ m{spotify://connect-} && $client) {
        $client = $client->master if $client->can('master');
        my $song = $client->playingSong();
        if ($song && (my $info = $song->pluginData('info'))) {
            return $info;
        }
    }

    # Normalize: cache is keyed on spotify://track:ID but LMS may pass spotify:track:ID
    my $canonical = $url;
    if ($canonical && $canonical =~ m{^spotify:(?!//)}) {
        $canonical =~ s{^spotify:}{spotify://};
    }

    my $meta = $cache->get('spoton_meta_' . md5_hex($canonical));

    # Fallback: try original URL if normalization didn't help
    if (!$meta && $canonical ne $url) {
        $meta = $cache->get('spoton_meta_' . md5_hex($url));
    }

    return {} unless $meta;

    if ($client) {
        require Plugins::SpotOn::Plugin;
        return { %$meta,
            type    => Plugins::SpotOn::Plugin->_typeString($client, 'Browse'),
            bitrate => Plugins::SpotOn::Plugin->_bitrateForClient($client) . 'k',
        };
    }

    return $meta;
}
```
The `return {} unless $meta;` line (line 293) is the exact insertion point for the async re-fetch trigger.

**Async getTrack pattern** (Connect.pm lines 809-877 — copy and adapt):
```perl
# From Connect.pm _fetchTrackMetadata:
require Plugins::SpotOn::API::Client;
my $accountId = $prefs->client($client)->get('activeAccount')
             || $prefs->get('activeAccount')
             || '';

Plugins::SpotOn::API::Client->getTrack($accountId, $trackId, sub {
    my ($trackInfo) = @_;
    return unless $trackInfo && $trackInfo->{name};
    # ... process metadata ...
    Slim::Control::Request::notifyFromArray($client, ['newmetadata']);
});
```

**notifyFromArray pattern** (Connect.pm line 869 — exact pattern to copy):
```perl
Slim::Control::Request::notifyFromArray($client, ['newmetadata']);
```

**_largestImage helper** (Connect.pm lines 880-888 — already defined in ProtocolHandler.pm via Plugin.pm require, but same logic available):
```perl
sub _largestImage {
    my ($images) = @_;
    return '' unless ref $images eq 'ARRAY' && @{$images};
    my ($largest) = sort { ($b->{width} || 0) <=> ($a->{width} || 0) } @{$images};
    return $largest->{url} || '';
}
```

**Connect URL detection pattern** (ProtocolHandler.pm line 30 + line 272 — existing):
```perl
# URL type detection — already used in getFormatForURL and getMetadataFor:
$url =~ m{spotify://connect-}   # Connect stream
$url =~ m{spotify://track:([A-Za-z0-9]+)}  # Browse stream, captures ID
```

**undef-client guard pattern** (Connect.pm line 869 via surrounding context — must wrap notify):
```perl
if ($client) {
    Slim::Control::Request::notifyFromArray($client, ['newmetadata']);
}
```

**accountId resolution pattern** (Connect.pm lines 805-807 — exact pattern):
```perl
my $accountId = $prefs->client($client)->get('activeAccount')
             || $prefs->get('activeAccount')
             || '';
```

---

### `Plugins/SpotOn/Connect.pm` — `_fetchTrackMetadata` addition (service, event-driven + cache-write)

**Analog:** `Plugins/SpotOn/Plugin.pm` `_trackItem` (lines 397-407)

**Existing pluginData write** (Connect.pm lines 847-857 — the code immediately before the insertion point):
```perl
require Plugins::SpotOn::Plugin;
my $type_str = Plugins::SpotOn::Plugin->_typeString($client, 'Connect');
my $bitrate = Plugins::SpotOn::Plugin->_bitrateForClient($client);
$song->pluginData(info => {
    title        => $title,
    artist       => $artist,
    album        => $album,
    duration     => $duration,
    cover        => $cover,
    url          => $song->streamUrl,
    bitrate      => $bitrate . 'k',
    originalType => $type_str,
    type         => $type_str,
});
```

**Cache set pattern to add** — copy structure from Plugin.pm `_trackItem` lines 397-407, add `spotifyUri` field:
```perl
# From Plugin.pm _trackItem (the canonical cache-set pattern):
$cache->set('spoton_meta_' . md5_hex($spotify_url), {
    title    => $title,
    artist   => $artist,
    album    => $album,
    duration => $duration,
    cover    => $image,
    icon     => $image,
    bitrate  => __PACKAGE__->_bitrateForClient($client) . 'k',
    type     => __PACKAGE__->_typeString($client, 'Browse'),
}, 3600);   # Phase 11: change to 604800
```
For Connect.pm the adapted version adds `spotifyUri` and uses the connect-stream URL as key:
```perl
# Add AFTER $song->pluginData(info => {...}) in _fetchTrackMetadata:
my $connectUrl = $song->streamUrl || '';
if ($connectUrl) {
    $cache->set('spoton_meta_' . md5_hex($connectUrl), {
        title      => $title,
        artist     => $artist,
        album      => $album,
        duration   => $duration,
        cover      => $cover,
        icon       => $cover,
        bitrate    => $bitrate . 'k',
        type       => $type_str,
        spotifyUri => $trackInfo->{uri},  # D-01: e.g. "spotify:track:ABC123"
    }, 604800);  # D-02: 7 days
}
```
`md5_hex` is already imported via `use Digest::MD5 qw(md5_hex)` — check Connect.pm imports; if absent, add it.

**Existing $cache variable** — Connect.pm already has `use Slim::Utils::Cache` but check whether `my $cache = Slim::Utils::Cache->new()` is declared at package level or if `Slim::Utils::Cache->new()` is called inline. If only `use Slim::Utils::Cache` exists, use `Slim::Utils::Cache->new()->set(...)`.

---

### `Plugins/SpotOn/Plugin.pm` — `_trackItem` and `_albumTrackItem` TTL bump (service, CRUD)

**Analog:** self — both functions are in the same file, same TTL literal appears twice.

**`_trackItem` cache set** (lines 397-407 — change `3600` to `604800`):
```perl
$cache->set('spoton_meta_' . md5_hex($spotify_url), {
    title    => $title,
    artist   => $artist,
    album    => $album,
    duration => $duration,
    cover    => $image,
    icon     => $image,
    bitrate  => __PACKAGE__->_bitrateForClient($client) . 'k',
    type     => __PACKAGE__->_typeString($client, 'Browse'),
}, 3600);   # <-- change to 604800
```

**`_albumTrackItem` cache set** (lines 1136-1145 — change `3600` to `604800`):
```perl
$cache->set('spoton_meta_' . md5_hex($spotify_url), {
    title    => $title,
    artist   => $artists,
    album    => $albumName,
    duration => $duration,
    cover    => $image,
    icon     => $image,
    bitrate  => __PACKAGE__->_bitrateForClient($client) . 'k',
    type     => __PACKAGE__->_typeString($client, 'Browse'),
}, 3600);   # <-- change to 604800
```

Both are single-line changes: `3600` → `604800`.

---

### `Plugins/SpotOn/DontStopTheMusic.pm` — cache set TTL bump (service, batch)

**Analog:** `Plugins/SpotOn/Plugin.pm` `_trackItem` (same cache-set structure)

**Existing cache set** (DontStopTheMusic.pm lines 265-274 — change `3600` to `604800`):
```perl
$cache->set('spoton_meta_' . md5_hex($uri), {
    title    => $track->{name} // '',
    artist   => $artist,
    album    => $track->{album}{name} // '',
    duration => ($track->{duration_ms} || 0) / 1000,
    cover    => $image,
    icon     => $image,
    bitrate  => Plugins::SpotOn::Plugin->_bitrateForClient(undef) . 'k',
    type     => $type_str,
}, 3600);   # <-- change to 604800
```

Single-line change: `3600` → `604800`.

---

### `t/11_track_history.t` (test, unit)

**Analog:** `t/10_stream_metadata.t` — exact structural match

**Test file structure pattern** (lines 1-244 of 10_stream_metadata.t):

File layout to replicate:
1. `#!/usr/bin/perl` + `use strict; use warnings; use Test::More;`
2. Project root resolution via `File::Basename::dirname` + `Cwd::abs_path`
3. `write_stub()` helper (lines 21-30) — write stubs to tempdir
4. LMS module stubs (lines 36-240): `Slim::Utils::Log`, `Slim::Utils::Prefs`, `Slim::Utils::Cache`, `Slim::Utils::Timers`, `Slim::Utils::Strings`, `Slim::Formats::RemoteStream`, `Slim::Plugin::OPMLBased`, `Slim::Player::ProtocolHandlers`, `Slim::Player::TranscodingHelper`
5. `main::` constant injection via `BEGIN` block (lines 231-241)
6. `unshift @INC, $stub_dir, $project_dir` (line 244)
7. Module load: `require_ok('Plugins::SpotOn::ProtocolHandler')` — analogous to `require_ok('Plugins::SpotOn::Plugin')`

**Additional stubs needed for ProtocolHandler.pm** (not in 10_stream_metadata.t):
```perl
# Slim::Utils::Network — used in ProtocolHandler.pm
write_stub($stub_dir, 'Slim::Utils::Network', <<'END');
package Slim::Utils::Network;
sub blocking { }
sub AUTOLOAD { }
1;
END

# Slim::Utils::Versions
write_stub($stub_dir, 'Slim::Utils::Versions', <<'END');
package Slim::Utils::Versions;
sub compareVersions { 0 }
sub AUTOLOAD { }
1;
END

# Plugins::SpotOn::API::Client stub (for _asyncRefetch)
write_stub($stub_dir, 'Plugins::SpotOn::API::Client', <<'END');
package Plugins::SpotOn::API::Client;
our $mock_track;
sub getTrack {
    my ($class, $accountId, $trackId, $cb) = @_;
    $cb->($mock_track) if $cb;
}
1;
END
```

**MockClient pattern** (lines 259-264 — reuse exactly):
```perl
{
    package MockClient;
    use overload '""' => sub { ${$_[0]} };
    sub new { my $id = $_[1] // 'player1'; bless \$id, $_[0] }
    sub can { return $_[1] eq 'master' ? 0 : 0 }
}
```

**Stub cache with TTL inspection** (lines 119-131 — reuse exactly, includes `ttl()` method):
```perl
package Slim::Utils::Cache;
my %_store;
my %_ttl;
sub new    { bless {}, shift }
sub get    { $_store{$_[1]} }
sub set    { $_store{$_[1]} = $_[2]; $_ttl{$_[1]} = $_[3]; 1 }
sub remove { delete $_store{$_[1]}; delete $_ttl{$_[1]} }
sub ttl    { $_ttl{$_[1]} }
sub clear  { %_store = (); %_ttl = () }
1;
```
The `ttl()` method is essential for TTL-check test assertions.

**Grep-gate pattern** (lines 467-500 — reuse for TTL grep gate):
```perl
{
    my $file = "$project_dir/Plugins/SpotOn/Plugin.pm";
    open(my $fh, '<', $file) or die "Cannot read Plugin.pm: $!";
    my @matches;
    while (my $line = <$fh>) {
        chomp $line;
        next if $line =~ /^\s*#/;
        push @matches, $. if $line =~ /\},\s*3600\s*\)/;  # old TTL
    }
    close($fh);
    is(scalar @matches, 0,
        "Grep gate: no remaining TTL=3600 cache->set in Plugin.pm");
}
```
Apply same grep gate for `DontStopTheMusic.pm` and `Connect.pm`.

---

## Shared Patterns

### Cache key construction
**Source:** `Plugins/SpotOn/Plugin.pm` lines 397-398, `Plugins/SpotOn/ProtocolHandler.pm` lines 282-286
**Apply to:** All cache read/write operations in all four modified files
```perl
# Key pattern — consistent across all files:
'spoton_meta_' . md5_hex($url)
# where $url is the canonical spotify:// form (e.g., 'spotify://track:ABCDEF')
```

### require-on-demand pattern
**Source:** `Plugins/SpotOn/ProtocolHandler.pm` lines 48-49, Connect.pm lines 803, 844
**Apply to:** Any new `API::Client` or `Plugin` calls in ProtocolHandler.pm
```perl
require Plugins::SpotOn::API::Client;
require Plugins::SpotOn::Plugin;
```
This is the established pattern: `use` at top is avoided for cross-module requires to prevent circular load issues; `require` inline is the project convention.

### undef-client guard
**Source:** `Plugins/SpotOn/Connect.pm` line 827-828, `Plugins/SpotOn/Plugin.pm` `_bitrateForClient` line 1341
**Apply to:** All `$client->...` calls in `_asyncRefetch` and the Connect→Browse translation block
```perl
$client = $client->master if $client && $client->can('master');
# and before notify:
if ($client) { Slim::Control::Request::notifyFromArray($client, ['newmetadata']); }
```

### Metadata hash field set
**Source:** `Plugins/SpotOn/Plugin.pm` `_trackItem` lines 398-407
**Apply to:** All new `$cache->set('spoton_meta_...')` calls
```perl
{
    title    => $title,
    artist   => $artist,
    album    => $album,
    duration => $duration,
    cover    => $image,
    icon     => $image,       # icon mirrors cover — both needed by LMS
    bitrate  => ... . 'k',
    type     => ...,
}
```
The `icon` field mirrors `cover` — both must be present. Browse metadata uses `_bitrateForClient` + `_typeString`; Connect metadata passes already-resolved `$bitrate . 'k'` and `$type_str`.

### 7-day TTL constant
**Source:** Decision D-02 — replaces all `3600` occurrences
**Apply to:** Every `$cache->set('spoton_meta_...')` in all four files
```perl
604800   # 7 days — Phase 11 D-02 unified TTL
```

---

## No Analog Found

All files have close analogs in the codebase. No file requires patterns from RESEARCH.md alone.

---

## Metadata

**Analog search scope:** `Plugins/SpotOn/`, `t/`
**Files scanned:** 5 source files + 1 test file (full reads)
**Pattern extraction date:** 2026-06-04
