---
phase: 11-track-history-metadata
verified: 2026-06-04T18:30:00Z
status: human_needed
score: 7/7 must-haves verified
overrides_applied: 0
human_verification:
  - test: "Browse-Track in History: Artwork und Metadata prüfen"
    expected: "Nach dem Abspielen eines Browse-Tracks erscheint dieser im Verlauf mit korrektem Album-Cover und Metadata (kein generisches Icon)"
    why_human: "Erfordert LMS-Deployment und manuelle Navigation durch Track History — nicht programmatisch prüfbar"
  - test: "Connect-Track in History: Artwork-Anzeige"
    expected: "Nach dem Abspielen via Spotify Connect erscheint der Track im Verlauf mit Cover-Artwork (nicht generisches Icon)"
    why_human: "Erfordert aktive Spotify Connect Session und LMS-Deployment"
  - test: "Cache-Miss Placeholder + Re-Fetch auf Live-System"
    expected: "Nach >1h oder manuellem Cache-Clear zeigt der History-Eintrag kurz 'Loading...' und füllt sich dann mit korrekten Metadata"
    why_human: "Zeitabhängiges Verhalten und notifyFromArray-Reaktion im UI nicht isoliert testbar"
  - test: "Connect History Track ist abspielbar via Browse Pipeline"
    expected: "Klick auf ehemaligen Connect-Track in History startet Wiedergabe ohne Fehler — kein 'Connect mode'-Fehler"
    why_human: "Erfordert LMS-Deployment mit aktivem Player; URL-Translation-Resultat nur live verifizierbar"
---

# Phase 11: Track History Metadata — Verification Report

**Phase Goal:** "Was lief da eben?" shows correct artwork, format, and bitrate for Browse tracks; Connect tracks translate to playable Browse URLs
**Verified:** 2026-06-04T18:30:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Browse-mode tracks replayed from history show correct album artwork (not generic icon) | VERIFIED | 604800s TTL setzt Browse-Metadata für 7 Tage; `_asyncRefetch` füllt Cache nach Miss wieder auf; Test F/I bestätigt Placeholder + Re-Fetch funktioniert |
| 2 | Browse-mode tracks replayed from history show correct streaming format and bitrate in Songinfo | VERIFIED | `getMetadataFor` gibt `_typeString(client, 'Browse')` und `_bitrateForClient(client).'k'` zurück wenn Cache-Hit; Test E bestätigt; async Re-Fetch (Test I) holt Metadata nach TTL-Ablauf |
| 3 | Connect-mode tracks in history are translatable to Browse URLs and can be replayed | VERIFIED | Connect.pm persistiert `spotifyUri`-Feld (Test D/C); ProtocolHandler.pm erkennt `spotify://connect-` URLs, liest `spotifyUri` aus Cache, gibt `play => spotify://track:$trackId` zurück; Test G+J bestätigt Translation und Browse-Label |
| 4 | Cache-miss in getMetadataFor triggers async API re-fetch and populates metadata for expired entries | VERIFIED | `return {} unless $meta` vollständig ersetzt durch `_asyncRefetch + _placeholderMeta`; `%_pendingRefetch` Debounce verhindert Doppel-Fetches; Test F+H+I bestätigt; `grep -c 'return {} unless' ProtocolHandler.pm` = 0 |

**Score:** 4/4 ROADMAP-Wahrheiten verifiziert

### Plan-Level Must-Have Truths (Plan 01 + Plan 02)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | D-02: All spoton_meta_ cache entries use 604800s TTL | VERIFIED | Plugin.pm L422/L1161: `604800`; DontStopTheMusic.pm L275: `604800`; Connect.pm L876: `604800`; Test A/B/C: 0 Matches auf altes `},\s*3600\s*\)` Muster |
| 2 | D-01/D-04: Connect _fetchTrackMetadata persists metadata with spotifyUri field | VERIFIED | Connect.pm L862-878: vollständiger Cache-Write-Block mit `spotifyUri => $trackInfo->{uri}` und Guard `if ($song->streamUrl)`; Test D: 1 nicht-kommentierter Treffer |
| 3 | D-03: getMetadataFor returns placeholder on cache miss instead of empty hashref | VERIFIED | ProtocolHandler.pm L319-323: `unless ($meta) { _asyncRefetch(...); return _placeholderMeta($url); }`; `grep -c 'return {} unless' = 0`; Test F: cover = `/html/images/cover.png` |
| 4 | D-05: Debounce hash prevents duplicate concurrent fetches | VERIFIED | `our %_pendingRefetch` L21 (package-scope); `return if $_pendingRefetch{$url}` L359; `delete` als erstes in Callback L393; Test H: 0 API-Calls bei gesetztem Flag |
| 5 | D-06: Connect history URLs with spotifyUri return Browse-pipeline metadata | VERIFIED | ProtocolHandler.pm L283-304: Block erkennt `spotify://connect-`, liest Cache, extrahiert `trackId` via `m/^spotify:track:([A-Za-z0-9]+)$/`, gibt `play => "spotify://track:$trackId"` zurück; Test G: `play`-Feld enthält korrekten Browse-URL |
| 6 | D-07: Connect history tracks show Browse mode label, not Connect label | VERIFIED | ProtocolHandler.pm L296: `_typeString($client, 'Browse')`; Test J: `type` enthält "Browse", enthält nicht "Connect" |
| 7 | t/11_track_history.t exists and validates TTL, cache persistence, and all truths | VERIFIED | 11 Subtests, alle PASS; kein TODO-Block verbleibt; `perl t/11_track_history.t` = `1..11` alle ok |

