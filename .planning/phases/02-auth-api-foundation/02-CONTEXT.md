# Phase 2: Auth + API Foundation - Context

**Gathered:** 2026-05-27
**Status:** Ready for planning

<domain>
## Phase Boundary

Das Plugin kann Spotify-Tokens über Keymaster/login5 beschaffen, cachen und automatisch erneuern. Alle ausgehenden Spotify-API-Aufrufe laufen durch einen zentralen, rate-limitierten, cachenden HTTP-Client (`API/Client.pm`). Mehrere Spotify-Konten können konfiguriert und im Menü gewechselt werden. Credentials werden sicher im librespot-Cache gespeichert (chmod 600/700).

</domain>

<decisions>
## Implementation Decisions

### Credential-Eingabe & Speicherung
- **D-01:** Credential-Eingabemethode — Research-Phase evaluiert Username/Passwort (Settings-Formular) vs. Zeroconf-Discovery (Spotify-App). Beide Ansätze sind architekturkompatibel.
- **D-02:** Credentials werden im librespot-Cache-Verzeichnis gespeichert (`prefs/spoton/`), nicht in LMS Prefs. Auth-Blob im librespot-nativen Format. Verzeichnis chmod 700, Credential-Datei chmod 600 (AUTH-04).
- **D-03:** Token-Beschaffung via kurzlebigem Prozess — Plugin spawnt librespot mit `--get-token`, erhält Token über stdout, Prozess beendet sich. Kein persistenter Auth-Daemon nur für Tokens.
- **D-04:** Proaktiver Timer für Token-Erneuerung — LMS-Timer erneuert den Token vor Ablauf (z.B. bei 45 von 60 Minuten). Kein API-Call schlägt wegen abgelaufenem Token fehl.

### Multi-Account & Account-Wechsel
- **D-05:** Globale Kontoliste mit Menü-Switcher. Konten werden in den Plugin-Settings konfiguriert. Im OPML-Menü erscheint ein Account-Switcher als erste Zeile ("Aktiv: [Name] [wechseln]"). Wechsel ändert den aktiven Account für Browse/Search/Library und refresht das Menü via `nextWindow => 'refreshOrigin'`.
- **D-06:** Multi-Account bereits in Phase 2 implementiert — Settings-Seite, API/Client und Token-Management arbeiten von Anfang an mit Account-IDs. Kein späteres Refactoring nötig.
- **D-07:** Settings-Seite zeigt dynamische Kontoliste mit Hinzufügen/Entfernen. Keine festen Slots.
- **D-08:** Connect (Phase 5) funktioniert unabhängig von konfigurierten Settings-Konten. Spotify-App authentifiziert direkt am librespot-Daemon via Zeroconf. Nur Browse/Search/Library benötigt ein konfiguriertes Konto.

### Rate-Limiting-Strategie
- **D-09:** Rate-Limiting-Mechanismus (Token Bucket vs. Sliding Window vs. Adaptive) — Research-Phase evaluiert basierend auf Spotify-API-Verhalten und LMS Single-Thread-Architektur.
- **D-10:** Concurrency-Limit für gleichzeitige API-Requests — Research-Phase bestimmt optimalen Wert.
- **D-11:** Request-Queue-Priorisierung (High/Normal) — Research-Phase evaluiert Notwendigkeit.
- **D-12:** Bei aktiver Drosselung erscheint ein Hinweis im OPML-Menü ("Spotify-Anfragen gedrosselt"). Transparenz für den Nutzer.

### Batch-API-Entfall & API-Architektur
- **D-13:** Rein Einzel-Requests — keine Batch-Abstraktion. Extended Quota ist für Open-Source-Plugins unrealistisch (250k MAU + kommerzielle Org). YAGNI.
- **D-14:** API-Client-Modulstruktur (monolithisch vs. geschichtet) — Research-Phase evaluiert optimale Perl-Modul-Struktur für LMS-Plugins.
- **D-15:** Phase 2 implementiert nur Auth-relevante Endpoints: Token-Management, `GET /me` (Account-Validierung), Error-Handling, Rate-Limiting-Infrastruktur. Browse/Search/Library-Endpoints kommen in Phase 3.
- **D-16:** Response-Caching über LMS-Cache (`Slim::Utils::Cache` mit Namespace `spoton`). Persistiert über Neustarts, TTL-Support eingebaut. Bewährtes Pattern (Spotty, Qobuz).

