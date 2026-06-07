# Phase 14: Connect Fixes - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-07
**Phase:** 14-connect-fixes
**Areas discussed:** Cache-Dir Isolation, Volume-Strategie, Rückkopplung & Echo

---

## Cache-Dir Isolation

### Runde 1 — Erstvorschlag (abgelehnt)

| Option | Description | Selected |
|--------|-------------|----------|
| spoton/connect-{mac}/ | Flach per Player-MAC, wie in CON-01 beschrieben | |
| spoton/{account}/connect-{mac}/ | Per Account UND Player verschachtelt | |
| Claude entscheidet | Implementierungsdetail | |

**User's choice:** Abgelehnt — User wollte erst klären, ob der volle Kontext verstanden wurde.
**Notes:** User erklärte das konkrete Szenario: Frau verbindet per Spotify Connect auf ihrem Handy, danach zeigt Browse fremde Inhalte weil Connect-Daemon die Browse-Credentials überschreibt. User kritisierte, dass der volle Kontext (Research, Learnings, Memory) nicht berücksichtigt war.

### Runde 2 — Mit vollständigem Kontext

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, genau so | Connect → spoton/connect-{mac}/, Browse → spoton/{accountId}/ (unverändert) | ✓ |
| Anpassung nötig | Grundidee stimmt, aber ändern | |

**User's choice:** Ja, genau so
**Notes:** Nach Analyse von TokenManager.pm, Research SUMMARY.md, Memory-Einträgen, und dem konkreten Problemszenario. Kernverständnis: Connect-Daemon braucht nur Spirc-Credentials, keine API-Tokens. Pfadtrennung eliminiert die Credential-Überschreibung vollständig.

---

## Volume-Strategie

### Grace Period

| Option | Description | Selected |
|--------|-------------|----------|
| 3 Sekunden | Matches CON-03 Requirement. Konservativ. | ✓ |
| 2 Sekunden | Aggressiver, schnellere Sync | |
| Claude entscheidet | Empirisch testen | |

**User's choice:** 3 Sekunden

### Volume Init

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, beides | --initial-volume + --volume-ctrl linear | ✓ |
| Nur --volume-ctrl linear | Kein --initial-volume | |
| Claude entscheidet | Researcher entscheidet | |

**User's choice:** Ja, beides (--initial-volume $client->volume + --volume-ctrl linear)

---

## Rückkopplung & Echo

| Option | Description | Selected |
|--------|-------------|----------|
| 3s reicht allein | Mit --initial-volume + --volume-ctrl linear kein falsches VolumeChanged mehr | |
| 3s + Event-Filter | Zusätzlich Delta-Filter (Volume-Events ignorieren bei Δ < 2) | |
| Claude entscheidet | Researcher soll Mechanismen untersuchen | ✓ |

**User's choice:** Claude entscheidet
**Notes:** User delegiert Echo-Unterdrückungsstrategie an Researcher/Planner.

---

## Claude's Discretion

- Echo-Unterdrückungsstrategie bei 3s Grace Period (ob Zeitfenster allein reicht oder Event-Delta-Filter nötig)
- Binary-Capability-Checks für neue Flags
- Platzierung der Args in @helperArgs

## Deferred Ideas

None — discussion stayed within phase scope
