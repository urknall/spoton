# Phase 11: Track History Metadata - Context

**Gathered:** 2026-06-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Metadata-Persistenz für den LMS Track-Verlauf. Browse-Tracks behalten Artwork, Format und Bitrate auch nach Cache-Expiry. Connect-Tracks werden transparent zu abspielbaren Browse-URLs übersetzt. Der zentrale Integrationspunkt ist `getMetadataFor` in ProtocolHandler.pm, das bei Cache-Miss einen async Re-Fetch auslöst statt `{}` zurückzugeben.

</domain>

<decisions>
## Implementation Decisions

### Connect-Track Identität
- **D-01:** Connect-Metadata wird im selben `spoton_meta_` Cache persistiert wie Browse-Metadata. `_fetchTrackMetadata` in Connect.pm speichert den Datensatz zusätzlich unter `spoton_meta_` + md5(connect-URL) — enthält cover, title, artist, album, duration, bitrate, type UND die echte `spotify:track:ID` als `spotifyUri` Feld.
- **D-02:** TTL für alle Track-Metadata (Browse + Connect) einheitlich 604800s (7 Tage). Ersetzt den bisherigen 3600s TTL. Spotify Artwork-URLs halten deutlich länger als 7 Tage.

### Cache-Miss Re-Fetch
- **D-03:** Async Re-Fetch mit Placeholder. `getMetadataFor` gibt bei Cache-Miss sofort Minimal-Metadata zurück (generisches Icon, Title aus URL wenn parsbar), startet einen async API-Call via `SimpleAsyncHTTP`, cached das Ergebnis, und feuert `Slim::Control::Request::notifyFromArray($client, ['newmetadata'])` damit LMS die Anzeige aktualisiert.
- **D-04:** Connect-Cache-Einträge enthalten ein `spotifyUri` Feld (z.B. `spotify:track:ABC123`) für Re-Fetch nach Expiry. Bei Browse-URLs wird die Track-ID direkt aus dem URL extrahiert (`spotify://track:ID`).
- **D-05:** Ein laufender Re-Fetch pro Track-URL — Debounce via Package-Hash (`%_pendingRefetch`). Verhindert Doppel-Fetches bei schnellem History-Durchblättern und schont das Spotify Rate-Limit.

### Connect→Browse Replay
- **D-06:** Connect-Tracks in der History werden transparent zu Browse-URLs übersetzt. Wenn `getMetadataFor` für eine `spotify://connect-*` URL aufgerufen wird und ein `spotifyUri` im Cache liegt, wird die abspielbare Browse-URL verfügbar gemacht.
- **D-07:** Übersetzung ist unsichtbar — kein visueller Unterschied zwischen ehemaligen Connect- und Browse-Tracks in der History. Type-String wird nicht als "Connect" markiert.

### Claude's Discretion
- Technische Integration der URL-Translation (via getMetadataFor Redirect, canDirectStream, oder anderer LMS-Mechanismus) — Hauptsache Connect-Tracks aus der History sind abspielbar via Browse-Pipeline
- Placeholder-Inhalt bei Cache-Miss (Title-Parsing aus URL, generisches Icon-Wahl)
- Exakte Struktur des `%_pendingRefetch` Debounce-Hash
- Ob der Browse-Cache-TTL-Bump (3600s → 604800s) auch DontStopTheMusic.pm betrifft

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/REQUIREMENTS.md` — HIST-01 through HIST-04 requirement definitions (referenced in ROADMAP.md success criteria)
- `CLAUDE.md` §Protocol Handler Pattern — formatOverride, canDirectStream, getMetadataFor patterns
- `.planning/PROJECT.md` — Core Value, Constraints

### Phase 9 Context (established patterns)
- `.planning/phases/09-stream-metadata/09-CONTEXT.md` — D-01 through D-08: display template, format detection, bitrate source decisions

### Source Files (primary modification targets)
- `Plugins/SpotOn/ProtocolHandler.pm` — `getMetadataFor()` (line 268): main modification target for async re-fetch, placeholder return, Connect URL detection
- `Plugins/SpotOn/Connect.pm` — `_fetchTrackMetadata()` (line 798): add cache persistence with `spotifyUri` field alongside existing pluginData
- `Plugins/SpotOn/Plugin.pm` — `_trackItem()` (line 398) and `_albumTrackItem()`: update TTL from 3600 to 604800
- `Plugins/SpotOn/DontStopTheMusic.pm` — cache set at line 265: check if TTL update applies here too

### Existing Metadata Patterns
- `Plugins/SpotOn/Plugin.pm` — `_typeString()` (line 1352): format/mode display string helper
- `Plugins/SpotOn/Plugin.pm` — `_bitrateForClient()` (line 1340): bitrate resolution chain

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `spoton_meta_` + md5(url) cache pattern — existing, extend with `spotifyUri` field for Connect entries
- `_typeString($client, $mode)` — Phase 9 helper, returns `{format} (Spotify {mode})`
- `_bitrateForClient($client)` — resolves bitrate from global pref + per-player override
- `Slim::Control::Request::notifyFromArray($client, ['newmetadata'])` — existing notification pattern used by Connect.pm
- `SimpleAsyncHTTP` — LMS async HTTP for non-blocking API calls
- `API::Client->getTrack($accountId, $trackId, $callback)` — existing API method for single-track metadata

### Established Patterns
- Cache key: `spoton_meta_` + md5_hex(url), currently TTL 3600s → bump to 604800s
- Connect URLs: `spotify://connect-<timestamp>` — detected via `$url =~ m{spotify://connect-}`
- Browse URLs: `spotify://track:ID` — Track-ID extractable via regex
- pluginData('info') — ephemeral per-song Connect metadata, set by _fetchTrackMetadata
- `eventTrackUri` — `$client->pluginData('eventTrackUri')` stores real Spotify URI during Connect session

### Integration Points
- `getMetadataFor()` is the sole consumer for NowPlaying/History display — all changes flow through here
- `_fetchTrackMetadata()` in Connect.pm — add cache persistence alongside existing pluginData('info')
- `_trackItem()` and `_albumTrackItem()` in Plugin.pm — TTL bump
- No Settings UI changes needed
- No i18n needed

</code_context>

<specifics>
## Specific Ideas

- Phase 9 display template (`{bitrate}, {format} (Spotify {mode})`) bleibt unverändert
- Connect→Browse Translation soll für den User komplett unsichtbar sein
- Der 7-Tage TTL deckt "Was lief letzte Woche?" ab

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 11-track-history-metadata*
*Context gathered: 2026-06-04*
