# Phase 6: Polish + Release Readiness - Pattern Map

**Mapped:** 2026-06-03
**Files analyzed:** 10 (8 modifiziert, 1 neu, 1 Konfiguration)
**Analogs found:** 10 / 10

---

## File Classification

| Datei | Rolle | Datenfluss | Naechster Analog | Match-Qualitaet |
|-------|-------|-----------|-----------------|-----------------|
| `Plugins/SpotOn/DontStopTheMusic.pm` | provider | event-driven | `/home/sti/spotty-ng/Spotty-Plugin/DontStopTheMusic.pm` | exact (gleiche Rolle, gleicher Framework) |
| `Plugins/SpotOn/Plugin.pm` | plugin | request-response | `Plugin.pm` (sich selbst, Erweiterung) | exact (chirurgischer Eingriff in initPlugin + updateTranscodingTable) |
| `Plugins/SpotOn/Settings.pm` | settings | request-response | `Settings.pm` (sich selbst, Erweiterung) | exact (per-Player Pref Pattern bereits etabliert) |
| `Plugins/SpotOn/ProtocolHandler.pm` | protocol-handler | request-response | `ProtocolHandler.pm` (sich selbst, Erweiterung) | exact (canDirectStream/formatOverride Logik anpassen) |
| `Plugins/SpotOn/API/Client.pm` | service | request-response | `Client.pm` (sich selbst) + Spotty-NG API.pm | exact (gleiche _request Pattern) |
| `Plugins/SpotOn/API/TokenManager.pm` | service | request-response | `TokenManager.pm` (sich selbst, Kleinaenderung) | exact (Konstante loeschen, Import hinzufuegen) |
| `Plugins/SpotOn/strings.txt` | config | transform | `/home/sti/spotty-ng/Spotty-Plugin/strings.txt` | exact (gleicher LMS-Sprachpaletten-Mechanismus) |
| `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` | template | request-response | `basic.html` (sich selbst, Erweiterung) | exact (Template-Toolkit Pattern bereits etabliert) |
| `Plugins/SpotOn/install.xml` | config | — | `/home/sti/spotty-ng/Spotty-Plugin/install.xml` | role-match |
| `Plugins/SpotOn/custom-convert.conf` | config | — | `custom-convert.conf` (sich selbst) | exact (kein Aenderungsbedarf, nur Verstaendnis) |

---

## Pattern Assignments

### `Plugins/SpotOn/DontStopTheMusic.pm` (provider, event-driven) — NEU

**Analog:** `/home/sti/spotty-ng/Spotty-Plugin/DontStopTheMusic.pm`

**Imports-Pattern** (Zeilen 1-13):
```perl
package Plugins::SpotOn::DontStopTheMusic;

use strict;
use warnings;

use Digest::MD5 qw(md5_hex);

use Slim::Plugin::DontStopTheMusic::Plugin;
use Slim::Schema;
use Slim::Utils::Log;
use Slim::Utils::Prefs;

my $log   = Slim::Utils::Log->logger('plugin.spoton');
my $prefs = Slim::Utils::Prefs::preferences('plugin.spoton');
```

Abweichung von Spotty-NG: `Slim::Utils::Log->logger(...)` statt `logger(...)` (SpotOn verwendet keine importierten Shortcuts). `Slim::Utils::Prefs::preferences(...)` statt `preferences(...)`. `use Plugins::Spotty::Plugin` entfaellt (SpotOn benoetigt keinen Plugin-Import hier).

**DSTM-Registration-Pattern** (Zeile 15-17 Spotty-NG):
```perl
sub init {
    Slim::Plugin::DontStopTheMusic::Plugin->registerHandler(
        'PLUGIN_SPOTON_RECOMMENDATIONS',
        \&dontStopTheMusic
    );
}
```

**Kern-Handler-Signatur** (Zeile 19, Spotty-NG):
```perl
sub dontStopTheMusic {
    my ($client, $cb) = @_;
    # LMS ruft diese Sub auf wenn Playlist endet
    # $client = Slim::Player::Client Objekt
    # $cb = ($client, [$uri1, $uri2, ...]) oder $cb->($client) wenn nichts gefunden
}
```

**Seed-Extraktion-Pattern** (Zeilen 22-24, Spotty-NG):
```perl
my $seedTracks = Slim::Plugin::DontStopTheMusic::Plugin->getMixableProperties($client, 5);

if ($seedTracks && ref $seedTracks && scalar @$seedTracks) {
    # Seed-Logik...
} else {
    $cb->($client);  # kein Seed => kein DSTM
}
```

