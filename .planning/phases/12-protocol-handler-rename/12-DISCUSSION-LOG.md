# Phase 12: Protocol Handler Rename - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-05
**Phase:** 12-protocol-handler-rename
**Areas discussed:** Cache-Migration, LMS History-Kompatibilität, Übergangsperiode, Spotty-Koexistenz

---

## Cache-Migration

| Option | Description | Selected |
|--------|-------------|----------|
| TTL ablaufen lassen | Keine Migration. Alte Einträge expiren innerhalb von max 7 Tagen. | |
| Cache flush beim Update | Beim ersten Start nach Update: alle spoton_meta_* Einträge löschen. | ✓ |
| Key-Migration on-access | Beim Cache-Miss unter spoton://: auch unter altem spotify:// Key nachschauen. | |

**User's choice:** Cache flush beim Update
**Notes:** Sauberer Neustart bevorzugt.

### Flush-Trigger

| Option | Description | Selected |
|--------|-------------|----------|
| Version-Marker in Prefs | Pref 'cacheSchemaVersion' speichern. Bei initPlugin() prüfen. | ✓ |
| Einmal-Flag in Cache | Marker im Cache statt Prefs. Weniger persistent. | |
| Kein Trigger nötig | Immer beim Plugin-Start alle spoton_meta_* flush. | |

**User's choice:** Version-Marker in Prefs
**Notes:** Robust, idempotent.

### Flush-Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Nur spoton_meta_* | Gezielter Flush. API-Response-Caches bleiben. | |
| Alle spoton_* Keys | Kompletter Neustart aller SpotOn-Caches. | ✓ |

**User's choice:** Alle spoton_* Keys
**Notes:** Clean slate approach.

---

## LMS History-Kompatibilität

| Option | Description | Selected |
|--------|-------------|----------|
| Ignorieren | History-Einträge nicht abspielbar, akzeptabel für Pre-Release. | |
| DB-Migration beim Update | SQL UPDATE auf tracks Tabelle. Riskant. | |
| Dual-Handler Registration | Zusätzlich spotify:// Handler für History. Konflikte mit Spotty. | |
| Manuelle DB-Bereinigung | User löscht History-Einträge manuell auf dev und raspi. | ✓ |

**User's choice:** Manuelle DB-Bereinigung auf dev und raspi
**Notes:** Freeform-Antwort: "Kannst du einmalig alle History Einträge auf dev und raspi aus der DB löschen?" — wird als manueller Cleanup-Task behandelt, kein Code-Level-Feature.

### Importer.pm

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, komplett umstellen | Importer schreibt spoton:// URLs. Konsistent. | ✓ |
| Claude entscheidet | Technisches Detail — Claude löst es. | |

**User's choice:** Ja, komplett umstellen

---

## Übergangsperiode

| Option | Description | Selected |
|--------|-------------|----------|
| Clean Break | Nur spoton:// akzeptiert. Kein Dual-Schema-Code. | ✓ |
| Read-Only spotify:// Support | spoton:// für Neues, spotify:// noch abspielbar. | |
| Konfigurierbares Schema | Pref-basierter Schema-Switch. | |

**User's choice:** Clean Break
**Notes:** SpotOn ist Pre-Release, minimaler Nutzerkreis. Kein Backwards-Compatibility-Code nötig.

### URI-Konvertierung

| Option | Description | Selected |
|--------|-------------|----------|
| Kein Problem | Spotify-URI ist Eingabe, LMS-URL ist internes Routing. Klare Trennung. | ✓ |
| Bedenken besprechen | Potenzielle Probleme ansprechen. | |

**User's choice:** Kein Problem

---

## Spotty-Koexistenz

### Verifikationsstrategie

| Option | Description | Selected |
|--------|-------------|----------|
| Manueller Test auf raspi | Beide Plugins aktivieren, Browse + Connect testen. | ✓ |
| Checkliste im Plan | Verifikations-Checkliste als Plan-Task. | |
| Beides | Checkliste UND manueller Test. | |

**User's choice:** Manueller Test auf raspi

### Shared State

| Option | Description | Selected |
|--------|-------------|----------|
| Getrennte Cache-Dirs | Kein shared state. | |
| Gerätenamen-Konflikt möglich | Beide Plugins könnten denselben Connect-Namen verwenden. | |
| Beide Punkte relevant | Cache getrennt, aber Gerätenamen könnten kollidieren. | |

**User's choice:** Freeform — "Da Spotty derzeit keinen Connect-Mode hat sind nur die Cache Dirs relevant. Für die Weiterentwicklung von Spotty können wir ggf. immer noch auf einen Prefix gehen."

### Akzeptanzkriterium

| Option | Description | Selected |
|--------|-------------|----------|
| Gleichzeitig aktivierbar (strikt) | Beide Plugins können in LMS gleichzeitig aktiviert werden. Manueller Test. | ✓ |
| Kein Namespace-Konflikt (soft) | Keine technischen Konflikte, kein Dual-Activation-Test. | |

**User's choice:** Gleichzeitig aktivierbar (strikt)

---

## Claude's Discretion

- Reihenfolge der Datei-Änderungen
- Exakter Wert für cacheSchemaVersion
- _isDeadHistoryUrl Pattern-Update in Connect.pm
- Regex-Optimierungen

## Deferred Ideas

None — discussion stayed within phase scope.
