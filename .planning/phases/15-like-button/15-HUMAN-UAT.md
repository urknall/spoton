---
status: partial
phase: 15-like-button
source: [15-VERIFICATION.md]
started: 2026-06-11T14:35:00Z
updated: 2026-06-11T14:35:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Track-Kontextmenü zeigt 'Like' bzw. 'Unlike'
expected: Beim Öffnen des Info-Menüs eines SpotOn-Tracks erscheint 'Like' wenn der Track nicht geliked ist, 'Unlike' wenn er es ist. Dynamisches Label korrekt.
result: [pending]

### 2. Like-Aktion speichert Track in Liked Songs
expected: Auswahl von 'Like' -> Track erscheint danach in Liked-Songs-Menü. Kurze Bestätigungsmeldung 'Liked!' sichtbar, dann zurück ins übergeordnete Menü.
result: [pending]

### 3. Unlike-Aktion entfernt Track aus Liked Songs
expected: Auswahl von 'Unlike' -> Track erscheint nicht mehr im Liked-Songs-Menü. Bestätigungsmeldung 'Removed' sichtbar.
result: [pending]

### 4. State-Check verursacht keine wahrnehmbare Verzögerung
expected: Wiederholtes Öffnen des Track-Kontextmenüs innerhalb 60s: sofortige Anzeige (Cache-Hit). Erstmalig: kein wahrnehmbarer Versatz trotz async API-Call.
result: [pending]

### 5. Scope-Upgrade: einmaliger Re-Auth nach Version-Upgrade
expected: Nach Upgrade von v1.2 auf v1.3 (cacheSchemaVersion 2->3): Token-Cache wird geflusht, neuer Token mit user-library-modify/read Scopes abgerufen. Kein Re-Auth-Dialog nötig.
result: [pending]

## Summary

total: 5
passed: 0
issues: 0
pending: 5
skipped: 0
blocked: 0

## Gaps
