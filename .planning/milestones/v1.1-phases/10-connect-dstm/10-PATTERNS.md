# Phase 10: Connect-DSTM - Pattern Map

**Mapped:** 2026-06-04
**Files analyzed:** 7 modification targets (no new files)
**Analogs found:** 7 / 7

---

## File Classification

| Modified File | Role | Data Flow | Closest Analog | Match Quality |
|---------------|------|-----------|----------------|---------------|
| `librespot-spoton/src/main.rs` | CLI parser + check manifest | request-response | `main.rs` itself (existing flag patterns) | exact |
| `librespot-spoton/src/connect.rs` | async orchestrator | event-driven | `connect.rs` itself (`SessionConfig` setup at line 867) | exact |
| `Plugins/SpotOn/Connect/Daemon.pm` | daemon launcher | request-response | `spotty-ng/Spotty-Plugin/Connect/Daemon.pm` lines 102-103 | exact |
| `Plugins/SpotOn/Settings.pm` | settings handler | request-response | `Settings.pm` itself (`enableSpotifyConnect` block, lines 145-176) | exact |
| `Plugins/SpotOn/Plugin.pm` | plugin init | config | `Plugin.pm` itself (`$prefs->init` block, lines 41-50) | exact |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | settings template | request-response | `basic.html` itself (Connect/Discovery checkbox blocks, lines 21-39) | exact |
| `Plugins/SpotOn/strings.txt` | i18n strings | config | `strings.txt` itself (`PLUGIN_SPOTON_CONNECT_ENABLED` block, lines 690-727) | exact |

**Read-only (verify, no changes):**

| File | Role | Verify |
|------|------|--------|
| `Plugins/SpotOn/DontStopTheMusic.pm` | DSTM provider | `registerHandler` key matches DSTM sync target string |
| `Plugins/SpotOn/Helper.pm` | capability check | `getCapability('autoplay')` works as-is |

---

## Pattern Assignments

### `librespot-spoton/src/main.rs` — Add `--autoplay` flag + `"autoplay": true` in `--check`

**Analog:** `main.rs` itself — existing `--buffer-latency-ms` and `--disable-discovery` flag patterns.

**Variable declaration pattern** (lines 105-110, Connect mode variables block):
```rust
// Connect mode variables — add alongside existing ones:
let mut disable_discovery = false;
let mut buffer_latency_ms: u64 = 2000;
// ADD:
let mut autoplay: Option<bool> = None;
```

**Arg-loop match arm pattern** (lines 161-187, after `--disable-discovery`):
```rust
// Existing flag with value — use --buffer-latency-ms as exact template:
"--buffer-latency-ms" => {
    if i + 1 < args.len() {
        buffer_latency_ms = args[i + 1].parse().unwrap_or(2000);
        i += 1;
    }
}
// New arm copies the structure but maps "on"/"off" strings:
"--autoplay" => {
    if i + 1 < args.len() {
        autoplay = match args[i + 1].as_str() {
            "on"  => Some(true),
            "off" => Some(false),
            _     => None,
        };
        i += 1;
    }
}
```

**`--check` JSON manifest pattern** (lines 244-252):
```rust
// Existing manifest — add "autoplay": true alongside other capabilities:
let json = serde_json::json!({
    "version": VERSION,
    "discover-once": true,
    "lms-auth": false,
    "ogg-direct": has_passthrough,
    "passthrough": has_passthrough,
    "token-login": true,
    // ADD:
    "autoplay": true,
});
```

**`run_connect()` call site pattern** (lines 310-319, Mode::Connect arm):
```rust
// Existing call — add autoplay as final argument:
match connect::run_connect(
    &cache_dir,
    &device_name,
    if player_mac.is_empty() { None } else { Some(&player_mac) },
    if lms_host.is_empty() { None } else { Some(&lms_host) },
    if lms_auth.is_empty() { None } else { Some(&lms_auth) },
    disable_discovery,
    buffer_latency_ms,
    // ADD:
    autoplay,
)
```

---

### `librespot-spoton/src/connect.rs` — Add `autoplay: Option<bool>` to `run_connect()`

