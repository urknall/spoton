---
phase: 12-protocol-handler-rename
verified: 2026-06-05T13:39:12Z
status: human_needed
score: 9/11 must-haves verified
overrides_applied: 0
human_verification:
  - test: "Spotty und SpotOn gleichzeitig auf dem Raspi aktivieren und Browse-Wiedergabe in beiden testen"
    expected: "Keine Handler-Konflikte im LMS-Log; Browse funktioniert in SpotOn und Spotty unabhaengig"
    why_human: "Erfordert zwei gleichzeitig laufende LMS-Plugins auf echter Hardware (Raspi 192.168.13.5)"
  - test: "Browse-Track ueber das umbenannte spoton://-Schema abspielen"
    expected: "Audio-Ausgabe funktioniert; Track spielt ueber das neue URL-Schema"
    why_human: "End-to-End-Audio benoetigt librespot-Binary plus Spotify Premium"
  - test: "Connect-Wiedergabe vom Spotify-App aus starten und LMS-Log pruefen"
    expected: "Connect-URL nutzt spoton://connect- Praefix im LMS-Log sichtbar"
    why_human: "Erfordert Spotify-App Connect-Handoff auf echter Hardware"
---

# Phase 12: Protocol Handler Rename — Verification Report

**Phase Goal:** SpotOn uses spoton:// URI scheme for coexistence with Spotty
**Verified:** 2026-06-05T13:39:12Z
**Status:** human_needed
**Re-verification:** No — initial verification

---

## Vorabbefund: PROTO-* Requirements nicht in REQUIREMENTS.md

**WARNING:** Die PLANs deklarieren `requirements: PROTO-01` bis `PROTO-06`, aber diese IDs sind **nicht** in `.planning/REQUIREMENTS.md` eingetragen. Sie sind nur lokal in `12-RESEARCH.md` definiert. Die Traceability-Tabelle in REQUIREMENTS.md endet bei Phase 10 (DSTM-*). Phase 12 hat keine Eintraege in der projektweiten Traceability-Tabelle.

Dies ist eine Dokumentationsluecke, kein Implementierungsfehler — die Implementierung ist korrekt durchgefuehrt, aber die Anforderungen wurden nie in REQUIREMENTS.md aufgenommen.

---

## Goal Achievement

### Observable Truths (ROADMAP Success Criteria)

| # | Truth | Status | Evidenz |
|---|-------|--------|---------|
| SC1 | Alle URL-Konstruktionen in Plugin.pm, ProtocolHandler.pm, Connect.pm verwenden `spoton://` statt `spotify://` | VERIFIED | `grep -rn 'spotify://' Plugins/SpotOn/*.pm Plugins/SpotOn/API/*.pm \| grep -v '#'` liefert 0 Treffer; t/12_protocol_rename.t 16/16 PASS |
| SC2 | ProtocolHandler ist als `spoton` nicht `spotify` in LMS registriert | VERIFIED | Plugin.pm Zeile 90-93: `registerHandler('spoton', 'Plugins::SpotOn::ProtocolHandler')` bestaetigt |
| SC3 | custom-convert.conf verwendet Content-Types passend zum `spoton://`-Schema | VERIFIED | Datei hat `son`/`soc` Content-Types (bereits SpotOn-spezifisch); `$URL$` gibt volle URL inkl. Schema weiter; t/03_convert_conf.t PASS (10/10); keine Aenderung noetig gemaess Plan-Entscheidung |
| SC4 | Connect-URLs verwenden `spoton://connect-` Praefix | VERIFIED | Connect.pm Zeilen 630, 722, 828: `sprintf("spoton://connect-%u", $ts)` — 3 Konstruktionsstellen; t/12_protocol_rename.t Assertion PROTO-04 PASS |
| SC5 | Spotty und SpotOn koennen gleichzeitig in LMS aktiviert werden ohne URI-Handler-Konflikt | HUMAN NEEDED | Registrierungslogik pruefbar (LMS nutzt separate Hash-Keys pro Schema; `spoton` vs `spotify` sind unabhaengig), aber **Live-Koexistenztest auf Raspi steht aus** (Plan 02 Task 2 ist checkpoint:human-verify, noch nicht abgeschlossen) |
| SC6 | Vorhandene gecachte Metadaten unter alten `spotify://` Keys werden beim ersten Start invalidiert oder migriert | VERIFIED | Named Cache Namespace `spoton` mit Version 2 implementiert (alle 6 Module); `cacheSchemaVersion` Pref-Guard in `initPlugin()` vorhanden (Plugin.pm Zeilen 52-58); altes `spotify://`-basiertes Cache wird nicht mehr abgefragt (anderer MD5-Hash) |

