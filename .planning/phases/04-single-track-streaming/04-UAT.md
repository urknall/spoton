---
status: resolved
phase: 04-single-track-streaming
source:
  - 04-01-SUMMARY.md
  - 04-02-SUMMARY.md
started: 2026-05-28T00:00:00Z
updated: 2026-05-28T22:00:00Z
---

## Current Test

[testing paused — streaming bugs muessen in Phase 4.1 gefixt werden]

## Tests

### 1. Track-Wiedergabe aus Browse-Menu
expected: Ein Track aus einem Album, einer Playlist oder den Suchergebnissen auswaehlen. Audio startet innerhalb von 5 Sekunden ueber LMS. Der Player zeigt Titel und Kuenstler an.
result: issue
reported: "Urspruenglich mit Spotty aktiv getestet (pass). Re-Test ohne Spotty: Metadata kommt, kein Artwork, Song startet nicht, bleibt bei 0. Log: 'Couldn't resolve IP address for: spotify' und 'stream failed to open [spotify://spotify:track:...]'. Doppeltes spotify-Prefix in URL und ProtocolHandler wird nicht als Transcoding-Handler erkannt."
severity: blocker

### 2. Format-Auswahl: FLAC als Standard
expected: Bei einem Player ohne OGG-Faehigkeit (z.B. Standard-Squeezebox) wird FLAC als Format gewaehlt. In den LMS Debug-Logs erscheint "formatOverride returning 'flc'" oder das Transcoding zeigt son->flc.
result: blocked
blocked_by: streaming
reason: "Urspruenglich mit Spotty aktiv getestet (pass). Ohne Spotty funktioniert Streaming nicht — Re-Test nach Phase 4.1 URL-Fix."

### 3. Format-Auswahl: OGG-Passthrough
expected: Bei einem Player mit OGG-Faehigkeit UND einem librespot-Binary mit passthrough-Support wird OGG als Format gewaehlt. In den Debug-Logs erscheint "formatOverride returning 'ogg'" oder das Transcoding zeigt son->ogg.
result: blocked
blocked_by: prior-phase
reason: "librespot-Binary ohne passthrough-decoder Feature kompiliert. Cargo.toml braucht librespot-playback mit passthrough-decoder. Verschoben nach Phase 4.1."

### 4. Seeking innerhalb eines Tracks
expected: Waehrend der Wiedergabe eines Spotify-Tracks die Seek-Leiste (LMS Remote oder App) auf die Mitte des Tracks ziehen. Die Wiedergabe setzt an der neuen Position fort, nicht vom Anfang.
result: blocked
blocked_by: streaming
reason: "Urspruenglich mit Spotty aktiv getestet (pass). Ohne Spotty funktioniert Streaming nicht — Re-Test nach Phase 4.1 URL-Fix."

### 5. Kontext-Queueing: Album
expected: In einem Album einen Track antippen. Nicht nur dieser Track wird abgespielt, sondern alle Tracks des Albums werden in die Queue geladen. Nach Ende des angetippten Tracks spielt der naechste Track des Albums automatisch weiter.
result: issue
reported: "playall hat keinen sichtbaren Effekt. Play-Button neben Track = Single-Track-Play. Klick auf Titel-Text = nur Kontextmenue (Album/Kuenstler anzeigen), kein Play. LMS 'Alle Titel'-Eintrag: Klick auf Text = keine Reaktion; Klick auf Play-Button = Queue gefuellt, Track 1 startet, aber kein Audio und kein Artwork."
severity: blocker

### 6. Kontext-Queueing: Suchergebnisse / Playlist
expected: In den Suchergebnissen oder einer Playlist einen Track antippen. Alle sichtbaren Tracks werden in die Queue geladen, Wiedergabe startet beim angetippten Track und setzt mit den folgenden Tracks fort.
result: issue
reported: "Gleiches Verhalten wie Test 5 — playall ohne Effekt. Play-Button = Single Track. 'Alle Titel' Play-Button = Queue gefuellt, aber kein Audio, kein Artwork."
severity: blocker

### 7. Orphaned-Process-Cleanup
expected: Nach laengerem Gebrauch (mind. 1 Stunde) und nachdem alle Player gestoppt wurden, laeuft kein verwaister librespot-Prozess mehr. Pruefbar via `ps aux | grep librespot` — nach dem naechsten Cleanup-Zyklus sollten keine Prozesse mehr aktiv sein.
result: skipped
reason: "Braucht Langzeit-Beobachtung (mind. 1 Stunde)"

### 8. Settings: Normalisierungs-Checkbox
expected: Unter LMS Settings > SpotOn erscheint eine Checkbox "Lautstaerke normalisieren (ReplayGain)" (DE) bzw. "Normalize volume (ReplayGain)" (EN). Checkbox ankreuzen, speichern, Seite neu laden — Checkbox ist weiterhin aktiviert. Abwaehlen, speichern, neu laden — Checkbox ist deaktiviert.
result: pass
note: "Info-Mouseoverbox zeigt i18n-Platzhalter statt leerem Text — PLUGIN_SPOTON_NORMALIZATION_DESC ist leer, LMS rendert den Key-Namen. Kosmetisch."

