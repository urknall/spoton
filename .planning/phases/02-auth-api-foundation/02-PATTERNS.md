# Phase 2: Auth + API Foundation - Pattern Map

**Mapped:** 2026-05-27
**Files analyzed:** 8 new/modified files
**Analogs found:** 8 / 8

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `Plugins/SpotOn/API/TokenManager.pm` | service | request-response (binary spawn + stdout parse) | `Plugins/SpotOn/Helper.pm` | role-match (same backtick/stdout pattern) |
| `Plugins/SpotOn/API/Client.pm` | service | request-response (async HTTP) | `Plugins/SpotOn/Helper.pm` + `Plugins/SpotOn/Plugin.pm` | role-match |
| `Plugins/SpotOn/Plugin.pm` | plugin/controller | event-driven (timer + OPML) | `Plugins/SpotOn/Plugin.pm` (self, modify) | exact (modify existing) |
| `Plugins/SpotOn/Settings.pm` | controller | request-response (HTTP form) | `Plugins/SpotOn/Settings.pm` (self, modify) | exact (modify existing) |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | template | request-response | `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (self, modify) | exact (modify existing) |
| `Plugins/SpotOn/strings.txt` | config | — | `Plugins/SpotOn/strings.txt` (self, modify) | exact (modify existing) |
| `t/07_token_manager.t` | test | — | `t/06_binary_check.t` + `t/05_perl_syntax.t` | role-match |
| `t/08_api_client.t` | test | — | `t/05_perl_syntax.t` | role-match |
| `t/09_settings.t` | test | — | `t/05_perl_syntax.t` | role-match |
| `librespot-spoton/src/main.rs` | binary/CLI | request-response (modify) | `librespot-spoton/src/main.rs` (self, modify) | exact (modify existing) |

---

## Pattern Assignments

### `Plugins/SpotOn/API/TokenManager.pm` (service, binary-spawn + stdout-parse)

**Analog:** `Plugins/SpotOn/Helper.pm`

**Imports pattern** (lines 1-17 of Helper.pm):
```perl
package Plugins::SpotOn::API::TokenManager;

use strict;
use warnings;

use File::Path qw(mkpath);
use File::Spec::Functions qw(catdir catfile);
use Digest::MD5 qw(md5_hex);
use JSON::XS::VersionOneAndTwo;

use Slim::Utils::Cache;
use Slim::Utils::Log;
use Slim::Utils::Prefs;
use Slim::Utils::Timers;
use Time::HiRes;

my $log   = logger('plugin.spoton');
my $prefs = preferences('plugin.spoton');
my $cache = Slim::Utils::Cache->new();
```

**Binary spawn (backtick) pattern** — copy from `Plugins/SpotOn/Helper.pm` lines 67-69:
```perl
# Shell-safe quoting to prevent command injection from user-supplied binary paths
(my $safe = $candidate) =~ s/'/'\\''/g;
my $checkCmd = sprintf("'%s' -n 'SpotOn' --check", $safe);
$$check = `$checkCmd 2>&1`;
```

Apply the same quoting + backtick idiom for `--get-token` and `--authenticate` spawns:
```perl
(my $safeBin   = $binary)   =~ s/'/'\\''/g;
(my $safeCache = $cacheDir) =~ s/'/'\\''/g;
my $cmd = sprintf("'%s' -n 'SpotOn' --cache '%s' --get-token 2>/dev/null",
    $safeBin, $safeCache);
