---
phase: 04-single-track-streaming
verified: 2026-05-28T00:00:00Z
status: human_needed
score: 9/10 must-haves verified
overrides_applied: 0
gaps: []
deferred:
  - truth: "Gapless playback between consecutive tracks (STR-09)"
    addressed_in: "kein späterer Phase explizit — technisch unmöglich mit --single-track"
    evidence: "Plan 01 must_haves: 'Gapless playback (STR-09) is not achievable with --single-track mode; deferred per D-11 as nice-to-have'. CONTEXT.md D-11: 'Gapless ist nice-to-have, keine harte Anforderung.' ROADMAP.md listet STR-09 als Phase-4-Requirement, aber CONTEXT.md D-11 hat es als nice-to-have klassifiziert. Kein späterer Phase deckt es explizit ab."
human_verification:
  - test: "Spotify-Track aus Browse auswählen und Audio prüfen"
    expected: "Audio startet innerhalb von 5 Sekunden über LMS; Standard ist FLAC"
    why_human: "Setzt laufendes LMS + librespot-Binary voraus; kein runnable Entry Point ohne Server"
  - test: "OGG-fähigen Player testen (z.B. squeezelite mit OGG-Capability)"
    expected: "OGG-Pipeline wird gewählt wenn Player OGG unterstützt UND Binary passthrough-fähig ist"
    why_human: "Erfordert echten LMS-Player und Binary-Check-Output zur Verifizierung"
  - test: "MP3-Fallback prüfen: Player der weder OGG noch FLAC nativ unterstützt"
    expected: "LMS sollte MP3-Pipeline wählen; formatOverride gibt 'flc' zurück für alle non-OGG Player"
    why_human: "formatOverride gibt nur 'ogg' oder 'flc' zurück, nie 'mp3'. SC2 besagt 'FLAC or MP3 based on capability' — MP3-Pfad ist nicht via formatOverride erreichbar. Braucht menschliche Beurteilung ob das akzeptabel ist."
  - test: "Seeking: Zu Mitte eines Tracks springen"
    expected: "Playback setzt ab der korrekten Position fort (via --start-position), nicht vom Anfang"
    why_human: "Setzt laufendes LMS mit echtem Spotify-Track voraus"
  - test: "Zwei Player gleichzeitig verschiedene Tracks abspielen"
    expected: "Jeder Player spielt den korrekten Track; kein Transcoding-Table-Race"
    why_human: "Setzt zwei physische LMS-Player und gleichzeitigen Stream-Start voraus"
  - test: "Normalisierungs-Checkbox in Settings speichern und laden"
    expected: "Checkbox-Status bleibt nach Seitenreload erhalten; pref_normalization wird korrekt gespeichert"
    why_human: "Braucht LMS Web-UI und Browser"
---

# Phase 4: Single-Track Streaming Verification Report

**Phase-Ziel:** Users can play any Spotify track found via Browse, with correct transcoding pipeline selection and seeking support
**Verifiziert:** 2026-05-28
**Status:** human_needed
**Re-verification:** Nein — initiale Verifikation

## Ziel-Erreichung

### Observable Truths (Must-Haves aus PLAN-Frontmatter + ROADMAP)

