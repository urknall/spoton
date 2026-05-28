# Phase 4: Single-Track Streaming - Pattern Map

**Mapped:** 2026-05-28
**Files analyzed:** 7 (5 modified, 2 new content areas in existing files)
**Analogs found:** 7 / 7

---

## File Classification

| Zu aendernde Datei | Rolle | Data Flow | Naechstes Analog | Match-Qualitaet |
|--------------------|-------|-----------|------------------|-----------------|
| `Plugins/SpotOn/ProtocolHandler.pm` | protocol-handler | request-response | `/tmp/Spotty-Plugin/ProtocolHandler.pm` | exact |
| `Plugins/SpotOn/Plugin.pm` (updateTranscodingTable, _killOrphanedProcesses, _trackItem/playall, initPlugin) | plugin/controller | event-driven | `/tmp/Spotty-Plugin/Plugin.pm` | exact |
| `Plugins/SpotOn/custom-convert.conf` | config | transform | `/tmp/Spotty-Plugin/custom-convert.conf` | exact |
| `Plugins/SpotOn/Settings.pm` | settings | request-response | `Plugins/SpotOn/Settings.pm` (self, extend existing) | self |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | template | request-response | `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (self, extend) | self |
| `Plugins/SpotOn/strings.txt` | i18n | — | `Plugins/SpotOn/strings.txt` (self, extend) | self |
| `Plugins/SpotOn/Helper.pm` (getCapability) | utility | request-response | `Plugins/SpotOn/Helper.pm` (self, already implemented) | self — read-only |

---

## Pattern Assignments

### `Plugins/SpotOn/ProtocolHandler.pm` (protocol-handler, request-response)

**Analog:** `/tmp/Spotty-Plugin/ProtocolHandler.pm` (Zeilen 51-62) und `Plugins/SpotOn/ProtocolHandler.pm` (aktueller Stand)

**Imports-Pattern** (ProtocolHandler.pm Zeilen 1-12 — unveraendert uebernehmen):
```perl
package Plugins::SpotOn::ProtocolHandler;

use strict;
use warnings;

use base qw(Slim::Formats::RemoteStream);

use Slim::Utils::Log;
use Slim::Utils::Prefs;

my $log   = logger('plugin.spoton');
my $prefs = preferences('plugin.spoton');
```

**Neu: `formatOverride` — dynamisches Format pro Player** (basiert auf Spotty ProtocolHandler.pm Zeilen 51-62 + RESEARCH.md Pattern 1, angepasst auf SpotOn):
```perl
sub formatOverride {
    my ($class, $song) = @_;

    my $client = $song->master;

    # Transcoding-Parameter fuer diesen Player injizieren (D-01, Pattern 1)
    require Plugins::SpotOn::Plugin;
    Plugins::SpotOn::Plugin->updateTranscodingTable($client);

    # Format-Capabilities dieses Players abfragen
    my @formats = Slim::Player::CapabilitiesHelper::supportedFormats($client);

    # OGG-Direct: nur wenn Player OGG kann UND Binary passthrough unterstuetzt (STR-05, A2-Mitigation)
    if (grep { $_ eq 'ogg' } @formats) {
        if (Plugins::SpotOn::Helper->getCapability('passthrough')) {
            return 'ogg';
        }
    }

    # FLAC: Standard-Fallback fuer alle modernen Player (D-04)
    return 'flc';
}
```

**Bereits implementiert — unveraendert beibehalten** (ProtocolHandler.pm Zeilen 14-37):
```perl
sub contentType        { 'son' }
sub isRemote           { 1 }
sub canDirectStream    { 0 }
sub getFormatForURL    { 'flc' }   # statischer Fallback
sub canSeek            { Slim::Utils::Versions->compareVersions($::VERSION, '7.9.1') >= 0 }
sub canTranscodeSeek   { Slim::Utils::Versions->compareVersions($::VERSION, '7.9.1') >= 0 }

