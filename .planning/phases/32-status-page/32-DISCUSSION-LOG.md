# Phase 32: Status Page - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-25
**Phase:** 32-status-page
**Areas discussed:** Daten-Umfang, Update-Mechanismus, Seitenstruktur, Zugänglichkeit

---

## Daten-Umfang

### Welche Daten-Kategorien?

| Option | Description | Selected |
|--------|-------------|----------|
| Daemon-Health | Pro Player: PID, Uptime, alive/dead, Stream-Port, Connect-Modus, Crash-Loop-Status | ✓ |
| API & Token | Inflight-Requests, Rate-Limit-Status, Token-Zustand, letzter Refresh, Account-Übersicht | ✓ |
| Fehler-Übersicht | Letzte Fehler aus Log-Files, Log-Dateigrößen, Browse-Errors | ✓ |
| System-Info | Binary-Version + Capabilities, LMS-Version, Perl-Version, OS, Plugin-Version | ✓ |

**User's choice:** All four categories selected.

### Diagnostic Mode Abhängigkeit?

| Option | Description | Selected |
|--------|-------------|----------|
| Immer Basis-Daten | Basis immer verfügbar, DIAG-Events nur bei aktiviertem Mode | |
| Alles immer | Alle Daten immer sammeln und zeigen | ✓ |
| Nur bei Diag-Mode | Ganze Status Page nur bei Diagnostic Mode sichtbar | |

**User's choice:** "Alles immer" — User fragte "Was spricht gegen immer?" → Antwort: fast nichts, da Live-State schon im Speicher ist und In-Memory-Counter vernachlässigbar sind.
**Notes:** User wollte Klarstellung, warum die Optionen so aufgeteilt waren. Kein echter Nachteil für "immer an".

---

## Update-Mechanismus

### Polling-Strategie

| Option | Description | Selected |
|--------|-------------|----------|
| Auto-Polling (5s) | JavaScript pollt alle 5s den JSON-Endpoint | ✓ |
| Auto-Polling (2s) | Aggressiveres Polling | |
| Manueller Refresh | Nur bei Seitenaufruf oder Button-Klick | |
| Hybrid | Auto für kritisch, manuell für statisch | |

**User's choice:** "Auto-Polling alle 5sek, aber Dinge die sich nie ändern z.B. System-Info nicht."
**Notes:** User bestätigte nach Erklärung: System-Info ist Plugin/Binary/LMS-Version — ändert sich nur bei Restart.

### Tab-Visibility

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, pausieren | Polling stoppt bei inaktivem Tab (visibilitychange) | ✓ |
| Nein, immer pollen | Einfacher, Load bei 5s egal | |
| Du entscheidest | Claude wählt | |

**User's choice:** Ja, pausieren.

---

## Seitenstruktur

### Platzierung im LMS

| Option | Description | Selected |
|--------|-------------|----------|
| Eigene Seite | Unter eigenem URL, Link aus Settings, freies Layout | ✓ |
| Tab in Settings | Zusätzlicher Tab neben Basic/Player, LMS-Framework | |
| Standalone HTML | Komplett eigenständig, maximale Freiheit | |

**User's choice:** Eigene Seite (mit Preview: Settings → Link → Dashboard in neuem Tab)

### Layout

| Option | Description | Selected |
|--------|-------------|----------|
| Kachel-Grid | Sektionen als Cards in 2x2 Grid, responsive | ✓ |
| Vertikale Liste | Sektionen untereinander, volle Breite | |

**User's choice:** Kachel-Grid (mit Preview: 2x2 Grid mit Daemon/API/Errors/System)

### Styling

| Option | Description | Selected |
|--------|-------------|----------|
| Eigenes Dark Theme | Eigenständiges modernes Dark-Theme | |
| LMS-Theme erben | CSS-Variablen vom LMS nutzen | |
| Du entscheidest | Claude wählt | ✓ |

**User's choice:** Du entscheidest.

---

## Zugänglichkeit

### Zugangsrechte

| Option | Description | Selected |
|--------|-------------|----------|
| Immer offen | Jeder mit LMS-Zugang kann die Seite sehen | |
| Nur mit LMS-Auth | Login erforderlich wenn Auth konfiguriert | ✓ |
| Nur Admins | Nur Admin-Benutzer | |

**User's choice:** Nur mit LMS-Auth.

### Aktionen

| Option | Description | Selected |
|--------|-------------|----------|
| Rein informativ | Nur Anzeige, keine Aktions-Buttons | ✓ |
| Mit Basis-Aktionen | Daemon-Restart, Log-Clear, Token-Refresh | |
| Du entscheidest | Claude wählt | |

**User's choice:** Rein informativ.

---

## Claude's Discretion

- **Styling:** Dark theme vs. light vs. LMS-inherited — Claude decides
- **Error history depth:** How many entries to keep in ring buffer
- **Card detail level:** Exact metrics and formatting per card

## Deferred Ideas

None — discussion stayed within phase scope.
