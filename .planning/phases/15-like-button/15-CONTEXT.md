# Phase 15: Like Button - Context

**Gathered:** 2026-06-11
**Status:** Ready for planning

<domain>
## Phase Boundary

Benutzer können Tracks aus Browse-Menüs direkt als Liked Songs speichern und entfernen — ohne LMS zu verlassen. Die Funktionalität wird über das LMS Track-Info-Menü exponiert (trackInfoMenu-Pattern nach Qobuz-Vorbild), nicht inline im Track-Item.

</domain>

<decisions>
## Implementation Decisions

### Kontextmenü-Integration (LIB-01, LIB-02, LIB-03)
- **D-01:** Like/Unlike wird über den `trackInfoMenu()`-Hook registriert — das etablierte LMS-Pattern für Track-Aktionen (Qobuz-Referenz: `QobuzManageFavorites()`)
- **D-02:** NICHT im `items`-Array von `_trackItem()`. Das `items`-Array bleibt für Navigation (Artist View, Album View), Aktionen laufen über trackInfoMenu
- **D-03:** Dynamisches Label: "Like" wenn Track nicht geliked, "Unlike" wenn geliked — kurz und Spotify-nah
- **D-04:** Nur für Tracks (type=tracks). Kein Album-Like in dieser Phase — wäre Scope Creep
- **D-05:** Labels in strings.txt für alle 11 Sprachen (PLUGIN_SPOTON_LIKE / PLUGIN_SPOTON_UNLIKE)

### Liked-State-Check (LIB-03, LIB-04)
- **D-06:** On-demand State-Check: `GET /me/library/contains` wird erst aufgerufen wenn der User das Info-Menü öffnet — kein Pre-fetch bei Track-Listing (vermeidet N API-Calls pro Trackliste)
- **D-07:** 60s TTL-Cache für Liked-State (konsistent mit CLAUDE.md Library-Cache-TTL). Wiederholtes Menü-Öffnen innerhalb einer Minute macht keinen neuen API-Call
- **D-08:** Sofortige Cache-Invalidierung nach Like/Unlike-Aktion — nächstes Menü-Öffnen zeigt garantiert den neuen Status. Kein optimistisches Update

### Feedback (UX)
- **D-09:** Erfolg: `showBriefly => 1` + `nextWindow => 'grandparent'` — kurze Bestätigungsmeldung ("Liked!" / "Removed"), dann automatisch zurück ins vorherige Menü (Qobuz-Pattern)
- **D-10:** API-Fehler: Fehlermeldung im Menü via showBriefly ("Fehler beim Speichern"). Bei 403 zusätzlicher Hinweis auf fehlende Berechtigung. User bleibt im Menü
- **D-11:** Bei 429 (Rate Limit): Standard-Retry-Verhalten via Client.pm — kein spezielles Like-Handling nötig

