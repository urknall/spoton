# Phase 2: Auth + API Foundation - Research

**Researched:** 2026-05-27
**Domain:** Spotify authentication via librespot binary, Perl LMS API client architecture, rate limiting in single-threaded event loop
**Confidence:** HIGH (core stack verified from source code; two discretion items with MEDIUM confidence noted)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-02:** Credentials stored in librespot cache directory (`prefs/spoton/<account-id>/`), not LMS Prefs. Auth-blob in librespot-native format. Directory chmod 700, credentials file chmod 600 (AUTH-04).
- **D-03:** Token acquisition via short-lived process — plugin spawns librespot with `--get-token`, gets token via stdout, process exits. No persistent auth daemon for tokens only.
- **D-04:** Proactive timer for token refresh — LMS timer refreshes token before expiry (e.g., at 45 of 60 minutes). No API call should fail due to expired token.
- **D-05:** Global account list with menu switcher. Accounts configured in Plugin Settings. In the OPML menu, an account switcher appears as the first line ("Aktiv: [Name] [wechseln]"). Switching changes the active account for Browse/Search/Library and refreshes the menu via `nextWindow => 'refreshOrigin'`.
- **D-06:** Multi-Account implemented in Phase 2 — Settings, API/Client, and token management work with account IDs from the start. No later refactoring needed.
- **D-07:** Settings page shows a dynamic account list with add/remove. No fixed slots.
- **D-08:** Connect (Phase 5) operates independently of configured Settings accounts. Spotify app authenticates directly at the librespot daemon via Zeroconf. Only Browse/Search/Library needs a configured account.
- **D-12:** When throttled, an OPML menu hint ("Spotify-Anfragen gedrosselt") appears. Transparency for the user.
- **D-13:** Single requests only — no batch abstraction. Extended Quota is unrealistic for open-source plugins (250k MAU + commercial org). YAGNI.
- **D-15:** Phase 2 implements only auth-relevant endpoints: token management, `GET /me` (account validation), error handling, rate-limiting infrastructure. Browse/Search/Library endpoints come in Phase 3.
- **D-16:** Response caching via LMS Cache (`Slim::Utils::Cache` with namespace `spoton`). Persists across restarts, built-in TTL support.

### Claude's Discretion

- **D-01:** Credential input method (Username/Password vs. Zeroconf discovery) — research decides based on feasibility analysis
- **D-09:** Rate-limiting mechanism (Token Bucket vs. Sliding Window vs. Adaptive) — research decides
- **D-10:** Concurrency limit for simultaneous API requests — research determines optimal value
- **D-11:** Request queue prioritization (High/Normal) — research evaluates necessity
- **D-14:** API client module structure (monolithic vs. layered) — research decides

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| AUTH-01 | Plugin obtains Spotify access token via Keymaster/login5 through librespot binary | Herger's spotty `--get-token` flag confirmed: spawns binary with `--cache <dir> --get-token --client-id <id>`, token JSON printed to stdout |
| AUTH-02 | Access token is cached and automatically refreshed before expiry | LMS Timer pattern confirmed: `Slim::Utils::Timers::setTimer` with 45-min interval; `Slim::Utils::Cache` with expiry-300s TTL |
| AUTH-03 | Connect daemons are proactively restarted at 50-minute uptime | Timer-based daemon restart; covered in Phase 2 for token management, full daemon lifecycle in Phase 5 |
| AUTH-04 | Credentials stored with restricted permissions (chmod 600, directory 0700) | librespot-native `credentials.json` in per-account cache subdirectory; Perl `chmod()` after write |
| AUTH-05 | Multiple Spotify accounts can be configured per LMS instance | Spotty `AccountHelper.pm` pattern: per-account cache subdirs (8-char MD5 hash of username), account list from Prefs |
| AUTH-06 | Account switching is available in the plugin menu | OPML item with `nextWindow => 'refreshOrigin'` confirmed; per-client `$prefs->client($client)` stores active account |
| API-01 | Central HTTP client (`API/Client.pm`) as sole HTTP egress point | Single-module pattern; all requests funnel through `_request()` method |
| API-02 | Rate limiting via sliding window or adaptive throttle (max N concurrent requests) | Research recommends: sliding window counter with concurrency cap of 3; see Pitfalls for reasoning |
| API-03 | Response caching with domain-specific TTLs (60s Library, 300s Browse, 3600s Metadata) | `Slim::Utils::Cache->new()` confirmed; TTLs from CLAUDE.md |
| API-04 | `Retry-After` header respected on 429 responses | Spotty pattern confirmed: cache `rate_limit_exceeded` key with TTL = Retry-After value (default 5s) |
| API-05 | Batch API endpoints used where available | [ASSUMED] Dev mode removed batch endpoints; single requests only (D-13 locked) |
| API-06 | All HTTP calls use `SimpleAsyncHTTP` with explicit timeouts (never blocking) | LMS pattern confirmed: `Slim::Networking::SimpleAsyncHTTP->new($cb, $ecb, { timeout => 30 })` |
</phase_requirements>

---

## Summary

Phase 2 establishes the authentication and API infrastructure that every subsequent phase depends on. The core challenge is bridging Spotify's OAuth-only authentication (username/password was disabled by Spotify in Aug 2024) with a headless LMS plugin that cannot open a browser.

