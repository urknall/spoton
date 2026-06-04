# Phase 8: Multi-Arch Binary Distribution - Pattern Map

**Mapped:** 2026-06-03
**Files analyzed:** 4 (1 modify, 3 new/build)
**Analogs found:** 4 / 4

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `Plugins/SpotOn/Helper.pm` | utility | request-response | `Spotty-Plugin/Helper.pm` (Herger) + current `Helper.pm` | exact |
| `librespot-spoton/Cross.toml` | config | build | `.cargo/config.toml` (existing) | role-match |
| `Plugins/SpotOn/Bin/x86_64-win64/` | directory | N/A | `Plugins/SpotOn/Bin/x86_64-linux/` | exact |
| Build script (cross-compile 6 targets) | utility | batch | No existing analog | N/A |

## Pattern Assignments

### `Plugins/SpotOn/Helper.pm` (utility, modify - platform detection)

**Analog:** Current `Plugins/SpotOn/Helper.pm` (lines 1-162) + Herger's `Spotty-Plugin/Helper.pm` (lines 19-23, 136-201) + LMS `Slim::Utils::OS::initSearchPath` (lines 85-132)

#### Critical Discovery: LMS Already Handles Most Platform Detection

LMS's `Slim::Utils::PluginManager` (line 340-363) calls `initSearchPath` for every plugin's `Bin/` directory. The `Slim::Utils::OS::initSearchPath` method (lines 85-132) maps `$Config::Config{'archname'}` to `binArch` directory names and registers them with `addFindBinPaths`:

| Perl archname pattern | binArch (directory) | Paths registered |
|---|---|---|
| `x86_64-linux-*` | `i386-linux` (normalized) | `Bin/x86_64-linux/`, `Bin/i386-linux/`, `Bin/linux/`, `Bin/` |
| `aarch64-linux*` | `aarch64-linux` | `Bin/aarch64-linux/`, `Bin/linux/`, `Bin/` |
| `arm*linux*gnueabihf` | `armhf-linux` | `Bin/armhf-linux/`, `Bin/arm-linux/`, `Bin/linux/`, `Bin/` |
| `arm*linux` | `arm-linux` | `Bin/arm-linux/`, `Bin/linux/`, `Bin/` |

This means: x86_64, aarch64, armhf, and arm are already handled by LMS's built-in path resolution. The `findbin('spoton')` call in `_findBin` will find binaries in the correct platform subdirectory automatically.

**What Helper.pm init() must still do:**
1. Add fallback paths that LMS does NOT add (e.g., aarch64 -> armhf, armv7 -> arm)
2. Add the `i386-linux/` path for i386 systems (LMS normalizes both i386 and x86_64 to `i386-*` but only adds x86_64 as extra)
3. Add the Windows directory `x86_64-win64/` (LMS Windows uses `File::Which`, not `@findBinPaths`)

**Source: `Slim::Utils::OS::initSearchPath`** (lines 85-132 of `/usr/share/perl5/Slim/Utils/OS.pm`):
```perl
sub initSearchPath {
    my $class = shift;
    my $baseDir = shift || $class->dirsFor('Bin');

    my $binArch = $class->{osDetails}->{'binArch'} = $Config::Config{'archname'};
    $class->{osDetails}->{'binArch'} =~ s/^(?:i[3456]86|x86_64)-([^-]+).*/i386-$1/;

    # Reduce ARM to arm(hf)-linux
    if ( $class->{osDetails}->{'binArch'} =~ /^arm.*linux.*gnueabihf/ ||
        ($class->{osDetails}->{'binArch'} =~ /arm/ && (
            $Config::Config{'lddlflags'} =~ /\-mfloat\-abi=hard/ ||
            $Config::Config{'config_args'} =~ /\-mfloat\-abi=hard/
        ))
    ) {
        $class->{osDetails}->{'binArch'} = 'armhf-linux';
    }
    elsif ( $class->{osDetails}->{'binArch'} =~ /^arm.*linux/ ) {
        $class->{osDetails}->{'binArch'} = 'arm-linux';
    }
    elsif ( $class->{osDetails}->{'binArch'} =~ /^aarch64-linux/ ) {
        $class->{osDetails}->{'binArch'} = 'aarch64-linux';
    }

    my @paths = ( catdir($baseDir, $class->{osDetails}->{'binArch'}), catdir($baseDir, $^O), $baseDir );

    # Linux x86_64 should check its native folder first
    if ( $binArch =~ s/^x86_64-([^-]+).*/x86_64-$1/ ) {
        unshift @paths, catdir($baseDir, $binArch);
    }
    elsif ( $class->{osDetails}->{'binArch'} eq 'armhf-linux' ) {
        push @paths, catdir($baseDir, 'arm-linux');
    }

    Slim::Utils::Misc::addFindBinPaths(@paths);
}
```