**Score:** 5 von 6 ROADMAP Success Criteria automatisch verifiziert; 1 benoetigt Human-Verification.

### Zusaetzliche Must-Have Truths (PLAN 01 Frontmatter)

| # | Truth | Status | Evidenz |
|---|-------|--------|---------|
| 7 | D-06: Spotify API URIs (spotify:track:ID) bleiben unveraendert | VERIFIED | ProtocolHandler.pm Zeilen 266, 351, 431 und Connect.pm Zeilen 624, 714, 771: `spotify:track:` Muster intact |
| 8 | Rust-Binary normalisiert spoton:// zu spotify: fuer SpotifyUri | VERIFIED | main.rs Zeile 667: `track_uri.replace("spoton://", "spotify:")` |
| 9 | D-05: Kein Dual-Schema-Support — nur spoton:// wird akzeptiert | VERIFIED | Kein `spotify://`-Fallback in nicht-kommentiertem Code |
| 10 | D-08: Getrennte Cache-Namespaces, Content-Types und Binaries sichern Isolation von Spotty | VERIFIED | `spoton.db` via named namespace; `son`/`soc` != `spt`/`spc`; `[spoton]` != `[spotty]` |
| 11 | Alle 6 Platform-Binaries enthalten spoton://-Normalisierung, nicht spotify:// | VERIFIED | Python-Byte-Suche: alle 6 Binaries: `spotify://=0`, `spoton://>=1` |

**Gesamt-Score:** 9/11 automatisch verifiziert, 1 HUMAN NEEDED (SC5/D-07), 1 HUMAN NEEDED (Browser-Wiedergabe post-rename).

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `t/12_protocol_rename.t` | Grep-basierte Validierung PROTO-01 bis PROTO-06 | VERIFIED | Existiert, 217 Zeilen, 16 Assertions, alle PASS |
| `Plugins/SpotOn/Plugin.pm` | registerHandler('spoton'), SPOTON_CACHE_VERSION, cacheSchemaVersion Pref | VERIFIED | Alle 3 Elemente vorhanden und korrekt implementiert |
| `Plugins/SpotOn/ProtocolHandler.pm` | Alle spoton:// URL-Muster und Cache-Namespace | VERIFIED | 25 `spoton://` Vorkommen; `Cache->new('spoton', 2)` |
| `Plugins/SpotOn/Connect.pm` | spoton://connect- URL-Konstruktion und Matching | VERIFIED | 3 `sprintf("spoton://connect-%u", $ts)` Konstruktionen |
| `librespot-spoton/src/main.rs` | URI-Normalisierung spoton:// zu spotify: | VERIFIED | Zeile 667 bestaetigt |
| `Plugins/SpotOn/Bin/x86_64-linux/spoton` | x86_64 Binary mit spoton://-Normalisierung | VERIFIED | 17.802.680 Bytes, executable, spoton://=1, spotify://=0 |
| `Plugins/SpotOn/Bin/aarch64-linux/spoton` | aarch64 Binary | VERIFIED | 17.418.352 Bytes, executable, spoton://=1, spotify://=0 |
| `Plugins/SpotOn/Bin/armhf-linux/spoton` | armhf Binary | VERIFIED | 16.441.816 Bytes, executable, spoton://=3, spotify://=0 |
| `Plugins/SpotOn/Bin/arm-linux/spoton` | arm Binary | VERIFIED | 16.800.808 Bytes, executable, spoton://=3, spotify://=0 |
| `Plugins/SpotOn/Bin/i386-linux/spoton` | i386 Binary | VERIFIED | 16.711.156 Bytes, executable, spoton://=3, spotify://=0 |
| `Plugins/SpotOn/Bin/x86_64-win64/spoton.exe` | Windows Binary | VERIFIED | 37.371.543 Bytes, spoton://=1, spotify://=0 |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `Plugin.pm` | `Slim::Player::ProtocolHandlers` | `registerHandler('spoton', ...)` | WIRED | Zeilen 90-93 bestaetigt |
| `Plugin.pm` | `Slim::Utils::Cache` | `Cache->new('spoton', SPOTON_CACHE_VERSION)` | WIRED | Zeile 25 bestaetigt |
| `librespot-spoton/src/main.rs` | `SpotifyUri::from_uri` | `replace("spoton://", "spotify:")` | WIRED | Zeile 667 bestaetigt |
| `Connect.pm` | `spoton://connect-` URL-Schema | `sprintf("spoton://connect-%u", $ts)` | WIRED | Zeilen 630, 722, 828 |
| `ProtocolHandler.pm` | Kanonische Normalisierung | `s{^spoton:}{spoton://}` | WIRED | Zeile 371 bestaetigt |

