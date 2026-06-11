---
phase: 16-macos-universal-binary
verified: 2026-06-11T15:42:12Z
status: human_needed
score: 9/9 must-haves verified
overrides_applied: 0
human_verification:
  - test: "LMS Plugin Manager installiert SpotOn auf einem macOS-Host — Universal Binary wird gefunden und als Helper erkannt"
    expected: "LMS erkennt das Binary in Bin/darwin/spoton, HelperVersion wird im Settings-Panel angezeigt"
    why_human: "Erfordert macOS-Host mit laufendem LMS; main::ISMAC ist ein LMS-Compile-Time-Constant, der auf Linux-Testsystemen nicht gesetzt ist"
  - test: "Gatekeeper-Hint erscheint im Settings-Panel wenn Binary fehlt auf macOS"
    expected: "Orange Div mit PLUGIN_SPOTON_GATEKEEPER_HINT-Text unterhalb der roten BINARY_MISSING-Meldung sichtbar"
    why_human: "Erfordert macOS-Host mit LMS; isMac=1-Pfad kann nur auf echtem macOS-System durchlaufen werden"
  - test: "GitHub Actions Workflow produziert spoton-darwin Universal Binary bei tag-Push"
    expected: "GitHub Release enthält spoton-darwin-Artifact mit zwei Architekturen (lipo -info bestätigt x86_64 + arm64)"
    why_human: "CI-Workflow kann nur in GitHub Actions verifiziert werden; kein lokales Testen moeglich"
---

# Phase 16: macOS Universal Binary — Verification Report

**Phase Goal:** macOS users can install SpotOn via the LMS plugin manager and have a working librespot binary without manual steps beyond a one-time Gatekeeper workaround
**Verified:** 2026-06-11T15:42:12Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| #  | Truth | Status | Evidence |
|----|-------|--------|----------|
| 1  | D-01: Bin/darwin/ directory contains eine einzelne Datei (Universal Binary placeholder) | VERIFIED | `Plugins/SpotOn/Bin/darwin/.gitkeep` existiert; CI ersetzt .gitkeep durch das Universal Binary beim Build |
| 2  | D-02: Helper.pm ISMAC-Block mit addFindBinPaths auf Bin/darwin/ | VERIFIED | Zeile 34-38 in Helper.pm: `if ( main::ISMAC ) { Slim::Utils::Misc::addFindBinPaths(..., 'Bin', 'darwin') }` |
| 3  | D-03: lipo-Job mergt beide Architekturen via lipo -create -output | VERIFIED | Workflow-Zeile 168: `lipo -create -output spoton arm64/${{ env.BINARY_NAME }} x86_64/${{ env.BINARY_NAME }}` |
| 4  | D-04: Nur tag-Push (v*) oder workflow_dispatch triggert Builds — keine PR/push-Builds | VERIFIED | Workflow on:-Block enthält nur `workflow_dispatch` und `push: tags: - 'v*'` |
| 5  | D-05: Release-Job enthält das macOS Universal Binary als spoton-darwin im GitHub Release | VERIFIED | needs: [build, lipo]; Intermediate-Artefakte werden entfernt; `release-artifacts/**/spoton`-Glob matcht `spoton-darwin/spoton` |
| 6  | D-06: Universal Binary ist ad-hoc code-signiert via codesign --force --sign - | VERIFIED | Workflow-Zeile 172: `codesign --force --sign - spoton` |
| 7  | D-07: README listet macOS in der Plattform-Liste mit xattr-Workaround | VERIFIED | README.md Zeile 36: "macOS (Universal Binary: Intel + Apple Silicon). On macOS... xattr -d com.apple.quarantine" |
| 8  | D-08: Settings-Seite zeigt dynamischen Gatekeeper-Hinweis wenn Binary fehlt auf macOS | VERIFIED | basic.html Zeilen 81-85: `[% IF isMac %]<div style="color: orange; margin-top:8px">[% 'PLUGIN_SPOTON_GATEKEEPER_HINT' | string %]</div>[% END %]` im ELSE-Block |
| 9  | D-09: Gatekeeper-Warnung in strings.txt für alle 11 Sprachen | VERIFIED | strings.txt Zeilen 1173-1184: PLUGIN_SPOTON_GATEKEEPER_HINT mit CS, DA, DE, EN, ES, FR, IT, NL, NO, PL, SV (11 Sprachen gezaehlt) |