**The resolved path:** The librespot binary implements OAuth 2.0 with PKCE via `--enable-oauth`, which opens a local HTTP server for the browser redirect. For the plugin, this means credential acquisition happens in two steps: (1) user opens a Spotify-authorization URL in their browser (linked from the LMS settings page), (2) the binary captures the redirect and saves credentials.json to the cache directory. Subsequent token refreshes use `--get-token` against the cached credentials, printing a Web API token to stdout.

The API client (`API/Client.pm`) is a thin wrapper around `SimpleAsyncHTTP` with a sliding-window rate limiter (30-second window, max ~30 requests), a concurrency cap of 3, 429-flag caching, response caching via `Slim::Utils::Cache`, and callback-based async token injection. The module structure should be **layered**: `API/Client.pm` owns HTTP/auth/throttle; endpoint methods come in Phase 3. For Phase 2, only `getMe()` is needed to validate the token and display the account name.

**Primary recommendation:** Use the layered module structure (D-14 resolved: `API/Client.pm` for infrastructure only, endpoint modules in Phase 3). Use sliding window + concurrency cap of 3 for rate limiting (D-09, D-10 resolved). No queue prioritization needed in Phase 2 (D-11 resolved: FIFO sufficient until Phase 3 pagination reveals need).

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Credential acquisition (OAuth redirect) | LMS Settings (Settings.pm) | librespot binary (--enable-oauth) | User action triggers binary spawn; settings page provides the auth URL and status |
| Credential storage (credentials.json) | librespot binary | Filesystem (chmod 600/700 enforced by Plugin.pm) | Binary writes its own native credential format; plugin sets permissions after write |
| Token acquisition (`--get-token`) | librespot binary | API/TokenManager.pm (parses stdout) | Binary encapsulates Keymaster/Mercury protocol; Perl just parses the JSON from stdout |
| Token caching and refresh scheduling | API/TokenManager.pm | Plugin.pm (timer setup) | LMS Timer starts the 45-min cycle; TokenManager executes the binary spawn and cache write |
| API request routing | API/Client.pm | — | Central egress point; all outbound HTTP flows through here per API-01 |
| Rate limiting and 429 handling | API/Client.pm | — | Sliding window counter + concurrency cap + Retry-After cache key |
| Response caching | API/Client.pm | Slim::Utils::Cache | Client checks cache before making HTTP calls; writes cache on success |
| Account management (CRUD) | Settings.pm | Plugin.pm (prefs init) | Settings page handles add/remove; Plugin.pm initializes account prefs structure |
| Account switching UI | Plugin.pm (handleFeed) | — | OPML menu item as first row; `nextWindow => 'refreshOrigin'` triggers menu refresh |
| Active account per player | Slim::Utils::Prefs (per-client) | API/Client.pm (reads active account) | Per-player prefs (`$prefs->client($client)->get('account')`) following Spotty AccountHelper pattern |

---

## Standard Stack

### Core (no new external packages — all LMS bundled)

| Module | Source | Purpose | Why Standard |
|--------|--------|---------|--------------|
| `Slim::Utils::Timers` | LMS bundled | Schedule token refresh, daemon restart timers | LMS event loop integration — the only correct async scheduling mechanism |
| `Slim::Utils::Cache` | LMS bundled | Token cache, response cache, rate-limit flag | Persists across LMS restarts, built-in TTL, used by Spotty and Qobuz |
| `Slim::Networking::SimpleAsyncHTTP` | LMS bundled | Non-blocking HTTP for all Spotify API calls | LMS is single-threaded; blocking HTTP (LWP) would freeze the entire server |
| `JSON::XS::VersionOneAndTwo` | LMS bundled | Parse token stdout JSON, API responses | Bundled, fast XS version with LMS version compatibility shim |
| `Slim::Utils::Prefs` | LMS bundled | Account list, active account per-client, plugin config | Persistence, migration, change callbacks — LMS idiom |
| `File::Spec::Functions` | Perl core | Cross-platform path construction | catdir/catfile for credential directory paths |
| `Digest::MD5` | LMS bundled | 8-char account ID from username hash | Follows Spotty AccountHelper pattern for stable dir names |

**No new CPAN packages required for Phase 2.** [VERIFIED: from Spotty source and LMS bundle review]

### Package Legitimacy Audit

No external packages are installed in this phase. All functionality uses LMS-bundled modules.

| Package | Registry | Status |
|---------|----------|--------|
| All Phase 2 modules | LMS bundled / Perl core | No external install needed |

---

## Architecture Patterns

### System Architecture Diagram

```
User Browser
    |
    | (1) Click "Add Account" in LMS Settings
    v
Settings.pm::handler()
    |
    | (2) Spawn: spoton --name "SpotOn-Auth" --enable-oauth --cache <dir>
    |           Binary opens local HTTP server for redirect
    v
librespot binary (short-lived OAuth helper process)
    |
    | (3) User authorizes in browser → redirect to binary HTTP server
    | (4) Binary exchanges code, writes credentials.json to <dir>/<account-id>/
    | (5) Binary exits (prints "authorized" or account info to stdout)
    v
Plugin.pm::initPlugin()
    |
    | (6) Start token refresh timer (Slim::Utils::Timers::setTimer, 45 min cycle)
    v
API/TokenManager.pm::refreshToken($accountId, $cb)
    |
    | (7) Spawn: spoton -n "SpotOn" --cache <dir>/<account-id> --get-token --client-id <id>
    | (8) Capture stdout → parse JSON token → cache in Slim::Utils::Cache
    |     Cache key: "spoton_token_<accountId>", TTL: expires_in - 300s
    v
API/Client.pm::_request($method, $path, $params, $cb)
    |
    | (9) Rate limit check (sliding window + concurrency cap)
    |     If rate-limited: return early, set OPML hint flag
    | (10) Get token from TokenManager (cache hit 99% of the time)
    | (11) Slim::Networking::SimpleAsyncHTTP->new($cb, $ecb, {timeout=>30})->get($url, headers)
    |
    +--[200 OK]--> Cache response, invoke callback
    |
    +--[429]-----> Set cache key "spoton_rate_limit" TTL=Retry-After, invoke error callback
    |
    +--[401]-----> Invalidate cached token, trigger refresh, retry once
```

