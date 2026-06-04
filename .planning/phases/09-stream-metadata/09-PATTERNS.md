# Phase 9: Stream Metadata - Pattern Map

**Mapped:** 2026-06-04
**Files analyzed:** 3 (all modifications, no new files)
**Analogs found:** 3 / 3 (self-analogs — each file modifies its own existing pattern)

## File Classification

| Modified File | Role | Data Flow | Closest Analog | Match Quality |
|---------------|------|-----------|----------------|---------------|
| `Plugins/SpotOn/Plugin.pm` | service/controller | request-response | self (lines 397-406, 1135-1144) | exact — modifying existing cache-set blocks |
| `Plugins/SpotOn/Connect.pm` | service | event-driven | self (lines 844-853) | exact — modifying existing pluginData-set block |
| `Plugins/SpotOn/ProtocolHandler.pm` | protocol-handler | request-response | self (lines 268-281) | read-only reference — getMetadataFor already correct |

---

## Pattern Assignments

### `Plugins/SpotOn/Plugin.pm` — `_trackItem()` and `_albumTrackItem()`

**Role:** service (metadata cache writer)
**Data Flow:** request-response (called during OPML feed build)
**Change:** Replace `type => 'Spotify'` with the dynamic format+mode string in both cache-set blocks.

**Current code — `_trackItem()` cache block** (lines 397-406):
```perl
$cache->set('spoton_meta_' . md5_hex($spotify_url), {
    title    => $title,
    artist   => $artist,
    album    => $album,
    duration => $duration,
    cover    => $image,
    icon     => $image,
    bitrate  => ($prefs->get('bitrate') || 320) . 'k',
    type     => 'Spotify',                      # <-- replace this
}, 3600);
```

**Current code — `_albumTrackItem()` cache block** (lines 1135-1144):
```perl
$cache->set('spoton_meta_' . md5_hex($spotify_url), {
    title    => $title,
    artist   => $artists,
    album    => $albumName,
    duration => $duration,
    cover    => $image,
    icon     => $image,
    bitrate  => ($prefs->get('bitrate') || 320) . 'k',
    type     => 'Spotify',                      # <-- replace this
}, 3600);
```

**Bitrate pref chain to copy** (lines 1221-1228 of `updateTranscodingTable`):
```perl
my $bitrate = $prefs->get('bitrate') || 320;
if ($client) {
    my $override = $prefs->client($client)->get('bitrateOverride');
    $bitrate = $override if $override && $override =~ /^(?:96|160|320)$/;
}
```
Both cache blocks already use `$prefs->get('bitrate') || 320` — per D-07 per-player override is also needed. Apply the same two-step chain: global first, then client override when `$client` is available (it is — both subs receive `$client` as first arg).

**streamFormat pref read pattern** (from `formatOverride`, lines 53-57, and `updateTranscodingTable`, lines 1307-1309):
```perl
my $fmt = $client
    ? ($prefs->client($client)->get('streamFormat')
       || $prefs->client($client)->get('connectOggOverride')
       || 'auto')
    : 'auto';
```
Use this exact pattern in the helper sub (see Shared Patterns below).

**passthrough capability check** (lines 1298-1302 of `updateTranscodingTable`):
```perl
require Plugins::SpotOn::Helper;
unless (Plugins::SpotOn::Helper->getCapability('passthrough')) {
    delete $commandTable->{'son-ogg-*-*'};
    ...
}
```
For the `auto` format case (D-05 discretion): map `auto` to `OGG` when `getCapability('passthrough')` is true, otherwise `PCM`. This mirrors the passthrough guard logic already in Plugin.pm.

**Format name map** (from D-03 decisions):
```perl
my %FORMAT_LABEL = (
    ogg  => 'OGG',
    flac => 'FLAC',
    mp3  => 'MP3',
    pcm  => 'PCM',
);
# auto resolution:
#   passthrough available => 'OGG'
#   passthrough unavailable => 'PCM'
```

**Display string assembly** (from D-01, D-02, D-04):
```perl
# $bitrate  = integer (e.g. 320)
# $fmtLabel = 'OGG' | 'FLAC' | 'MP3' | 'PCM'
# $mode     = 'Browse' | 'Connect'
my $type_str = $bitrate
    ? "${bitrate}k, ${fmtLabel} (Spotify ${mode})"
    : "${fmtLabel} (Spotify ${mode})";
```

**Connect mode detection** — Browse mode: `_trackItem` and `_albumTrackItem` are only called during OPML feed navigation, never during active Connect playback. Mode is always `'Browse'` here. No runtime check needed for these two call sites.

---

### `Plugins/SpotOn/Connect.pm` — `_fetchTrackMetadata()`

**Role:** service (event-driven metadata writer)
**Data Flow:** event-driven (callback from Spotify Web API response)
**Change:** Replace hardcoded `type => 'Ogg Vorbis (Spotify)'` with dynamic format string. Mode is always `'Connect'` here.

**Current code to replace** (lines 844-853):
```perl
$song->pluginData(info => {
    title        => $title,
    artist       => $artist,
    album        => $album,
    duration     => $duration,
    cover        => $cover,
    url          => $song->streamUrl,
    originalType => 'Ogg Vorbis (Spotify)',     # <-- replace this
    type         => 'Ogg Vorbis (Spotify)',     # <-- replace this
});
```

**`$client` is available** in `_fetchTrackMetadata` (line 798: `my ($client, $trackId) = @_;`) — per-player bitrate override and streamFormat pref are both accessible.

**streamFormat read pattern** already used in Connect.pm's caller context (same pref namespace):
```perl
my $fmt = $prefs->client($client)->get('streamFormat')
       || $prefs->client($client)->get('connectOggOverride')
       || 'auto';
```

