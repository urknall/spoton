# Roadmap: SpotOn

**Project:** SpotOn — LMS Spotify Plugin
**Created:** 2026-05-26
**Granularity:** standard

## Milestones

- ✅ **v1.0 Foundation** - Phases 1-6 (shipped 2026-06-03)
- 🚧 **v1.1 Hardening & Reach** - Phases 7-10 (in progress)

## Phases

<details>
<summary>✅ v1.0 Foundation (Phases 1-6) — SHIPPED 2026-06-03</summary>

- [x] **Phase 1: Plugin Skeleton + Binary Foundation** - Plugin loads in LMS, correct manifest, binary scaffolding in place
- [x] **Phase 2: Auth + API Foundation** - Authenticated Spotify API requests work; token lifecycle managed (completed 2026-05-27)
- [x] **Phase 02.1: OAuth-PKCE Browser Auth** - Replace non-functional Keymaster/login5 auth with OAuth 2.0 PKCE browser flow (completed 2026-05-27)
- [x] **Phase 3: Browse + Navigation** - Users can navigate Home, Search, and Library via LMS menus (completed 2026-05-28)
- [x] **Phase 4: Single-Track Streaming** - Users can play any Spotify track from the Browse menus (completed 2026-05-28)
- [x] **Phase 04.1: Streaming Bug Fixes + Passthrough Binary** - Fix UAT blockers and build passthrough-decoder binary (completed 2026-05-28)
- [x] **Phase 04.2: Credentials + Made For You Fix** - Own librespot credentials + category endpoint for personal mixes (completed 2026-05-29)
- [x] **Phase 04.3: ZeroConf + Keymaster Auth** - Single auth step via Spotify app replaces PKCE browser flow for credential provisioning (completed 2026-05-29)
- [x] **Phase 04.4: Dual-Token API Routing** - Dual-flavor token routing for rate-limit distribution (completed 2026-05-29)
- [x] **Phase 5: Spotify Connect** - LMS players appear as Spotify Connect receivers; Spotify app controls playback (completed 2026-06-01)
- [x] **Phase 05.1: Connect Audio Streaming Bugfix** - Fix audio streaming pipeline: DirectStream connection, Spirc session stability, PCM relay (completed 2026-06-01)
- [x] **Phase 05.2: Connect Controls & Resume** - Fix Connect Resume, verify/fix bidirectional Volume/Pause/Resume, semi-bidirectional Skip, unidirectional Seek (completed 2026-06-01)
- [x] **Phase 05.3: Sync Groups + Connect Robustness** - Sync-group audio and Connect session-handover fix (completed 2026-06-02)
- [x] **Phase 05.4: mDNS Connect Discovery Fix** - mDNS discovery works reliably, devices connect via discovery, crash-loop protection auto-resets (completed 2026-06-02)
- [x] **Phase 6: Polish + DSTM + Settings** - Player-specific preferences, auto-play continuation, and custom binary override functional (completed 2026-06-03)

</details>

### 🚧 v1.1 Hardening & Reach (In Progress)

**Milestone Goal:** Connect-DSTM, Multi-Arch Binaries for all 6 platforms, full DE→EN code cleanup, and production deployment.

- [x] **Phase 7: DE→EN Code Cleanup** - All German comments and log strings replaced with English; codebase is language-clean (completed 2026-06-03)
- [x] **Phase 8: Multi-Arch Binary Distribution** - librespot binary available for all 8 platform targets; Helper.pm selects the correct binary automatically (completed 2026-06-03)
- [x] **Phase 9: Stream Metadata** - Songinfo shows active mode, format, and bitrate for every playing track (completed 2026-06-04)
- [x] **Phase 9.5: Prod Deployment & Monitoring** - GitHub repo public, LMS custom repo XML, plugin deployed on Pi, SpotOn monitoring replaces Spotty (completed 2026-06-04)
- [x] **Phase 10: Connect-DSTM** - Auto-play continues in Connect mode via Spirc-native autoplay when the Spotify queue is exhausted (completed 2026-06-04)

## Phase Details

### Phase 7: DE→EN Code Cleanup