**Source: `Slim::Utils::PluginManager` plugin Bin registration** (lines 340-363 of `/usr/share/perl5/Slim/Utils/PluginManager.pm`):
```perl
if (-d $binDir) {
    Slim::Utils::OSDetect::getOS()->initSearchPath($binDir);

    my $osDetails = Slim::Utils::OSDetect::details();
    my $binArch = $osDetails->{'binArch'};
    my @paths = ( catdir($binDir, $binArch), catdir($binDir, $^O), $binDir );

    if ( $binArch =~ /i386-linux/i ) {
        my $arch = $Config::Config{'archname'};
        if ( $arch && $arch =~ s/^x86_64-([^-]+).*/x86_64-$1/ ) {
            unshift @paths, catdir($binDir, $arch);
        }
    }
    elsif ( $binArch && $binArch eq 'armhf-linux' ) {
        push @paths, catdir($binDir, 'arm-linux');
    }

    Slim::Utils::Misc::addFindBinPaths( @paths );
}
```

**Current Helper.pm init() pattern** (lines 20-32):
```perl
sub init {
    # aarch64 can fall back to armhf binaries
    if ( !main::ISWINDOWS && !main::ISMAC
         && Slim::Utils::OSDetect::details()->{osArch} =~ /^aarch64/i ) {
        Slim::Utils::Misc::addFindBinPaths(
            catdir(Plugins::SpotOn::Plugin->_pluginDataFor('basedir'), 'Bin', 'armhf-linux')
        );
    }

    $prefs->setChange( sub {
        $helper = $helperVersion = $helperCapabilities = undef;
    }, 'binary') if !main::SCANNER;
}
```

**Current Helper.pm _findBin() pattern** (lines 116-149):
```perl
sub _findBin {
    my ($checkerCb, $customFirst) = @_;

    my @candidates = (HELPER);    # 'spoton'
    my $binary;

    if (Slim::Utils::OSDetect::OS() eq 'unix') {
        # on 64-bit x86, try the x86_64 build first
        if ( $Config::Config{'archname'} =~ /x86_64/ ) {
            push @candidates, HELPER . '-x86_64';
        }
    }

    # Custom override first (LMS-10 preparation)
    unshift @candidates, HELPER . '-custom';

    foreach my $name (@candidates) {
        my $candidate = Slim::Utils::Misc::findbin($name) || next;
        $candidate = Slim::Utils::OSDetect::getOS->decodeExternalHelperPath($candidate);
        next unless -f $candidate && -x $candidate;

        main::INFOLOG && $log->is_info && $log->info("Trying helper application: $candidate");

        if ( !$checkerCb || $checkerCb->($candidate) ) {
            main::INFOLOG && $log->is_info && $log->info("Found helper application: $candidate");
            $binary = $candidate;
            last;
        }
    }

    return $binary;
}
```

**Herger's Spotty init() ARM fallback pattern** (lines 19-23):
```perl
sub init {
    # aarch64 can potentially use helper binaries from armhf
    if ( !main::ISWINDOWS && !main::ISMAC && Slim::Utils::OSDetect::details()->{osArch} =~ /^aarch64/i ) {
        Slim::Utils::Misc::addFindBinPaths(catdir(Plugins::Spotty::Plugin->_pluginDataFor('basedir'), 'Bin', 'arm-linux'));
    }
    # ...
}
```