sub getSeekData {
    my ($class, $client, $song, $newtime) = @_;
    return { timeOffset => $newtime };
}
```

**Neuer Import benoetigt** (CapabilitiesHelper, noch nicht importiert):
```perl
# Nach dem bestehenden use-Block hinzufuegen:
use Slim::Player::CapabilitiesHelper;
```

---

### `Plugins/SpotOn/Plugin.pm` — `updateTranscodingTable` (plugin, event-driven)

**Analog:** `/tmp/Spotty-Plugin/Plugin.pm` Zeilen 234-282

**Imports-Ergaenzung** (oben in Plugin.pm, nach bestehenden uses):
```perl
use File::Basename;
use File::Spec::Functions qw(catdir);
use Slim::Player::TranscodingHelper;
use Slim::Player::Client;
```

**Neue Konstante** (nach bestehenden use-Zeilen):
```perl
use constant KILL_PROCESS_INTERVAL => 3600;    # Stundlicher Orphaned-Process-Cleanup (STR-10)
```

**Neues prefs->init — Ergaenzung um normalization** (Plugin.pm Zeile 35, bestehender init-Block erweitern):
```perl
$prefs->init({
    bitrate       => 320,
    normalization => 0,     # NEU: STR-08, Volume Normalization (globaler Toggle, Phase 4)
    binary        => '',
    clientId      => '',
    accounts      => {},
    activeAccount => '',
});
```

**updateTranscodingTable-Methode** (basiert auf Spotty Plugin.pm Zeilen 234-282, angepasst auf SpotOn):
```perl
sub updateTranscodingTable {
    my ($class, $client) = @_;

    my $bitrate     = $prefs->get('bitrate') || 320;
    my $normalize   = $prefs->get('normalization') || 0;    # Phase 4: globaler Toggle (D-06)

    my $serverPrefs = preferences('server');
    my $cacheDir    = catdir($serverPrefs->get('cachedir'), 'spoton');

    # Cache-Verzeichnis anlegen wenn noetig (Pattern 4 aus RESEARCH.md)
    unless (-d $cacheDir) {
        require File::Path;
        File::Path::make_path($cacheDir);
    }

    my ($helper) = Plugins::SpotOn::Helper->get();
    my $helperName = $helper ? basename($helper) : 'spoton';

    my $commandTable = Slim::Player::TranscodingHelper::Conversions();
    foreach my $key (keys %$commandTable) {
        next unless $key =~ /^son-/ && $commandTable->{$key} =~ /single-track/;

        # Cache-Pfad injizieren (Pitfall 4: Regex erwartet beliebige Zeichen ausser ")
        $commandTable->{$key} =~ s/-c "[^"]*"/-c "$cacheDir"/g;

        # Bitrate injizieren (oder entfernen wenn leer, z.B. kein bitrate-Flag gewuenscht)
        $commandTable->{$key} =~ s/--bitrate \d{2,3}/--bitrate $bitrate/;

        # Binary-Name aktualisieren (LMS-10 Vorbereitung, Custom-Binary)
        $commandTable->{$key} =~ s/\[spoton[^\]]*\]/[$helperName]/g;

        # Volume Normalisation: immer zuerst entfernen, dann bei Bedarf einsetzen (STR-08)
        $commandTable->{$key} =~ s/ --enable-volume-normalisation//g;
        $commandTable->{$key} =~ s/( -n )/ --enable-volume-normalisation $1/ if $normalize;

        main::INFOLOG && $log->is_info && $log->info("updateTranscodingTable: $key => $commandTable->{$key}");
    }
}
```

---

### `Plugins/SpotOn/Plugin.pm` — `_killOrphanedProcesses` + Timer (plugin, event-driven)

**Analog:** `/tmp/Spotty-Plugin/Plugin.pm` Zeilen 328-361

**In initPlugin einfuegen** — nach dem bestehenden Token-Refresh-Timer (Plugin.pm Zeile 58, innerhalb `if (!main::SCANNER)`):
```perl
# Orphaned-Process-Cleanup-Timer starten (STR-10)
Slim::Utils::Timers::killTimers($class, \&_killOrphanedProcesses);
Slim::Utils::Timers::setTimer(
    $class,
    Time::HiRes::time() + KILL_PROCESS_INTERVAL,
    \&_killOrphanedProcesses
);
```

**Neue Methode `_killOrphanedProcesses`** (basiert auf Spotty Plugin.pm Zeilen 328-361):
```perl
sub _killOrphanedProcesses {
    my ($class) = @_;

    Slim::Utils::Timers::killTimers($class, \&_killOrphanedProcesses);

    my $isBusy = 0;
    for my $client (Slim::Player::Client::clients()) {
        if ($client->isPlaying()) {
            main::DEBUGLOG && $log->is_debug && $log->debug("Player " . $client->name() . " is busy, skipping cleanup");
            $isBusy = 1;
            last;
        }
    }

    unless ($isBusy) {
        my ($helper) = Plugins::SpotOn::Helper->get();
        if ($helper) {
            eval {
                if (main::ISWINDOWS) {
                    my $name = basename($helper);
                    system("taskkill /IM $name /F 1>nul 2>&1");
                } else {
                    # PHASE-5-NOTE: Phase 5 muss hier Connect-PIDs ausschliessen (Pitfall 6)
                    `pkill -f "$helper"`;
                }
            };
            $@ && $log->warn("Could not kill orphaned spoton processes: $@");
        }
    }

    # Timer immer neu starten (auch wenn $isBusy)
    Slim::Utils::Timers::setTimer(
        $class,
        Time::HiRes::time() + KILL_PROCESS_INTERVAL,
        \&_killOrphanedProcesses
    );
}
```

---

### `Plugins/SpotOn/Plugin.pm` — `_trackItem` und `_albumTrackItem` (plugin, request-response)

**Analog:** `/tmp/Spotty-Plugin/OPML.pm` Zeile 1184 + RESEARCH.md Pattern 2

**`playall => 1` in `_trackItem` hinzufuegen** (Plugin.pm Zeilen 264-276 — im `%item`-Hash):
```perl
# VORHER (Zeile 272):
my %item = (
    name      => "$title \x{2014} $artist",
    line1     => $title,
    line2     => $artist . ($album ? " \x{2022} $album" : ''),
    url       => 'spotify://' . ($track->{uri} // ''),
    play      => 'spotify://' . ($track->{uri} // ''),
    on_select => 'play',
    image     => $image,
    duration  => $duration,
    type      => 'audio',
);

# NACHHER — playall hinzufuegen (D-09, D-10):
my %item = (
    name      => "$title \x{2014} $artist",
    line1     => $title,
    line2     => $artist . ($album ? " \x{2022} $album" : ''),
    url       => 'spotify://' . ($track->{uri} // ''),
    play      => 'spotify://' . ($track->{uri} // ''),
    on_select => 'play',
    playall   => 1,    # NEU: Kontext-Queueing (D-09/D-10) — XMLBrowser reiht alle Items ein
    image     => $image,
    duration  => $duration,
    type      => 'audio',
);
```

**`playall => 1` in `_albumTrackItem` hinzufuegen** (Plugin.pm Zeilen 910-924 — im `%item`-Hash):
```perl
# Im %item-Hash in _albumTrackItem (Zeile 911):
my %item = (
    name      => ($trackNum ? "$trackNum. " : '') . $title,
    line1     => ($trackNum ? "$trackNum. " : '') . $title,
    line2     => $line2,
    url       => 'spotify://' . ($track->{uri} // ''),
    play      => 'spotify://' . ($track->{uri} // ''),
    on_select => 'play',
    playall   => 1,    # NEU: Kontext-Queueing fuer Album-Track-Tap (D-09)
    image     => $image,
    duration  => $duration,
    type      => 'audio',
);
```

---

### `Plugins/SpotOn/custom-convert.conf` (config, transform)

**Analog:** Bestehende `Plugins/SpotOn/custom-convert.conf` (alle 4 Eintraege vorhanden) + `/tmp/Spotty-Plugin/custom-convert.conf` als Referenz

**Aktueller Stand** (alle Eintraege bereits korrekt, STR-07 T-Flag vorhanden):
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

**Was updateTranscodingTable an diesem Literal-Template aendert** (zur Laufzeit per Regex):
- `-c "$CACHE$"` → `-c "/pfad/zum/lms-cache/spoton"` (Pitfall 4: Regex `s/-c "[^"]*"/-c "$cacheDir"/g`)
- `--bitrate 320` → `--bitrate {96|160|320}` (aus pref, Regex `s/--bitrate \d{2,3}/--bitrate $bitrate/`)
- `[spoton]` → `[spoton-x86_64]` o.ae. (Binary-Name aus Helper, Regex `s/\[spoton[^\]]*\]/[$helperName]/g`)
- Volume Normalisation: ` --enable-volume-normalisation` wird nach `-n Squeezebox` eingefuegt wenn pref `normalization=1`

**Wichtig — $CACHE$ nicht ersetzen:** Der Literal-String `$CACHE$` im Conf-File bleibt stehen. `updateTranscodingTable` ersetzt ihn erst beim ersten Track-Start durch den echten Pfad. Das funktioniert, weil die Regex auf das Muster `-c "[^"]*"` matcht (das `$CACHE$` als Inhalt hat beim initialen Load).

---

### `Plugins/SpotOn/Settings.pm` (settings, request-response)

**Analog:** `Plugins/SpotOn/Settings.pm` selbst — Erweiterung des bestehenden Musters

**prefs()-Methode erweitern** (Settings.pm Zeile 29 — `normalization` hinzufuegen):
```perl
# VORHER:
sub prefs {
    return ($prefs, 'bitrate', 'binary', 'clientId');
}

# NACHHER:
sub prefs {
    return ($prefs, 'bitrate', 'binary', 'clientId', 'normalization');
}
```

**handler() — saveSettings-Block erweitern** (nach Zeile 48, nach dem Bitrate-Validierungsblock):
```perl
# NEU: normalization pref speichern (STR-08)
# Checkbox-Wert: 1 wenn angehakt, undef/leer wenn nicht — explizit auf 0/1 normieren
my $norm = $paramRef->{'pref_normalization'} ? 1 : 0;
$prefs->set('normalization', $norm);
```

**Bestehendes Muster fuer Bitrate-Validierung** (Settings.pm Zeilen 45-48 — als Referenz fuer normalization):
```perl
my %valid_bitrates = map { $_ => 1 } (96, 160, 320);
my $bitrate = $paramRef->{'pref_bitrate'};
$bitrate = 320 unless $valid_bitrates{$bitrate};
$prefs->set('bitrate', $bitrate);
```

---

### `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` (template, request-response)

**Analog:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` selbst — Bitrate-Block (Zeilen 15-21) als Muster fuer neuen Normalization-Toggle

**Bestehendes Bitrate-Pattern** (Zeilen 15-21 — als Kopier-Vorlage):
```html
[% WRAPPER setting title="PLUGIN_SPOTON_BITRATE" desc="" %]
    <select class="stdedit" name="pref_bitrate" id="pref_bitrate">
        <option value="320" [% IF prefs.pref_bitrate == 320 %]selected[% END %]>320 kbps</option>
        <option value="160" [% IF prefs.pref_bitrate == 160 %]selected[% END %]>160 kbps</option>
        <option value="96"  [% IF prefs.pref_bitrate == 96  %]selected[% END %]>96 kbps</option>
    </select>
[% END %]
```

**Neu hinzuzufuegender Normalization-Toggle** (D-06, STR-08 — nach dem Bitrate-Block einfuegen):
```html
[% WRAPPER setting title="PLUGIN_SPOTON_NORMALIZATION" desc="PLUGIN_SPOTON_NORMALIZATION_DESC" %]
    <input type="checkbox" class="stdedit" name="pref_normalization" id="pref_normalization"
        value="1" [% IF prefs.pref_normalization %]checked[% END %]/>
    <label for="pref_normalization">[% 'PLUGIN_SPOTON_NORMALIZATION_LABEL' | string %]</label>
[% END %]
```

**Hinweis zur Template-Struktur:** Der Bitrate-Block befindet sich aktuell als eigenstaendiger `WRAPPER setting`-Block (nicht in einem umschliessenden Streaming-Wrapper). Der neue Normalization-Block folgt direkt danach im gleichen Stil. Ein gemeinsamer "Streaming"-WRAPPER-Block (`PLUGIN_SPOTON_STREAMING_SETTINGS`) waere optisch sauberer aber erfordert Umstrukturierung des bestehenden Bitrate-Blocks.

---

### `Plugins/SpotOn/strings.txt` (i18n, —)

**Analog:** `Plugins/SpotOn/strings.txt` selbst — bestehendes Zwei-Sprachen-DE/EN-Pattern (Zeilen 1-218)

**Bestehendes String-Pattern** (Zeilen 21-24 als Muster):
```
PLUGIN_SPOTON_BITRATE
    DE  Bitrate
    EN  Bitrate
```

**Neu hinzuzufuegende Strings** (an Ende der strings.txt anhaengen):
```
PLUGIN_SPOTON_NORMALIZATION
    DE  Lautstaerkenormalisierung
    EN  Volume Normalization

PLUGIN_SPOTON_NORMALIZATION_DESC
    DE  
    EN  

PLUGIN_SPOTON_NORMALIZATION_LABEL
    DE  Lautstaerke normalisieren (librespot --enable-volume-normalisation)
    EN  Normalize volume (librespot --enable-volume-normalisation)
```

---

## Shared Patterns

### LMS Transcoding commandTable Zugriff
**Quelle:** `/tmp/Spotty-Plugin/Plugin.pm` Zeilen 265-281 (verifiziert)
**Gilt fuer:** `updateTranscodingTable` in Plugin.pm
```perl
my $commandTable = Slim::Player::TranscodingHelper::Conversions();
foreach my $key (keys %$commandTable) {
    next unless $key =~ /^son-/ && $commandTable->{$key} =~ /single-track/;
    # Regex-basierte Injektion — alles per s/// direkt in $commandTable->{$key}
}
```

### Timer-Pattern (Slim::Utils::Timers)
**Quelle:** `Plugins/SpotOn/Plugin.pm` Zeilen 53-59 (Token-Refresh-Timer — bestehendes Muster)
**Gilt fuer:** Orphaned-Process-Cleanup-Timer in `initPlugin` und `_killOrphanedProcesses`
```perl
Slim::Utils::Timers::killTimers($class, \&_callback);    # Duplikate verhindern
Slim::Utils::Timers::setTimer(
    $class,
    Time::HiRes::time() + INTERVAL,
    \&_callback
);
```

### Prefs-Zugriff Pattern
**Quelle:** `Plugins/SpotOn/Plugin.pm` Zeilen 17, 35-41 (bestehend)
**Gilt fuer:** updateTranscodingTable, Settings.pm
```perl
my $prefs = preferences('plugin.spoton');
$prefs->get('bitrate')         # lesen
$prefs->set('normalization', 1) # schreiben
```

### cstring i18n Pattern
**Quelle:** `Plugins/SpotOn/Plugin.pm` Zeile 100 ff. (bestehend)
**Gilt fuer:** Alle neuen OPML-Items
```perl
cstring($client, 'PLUGIN_SPOTON_STRING_KEY')
```

### OPML Audio-Item Pattern
**Quelle:** `Plugins/SpotOn/Plugin.pm` Zeilen 264-276 (`_trackItem`)
**Gilt fuer:** Alle Track-Items mit `playall`-Erweiterung
```perl
my %item = (
    name      => ...,
    line1     => ...,
    line2     => ...,
    url       => 'spotify://' . $uri,
    play      => 'spotify://' . $uri,
    on_select => 'play',
    playall   => 1,     # NEU
    image     => $image,
    duration  => $duration,
    type      => 'audio',
);
```

---

## Kein Analog gefunden

Keine Datei in diesem Phase ohne Analog — alle Aenderungen basieren auf verifizierten Patterns aus dem Codebase oder Spotty-Prior-Art.

---

## Metadata

**Analog-Suchbereich:** `Plugins/SpotOn/`, `/tmp/Spotty-Plugin/`, `/usr/share/perl5/Slim/Player/`
**Gescannte Dateien:** 12
**Pattern-Extraction-Datum:** 2026-05-28

### Kritische Abhängigkeiten zwischen Patterns

| Abhaengigkeit | Erklaerung |
|---------------|------------|
| `formatOverride` MUSS `updateTranscodingTable` aufrufen | Ohne den Aufruf laeuft `commandTable` mit alten Parametern (hardcoded Bitrate 320, falschem Cache-Pfad) |
| `updateTranscodingTable` MUSS aus `Plugin.pm` sein (nicht ProtocolHandler.pm) | `TranscodingHelper::Conversions()` ist in `Slim::Player::TranscodingHelper` — das Modul muss in Plugin.pm geladen sein |
| `$CACHE$` Literal im conf bleibt stehen | Der Regex `s/-c "[^"]*"/-c "$cacheDir"/g` matched auf den ersten Lauf den Wert `$CACHE$` — nur wenn LMS das conf-File eingelesen hat |
| `playall => 1` braucht ALLE Items im Feed | playall funktioniert nur wenn alle Items bereits geladen sind (paginierte Feeds: nur die aktuelle Seite) |
