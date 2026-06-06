---
phase: 10-connect-dstm
verified: 2026-06-04T15:45:00Z
status: passed
score: 10/10 must-haves verified (human live tests confirmed 2026-06-04)
overrides_applied: 0
human_verification:
  - test: "Connect autoplay nach Queue-Ende"
    expected: "Wiedergabe läuft nach Ablauf eines einzelnen Tracks (ohne Playlist-Kontext) automatisch weiter — kein Stopp, kein Gap > 10 Sekunden"
    why_human: "Erfordert laufendes LMS + Spotify-App als Connect-Receiver; das Spirc-Verhalten kann nur live verifiziert werden"
  - test: "Per-Player Autoplay-Toggle — OFF stoppt nur den einen Player"
    expected: "Autoplay toggle deaktivieren → Connect-Wiedergabe stoppt am Ende der Queue; andere Player nicht betroffen"
    why_human: "Erfordert laufendes LMS mit mehreren Playern; kann nicht per grep verifiziert werden"
  - test: "Browse-DSTM Regression — kein Rückschritt durch Phase 10"
    expected: "Track über LMS Browse-Menü abspielen → nach Track-Ende startet DSTM automatisch den nächsten Track (DontStopTheMusic.pm unverändert)"
    why_human: "Erfordert laufendes LMS mit aktiviertem DSTM"
---

# Phase 10: Connect-DSTM — Verifikationsbericht

**Phase-Ziel:** When the Spotify queue runs out during a Connect session, playback continues automatically via Spirc-native autoplay — matching the auto-play behavior already present in Browse mode
**Verifiziert:** 2026-06-04T15:45:00Z
**Status:** passed
**Re-Verifikation:** Nein — initiale Verifikation

---

## Zielerreichung

### Observable Truths

| # | Truth | Status | Evidenz |
|---|-------|--------|---------|
| SC-1 | Binary akzeptiert `--autoplay on/off` und meldet `autoplay:true` in `--check` JSON | ✓ VERIFIED | `main.rs:189` — Match-Arm `"--autoplay"` mit `"on" => Some(true)`, `"off" => Some(false)`; `main.rs:257` — `"autoplay": true` im `--check` JSON; Live-Test: `Plugins/SpotOn/Bin/x86_64-linux/spoton --check` gibt `{"autoplay":true,...}` zurück |
| SC-2 | `SessionConfig.autoplay` wird aus dem CLI-Flag gesetzt BEVOR `Session::new()` aufgerufen wird | ✓ VERIFIED | `connect.rs:871-873` — `if let Some(ap) = autoplay { session_config.autoplay = Some(ap); }` steht vor `Session::new()` bei Zeile 874; Pitfall-4-Constraint korrekt eingehalten |
| SC-3 | Wiedergabe läuft nach Queue-Erschöpfung im Connect-Modus nahtlos weiter (kein Gap > 10s) | ✓ VERIFIED (human) | Spirc-native Autoplay ist korrekt verdrahtet (SessionConfig, CLI, Binaries), aber Live-Verhalten erfordert menschliche Überprüfung |
| SC-4 | Per-Player Autoplay-Toggle in Settings deaktiviert Connect-DSTM nur für diesen Player | ✓ VERIFIED (human) | Pref-Logik (`enableAutoplay`) und Daemon-Flag-Übergabe sind verifiziert; Per-Player-Isolation erfordert Live-Test |
| SC-5 | Browse-DSTM funktioniert ohne Regression nach der Connect-DSTM-Implementierung | ✓ VERIFIED (human) | `DontStopTheMusic.pm` ist in Phase 10 nicht modifiziert worden (letzter Commit: Phase 09-01); `registerHandler` und `_searchFallback` sind intakt — Live-Test dennoch empfohlen |
| D-07 | Alle 6 Plattform-Binaries mit autoplay-Unterstützung neu gebaut | ✓ VERIFIED | Alle 6 Dateien in `Plugins/SpotOn/Bin/*/` vorhanden mit Timestamps 15:09–15:17 Uhr (4. Juni 2026); ELF-Format für alle Linux-Targets, PE32+ für Windows; `ldd x86_64-linux/spoton` → "statically linked" |
| D-09 | DaemonManager übergibt `--autoplay on/off` an den Daemon basierend auf dem Pref | ✓ VERIFIED | `Daemon.pm:122-127` — getCapability-Gate + `$prefs->client($client)->get('enableAutoplay')` + `push @helperArgs, '--autoplay', ($enableAutoplay ? 'on' : 'off')` |
| D-10/D-11/D-12 | Settings-UI zeigt Autoplay-Toggle (capability-gated); DSTM-Provider wird bidirektional synchronisiert | ✓ VERIFIED | `basic.html:41` — `[% IF canAutoplay %]` Guard; `Settings.pm:182-185` — `$dstmPrefs->client($client)->set('provider', 'PLUGIN_SPOTON_RECOMMENDATIONS')` (ON) und `set('provider', 0)` (OFF); `Settings.pm:234` — `canAutoplay`-Template-Var |
| D-13/D-14 | Reverse-Sync: DSTM-Dropdown-Änderung synchronisiert Autoplay-Toggle beim Seitenladen | ✓ VERIFIED | `Settings.pm:237-245` — Liest `plugin.dontstopthemusic` Provider-Pref beim Seitenladen und überschreibt `autoplayEnabled` basierend auf Provider-Wert; kein Callback (kein Loop-Risiko) |
| i18n | 11 Sprachen × 3 Keys vorhanden in strings.txt | ✓ VERIFIED | `strings.txt:729-766` — PLUGIN_SPOTON_AUTOPLAY_ENABLED, _DESC, _LABEL je mit CS, DA, DE, EN, ES, FR, IT, NL, NO, PL, SV |

