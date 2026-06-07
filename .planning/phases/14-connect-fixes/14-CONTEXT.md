# Phase 14: Connect Fixes - Context

**Gathered:** 2026-06-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Connect-Sessions starten mit korrekter Lautstärke und jeder Player isoliert seine Spotify-Credentials von anderen Playern und anderen Benutzern. Drei Fixes: Credential-Isolation per Player-MAC, Volume-Kurvenkorrektur, Grace-Period-Reduktion.

</domain>

<decisions>
## Implementation Decisions

### Cache-Dir Isolation (CON-01)
- **D-01:** Connect daemon bekommt eigenen Cache-Pfad: `spoton/connect-{mac}/` (MAC ohne Doppelpunkte, wie `Daemon.pm::$self->id` bereits liefert)
- **D-02:** Browse-Token-Pfad `spoton/{accountId}/` bleibt unverändert — TokenManager.pm wird NICHT angefasst
- **D-03:** Die Trennung funktioniert, weil der Connect-Daemon nur Spirc-Credentials braucht (für das Spotify Connect Protokoll), keine API-Tokens. API-Tokens holt TokenManager weiterhin aus dem Account-Dir
- **D-04:** Hintergrund: Wenn ein anderer Spotify-User per Connect verbindet (z.B. Handy der Partnerin), schreibt librespot dessen `credentials.json` ins Cache-Dir. Mit geteiltem Dir überschreibt das die Browse-Credentials → Browse zeigt fremde Inhalte. Mit getrenntem Dir ist Browse immer der konfigurierte Account
- **D-05:** Einmaliger Reconnect aller Player nach Deploy nötig (neue Cache-Dirs, alte Connect-Credentials nicht migriert)
- **D-06:** Änderung ist ~15 Zeilen in `Daemon.pm` (Cache-Pfad-Berechnung in `start()`)

### Volume bei Session-Start (CON-02)
- **D-07:** Daemon startet mit `--initial-volume` = aktuelles LMS-Player-Volume (`$client->volume`). LMS ist die Autoritätsquelle beim Session-Start
- **D-08:** Daemon startet mit `--volume-ctrl linear`. Behebt P-50: librespot-Default ist `log` (logarithmisch), squeezelite nutzt linear → Volume-Werte stimmen nicht überein

### Grace Period Reduktion (CON-03)
- **D-09:** `VOLUME_GRACE_PERIOD` von 20s auf 3s reduzieren. Matches CON-03 Requirement (Volume-Sync within 3 seconds)
- **D-10:** Die Kombination von `--initial-volume` (D-07) + `--volume-ctrl linear` (D-08) eliminiert den Haupt-Anlass für die lange Grace Period: Der Daemon startet jetzt mit dem richtigen Volume-Wert statt einem internen Default

### Claude's Discretion
- Echo-Unterdrückungsstrategie bei reduzierter Grace Period: Ob 3s allein reicht oder ein zusätzlicher Event-Delta-Filter nötig ist (Volume-Events ignorieren wenn Δ < 2 zum aktuellen LMS-Volume). Researcher soll die Mechanismen untersuchen und die optimale Strategie bestimmen
- Genaue Platzierung der `--initial-volume` und `--volume-ctrl` Args in der `@helperArgs`-Liste
- Ob `Plugins::SpotOn::Helper->getCapability()` für die neuen Flags geprüft werden muss (binary compatibility)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Connect-Architektur
- `Plugins/SpotOn/Connect/Daemon.pm` — Cache-Dir-Berechnung (Zeile 88-93) + `@helperArgs` (Zeile 103-127): HIER sind die Änderungen
- `Plugins/SpotOn/Connect/DaemonManager.pm` — Daemon-Lifecycle, Watchdog, Sync-Group-Handling: Context für wann/wie Daemons gestartet werden
- `Plugins/SpotOn/Connect.pm` — `VOLUME_GRACE_PERIOD` (Zeile 27) + Volume-Handler (Zeile 513-538): Grace-Period-Fix

### Token-Architektur (READ-ONLY, nicht ändern)
- `Plugins/SpotOn/API/TokenManager.pm` — Browse-Token-Pfad `spoton/{accountId}/`, ZeroConf Discovery: Verstehen warum die Pfade getrennt sein müssen, aber NICHT modifizieren

### Research & Pitfalls
- `.planning/research/SUMMARY.md` — P-49 (Credential overwrite), P-50 (Volume curve mismatch): Problemanalyse und Lösungsansätze
- `.planning/research/PITFALLS.md` — Vollständige Pitfall-Liste mit Prevention-Strategien

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Daemon.pm::$self->id` — MAC ohne Doppelpunkte, bereits verfügbar als Accessor. Direkt für Cache-Dir-Namen nutzbar (`connect-$self->id`)
- `$client->volume` — LMS-Player-Volume (0-100), direkt für `--initial-volume` nutzbar
- `Plugins::SpotOn::Helper->getCapability()` — Binary-Capability-Check, nutzen für `--volume-ctrl` und `--initial-volume` Feature-Gates

### Established Patterns
- `@helperArgs` in `Daemon.pm::start()` — alle Binary-Flags werden hier gesammelt. Neues Flag einfach per `push @helperArgs` hinzufügen
- Per-player Prefs via `$prefs->client($client)->get(...)` — falls Volume-Ctrl per Player konfigurierbar sein soll (aber wahrscheinlich global reicht)
- Source-Marking (`$request->source(__PACKAGE__)`) für Loop-Prevention — bestehendes Muster, nicht anfassen

### Integration Points
- `Daemon.pm::start()` — Einziger Änderungspunkt für Cache-Dir und neue Flags
- `Connect.pm` Zeile 27 — Einziger Änderungspunkt für Grace Period Konstante
- `Connect.pm::_connectEvent` Volume-Handler (Zeile 513-538) — Falls Event-Delta-Filter nötig, hier ergänzen

</code_context>

<specifics>
## Specific Ideas

- MAC-Format im Dir-Namen: Ohne Doppelpunkte, wie `Daemon.pm::$self->id` liefert (z.B. `connect-aabbccddeeff/`)
- Keine Migration alter Connect-Credentials — Benutzer müssen einmalig reconnecten nach Plugin-Update
- LMS ist Autoritätsquelle für Volume bei Session-Start, nicht Spotify

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 14-connect-fixes*
*Context gathered: 2026-06-07*