**Analog:** `connect.rs` itself — `SessionConfig` setup block at lines 866-869.

**Function signature pattern** (lines 821-829):
```rust
// Current signature — append autoplay as final parameter:
pub async fn run_connect(
    cache_dir: &str,
    device_name: &str,
    player_mac: Option<&str>,
    lms_host_port: Option<&str>,
    lms_auth: Option<&str>,
    disable_discovery: bool,
    buffer_latency_ms: u64,
    // ADD:
    autoplay: Option<bool>,
) -> Result<(), Box<dyn std::error::Error>> {
```

**SessionConfig setup pattern** (lines 866-869 — CRITICAL: set autoplay BEFORE `Session::new()`):
```rust
// Current code:
let mut session_config = SessionConfig::default();
session_config.device_id = device_id_shared.clone();
// ADD immediately after device_id, BEFORE Session::new():
if let Some(ap) = autoplay {
    session_config.autoplay = Some(ap);
}
// Existing — must remain after the new block:
let session = Session::new(session_config.clone(), Some(cache.clone()));
```

**No changes needed** to the reconnect path at line 1053 (`session_config.clone()` already carries the autoplay field).

---

### `Plugins/SpotOn/Connect/Daemon.pm` — Add `--autoplay on/off` to `@helperArgs`

**Analog:** `spotty-ng/Spotty-Plugin/Connect/Daemon.pm` lines 102-103 — exact proven pattern.

**`@helperArgs` construction pattern** (lines 103-120 in SpotOn Daemon.pm):
```perl
# Existing flags:
my @helperArgs = (
    '-c', $self->cache,
    '-n', $self->name,
    '--disable-audio-cache',
    '--player-mac', $self->mac,
    '--lms', '127.0.0.1:' . $serverPrefs->get('httpport'),
    '--connect',
);

# After '--enable-volume-normalisation' push (line 120):
push @helperArgs, '--enable-volume-normalisation' if $prefs->get('normalization');

# ADD — capability-gated, same pattern as Spotty-NG lines 102-103:
if ( Plugins::SpotOn::Helper->getCapability('autoplay') ) {
    my $enableAutoplay = $prefs->client($client)->get('enableAutoplay');
    $enableAutoplay = 1 unless defined $enableAutoplay;  # default on
    push @helperArgs, '--autoplay', ($enableAutoplay ? 'on' : 'off');
}
```

**Placement:** After the `--enable-volume-normalisation` push and BEFORE the security log line (line 123), matching Spotty-NG's ordering pattern.

---

### `Plugins/SpotOn/Settings.pm` — Add `enableAutoplay` save + DSTM sync + daemon restart

**Analog:** `Settings.pm` itself — `enableSpotifyConnect` per-player block (lines 145-176) is the exact template.

**Per-player pref save + daemon restart pattern** (lines 145-176):
```perl
# Existing Connect toggle — exact template to follow:
if ($client) {
    my $enableConnect = $paramRef->{'pref_enableSpotifyConnect'} ? 1 : 0;
    $prefs->client($client)->set('enableSpotifyConnect', $enableConnect);

    # ... other per-player prefs ...

    require Plugins::SpotOn::Connect::DaemonManager;
    Plugins::SpotOn::Connect::DaemonManager->initHelpers();
}
```

**New `enableAutoplay` block** — insert WITHIN the existing `if ($client)` block, before `initHelpers()`:
```perl
if ($client) {
    # ... existing per-player saves ...

    # ADD: enableAutoplay toggle (D-08, D-09)
    if ( defined $paramRef->{'pref_enableAutoplay'} || $paramRef->{saveSettings} ) {
        my $enableAutoplay = $paramRef->{'pref_enableAutoplay'} ? 1 : 0;
        $prefs->client($client)->set('enableAutoplay', $enableAutoplay);

        # DSTM bidirectional sync (D-11, D-12)
        if ( Slim::Utils::PluginManager->isEnabled('Slim::Plugin::DontStopTheMusic::Plugin') ) {
            my $dstmPrefs = preferences('plugin.dontstopthemusic');
            if ($enableAutoplay) {
                $dstmPrefs->client($client)->set('provider', 'PLUGIN_SPOTON_RECOMMENDATIONS');
            } else {
                $dstmPrefs->client($client)->set('provider', 0);
            }
        }

        # Daemon restart: stop first (startHelper skips live daemons — RESEARCH Pitfall 1)
        require Plugins::SpotOn::Connect::DaemonManager;
        my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client);
        $helper->stopForSync() if $helper && $helper->alive;
    }

    # Existing daemon init call (runs after all pref saves):
    require Plugins::SpotOn::Connect::DaemonManager;
    Plugins::SpotOn::Connect::DaemonManager->initHelpers();
}
```