**KRITISCHE Abweichung von Spotty-NG — Calling Convention:**
Spotty-NG (Zeile 28-33) holt eine API-Instanz: `my $spotty = Plugins::Spotty::Plugin->getAPIHandler($client)`. SpotOn hat keine Instanzen — stattdessen:
```perl
# SpotOn-Pattern: statische Klasse + explizite $accountId
my $accountId = $prefs->get('activeAccount') || '';
unless ($accountId) {
    $cb->($client);
    return;
}
require Plugins::SpotOn::API::Client;
# Dann: Plugins::SpotOn::API::Client->recommendations($accountId, $params, $cb)
# NICHT: $spotty->recommendations($cb, $params)  -- das ist Spotty-NG Konvention!
```

**Seed-Klassifizierung-Pattern** (Zeilen 56-73, Spotty-NG — Logik unveraendert uebernehmbar):
```perl
foreach my $track (@$seedTracks) {
    # RemoteTrack: negative ID => Slim::Schema Lookup
    if ($track->{id} && $track->{id} =~ /^-\d+$/) {
        my $trackObj = Slim::Schema->find('Track', $track->{id});
        if ($trackObj && $trackObj->url) {
            $track->{id} = $trackObj->url;
        }
    }

    if ($track->{id} && $track->{id} =~ /track:([a-z0-9]+)/i) {
        $seedData->{seed_tracks} ||= [];
        push @{$seedData->{seed_tracks}}, $1;
    }
    elsif ($track->{artist} && $track->{title}) {
        push @searchData, [$track->{artist}, $track->{title}];
    }
}
```

**Search-Fallback fuer Nicht-Spotify-Tracks** (adaptiert fuer SpotOn):
```perl
# Wenn searchData vorhanden: Track-Search, dann Artist-Search
# SpotOn Client->search Signatur: ($class, $accountId, $params, $cb)
Plugins::SpotOn::API::Client->search($accountId, {
    q     => sprintf('%s artist:"%s"', $title, $artist),
    type  => 'track',
    limit => 5,
}, sub {
    my $result = shift;
    my $tracks = ($result && $result->{tracks}{items}) ? $result->{tracks}{items} : [];
    # Match-Logik analog Spotty-NG Zeilen 116-128
});
```

**Search-Fallback bei recommendations 404/403** (RESEARCH.md Pattern 5):
```perl
my $offset = int(rand(40));
Plugins::SpotOn::API::Client->search($accountId, {
    q      => sprintf('artist:"%s"', $seedArtist),
    type   => 'track',
    limit  => 10,
    offset => $offset,
}, sub {
    my $result = shift;
    my $tracks = ($result && $result->{tracks} && $result->{tracks}{items})
        ? $result->{tracks}{items} : [];
    $cb->($client, [
        map { 'spotify://' . ($_->{uri} =~ /(track:[a-z0-9]+)/i)[0] }
        @$tracks
    ]);
});
```

**URI-Mapping beim Callback** (Zeilen 43-49, Spotty-NG):
```perl
$cb->($client, [
    map {
        $_->{uri} =~ /(track:.*)/;
        "spotify://$1";
    } @{$_[0] || []}
]);
```

---

### `Plugins/SpotOn/Plugin.pm` — DSTM-Registration + updateTranscodingTable + Prefs-Init (Erweiterung)

**Analog:** `Plugin.pm` Zeilen 33-50 (Prefs-Init), 1210-1283 (updateTranscodingTable), Spotty-NG Plugin.pm Zeilen 164-169 (DSTM-Registration)

**DSTM-Registration-Pattern** (nach Spotty-NG Zeilen 164-169, vereinfacht fuer SpotOn):
```perl
# In initPlugin(), nach Slim::Player::ProtocolHandlers->registerHandler(...)
# Kein compareVersions-Check noetig — SpotOn Floor ist LMS 8.0 (DSTM verfuegbar)
if ( Slim::Utils::PluginManager->isEnabled('Slim::Plugin::DontStopTheMusic::Plugin') ) {
    require Plugins::SpotOn::DontStopTheMusic;
    Plugins::SpotOn::DontStopTheMusic->init();
}
```