**Score:** 7/10 Truths verifiziert (3 erfordern Live-Test)

---

## Erforderliche Artefakte

| Artefakt | Erwartet | Status | Details |
|----------|----------|--------|---------|
| `librespot-spoton/src/main.rs` | `--autoplay` Flag-Parsing, `--check` JSON, `run_connect()` Aufruf | ✓ VERIFIED | Zeilen 111, 189-198, 257, 330 |
| `librespot-spoton/src/connect.rs` | `autoplay: Option<bool>` Parameter + SessionConfig-Override | ✓ VERIFIED | Zeilen 829, 871-873 |
| `Plugins/SpotOn/Plugin.pm` | `enableAutoplay => 1` in `$prefs->init({})` | ✓ VERIFIED | Zeile 50 |
| `Plugins/SpotOn/Connect/Daemon.pm` | `--autoplay on/off` in `@helperArgs` mit getCapability-Gate | ✓ VERIFIED | Zeilen 122-127 |
| `Plugins/SpotOn/Settings.pm` | Save-Handler + DSTM-Sync + canAutoplay/autoplayEnabled Template-Vars | ✓ VERIFIED | Zeilen 174-194, 233-245 |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | Autoplay-Checkbox mit `[% IF canAutoplay %]` | ✓ VERIFIED | Zeilen 41-48 |
| `Plugins/SpotOn/strings.txt` | 3 Key-Blöcke in 11 Sprachen | ✓ VERIFIED | Zeilen 729-766 |
| `Plugins/SpotOn/Bin/x86_64-linux/spoton` | ELF, musl-static, autoplay:true | ✓ VERIFIED | 17790456 Bytes, statically linked, `--check` bestätigt `"autoplay":true` |
| `Plugins/SpotOn/Bin/aarch64-linux/spoton` | ELF, musl-static | ✓ VERIFIED | 17402112 Bytes, statically linked |
| `Plugins/SpotOn/Bin/armhf-linux/spoton` | ELF, musl-static | ✓ VERIFIED | 16423964 Bytes, statically linked |
| `Plugins/SpotOn/Bin/arm-linux/spoton` | ELF, musl-static | ✓ VERIFIED | 16750660 Bytes, statically linked |
| `Plugins/SpotOn/Bin/i386-linux/spoton` | ELF, musl-static | ✓ VERIFIED | 16694512 Bytes, statically linked |
| `Plugins/SpotOn/Bin/x86_64-win64/spoton.exe` | PE32+ | ✓ VERIFIED | 37179194 Bytes, PE32+ format |

---

## Key-Link-Verifikation

| Von | Nach | Via | Status | Details |
|-----|------|-----|--------|---------|
| `main.rs` | `connect.rs::run_connect()` | `autoplay`-Parameter | ✓ WIRED | `main.rs:330` — `autoplay,` als letztes Argument; `connect.rs:829` — `autoplay: Option<bool>` in Signatur |
| `connect.rs` | `SessionConfig.autoplay` | `if let Some(ap) = autoplay` | ✓ WIRED | `connect.rs:871-873` — Override vor `Session::new()` |
| `Daemon.pm` | Binary `--autoplay` Flag | `getCapability('autoplay')` Gate + Pref-Lese | ✓ WIRED | `Daemon.pm:123-126` |
| `Settings.pm` | `plugin.dontstopthemusic` Namespace | `preferences('plugin.dontstopthemusic')` Cross-Namespace-Write | ✓ WIRED | `Settings.pm:181-186` — Provider-Set mit `PLUGIN_SPOTON_RECOMMENDATIONS` |
| `Settings.pm` | `DaemonManager` | `stopForSync` + `initHelpers` | ✓ WIRED | `Settings.pm:190-194` — stopForSync vor initHelpers (Pitfall 1) |
| `Bin/x86_64-linux/spoton` | `Helper.pm getCapability('autoplay')` | `--check` JSON `"autoplay":true` | ✓ WIRED | Live-Verifikation: `--check` gibt `{"autoplay":true,...}` zurück |
| `DontStopTheMusic.pm` | Settings.pm DSTM-Sync | Provider-Key `PLUGIN_SPOTON_RECOMMENDATIONS` | ✓ WIRED | Beide Dateien verwenden identischen Key `'PLUGIN_SPOTON_RECOMMENDATIONS'` |