**Template var pattern** (lines 200-213 — populate for GET requests):
```perl
# Existing template vars — add canAutoplay + autoplayEnabled alongside:
if ($client) {
    $paramRef->{connectEnabled}      = $prefs->client($client)->get('enableSpotifyConnect') // 1;
    # ... other existing vars ...
    # ADD:
    $paramRef->{canAutoplay}     = Plugins::SpotOn::Helper->getCapability('autoplay') ? 1 : 0;
    $paramRef->{autoplayEnabled} = $prefs->client($client)->get('enableAutoplay') // 1;
}
```

**D-13/D-14 sync (DSTM->SpotOn direction):** Read DSTM pref at page-load to override `autoplayEnabled` template var — no callback needed, no loop risk:
```perl
# ADD after setting autoplayEnabled above:
if ( $paramRef->{canAutoplay}
     && Slim::Utils::PluginManager->isEnabled('Slim::Plugin::DontStopTheMusic::Plugin') )
{
    my $dstmPrefs    = preferences('plugin.dontstopthemusic');
    my $dstmProvider = $dstmPrefs->client($client)->get('provider') // '';
    # Override autoplayEnabled based on current DSTM provider state
    $paramRef->{autoplayEnabled} =
        ($dstmProvider eq 'PLUGIN_SPOTON_RECOMMENDATIONS') ? 1 : 0;
}
```

---

### `Plugins/SpotOn/Plugin.pm` — Add `enableAutoplay => 1` to `$prefs->init`

**Analog:** `Plugin.pm` itself — existing `$prefs->init` block (lines 41-50).

**Pattern** (lines 41-50):
```perl
# Existing init block — add enableAutoplay alongside enableSpotifyConnect:
$prefs->init({
    bitrate              => 320,
    normalization        => 0,
    binary               => '',
    accounts             => {},
    activeAccount        => '',
    enableSpotifyConnect => 1,
    connectOggOverride   => 'auto',
    disableDiscovery     => 0,
    # ADD:
    enableAutoplay       => 1,    # D-08: default on; controls Connect autoplay + DSTM
});
```

**No other changes to Plugin.pm.** The DSTM registration in lines 87-92 stays as-is (D-15: provider stays registered regardless of toggle).

---

### `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Add autoplay checkbox

**Analog:** `basic.html` itself — `pref_enableSpotifyConnect` checkbox block (lines 21-26) is the exact template.

**Existing checkbox pattern** (lines 21-26):
```html
[% WRAPPER setting title="PLUGIN_SPOTON_CONNECT_ENABLED" desc="PLUGIN_SPOTON_CONNECT_ENABLED_DESC" %]
    <input type="checkbox" class="stdedit" name="pref_enableSpotifyConnect"
           id="pref_enableSpotifyConnect"
           value="1" [% IF connectEnabled %]checked[% END %]/>
    <label for="pref_enableSpotifyConnect">[% 'PLUGIN_SPOTON_CONNECT_ENABLED_LABEL' | string %]</label>
[% END %]
```

**New autoplay checkbox** — insert after the `pref_enableSpotifyConnect` block, within `[% IF playerid %]`:
```html
[% IF canAutoplay %]
[% WRAPPER setting title="PLUGIN_SPOTON_AUTOPLAY_ENABLED" desc="PLUGIN_SPOTON_AUTOPLAY_ENABLED_DESC" %]
    <input type="checkbox" class="stdedit" name="pref_enableAutoplay"
           id="pref_enableAutoplay"
           value="1" [% IF autoplayEnabled %]checked[% END %]/>
    <label for="pref_enableAutoplay">[% 'PLUGIN_SPOTON_AUTOPLAY_ENABLED_LABEL' | string %]</label>
[% END %]
[% END %]
```

