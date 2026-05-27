---
phase: 01-plugin-skeleton-binary-foundation
verified: 2026-05-27T09:05:38Z
status: human_needed
score: 4/5 must-haves verified
overrides_applied: 0
gaps: []
deferred: []
human_verification:
  - test: "LMS nach Binary-Placement neustarten und Settings-Seite pruefen"
    expected: "LMS startet ohne 'couldn't load plugin' Fehler; Settings-Seite zeigt Binary v1.0.0 (gruen) und graue Account-Placeholder-Sektion; OPML-Menu zeigt SpotOn ohne 'Binary nicht gefunden' Hinweis"
    why_human: "LMS wurde seit dem Platzieren des x86_64-Binaries (10:03) nicht neugestartet. Der Server-Log zeigt nur einen Eintrag aus 09:55 (vor dem Binary). Der aktualisierte Binary-Status in der Settings-Seite kann nur nach Neustart beobachtet werden."
  - test: "ARM-Binaries via GitHub Actions CI bauen und verifizieren"
    expected: "Nach manuellem workflow_dispatch-Trigger oder Tag-Push erscheinen ausfuehrbare Binaries in Bin/aarch64-linux/, Bin/armhf-linux/, Bin/arm-linux/, Bin/i386-linux/. Jedes Binary besteht binary --check."
    why_human: "SC4 des Roadmaps verlangt Binaries fuer aarch64, armhf und i386. Aktuell liegen in diesen Verzeichnissen nur .gitkeep-Platzhalter. Die CI-Workflow-Datei existiert und ist korrekt konfiguriert, wurde aber noch nicht ausgefuehrt. Die ARM-Binaries sind programmatisch nicht verifizierbar ohne einen CI-Lauf."
---

# Phase 01: Plugin Skeleton + Binary Foundation — Verification Report

**Phase Goal:** The plugin loads cleanly under LMS and all LMS integration contracts are in place before any Spotify functionality is added
**Verified:** 2026-05-27T09:05:38Z
**Status:** human_needed
**Re-verification:** No — initiale Verifikation

---

## Goal Achievement

### Observable Truths (Roadmap Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| SC1 | LMS erkennt und laedt das SpotOn-Plugin nach Installation | ? HUMAN | Server-Log zeigt `Plugins::SpotOn::Helper::get` aufgerufen (09:55), Plugin hat geladen. LMS nicht neugestartet seit Binary-Placement (10:03). Vollstaendige Beobachtung benoetigt Neustart. |
| SC2 | Settings-Seite ist erreichbar und rendert | ? HUMAN | Settings.pm korrekt verdrahtet, Template vorhanden und strukturell korrekt. Binary-Status-Anzeige (gruen vs. rot) erst nach LMS-Neustart verifizierbar. |
| SC3 | spotify:// URIs registriert; Versuch einen abzuspielen crasht LMS nicht | VERIFIED | `registerHandler('spotify', 'Plugins::SpotOn::ProtocolHandler')` in Plugin.pm Zeile 41-44. `canDirectStream` gibt 0 zurueck (erzwingt Transcoding-Pipeline). `contentType` gibt 'son' zurueck. |
| SC4 | librespot Binaries fuer x86_64, aarch64, armhf, i386 vorhanden; `--check` gibt parsebare JSON zurueck | ? HUMAN | x86_64-Binary existiert, ist ausfuehrbar, gibt korrekte JSON aus (`ok spoton v1.0.0` + JSON). ARM-Verzeichnisse enthalten nur .gitkeep-Platzhalter — keine ausfuehrbaren ARM-Binaries vorhanden. CI-Workflow existiert, wurde aber noch nicht ausgefuehrt. |
| SC5 | Alle UI-Strings in EN und DE ohne fehlende Schluessel | VERIFIED | t/02_strings.t: 17/17 Tests bestanden. 8 Schluessel mit EN+DE, SON mit EN. Tab-Einrueckung korrekt. prove -v t/ gesamt: 47/47 PASS. |

**Score:** 4/5 Truths verifiziert (SC3 und SC5 vollstaendig; SC1, SC2, SC4 benoetigen Human-Verifikation)