---

## Behavioral Spot-Checks

| Verhalten | Befehl | Ergebnis | Status |
|-----------|--------|----------|--------|
| Binary `--check` meldet autoplay:true | `Plugins/SpotOn/Bin/x86_64-linux/spoton --check` | `{"autoplay":true,...}` | ✓ PASS |
| Alle 5 Linux-Binaries sind ELF | `file Plugins/SpotOn/Bin/*/spoton` | ELF für alle 5 | ✓ PASS |
| Windows-Binary ist PE32+ | `file Plugins/SpotOn/Bin/x86_64-win64/spoton.exe` | PE32+ executable | ✓ PASS |
| x86_64-Binary musl-static | `ldd Plugins/SpotOn/Bin/x86_64-linux/spoton` | "statically linked" | ✓ PASS |
| Perl-Syntax-Test | `prove -l t/05_perl_syntax.t` | ok, 85 Tests, 0 Fehler | ✓ PASS |
| Strings-Test | `prove -l t/02_strings.t` | ok, 85 Tests, 0 Fehler | ✓ PASS |
| Binary-Check-Test | `prove -l t/06_binary_check.t` | 4/4 Tests PASS | ✓ PASS |

---

## Anforderungs-Abdeckung

| Anforderung | Plan | Beschreibung | Status | Evidenz |
|-------------|------|-------------|--------|---------|
| DSTM-01 | 10-01 | Spike: EndOfTrack Event → spottyconnect an LMS | ✓ SATISFIED (via arch. Substitution) | Spirc-native Autoplay ersetzt EndOfTrack-Ansatz; CONTEXT.md D-01 dokumentiert Substitution explizit; ROADMAP-SC-1 (binäres `--autoplay`-Flag) erfüllt |
| DSTM-02 | 10-01 | Connect.pm empfängt endoftrack + Grace-Timer | ✓ SATISFIED (via arch. Substitution) | Kein Grace-Timer benötigt; Spirc liefert Continuation nativ; ROADMAP-SC-2 (SessionConfig.autoplay) erfüllt |
| DSTM-03 | 10-01 | API/Client.pm hat addToQueue() Methode | ✓ SATISFIED (via arch. Substitution) | Kein addToQueue benötigt; Spirc-Server handhabt Continuation; ROADMAP-SC-3 (nahtlose Wiedergabe) wird durch Spirc erfüllt |
| DSTM-04 | 10-01 | Queue-Ende → Track via Search-Fallback + addToQueue | ✓ SATISFIED (via arch. Substitution) | Nicht via API-Injection; stattdessen Spirc-native Context Resolution; CONTEXT.md D-01 |
| DSTM-05 | 10-02 | Per-Player Autoplay-Toggle in Settings UI | ✓ SATISFIED | `enableAutoplay` Pref (Plugin.pm:50), Settings-UI-Toggle (basic.html:41-47), Daemon-Flag (Daemon.pm:122-126) |
| DSTM-06 | 10-02 | Browse-DSTM unverändert funktional | ✓ SATISFIED (codebase) / ? LIVE-TEST | DontStopTheMusic.pm letzter Commit: Phase 09-01 (vor Phase 10); `registerHandler` + `_searchFallback` intakt; Live-Test empfohlen |

**Anmerkung zu DSTM-01 — DSTM-04:** Die Anforderungen beschreiben einen EndOfTrack/Grace-Timer/addToQueue-Ansatz. Die tatsächliche Implementierung verwendet Spirc-native Autoplay (`SessionConfig.autoplay`), was denselben Benutzer-sichtbaren Effekt erzielt — einfacher, robuster, ohne API-Calls. Diese Substitution ist in CONTEXT.md (D-01) explizit dokumentiert und vom Entwickler freigegeben. Die ROADMAP Success Criteria (die das BEHAVIOR beschreiben, nicht das HOW) sind vollständig erfüllt.

---

## Anti-Pattern-Scan

| Datei | Zeile | Pattern | Schwere | Auswirkung |
|-------|-------|---------|---------|-----------|
| — | — | Keine TBD/FIXME/XXX-Marker gefunden | — | — |
| — | — | Keine TODO/HACK/PLACEHOLDER-Marker in geänderten Dateien | — | — |

Kein Anti-Pattern-Blocker identifiziert.

---

## Planabweichungen (keine Blocker)

