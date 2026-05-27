---
phase: 01-plugin-skeleton-binary-foundation
plan: "01"
subsystem: infra
tags: [lms, perl, plugin-skeleton, opml, protocol-handler, transcoding, i18n]

# Dependency graph
requires: []
provides:
  - LMS plugin manifest (install.xml) with correct UUID, minVersion=8.0, maxVersion=*
  - spotify:// protocol handler registered via Plugins::SpotOn::ProtocolHandler
  - son audio format declared in custom-types.conf
  - Four transcoding pipelines: son->pcm, son->flc, son->mp3, son->ogg with [spoton] binary reference
  - i18n strings for EN and DE (9 PLUGIN_SPOTON_* keys + SON format key)
  - Plugin.pm OPMLBased entry point with handleFeed stub and binary-missing fallback
affects:
  - 01-02 (Helper.pm binary discovery uses Plugin.pm _pluginDataFor)
  - 01-03 (Settings.pm imports Plugin.pm prefs namespace)
  - all subsequent phases (plugin must load for any feature to work)

# Tech tracking
tech-stack:
  added:
    - Slim::Plugin::OPMLBased (plugin base class)
    - Slim::Formats::RemoteStream (protocol handler base class)
    - Slim::Player::ProtocolHandlers (handler registry)
    - Slim::Utils::Prefs (preferences: plugin.spoton namespace)
    - Slim::Utils::Log (log category: plugin.spoton)
    - Slim::Utils::Versions (seek capability check)
  patterns:
    - LMS plugin entry via OPMLBased with handleFeed callback
    - spotify:// URI routing via ProtocolHandler registration
    - contentType=son + canDirectStream=0 forces LMS transcoding pipeline
    - Transcoding guard (main::TRANSCODING check) at initPlugin start
    - WEBUI-conditional settings init pattern
    - i18n via strings.txt Tab-delimited format with EN/DE per key

key-files:
  created:
    - Plugins/SpotOn/Plugin.pm
    - Plugins/SpotOn/ProtocolHandler.pm
    - Plugins/SpotOn/install.xml
    - Plugins/SpotOn/strings.txt
    - Plugins/SpotOn/custom-types.conf
    - Plugins/SpotOn/custom-convert.conf
  modified: []

key-decisions:
  - "handleFeed implemented directly in Plugin.pm (no OPML.pm in Phase 1) — Phase 3 will add dedicated OPML.pm"
  - "canDirectStream=0 hardcoded — critical for forcing transcoding pipeline, never changes"
  - "son ogg pipeline uses --passthrough for OGG-Direct on capable players (D-09)"
  - "prefs init with only bitrate and binary keys — additional prefs added in later phases as needed"
  - "perl -c verification with stub modules required since LMS modules depend on runtime environment (main::SCANNER, main::PERFMON constants not available outside LMS)"

patterns-established:
  - "LMS plugin package namespace: Plugins::SpotOn::* (never Slim::Plugin::SpotOn::*)"
  - "Prefs namespace: plugin.spoton (matches log category)"
  - "Log category: plugin.spoton with defaultLevel WARN"
  - "Protocol handler: spotify:// -> Plugins::SpotOn::ProtocolHandler via registerHandler"
  - "Audio format identifier: son (not spt, not spotify)"
  - "Binary placeholder in convert.conf: [spoton] (LMS resolves via findbin)"
  - "OPML status items use type=textarea (not type=text) to prevent navigation"

requirements-completed:
  - LMS-01
  - LMS-03
  - LMS-04
  - LMS-05

# Metrics
duration: 15min
completed: 2026-05-27
---

# Phase 01 Plan 01: Plugin Skeleton Summary

**LMS-loadable SpotOn plugin skeleton with OPMLBased entry point, spotify:// protocol handler (son/canDirectStream=0), four transcoding pipelines referencing [spoton] binary, and full EN+DE i18n**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-05-27T06:45:00Z
- **Completed:** 2026-05-27T07:00:00Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments

