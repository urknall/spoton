# Phase 19: Podcast Browse - Context

**Gathered:** 2026-06-14
**Status:** Ready for planning

<domain>
## Phase Boundary

Podcast-Navigation im SpotOn-Menü: "Podcasts" als eigener Top-Level-Menüpunkt mit "Meine Podcasts" (gespeicherte Shows), Show-Detail (Episodenliste), Episoden-Wiedergabe über bestehenden ProtocolHandler, und voll funktionale "Podcast-Suche" (Shows + Episoden getrennt). Phase 18 API-Methoden werden zur Menü-UI verdrahtet.

</domain>

<decisions>
## Implementation Decisions

### Show-Darstellung (POD-01, NAV-02)
- **D-01:** Show-Items in "Meine Podcasts" zeigen Publisher auf line2 — konsistent mit Playlist-Pattern (Owner auf line2)
- **D-02:** Show-Items nutzen `type => 'link'` — Show öffnet Episodenliste, kein Play-Overlay
- **D-03:** Shows werden in API-Reihenfolge angezeigt (added_at desc, neueste zuerst) — KEIN alphabetisches Sortieren. SC2 wird angepasst: "Meine Podcasts lists all saved shows sorted by add date (most recently added first)"
- **D-04:** Show-Artwork aus `images` Array (größtes Bild via `_largestImage`-Pattern)

### Episoden-Anzeige (POD-02, POD-03)
- **D-05:** Episode-Items zeigen Dauer + Datum auf line2 — z.B. "45 Min · 12. Jun 2026". Deckt SC3 ab (title=line1, duration+date=line2)
- **D-06:** Dauer-Format menschenlesbar: "45 Min", "1 Std 23 Min". Muss für Phase 21 i18n-fähig sein, kann Phase 19 erstmal mit deutschen Einheiten bauen
- **D-07:** Datum relativ anzeigen: "Heute", "Gestern", "Vor 3 Tagen", dann absolut "12. Jun", "12. Jun 2025" (für ältere Einträge mit Jahreszahl)
- **D-08:** Episode-Artwork bevorzugen (episode.images), Show-Artwork als Fallback wenn Episode kein eigenes hat
- **D-09:** Episoden-Reihenfolge: neueste zuerst (API-Default). Phase 21 (UX-01) überlagert mit globalem Setting

### Podcast-Suche (NAV-03, SRC-01, SRC-02, SRC-03)
- **D-10:** Voll funktionale Podcast-Suche in Phase 19 — nicht nur Platzhalter. API ist fertig, Search-Pattern existiert
- **D-11:** Suchergebnisse zeigen Shows und Episoden als getrennte Unterebenen — konsistent mit globalem Such-Pattern (_searchFeed → _searchTypeFeed mit type-Passthrough)
- **D-12:** Dev Mode Limit: max 10 Ergebnisse pro Typ (shows, episodes)
- **D-13:** Damit werden SRC-01, SRC-02, SRC-03 effektiv in Phase 19 vorgezogen. Phase 20 fokussiert nur noch auf Library Actions (Follow/Unfollow)

### Claude's Discretion
- **Show-Detail-Struktur:** Ob beim Öffnen einer Show nur die Episodenliste gezeigt wird (wie Album-Detail) oder ein Info-Header mit Show-Beschreibung voransteht (type=textarea). Claude wählt basierend auf Album-Detail-Pattern und OPML-Constraints
- **Episode-OPML-Type:** Ob `type => 'audio'` (direktes Play) oder anderer Typ. Researcher soll das bestehende _trackItem-Pattern prüfen
- **Episode-URI-Schema:** `spoton://episode:ID` oder `spoton://track:ID` mit Episode-URI — Researcher soll ProtocolHandler und librespot `--single-track` Episode-Support prüfen
- **Podcast-Suche Top-Result:** Ob ein Top-Result (erste Show) inline angezeigt wird wie bei der globalen Suche (D-10 aus Phase 3), oder ob Podcast-Suche schlichter ist

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase 18 Kontext (direkte Vorgänger-Phase)
- `.planning/phases/18-podcast-api-foundation/18-CONTEXT.md` — API-Entscheidungen: Cache-TTLs, Token-Routing, Scope-Erweiterung. Alle D-01/D-02 Entscheidungen sind locked

### SpotOn Menü-Architektur
- `Plugins/SpotOn/Plugin.pm` Zeile 286-300 — Top-Level-Menü: Home, Suche, Bibliothek. Podcasts wird NACH Bibliothek eingefügt
- `Plugins/SpotOn/Plugin.pm` Zeile 821-848 — `_libraryFeed`: Pattern für Menü mit Unterebenen (Liked Songs, Albums, Artists, Playlists)
- `Plugins/SpotOn/Plugin.pm` Zeile 855-875 — `_savedTracksFeed`: Pattern für paginated Listen mit offset/index/quantity Mapping
- `Plugins/SpotOn/Plugin.pm` Zeile 984-1090 — `_searchFeed` + `_searchTypeFeed`: Pattern für Suchfunktion mit type-Passthrough

