---
phase: 04
slug: single-track-streaming
status: verified
threats_open: 0
asvs_level: 1
created: 2026-05-28
---

# Phase 04 — Security

> Per-phase security contract: threat register, accepted risks, and audit trail.

---

## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| Plugin.pm -> LMS TranscodingHelper | Plugin writes into global commandTable hashref via regex substitutions | Bitrate (integer), cache path (filesystem path), helper name (binary basename), normalization flag (0/1) |
| Plugin.pm -> OS process management | pkill/taskkill execution to clean up orphaned librespot processes | Helper binary path (from trusted search paths only) |
| Settings.pm POST handler | User-submitted form data processed in handler() | Normalization checkbox value (ternary-normalized to 0/1), bitrate (whitelist-validated) |
| ProtocolHandler.pm -> Helper.pm | getCapability reads from binary --check output (parsed JSON) | passthrough capability flag (boolean) |

---

## Threat Register

| Threat ID | Category | Component | Disposition | Mitigation | Status |
|-----------|----------|-----------|-------------|------------|--------|
| T-04-01 | Tampering | updateTranscodingTable bitrate regex | mitigate | Whitelist validation in Settings.pm:45-47 (`%valid_bitrates` hash, only 96/160/320 accepted); pref value set only after validation; regex `\d{2,3}` safe against injected non-digit content | closed |
| T-04-02 | Tampering | updateTranscodingTable helper name regex | mitigate | Helper path from Helper.pm::_findBin via `Slim::Utils::Misc::findbin` (trusted search paths only, Helper.pm:133); `basename()` applied at Plugin.pm:1060 strips all path components before regex injection | closed |
| T-04-03 | Tampering | updateTranscodingTable cacheDir in commandTable | mitigate | cacheDir computed at Plugin.pm:1050 via `catdir($serverPrefs->get('cachedir'), 'spoton')` — LMS server pref (not user input) plus fixed suffix 'spoton'; no user-controlled path component | closed |
| T-04-04 | Information Disclosure | commandTable logging at INFOLOG | accept | Logged at INFOLOG level with double guard `main::INFOLOG && $log->is_info` (Plugin.pm:1090); requires explicit debug enable; exposes cache path (non-sensitive filesystem path, no tokens or credentials) | closed |
| T-04-05 | Tampering | Settings.pm normalization pref | mitigate | Ternary `$paramRef->{'pref_normalization'} ? 1 : 0` at Settings.pm:52 enforces only 0 or 1; absent/undef/empty becomes 0; no arbitrary string accepted | closed |
| T-04-06 | Tampering | _killOrphanedProcesses pkill | mitigate | Unix path single-quote-escaped at Plugin.pm:136 (`s/'/'\\''/g`) matching Helper.pm::helperCheck pattern; `pkill -f '$safeHelper'` uses single-quoted argument. Windows path double-quote-wrapped with `"//g` stripping at Plugin.pm:132-133. Entire block wrapped in `eval {}` at Plugin.pm:129 for error containment | closed |
| T-04-07 | Denial of Service | _killOrphanedProcesses kills active processes | mitigate | `isPlaying()` check on all clients via `Slim::Player::Client::clients()` loop at Plugin.pm:118-124; `$isBusy` flag set on first active player; kill block skipped entirely when `$isBusy` is true | closed |
| T-04-08 | Tampering | playall flag manipulation | accept | `playall => 1` set server-side in Plugin.pm at lines 329 (_trackItem) and 977 (_albumTrackItem); XMLBrowser reads it from the server-generated feed array; no user-controlled HTTP parameter path exists for this flag | closed |

*Status: open · closed*
*Disposition: mitigate (implementation required) · accept (documented risk) · transfer (third-party)*

---

## Accepted Risks Log

| Risk ID | Threat Ref | Rationale | Accepted By | Date |
|---------|------------|-----------|-------------|------|
| AR-04-01 | T-04-04 | commandTable logging at INFOLOG exposes cache directory path. Cache dir is a non-sensitive filesystem path derived from LMS server configuration with fixed 'spoton' suffix — no tokens, credentials, or user data in scope. Logging requires explicit debug level enable by an administrator. Risk accepted. | gsd-security-auditor | 2026-05-28 |
| AR-04-02 | T-04-08 | playall flag is a server-side OPML feed value set by Plugin.pm code paths (_trackItem, _albumTrackItem). LMS XMLBrowser reads this flag exclusively from the feed array returned by plugin-controlled callbacks. There is no HTTP POST or query parameter that allows a user to inject or override this flag. Risk accepted. | gsd-security-auditor | 2026-05-28 |

*Accepted risks do not resurface in future audit runs.*

---

## Unregistered Flags

Both SUMMARY.md files (`04-01-SUMMARY.md` and `04-02-SUMMARY.md`) declare no new threat flags beyond those covered by the threat register. No unregistered attack surface detected.

---

## Security Audit Trail

| Audit Date | Threats Total | Closed | Open | Run By |
|------------|---------------|--------|------|--------|
| 2026-05-28 | 8 | 8 | 0 | gsd-security-auditor (claude-sonnet-4-6) |

---

## Sign-Off

- [x] All threats have a disposition (mitigate / accept / transfer)
- [x] Accepted risks documented in Accepted Risks Log
- [x] `threats_open: 0` confirmed
- [x] `status: verified` set in frontmatter

**Approval:** verified 2026-05-28