**Placement:** After line 26 (after the Connect checkbox `[% END %]`), before the Discovery checkbox block at line 28. The `[% IF canAutoplay %]` guard matches the D-10 requirement that the toggle only shows when the binary supports autoplay.

---

### `Plugins/SpotOn/strings.txt` — Add three new i18n key blocks

**Analog:** `strings.txt` itself — `PLUGIN_SPOTON_CONNECT_ENABLED` triple block (lines 690-727) is the exact template structure.

**Pattern** (lines 690-727 — three-key block: title / description / label):
```
PLUGIN_SPOTON_CONNECT_ENABLED
    CS  Spotify Connect
    DA  Spotify Connect
    DE  Spotify Connect
    EN  Spotify Connect
    ...all 11 languages...

PLUGIN_SPOTON_CONNECT_ENABLED_DESC
    CS  Aktivovat Spotify Connect pro tohoto přehrávače
    ...

PLUGIN_SPOTON_CONNECT_ENABLED_LABEL
    CS  Aktivovat Spotify Connect
    ...
```

**New three-key block to add** (insert adjacent to `PLUGIN_SPOTON_CONNECT_ENABLED` block, ordering alphabetically or after the Connect group):
```
PLUGIN_SPOTON_AUTOPLAY_ENABLED
    CS  Automatické přehrávání (Autoplay)
    DA  Automatisk afspilning (Autoplay)
    DE  Automatische Wiedergabe (Autoplay)
    EN  Auto-play (Autoplay)
    ES  Reproducción automática (Autoplay)
    FR  Lecture automatique (Autoplay)
    IT  Riproduzione automatica (Autoplay)
    NL  Automatisch afspelen (Autoplay)
    NO  Automatisk avspilling (Autoplay)
    PL  Automatyczne odtwarzanie (Autoplay)
    SV  Automatisk uppspelning (Autoplay)

PLUGIN_SPOTON_AUTOPLAY_ENABLED_DESC
    CS  Pokračovat v přehrávání s doporučenými skladbami, když skončí fronta
    DA  Fortsæt afspilning med anbefalede numre, når køen er tom
    DE  Wiedergabe mit empfohlenen Titeln fortsetzen, wenn die Warteschlange endet
    EN  Continue playback with recommended tracks when the queue ends
    ES  Continuar la reproducción con canciones recomendadas cuando se acabe la cola
    FR  Continuer la lecture avec des titres recommandés quand la file d'attente se termine
    IT  Continua la riproduzione con brani consigliati quando la coda finisce
    NL  Doorgaan met afspelen met aanbevolen nummers als de wachtrij leeg is
    NO  Fortsett avspilling med anbefalte spor når køen er tom
    PL  Kontynuuj odtwarzanie z polecanymi utworami, gdy kolejka się skończy
    SV  Fortsätt uppspelning med rekommenderade låtar när kön tar slut

PLUGIN_SPOTON_AUTOPLAY_ENABLED_LABEL
    CS  Automatické přehrávání aktivovat
    DA  Aktiver automatisk afspilning
    DE  Automatische Wiedergabe aktivieren
    EN  Enable auto-play
    ES  Activar reproducción automática
    FR  Activer la lecture automatique
    IT  Abilita riproduzione automatica
    NL  Automatisch afspelen inschakelen
    NO  Aktiver automatisk avspilling
    PL  Włącz automatyczne odtwarzanie
    SV  Aktivera automatisk uppspelning
```

---

## Shared Patterns

### Per-Player Pref: Read/Write
**Source:** `Plugins/SpotOn/Settings.pm` lines 145-147, `Plugins/SpotOn/Plugin.pm` lines 41-50
**Apply to:** All per-player pref operations in Settings.pm handler
```perl
# Write (in saveSettings block):
my $value = $paramRef->{'pref_NAME'} ? 1 : 0;
$prefs->client($client)->set('NAME', $value);

# Read (in template-vars block):
$paramRef->{templateVar} = $prefs->client($client)->get('NAME') // $default;
```