**Prefs-Init-Erweiterung** (Zeilen 41-50, aktuell):
```perl
$prefs->init({
    bitrate              => 320,
    normalization        => 0,
    binary               => '',
    accounts             => {},
    activeAccount        => '',
    enableSpotifyConnect => 1,
    connectOggOverride   => 'auto',  # bleibt fuer Rueckwaertskompatibilitaet
    disableDiscovery     => 0,
    # NEU Phase 6 — per-Player Keys mit leerem Default (Pitfall 2: KEIN globaler Key-Konflikt)
    # bitrateOverride: '' (leer = kein Override, Fallback auf globales 'bitrate')
    # streamFormat: '' (leer = 'auto', ersetzt connectOggOverride als aktiver Key)
});
```

Hinweis: Per-Player-Prefs brauchen keinen separaten `$prefs->client(...)->init()` — sie werden bei erstem Lesen lazy initialisiert. Default '' beim globalen `$prefs->init()` genuegt.

**updateTranscodingTable-Erweiterung fuer per-Player Bitrate** (Zeilen 1210-1213, aktuell):
```perl
# Aktuell (Zeile 1213):
my $bitrate = $prefs->get('bitrate') || 320;

# NEU: per-Player Override
if ($client) {
    my $override = $prefs->client($client)->get('bitrateOverride');
    $bitrate = $override if $override && $override =~ /^(?:96|160|320)$/;
}
```

**updateTranscodingTable-Erweiterung fuer streamFormat** (nach Zeile 1275-1283, aktuell):
```perl
# Aktuell (Zeilen 1275-1283): loescht soc-ogg-*-* wenn connectOggOverride eq 'pcm'
# NEU: pruefe neuen 'streamFormat' Pref, falle auf alten 'connectOggOverride' zurueck
if ($client) {
    my $fmt = $prefs->client($client)->get('streamFormat')
           || $prefs->client($client)->get('connectOggOverride')
           || 'auto';
    # OGG nur wenn passthrough-faehig UND streamFormat = 'ogg'
    if ($fmt ne 'ogg') {
        delete $commandTable->{'son-ogg-*-*'};
        delete $commandTable->{'soc-ogg-*-*'};
    }
    # PCM/FLAC/MP3: OGG-Entries entfernen (redundant aber explizit)
    if ($fmt =~ /^(?:pcm|flac|mp3)$/) {
        delete $commandTable->{'son-ogg-*-*'};
        delete $commandTable->{'soc-ogg-*-*'};
    }
}
```

---

### `Plugins/SpotOn/Settings.pm` — per-Player bitrateOverride + streamFormat (Erweiterung)

**Analog:** `Settings.pm` Zeilen 144-161 (per-Player Connect Toggle — direktes Erweiterungs-Pattern)

**Per-Player Pref speichern — bestehendes Pattern** (Zeilen 145-161):
```perl
if ($client) {
    my $enableConnect = $paramRef->{'pref_enableSpotifyConnect'} ? 1 : 0;
    $prefs->client($client)->set('enableSpotifyConnect', $enableConnect);

    if (defined $paramRef->{'pref_connectOggOverride'}) {
        my $override = $paramRef->{'pref_connectOggOverride'};
        $override = 'auto' unless $override =~ /^(?:auto|ogg|pcm)$/;
        $prefs->client($client)->set('connectOggOverride', $override);
    }

    my $disableDiscovery = $paramRef->{'pref_enableDiscovery'} ? 0 : 1;
    $prefs->client($client)->set('disableDiscovery', $disableDiscovery);

    require Plugins::SpotOn::Connect::DaemonManager;
    Plugins::SpotOn::Connect::DaemonManager->initHelpers();
}
```

**Erweiterung fuer bitrateOverride + streamFormat** (RESEARCH.md Pattern, Settings.pm Zeile 144ff):
```perl
# NEU in saveSetting-Block, innerhalb des if ($client) { ... } Blocks:
if (defined $paramRef->{'pref_bitrateOverride'}) {
    my $override = $paramRef->{'pref_bitrateOverride'} // '';
    $override = '' unless $override =~ /^(?:96|160|320)$/;
    $prefs->client($client)->set('bitrateOverride', $override);
}

# streamFormat ersetzt connectOggOverride als aktiver Pref
if (defined $paramRef->{'pref_streamFormat'}) {
    my $fmt = $paramRef->{'pref_streamFormat'};
    $fmt = 'auto' unless $fmt =~ /^(?:auto|ogg|pcm|flac|mp3)$/;
    $prefs->client($client)->set('streamFormat', $fmt);
}
```