| # | Truth | Status | Evidenz |
|---|-------|--------|---------|
| 1 | formatOverride returns 'ogg' for players with OGG capability and passthrough binary support, 'flc' otherwise | VERIFIED | `ProtocolHandler.pm:40-47`: grep OGG + getCapability('passthrough') Guard; return 'ogg'/'flc' korrekt |
| 2 | updateTranscodingTable injects current bitrate, cache dir, helper name, and normalization flag into all son-* commandTable entries | VERIFIED | `Plugin.pm:1061-1083`: 4 Regex-Substitutionen; iteriert alle `son-*` + `/single-track/` Keys |
| 3 | LMS single-threaded event loop guarantees no race condition between formatOverride calls for different players (LMS-11) | VERIFIED | Dokumentiert in Plugin.pm:1039; LMS-Architektur-Garantie (non-preemptive event loop) |
| 4 | Seeking works via canTranscodeSeek + getSeekData providing timeOffset to --start-position | VERIFIED | `ProtocolHandler.pm:51-57`: canSeek + canTranscodeSeek mit Version-Guard; getSeekData gibt `{ timeOffset => $newtime }` zurück; `custom-convert.conf`: alle 4 Pipelines haben `RT:{START=--start-position %s}` |
| 5 | Audio cache is always disabled (--disable-audio-cache present in all four son-* pipelines) | VERIFIED | `grep -c 'disable-audio-cache' custom-convert.conf` = 4; updateTranscodingTable berührt diesen Flag nicht |
| 6 | Gapless playback (STR-09) deferred per D-11 — no Phase 4 implementation | VERIFIED | Plan 01 must_haves explizit dokumentiert; CONTEXT.md D-11 bestätigt; kein Gapless-Code vorhanden |
| 7 | Tapping a track in album/playlist context queues all feed tracks and starts at tapped track | VERIFIED | `Plugin.pm:327` + `Plugin.pm:975`: `playall => 1` in _trackItem und _albumTrackItem |
| 8 | Orphaned librespot processes are cleaned up every 3600 seconds when no player is actively playing | VERIFIED | `Plugin.pm:20`: KILL_PROCESS_INTERVAL=3600; `Plugin.pm:67-73`: killTimers+setTimer in initPlugin; `Plugin.pm:112-148`: _killOrphanedProcesses mit isPlaying-Guard + pkill |
| 9 | Settings page shows normalization checkbox; persists across page reloads | VERIFIED | `Settings.pm:29`: 'normalization' in prefs(); `Settings.pm:52-53`: ternary save; `basic.html:23-27`: WRAPPER mit pref_normalization checkbox + checked condition |
| 10 | Players that do not support OGG receive FLAC or MP3 based on capability (SC2) | UNCERTAIN | formatOverride gibt nur 'ogg' oder 'flc' zurück — MP3 nie via formatOverride wählbar. MP3-Pipeline existiert in custom-convert.conf. Ob LMS MP3 als weiteren Fallback wählen kann (wenn Player FLAC ablehnt), ist ohne echten Player nicht verifizierbar. |

**Punkte:** 9/10 (1 UNCERTAIN — braucht human verify)

### Deferred Items

Items die technisch nicht erreicht wurden, aber per Roadmap-Kontext akzeptabel sind.

| # | Item | Addressed In | Evidenz |
|---|------|-------------|---------|
| 1 | STR-09: Gapless playback between consecutive tracks | Kein späterer Phase — technische Unmöglichkeit | D-11 CONTEXT.md: "nice-to-have, nicht kritisch". Plan 01 must_haves: explizit als nicht erreichbar mit --single-track dokumentiert. kein späterer Phase übernimmt STR-09. |

### Required Artifacts