**Score:** 9/9 Truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `.github/workflows/build-librespot.yml` | build-macos + lipo + aktualisierter release-Job | VERIFIED | build-macos-Job (Zeilen 109-146), lipo-Job (148-190), release needs: [build, lipo] (Zeile 195) |
| `Plugins/SpotOn/Bin/darwin/.gitkeep` | Placeholder-Verzeichnis fuer macOS Binary | VERIFIED | Datei existiert, 0 Byte, git-tracked |
| `Plugins/SpotOn/Helper.pm` | ISMAC-Block fuer darwin-Pfad-Registrierung | VERIFIED | Zeilen 34-38: if (main::ISMAC) mit addFindBinPaths auf Bin/darwin |
| `Plugins/SpotOn/Settings.pm` | isMac Template-Variable | VERIFIED | Zeile 62: `$paramRef->{isMac} = main::ISMAC ? 1 : 0;` |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | Gatekeeper-Warnung-Div mit IF isMac | VERIFIED | Zeilen 81-85: Conditional orange div mit PLUGIN_SPOTON_GATEKEEPER_HINT |
| `Plugins/SpotOn/strings.txt` | PLUGIN_SPOTON_GATEKEEPER_HINT in 11 Sprachen | VERIFIED | Zeilen 1173-1184: Alle 11 Sprachen vorhanden |
| `README.md` | macOS in Plattform-Liste | VERIFIED | Zeile 36: "macOS (Universal Binary: Intel + Apple Silicon)" + xattr-Hinweis |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| build-macos-Job | lipo-Job | needs: build-macos | WIRED | Workflow-Zeile 151: `needs: build-macos` |
| lipo-Job | release-Job | needs: [build, lipo] | WIRED | Workflow-Zeile 195: `needs: [build, lipo]` |
| Settings.pm | basic.html | isMac Template-Variable | WIRED | Settings.pm Zeile 62 setzt `$paramRef->{isMac}`; basic.html Zeile 81 wertet `[% IF isMac %]` aus |
| basic.html | strings.txt | PLUGIN_SPOTON_GATEKEEPER_HINT String-Referenz | WIRED | basic.html Zeile 83 referenziert den Key; strings.txt Zeile 1173 definiert ihn |
| Helper.pm | Plugins/SpotOn/Bin/darwin/ | addFindBinPaths-Aufruf | WIRED | Zeilen 34-38 in Helper.pm registrieren darwin/ wenn ISMAC gesetzt ist |

### Data-Flow Trace (Level 4)

Nicht applicable — Phase produziert CI-Konfiguration und Perl-Konfiguration, keine Komponente die dynamische Daten rendert. Die isMac-Variable ist ein Boolean-Flag (kein DB-Query).

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| YAML-Syntax gueltig | `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/build-librespot.yml'))"` | Exit 0 | PASS |
| Volle Testsuite gruen | `prove t/` | All tests successful (261 Tests, 12 Dateien) | PASS |
| strings.txt t/02_strings.t | enthalten in prove t/ | PASS | PASS |
| Perl-Syntax t/05_perl_syntax.t | enthalten in prove t/ | PASS | PASS |
| GATEKEEPER_HINT hat 11 Sprachen | awk-Zaehlung auf strings.txt | 11 | PASS |

### Probe Execution

Keine Probe-Skripte fuer diese Phase definiert. CI-Workflow kann nur in GitHub Actions ausgefuehrt werden.

### Requirements Coverage

