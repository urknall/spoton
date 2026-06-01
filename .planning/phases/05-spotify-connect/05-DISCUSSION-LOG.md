# Phase 5: Spotify Connect - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-01
**Phase:** 5-Spotify Connect
**Areas discussed:** Audio-Transport, Daemon-Lifecycle, Sync-Gruppen, Event-Protokoll

---

## Audio-Transport

### Transport-Modell

| Option | Description | Selected |
|--------|-------------|----------|
| HTTP-Streaming (Empfohlen) | librespot-Daemon hat eingebauten HTTP-Server, LMS fetcht Audio als Stream | ✓ |
| FIFO mit HTTP-Fallback | FIFO als Default, HTTP nur bei Seek-Problemen | |
| Reines FIFO | Bekannte Probleme: Seek-Latenz, White Noise | |

**User's choice:** Pur HTTP — nach Rückfrage zu Alternativen (Unix Socket, Shared Memory, Raw TCP) bestätigt, dass HTTP der einzige Ansatz ist der FIFO-Probleme löst UND in LMS' Remote-Stream-Infrastruktur passt.

### Audio-Format

| Option | Description | Selected |
|--------|-------------|----------|
| Immer PCM (S16LE) | Ein Code-Pfad, konsistent | |
| PCM + OGG-Passthrough | Default PCM, OGG für fähige Player | ✓ |
| Immer OGG Passthrough | Limitiert Kompatibilität | |

### Port-Zuweisung

| Option | Description | Selected |
|--------|-------------|----------|
| Dynamisch (OS wählt) | Port 0, OS vergibt, Binary meldet auf stdout | ✓ |
| Manager-Pool (Basis+Offset) | Vorhersehbar, aber Kollisionsrisiko | |
| Du entscheidest | | |

### Pipeline-Architektur

**User's choice:** Transcoding-Pipeline erforderlich — Verweis auf Spotty-NG-Erfahrung. LMS braucht die Pipeline für Format-Erkennung und Sync-Distribution. OGG-Passthrough bei supporteten Playern als Default, mit per-Player-Override zum Wegkonfigurieren bei Problemen.

**Notes:** Spotty-NG-Analyse ergab: `spc` Content-Type + `spc pcm * *` Profil mit Flag `I` und Command `-`. `canDirectStream` für Einzelspieler, `new()` Override für Sync-Gruppen-Proxy. Dieses Pattern wird für SpotOn übernommen (mit `soc` statt `spc`).

### OGG-Passthrough-Konfiguration

| Option | Description | Selected |
|--------|-------------|----------|
| Auto-Detect + Override | Player-Capability + per-Player Override in Settings | ✓ |
| Nur manuell | Default immer PCM, User aktiviert explizit | |
| Du entscheidest | | |

---

## Daemon-Lifecycle

### Start-Zeitpunkt

| Option | Description | Selected |
|--------|-------------|----------|
| LMS-Start (immer an) | Daemon bei Plugin-Init, Player sofort sichtbar | ✓ |
| On-Demand | Erst bei Nutzung, spart Ressourcen | |
| Du entscheidest | | |

### Mutual Exclusion

| Option | Description | Selected |
|--------|-------------|----------|
| Connect verdrängt Browse | Immer nur ein Modus aktiv | ✓ |
| Koexistenz erlauben | Beide Modi parallel | |
| Du entscheidest | | |

### Token-Expiry

| Option | Description | Selected |
|--------|-------------|----------|
| Proaktiver Daemon-Neustart | Timer bei ~50min, kurze Unterbrechung | |
| Token-Refresh im Binary | librespot refresht intern, kein Neustart | ✓ |
| Du entscheidest | | |

### Crash-Recovery

| Option | Description | Selected |
|--------|-------------|----------|
| Exponential Backoff | Spotty-NG-Pattern | |
| Fester Retry | Immer 5s, max 3 Versuche | |
| Du entscheidest | | ✓ |

### Per-Player Toggle

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, per Player | Toggle pro Player, default: an | ✓ |
| Global an/aus | Ein Toggle für alle | |
| Du entscheidest | | |

---

## Sync-Gruppen

### Daemon-Zuordnung

| Option | Description | Selected |
|--------|-------------|----------|
| Master-Only Daemon | Spotty-NG-Pattern | |
| Du entscheidest | | ✓ |

### Device-Name

| Option | Description | Selected |
|--------|-------------|----------|
| Verkettete Namen | 'Wohnzimmer + Küche' via syncname() | ✓ |
| Nur Master-Name | Einfacher | |
| Custom-Name | User vergibt eigenen Namen | |

### Dynamische Gruppen-Änderung

| Option | Description | Selected |
|--------|-------------|----------|
| Differentieller Neustart | Nur betroffene Daemons | |
| Komplett-Neustart | Alle Daemons bei jeder Änderung | |
| Du entscheidest | | ✓ |

### B&O/UPnPBridge

**User's choice:** Keine Vorab-Einschränkungen. Noch nicht erforscht, Sonderfälle im Livebetrieb identifizieren.

---

## Event-Protokoll

### Protokoll-Architektur

| Option | Description | Selected |
|--------|-------------|----------|
| Spotty-NG-kompatibel | Gleiche Struktur, nur Name ändern | |
| Neu denken | Neues Design, ggf. WebSocket | ✓ |
| Du entscheidest | | |

**Notes:** User wollte freie Analyse. Nach Durchdenken der Optionen (JSON-RPC POST, WebSocket, SSE) empfahl Claude: JSON-RPC POST für Binary→LMS (LMS-natives Dispatch), HTTP-Control-Endpoints für LMS→Binary (schneller als Spotify Web API). User stimmte zu.

### Volume-Synchronisation

| Option | Description | Selected |
|--------|-------------|----------|
| Grace-Period übernehmen | 20s wie Spotty-NG | |
| Bessere Lösung suchen | Research prüft librespot 0.8 Optionen | |
| Du entscheidest | | ✓ |

### Loop-Prevention

| Option | Description | Selected |
|--------|-------------|----------|
| Source-Marking übernehmen | Spotty-NG-Pattern | |
| Du entscheidest | | ✓ |

---

## Claude's Discretion

- Crash-Recovery-Strategie (Backoff-Parameter, Discovery-Deaktivierungsschwelle)
- Sync-Gruppen: Master-Only vs. Alternativ, Differentiell vs. Komplett-Neustart
- Loop-Prevention-Mechanismus
- Volume-Grace-Period-Strategie
- Debouncing für Volume/Seek-Events
- Enriched Event-Payload-Schema
- killHangingProcesses-Schutz (CON-09)

## Deferred Ideas

- Per-Player OGG-Passthrough auch für Browse/Single-Track-Modus (nicht nur Connect) → Phase 6 (LMS-08)
- CON-12 Requirement-Update: "FIFO-based" → "HTTP-streaming" in REQUIREMENTS.md