| Artifact | Erwartet | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/ProtocolHandler.pm` | Dynamic formatOverride mit CapabilitiesHelper | VERIFIED | Existiert, substantiell (59 Zeilen), korrekt verdrahtet: require Plugin + updateTranscodingTable Aufruf + supportedFormats + passthrough Guard |
| `Plugins/SpotOn/Plugin.pm` | updateTranscodingTable + normalization pref + playall + _killOrphanedProcesses | VERIFIED | Existiert, substantiell (1087 Zeilen), alle Methoden implementiert und verdrahtet |
| `Plugins/SpotOn/Settings.pm` | normalization pref handling | VERIFIED | Existiert, 'normalization' in prefs() + saveSettings |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | pref_normalization Checkbox | VERIFIED | Existiert, 3 Vorkommen von pref_normalization; WRAPPER mit PLUGIN_SPOTON_NORMALIZATION |
| `Plugins/SpotOn/strings.txt` | PLUGIN_SPOTON_NORMALIZATION* i18n-Strings | VERIFIED | 3 Vorkommen von PLUGIN_SPOTON_NORMALIZATION; SON bleibt letzter Eintrag |
| `Plugins/SpotOn/custom-convert.conf` | 4 son-* Pipelines mit disable-audio-cache + single-track | VERIFIED | Alle 4 Pipelines vorhanden; 4x --disable-audio-cache; 4x --single-track; START-Seeking in allen Pipelines |

### Key Link Verification

| Von | Nach | Via | Status | Details |
|-----|------|-----|--------|---------|
| `ProtocolHandler.pm::formatOverride` | `Plugin.pm::updateTranscodingTable` | `require Plugins::SpotOn::Plugin; Plugin->updateTranscodingTable($client)` | WIRED | Direkter Aufruf in formatOverride vor Format-Selektion |
| `Plugin.pm::updateTranscodingTable` | `Slim::Player::TranscodingHelper` | `TranscodingHelper::Conversions()` Hashref-Modifikation | WIRED | `Plugin.pm:1060`: `my $commandTable = Slim::Player::TranscodingHelper::Conversions()` |
| `Plugin.pm::initPlugin` | `Slim::Utils::Timers` | `setTimer($class, time+KILL_PROCESS_INTERVAL, \&_killOrphanedProcesses)` | WIRED | `Plugin.pm:68-73`: killTimers + setTimer in !main::SCANNER Block |
| `Settings.pm::prefs` | `Plugin.pm::updateTranscodingTable` | normalization pref gelesen via `$prefs->get('normalization')` | WIRED | `Settings.pm:29` gibt 'normalization' zurück; `Plugin.pm:1044` liest es |
| `Plugin.pm::_trackItem` | `Slim::Control::XMLBrowser` | `playall => 1` in OPML item hash | WIRED | `Plugin.pm:327` + `Plugin.pm:975`: playall in beiden Track-Item-Buildern |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Quelle | Echte Daten | Status |
|----------|---------------|--------|-------------|--------|
| `Plugin.pm::updateTranscodingTable` | `$bitrate` | `$prefs->get('bitrate')` | Ja — LMS Prefs-Store | FLOWING |
| `Plugin.pm::updateTranscodingTable` | `$normalize` | `$prefs->get('normalization')` | Ja — LMS Prefs-Store | FLOWING |
| `Plugin.pm::updateTranscodingTable` | `$cacheDir` | `preferences('server')->get('cachedir') . '/spoton'` | Ja — LMS Server-Prefs | FLOWING |
| `Plugin.pm::updateTranscodingTable` | `$commandTable` | `TranscodingHelper::Conversions()` | Ja — globaler LMS CommandTable Hashref | FLOWING |
| `Settings.pm::handler` | `$norm` | `$paramRef->{'pref_normalization'}` POST-Parameter | Ja — Browser-POST, ternary erzwingt 0 oder 1 | FLOWING |

### Behavioral Spot-Checks

Nicht ausführbar ohne laufenden LMS-Server. Perl-Syntax-Checks scheitern erwartungsgemäß an fehlenden LMS-Modulen (Log::Log4perl, Path::Class) außerhalb des LMS-Kontexts.

| Verhalten | Prüfmethode | Ergebnis | Status |
|-----------|-------------|---------|--------|
| ProtocolHandler.pm Perl-Struktur | `perl -c` | Scheitert wegen Slim::Formats::RemoteStream (LMS-abhängig, erwartet) | SKIP |
| Plugin.pm Perl-Struktur | `perl -c` | Scheitert wegen Log::Log4perl (LMS-abhängig, erwartet) | SKIP |
| custom-convert.conf: 4 Pipelines vorhanden | `grep -c 'son pcm\|son flc\|son mp3\|son ogg'` | 4 Treffer | PASS |
| --disable-audio-cache in allen Pipelines | `grep -c 'disable-audio-cache' custom-convert.conf` | 4 | PASS |
| seeking in allen Pipelines | `grep -c 'start-position'` | 4 Pipelines | PASS |
| playall in beiden Track-Item-Buildern | `grep -c 'playall'` in Plugin.pm | 2 | PASS |
| KILL_PROCESS_INTERVAL = 3600 | `grep 'KILL_PROCESS_INTERVAL'` | 4 Vorkommen (Deklaration + Nutzung) | PASS |

### Anforderungs-Coverage

| Anforderung | Plan | Beschreibung | Status | Evidenz |
|-------------|------|--------------|--------|---------|
| STR-01 | 04-01 | Single-track via librespot --single-track | SATISFIED | 4 Pipelines in custom-convert.conf mit --single-track |
| STR-02 | 04-01 | FLAC als Default-Pipeline | SATISFIED | getFormatForURL='flc'; formatOverride gibt 'flc' als Fallback |
| STR-03 | 04-01 | PCM Passthrough Pipeline | SATISFIED | `son pcm * *` in custom-convert.conf mit RT-Flag + START |
| STR-04 | 04-01 | MP3 Transcoding als Legacy-Fallback | SATISFIED | `son mp3 * *` in custom-convert.conf mit lame-Pipe |
| STR-05 | 04-01 | OGG-Direct Passthrough | SATISFIED | `son ogg * *` mit --passthrough; formatOverride wählt 'ogg' mit doppelter Guard |
| STR-06 | 04-01/04-02 | Bitrate 96/160/320 kbps via Settings | SATISFIED | Settings.pm bitrate-Validierung; updateTranscodingTable injiziert Bitrate; basic.html Select |
| STR-07 | 04-01 | Seeking via --start-position | SATISFIED | canSeek + canTranscodeSeek + getSeekData in ProtocolHandler.pm; RT-Flag in allen Pipelines |
| STR-08 | 04-01/04-02 | Volume Normalization optional | SATISFIED | normalization pref init (Plugin.pm:42); updateTranscodingTable Regex-Injektion; Settings UI + strings.txt |
| STR-09 | — | Gapless Playback | DEFERRED | D-11: nice-to-have, nicht realisierbar mit --single-track (separater Prozess pro Track) |
| STR-10 | 04-02 | Hourly orphaned process cleanup | SATISFIED | _killOrphanedProcesses Timer 3600s; isPlaying-Guard; pkill/taskkill |
| STR-11 | 04-01 | Audio Cache deaktiviert | SATISFIED | --disable-audio-cache in allen 4 Pipelines; updateTranscodingTable berührt ihn nicht |
| LMS-11 | 04-01 | Transcoding table per-track, kein Race | SATISFIED | LMS single-threaded event loop + serialisierte formatOverride-Aufrufe; dokumentiert in Plugin.pm |

**Orphaned Requirements:** STR-09 erscheint in ROADMAP Phase 4 Requirements, aber kein Plan hat ihn als `requirements:` deklariert. Plan 01 must_haves dokumentiert ihn explizit als nicht lieferbar. Per D-11 (CONTEXT.md) vom User als nice-to-have akzeptiert.

### Anti-Pattern-Scan

| Datei | Zeile | Pattern | Schwere | Auswirkung |
|-------|-------|---------|---------|------------|
| — | — | Keine TBD/FIXME/XXX ohne Ticketreferenz | — | — |
| Plugin.pm | 1056 | "placeholder" im Kommentar | Info | Beschreibt `[spoton]` Platzhalter in custom-convert.conf — korrekte Dokumentation, kein Stub |
| Plugin.pm | 1016 | "null track entries" im Kommentar | Info | T-03-10 Sicherheitscheck gegen null-Einträge — korrekter Code |

Keine Blocker-Anti-Pattern gefunden.

### PHASE-5-NOTE Audit

| Datei | Zeile | Inhalt | Status |
|-------|-------|--------|--------|
| Plugin.pm | 134 | `# PHASE-5-NOTE: Phase 5 must exclude Connect daemon PIDs here (Pitfall 6, CON-09)` | Korrekt — explizite Markierung für Phase 5 Connect-Daemon-PID-Ausschluss |

