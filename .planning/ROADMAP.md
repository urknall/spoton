# Roadmap: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Created:** 2026-05-26
**Granularity:** standard

## Milestones

- ✅ **v1.0 Foundation** — Phases 1-6 (shipped 2026-06-03)
- ✅ **v1.1 Hardening & Reach** — Phases 7-12 (shipped 2026-06-06)

## Phases

<details>
<summary>✅ v1.0 Foundation (Phases 1-6) — SHIPPED 2026-06-03</summary>

- [x] **Phase 1: Plugin Skeleton + Binary Foundation** — completed 2026-05-26
- [x] **Phase 2: Auth + API Foundation** (6/6 plans) — completed 2026-05-27
- [x] **Phase 02.1: OAuth-PKCE Browser Auth** (4/4 plans) — completed 2026-05-27
- [x] **Phase 3: Browse + Navigation** (3/3 plans) — completed 2026-05-28
- [x] **Phase 4: Single-Track Streaming** (2/2 plans) — completed 2026-05-28
- [x] **Phase 04.1: Streaming Bug Fixes + Passthrough Binary** (2/2 plans) — completed 2026-05-28
- [x] **Phase 04.2: Credentials + Made For You Fix** (2/2 plans) — completed 2026-05-29
- [x] **Phase 04.3: ZeroConf + Keymaster Auth** (4/4 plans) — completed 2026-05-29
- [x] **Phase 04.4: Dual-Token API Routing** (2/2 plans) — completed 2026-05-29
- [x] **Phase 5: Spotify Connect** (5/5 plans) — completed 2026-06-01
- [x] **Phase 05.1: Connect Audio Streaming Bugfix** (3/3 plans) — completed 2026-06-01
- [x] **Phase 05.2: Connect Controls & Resume** (2/2 plans) — completed 2026-06-01
- [x] **Phase 05.3: Sync Groups + Connect Robustness** (3/3 plans) — completed 2026-06-02
- [x] **Phase 05.4: mDNS Connect Discovery Fix** (3/3 plans) — completed 2026-06-02
- [x] **Phase 6: Polish + DSTM + Settings** (5/5 plans) — completed 2026-06-03

</details>

<details>
<summary>✅ v1.1 Hardening & Reach (Phases 7-12) — SHIPPED 2026-06-06</summary>

- [x] **Phase 7: DE→EN Code Cleanup** (1/1 plan) — completed 2026-06-03
- [x] **Phase 8: Multi-Arch Binary Distribution** (2/2 plans) — completed 2026-06-03
- [x] **Phase 9: Stream Metadata** (1/1 plan) — completed 2026-06-04
- [x] **Phase 9.5: Prod Deployment & Monitoring** (2/2 plans) — completed 2026-06-04
- [x] **Phase 10: Connect-DSTM** (3/3 plans) — completed 2026-06-04
- [x] **Phase 11: Track History Metadata** (2/2 plans) — completed 2026-06-05
- [x] **Phase 12: Protocol Handler Rename** (2/2 plans) — completed 2026-06-05

</details>

## Progress Table

| Phase | Milestone | Plans | Status | Completed |
|-------|-----------|-------|--------|-----------|
| 1-6 (15 phases) | v1.0 | 50/50 | Complete | 2026-06-03 |
| 7-12 (7 phases) | v1.1 | 13/13 | Complete | 2026-06-06 |

## Backlog

Items discovered during UAT — not blocking current milestone.

1. **Connect Credential Isolation** — Connect-Daemon überschreibt `credentials.json` im Account-Dir wenn ein anderer Spotify-User per Connect verbindet → Browse zeigt fremde Inhalte. Fix: Eigenes Cache-Dir für Connect (`spoton/connect-{mac}/`), Browse-Token bleibt isoliert im Account-Dir.
2. **Eigene SpotOn Client-ID bei Spotify registrieren** — Aktuell nutzt bundled-Token ncspot's App-ID (Extended Quota bestätigt). Langfristig braucht SpotOn eine eigene registrierte App mit Extended Quota Mode.
3. **Format-Dropdown mit Nicht-OGG-Playern testen** — Auto-Modus mit B&O/Chromecast verifizieren (kein OGG-Support → Auto sollte FLAC wählen). Bisher nur mit squeezelite getestet.
4. **Connect-Mode Lautstärke-Diskrepanz** — Bei gleichem %-Setting ist Connect deutlich lauter als Browse. Ursache: librespot nutzt eigene Volume-Kurve (Spirc-Protokoll, 0–65535 logarithmisch), während Browse die LMS/squeezelite-Kurve verwendet. Unabhängig von Normalisation-Settings. Mögliche Ansätze: `--volume-ctrl` Flag, Volume-Scaling im PCM-Relay, oder pragmatisch akzeptieren.
5. **~~Online-Musiksammlung (Importer.pm / OnlineLibraryBase)~~** — Evaluiert und bewusst abgelehnt. Spotty-NG importiert Spotify-Playlists/Alben in die LMS-Bibliothek via `Slim::Plugin::OnlineLibraryBase`. Qobuz, TIDAL und Deezer machen das auch. Für SpotOn abgelehnt wegen: (a) API-Quota im Dev Mode macht Library-Scan extrem teuer (keine Batch-Endpoints, jeder Track einzeln), (b) Browse > Library deckt den Use Case on-demand ab, (c) hohe Wartungslast (~300 Zeilen Importer-Code) für fraglichen Mehrwert, (d) Sync-Drift (lokale DB immer hinter Live-State). Kann bei eigener App mit Extended Quota neu evaluiert werden.

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-06-06 — v1.1 milestone archived*
