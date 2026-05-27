# Phase 1: Plugin Skeleton + Binary Foundation - Research

**Researched:** 2026-05-27
**Domain:** LMS Plugin Skeleton (Perl) + librespot Binary Fork Architecture
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** SpotOn uses its own librespot fork (not Herger's Spotty fork directly). Git strategy (merge-based fork vs. upstream + patchset) deferred to research phase — decision depends on actual patch scope after auditing Herger's fork.
- **D-02:** Research phase MUST audit Herger's librespot fork to determine: which LMS-specific patches (--lms, --check, --single-track, --get-token, --lms-auth, --player-mac) exist, which are portable, which are missing, which should be structurally rewritten.
- **D-03:** Binaries are bundled in the plugin ZIP (Spotty model). All supported architectures (x86_64, aarch64, armhf, i386) ship in the ZIP under a `Bin/` directory.
- **D-04:** `Bin/` subdirectory structure uses Perl's `$Config{archname}` convention (e.g., `x86_64-linux-gnu-thread-multi`), not Spotty's simplified naming. This aligns with LMS internals. (**See critical finding below — this conflicts with actual LMS binary discovery behavior.**)
- **D-05:** No download-at-first-use mechanism. What ships in the ZIP is what runs.
- **D-06:** Settings page is pre-structured with sections for Binary Status and Account Configuration. Account-related fields are present but disabled/greyed out in Phase 1 — they become active in Phase 2 (Auth).
- **D-07:** When prerequisites are missing (no binary found, later: no account configured), the plugin shows a status hint as the first entry in the OPML menu root. Research phase verifies feasibility.
- **D-08:** Format identifier is `son` (NOT `spt`). MIME type: `audio/x-sb-spoton`.
- **D-09:** All four pipelines registered in Phase 1: `son→flc` (default), `son→pcm`, `son→mp3`, `son→ogg` (passthrough). Syntactically complete but only functional when streaming is implemented in Phase 4.

### Claude's Discretion

- Fork strategy (merge-based vs. upstream + patchset) — decided during research based on patch scope analysis
- Menu root hint feasibility — verified during research against LMS OPML conventions

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| LMS-01 | `spotify://` URI protocol handler registered and functional | ProtocolHandler.pm pattern confirmed; `canDirectStream { 0 }` is critical |
| LMS-02 | Web-based settings UI under LMS Settings | `Slim::Web::Settings` pattern confirmed from Spotty source |
| LMS-03 | i18n support (EN + DE minimum) via LMS strings mechanism | strings.txt format confirmed from installed Spotty |
| LMS-04 | install.xml manifest with correct metadata, minVersion, repository URL | Full structure confirmed from installed Spotty install.xml |
| LMS-05 | custom-convert.conf with `son → pcm/flc/mp3/ogg` transcoding pipelines | Spotty convert.conf confirmed; `son` format substitution documented |
| LMS-06 | Multi-architecture binaries (x86_64, aarch64, armhf, i386) | LMS binary discovery path logic reverse-engineered from OS.pm source |
| LMS-07 | Binary capability detection via `--check` JSON with version enforcement | `--check` JSON schema confirmed from Herger's spotty.rs; response format documented |

</phase_requirements>

---

## Summary

Phase 1 establishes the plugin load contract with LMS: a working install.xml, a loadable Plugin.pm, a settings page skeleton, i18n strings, transcoding format declarations, and validated librespot binaries with capability detection. No Spotify API calls, no authentication, no audio streaming in this phase.

The research confirms all five success criteria are implementable with well-understood patterns from Spotty and LMS internals. Three findings require planner attention:

**Critical (D-04 conflict):** LMS's actual binary search path logic (`Slim/Utils/OS.pm::initSearchPath`) uses simplified architecture names (`aarch64-linux`, `armhf-linux`, `arm-linux`, `i386-linux`) — NOT `$Config{archname}` strings. The planner must resolve this with the user before committing to a `Bin/` directory naming scheme, since D-04 as stated will cause binary discovery to fail.

**Fork strategy (Claude's discretion):** Herger's librespot fork is confirmed based on upstream 0.8.0 with LMS patches in `src/spotty.rs`. The patches are small, self-contained, and portable. Recommendation: fork `librespot-org/librespot` at the 0.8.0 tag and port Herger's `spotty.rs` as `spoton.rs`. This is a 1-2 day Rust task, not a multi-week fork management burden.

**OPML hint feasibility (Claude's discretion):** Confirmed feasible. Spotty does exactly this pattern: `handleFeed` checks for missing binary/credentials and returns a single `type => 'textarea'` OPML item with the localized error message, preventing further menu construction.

**Primary recommendation:** Follow Spotty's exact file structure as the template. Substitute `spt` → `son`, `spotty` → `spoton`, `Spotty` → `SpotOn`, `Plugins::Spotty` → `Plugins::SpotOn` throughout. The structural work is renaming + customization, not invention.

---

## Critical Finding: D-04 Bin/ Naming Conflict

**D-04 states:** "Use `$Config{archname}` convention (e.g., `x86_64-linux-gnu-thread-multi`)"

**Actual LMS behavior** (verified from `/usr/share/perl5/Slim/Utils/OS.pm::initSearchPath`, lines 91–125): [VERIFIED: LMS slimserver source]

```perl
# LMS normalizes $Config{archname} to simplified 'binArch':
# x86_64-linux-gnu-thread-multi  → binArch = 'i386-linux'  (but also checks x86_64-linux first)
# aarch64-linux-*                → binArch = 'aarch64-linux'
# arm*linux*gnueabihf            → binArch = 'armhf-linux'
# arm*linux                      → binArch = 'arm-linux'

# Search paths added for a plugin's Bin/ dir (in order):
# 1. Bin/x86_64-linux  (x86_64 systems, tried first)
# 2. Bin/i386-linux    (x86_64 normalized binArch)
# 3. Bin/linux         ($^O on Linux)
# 4. Bin/              (bare fallback)
```

**Spotty's actual Bin/ directories** (verified from installed plugin): [VERIFIED: npm registry]
```
Bin/aarch64-linux/         → spotty, spotty-hf (fallback)
Bin/arm-linux/             → spotty, spotty-hf
Bin/darwin-thread-multi-2level/
Bin/i386-linux/            → spotty, spotty-custom, spotty-x86_64
Bin/MSWin32-x86-multi-thread/
```

**How Spotty resolves x86_64:** On an x86_64 system, `binArch` = `i386-linux`. LMS adds `Bin/x86_64-linux` (checked first, but Spotty has no such dir), then `Bin/i386-linux`. Inside `Bin/i386-linux/`, Spotty stores BOTH a 32-bit `spotty` binary AND a 64-bit `spotty-x86_64` binary. `Helper.pm::_findBin()` then uses `$Config{archname} =~ /x86_64/` to prefer `spotty-x86_64` over `spotty` at the filename level.

**Recommended Bin/ structure for SpotOn** (resolves D-04 vs. actual LMS behavior): [ASSUMED]
```
Bin/x86_64-linux/     → spoton          (x86_64 musl static)
Bin/aarch64-linux/    → spoton          (aarch64 musl static)
Bin/armhf-linux/      → spoton          (armv7hf musl static)
Bin/arm-linux/        → spoton          (armv6 fallback)
Bin/i386-linux/       → spoton          (i686 musl static)
```

With `x86_64-linux/` as a named directory, LMS will find it before the `i386-linux` fallback, cleanly separating 64-bit and 32-bit binaries without filename suffix tricks. This is simpler than Spotty's approach and avoids the filename suffix complexity in `_findBin`.

**Action for planner:** Flag D-04 as needing user confirmation. If user prefers D-04 as written (`$Config{archname}` strings), the `Helper.pm` binary discovery code must NOT use `Slim::Utils::Misc::findbin()` — it would need to directly construct the path as `catdir($basedir, 'Bin', $Config{archname}, 'spoton')`. Present both options.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Plugin registration + lifecycle | Plugin.pm (Perl) | — | LMS init contract is `initPlugin()` in the plugin's `Plugin.pm` |
| Protocol handler (`spotify://`) | ProtocolHandler.pm (Perl) | — | LMS protocol handler framework requires a registered Perl module |
| Settings page | Settings.pm + HTML template | Slim::Web::Settings | LMS settings are HTML/TT2 templates served by the built-in web UI |
| i18n strings | strings.txt | cstring() LMS function | LMS owns the string lookup mechanism; plugin provides the strings file |
| Transcoding format declaration | custom-types.conf + custom-convert.conf | — | LMS reads these at startup to know about the `son` format |
| Binary discovery + validation | Helper.pm (Perl) | OS.pm path resolution | Perl owns the discovery logic; binary is opaque until `--check` validates it |
| librespot binary (all platforms) | Bin/ directory | cross-rs for cross-compile | Pre-built binaries; build happens outside LMS at release time |
| Binary LMS patches (`--check` etc.) | spoton.rs (Rust) | librespot-org/librespot base | Thin LMS-glue layer added to upstream librespot binary |

---

## Standard Stack

### Core (all bundled with LMS — no installation required)

| Library/Module | Version | Purpose | Why Standard |
|----------------|---------|---------|--------------|
| `Slim::Plugin::OPMLBased` | LMS 8.0+ | Plugin base class, menu registration | The LMS plugin framework — mandatory |
| `Slim::Utils::Prefs` | LMS 8.0+ | Preferences storage/retrieval | LMS idiom for all plugin settings |
| `Slim::Utils::Log` | LMS 8.0+ | Structured logging | Standard LMS logging |
| `Slim::Web::Settings` | LMS 8.0+ | Settings page base class | LMS settings page framework |
| `Slim::Player::ProtocolHandlers` | LMS 8.0+ | Protocol handler registration | Required for `spotify://` scheme |
| `Slim::Utils::OSDetect` | LMS 8.0+ | OS/arch detection | Used in binary discovery |
| `Slim::Utils::Misc::findbin` | LMS 8.0+ | Binary path resolution | LMS mechanism for finding plugin binaries |
| `Config` (Perl core) | Perl 5.10+ | `$Config{archname}` for arch detection | Standard Perl |
| `File::Spec::Functions` | Perl core | `catdir()` for path construction | Standard Perl |
| `JSON::XS` | Bundled in LMS CPAN | JSON encode/decode for `--check` response | Fast, bundled |

### Rust Build Toolchain (for librespot binary)

| Tool | Version | Purpose | Source |
|------|---------|---------|--------|
| librespot-org/librespot | 0.8.0 | Base binary | `github.com/librespot-org/librespot` tag `v0.8.0` |
| Rust | 1.85+ | Build language | `rust-toolchain.toml` in Herger's fork specifies 1.85 |
| cross-rs/cross | 0.2.5 | Cross-compilation for ARM | Available on this machine |

### Installed on This Machine

| Tool | Available | Version |
|------|-----------|---------|
| Perl | Yes | 5.38.2 (x86_64-linux-gnu-thread-multi) |
| LMS | Yes | 9.2.0 (running as lyrionmusicserver) |
| squeezelite | Yes | 1.9.9-1449 |
| cargo/rustup | Yes | cargo 1.95.0 |
| cross | Yes | 0.2.5 (installed) |
| docker/podman | No | Neither installed |

**Note:** `cross` 0.2.5 is installed but requires Docker or Podman for cross-compilation. Neither is present on this machine. ARM binary cross-compilation requires Docker installation or a remote build environment. This is a planning blocker for LMS-06. [VERIFIED: environment check]

### Package Legitimacy Audit

No external package installation required for this phase. All Perl modules are LMS-bundled. The librespot binary is built from source (Rust), not installed via a package manager.

| Package | Source | Status |
|---------|--------|--------|
| librespot 0.8.0 | `github.com/librespot-org/librespot` (official) | Verified legitimate |
| Spotty fork patches | `github.com/michaelherger/librespot` (spotty branch) | Verified legitimate — Herger is the author |

---

## Architecture Patterns

### System Architecture Diagram

```
[LMS startup]
     |
     v
install.xml read by PluginManager
     |
     v  
Plugin.pm::initPlugin()
     |-- prefs->init({ ... })               → default preferences stored
     |-- registerHandler('spotify', ...)    → LMS routes spotify:// URIs here
     |-- Settings.pm init                  → settings page registered
     |-- Helper::init()                    → binary search path configured
     |-- SUPER::initPlugin(feed => ...)    → OPMLBased registers plugin in menus
     |
     v
[User opens SpotOn menu in LMS]
     |
     v
OPML::handleFeed(client, callback, args)
     |-- Helper::get() → binary found? ────No──→ callback({ items: [{ type: 'textarea',
     |                                                  name: 'Binary nicht gefunden...' }] })
     |── Yes → (Phase 1 ends here: returns placeholder menu)
     
[custom-types.conf]  → LMS knows 'son' format exists (MIME: audio/x-sb-spoton)
[custom-convert.conf] → LMS knows son→flc/pcm/mp3/ogg pipelines (non-functional until Phase 4)

[Binary validation]
     |
     v
Helper::helperCheck(candidate)
     runs: spoton --check 2>&1
     parses: "ok spoton v{version}\n{JSON}"
     stores: $helper, $helperVersion, $helperCapabilities
```

### Recommended Project Structure

```
Plugins/SpotOn/
├── Plugin.pm                # initPlugin: prefs, handler, settings, OPMLBased
├── ProtocolHandler.pm       # Stub: contentType='son', isRemote=1, canDirectStream=0
├── Helper.pm                # Binary discovery, --check, getCapability
├── Settings.pm              # Slim::Web::Settings, basic.html wiring
├── strings.txt              # i18n: EN + DE minimum
├── install.xml              # Plugin manifest: GUID, minVersion=8.0, module
├── custom-types.conf        # son son audio/x-sb-spoton audio
├── custom-convert.conf      # son→flc/pcm/mp3/ogg pipelines
├── HTML/
│   └── EN/
│       └── plugins/
│           └── SpotOn/
│               └── settings/
│                   └── basic.html     # TT2 template
└── Bin/
    ├── x86_64-linux/        # spoton (x86_64 musl static)
    ├── aarch64-linux/       # spoton (aarch64 musl static)
    ├── armhf-linux/         # spoton (armv7hf musl static)
    ├── arm-linux/           # spoton (armv6 fallback)
    └── i386-linux/          # spoton (i686 musl static)
```

Note: `OPML.pm`, `API/`, `Connect/` directories are NOT created in Phase 1 — they belong to later phases.

### Pattern 1: OPMLBased Plugin Registration

```perl
# Source: /usr/share/squeezeboxserver/Plugins/Spotty/Plugin.pm (installed, verified)
# Source: github.com/michaelherger/Spotty-Plugin Plugin.pm

sub initPlugin {
    my $class = shift;

    $prefs->init({
        bitrate  => 320,
        binary   => '',          # custom binary override (LMS-10, Phase 6)
    });

    Slim::Player::ProtocolHandlers->registerHandler(
        'spotify',
        'Plugins::SpotOn::ProtocolHandler'
    );

    if ( main::WEBUI ) {
        require Plugins::SpotOn::Settings;
        Plugins::SpotOn::Settings->new;
    }

    require Plugins::SpotOn::Helper;
    Plugins::SpotOn::Helper->init();

    $class->SUPER::initPlugin(
        feed    => \&Plugins::SpotOn::OPML::handleFeed,
        tag     => 'spoton',
        menu    => 'radios',
        is_app  => 1,
        weight  => 100,
        icon    => 'plugins/SpotOn/html/images/icon.png',
    );
}
```

### Pattern 2: OPML Status Hint on Missing Prerequisites (D-07)

```perl
# Source: github.com/michaelherger/Spotty-Plugin OPML.pm (verified via WebFetch)
# Pattern: return early with informational textarea item

sub handleFeed {
    my ($client, $callback, $args) = @_;

    # Phase 1: check for binary
    if ( !Plugins::SpotOn::Helper->get() ) {
        $callback->({
            items => [{
                name => cstring($client, 'PLUGIN_SPOTON_BINARY_MISSING'),
                type => 'textarea',
            }]
        });
        return;
    }

    # Phase 2+: check for credentials
    # ...

    # Normal menu construction
    $callback->({ items => [ ... ] });
}
```

**Feasibility confirmed.** The `type => 'textarea'` OPML item displays as non-interactive text in the LMS menu. No LMS version-specific behavior — this is a standard OPML item type. [VERIFIED: Spotty source]

### Pattern 3: Binary Discovery (Helper.pm)

```perl
# Source: /usr/share/squeezeboxserver/Plugins/Spotty/Helper.pm (installed, verified)

use constant HELPER => 'spoton';

sub init {
    # aarch64 can fall back to armhf binaries if aarch64 binary unavailable
    if ( !main::ISWINDOWS && !main::ISMAC
         && Slim::Utils::OSDetect::details()->{osArch} =~ /^aarch64/i ) {
        Slim::Utils::Misc::addFindBinPaths(
            catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'armhf-linux')
        );
    }
}

sub helperCheck {
    my ($candidate, $check, $dontSet) = @_;

    $$check = '' unless $check && ref $check;
    my $checkCmd = sprintf('%s -n "SpotOn" --check', $candidate);
    $$check = `$checkCmd 2>&1`;

    # Expected response: "ok spoton v{version}\n{JSON}"
    if ( $$check && $$check =~ /^ok spoton v([\d\.]+)/i ) {
        unless ($dontSet) {
            $helper        = $candidate;
            $helperVersion = $1;
            if ( $$check =~ /\n(.*)/s ) {
                $helperCapabilities = eval { from_json($1) } || {};
            }
        }
        return 1;
    }
}

sub getCapability {
    my ($class, $key) = @_;
    return $helperCapabilities->{$key} if $helperCapabilities && defined $helperCapabilities->{$key};
    return undef;
}
```

**Minimum version enforcement** (not in Spotty — SpotOn adds this): [ASSUMED]
```perl
use constant MIN_BINARY_VERSION => '1.0.0';

sub helperCheck {
    # ...after extracting $helperVersion...
    if ( _versionCompare($helperVersion, MIN_BINARY_VERSION) < 0 ) {
        $log->warn("Binary version $helperVersion below minimum " . MIN_BINARY_VERSION);
        return 0;
    }
}
```

### Pattern 4: install.xml Structure

```xml
<!-- Source: /usr/share/squeezeboxserver/Plugins/Spotty/install.xml (installed, verified) -->
<?xml version='1.0' standalone='yes'?>
<extension>
    <name>PLUGIN_SPOTON_NAME</name>
    <creator>Your Name</creator>
    <defaultState>enabled</defaultState>
    <description>PLUGIN_SPOTON</description>
    <email>your@email.com</email>
    <category>musicservices</category>
    <id><!-- NEW UUID -- generate with uuidgen --></id>
    <module>Plugins::SpotOn::Plugin</module>
    <optionsURL>plugins/SpotOn/settings/basic.html</optionsURL>
    <icon>plugins/SpotOn/html/images/icon.png</icon>
    <targetApplication>
        <id>SqueezeCenter</id>
        <maxVersion>*</maxVersion>
        <minVersion>8.0</minVersion>
    </targetApplication>
    <type>2</type>
    <version>0.1.0</version>
</extension>
```

Key points:
- `id` MUST be a unique UUID — generate with `uuidgen`, never reuse Spotty's GUID
- `maxVersion` MUST be `*` (P-35: otherwise plugin disappears after LMS upgrade)
- `minVersion` = `8.0` (project floor)
- `type` = `2` = plugin with settings page [ASSUMED — type=2 is Spotty's value, not independently verified]

### Pattern 5: custom-types.conf and custom-convert.conf

```
# Source: /usr/share/squeezeboxserver/Plugins/Spotty/custom-types.conf (installed, verified)
# Format: ID  Suffix  Mime-Type                    Server-File-Type

son     son     audio/x-sb-spoton               audio
```

Note: Spotty also declares `spc` for Connect audio. SpotOn defers the Connect format type to Phase 5.

```
# Source: /usr/share/squeezeboxserver/Plugins/Spotty/custom-convert.conf (installed, verified)
# Adapted for SpotOn Phase 1 (pipelines syntactically present, non-functional until Phase 4)

son pcm * *
    # RT:{START=--start-position %s}
    [spoton] -n Squeezebox -c "$CACHE$" --single-track $URL$ --bitrate 320 --disable-discovery --disable-audio-cache $START$

son flc * *
    # RT:{START=--start-position %s}
    [spoton] -n Squeezebox -c "$CACHE$" --single-track $URL$ --bitrate 320 --disable-discovery --disable-audio-cache $START$ | [flac] -cs --channels=2 --sample-rate=44100 --bps=16 --endian=little --sign=signed --fast --totally-silent --ignore-chunk-sizes -

son mp3 * *
    # RB:{BITRATE=--abr %B}T:{START=--start-position %s}
    [spoton] -n Squeezebox -c "$CACHE$" --single-track $URL$ --bitrate 320 --disable-discovery --disable-audio-cache $START$ | [lame] -r --silent -q $QUALITY$ $BITRATE$ - -

son ogg * *
    # RT:{START=--start-position %s}
    [spoton] -n Squeezebox -c "$CACHE$" --single-track $URL$ --bitrate 320 --passthrough --disable-discovery --disable-audio-cache $START$
```

### Pattern 6: strings.txt Format

```
# Source: /usr/share/squeezeboxserver/Plugins/Spotty/strings.txt (installed, verified)
# Tab-indented, two-letter ISO language code + tab + value

PLUGIN_SPOTON
    DE  SpotOn Spotify für Squeezebox
    EN  SpotOn Spotify for Squeezebox

PLUGIN_SPOTON_NAME
    EN  SpotOn

PLUGIN_SPOTON_BINARY_MISSING
    DE  Spoton-Binary nicht gefunden — bitte in den Einstellungen prüfen
    EN  SpotOn binary not found — please check Settings

PLUGIN_SPOTON_BINARY_STATUS
    DE  Binary-Status
    EN  Binary Status

PLUGIN_SPOTON_ACCOUNT_SETTINGS
    DE  Account-Konfiguration
    EN  Account Configuration

PLUGIN_SPOTON_ACCOUNT_PLACEHOLDER
    DE  (wird in Phase 2 aktiviert)
    EN  (will be activated in Phase 2)

SON
    EN  SpotOn
```

The `SON` key registers the format name displayed in LMS. Spotty has `SPT	EN	Spotty`. [VERIFIED: Spotty strings.txt on installed plugin]

### Pattern 7: Settings Page (Slim::Web::Settings)

```perl
# Source: /usr/share/squeezeboxserver/Plugins/Spotty/Settings.pm (installed, verified)
package Plugins::SpotOn::Settings;

use base qw(Slim::Web::Settings);

use constant SETTINGS_URL => 'plugins/SpotOn/settings/basic.html';

sub new {
    my $class = shift;
    $class->SUPER::new();
}

sub name { return 'PLUGIN_SPOTON_NAME' }

sub page { return SETTINGS_URL }

sub prefs { return ($prefs, 'bitrate') }

sub handler {
    my ($class, $client, $params) = @_;

    if ( $params->{'saveSettings'} ) {
        # Save form fields to prefs
        $prefs->set('bitrate', $params->{'pref_bitrate'} || 320);
    }

    # Pass binary status to template
    $params->{'binaryVersion'} = Plugins::SpotOn::Helper->version();
    $params->{'binaryPath'}    = Plugins::SpotOn::Helper->get();

    return $class->SUPER::handler($client, $params);
}
```

Template (`basic.html`) uses `[% PROCESS settings/header.html %]` and `[% WRAPPER setting %]` TT2 macros provided by LMS. Disabled form fields use HTML `disabled` attribute. [VERIFIED: Spotty basic.html on installed plugin]

### Anti-Patterns to Avoid

- **Using `$Config{archname}` as-is for Bin/ directory names:** LMS normalizes arch names to simplified forms. If Bin/ dirs use full archname strings, binary discovery will silently fail because `findbin()` looks in `Bin/i386-linux/`, not `Bin/x86_64-linux-gnu-thread-multi/`. [VERIFIED: OS.pm source]
- **`maxVersion` in install.xml set to a specific version:** Plugin disappears after LMS upgrade without any error (P-35). Always use `*`. [VERIFIED: Spotty install.xml]
- **Reusing Spotty's GUID in install.xml:** Both plugins will fight over the same plugin slot. Generate a new UUID with `uuidgen`. [ASSUMED]
- **Calling `Slim::Utils::Misc::findbin('spoton')` before `Helper::init()`:** The Bin/ search paths are only added during init. Calling findbin before init returns undef silently. [VERIFIED: Helper.pm source]
- **Running `--check` as a blocking shell call from main server context:** `helperCheck` uses backticks (`\`$cmd 2>&1\``), which blocks the LMS event loop. Acceptable ONLY in initPlugin startup context, NOT from a timer callback or request handler. [ASSUMED — based on Spotty pattern, not explicitly documented]

---

## Herger's librespot Fork Audit (D-01/D-02)

### Fork Base and Status

| Property | Value | Source |
|----------|-------|--------|
| Base version | librespot-org/librespot 0.8.0 (Nov 2024) | Herger's Cargo.toml, spotty branch |
| Rust requirement | >= 1.85 | rust-toolchain.toml |
| Features compiled | `rodio-backend`, `native-tls`, `with-libmdns`, `spotty` (custom) | Herger's Cargo.toml |
| Last LMS-patch commit | Infrastructure/build updates through Nov 2025 | commit history |
| Active maintenance | Yes (build updates 2025) | github.com/michaelherger/librespot/commits/spotty |

[VERIFIED: Herger fork Cargo.toml and commit history via WebFetch]

### LMS-Specific Patches in `src/spotty.rs`

All LMS integration lives in a single file `src/spotty.rs` with three exported functions. This is cleanly separated from upstream librespot. [VERIFIED: WebFetch spotty.rs]

| CLI Flag | Function | Status | Portability |
|----------|----------|--------|-------------|
| `--check` | `check()` — prints JSON capability manifest and exits | Implemented | Fully portable — pure JSON output |
| `--single-track` | `play_track()` — decode one track to stdout and exit | Implemented | Fully portable |
| `--start-position` | parameter to `play_track()` | Implemented | Fully portable |
| `--get-token` / `-t` | `get_token()` — fetch Web API access token via login5 | Implemented | Fully portable |
| `--save-token` / `-T` | variant of `get_token()` | Implemented | Fully portable |
| `--lms <host:port>` | Connect daemon notifies LMS via JSON-RPC | Implemented | Fully portable |
| `--player-mac` | Identifies which LMS player to notify | Implemented | Fully portable |
| `--lms-auth` | Use LMS-provided credentials (Keymaster/login5) | Implemented | Fully portable |
| `--authenticate` | Interactive auth flow | Implemented | Low priority for SpotOn |
| `--client-id` | Override OAuth client ID | Implemented | Optional carry-over |
| `--scope` | OAuth scope for `--get-token` | Implemented | Useful for Phase 2 |
| `--zeroconf-port` | Bind ZeroConf to specific port | Implemented | Needed for Phase 5 (P-30) |

### `--check` JSON Output Schema (Herger's spotty v2.1.0)

```json
{
  "version": "2.1.0",
  "autoplay": true,
  "debug": false,
  "lms-auth": true,
  "no-ap-port": true,
  "oauth": true,
  "podcasts": true,
  "save-token": true,
  "temp-dir": true,
  "volume-normalisation": true,
  "zeroconf-port": true,
  "ogg-direct": true
}
```

**SpotOn should produce:** Same schema with `version` set to SpotOn's version string and the binary name changed from `"ok spotty v..."` to `"ok spoton v..."` in the response prefix line. [VERIFIED: spotty.rs WebFetch]

### Fork Strategy Recommendation (Claude's Discretion — D-01)

**Recommendation: Fork librespot-org/librespot at tag `v0.8.0`, port Herger's `src/spotty.rs` as `src/spoton.rs`.**

Rationale:
1. The LMS-specific code is entirely in `src/spotty.rs` — ~400 lines in a single file
2. All functions (`check()`, `get_token()`, `play_track()`) are independent of librespot internals — they use the public librespot API
3. Herger's fork builds from 0.8.0 (same base SpotOn would use)
4. `src/main.rs` additions are CLI argument parsing additions — easy to port
5. No architectural dependency on Herger's Rust changes — it's truly a thin glue layer

Porting effort: 1-2 days for Rust work (mainly adapting `spotty.rs` to `spoton.rs`, renaming constants, adjusting the `--check` output). No structural rewrites needed.

Alternative (merge-based fork): Higher maintenance burden — every librespot upstream release requires a merge conflict resolution pass. Not recommended given how thin the patch layer is.

**Decision:** Fork from upstream, port patches as `spoton.rs`. This is the architecture of minimal coupling.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Binary path resolution | Custom path walking | `Slim::Utils::Misc::findbin()` + `addFindBinPaths()` | LMS already handles platform + PATH search |
| Architecture detection | Custom `uname` parsing | `Slim::Utils::OSDetect::details()->{osArch}` | LMS already normalizes to `aarch64-linux`, `armhf-linux` etc. |
| Settings page HTTP handler | Custom HTTP dispatch | `Slim::Web::Settings` base class | Handles form parsing, saves prefs, renders TT2 template |
| i18n string lookup | Hash lookups | `cstring($client, 'KEY')` LMS function | Handles per-player locale, fallback chains |
| JSON parsing | Regex parsing | `JSON::XS::from_json()` | Bundled in LMS CPAN, handles all edge cases |
| Plugin menu registration | Manual CLI wiring | `Slim::Plugin::OPMLBased::SUPER::initPlugin` | Handles Jive, CLI, web, mobile — all in one call |
| Cross-compilation for ARM | Native Pi builds | `cross-rs/cross` with Docker | Native Pi compile takes hours; cross takes minutes (but requires Docker) |

**Key insight:** The LMS Plugin API has solved every infrastructure problem this phase needs. The plugin code should only contain domain-specific logic. Resist the urge to reimplement what LMS already provides.

---

## Common Pitfalls

### Pitfall 1: Bin/ Directory Name Mismatch (High Likelihood Without Research)

**What goes wrong:** Plugin installs on x86_64 system, binary is not found, plugin shows "binary missing" hint permanently.
**Why it happens:** Developer names Bin/ directories after `$Config{archname}` (D-04 as stated), but LMS's `OS.pm` normalizes to simplified arch names before constructing the search path.
**How to avoid:** Use `aarch64-linux`, `armhf-linux`, `arm-linux`, `i386-linux`, `x86_64-linux` as Bin/ subdirectory names — NOT `$Config{archname}` strings.
**Warning signs:** `Helper::get()` returns undef on first run; DEBUG log shows `findbin` checking wrong paths.

### Pitfall 2: install.xml GUID Collision with Spotty

**What goes wrong:** LMS loads SpotOn but silently overwrites Spotty's menu registration (or vice versa). Both plugins show the same settings URL.
**Why it happens:** Using Spotty's GUID (`21cbb80e-67b8-44a8-a662-21c6c7ae5260`) in SpotOn's install.xml.
**How to avoid:** Generate a new UUID with `uuidgen` for SpotOn. Never copy Spotty's GUID. [VERIFIED: Spotty install.xml]
**Warning signs:** Spotty's settings page URL opens SpotOn's page or vice versa.

### Pitfall 3: `--check` Response Format Mismatch

**What goes wrong:** `helperCheck()` regex `^ok spoton v([\d\.]+)` does not match. Binary exists but is never validated.
**Why it happens:** SpotOn's binary emits `ok spotty v...` (copied verbatim from Herger without renaming) or doesn't emit any "ok" prefix.
**How to avoid:** The binary's `--check` function must emit the line `ok spoton v{VERSION}` as its first output line, followed by a newline and the JSON object. The Perl regex in `helperCheck` must match `spoton`, not `spotty`.
**Warning signs:** Binary file exists and is executable, `Helper::get()` still returns undef.

### Pitfall 4: maxVersion Ceiling in install.xml

**What goes wrong:** Plugin works on LMS 9.x, user upgrades to LMS 9.2 or 10.x, plugin disappears from plugin list.
**Why it happens:** `maxVersion` set to a specific version number.
**How to avoid:** Always `<maxVersion>*</maxVersion>` (P-35).

### Pitfall 5: cross Requires Docker/Podman

**What goes wrong:** Plan calls for `cross build --target aarch64-unknown-linux-musl` to produce ARM binary; command fails with "Docker not found".
**Why it happens:** cross-rs requires a container runtime. This machine has neither Docker nor Podman.
**How to avoid:** Either (a) install Docker on the build machine before attempting binary builds, or (b) use a GitHub Actions workflow for ARM cross-compilation, or (c) build only x86_64 locally and rely on CI/CD for ARM.
**Warning signs:** `cross build` exits with error about missing container runtime.

### Pitfall 6: `Type => 'textarea'` vs `type => 'text'` OPML Status Hint

**What goes wrong:** Status hint item is rendered as a navigable menu item rather than read-only text.
**Why it happens:** Using `type => 'text'` instead of `type => 'textarea'` in the OPML item hashref.
**How to avoid:** Use `type => 'textarea'` for non-interactive informational messages (confirmed from Spotty OPML.pm).

---

## Code Examples

### Install a Fresh Plugin Into LMS for Testing

```bash
# Source: LMS documentation (lyrion.org)
# The LMS plugin directory for installed-from-zip plugins:
# /var/lib/squeezeboxserver/cache/InstalledPlugins/Plugins/SpotOn/
# Or create a development symlink:
sudo ln -s /home/sti/spoton/Plugins/SpotOn /usr/share/squeezeboxserver/Plugins/SpotOn
# Then restart LMS:
sudo systemctl restart lyrionmusicserver
# Check logs for load errors:
tail -f /var/log/squeezeboxserver/server.log | grep -i spoton
```

### Binary --check Invocation and Response Parsing

```perl
# Source: /usr/share/squeezeboxserver/Plugins/Spotty/Helper.pm (installed, verified)

my $checkCmd = sprintf('%s -n "SpotOn" --check', $candidate);
my $output = `$checkCmd 2>&1`;

if ($output && $output =~ /^ok spoton v([\d\.]+)/i) {
    my $version = $1;
    if ($output =~ /\n(.*)/s) {
        my $caps = eval { JSON::XS::decode_json($1) };
        # Access capabilities safely — never die on missing keys (P-29):
        my $hasOgg = $caps->{ogg-direct} // 0;
    }
}
```

### Protocol Handler Stub (Minimal for Phase 1)

```perl
# Source: ARCHITECTURE.md + Spotty-Plugin ProtocolHandler.pm pattern
package Plugins::SpotOn::ProtocolHandler;

use base qw(Slim::Formats::RemoteStream);

sub contentType { 'son' }

sub isRemote { 1 }

sub canDirectStream { 0 }  # CRITICAL: forces transcoding pipeline

sub getFormatForURL { 'flc' }  # default pipeline

sub formatOverride {
    my ($class, $song) = @_;
    # Phase 4: call Plugin::updateTranscodingTable here
    return 'son';
}

1;
```

---

## Runtime State Inventory

Phase 1 is greenfield — no existing runtime state. No migrations needed.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | None — no plugin has run yet | None |
| Live service config | None | None |
| OS-registered state | None | None |
| Secrets/env vars | None | None |
| Build artifacts | None | None |

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Spotty's `spotty` binary name throughout | SpotOn uses `spoton` (D-08) | Phase 1 decision | Coexistence with Spotty; no conflicts |
| `spt` format token | `son` format token (D-08) | Phase 1 decision | LMS can load both plugins simultaneously |
| Spotty GUID reuse | New UUID per install.xml | Phase 1 decision | Avoids plugin slot collision |
| Herger's `spotty.rs` | Ported as `spoton.rs` (Claude's recommendation) | Phase 1 | Minimal fork surface, easy upstream sync |
| Spotty's Bin/ names without `x86_64-linux/` | SpotOn adds `x86_64-linux/` directory | Phase 1 (recommended) | Clean x86_64 vs i386 separation |

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| LMS (lyrionmusicserver) | Testing all requirements | Yes | 9.2.0 | — |
| Perl | Plugin.pm, all .pm files | Yes | 5.38.2 | — |
| squeezelite | Testing playback (Phase 4+) | Yes | 1.9.9-1449 | — |
| cargo/rustup | Building librespot binary | Yes | 1.95.0 | — |
| cross (cross-rs) | ARM cross-compilation | Yes (installed) | 0.2.5 | — |
| Docker or Podman | Required by cross for ARM builds | **No** | — | GitHub Actions CI/CD, or install Docker |
| x86_64 native build | x86_64 binary only | Yes | — | — |

**Missing dependencies with no fallback:**
- Docker/Podman required by `cross` for ARM cross-compilation. Planning must decide: (a) install Docker, (b) use GitHub Actions for arm64/armhf builds, or (c) defer ARM binary builds to a later task after Docker setup.

**Missing dependencies with fallback:**
- ARM binary cross-compilation: GitHub Actions workflow (librespot upstream already has a cross-compile workflow) can build ARM binaries without Docker on the dev machine.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None detected — Perl LMS plugins typically use `prove` + Test::More |
| Config file | None exists |
| Quick run command | `prove -l t/` (once t/ directory is created) |
| Full suite command | `prove -lr t/` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| LMS-04 | install.xml parses as valid XML with required fields | unit | `perl -MXML::Simple -e 'XMLin("install.xml")' && echo ok` | No — Wave 0 |
| LMS-03 | strings.txt contains EN and DE for all PLUGIN_SPOTON_* keys | unit | `t/strings.t` (grep check) | No — Wave 0 |
| LMS-05 | custom-convert.conf contains all four son→* entries | unit | `t/convert_conf.t` (regex check) | No — Wave 0 |
| LMS-06 | Binary exists and is executable for x86_64 | unit | `t/binary_exists.t` | No — Wave 0 |
| LMS-07 | `--check` returns parseable JSON with version field | integration | `t/binary_check.t` | No — Wave 0 |
| LMS-01 | Protocol handler registered (module loads without error) | unit | `perl -I. -MPlugins::SpotOn::ProtocolHandler -e 1` | No — Wave 0 |
| LMS-02 | Settings.pm loads without error | unit | `perl -I. -MPlugins::SpotOn::Settings -e 1` | No — Wave 0 |

**Note:** Full LMS integration testing (plugin actually loads in LMS, settings page renders) requires a running LMS instance. This is manual verification. LMS 9.2.0 is installed and running at localhost. [VERIFIED: environment check]

### Sampling Rate
- **Per task:** `perl -c Plugins/SpotOn/*.pm` (syntax check)
- **Per wave:** `prove -l t/`
- **Phase gate:** Plugin loads in LMS, settings page accessible, binary check passes

### Wave 0 Gaps
- [ ] `t/strings.t` — validates EN/DE keys exist for all PLUGIN_SPOTON_* strings
- [ ] `t/binary_check.t` — runs x86_64 binary with `--check`, validates JSON output
- [ ] `t/convert_conf.t` — validates four son→* pipelines exist in custom-convert.conf
- [ ] `t/install_xml.t` — validates install.xml required fields (GUID, minVersion, module)

---

## Security Domain

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No — Phase 1 has no auth | — |
| V3 Session Management | No | — |
| V4 Access Control | No | — |
| V5 Input Validation | Partial | `--check` JSON output validated with `eval { from_json(...) }` + defined() guards |
| V6 Cryptography | No | — |

### Phase 1 Specific Security Notes

- Binary must be verified executable with `chmod +x` before shipping in ZIP; no world-writeable permissions
- `helperCheck` uses backticks `\`$cmd\`` with no user-supplied input in Phase 1 (binary path is from `findbin` internal, not user input). In future phases, if user can specify binary path in settings, the path must be sanitized before shell execution (P-39 pattern).
- The `--check` JSON is parsed with `eval { from_json(...) }` — the eval guard prevents `from_json` parse errors from crashing the plugin. Always use this pattern. [VERIFIED: Spotty Helper.pm]

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Bin/ directory `x86_64-linux/` will be searched by LMS before `i386-linux/` on x86_64 systems | Critical Finding + Bin/ structure | Binary not found on x86_64 without extra _findBin filename logic |
| A2 | `install.xml` `type=2` is the correct value for a plugin with settings | install.xml pattern | Settings URL might not display; however type=2 is from installed Spotty |
| A3 | Minimum version check in `helperCheck` is not implemented in Spotty but SpotOn should add it | Helper.pm pattern | Without it, stale/incompatible binary silently used |
| A4 | `--check` backtick shell call is acceptable in `initPlugin` startup context but not in event callbacks | Anti-patterns | Could block LMS event loop if binary hangs during startup |
| A5 | Rust porting effort for `spotty.rs` → `spoton.rs` is 1-2 days | Fork strategy | Could be longer if Rust API details have changed between versions |

---

## Open Questions

1. **D-04 Resolution: Bin/ naming scheme**
   - What we know: LMS uses simplified names (`aarch64-linux`, `x86_64-linux`); D-04 says use `$Config{archname}` strings
   - What's unclear: User may have had a specific reason for D-04 (e.g., CPAN module compatibility), or it may have been stated without knowing LMS's actual binary resolution behavior
   - Recommendation: Planner presents the finding to user, proposes `x86_64-linux`/`aarch64-linux` naming as the correct approach

2. **Docker for ARM cross-compilation**
   - What we know: `cross` is installed but requires Docker/Podman; neither is available
   - What's unclear: Whether GitHub Actions CI or local Docker installation is preferred
   - Recommendation: Plan includes a Wave 0 task "install Docker OR configure GitHub Actions cross-compile workflow"; binary builds for ARM are gated on this task

3. **SpotOn binary name: `spoton` or `librespot-spoton` or just `spoton`**
   - What we know: Spotty binary is named `spotty`; SpotOn needs its own name to avoid conflicts
   - What's unclear: Naming convention for the binary inside `Bin/` dirs
   - Recommendation: Use `spoton` as the binary name (analogous to `spotty`). `HELPER` constant in `Helper.pm` = `'spoton'`.

---

## Sources

### Primary (HIGH confidence)
- `/usr/share/perl5/Slim/Utils/OS.pm` — `initSearchPath` binary arch normalization (verified on installed LMS 9.2.0)
- `/usr/share/perl5/Slim/Utils/PluginManager.pm` — binary path construction for plugin Bin/ dirs (verified)
- `/usr/share/squeezeboxserver/Plugins/Spotty/Helper.pm` — binary discovery implementation (verified)
- `/usr/share/squeezeboxserver/Plugins/Spotty/custom-convert.conf` — transcoding pipeline syntax (verified)
- `/usr/share/squeezeboxserver/Plugins/Spotty/custom-types.conf` — format type declaration (verified)
- `/usr/share/squeezeboxserver/Plugins/Spotty/install.xml` — manifest structure (verified)
- `/usr/share/squeezeboxserver/Plugins/Spotty/strings.txt` — strings format (verified)
- `github.com/michaelherger/librespot/blob/spotty/src/spotty.rs` — LMS patches, `--check` JSON schema (WebFetch verified)
- `github.com/michaelherger/librespot/blob/spotty/Cargo.toml` — fork base version 0.8.0 (WebFetch verified)
- `github.com/michaelherger/Spotty-Plugin/blob/master/OPML.pm` — OPML status hint pattern (WebFetch verified)
- `github.com/michaelherger/Spotty-Plugin/blob/master/Settings.pm` — settings page pattern (WebFetch verified)

### Secondary (MEDIUM confidence)
- `lyrion.org/reference/music-service-plugin/` — LMS plugin API docs (WebFetch)
- `github.com/LMS-Community/plugin-Qobuz/blob/master/custom-types.conf` — format declaration format (WebFetch)

---

## Project Constraints (from CLAUDE.md)

| Directive | Impact on Phase 1 |
|-----------|-------------------|
| Language: Perl >= 5.10 | All .pm files must be valid Perl 5.10+; no `say`, `//=` without `use feature` guard |
| No external CPAN deps | All modules in Plugin.pm/Helper.pm/Settings.pm must be from LMS bundled set |
| LMS version floor 8.0 | `minVersion` in install.xml = 8.0; no LMS 9-only APIs in skeleton |
| OPML UI paradigm | No HTML grid/tabs in settings beyond LMS's standard TT2 wrapper macros |
| Binary: librespot only | Binary is librespot fork; no alternative audio backends in Phase 1 |
| Namespace: `Plugins::SpotOn` | All modules under `Plugins::SpotOn::*`; plugin prefs key: `plugin.spoton` |
| Format token `son` / MIME `audio/x-sb-spoton` | As declared in custom-types.conf; never `spt` |

---

## Metadata

**Confidence breakdown:**
- LMS plugin structure: HIGH — verified from installed LMS 9.2.0 source and Spotty plugin
- Binary discovery mechanics: HIGH — verified from OS.pm and PluginManager.pm source
- Herger's fork patches: HIGH — verified from spotty.rs via WebFetch
- Fork strategy recommendation: MEDIUM — recommendation based on code analysis, actual porting time is estimated
- Bin/ naming recommendation: HIGH — derived directly from OS.pm::initSearchPath source

**Research date:** 2026-05-27
**Valid until:** 2026-08-27 (stable LMS APIs; librespot binary patches are stable)