### Item-Builder Patterns
- `Plugins/SpotOn/Plugin.pm` Zeile 522-560 — `_trackItem`: Audio-Item mit line2, image, passthrough, favorites_url. Pattern für `_episodeItem`
- `Plugins/SpotOn/Plugin.pm` Zeile 602-651 — `_albumItem`, `_artistItem`, `_playlistItem`: Link-Item Patterns. Pattern für `_showItem`

### API-Methoden (Phase 18)
- `Plugins/SpotOn/API/Client.pm` Zeile 204-215 — `getSavedShows`: Paginated, offset/limit
- `Plugins/SpotOn/API/Client.pm` Zeile 340-347 — `getShow`: Show-Metadata
- `Plugins/SpotOn/API/Client.pm` Zeile 350-362 — `getShowEpisodes`: Paginated Episode-Liste
- `Plugins/SpotOn/API/Client.pm` Zeile 365-372 — `getEpisode`: Einzelne Episode
- `Plugins/SpotOn/API/Client.pm` Zeile 750-762 — Cache-TTLs: episodes lists=60s, episodes single=300s, shows metadata=3600s

### Playback-Infrastruktur
- `Plugins/SpotOn/ProtocolHandler.pm` Zeile 27-37 — `contentType`: son/soc Routing. Muss Episode-URIs unterstützen
- `Plugins/SpotOn/ProtocolHandler.pm` Zeile 203-270 — `new()`: URL-Parsing und Binary-Aufruf. Episode-URI-Handling prüfen

### i18n
- `Plugins/SpotOn/strings.txt` — Neue Podcast-Strings: Menüpunkte, Zeiteinheiten, No-Results
- CLAUDE.md § Spotify Web API v1 — Endpoint Status Table für shows/episodes Endpoints

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `_savedTracksFeed` (Zeile 855-875) — Exaktes Pattern für `_savedShowsFeed`: offset/index Mapping, getSavedShows API-Call, _showItem Builder
- `_searchFeed` (Zeile 984-1060) — Pattern für Podcast-Suche: type=search Input, dann Unterebenen für Shows/Episodes
- `_searchTypeFeed` (Zeile 1074-1090) — Pattern für typisierte Suche: query+type Passthrough, API-Call, Item-Builder
- `_albumFeed` — Pattern für Show-Detail: Parent-ID Passthrough, paginated Sub-Resource
- `_largestImage()` — Artwork-Extraktion, wiederverwendbar für Show/Episode-Images
- `_getAccountId()` — Account-Resolution, in allen neuen Feeds verwenden
- `cstring()` — Lokalisierte Strings, für alle neuen Menüpunkte

### Established Patterns
- LMS OPMLBased Pagination: `$args->{index}` → offset, `$args->{quantity}` → limit, Response mit `offset`+`total`
- Passthrough-Array für Feed-Parameter: `[{ showId => $id }]`
- Callback-Pattern: `$callback->({ items => \@items, offset => $offset, total => $data->{total} })`
- Search-Pattern: `type => 'search'` für Input-Box, URL → Feed mit `$args->{search}` als Query

### Integration Points
- `Plugin.pm` Top-Level-Menü: Neuer "Podcasts" Eintrag nach "Bibliothek" (Zeile 296-300)
- `Plugin.pm` neue Subs: `_podcastsFeed`, `_savedShowsFeed`, `_showFeed`, `_episodeFeed`, `_podcastSearchFeed`, `_podcastSearchTypeFeed`, `_showItem`, `_episodeItem`
- `strings.txt`: Neue Keys für PLUGIN_SPOTON_PODCASTS, PLUGIN_SPOTON_MY_PODCASTS, PLUGIN_SPOTON_PODCAST_SEARCH, Zeiteinheiten
- `ProtocolHandler.pm`: Episode-URI-Handling prüfen/erweitern

</code_context>

<specifics>
## Specific Ideas

- Podcast-Suche folgt exakt dem globalen Such-Pattern: type=search Menüpunkt → _podcastSearchFeed mit zwei Unterebenen (Shows, Episoden) → je ein _podcastSearchTypeFeed mit type=show bzw. type=episode
- SC2 muss in ROADMAP.md angepasst werden: "sorted by add date (most recently added first)" statt "sorted alphabetically"
- Phase 20 reduziert sich auf Library Actions (Follow/Unfollow) da Search in Phase 19 vorgezogen wird — ROADMAP.md Update nötig

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 19-podcast-browse*
*Context gathered: 2026-06-14*
