# Phase 4: Single-Track Streaming - Context

**Gathered:** 2026-05-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Nutzer können jeden Spotify-Track aus den Browse-Menüs abspielen. Das Plugin wählt die richtige Transcoding-Pipeline (OGG/FLAC/MP3/PCM) pro Player, unterstützt Seeking, verwaltet librespot-Prozesse und Audio-Cache. Kontext-Queueing reiht bei Track-Tap das gesamte Album/die gesamte Playlist in die LMS-Queue ein. Bitrate ist global konfigurierbar.

</domain>

<decisions>
## Implementation Decisions

### Format-Auswahl & Transcoding
- **D-01:** Pipeline-Auswahl-Mechanismus — Research evaluiert Spotty-Ansatz (updateTranscodingTable, dynamisches Umschreiben der Transcoding-Tabelle) vs. formatOverride-basiert (ProtocolHandler gibt Format pro Player zurück, LMS wählt Pipeline aus statischer custom-convert.conf). Claude wählt den zuverlässigsten Ansatz für LMS 8.0+.
- **D-02:** B&O/UPnPBridge OGG-Support — Research klärt ob B&O-Geräte via UPnPBridge OGG Vorbis unterstützen und erstellt Format-Support-Matrix für die Test-Geräte (squeezelite, B&O/UPnPBridge).
- **D-03:** Race Condition LMS-11 — Multi-Player-Streaming (zwei unabhängige Player gleichzeitig verschiedene Tracks) kommt beim User selten vor, aber muss für andere User sauber gelöst werden. Research bestimmt den besten Ansatz für atomische Pro-Player-Transcoding-Auswahl.
- **D-04:** FLAC als Standard-Fallback-Format für Player ohne besondere Capabilities. Konsistent mit bestehendem `getFormatForURL => 'flc'` in ProtocolHandler.pm.

### Settings-Scope Phase 4
- **D-05:** Globales Bitrate-Setting (96/160/320 kbps) in Phase 4 auf der Settings-Seite. Per-Player-Override kommt in Phase 6 (LMS-08). Plugin-Default bleibt 320 kbps (bereits in prefs init).
- **D-06:** Volume Normalization (STR-08) — Research evaluiert ob Toggle (an/aus) in Phase 4 oder Phase 6. Methode (basic/dynamic) als Claude's Discretion.
- **D-07:** Audio Cache (STR-11) — Research evaluiert ob Audio-Cache in der Single-Track-Pipeline Sinn macht (jeder Track ist ein separater librespot-Prozess mit --single-track).
- **D-08:** Settings UI — Neuer Abschnitt "Streaming" auf der bestehenden Settings-Seite (basic.html) unterhalb der Auth-Einstellungen. Mindestens Bitrate-Dropdown.

### Playback-Modus
- **D-09:** Kontext-Queueing — Track-Tap in einem Album/Playlist-Kontext reiht das gesamte Album/die gesamte Playlist in die LMS-Queue ein und startet Playback beim angetippten Track. Nicht nur ab angeklicktem Track, sondern das ganze Album (Tracks vor dem angeklickten sind via Skip Previous erreichbar). Research prüft die Machbarkeit in LMS (Playlist-Position als Start-Index). Fallback: ab angeklicktem Track bis Ende.
- **D-10:** Tracks außerhalb eines Album/Playlist-Kontexts (Suchergebnisse, Kürzlich gehört, Top Tracks) — Research prüft wie Spotty/Qobuz diese Fälle handhaben. Claude's Discretion.
- **D-11:** Gapless Playback (STR-09) — Nice-to-have, keine harte Anforderung. Kurze Pause zwischen Tracks ist akzeptabel. Priorität liegt auf zuverlässigem Playback. Research evaluiert ob Gapless mit --single-track (separater librespot-Prozess pro Track) überhaupt möglich ist.

### Claude's Discretion
- Pipeline-Auswahl-Mechanismus (D-01): updateTranscodingTable vs. formatOverride — Research entscheidet
- B&O OGG-Support-Matrix (D-02): Format-Capabilities der Test-Geräte
- Race Condition Lösung (D-03): Atomische Pro-Player-Transcoding-Strategie
- Volume Normalization Timing (D-06): Phase 4 oder Phase 6
- Audio Cache Strategie (D-07): An/Aus/Konfigurierbar oder komplett deaktiviert
- Kontextloses Queue-Verhalten (D-10): Single-Track vs. alle sichtbaren Tracks
- Gapless Machbarkeit (D-11): Ob und wie mit --single-track realisierbar
- Orphaned-Process-Cleanup (STR-10): Timer-Intervall und Cleanup-Strategie

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Projekt-Kontext
- `.planning/PROJECT.md` — Core Value, Constraints, Key Decisions (OGG-Direct als Option, FLAC als Default, Binary-Strategie offen)
- `.planning/REQUIREMENTS.md` — STR-01 bis STR-11, LMS-11 Anforderungsdefinitionen
- `.planning/phases/02-auth-api-foundation/02-CONTEXT.md` — Phase 2 Entscheidungen (Client.pm Architektur, Rate-Limiting, Caching-TTLs, kein Batch D-13)
- `.planning/phases/02.1-oauth-pkce-browser-auth/02.1-CONTEXT.md` — Phase 02.1 Entscheidungen (PKCE-Auth, TokenManager, Settings-Seite Struktur)
- `.planning/phases/03-browse-navigation/03-CONTEXT.md` — Phase 3 Entscheidungen (D-06 Play-Intent, D-08 kein "Alle abspielen", D-12 LMS-Pagination, D-13 Artwork)