- Complete LMS plugin manifest (install.xml) with fresh UUID 7fdb8daa, minVersion=8.0, maxVersion=* (P-35 compliance)
- OPMLBased Plugin.pm with prefs init (bitrate/binary), Helper/Settings lazy-require, protocol handler registration, and handleFeed with binary-missing fallback
- ProtocolHandler.pm enforcing transcoding pipeline via contentType=son + canDirectStream=0, with seek support for LMS 7.9.1+
- Four transcoding pipelines covering all output formats (PCM/FLAC/MP3/OGG) with [spoton] binary reference; OGG pipeline uses --passthrough for direct Vorbis passthrough
- 9 PLUGIN_SPOTON_* i18n keys with EN+DE translations, SON format key EN-only

## Task Commits

1. **Task 1: install.xml + strings.txt + custom-types.conf + custom-convert.conf** - `6ac2d42` (chore)
2. **Task 2: Plugin.pm + ProtocolHandler.pm** - `3774741` (feat)

**Plan metadata:** (see final commit below)

## Files Created/Modified

- `Plugins/SpotOn/Plugin.pm` - OPMLBased plugin root: prefs init, Handler registration, handleFeed stub, Helper/Settings init
- `Plugins/SpotOn/ProtocolHandler.pm` - spotify:// handler: son contentType, canDirectStream=0, seek support
- `Plugins/SpotOn/install.xml` - Plugin manifest: UUID 7fdb8daa, minVersion=8.0, maxVersion=*, category=musicservices
- `Plugins/SpotOn/strings.txt` - i18n: 9 PLUGIN_SPOTON_* keys EN+DE, SON format key EN
- `Plugins/SpotOn/custom-types.conf` - Audio format: son / audio/x-sb-spoton / audio
- `Plugins/SpotOn/custom-convert.conf` - 4 transcoding pipelines: son->pcm/flc/mp3/ogg, all referencing [spoton]

## Decisions Made

- `handleFeed` implemented directly in `Plugin.pm` rather than in a separate `OPML.pm` — Phase 1 only needs a stub, Phase 3 will add the full OPML tree module
- `perl -c` verification with stub LMS modules was required; the standard LMS modules (Slim::Utils::Log etc.) depend on `main::SCANNER` and `main::PERFMON` constants that only exist inside the running LMS process. This is the same behavior as Spotty's own modules and is not a bug in our code.

## Deviations from Plan

None - plan executed exactly as written.

## Known Stubs

The following are intentional Phase-1 stubs by design:

| Stub | File | Line | Reason |
|------|------|------|--------|
| `handleFeed` placeholder (type=textarea showing PLUGIN_SPOTON_NAME) | `Plugins/SpotOn/Plugin.pm` | ~75 | Phase 3 (Browse/OPML) will replace with full menu tree |
| `formatOverride` comment "Phase 4: updateTranscodingTable" | `Plugins/SpotOn/ProtocolHandler.pm` | ~25 | Phase 4 (Streaming) will call updateTranscodingTable for player-specific format selection |

These stubs do not prevent the plan's goal (loadable LMS plugin skeleton). They are planmäßige Platzhalter.

## Issues Encountered

- `perl -c` on LMS plugin modules fails outside the LMS runtime environment because `Slim::Utils::Log` inherits from `Log::Log4perl::Logger` and uses `main::SCANNER`/`main::PERFMON` constants that are not defined. This affects Spotty's own modules identically. Resolution: created temporary stub modules for syntax verification, confirming both PM files load correctly with proper return values from all methods.

## User Setup Required

None - no external service configuration required. Plugin binary ([spoton]) is referenced but not yet provided (Phase 1 scope: skeleton only).

## Next Phase Readiness

- Plan 01-02 (Helper.pm + Settings.pm) can proceed — Plugin.pm provides `_pluginDataFor('basedir')` and `$prefs` namespace
- Plan 01-03 (binary foundation) can proceed — custom-convert.conf pipelines reference [spoton] placeholder
- All base contracts (manifest, protocol, format, i18n, transcoding) are in place for LMS to recognize SpotOn as a valid plugin

---
*Phase: 01-plugin-skeleton-binary-foundation*
*Completed: 2026-05-27*
