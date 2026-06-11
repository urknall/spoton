# Phase 16: macOS Universal Binary - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-11
**Phase:** 16-macos-universal-binary
**Areas discussed:** Binary-Verzeichnis, Build-Workflow, Gatekeeper-Doku

---

## Binary-Verzeichnis

| Option | Description | Selected |
|--------|-------------|----------|
| darwin/ (Empfohlen) | Ein Verzeichnis, ein Universal Binary. Konsistent mit Herger's Spotty-Pattern. Helper.pm braucht nur einen ISMAC-Block. | ✓ |
| x86_64-darwin/ + aarch64-darwin/ | Zwei Verzeichnisse, architektur-spezifische Binaries. Folgt der {arch}-{os} Konvention der Linux-Targets. | |
| Claude entscheidet | Researcher/Planner entscheidet basierend auf Spotty-Referenz und LMS-Konventionen. | |

**User's choice:** darwin/ — ein Universal Binary
**Notes:** Konsistent mit Spotty, minimaler Helper.pm-Aufwand

### Release-Distribution

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, im Release | macOS-Binary als spoton-darwin Artifact im GitHub Release veröffentlicht | ✓ |
| Nur im Plugin-ZIP | macOS-Binary nur über LMS Plugin Manager verteilt | |
| Claude entscheidet | Researcher/Planner entscheidet | |

**User's choice:** Im GitHub Release als separater Download
**Notes:** Konsistent mit bestehenden Plattform-Artifacts

---

## Build-Workflow

### CI-Integration

| Option | Description | Selected |
|--------|-------------|----------|
| build-librespot.yml erweitern (Empfohlen) | Zwei neue Matrix-Einträge + lipo-Job in bestehender Workflow-Datei | |
| Eigener macOS-Workflow | Separate build-macos.yml. Saubere Trennung, dupliziert Setup-Logic. | |
| Claude entscheidet | Researcher/Planner entscheidet basierend auf CI-Best-Practices | ✓ |

**User's choice:** Claude entscheidet
**Notes:** Workflow-Struktur ist Claude's Discretion

### Build-Trigger

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Tags + manual (Empfohlen) | Wie bestehender Build: v* Tags und workflow_dispatch. Spart CI-Minuten. | ✓ |
| Auch bei Push/PR | Jeder Push/PR baut auch macOS. Früh Fehler finden, aber teurer. | |
| Claude entscheidet | Researcher/Planner entscheidet | |

**User's choice:** Nur Tags + manueller Trigger
**Notes:** macOS-Runners kosten mehr CI-Minuten

---

## Gatekeeper-Doku

### Dokumentations-Platzierung

| Option | Description | Selected |
|--------|-------------|----------|
| README + Settings-Seite (Empfohlen) | README Platform-Liste + dynamischer Gatekeeper-Hinweis in basic.html bei Binary-Check-Fehler | ✓ |
| Nur README | Nur im README, kein dynamischer Hinweis in Settings | |
| Claude entscheidet | Researcher/Planner entscheidet | |

**User's choice:** README + Settings-Seite mit dynamischem Hinweis
**Notes:** Settings-Hinweis erscheint nur wenn Binary-Check auf macOS fehlschlägt

### i18n

| Option | Description | Selected |
|--------|-------------|----------|
| i18n (Empfohlen) | Gatekeeper-Warnung als String in strings.txt für alle 11 Sprachen | ✓ |
| Nur Englisch | Terminal-Befehl ist sowieso Englisch, Hinweis bleibt hardcoded | |
| Claude entscheidet | Researcher/Planner entscheidet | |

**User's choice:** i18n via strings.txt
**Notes:** Konsistent mit bestehender 11-Sprachen-Abdeckung

---

## Claude's Discretion

- CI-Workflow-Struktur (eigener Workflow vs erweitert)
- macOS Runner-Auswahl (spezifische Labels)
- Lipo-Job-Platzierung

## Deferred Ideas

- PLT-04: Developer ID Signing + Notarization (requires $99/year Apple Developer membership)
