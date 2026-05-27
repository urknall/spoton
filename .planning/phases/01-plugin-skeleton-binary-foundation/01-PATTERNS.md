# Phase 1: Plugin Skeleton + Binary Foundation - Pattern Map

**Mapped:** 2026-05-27
**Files analyzed:** 9 new files to create
**Analogs found:** 9 / 9 (alle aus installiertem Spotty-Plugin unter `/usr/share/squeezeboxserver/Plugins/Spotty/`)

---

## File Classification

| Neue Datei | Role | Data Flow | Nächstes Analog | Match Quality |
|---|---|---|---|---|
| `Plugins/SpotOn/Plugin.pm` | module / plugin-root | request-response (initPlugin lifecycle) | `/usr/share/squeezeboxserver/Plugins/Spotty/Plugin.pm` | exact |
| `Plugins/SpotOn/ProtocolHandler.pm` | protocol-handler | request-response (URI routing) | `/usr/share/squeezeboxserver/Plugins/Spotty/ProtocolHandler.pm` | exact |
| `Plugins/SpotOn/Helper.pm` | utility / binary-mgmt | batch (binary discovery + subprocess check) | `/usr/share/squeezeboxserver/Plugins/Spotty/Helper.pm` | exact |
| `Plugins/SpotOn/Settings.pm` | settings-controller | request-response (HTTP form handler) | `/usr/share/squeezeboxserver/Plugins/Spotty/Settings.pm` | exact |
| `Plugins/SpotOn/strings.txt` | i18n config | — | `/usr/share/squeezeboxserver/Plugins/Spotty/strings.txt` | exact |
| `Plugins/SpotOn/install.xml` | manifest config | — | `/usr/share/squeezeboxserver/Plugins/Spotty/install.xml` | exact |
| `Plugins/SpotOn/custom-types.conf` | format config | — | `/usr/share/squeezeboxserver/Plugins/Spotty/custom-types.conf` | exact |
| `Plugins/SpotOn/custom-convert.conf` | transcoding config | pipeline (shell subprocess) | `/usr/share/squeezeboxserver/Plugins/Spotty/custom-convert.conf` | exact |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | settings template | request-response (TT2 render) | `/usr/share/squeezeboxserver/Plugins/Spotty/HTML/EN/plugins/Spotty/settings/basic.html` | exact |

---

## Pattern Assignments

### `Plugins/SpotOn/Plugin.pm` (module, request-response)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/Plugin.pm`

**Imports-Pattern** (Zeilen 1–35 des Analogs):
```perl
package Plugins::SpotOn::Plugin;

use strict;
use base qw(Slim::Plugin::OPMLBased);

use vars qw($VERSION);

use File::Basename;
use Slim::Utils::Log;
use Slim::Utils::Prefs;
use Slim::Utils::Strings qw(string);
use Slim::Utils::Timers;

use Plugins::SpotOn::Helper;
use Plugins::SpotOn::ProtocolHandler;

my $prefs = preferences('plugin.spoton');
my $log = Slim::Utils::Log->addLogCategory( {
    category     => 'plugin.spoton',
    defaultLevel => 'WARN',
    description  => 'PLUGIN_SPOTON',
    logGroups    => 'SCANNER',
} );
```
Hinweis: `OPML.pm` wird in Phase 1 noch NICHT importiert — nur der Stub-Feed.

**Prefs-Init-Pattern** (Zeilen 59–81 des Analogs, vereinfacht auf Phase-1-Scope):
```perl
$prefs->init({
    bitrate => 320,
    binary  => '',    # custom binary override (LMS-10, Phase 6)
});
```
Nur die zwei Schlüssel, die Phase 1 benötigt. Alle Spotty-spezifischen Prefs (`cleanupTags`, `iconCode` etc.) werden NICHT kopiert.

**Protocol-Handler-Registrierung** (Zeilen 123–127 des Analogs):
```perl
Slim::Player::ProtocolHandlers->registerHandler(
    'spotify',
    'Plugins::SpotOn::ProtocolHandler'
);
```

