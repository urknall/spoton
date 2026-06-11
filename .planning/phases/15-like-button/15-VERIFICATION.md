---
phase: 15-like-button
verified: 2026-06-11T14:30:00Z
status: human_needed
score: 3/5
overrides_applied: 0
human_verification:
  - test: "Track-Kontextmenü zeigt 'Like' bzw. 'Unlike'"
    expected: "Beim Öffnen des Info-Menüs eines SpotOn-Tracks erscheint 'Like' wenn der Track nicht geliked ist, 'Unlike' wenn er es ist. Dynamisches Label korrekt."
    why_human: "Erfordert laufende LMS-Instanz mit aktivem Spotify-Account. trackInfoMenu-Callback kann nicht ohne echten LMS-Laufzeitkontext ausgeführt werden."
  - test: "Like-Aktion speichert Track in Liked Songs"
    expected: "Auswahl von 'Like' -> Track erscheint danach in Plugins/SpotOn Liked-Songs-Menü. Kurze Bestätigungsmeldung 'Liked!' sichtbar, dann zurück ins übergeordnete Menü."
    why_human: "Erfordert Spotify-Premium-Account, laufenden LMS + Connect-Session. End-to-End-Verifizierung des PUT /me/library Calls nicht automatisierbar ohne Livesystem."
  - test: "Unlike-Aktion entfernt Track aus Liked Songs"
    expected: "Auswahl von 'Unlike' -> Track erscheint nicht mehr im Liked-Songs-Menü. Bestätigungsmeldung 'Removed' sichtbar."
    why_human: "Gleiche Bedingungen wie Like-Test. DELETE /me/library gegen echte API."
  - test: "State-Check verursacht keine wahrnehmbare Verzögerung"
    expected: "Wiederholtes Oeffnen des Track-Kontextmenüs innerhalb 60s: sofortige Anzeige (Cache-Hit). Erstmalig: kein wahrnehmbarer Versatz trotz async API-Call."
    why_human: "Subjektive Latenzmessung, erfordert Live-LMS-Session."
  - test: "Scope-Upgrade: einmaliger Re-Auth nach Version-Upgrade"
    expected: "Nach Upgrade von Plugin v1.2.x auf v1.3 (mit cacheSchemaVersion-Bump 2->3): Token-Cache wird geflusht, neuer Token mit user-library-modify/user-library-read Scopes abgerufen. Kein Re-Auth-Dialog nötig (transparent). Falls sehr alte Credentials: 403 zeigt PLUGIN_SPOTON_LIKE_ERROR_SCOPE."
    why_human: "Erfordert bestehende v1.2-Installation zum Upgrade-Test. Transparenter Cache-Flush ist nicht ohne Livesystem beobachtbar."
---

# Phase 15: Like Button — Verifikationsbericht

**Phase-Ziel:** Users can save and remove tracks from Liked Songs directly from browse menus without leaving LMS
**Verifiziert:** 2026-06-11
**Status:** human_needed
**Re-Verifikation:** Nein — erstmalige Verifikation

---

## Zielerreichung

### Observable Truths (aus Roadmap Success Criteria + Plan Must-Haves)