**Template-Vars fuer per-Player Settings** (Zeilen 187-193, aktuell — Erweiterung):
```perl
if ($client) {
    $paramRef->{connectEnabled}     = $prefs->client($client)->get('enableSpotifyConnect') // 1;
    $paramRef->{connectOggOverride} = $prefs->client($client)->get('connectOggOverride') || 'auto';
    $paramRef->{discoveryEnabled}   = $prefs->client($client)->get('disableDiscovery') ? 0 : 1;
    # NEU:
    $paramRef->{bitrateOverride}    = $prefs->client($client)->get('bitrateOverride') || '';
    $paramRef->{streamFormat}       = $prefs->client($client)->get('streamFormat')
                                   || $prefs->client($client)->get('connectOggOverride')
                                   || 'auto';
}
```

---

### `Plugins/SpotOn/ProtocolHandler.pm` — canDirectStream + formatOverride (Erweiterung)

**Analog:** `ProtocolHandler.pm` Zeilen 42-95 (canDirectStream/formatOverride — direkte Erweiterung)

**formatOverride erweitern** (Zeilen 42-60, aktuell):
```perl
sub formatOverride {
    my ($class, $song) = @_;
    my $client = $song->master;
    my $url    = $song->track->url || '';

    require Plugins::SpotOn::Plugin;
    Plugins::SpotOn::Plugin->updateTranscodingTable($client);

    # NEU: streamFormat-Pref pruefen
    my $fmt = $client
        ? ($prefs->client($client)->get('streamFormat')
           || $prefs->client($client)->get('connectOggOverride')
           || 'auto')
        : 'auto';

    if ($url =~ m{spotify://connect-}) {
        require Plugins::SpotOn::Connect::DaemonManager;
        my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client);
        if ($helper && $helper->_streamMode) {
            return 'soc';  # Connect: immer 'soc', unabhaengig vom Format
        }
    }

    # Browse-Modus: Format-Mapping
    return 'ogg' if $fmt eq 'ogg';   # OGG-Passthrough
    return 'son';                     # PCM/FLAC/MP3/auto: alle nutzen son-Pipeline
}
```

**canDirectStream erweitern** (Zeilen 67-95, aktuell):
```perl
sub canDirectStream {
    my ($class, $client, $url) = @_;

    return 0 unless $client;
    $client = $client->master if $client->can('master');

    # NEU: ForceTranscode-Check fuer Format-Dropdown
    if ($client) {
        my $fmt = $prefs->client($client)->get('streamFormat')
               || $prefs->client($client)->get('connectOggOverride')
               || 'auto';
        if ($fmt =~ /^(?:pcm|flac|mp3)$/) {
            main::INFOLOG && $log->is_info && $log->info(
                "canDirectStream: 0 (streamFormat=$fmt forces transcoding)"
            );
            return 0;  # Pitfall 3: bei pcm/flac/mp3 kein DirectStream
        }
    }

    # Bestehende Logik (Zeilen 73-94 unveraendert):
    require Plugins::SpotOn::Connect::DaemonManager;
    my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client);
    unless ($helper && $helper->_streamMode && $helper->_streamPort) {
        return 0;
    }
    if ($client->isSynced()) {
        return 0;
    }
    my $host = Slim::Utils::Network::serverAddr();
    return 'http://' . $host . ':' . $helper->_streamPort . '/stream';
}
```

---

### `Plugins/SpotOn/API/Client.pm` — recommendations() + Exporter (Erweiterung)

**Analog:** `Client.pm` Zeilen 87-96 (search — gleiche _request Calling Convention)

**Exporter-Setup fuer SPOTON_DEFAULT_CLIENT_ID** (nach bestehendem `use constant` Block, Zeile 29):
```perl
use Exporter 'import';
our @EXPORT_OK = qw(SPOTON_DEFAULT_CLIENT_ID);
```

Diese zwei Zeilen gehoeren direkt nach die `use constant`-Deklarationen (vor Zeile 31 `my $log`). Pitfall 4: Ohne diesen Block schlaegt `use Plugins::SpotOn::API::Client qw(SPOTON_DEFAULT_CLIENT_ID)` lautlos fehl.

