# Phase 8: Multi-Arch Binary Distribution - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-03
**Phase:** 08-multi-arch-binary-distribution
**Areas discussed:** Build-Infrastruktur, Binary-Distribution, Verzeichnis-Mapping, Helper.pm Erkennung

---

## Build-Infrastruktur

| Option | Description | Selected |
|--------|-------------|----------|
| Lokal mit cross-rs | Alle Linux-Targets lokal mit cross-rs (Docker). macOS/Windows separat. | ✓ |
| GitHub Actions CI | CI Pipeline baut alle Targets automatisch bei Release-Tag. Matrix-Build. | |
| Hybrid | Linux lokal, CI später. macOS/Windows deferred oder manuell. | |

**User's choice:** Lokal mit cross-rs (nach Claude-Empfehlung — User hatte keine Erfahrung mit Cross-Compilation)
**Notes:** User hat keinen Mac → macOS-Targets (ARCH-05, ARCH-06) auf v1.2 deferred. Phase 8 Scope: 5 Linux + 1 Windows = 6 Targets.

**Follow-up: macOS-Zugang**

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, Mac vorhanden | Nativ bauen | |
| Nein, kein Mac | macOS auf v1.2 deferren | ✓ |

---

## Binary-Distribution

| Option | Description | Selected |
|--------|-------------|----------|
| Alles im Git-Repo | Binaries in Plugins/SpotOn/Bin/ committen. ~85-120MB. Spotty-NG Konvention. | ✓ |
| GitHub Releases | Release-Assets. Repo bleibt schlank, aber install.xml pro Release anpassen. | |

**User's choice:** Alles im Git-Repo
**Notes:** Matches Spotty-NG Konvention. install.xml braucht keine Änderung.

---

## Verzeichnis-Mapping

| Option | Description | Selected |
|--------|-------------|----------|
| Bestehend beibehalten | Existierende Namen, fehlende ergänzen. arm-linux = armv6, armhf-linux = armv7. | ✓ |
| Rust-Triple-Namen | Verzeichnisse nach Rust Target Triple. Eindeutig, bricht aber LMS-Konvention. | |

**User's choice:** Bestehend beibehalten
**Notes:** Spotty-NG Konvention. x86_64-win64/ als einziges neues Verzeichnis.

---

## Helper.pm Erkennung

| Option | Description | Selected |
|--------|-------------|----------|
| LMS findbin + Plattform-Pfade | addFindBinPaths() mit Plattform-Dirs. Spotty-NG Muster. | ✓ |
| Du entscheidest | Claude wählt die beste Lösung | |

**User's choice:** LMS findbin + Plattform-Pfade
**Notes:** _detectArch() Funktion, Fallback-Kette (aarch64→armhf, armv7→arm).

---

## Claude's Discretion

Keine — User hat bei allen Bereichen eine Auswahl getroffen.

## Deferred Ideas

- macOS-Targets (ARCH-05, ARCH-06) → v1.2 (kein Mac vorhanden)
- GitHub Actions CI → v1.2+ (wenn Projekt stabil/öffentlich)
- Binary-Size-Optimierung (strip/UPX) → v1.2+
- macOS Universal Fat Binary (ARCH-F01) → v1.2+