**Herger's Spotty _findBin() architecture candidates pattern** (lines 136-175):
```perl
sub _findBin {
    my ($checkerCb, $customFirst) = @_;

    my @candidates = (HELPER);

    if (Slim::Utils::OSDetect::OS() eq 'unix') {
        if ( $Config::Config{'archname'} =~ /x86_64/ ) {
            if ($customFirst) {
                unshift @candidates, HELPER . '-x86_64';
            } else {
                push @candidates, HELPER . '-x86_64';
            }
        }
        elsif ( $Config::Config{'archname'} =~ /[3-6]86/ ) {
            if ($customFirst) {
                unshift @candidates, HELPER . '-i386';
            } else {
                push @candidates, HELPER . '-i386';
            }
        }
        elsif ( $Config::Config{'archname'} =~ /(aarch64|arm).*linux/ ) {
            if ($customFirst) {
                unshift @candidates, HELPER . '-hf', HELPER . '-muslhf';
            } else {
                push @candidates, HELPER . '-hf', HELPER . '-muslhf';
            }
        }
    }

    unshift @candidates, HELPER . '-custom';
    # ...
}
```

**Key difference from Spotty:** SpotOn uses ONE binary name `spoton` per platform directory (not suffixed binaries like `spotty-x86_64`). LMS's `findbin('spoton')` searches `@findBinPaths` in order and returns the first match. So the directory search order IS the architecture priority order.

---

### `Plugins/SpotOn/Bin/x86_64-win64/` (new directory)

**Analog:** `Plugins/SpotOn/Bin/x86_64-linux/` (existing)

The Windows directory follows the same convention: one binary per directory. Binary name will be `spoton.exe` (LMS `findbin` appends `.exe` on Windows automatically -- line 113 of `Slim/Utils/Misc.pm`):

```perl
if (main::ISWINDOWS && $executable !~ /\.\w{3}$/) {
    $executable .= '.exe';
}
```

**No suffix needed.** The binary is simply named `spoton` (or `spoton.exe` on Windows) in each platform directory.

---

### `librespot-spoton/Cross.toml` (new config file)

**Analog:** `librespot-spoton/.cargo/config.toml` (existing)

**Current `.cargo/config.toml`** (lines 1-14):
```toml
# Cargo configuration for librespot-spoton
#
# LMS deployment binaries: use cross-rs with musl target for fully static binaries:
#   cross build --release --target x86_64-unknown-linux-musl
#
# Local development builds: plain `cargo build --release` (dynamically linked, dev only)
#
# NOTE: The previous +crt-static flag on x86_64-unknown-linux-gnu was removed.
# It causes proc-macro compilation to fail on Rust >= 1.87 with the error:
#   "cannot produce proc-macro for async-trait as target does not support these crate types"
# This is a known limitation: proc-macros must be compiled as dynamic libraries for the
# build host, but +crt-static prevents that on x86_64-unknown-linux-gnu.
# Use x86_64-unknown-linux-musl (with musl-tools or cross-rs) for static LMS deployment.
```

**Cargo.toml build features** (lines 9-14):
```toml
[features]
# Mirror the librespot-playback passthrough-decoder feature for --check capability reporting.
# When building without passthrough, omit this feature to make --check report false.
passthrough-decoder = []
default = ["passthrough-decoder"]
```

**Cargo.toml TLS configuration** (lines 20-23):
```toml
librespot-core      = { version = "0.8", default-features = false, features = ["rustls-tls-native-roots"] }
librespot-connect   = { version = "0.8", default-features = false, features = ["rustls-tls-native-roots"] }
librespot-discovery = { version = "0.8", default-features = false, features = ["with-libmdns", "rustls-tls-native-roots"] }
librespot-playback  = { version = "0.8", default-features = false, features = ["passthrough-decoder", "rustls-tls-native-roots"] }
```