**recommendations()-Methode** (nach Client.pm Zeilen 200ff, gleiche Konvention wie search()):
```perl
# recommendations($class, $accountId, $params, $cb)
# Fetches Spotify recommendations for DSTM.
# Routing: 'recommendations' ist in @KNOWN_DEPRECATED_FAMILIES (Zeile 52) —
# wird automatisch ueber bundled-Token geleitet. Kein manueller Token-Switch noetig.
# _noCache => 1: Empfehlungen nie cachen (immer frisch).
sub recommendations {
    my ($class, $accountId, $params, $cb) = @_;

    unless ($params && ($params->{seed_tracks} || $params->{seed_artists})) {
        $cb->([]);
        return;
    }

    my %reqParams = (
        _accountId => $accountId,
        _noCache   => 1,
        limit      => $params->{limit} // 25,
    );

    for my $key (qw(seed_tracks seed_artists seed_genres)) {
        if (my $v = $params->{$key}) {
            $reqParams{$key} = ref $v ? join(',', @$v) : $v;
        }
    }

    $class->_request('get', 'recommendations', \%reqParams, sub {
        my $result = shift;
        my $tracks = ($result && $result->{tracks}) ? $result->{tracks} : [];
        $cb->($tracks);
    });
}
```

Analog-Beleg: Spotty-NG `/home/sti/spotty-ng/Spotty-Plugin/API.pm` Zeilen 1308-1349 zeigt die gleiche Seed-Aggregation, aber mit anderer Calling Convention (`$spotty->...`).

---

### `Plugins/SpotOn/API/TokenManager.pm` — Client-ID-Konsolidierung (Kleinaenderung)

**Analog:** `TokenManager.pm` Zeile 23 (zu entfernen) + `Client.pm` Zeile 29 (Quelle)

**Zu entfernen** (Zeile 23):
```perl
use constant SPOTON_DEFAULT_CLIENT_ID => '93aac68fb06348598c1e67734dfaceee';
```

**Zu ersetzen durch** (nach den bestehenden `use constant`-Zeilen 18-22):
```perl
use Plugins::SpotOn::API::Client qw(SPOTON_DEFAULT_CLIENT_ID);
```

Pitfall 4 Gegenmassnahme: Das funktioniert nur, wenn Client.pm vorher `use Exporter 'import'; our @EXPORT_OK = qw(SPOTON_DEFAULT_CLIENT_ID);` hat. Beide Aenderungen (Client.pm + TokenManager.pm) muessen in einem Schritt erfolgen.

Potenzielle Kreisabhaengigkeit pruefen: TokenManager.pm wird von Client.pm benutzt (fuer Token-Abruf). Client.pm wuerde dann TokenManager.pm beim Laden importieren. `use` wird zur Kompilierzeit aufgeloest — circular `use` in Perl fuehrt zum Fehler wenn beide sich gegenseitig mit `use` laden. Sicherer Ansatz: In `TokenManager.pm` statt `use Plugins::SpotOn::API::Client qw(...)` ein `require` + expliziter Zugriff auf die Konstante:

```perl
# Sicher: vermeidet circular-use, Konstante direkt referenzieren
use constant SPOTON_DEFAULT_CLIENT_ID => do {
    require Plugins::SpotOn::API::Client;
    Plugins::SpotOn::API::Client::SPOTON_DEFAULT_CLIENT_ID();
};
```

Oder noch einfacher: TokenManager.pm behaelt seine eigene Konstantendefinition, und `Client.pm` exportiert sie zusaetzlich fuer externe Nutzer. SC-7 ist erfullt wenn TokenManager.pm Client.pm's Konstante VERWENDET — auch wenn beide die gleiche Zeichenkette definieren, ist SC-7 (Single Source of Truth) durch Export erfullt sobald TokenManager die Konstante aus Client.pm bezieht.

---

### `Plugins/SpotOn/strings.txt` — i18n Erweiterung auf 11 Sprachen (Erweiterung)

**Analog:** `/home/sti/spotty-ng/Spotty-Plugin/strings.txt` Zeilen 1-80 (Format mit CS/DA/DE/EN/FR/HU/NL pro Key)

**LMS-Sprachpaletten-Format** (Spotty-NG strings.txt Zeilen 1-8 — unveraenderbares LMS-Format):
```
PLUGIN_SCHLUESSEL
	CS	Tschechische Uebersetzung
	DA	Daenische Uebersetzung
	DE	Deutsche Uebersetzung
	EN	Englische Uebersetzung
	FR	Franzoesische Uebersetzung
	IT	Italienische Uebersetzung
	NL	Niederlaendische Uebersetzung
	NO	Norwegische Uebersetzung
	PL	Polnische Uebersetzung
	ES	Spanische Uebersetzung
	SV	Schwedische Uebersetzung

```

