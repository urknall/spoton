# Phase 18: Podcast API Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-14
**Phase:** 18-podcast-api-foundation
**Areas discussed:** Scope-Erweiterung, Token-Routing Shows/Episodes, Resume-Point Caching

---

## Scope-Erweiterung

### Re-Auth-Handling bei fehlendem Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Graceful Degradation | resume_point fehlt → Episoden zeigen einfach keinen Fortschritt an. Kein Error, kein Re-Auth-Zwang. | |
| Proaktives Re-Auth | Cache-Schema-Version bumpen (3→4) → Token-Cache wird gelöscht → nächster Token-Fetch bekommt automatisch neue Scopes. | |
| Du entscheidest | Claude/Researcher klärt den Keymaster-Scope-Mechanismus und wählt die passende Strategie. | ✓ |

**User's choice:** Du entscheidest
**Notes:** Komplett an Researcher delegiert. Keymaster-Mechanismus muss zuerst verstanden werden bevor eine Strategie gewählt werden kann.

### Scope-Zuordnung zu Token-Flavors

| Option | Description | Selected |
|--------|-------------|----------|
| Nur own-Token | Resume-Point ist User-spezifisch → nur der own-Token braucht den Scope. | |
| Beide Tokens | Für den Fall dass Show/Episode-Endpoints über den bundled-Token laufen. | |
| Du entscheidest | Researcher klärt welcher Token-Flavor welche Endpoints bedient. | ✓ |

**User's choice:** Du entscheidest
**Notes:** Abhängig von Token-Routing-Klärung im nächsten Bereich.

---

## Token-Routing Shows/Episodes

### Routing-Strategie für Podcast-Endpoints

| Option | Description | Selected |
|--------|-------------|----------|
| Default reicht | Own-first Routing (D-04) deckt das ab. Shows/Episodes laufen wie artists/{id} oder albums/{id}. | |
| Expliziter Guard | Eigene Regex hinzufügen die shows/* und episodes/* immer auf own-Token erzwingt. | |
| Du entscheidest | Researcher soll klären ob ein expliziter Guard einen echten Vorteil bringt. | ✓ |

**User's choice:** Du entscheidest
**Notes:** Claude hat die bestehende Routing-Logik analysiert und erklärt: `_resolveStartFlavor` routet non-me/*-Endpoints per Default zum own-Token (D-04). Shows/Episodes sind nicht in `@KNOWN_DEPRECATED_FAMILIES`. Own-first funktioniert bereits korrekt.

---

## Resume-Point Caching

### Episodenlisten-TTL

| Option | Description | Selected |
|--------|-------------|----------|
| 0s — immer live | Jedes Öffnen einer Show holt frische Daten. Genauer Resume-Status, aber mehr API-Calls. | |
| 60s — kurz | Wie Library-Items (Liked Songs). Wiederholtes Öffnen innerhalb einer Minute nutzt Cache. | ✓ |
| 300s — moderat | Wie Playlist-Tracks. Weniger API-Last, aber Resume-Status kann bis 5 Minuten veraltet sein. | |

**User's choice:** 60s — kurz
**Notes:** Konsistent mit dem bestehenden Library-Items-Pattern. Einzige explizite User-Entscheidung in dieser Diskussion.

### Show-Metadata-TTL

| Option | Description | Selected |
|--------|-------------|----------|
| 3600s — 1 Stunde | Wie Artist/Album/Track-Metadata. Show-Info ändert sich selten. | |
| 60s — gleich wie Episoden | Einheitliches Caching. Einfacher, aber unnötig viele API-Calls. | |
| Du entscheidest | Researcher wählt basierend auf CLAUDE.md Cache-Empfehlungen. | ✓ |

**User's choice:** Du entscheidest

### getEpisode Einzelabruf-TTL

| Option | Description | Selected |
|--------|-------------|----------|
| 60s wie Episodenliste | Einheitlich. Kein doppelter Fetch. | |
| 0s — immer live | Expliziter Episode-Abruf bekommt immer aktuelle Daten. | |
| Du entscheidest | Researcher wählt passende Strategie. | ✓ |

**User's choice:** Du entscheidest

---

## Claude's Discretion

- Keymaster-Scope-Mechanismus und Re-Auth-Strategie
- Scope-Zuordnung zu Token-Flavors (own vs. bundled vs. beide)
- Token-Routing für Show/Episode-Endpoints (default vs. expliziter Guard)
- Show-Metadata Cache-TTL
- getEpisode Einzelabruf Cache-TTL
- Binary-Version-Bump falls librespot-Änderung nötig

## Deferred Ideas

None — discussion stayed within phase scope
