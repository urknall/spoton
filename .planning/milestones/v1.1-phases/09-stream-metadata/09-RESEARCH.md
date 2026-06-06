# Phase 9: Stream Metadata - Research

**Researched:** 2026-06-04
**Domain:** LMS metadata display, ProtocolHandler `type` field, per-player pref chain
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Format template is `{bitrate}, {format} (Spotify {mode})` — e.g. `320k, OGG (Spotify Browse)`. Bitrate leads, mode in parentheses.
- **D-02:** Mode label is ALWAYS present — even as a fallback, at minimum `(Spotify Browse)` or `(Spotify Connect)` appears.
- **D-03:** Format names use short form: `OGG`, `FLAC`, `MP3`, `PCM`. Not `OGG Vorbis` or `MPEG`.
- **D-04:** When bitrate is absent (guard), the string is `{format} (Spotify {mode})` — no leading comma or empty slot.
- **D-05:** Browse and Connect use the same format detection mechanism. The `streamFormat` per-player pref (`auto`/`ogg`/`flac`/`mp3`/`pcm`) determines the displayed format. Claude has discretion over the `auto` case.
- **D-06:** Connect.pm's hardcoded `'Ogg Vorbis (Spotify)'` must be replaced with the dynamic format string using the same logic as Browse.
- **D-07:** Always show the Spotify source bitrate (from `bitrate` pref, respecting per-player `bitrateOverride`).
- **D-08:** For MP3, show the Spotify source bitrate (not the LAME output bitrate).

### Claude's Discretion

- Format detection for the `auto` case: pick the most pragmatic approach. Options include pref-based inference (OGG if passthrough available, PCM otherwise) or LMS pipeline lookup if the API supports it.

### Deferred Ideas (OUT OF SCOPE)

None.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| META-01 | Songinfo shows "(Spotify Browse)" or "(Spotify Connect)" per active mode | `type` field in metadata hash returned by `getMetadataFor()` is consumed directly by LMS NowPlaying display |
| META-02 | Songinfo shows active stream format (OGG, FLAC, MP3, PCM) | `streamFormat` per-player pref drives both format display and pipeline selection; same pref already used in `formatOverride()` and `updateTranscodingTable()` |
| META-03 | Songinfo shows bitrate when available (e.g. "320k, OGG (Spotify Connect)") | Bitrate pref chain already implemented in `updateTranscodingTable()` lines 1221-1228; same logic reusable for type string |
</phase_requirements>

---

## Summary

Phase 9 is a pure metadata enrichment phase. The `type` field in the hash returned by `getMetadataFor()` is the single display string that LMS shows in Songinfo as the format/source line. Currently it is hardcoded to `'Spotify'` in Browse paths (`Plugin.pm` `_trackItem`, `_albumTrackItem`, `DontStopTheMusic.pm`) and to `'Ogg Vorbis (Spotify)'` in the Connect path (`Connect.pm` `_fetchTrackMetadata`).

The change is mechanically straightforward: extract a shared helper function `_buildTypeString($client)` that constructs the display string from the already-available pref chain, then call it from every location that writes a `type` field into a metadata hash. No new data sources, no new APIs, no UI changes.

The only design decision left to Claude (D-05 discretion) is the `auto` case: when `streamFormat` is `auto`, infer the effective format from `Helper->getCapability('passthrough')`. If passthrough is available, the effective format is `OGG` (librespot passes through Ogg Vorbis). If passthrough is absent, the effective format is `PCM` (librespot decodes to raw PCM and the `son-pcm` pipeline transcodes it). This matches exactly what `updateTranscodingTable()` does: the OGG-Passthrough Guard deletes `son-ogg` when passthrough is absent, leaving `son-pcm` as the only active pipeline.

**Primary recommendation:** Implement `_buildTypeString($client, $mode)` as a private sub in `Plugin.pm`, call it from `_trackItem`, `_albumTrackItem`, and `DontStopTheMusic.pm` with `mode => 'Browse'`, and call equivalent logic in `Connect.pm` `_fetchTrackMetadata` with `mode => 'Connect'`. Four call sites, one shared logic block, zero new dependencies.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Format/mode display string construction | Plugin.pm (Browse) / Connect.pm (Connect) | — | Each path sets its own metadata hash; shared helper sub extracted to Plugin.pm |
| Type string consumption (NowPlaying display) | ProtocolHandler.pm `getMetadataFor()` | — | Single consumer; returns hash as-is — no change needed here |
| Format inference for `auto` case | Plugin.pm `_buildTypeString` | Helper.pm `getCapability` | `getCapability('passthrough')` is the canonical passthrough flag |
| Bitrate resolution | Plugin.pm `_buildTypeString` | Prefs | Identical chain already in `updateTranscodingTable()` lines 1221-1228 |

