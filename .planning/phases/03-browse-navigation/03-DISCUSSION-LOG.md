# Phase 3: Browse + Navigation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-28
**Phase:** 3-Browse + Navigation
**Areas discussed:** Home-Feed-Aufbau, Track-Interaktion, Such-Ergebnisse, Library & Pagination

---

## Home-Feed-Aufbau

### Top-Level-Menüstruktur

| Option | Description | Selected |
|--------|-------------|----------|
| NAV-Standard | Home / Suche / Library als drei Top-Level-Einträge | ✓ |
| Flacher Start | Alles direkt auf Top-Level ohne Grouping | |
| Spotty-orientiert | Research analysiert Spotty-Struktur und übernimmt bewährte Patterns | |

**User's choice:** NAV-Standard
**Notes:** User korrigierte initial zu enge Home-Feed-Perspektive — die Frage muss die gesamte Menüstruktur (Home + Suche + Library) zeigen, nicht nur Home isoliert.

### Made-For-You-Erkennung

| Option | Description | Selected |
|--------|-------------|----------|
| Owner-Filter | Playlists mit owner.id = 'spotify' als Made For You einstufen | |
| Name-Pattern-Match | Playlists nach bekannten Namen filtern (Daily Mix, Discover Weekly etc.) | |
| Du entscheidest | Research analysiert Lösung | |

**User's choice:** "Mach Research dazu, aber ich würde ein Match nach Namen nicht befürworten. Lieber ein eindeutiges generisches Merkmal, wie die owner.id"
**Notes:** Klare Ablehnung von Name-Pattern-Matching (sprachabhängig, fragil). Richtung: generisches Feld wie owner.id.

### Top Tracks time_range

| Option | Description | Selected |
|--------|-------------|----------|
| medium_term (Empfohlen) | 6 Monate — gute Balance zwischen aktuell und repräsentativ | ✓ |
| short_term | 4 Wochen — sehr aktuell | |
| Alle drei anbieten | 3 Sub-Einträge mit verschiedenen Zeiträumen | |

**User's choice:** medium_term
**Notes:** —

---

## Track-Interaktion

### Track-Tap-Verhalten

| Option | Description | Selected |
|--------|-------------|----------|
| Play-Intent setzen | Track als spotify:// URI in Playlist einreihen | ✓ |
| Track-Info anzeigen | Detail-Ansicht ohne Play-Versuch | |
| Phase 3+4 zusammen | Streaming-Basics gleich mit einbauen | |

**User's choice:** Play-Intent setzen
**Notes:** —

### Kontextmenü-Aktionen

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, Artist + Album | Zusätzliche Navigation zu Artist- und Album-Detailseite | ✓ |
| Nur Play-Action | Track-Tap = Play, keine weiteren Aktionen | |
| Du entscheidest | Research analysiert OPML-Möglichkeiten | |

**User's choice:** Ja, Artist + Album
**Notes:** —

### Alle abspielen

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, erster Eintrag | "Alle abspielen" als erster Eintrag in Album/Playlist-Tracklisten | |
| Nein, nur Einzeltracks | Kein "Alle abspielen", User wählt einzelne Tracks | ✓ |
| Du entscheidest | Research prüft Spotty/Qobuz-Praxis | |

**User's choice:** Nein, nur Einzeltracks
**Notes:** —

### Artist-Detailseite

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Discography | Alben, Singles, Compilations | |
| Discography + Appears On | Plus "Erscheint auf" via include_groups=appears_on | ✓ |
| Du entscheidest | Research prüft Dev Mode Verfügbarkeit | |

**User's choice:** Discography + Appears On
**Notes:** —

---

## Such-Ergebnisse

### Ergebnis-Gruppierung

| Option | Description | Selected |
|--------|-------------|----------|
| Sub-Menüs pro Typ | 4 Einträge (Tracks, Alben, Künstler, Playlists) | |
| Gemischte Liste | Alle Typen in einer flachen Liste | |
| Top-Ergebnis + Rest | Bestes Match oben, dann Sub-Menüs pro Typ | ✓ |

**User's choice:** Top-Ergebnis + Rest
**Notes:** —

### Ergebnisse pro Kategorie

| Option | Description | Selected |
|--------|-------------|----------|
| 10 (Maximum) | Volle 10 Ergebnisse pro Typ laden | |
| 5 (Spotify Default) | 5 pro Typ, Dev Mode Default | |
| Du entscheidest | Research entscheidet | ✓ |

**User's choice:** Du entscheidest
**Notes:** —

### Leere Kategorien

| Option | Description | Selected |
|--------|-------------|----------|
| Nur mit Ergebnissen | Leere Kategorien ausblenden | ✓ |
| Immer alle zeigen | Konsistente Struktur auch wenn leer | |
| Du entscheidest | Research prüft UX | |

**User's choice:** Nur mit Ergebnissen
**Notes:** —

---

## Library & Pagination

### Pagination

**User's choice:** LMS-internes OPMLBased-Framework nutzen
**Notes:** User wies darauf hin, dass LMS eingebaute Pagination hat (index/quantity). Kein manuelles "Mehr laden"-Pattern nötig.

### Sortierung

| Option | Description | Selected |
|--------|-------------|----------|
| Nur recently added | Feste Sortierung | |
| Umschaltbar | Zusätzliche Sortieroptionen | |
| Du entscheidest | Research prüft API-Möglichkeiten | ✓ |

**User's choice:** Du entscheidest
**Notes:** —

### Track-Metadaten

| Option | Description | Selected |
|--------|-------------|----------|
| Kompakt: Artist – Dauer | Standard für die meisten Listen | |
| Kontextabhängig | Verschiedene Formate je nach Kontext | |
| Du entscheidest | Research prüft OPML-Support | ✓ |

**User's choice:** Du entscheidest
**Notes:** —

### Artwork

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, Artwork anzeigen | Album-Cover, Playlist-Bilder, Artist-Fotos als Icons | ✓ |
| Nein, nur Text | Keine Bilder in Listen | |
| Du entscheidest | Research prüft Performance | |

**User's choice:** "Ja, die Icons brauchen wir. Bitte prüfe die Caching Möglichkeiten. Ich glaube Spotty hat mit Cache gearbeitet"
**Notes:** User erinnert sich explizit an Spotty's Bild-Caching. Research soll Caching-Ansatz untersuchen.

---

## Claude's Discretion

- Suchergebnisse pro Kategorie (5 vs 10)
- Library-Sortierung (fest vs. umschaltbar)
- Track-Metadaten-Format (OPML line2/subtext Möglichkeiten)
- Spotify-API → LMS-Pagination-Mapping

## Deferred Ideas

None — discussion stayed within phase scope