**Goal**: The codebase contains no German text in code comments or log strings; every comment and log call reads in English
**Depends on**: Nothing (v1.1 start)
**Requirements**: CLEAN-01, CLEAN-02, CLEAN-03
**Success Criteria** (what must be TRUE):

  1. Running `grep -rn` for German special characters (ä, ö, ü, ß, Ä, Ö, Ü) against all Perl and Rust source files (excluding strings.txt and i18n files) returns zero matches
  2. Every `# Kommentar`-style comment block in Plugin.pm, Connect.pm, Client.pm, and Helper.pm is readable in English without ambiguity
  3. Every DEBUGLOG, INFOLOG, WARNLOG, and ERRORLOG call emits an English string — no German words in any log line visible at runtime

**Plans**: 1 plan
Plans:

- [x] 07-01-PLAN.md — Translate all German comments to English + full codebase verification

### Phase 8: Multi-Arch Binary Distribution

**Goal**: A librespot binary is available for every supported platform target; the plugin selects the correct binary at runtime without user configuration
**Depends on**: Nothing (independent of Phase 7, can run in parallel)
**Requirements**: ARCH-01, ARCH-02, ARCH-03, ARCH-04, ARCH-05, ARCH-06, ARCH-07, ARCH-08, ARCH-09, ARCH-10
**Success Criteria** (what must be TRUE):

  1. The Bin/ directory contains subdirectories for all 8 targets (x86_64-linux, aarch64-linux, armv7-linux, i386-linux, armv6-linux, x86_64-darwin, aarch64-darwin, x86_64-win64) each holding a static librespot binary
  2. Helper.pm correctly identifies all 8 platform/arch combinations and returns the matching binary path without falling back to an incorrect binary
  3. On an aarch64 Linux system (Pi 4, NAS), the plugin loads the aarch64 binary, starts the Connect daemon, and streams a track successfully
  4. All Linux binaries are musl-statically linked — no glibc dependency, confirmed by `ldd` returning "not a dynamic executable"
  5. The x86_64 Linux binary replaces the previous glibc-linked binary and passes `--check` version verification

**Plans**: 2 plans
Plans:

- [x] 08-01-PLAN.md — Cross-compile all 6 platform binaries (5 Linux musl-static + 1 Windows GNU)
- [x] 08-02-PLAN.md — Helper.pm platform detection cleanup + aarch64 live verification

### Phase 9: Stream Metadata

**Goal**: Songinfo for a playing Spotify track shows the active playback mode, stream format, and bitrate so the user can confirm what the plugin is delivering
**Depends on**: Phase 7 (cleanup complete before adding new code)
**Requirements**: META-01, META-02, META-03
**Success Criteria** (what must be TRUE):

  1. Songinfo for a track played via LMS Browse menus shows "(Spotify Browse)" in the source line
  2. Songinfo for a track playing through Spotify Connect shows "(Spotify Connect)" in the source line
  3. Songinfo shows the active stream format (e.g., "OGG", "FLAC", "MP3", "PCM") for the currently playing track
  4. When bitrate information is available, Songinfo shows it alongside the format (e.g., "320k, OGG (Spotify Connect)")

**Plans**: 1 plan
Plans:

- [x] 09-01-PLAN.md — TDD: _typeString helper + update all 4 metadata call sites

### Phase 9.5: Prod Deployment & Monitoring

**Goal**: SpotOn is installable from a public GitHub repo via LMS custom repository, deployed on the production Pi, with live monitoring replacing the old Spotty setup
**Depends on**: Phase 9 (all code changes complete before public release)
**Requirements**: DEPLOY-01, DEPLOY-02, DEPLOY-03, DEPLOY-04, DEPLOY-05
**Success Criteria** (what must be TRUE):

  1. GitHub repo stiefenm/spoton is public; .planning/, .claude/, CLAUDE.md and other internal files are excluded via .gitignore and git rm --cached
  2. install.xml exists with correct version, creator, and module paths
  3. repo.xml has correct SHA and download URL; adding the raw GitHub URL as LMS custom repo shows SpotOn in the plugin list
  4. SpotOn is installed and running on the Pi (192.168.13.5) via the LMS plugin manager
  5. SpotOn monitor is active on the Pi (cron job, daily log rotation); old Spotty monitoring and Spotty-Plugin directory cleaned up

**Plans**: 2 plans
Plans:

- [x] 09.5-01-PLAN.md — Repo preparation: exclude internal files, version bump, release zip, finalize repo.xml, make public
- [x] 09.5-02-PLAN.md — Pi deployment: install SpotOn via LMS, monitoring setup, Spotty cleanup

### Phase 10: Connect-DSTM

