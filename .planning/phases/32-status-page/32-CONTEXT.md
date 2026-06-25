# Phase 32: Status Page - Context

**Gathered:** 2026-06-25
**Status:** Ready for planning

<domain>
## Phase Boundary

A dedicated web page under `/plugins/SpotOn/status.html` that visualizes the real-time state of the SpotOn plugin — daemon health per player, API quota and token status, recent errors, and system information. Read-only dashboard with auto-polling, accessible via link from SpotOn Settings. No actions/mutations — purely informational.

</domain>

<decisions>
## Implementation Decisions

### Data Scope
- **D-01:** Four data categories: Daemon Health, API & Tokens, Error Overview, System Info.
- **D-02:** All data always available — no dependency on diagnosticMode pref. Basis data (daemon PID, uptime, token status) is already held in-memory; new in-memory counters needed for API request counts, 429 tracking, error history.
- **D-03:** System Info (plugin version, binary version + capabilities, LMS version, Perl version, OS) is static per LMS session — load once, don't poll.

### Update Mechanism
- **D-04:** Auto-polling every 5 seconds for dynamic data (Daemon Health, API/Token, Errors). Single JSON endpoint returns all dynamic data in one response.
- **D-05:** System Info loaded once on page load (separate endpoint or included in first poll response).
- **D-06:** Polling pauses when browser tab is not visible (visibilitychange API). Resumes immediately on tab focus with an instant poll.

### Page Structure
- **D-07:** Own standalone page at `/plugins/SpotOn/status.html` — not a tab in Settings. Link from Settings page to open Status Page (new tab).
- **D-08:** Tile/card grid layout — four cards (Daemon Health, API & Tokens, Recent Errors, System Info) in a 2x2 grid on desktop, stacking vertically on narrow screens.
- **D-09:** Registered via `Slim::Web::Pages->addRawFunction()` for the JSON endpoint. HTML served as static file from the plugin's HTML directory.

### Accessibility & Auth
- **D-10:** LMS-Auth gated — if LMS authentication is configured, user must be logged in. Otherwise open. Same behavior as the Settings pages.
- **D-11:** Read-only — no action buttons (no daemon restart, no log clear, no token refresh). All mutations stay in the Settings page.
- **D-12:** No CSRF protection needed on the status JSON endpoint (read-only).

### Claude's Discretion
- **Styling:** Claude chooses between dark theme, light theme, or LMS-inherited styling based on what works best in the LMS ecosystem.
- **Error history depth:** Claude decides how many recent errors to keep in memory (suggestion: ~20-50 entries, ring buffer).
- **Card detail level:** Claude decides the exact metrics shown per card and their formatting.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Web Framework Patterns
- `Plugins/SpotOn/Settings.pm` lines 24-57 — AJAX endpoint registration via `addRawFunction()`, `_jsonResponse()` helper, CSRF check pattern
- `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Existing AJAX polling pattern (XHR + JSON parse)

### Data Sources
- `Plugins/SpotOn/Unified/DaemonManager.pm` — `helperInstances()`, `helperForClient()`, `uptime()`, `streamPortForClient()`, `helperPids()`
- `Plugins/SpotOn/Unified/Daemon.pm` — `pid()`, `alive()`, `uptime()`, `mac`, `_connectEnabled`, `_streamPort`, `_startTimes` (crash-loop)
- `Plugins/SpotOn/API/Client.pm` — `$inflightCount`, `MAX_CONCURRENT_REQUESTS`, rate-limit keys (`spoton_rate_limit_own`/`spoton_rate_limit_bundled`), 429 handling (line 684)
- `Plugins/SpotOn/API/TokenManager.pm` — `getAccountIds()`, `getActiveAccountName()`, `isDiscoveryRunning()`, token cache keys, `TOKEN_REFRESH_TIMER`
- `Plugins/SpotOn/Helper.pm` — `get()`, `getVersion()`, `getCapability()`
- `Plugins/SpotOn/Connect.pm` — `$_activeConnectPlayer`, `isSpotifyConnect()`

### Architecture Decisions
- Phase 31 CSRF protection: write endpoints need `_csrfCheck()` — but status page is read-only, so not applicable

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `_jsonResponse()` in Settings.pm (lines 466-475) — JSON response helper, use directly or copy to status module
- `addRawFunction()` pattern — 5 endpoints already registered, proven AJAX pattern
- XHR polling JavaScript in basic.html (lines 210-251) — existing polling loop for discoveryStatus, reuse pattern

### Established Patterns
- AJAX endpoints live in Settings.pm as `_handlerName` subs, registered in `new()`
- JSON responses use `Slim::Web::HTTP::addHTTPResponse()` with Content-Type application/json
- HTML files go in `HTML/EN/plugins/SpotOn/` directory
- `Slim::Web::Pages->addRawFunction()` for endpoints outside the Settings framework

### Integration Points
- New JSON endpoint in Settings.pm (or new Status.pm module) — `plugins/SpotOn/status/data`
- New HTML file — `HTML/EN/plugins/SpotOn/status.html`
- Link from basic.html Settings page to status.html
- New in-memory counters in API/Client.pm for request counting and 429 tracking
- Plugin.pm `initPlugin()` — register status endpoint if `main::WEBUI`

</code_context>

<specifics>
## Specific Ideas

- Dashboard should feel like a modern monitoring page — tile grid is the explicit layout choice
- This is a first-of-its-kind for LMS plugins — no prior art in the LMS ecosystem for plugin status dashboards

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 32-Status Page*
*Context gathered: 2026-06-25*
