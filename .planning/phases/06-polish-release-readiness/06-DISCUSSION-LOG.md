# Phase 6: Polish + Release Readiness - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-03
**Phase:** 06-polish-release-readiness
**Areas discussed:** Per-Player Settings, DSTM Track-Quelle, Setup Guide & Distribution, Transcoding Fallback

---

## Per-Player Settings

### Bitrate-Modell

| Option | Description | Selected |
|--------|-------------|----------|
| Global + Override | Globales Bitrate-Setting bleibt (320). Per-Player optionaler Override (96/160/320). Pattern wie OGG-Override. | ✓ |
| Nur per-Player | Kein globales Bitrate. Jeder Player eigener Wert. Mehr Klickarbeit. | |

**User's choice:** Global + Override
**Notes:** Bestehendes Pattern beibehalten

### Volume Normalisierung Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Global bleibt | Normalisierung bleibt globaler Toggle. Weniger UI-Komplexität. | ✓ |
| Global + per-Player Override | Sinnvoll wenn z.B. B&O eigene Normalisierung hat. | |
| Du entscheidest | Claude wählt basierend auf Research. | |

**User's choice:** Global bleibt
**Notes:** Alle Player gleiche Lautstärke-Angleichung

### Settings-Organisation

| Option | Description | Selected |
|--------|-------------|----------|
| Eine Sektion pro Player | Alles untereinander, Player-Dropdown wählt Player. Einfach. | ✓ |
| Gruppiert nach Funktion | Aufgeteilt in Streaming/Connect/Wiedergabe. Mehr Struktur. | |
| Du entscheidest | Claude wählt basierend auf LMS-Konventionen. | |

**User's choice:** Eine Sektion pro Player
**Notes:** —

### Client-ID Konsolidierung

| Option | Description | Selected |
|--------|-------------|----------|
| Client.pm | SPOTON_DEFAULT_CLIENT_ID bleibt in Client.pm. TokenManager importiert. | ✓ |
| Plugin.pm | Zentrale Stelle im Haupt-Modul. Höhere Sichtbarkeit. | |
| Du entscheidest | Claude wählt basierend auf Abhängigkeitsstruktur. | |

**User's choice:** Client.pm
**Notes:** Logischer Ort für API-Credentials

---

## DSTM Track-Quelle

### Track-Quelle

| Option | Description | Selected |
|--------|-------------|----------|
| Top Tracks | me/top/tracks. Einfach, zuverlässig, nicht kontextuell. | |
| Kürzlich gehört | me/player/recently-played. Risiko: Wiederholung. | |
| Queue | me/player/queue. Abhängig von Spotify Autoplay. | |
| Du entscheidest | Claude recherchiert Optionen. | |

**User's choice:** "Wir nutzen doch aber auch die Bundled ID von Herger im extended Quota Mode"
**Notes:** User erinnerte an Dual-Token-Architektur. Bundled-Token (Herger-ID) hat Extended Quota → recommendations-Endpoint möglicherweise noch verfügbar.

### DSTM-Strategie

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, mit Fallback | recommendations via bundled-Token, Search-Fallback bei 404/403. | ✓ |
| Nur Search-basiert | Kein recommendations-Versuch. Robuster, weniger vielfältig. | |
| Nur bundled-Token | Recommendations ohne Fallback. Einfacher, aber fragil. | |

**User's choice:** Ja, mit Fallback
**Notes:** Spotty-NG Seed-Logik übernehmen, aber über bundled-Token routen

### DSTM-Toggle

| Option | Description | Selected |
|--------|-------------|----------|
| LMS-Standard | SpotOn registriert sich als DSTM-Provider via LMS-Framework. | ✓ |
| Eigener Toggle + LMS | Zusätzlicher per-Player Toggle in SpotOn-Settings. Redundant. | |

**User's choice:** LMS-Standard
**Notes:** LMS-Konvention, kein eigener Toggle nötig

---

## Setup Guide & Distribution

### Guide-Platzierung

| Option | Description | Selected |
|--------|-------------|----------|
| Settings-Seite oben | Vor allen Einstellungen, kollabierbar. | |
| Eigene Hilfe-Seite | Separater Link. LMS hat kein natives Tab-Konzept. | |
| Du entscheidest | Claude wählt basierend auf Konventionen. | ✓ |

**User's choice:** Du entscheidest
**Notes:** —

### Credits

| Option | Description | Selected |
|--------|-------------|----------|
| Footer auf Settings-Seite | Dezenter Footer: "SpotOn nutzt librespot. Inspiriert von Hergers Spotty Plugin." | ✓ |
| Im Setup Guide | Prominentere Platzierung im Guide-Text. | |
| Du entscheidest | Claude platziert Credits passend. | |

**User's choice:** Footer auf Settings-Seite
**Notes:** Dezent aber sichtbar

### Repository-Distribution

| Option | Description | Selected |
|--------|-------------|----------|
| GitHub repo.xml | repo.xml als Raw-File im SpotOn-Repo. Standard-Weg für Third-Party Plugins. | ✓ |
| Eigener Server | Mehr Kontrolle, aber zusätzliche Infrastruktur. | |
| Du entscheidest | Claude recherchiert LMS Repository-Konventionen. | |

**User's choice:** GitHub repo.xml
**Notes:** —

### i18n-Umfang

| Option | Description | Selected |
|--------|-------------|----------|
| Claude übersetzt | Claude generiert alle Sprachen (EN, DE, FR, NL, IT, ES, SV, NO, DA, PL, CS). | ✓ |
| Nur EN+DE+FR | Reduzierter Umfang für v1. | |
| Community-first | EN+DE bleibt, Community übersetzt Rest. | |

**User's choice:** Claude übersetzt
**Notes:** Community kann später korrigieren

---

## Transcoding Fallback

### Fallback-Mechanismus

| Option | Description | Selected |
|--------|-------------|----------|
| Per-Player ForceTranscode | canDirectStream() gibt 0 zurück → LMS proxied via Pipeline. | ✓ |
| Auto-Detect | Plugin erkennt automatisch. Schwieriger, möglicherweise unzuverlässig. | |
| Du entscheidest | Claude recherchiert Player-Typen. | |

**User's choice:** Per-Player ForceTranscode
**Notes:** —

### Fallback-UI

| Option | Description | Selected |
|--------|-------------|----------|
| Separate Checkbox | "Transcoding erzwingen" als eigene Checkbox. Klar getrennt. | |
| In OGG-Override integriert | Dropdown erweitern: Auto/OGG/PCM/FLAC(transkodiert)/MP3(transkodiert). | ✓ |
| Du entscheidest | Claude wählt basierend auf Klarheit. | |

**User's choice:** In OGG-Override integriert
**Notes:** "mit ausreichender Erklärung im Mouseover" — Tooltips für jede Option

### Format-Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Einheitlich für beide Modi | Ein per-Player Format-Setting für Connect UND Browse. Deferred Item Phase 5. | ✓ |
| Nur Connect | Format-Dropdown nur für Connect. Browse nutzt globalen Mechanismus. | |

**User's choice:** Einheitlich für beide Modi
**Notes:** Deferred Item aus Phase 5 eingelöst

---

## Claude's Discretion

- Setup Guide Platzierung und Detailtiefe (D-07)
- Binary-Build-Pipeline für Multi-Architektur
- repo.xml Struktur und Versionierung
- Security/Code Review Scope
- DSTM Search-Fallback Details

## Deferred Ideas

None — discussion stayed within phase scope
