# Requirements: SpotOn

**Defined:** 2026-06-06
**Core Value:** Reliable Spotify playback and Connect integration on LMS — Browse, stream, and control via Spotify app, without 429 bursts, zombie daemons, or audio glitches.

## v1.3 Requirements

Requirements for milestone v1.3 — Polish & Publish.

### Connect

- [x] **CON-01**: Connect daemon uses separate cache directory per player MAC (`spoton/connect-{mac}/`), preventing credential overwrite when different Spotify users connect
- [x] **CON-02**: Connect volume matches LMS player volume at session start (no initial mismatch from hardcoded default)
- [x] **CON-03**: Connect volume changes sync within 3 seconds of user action (reduced from 20s grace period)

### Library

- [x] **LIB-01**: User can save currently playing or browsed track to Liked Songs via context menu action
- [x] **LIB-02**: User can remove track from Liked Songs via context menu action (Unlike)
- [x] **LIB-03**: Track context menu shows current liked state — displays "Like" or "Unlike" based on library check
- [x] **LIB-04**: Liked state check uses `GET /me/library/contains` without adding noticeable delay to menu rendering
- [x] **LIB-05**: Plugin requests `user-library-modify` and `user-library-read` scopes; upgrade triggers one-time re-auth via cacheSchemaVersion bump

### Platform

- [x] **PLT-01**: macOS Universal Binary available covering Intel x86_64 and Apple Silicon aarch64 via lipo
- [x] **PLT-02**: macOS binary works with LMS plugin manager installation (ad-hoc code signing, no quarantine xattr)
- [x] **PLT-03**: Setup guide documents Gatekeeper workaround (`xattr -d`) for manual binary downloads

### Repository

- [x] **REPO-01**: GitHub Actions CI runs full test suite (`prove t/`) on push to main and on pull requests
- [x] **REPO-02**: CI tests against Perl 5.36 and 5.38 (spanning LMS 8.x and 9.x)
- [x] **REPO-03**: Bug Report issue template available with structured fields
- [x] **REPO-04**: Feature Request issue template available with structured fields
- [x] **REPO-05**: CONTRIBUTING.md documents development setup, test running, and PR guidelines

### QA

- [ ] **QA-01**: Format dropdown verified functional with B&O player via UPnPBridge
- [ ] **QA-02**: All 5 format modes (Auto/OGG/PCM/FLAC/MP3) produce correct audio output on verified players

## Future Requirements

Deferred to future milestone. Tracked but not in current roadmap.

### Distribution

- **DIST-01**: SpotOn listed in LMS Community Plugin Repository (include.json PR)
- **DIST-02**: Extended Quota Client-ID registered with Spotify (blocked: requires 250k MAU + org)
- **DIST-03**: Code switched to own Client-ID after Spotify approval

### Platform

- **PLT-04**: Apple Developer ID signed and notarized macOS binary (requires $99/year membership)

### Library

- **LIB-06**: Like/Unlike action available from Connect now-playing context

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Lossless/HiFi streaming | Blocked by PlayPlay DRM, architecturally prepared but not implementable |
| PlayPlay DRM reverse engineering | Explicit prohibition, legal + ethical |
| Mobile app | LMS plugin only |
| Online-Musiksammlung (Importer.pm) | API-Quota im Dev Mode zu teuer |
| Extended Quota Application | Spotify requires 250k MAU + legally registered business since May 2025 |
| LMS Community Repo Submission | Deferred to future milestone — ship stable v1.3 first |
| Client-ID Code-Umstellung | Erst nach Genehmigung durch Spotify |
| Changelog Automation | Nice-to-have, nicht kritisch für v1.3 |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| CON-01 | Phase 14 | Complete |
| CON-02 | Phase 14 | Complete |
| CON-03 | Phase 14 | Complete |
| LIB-01 | Phase 15 | Complete |
| LIB-02 | Phase 15 | Complete |
| LIB-03 | Phase 15 | Complete |
| LIB-04 | Phase 15 | Complete |
| LIB-05 | Phase 15 | Complete |
| PLT-01 | Phase 16 | Complete |
| PLT-02 | Phase 16 | Complete |
| PLT-03 | Phase 16 | Complete |
| REPO-01 | Phase 13 | Complete |
| REPO-02 | Phase 13 | Complete |
| REPO-03 | Phase 13 | Complete |
| REPO-04 | Phase 13 | Complete |
| REPO-05 | Phase 13 | Complete |
| QA-01 | Phase 17 | Pending |
| QA-02 | Phase 17 | Pending |

**Coverage:**
- v1.3 requirements: 18 total
- Mapped to phases: 18
- Unmapped: 0 ✓

---
*Requirements defined: 2026-06-06*
*Last updated: 2026-06-06 — traceability filled in after roadmap creation*
