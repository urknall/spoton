# Phase 3: Browse + Navigation - Context

**Gathered:** 2026-05-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Nutzer navigieren die vollständige Spotify-Inhaltshierarchie — Home, Suche, Bibliothek — über LMS OPML-Menüs. Dies umfasst: Top-Level-Menüstruktur (Home/Suche/Library), Home-Feed mit Kürzlich gehört/Made For You/Top Tracks, Suche mit kategorisierten Ergebnissen, Library mit Liked Songs/Alben/Künstler/Playlists, Artist-Detailseiten mit Discography, Album-Tracklisten und Playlist-Ansichten. Alle Spotify-API-Endpunkte für Browse/Search/Library werden in Client.pm ergänzt. Track-Auswahl setzt spotify:// Play-Intent (Playback wird in Phase 4 aktiviert).

</domain>

<decisions>
## Implementation Decisions

### Menüstruktur
- **D-01:** NAV-Standard — Home, Suche, Library als drei Top-Level-Einträge nach dem Account-Switcher. Keine flache Struktur, kein Spotty-Klon.
- **D-02:** Home-Feed besteht aus drei Einträgen: Kürzlich gehört (via /me/player/recently-played), Made For You (gefiltert aus /me/playlists), Top Tracks (/me/top/tracks). Jeder Eintrag öffnet seine eigene Liste.
- **D-03:** Library hat vier Einträge: Liked Songs (/me/tracks), Alben (/me/albums), Künstler (via /me/following?type=artist), Playlists (/me/playlists — ohne Made-For-You-Playlists).

### Made-For-You-Erkennung
- **D-04:** Generisches Merkmal statt Namens-Pattern-Matching. Richtung: owner.id oder ähnliches Feld, das Spotify-generierte Playlists von User-erstellten unterscheidet. Research klärt die zuverlässigste Methode. Kein Name-basiertes Filtern (sprachabhängig, fragil).

### Top Tracks
- **D-05:** time_range = medium_term (6 Monate) als fester Wert für den Home-Feed. Kein Umschalter zwischen Zeiträumen.

### Track-Interaktion
- **D-06:** Track-Tap setzt Play-Intent — spotify://{uri} wird in die LMS-Playlist eingereiht. Playback scheitert in Phase 3 (kein aktiver Transcoder), Infrastruktur steht für Phase 4 bereit.
- **D-07:** Tracks bekommen Kontextnavigation: "Artist anzeigen" → Artist-Detailseite, "Album anzeigen" → Album-Detailseite. Ermöglicht Browsen ausgehend vom Track.
- **D-08:** Kein "Alle abspielen"-Eintrag auf Album- oder Playlist-Ebene. Nur Einzeltrack-Auswahl.

### Artist-Detailseite
- **D-09:** Vier Sektionen: Alben, Singles, Compilations, Erscheint auf (via GET /artists/{id}/albums mit include_groups=album,single,compilation,appears_on). Artist Top Tracks entfällt (im Dev Mode entfernt, NAV-10).

### Such-Ergebnisse
- **D-10:** Top-Ergebnis prominent oben (bestes Match direkt navigierbar), dann Sub-Menüs pro Typ (Tracks, Alben, Künstler, Playlists). Kategorien mit 0 Ergebnissen werden ausgeblendet.
- **D-11:** Dev-Mode-entfernte Endpoints (Artist Top Tracks, Related Artists, Browse Categories, New Releases) werden stillschweigend ausgelassen — keine Fehler, keine leeren Platzhalter (NAV-10).

### Pagination
- **D-12:** LMS-internes OPMLBased-Pagination-Framework nutzen (index/quantity). Spotify-API offset/limit wird auf LMS-Pagination gemappt. Kein manuelles "Mehr laden"-Pattern.

### Artwork & Caching
- **D-13:** Album-Cover, Playlist-Bilder und Artist-Fotos als OPML-image-Icons in allen Listen. Spotify-API liefert image-URLs in mehreren Größen. Research untersucht Spotty's Bild-Caching-Ansatz (User erinnert sich an Cache-Nutzung bei Spotty).