### Recommended Project Structure

```
Plugins/SpotOn/
├── Plugin.pm              # initPlugin: timer setup, account prefs init
├── Helper.pm              # Binary discovery (Phase 1, reused)
├── Settings.pm            # Account CRUD, auth status display
├── HTML/EN/plugins/SpotOn/settings/
│   └── basic.html         # Dynamic account list + "Add Account" flow
├── API/
│   ├── Client.pm          # HTTP egress: rate limit, cache, token inject
│   └── TokenManager.pm    # Binary spawn for --get-token, token cache
└── strings.txt            # i18n strings for account switcher, throttle hint
```

### Pattern 1: Token Acquisition via Binary Stdout (AUTH-01)

**What:** Spawn the librespot binary with `--get-token`, read stdout, parse JSON token.
**When to use:** On first token request after plugin start, and on each timer-triggered refresh.

```perl
# Source: Herger's spotty src/main.rs --get-token implementation (github.com/michaelherger/spotty)
sub _spawnGetToken {
    my ($class, $accountId, $cb) = @_;

    my $binary = Plugins::SpotOn::Helper->get() or do {
        $log->error("No binary found for --get-token");
        $cb->(undef);
        return;
    };

    my $cacheDir = $class->_cacheDir($accountId);
    my $clientId = $class->_clientId();
    my $scope    = join(',', @REQUIRED_SCOPES);

    # Shell-safe quoting
    (my $safeBin   = $binary)   =~ s/'/'\\''/g;
    (my $safeCache = $cacheDir) =~ s/'/'\\''/g;
    (my $safeId    = $clientId) =~ s/'/'\\''/g;
    (my $safeScope = $scope)    =~ s/'/'\\''/g;

    my $cmd = sprintf(
        "'%s' -n 'SpotOn' --cache '%s' --get-token --client-id '%s' --scope '%s' 2>/dev/null",
        $safeBin, $safeCache, $safeId, $safeScope
    );

    # NOTE: This is a blocking backtick call.
    # Acceptable because: --get-token exits immediately after token retrieval.
    # Token retrieval involves one Spotify API round-trip (~200-500ms).
    # For async spawn, use Proc::Background (available in LMS) if latency becomes an issue.
    my $output = `$cmd`;

    if ($output && $output =~ /^\{/) {
        my $token = eval { from_json($output) };
        if (!$@ && $token && $token->{accessToken}) {
            $cb->($token);
            return;
        }
    }

    $log->error("Failed to get token for account $accountId: $output");
    $cb->(undef);
}
```

**Important:** The `--get-token` spawn is blocking (backtick). This is acceptable for token refresh because: (a) it happens on a timer, not in a hot path; (b) the binary exits immediately after one token round-trip (~200-500ms). The `--check` call in Phase 1 uses the same pattern. `[VERIFIED: Spotty Helper.pm backtick pattern]`

### Pattern 2: Credential Acquisition via OAuth (D-01 resolved)

**Decision (D-01): Use username/password via librespot `--authenticate` flag.**

**Rationale:** Spotify disabled username/password in Aug 2024 for most accounts. However, the `--authenticate` flow works differently from simple HTTP Basic auth — librespot uses the login5 protocol which accepts username + password to obtain stored credentials (credentials.json). Current librespot 0.8.0 (upstream) dropped this; **Herger's spotty fork still supports it** for the LMS use case.

**Two paths for the librespot-spoton binary:**

Option A (Phase 2 implementation): **Username/Password via login5-style binary authentication**
- User enters username + password in LMS Settings form
- Plugin spawns: `spoton -n "SpotOn-Auth" --username <u> --password <p> --authenticate --cache <dir>`
- Binary authenticates against Spotify via login5 protocol and writes credentials.json
- Plugin calls chmod 600 on credentials.json, chmod 700 on parent dir
- Plugin reads back username from credentials.json to derive the account ID (MD5 hash)

Option B (fallback/future): **OAuth via browser redirect**
- For users where login5 username/password fails (API key gating, TFA, etc.)
- Binary spawns with `--enable-oauth`, opens local HTTP server, plugin links user to auth URL
- More complex UI in settings, requires port availability

**Recommendation: Implement Option A (username/password) first** — it is what Herger's spotty supports and is the simplest UX. The librespot-spoton Rust binary must implement `--authenticate` using login5 credentials. Mark the `lms-auth` capability as `true` in `--check` JSON when this is implemented.

Option B should be noted as a future enhancement if Spotify locks down login5 further. `[ASSUMED: username/password via login5 still works in Herger's spotty fork as of 2024 — needs verification against latest spotty binary behavior]`

