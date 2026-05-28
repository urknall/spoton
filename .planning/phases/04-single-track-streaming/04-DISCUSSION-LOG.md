# Phase 4: Single-Track Streaming - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-28
**Phase:** 04-single-track-streaming
**Areas discussed:** Format-Auswahl & Transcoding, Settings-Scope Phase 4, Playback-Modus

---

## Format-Auswahl & Transcoding

### Pipeline-Auswahl-Mechanismus

| Option | Description | Selected |
|--------|-------------|----------|
| Spotty-Ansatz | updateTranscodingTable() schreibt custom-convert.conf vor jedem Track dynamisch um | |
| formatOverride-basiert | ProtocolHandler::formatOverride() gibt pro Player das Format zurück, LMS wählt Pipeline aus statischer conf | |
| Claude entscheidet | Research evaluiert beide Ansätze und wählt den zuverlässigsten für LMS 8.0+ | ✓ |

**User's choice:** Claude entscheidet
**Notes:** —

### B&O OGG-Support

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, OGG geht | B&O/UPnPBridge kann OGG Vorbis abspielen | |
| Nein, nur FLAC/PCM | B&O/UPnPBridge braucht FLAC oder PCM | |
| Unsicher, Research klären | Research soll die Format-Matrix prüfen | ✓ |

**User's choice:** Unsicher, Research klären
**Notes:** —

### Race Condition LMS-11

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, kommt vor | Mehrere Player spielen regelmäßig gleichzeitig | |
| Selten/nie | Eigentlich nur ein Player auf Spotify gleichzeitig | |
| Claude entscheidet | Research evaluiert das Risiko und die Lösung | |

**User's choice:** (Freetext) "Ich mache das selten, aber ich denke für andere User könnte das ein Thema sein."
**Notes:** User musste erst geklärt bekommen, was "zwei Player" bedeutet (nicht sync-Gruppen, sondern zwei unabhängige Player auf derselben LMS-Instanz). Danach klare Einschätzung: selten persönlich, aber wichtig für andere User.

### Default-Format

| Option | Description | Selected |
|--------|-------------|----------|
| FLAC (Empfohlen) | Verlustfrei, beste Qualität, squeezelite und moderne Player können FLAC | ✓ |
| PCM raw | Kein Encoding-Overhead, maximale Kompatibilität, aber hohe Bandbreite | |
| Claude entscheidet | Research evaluiert basierend auf LMS-Player-Landschaft | |

**User's choice:** FLAC (Empfohlen)
**Notes:** —

---

## Settings-Scope Phase 4

### Bitrate

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, global in Phase 4 | Ein globales Bitrate-Setting auf der Settings-Seite. Per-Player in Phase 6. | ✓ |
| Hardcoded 320 | Kein UI, Bitrate fest auf 320. Konfiguration komplett in Phase 6. | |
| Claude entscheidet | Research evaluiert ob globales Setting jetzt sinnvoll ist | |

**User's choice:** Ja, global in Phase 4
**Notes:** —

### Volume Normalization

| Option | Description | Selected |
|--------|-------------|----------|
| Global in Phase 4 | Einfacher Toggle auf Settings-Seite | |
| Phase 6 | Erst mit Player-spezifischen Settings | |
| Claude entscheidet | Research evaluiert Timing | ✓ |

**User's choice:** Claude entscheidet
**Notes:** —

### Audio Cache

| Option | Description | Selected |
|--------|-------------|----------|
| Phase 4: an/aus + Größe | Settings-Toggle plus Größenlimit | |
| Phase 4: immer aus | --disable-audio-cache wie aktuell | |
| Claude entscheidet | Research evaluiert ob Cache in Single-Track sinnvoll | ✓ |

**User's choice:** Claude entscheidet
**Notes:** —

### Settings UI Platzierung

| Option | Description | Selected |
|--------|-------------|----------|
| Gleiche Seite, neuer Abschnitt | Neuer Abschnitt 'Streaming' unterhalb Auth-Einstellungen | ✓ |
| Claude entscheidet | Research prüft Spotty/Qobuz Organisationsmuster | |

**User's choice:** Gleiche Seite, neuer Abschnitt
**Notes:** —

---

## Playback-Modus

### Queue-Verhalten bei Track-Tap

| Option | Description | Selected |
|--------|-------------|----------|
| Kontext-Queueing | Track + restliche Tracks aus Album/Playlist in Queue | ✓ |
| Echtes Single-Track | Nur angetippter Track, danach Stille | |
| Single-Track jetzt, Queue später | Phase 4 nur Single-Track, Kontext-Queue in Phase 6 | |

**User's choice:** Kontext-Queueing
**Notes:** —

### Queue-Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Ab angeklicktem Track | Track 5 von 12: Tracks 5-12 in Queue | |
| Ganzes Album/Playlist | Alle 12 Tracks in Queue, Start bei Track 5 | |
| Claude entscheidet | Research prüft Spotty/Qobuz Pattern | |

**User's choice:** (Freetext) "2. fände ich aus UX Sicht Mega - aber das ist zu in der Machbarkeit zu prüfen. Wenn es nicht geht ist 1 auch ok"
**Notes:** User bevorzugt klar Option 2 (ganzes Album einreihen, Start bei angeklicktem Track), akzeptiert aber Option 1 als Fallback falls technisch nicht machbar.

### Kontextlose Tracks

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Single-Track | Suchergebnisse, Kürzlich gehört: nur der angetippte Track | |
| Alle sichtbaren Tracks | Auch hier alle Tracks der aktuellen Liste einreihen | |
| Claude entscheidet | Research prüft Spotty/Qobuz Verhalten | ✓ |

**User's choice:** Claude entscheidet
**Notes:** —

### Gapless Playback

| Option | Description | Selected |
|--------|-------------|----------|
| Muss funktionieren | Nahtlose Übergänge, besonders bei Live-Alben | |
| Nice-to-have | Kleine Pause akzeptabel, Priorität auf Zuverlässigkeit | ✓ |
| Claude entscheidet | Research evaluiert Machbarkeit mit --single-track | |

**User's choice:** Nice-to-have
**Notes:** —

---

## Claude's Discretion

- Pipeline-Auswahl-Mechanismus (updateTranscodingTable vs. formatOverride)
- B&O/UPnPBridge OGG-Support-Matrix
- Race Condition Lösung (atomische Pro-Player-Transcoding)
- Volume Normalization Timing (Phase 4 oder 6)
- Audio Cache Strategie
- Kontextloses Queue-Verhalten
- Gapless Machbarkeit mit --single-track
- Orphaned-Process-Cleanup Strategie

## Deferred Ideas

None — discussion stayed within phase scope
