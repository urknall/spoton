# Project Research Summary

**Project:** SpotOn v1.3 — Polish & Publish
**Domain:** UX polish + distribution readiness for existing LMS Spotify plugin
**Researched:** 2026-06-06
**Confidence:** HIGH (all critical claims verified against official sources or codebase)

## Executive Summary

v1.3 is a focused polish milestone that adds no new architectural components. Every feature is a surgical modification to an existing subsystem. The two highest-risk discoveries are fully resolved by research:

1. **Like Button endpoint changed:** `PUT /me/tracks` was removed Feb 2026 — must use `PUT /me/library` with `{"uris": ["spotify:track:ID"]}` body. Silent 403 otherwise.
2. **Connect volume has two root causes:** The 20-second `VOLUME_GRACE_PERIOD` in Connect.pm prevents sync, and librespot's default `--volume-ctrl log` curve compounds with squeezelite's linear curve.

**Extended Quota is blocked:** Spotify requires 250k MAU + legally registered business since May 2025. Phase scoped to documentation only.

## Key Findings

### Stack Additions

- **macOS CI:** `macos-15-intel` + `macos-latest` runners, `lipo -create` for Universal Binary. Ad-hoc codesign sufficient — LMS plugin downloader doesn't set quarantine xattr.
- **Perl CI:** `shogo82148/actions-setup-perl@v1` + `prove -lv t/` on Perl 5.36 + 5.38. Zero CPAN deps needed.
- **LMS Community Repo:** Single PR adding repo.xml URL to `include.json` in `LMS-Community/lms-plugin-repository`. SHA1 format (not SHA256). Submit early — human reviewer merge required.
- **Extended Quota:** Blocked for individuals. Register own Spotify Dev Mode app to reduce legal risk from bundled ncspot Client-ID.

### Feature Table Stakes

| Feature | Priority | Complexity | Risk |
|---------|----------|------------|------|
| Connect Credential Isolation | P1 — must | Low | Low |
| macOS Universal Binary | P1 — must | Medium | Medium (CI) |
| LMS Community Repo Submission | P1 — must | Low | Low (process) |
| Like Button (Save to Library) | P2 — should | Medium | Medium (API change) |
| Connect Volume Fix | P2 — should | Low | Low |
| Repo Maintenance (CI, Templates) | P2 — should | Low | Low |
| Format Dropdown Verification | P3 — verify | Low | Low (hardware test) |
| Extended Quota Documentation | P3 — docs | Low | None |

### Architecture — Components Changed

| Component | Change | Scope |
|-----------|--------|-------|
| `API/Client.pm` | saveTracks, removeTracks, checkTracks methods | Additive |
| `Plugin.pm` | Like context item in _trackItem; handler | Additive |
| `Connect.pm` | VOLUME_GRACE_PERIOD 20s → 2-3s | Modification |
| `Connect/Daemon.pm` | cacheDir fix (`connect-{mac}/`); `--volume-ctrl linear` | Modification |
| `Helper.pm` | ISMAC block for Bin/darwin/ path | Additive |
| `strings.txt` | Like button labels (11 languages) | Additive |
| `.github/` | perl-tests.yml, issue templates, CONTRIBUTING.md | New |
| `Bin/darwin/` | Universal Binary directory | New |

### Critical Pitfalls

| ID | Pitfall | Prevention | Phase |
|----|---------|------------|-------|
| P-47 | Like endpoint `PUT /me/tracks` removed Feb 2026 | Use `PUT /me/library` with URI body | Like Button |
| P-48 | Missing `user-library-modify` scope in cached token | Bump cacheSchemaVersion on upgrade | Like Button |
| P-49 | Connect daemon overwrites Browse credentials | Separate `--cache` dir: `spoton/connect-{mac}/` | Credential Isolation |
| P-50 | Volume: librespot log curve + squeezelite linear | `--volume-ctrl linear` + reduce grace period | Volume Fix |
| P-52 | macOS Gatekeeper blocks unsigned binary (Sequoia) | Document `xattr -d` workaround; ad-hoc codesign | macOS Binary |
| P-56 | Extended Quota requires 250k MAU + org | Documentation only, no application | Quota Docs |

## Suggested Phase Structure

| # | Phase | Risk | Depends On |
|---|-------|------|------------|
| 1 | CI Infrastructure + Repo Maintenance | Low | — |
| 2 | Connect Credential Isolation | Low | — |
| 3 | Connect Volume Fix | Low | — |
| 4 | Like Button | Medium | — |
| 5 | macOS Universal Binary | Medium | Phase 1 (CI) |
| 6 | Format Dropdown B&O Verification | Low | Hardware |
| 7 | Extended Quota Documentation | None | — |
| 8 | LMS Community Repo Submission | Low | Phases 2, 5 |

## Research Flags

**Needs deeper investigation during phase planning:**
- Volume Fix: Option A (reduce grace period) vs Option B (post-start API poll) — test empirically
- Like Button: `PUT /me/library` behavior with duplicate saves (expected: idempotent)
- Credential Isolation: TokenManager.pm DISCOVER_DIR concurrent-ZeroConf edge case

**Standard patterns (skip research):**
- CI setup, Repo Maintenance, macOS Binary, LMS Repo Submission

## Gaps

- Volume fix adequacy — test Option A first, Option B documented as fallback
- macOS build verification — architecturally correct but untested
- B&O UPnP format behavior — requires hardware test
- LMS Community Repo acceptance — social question (competing with Spotty), not technical

---
*Research completed: 2026-06-06*
*Ready for requirements: yes*
