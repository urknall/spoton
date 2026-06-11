# Phase 16: macOS Universal Binary - Context

**Gathered:** 2026-06-11
**Status:** Ready for planning

<domain>
## Phase Boundary

macOS-Benutzer bekommen ein funktionierendes librespot-Binary über den LMS Plugin Manager — Intel (x86_64) + Apple Silicon (aarch64) als Universal Binary via `lipo`, ad-hoc signiert, mit Gatekeeper-Workaround-Dokumentation in README und Settings-Seite.

</domain>

<decisions>
## Implementation Decisions

### Binary-Verzeichnis (PLT-01)
- **D-01:** macOS Universal Binary liegt in `Bin/darwin/spoton` — ein Verzeichnis, ein Binary für beide Architekturen
- **D-02:** Konsistent mit Herger's Spotty-Pattern. Helper.pm braucht nur einen `ISMAC`-Block mit `addFindBinPaths` auf `Bin/darwin/`
- **D-03:** Universal Binary wird via `lipo -create -output` aus x86_64 und aarch64 Builds erzeugt

### Build-Trigger
- **D-04:** macOS-Build nur bei Tag-Releases (`v*`) und `workflow_dispatch` — gleicher Trigger wie bestehender Linux/Windows-Build. Keine PR/Push-Builds (CI-Minuten sparen)
- **D-05:** macOS Universal Binary wird als `spoton-darwin` Artifact im GitHub Release veröffentlicht — konsistent mit den bestehenden Plattform-Artifacts

### Code-Signing (PLT-02)
- **D-06:** Ad-hoc Code-Signing (`codesign --force --sign -`) reicht aus — LMS Plugin Manager setzt kein quarantine-xattr (v1.3 Research bestätigt)

### Gatekeeper-Dokumentation (PLT-03)
- **D-07:** README.md bekommt macOS in die Platform-Liste + kurzen xattr-Hinweis für manuelle Downloads
- **D-08:** Settings-Seite (basic.html) zeigt dynamischen Gatekeeper-Hinweis wenn Binary-Check auf macOS fehlschlägt
- **D-09:** Gatekeeper-Warnung in strings.txt für alle 11 Sprachen (i18n via bestehendem Pattern)

### Claude's Discretion
- CI-Workflow-Struktur: Ob bestehender `build-librespot.yml` erweitert oder separater macOS-Workflow. Researcher/Planner entscheidet basierend auf GitHub Actions Best Practices und Workflow-Komplexität
- macOS Runner-Auswahl: Welche spezifischen GitHub Runner-Labels (macos-13 vs macos-14 vs macos-15) für Intel und ARM Builds
- Lipo-Job-Platzierung: Ob der Universal Binary Merge als separater Job oder innerhalb eines bestehenden Jobs läuft

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### CI Infrastructure
- `.github/workflows/build-librespot.yml` — Bestehender Build-Workflow für 6 Linux-Targets + Windows. Pattern-Referenz für Matrix-Strategie, Artifact-Upload, Release-Job
- `.github/workflows/perl-tests.yml` — Perl CI (Phase 13). Nur Referenz für Actions-Patterns

### Binary Distribution
- `Plugins/SpotOn/Helper.pm` — Binary-Detection und Platform-Routing. Muss um ISMAC-Block für `Bin/darwin/` erweitert werden (Zeile 21-32 zeigt bestehendes Pattern)
- `Plugins/SpotOn/Bin/` — Binary-Verzeichnisstruktur (6 Linux + 1 Windows). `darwin/` muss hinzugefügt werden

### Existing Docs
- `README.md` Zeile 36 — Aktuelle Platform-Liste (macOS als "not yet included" erwähnt)
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Settings-Seite für Gatekeeper-Hinweis
- `Plugins/SpotOn/strings.txt` — i18n Strings (11 Sprachen)

No external specs — requirements fully captured in decisions above

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Helper.pm` Zeile 28-32: Windows `addFindBinPaths`-Block — exaktes Pattern für den neuen ISMAC-Block
- `build-librespot.yml` Matrix-Strategie: `target`, `bin_dir`, `binary_ext`, `use_cross` Felder — erweiterbar um macOS-spezifische Felder (`runs-on`)
- `build-librespot.yml` Release-Job: Artifact-Download + Checksum-Erstellung — macOS-Binary integriert sich nahtlos

### Established Patterns
- Binary-Verzeichnisse folgen `{descriptor}-{os}` Schema (z.B. `x86_64-linux`, `x86_64-win64`)
- `lipo` Universal Binary bricht dieses Schema bewusst: `darwin/` statt `x86_64-darwin/` — weil ein Binary beide Architekturen abdeckt
- Helper.pm nutzt `main::ISMAC` und `main::ISWINDOWS` LMS-Konstanten für Platform-Detection

### Integration Points
- Helper.pm `init()`: Neuer `if (main::ISMAC)` Block nach dem bestehenden Windows-Block (Zeile 32)
- `build-librespot.yml` Matrix: Neue Einträge für macOS-Targets mit eigenem `runs-on` statt `ubuntu-latest`
- `build-librespot.yml` Jobs: Neuer `lipo`-Job der nach den macOS-Builds läuft
- README.md: Platform-Liste aktualisieren
- `basic.html` + `strings.txt`: Gatekeeper-Warnung hinzufügen

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches

</specifics>

<deferred>
## Deferred Ideas

- **PLT-04: Developer ID Signing + Notarization** — Erfordert $99/Jahr Apple Developer Membership. Bereits als Future Requirement erfasst. Nicht Teil dieser Phase.

</deferred>

---

*Phase: 16-macos-universal-binary*
*Context gathered: 2026-06-11*
