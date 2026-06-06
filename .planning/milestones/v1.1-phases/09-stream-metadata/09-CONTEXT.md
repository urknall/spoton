# Phase 9: Stream Metadata - Context

**Gathered:** 2026-06-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Enrich the Songinfo display for playing Spotify tracks to show the active playback mode (Browse vs Connect), stream format, and bitrate. The `type` field in metadata returned by `getMetadataFor` and stored in pluginData/cache is the primary integration point.

</domain>

<decisions>
## Implementation Decisions

### Display String Template
- **D-01:** Format template is `{bitrate}, {format} (Spotify {mode})` — e.g. `320k, OGG (Spotify Browse)`. Bitrate leads, mode in parentheses.
- **D-02:** Mode label is ALWAYS present — even as a fallback, at minimum `(Spotify Browse)` or `(Spotify Connect)` appears. This ensures META-01 is always satisfied.
- **D-03:** Format names use short form: `OGG`, `FLAC`, `MP3`, `PCM`. Not `OGG Vorbis` or `MPEG`.
- **D-04:** When bitrate is absent (shouldn't happen with D-06, but as a guard), the string is `{format} (Spotify {mode})` — no leading comma or empty slot.

### Format Detection
- **D-05:** Browse and Connect use the same format detection mechanism. The `streamFormat` per-player pref (`auto`/`ogg`/`flac`/`mp3`/`pcm`) determines the displayed format. Claude has discretion over the exact implementation for the `auto` case — choose the most pragmatic approach based on LMS API availability.
- **D-06:** Connect.pm's hardcoded `'Ogg Vorbis (Spotify)'` must be replaced with the dynamic format string using the same logic as Browse.

### Bitrate Source
- **D-07:** Always show the Spotify source bitrate (from the `bitrate` pref, respecting per-player override). This is what the user configured and what Spotify delivers. Consistent across all formats.
- **D-08:** For MP3, show the Spotify source bitrate (not the LAME output bitrate). Consistency over accuracy — the user sees "what Spotify delivered" in every format.

### Claude's Discretion
- Format detection for the `auto` case: Claude picks the most pragmatic approach. Options include pref-based inference (OGG if passthrough available, PCM otherwise) or LMS pipeline lookup if the API supports it.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/REQUIREMENTS.md` — META-01, META-02, META-03 requirement definitions
- `CLAUDE.md` §Protocol Handler Pattern — formatOverride, canDirectStream patterns
- `CLAUDE.md` §Transcoding — custom-convert.conf pipeline structure (son-*, soc-*)

### Source Files (primary modification targets)
- `Plugins/SpotOn/Plugin.pm` — `_trackItem()` (line ~397) and `_albumTrackItem()` (line ~1135) set Browse metadata cache with `type => 'Spotify'`. Both need the enriched type string.
- `Plugins/SpotOn/Connect.pm` — `_fetchTrackMetadata()` (line ~844) sets Connect metadata via pluginData with hardcoded `type => 'Ogg Vorbis (Spotify)'`. Must be replaced with dynamic format string.
- `Plugins/SpotOn/ProtocolHandler.pm` — `formatOverride()` (line ~42) and `getMetadataFor()` (line ~268) — understand the format selection and metadata return flow.
- `Plugins/SpotOn/Plugin.pm` — `updateTranscodingTable()` (line ~1218) — understand how `streamFormat` pref controls pipeline deletion and how the OGG Passthrough Guard works.

### Reference Implementation
- Qobuz ProtocolHandler: `https://github.com/LMS-Community/plugin-Qobuz/blob/master/ProtocolHandler.pm` — How another LMS plugin handles metadata/type display

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `$prefs->client($client)->get('streamFormat')` — per-player format pref, already used by formatOverride and updateTranscodingTable. Central source of truth for format selection.
- `$prefs->get('bitrate')` and per-player `bitrateOverride` — bitrate pref chain already implemented in updateTranscodingTable (line 1221-1228).
- `Helper->getCapability('passthrough')` — binary capability check, used by OGG Passthrough Guard. Determines if OGG is a viable format.
- `Connect::isConnectActive($client)` — returns true if player is in Connect mode. Clean mode detection.

### Established Patterns
- Metadata cache: `spoton_meta_` + md5_hex(url) with TTL 3600s — Browse mode metadata storage.
- pluginData('info') — Connect mode metadata storage, set by _fetchTrackMetadata, read by getMetadataFor.
- Both paths flow through `getMetadataFor()` in ProtocolHandler.pm — single point where LMS reads metadata.

### Integration Points
- `getMetadataFor()` is the sole consumer of metadata for NowPlaying display — changing the `type` field in the data it returns is all that's needed for Songinfo.
- No Settings UI changes needed — all data comes from existing prefs.
- No i18n needed — format names (OGG, FLAC, etc.) and mode names (Spotify Browse/Connect) are technical labels, not user-translatable strings.

</code_context>

<specifics>
## Specific Ideas

No specific requirements — the display template and data sources are fully defined by the decisions above.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 9-Stream Metadata*
*Context gathered: 2026-06-04*
