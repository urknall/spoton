# Phase 13: Repo Maintenance - Context

**Gathered:** 2026-06-06
**Status:** Ready for planning

<domain>
## Phase Boundary

GitHub repo bekommt eine funktionierende CI-Pipeline und Contributor-Infrastruktur: Perl-Tests in GitHub Actions, strukturierte Issue Templates, CONTRIBUTING.md, LICENSE, und Repo-Hygiene.

</domain>

<decisions>
## Implementation Decisions

### CI-Workflow
- **D-01:** CI triggert auf Push und Pull Requests gegen `main`
- **D-02:** Perl-Setup via `shogo82148/actions-setup-perl` Action mit Matrix: 5.36 + 5.38
- **D-03:** Workflow-Datei: `.github/workflows/perl-tests.yml`
- **D-04:** Tests laufen standalone mit LMS-Stubs (keine LMS-Installation nötig in CI)

### Issue Templates
- **D-05:** YAML Forms Format (`.github/ISSUE_TEMPLATE/*.yml`) — strukturiert mit Validierung
- **D-06:** Bug Report Felder: LMS Version + OS (Pflicht), Reproduktionsschritte (Pflicht), Log-Auszug (Optional), Player-Typ (Optional)
- **D-07:** Feature Request Template mit Problem Statement und Alternatives-Feldern

### CONTRIBUTING.md
- **D-08:** Kompakter Umfang: Entwicklungsumgebung einrichten, Tests laufen lassen, PR-Guidelines — 1-2 Seiten
- **D-09:** Sprache: Englisch

### Repo Hygiene
- **D-10:** `.gitignore` aufräumen — SpotOn-*.zip und Build-Artefakte sauber ignorieren
- **D-11:** LICENSE-Datei hinzufügen: MIT License
- **D-12:** README aktualisieren — CI-Badge, aktuelle Feature-Liste, Install-Anweisungen
- **D-13:** Alte Artefakte bereinigen — SpotOn-v1.2.x.zip aus Root entfernen, unstaged Phase-Deletes committen

### Claude's Discretion
- CI Job-Timeouts und Retry-Strategie
- Genaue YAML-Feldvalidierung und Dropdown-Optionen in Issue Templates
- CONTRIBUTING.md Formatierung und Gliederung
- config.yml für Issue Template Chooser (optional)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing CI
- `.github/workflows/build-librespot.yml` — Existing binary build workflow (reference for Actions patterns)

### Test Suite
- `t/05_perl_syntax.t` — Perl syntax check with LMS stub framework (reference for CI test dependencies)
- `t/` — Full test directory (12 tests total)

### Project Config
- `.gitignore` — Current ignore rules (needs update)
- `install.xml` — Plugin manifest (version info for README)

No external specs — requirements fully captured in decisions above

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `t/05_perl_syntax.t` stub framework: comprehensive LMS module stubs that allow standalone Perl testing without LMS installation — CI can use `prove t/` directly
- `.github/workflows/build-librespot.yml`: existing Actions workflow with matrix strategy — pattern reference for the Perl test workflow

### Established Patterns
- Test files use `Test::More`, `File::Basename`, `Cwd` — standard Perl test modules
- Tests are numbered sequentially (`01_` through `12_`)
- All tests use `#!/usr/bin/perl` shebang with strict/warnings

### Integration Points
- CI badge will link to the new `perl-tests.yml` workflow in README
- Issue templates reference SpotOn-specific concepts (LMS version, player type, Connect mode)

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches

</specifics>

<deferred>
## Deferred Ideas

- **Debug-Info-Tool:** Automatisiertes Tool das relevante Log-Einträge/Debug-Infos filtert, zusammenstellt und für die Übermittlung paketiert — eigene Phase, nicht Teil von Repo Maintenance
- **Changelog Automation:** Automatische Changelog-Generierung aus Commits/PRs — nice-to-have, nicht kritisch für v1.3

</deferred>

---

*Phase: 13-repo-maintenance*
*Context gathered: 2026-06-06*