D-05 confirms rustls-tls is already configured -- no system OpenSSL dependency, which is required for musl-static builds.

---

### Build script (cross-compile 6 targets)

No existing analog in the codebase. This is new infrastructure.

**Target mapping from CONTEXT.md D-08:**

| Rust Target Triple | Bin/ Directory | Binary Name |
|---|---|---|
| `x86_64-unknown-linux-musl` | `x86_64-linux/` | `spoton` |
| `aarch64-unknown-linux-musl` | `aarch64-linux/` | `spoton` |
| `armv7-unknown-linux-musleabihf` | `armhf-linux/` | `spoton` |
| `arm-unknown-linux-musleabihf` | `arm-linux/` | `spoton` |
| `i686-unknown-linux-musl` | `i386-linux/` | `spoton` |
| `x86_64-pc-windows-gnu` | `x86_64-win64/` | `spoton.exe` |

---

## Shared Patterns

### Binary Path Resolution Chain

**Source:** `Slim::Utils::OS::initSearchPath` + `Slim::Utils::PluginManager` + `Plugins::SpotOn::Helper`
**Apply to:** Helper.pm init() and _findBin()

The complete binary resolution chain for `findbin('spoton')`:

1. LMS PluginManager calls `initSearchPath(Bin/)` which registers: `Bin/{binArch}/`, `Bin/{$^O}/`, `Bin/`
2. LMS PluginManager itself adds: `Bin/{binArch}/`, `Bin/{$^O}/`, `Bin/` (duplicates filtered by addFindBinPaths)
3. LMS adds x86_64 priority: On x86_64, `Bin/x86_64-linux/` is unshifted to front
4. LMS adds armhf fallback: On armhf, `Bin/arm-linux/` is appended
5. SpotOn Helper.pm init() adds: additional fallback paths via `addFindBinPaths`
6. `_findBin()` iterates candidate names (`spoton-custom`, `spoton`) and calls `findbin()` for each

**Key insight:** On x86_64 Linux, LMS already registers `Bin/x86_64-linux/` FIRST in the search path. The current `_findBin()` code that pushes `HELPER . '-x86_64'` as a candidate name is redundant -- it searches for a binary named `spoton-x86_64`, which does not exist. This can be simplified.

### Binary Validation (--check)

**Source:** `Plugins/SpotOn/Helper.pm` lines 61-92
**Apply to:** All new binaries must pass this check

```perl
sub helperCheck {
    my ($candidate, $check, $dontSet) = @_;

    $$check = '' unless $check && ref $check;

    # Shell-safe quoting to prevent command injection from user-supplied binary paths
    (my $safe = $candidate) =~ s/'/'\\''/g;
    my $checkCmd = sprintf("'%s' -n 'SpotOn' --check", $safe);
    $$check = `$checkCmd 2>&1`;

    # CRITICAL: match 'spoton', not 'spotty'
    if ( $$check && $$check =~ /^ok spoton v([\d\.]+)/i ) {
        my $version = $1;

        if ( _versionCompare($version, MIN_BINARY_VERSION) < 0 ) {
            $log->warn("Binary version $version below minimum " . MIN_BINARY_VERSION);
            return 0;
        }

        return 1 if $dontSet;

        $helper        = $candidate;
        $helperVersion = $version;

        if ( $$check =~ /\n(.*)/s ) {
            $helperCapabilities = eval { from_json($1) } || {};
        }

        return 1;
    }
}
```

**Binary --check output contract** (from `main.rs` lines 240-253):
```rust
Mode::Check => {
    println!("ok spoton v{}", VERSION);
    let has_passthrough = cfg!(feature = "passthrough-decoder");
    let json = serde_json::json!({
        "version": VERSION,
        "discover-once": true,
        "lms-auth": false,
        "ogg-direct": has_passthrough,
        "passthrough": has_passthrough,
        "token-login": true,
    });
    println!("{}", json);
    process::exit(0);
}
```

Every cross-compiled binary must produce this exact output format when invoked with `--check`.