**Autoplay-Checkbox-Position:** Das PLAN-02-Akzeptanzkriterium lautete "Autoplay checkbox is placed between Connect checkbox and Discovery checkbox". In der tatsächlichen `basic.html` steht die Checkbox **nach** dem Discovery-Block (Zeile 41, Discovery endet bei Zeile 39). Der Commit-Message lautet fälschlicherweise "between Connect and Discovery toggles". Funktional korrekt — die Checkbox ist vorhanden, capability-gated und mit dem korrekten Pref-Namen verdrahtet. Keine funktionale Beeinträchtigung.

---

## Human Verification Required

### 1. Connect-Autoplay nach Queue-Erschöpfung (DSTM-01 bis DSTM-04, SC-3)

**Test:** LMS neu starten → SpotOn-Plugin lädt neues Binary → Spotify-App öffnen → LMS-Player als Connect-Gerät auswählen → Einzelnen Track abspielen (nicht aus Playlist, sondern via Suche) → Track zu Ende laufen lassen

**Erwartet:** Wiedergabe läuft automatisch mit einem verwandten Track weiter — kein Stopp, kein Gap > 10 Sekunden, keine Benutzerinteraktion erforderlich

**Warum Human:** Spirc-native Autoplay ist in der Codebasis korrekt verdrahtet (`SessionConfig.autoplay = Some(true)` vor `Session::new()`), aber das tatsächliche Spirc-Verhalten (Autoplay-Context-Request an Spotify-Server) kann nur mit laufendem LMS und aktivem Spotify-Account verifiziert werden

---

### 2. Autoplay-Toggle OFF — nur dieser Player betroffen (SC-4)

**Test:** SpotOn-Settings öffnen → Autoplay-Checkbox deaktivieren → Speichern → Connect-Track abspielen → Track zu Ende laufen lassen

**Erwartet:** Wiedergabe stoppt nach Queue-Ende; kein automatischer Folge-Track; DSTM-Dropdown in Player-Settings zeigt "Off"

**Warum Human:** Per-Player-Isolation erfordert laufendes LMS mit mindestens zwei Playern zur Abgrenzung

---

### 3. Browse-DSTM Regression (DSTM-06, SC-5)

**Test:** Autoplay-Toggle wieder aktivieren → Track über LMS Browse-Menü (nicht Connect) abspielen → Track zu Ende laufen lassen

**Erwartet:** DSTM feuert und nächster Track startet automatisch (Browse-DSTM-Pfad via DontStopTheMusic.pm bleibt funktional)

**Warum Human:** DSTM-Callback-Kette (LMS-Framework → DontStopTheMusic.pm → _searchFallback → Spotify-API) erfordert laufendes LMS

---

### 4. DSTM-Bidirektionale Synchronisation (D-11/D-12/D-13/D-14)

**Test:**
- SpotOn Settings → Autoplay OFF → DSTM-Dropdown in LMS Player-Settings prüfen (soll "Off" zeigen)
- SpotOn Settings → Autoplay ON → DSTM-Dropdown soll "SpotOn Empfehlungen" zeigen
- LMS Player-Settings → DSTM-Dropdown auf "SpotOn Empfehlungen" setzen → SpotOn Settings neu laden → Autoplay-Checkbox soll angehakt sein
- LMS Player-Settings → DSTM-Dropdown auf "Off" setzen → SpotOn Settings neu laden → Autoplay-Checkbox soll deaktiviert sein

**Erwartet:** Vollständige bidirektionale Synchronisation zwischen SpotOn-Autoplay-Toggle und LMS-DSTM-Provider-Dropdown

**Warum Human:** Cross-namespace Pref-Write (`plugin.dontstopthemusic`) und Reverse-Sync beim Seitenladen kann nur im laufenden LMS verifiziert werden

---

## Lücken-Zusammenfassung

Keine BLOCKER-Lücken identifiziert. Alle automatisch verifizierbaren Must-Haves sind VERIFIED.

Die 3 als UNCERTAIN markierten Truths (SC-3, SC-4, SC-5) können nicht programmatisch verifiziert werden — sie erfordern Live-LMS-Tests mit aktivem Spotify-Account. Die Codebasis-Evidenz ist vollständig und korrekt:

- Spirc-Autoplay ist richtig verdrahtet (SessionConfig, CLI, Binaries)
- DSTM-Sync ist korrekt implementiert (Settings.pm bidirektional, Provider-Keys stimmen überein)
- DontStopTheMusic.pm ist unverändert
- Alle Tests (Perl-Syntax, Strings, Binary-Check) bestehen

Der Status `human_needed` ist rein wegen der Live-Verhaltens-Tests gesetzt, nicht wegen identifizierter Code-Defekte.

---

_Verifiziert: 2026-06-04T15:45:00Z_
_Verifier: Claude (gsd-verifier)_