**Settings + Helper init** (Zeilen 120–127 des Analogs):
```perl
Plugins::SpotOn::Helper->init();
$VERSION = $class->_pluginDataFor('version');

if (main::WEBUI) {
    require Plugins::SpotOn::Settings;
    Plugins::SpotOn::Settings->new();
}
```

**OPMLBased-Registrierung** (Zeilen 130–136 des Analogs, angepasst):
```perl
$class->SUPER::initPlugin(
    feed   => \&handleFeed,    # lokale Sub in Plugin.pm — OPML.pm gibt es in Phase 1 nicht
    tag    => 'spoton',
    menu   => 'radios',
    is_app => 1,
    weight => 100,
    icon   => 'plugins/SpotOn/html/images/icon.png',
);
```

**Transcoding-Guard** (Zeilen 41–44 des Analogs):
```perl
if ( !main::TRANSCODING ) {
    $log->error('Transcoding is required for SpotOn to work');
    return;
}
```

**OPML handleFeed (Phase-1-Stub, in Plugin.pm direkt — kein separates OPML.pm)**:
```perl
# Muster aus: github.com/michaelherger/Spotty-Plugin/OPML.pm (verifiziert per RESEARCH.md)
sub handleFeed {
    my ($client, $callback, $args) = @_;

    if ( !Plugins::SpotOn::Helper->get() ) {
        $callback->({
            items => [{
                name => Slim::Utils::Strings::cstring($client, 'PLUGIN_SPOTON_BINARY_MISSING'),
                type => 'textarea',    # WICHTIG: 'textarea' nicht 'text' — sonst navigierbar
            }]
        });
        return;
    }

    # Phase 1 Placeholder
    $callback->({
        items => [{
            name => Slim::Utils::Strings::cstring($client, 'PLUGIN_SPOTON_NAME'),
            type => 'textarea',
        }]
    });
}
```

---

### `Plugins/SpotOn/ProtocolHandler.pm` (protocol-handler, request-response)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/ProtocolHandler.pm`

**Imports-Pattern** (Zeilen 1–18 des Analogs, auf Phase-1-Stub reduziert):
```perl
package Plugins::SpotOn::ProtocolHandler;

use strict;
use base qw(Slim::Formats::RemoteStream);

use Slim::Utils::Log;
use Slim::Utils::Prefs;

my $log   = logger('plugin.spoton');
my $prefs = preferences('plugin.spoton');
```

**Kern-Pattern — Pflicht-Methoden für Phase 1** (Zeilen 27, 75 des Analogs):
```perl
sub contentType    { 'son' }        # D-08: 'son', nicht 'spt'
sub isRemote       { 1 }
sub canDirectStream { 0 }           # KRITISCH: erzwingt Transcoding-Pipeline

sub getFormatForURL { 'flc' }       # Default-Pipeline (son→flc)

sub formatOverride {
    my ($class, $song) = @_;
    # Phase 4: updateTranscodingTable hier aufrufen
    return 'son';
}
```

**canSeek/canTranscodeSeek** (Zeilen 30–31 des Analogs):
```perl
sub canSeek          { Slim::Utils::Versions->compareVersions($::VERSION, '7.9.1') >= 0 }
sub canTranscodeSeek { Slim::Utils::Versions->compareVersions($::VERSION, '7.9.1') >= 0 }

sub getSeekData {
    my ($class, $client, $song, $newtime) = @_;
    return { timeOffset => $newtime };
}
```

---

### `Plugins/SpotOn/Helper.pm` (utility, batch/subprocess)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/Helper.pm`