---

## Standard Stack

No new packages. This phase uses only what is already imported in the modified files.

| Library | Purpose | Already Imported In |
|---------|---------|---------------------|
| `Slim::Utils::Prefs` | Read `streamFormat`, `bitrate`, `bitrateOverride` per-player prefs | Plugin.pm, Connect.pm |
| `Plugins::SpotOn::Helper` | `getCapability('passthrough')` for `auto` inference | Plugin.pm (already `require`d inside `updateTranscodingTable`) |

No installation step. No package legitimacy audit needed.

---

## Package Legitimacy Audit

Not applicable — phase installs no external packages.

---

## Architecture Patterns

### System Architecture Diagram

```
[LMS NowPlaying / Songinfo request]
        |
        v
ProtocolHandler::getMetadataFor($client, $url)
        |
        +-- spotify://connect-* URL?
        |       yes --> $song->pluginData('info') hash  <--- Connect.pm sets this
        |                                                     _fetchTrackMetadata()
        |                                                     type => _buildTypeString($client, 'Connect')
        |
        +-- no  --> $cache->get('spoton_meta_' . md5_hex($url))  <--- Plugin.pm/DSTM.pm sets this
                                                                       _trackItem() / _albumTrackItem() / DSTM
                                                                       type => _buildTypeString($client, 'Browse')
        |
        v
  { type => "320k, OGG (Spotify Browse)" }   <-- what LMS displays
```

### Data Flow: `_buildTypeString($client, $mode)`

```
$client, $mode ('Browse' | 'Connect')
        |
        v
1. Read bitrate:
   $bitrate = $prefs->client($client)->get('bitrateOverride')  [if valid 96/160/320]
            || $prefs->get('bitrate')
            || 320

2. Read streamFormat:
   $fmt_pref = $prefs->client($client)->get('streamFormat')
            || $prefs->client($client)->get('connectOggOverride')
            || 'auto'

3. Resolve effective format:
   if $fmt_pref eq 'auto':
     $fmt = Helper->getCapability('passthrough') ? 'OGG' : 'PCM'
   else:
     $fmt = uc($fmt_pref)   # 'ogg' -> 'OGG', 'flac' -> 'FLAC', etc.

4. Assemble string:
   "$bitrate_str, $fmt (Spotify $mode)"   [with bitrate]
   "$fmt (Spotify $mode)"                 [without bitrate — D-04 guard]
```

### Modification Sites

Four files need changes. Three Browse callers + one Connect caller:

```
Plugins/SpotOn/Plugin.pm
  _trackItem()          line ~405  — type => 'Spotify'
  _albumTrackItem()     line ~1143 — type => 'Spotify'
  (add _buildTypeString sub — can live near end of file, before `1;`)

Plugins/SpotOn/DontStopTheMusic.pm
  (anonymous sub)       line ~270  — type => 'Spotify'
  (require Plugin for _buildTypeString, or duplicate logic inline)

Plugins/SpotOn/Connect.pm
  _fetchTrackMetadata() line ~851  — type => 'Ogg Vorbis (Spotify)'
  (require Plugin for _buildTypeString, or duplicate logic inline)
```

### Recommended Project Structure

No new files. Changes are concentrated in existing files:

```
Plugins/SpotOn/
├── Plugin.pm           # Add _buildTypeString(); update _trackItem, _albumTrackItem
├── Connect.pm          # Update _fetchTrackMetadata type field
├── DontStopTheMusic.pm # Update cache-set type field
└── ProtocolHandler.pm  # No change — already a pass-through consumer
```

### Pattern: Shared Helper via `require Plugin`

Both `Connect.pm` and `DontStopTheMusic.pm` already `require Plugins::SpotOn::Plugin` or are called from within the Plugin namespace. The cleanest approach is to define `_buildTypeString` as a package sub in `Plugin.pm` and call it as `Plugins::SpotOn::Plugin::_buildTypeString($client, $mode)` from the other files. The `require` in `_fetchTrackMetadata` already exists for other purposes.

Alternatively, the helper can be duplicated in each file (three lines of logic, low duplication risk), but a single function is cleaner given it will be called from at least three distinct files.

### Anti-Patterns to Avoid