### 9. Settings: Bitrate-Dropdown
expected: Das bestehende Bitrate-Dropdown (96/160/320 kbps) funktioniert weiterhin. Aenderung der Bitrate wird gespeichert und beeinflusst die naechste Transcoding-Tabellen-Aktualisierung.
result: pass
note: "Persistenz bestaetigt. Log-Verifikation der Transcoding-Auswirkung ausstehend."

### 10. Zwei Player gleichzeitig
expected: Zwei verschiedene LMS-Player starten unterschiedliche Spotify-Tracks gleichzeitig. Jeder Player spielt seinen korrekten Track ab — kein Vertauschen oder Ueberschreiben der Transcoding-Tabelle.
result: blocked
blocked_by: streaming
reason: "Streaming funktioniert ohne Spotty nicht — Re-Test nach Phase 4.1 URL-Fix."

## Summary

total: 10
passed: 2
issues: 3
blocked: 4
skipped: 1
pending: 0

## Gaps

- truth: "SpotOn ProtocolHandler muss spotify:// URIs eigenstaendig streamen koennen"
  status: resolved
  reason: "URL-Konstruktion erzeugt doppeltes Prefix: spotify://spotify:track:... statt spotify://track:... LMS faellt auf RemoteStream zurueck und versucht 'spotify' als Hostnamen aufzuloesen. ProtocolHandler wird nicht als Transcoding-Handler erkannt."
  severity: blocker
  test: 1
  artifacts:
    - Plugins/SpotOn/Plugin.pm (Zeile 326-327, 974-975: URL-Konstruktion)
    - Plugins/SpotOn/ProtocolHandler.pm (Transcoding-Chain-Verdrahtung)
  missing:
    - URL-Prefix-Fix: spotify:track:ID -> track:ID vor dem Prepend von spotify://
    - ProtocolHandler muss von LMS als Transcoding-Handler erkannt werden (nicht RemoteStream)
    - Artwork-Metadata fuer queued Tracks (getMetadataFor Implementierung)
- truth: "Tapping a track in an album feed queues the entire album and starts at the tapped track"
  status: resolved
  reason: "playall => 1 hat keinen sichtbaren Effekt auf Track-Items. LMS 'Alle Titel' Play-Button fuellt Queue, aber kein Audio und kein Artwork."
  severity: blocker
  test: 5
  artifacts:
    - Plugins/SpotOn/Plugin.pm (_trackItem, _albumTrackItem)
  missing:
    - playall-Mechanismus wird von XMLBrowser nicht ausgewertet (Feed-Struktur-Problem?)
    - 'Alle Titel' Queue: kein Audio wegen URL-Prefix-Bug (siehe Gap 1)
    - Artwork fehlt bei Queue-Modus (getMetadataFor fehlt)
- truth: "Tapping a track in search/playlist queues all visible tracks"
  status: resolved
  reason: "Gleiches Problem wie Test 5 — playall ohne Effekt, 'Alle Titel' kein Audio/Artwork."
  severity: blocker
  test: 6
  artifacts:
    - Plugins/SpotOn/Plugin.pm (_trackItem)
  missing: []
- truth: "strings.txt NORMALIZATION_DESC leer verursacht Parse-Fehler in LMS"
  status: resolved
  reason: "LMS Slim::Utils::Strings::parseStrings meldet Error auf Zeilen 226-227. Leere Werte in strings.txt werden nicht korrekt geparst."
  severity: minor
  test: 8
  artifacts:
    - Plugins/SpotOn/strings.txt (Zeile 226-227)
  missing:
    - Leere String-Werte brauchen mindestens einen Platzhalter-Text oder muessen entfernt werden
- truth: "Settings template clientId braucht pref_ Prefix"
  status: resolved
  reason: "LMS warnt: 'Preference names must be prefixed by pref_ in the page template: clientId'. Template-Variable heisst clientId statt pref_clientId."
  severity: minor
  test: 9
  artifacts:
    - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
  missing:
    - clientId im Template auf pref_clientId umbenennen

## Log-Analyse (2026-05-28)

Erkenntnisse aus /var/log/squeezeboxserver/server.log nach Spotty-Deaktivierung:

1. **URL-Doppel-Prefix:** `spotify://spotify:track:75PWvCyXFTuXvwmntGMhuB` — Plugin.pm prepended `spotify://` auf URIs die bereits `spotify:` enthalten
2. **RemoteStream-Fallback:** `Slim::Formats::RemoteStream::new: Error: Couldn't resolve IP address for: spotify` — ProtocolHandler wird nicht korrekt fuer Transcoding erkannt
3. **strings.txt Parse-Fehler:** `Error: Parsing strings.txt line 226/227` — leere DE/EN-Werte fuer NORMALIZATION_DESC
4. **Settings pref_ Warnung:** `Preference names must be prefixed by "pref_"` fuer clientId
5. **Mit Spotty aktiv:** Spotty's ProtocolHandler uebernahm alle spotify:// URIs — SpotOn's ProtocolHandler wurde nie aufgerufen
