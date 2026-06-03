# Phase 6: Polish + Release Readiness - Context

**Gathered:** 2026-06-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Feature-Polish (per-Player Prefs, DSTM, Normalisierung), Release-Vorbereitung (vollständige i18n, Setup-Guide, Client-ID-Konsolidierung, Custom-Binary-Support), und Distribution als LMS Custom Repository für Early-Adopter-Feedback. Per-Player Settings werden zu einer einheitlichen Format-/Streaming-Konfiguration zusammengeführt (Connect UND Browse). DSTM nutzt Spotty-NG Seed-Logik mit recommendations-via-bundled-Token + Search-Fallback.

</domain>

<decisions>
## Implementation Decisions

### Per-Player Settings Architektur
- **D-01:** Bitrate per Player = Global + Override. Globales Bitrate-Setting (320 Default) bleibt. Jeder Player kann optional einen eigenen Wert setzen (96/160/320). Ohne Override gilt der globale Wert. Pattern analog zu bestehendem OGG-Override.
- **D-02:** Volume Normalisierung bleibt global. Kein per-Player Override. Alle Player bekommen die gleiche Lautstärke-Angleichung. Normalisierung muss auch für Connect-Modus gelten (Daemon.pm übergibt bereits `--enable-volume-normalisation`).
- **D-03:** Per-Player Settings-Seite = eine Sektion pro Player. Alle per-Player Prefs in einer Sektion untereinander. Player-Dropdown oben wählt den Player aus (existiert bereits). Kein Splitting nach Funktionsbereichen.
- **D-04:** Client-ID Konsolidierung — `SPOTON_DEFAULT_CLIENT_ID` wird in `Client.pm` definiert (einzige Stelle). `TokenManager.pm` importiert die Konstante von dort. SC-7 erfüllt.

### DSTM (Don't Stop The Music)
- **D-05:** Primär: `recommendations`-Endpoint via bundled-Token (Herger-ID mit Extended Quota). Gleiche Seed-Logik wie Spotty-NG: bis zu 5 Seeds aus aktueller Playlist, Spotify-Tracks direkt als seed_tracks, Nicht-Spotify-Tracks via Search-Match. Fallback bei 404/403: Search-basiert (`artist:"Seed-Artist"` mit randomisiertem Offset).
- **D-06:** DSTM-Integration über LMS-Standard-Framework. SpotOn registriert sich als DSTM-Provider via `Slim::Plugin::DontStopTheMusic::Plugin->registerHandler()`. Aktivierung durch User in LMS Player-Settings → Don't Stop The Music → "SpotOn Empfehlungen". Kein eigener Toggle in SpotOn-Settings.

### Setup Guide & Distribution
- **D-07:** Setup Guide Platzierung — Claude entscheidet basierend auf LMS-Konventionen und Spotty-NG-Referenz. Muss die Schritte in korrekter Reihenfolge zeigen (Developer App, Client-ID, Spotify App verbinden).
- **D-08:** Credits-Text als dezenter Footer am Ende der Settings-Seite: "SpotOn nutzt librespot. Inspiriert von Hergers Spotty Plugin."
- **D-09:** Repository-Distribution via GitHub repo.xml. LMS-Nutzer fügen die URL in Settings → Plugins → Additional Repositories ein. Standard-Weg für Third-Party LMS Plugins.
- **D-10:** i18n-Übersetzungen von Claude generiert für die volle LMS-Sprachpalette (EN, DE, FR, NL, IT, ES, SV, NO, DA, PL, CS). Community kann später korrigieren.

### Transcoding Fallback
- **D-11:** Per-Player Format-Dropdown — einheitlich für Connect UND Browse/Single-Track. Erweitert den bestehenden OGG-Override-Dropdown: Auto / OGG (DirectStream) / PCM (DirectStream) / FLAC (transkodiert) / MP3 (transkodiert). Bei FLAC/MP3: `canDirectStream()` gibt 0 zurück → LMS proxied via custom-convert.conf Pipeline. Tooltips/Mouseover erklären die Optionen.
- **D-12:** Deferred Item aus Phase 5 eingelöst: Per-Player OGG-Passthrough gilt jetzt für BEIDE Modi (Connect und Browse), nicht nur Connect.

