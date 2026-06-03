---
phase: 06-polish-release-readiness
plan: 05
subsystem: distribution
tags: [install.xml, repo.xml, lms-repository, security-review, distribution]

requires:
  - phase: 06-04
    provides: "Setup Guide, Credits, full i18n (11 languages) in Settings"
  - phase: 06-01
    provides: "Client-ID consolidation to single constant in Client.pm"
  - phase: 06-02
    provides: "DSTM handler and per-player bitrate override"
  - phase: 06-03
    provides: "Custom binary support, per-player format dropdown"

provides:
  - "install.xml bumped to version 1.0.0"
  - "repo.xml LMS Custom Repository descriptor template at repository root"
  - "ASVS L1 security review across all user-input-handling modules"
  - "Task 2 UAT checkpoint pending human verification"

affects:
  - phase-6.1-binary-distribution

tech-stack:
  added: []
  patterns:
    - "repo.xml: PLACEHOLDER_SHA1 / PLACEHOLDER_URL pattern for deferred binary builds"

key-files:
  created:
    - repo.xml
  modified:
    - Plugins/SpotOn/install.xml

key-decisions:
  - "PLACEHOLDER values in repo.xml are intentional — SHA1 and download URL require Phase 6.1 binary builds"
  - "repo.xml version 1.0.0 matches install.xml exactly (link integrity requirement)"

patterns-established:
  - "repo.xml + install.xml version attribute must always match"

requirements-completed:
  - LMS-06
  - LMS-03

duration: 15min
completed: 2026-06-03
---

# Phase 6 Plan 05: Distribution + Security Review Summary

**install.xml bumped to 1.0.0, repo.xml LMS Custom Repository template created, ASVS L1 security review completed with no HIGH/CRITICAL findings**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-06-03
- **Completed:** 2026-06-03
- **Tasks:** 1 of 2 (Task 2 is a checkpoint awaiting human UAT)
- **Files modified:** 2

## Accomplishments

- `install.xml` version 0.1.0 → 1.0.0 (matches repo.xml)
- `repo.xml` created at repository root following LMS Custom Repository specification (Pattern 7 from RESEARCH.md): `<extensions>/<details>/<plugins>/<plugin>` structure with EN+DE titles/descriptions, minTarget=8.0, category=musicservices, placeholder SHA1 and URL for Phase 6.1 finalization
- Full ASVS L1 security review of all user-input-handling modules: no HIGH or CRITICAL findings. Medium findings (noted below) are accepted by design.

## Task Commits

1. **Task 1: install.xml + repo.xml template + Security Review** - `4ce5982` (chore)
2. **Task 2: Phase 6 UAT** - PENDING human verification (checkpoint:human-verify)

## Files Created/Modified

- `Plugins/SpotOn/install.xml` — version bumped from 0.1.0 to 1.0.0
- `repo.xml` — new LMS Custom Repository descriptor template

## Security Review Findings

### Settings.pm
- **Bitrate validation** (`%valid_bitrates` map, lines 64-66): PASS — only 96/160/320 accepted; invalid values default to 320.
- **clientId sanitization** (`s/[^a-zA-Z0-9]//g; substr($id, 0, 32)`, lines 80-82): PASS — strips all non-alphanumeric chars, caps at 32 chars. Prevents shell metachar injection into `--client-id` flag.
- **removeAccount validation** (`/\A[0-9a-f]{8}\z/` + exists check, line 109): PASS — strict 8-char hex guard + existence verification before acting. Prevents path traversal via `_cacheDir`.
- **switchAccount validation** (exists check only, line 137): PASS — validates against known accounts before setting activeAccount.
- **bitrateOverride** (`/^(?:96|160|320)$/`, line 167): PASS — strict numeric whitelist; empty string (= use global) is also permitted.
- **streamFormat** (`/^(?:auto|ogg|pcm|flac|mp3)$/`, line 160): PASS — strict enum whitelist; invalid values default to 'auto'.
- **connectOggOverride** (`/^(?:auto|ogg|pcm)$/`, line 153): PASS — strict enum whitelist; invalid values default to 'auto'.
- **autoSetupAccount / _autoSetupAccount**: accountId is derived as `substr(md5_hex($username), 0, 8)` — 8-char hex string, not user-controlled. No path traversal risk.

### Helper.pm
- **helperCheck shell quoting** (`$safe =~ s/'/'\\''/g`, line 67): PASS — single-quote escaping for shell-safe execution. User-controlled binary path is sandboxed.
- **_findBin candidate list** (lines 119-149): PASS — candidates are hardcoded names ('spoton', 'spoton-x86_64', 'spoton-custom'). Only `findbin()` + `-f` + `-x` validated paths reach `helperCheck`. No user-controlled input enters the candidate list.
- **`--check` output parsing** (regex `/^ok spoton v([\d\.]+)/i`): PASS — strict prefix match; only version string extracted.

### Plugin.pm (updateTranscodingTable)
- **Bitrate injection** (lines 1221-1228): PASS — `$bitrate` is either the validated pref value (96/160/320) or the bitrateOverride already validated by `/^(?:96|160|320)$/`. The regex substitution `s/--bitrate \d+/--bitrate $bitrate/` receives only a validated digit string. No user input reaches substitution unvalidated.
- **cacheDir injection** (`s/-c "[^"]*"/-c "$cacheDir"/g`): MEDIUM — `$cacheDir` is built from `catdir($serverPrefs->get('cachedir'), 'spoton', $activeAccountId)`. The `$activeAccountId` originates from `$prefs->get('activeAccount')`, which is set either by `md5_hex($username)` substring (8-char hex, controlled) or `switchAccount` validation (existence-checked). No unvalidated user input reaches the path. Accepted by design.
- **helperName injection** (`s/\[spoton[^\]]*\]/[$helperName]/g`): PASS — `$helperName` is `basename($helper)` where `$helper` passed `helperCheck`. Only binaries that output the expected `ok spoton vX.Y.Z` response are accepted.

