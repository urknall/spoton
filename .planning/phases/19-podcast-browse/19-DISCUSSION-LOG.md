# Phase 19: Podcast Browse - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-14
**Phase:** 19-podcast-browse
**Areas discussed:** Show-Darstellung, Episoden-Anzeige, Podcast-Suche Platzierung, Show-Detail-Struktur

---

## Show-Darstellung

### line2-Inhalt

| Option | Description | Selected |
|--------|-------------|----------|
| Publisher (Empfohlen) | Nur der Publisher-Name — konsistent mit Playlist-Pattern (Owner auf line2) | ✓ |
| Publisher + Episodenanzahl | z.B. 'BBC · 142 Episoden' — mehr Kontext, aber Extra-API-Call | |
| Publisher + Beschreibung | Publisher auf line2, kurze Beschreibung — aber OPML hat nur line1/line2 | |

**User's choice:** Publisher
**Notes:** Konsistenz mit Playlist-Pattern war ausschlaggebend

### OPML-Type

| Option | Description | Selected |
|--------|-------------|----------|
| link (Empfohlen) | Standard-Ordner-Icon, öffnet Unterebene — wie Artist-Items | ✓ |
| playlist | Playlist-Icon mit Play-Overlay — suggeriert direktes Abspielen | |

**User's choice:** link
**Notes:** Show ist kein direkt abspielbares Element

### Sortierung

| Option | Description | Selected |
|--------|-------------|----------|
| Client-seitig sortieren (Empfohlen) | Alle Shows laden, alphabetisch sortieren | |
| API-Reihenfolge beibehalten | Zuletzt hinzugefügt zuerst — einfacher | ✓ |

**User's choice:** API-Reihenfolge (added_at desc)
**Notes:** SC2 in ROADMAP.md muss angepasst werden (war "alphabetically", wird "by add date")

---

## Episoden-Anzeige

### line2-Inhalt

| Option | Description | Selected |
|--------|-------------|----------|
| Dauer + Datum (Empfohlen) | z.B. '45 Min · 12. Jun 2026' — kompakt, deckt SC3 | ✓ |
| Datum + kurze Beschreibung | z.B. '12. Jun 2026 — In dieser Folge...' | |
| Dauer + Datum + Beschreibung | Alles zusammen — wird lang | |

**User's choice:** Dauer + Datum

### Dauer-Format

| Option | Description | Selected |
|--------|-------------|----------|
| Menschenlesbar (Empfohlen) | '45 Min' oder '1 Std 23 Min' | ✓ |
| mm:ss / hh:mm:ss | '45:00' oder '1:23:00' — technischer | |
| Du entscheidest | Claude wählt basierend auf LMS-Konventionen | |

**User's choice:** Menschenlesbar

### Datum-Format

| Option | Description | Selected |
|--------|-------------|----------|
| Relative Angabe (Empfohlen) | 'Heute', 'Gestern', 'Vor 3 Tagen', dann absolut | ✓ |
| Immer absolut | Immer '12. Jun 2026' — einfacher | |
| Du entscheidest | Claude wählt | |

**User's choice:** Relative Angabe

### Episode-Artwork

| Option | Description | Selected |
|--------|-------------|----------|
| Episode-Artwork bevorzugen (Empfohlen) | Eigenes Artwork wenn vorhanden, sonst Show-Artwork | ✓ |
| Immer Show-Artwork | Einheitlicher Look | |

**User's choice:** Episode-Artwork bevorzugen

---

## Podcast-Suche Platzierung

### Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Voll funktional in Phase 19 (Empfohlen) | Suche schon jetzt implementieren — API fertig, Pattern existiert | ✓ |
| Nur Platzhalter | Menüpunkt mit 'Coming soon' | |
| Weglassen | Komplett in Phase 20 | |

**User's choice:** Voll funktional
**Notes:** Zieht SRC-01/02/03 in Phase 19 vor. Phase 20 fokussiert nur noch auf Follow/Unfollow

### Such-Typen

| Option | Description | Selected |
|--------|-------------|----------|
| Shows + Episoden getrennt (Empfohlen) | Wie globale Suche: separate Unterebenen | ✓ |
| Nur Shows | Einfacher, weniger API-Calls | |
| Gemischt | Flache Liste — ungewöhnlich für OPML | |

**User's choice:** Shows + Episoden getrennt

---

## Show-Detail-Struktur

### Layout

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Episodenliste (Empfohlen) | Direkt die Episoden — wie Album-Detail | |
| Info-Header + Episoden | Erstes Item ist Show-Beschreibung | |
| Du entscheidest | Claude wählt | ✓ |

**User's choice:** Claude's Discretion

### Episoden-Reihenfolge

| Option | Description | Selected |
|--------|-------------|----------|
| Neueste zuerst (Empfohlen) | API-Default, Phase 21 überlagert | ✓ |
| Chronologisch (älteste zuerst) | Braucht client-seitige Umkehrung | |

**User's choice:** Neueste zuerst

---

## Claude's Discretion

- Show-Detail-Struktur: Info-Header vs. nur Episodenliste
- Episode-OPML-Type: type => 'audio' oder anderer Typ
- Episode-URI-Schema: spoton://episode:ID oder alternative
- Podcast-Suche Top-Result: Inline Top-Result wie globale Suche oder schlichter

## Deferred Ideas

None — discussion stayed within phase scope