### Claude's Discretion
- Suchergebnisse pro Kategorie: Research entscheidet optimale Anzahl (5 vs 10 bei Dev Mode limit=10)
- Library-Sortierung: Research prüft ob Umschaltung sinnvoll oder "recently added" als feste Sortierung ausreicht
- Track-Metadaten-Format: Research prüft OPML line2/subtext Support und orientiert sich an Spotty/Qobuz-Praxis
- Spotify-API → LMS-Pagination-Mapping: Research klärt technische Details

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Projekt-Kontext
- `.planning/PROJECT.md` — Core Value, Constraints, Key Decisions (HTTP-streaming deferred, kein Batch)
- `.planning/REQUIREMENTS.md` — NAV-01 bis NAV-11 Anforderungsdefinitionen
- `.planning/phases/01-plugin-skeleton-binary-foundation/01-CONTEXT.md` — Phase 1 Entscheidungen (Format `son`, Binary-Strategie, Helper.pm)
- `.planning/phases/02-auth-api-foundation/02-CONTEXT.md` — Phase 2 Entscheidungen (Client.pm Architektur, Rate-Limiting, Caching, Account-Switcher D-05, kein Batch D-13)
- `.planning/phases/02.1-oauth-pkce-browser-auth/02.1-CONTEXT.md` — Phase 02.1 Entscheidungen (PKCE-Auth, TokenManager, Client-ID per User)

### Bestehender Code (wird erweitert)
- `Plugins/SpotOn/Plugin.pm` — handleFeed() wird um Home/Suche/Library-Menüpunkte erweitert
- `Plugins/SpotOn/API/Client.pm` — Alle Browse/Search/Library-Endpunkte werden hier ergänzt (bisher nur getMe)
- `Plugins/SpotOn/ProtocolHandler.pm` — spotify:// URI Handler (Play-Intent Ziel)

### Technologie-Referenz
- `CLAUDE.md` §Spotify Web API v1 — Endpoint-Status-Tabelle (Working/Deprecated/Removed), Dev Mode Einschränkungen (limit=10, keine Batch-Endpoints, entfernte Felder)
- `CLAUDE.md` §Technology Stack — OPMLBased Menü-Framework, SimpleAsyncHTTP, Slim::Utils::Cache TTLs
- `CLAUDE.md` §LMS Plugin API Modules — Slim::Plugin::OPMLBased OPML-Item-Struktur

### Spotty-Referenz (Prior Art)
- https://github.com/michaelherger/Spotty-Plugin — Herger's Plugin (Menüstruktur, Bild-Caching, Pagination-Pattern, Browse/Search-Implementierung)

### LMS-Referenz
- https://github.com/LMS-Community/slimserver — OPMLBased Framework (index/quantity Pagination), SimpleAsyncHTTP
- https://github.com/LMS-Community/plugin-Qobuz — Qobuz-Plugin (Browse/Search-Referenz, OPML-Item-Patterns, Artwork-Handling)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `API/Client.pm::_request()` — Vollständige Request-Pipeline (Rate-Limiting, Caching, Concurrency-Cap, Token-Injection). Neue Endpunkt-Methoden bauen darauf auf.
- `API/Client.pm::_cacheTTL()` — Domain-spezifische TTLs bereits definiert: Library 60s, Metadata 3600s, Playlists/Browse 300s.
- `Plugin.pm::handleFeed()` — Account-Switcher + Rate-Limit-Hint bereits implementiert. Home/Suche/Library-Items werden darunter eingefügt.
- `Plugin.pm::_accountSwitcherFeed()` — Referenz-Pattern für OPML-Sub-Feeds mit passthrough-Parametern und nextWindow.
- `ProtocolHandler.pm` — spotify:// Handler registriert, contentType 'son', canDirectStream=0.

### Established Patterns
- `$prefs->client($client)->get('activeAccount')` — Per-Player Account-Auswahl (für Browse-Kontext relevant)
- `cstring($client, 'PLUGIN_SPOTON_...')` — i18n-Strings-Pattern (EN + DE)
- `Slim::Networking::SimpleAsyncHTTP` — Async-HTTP für alle API-Calls
- Callback-Pattern: `$callback->({ items => \@items })` für OPML-Feed-Antworten
- passthrough-Pattern: `passthrough => [{ key => $value }]` für Parameter-Weitergabe an Sub-Feeds

### Integration Points
- `Plugin.pm::handleFeed()` → Home/Suche/Library als Items einfügen (nach Account-Switcher)
- `Client.pm` → Neue Methoden: search(), getRecentlyPlayed(), getTopTracks(), getSavedTracks(), getSavedAlbums(), getFollowedArtists(), getPlaylists(), getArtist(), getArtistAlbums(), getAlbum(), getAlbumTracks(), getPlaylistItems()
- `strings.txt` → Neue i18n-Strings für alle Menü-Labels (EN + DE)

</code_context>

<specifics>
## Specific Ideas

- Made-For-You-Filter via generisches Merkmal (owner.id = 'spotify' o.ä.) — kein Namens-Pattern
- Top-Ergebnis bei Suche prominent oben anzeigen (bestes Match direkt navigierbar)
- Artwork-Caching untersuchen — User erinnert sich an Spotty's Cache-Nutzung für Bilder
- LMS-interne OPMLBased-Pagination nutzen statt eigenes "Mehr laden"-Pattern

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 3-Browse + Navigation*
*Context gathered: 2026-05-28*
