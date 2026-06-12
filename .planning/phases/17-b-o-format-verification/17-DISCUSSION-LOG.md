# Phase 17: B&O Format Verification - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-12
**Phase:** 17-b-o-format-verification
**Areas discussed:** Testmatrix & Protokoll, Auto-Mode Erwartung, Fehler-Eskalation, Echtzeitfähigkeit/Latenz

---

## Testmatrix & Protokoll

### Test-Player

| Option | Description | Selected |
|--------|-------------|----------|
| Nur B&O via UPnPBridge | Wie in ROADMAP definiert. squeezelite implizit getestet. | ✓ |
| B&O + Chromecast | Beide non-native Player testen. | |
| B&O + Chromecast + Referenz | Zusätzlich squeezelite als Baseline. | |

**User's choice:** Nur B&O via UPnPBridge
**Notes:** User klärte, dass der Chromecast-Test sich auf Format-Kompatibilität mit eingeschränkten Playern bezieht, nicht auf spezifische Chromecast-Hardware. B&O via UPnPBridge repräsentiert diesen Use Case.

### Testtiefe

| Option | Description | Selected |
|--------|-------------|----------|
| Funktional: Audio ja/nein | Pro Modus: Track starten, Audio ja/nein. Pass/Fail. | |
| Funktional + Stabilität | Zusätzlich: kein Stottern, kein Abbruch, Seek, Track-Wechsel. | |
| Voll: Audio + Meta + Qualität | Audio-Output, Songinfo, subjektive Qualität, Stabilität. | ✓ |

**User's choice:** Voll: Audio + Meta + Qualität

### Dokumentation

| Option | Description | Selected |
|--------|-------------|----------|
| VERIFICATION.md Tabelle | Standard 5×N Matrix in 17-VERIFICATION.md. | |
| Eigenständiges QA-Dokument | Separates 17-QA-RESULTS.md mit Details. | |
| Claude entscheidet | Format dem Planner überlassen. | ✓ |

**User's choice:** Claude entscheidet

---

## Auto-Mode Erwartung

| Option | Description | Selected |
|--------|-------------|----------|
| PCM-Fallback erwartet | UPnPBridge meldet Capabilities, LMS wählt PCM. | |
| FLAC-Fallback erwartet | B&O unterstützt FLAC, LMS sollte son-flc wählen. | |
| Unklar — muss getestet werden | Teil der Verifikation ist herauszufinden was Auto wählt. | ✓ |

**User's choice:** Unklar — muss getestet werden
**Notes:** Kein Vorwissen über UPnPBridge Player-Capability-Meldung an LMS.

---

## Fehler-Eskalation

| Option | Description | Selected |
|--------|-------------|----------|
| Fix in Phase 17 | QA + Bugfix kombiniert. | |
| Nur dokumentieren | Reine QA, Bugs in Folge-Phase. | |
| Kleine Fixes ja, große nein | Config/Pipeline-Tweaks direkt, Architekturänderungen separat. | ✓ |

**User's choice:** Kleine Fixes ja, große nein

---

## Echtzeitfähigkeit/Latenz

| Option | Description | Selected |
|--------|-------------|----------|
| Nur beobachten | Latenz notieren, kein Fix-Ziel. | |
| Eigene Phase/Projekt | Separates Thema, evtl. MozartBridge. | ✓ |
| In Phase 17 testen | Sync-Verhalten als Teil der Format-Verifikation. | |

**User's choice:** Ignorieren, gehört zu MozartBridge
**Notes:** User plant MozartBridge als eigenständiges LMS-Plugin für B&O Mozart-Platform-Integration. Latenz/Sync ist kein SpotOn-Thema.

---

## Claude's Discretion

- Dokumentationsformat (VERIFICATION.md vs. eigenständiges QA-Dokument)
- Testprotokoll-Reihenfolge
- Release-Notes-Integration

## Deferred Ideas

- **MozartBridge:** Eigenständiges LMS-Plugin für B&O Mozart-Platform (Latenz-Kompensation, Echtzeit-Sync). Separates Projekt.
- **Player-Capability-Detection:** Auto-Format könnte Player-Capabilities aktiv prüfen — eigene Phase falls Tests Probleme zeigen.
