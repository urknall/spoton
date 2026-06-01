# Phase 5: Spotify Connect - Context

**Gathered:** 2026-06-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Jeder LMS-Player erscheint als Spotify Connect-Receiver in der Spotify App. Playback kann von der Spotify App auf einen LMS-Player übertragen werden — Audio startet innerhalb von 3 Sekunden mit korrekter Position. Transport-Controls (Play, Pause, Skip, Volume, Seek) funktionieren bidirektional: Spotify App → LMS und LMS → Spotify App. Pro Player (oder Sync-Gruppe) läuft ein librespot-Connect-Daemon mit eingebautem HTTP-Server für Audio-Streaming. Sync-Gruppen erscheinen als ein einzelnes Gerät.

</domain>

<decisions>
## Implementation Decisions

### Audio-Transport
- **D-01:** Pur HTTP-Streaming — kein FIFO, kein Fallback. Das Binary hat einen eingebauten HTTP-Server, LMS fetcht Audio von `http://127.0.0.1:PORT/stream`. Seek = neue HTTP-Verbindung. Bewährt in Spotty-NG, löst FIFO-Probleme (Seek-Lag P-19, White Noise P-20).
- **D-02:** PCM (S16LE) als Default-Format + OGG-Passthrough-Option für fähige Player. OGG-Passthrough spart CPU auf schwachen Geräten. Sync-Gruppen immer PCM (kleinstes gemeinsames Format).
- **D-03:** Dynamische Port-Zuweisung — Binary bindet Port 0, OS vergibt freien Port. Binary meldet Port auf stdout (`stream_port=XXXXX`). Keine Kollisionen, keine Konfiguration.
- **D-04:** Transcoding-Pipeline (eigener Content-Type `soc`) — wie Spotty-NG's `spc`. `formatOverride()` gibt `soc` zurück wenn Connect aktiv. Profile: `soc pcm * *` (Passthrough) und `soc ogg * *` (OGG-Direct). LMS braucht die Pipeline für Format-Erkennung und Sync-Gruppen-Distribution.
- **D-05:** OGG-Passthrough Auto-Detect + Override — Default: Player-Capability entscheidet automatisch (OGG-fähig → Passthrough, sonst PCM). Pro Player ein Override in Settings: 'OGG-Passthrough erzwingen' / 'PCM erzwingen' / 'Auto'.
- **D-06:** `canDirectStream` gibt HTTP-URL zurück für einzelne Player (LMS fetcht direkt). Für Sync-Gruppen gibt es 0 zurück → LMS proxied via `new()` Override (Spotty-NG-Pattern: `Slim::Player::Protocols::HTTP->new` mit substituierter URL).

### Daemon-Lifecycle
- **D-07:** Daemons starten bei LMS-Start (Plugin-Init) für jeden verbundenen Player mit aktiviertem Connect. Player erscheinen sofort in der Spotify App. Kein On-Demand.
- **D-08:** Mutual Exclusion — Connect verdrängt Browse. Connect-Start stoppt laufendes Browse-Streaming sofort. Browse-Start stoppt Connect-Daemon. Immer nur ein Modus aktiv pro Player. Klar und vorhersehbar.
- **D-09:** Token-Refresh im Binary — librespot-Session refresht intern über Keymaster/Spirc. Kein Daemon-Neustart alle 50 Minuten nötig. Research muss prüfen ob librespot 0.8 Session-Refresh nativ unterstützt oder ob Binary-Code nötig ist.
- **D-10:** Connect per Player an/aus — Settings-Toggle pro Player. Default: an für alle. User kann bestimmte Player ausschließen (z.B. wenn ein UPnP-Gerät Probleme macht). CON-10.

### Sync-Gruppen
- **D-11:** Device-Name = verkettete Player-Namen — z.B. 'Wohnzimmer + Küche'. Nutzt `Slim::Player::Sync::syncname()` (Spotty-NG-Pattern). CON-06.
- **D-12:** B&O/UPnPBridge-Sonderfälle — keine Vorab-Einschränkungen. Eventuelle Format- oder Latenz-Probleme werden im Livebetrieb identifiziert und dann als Bug-Fix-Phase adressiert.

### Event-Protokoll
- **D-13:** Binary → LMS: JSON-RPC POST mit angereicherten Events. Gleicher LMS-Dispatch-Mechanismus wie Spotty-NG (`Slim::Control::Request::addDispatch`), aber Events enthalten Metadata direkt im Payload (Track-Name, Artist, Album, Duration, Cover-URL, Position). Spart separate API-Calls nach Track-Wechsel.
- **D-14:** LMS → Binary: HTTP-Control-Endpoints am Binary-HTTP-Server. Keine Spotify Web API für Rückkanal. Endpoints: `POST /control/pause`, `POST /control/play`, `POST /control/volume`, `POST /control/seek`. Binary leitet Befehle an Spirc weiter → Spotify Cloud State bleibt konsistent. Schnell (localhost), keine Rate-Limits.
- **D-15:** Spotify Web API (`me/player/*`) als Fallback wenn Binary-Control-Endpoint nicht erreichbar.