### API/Client.pm
- **URL construction** (lines 454-459): PASS — all query parameter values pass through `uri_escape()`. Underscore-prefixed params (`_accountId`, `_noCache`, etc.) are explicitly excluded from URL construction.
- **Authorization header**: PASS — token value only goes into the `Authorization` header, never logged (T-02-10 enforced at lines 479, 562).
- **Retry-After cap** (line 529): PASS — cap at 300s prevents self-DoS from malicious server response.
- **accountId in cache keys**: PASS — accountId is MD5-derived 8-char hex (not user-controllable beyond the hash scope).

### API/TokenManager.pm
- **Credential file paths** (line 364): PASS — `$cacheDir = catdir($serverPrefs->get('cachedir'), 'spoton', $accountId)` where `$accountId` is always 8-char hex from `md5_hex()`. No path traversal vector.
- **Shell quoting for --get-token** (lines 367-368): PASS — `$helper` and `$cacheDir` both get `s/'/'\\''/g` escaping.
- **clientId in shell command** (lines 373-388): PASS — bundled ID is a constant; own ID from prefs passes `[^a-zA-Z0-9]` stripping in Settings.pm before storage. Safe to use in single-quoted shell argument.
- **Token logging** (line 390, _cacheToken): PASS — T-04.3-06 honored; only accountId, flavor, and TTL logged. Token value never appears in logs.
- **chmod 0700 on account dir** (line 326): PASS — credential directories get restricted permissions.

### DontStopTheMusic.pm
- **Seed data source** (`getMixableProperties($client, 5)`): PASS — LMS-internal call, not user-controlled input. Returns LMS Track objects from the current queue.
- **Spotify URI extraction** (`/track:([a-z0-9]+)/i`, lines 71, 195, 247): PASS — strict regex extracts only alphanumeric track IDs from URIs. Malformed or injected URIs produce no match and are silently skipped.
- **Search query construction** (`sprintf('%s artist:"%s"', $title, $artist)`): LOW — artist/title values originate from LMS Track metadata (local DB or Spotify API response), not direct user form input. The values pass through Spotify's own search syntax. Accepted by design (these are not shell commands, just Spotify search API parameters sent via uri_escape in Client.pm).
- **seed_tracks/seed_artists caps** (lines 109-114): PASS — explicitly capped at 5 each before Spotify API call.

### Overall Security Assessment

**No HIGH or CRITICAL findings.** All primary injection vectors (shell execution, path traversal, bitrate/format parameter injection) are properly mitigated. The MEDIUM finding in Plugin.pm (cacheDir injection) is accepted by design — the path components are derived from MD5-hashed values, not raw user input.

ASVS L1 compliance achieved for all modules in scope.

## Decisions Made

- `repo.xml` uses PLACEHOLDER values for SHA1 and URL — finalization requires Phase 6.1 binary builds. This is the correct approach as documented in the plan's `deferred_to_phase_6_1` section.
- The `<plugin>` element's `target="unix|mac|windows"` covers all LMS-supported platforms, even though multi-arch binaries are deferred.

## Deviations from Plan

None — plan executed exactly as written. Task 2 is a checkpoint:human-verify (not auto-executable).

## Known Stubs

None — `repo.xml` PLACEHOLDER values are intentional and documented (not stubs that prevent functionality; they await Phase 6.1 binary builds and are explicitly labelled).

## Threat Flags

| Flag | File | Description |
|------|------|-------------|
| (none) | — | No new security surface introduced beyond static XML files |

## Task 2: UAT Checkpoint (Pending)

Task 2 is a `checkpoint:human-verify` requiring the user to verify all Phase 6 ROADMAP Success Criteria (SC-1 through SC-12, with SC-10 explicitly deferred to Phase 6.1):

- SC-1: Per-Player Bitrate Override working
- SC-2: DSTM auto-play after playlist end
- SC-3: Custom Binary support
- SC-4: Per-Player Settings page with format dropdown
- SC-5: Transcoding fallback for FLAC/MP3
- SC-6: Normalization flag in Connect daemon
- SC-7: Client-ID in exactly one location
- SC-8: Setup Guide visible for new users
- SC-9: i18n in all 11 languages
- SC-10: DEFERRED TO PHASE 6.1 (multi-arch binaries)
- SC-11: Security review complete (done in Task 1)
- SC-12: repo.xml template at repository root (done in Task 1)

See PLAN.md Task 2 for detailed verification steps for each success criterion.

## Self-Check: PASSED

- `Plugins/SpotOn/install.xml` contains `<version>1.0.0</version>`: FOUND
- `repo.xml` at repository root: FOUND
- `repo.xml` has `version="1.0.0"`, `minTarget="8.0"`, `category="musicservices"`, `PLACEHOLDER_SHA1_PHASE_6_1`: FOUND
- Task 1 commit `4ce5982` in git log: FOUND

## Next Phase Readiness

- Phase 6.1 (binary distribution): `repo.xml` is ready for SHA1 + URL finalization after cross-compilation builds complete
- Plugin installable via LMS custom repository once Phase 6.1 populates the placeholder values
- All Phase 6 code features complete and security-reviewed

---
*Phase: 06-polish-release-readiness*
*Completed: 2026-06-03*
