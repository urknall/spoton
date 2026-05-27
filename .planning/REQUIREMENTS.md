# Requirements: SpotOn

**Defined:** 2026-05-26
**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## v1 Requirements

### Authentication

- [x] **AUTH-01**: Plugin obtains Spotify access token via OAuth 2.0 Authorization Code + PKCE browser flow through per-user Spotify Developer App
- [ ] **AUTH-02**: Access token is cached and automatically refreshed before expiry
- [ ] **AUTH-03**: Connect daemons are proactively restarted at 50-minute uptime to prevent silent token expiry
- [x] **AUTH-04**: Credential storage uses LMS Prefs with restricted plugin namespace (PKCE replaced filesystem credential files)
- [x] **AUTH-05**: Multiple Spotify accounts can be configured per LMS instance
- [x] **AUTH-06**: Account switching is available in the plugin menu

### Browse & Navigation

- [ ] **NAV-01**: Top-level menu structure: Home, Search, Library
- [ ] **NAV-02**: Home feed shows Recently Played, Made For You mixes (via category ID trick), Top Tracks
- [ ] **NAV-03**: Library shows Liked Songs, Saved Albums, Followed Artists, User Playlists as sub-items
- [ ] **NAV-04**: Search supports free-text with results categorized (Tracks, Albums, Artists, Playlists)
- [ ] **NAV-05**: Artist detail page shows Discography (Albums, Singles, Compilations)
- [ ] **NAV-06**: Album detail page shows tracklist with track number, duration, featuring artists
- [ ] **NAV-07**: Playlist detail page shows paginated tracks, description, creator
- [ ] **NAV-08**: Liked Songs are exposed unconditionally (no gating behind custom Client ID)
- [ ] **NAV-09**: Library items sortable (recently added as default)
- [ ] **NAV-10**: Endpoints removed in Dev Mode (Artist Top Tracks, Browse Categories, New Releases, Related Artists) are gracefully hidden, not errored
- [ ] **NAV-11**: Search pagination handles limit=10 per request (Dev Mode constraint)

### Audio Streaming

- [ ] **STR-01**: Single-track playback via librespot `--single-track` mode writing PCM to stdout
- [ ] **STR-02**: FLAC transcoding pipeline as default (`spt → flc` via custom-convert.conf)
- [ ] **STR-03**: PCM passthrough pipeline for capable players
- [ ] **STR-04**: MP3 transcoding pipeline as legacy fallback
- [ ] **STR-05**: OGG-Direct passthrough for players that support OGG natively
- [ ] **STR-06**: Bitrate selection (96/160/320 kbps) configurable per plugin settings
- [ ] **STR-07**: Seeking via `--start-position` parameter in transcoding pipeline
- [ ] **STR-08**: Volume normalization (Replay Gain) optional, per player configurable
- [ ] **STR-09**: Gapless playback between consecutive tracks
- [ ] **STR-10**: Hourly cleanup of orphaned librespot processes
- [ ] **STR-11**: Audio cache management (on/off, size limit configurable)

### Spotify Connect

- [ ] **CON-01**: One librespot Connect daemon per LMS player
- [ ] **CON-02**: Daemon lifecycle management (start at init, stop at shutdown, restart on crash with backoff)
- [ ] **CON-03**: Event dispatching from binary to LMS via JSON-RPC (start/stop/change/volume/pause)
- [ ] **CON-04**: Transfer playback from Spotify app to LMS player starts audio within 3 seconds
- [ ] **CON-05**: Play/Pause/Skip/Volume controllable from Spotify app
- [ ] **CON-06**: Sync-group handling: one daemon on master player, name = concatenated player names
- [ ] **CON-07**: mDNS/ZeroConf discovery for Connect receivers (optionally disableable)
- [ ] **CON-08**: Mutual exclusion between Browse-streaming and Connect sessions
- [ ] **CON-09**: Connect daemon PIDs excluded from LMS `killHangingProcesses` guard
- [ ] **CON-10**: Connect per player enable/disable in settings
- [ ] **CON-11**: Volume suppression window after Connect start (prevents volume jumps)
- [ ] **CON-12**: FIFO-based audio transport (HTTP-streaming is v2 upgrade)
- [ ] **CON-13**: Position sync via `startOffset` (never `['time', N]` in stream mode)
- [ ] **CON-14**: Sink-level rate-limiting (wall-clock speed, nanosecond-accurate)
- [ ] **CON-15**: Differential daemon restart on sync-group changes (only affected daemons)
- [ ] **CON-16**: Unique port assignment per daemon (HTTP audio + ZeroConf) from Manager pool
- [ ] **CON-17**: Progress stored in `pluginData` before `playlist play` command (race condition prevention)

### API Infrastructure

- [ ] **API-01**: Central HTTP client (`API/Client.pm`) as sole HTTP egress point
- [ ] **API-02**: Rate limiting via sliding window or adaptive throttle (max N concurrent requests)
- [ ] **API-03**: Response caching with domain-specific TTLs (60s Library, 300s Browse, 3600s Metadata)
- [ ] **API-04**: `Retry-After` header respected on 429 responses
- [ ] **API-05**: Batch API endpoints used where available
- [ ] **API-06**: All HTTP calls use `SimpleAsyncHTTP` with explicit timeouts (never blocking)

### LMS Integration