Das PHASE-5-NOTE-Kommentar ist eine planmäßig gesetzte Forward-Reference, keine offene Schuld.

### Human Verification Required

#### 1. Komplettes Playback-Szenario (SC1)

**Test:** Einen Spotify-Track aus dem Browse-Menü in LMS auswählen  
**Erwartet:** Audio startet innerhalb von 5 Sekunden; Standard-Format ist FLAC (son-flc Pipeline)  
**Warum human:** Erfordert laufendes LMS + librespot-Binary + Spotify-Konto

#### 2. OGG-Direct Pfad (SC2 / STR-05)

**Test:** squeezelite mit OGG-Unterstützung als Player konfigurieren; Track abspielen  
**Erwartet:** `son ogg` Pipeline wird gewählt (erkennbar in LMS-Debug-Log)  
**Warum human:** Erfordert echten OGG-fähigen Player + librespot-Binary mit passthrough-Capability

#### 3. MP3-Fallback (SC2 — UNCERTAIN)

**Test:** Player ohne OGG- und FLAC-Fähigkeit konfigurieren; Track abspielen  
**Erwartet:** Per SC2: "FLAC or MP3 based on capability". formatOverride gibt immer 'flc' für non-OGG — MP3 kann LMS nur intern wählen wenn FLAC abgelehnt wird  
**Warum human:** Die Implementierung gibt nie 'mp3' von formatOverride zurück. Es ist unklar ob LMS automatisch auf MP3 zurückfällt. Falls nicht, ist SC2 teilweise nicht erfüllt — muss mit echtem veralteten Player getestet werden.