### Capability-Gated UI Element
**Source:** `spotty-ng/Spotty-Plugin/Settings/Player.pm` lines 27, 38 + `basic.html` Discovery block (lines 28-39)
**Apply to:** `canAutoplay` guard in both Settings.pm and basic.html
```perl
# Settings.pm:
$paramRef->{canAutoplay} = Plugins::SpotOn::Helper->getCapability('autoplay') ? 1 : 0;
```
```html
[% IF canAutoplay %]
  [% WRAPPER setting ... %] ... [% END %]
[% END %]
```

### Daemon Restart After Pref Change
**Source:** `Plugins/SpotOn/Settings.pm` lines 173-175 + `Plugins/SpotOn/Connect/Daemon.pm::stopForSync()` (lines 313-327)
**Apply to:** enableAutoplay change in Settings.pm
```perl
# Must stop BEFORE initHelpers — startHelper skips live daemons (RESEARCH Pitfall 1):
my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client);
$helper->stopForSync() if $helper && $helper->alive;
Plugins::SpotOn::Connect::DaemonManager->initHelpers();
```

### Cross-Namespace Pref Write
**Source:** RESEARCH.md Common Pitfalls 2 + LMS DSTM Framework section
**Apply to:** DSTM provider sync in Settings.pm
```perl
# Always use a separate preferences() object — $prefs is bound to 'plugin.spoton':
my $dstmPrefs = preferences('plugin.dontstopthemusic');
$dstmPrefs->client($client)->set('provider', 'PLUGIN_SPOTON_RECOMMENDATIONS');  # ON
$dstmPrefs->client($client)->set('provider', 0);                                # OFF
```

### Checkbox Unchecked = 0 Guard
**Source:** `Plugins/SpotOn/Settings.pm` lines 71, 146 (established pattern throughout)
**Apply to:** `pref_enableAutoplay` form field handling
```perl
# Browser sends no value for unchecked checkbox — must coerce to 0:
my $value = $paramRef->{'pref_enableAutoplay'} ? 1 : 0;
```

---

## Verification: No-Change Files

### `Plugins/SpotOn/DontStopTheMusic.pm`
- `registerHandler('PLUGIN_SPOTON_RECOMMENDATIONS', ...)` at lines 22-25 — key must match DSTM sync target string exactly. **No modification needed** (D-15).
- `dontStopTheMusic` handler at line 32 — remains unchanged. Browse-DSTM continues to function; DSTM framework won't call it when `provider != 'PLUGIN_SPOTON_RECOMMENDATIONS'` (implicitly disabled by sync setting `provider=0`).

### `Plugins/SpotOn/Helper.pm`
- `getCapability($class, $key)` at lines 104-108 — reads from `$helperCapabilities` parsed from `--check` JSON. Once binary is rebuilt with `"autoplay": true`, `getCapability('autoplay')` returns `1` with no code changes.

---

## No Analog Found

All Phase 10 files have exact or role-match analogs in the existing codebase. No files lack a pattern source.

---

## Pitfall Summary for Planner

| Pitfall | Where It Bites | Guard |
|---------|---------------|-------|
| `startHelper` skips live daemons | Settings.pm after enableAutoplay save | `stopForSync()` before `initHelpers()` |
| `session_config.autoplay` must be set BEFORE `Session::new()` | connect.rs | Set field before `Session::new(session_config.clone(), ...)` on line 869 |
| Cross-namespace pref write fails if using `$prefs` | Settings.pm DSTM sync | Separate `preferences('plugin.dontstopthemusic')` object |
| Checkbox absent in POST = unchecked | Settings.pm handler | `$paramRef->{'pref_enableAutoplay'} ? 1 : 0` |
| DSTM callback loop | Settings.pm | D-13/D-14 sync only in page-load direction (read DSTM pref, don't set a callback) |

---

## Metadata

**Analog search scope:** `librespot-spoton/src/`, `Plugins/SpotOn/`, `Plugins/SpotOn/Connect/`, `spotty-ng/Spotty-Plugin/`
**Files read:** 11 source files
**Pattern extraction date:** 2026-06-04