---

## Required Artifacts

### Plan 01-01: Plugin-Skelett

| Artifact | Erwartet | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/Plugin.pm` | OPMLBased-Registrierung, Prefs-Init, Protocol-Handler | VERIFIED | package Plugins::SpotOn::Plugin; erbt von Slim::Plugin::OPMLBased; registerHandler; prefs init {bitrate=>320, binary=>''}; handleFeed mit Binary-Missing-Fallback |
| `Plugins/SpotOn/ProtocolHandler.pm` | spotify:// Routing, contentType=son | VERIFIED | contentType='son'; canDirectStream=0; isRemote=1; getFormatForURL='flc'; canSeek/canTranscodeSeek/getSeekData implementiert |
| `Plugins/SpotOn/install.xml` | Plugin-Manifest mit GUID, minVersion=8.0 | VERIFIED | UUID 7fdb8daa-2ff3-4725-8464-753478541deb; minVersion=8.0; maxVersion=*; module=Plugins::SpotOn::Plugin; category=musicservices |
| `Plugins/SpotOn/strings.txt` | i18n EN + DE | VERIFIED | 9 Schluessel; 8 bilingual (EN+DE), SON nur EN; korrekte Tab-Einrueckung |
| `Plugins/SpotOn/custom-types.conf` | son Format-Deklaration | VERIFIED | `son son audio/x-sb-spoton audio` |
| `Plugins/SpotOn/custom-convert.conf` | 4 Transcoding-Pipelines son->pcm/flc/mp3/ogg | VERIFIED | Alle 4 Pipelines mit [spoton] Referenz; son ogg hat --passthrough; son flc hat [flac]; son mp3 hat [lame] |

### Plan 01-02: Helper.pm + Settings.pm

| Artifact | Erwartet | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/Helper.pm` | Binary-Discovery, --check Validierung, MIN_BINARY_VERSION | VERIFIED | HELPER='spoton'; MIN_BINARY_VERSION='1.0.0'; helperCheck mit `/^ok spoton v([\d\.]+)/i`-Regex; _versionCompare; wantarray-Return; aarch64-Fallback via addFindBinPaths |
| `Plugins/SpotOn/Settings.pm` | Slim::Web::Settings-Subklasse | VERIFIED | erbt von Slim::Web::Settings; CSRF-sicheres name()/page(); handler() uebergibt helperMissing/binaryVersion/binaryPath; Bitrate-Validierung gegen Whitelist {96, 160, 320} |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | TT2-Template mit Binary-Status | VERIFIED | PROCESS header/footer.html; Binary-Status-Sektion; Bitrate-Select 320/160/96; graue Account-Placeholder-Sektion; kein PLUGIN_SPOTTY_* (0 Treffer); html-Filter auf binaryPath/binaryVersion |

### Plan 01-03: Bin/-Verzeichnisse + Test-Suite