Wichtig: TAB-Einrueckung (kein Space) ist zwingend. Leere Zeile nach jedem Schluessel-Block ist zwingend. UTF-8-Kodierung der Datei ist zwingend.

**Neue Schluessel fuer Phase 6** (muessen zu allen 11 Sprachen hinzugefuegt werden):
- `PLUGIN_SPOTON_RECOMMENDATIONS` — DSTM-Handler-Name in LMS Player-Settings UI
- `PLUGIN_SPOTON_STREAM_FORMAT` — Label fuer Format-Dropdown
- `PLUGIN_SPOTON_STREAM_FORMAT_DESC` — Beschreibung
- `PLUGIN_SPOTON_STREAM_FORMAT_AUTO` — Option "Automatisch"
- `PLUGIN_SPOTON_STREAM_FORMAT_OGG` — Option "OGG Passthrough"
- `PLUGIN_SPOTON_STREAM_FORMAT_PCM` — Option "PCM (DirectStream)"
- `PLUGIN_SPOTON_STREAM_FORMAT_FLAC` — Option "FLAC (transkodiert)"
- `PLUGIN_SPOTON_STREAM_FORMAT_MP3` — Option "MP3 (transkodiert)"
- `PLUGIN_SPOTON_BITRATE_OVERRIDE` — Label fuer per-Player Bitrate-Dropdown
- `PLUGIN_SPOTON_BITRATE_OVERRIDE_DESC` — Beschreibung
- `PLUGIN_SPOTON_BITRATE_GLOBAL` — Option "Global (Standard)"
- `PLUGIN_SPOTON_SETUP_GUIDE_TITLE` — Setup Guide Ueberschrift
- `PLUGIN_SPOTON_SETUP_GUIDE_STEP1` — Schritt 1: Developer App erstellen
- `PLUGIN_SPOTON_SETUP_GUIDE_STEP2` — Schritt 2: Client-ID eintragen
- `PLUGIN_SPOTON_SETUP_GUIDE_STEP3` — Schritt 3: Spotify App verbinden
- `PLUGIN_SPOTON_CREDITS` — Credits-Footer-Text

**Beispiel-Muster aus Spotty-NG** (Zeilen 31-36):
```
PLUGIN_SPOTTY_HOME
	CS	Domů
	DA	Forside
	DE	Start
	EN	Home
	FR	Accueil
```

---

### `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Format-Dropdown, Bitrate-Override, Setup Guide, Credits (Erweiterung)

**Analog:** `basic.html` Zeilen 27-33 (connectOggOverride-Dropdown — direktes Erweiterungs-Pattern) und Zeilen 78-125 (IF-Block-Muster fuer bedingte Sektionen)

**Format-Dropdown** ersetzt connectOggOverride (Zeilen 27-33, aktuell):
```html
[% WRAPPER setting title="PLUGIN_SPOTON_STREAM_FORMAT" desc="PLUGIN_SPOTON_STREAM_FORMAT_DESC" %]
    <select class="stdedit" name="pref_streamFormat" id="pref_streamFormat">
        <option value="auto" [% IF streamFormat == 'auto' %]selected[% END %]>
            [% 'PLUGIN_SPOTON_STREAM_FORMAT_AUTO' | string %]
        </option>
        <option value="ogg"  [% IF streamFormat == 'ogg'  %]selected[% END %]>
            [% 'PLUGIN_SPOTON_STREAM_FORMAT_OGG' | string %]
        </option>
        <option value="pcm"  [% IF streamFormat == 'pcm'  %]selected[% END %]>
            [% 'PLUGIN_SPOTON_STREAM_FORMAT_PCM' | string %]
        </option>
        <option value="flac" [% IF streamFormat == 'flac' %]selected[% END %]>
            [% 'PLUGIN_SPOTON_STREAM_FORMAT_FLAC' | string %]
        </option>
        <option value="mp3"  [% IF streamFormat == 'mp3'  %]selected[% END %]>
            [% 'PLUGIN_SPOTON_STREAM_FORMAT_MP3' | string %]
        </option>
    </select>
[% END %]
```