### Claude's Discretion
- Crash-Recovery-Strategie: Exponential Backoff vs. fester Retry, Schwellwerte für Discovery-Deaktivierung
- Sync-Gruppen: Master-Only Daemon vs. alternatives Pattern, Differentieller vs. Komplett-Neustart bei Gruppen-Änderung (CON-15)
- Loop-Prevention: Source-Marking (Spotty-NG-Pattern) vs. robusterer Mechanismus
- Volume-Grace-Period: Übernehmen (20s) vs. bessere Lösung mit librespot 0.8
- Debouncing-Strategie für Volume- und Seek-Events
- killHangingProcesses-Schutz (CON-09): Integration mit bestehendem `_killOrphanedProcesses()` Code (PHASE-5-NOTE bereits im Code)
- Enriched Event-Payload-Schema: welche Felder im JSON, Format, optionale vs. required Felder

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Projekt-Kontext
- `.planning/PROJECT.md` — Core Value, Constraints, Key Decisions (HTTP-streaming für Connect, Binary-Strategie)
- `.planning/REQUIREMENTS.md` — CON-01 bis CON-17 Anforderungsdefinitionen
- `.planning/phases/04-single-track-streaming/04-CONTEXT.md` — Transcoding-Pipeline-Architektur (D-01 updateTranscodingTable, D-04 FLAC-Default, D-06 Normalization)
- `.planning/phases/04.3-zeroconf-keymaster-auth/04.3-CONTEXT.md` — ZeroConf/Keymaster-Auth (D-01 Auto-Start, D-03 Device-Name), Credential-Speicherung
- `.planning/phases/04.4-dual-token-api-routing/04.4-CONTEXT.md` — Dual-Token-Routing (D-05 harter me/*-Guard: Connect me/player/* immer own-Token)

### Bestehender Code (wird erweitert/modifiziert)
- `Plugins/SpotOn/Plugin.pm` — `_killOrphanedProcesses()` hat PHASE-5-NOTE für Connect-PID-Ausschluss (Zeile 157). `initPlugin()` braucht Connect-Daemon-Start. `updateTranscodingTable()` muss Connect-Pipeline unterstützen.
- `Plugins/SpotOn/ProtocolHandler.pm` — `formatOverride()` muss Connect-Content-Type (`soc`) zurückgeben. `canDirectStream()` muss HTTP-URL für Connect liefern. `new()` Override für Sync-Gruppen-Proxy.
- `librespot-spoton/src/main.rs` — `Mode::Connect` Stub (Zeile 268-272). Muss implementiert werden: HTTP-Audio-Server, Spirc-Event-Loop, JSON-RPC-Dispatch, Control-Endpoints.
- `Plugins/SpotOn/Settings.pm` — Per-Player Connect-Toggle, OGG-Passthrough-Override
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Per-Player Settings UI

### Bestehender Code (wird genutzt)
- `Plugins/SpotOn/Helper.pm` — Binary-Finder, Capability-Check. Wiederverwendbar für Connect-Daemon-Spawn.
- `Plugins/SpotOn/API/Client.pm` — Zentraler HTTP-Client für Spotify Web API (Fallback-Rückkanal). Dual-Token-Routing (Phase 04.4) routet me/player/* korrekt.
- `Plugins/SpotOn/API/TokenManager.pm` — Multi-Account Token-Management. Connect-Daemons nutzen Account-spezifische Cache-Dirs.

### Technologie-Referenz
- `CLAUDE.md` §librespot — CLI-Flags für Connect: `--name`, `--device-type`, `--backend pipe`, `--cache`, `--disable-discovery`, `--device-id`, `--lms`, `--player-mac`, `--format`, `--passthrough`
- `CLAUDE.md` §Spotify Web API v1 — Player-Endpoints (me/player/*), Device-Endpoints (me/player/devices), Rate Limits
- `CLAUDE.md` §Protocol Handler Pattern — ProtocolHandler Registration, Transcoding custom-convert.conf

### Referenz-Implementierung (Spotty-NG)
- `/home/sti/spotty-ng/Spotty-Plugin/Connect.pm` — Event-Dispatch, Source-Marking, Volume-Grace-Period, Stale-API-Fallback (904 Zeilen)
- `/home/sti/spotty-ng/Spotty-Plugin/Connect/DaemonManager.pm` — Daemon-Lifecycle, Sync-Gruppen, Watchdog (337 Zeilen)
- `/home/sti/spotty-ng/Spotty-Plugin/Connect/Daemon.pm` — Process-Wrapper, Stream-Port-Capture, Crash-Backoff (321 Zeilen)
- `/home/sti/spotty-ng/Spotty-Plugin/ProtocolHandler.pm` — `formatOverride()` spc/spt-Routing, `canDirectStream()` HTTP-URL, `new()` Sync-Proxy (418 Zeilen)
- `/home/sti/spotty-ng/Spotty-Plugin/custom-convert.conf` — `spc pcm * *` Connect-Pipeline-Profile

### LMS-Referenz
- https://github.com/LMS-Community/slimserver — Slim::Control::Request (Dispatch), Slim::Player::Sync (syncname), Slim::Player::Protocols::HTTP
- https://github.com/LMS-Community/plugin-Qobuz — Qobuz ProtocolHandler (canDirectStream-Referenz für HTTP-Streaming)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Plugin.pm::_killOrphanedProcesses()` (Zeile 120-178) — Bereits PHASE-5-NOTE für Connect-PID-Ausschluss. Muss erweitert werden um Connect-Daemon-PIDs zu schützen.
- `Plugin.pm::updateTranscodingTable()` (Zeile 1186-1249) — Dynamische Transcoding-Tabelle. Muss um `soc-*` Connect-Profile erweitert werden.
- `ProtocolHandler.pm` — Grundgerüst mit formatOverride, canSeek, getSeekData. Muss für Connect-Modus erweitert werden (soc Content-Type, canDirectStream, new() Override).
- `main.rs::Mode::Connect` (Zeile 73, 113-115, 268-272) — Stub existiert, CLI-Parsing vorhanden.
- `main.rs::run_discover_once()` (Zeile 477-528) — Discovery-Pattern wiederverwendbar für Connect-Daemon-mDNS.
- `Helper.pm::get()` — Binary-Discovery, Version-Check. Wiederverwendbar für Daemon-Spawn.
- `TokenManager.pm::_fetchKeymasterToken()` — Token-Beschaffung für Connect me/player/*-Requests.

### Established Patterns
- Binary-Spawn: `_killOrphanedProcesses()` zeigt pgrep/kill-Pattern. Connect-Daemon braucht `Proc::Background` (Spotty-NG-Pattern).
- Cache-Dir: `catdir($serverPrefs->get('cachedir'), 'spoton', $activeAccountId)` — pro-Account Cache-Verzeichnis für Daemon-Credentials.
- Timer: `Slim::Utils::Timers::setTimer()` — für Watchdog, Debouncing, Grace-Periods.
- Prefs: `$prefs->client($client)->get(...)` — per-Player Preferences (activeAccount bereits genutzt).
- JSON-RPC: LMS `Slim::Control::Request::addDispatch()` — CLI-Command-Registrierung für Connect-Events.

### Integration Points
- `Plugin.pm::initPlugin()` → Connect-DaemonManager starten
- `ProtocolHandler.pm::formatOverride()` → `soc` bei aktivem Connect
- `ProtocolHandler.pm::canDirectStream()` → HTTP-URL für Connect-Einzelspieler
- `ProtocolHandler.pm::new()` → URL-Substitution für Sync-Gruppen
- `custom-convert.conf` → `soc pcm * *` und `soc ogg * *` Profile
- `custom-types.conf` → `soc` Type registrieren
- `Settings.pm` → Per-Player Connect-Toggle + OGG-Override
- `_killOrphanedProcesses()` → Connect-Daemon-PIDs ausschließen (CON-09)

</code_context>

<specifics>
## Specific Ideas

- HTTP-Control-Endpoints am Binary statt Spotify Web API für Rückkanal (LMS → Binary) — schneller, keine Rate-Limits, Spirc-konsistent
- Enriched JSON-RPC Events mit Track-Metadata direkt im Payload — spart separate API-Calls nach Track-Start
- OGG-Passthrough Auto-Detect basierend auf Player-Capabilities mit per-Player Override
- Spotty-NG als Referenz für bewährte Patterns (Source-Marking, Volume-Grace-Period, Stale-API-Fallback), aber Architektur neu gedacht (HTTP-Control statt Web-API-Rückkanal, Enriched Events)
- CON-12 in REQUIREMENTS.md sagt "FIFO-based" — muss auf "HTTP-streaming" aktualisiert werden (Entscheidung D-01 überschreibt initiales Requirement)

</specifics>

<deferred>
## Deferred Ideas

- **Per-Player OGG-Passthrough auch für Browse/Single-Track-Modus** — aktuell ist OGG-Passthrough im Single-Track-Modus rein automatisch (Binary+Player-Capability), ohne per-Player-Override. Der per-Player Override (Auto/Erzwingen/PCM) sollte für BEIDE Modi gelten (Connect UND Browse). Am besten als einheitliche per-Player-Setting-Seite im SpotOn-Setup zusammen mit dem Connect-Toggle. → Phase 6 (Player-specific Preferences, LMS-08)
- **CON-12 Requirement-Update** — "FIFO-based audio transport" → "HTTP-streaming audio transport" in REQUIREMENTS.md. Sollte bei Phase-Start aktualisiert werden.

</deferred>

---

*Phase: 5-Spotify Connect*
*Context gathered: 2026-06-01*