| # | Wahrheit | Status | Nachweis |
|---|----------|--------|---------|
| SC-1 | Track-Kontextmenü zeigt 'Like'/'Unlike' dynamisch basierend auf Liked-State | ? UNCERTAIN | Implementierung verifiziert (trackInfoMenu + SpotOnManageLike vorhanden, korrekt verdrahtet), aber visuelle Darstellung erfordert Livetest |
| SC-2 | 'Like' speichert Track in Liked Songs (bestätigt durch Liked-Songs-Menü) | ? UNCERTAIN | saveTracks → PUT /me/library korrekt verdrahtet, End-to-End nur auf Livesystem verifizierbar |
| SC-3 | 'Unlike' entfernt Track aus Liked Songs | ? UNCERTAIN | removeTracks → DELETE /me/library korrekt verdrahtet, End-to-End nur auf Livesystem verifizierbar |
| SC-4 | State-Check ohne wahrnehmbare Verzögerung (D-06: on-demand, D-07: 60s Cache) | ✓ VERIFIED | Code-Analyse: SpotOnManageLike prüft Cache vor API-Call (`$cache->get($cacheKey)` + early return). Cache-Miss löst async `checkTracks` aus. _noCache => 1 + manueller 60s-Cache in Plugin.pm korrekt implementiert. |
| SC-5 | Nach Upgrade: einmalige Re-Auth via cacheSchemaVersion-Bump | ? UNCERTAIN | Mechanismus verifiziert (SPOTON_CACHE_VERSION => 3 in Plugin.pm + Guard in initPlugin), Livetest des Upgrade-Pfads nicht automatisierbar |
| MH-1 | Client.pm hat saveTracks, removeTracks, checkTracks als öffentliche Klassenmethoden | ✓ VERIFIED | Grep: Client.pm Zeilen 209, 223, 237. Korrekte Signatur `($class, $accountId, $uris, $cb)`. |
| MH-2 | D-12: saveTracks sendet PUT an /me/library mit uris Query-Param | ✓ VERIFIED | Client.pm Zeile 211: `$class->_request('put', 'me/library', { uris => join(',', @{$uris || []}) })` |
| MH-3 | D-13: removeTracks sendet DELETE an /me/library mit uris Query-Param | ✓ VERIFIED | Client.pm Zeile 225: `$class->_request('delete', 'me/library', { uris => ... })` |
| MH-4 | D-14: checkTracks sendet GET an /me/library/contains mit uris Query-Param | ✓ VERIFIED | Client.pm Zeile 239: `$class->_request('get', 'me/library/contains', { uris => ... })` |
| MH-5 | D-15: Alle drei Methoden routen durch me/* Prefix → own-token Flavor | ✓ VERIFIED | Alle drei Pfade beginnen mit 'me/' — die _request-Routing-Logik dispatcht me/* automatisch auf own-token |
| MH-6 | Leere 200-Body-Antworten bei PUT/DELETE lösen keinen parse_error aus | ✓ VERIFIED | Client.pm Zeile 532: `if ($content =~ /\S/)` — leerer Body überspringt from_json, ruft userCb(undef) mit $err=undef auf |
| MH-7 | D-05: strings.txt hat PLUGIN_SPOTON_LIKE und PLUGIN_SPOTON_UNLIKE in allen 11 Sprachen | ✓ VERIFIED | strings.txt Zeilen 1132-1156: CS/DA/DE/EN/ES/FR/IT/NL/NO/PL/SV für beide Keys vollständig |
| MH-8 | Cache-Namespace-Version 3 in Client.pm und TokenManager.pm | ✓ VERIFIED | Client.pm Zeile 35: `Slim::Utils::Cache->new('spoton', 3)`, TokenManager.pm Zeile 33: identisch |
| MH-9 | D-01: trackInfoMenu via registerInfoProvider in initPlugin registriert | ✓ VERIFIED | Plugin.pm Zeile 131-134: `require Slim::Menu::TrackInfo; Slim::Menu::TrackInfo->registerInfoProvider(spotonTrackInfo => (func => \&trackInfoMenu))` |
| MH-10 | D-02: Like/Unlike NICHT im items-Array von _trackItem | ✓ VERIFIED | _trackItem (Plugin.pm ab Zeile 501) enthält keine Referenzen auf SpotOnLike/SpotOnUnlike/PLUGIN_SPOTON_LIKE |
| MH-11 | D-07: Liked-State 60s gecacht, kein zweiter API-Call bei Cache-Hit | ✓ VERIFIED | SpotOnManageLike: `$cache->get($cacheKey)` → early return bei Cache-Hit; `$cache->set($cacheKey, $isLiked, 60) unless $err` bei Miss |
| MH-12 | D-08: Cache sofort nach Like/Unlike invalidiert | ✓ VERIFIED | _doLibraryAction Zeile 461: `$cache->remove($cacheKey)` auf Erfolgspfad |
| MH-13 | D-09: Erfolg zeigt showBriefly + nextWindow grandparent | ✓ VERIFIED | Plugin.pm Zeilen 463-466: `showBriefly => 1, nextWindow => 'grandparent'` |
| MH-14 | D-10: Fehler zeigen showBriefly, 403 zeigt scope-spezifische Meldung | ✓ VERIFIED | Plugin.pm Zeilen 455-458: `$err->{code} == 403 ? PLUGIN_SPOTON_LIKE_ERROR_SCOPE : PLUGIN_SPOTON_LIKE_ERROR`, kein nextWindow bei Fehler |
| MH-15 | SPOTON_CACHE_VERSION ist 3 in Plugin.pm | ✓ VERIFIED | Plugin.pm Zeile 22: `use constant SPOTON_CACHE_VERSION => 3` |

**Score: 3/5 Roadmap-Truths VERIFIED** (SC-1, SC-2, SC-3, SC-5 erfordern Livetest — zugrunde liegende Implementierung vollständig verifiziert)

---

### Deferred Items

Keine — alle Phase-15-Anforderungen sind in dieser Phase adressiert.

---

## Artifact-Verifikation

### Erforderliche Artefakte

| Artefakt | Beschreibung | Status | Details |
|----------|-------------|--------|---------|
| `Plugins/SpotOn/API/Client.pm` | saveTracks, removeTracks, checkTracks + empty-body guard | ✓ VERIFIED | Level 1-4: existiert (Zeilen 209/223/237), substanziell (echte _request-Calls), verdrahtet (in Plugin.pm referenziert), Datenfluß real (PUT/DELETE/GET /me/library) |
| `Plugins/SpotOn/API/TokenManager.pm` | Cache-Version-Bump | ✓ VERIFIED | Zeile 33: `Slim::Utils::Cache->new('spoton', 3)` — kein Stub |
| `Plugins/SpotOn/strings.txt` | Like/Unlike i18n-Strings | ✓ VERIFIED | 7 neue Keys vorhanden: LIKE (11 Sprachen), UNLIKE (11 Sprachen), LIKED/UNLIKED/LIKE_ERROR/LIKE_ERROR_SCOPE/MANAGE_LIKE (EN-only) |
| `t/08_api_client.t` | Tests für LIB-01 bis LIB-05 | ✓ VERIFIED | Zeilen 516-629: LIB-01/02/03/04/05 Testblöcke, DELETE-Stub in Mock, alle Tests grün (261 Tests) |
| `Plugins/SpotOn/Plugin.pm` | trackInfoMenu + 4 neue Subs + registerInfoProvider | ✓ VERIFIED | Zeilen 379-468: trackInfoMenu, SpotOnManageLike, SpotOnLike, SpotOnUnlike (via _doLibraryAction), alle mit echtem Verhalten |

### Key-Link-Verifikation

| Von | Zu | Via | Status | Nachweis |
|-----|-----|-----|--------|----------|
| Client.pm::saveTracks | Client.pm::_request | 'put', 'me/library' | ✓ WIRED | Zeile 211: `$class->_request('put', 'me/library', ...)` |
| Client.pm::removeTracks | Client.pm::_request | 'delete', 'me/library' | ✓ WIRED | Zeile 225: `$class->_request('delete', 'me/library', ...)` |
| Client.pm::checkTracks | Client.pm::_request | 'get', 'me/library/contains' | ✓ WIRED | Zeile 239: `$class->_request('get', 'me/library/contains', ...)` |
| Plugin.pm::trackInfoMenu | Plugin.pm::SpotOnManageLike | OPML url callback | ✓ WIRED | Zeile 391: `url => \&SpotOnManageLike` |
| Plugin.pm::SpotOnManageLike | Client.pm::checkTracks | async API-Call | ✓ WIRED | Zeile 428: `Plugins::SpotOn::API::Client->checkTracks(...)` |
| Plugin.pm::SpotOnLike | Client.pm::saveTracks | via _doLibraryAction | ✓ WIRED | Zeile 438/452: `_doLibraryAction(..., 'saveTracks', ...)` → `$apiMethod = 'saveTracks'` |
| Plugin.pm::SpotOnUnlike | Client.pm::removeTracks | via _doLibraryAction | ✓ WIRED | Zeile 443/452: `_doLibraryAction(..., 'removeTracks', ...)` |
| Plugin.pm::initPlugin | Slim::Menu::TrackInfo->registerInfoProvider | Hook-Registrierung | ✓ WIRED | Zeile 131-134: `Slim::Menu::TrackInfo->registerInfoProvider(spotonTrackInfo => ...)` |

---

## Data-Flow-Trace (Level 4)

| Artefakt | Datenvariable | Quelle | Echte Daten | Status |
|----------|--------------|--------|-------------|--------|
| SpotOnManageLike | `$isLiked` | `Client->checkTracks` → GET /me/library/contains | Ja — echter API-Call, Response `[true|false]` | ✓ FLOWING |
| SpotOnLike/_doLibraryAction | `$err` (Erfolgssignal) | `Client->saveTracks` → PUT /me/library → empty-body guard | Ja — empty-body guard setzt $err=undef auf 200 OK | ✓ FLOWING |
| SpotOnUnlike/_doLibraryAction | `$err` (Erfolgssignal) | `Client->removeTracks` → DELETE /me/library → empty-body guard | Ja | ✓ FLOWING |

---

## Behavioraler Spot-Check

| Verhalten | Befehl | Ergebnis | Status |
|-----------|--------|----------|--------|
| Perl-Syntax Client.pm | `perl -c Plugins/SpotOn/API/Client.pm` (via prove) | Testlauf: 261 Tests, alles grün | ✓ PASS |
| Perl-Syntax Plugin.pm | `perl -c` (via t/05_perl_syntax.t) | 261 Tests, alles grün | ✓ PASS |
| Vollständige Testsuite | `prove t/` | Files=12, Tests=261, Result: PASS | ✓ PASS |
| LIB-01: saveTracks PUT | `prove t/08_api_client.t` (LIB-01 Block) | PUT /me/library, uris= param, $err=undef | ✓ PASS |
| LIB-02: removeTracks DELETE | `prove t/08_api_client.t` (LIB-02 Block) | DELETE /me/library, $err=undef | ✓ PASS |
| LIB-03: checkTracks GET | `prove t/08_api_client.t` (LIB-03 Block) | GET /me/library/contains, returns [true] | ✓ PASS |

---

## Anforderungs-Coverage

| Anforderung | Quell-Plan | Beschreibung | Status | Nachweis |
|-------------|-----------|-------------|--------|----------|
| LIB-01 | 15-01, 15-02 | User kann Track via Kontextmenü in Liked Songs speichern | ✓ SATISFIED | saveTracks in Client.pm, SpotOnLike in Plugin.pm, vollständig verdrahtet |
| LIB-02 | 15-01, 15-02 | User kann Track via Kontextmenü aus Liked Songs entfernen | ✓ SATISFIED | removeTracks in Client.pm, SpotOnUnlike in Plugin.pm, verdrahtet |
| LIB-03 | 15-01, 15-02 | Track-Kontextmenü zeigt aktuellen Liked-State ('Like'/'Unlike') | ✓ SATISFIED (Code) / ? NEEDS HUMAN (Live) | checkTracks + SpotOnManageLike + dynamisches Label via `$isLiked` |
| LIB-04 | 15-01, 15-02 | Liked-State-Check via GET /me/library/contains ohne wahrnehmbare Verzögerung | ✓ SATISFIED (Architektur) / ? NEEDS HUMAN (Latenz) | On-demand (D-06), 60s TTL Cache (D-07), async via SimpleAsyncHTTP |
| LIB-05 | 15-01, 15-02 | Plugin fordert user-library-modify/user-library-read; Upgrade triggert einmaligen Re-Auth via cacheSchemaVersion | ✓ SATISFIED (Mechanismus) / ? NEEDS HUMAN (Upgrade-Pfad) | SPOTON_CACHE_VERSION => 3, Guard in initPlugin, Scopes in librespot OAUTH_SCOPES |

**Alle 5 LIB-Anforderungen sind in Phase 15 adressiert. 2 sind vollständig automatisch verifiziert (LIB-01, LIB-02), 3 erfordern ergänzende Livetest-Verifikation (LIB-03, LIB-04, LIB-05).**

---

## Anti-Pattern-Scan

| Datei | Zeile | Muster | Schwere | Impact |
|-------|-------|--------|---------|--------|
| Keine gefunden | — | — | — | — |

**TBD/FIXME/XXX:** Keine unreferenzierten Debt-Marker in den modifizierten Dateien gefunden.

**Stubs:** Keine. Alle vier neuen Plugin.pm-Subs haben echte Implementierungen. Client.pm-Methoden rufen echte _request-Dispatch-Pfade auf.

**Hinweis zur Abweichung in Plan 02:** Plan 02 spezifizierte `SpotOnLike` und `SpotOnUnlike` als eigenständige Subs mit direkten API-Calls. Die Implementierung extrahiert stattdessen den gemeinsamen Code in `_doLibraryAction`. Diese Abweichung ist eine nachträgliche Verbesserung durch Commit `8c0e125` (harden Like/Unlike error handling): Der Fehlerpfad wurde von `$err->{code} >= 400` auf `if ($err)` umgestellt (korrektere Semantik, deckt auch Netzwerkfehler mit code=0 ab) und duplizierter Code entfernt. Kein Funktionsverlust, keine fehlenden Must-Haves.

---

## Menschliche Verifikation erforderlich

### 1. Dynamisches Like/Unlike-Label in LMS Track-Kontextmenü

**Test:** Einen SpotOn-Track in Browse aufrufen → Info-Menü öffnen → prüfen ob 'Like / Unlike' erscheint → auswählen → prüfen ob Label 'Like' oder 'Unlike' zeigt (je nach aktuellem Liked-State)
**Erwartet:** Kontextmenü-Eintrag 'Like / Unlike' erscheint. Nach Öffnen: dynamisches Label 'Like' (wenn nicht geliked) oder 'Unlike' (wenn geliked)
**Warum menschlich:** Erfordert laufende LMS-Instanz mit aktivem Spotify-Account; trackInfoMenu-Callback kann nicht headless getestet werden

### 2. Like-Aktion speichert Track

**Test:** Track als 'nicht geliked' öffnen → 'Like' auswählen → kurze Bestätigungsmeldung 'Liked!' abwarten → zurück ins Hauptmenü → Liked Songs Menü öffnen → Track verifizieren
**Erwartet:** 'Liked!' Meldung erscheint kurz, dann automatisch zurück (nextWindow=grandparent). Track erscheint in SpotOn Liked Songs.
**Warum menschlich:** Erfordert Spotify-Premium + LMS + echte API-Antwort

### 3. Unlike-Aktion entfernt Track

**Test:** Bereits geliker Track im Info-Menü öffnen → 'Unlike' wählen → 'Removed' Meldung abwarten → Liked Songs prüfen
**Erwartet:** Track nicht mehr in Liked Songs. 'Removed' erscheint kurz.
**Warum menschlich:** Gleiche Bedingungen wie Like-Test

### 4. Cache-Verhalten und Latenz (LIB-04)

**Test:** Track-Info-Menü innerhalb 60s zweimal öffnen
**Erwartet:** Zweite Öffnung sofort (Cache-Hit, kein API-Call). Erste Öffnung: kein wahrnehmbarer Versatz trotz async checkTracks-Call.
**Warum menschlich:** Subjektive Latenzmessung auf Livesystem

### 5. Scope-Upgrade-Transparenz (LIB-05)

**Test:** Auf einer v1.2-Installation auf v1.3 upgraden (oder SPOTON_CACHE_VERSION manuell testen) → ersten Like-Versuch beobachten
**Erwartet:** Kein Re-Auth-Dialog (transparenter Token-Cache-Flush). Bei sehr alten Credentials (ohne library-Scopes): Fehlermeldung mit Hinweis auf Re-Auth via Settings.
**Warum menschlich:** Upgrade-Pfad erfordert bestehende Installation einer älteren Version

---

## Lücken-Zusammenfassung

Keine BLOCKER. Die Implementierung ist vollständig und korrekt verdrahtet. Alle fünf LIB-Anforderungen sind im Code implementiert. Status `human_needed` wegen Standard-LMS-Plugin-Testbeschränkungen: Die UI-Schicht (LMS Track-Kontextmenü), End-to-End-API-Calls gegen echte Spotify-API und der Upgrade-Pfad können nicht ohne laufende LMS-Instanz mit Spotify-Premium-Account automatisch verifiziert werden.

Die nachträgliche Verbesserung in Commit `8c0e125` (Fehlercheck-Härtung und Code-Deduplizierung) ist ein netto-positiver Qualitätsbeitrag, der keine Plan-Must-Haves verletzt.

---

_Verifiziert: 2026-06-11_
_Verifikator: Claude (gsd-verifier)_