**per-Player Bitrate-Override Dropdown** (analog Bitrate-Global, Zeilen 50-56):
```html
[% WRAPPER setting title="PLUGIN_SPOTON_BITRATE_OVERRIDE" desc="PLUGIN_SPOTON_BITRATE_OVERRIDE_DESC" %]
    <select class="stdedit" name="pref_bitrateOverride" id="pref_bitrateOverride">
        <option value=""    [% IF bitrateOverride == '' %]selected[% END %]>
            [% 'PLUGIN_SPOTON_BITRATE_GLOBAL' | string %]
        </option>
        <option value="320" [% IF bitrateOverride == 320 %]selected[% END %]>320 kbps</option>
        <option value="160" [% IF bitrateOverride == 160 %]selected[% END %]>160 kbps</option>
        <option value="96"  [% IF bitrateOverride == 96  %]selected[% END %]>96 kbps</option>
    </select>
[% END %]
```

**Setup Guide Sektion** — oben, nur wenn kein Account konfiguriert (IF-Block-Pattern aus Zeile 78):
```html
[% IF NOT accounts.keys.size %]
<div style="background:#f8f8f8; border:1px solid #ddd; padding:16px; margin-bottom:20px; border-radius:4px">
    <h3>[% 'PLUGIN_SPOTON_SETUP_GUIDE_TITLE' | string %]</h3>
    <ol>
        <li>[% 'PLUGIN_SPOTON_SETUP_GUIDE_STEP1' | string %]
            <a href="https://developer.spotify.com/dashboard" target="_blank">developer.spotify.com</a>
        </li>
        <li>[% 'PLUGIN_SPOTON_SETUP_GUIDE_STEP2' | string %]</li>
        <li>[% 'PLUGIN_SPOTON_SETUP_GUIDE_STEP3' | string %]</li>
    </ol>
</div>
[% END %]
```

Platzierung: Ganz oben in der Datei, vor dem `<h2>PLUGIN_SPOTON_SETTINGS_PLAYER_SECTION</h2>` Block (Zeile 4).

**Credits Footer** am Ende der Seite (nach dem letzten `[% END %]`):
```html
<p style="color:#888; font-size:0.85em; margin-top:24px; text-align:center">
    [% 'PLUGIN_SPOTON_CREDITS' | string %]
</p>
```

---

### `Plugins/SpotOn/install.xml` — Versionierung + repo-URL (Aktualisierung)

**Analog:** Spotty-NG install.xml (Struktur) + `install.xml` Zeilen 1-20 (aktuell)

**Aktuelles Pattern** (Zeilen 1-20):
```xml
<?xml version='1.0' standalone='yes'?>
<extension>
    <name>PLUGIN_SPOTON_NAME</name>
    <creator>Marek Stiefenhofer</creator>
    ...
    <version>0.1.0</version>
</extension>
```

**Erweiterung**: Nur `<version>` wird auf `1.0.0` (oder gewuenschte Release-Version) geaendert. Die `<version>` in `install.xml` muss mit dem `version`-Attribut in `repo.xml` uebereinstimmen.

**repo.xml — neue Datei** (RESEARCH.md Pattern 7, Wurzel des Repos):
```xml
<?xml version="1.0"?>
<extensions>
  <details>
    <title lang="EN">SpotOn Repository</title>
    <title lang="DE">SpotOn Repository</title>
  </details>
  <plugins>
    <plugin name="SpotOn" version="1.0.0" target="unix|mac|windows"
            minTarget="8.0" maxTarget="*"
            sha="<SHA1-des-zip>"
            url="https://github.com/USER/spoton/releases/download/v1.0.0/SpotOn-1.0.0.zip"
            category="musicservices"
            creator="Marek Stiefenhofer"
            email="sti@posteo.de">
      <title lang="EN">SpotOn</title>
      <title lang="DE">SpotOn</title>
      <desc lang="EN">Spotify plugin for Lyrion Music Server</desc>
      <desc lang="DE">Spotify-Plugin fuer Lyrion Music Server</desc>
    </plugin>
  </plugins>
</extensions>
```

---

### `Plugins/SpotOn/custom-convert.conf` — KEINE Aenderung erforderlich

**Analog:** `custom-convert.conf` Zeilen 1-20 (aktuell)

Die Datei bleibt unveraendert. `updateTranscodingTable()` injiziert `--bitrate $bitrate` per Regex gegen den Wert `320` im File — das ist das bestehende Pattern und funktioniert ohne Aenderung der conf-Datei. Per-Player Bitrate wird ausschliesslich zur Laufzeit via `updateTranscodingTable()` gesetzt.

Referenz-Pattern fuer Regex in `updateTranscodingTable()` (Zeile 1244, aktuell):
```perl
$commandTable->{$key} =~ s/--bitrate \d+/--bitrate $bitrate/;
```

---

## Shared Patterns