### API-Endpoints (Feb 2026)
- **D-12:** Save: `PUT /me/library` mit `type=tracks` (neuer unified Endpoint, NICHT `PUT /me/tracks`)
- **D-13:** Remove: `DELETE /me/library` mit `type=tracks`
- **D-14:** Check: `GET /me/library/contains` mit `type=tracks`
- **D-15:** Token-Flavor: "own" Client-ID (me/*-Endpoints)

### Claude's Discretion
- Scope-Upgrade-Mechanismus: Researcher soll klären ob Keymaster-Tokens bei --get-token automatisch neue Scopes (user-library-modify) bekommen wenn die App-Config im Spotify Developer Dashboard aktualisiert wurde, oder ob ein Token-Refresh/Re-Auth nötig ist. Falls Re-Auth nötig: cacheSchemaVersion-Bump (2→3) als Mechanismus evaluieren vs. Graceful-Fallback bei 403
- Genaue trackInfoMenu-Registrierung: Researcher soll das LMS Plugin API Muster prüfen (registerInfoProvider? trackInfoMenu callback signature?)
- Cache-Key-Format für Liked-State (z.B. `spoton_liked_{trackId}` vs. anderes Schema)
- Request-Body-Format für die neuen /me/library Endpoints (JSON body vs. query params)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### LMS trackInfoMenu Pattern
- Qobuz Plugin `Plugin.pm` — trackInfoMenu(), QobuzManageFavorites(), QobuzAddFavorite(), QobuzDeleteFavorite(): Referenz-Implementierung für Like/Unlike via Info-Menü. Zu finden unter https://github.com/LMS-Community/plugin-Qobuz/blob/master/Plugin.pm

### SpotOn API-Schicht
- `Plugins/SpotOn/API/Client.pm` — Bestehende API-Methoden (getSavedTracks, getSavedAlbums, getFollowedArtists): Pattern für neue saveTracks/removeTracks/checkTracks Methoden. Zeile 167-176 zeigt getSavedTracks als Vorlage
- `Plugins/SpotOn/API/Client.pm` Zeile 20-30 — Konstanten, Dual-Token-Routing, API_BASE

### SpotOn Track-Item-Struktur
- `Plugins/SpotOn/Plugin.pm` Zeile 389-456 — `_trackItem()`: Bestehende Track-Item-Struktur (items-Array für Navigation, NICHT für Aktionen)
- `Plugins/SpotOn/Plugin.pm` Zeile 674-729 — `_libraryFeed()`: Liked Songs Listing (getSavedTracks), muss nach Like/Unlike den neuen Track zeigen

### Token-Architektur
- `Plugins/SpotOn/API/TokenManager.pm` Zeile 336-410 — `_fetchKeymasterToken()`: --get-token Flow, Flavor-Dispatch (own vs bundled). Relevant für Scope-Klärung

### Cache-Infrastruktur
- `Plugins/SpotOn/Plugin.pm` Zeile 21-25 — `SPOTON_CACHE_VERSION` (aktuell 2), Cache-Namespace. Relevant falls cacheSchemaVersion-Bump nötig
- CLAUDE.md § Slim::Utils::Cache — TTL-Empfehlungen (Library items: 60s)

### Spotify API (Feb 2026)
- CLAUDE.md § Endpoint Status Table — `PUT /me/library`, `DELETE /me/library`, `GET /me/library/contains` als neue unified Endpoints
- CLAUDE.md § Field Removals — Track-Objekt-Änderungen in Dev Mode

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Client.pm::getSavedTracks` (Zeile 167-176) — Pattern für me/*-Requests mit Offset-Pagination. Neue Methoden (saveTracks, removeTracks, checkTracks) folgen diesem Muster
- `Client.pm::_request` — Zentraler Request-Dispatcher mit Rate-Limiting, Dual-Token-Routing. Alle neuen API-Calls laufen hierüber
- `$cache` (Slim::Utils::Cache, Namespace 'spoton') — Bereits initialisiert in Plugin.pm und Client.pm, nutzen für Liked-State-Cache
- `cstring($client, 'KEY')` — i18n-Funktion für dynamische Labels, bereits überall genutzt

### Established Patterns
- trackInfoMenu-Hook: Nicht im SpotOn-Code, aber im LMS-Ökosystem etabliert (Qobuz). Researcher muss die genaue Registrierung recherchieren
- `showBriefly` + `nextWindow => 'grandparent'`: Standard-LMS-Feedback-Pattern für Menü-Aktionen
- `passthrough`-Array für Daten-Weitergabe zwischen Menü-Callbacks (Track-ID, Account-ID)
- `favorites_url` existiert auf Album/Playlist-Items (Zeile 475, 505) — wird für Tracks NICHT genutzt (D-02)

### Integration Points
- trackInfoMenu-Registrierung in `initPlugin()` — neuer Hook, bisher nicht vorhanden
- `Client.pm` — 3 neue Methoden: saveTracks, removeTracks, checkTracks
- `strings.txt` — 2 neue Keys (PLUGIN_SPOTON_LIKE, PLUGIN_SPOTON_UNLIKE) in 11 Sprachen
- `_libraryFeed()` — Muss nach Like/Unlike konsistent sein (gecachte Liked-Songs-Liste)

</code_context>

<specifics>
## Specific Ideas

- Qobuz als Referenz-Implementierung: trackInfoMenu-Pattern mit dynamischem Label, on-demand State-Check, showBriefly-Feedback
- Labels bewusst kurz: "Like" / "Unlike" statt "Zu Liked Songs hinzufügen" / "Aus Liked Songs entfernen"
- Neue Feb-2026 unified Library-Endpoints nutzen (nicht die alten /me/tracks Endpoints)

</specifics>

<deferred>
## Deferred Ideas

- **Album-Like:** PUT /me/library unterstützt auch type=albums — könnte in späterer Phase ergänzt werden
- **Connect Now-Playing Like:** LIB-06 in REQUIREMENTS.md bereits als Future Requirement erfasst — Like/Unlike aus dem Connect-Kontext heraus

</deferred>

---

*Phase: 15-like-button*
*Context gathered: 2026-06-11*
