# Phase 17: B&O Format Verification - Context

**Gathered:** 2026-06-12
**Status:** Ready for planning

<domain>
## Phase Boundary

Hardware-QA: Das Format-Dropdown wird auf einem B&O-Player via UPnPBridge verifiziert. Alle fünf Format-Modi (Auto/OGG/PCM/FLAC/MP3) werden auf korrekten Audio-Output, Metadata-Anzeige und subjektive Audioqualität getestet. B&O via UPnPBridge repräsentiert Player, die bestimmte Formate (z.B. OGG Passthrough) nicht nativ unterstützen.

</domain>

<decisions>
## Implementation Decisions

### Testmatrix (QA-01, QA-02)
- **D-01:** Testplayer ist B&O via UPnPBridge — repräsentiert non-OGG-fähige Player im LMS-Ökosystem
- **D-02:** Alle 5 Format-Modi werden getestet: Auto, OGG, PCM, FLAC, MP3
- **D-03:** Testtiefe ist vollständig: Audio-Output (ja/nein/Stille), Songinfo-Metadata (korrektes Format/Bitrate), subjektive Audioqualität (kein hörbarer Qualitätsverlust, kein Stottern), Stabilität (kein Abbruch, Seek, Track-Wechsel)
- **D-04:** squeezelite ist implizit getestet (täglicher Gebrauch) und dient als Referenz, wird aber nicht formal in die Testmatrix aufgenommen

### Auto-Mode (QA-02)
- **D-05:** Das Verhalten von Auto-Modus auf B&O/UPnPBridge ist unklar — die Verifikation wird zeigen, welches Format LMS tatsächlich wählt, wenn OGG nicht unterstützt wird
- **D-06:** Die OGG-Passthrough-Guard (`Plugin.pm:1461-1468`) entfernt `son-ogg` nur wenn das Binary kein Passthrough kann — nicht basierend auf Player-Capabilities. LMS-Transcoding-Negotiation entscheidet für den Player

### Fehler-Eskalation
- **D-07:** Kleine Fixes (Config-Änderungen, Pipeline-Tweaks, custom-convert.conf Anpassungen) werden direkt in Phase 17 gemacht
- **D-08:** Größere Architekturänderungen (z.B. Player-Capability-Detection für Format-Auto-Selection) werden als Findings dokumentiert und in einer Folge-Phase oder im Backlog behandelt

### Claude's Discretion
- Dokumentationsformat: Ob VERIFICATION.md-Tabelle, eigenständiges QA-Dokument, oder kombinierter Ansatz — Planner entscheidet
- Testprotokoll-Reihenfolge: Welche Formate zuerst getestet werden (z.B. PCM als sicherstes zuerst, oder Auto zuerst um Baseline zu etablieren)
- Ob und wie Testergebnisse in repo.xml Release Notes oder README einfließen

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Format-Pipeline-Architektur
- `Plugins/SpotOn/Plugin.pm` Zeile 1378-1501 — `updateTranscodingTable()`: Pipeline-Deletion-Pattern, OGG-Passthrough-Guard, per-player streamFormat-Logik
- `Plugins/SpotOn/ProtocolHandler.pm` Zeile 42-82 — `formatOverride()`: Input-Type-Auswahl (son/soc), streamFormat-Pref-Lesen
- `Plugins/SpotOn/custom-convert.conf` — Alle 4+1 Transcoding-Pipelines (son-pcm, son-flc, son-mp3, son-ogg, soc-pcm)

### Player-Settings
- `Plugins/SpotOn/Settings.pm` Zeile 150-161 — Per-player streamFormat Pref-Validierung (auto|ogg|pcm|flac|mp3)
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Format-Dropdown UI

### Binary-Capabilities
- `Plugins/SpotOn/Helper.pm` — `getCapability('passthrough')`: Bestimmt ob OGG-Pipeline verfügbar ist

No external specs — requirements fully captured in ROADMAP.md (QA-01, QA-02) and decisions above

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `updateTranscodingTable()` Pipeline-Snapshot-Restore: Stellt Base-Pipelines bei jedem Aufruf wieder her — erlaubt sicheres Format-Switching ohne Seiteneffekte
- `Helper->getCapability()`: Binary-Feature-Check, bereits für Passthrough-Guard im Einsatz
- Songinfo-Metadata aus Phase 9: Stream-Metadata (Format, Bitrate) in Songinfo-Menü — kann zur Verifizierung genutzt werden

### Established Patterns
- Pipeline-Deletion statt Pipeline-Selektion: `updateTranscodingTable` löscht konkurrierende Pipelines, LMS wählt aus den verbleibenden
- OGG ist der einzige Format-Modus mit Passthrough (Raw Ogg Vorbis Container) — alle anderen dekodieren zu PCM und re-encodieren
- Connect-Pipelines (soc-*) sind unabhängig von Browse-Pipelines (son-*) — Phase 17 testet nur Browse-Modus

### Integration Points
- LMS TranscodingHelper::Conversions() — Shared mutable state, von updateTranscodingTable modifiziert
- Per-player `streamFormat` Pref — Setzt den aktiven Format-Modus pro Player
- UPnPBridge Player-Capabilities — Von LMS abgefragt, bestimmt welche Output-Formate der Player akzeptiert

</code_context>

<specifics>
## Specific Ideas

- B&O via UPnPBridge als repräsentativer "eingeschränkter Player" — testet den realen Pfad für non-native-OGG-Geräte
- Songinfo-Anzeige (Phase 9) als Verifizierungs-Tool: Format/Bitrate im Songinfo-Menü prüfen, nicht nur Audio hören
- Falls Auto-Modus OGG an B&O schickt und Stille produziert: Erst untersuchen ob UPnPBridge die Player-Capabilities korrekt an LMS meldet, dann entscheiden ob SpotOn-seitig gefixt werden muss

</specifics>

<deferred>
## Deferred Ideas

- **MozartBridge:** Eigenständiges LMS-Plugin für B&O Mozart-Platform-Integration (Latenz-Kompensation, Echtzeit-Sync mit Bridge-Playern). Separates Projekt, nicht Teil von SpotOn
- **Player-Capability-Detection für Auto-Format:** Falls Auto-Modus Probleme zeigt, könnte SpotOn Player-Capabilities aktiv prüfen und OGG-Pipeline pro Player entfernen. Architekturänderung — eigene Phase falls nötig

</deferred>

---

*Phase: 17-b-o-format-verification*
*Context gathered: 2026-06-12*