#### 4. Seeking (SC3 / STR-07)

**Test:** Track 5 Minuten abspielen, dann auf 2:30 seeked; prüfen ob korrekte Position  
**Erwartet:** Playback startet bei 2:30 (nicht am Anfang)  
**Warum human:** Erfordert laufendes LMS + librespot-Binary

#### 5. Zwei-Player-Race (SC4 / LMS-11)

**Test:** Zwei LMS-Player gleichzeitig verschiedene Tracks starten  
**Erwartet:** Jeder Player spielt den korrekten Track; kein falsch injizierter commandTable-Eintrag  
**Warum human:** Erfordert zwei physische Player; Race-Condition-Test

#### 6. Normalisierungs-Persistenz (STR-08)

**Test:** Normalisierungs-Checkbox in Settings aktivieren, speichern, Seite neu laden  
**Erwartet:** Checkbox bleibt aktiviert; pref_normalization=1 in LMS-Prefs gespeichert  
**Warum human:** Erfordert LMS Web-UI + Browser

### Gaps-Zusammenfassung

**Keine harten Blocker.** Alle Must-Haves aus den PLAN-Frontmattern sind im Code verifiziert.

**Beobachtung zu SC2 (MP3-Pfad):** Success Criterion 2 besagt "Players that do not support OGG receive FLAC or MP3 based on capability." Die Implementierung gibt `formatOverride` nur 'ogg' oder 'flc' zurück — nie 'mp3'. Die son-mp3 Pipeline existiert in custom-convert.conf. Es ist unbekannt, ob LMS intern auf MP3 zurückfällt wenn ein Player FLAC ablehnt. Dies ist ein UNCERTAIN (nicht FAILED), da:
1. Die PLAN-Frontmatter und RESEARCH.md dokumentieren FLAC als Default-Fallback (D-04)
2. LMS könnte intern MP3 wählen wenn FLAC nicht vom Player akzeptiert wird
3. Ohne echten non-FLAC-Player ist dies nicht programmatisch verifizierbar

Empfehlung: Im human-verify explizit einen sehr alten/eingeschränkten LMS-Client testen.

---

*Verifiziert: 2026-05-28*  
*Verifier: Claude (gsd-verifier)*