### Claude's Discretion
- Credential-Eingabemethode (D-01): Username/Passwort vs. Zeroconf — Research entscheidet nach Machbarkeitsanalyse
- Rate-Limiting-Mechanismus (D-09): Token Bucket vs. Sliding Window vs. Adaptive — Research entscheidet
- Concurrency-Limit (D-10): Optimaler Wert basierend auf Spotify-Rate-Limits
- Queue-Priorisierung (D-11): Ob High/Normal-Trennung nötig ist
- API-Client-Modulstruktur (D-14): Monolithisch vs. geschichtet — Research entscheidet

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Projekt-Kontext
- `.planning/PROJECT.md` — Keymaster-only Auth-Entscheidung, Core Value, Constraints
- `.planning/REQUIREMENTS.md` — AUTH-01 bis AUTH-06, API-01 bis API-06 Anforderungsdefinitionen
- `.planning/phases/01-plugin-skeleton-binary-foundation/01-CONTEXT.md` — Phase 1 Entscheidungen (Binary-Strategie, Format `son`, Helper.pm)

### Technologie-Referenz
- `CLAUDE.md` §Technology Stack — LMS Plugin API Module, librespot CLI-Flags (--get-token, --lms-auth, --check), SimpleAsyncHTTP, Slim::Utils::Cache TTLs
- `CLAUDE.md` §Spotify Web API v1 — Endpoint-Status, Dev Mode Einschränkungen, OAuth Scopes, Rate Limits
- `CLAUDE.md` §Keymaster / login5 Authentication — Auth-Flow-Details

### Spotty-Referenz (Prior Art)
- https://github.com/michaelherger/Spotty-Plugin — Herger's Plugin (Token-Management-Muster, Account-Konfiguration, API.pm Architektur)
- https://github.com/michaelherger/librespot — Herger's aktiver librespot-Fork (--get-token Implementierung, login5-Patches)

### LMS-Referenz
- https://github.com/LMS-Community/slimserver — SimpleAsyncHTTP, Slim::Utils::Cache, Slim::Utils::Prefs
- https://github.com/LMS-Community/plugin-Qobuz — Qobuz-Plugin (API-Client-Referenz, Cache-Pattern, Account-Settings)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Helper.pm` — Binary-Discovery und `--check` Validierung. Wird für `--get-token` Spawning wiederverwendet (gleicher Binary-Pfad).
- `Settings.pm` — Basis-Settings mit `handler()`. Wird um dynamische Kontoliste erweitert.
- `Plugin.pm::handleFeed()` — OPML-Menü-Root. Account-Switcher wird hier als erstes Item eingefügt.
- `basic.html` — Settings-Template. Wird um Konto-Management erweitert.

### Established Patterns
- `$prefs = preferences('plugin.spoton')` — Prefs-Namespace etabliert, Account-Daten kommen hinzu
- `Slim::Utils::Log::logger('plugin.spoton')` — Log-Kategorie vorhanden
- `Slim::Player::ProtocolHandlers->registerHandler('spotify', ...)` — URI-Handler registriert
- Helper.pm `_findBin()` Pattern für Binary-Lokalisierung — wiederverwendbar für `--get-token` Aufrufe

### Integration Points
- `Plugin.pm::initPlugin()` → Token-Refresh-Timer starten, API-Client initialisieren
- `Plugin.pm::handleFeed()` → Account-Switcher einfügen, Auth-Status prüfen
- `Settings.pm::handler()` → Account-CRUD verarbeiten
- `Settings.pm::prefs()` → Neue Prefs registrieren (accounts, activeAccount)
- `basic.html` → Dynamische Kontoliste rendern

</code_context>

<specifics>
## Specific Ideas

- Account-Switcher im OPML-Menü als erste Zeile: "Aktiv: [Kontoname] [wechseln]" — Tap zeigt Kontoliste, Auswahl wechselt und refresht
- Menü-Hinweis bei Rate-Limiting-Drosselung sichtbar im OPML (nicht nur im Log)
- Settings-Seite: Binary-Status (aus Phase 1) plus dynamische Kontoliste mit Hinzufügen/Entfernen
- Token-Refresh-Timer soll konfigurierbar sein im Debug-Log (Intervall sichtbar)

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 2-Auth + API Foundation*
*Context gathered: 2026-05-27*