- **Hard-coding format in Connect path:** D-06 explicitly forbids leaving `'Ogg Vorbis (Spotify)'`. Always call `_buildTypeString`.
- **Missing `auto` inference:** Leaving `auto` as the literal display string. Must resolve to either `OGG` or `PCM` via `getCapability('passthrough')`.
- **Client undef guard missing:** `_buildTypeString` must handle `$client = undef` gracefully (e.g., during scanner context or when metadata is cached without a live player). Fall back to global `$prefs->get('bitrate')` and `auto` resolution without client prefs.
- **Overwriting DontStopTheMusic.pm's type with stale client:** `DontStopTheMusic.pm` caches metadata for multiple tracks before playback. At cache-write time, `$client` is available (it is passed in). Use it.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Format detection | Custom pipeline inspection | `$prefs->client($client)->get('streamFormat')` + `getCapability('passthrough')` | The pref is the canonical user intent; `getCapability` is the canonical capability flag |
| Bitrate reading | Re-implement bitrate chain | Copy the 4-line chain from `updateTranscodingTable()` lines 1221-1228 | Already validated; handles `bitrateOverride` correctly |
| Mode detection | URL-parse or daemon query | `Connect::isSpotifyConnect($client)` (caller passes `$mode`) | Mode is known at metadata-write time; pass it as a parameter |

**Key insight:** All data sources are already available at the point where `type` is set. No new lookups, no new async calls, no new state.

---

## Common Pitfalls

### Pitfall 1: `$client` Not Available at Browse Cache-Write Time
**What goes wrong:** `_buildTypeString` called with `undef` client; `$prefs->client(undef)` throws or returns wrong namespace.
**Why it happens:** `_albumTrackItem` is sometimes called during scan context or menu rendering without a live `$client`.
**How to avoid:** Guard with `if ($client)` before reading client prefs; fall back to `$prefs->get('bitrate')` (global) and `$prefs->get('streamFormat')` (global default, if it exists) or literal `'auto'` resolution.
**Warning signs:** Test passes on desktop but fails on first-boot scan.

### Pitfall 2: `auto` Displayed Literally
**What goes wrong:** `streamFormat` is `'auto'` → `_buildTypeString` returns `"320k, AUTO (Spotify Browse)"`.
**Why it happens:** Forgetting the `auto` → `OGG`/`PCM` resolution step.
**How to avoid:** Explicit `if ($fmt_pref eq 'auto') { $fmt = Helper->getCapability('passthrough') ? 'OGG' : 'PCM' }`.
**Warning signs:** Songinfo shows `AUTO` instead of `OGG` or `PCM`.

### Pitfall 3: TTL Mismatch Serving Stale Type String
**What goes wrong:** User changes `streamFormat` pref, but the 3600s cache still shows the old format string in Songinfo.
**Why it happens:** `_trackItem` caches metadata for 3600s. The type string is baked in at cache-write time.
**How to avoid:** This is acceptable per the architecture — the type string reflects the format at the time the track was queued, not the current pref. Document this as expected. If cache invalidation on pref change is desired, it is out of scope for Phase 9 (no such requirement in META-01..03). [ASSUMED: no invalidation requirement]
**Warning signs:** User changes format mid-session and sees old label. Document as "takes effect on next track".

### Pitfall 4: `DontStopTheMusic.pm` Has Its Own Cache-Set
**What goes wrong:** DSTM tracks display `type => 'Spotify'` (old hardcoded value) while Browse/Connect tracks display the enriched string.
**Why it happens:** DSTM.pm has its own `$cache->set(...)` call at line 270. It is not called from `_trackItem` or `_albumTrackItem`.
**How to avoid:** Explicitly update DontStopTheMusic.pm as a fourth modification site. Do not assume it shares logic with Plugin.pm.
**Warning signs:** Grep for `type.*Spotify` after implementation — should return zero matches.

### Pitfall 5: Connect Metadata Updated Per Track; Browse Cached Per Track
**What goes wrong:** For Connect, `_fetchTrackMetadata` is called on every new track — the `$client` is always live. For Browse, metadata is cached at menu-render time (before playback). This asymmetry means Browse uses the format pref at menu-render time, Connect uses the format pref at playback time.
**Why it happens:** Architectural difference between Browse (pre-cache at render) and Connect (live update at playback).
**How to avoid:** Accept asymmetry. Document in code comments. Connect will always show current pref; Browse will show pref at last render.

---

## Code Examples

Verified patterns from the actual codebase:

### Bitrate Pref Chain (from updateTranscodingTable, Plugin.pm lines 1221-1228)
```perl
# [VERIFIED: codebase grep]
my $bitrate = $prefs->get('bitrate') || 320;
if ($client) {
    my $override = $prefs->client($client)->get('bitrateOverride');
    $bitrate = $override if $override && $override =~ /^(?:96|160|320)$/;
}
```

