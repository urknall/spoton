# Phase 10: Connect-DSTM - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-04
**Phase:** 10-connect-dstm
**Areas discussed:** DSTM-Logik Wiederverwendung, Grace-Timer & Spirc-Autoplay, Queue-Injection Methode, Autoplay-Toggle Scope

---

## DSTM-Logik Wiederverwendung

### Track-Auswahl Methode

| Option | Description | Selected |
|--------|-------------|----------|
| Nur _searchFallback wiederverwenden | Connect-DSTM nutzt den aktuell spielenden Track als Seed-Artist und ruft direkt _searchFallback auf | |
| Eigene Connect-DSTM Logik | Neuer Code in Connect.pm mit eigener Search-Query | |
| Claude entscheidet | Pragmatischste Lösung basierend auf Code-Analyse | ✓ |

### Seed-Quelle

| Option | Description | Selected |
|--------|-------------|----------|
| Aus dem letzten Track-Metadata | pluginData('info') bereits vorhanden, kein extra API-Call | |
| Frischer getTrack() API-Call | Zuverlässiger falls pluginData veraltet | |
| Claude entscheidet | Pragmatischste Lösung | ✓ |

### Duplikat-Tracking

| Option | Description | Selected |
|--------|-------------|----------|
| Kein Duplikat-Tracking | Random-Offset sorgt für natürliche Variation | |
| Einfaches Duplikat-Tracking | Letzte N Track-URIs merken und filtern | |
| Claude entscheidet | Pragmatischste Lösung | ✓ |

**Notes:** Alle drei Fragen mit "Claude entscheidet" beantwortet — DSTM-Logik ist Implementierungsdetail.

**PIVOTAL MOMENT:** Nach diesen Fragen stellte der User die gesamte Architektur in Frage: "Spotify ist im Connect-Mode und sendet einen kontinuierlichen Stream. Ist es nicht nur eine Sache der App, dann mit DSTM einfach weiterzumachen. Mir scheint unser Ansatz zu komplex. Schau dir mal spotty-ng an."

---

## Spotty-NG Analyse (Mid-Discussion Pivot)

Analyse der Spotty-NG Codebasis ergab:
- **Spirc hat nativen Autoplay-Kontext** (`add_autoplay_resolving_when_required()` in spirc.rs)
- **`SessionConfig.autoplay = Option<bool>`** steuert das Verhalten
- **Binary meldet `"autoplay": true`** in `--check` Capabilities
- **`--autoplay on/off` CLI Flag** wird von Daemon.pm übergeben
- **Kein EndOfTrack-Event, kein Grace-Timer, kein API Queue-Injection** in Spotty-NG
- librespot 0.8 (unsere Version) hat identische Spirc-Autoplay Infrastruktur

**Ergebnis:** Phase 10 Scope komplett reduziert von "EndOfTrack + Grace-Timer + API Queue-Injection" auf "Wire Spirc-Autoplay through + Toggle".

---

## Grace-Timer & Spirc-Autoplay

### Spirc-Autoplay Interaktion

| Option | Description | Selected |
|--------|-------------|----------|
| Grace-Timer canceln | Wenn während Timer neues TrackChanged kommt, abbrechen | ✓ (initial) |
| Immer DSTM feuern | Unabhängig von Spirc-Autoplay immer eigenen Track injizieren | |
| Claude entscheidet | Sicherste Variante wählen | |

**User's choice:** Grace-Timer canceln (initial ausgewählt)
**Notes:** Nach dem Spotty-NG Pivot wurde dieser gesamte Bereich obsolet — Spirc handelt Autoplay intern, kein Timer nötig.

### Timer-Länge

| Option | Description | Selected |
|--------|-------------|----------|
| 3 Sekunden | Kurz genug für nahtloses Erlebnis | |
| 5 Sekunden | Konservativer | |
| Claude kalibriert | Startwert wählen, später per Testing anpassen | ✓ |

**Notes:** Obsolet nach Pivot — kein Timer nötig.

---

## Queue-Injection Methode

### Injection-Ansatz

| Option | Description | Selected |
|--------|-------------|----------|
| Spotify API Queue | POST /me/player/queue | ✓ (initial) |
| LMS-Playlist Fallback | Track in LMS-Playlist einfügen, verlässt Connect-Modus | |
| API Queue + Spirc Play | Doppelte Absicherung | |
| Claude entscheidet | Basierend auf API-Verhalten | |

**User's choice:** Spotify API Queue (initial ausgewählt)
**Notes:** Komplett obsolet nach Spotty-NG Pivot — Spirc-Autoplay braucht keine API-Calls.

---

## Autoplay-Toggle Scope

### Neuer Scope nach Pivot

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Verifizieren + Toggle | Spirc-Autoplay verifizieren, per-Player Toggle, --check Capability | ✓ |
| Verifizieren + LMS-Fallback | Plus EndOfTrack-Event als Backup wenn Spirc versagt | |
| Phase streichen | Kein Code nötig, nur UAT | |

### Toggle Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Connect-Autoplay | Toggle steuert nur --autoplay on/off | |
| Beides zusammen | Toggle steuert Connect-Autoplay UND Browse-DSTM, synchronisiert mit LMS DSTM-Dropdown | ✓ |

**User's clarification:** LMS DSTM-Dropdown bestehen lassen ("SpotOn Empfehlungen"), aber mit dem SpotOn Autoplay-Toggle bidirektional synchronisieren.

### Default-Wert

| Option | Description | Selected |
|--------|-------------|----------|
| An | Autoplay standardmäßig aktiv — Musik spielt weiter | ✓ |
| Aus | User muss bewusst aktivieren | |

### Synchronisation: Toggle OFF

| Option | Description | Selected |
|--------|-------------|----------|
| DSTM-Provider deregistrieren | SpotOn aus dem Dropdown entfernen | |
| DSTM-Dropdown auf 'Off' setzen | Provider bleibt registriert, Dropdown auf Off | ✓ |
| Claude entscheidet | | |

### Rück-Synchronisation: DSTM Dropdown → Toggle

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, bidirektional | Änderung in eine Richtung aktualisiert die andere | ✓ |
| Nein, nur Toggle → DSTM | Einfacher, vermeidet Sync-Loops | |
| Claude entscheidet | | |

---

## Claude's Discretion

- DSTM-Logik Wiederverwendung (Track-Auswahl, Seed-Quelle, Duplikat-Tracking) — alle obsolet nach Pivot
- Grace-Timer Kalibrierung — obsolet nach Pivot
- DSTM sync implementation details (pref change callbacks, timing)
- Settings UI layout for the autoplay toggle
- Whether `enableAutoplay` needs a daemon restart or can be applied live
- i18n string keys for the toggle label

## Deferred Ideas

- **DSTM-F01 (v1.2+):** LMS-side DSTM fallback if Spirc-native autoplay fails
- **Autoplay context customization:** Genre preferences etc. — not possible via Spirc

---

*Discussion log generated: 2026-06-04*
