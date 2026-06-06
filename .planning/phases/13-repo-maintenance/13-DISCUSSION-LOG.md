# Phase 13: Repo Maintenance - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-06
**Phase:** 13-repo-maintenance
**Areas discussed:** CI-Workflow Design, Issue Template Gestaltung, CONTRIBUTING.md Umfang, Repo Hygiene

---

## CI-Workflow Design

### CI Trigger

| Option | Description | Selected |
|--------|-------------|----------|
| Push + PR auf main | Jeder Push und jeder PR gegen main löst Tests aus | ✓ |
| Nur PR gegen main | Tests nur bei Pull Requests | |
| Push auf alle Branches + PR | Maximale Abdeckung | |

**User's choice:** Push + PR auf main
**Notes:** Standard für kleine Projekte

### Perl Setup

| Option | Description | Selected |
|--------|-------------|----------|
| shogo82148/actions-setup-perl | Spezialisierte Action, Perl 5.36 + 5.38 als Matrix | ✓ |
| Ubuntu apt + perlbrew | System-Perl + perlbrew für Versionen | |
| Docker Container | Offizielles perl Image | |

**User's choice:** shogo82148/actions-setup-perl
**Notes:** Community-Standard für Perl-CI

---

## Issue Template Gestaltung

### Template Format

| Option | Description | Selected |
|--------|-------------|----------|
| YAML Forms | Strukturierte Formulare mit Dropdowns, Pflichtfeldern, Validierung | ✓ |
| Markdown Templates | Klassisches Format mit Platzhaltern | |

**User's choice:** YAML Forms

### Bug Report Felder

**User's choice:** Alle vier Felder ausgewählt:
- LMS Version + OS (Pflicht)
- Reproduktionsschritte (Pflicht)
- Log-Auszug (Optional)
- Player-Typ (Optional)

**Notes:** User schlug zusätzlich ein Debug-Info-Tool vor, das relevante Logs filtert und paketiert — als Deferred Idea notiert (eigene Phase).

---

## CONTRIBUTING.md Umfang

### Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Kompakt: Setup + Tests + PR | 1-2 Seiten, unkompliziert | ✓ |
| Ausführlich mit Architektur | 3-4 Seiten mit Modul-Überblick | |
| Minimal: nur Tests + PR | Halbe Seite | |

### Sprache

| Option | Description | Selected |
|--------|-------------|----------|
| Englisch | Standard für Open-Source | ✓ |
| Deutsch | Passt zum LMS-Community-Kern | |

**User's choice:** Kompakt auf Englisch

---

## Repo Hygiene

**User's choice:** Alle vier Bereiche ausgewählt:
- .gitignore aufräumen
- LICENSE-Datei hinzufügen (MIT)
- README aktualisieren (CI-Badge, Features, Install)
- Alte Artefakte bereinigen (zip-Dateien, Phase-Deletes)

### Lizenz

| Option | Description | Selected |
|--------|-------------|----------|
| MIT License | Permissiv, LMS-Community-Standard | ✓ |
| GPL v3 | Copyleft, strenger als nötig | |
| GPL v2 | Gleiche Lizenz wie LMS | |
| Apache 2.0 | Selten im LMS-Ökosystem | |

**User's choice:** MIT License

---

## Claude's Discretion

- CI Job-Timeouts und Retry-Strategie
- YAML-Feldvalidierung und Dropdown-Optionen in Issue Templates
- CONTRIBUTING.md Formatierung und Gliederung
- config.yml für Issue Template Chooser

## Deferred Ideas

- Debug-Info-Tool: Automatisiertes Tool für Log-Filterung und Debug-Info-Paketierung
- Changelog Automation: Automatische Changelog-Generierung aus Commits/PRs
