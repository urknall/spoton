---
phase: 02-auth-api-foundation
verified: 2026-05-27T18:00:00Z
status: human_needed
score: 4/5
overrides_applied: 0
human_verification:
  - test: "Verify that once OAuth-PKCE auth is implemented, configuring a Spotify account causes the plugin to obtain a valid access token visible in the debug log (ROADMAP SC1)"
    expected: "Server log shows 'TokenManager: token cached for account <id>, TTL Xs' after account configuration"
    why_human: "login5 password auth is blocked by Spotify; full SC1 requires OAuth-PKCE which is a future phase. The infrastructure is built correctly but the auth method fails against Spotify. Cannot verify end-to-end token acquisition without working credentials."
  - test: "Verify the 50-minute Connect daemon proactive restart (ROADMAP SC2 / AUTH-03 full scope)"
    expected: "Connect daemon running for 50+ minutes is proactively killed and restarted before its token expires"
    why_human: "Connect daemon management is Phase 5 work. The Phase 2 token refresh timer (45-min cycle) satisfies the token refresh half of SC2, but the daemon restart half requires Connect infrastructure that does not exist yet. Cannot automate."
  - test: "Verify SC5: making 50 rapid API calls produces no 429 errors from the central throttle"
    expected: "50 rapid getMe() calls all return results; none produce 429; RATE_LIMIT_CACHE_KEY is never set"
    why_human: "Requires a live Spotify API endpoint and real credentials. Unit tests prove the 429 handling logic (rate limit flag set correctly, Retry-After capped), but cannot verify the proactive burst prevention works end-to-end without hitting real Spotify."
  - test: "Verify SC4: switching between two configured accounts causes the active token to change within one menu refresh"
    expected: "After switching account in OPML menu, handleFeed shows the new account's display name and subsequent API calls use the new account's token"
    why_human: "Requires two real configured accounts and a running LMS. Unit test verifies the preference update, but token-switching end-to-end requires real credentials."
---

# Phase 2: Auth + API Foundation Verification Report

