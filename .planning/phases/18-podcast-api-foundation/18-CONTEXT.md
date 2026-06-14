# Phase 18: Podcast API Foundation - Context

**Gathered:** 2026-06-14
**Status:** Ready for planning

<domain>
## Phase Boundary

Die API-Schicht bekommt Podcast-Fähigkeiten: Der OAuth-Scope `user-read-playback-position` wird zum Auth-Flow hinzugefügt und 5 neue Client.pm-Methoden (`getSavedShows`, `getShow`, `getShowEpisodes`, `getEpisode`, plus ggf. Scope-Management) ermöglichen Show- und Episoden-Zugriff für die nachfolgenden Browse- und Search-Phasen.

</domain>

<decisions>
## Implementation Decisions

### Episodenlisten-Caching (API-02)
- **D-01:** Episodenlisten (`getShowEpisodes`) werden mit 60s TTL gecacht — konsistent mit dem Library-Items-Pattern (Liked Songs). Wiederholtes Öffnen einer Show innerhalb einer Minute macht keinen neuen API-Call
- **D-02:** 60s ist der Kompromiss: Resume-Status bleibt aktuell genug für den UX-Flow (User hört nicht gleichzeitig in Spotify App UND browst in SpotOn), aber unnötige API-Calls werden vermieden

