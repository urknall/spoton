# Phase 7: DE→EN Code Cleanup - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-03
**Phase:** 7-DE→EN Code Cleanup
**Areas discussed:** Übersetzungsstrategie, Scope-Ränder, Technische Referenzen

---

## Übersetzungsstrategie

### Frage 1: Wie sollen deutsche Kommentare übersetzt werden?

| Option | Description | Selected |
|--------|-------------|----------|
| Idiomatisch neu formulieren | Englisch neu schreiben, nicht wörtlich übersetzen. Dabei darf gekürzt/verbessert werden. Redundante Kommentare werden gelöscht statt übersetzt. | ✓ |
| 1:1 Übersetzen | Jeder deutsche Kommentar bekommt ein englisches Äquivalent, auch wenn er offensichtlich oder redundant ist. | |
| Nur löschen | Deutsche Kommentare werden entfernt. Nur Kommentare, die ein nicht-offensichtliches WHY erklären, bekommen eine englische Version. | |

**User's choice:** Idiomatisch neu formulieren
**Notes:** —

### Frage 2: Wie mit bereits englischen Kommentaren umgehen, die veraltet oder redundant sind?

| Option | Description | Selected |
|--------|-------------|----------|
| Nur deutsche bereinigen | Phase 7 beschränkt sich auf deutsche Texte. Englische Kommentare bleiben wie sie sind. | |
| Auch englische aufräumen | Wenn wir schon in der Datei sind, redundante englische Kommentare gleich mit entfernen. | ✓ |

**User's choice:** Auch englische aufräumen
**Notes:** Erweitert den Scope über CLEAN-01/02/03 hinaus, aber sinnvoll für ein konsistent sauberes Ergebnis.

---

## Scope-Ränder

### Frage 1: Sollen Umlaut-Umschreibungen auch als 'deutsch' gelten?

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, alles bereinigen | Auch 'aendern', 'Laengen', 'Zeichen' etc. übersetzen. Durchgängig englischer Codebase. | ✓ |
| Nur echte Umlaute | Nur Kommentare mit ä/ö/ü/ß bereinigen. ASCII-Workarounds bleiben. | |

**User's choice:** Ja, alles bereinigen
**Notes:** —

### Frage 2: Gehören die Rust-Quelldateien auch zum Scope?

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Perl (.pm Dateien) | CLEAN-01/02/03 sprechen von Perl-Quellcode. Rust nicht anfassen. | |
| Perl + Rust | Rust-Files mit prüfen und ggf. bereinigen. Vollständigkeit. | ✓ |

**User's choice:** Perl + Rust
**Notes:** Aktuell kein deutscher Text in Rust-Source, aber Verifikation soll trotzdem laufen.

---

## Technische Referenzen

### Frage 1: Wie mit Task-IDs und Phase-Referenzen umgehen?

| Option | Description | Selected |
|--------|-------------|----------|
| Beibehalten | IDs wie T-04.4-01, D-09, WR-01 bleiben stehen. Nur der deutsche Begleittext wird übersetzt. | ✓ |
| Entfernen | Phase-Referenzen und Task-IDs komplett entfernen. | |
| Vereinheitlichen | IDs beibehalten, Format vereinheitlichen. | |

**User's choice:** Beibehalten
**Notes:** —

### Frage 2: Sollen Pitfall-Referenzen einen kurzen englischen Kontext bekommen?

| Option | Description | Selected |
|--------|-------------|----------|
| Kurzer Kontext | z.B. '# P-01: never parallelize pagination'. Selbsterklärend ohne RESEARCH.md. | ✓ |
| Nur ID | Pitfall-IDs stehen lassen wie sie sind. | |

**User's choice:** Kurzer Kontext
**Notes:** —

---

## Claude's Discretion

None — all decisions made by user.

## Deferred Ideas

None — discussion stayed within phase scope.