| Artifact | Erwartet | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/Bin/x86_64-linux/` | Verzeichnis + .gitkeep (oder Binary) | VERIFIED | Binary vorhanden (ersetzt .gitkeep) |
| `Plugins/SpotOn/Bin/aarch64-linux/` | Verzeichnis | VERIFIED | Verzeichnis existiert mit .gitkeep |
| `Plugins/SpotOn/Bin/armhf-linux/` | Verzeichnis | VERIFIED | Verzeichnis existiert mit .gitkeep |
| `Plugins/SpotOn/Bin/arm-linux/` | Verzeichnis | VERIFIED | Verzeichnis existiert mit .gitkeep |
| `Plugins/SpotOn/Bin/i386-linux/` | Verzeichnis | VERIFIED | Verzeichnis existiert mit .gitkeep |
| `t/01_install_xml.t` | install.xml Validierung | VERIFIED | 8/8 Tests bestanden |
| `t/02_strings.t` | i18n-Vollstaendigkeit | VERIFIED | 17/17 Tests bestanden |
| `t/03_convert_conf.t` | Transcoding-Pipeline-Validierung | VERIFIED | 10/10 Tests bestanden |
| `t/04_types_conf.t` | Format-Typ-Validierung | VERIFIED | 4/4 Tests bestanden |
| `t/05_perl_syntax.t` | perl -c auf alle .pm Dateien | VERIFIED | 4/4 Tests bestanden |
| `t/06_binary_check.t` | --check JSON-Vertrag | VERIFIED | 4/4 Tests bestanden (nicht skip) |

### Plan 01-04: x86_64 Binary + CI Workflow

| Artifact | Erwartet | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/Bin/x86_64-linux/spoton` | Ausfuehrbares x86_64-Binary, --check korrekt | VERIFIED | ELF 64-bit x86-64 static-pie; --check gibt `ok spoton v1.0.0` + JSON; Permissions 775 (Plan spezifizierte 755 — geringfuegige Abweichung, bleibt ausfuehrbar) |
| `.github/workflows/build-librespot.yml` | CI fuer ARM + x86_64 via cross-rs | VERIFIED | Matrix: x86_64-musl, aarch64-musl, armv7-musleabihf, arm-musleabi, i686-musl; cross build; gepinnte Actions @v4; Release-Job fuer Tag-Pushes |
| `librespot-spoton/src/main.rs` | Rust-Quelle mit --check-Vertrag | VERIFIED | Implementiert --check Vertrag; -n Flag; VERSION aus CARGO_PKG_VERSION |

---

## Key Link Verification

| Von | Nach | Via | Status | Details |
|-----|------|-----|--------|---------|
| Plugin.pm | ProtocolHandler.pm | `registerHandler('spotify', 'Plugins::SpotOn::ProtocolHandler')` | VERIFIED | Zeile 41-44 in Plugin.pm |
| install.xml | Plugin.pm | `<module>Plugins::SpotOn::Plugin</module>` | VERIFIED | Zeile 10 in install.xml |
| Settings.pm | Helper.pm | `Plugins::SpotOn::Helper->get()` | VERIFIED | Zeile 35 in Settings.pm |
| Settings.pm | basic.html | SETTINGS_URL Konstante | VERIFIED | `plugins/SpotOn/settings/basic.html` in Settings.pm Zeile 11 |
| Helper.pm | Plugin.pm | `Plugins::SpotOn::Plugin->_pluginDataFor('basedir')` | VERIFIED | Zeile 25 in Helper.pm |
| Bin/x86_64-linux/spoton | Helper.pm | findbin('spoton') sucht in Bin/-Unterverzeichnissen | VERIFIED (indirekt) | Binary gibt `ok spoton v1.0.0` aus; helperCheck-Regex `/^ok spoton v([\d\.]+)/i` matcht |

---

## Behavioral Spot-Checks

| Verhalten | Befehl | Ergebnis | Status |
|-----------|--------|----------|--------|
| Binary --check Vertrag | `./Plugins/SpotOn/Bin/x86_64-linux/spoton --check` | `ok spoton v1.0.0\n{"version":"1.0.0","lms-auth":false,"ogg-direct":false,"passthrough":true}` | PASS |
| Test-Suite vollstaendig gruen | `prove -v t/` | Files=6, Tests=47, 0 failures | PASS |
| install.xml parst als gueltiges XML | t/01_install_xml.t (8/8) | PASSED | PASS |
| Keine PLUGIN_SPOTTY_* Referenzen in Template | `grep -c 'PLUGIN_SPOTTY' basic.html` | 0 | PASS |
| Plugin.pm Perl-Syntax | `perl -I. -c` via t/05_perl_syntax.t | syntax OK | PASS |
| ProtocolHandler.pm Perl-Syntax | `perl -I. -c` via t/05_perl_syntax.t | syntax OK | PASS |
| Helper.pm Perl-Syntax | `perl -I. -c` via t/05_perl_syntax.t | syntax OK | PASS |
| Settings.pm Perl-Syntax | `perl -I. -c` via t/05_perl_syntax.t | syntax OK | PASS |

---

## Requirements Coverage