### Claude's Discretion
- Setup Guide Platzierung und Detailtiefe (D-07): Settings-Seite oben vs. inline vs. eigene Section
- Binary-Build-Pipeline: Wie die Multi-Architektur-Binaries (x86_64, aarch64, armhf, i386) bereitgestellt werden
- repo.xml Struktur und Versionierung
- Security Review und Code Review Scope (SC-11): Welche Module und in welcher Tiefe
- DSTM Search-Fallback Details: Welche Search-Parameter, wie viele Results, Randomisierung

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Projekt-Kontext
- `.planning/PROJECT.md` — Core Value, Constraints, Key Decisions
- `.planning/REQUIREMENTS.md` — LMS-03, LMS-06, LMS-08, LMS-09, LMS-10 Anforderungsdefinitionen
- `.planning/ROADMAP.md` — Phase 6 Success Criteria (12 Kriterien), Backlog #2 (eigene Client-ID)

### Vorherige Phase-Entscheidungen
- `.planning/phases/04-single-track-streaming/04-CONTEXT.md` — D-01 updateTranscodingTable, D-04 FLAC-Default, D-05 Globales Bitrate, D-06 Normalization
- `.planning/phases/05-spotify-connect/05-CONTEXT.md` — D-01 HTTP-Streaming, D-04 soc-Pipeline, D-05 OGG Auto-Detect+Override, D-10 Connect per Player Toggle, D-13/D-14 Event-Protokoll/Control-Endpoints
- `.planning/phases/05.4-mdns-connect-discovery-fix/05.4-CONTEXT.md` — D-04/D-05 Discovery-Toggle per Player

### Bestehender Code (wird erweitert/modifiziert)
- `Plugins/SpotOn/Settings.pm` — Per-Player Prefs (enableSpotifyConnect, connectOggOverride, disableDiscovery). Muss erweitert um: Bitrate-Override, Format-Dropdown (vereinheitlicht OGG-Override + Transcode-Fallback), Setup Guide, Credits Footer
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Settings-Template. Muss erweitert um: per-Player Bitrate, Format-Dropdown, Setup Guide Section, Credits
- `Plugins/SpotOn/Plugin.pm` — `initPlugin()` braucht DSTM-Registration. `updateTranscodingTable()` (Zeile 1186-1249) muss per-Player Bitrate und Format-Override unterstützen. Prefs-Init erweitern.
- `Plugins/SpotOn/ProtocolHandler.pm` — `canDirectStream()` muss per-Player ForceTranscode respektieren (0 zurückgeben wenn transkodiert). `formatOverride()` muss per-Player Format-Setting berücksichtigen.
- `Plugins/SpotOn/API/Client.pm` — SPOTON_DEFAULT_CLIENT_ID bleibt hier (einzige Definition). `recommendations()`-Methode muss implementiert werden (für DSTM). Export der Konstante für TokenManager.
- `Plugins/SpotOn/API/TokenManager.pm` — SPOTON_DEFAULT_CLIENT_ID entfernen, Import von Client.pm
- `Plugins/SpotOn/strings.txt` — 66 Keys, nur EN+DE. Muss auf volle LMS-Sprachpalette erweitert werden (11 Sprachen)
- `Plugins/SpotOn/install.xml` — Version, Repository-Metadaten aktualisieren
- `Plugins/SpotOn/custom-convert.conf` — Bitrate-Platzhalter für per-Player Bitrate

### Neuer Code (zu erstellen)
- `Plugins/SpotOn/DontStopTheMusic.pm` — DSTM-Provider (nach Spotty-NG Pattern)

### Referenz-Implementierung (Spotty-NG)
- `/home/sti/spotty-ng/Spotty-Plugin/DontStopTheMusic.pm` — DSTM-Implementierung: Seed-Logik, Search-Matching, recommendations-Aufruf (165 Zeilen)
- `/home/sti/spotty-ng/Spotty-Plugin/Plugin.pm` Zeile 164-168 — DSTM-Registration in initPlugin()