| Requirement | Quell-Plan | Beschreibung | Status | Evidence |
|-------------|------------|--------------|--------|---------|
| PLT-01 | Plan 01 | macOS Universal Binary via lipo (Intel x86_64 + Apple Silicon aarch64) | SATISFIED | build-macos-Matrix + lipo-Job in build-librespot.yml implementiert; codesign-Ad-hoc-Signing vorhanden |
| PLT-02 | Plan 01 + 02 | macOS Binary funktioniert mit LMS Plugin Manager (ad-hoc signiert, kein Quarantine-xattr) | SATISFIED (CI-Teil); HUMAN NEEDED (LMS-Runtime-Teil) | Ad-hoc-Signing im Workflow vorhanden; Helper.pm ISMAC-Block registriert Bin/darwin/; Runtime-Verhalten erfordert macOS-Host |
| PLT-03 | Plan 02 | Setup-Anleitung dokumentiert Gatekeeper-Workaround (xattr -d) fuer manuelle Downloads | SATISFIED | README.md + basic.html Gatekeeper-Hint + strings.txt mit 11 Sprachen |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| basic.html | 114 | `placeholder=` | Info | HTML-Formular-Attribut fuer Client-ID-Eingabefeld — kein Code-Stub, kein Schuldenmarker. Vorhandener Code, nicht von Phase 16 eingefuehrt. |

Keine Schuldenmarker (TBD, FIXME, XXX) in den von Phase 16 modifizierten Dateien gefunden.

### Human Verification Required

#### 1. LMS Plugin Manager macOS Binary Detection

**Test:** SpotOn auf einem macOS-Host (Intel oder Apple Silicon) installieren, wo LMS laeuft. Nach Installation pruefen ob die Helper-Version im Settings-Panel angezeigt wird.
**Expected:** LMS findet das Binary in `Bin/darwin/spoton` und zeigt "v0.8.x (commit...)" als Binary-Status an.
**Why human:** `main::ISMAC` ist ein LMS-Compile-Time-Constant. Auf Linux-Entwicklungsrechnern ist ISMAC=0, sodass der ISMAC-Codepfad nicht lokal ausfuehbar ist. Der Test erfordert einen echten macOS-LMS-Host.

#### 2. Gatekeeper-Hint auf Settings-Seite (macOS, Binary fehlt)

**Test:** Auf macOS-LMS-Host: Binary aus `Bin/darwin/` temporaer entfernen oder umbenennen, Settings-Seite oeffnen.
**Expected:** Unterhalb der roten "Binary Missing"-Meldung erscheint ein oranger Hinweis mit dem Gatekeeper-xattr-Befehl in der korrekten Sprache.
**Why human:** Template-Rendering mit `isMac=1` kann nur auf echtem macOS-LMS-Host beobachtet werden.

#### 3. GitHub Actions CI produziert gueltiges Universal Binary

**Test:** Tag-Push `v*` oder workflow_dispatch auf GitHub ausfuehren. Im GitHub Release pruefen: (a) spoton-darwin-Artefakt vorhanden, (b) kein spoton-aarch64-darwin oder spoton-x86_64-darwin im Release, (c) `lipo -info spoton` berichtet zwei Architekturen (x86_64 + arm64).
**Expected:** GitHub Release enthaelt genau ein macOS-Binary (spoton-darwin) das als Universal Binary verifiziert ist.
**Why human:** CI-Workflow laeuft nur auf GitHub; kein lokales Ausfuehren von GitHub Actions moeglich.

### Gaps Summary

Keine Gaps. Alle 9 Observable Truths sind in der Codebasis verifiziert. Drei Human-Verification-Items stehen noch aus (macOS-Runtime-Verhalten, CI-Ausfuehrung auf GitHub). Diese erfordern Zugang zu einem macOS-LMS-Host und einem GitHub-Tag-Push.

---

_Verified: 2026-06-11T15:42:12Z_
_Verifier: Claude (gsd-verifier)_