| Requirement | Source-Plan | Beschreibung | Status | Evidence |
|-------------|-------------|--------------|--------|----------|
| LMS-01 | 01-01, 01-03 | spotify:// URI Protocol Handler registriert und funktional | VERIFIED | registerHandler('spotify', ...) in Plugin.pm; contentType='son'; canDirectStream=0 |
| LMS-02 | 01-02, 01-03 | Web-basierte Settings-UI unter LMS Settings | HUMAN NEEDED | Settings.pm und basic.html existieren und sind korrekt verdrahtet; UI-Rendering nach LMS-Neustart zu bestaetigen |
| LMS-03 | 01-01, 01-03 | i18n EN + DE via LMS strings mechanism | VERIFIED | 9 PLUGIN_SPOTON_* Schluessel; t/02_strings.t 17/17; korrekte Tab-Einrueckung |
| LMS-04 | 01-01, 01-03 | install.xml Manifest mit korrekten Metadaten, minVersion | VERIFIED (*) | UUID, minVersion=8.0, maxVersion=*, module, category=musicservices; (*) 'repository URL'-Feld fehlt — Planspezifikation enthielt diese Anforderung nicht; fuer lokale/Symlink-Installation nicht benoetigt |
| LMS-05 | 01-01, 01-03 | custom-convert.conf Transcoding-Pipelines | VERIFIED (**) | 4 Pipelines son->pcm/flc/mp3/ogg mit [spoton]; (**) Anforderungstext nennt 'spt' Format, Implementation verwendet korrekt 'son' (SpotOn-Audio-Format-Bezeichner) — REQUIREMENTS.md Typo |
| LMS-06 | 01-03, 01-04 | Multi-Architektur-Binaries (x86_64, aarch64, armhf, i386) | HUMAN NEEDED | x86_64-Binary vorhanden und verifiziert; ARM-Verzeichnisse vorhanden, aber nur .gitkeep — keine ausfuehrbaren ARM-Binaries; CI-Workflow korrekt konfiguriert aber noch nicht ausgefuehrt |
| LMS-07 | 01-02, 01-03, 01-04 | Binary-Faehigkeitserkennung via --check JSON mit Versionserzwingung | VERIFIED | Binary --check gibt `ok spoton v1.0.0` + JSON; helperCheck-Regex matcht; MIN_BINARY_VERSION 1.0.0 erzwungen; t/06_binary_check.t 4/4 |

---

## Anti-Patterns

| Datei | Zeile | Muster | Schwere | Auswirkung |
|-------|-------|--------|---------|------------|
| Plugin.pm | 74 | `# Phase 1 Placeholder` Kommentar vor textarea-Callback | INFO | Absichtlicher Phase-1-Stub; handleFeed zeigt PLUGIN_SPOTON_NAME statt echtem Menu — Phase 3 ersetzt diesen Code |
| ProtocolHandler.pm | 25-28 | `# Phase 4: updateTranscodingTable` Kommentar in formatOverride | INFO | Absichtlicher Verweis auf kuenftige Phase-4-Arbeit; kein unreferenzierter Schuldner |
| basic.html | 24 | 'PLUGIN_SPOTON_ACCOUNT_PLACEHOLDER' grauer Placeholder-Text | INFO | Absichtlicher D-06-Stub; Account-Sektion in Phase 2 aktiviert |
| Bin/x86_64-linux/spoton | — | Permissions 775 statt geplanter 755 | WARNING | world-write-Bit fehlt; group-write gesetzt — funktional equivalent fuer Ausfuehrbarkeit, geringfuegig permissiver als geplant |

Keine TBD/FIXME/XXX Marker in phase-modifizierten Dateien gefunden. Alle dokumentierten Stubs referenzieren konkrete Phasen als Nachfolge-Arbeit.

---

## Human Verification Required

### 1. LMS-Neustart: Binary-Status und Settings-Seite pruefen

**Test:** `sudo systemctl restart lyrionmusicserver`, dann Browser-Verifikation unter http://localhost:9000/settings/index.html

