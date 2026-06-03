# Phase 7: DE→EN Code Cleanup - Context

**Gathered:** 2026-06-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Replace all German text in code comments and log strings with idiomatic English across all Perl and Rust source files. Opportunistically remove redundant English comments encountered during the sweep. The codebase should read as a consistently English project afterwards.

</domain>

<decisions>
## Implementation Decisions

### Translation Strategy
- **D-01:** Idiomatic English rewrite, not literal translation. Comments are rephrased naturally rather than word-for-word translated.
- **D-02:** Redundant comments (stating what the code already says) are deleted rather than translated — applies to both German and English comments encountered during the sweep.
- **D-03:** Comments that explain a non-obvious WHY are kept and translated/improved.

### Scope Boundaries
- **D-04:** All German text is in scope — real Umlauts (ä/ö/ü/ß/Ä/Ö/Ü) AND ASCII workarounds (aendern, Laengen, Zeichen, Schutz, etc.). Goal is a fully English codebase, not just passing the CLEAN-03 grep check.
- **D-05:** Both Perl (.pm) and Rust (.rs) source files are in scope. Rust source currently has no German text but should be verified and included in the sweep.
- **D-06:** i18n files (strings.txt) are explicitly excluded — they contain user-facing translations and are not code comments.

### Technical References
- **D-07:** Task-IDs (T-04.4-01, D-09, WR-01) and Phase references are preserved. Only the surrounding German description text is translated.
- **D-08:** Pitfall references (P-01 through P-20) get a short English context added, e.g. `# P-01: never parallelize pagination` — making the comment self-explanatory without consulting RESEARCH.md.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/PROJECT.md` — Pitfall list (P-01 through P-20) referenced in code comments; needed for accurate Pitfall context annotations (D-08)
- `.planning/REQUIREMENTS.md` — CLEAN-01, CLEAN-02, CLEAN-03 requirement definitions

### Source Files (all in scope)
- `Plugins/SpotOn/Plugin.pm` — Largest file (1337 LOC), most German comments
- `Plugins/SpotOn/Settings.pm` — German comments about input validation and degraded mode
- `Plugins/SpotOn/Helper.pm` — German comments about architecture fallbacks and binary selection
- `Plugins/SpotOn/Connect.pm` — Scattered German comments
- `Plugins/SpotOn/API/Client.pm` — Minimal German, mostly English already
- `Plugins/SpotOn/API/TokenManager.pm` — Minimal German
- `Plugins/SpotOn/Connect/DaemonManager.pm` — Minimal German
- `Plugins/SpotOn/Connect/Daemon.pm` — Minimal German
- `Plugins/SpotOn/DontStopTheMusic.pm` — Already mostly English
- `Plugins/SpotOn/ProtocolHandler.pm` — Already mostly English
- `librespot-spoton/src/main.rs` — Verify clean (currently no German text)
- `librespot-spoton/src/connect.rs` — Verify clean (currently no German text)

</canonical_refs>

<code_context>
## Existing Code Insights

### Scope Assessment
- ~18 German comment lines across 10 Perl modules (5,159 LOC total)
- 0 German log strings found — all DEBUGLOG/INFOLOG/WARNLOG/ERRORLOG calls already emit English
- 0 German text in Rust source files (librespot-spoton/src/)
- Heaviest concentration: Helper.pm, Settings.pm, Plugin.pm

### Comment Patterns Found
- Architecture notes: `# aarch64 kann als Fallback armhf-Binaries verwenden`
- Safety notes: `# KRITISCH: 'spoton' nicht 'spotty' im Regex`
- Input validation: `# T-04.4-01: Input-Validierung — nur alphanumerische Zeichen`
- Feature context: `# Kontext-Queueing (D-09/D-10)`
- Mode descriptions: `# Degraded = kein Custom-Client-ID konfiguriert`
- ASCII Umlaut workarounds: `aendern`, `Laengen` (used where editor/encoding prevented real Umlauts)

### Integration Points
- No structural changes needed — this is a text-only sweep
- No imports, exports, function signatures, or tests affected

</code_context>

<specifics>
## Specific Ideas

No specific requirements — standard idiomatic translation with the decisions above applied consistently.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 7-DE→EN Code Cleanup*
*Context gathered: 2026-06-03*