**Bitrate in Connect.pm** — Connect.pm does not currently read the `bitrate` pref. For `_fetchTrackMetadata`, apply the same two-step chain as Browse:
```perl
my $bitrate = $prefs->get('bitrate') || 320;
my $override = $prefs->client($client)->get('bitrateOverride');
$bitrate = $override if $override && $override =~ /^(?:96|160|320)$/;
```

**Mode is always `'Connect'`** for this call site — no runtime check required.

---

### `Plugins/SpotOn/ProtocolHandler.pm` — `getMetadataFor()` (read-only reference)

**Role:** protocol-handler (metadata reader/dispatcher)
**Data Flow:** request-response (LMS calls this on each NowPlaying refresh)
**Change:** None required. `getMetadataFor` is a pass-through; it returns whatever is stored in `pluginData('info')` (Connect) or the cache (Browse). Fixing the stored values in Plugin.pm and Connect.pm is sufficient.

**Current read path** (lines 268-281) — provided for planner orientation only:
```perl
sub getMetadataFor {
    my ($class, $client, $url) = @_;

    # Connect: pluginData set by Connect.pm _fetchTrackMetadata
    if ($url && $url =~ m{spotify://connect-} && $client) {
        $client = $client->master if $client->can('master');
        my $song = $client->playingSong();
        if ($song && (my $info = $song->pluginData('info'))) {
            return $info;
        }
    }

    # Browse: cache set by Plugin.pm _trackItem/_albumTrackItem
    return $cache->get('spoton_meta_' . md5_hex($url)) || {};
}
```

---

## Shared Patterns

### Format String Helper Sub

Both Plugin.pm and Connect.pm need the same format-label + display-string logic. Implement once as a private helper sub in Plugin.pm (since `updateTranscodingTable` already lives there and holds the streamFormat knowledge), and `require Plugins::SpotOn::Plugin` on-demand in Connect.pm (the same on-demand require pattern already used by ProtocolHandler.pm line 48).

**Pattern to follow for the helper** (mirrors existing pref-read and capability-check patterns):
```perl
# _typeString($client, $mode)
# Returns the display string for the 'type' metadata field.
# $mode: 'Browse' or 'Connect'
sub _typeString {
    my ($class, $client, $mode) = @_;

    # Bitrate: global pref, with per-player override (D-07, mirrors updateTranscodingTable)
    my $bitrate = $prefs->get('bitrate') || 320;
    if ($client) {
        my $override = $prefs->client($client)->get('bitrateOverride');
        $bitrate = $override if $override && $override =~ /^(?:96|160|320)$/;
    }

    # Format: per-player pref with migration fallback (mirrors formatOverride / updateTranscodingTable)
    my $fmt = $client
        ? ($prefs->client($client)->get('streamFormat')
           || $prefs->client($client)->get('connectOggOverride')
           || 'auto')
        : 'auto';

    # Resolve 'auto': OGG if binary has passthrough, else PCM (D-05 discretion)
    if ($fmt eq 'auto') {
        require Plugins::SpotOn::Helper;
        $fmt = Plugins::SpotOn::Helper->getCapability('passthrough') ? 'ogg' : 'pcm';
    }

    my %LABEL = (ogg => 'OGG', flac => 'FLAC', mp3 => 'MP3', pcm => 'PCM');
    my $fmtLabel = $LABEL{$fmt} || 'OGG';

    # D-01 / D-04: bitrate leads, mode in parens; omit bitrate slot if absent
    return $bitrate
        ? "${bitrate}k, ${fmtLabel} (Spotify ${mode})"
        : "${fmtLabel} (Spotify ${mode})";
}
```

**Placement:** Add to `Plugins::SpotOn::Plugin` (already has all required prefs and requires). Call from both `_trackItem` and `_albumTrackItem` in Plugin.pm. In Connect.pm's `_fetchTrackMetadata`, call as:
```perl
require Plugins::SpotOn::Plugin;
my $type_str = Plugins::SpotOn::Plugin->_typeString($client, 'Connect');
```

### Pref Read Pattern (all three files)

**Source:** `Plugins/SpotOn/Plugin.pm` lines 1221-1228, `Plugins/SpotOn/ProtocolHandler.pm` lines 53-57
**Apply to:** `_typeString` helper, both Browse cache-set blocks

```perl
# Global pref
my $prefs = preferences('plugin.spoton');   # module-level, already declared in each file

# Per-player with migration fallback
my $fmt = $prefs->client($client)->get('streamFormat')
       || $prefs->client($client)->get('connectOggOverride')
       || 'auto';

# Per-player bitrate override
my $bitrate = $prefs->get('bitrate') || 320;
my $override = $prefs->client($client)->get('bitrateOverride');
$bitrate = $override if $override && $override =~ /^(?:96|160|320)$/;
```

### On-Demand Require Pattern

**Source:** `Plugins/SpotOn/ProtocolHandler.pm` line 48
**Apply to:** Connect.pm calling Plugin.pm's `_typeString`

```perl
require Plugins::SpotOn::Plugin;
Plugins::SpotOn::Plugin->updateTranscodingTable($client);
```
Mirror this with:
```perl
require Plugins::SpotOn::Plugin;
my $type_str = Plugins::SpotOn::Plugin->_typeString($client, 'Connect');
```

---

## No Analog Found

None — all three files are self-analogs with clear existing patterns to extend.

---

## Metadata

**Analog search scope:** `Plugins/SpotOn/` (Plugin.pm, Connect.pm, ProtocolHandler.pm, Helper.pm)
**Files scanned:** 4
**Pattern extraction date:** 2026-06-04
