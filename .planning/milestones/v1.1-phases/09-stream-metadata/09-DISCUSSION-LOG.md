# Phase 9: Stream Metadata - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-04
**Phase:** 09-Stream Metadata
**Areas discussed:** Display-String, Format-Erkennung, Bitrate-Semantik

---

## Display-String

### Q1: Template-Format

| Option | Description | Selected |
|--------|-------------|----------|
| Bitrate, Format (Modus) | z.B. '320k, OGG (Spotify Browse)'. Wie im META-03 Requirement. | ✓ |
| Format (Modus) Bitrate | z.B. 'OGG (Spotify Browse) 320k'. Format zuerst. | |
| Du entscheidest | Claude wählt passend zu LMS-Konventionen. | |

**User's choice:** Bitrate, Format (Modus)
**Notes:** Keine

### Q2: Modus-Anzeige

| Option | Description | Selected |
|--------|-------------|----------|
| Immer mit Modus | Selbst als Fallback steht mindestens '(Spotify Browse)' oder '(Spotify Connect)' | ✓ |
| Nur bei Format-Info | Modus nur zusammen mit Format/Bitrate, sonst 'Spotify' | |

**User's choice:** Immer mit Modus
**Notes:** Keine

### Q3: Format-Bezeichnungen

| Option | Description | Selected |
|--------|-------------|----------|
| Langform | OGG Vorbis, FLAC, MP3, PCM | |
| Kurzform | OGG, FLAC, MP3, PCM | ✓ |

**User's choice:** Kurzform
**Notes:** Keine

---

## Format-Erkennung

### Q1: Auto-Format-Bestimmung

| Option | Description | Selected |
|--------|-------------|----------|
| Pref-basiert | Bei 'auto' zeige OGG wenn Passthrough, sonst PCM | |
| Pipeline-Lookup | Tatsächlichen Transcoding-Key aus LMS abfragen | |
| Du entscheidest | Claude wählt pragmatischsten Ansatz | ✓ |

**User's choice:** Du entscheidest
**Notes:** Keine

### Q2: Connect-Format

| Option | Description | Selected |
|--------|-------------|----------|
| Gleicher Mechanismus | Browse und Connect nutzen dieselbe streamFormat-Pref-Logik | ✓ |
| Connect immer PCM | Connect zeigt immer PCM außer bei soc-ogg | |

**User's choice:** Gleicher Mechanismus
**Notes:** Keine

---

## Bitrate-Semantik

### Q1: Bitrate bei PCM/FLAC

| Option | Description | Selected |
|--------|-------------|----------|
| Quell-Bitrate | Immer Spotify-Quellbitrate (96/160/320k), unabhängig vom Ausgabeformat | ✓ |
| Nur bei OGG/MP3 | Bitrate weglassen bei PCM und FLAC | |
| Immer Quell-Bitrate | Identisch mit Option 1 | |

**User's choice:** Quell-Bitrate
**Notes:** Keine

### Q2: MP3-Bitrate

| Option | Description | Selected |
|--------|-------------|----------|
| Spotify-Quelle | Immer eingestellte Spotify-Bitrate (konsistent) | ✓ |
| LAME-Ziel | Tatsächliche MP3-Bitrate die LMS ausgibt | |

**User's choice:** Spotify-Quelle
**Notes:** Keine

---

## Claude's Discretion

- Format-Erkennung bei `auto`: Claude wählt den pragmatischsten Ansatz basierend auf LMS-API-Verfügbarkeit

## Deferred Ideas

None — discussion stayed within phase scope.