### Consumer Pattern: How DaemonManager and ProtocolHandler Use the Binary

**Source:** `Plugins/SpotOn/Connect/Daemon.pm` line 66
**Apply to:** No changes needed -- consumers are binary-path-agnostic

```perl
sub start {
    my $self = shift;

    require Proc::Background;

    my $helperPath = Plugins::SpotOn::Helper->get();
    # ...
    unless ($helperPath) {
        $log->warn("SpotOn Connect daemon: no helper binary found, cannot start");
        return;
    }
    # ...
    $self->_proc( Proc::Background->new(
        { 'die_upon_destroy' => 1, stdout => $port_w,
          ($stderr_fh ? (stderr => $stderr_fh) : ()) },
        $helperPath,
        @helperArgs,
    ) );
}
```

**Source:** `Plugins/SpotOn/Plugin.pm` lines 1247-1248 (updateTranscodingTable):
```perl
my ($helper) = Plugins::SpotOn::Helper->get();
my $helperName = $helper ? basename($helper) : 'spoton';
```

**Source:** `Plugins/SpotOn/Plugin.pm` lines 171-198 (_killOrphanedProcesses):
```perl
my ($helper) = Plugins::SpotOn::Helper->get();
if ($helper) {
    eval {
        if (main::ISWINDOWS) {
            my $name = basename($helper);
            $name =~ s/[^A-Za-z0-9._-]//g;
            if ($name) {
                system(qq{taskkill /IM "$name" /F 1>nul 2>&1});
            }
        } else {
            (my $safeHelper = $helper) =~ s/'/'\\''/g;
            my @pids = map { /^\s*(\d+)/ ? $1 : () } `pgrep -f '$safeHelper'`;
            # ...
        }
    };
}
```

All consumers call `Plugins::SpotOn::Helper->get()` which returns a single binary path. They do not need to know which platform was selected. No changes needed in DaemonManager, ProtocolHandler, Plugin.pm, or Settings.pm.

### Windows Binary Detection

**Source:** `Slim::Utils::Misc::findbin` lines 107-144
**Apply to:** Helper.pm _findBin() for Windows support

```perl
sub findbin {
    my $executable = shift;

    if (main::ISWINDOWS && $executable !~ /\.\w{3}$/) {
        $executable .= '.exe';
    }

    for my $search (@findBinPaths) {
        my $path = catdir($search, $executable);
        if (-x $path) {
            return $path;
        }
    }

    # For Windows we don't include the path in @findBinPaths so now search this
    if (main::ISWINDOWS && (my $path = File::Which::which($executable))) {
        return $path;
    }
    # ...
}
```

LMS appends `.exe` automatically on Windows. But the `@findBinPaths` may NOT contain the Windows Bin directory because `initSearchPath` sets `binArch` from `$Config::Config{'archname'}` which on Windows may not map to `x86_64-win64`. Helper.pm init() should add this path on Windows.

### Settings.pm Binary Display

**Source:** `Plugins/SpotOn/Settings.pm` lines 56-61
**Apply to:** No changes needed

```perl
my ($helperPath, $helperVersion) = Plugins::SpotOn::Helper->get();
$paramRef->{helperMissing} = string('PLUGIN_SPOTON_BINARY_MISSING') unless $helperPath;
$paramRef->{binaryVersion} = $helperVersion || '';
$paramRef->{binaryPath}    = $helperPath    || '';
```

Settings.pm is fully agnostic to platform -- it just displays whatever `Helper->get()` returns.

---

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| Build script (cross-compile) | utility | batch | No build scripts exist in the repo yet. Use cross-rs CLI commands documented in `.cargo/config.toml`. |
| `Cross.toml` | config | build | No cross-rs configuration exists yet. Standard cross-rs format. |

---

## Metadata

**Analog search scope:** `Plugins/SpotOn/`, `librespot-spoton/`, Herger's `Spotty-Plugin/`, LMS core (`Slim/Utils/`)
**Files scanned:** 14
**Pattern extraction date:** 2026-06-03