### streamFormat Pref Read with Migration Fallback (from formatOverride, ProtocolHandler.pm lines 53-57)
```perl
# [VERIFIED: codebase grep]
my $fmt = $client
    ? ($prefs->client($client)->get('streamFormat')
       || $prefs->client($client)->get('connectOggOverride')
       || 'auto')
    : 'auto';
```

### Passthrough Capability Check (Helper.pm lines 104-108)
```perl
# [VERIFIED: codebase grep]
sub getCapability {
    my ($class, $key) = @_;
    return $helperCapabilities->{$key} if $helperCapabilities && defined $helperCapabilities->{$key};
    return undef;
}
# Usage: Plugins::SpotOn::Helper->getCapability('passthrough')
```

### Proposed `_buildTypeString` (pseudocode — planner turns into real Perl)
```perl
# Parameters: $client (may be undef), $mode ('Browse' | 'Connect')
# Returns: display string for the `type` metadata field
sub _buildTypeString {
    my ($client, $mode) = @_;

    # Bitrate chain
    my $bitrate = $prefs->get('bitrate') || 320;
    if ($client) {
        my $override = $prefs->client($client)->get('bitrateOverride');
        $bitrate = $override if $override && $override =~ /^(?:96|160|320)$/;
    }
    my $bitrate_str = $bitrate . 'k';

    # streamFormat chain (with migration fallback)
    my $fmt_pref = $client
        ? ($prefs->client($client)->get('streamFormat')
           || $prefs->client($client)->get('connectOggOverride')
           || 'auto')
        : 'auto';

    # Resolve auto: OGG if passthrough available, else PCM
    my $fmt;
    if ($fmt_pref eq 'auto') {
        require Plugins::SpotOn::Helper;
        $fmt = Plugins::SpotOn::Helper->getCapability('passthrough') ? 'OGG' : 'PCM';
    } else {
        $fmt = uc($fmt_pref);   # 'ogg' -> 'OGG', 'flac' -> 'FLAC', etc.
    }

    # Assemble per D-01/D-04
    return "$bitrate_str, $fmt (Spotify $mode)";
    # D-04 guard (if bitrate ever absent): return "$fmt (Spotify $mode)"
}
```

### Existing metadata cache-set in `_trackItem` (lines 397-406) — to be updated
```perl
# [VERIFIED: codebase grep]
$cache->set('spoton_meta_' . md5_hex($spotify_url), {
    title    => $title,
    artist   => $artist,
    album    => $album,
    duration => $duration,
    cover    => $image,
    icon     => $image,
    bitrate  => ($prefs->get('bitrate') || 320) . 'k',
    type     => 'Spotify',   # <-- replace with _buildTypeString($client, 'Browse')
}, 3600);
```

### Existing pluginData set in `_fetchTrackMetadata` (Connect.pm lines 844-853) — to be updated
```perl
# [VERIFIED: codebase grep]
$song->pluginData(info => {
    title        => $title,
    artist       => $artist,
    album        => $album,
    duration     => $duration,
    cover        => $cover,
    url          => $song->streamUrl,
    originalType => 'Ogg Vorbis (Spotify)',   # <-- replace with _buildTypeString($client, 'Connect')
    type         => 'Ogg Vorbis (Spotify)',   # <-- replace with _buildTypeString($client, 'Connect')
});
```

---

## State of the Art

No external API changes relevant. This phase is entirely internal to the plugin.

| Old Value | New Value | Scope |
|-----------|-----------|-------|
| `'Spotify'` | `"320k, OGG (Spotify Browse)"` | `_trackItem`, `_albumTrackItem`, `DontStopTheMusic.pm` |
| `'Ogg Vorbis (Spotify)'` | `"320k, OGG (Spotify Connect)"` | `Connect.pm _fetchTrackMetadata` |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | No cache-invalidation requirement on pref change — "takes effect on next track" is acceptable | Common Pitfalls #3 | If user expects immediate update, Pitfall #3 becomes a bug rather than accepted behavior |
| A2 | `uc($fmt_pref)` correctly maps all pref values: `'ogg'`→`'OGG'`, `'flac'`→`'FLAC'`, `'mp3'`→`'MP3'`, `'pcm'`→`'PCM'` | Code Examples | If pref values ever contain unexpected strings, `uc()` will produce garbled output |

---

## Open Questions (RESOLVED)