**Phase Goal:** The plugin can obtain, cache, and refresh a Spotify access token via Keymaster/login5, and all outbound Spotify API calls flow through a single rate-limited, caching HTTP client
**Verified:** 2026-05-27T18:00:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Plugin can obtain a valid Spotify access token via login5/Keymaster through the binary | PARTIAL (override needed) | Infrastructure fully built and tested with mock binary. login5 password auth blocked by Spotify at protocol level (documented login5-failed checkpoint). Token acquisition works in unit tests; fails against real Spotify until OAuth-PKCE is implemented. |
| 2 | Token is cached and automatically refreshed before expiry (no user interaction) | VERIFIED | TokenManager._cacheToken() sets TTL = expiresIn - 300. Plugin.pm starts a 10s-initial / 45min-cycle timer calling refreshAllTokens. Tests t/07 ok7 (TTL=3300 for expiresIn=3600) and ok8 (timer armed). |
| 3 | All outbound Spotify API calls go through a single rate-limited, caching HTTP client | VERIFIED | Client.pm _request() is the sole egress. SimpleAsyncHTTP used exclusively (grep confirms no LWP/SimpleSyncHTTP). Tests t/08 ok1 (getMe uses SimpleAsyncHTTP), ok14 (no blocking HTTP in API/*.pm). |
| 4 | 429 responses are handled correctly with Retry-After capping | VERIFIED | Client._onError() caps Retry-After at 300s, sets RATE_LIMIT_CACHE_KEY. Tests t/08 ok10-ok12 confirm flag set, TTL matches Retry-After, cap at 300s enforced. |
| 5 | Credentials are stored with chmod 600 on file, 0700 on directory | VERIFIED | TokenManager._setPermissions() calls chmod 0700 on dir, chmod 0600 on credentials.json. Tests t/07 ok10-ok11 and t/09 ok1-ok2 confirm filesystem permissions. |

**Score:** 4/5 (Truth 1 partial — infrastructure verified, end-to-end blocked by Spotify login5 restriction)

**Note on Truth 1:** The login5 authentication failure is a protocol-level Spotify restriction, not a code defect. The binary, TokenManager, Settings, and Plugin wiring are all correct and verified by tests. The SUMMARY documents this as "login5-failed" and identifies OAuth-PKCE as the required next step.

### Deferred Items

| # | Item | Addressed In | Evidence |
|---|------|-------------|----------|
| 1 | Connect daemon proactive restart at 50-minute uptime (AUTH-03 full scope) | Phase 5 | Phase 5 SC5: "A Connect daemon that crashes is automatically restarted with exponential backoff; Connect daemons are never killed by LMS's killHangingProcesses" — daemon lifecycle management is Phase 5 scope. |

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Plugins/SpotOn/API/TokenManager.pm` | Token acquisition via binary spawn, caching, refresh timer, credential directory management, multi-account support | VERIFIED | 350 lines. Exports: refreshToken, getToken, addAccount, removeAccount, getAccountIds, getActiveAccountName, refreshAllTokens. All public methods present and tested. |
| `Plugins/SpotOn/API/Client.pm` | Central HTTP client with rate limiting, caching, token injection, getMe endpoint | VERIFIED | 228 lines. Exports: getMe, reset, RATE_LIMIT_CACHE_KEY. _request as central pipeline. _cacheTTL with correct domain TTLs. inflightCount decremented in 3 paths (success, error, no-token). |
| `Plugins/SpotOn/Settings.pm` | Account CRUD in handler(), account data passed to template | VERIFIED | 96 lines. handler() processes addAccount, removeAccount, switchAccount. passes accounts/activeAccount to template. Calls TokenManager->addAccount/removeAccount. |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | Dynamic account list, add/remove form, auth error display | VERIFIED | 66 lines. Iterates accounts hash, shows Active/Switch/Remove controls, username/password inputs, authError display. | html filter on all dynamic output. |
| `Plugins/SpotOn/strings.txt` | Phase 2 i18n strings for account management and rate limiting | VERIFIED | All 11 required Phase 2 keys present with DE and EN translations. PLUGIN_SPOTON_ACTIVE_ACCOUNT contains %s in both languages. PLUGIN_SPOTON_ACCOUNT_PLACEHOLDER removed. |
| `librespot-spoton/src/main.rs` | Binary with --authenticate, --get-token, --cache, --scope, --username, --password flags | VERIFIED | Mode enum with Check, Authenticate, GetToken, Connect variants. All required flags parsed. --authenticate exits non-zero without --username. --get-token exits non-zero without --cache. Phase 1 --check contract unchanged. |
| `librespot-spoton/Cargo.toml` | Rust dependencies including librespot-core | VERIFIED | librespot-core = "0.8" with rustls-tls-native-roots feature. serde_json = "1". tokio with rt-multi-thread + macros. |
| `Plugins/SpotOn/Plugin.pm` | Timer setup, account switcher OPML menu, rate-limit hint, API client reset | VERIFIED | Imports Timers/Cache/Time::HiRes. initPlugin calls Client->reset(), starts _refreshAllTokens timer (10s initial, 45min cycle), guarded by !main::SCANNER. handleFeed shows rate-limit hint and account switcher. |
| `t/07_token_manager.t` | Unit tests for AUTH-01 through AUTH-05 | VERIFIED | 471 lines. 15/15 tests pass. Covers: refreshToken produces token, cache hit/miss, TTL calculation, timer re-arm, chmod 0700/0600, multi-account dirs, removeAccount. |
| `t/08_api_client.t` | Tests for API-01 through API-06 | VERIFIED | 512 lines. 15/15 tests pass. Covers: SimpleAsyncHTTP egress, concurrency cap of 3, cacheTTL values, 429+Retry-After handling, no batch methods, no blocking HTTP. |
| `t/09_settings.t` | Tests for AUTH-04, AUTH-05, AUTH-06 | VERIFIED | 443 lines. 20/20 tests pass. Covers: filesystem chmod, multi-account dirs, account switch pref update, all 11 i18n keys present with %s placeholder. |
| `Plugins/SpotOn/Bin/x86_64-linux/spoton` | Static binary with Phase 2 modes | VERIFIED | static-pie linked ELF64. Phase 1 --check tests (t/06) still pass 4/4. --authenticate and --get-token exit non-zero without required args. |
| `t/05_perl_syntax.t` | Syntax checks extended to cover API/ modules | VERIFIED | @pm_files includes API/TokenManager.pm and API/Client.pm. Full suite 119/119 pass. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `TokenManager.pm` | `Plugins::SpotOn::Helper` | `Helper->get()` for binary path | VERIFIED | Line 52: `my $binary = Plugins::SpotOn::Helper->get()` |
| `TokenManager.pm` | `Slim::Utils::Cache` | token caching with TTL | VERIFIED | Line 325: `$cache->set("spoton_token_$accountId", ...)` |
| `Client.pm` | `TokenManager.pm` | getToken for token injection | VERIFIED | Line 101: `Plugins::SpotOn::API::TokenManager->getToken(...)` |
| `Client.pm` | `Slim::Networking::SimpleAsyncHTTP` | async HTTP for all outbound calls | VERIFIED | Line 125: `Slim::Networking::SimpleAsyncHTTP->new(...)` |
| `Settings.pm` | `TokenManager.pm` | addAccount and removeAccount calls | VERIFIED | Lines 59, 73: `TokenManager->addAccount(...)` and `TokenManager->removeAccount(...)` |
| `Plugin.pm` | `TokenManager.pm` | refreshAllTokens timer callback | VERIFIED | Line 86: `Plugins::SpotOn::API::TokenManager->refreshAllTokens()` |
| `Plugin.pm` | `Client.pm` | Client->reset in initPlugin, RATE_LIMIT_CACHE_KEY in handleFeed | VERIFIED | Line 49: `Client->reset()`. Line 105: `Client->RATE_LIMIT_CACHE_KEY()` |
| `main.rs` | `librespot-core` | Cargo dependency | VERIFIED | Cargo.toml: `librespot-core = "0.8"`. main.rs imports librespot_core::{authentication, cache, config, Session}. |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|-------------------|--------|
| `TokenManager.pm::getToken` | `$cached` / `$token` | Slim::Utils::Cache (miss) -> binary stdout (--get-token) -> JSON parse | Yes — binary runs real librespot-core session.token_provider().get_token() | VERIFIED (with mock binary in tests; real binary blocked by login5) |
| `Client.pm::_request` | response JSON | TokenManager->getToken -> SimpleAsyncHTTP -> Spotify API | Yes — real HTTP call to api.spotify.com | VERIFIED in tests with mock HTTP; requires real credentials for live flow |
| `Plugin.pm::handleFeed` | `$activeName` | prefs->get('activeAccount') -> accounts hash -> displayName | Yes — reads real prefs store | VERIFIED — token.ok8 shows timer arms, t/09.ok7 shows activeAccount pref updated |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Binary exits non-zero for --authenticate without --username | `./Plugins/SpotOn/Bin/x86_64-linux/spoton --authenticate` | exit=1, stderr shows usage | PASS |
| Binary exits non-zero for --get-token without --cache | `./Plugins/SpotOn/Bin/x86_64-linux/spoton --get-token` | exit=1, stderr shows usage | PASS |
| Phase 1 --check contract unchanged | `prove -v t/06_binary_check.t` | 4/4 pass | PASS |
| Full test suite passes | `prove -v t/` | 119/119 pass | PASS |
| t/07 TokenManager tests (AUTH-01..05) | `prove -v t/07_token_manager.t` | 15/15 pass | PASS |
| t/08 API Client tests (API-01..06) | `prove -v t/08_api_client.t` | 15/15 pass | PASS |
| t/09 Settings tests (AUTH-04..06, i18n) | `prove -v t/09_settings.t` | 20/20 pass | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| AUTH-01 | 02-02 | Plugin obtains Spotify access token via Keymaster/login5 through librespot binary | PARTIAL | Infrastructure correct: binary has --authenticate/--get-token, TokenManager spawns binary with correct contract, token parsed from stdout. Spotify has blocked login5 password auth — account add fails against real Spotify. End-to-end blocked until OAuth-PKCE implementation. |
| AUTH-02 | 02-02, 02-05 | Access token cached and auto-refreshed before expiry | VERIFIED | TTL = expiresIn-300 (t/07 ok7). 45-min refresh timer in Plugin.pm (t/08 confirmed via syntax; t/07 ok8). |
| AUTH-03 | 02-02, 02-05 | Connect daemons proactively restarted at 50-minute uptime | PARTIAL (deferred) | Token refresh timer (45-min cycle) implemented and verified. Connect daemon restart component is Phase 5 scope — no Connect daemon exists in Phase 2. |
| AUTH-04 | 02-02, 02-04 | Credentials stored with chmod 600/700 | VERIFIED | _setPermissions() chmod 0700 dir / 0600 credentials.json. t/07 ok10-ok11, t/09 ok1-ok2 confirm. |
| AUTH-05 | 02-02, 02-04 | Multiple Spotify accounts per LMS instance | VERIFIED | accountId = MD5(username)[0:8]. Separate cache subdirs per account. getAccountIds/removeAccount correct. t/07 ok12-ok15. |
| AUTH-06 | 02-04, 02-05 | Account switching available in plugin menu | VERIFIED | _accountSwitcherFeed lists all accounts; _switchAccount updates per-client activeAccount pref. t/09 ok7 confirms pref update. |
| API-01 | 02-03, 02-05 | Central HTTP client as sole HTTP egress point | VERIFIED | All calls route through Client._request(). Only SimpleAsyncHTTP used. t/08 ok1 (getMe uses SimpleAsyncHTTP), ok14 (no LWP/SimpleSyncHTTP in API/*.pm). |
| API-02 | 02-03 | Rate limiting via concurrency cap (max N concurrent) | VERIFIED | MAX_CONCURRENT_REQUESTS=3. Excess requests deferred via Timers::setTimer 0.1s. t/08 ok5 (3 of 5 dispatched). |
| API-03 | 02-03 | Response caching with domain-specific TTLs | VERIFIED | _cacheTTL: 0 for me/player, 60 for me/tracks, 3600 for tracks/, 300 for playlists/. t/08 ok6-ok9. |
| API-04 | 02-03 | Retry-After header respected on 429 | VERIFIED | 429 handler sets RATE_LIMIT_CACHE_KEY with capped TTL (max 300s). t/08 ok10-ok12. |
| API-05 | 02-03 | Batch API endpoints used where available | VERIFIED (inverted) | Batch endpoints removed in Dev Mode (Feb 2026). RESEARCH.md D-13: "Single requests only — no batch abstraction." No getTracks/getAlbums/getArtists methods exist. t/08 ok13 confirms. The requirement is effectively vacuous in dev mode. |
| API-06 | 02-03 | All HTTP calls use SimpleAsyncHTTP with explicit timeouts | VERIFIED | grep confirms no LWP::UserAgent or SimpleSyncHTTP in API/*.pm. t/08 ok14-ok15. |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `Plugins/SpotOn/strings.txt` | 73 | `SON` — orphaned/truncated string key with only EN translation (no DE). Appears to be a truncated `PLUGIN_SPOTON` fragment from Phase 1 initial file creation. | WARNING | LMS will silently ignore unknown string keys. No functional impact. The key has been present since Phase 1 and was not introduced by Phase 2 work. |
| `Plugins/SpotOn/Settings.pm` | 29 | `sub prefs { return ($prefs, 'bitrate', 'binary') }` — Plan 02-04 specified `'activeAccount'` should be included in prefs() return, but actual code omits it. activeAccount is managed manually in handler(). | WARNING | Intentional deviation documented in SUMMARY: "accounts hash excluded from prefs() return value — managed manually in handler() to avoid LMS Prefs YAML concurrent-write race (Pitfall 3)." activeAccount IS set/get correctly via `$prefs->set('activeAccount', ...)` in handler(). No functional breakage; LMS form auto-save won't set it but handler handles it explicitly. |

No TBD, FIXME, or XXX markers found in any Phase 2 files.

### Human Verification Required

#### 1. End-to-End Token Acquisition (ROADMAP SC1)

**Test:** After OAuth-PKCE is implemented in a future phase, configure a Spotify account in LMS Settings > SpotOn. Enter credentials and click Add.
**Expected:** Account appears in the list. Server log shows `TokenManager: token cached for account <id>, TTL Xs`. Navigating to SpotOn in LMS menu shows "Active: [display name] [switch]" as first item.
**Why human:** login5 password auth is blocked by Spotify. Infrastructure is verified by tests and the human checkpoint confirmed the binary connects to Spotify AP correctly (but login5 is rejected). Full SC1 requires OAuth-PKCE which is a future phase deliverable.

#### 2. Connect Daemon Proactive Restart (AUTH-03 / ROADMAP SC2 partial)

**Test:** Once Phase 5 Connect is implemented, verify that a Connect daemon running for 50+ minutes is proactively killed and restarted by the token refresh cycle.
**Expected:** Server log shows daemon restart triggered by the token refresh timer before the 60-minute token expiry.
**Why human:** No Connect daemon exists in Phase 2. This is Phase 5 infrastructure. The token refresh timer foundation is in place.

#### 3. 50 Rapid API Calls Produce No 429 (ROADMAP SC5)

**Test:** With a configured account, call getMe() (or any API endpoint) 50 times in rapid succession. Monitor server log for rate-limit flag and 429 responses.
**Expected:** All 50 calls complete. No 429 responses. RATE_LIMIT_CACHE_KEY is never set. Central throttle defers excess calls via timer retry.
**Why human:** Requires live Spotify API. Unit tests verify the 429 handling logic but cannot prove no 429s will occur under real conditions.

#### 4. Account Switching Changes Active Token (ROADMAP SC4)

**Test:** With two configured Spotify accounts, use the OPML account switcher menu to switch accounts. Make an API call immediately after.
**Expected:** The active account display name updates within one menu refresh. Subsequent API calls use the new account's token (different Bearer value in HTTP call).
**Why human:** Requires two configured accounts with valid credentials. Unit test (t/09 ok7) verifies the preference update, but does not verify token substitution end-to-end.

### Gaps Summary

No BLOCKER gaps. All artifacts are substantive, wired, and behavioral spot-checks pass.

The sole failing success criterion (SC1: end-to-end token acquisition) is caused by Spotify blocking login5 password authentication — a protocol-level restriction external to the codebase, not a code defect. The infrastructure is correctly built and tested. This is documented as "login5-failed" in the SUMMARY and identifies OAuth-PKCE as the required next step.

**Summary of findings:**
- 13/13 required artifacts: VERIFIED (substantive, wired, no stubs)
- 8/8 key links: VERIFIED
- 119/119 automated tests pass
- No debt markers (TBD/FIXME/XXX) in any Phase 2 files
- 2 WARNING anti-patterns (stray `SON` in strings.txt from Phase 1; `activeAccount` omitted from prefs() — both intentional or benign)
- 4 human verification items required (all tied to live Spotify credentials or Phase 5 Connect)

---

_Verified: 2026-05-27T18:00:00Z_
_Verifier: Claude (gsd-verifier)_