### Claude's Discretion
- **Scope-Erweiterung:** Researcher soll den Keymaster-Scope-Mechanismus klären — bekommt `--get-token` automatisch neue Scopes wenn sie im Spotify Developer Dashboard hinzugefügt werden, oder muss das Binary eine Scope-Liste übergeben? Basierend darauf: Re-Auth-Strategie wählen (Graceful Degradation vs. proaktiver Cache-Schema-Version-Bump vs. transparenter Token-Refresh)
- **Scope-Zuordnung zu Token-Flavors:** Researcher soll klären ob `user-read-playback-position` nur für den own-Token oder auch für den bundled-Token nötig ist. Abhängig davon ob die Show/Episode-Endpoints über eigene oder bundled Client-ID laufen
- **Token-Routing für Shows/Episodes:** Die bestehende own-first Logik (D-04 aus Phase 04.4) routet `shows/{id}`, `episodes/{id}`, `shows/{id}/episodes` korrekt zum own-Token (da sie nicht in `@KNOWN_DEPRECATED_FAMILIES` stehen). Researcher soll prüfen ob ein expliziter Guard (wie `$_meFamilyRegex` für me/*) einen echten Vorteil bringt oder das Default-Verhalten ausreicht
- **Show-Metadata Cache-TTL:** Researcher soll basierend auf CLAUDE.md Cache-Empfehlungen entscheiden — 3600s (wie Artist/Album-Metadata) oder kürzer. Show-Info (Name, Beschreibung, Artwork) ändert sich selten
- **getEpisode Einzelabruf-TTL:** 60s wie Episodenliste oder 0s (immer live) wenn der User explizit eine Episode öffnet? Researcher soll die sinnvollste Strategie bestimmen
- **Binary-Änderung:** Falls librespot eine Code-Änderung für den neuen Scope braucht → Cargo.toml Version-Bump (1.0.0 → 1.1.0, gemäß Feedback-Memory)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### SpotOn API-Schicht
- `Plugins/SpotOn/API/Client.pm` — Bestehende API-Methoden und Patterns. Zeile 169-201: `getSavedTracks`, `getSavedAlbums`, `getFollowedArtists` als Vorlagen für neue Podcast-Methoden. Zeile 394-466: `_request` + `_resolveStartFlavor` für Dual-Token-Routing-Verständnis
- `Plugins/SpotOn/API/Client.pm` Zeile 20-35 — Konstanten: `API_BASE`, `SPOTON_DEFAULT_CLIENT_ID`, Dual-Token-Routing-Constants
- `Plugins/SpotOn/API/Client.pm` Zeile 44-58 — `_meFamilyRegex` und `@KNOWN_DEPRECATED_FAMILIES`: Routing-Regeln die Podcast-Endpoints NICHT betreffen (kein Eintrag nötig)

### Token-Architektur
- `Plugins/SpotOn/API/TokenManager.pm` Zeile 365-404 — `_fetchKeymasterToken()`: `--get-token` Aufruf, Flavor-Dispatch. HIER muss geprüft werden ob/wie Scopes konfiguriert werden

### Cache-Infrastruktur
- `Plugins/SpotOn/Plugin.pm` Zeile 21-25 — `SPOTON_CACHE_VERSION` (aktuell 3), Cache-Namespace. Relevant falls Cache-Schema-Version-Bump für Re-Auth nötig
- CLAUDE.md § Slim::Utils::Cache — TTL-Empfehlungen (Library items: 60s, Metadata: 3600s)

### Spotify API (Podcast-Endpoints)
- CLAUDE.md § Endpoint Status Table — Podcast-Endpoints nicht explizit gelistet; Researcher muss Spotify API Docs für `/me/shows`, `/shows/{id}`, `/shows/{id}/episodes`, `/episodes/{id}` konsultieren
- CLAUDE.md § Rate Limits — Rolling 30-second window, proaktive Burst-Prevention

### Phase 15 Like Button (Pattern-Referenz)
- `.planning/phases/15-like-button/15-CONTEXT.md` — Etabliertes Pattern für neue me/*-API-Methoden, Feb-2026 unified `/me/library` Endpoints, Cache-Invalidierung, Scope-Handling

### librespot Binary
- `librespot-spoton/Cargo.toml` — Version-Management falls Binary-Änderung für Scope nötig
- CLAUDE.md § librespot Key CLI Flags — `--get-token` Dokumentation

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Client.pm::getSavedTracks` (Zeile 169-176) — Exaktes Pattern für `getSavedShows`: `_request('get', 'me/shows', { _accountId => $accountId, limit => ..., offset => ... }, $cb)`
- `Client.pm::getAlbumTracks` (Zeile 311-322) — Pattern für `getShowEpisodes`: Paginated Sub-Resource mit Parent-ID
- `Client.pm::_request` — Zentraler Dispatcher: Rate-Limiting, Dual-Token-Routing, Concurrency-Cap. Alle neuen Methoden nutzen diesen Eintrittspunkt
- `$cache` (Slim::Utils::Cache, Namespace 'spoton', Version 3) — Bereits initialisiert, nutzen für Episode/Show-Cache

### Established Patterns
- Offset-Pagination: `limit` + `offset` Parameter durchgereicht via `_request` params hash
- Dual-Token-Routing: `me/*` → own (hard guard), Rest → own-first mit bundled-Fallback
- `_accountId` in jedem Request für Multi-Account-Support
- `_noCache => 1` für Requests die nie gecacht werden sollen

### Integration Points
- `Client.pm` — 5 neue Methoden: `getSavedShows`, `getShow`, `getShowEpisodes`, `getEpisode` (+ ggf. Scope-Management-Logik)
- `TokenManager.pm` — Scope-Erweiterung falls Code-Änderung nötig (Keymaster-Aufruf)
- `Plugin.pm` — Neue Scope in Auth-Flow registrieren (falls nicht automatisch via Dashboard)

</code_context>

<specifics>
## Specific Ideas

- Episodenlisten-Endpoint liefert resume_point direkt mit — kein separater Einzelabruf nötig für Phase 19 (Browse)
- Follow/Unfollow für Shows nutzt bestehende unified `/me/library` Endpoints (STATE.md v1.5 Decision)
- Show/Episode-Endpoints sind NICHT in `@KNOWN_DEPRECATED_FAMILIES` — kein spezielles Routing nötig

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 18-podcast-api-foundation*
*Context gathered: 2026-06-14*