my $output = `$cmd`;
```

**JSON stdout parse pattern** — copy from `Plugins/SpotOn/Helper.pm` lines 86-88:
```perl
if ( $$check =~ /\n(.*)/s ) {
    $helperCapabilities = eval { from_json($1) } || {};
}
```

For `--get-token`: the entire stdout is the JSON token object.
```perl
if ($output && $output =~ /^\{/) {
    my $token = eval { from_json($output) };
    if (!$@ && $token && $token->{accessToken}) {
        # proceed
    }
}
```

**Token cache write pattern** (derived from RESEARCH.md Pattern 3):
```perl
use constant TOKEN_EXPIRY_BUFFER => 300;   # refresh 5 min before expiry
use constant TOKEN_REFRESH_TIMER => 45 * 60;

sub _cacheToken {
    my ($class, $accountId, $tokenData) = @_;
    my $expiresIn = $tokenData->{expiresIn} || 3600;
    my $ttl       = $expiresIn > TOKEN_EXPIRY_BUFFER
                    ? $expiresIn - TOKEN_EXPIRY_BUFFER
                    : $expiresIn;
    $cache->set("spoton_token_$accountId", $tokenData->{accessToken}, $ttl);
    main::INFOLOG && $log->info("Token cached for $accountId, TTL ${ttl}s");
}
```

**Credential directory pattern** (RESEARCH.md Pattern 6):
```perl
sub _cacheDir {
    my ($class, $accountId) = @_;
    return catdir(preferences('server')->get('cachedir'), 'spoton', $accountId);
}

sub _setPermissions {
    my ($class, $accountId) = @_;
    my $dir  = $class->_cacheDir($accountId);
    my $cred = catfile($dir, 'credentials.json');
    chmod 0700, $dir  if -d $dir;
    chmod 0600, $cred if -f $cred;
}
```

**Alarm wrapper for blocking backtick** (Pitfall 1 from RESEARCH.md):
```perl
# Wrap every backtick binary call with alarm() to prevent LMS freeze on network outage
local $SIG{ALRM} = sub { die "timeout\n" };
eval {
    alarm(10);
    $output = `$cmd`;
    alarm(0);
};
alarm(0);
if ($@) {
    $log->error("Binary spawn timed out: $cmd");
    $cb->(undef);
    return;
}
```

---

### `Plugins/SpotOn/API/Client.pm` (service, async HTTP)

**Analog:** `Plugins/SpotOn/Helper.pm` (binary/prefs init pattern) + `Plugins/SpotOn/Plugin.pm` (SimpleAsyncHTTP pattern)

**Imports pattern:**
```perl
package Plugins::SpotOn::API::Client;

use strict;
use warnings;

use JSON::XS::VersionOneAndTwo;

use Slim::Networking::SimpleAsyncHTTP;
use Slim::Utils::Cache;
use Slim::Utils::Log;
use Slim::Utils::Prefs;
use Slim::Utils::Timers;
use Time::HiRes;

my $log   = logger('plugin.spoton');
my $prefs = preferences('plugin.spoton');
my $cache = Slim::Utils::Cache->new();
```

**Module-level state** (inflightCount must be reset on initPlugin — Pitfall 2 from RESEARCH.md):
```perl
use constant RATE_LIMIT_CACHE_KEY    => 'spoton_rate_limit_exceeded';
use constant RATE_LIMIT_DEFAULT_BACKOFF => 5;
use constant MAX_CONCURRENT_REQUESTS => 3;

my $inflightCount = 0;  # Reset to 0 in initPlugin via Client->reset()
```

**Core async HTTP pattern** — modelled on LMS SimpleAsyncHTTP idiom from RESEARCH.md Pattern 4:
```perl
sub _request {
    my ($class, $method, $path, $params, $cb) = @_;

    # 1. Rate limit flag check (cache key set on 429)
    if ($cache->get(RATE_LIMIT_CACHE_KEY)) {
        $cb->(undef, { error => 'rate_limited', code => 429 });
        return;
    }

    # 2. Concurrency cap — FIFO defer via timer (Phase 3 will add a proper queue)
    if ($inflightCount >= MAX_CONCURRENT_REQUESTS) {
        Slim::Utils::Timers::setTimer(undef, Time::HiRes::time() + 0.1,
            sub { $class->_request($method, $path, $params, $cb) });
        return;
    }

    $inflightCount++;

    # 3. Token injection (near-instant cache hit 99%)
    Plugins::SpotOn::API::TokenManager->getToken($params->{_accountId}, sub {
        my $token = shift;
        unless ($token) {
            $inflightCount--;
            $cb->(undef, { error => 'no_token' });
            return;
        }

        # 4. Async HTTP call — copy SimpleAsyncHTTP constructor pattern from RESEARCH.md
        Slim::Networking::SimpleAsyncHTTP->new(
            sub { $class->_onSuccess(shift, $cb) },
            sub { $class->_onError(shift, shift, $cb) },
            { timeout => 30, cache => 0 }
        )->$method(
            "https://api.spotify.com/v1/$path",
            'Authorization' => "Bearer $token",
            'Accept'        => 'application/json',
        );
    });
}
```

**Error handling pattern** — MUST decrement $inflightCount in all paths (Pitfall 2):
```perl
sub _onSuccess {
    my ($class, $http, $cb) = @_;
    $inflightCount--;
    my $result = eval { from_json($http->content) };
    if ($@) {
        $log->error("JSON parse error: $@");
        $cb->(undef, { error => 'parse_error' });
        return;
    }
    $cb->($result);
}

sub _onError {
    my ($class, $http, $error, $cb) = @_;
    $inflightCount--;
    my $code = $http->response ? $http->response->code : 0;
    if ($code == 429) {
        my $retryAfter = $http->response->header('Retry-After') || RATE_LIMIT_DEFAULT_BACKOFF;
        $retryAfter = 300 if $retryAfter > 300;  # cap at 300s (security: Pitfall 4 in RESEARCH.md)
        $cache->set(RATE_LIMIT_CACHE_KEY, 1, $retryAfter);
        $cb->(undef, { error => 'rate_limited', code => 429 });
        return;
    }
    $log->error("HTTP $code: $error");
    $cb->(undef, { error => $error, code => $code });
}
```

**Response caching pattern** (API-03, D-16):
```perl
# Before _request: check cache
my $cacheKey = "spoton_$path";
if (my $cached = $cache->get($cacheKey)) {
    $cb->($cached);
    return;
}

# In _onSuccess, after parse:
# TTL selection by path prefix (from CLAUDE.md):
# me/tracks, me/albums → 60s
# metadata (tracks/{id}, albums/{id}, artists/{id}) → 3600s
# playlists, browse → 300s
my $ttl = _cacheTTL($path);
$cache->set($cacheKey, $result, $ttl) if $ttl;
```

---

### `Plugins/SpotOn/Plugin.pm` (modify existing — add timer + account switcher)

**Analog:** `Plugins/SpotOn/Plugin.pm` (self)

**Current imports** (lines 1-14) — add `Slim::Utils::Timers` and `Time::HiRes`:
```perl
use Slim::Utils::Timers;
use Time::HiRes;
```

**Timer init pattern** — add to `initPlugin` after `Helper->init()` (lines 37-38):
```perl
# Start token refresh timer (D-04, AUTH-02)
# Timer fires every 45 min; TokenManager->refreshAllTokens re-arms it
Slim::Utils::Timers::killTimers($class, \&_refreshAllTokens);
Slim::Utils::Timers::setTimer(
    $class,
    Time::HiRes::time() + TOKEN_REFRESH_TIMER,
    \&_refreshAllTokens
);

# Reset API client inflight counter (Pitfall 2 in RESEARCH.md)
Plugins::SpotOn::API::Client->reset();
```

**Prefs init** — extend existing `$prefs->init({...})` call (lines 31-34):
```perl
$prefs->init({
    bitrate       => 320,
    binary        => '',
    accounts      => {},     # hash: accountId => { username => ..., displayName => ... }
    activeAccount => '',     # default active account ID (global fallback)
});
```

**handleFeed modification** — extend existing sub (lines 61-81) to prepend rate-limit hint and account switcher before the Phase 1 placeholder:
```perl
sub handleFeed {
    my ($client, $callback, $args) = @_;

    # (existing binary check at line 64 stays unchanged)

    my @items;

    # Rate limit hint (D-12)
    if ($cache->get(Plugins::SpotOn::API::Client::RATE_LIMIT_CACHE_KEY)) {
        push @items, {
            name => cstring($client, 'PLUGIN_SPOTON_RATE_LIMIT_HINT'),
            type => 'textarea',
        };
    }

    # Account switcher (D-05, AUTH-06)
    my $activeId   = $prefs->client($client)->get('activeAccount')
                  || $prefs->get('activeAccount');
    my $accounts   = $prefs->get('accounts') || {};
    my $activeName = $activeId && $accounts->{$activeId}
                   ? $accounts->{$activeId}{displayName}
                   : undef;
    if ($activeName) {
        push @items, {
            name => cstring($client, 'PLUGIN_SPOTON_ACTIVE_ACCOUNT', $activeName),
            url  => \&_accountSwitcherFeed,
            type => 'link',
        };
    }

    # Phase 2: No Browse/Search/Library items yet (Phase 3)
    push @items, {
        name => cstring($client, 'PLUGIN_SPOTON_NAME'),
        type => 'textarea',
    } unless @items;

    $callback->({ items => \@items });
}
```

---

### `Plugins/SpotOn/Settings.pm` (modify existing — add account CRUD)

**Analog:** `Plugins/SpotOn/Settings.pm` (self)

**Current handler pattern** (lines 32-49) — extend for account add/remove:
```perl
sub handler {
    my ($class, $client, $paramRef, $callback, $httpClient, $response) = @_;

    # (existing binary status lines 35-39 stay)

    if ($paramRef->{saveSettings}) {
        # (existing bitrate validation lines 43-46 stay)

        # Account management (D-07)
        if ($paramRef->{addAccount}) {
            my $username = $paramRef->{username} // '';
            my $password = $paramRef->{password} // '';
            $username =~ s/^\s+|\s+$//g;  # trim

            if ($username && $password) {
                Plugins::SpotOn::API::TokenManager->addAccount($username, $password, sub {
                    my ($accountId, $err) = @_;
                    if ($err) {
                        $paramRef->{authError} = $err;
                    } else {
                        $prefs->set('activeAccount', $accountId)
                            unless $prefs->get('activeAccount');
                    }
                    return $class->SUPER::handler($client, $paramRef, $callback, $httpClient, $response);
                });
                return;  # async — return early, callback will finish
            }
        }

        if (my $removeId = $paramRef->{removeAccount}) {
            Plugins::SpotOn::API::TokenManager->removeAccount($removeId);
        }
    }

    # Pass account data to template
    $paramRef->{accounts}      = $prefs->get('accounts') || {};
    $paramRef->{activeAccount} = $prefs->get('activeAccount') || '';

    return $class->SUPER::handler($client, $paramRef, $callback, $httpClient, $response);
}
```

**prefs sub** — extend to register new prefs (line 29):
```perl
sub prefs {
    return ($prefs, 'bitrate', 'binary', 'activeAccount');
    # Note: 'accounts' hash is managed manually in handler, not via auto-pref saving
}
```

---

### `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (modify existing)

**Analog:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (self)

**Current template pattern** (all 28 lines) — replace the placeholder `[% WRAPPER setting title="PLUGIN_SPOTON_ACCOUNT_SETTINGS" ... %]` block (lines 23-25) with a dynamic account list:

```html
[% WRAPPER setting title="PLUGIN_SPOTON_ACCOUNT_SETTINGS" desc="" %]
    [% IF accounts.keys.size > 0 %]
        <table style="width:100%">
        [% FOREACH id IN accounts.keys %]
            <tr>
                <td>[% accounts.$id.displayName | html %]</td>
                <td>[% accounts.$id.username | html %]</td>
                <td>
                    [% IF id == activeAccount %]
                        <strong>[% 'PLUGIN_SPOTON_ACCOUNT_ACTIVE' | string %]</strong>
                    [% ELSE %]
                        <button name="switchAccount" value="[% id | html %]">
                            [% 'PLUGIN_SPOTON_ACCOUNT_SWITCH' | string %]
                        </button>
                    [% END %]
                </td>
                <td>
                    <button name="removeAccount" value="[% id | html %]">
                        [% 'PLUGIN_SPOTON_ACCOUNT_REMOVE' | string %]
                    </button>
                </td>
            </tr>
        [% END %]
        </table>
    [% ELSE %]
        <p>[% 'PLUGIN_SPOTON_ACCOUNT_NONE' | string %]</p>
    [% END %]

    <hr/>
    <p><strong>[% 'PLUGIN_SPOTON_ACCOUNT_ADD' | string %]</strong></p>
    [% IF authError %]
        <div style="color:red">[% authError | html %]</div>
    [% END %]
    <label>[% 'PLUGIN_SPOTON_ACCOUNT_USERNAME' | string %]
        <input type="text" name="username" value="" autocomplete="username"/>
    </label>
    <label>[% 'PLUGIN_SPOTON_ACCOUNT_PASSWORD' | string %]
        <input type="password" name="password" value="" autocomplete="current-password"/>
    </label>
    <button name="addAccount" value="1">[% 'PLUGIN_SPOTON_ACCOUNT_ADD_BTN' | string %]</button>
[% END %]
```

**Keep unchanged:** The header (`[% PROCESS settings/header.html %]`), binary status block, and bitrate block. Only replace the placeholder account section.

---

### `Plugins/SpotOn/strings.txt` (modify existing — add Phase 2 strings)

**Analog:** `Plugins/SpotOn/strings.txt` (self)

**Current file pattern** (lines 1-35) — each string block:
```
STRING_KEY
	DE	German translation
	EN	English translation
```

**New strings to append** (follow exact tab-indented format of existing file):
```
PLUGIN_SPOTON_ACTIVE_ACCOUNT
	DE	Aktiv: %s [wechseln]
	EN	Active: %s [switch]

PLUGIN_SPOTON_RATE_LIMIT_HINT
	DE	Spotify-Anfragen gedrosselt — bitte warten
	EN	Spotify requests throttled — please wait

PLUGIN_SPOTON_ACCOUNT_NONE
	DE	Kein Konto konfiguriert
	EN	No account configured

PLUGIN_SPOTON_ACCOUNT_ADD
	DE	Konto hinzufügen
	EN	Add Account

PLUGIN_SPOTON_ACCOUNT_ADD_BTN
	DE	Hinzufügen
	EN	Add

PLUGIN_SPOTON_ACCOUNT_ACTIVE
	DE	Aktiv
	EN	Active

PLUGIN_SPOTON_ACCOUNT_SWITCH
	DE	Wechseln
	EN	Switch

PLUGIN_SPOTON_ACCOUNT_REMOVE
	DE	Entfernen
	EN	Remove

PLUGIN_SPOTON_ACCOUNT_USERNAME
	DE	Benutzername (Spotify)
	EN	Username (Spotify)

PLUGIN_SPOTON_ACCOUNT_PASSWORD
	DE	Passwort
	EN	Password

PLUGIN_SPOTON_AUTH_ERROR
	DE	Anmeldung fehlgeschlagen
	EN	Authentication failed
```

---

### `t/07_token_manager.t` (new test — covers AUTH-01 through AUTH-03)

**Analog:** `t/06_binary_check.t` (binary mock + JSON stdout) + `t/05_perl_syntax.t` (stub pattern)

**Test structure pattern** from `t/06_binary_check.t` (lines 1-10):
```perl
#!/usr/bin/perl
use strict;
use warnings;
use Test::More;
use File::Basename qw(dirname);
use Cwd qw(abs_path);

my $test_dir    = dirname(abs_path($0));
my $project_dir = dirname($test_dir);
```

**Stub directory + write_stub helper** — copy verbatim from `t/05_perl_syntax.t` lines 36-48:
```perl
use File::Temp qw(tempdir);
use File::Path qw(make_path);

my $stub_dir = tempdir(CLEANUP => 1);

sub write_stub {
    my ($dir, $pkg, $code) = @_;
    my @parts = split /::/, $pkg;
    my $file  = pop @parts;
    my $path  = $dir . '/' . join('/', @parts);
    make_path($path) unless -d $path;
    open(my $fh, '>', "$path/$file.pm") or die "Cannot write stub $pkg: $!";
    print $fh $code;
    close($fh);
}
```

**Mock binary approach** for AUTH-01 tests: create a tiny mock binary in tempdir that prints JSON to stdout, then test `TokenManager->refreshToken` against it:
```perl
# Create a mock spoton binary that prints a token JSON and exits
my $mock_bin = "$stub_dir/spoton";
open(my $fh, '>', $mock_bin) or die $!;
print $fh "#!/bin/sh\n";
print $fh q(echo '{"accessToken":"mock_token_abc","expiresIn":3600}') . "\n";
close($fh);
chmod 0755, $mock_bin;
```

**Cache stub pattern** — copy from `t/05_perl_syntax.t` write_stub idiom:
```perl
write_stub($stub_dir, 'Slim::Utils::Cache', <<'END');
package Slim::Utils::Cache;
my %store;
sub new     { bless {}, shift }
sub get     { $store{$_[1]} }
sub set     { $store{$_[1]} = $_[2]; 1 }
sub remove  { delete $store{$_[1]} }
1;
END
```

---

### `t/08_api_client.t` (new test — covers API-02 through API-04, API-06)

**Analog:** `t/05_perl_syntax.t` (stub pattern + perl -c linting approach)

**API-06 grep test** (no blocking HTTP) — extend the existing `t/05_perl_syntax.t` grep approach:
```perl
# API-06: Verify no LWP or SimpleSyncHTTP usage in non-scanner modules
my @api_files = glob("$project_dir/Plugins/SpotOn/API/*.pm");
for my $f (@api_files) {
    open my $fh, '<', $f or die $!;
    my $content = do { local $/; <$fh> };
    unlike($content, qr/LWP::UserAgent|SimpleSyncHTTP/,
        "$f: no blocking HTTP");
}
```

**Rate-limit mock test** (API-02, API-04):
```perl
# Mock cache and simulate 429 → verify RATE_LIMIT_CACHE_KEY is set with correct TTL
# Use the same Cache stub as t/07_token_manager.t
```

---

### `t/09_settings.t` (new test — covers AUTH-04, AUTH-05, AUTH-06)

**Analog:** `t/05_perl_syntax.t` (stub pattern) + `t/06_binary_check.t` (filesystem checks)

**Filesystem permission test** (AUTH-04):
```perl
# AUTH-04: credentials.json chmod check
use File::Temp qw(tempdir);
my $tmpdir = tempdir(CLEANUP => 1);

# Create a fake credentials.json
my $cred_file = "$tmpdir/credentials.json";
open(my $fh, '>', $cred_file) or die $!;
print $fh '{"username":"test"}';
close $fh;

chmod 0700, $tmpdir;
chmod 0600, $cred_file;

my @stat_dir  = stat($tmpdir);
my @stat_cred = stat($cred_file);

is($stat_dir[2]  & 07777, 0700, "cache dir has mode 0700");
is($stat_cred[2] & 07777, 0600, "credentials.json has mode 0600");
```

---

### `librespot-spoton/src/main.rs` (modify existing — add `--authenticate` and `--get-token`)

**Analog:** `librespot-spoton/src/main.rs` (self)

**Current arg parsing pattern** (lines 17-43) — extend the `match args[i].as_str()` block:
```rust
"--authenticate" => { mode = Mode::Authenticate; }
"--get-token"    => { mode = Mode::GetToken; }
"--username" | "-u" => {
    if i + 1 < args.len() { username = args[i + 1].clone(); i += 1; }
}
"--password" | "-p" => {
    if i + 1 < args.len() { password = args[i + 1].clone(); i += 1; }
}
"--cache" | "-c" => {
    if i + 1 < args.len() { cache_dir = args[i + 1].clone(); i += 1; }
}
"--scope" => {
    if i + 1 < args.len() { scope = args[i + 1].clone(); i += 1; }
}
```

**Dispatch pattern** (extend the `if check_mode {` block at line 44):
```rust
enum Mode { Check, Authenticate, GetToken, Connect }

match mode {
    Mode::Check => {
        // existing --check output (lines 47-56 of main.rs, keep unchanged)
        println!("ok spoton v{}", VERSION);
        println!(r#"{{"version":"{}","lms-auth":false,"ogg-direct":false,"passthrough":true}}"#, VERSION);
    }
    Mode::Authenticate => {
        // login5 auth → write credentials.json → print "authorized"
        // Requires librespot-core dependency in Cargo.toml
        run_authenticate(&username, &password, &cache_dir);
    }
    Mode::GetToken => {
        // Read credentials.json → Mercury Keymaster → print JSON token to stdout
        run_get_token(&cache_dir, &scope);
    }
    Mode::Connect => {
        eprintln!("Connect mode not yet implemented (Phase 5)");
        process::exit(1);
    }
}
```

**Cargo.toml additions** (new dependencies for librespot-core):
```toml
[dependencies]
librespot-core     = { version = "0.8", default-features = false }
# NOTE: Do NOT add librespot-oauth in Phase 2. OAuth fallback is out of scope (Phase 2 uses login5 only).
serde_json         = "1"
tokio              = { version = "1", features = ["rt-multi-thread", "macros"] }
```

---

## Shared Patterns

### Module Logger + Prefs Init
**Source:** Every existing `.pm` file (lines 1-17 of each)
**Apply to:** `API/TokenManager.pm`, `API/Client.pm`

```perl
# Every SpotOn module uses this exact pattern — copy verbatim
my $log   = logger('plugin.spoton');
my $prefs = preferences('plugin.spoton');
```

Note: `Plugin.pm` uses `Slim::Utils::Log->addLogCategory(...)` — that is only for the plugin entry point. All other modules use `logger('plugin.spoton')` directly (no addLogCategory).

### Shell-Safe Quoting for Binary Spawns
**Source:** `Plugins/SpotOn/Helper.pm` lines 67-68
**Apply to:** `API/TokenManager.pm` — all backtick binary invocations

```perl
(my $safe = $value) =~ s/'/'\\''/g;
# Then embed in single-quoted shell argument: sprintf("'%s'", $safe)
```

This is the established project pattern. Never use double-quoted shell interpolation for user-supplied values.

### INFOLOG Guard for Debug Logging
**Source:** `Plugins/SpotOn/Helper.pm` lines 138-139
**Apply to:** `API/TokenManager.pm`, `API/Client.pm`, `Plugin.pm`

```perl
main::INFOLOG && $log->info("...");
```

Only `$log->error()` and `$log->warn()` are called unconditionally. Info-level messages must be guarded. This avoids string construction overhead in production.

### LMS Timer Pattern
**Source:** RESEARCH.md Pattern 3; `Slim::Utils::Timers` from LMS bundle
**Apply to:** `Plugin.pm` (timer init), `API/Client.pm` (concurrency defer)

```perl
# Kill existing timer before setting new one (prevents duplicate timers on plugin reload)
Slim::Utils::Timers::killTimers($object, \&callback_sub);
Slim::Utils::Timers::setTimer($object, Time::HiRes::time() + $interval, \&callback_sub);
```

### Slim::Utils::Cache Usage
**Source:** RESEARCH.md §Standard Stack; established in Helper.pm via `Slim::Utils::Cache`
**Apply to:** `API/TokenManager.pm` (token cache), `API/Client.pm` (rate-limit flag, response cache)

```perl
# Module-level singleton — one instance per module is sufficient
my $cache = Slim::Utils::Cache->new();

# Set with TTL:
$cache->set($key, $value, $ttl_seconds);

# Get (returns undef on miss/expiry):
my $val = $cache->get($key);

# Remove:
$cache->remove($key);
```

### Test Stub Infrastructure
**Source:** `t/05_perl_syntax.t` lines 36-48 (`write_stub` helper + tempdir)
**Apply to:** `t/07_token_manager.t`, `t/08_api_client.t`, `t/09_settings.t`

Copy the `write_stub()` function and all existing stubs verbatim. Each new test file needs: `Slim::Utils::Log`, `Slim::Utils::Prefs`, `Slim::Utils::Cache`, `JSON::XS::VersionOneAndTwo`, `Slim::Networking::SimpleAsyncHTTP`, and `Slim::Utils::Timers` stubs. Reuse the `$main_constants` perl `-e` pattern for `main::INFOLOG` and friends.

---

## No Analog Found

All Phase 2 files have sufficiently close analogs in the existing codebase. No file lacks a pattern reference.

---

## Metadata

**Analog search scope:** `/home/sti/spoton/Plugins/SpotOn/`, `/home/sti/spoton/t/`, `/home/sti/spoton/librespot-spoton/src/`
**Files scanned:** 10 (4 `.pm`, 6 `.t` / `.html` / `.txt` / `.rs`)
**Pattern extraction date:** 2026-05-27