```perl
# Plugin spawns binary for initial credential acquisition
# Source: Herger's spotty src/main.rs --authenticate flow
sub addAccount {
    my ($class, $username, $password, $cb) = @_;

    my $binary   = Plugins::SpotOn::Helper->get() or return $cb->(undef, 'No binary');
    my $cacheDir = $class->_newAccountCacheDir();  # creates __AUTHENTICATE__ temp dir

    # Shell-safe quoting
    (my $safeBin  = $binary)   =~ s/'/'\\''/g;
    (my $safeDir  = $cacheDir) =~ s/'/'\\''/g;
    (my $safeUser = $username) =~ s/'/'\\''/g;
    (my $safePass = $password) =~ s/'/'\\''/g;

    my $cmd = sprintf(
        "'%s' -n 'SpotOn' --username '%s' --password '%s' --authenticate --cache '%s' 2>&1",
        $safeBin, $safeUser, $safePass, $safeDir
    );

    my $output = `$cmd`;

    if ($output =~ /^authorized/i) {
        # Move temp dir to permanent location (MD5 of username)
        my $accountId = substr(md5_hex(Slim::Utils::Unicode::utf8toLatin1Transliterate($username)), 0, 8);
        $class->_finalizeAccountDir($cacheDir, $accountId);
        $class->_setPermissions($accountId);  # chmod 600/700
        $cb->($accountId, undef);
    } else {
        $log->error("Authentication failed: $output");
        $cb->(undef, "Authentication failed");
    }
}
```

### Pattern 3: Token Caching with Pre-Expiry Refresh (AUTH-02)

```perl
# Source: Derived from Spotty API/Token.pm caching pattern
use constant TOKEN_EXPIRY_BUFFER => 300;  # Refresh 5 minutes before expiry
use constant TOKEN_REFRESH_TIMER => 45 * 60;  # 45-minute timer interval

sub cacheToken {
    my ($class, $accountId, $tokenData) = @_;

    my $expiresIn = $tokenData->{expiresIn} || 3600;
    my $cacheTTL  = $expiresIn > TOKEN_EXPIRY_BUFFER
                    ? $expiresIn - TOKEN_EXPIRY_BUFFER
                    : $expiresIn;

    my $cacheKey = "spoton_token_$accountId";
    $cache->set($cacheKey, $tokenData->{accessToken}, $cacheTTL);

    main::INFOLOG && $log->info("Token cached for $accountId, expires in ${cacheTTL}s");
}

# In Plugin.pm::initPlugin or postinitPlugin:
sub _startTokenRefreshTimer {
    my $class = shift;
    Slim::Utils::Timers::killTimers($class, \&_refreshAllTokens);
    Slim::Utils::Timers::setTimer(
        $class,
        Time::HiRes::time() + TOKEN_REFRESH_TIMER,
        \&_refreshAllTokens
    );
}

sub _refreshAllTokens {
    my $class = shift;
    # Refresh token for each configured account
    for my $accountId (Plugins::SpotOn::Settings->getAccountIds()) {
        Plugins::SpotOn::API::TokenManager->refreshToken($accountId, sub {
            my $token = shift;
            $log->warn("Token refresh failed for $accountId") unless $token;
        });
    }
    # Re-arm timer
    $class->_startTokenRefreshTimer();
}
```

### Pattern 4: Rate Limiting in Single-Threaded LMS (API-02, API-04) — D-09, D-10, D-11 resolved

**D-09 resolution: Sliding Window + 429-flag cache (not adaptive, not token bucket)**

Rationale: Spotify's rate limit is a rolling 30-second window, app-wide. A sliding window counter mirrors this exactly. Token bucket allows burst which is dangerous — Spotify punishes burst patterns. Adaptive is over-engineered for Phase 2 (we have no telemetry to drive adaptation).

**D-10 resolution: Max 3 concurrent requests**

Rationale: LMS's single-threaded event loop means "concurrent" requests interleave at I/O boundaries, not true parallelism. Spotty-NG had problems at 5+. 3 balances responsiveness (pagination needs multiple calls) against safety. `[ASSUMED: Spotify has no documented per-request concurrency limit; 3 is pragmatic based on Spotty-NG operational history]`

**D-11 resolution: No priority queue for Phase 2**

Rationale: Phase 2 only has `GET /me` as an endpoint. Priority queuing becomes relevant in Phase 3 when Browse/Search trigger many simultaneous requests. FIFO is correct now.