- [ ] **LMS-01**: `spotify://` URI protocol handler registered and functional
- [ ] **LMS-02**: Web-based settings UI under LMS Settings
- [ ] **LMS-03**: i18n support (EN + DE minimum) via LMS strings mechanism
- [ ] **LMS-04**: install.xml manifest with correct metadata, minVersion, repository URL
- [ ] **LMS-05**: custom-convert.conf with `spt → pcm/flc/mp3` transcoding pipelines
- [ ] **LMS-06**: Multi-architecture binaries (x86_64, aarch64, armhf, i386)
- [ ] **LMS-07**: Binary capability detection via `--check` JSON with version enforcement
- [ ] **LMS-08**: Player-specific preferences (bitrate, normalization, Connect on/off)
- [ ] **LMS-09**: Don't Stop The Music integration for auto-play after playlist end
- [ ] **LMS-10**: Custom binary support (user-provided binary override)
- [ ] **LMS-11**: Transcoding table updated per-track (not globally) to avoid race conditions

## v2 Requirements

### Write-Back Actions

- **WB-01**: Like/Save/Follow actions via unified `PUT /me/library` endpoint
- **WB-02**: Artist Radio / Track Radio via `recommendations` endpoint

### Audio Transport Upgrade

- **AT-01**: HTTP-streaming as primary Connect audio transport (replaces FIFO)
- **AT-02**: Clean seek in Connect mode via HTTP Content-Range
- **AT-03**: No white noise on reconnect (proper connection semantics)

### Extended Features

- **EXT-01**: Online Library Importer (Spotify library → LMS database for local search)
- **EXT-02**: Podcast & Audiobook support
- **EXT-03**: Browse Categories and New Releases (requires Extended Quota or workaround)
- **EXT-04**: "Fans also like" / Related Artists (requires Extended Quota)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Lossless/HiFi streaming | Blocked by PlayPlay DRM — architecturally prepared but not implementable |
| PlayPlay DRM reverse engineering | Explicit prohibition — legal + ethical (HIF-04) |
| Extended Quota Mode application | Requires 250k MAU + commercial org — not feasible for open-source plugin |
| Mobile app | LMS plugin only |
| Real-time collaborative playlists | High complexity, not core value |
| Spotify Canvas (video loops) | LMS has no video capability |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| AUTH-01 | Phase 02.1 | Complete |
| AUTH-02 | Phase 02.1 | Complete |
| AUTH-03 | Phase 02.1 | Complete |
| AUTH-04 | Phase 02.1 | Complete |
| AUTH-05 | Phase 02.1 | Complete |
| AUTH-06 | Phase 02.1 | Complete |
| NAV-01 | Phase 3 | Pending |
| NAV-02 | Phase 3 | Pending |
| NAV-03 | Phase 3 | Pending |
| NAV-04 | Phase 3 | Pending |
| NAV-05 | Phase 3 | Pending |
| NAV-06 | Phase 3 | Pending |
| NAV-07 | Phase 3 | Pending |
| NAV-08 | Phase 3 | Pending |
| NAV-09 | Phase 3 | Pending |
| NAV-10 | Phase 3 | Pending |
| NAV-11 | Phase 3 | Pending |
| STR-01 | Phase 4 | Pending |
| STR-02 | Phase 4 | Pending |
| STR-03 | Phase 4 | Pending |
| STR-04 | Phase 4 | Pending |
| STR-05 | Phase 4 | Pending |
| STR-06 | Phase 4 | Pending |
| STR-07 | Phase 4 | Pending |
| STR-08 | Phase 4 | Pending |
| STR-09 | Phase 4 | Pending |
| STR-10 | Phase 4 | Pending |
| STR-11 | Phase 4 | Pending |
| CON-01 | Phase 5 | Pending |
| CON-02 | Phase 5 | Pending |
| CON-03 | Phase 5 | Pending |
| CON-04 | Phase 5 | Pending |
| CON-05 | Phase 5 | Pending |
| CON-06 | Phase 5 | Pending |
| CON-07 | Phase 5 | Pending |
| CON-08 | Phase 5 | Pending |
| CON-09 | Phase 5 | Pending |
| CON-10 | Phase 5 | Pending |
| CON-11 | Phase 5 | Pending |
| CON-12 | Phase 5 | Pending |
| CON-13 | Phase 5 | Pending |
| CON-14 | Phase 5 | Pending |
| CON-15 | Phase 5 | Pending |
| CON-16 | Phase 5 | Pending |
| CON-17 | Phase 5 | Pending |
| API-01 | Phase 2 | Pending |
| API-02 | Phase 2 | Pending |
| API-03 | Phase 2 | Pending |
| API-04 | Phase 2 | Pending |
| API-05 | Phase 2 | Pending |
| API-06 | Phase 2 | Pending |
| LMS-01 | Phase 1 | Pending |
| LMS-02 | Phase 1 | Pending |
| LMS-03 | Phase 1 | Pending |
| LMS-04 | Phase 1 | Pending |
| LMS-05 | Phase 1 | Pending |
| LMS-06 | Phase 1 | Pending |
| LMS-07 | Phase 1 | Pending |
| LMS-08 | Phase 6 | Pending |
| LMS-09 | Phase 6 | Pending |
| LMS-10 | Phase 6 | Pending |
| LMS-11 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 62 total
- Mapped to phases: 62
- Unmapped: 0

---
*Requirements defined: 2026-05-26*
*Last updated: 2026-05-27 after Phase 02.1 gap closure — AUTH requirements reflect OAuth PKCE*