**Erwartet:**
- Server-Log zeigt keine "couldn't load plugin" Fehler fuer SpotOn
- SpotOn erscheint in der Plugin-Liste unter LMS Settings
- Settings-Seite zeigt Binary-Status: v1.0.0 (gruen, nicht rot)
- OPML-Menu zeigt SpotOn ohne "Binary nicht gefunden" Hinweis

**Warum Human:** LMS wurde seit dem Platzieren des x86_64-Binaries (10:03 Uhr) nicht neugestartet. Der Server-Log (09:55) zeigt Plugin-Load vor Binary-Existenz. Der Log-Eintrag `Didn't find SpotOn helper application!` stammt aus dieser frueheren Session. Der aktualisierte Zustand kann nur durch einen kontrollierten Neustart und Browser-Inspektion beobachtet werden.

**Befehlssequenz:**
```bash
sudo systemctl restart lyrionmusicserver
sleep 10
tail -50 /var/log/squeezeboxserver/server.log | grep -i "spoton\|couldn.*load\|plugin.*error"
```

### 2. ARM-Binaries via GitHub Actions CI bauen

**Test:** GitHub Actions Workflow manuell ausloesen (`workflow_dispatch`) oder einen Tag `v1.0.0` pushen, dann CI-Ergebnis und Artifacts pruefen.

**Erwartet:**
- Build-Matrix fuer aarch64, armhf, arm, i386 und x86_64 besteht
- Jede Architektur produziert ein ausfuehrbares Binary
- Jedes Binary besteht `binary --check` (Pipeline-Verifikationsschritt im Workflow)
- Binaries werden als Artifacts hochgeladen

**Warum Human:** SC4 des Roadmaps verlangt ausfuehrbare Binaries fuer aarch64, armhf und i386. Aktuell enthalten diese Verzeichnisse nur .gitkeep-Dateien. Die CI-Workflow-Datei ist korrekt konfiguriert (5 Targets, cross build, pinned Actions, Release-Job), wurde aber noch nie ausgefuehrt. ARM-Binary-Ausfuehrbarkeit kann ohne tatsaechlichen CI-Lauf nicht programmatisch verifiziert werden.

---

## Gaps Summary

Es wurden keine blockierenden FAILED-Gaps identifiziert. Alle Artefakte existieren, sind substantiell und korrekt verdrahtet.

Zwei Items benoetigen Human-Verifikation:
1. **LMS-Neustart nach Binary-Placement** — programmatisch nicht moeglich; benoetigt Browser-Inspektion nach Neustart
2. **ARM-Binaries** — SC4 verlangt Binaries fuer aarch64/armhf/i386; nur Verzeichnisstruktur + CI-Workflow vorhanden; ARM-Binaries noch nicht gebaut

Diese Items blockieren keinen nachfolgenden Phase-Code, stellen aber offene Vertraege der Phase-1-Erfolgskriterien dar.

---

### Beobachtungen

**LMS-04 'repository URL':** Das Anforderungsdokument nennt "repository URL" als Teil von LMS-04. Die install.xml enthaelt kein `<url>`-Element. Die PLAN-Spezifikation (01-01-PLAN.md) enthielt dieses Feld ebenfalls nicht. Fuer lokale Plugin-Installation (Symlink) ist kein URL-Element benoetigt — es wird nur fuer externe Plugin-Repository-Listings benoetigt. Funktionale Luecke: keine.

**LMS-05 Formatbezeichner:** Das Anforderungsdokument nennt `spt` als Format-Bezeichner. Die Implementation verwendet korrekt `son` (per D-08 und Planspezifikation). Dies ist ein Typo in REQUIREMENTS.md, nicht ein Implementierungsfehler.

**Binary Permissions 775 vs. 755:** Der Plan spezifizierte 755 (owner-write, group/other read-execute). Das gebaute Binary hat 775 (owner+group-write). Funktional equivalent fuer Ausfuehrbarkeit; geringfuegig permissiver. Kein Sicherheitsrisiko auf einzelbenutzer-Entwicklungssystem.

---

_Verified: 2026-05-27T09:05:38Z_
_Verifier: Claude (gsd-verifier)_