```perl
# Source: Derived from Spotty API.pm error429() pattern + LMS Slim::Utils::Cache

use constant RATE_LIMIT_CACHE_KEY => 'spoton_rate_limit_exceeded';
use constant RATE_LIMIT_DEFAULT_BACKOFF => 5;  # seconds, if no Retry-After header
use constant MAX_CONCURRENT_REQUESTS => 3;

my $inflightCount = 0;  # module-level concurrency counter

sub _request {
    my ($class, $method, $path, $params, $cb) = @_;

    # 1. Check rate limit flag
    if ($cache->get(RATE_LIMIT_CACHE_KEY)) {
        main::INFOLOG && $log->info("Rate limit active, rejecting request to $path");
        $cb->(undef, { error => 'rate_limited', code => 429 });
        return;
    }

    # 2. Check concurrency cap
    if ($inflightCount >= MAX_CONCURRENT_REQUESTS) {
        # Simple FIFO: defer to next event loop tick
        # Phase 3 upgrade: push to queue
        Slim::Utils::Timers::setTimer(undef, Time::HiRes::time() + 0.1,
            sub { $class->_request($method, $path, $params, $cb) }
        );
        return;
    }

    $inflightCount++;

    # 3. Get token (from cache, near-instant)
    Plugins::SpotOn::API::TokenManager->getToken($params->{_accountId}, sub {
        my $token = shift;

        unless ($token) {
            $inflightCount--;
            $cb->(undef, { error => 'no_token' });
            return;
        }

        # 4. Make async HTTP call
        Slim::Networking::SimpleAsyncHTTP->new(
            sub { _onSuccess(shift, $cb) },
            sub { _onError(shift, shift, $cb) },
            { timeout => 30, cache => 0 }
        )->$method(
            "https://api.spotify.com/v1/$path",
            'Authorization' => "Bearer $token",
            'Accept'        => 'application/json',
        );
    });
}

sub _onSuccess {
    my ($response, $cb) = @_;
    $inflightCount--;

    my $result = eval { from_json($response->content) };
    if ($@) {
        $log->error("JSON parse error: $@");
        $cb->(undef, { error => 'parse_error' });
        return;
    }

    $cb->($result);
}

sub _onError {
    my ($http, $error, $cb) = @_;
    $inflightCount--;

    my $code = $http->response ? $http->response->code : 0;

    if ($code == 429) {
        my $retryAfter = $http->response->header('Retry-After') || RATE_LIMIT_DEFAULT_BACKOFF;
        $cache->set(RATE_LIMIT_CACHE_KEY, 1, $retryAfter);
        main::INFOLOG && $log->info("429 received, backing off for ${retryAfter}s");
        $cb->(undef, { error => 'rate_limited', code => 429 });
        return;
    }

    $log->error("HTTP error $code for request: $error");
    $cb->(undef, { error => $error, code => $code });
}
```

### Pattern 5: OPML Rate-Limit Hint (D-12)

```perl
# In Plugin.pm::handleFeed — insert throttle hint as first item when rate-limited
sub handleFeed {
    my ($client, $callback, $args) = @_;

    my @items;

    # Rate limit hint (D-12)
    if ($cache->get(Plugins::SpotOn::API::Client::RATE_LIMIT_CACHE_KEY)) {
        push @items, {
            name => cstring($client, 'PLUGIN_SPOTON_RATE_LIMIT_HINT'),
            type => 'textarea',
        };
    }

    # Account switcher (D-05) — first real item
    my $activeAccount = Plugins::SpotOn::Settings->getActiveAccountName($client);
    if ($activeAccount) {
        push @items, {
            name    => cstring($client, 'PLUGIN_SPOTON_ACTIVE_ACCOUNT', $activeAccount),
            url     => \&_accountSwitcherFeed,
            type    => 'link',
        };
    }

    # ... rest of menu items (Phase 3)

    $callback->({ items => \@items });
}
```

### Pattern 6: Credential Directory Setup (AUTH-04)

```perl
# Source: Spotty AccountHelper.pm cacheFolder pattern + chmod enforcement
use File::Path qw(mkpath);
use File::Spec::Functions qw(catdir catfile);

sub _cacheDir {
    my ($class, $accountId) = @_;
    my $base = catdir(preferences('server')->get('cachedir'), 'spoton', $accountId);
    return $base;
}

sub _setPermissions {
    my ($class, $accountId) = @_;
    my $dir  = $class->_cacheDir($accountId);
    my $cred = catfile($dir, 'credentials.json');

    chmod 0700, $dir  if -d $dir;
    chmod 0600, $cred if -f $cred;

    main::INFOLOG && $log->info("Set permissions: $dir (0700), $cred (0600)");
}
```

### Anti-Patterns to Avoid

- **Storing credentials in LMS Prefs:** LMS Prefs are plaintext YAML — credentials would be world-readable. Always use the cache directory with chmod 600/700.
- **Blocking HTTP in the main loop:** Never use `LWP::UserAgent` or `Slim::Networking::SimpleSyncHTTP` outside of scanner/importer context. SimpleAsyncHTTP only.
- **Using `--get-token` with a global client ID hardcoded in Perl:** The client ID belongs in the binary (or a config file). Embedding in Perl makes it visible in plugin source. Follow Spotty's pattern: binary has a bundled `client_id.txt`.
- **Restarting the token refresh timer after each token use:** Start one timer at plugin init, re-arm it after each refresh cycle. Never set a timer per-request.
- **Blocking on binary stdout:** `--get-token` is a short-lived call but it does make a network round-trip to Spotify's Mercury endpoint. If it hangs (network outage), it will freeze LMS. Consider a timeout wrapper or the `alarm()` pattern.
- **Not invalidating the `inflightCount` counter on error:** If `_onError` is called without decrementing `$inflightCount`, the concurrency cap will fill up permanently. Both success and error paths MUST decrement.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTTP caching with TTL | Custom hash with expiry timestamps | `Slim::Utils::Cache` | Cache-Control parsing, disk persistence, cross-restart TTL — all built in |
| Async HTTP with timeout | `LWP::UserAgent` in a thread | `SimpleAsyncHTTP` with `{timeout=>30}` | LMS is single-threaded; threads are not supported |
| Timer-based scheduling | `sleep()` or `alarm()` loops | `Slim::Utils::Timers::setTimer` | Integrates with LMS event loop; does not block audio |
| JSON parsing | Regex on Spotify API responses | `JSON::XS::VersionOneAndTwo` | Handles edge cases, Unicode, malformed JSON with proper errors |
| Credential storage with permissions | Custom file writing | librespot-native credentials.json + `chmod()` | Binary expects this exact format; chmod is one Perl call |

**Key insight:** LMS is a single-threaded event loop built on EV/AnyEvent. Any blocking operation (sleep, LWP, synchronous file I/O on slow mounts) freezes all audio streaming on the server. Every external operation must be non-blocking or extremely short.