**Imports + Konstante** (Zeilen 1–16 des Analogs):
```perl
package Plugins::SpotOn::Helper;

use strict;
use File::Spec::Functions qw(catdir);
use JSON::XS::VersionOneAndTwo;

use Slim::Utils::Log;
use Slim::Utils::Prefs;

use constant HELPER              => 'spoton';
use constant MIN_BINARY_VERSION  => '1.0.0';    # SpotOn-Erweiterung (nicht in Spotty)

my $prefs = preferences('plugin.spoton');
my $log   = logger('plugin.spoton');

my ($helper, $helperVersion, $helperCapabilities);
```
Hinweis: `File::Slurp` aus Spotty wird NICHT verwendet — kein Pi-Modell-Check in Phase 1.

**init()-Methode** (Zeilen 19–28 des Analogs):
```perl
sub init {
    # aarch64 kann als Fallback armhf-Binaries verwenden
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
Abweichung von Spotty: `armhf-linux` statt `arm-linux` als Fallback, da SpotOn `Bin/armhf-linux/` verwendet.

**helperCheck()-Methode** (Zeilen 90–118 des Analogs, angepasst + MIN_VERSION-Erweiterung):
```perl
sub helperCheck {
    my ($candidate, $check, $dontSet) = @_;

    $$check = '' unless $check && ref $check;

    my $checkCmd = sprintf('%s -n "SpotOn" --check', $candidate);
    $$check = `$checkCmd 2>&1`;

    # KRITISCH: 'spoton' nicht 'spotty' im Regex
    if ( $$check && $$check =~ /^ok spoton v([\d\.]+)/i ) {
        my $version = $1;

        # SpotOn-Erweiterung: Mindestversions-Prüfung
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

**_findBin()-Methode** (Zeilen 135–200 des Analogs, SpotOn-Vereinfachung):
```perl
sub _findBin {
    my ($checkerCb, $customFirst) = @_;

    my @candidates = (HELPER);        # 'spoton'

    if (Slim::Utils::OSDetect::OS() eq 'unix') {
        if ( $Config::Config{'archname'} =~ /x86_64/ ) {
            push @candidates, HELPER . '-x86_64';
        }
    }

    # Custom-Override zuerst (LMS-10-Vorbereitung)
    unshift @candidates, HELPER . '-custom';

    my $binary;
    foreach my $name (@candidates) {
        my $candidate = Slim::Utils::Misc::findbin($name) || next;
        $candidate = Slim::Utils::OSDetect::getOS->decodeExternalHelperPath($candidate);
        next unless -f $candidate && -x $candidate;

        if ( !$checkerCb || $checkerCb->($candidate) ) {
            $binary = $candidate;
            last;
        }
    }

    return $binary;
}
```

**get(), getCapability(), getVersion() Methoden** (Zeilen 30–132 des Analogs):
```perl
sub get {
    if ( !$helper && (my $candidate = $prefs->get('binary')) ) {
        helperCheck($candidate);
    }

    if (!$helper) {
        my $check;
        $helper = _findBin(sub { helperCheck(@_, \$check) }, 'custom-first');
        $log->warn("Didn't find SpotOn helper!") unless $helper;
    }

    return wantarray ? ($helper, $helperVersion) : $helper;
}

sub getCapability {
    my ($class, $key) = @_;
    return $helperCapabilities->{$key} if $helperCapabilities && defined $helperCapabilities->{$key};
    return undef;
}

sub getVersion {
    my $class = shift;
    $class->get() unless $helperVersion;
    return $helperVersion;
}
```

**Hilfsfunktion _versionCompare** (nicht in Spotty — SpotOn-Ergänzung):
```perl
sub _versionCompare {
    my ($v1, $v2) = @_;
    my @a = split /\./, $v1;
    my @b = split /\./, $v2;
    for my $i (0 .. $#b) {
        my $diff = ($a[$i] || 0) <=> ($b[$i] || 0);
        return $diff if $diff;
    }
    return 0;
}
```

---

### `Plugins/SpotOn/Settings.pm` (settings-controller, request-response)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/Settings.pm`

**Imports + Konstante** (Zeilen 1–19 des Analogs, vereinfacht):
```perl
package Plugins::SpotOn::Settings;

use strict;
use base qw(Slim::Web::Settings);

use Slim::Utils::Prefs;
use Slim::Utils::Strings qw(string);
use Plugins::SpotOn::Plugin;
use Plugins::SpotOn::Helper;

use constant SETTINGS_URL => 'plugins/SpotOn/settings/basic.html';

my $prefs = preferences('plugin.spoton');
```

**Pflicht-Methoden** (Zeilen 21–54 des Analogs):
```perl
sub new {
    my $class = shift;
    return $class->SUPER::new(@_);
}

sub name { return Slim::Web::HTTP::CSRF->protectName('PLUGIN_SPOTON_NAME') }

sub page { return Slim::Web::HTTP::CSRF->protectURI(SETTINGS_URL) }

sub prefs { return ($prefs, 'bitrate', 'binary') }
```

**handler()-Methode** (Zeilen 56–80 des Analogs, auf Phase-1-Scope reduziert):
```perl
sub handler {
    my ($class, $client, $paramRef, $callback, $httpClient, $response) = @_;

    my ($helperPath, $helperVersion) = Plugins::SpotOn::Helper->get();

    # Binary-Status an Template übergeben
    $paramRef->{helperMissing}   = string('PLUGIN_SPOTON_BINARY_MISSING') unless $helperPath;
    $paramRef->{binaryVersion}   = $helperVersion || '';
    $paramRef->{binaryPath}      = $helperPath    || '';

    if ($paramRef->{saveSettings}) {
        $prefs->set('bitrate', $paramRef->{'pref_bitrate'} || 320);
        # 'binary' wird von Slim::Web::Settings automatisch gespeichert (via prefs() Methode)
    }

    return $class->SUPER::handler($client, $paramRef, $callback, $httpClient, $response);
}
```

---

### `Plugins/SpotOn/strings.txt` (i18n config)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/strings.txt`

**Format-Muster** (Zeilen 1–20 des Analogs):
```
PLUGIN_SPOTON
	DE	SpotOn Spotify für Squeezebox
	EN	SpotOn Spotify for Squeezebox

PLUGIN_SPOTON_NAME
	EN	SpotOn

SON
	EN	SpotOn
```
Regeln (aus Spotty-Quelle verifiziert):
- Schlüssel-Zeile: kein führendes Leerzeichen
- Sprach-Zeile: genau ein Tab + zwei Großbuchstaben-Sprachcode + Tab + Wert
- Leerzeile zwischen Schlüsselblöcken
- `SON` (Großbuchstaben) registriert den Formatnamen in LMS (analog zu `SPT  EN  Spotty`)

**Mindest-Schlüsselsatz für Phase 1:**
```
PLUGIN_SPOTON
PLUGIN_SPOTON_NAME
PLUGIN_SPOTON_BINARY_MISSING
PLUGIN_SPOTON_BINARY_STATUS
PLUGIN_SPOTON_ACCOUNT_SETTINGS
PLUGIN_SPOTON_ACCOUNT_PLACEHOLDER
SON
```

---

### `Plugins/SpotOn/install.xml` (manifest config)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/install.xml` (Zeilen 1–22, vollständig gelesen)

```xml
<?xml version='1.0' standalone='yes'?>
<extension>
    <name>PLUGIN_SPOTON_NAME</name>
    <creator><!-- Autorenname --></creator>
    <defaultState>enabled</defaultState>
    <description>PLUGIN_SPOTON</description>
    <email><!-- E-Mail --></email>
    <category>musicservices</category>
    <id><!-- NEU GENERIEREN: uuidgen --></id>
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
Kritische Abweichungen von Spotty:
- `<id>` MUSS ein neues UUID sein — NIEMALS Spotty's `21cbb80e-67b8-44a8-a662-21c6c7ae5260` verwenden
- `<minVersion>8.0</minVersion>` (SpotOn-Floor; Spotty hat `7.7`)
- `<maxVersion>*</maxVersion>` — immer `*`, niemals eine Versionsnummer (P-35)
- Kein `<importmodule>` oder `<onlineLibrary>` in Phase 1

---

### `Plugins/SpotOn/custom-types.conf` (format config)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/custom-types.conf`

```
#########################################################################
#ID     Suffix          Mime Content-Type               Server File Type#
#########################################################################
son     son     audio/x-sb-spoton               audio
```
Spotty deklariert zusätzlich `spc` für Spotify Connect. SpotOn deklariert `spc` erst in Phase 5.

---

### `Plugins/SpotOn/custom-convert.conf` (transcoding config, pipeline)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/custom-convert.conf`

```
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
Hinweise:
- `[spoton]` ist der Binary-Platzhalter — LMS löst ihn über `findbin('spoton')` auf
- Alle vier Pipelines sind syntaktisch vollständig aber funktional erst ab Phase 4
- Spotty hat keine `ogg`-Pipeline — SpotOn fügt sie hinzu (D-09, `--passthrough`)
- Einrückung der Befehlszeile: genau ein Tab (LMS-Konvention aus Spotty verifiziert)

---

### `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (settings template, TT2)

**Analog:** `/usr/share/squeezeboxserver/Plugins/Spotty/HTML/EN/plugins/Spotty/settings/basic.html`

**Header + Binary-Fehlermeldung** (Zeilen 1–6 des Analogs):
```html
[% PROCESS settings/header.html %]
[% title = "PLUGIN_SPOTON" %]
[% IF helperMissing; WRAPPER setting title="PLUGIN_SPOTON_BINARY_STATUS" desc="" %]
    <div style="color: red">[% helperMissing %]</div>
[% END; END %]
```

**Binary-Status-Sektion (Phase 1 — angepasst):**
```html
[% WRAPPER setting title="PLUGIN_SPOTON_BINARY_STATUS" desc="" %]
    [% IF binaryPath %]
        <p>[% binaryPath %] (v[% binaryVersion %])</p>
    [% ELSE %]
        <div style="color: red">[% 'PLUGIN_SPOTON_BINARY_MISSING' | string %]</div>
    [% END %]
[% END %]
```

**Account-Sektion (Phase 1 — disabled):**
```html
[% WRAPPER setting title="PLUGIN_SPOTON_ACCOUNT_SETTINGS" desc="" %]
    <p style="color: grey">[% 'PLUGIN_SPOTON_ACCOUNT_PLACEHOLDER' | string %]</p>
[% END %]
```

**Bitrate-Einstellung:**
```html
[% WRAPPER setting title="PLUGIN_SPOTTY_BITRATE" desc="" %]
    <select class="stdedit" name="pref_bitrate">
        <option value="320" [% IF prefs.pref_bitrate == 320 %]selected[% END %]>320 kbps</option>
        <option value="160" [% IF prefs.pref_bitrate == 160 %]selected[% END %]>160 kbps</option>
        <option value="96"  [% IF prefs.pref_bitrate == 96  %]selected[% END %]> 96 kbps</option>
    </select>
[% END %]
```

**Footer:**
```html
[% PROCESS settings/footer.html %]
```

---

## Shared Patterns

### Prefs-Namespace
**Quelle:** `Plugin.pm` aus Spotty (Zeile 26)
**Anwenden auf:** alle `.pm`-Dateien
```perl
my $prefs = preferences('plugin.spoton');    # Spotty hat 'plugin.spotty'
```

### Logger
**Quelle:** `Plugin.pm` aus Spotty (Zeilen 30–35)
**Anwenden auf:** alle `.pm`-Dateien
```perl
my $log = logger('plugin.spoton');
# oder in Plugin.pm:
my $log = Slim::Utils::Log->addLogCategory( {
    category     => 'plugin.spoton',
    defaultLevel => 'WARN',
    description  => 'PLUGIN_SPOTON',
} );
```

### JSON-Parsing mit eval-Guard
**Quelle:** `Helper.pm` aus Spotty (Zeile 109)
**Anwenden auf:** `Helper.pm` (--check Response), später alle API-Antworten
```perl
my $data = eval { from_json($json_string) } || {};
# NIEMALS without eval — from_json wirft bei Parse-Fehler eine Exception
```

### Perl 5.10-Kompatibilität
**Anwenden auf:** alle `.pm`-Dateien
```perl
use strict;
# 'use feature' NUR wenn explizit benötigt (say, //, state)
# '//' (defined-or) erst ab 5.10 — ist OK laut CLAUDE.md-Floor
# '//=' ist ebenfalls OK
```

### OPML textarea für Status-Meldungen
**Quelle:** `OPML.pm` aus Spotty (aus RESEARCH.md verifiziert)
**Anwenden auf:** `Plugin.pm` handleFeed
```perl
# IMMER 'textarea' für nicht-navigierbare Informations-Items
{ type => 'textarea', name => cstring($client, 'PLUGIN_SPOTON_BINARY_MISSING') }
# NIEMALS 'text' — das wäre navigierbar
```

---

## Bin/-Verzeichnisstruktur

**Kein direktes Datei-Analog** — aber Muster aus LMS OS.pm + Spotty Bin/ abgeleitet.

**Spotty Bin/-Verzeichnisse** (live verifiziert):
```
Bin/aarch64-linux/
Bin/arm-linux/
Bin/darwin-thread-multi-2level/
Bin/i386-linux/
Bin/MSWin32-x86-multi-thread/
```

**SpotOn Bin/-Verzeichnisse** (empfohlene Struktur aus RESEARCH.md):
```
Bin/x86_64-linux/     → spoton    (x86_64 musl static)
Bin/aarch64-linux/    → spoton    (aarch64 musl static)
Bin/armhf-linux/      → spoton    (armv7hf musl static)
Bin/arm-linux/        → spoton    (armv6 Fallback)
Bin/i386-linux/       → spoton    (i686 musl static)
```
Abweichung von Spotty: `x86_64-linux/` als eigenes Verzeichnis (statt Spotty's Dateinamen-Trick `spotty-x86_64` in `i386-linux/`). `armhf-linux/` statt `arm-linux/` als primäres armv7-Verzeichnis.

---

## Kein Analog gefunden

Keine — alle Dateien haben direkte Analogs im installierten Spotty-Plugin.

---

## Kritische Anti-Patterns (aus RESEARCH.md)

| Anti-Pattern | Warum falsch | Korrekt |
|---|---|---|
| `Bin/x86_64-linux-gnu-thread-multi/` als Verzeichnisname | LMS OS.pm normalisiert Arch-Namen; dieser Pfad wird nie gesucht | `Bin/x86_64-linux/` |
| `maxVersion` auf konkrete Versionsnummer setzen | Plugin verschwindet nach LMS-Upgrade (P-35) | `<maxVersion>*</maxVersion>` |
| Spotty's GUID `21cbb80e-...` in `install.xml` | Plugin-Slot-Kollision mit Spotty | Neues UUID mit `uuidgen` |
| `type => 'text'` für Status-Hints | Item wird navigierbar | `type => 'textarea'` |
| `$checkCmd` mit `^ok spotty` Regex matchen | Binary gibt `ok spoton v...` aus — kein Match | Regex auf `^ok spoton v([\d\.]+)` |
| `helperCheck()` aus Event-Callback aufrufen (Backtick-Blocking) | Blockiert LMS Event-Loop | Nur aus `initPlugin()` Startup-Kontext aufrufen |

---

## Metadata

**Analog-Suchbereich:** `/usr/share/squeezeboxserver/Plugins/Spotty/` (installiertes Plugin, LMS 9.2.0)
**Dateien gescannt:** 9 Analog-Dateien vollständig gelesen
**Pattern-Extraction-Datum:** 2026-05-27
**Konfidenz:** HIGH — alle Analogs sind live auf dem Entwicklungsrechner verifiziert