1. **`originalType` field in Connect pluginData**
   - What we know: `_fetchTrackMetadata` sets both `originalType` and `type` to the same value.
   - What's unclear: `originalType` is set but never read by `getMetadataFor` (which just returns the hash as-is). LMS may or may not use `originalType` internally.
   - RESOLVED: Update both `type` and `originalType` to the dynamic string. No harm in keeping them consistent. If LMS uses `originalType` for anything, it should also get the enriched value. Plan 09-01 Task 2 Step D explicitly covers both fields.

2. **Connect `$client` availability at `_fetchTrackMetadata` call site**
   - What we know: `_fetchTrackMetadata` receives `$client` and `$song` as parameters (visible at line ~800 in Connect.pm). The function is always called with a live player context.
   - What's unclear: Whether `$client->master` needs to be called before passing to `_typeString`.
   - RESOLVED: Apply `$client = $client->master if $client && $client->can('master')` as a guard inside `_typeString`, consistent with ProtocolHandler.pm `getMetadataFor` and all Connect.pm pref-read patterns. Plan 09-01 Task 2 Step A includes this guard.

---

## Environment Availability

Step 2.6: SKIPPED — Phase 9 is code-only changes. No external tools, services, CLIs, or runtimes required beyond the existing Perl test infrastructure.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Test::More (built-in Perl, confirmed via t/09_settings.t) |
| Config file | none — tests run directly via `perl t/NN_*.t` |
| Quick run command | `perl t/10_stream_metadata.t` |
| Full suite command | `prove -l t/` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| META-01 | Browse mode produces `(Spotify Browse)` in type string | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-01 | Connect mode produces `(Spotify Connect)` in type string | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-02 | `streamFormat=ogg` → type string contains `OGG` | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-02 | `streamFormat=flac` → type string contains `FLAC` | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-02 | `streamFormat=mp3` → type string contains `MP3` | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-02 | `streamFormat=pcm` → type string contains `PCM` | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-02 | `streamFormat=auto` + passthrough=1 → `OGG` | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-02 | `streamFormat=auto` + passthrough=0 → `PCM` | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-03 | Bitrate 320 + format + mode → `"320k, OGG (Spotify Browse)"` | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-03 | bitrateOverride 160 overrides global 320 | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-03 | `$client=undef` → falls back to global bitrate, no crash | unit | `perl t/10_stream_metadata.t` | Wave 0 |
| META-01..03 | No `'Spotify'` literal remains in metadata cache-sets | static (grep) | `perl t/05_perl_syntax.t` + grep | Wave 0 |
| META-01..03 | No `'Ogg Vorbis (Spotify)'` literal remains | static (grep) | `perl t/10_stream_metadata.t` | Wave 0 |

### Sampling Rate
- **Per task commit:** `perl t/10_stream_metadata.t && perl t/05_perl_syntax.t`
- **Per wave merge:** `prove -l t/`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `t/10_stream_metadata.t` — covers all META-01..03 cases listed above. Must be created in Wave 0 before implementation. The test infrastructure pattern from `t/09_settings.t` (stub-dir + write_stub) is the established model — copy and adapt.

---

## Security Domain

No security surface. Phase 9 only modifies display strings derived from prefs data that the user themselves configured. No external input, no network calls, no filesystem writes, no auth.

ASVS does not apply to this phase. `security_enforcement` remains enabled globally but no ASVS categories are relevant to a string-formatting change over internal prefs.

---

## Sources

### Primary (HIGH confidence)
- Codebase grep + direct file read — `ProtocolHandler.pm`, `Plugin.pm`, `Connect.pm`, `Helper.pm`, `DontStopTheMusic.pm` — all claims tagged `[VERIFIED: codebase grep]`
- `CONTEXT.md` — decisions D-01 through D-08 verbatim
- `t/09_settings.t` — test infrastructure pattern confirmed

### Secondary (MEDIUM confidence)
- `CLAUDE.md` §LMS Plugin API Modules — `Slim::Utils::Cache`, `Slim::Utils::Prefs` patterns confirmed match codebase

### Tertiary (LOW confidence)
- A1, A2 in Assumptions Log — design inferences, not verified against LMS source

---

## Metadata

**Confidence breakdown:**
- Modification sites: HIGH — all four sites identified via direct file read
- `_buildTypeString` design: HIGH — all data sources confirmed present at call sites
- `auto` inference logic: HIGH — matches exactly the OGG-Passthrough Guard in `updateTranscodingTable`
- Test infrastructure: HIGH — `prove -l t/` + Test::More confirmed via existing tests

**Research date:** 2026-06-04
**Valid until:** 2026-07-04 (stable domain — no external APIs involved)