---

## Common Pitfalls

### Pitfall 1: Binary `--get-token` Hanging on Network Outage
**What goes wrong:** `spoton --get-token` makes a Keymaster Mercury request over the Spotify AP connection. If the Spotify AP is unreachable, the binary may hang indefinitely, blocking LMS.
**Why it happens:** Backtick (``) in Perl does not have a built-in timeout. The binary itself may not timeout promptly.
**How to avoid:** Wrap the backtick call with a `local $SIG{ALRM}` handler and `alarm(10)` to kill the binary after 10 seconds. Alternatively use `Proc::Background` (available in LMS) for non-blocking spawn with kill support.
**Warning signs:** LMS becomes unresponsive for ~30 seconds periodically, particularly on unreliable network connections.

### Pitfall 2: `inflightCount` Leaking on Callback Never Invoked
**What goes wrong:** `SimpleAsyncHTTP` callbacks are not guaranteed to fire on LMS shutdown or plugin reload. If `_onSuccess` or `_onError` never fires, `$inflightCount` is never decremented, permanently blocking new requests after 3 timeouts.
**Why it happens:** Module-level `$inflightCount` variable persists across requests but cleanup on LMS shutdown is not guaranteed.
**How to avoid:** Set a `setTimer` watchdog per request that decrements the counter if the callback hasn't fired within 35 seconds (5s longer than the HTTP timeout). Or reset `$inflightCount = 0` in `initPlugin`.
**Warning signs:** After an LMS restart or a period of API errors, all requests silently fail (429-like behavior but no 429 was received).

### Pitfall 3: LMS Prefs Serialization for Account Lists
**What goes wrong:** Storing the accounts as a hash in LMS Prefs (`$prefs->set('accounts', \%accounts)`) works on set but can corrupt on concurrent writes if two processes (plugin + scanner) write simultaneously.
**Why it happens:** LMS Prefs files are written as YAML; concurrent writes are not transactional.
**How to avoid:** Use the filesystem (one credentials.json per account directory) as the canonical store, as Spotty does. LMS Prefs stores only the active account ID per client, not the full credentials.
**Warning signs:** Accounts disappearing from the settings page after a scan.

### Pitfall 4: Spotify Username ≠ Display Name ≠ Account ID
**What goes wrong:** Using the Spotify username as the display name in the UI, or as the account ID/cache key.
**Why it happens:** Spotty uses MD5(username) as the cache directory name; display names come from `GET /me` (`display_name` field).
**How to avoid:** Account ID = `substr(md5_hex($username), 0, 8)`. Display name is fetched via `GET /me` and stored separately in Prefs (`displayNames` hash).
**Warning signs:** Accounts with email-format usernames or unicode characters producing filesystem errors or duplicate entries.