---

## Data-Flow Trace (Level 4)

Nicht anwendbar auf diese Phase — reine String-Substitutions-Refaktorierung ohne neue Datenfluesse.

---

## Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| t/12_protocol_rename.t alle 16 Assertions | `prove t/12_protocol_rename.t` | 16/16 PASS | PASS |
| Perl-Syntax gueltig | `prove t/05_perl_syntax.t` | PASS | PASS |
| custom-convert.conf unveraendert (PROTO-03) | `prove t/03_convert_conf.t` | 10/10 PASS | PASS |
| t/11_track_history.t Regression | `prove t/11_track_history.t` | 10/10 PASS | PASS |
| Alle 6 Binaries enthalten spoton:// | Python byte search | spotify://=0, spoton://>=1 fuer alle 6 | PASS |
| Kein `spotify://` in Perl non-comment Lines | `grep -rn 'spotify://' Plugins/SpotOn/*.pm \| grep -v '#'` | 0 Treffer | PASS |

---

## Probe Execution

Keine formalen Probe-Scripts fuer diese Phase definiert.

---

## Requirements Coverage

| Requirement | Source Plan | Beschreibung | Status | Evidenz |
|-------------|-------------|--------------|--------|---------|
| PROTO-01 | 12-01, 12-02 | Alle URL-Konstruktionen verwenden `spoton://` | SATISFIED | 0 `spotify://`-Treffer in non-comment Lines; Binaries verifiziert |
| PROTO-02 | 12-01 | ProtocolHandler als `spoton` registriert | SATISFIED | Plugin.pm Zeile 90-93 |
| PROTO-03 | 12-01 | custom-convert.conf Content-Types passen | SATISFIED | `son`/`soc` waren bereits SpotOn-spezifisch; t/03 PASS |
| PROTO-04 | 12-01 | Connect-URLs verwenden `spoton://connect-` | SATISFIED | 3 Konstruktionsstellen in Connect.pm |
| PROTO-05 | 12-01, 12-02 | Spotty+SpotOn ohne Handler-Konflikt | NEEDS HUMAN | Registrierungs-Logik korrekt; Live-Test steht aus |
| PROTO-06 | 12-01 | Cache-Invalidierung beim ersten Start | SATISFIED | Named namespace + cacheSchemaVersion guard |

**Kritischer Befund — ORPHANED Requirements:**
PROTO-01 bis PROTO-06 sind **nicht** in `.planning/REQUIREMENTS.md` eingetragen. Die Traceability-Tabelle endet bei Phase 10. Phase 12 hat keine Eintraege. Die Anforderungen existieren nur in der phasen-lokalen `12-RESEARCH.md`. Dies ist eine Traceability-Luecke, kein Implementierungsfehler.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `Plugin.pm` | 392 | `# D-06: url and play set to spotify:// URI for Play-Intent.` | Info | Kommentar beschreibt korrekt, dass `spotify://` *als API-URI-Format* verwendet wird — kein Code-Problem, korrekter Kommentar per D-06 |