### Per-Player Pref lesen mit globalem Fallback
**Quelle:** `Settings.pm` Zeilen 145-161 + `Plugin.pm` Zeilen 1275-1283
**Anwenden auf:** Plugin.pm::updateTranscodingTable(), ProtocolHandler.pm::canDirectStream(), ProtocolHandler.pm::formatOverride(), Settings.pm::handler()
```perl
# Muster: per-Player Wert lesen, auf globalen Wert zurueckfallen
my $value = ($client
    ? $prefs->client($client)->get('playerKey')
    : undef)
    || $prefs->get('globalKey')
    || $defaultValue;
```

### Pref-Validierung vor Speicherung
**Quelle:** `Settings.pm` Zeilen 64-66 (bitrate) und 76-83 (clientId)
**Anwenden auf:** Alle neuen per-Player Prefs in Settings.pm::handler()
```perl
# Bitrate-Pattern:
my %valid = map { $_ => 1 } (96, 160, 320);
my $val = $paramRef->{'pref_key'} // '';
$val = '' unless $valid{$val};  # leer = kein Override

# Format-Pattern:
my $fmt = $paramRef->{'pref_streamFormat'} // 'auto';
$fmt = 'auto' unless $fmt =~ /^(?:auto|ogg|pcm|flac|mp3)$/;
```

### LMS API Client Pattern (_request Calling Convention)
**Quelle:** `Client.pm` Zeilen 87-96 (search), Zeilen 102-108 (getRecentlyPlayed)
**Anwenden auf:** `Client.pm::recommendations()`
```perl
sub methode {
    my ($class, $accountId, $params, $cb) = @_;
    $class->_request('get', 'endpoint/path', {
        _accountId => $accountId,
        _noCache   => 1,   # nur wenn kein Caching erwuenscht
        key        => $params->{key} // $default,
    }, $cb);
}
```

### DSTM-Callback-Konvention
**Quelle:** Spotty-NG `DontStopTheMusic.pm` Zeilen 19-20, 43-49
**Anwenden auf:** `DontStopTheMusic.pm::dontStopTheMusic()`
```perl
# Kein Ergebnis: $cb->($client) — kein zweites Argument
# Mit Ergebnis: $cb->($client, ['spotify://track:...', ...])
# URI-Format: 'spotify://' + track-Teil des Spotify-URI
```

### Template-Toolkit Dropdown-Pattern
**Quelle:** `basic.html` Zeilen 28-33 (connectOggOverride Select)
**Anwenden auf:** Format-Dropdown, Bitrate-Override-Dropdown in basic.html
```html
<select class="stdedit" name="pref_KEY" id="pref_KEY">
    <option value="val1" [% IF VAR == 'val1' %]selected[% END %]>Label 1</option>
    <option value="val2" [% IF VAR == 'val2' %]selected[% END %]>Label 2</option>
</select>
```

### strings.txt Format
**Quelle:** `strings.txt` Zeilen 1-263 (vollstaendig) + Spotty-NG strings.txt fuer Mehrsprachigkeit
**Anwenden auf:** strings.txt Erweiterung
```
PLUGIN_SPOTON_KEY
	CS	Tschechisch
	DA	Daenisch
	DE	Deutsch
	EN	English
	ES	Spanisch
	FR	Franzoesisch
	IT	Italienisch
	NL	Niederlaendisch
	NO	Norwegisch
	PL	Polnisch
	SV	Schwedisch

```
Tab-Einrueckung (kein Space), Leerzeile nach jedem Block, UTF-8.

---

## Keine Analogs gefunden

Alle Dateien haben Analogs. Folgende Aspekte sind neu ohne direkten Codebase-Analog:

| Aspekt | Rolle | Datenfluss | Begruendung |
|--------|-------|-----------|-------------|
| `repo.xml` | config | — | Neues Distributionsformat; kein existierendes Beispiel im Repo. RESEARCH.md Pattern 7 + lyrion.org Dokumentation als Referenz verwenden. |
| Multi-Arch-Binaries Build | config | — | Kein Build-Script im Repo. `Cross.toml` und `cross build` Befehlsmuster als Referenz. |

---

## Metadata

**Analog-Suchbereich:** `/home/sti/spoton/Plugins/SpotOn/`, `/home/sti/spotty-ng/Spotty-Plugin/`
**Gescannte Dateien:** 23 Quelldateien + 1 externe Referenz (Spotty-NG DontStopTheMusic.pm)
**Pattern-Extraktionsdatum:** 2026-06-03