### Pitfall 5: OPML `nextWindow => 'refreshOrigin'` Timing
**What goes wrong:** After account switch, the menu refreshes before the token for the new account has been fetched, showing an auth error briefly.
**Why it happens:** `nextWindow => 'refreshOrigin'` triggers an immediate menu reload; token acquisition is async.
**How to avoid:** Trigger token pre-fetch synchronously (blocking is acceptable in the account-switch path since it's user-initiated) before returning the menu refresh response. Or show a loading indicator item.
**Warning signs:** "Spotify API Error" flash on account switch, disappearing on second navigation.

### Pitfall 6: `--check` Capability `lms-auth` Flag Confusion
**What goes wrong:** The `lms-auth` capability in the `--check` JSON manifest is about whether the binary supports the `--lms-auth` flag (passing LMS credentials to the binary for the binary to call LMS's JSON-RPC API). It is NOT about Spotify authentication.
**Why it happens:** The name is misleading. `--lms-auth` enables the binary to call back to LMS (e.g., for Connect event dispatching). In Phase 2, this is `false` in our binary.
**How to avoid:** Treat `lms-auth` capability as a Connect/Phase 5 feature flag. Phase 2 binary sets `lms-auth: false` in the check manifest.

---

## D-01 Recommendation: Credential Input Method

**Recommendation: Username/Password via `--authenticate` flag (login5 protocol)**

**Evidence:**
- Herger's spotty supports `--username`, `--password`, `--authenticate` flags [VERIFIED: spotty src/main.rs]
- The `--authenticate` flow stores credentials.json in librespot-native format (login5 tokens, not raw passwords) [VERIFIED: spotty source]
- The `--enable-oauth` alternative (browser redirect) is more complex UI and requires the binary to run an HTTP server, which conflicts with the short-lived spawn model (D-03)
- Spotty-NG's community confirms login5 username/password still works in Herger's fork as of 2024 [MEDIUM confidence]
- Spotify disabled "Spotify password" for OAuth2 apps but the login5 binary protocol (used by the official client) still accepts passwords on the AP connection [ASSUMED: needs verification against current spotty binary behavior]

**Implementation path for the librespot-spoton binary (Rust, Phase 2 scope):**
The binary currently only implements `--check`. Phase 2 extends it with:
1. `--authenticate` flag: accepts `--username`, `--password`, opens a session via login5, writes credentials.json
2. `--get-token` flag: reads cached credentials.json, opens a session, fetches token via Keymaster Mercury, prints JSON to stdout, exits

Both operations are short-lived and block the plugin call for ~200-500ms, which is acceptable.

**Fallback:** If login5 username/password is found to be broken during Phase 2 implementation, add `--enable-oauth` with stdin-mode (the OAuth token can be pasted into a text field in LMS settings — headless alternative to browser redirect).

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Spotify username/password HTTP auth | login5 binary protocol (stored credentials.json) | Aug 2024 — Spotify | Must use librespot binary for auth, not direct Spotify HTTP |
| Batch API endpoints (`GET /tracks?ids=...`) | Individual requests only (dev mode) | Feb 2026 | All fetches are single-resource; API-05 is effectively no-op in dev mode |
| `GET /browse/featured-playlists` | `GET /me/playlists` filtered | Deprecated (still works) | Phase 3: use user playlists for "Home" feed |
| Mercury Keymaster (hm://) token | Still Mercury/Keymaster in librespot | Current | The `--get-token` approach remains valid with current librespot |
| Spotty iconCode (client ID from LMS community proxy) | Required in SpotOn binary | Ongoing | Client ID must be bundled in the binary per Spotty pattern |

**Deprecated/outdated:**
- Spotify username/password as direct HTTP credentials: broken since Aug 2024
- `GET /browse/categories`: removed in dev mode Feb 2026
- `GET /recommendations`: removed Nov 2024

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | login5 username/password authentication still works in Herger's spotty fork for new SpotOn binary | D-01 Recommendation, Pattern 2 | If broken: must use OAuth/browser redirect flow for credential acquisition; more complex Settings UI |
| A2 | Max 3 concurrent Spotify API requests is safe (no 429 risk) | Pattern 4 (API-02) | If too aggressive: increase 429 frequency; lower to 1-2. If too conservative: slower Browse pagination in Phase 3 |
| A3 | `--get-token` binary spawn takes ~200-500ms (acceptable blocking) | Pattern 1 | If slower (e.g., 5+ seconds on slow networks): must switch to async spawn via Proc::Background |
| A4 | Spotify's Keymaster Mercury token is still valid for the Spotify Web API | AUTH-01, Standard Stack | If Keymaster is deprecated by Spotify: must use OAuth2 refresh token flow instead of binary spawn |

---

## Open Questions (RESOLVED)

1. **Client ID source for `--get-token`** (RESOLVED)
   - What we know: Herger's spotty bundles a `client_id.txt` in the binary. SpotOn needs its own.
   - Resolution: Use the Spotty community client ID (`65b708073fc0480ea92a077233ca87bd`) as a constant in TokenManager.pm for initial development. This is the same client ID used by the established Spotty plugin for the LMS ecosystem. If Herger objects or Spotify restricts it, register a new SpotOn app in the Spotify Developer Console and update the constant. The client ID is passed via `--client-id` flag to the binary, not hardcoded in Rust.

2. **librespot-spoton binary scope in Phase 2** (RESOLVED)
   - What we know: Current binary only implements `--check`. Phase 2 needs `--authenticate` and `--get-token`.
   - Resolution: Extend the existing librespot-spoton Rust binary (Plans 02-01). Add `--authenticate` and `--get-token` subcommands backed by librespot-core as a dependency in Cargo.toml. Do NOT use Herger's fork as base -- maintain independent binary with librespot-core crate dependency.

3. **Login5 password auth broken?** (RESOLVED)
   - What we know: Spotify disabled username/password for many OAuth flows in Aug 2024. login5 binary protocol may still work.
   - Resolution: Plan 02-05 Task 3 is a blocking human checkpoint that tests `--authenticate` with real Spotify credentials. If login5 fails, the checkpoint has an explicit "login5-failed" signal that triggers a fallback discussion for OAuth browser redirect. The risk is accepted and the fallback path is documented. Confidence: LOW -- login5 may not work, but the binary and plugin architecture support pivoting to OAuth without structural changes.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Perl | All plugin modules | ✓ | v5.38.2 | — |
| prove (Test::Harness) | Phase 2 tests | ✓ | v3.44 | — |
| cargo / Rust | Binary extension | — | Not checked | Pre-built binary in Bin/ |
| librespot binary (built) | AUTH-01, AUTH-02 | Partially (--check only) | Phase 1 stub | Implement --get-token, --authenticate in Phase 2 |

**Missing dependencies with no fallback:**
- `--get-token` and `--authenticate` binary flags: must be implemented in the Rust binary as part of Phase 2 execution. No Perl-only fallback exists.

**Note:** cargo/Rust availability was not probed. The binary extension (adding `--get-token`, `--authenticate`) is part of the Phase 2 deliverables. If cross-compilation toolchain is unavailable, pre-built binaries can be distributed and the Perl plugin tested against them.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Test::More (Perl built-in via prove) |
| Config file | none — test files in `t/` |
| Quick run command | `prove -l t/07_token_manager.t t/08_api_client.t` |
| Full suite command | `prove -l t/` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| AUTH-01 | Binary `--get-token` outputs JSON with accessToken key | unit (binary mock) | `prove -l t/07_token_manager.t` | ❌ Wave 0 |
| AUTH-02 | Token cached with TTL = expires_in - 300s | unit | `prove -l t/07_token_manager.t` | ❌ Wave 0 |
| AUTH-03 | Timer re-arms after each refresh cycle | unit | `prove -l t/07_token_manager.t` | ❌ Wave 0 |
| AUTH-04 | credentials.json has mode 0600, parent dir 0700 | unit | `prove -l t/08_api_client.t` | ❌ Wave 0 |
| AUTH-05 | Multiple accounts stored in separate cache subdirs | unit | `prove -l t/09_settings.t` | ❌ Wave 0 |
| AUTH-06 | Account switch updates active account pref | unit | `prove -l t/09_settings.t` | ❌ Wave 0 |
| API-01 | All outbound HTTP calls go via Client.pm | syntax/grep | `prove -l t/05_perl_syntax.t` | ✅ (extend) |
| API-02 | 50 rapid calls produce no 429 (mock) | integration-ish | `prove -l t/08_api_client.t` | ❌ Wave 0 |
| API-03 | Responses cached with correct TTLs | unit | `prove -l t/08_api_client.t` | ❌ Wave 0 |
| API-04 | 429 response sets rate-limit cache key with Retry-After TTL | unit | `prove -l t/08_api_client.t` | ❌ Wave 0 |
| API-06 | No blocking HTTP calls in non-scanner context | grep/linting | `prove -l t/05_perl_syntax.t` | ✅ (extend) |

### Sampling Rate
- **Per task commit:** `prove -l t/05_perl_syntax.t` (syntax check only, fast)
- **Per wave merge:** `prove -l t/`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `t/07_token_manager.t` — covers AUTH-01 through AUTH-03
- [ ] `t/08_api_client.t` — covers API-02 through API-04, API-06
- [ ] `t/09_settings.t` — covers AUTH-04, AUTH-05, AUTH-06

*(Existing `t/05_perl_syntax.t` covers Perl syntax and can be extended with no-LWP grep check for API-06)*

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | Yes | login5 via binary (no passwords stored in LMS Prefs) |
| V3 Session Management | Yes | Token TTL with pre-expiry refresh; per-account isolation |
| V4 Access Control | No | Single LMS admin only |
| V5 Input Validation | Yes | Shell-escape all user-supplied binary paths and credentials before backtick spawn |
| V6 Cryptography | No | Token transport via HTTPS (Spotify API); binary handles TLS |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Shell injection via user-supplied username/password in `--authenticate` call | Tampering | Single-quote + `'\''` escaping before backtick; consider `IPC::Open3` or `Proc::Background` for zero-injection-risk |
| Credential file world-readable | Information Disclosure | `chmod 0600` credentials.json, `chmod 0700` parent dir immediately after binary write |
| Rate limit token written to logs | Information Disclosure | Never log the `accessToken` value; log only expiry time and account ID |
| 429 Retry-After value injection (malicious HTTP response) | Elevation of Privilege | Cap Retry-After at 300 seconds maximum before caching as TTL |

---

## Sources

### Primary (HIGH confidence)
- Herger's spotty binary source — `src/main.rs` — `--get-token`, `--authenticate`, `--client-id` flags verified
  `https://raw.githubusercontent.com/michaelherger/spotty/master/src/main.rs`
- Spotty-Plugin AccountHelper.pm — multi-account pattern, credential directory structure, MD5 account ID
  `https://raw.githubusercontent.com/michaelherger/Spotty-Plugin/master/AccountHelper.pm`
- Spotty-Plugin API/Token.pm — token caching, pre-expiry buffer, callback queuing
  `https://raw.githubusercontent.com/michaelherger/Spotty-Plugin/master/API/Token.pm`
- Spotty-Plugin API.pm — rate limiting via cache key, 429 handler, `getToken` delegation pattern
  `https://raw.githubusercontent.com/michaelherger/Spotty-Plugin/master/API.pm`
- librespot core/src/token.rs — Keymaster Mercury token URL confirmed: `hm://keymaster/token/authenticated`
  `https://raw.githubusercontent.com/librespot-org/librespot/master/core/src/token.rs`
- Slim::Utils::Timers — `setTimer`, `killTimers` API
  `https://raw.githubusercontent.com/LMS-Community/slimserver/master/Slim/Utils/Timers.pm`
- Slim::Networking::SimpleAsyncHTTP — constructor, `_params`, timeout pattern
  `https://raw.githubusercontent.com/LMS-Community/slimserver/master/Slim/Networking/SimpleAsyncHTTP.pm`
- Qobuz plugin API.pm — `_get()` pattern: check token, build URL, `SimpleAsyncHTTP`
  `https://raw.githubusercontent.com/LMS-Community/plugin-Qobuz/master/API.pm`
- Phase 1 codebase — `Helper.pm` (binary spawn with backtick), `Plugin.pm` (prefs init), `Settings.pm` (CSRF, base handler)

### Secondary (MEDIUM confidence)
- Spotty-Plugin Settings/Auth.pm — OAuth browser redirect pattern (alternative credential flow)
- librespot oauth/src/lib.rs — PKCE OAuth flow implementation (fallback if login5 fails)
- WebSearch: Spotify disabled username/password in Aug 2024 (multiple librespot issue reports confirm)

### Tertiary (LOW confidence — validate during execution)
- A1: login5 username/password still works in Herger's spotty fork
- A3: `--get-token` spawn duration is acceptable for blocking calls

---

## Metadata

**Confidence breakdown:**
- Standard Stack: HIGH — all LMS bundled, verified from source
- Authentication flow (binary `--get-token`): HIGH — verified from spotty src/main.rs
- Credential acquisition method (D-01): MEDIUM — login5 password auth may be broken (Aug 2024 Spotify changes)
- Rate limiting (D-09, D-10): MEDIUM — Spotify's exact rate limit window undocumented; 3-concurrent recommendation based on Spotty-NG operational experience
- Architecture patterns: HIGH — verified from Spotty and Qobuz reference implementations

**Research date:** 2026-05-27
**Valid until:** 2026-08-27 (stable LMS APIs; re-verify if Spotify makes further API changes)