Keine `TBD`, `FIXME`, `XXX`-Marker in Phase-12-modifizierten Dateien gefunden.

**Pre-existing test failures (nicht durch Phase 12 verursacht, auf Baseline `ac0993a` bestaetigt):**
- `t/07_token_manager.t`: 4 fehlschlagende Tests (AUTH-02, getToken cache) — `::own` Subroutinen-Problem in Temp-Stub
- `t/08_api_client.t`: Parse-Error (kein Plan) — Stub-Ladefehler
- `t/09_settings.t`: 1 fehlschlagender Test (clientId reference)

Diese Fehler existierten vor Phase 12 und wurden durch das `git stash`-Rollback auf `ac0993a` bestaetigt.

---

## Human Verification Required

### 1. Spotty + SpotOn Koexistenz auf Raspi

**Test:** SpotOn und Spotty gleichzeitig in LMS Plugin-Settings aktivieren, LMS neu starten, in beiden Plugins Browse-Menues oeffnen und je einen Track abspielen.
**Expected:** Kein Handler-Konflikt im LMS-Log; Browse-Wiedergabe in beiden Plugins funktioniert unabhaengig voneinander.
**Why human:** Erfordert zwei gleichzeitig laufende LMS-Plugins auf echter Hardware (Raspi 192.168.13.5). Plan 02 Task 2 ist `checkpoint:human-verify` und noch ausstehend.

**Pruefschritte:**
1. SpotOn-Plugin auf Raspi deployen (ZIP installieren via LMS Plugin Manager oder direktes Kopieren)
2. Beide Plugins (SpotOn + Spotty) in LMS Plugin-Settings aktivieren
3. LMS neu starten
4. SpotOn Browse: Menues navigieren, Track suchen, abspielen — Audio pruefe
5. Spotty Browse: Menues navigieren, Track suchen, abspielen — Audio pruefe (unabhaengig)
6. LMS-Log auf Fehler pruefen: `grep -i 'error\|conflict\|handler' /var/log/squeezeboxserver/server.log | tail -20`
7. LMS-History auf `spoton://`-URLs (nicht `spotify://`) fuer neu gespielte SpotOn-Tracks pruefen

### 2. Browse-Wiedergabe nach Rename

**Test:** Track ueber LMS Browse-Menue in SpotOn suchen und abspielen.
**Expected:** Audio-Ausgabe funktioniert; librespot empfaengt `spoton://track:ID`, normalisiert zu `spotify:track:ID`, streamt erfolgreich.
**Why human:** End-to-End Audio erfordert librespot-Binary plus Spotify Premium-Konto.

### 3. Connect-Wiedergabe nach Rename

**Test:** Spotify-App oeffnen, SpotOn Connect-Geraet auswaehlen, Track starten; LMS-Log auf Connect-URLs pruefen.
**Expected:** Connect-URL zeigt `spoton://connect-<timestamp>` Praefix im LMS-Log.
**Why human:** Erfordert Spotify-App Connect-Handoff auf echter Hardware.

---

## Gaps Summary

Keine Blocker-Gaps identifiziert. Die automatisch verifizierbaren Teile des Phase-Ziels sind vollstaendig implementiert und korrekt.

**Ausstehend (Human Verification):**
- Raspi-Koexistenztest (Plan 02 Task 2 — `checkpoint:human-verify`) ist der einzige verbleibende Schritt zur vollstaendigen Phase-Abnahme.

**Dokumentations-Luecke (keine Implementierungs-Luecke):**
- PROTO-01 bis PROTO-06 fehlen in `.planning/REQUIREMENTS.md`. Empfehlung: Diese nach Abschluss der Phase nachtragen.

---

_Verified: 2026-06-05T13:39:12Z_
_Verifier: Claude (gsd-verifier)_