### Technologie-Referenz
- `CLAUDE.md` §librespot — CLI-Flags: `--bitrate`, `--enable-volume-normalisation`, `--cache`, `--single-track`
- `CLAUDE.md` §Spotify Web API v1 — recommendations (removed in Dev Mode, möglicherweise via Extended Quota verfügbar), me/top/tracks, me/player/recently-played
- `CLAUDE.md` §LMS Plugin API — Slim::Plugin::DontStopTheMusic, OPMLBased, Prefs

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Settings.pm` per-Player Prefs Pattern: `$prefs->client($client)->get(...)` / `->set(...)` — bereits für enableSpotifyConnect, connectOggOverride, disableDiscovery etabliert. Direkt erweiterbar für Bitrate-Override und Format-Setting.
- `Plugin.pm::updateTranscodingTable()` (Zeile 1186-1249) — Dynamische Transcoding-Tabelle mit Bitrate-Injection und Normalisierung. Muss für per-Player Bitrate und Format-Override erweitert werden.
- `ProtocolHandler.pm::canDirectStream()` — Gibt HTTP-URL für Connect-Einzelspieler zurück. Muss per-Player ForceTranscode prüfen.
- `ProtocolHandler.pm::formatOverride()` — Gibt `soc` für Connect, `flc` für Browse zurück. Muss per-Player Format-Setting berücksichtigen.
- `Spotty-NG DontStopTheMusic.pm` — Komplette DSTM-Referenzimplementierung mit Seed-Logik und recommendations-Aufruf. Adaptierbar für SpotOn mit Dual-Token-Routing.
- `Helper.pm::get()` (Zeile 129) — Hat bereits `LMS-10 Vorbereitung` Kommentar für Custom-Binary-Override.

### Established Patterns
- Per-Player Prefs: `$prefs->client($client)->get('key')` / `$prefs->init({ key => default })`
- Pref-Change-Callback: `$prefs->setChange(\&callback, 'key')` für Daemon-Reinit bei Settings-Änderung
- DSTM-Registration: `Slim::Plugin::DontStopTheMusic::Plugin->registerHandler('NAME', \&handler)`
- i18n: `cstring($client, 'PLUGIN_SPOTON_...')` für Perl, `strings.txt` Format mit TAB-Einrückung pro Sprache
- Settings-Template: `basic.html` mit Perl-Template-Toolkit Syntax `[% ... %]`

### Integration Points
- `Plugin.pm::initPlugin()` → DSTM-Registration (`require + init()`)
- `Settings.pm::handler()` → per-Player Bitrate + Format-Dropdown verarbeiten
- `basic.html` → per-Player Bitrate-Dropdown, Format-Dropdown mit Tooltips, Setup Guide Section, Credits Footer
- `ProtocolHandler.pm::canDirectStream()` → per-Player ForceTranscode Check
- `ProtocolHandler.pm::formatOverride()` → per-Player Format-Mapping (ogg/pcm/flc/mp3)
- `updateTranscodingTable()` → per-Player Bitrate-Injection
- `Client.pm` → `recommendations()` API-Methode für DSTM, Export der Client-ID Konstante
- `custom-convert.conf` → Bitrate dynamisch statt hardcoded 320

</code_context>

<specifics>
## Specific Ideas

- Format-Dropdown vereinheitlicht Connect OGG-Override + Browse Format-Wahl + Transcode-Fallback in einem Setting: Auto / OGG / PCM / FLAC (transkodiert) / MP3 (transkodiert). Tooltips erklären den Unterschied zwischen DirectStream und transkodiert.
- DSTM-Seed-Logik 1:1 von Spotty-NG übernehmen, aber recommendations über bundled-Token routen (Dual-Token-Architektur Phase 04.4). Search-Fallback als Absicherung falls Extended Quota den Endpoint nicht mehr hat.
- Client-ID Konsolidierung ist mechanisch: Konstante in Client.pm behalten, aus TokenManager.pm entfernen, Import hinzufügen. SC-7 mit minimalem Eingriff erledigt.
- repo.xml auf GitHub: Kann als Datei im Repo liegen, GitHub Raw URL als Repository-URL. install.xml mit Versionierung und Download-URL aktualisieren.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 06-polish-release-readiness*
*Context gathered: 2026-06-03*