**Score:** 7/7 Plan-Must-Haves verifiziert

### Deferred Items

Keine. Alle Requirements vollständig in Phase 11 umgesetzt.

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/Plugin.pm` | 7-day TTL for Browse metadata cache | VERIFIED | L422: `}, 604800)` (_trackItem); L1161: `}, 604800)` (_albumTrackItem); KILL_PROCESS_INTERVAL=3600 bleibt korrekt (Stundentakt Prozessbereinigung, kein Cache-TTL) |
| `Plugins/SpotOn/DontStopTheMusic.pm` | 7-day TTL for DSTM metadata cache | VERIFIED | L275: `}, 604800)` (_cacheAndExtractUris) |
| `Plugins/SpotOn/Connect.pm` | Connect metadata cache persistence with spotifyUri | VERIFIED | L5: `use Digest::MD5 qw(md5_hex)` importiert; L862-878: vollständiger Cache-Write-Block mit `spotifyUri`-Feld und `604800` TTL |
| `Plugins/SpotOn/ProtocolHandler.pm` | Async re-fetch, debounce, placeholder, Connect-to-Browse translation | VERIFIED | `%_pendingRefetch` (our), `_asyncRefetch`, `_placeholderMeta`, `_largestImage` alle vorhanden; 14 Treffer auf diese Funktionen; Connect-Translation-Block L283-304 |
| `t/11_track_history.t` | Unit tests for Phase 11 history metadata | VERIFIED | 11 Subtests (A-J); 0 TODO-Blöcke; vollständige Stub-Suite inkl. API::Client mit `$mock_track` und `Slim::Control::Request` mit `$notify_count` |

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `Connect.pm` | `Slim::Utils::Cache` | `cache->set` mit `spoton_meta_` Key und `spotifyUri`-Feld | WIRED | L863-877: `Slim::Utils::Cache->new()->set('spoton_meta_' . md5_hex($song->streamUrl), {..., spotifyUri => $trackInfo->{uri}}, 604800)` |
| `ProtocolHandler.pm` | `Plugins::SpotOn::API::Client` | `require + getTrack` in `_asyncRefetch` | WIRED | L387: `require Plugins::SpotOn::API::Client`; L389: `Plugins::SpotOn::API::Client->getTrack($accountId, $trackId, sub {...})` |
| `ProtocolHandler.pm` | `Slim::Control::Request` | `notifyFromArray` in async Callback | WIRED | L422: `require Slim::Control::Request`; L423: `Slim::Control::Request::notifyFromArray($client, ['newmetadata'])` mit `$client`-Guard |

## Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|--------------------|--------|
| `ProtocolHandler.pm::getMetadataFor` | `$meta` (Browse cache hit) | `$cache->get('spoton_meta_' . md5_hex($canonical))` | Ja — Plugin.pm/_albumTrackItem schreiben echte API-Daten mit 604800s TTL | FLOWING |
| `ProtocolHandler.pm::_asyncRefetch` | `$trackInfo` (Re-Fetch) | `API::Client->getTrack($accountId, $trackId, cb)` | Ja — echter API-Call; Callback schreibt in Cache via `$cache->set` | FLOWING |
| `ProtocolHandler.pm::getMetadataFor` | `$connect_meta` (Connect Translation) | `$cache->get('spoton_meta_' . md5_hex($url))` | Ja — Connect.pm schreibt in Plan 01 mit `spotifyUri`-Feld | FLOWING |

## Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| 11 Phase-11-Tests alle grün | `perl t/11_track_history.t` | `1..11`, alle ok | PASS |
| Syntax aller modifizierten Dateien | `perl t/05_perl_syntax.t` | `1..6`, alle ok | PASS |
| `return {} unless $meta` vollständig entfernt | `grep -c 'return {} unless' ProtocolHandler.pm` | `0` | PASS |
| Connect.pm hat spotifyUri | `grep -c 'spotifyUri' Connect.pm` | `2` (1 Code + 1 Kommentar) | PASS |
| Plugin.pm hat 2x 604800 | `grep -c '604800' Plugin.pm` | `4` (2 Cache-Sets + 2 Kommentarzeilen) | PASS |
| DontStopTheMusic.pm hat 604800 | `grep -c '604800' DontStopTheMusic.pm` | `2` (1 Cache-Set + 1 Kommentarzeile) | PASS |
| `_asyncRefetch` Definition + Aufruf | `grep -c '_asyncRefetch' ProtocolHandler.pm` | `>= 2` | PASS |

## Probe Execution

Keine konventionellen `scripts/*/tests/probe-*.sh` Probes für diese Phase definiert.

## Requirements Coverage

| Requirement | Source Plan | Beschreibung | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| HIST-01 | 11-01-PLAN.md | Browse-mode tracks replayed from history show correct album artwork | SATISFIED | 604800s TTL hält Artwork 7 Tage; `_asyncRefetch` füllt nach Cache-Miss auf; Test A+E+I |
| HIST-02 | 11-01-PLAN.md | Browse-mode tracks show correct streaming format and bitrate in Songinfo | SATISFIED | `_typeString`/`_bitrateForClient` Pipeline greift bei Cache-Hit; Re-Fetch stellt Metadata wieder her; Test E |
| HIST-03 | 11-01-PLAN.md + 11-02-PLAN.md | Connect-mode tracks translatable to Browse URLs and replayable | SATISFIED | Connect.pm persistiert `spotifyUri`; ProtocolHandler.pm übersetzt zu `spotify://track:ID`; Test G+J |
| HIST-04 | 11-02-PLAN.md | Cache-miss triggers async API re-fetch and populates metadata | SATISFIED | `_asyncRefetch` + `%_pendingRefetch` Debounce + `notifyFromArray`; Test F+H+I |

**Hinweis:** HIST-01 bis HIST-04 sind in `REQUIREMENTS.md` (v1.1/v1.2+ Abschnitte) **nicht registriert** — sie erscheinen nur in den Phasen-Kontextdokumenten (`11-RESEARCH.md`, `11-CONTEXT.md`) und ROADMAP.md. Die Anforderungen existieren und sind erfüllt, fehlen aber im zentralen Requirements-Register. Dies ist eine Dokumentationslücke, kein Implementierungsfehler.

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `ProtocolHandler.pm` | 269 | Kommentar "TTL: 3600s" im Docblock von `getMetadataFor` | INFO | Irreführender Docstring — der eigentliche TTL im Cache-Set ist 604800; Cache-Miss wird nicht mehr mit `{}` beantwortet. Kein Blocker, nur veralteter Kommentar. |

Keine TBD/FIXME/XXX-Marker in phasen-modifizierten Dateien gefunden. Keine Stub-Implementierungen. Keine leeren Rückgaben in produktivem Code-Pfad.

**Vorhandene Pre-existing Test-Failures (nicht durch Phase 11 verursacht):**
`t/07_token_manager.t` (4 Fehler), `t/08_api_client.t` (Exit 255), `t/09_settings.t` (1 Fehler) — diese existierten bereits vor dem ersten Phase-11-Commit (`bd08192`); Phase 11 hat keinen dieser Dateien modifiziert; bestätigt durch `git log -- t/07 t/08 t/09` (letzter Commit ist `56166e4` aus Phase 04.3, vor Phase 11).

## Human Verification Required

### 1. Browse-Track in History: Artwork und Metadata

**Test:** Einen Track via Browse-Menü abspielen lassen, dann Track History öffnen
**Expected:** Der Track erscheint mit korrektem Album-Cover und Metadata (Titel, Artist, Format, Bitrate) — kein generisches Icon
**Warum Human:** Navigation durch LMS UI und visuelles Prüfen der History-Anzeige nicht programmatisch möglich

### 2. Connect-Track in History: Artwork-Anzeige

**Test:** Einen Track via Spotify Connect abspielen lassen, dann Track History in LMS öffnen
**Expected:** Der Track erscheint mit Cover-Artwork (nicht dem generischen `/html/images/cover.png`-Fallback)
**Warum Human:** Erfordert aktive Spotify Connect Session und deploytes Plugin

### 3. Cache-Miss Placeholder und Re-Fetch auf Live-System

**Test:** Cache-Eintrag manuell löschen (LMS CLI: `pref clear spoton_meta_...`) oder >1h warten, dann Track History aufrufen
**Expected:** Kurze Anzeige von "Loading..." (oder generischem Icon), danach automatisches Update mit korrekter Metadata nach ca. 1-2 Sekunden
**Warum Human:** Zeitabhängiges Verhalten und LMS-UI-Update-Reaktion auf `notifyFromArray` nicht isoliert testbar

### 4. Ehemaliger Connect-Track ist über Browse-Pipeline abspielbar

**Test:** In der Track History einen Track anklicken, der ursprünglich via Spotify Connect gespielt wurde
**Expected:** Track startet via Browse-Pipeline ohne Fehlermeldung; kein "Connect mode required"-Error; URL `spotify://track:ID` wird verwendet
**Warum Human:** Playback-Start und URL-Translation-Resultat erfordern deploytes LMS mit aktivem Player

## Gaps Summary

Keine Gaps. Alle 7 Must-Haves verifiziert.

**Dokumentationslücke (kein Blocker):** HIST-01 bis HIST-04 sind im zentralen `REQUIREMENTS.md` nicht eingetragen. Sie existieren in Phase-11-internen Dokumenten (RESEARCH.md, CONTEXT.md) und ROADMAP.md, aber nicht im Requirements-Register. Empfehlung: Nacherfassen in REQUIREMENTS.md unter einem neuen Abschnitt "Track History" für v1.1.

---

_Verified: 2026-06-04T18:30:00Z_
_Verifier: Claude (gsd-verifier)_
