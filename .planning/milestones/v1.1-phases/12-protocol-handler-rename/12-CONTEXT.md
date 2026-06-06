# Phase 12: Protocol Handler Rename - Context

**Gathered:** 2026-06-05
**Status:** Ready for planning

<domain>
## Phase Boundary

URI-Schema-Umbenennung von `spotify://` → `spoton://` in allen Plugin-Dateien. ProtocolHandler wird als `spoton` statt `spotify` registriert. Alle URL-Konstruktionen, Regex-Matches und Content-Type-Zuordnungen werden aktualisiert. Ziel: SpotOn und Spotty können gleichzeitig in LMS aktiviert werden ohne Protocol-Handler-Konflikte.

</domain>

<decisions>
## Implementation Decisions

### Cache-Migration
- **D-01:** Beim Plugin-Update wird ein kompletter Cache-Flush aller `spoton_*` Keys durchgeführt. Keine granulare Migration — sauberer Neustart.
- **D-02:** Trigger via Version-Marker in Prefs (`cacheSchemaVersion`). Bei `initPlugin()` prüfen: wenn Marker fehlt oder veraltet → alle `spoton_*` Cache-Einträge löschen, Marker auf neue Version setzen. Idempotent.

### LMS History & Library
- **D-03:** Keine automatische History-Migration im Code. Alte `spotify://` Einträge in der LMS-DB werden ignoriert. Der User bereinigt die DB manuell auf dev und raspi.
- **D-04:** Importer.pm wird komplett auf `spoton://` umgestellt. Library-Einträge werden beim nächsten Scan-Lauf automatisch mit dem neuen Schema neu geschrieben.

### Übergangsstrategie
- **D-05:** Clean Break — nur `spoton://` wird akzeptiert. Kein Dual-Schema-Support, kein Fallback auf `spotify://`. SpotOn ist Pre-Release mit minimalem Nutzerkreis.
- **D-06:** Die Spotify-API-URI (`spotify:track:ABC123`) ist nur Eingabe. Die LMS-URL (`spoton://track:ABC123`) ist ein internes Routing-Schema. Klare Trennung: API-URIs bleiben `spotify:`, LMS-URLs werden `spoton://`.

### Spotty-Koexistenz
- **D-07:** Akzeptanzkriterium: Beide Plugins (SpotOn + Spotty) können gleichzeitig in LMS aktiviert werden. Browse funktioniert in beiden unabhängig. Manueller Test auf raspi.
- **D-08:** Kein shared state zwischen SpotOn und Spotty: getrennte Cache-Dirs, getrennte Prefs-Namespaces (`plugin.spoton` vs `plugin.spotty`), getrennte Content-Types (`son`/`soc` vs `spt`/`spc`), getrennte Binaries (`[spoton]` vs `[spotty]`).
- **D-09:** Spotty hat aktuell keinen Connect-Mode — keine Gerätenamen-Kollision. Ggf. Prefix für die Zukunft, aber nicht in dieser Phase.

### Claude's Discretion
- Reihenfolge der Datei-Änderungen (ob ProtocolHandler.pm zuerst oder Plugin.pm)
- Exakter Wert für `cacheSchemaVersion` (z.B. `2` oder ein Versionsstring)
- Ob `_isDeadHistoryUrl` in Connect.pm das neue Schema-Pattern berücksichtigen muss
- Regex-Optimierungen (z.B. ob `m{spoton://connect-}` oder `m{^spoton://connect-}` besser ist)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/REQUIREMENTS.md` — PROTO-01 through PROTO-06 requirement definitions
- `CLAUDE.md` §Protocol Handler Pattern — registerHandler, contentType, formatOverride patterns
- `CLAUDE.md` §Transcoding — custom-convert.conf pipeline structure (son-*, soc-*)
- `.planning/PROJECT.md` — Core Value, Constraints

### Prior Phase Context (affected by rename)
- `.planning/phases/11-track-history-metadata/11-CONTEXT.md` — D-01 through D-07: Cache patterns, Connect→Browse replay, spoton_meta_ keys
- `.planning/phases/09-stream-metadata/09-CONTEXT.md` — D-01 through D-08: type string, format detection using content types son/soc

### Source Files (modification targets)
- `Plugins/SpotOn/Plugin.pm` — `registerHandler('spotify', ...)` (line 82-83), URL construction `'spotify://' . $track_path` (lines 418, 1155), URL regex matches (line 182)
- `Plugins/SpotOn/ProtocolHandler.pm` — `contentType` 'son'/'soc' (lines 26-38), ~30 regex matches on `spotify://`, `formatOverride` (line 47+), `requestString` (line 196+), `getMetadataFor` (line 337+)
- `Plugins/SpotOn/Connect.pm` — URL construction `sprintf("spotify://connect-%u", $ts)` (lines 630, 722, 828), ~15 regex matches on `spotify://connect-`, `_isDeadHistoryUrl` (line 101+)
- `Plugins/SpotOn/DontStopTheMusic.pm` — `"spotify://$1"` (line 259)
- `Plugins/SpotOn/Importer.pm` — Library URL construction (needs full audit for spotify:// references)
- `Plugins/SpotOn/custom-convert.conf` — `$URL$` placeholder (LMS-managed), content types `son`/`soc` already SpotOn-specific

### Spotty Reference (verify namespace separation)
- Spotty registers handler as `'spotify'` — after rename, no collision with SpotOn's `'spoton'`
- Spotty content types: `spt`/`spc` — no collision with `son`/`soc`

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `spoton_meta_` Cache-Key-Prefix — bleibt identisch, nur die gehashte URL ändert sich
- Content-Types `son`/`soc` — bereits SpotOn-spezifisch, keine Änderung nötig
- `Slim::Utils::Prefs` — für cacheSchemaVersion Marker, `$prefs->init()` Pattern bereits etabliert
- `Slim::Utils::Cache` — `$cache->remove()` für Einzelkeys oder Namespace-basiertes Cleanup

### Established Patterns
- Protocol Handler Registration: `Slim::Player::ProtocolHandlers->registerHandler('spotify', 'Plugins::SpotOn::ProtocolHandler')` → wird zu `'spoton'`
- URL-Konstruktion: `'spotify://' . $track_path` → wird zu `'spoton://' . $track_path`
- Connect-URL-Format: `sprintf("spotify://connect-%u", $ts)` → wird zu `sprintf("spoton://connect-%u", $ts)`
- URL-Regex-Pattern: `m{spotify://connect-}` → wird zu `m{spoton://connect-}`
- Cache-Normalisierung in getMetadataFor: `$canonical =~ s{^spotify:}{spotify://}` → wird zu `s{^spoton:}{spoton://}`

### Integration Points
- `Plugin.pm::initPlugin()` — Handler-Registration + cacheSchemaVersion Check
- `ProtocolHandler.pm` — Alle URL-Pattern-Matches
- `Connect.pm` — URL-Erzeugung + Pattern-Matches
- `DontStopTheMusic.pm` — URL-Erzeugung
- `Importer.pm` — Library-URL-Erzeugung
- `custom-convert.conf` — Keine Änderung nötig (Content-Types `son`/`soc` bleiben, `$URL$` ist LMS-Platzhalter)

</code_context>

<specifics>
## Specific Ideas

- Manuelle DB-Bereinigung auf dev und raspi nach Deployment (keine Code-Migration)
- Manueller Koexistenz-Test auf raspi: Spotty + SpotOn gleichzeitig aktiviert, Browse in beiden testen
- Cache-Flush ist einmalig beim ersten Start nach Update — danach greift der Version-Marker

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 12-protocol-handler-rename*
*Context gathered: 2026-06-05*