**Goal**: When the Spotify queue runs out during a Connect session, playback continues automatically via Spirc-native autoplay — matching the auto-play behavior already present in Browse mode
**Depends on**: Phase 8 (binary rebuild required), Phase 9
**Requirements**: DSTM-01, DSTM-02, DSTM-03, DSTM-04, DSTM-05, DSTM-06
**Success Criteria** (what must be TRUE):

  1. The binary accepts `--autoplay on/off` and reports `autoplay: true` in `--check` JSON — Spirc's `add_autoplay_resolving_when_required()` handles queue continuation natively
  2. `SessionConfig.autoplay` is set from the CLI flag before `Session::new()`, overriding Spotify user settings when explicitly configured
  3. Playback continues seamlessly in Connect mode after queue exhaustion — no gap longer than 10 seconds, no user intervention required
  4. Disabling the per-player Autoplay toggle in Settings stops Connect-DSTM for that player only; other players continue with auto-play
  5. Browse-mode DSTM continues to work without regression after the Connect-DSTM implementation

**Plans**: 3 plans
Plans:
**Wave 1**

- [x] 10-01-PLAN.md — Binary source: --autoplay flag parsing, --check capability, SessionConfig override
- [x] 10-02-PLAN.md — Perl plugin: enableAutoplay pref, daemon flag, Settings UI toggle, DSTM sync, i18n strings

**Wave 2** *(blocked on Wave 1 completion)*

- [x] 10-03-PLAN.md — Binary rebuild for all 6 platforms + end-to-end verification checkpoint

## Progress Table

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Plugin Skeleton + Binary Foundation | v1.0 | 0/? | Complete | 2026-06-03 |
| 2. Auth + API Foundation | v1.0 | 6/6 | Complete | 2026-05-27 |
| 02.1. OAuth-PKCE Browser Auth | v1.0 | 4/4 | Complete | 2026-05-27 |
| 3. Browse + Navigation | v1.0 | 3/3 | Complete | 2026-05-28 |
| 4. Single-Track Streaming | v1.0 | 2/2 | Complete | 2026-05-28 |
| 04.1. Streaming Bug Fixes + Passthrough Binary | v1.0 | 2/2 | Complete | 2026-05-28 |
| 04.2. Credentials + Made For You Fix | v1.0 | 2/2 | Complete | 2026-05-29 |
| 04.3. ZeroConf + Keymaster Auth | v1.0 | 4/4 | Complete | 2026-05-29 |
| 04.4. Dual-Token API Routing | v1.0 | 2/2 | Complete | 2026-05-29 |
| 5. Spotify Connect | v1.0 | 5/5 | Complete | 2026-06-01 |
| 05.1. Connect Audio Streaming Bugfix | v1.0 | 3/3 | Complete | 2026-06-01 |
| 05.2. Connect Controls & Resume | v1.0 | 2/2 | Complete | 2026-06-01 |
| 05.3. Player Sync Groups | v1.0 | 3/3 | Complete | 2026-06-02 |
| 05.4. mDNS Connect Discovery Fix | v1.0 | 3/3 | Complete | 2026-06-02 |
| 6. Polish + DSTM + Settings | v1.0 | 5/5 | Complete | 2026-06-03 |
| 7. DE→EN Code Cleanup | v1.1 | 1/1 | Complete   | 2026-06-03 |
| 8. Multi-Arch Binary Distribution | v1.1 | 2/2 | Complete   | 2026-06-03 |
| 9. Stream Metadata | v1.1 | 1/1 | Complete   | 2026-06-04 |
| 9.5. Prod Deployment & Monitoring | v1.1 | 2/2 | Complete   | 2026-06-04 |
| 10. Connect-DSTM | v1.1 | 3/3 | Complete   | 2026-06-04 |

## Backlog

Items discovered during UAT — not blocking current milestone.

1. **Eigene SpotOn Client-ID bei Spotify registrieren** — Aktuell nutzt bundled-Token Hergers Spotty-NG App-ID. Langfristig braucht SpotOn eine eigene registrierte App mit Extended Quota Mode.
2. **Format-Dropdown mit Nicht-OGG-Playern testen** — Auto-Modus mit B&O/Chromecast verifizieren (kein OGG-Support → Auto sollte FLAC wählen). Bisher nur mit squeezelite getestet.

---
*Roadmap created: 2026-05-26*
*Last updated: 2026-06-04 — Phase 10 plans created*
