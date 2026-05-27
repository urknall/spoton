# Phase 1: Plugin Skeleton + Binary Foundation - Context

**Gathered:** 2026-05-27
**Status:** Ready for planning

<domain>
## Phase Boundary

The plugin loads cleanly under LMS and all LMS integration contracts are in place before any Spotify functionality is added. This phase delivers: install.xml manifest, `spotify://` protocol handler registration, settings page skeleton, i18n strings (EN + DE), custom-convert.conf with transcoding pipelines, multi-architecture librespot binaries with capability detection.

</domain>

<decisions>
## Implementation Decisions

### librespot Binary Source
- **D-01:** SpotOn uses its own librespot fork (not Herger's Spotty fork directly). Git strategy (merge-based fork vs. upstream + patchset) deferred to research phase — decision depends on actual patch scope after auditing Herger's fork.
- **D-02:** Research phase MUST audit Herger's librespot fork to determine: which LMS-specific patches (--lms, --check, --single-track, --get-token, --lms-auth, --player-mac) exist, which are portable, which are missing, which should be structurally rewritten.

### Binary Distribution
- **D-03:** Binaries are bundled in the plugin ZIP (Spotty model). All supported architectures (x86_64, aarch64, armhf, i386) ship in the ZIP under a `Bin/` directory.
- **D-04:** `Bin/` subdirectory structure uses Perl's `$Config{archname}` convention (e.g., `x86_64-linux-gnu-thread-multi`), not Spotty's simplified naming. This aligns with LMS internals.
- **D-05:** No download-at-first-use mechanism. What ships in the ZIP is what runs.

### Settings Page (Phase 1)
- **D-06:** Settings page is pre-structured with sections for Binary Status and Account Configuration. Account-related fields are present but disabled/greyed out in Phase 1 — they become active in Phase 2 (Auth).
- **D-07:** When prerequisites are missing (no binary found, later: no account configured), the plugin shows a status hint as the first entry in the OPML menu root (e.g., "Binary nicht gefunden — siehe Settings"). Research phase should verify this pattern is feasible within LMS OPML conventions.

### Transcoding Pipelines
- **D-08:** Format identifier is `son` (NOT `spt` — that's Spotty). MIME type: `audio/x-sb-spoton`. This allows SpotOn and Spotty to coexist without conflicts.
- **D-09:** All four pipelines registered in Phase 1: `son→flc` (default), `son→pcm`, `son→mp3`, `son→ogg` (passthrough). Pipelines are syntactically complete but only become functional when streaming is implemented in Phase 4.

### Claude's Discretion
- Fork strategy (merge-based vs. upstream + patchset) — decided during research based on patch scope analysis
- Menu root hint feasibility — verified during research against LMS OPML conventions

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### LMS Plugin API
- `.planning/PROJECT.md` — Project constraints, key decisions, prior art context
- `.planning/REQUIREMENTS.md` — LMS-01 through LMS-07 requirement definitions
- `CLAUDE.md` §Technology Stack — LMS Plugin API modules, librespot CLI flags, protocol handler pattern, transcoding format

### Spotty Reference (prior art)
- https://github.com/michaelherger/Spotty-Plugin — Herger's plugin (binary distribution model, Helper.pm architecture detection, custom-convert.conf format)
- https://github.com/michaelherger/spotty — Herger's librespot fork (archived, Rust source for LMS patches)
- https://github.com/michaelherger/librespot — Herger's active librespot fork (spotty branch)

### LMS Source
- https://github.com/LMS-Community/slimserver — LMS server source (Plugin API, types.conf, convert.conf conventions)
- https://github.com/LMS-Community/plugin-Qobuz — Qobuz plugin (reference for streaming plugin patterns, `qbz` format registration)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — greenfield project, no existing code

### Established Patterns
- Spotty's `Bin/` + `Helper.pm` pattern for binary discovery and `--check` validation is the proven LMS convention
- Qobuz's `custom-types.conf` + `custom-convert.conf` pattern for format registration
- LMS `OPMLBased` for menu trees, `Slim::Utils::Prefs` for settings

### Integration Points
- `install.xml` → LMS plugin manager recognizes and loads the plugin
- `custom-types.conf` → LMS learns the `son` format
- `custom-convert.conf` → LMS knows how to transcode `son` → output formats
- `Plugin.pm::initPlugin()` → registers protocol handler for `spotify://` URIs
- Settings HTML → LMS Settings page renders the SpotOn configuration panel

</code_context>

<specifics>
## Specific Ideas

- Binary validation via `--check` returning JSON with version and capabilities (Spotty pattern)
- Settings page shows binary version, architecture, and path even when account is not yet configured
- Custom binary override support (user drops own binary in Bin/ folder) — aligns with LMS-10

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 1-Plugin Skeleton + Binary Foundation*
*Context gathered: 2026-05-27*
