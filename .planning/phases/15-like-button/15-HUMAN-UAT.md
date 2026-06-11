---
status: passed
phase: 15-like-button
source: [15-VERIFICATION.md]
started: 2026-06-11T14:35:00Z
updated: 2026-06-11T15:35:00Z
---

## Current Test

[all tests complete]

## Tests

### 1. Track-Kontextmenü zeigt 'Like' bzw. 'Unlike'
expected: Beim Öffnen des Info-Menüs eines SpotOn-Tracks erscheint 'Like' wenn der Track nicht geliked ist, 'Unlike' wenn er es ist. Dynamisches Label korrekt.
result: passed — Classic Skin: "Like / Unlike" als direkter Toggle in Track-Kontextmenü (neben Künstler/Album anzeigen). Material Skin: über ...Mehr erreichbar. Mehrere Fixes nötig: URI-Extraktion aus spoton:// URL, _trackItem context items statt registerInfoProvider, trackInfoURL im ProtocolHandler.

### 2. Like-Aktion speichert Track in Liked Songs
expected: Auswahl von 'Like' -> Track erscheint danach in Liked-Songs-Menü. Kurze Bestätigungsmeldung 'Liked!' sichtbar, dann zurück ins übergeordnete Menü.
result: passed — PUT /me/library erfolgreich, Log zeigt korrekte API-Calls. Toggle-UX: ein Klick statt drei Ebenen.

### 3. Unlike-Aktion entfernt Track aus Liked Songs
expected: Auswahl von 'Unlike' -> Track erscheint nicht mehr im Liked-Songs-Menü. Bestätigungsmeldung 'Removed' sichtbar.
result: passed — DELETE /me/library erfolgreich, wiederholtes Togglen Like/Unlike/Like/Unlike funktioniert zuverlässig.

### 4. State-Check verursacht keine wahrnehmbare Verzögerung
expected: Wiederholtes Öffnen des Track-Kontextmenüs innerhalb 60s: sofortige Anzeige (Cache-Hit). Erstmalig: kein wahrnehmbarer Versatz trotz async API-Call.
result: passed — Cache-Hit bei wiederholtem Toggle, API-Calls nur bei Cache-Miss (Log bestätigt).

### 5. Scope-Upgrade: einmaliger Re-Auth nach Version-Upgrade
expected: Nach Upgrade von v1.2 auf v1.3 (cacheSchemaVersion 2->3): Token-Cache wird geflusht, neuer Token mit user-library-modify/read Scopes abgerufen. Kein Re-Auth-Dialog nötig.
result: passed — cacheSchemaVersion 3 in allen Modulen (Client.pm, TokenManager.pm, Plugin.pm, ProtocolHandler.pm). Token-Refresh erfolgt transparent.

## Summary

total: 5
passed: 5
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps

None.

## Issues Found During UAT (all fixed)

1. **trackInfoMenu URI mismatch** — Guard prüfte $remoteMeta->{uri} auf spotify:track:, aber SpotOn nutzt spoton:// URLs. Fix: URI aus $url Parameter extrahieren.
2. **registerInfoProvider nicht im OPML-Browse-Kontext** — LMS TrackInfo menu() scheitert an objectForUrl für Remote-Tracks. Fix: Like/Unlike als _trackItem context item hinzugefügt.
3. **3-Ebenen-Nesting** — Track → ManageLike → Like → Liked! war zu umständlich. Fix: _toggleLike als direkter Ein-Klick-Toggle.
4. **Material Skin** — Kein Zugang ohne trackInfoURL im ProtocolHandler. Fix: trackInfoURL implementiert, erreichbar über ...Mehr.
5. **Leere Einträge in "Für dich gemacht"** — Browse-Category-Endpoint gibt degradierte Items ohne Namen zurück seit Feb 2026. Fix: Namens-Filter in _fetchAllPersonalMixes.
6. **ProtocolHandler Cache-Version** — War noch auf 2 statt 3. Fix: Auf 3 geändert.
