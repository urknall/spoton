# Phase 1: Plugin Skeleton + Binary Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-27
**Phase:** 1-Plugin Skeleton + Binary Foundation
**Areas discussed:** librespot-Binary-Herkunft, Binary-Distribution, Settings-Seite Phase 1, Transcoding-Pipelines

---

## librespot-Binary-Herkunft

### Fork-Strategie

| Option | Description | Selected |
|--------|-------------|----------|
| Herger's Fork weiternutzen | Existierender LMS-Fork mit --lms, --check etc. Bewährt, aber vermutlich nicht mehr aktiv gepflegt. | |
| Eigener SpotOn-Fork | Neuer Fork von librespot-org/librespot mit eigenen LMS-Patches. Volle Kontrolle. | |
| Upstream + Patches on top | Offizielles librespot als Base, LMS-Features als Patchset. Leichter aufzufrischen. | |

**User's choice:** "Entscheide du" — Claude soll basierend auf Patch-Umfang entscheiden (Research-Phase)
**Notes:** User tendierte zu Option 2 oder 3, war unsicher bei Vor-/Nachteilen. Nach Erklärung der Tradeoffs delegierte User die Entscheidung an die Research-Phase.

### Patch-Herkunft

| Option | Description | Selected |
|--------|-------------|----------|
| Herger's Patches existieren | LMS-Features existieren in Herger's Fork, portierbar | |
| Komplett neu schreiben | Keine brauchbaren Patches, from scratch in Rust | |
| Teilweise vorhanden | Manche Features existieren, andere müssen neu | |

**User's choice:** Unsicher — "Ich bin mir nicht sicher, welche Features wiederverwertbar sind"
**Notes:** Zum Research-Auftrag gemacht: Herger's Fork auditieren, Portierbarkeit jedes LMS-Patches bewerten, Empfehlung für Fork-Strategie ableiten.

---

## Binary-Distribution

### Verteilungsmethode

| Option | Description | Selected |
|--------|-------------|----------|
| Bundled im ZIP (wie Spotty) | Alle Arch-Binaries im Plugin-ZIP unter Bin/. ~60-80MB. | ✓ |
| Download bei Erstnutzung | Plugin-ZIP nur Perl-Code. Binary wird beim Start geladen. | |
| Entscheide du | Claude wählt basierend auf Konventionen. | |

**User's choice:** Bundled im ZIP (wie Spotty)
**Notes:** User fragte nach, wie Spotty es tatsächlich macht. Recherche bestätigte: Spotty bundled alles im ZIP, kein Download-Mechanismus. User folgt diesem bewährten Modell.

### Bin/-Ordnerstruktur

| Option | Description | Selected |
|--------|-------------|----------|
| Spotty-Konvention | aarch64-linux/, arm-linux/ etc. Community kennt Schema. | |
| Perl $Config{archname} | x86_64-linux-gnu-thread-multi etc. Konsistenter mit LMS. | ✓ |
| Entscheide du | Researcher vergleicht beide Ansätze. | |

**User's choice:** Perl $Config{archname}
**Notes:** Keine weiteren Fragen.

---

## Settings-Seite Phase 1

### Initialer Inhalt

| Option | Description | Selected |
|--------|-------------|----------|
| Minimal: Binary-Status only | Nur Binary-Info, keine Account-Felder. | |
| Vorstrukturiert | Sektionen für Account + Binary angelegt, Account-Felder deaktiviert. | ✓ |
| Entscheide du | Researcher/Planner entscheidet nach Patterns. | |

**User's choice:** Vorstrukturiert
**Notes:** Account-Felder sollen sichtbar aber grau/deaktiviert sein bis Phase 2.

### Fehlerverhalten bei fehlendem Setup

| Option | Description | Selected |
|--------|-------------|----------|
| Hinweis im Menü-Root | Erster OPML-Eintrag zeigt Statushinweis, verschwindet wenn OK. | ✓ |
| Nur in Settings sichtbar | Fehler nur auf Settings-Seite, Menü zeigt normalen Zustand. | |
| Entscheide du | Researcher entscheidet nach Konventionen. | |

**User's choice:** "Ich bevorzuge Opt 1. Prüfe du die Machbarkeit gem. LMS-Plugin-Konventionen"
**Notes:** Machbarkeitsprüfung als Research-Auftrag notiert.

---

## Transcoding-Pipelines

### Anzahl Pipelines in Phase 1

| Option | Description | Selected |
|--------|-------------|----------|
| Alle vier (son→flc/pcm/mp3/ogg) | Komplette convert.conf von Anfang an. | ✓ |
| Nur son→flc (Minimum) | Nur Default-Pipeline, Rest in Phase 4. | |
| Entscheide du | Researcher/Planner entscheidet. | |

**User's choice:** Alle vier
**Notes:** User wies auf Konfliktpotenzial mit Spotty's `spt`-Format hin und schlug eigenes Kürzel vor.

### Format-Kürzel

| Option | Description | Selected |
|--------|-------------|----------|
| son | Frei, klar, koexistiert mit Spotty's spt. audio/x-sb-spoton. | ✓ |
| Anderes Kürzel | Eigener Vorschlag. | |

**User's choice:** `son`
**Notes:** Recherche bestätigte: `son` ist in keinem LMS-Plugin oder im LMS-Core belegt. Spotty nutzt `spt`, Qobuz nutzt `qbz`. SpotOn-Konvention: `son` / `audio/x-sb-spoton`.

---

## Claude's Discretion

- Fork-Strategie (merge-basiert vs. upstream + patchset) — wird in Research-Phase basierend auf Patch-Scope entschieden
- Menü-Root-Hinweis bei fehlendem Setup — Machbarkeit wird gegen LMS OPML-Konventionen geprüft

## Deferred Ideas

None — discussion stayed within phase scope