### Bestehender Code (wird erweitert/modifiziert)
- `Plugins/SpotOn/ProtocolHandler.pm` — formatOverride() aktuell statisch 'flc', canSeek/getSeekData implementiert. Kernstück der Phase 4 Änderungen.
- `Plugins/SpotOn/custom-convert.conf` — Alle 4 Pipelines definiert (son→pcm/flc/mp3/ogg). Bitrate hardcoded auf 320, muss dynamisch werden.
- `Plugins/SpotOn/custom-types.conf` — 'son' Type registriert
- `Plugins/SpotOn/Helper.pm` — Binary-Discovery, getCapability() für Format-Erkennung
- `Plugins/SpotOn/Plugin.pm` — _trackItem() mit spotify:// URI und on_select=play. Muss für Kontext-Queueing erweitert werden.
- `Plugins/SpotOn/Settings.pm` — Settings-Handler, wird um Streaming-Abschnitt erweitert
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Settings-Template, neuer Streaming-Abschnitt

### Technologie-Referenz
- `CLAUDE.md` §Technology Stack — ProtocolHandler Pattern, Transcoding (custom-convert.conf), librespot CLI-Flags (--single-track, --bitrate, --passthrough, --start-position, --enable-volume-normalisation, --cache, --cache-size-limit, --disable-audio-cache)
- `CLAUDE.md` §librespot — Audio-Backends (Pipe-Backend für stdout), Format-Optionen (S16/S24/S32), Bitrate-Tiers (96/160/320)
- `CLAUDE.md` §Spotify Web API v1 — Endpoint-Status (keine Batch-Endpoints in Dev Mode)

### Spotty-Referenz (Prior Art)
- https://github.com/michaelherger/Spotty-Plugin — Herger's Plugin (updateTranscodingTable, Format-Auswahl, Gapless-Pattern, Queue-Management, Orphaned-Process-Cleanup)

### LMS-Referenz
- https://github.com/LMS-Community/slimserver — ProtocolHandler Framework, Transcoding Pipeline, Player-Capabilities-API
- https://github.com/LMS-Community/plugin-Qobuz — Qobuz ProtocolHandler (Format-Auswahl-Referenz, Queue-Verhalten, Gapless)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ProtocolHandler.pm` — Grundgerüst komplett: contentType 'son', canDirectStream=0, canSeek, canTranscodeSeek, getSeekData mit timeOffset. formatOverride() muss dynamisch werden.
- `custom-convert.conf` — Alle Pipelines definiert. $CACHE$, $URL$, $START$ Platzhalter eingerichtet. Bitrate-Platzhalter fehlt (aktuell hardcoded 320).
- `Helper.pm::getCapability()` — Kann Binary-Capabilities abfragen (z.B. unterstützte Audio-Backends). Nutzbar für Format-Detection.
- `Plugin.pm::_trackItem()` — Track-OPML-Items mit spotify:// URI. Muss für Kontext-Queueing um Kontext-Info (Album-ID, Playlist-ID, Position) erweitert werden.
- `API/Client.pm` — getAlbumTracks(), getPlaylistItems() existieren bereits für Queue-Befüllung.
- `Settings.pm + basic.html` — Settings-Infrastruktur mit AJAX-Handler. Bereit für Streaming-Abschnitt.

### Established Patterns
- `$prefs->init({ bitrate => 320 })` — Bitrate-Pref bereits initialisiert, nur Settings-UI fehlt
- `custom-convert.conf` Flag-Syntax: `R` (remote), `T` (seek to time), `B` (bitrate selectable)
- `cstring($client, 'PLUGIN_SPOTON_...')` — i18n-Pattern für neue Streaming-Strings
- `Slim::Player::ProtocolHandlers->registerHandler('spotify', ...)` — URI-Handler registriert

### Integration Points
- `ProtocolHandler.pm::formatOverride()` → Dynamische Format-Auswahl pro Player
- `custom-convert.conf` → Bitrate-Platzhalter ($BITRATE$ oder Pref-basiert)
- `Plugin.pm::_trackItem()` → Kontext-Info für Queueing mitsenden
- `Plugin.pm::initPlugin()` → Orphaned-Process-Cleanup-Timer starten
- `Settings.pm::handler()` → Streaming-Prefs verarbeiten
- `basic.html` → Streaming-Abschnitt mit Bitrate-Dropdown
- `strings.txt` → Neue i18n-Strings für Settings und Streaming

</code_context>

<specifics>
## Specific Ideas

- Kontext-Queueing: Ganzes Album/Playlist einreihen beim Track-Tap, Playback startet bei angeklicktem Track — so kann man auch zu vorherigen Tracks springen. User findet das aus UX-Sicht ideal.
- Gapless ist nice-to-have, nicht kritisch. Zuverlässiges Playback hat Vorrang.
- Settings-Seite: neuer Abschnitt "Streaming" unterhalb Auth, Bitrate als erstes Setting

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 4-Single-Track Streaming*
*Context gathered: 2026-05-28*
