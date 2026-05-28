# Phase 4: Single-Track Streaming - Research

**Researched:** 2026-05-28
**Domain:** LMS Transcoding Pipeline, librespot `--single-track`, OPML Queueing, Player Format Capabilities
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Pipeline-Auswahl-Mechanismus — Claude waehlt (updateTranscodingTable vs. formatOverride). Research-Empfehlung: updateTranscodingTable (siehe Architecture Patterns).
- **D-02:** B&O/UPnPBridge OGG-Support — Research klaert Format-Support-Matrix.
- **D-03:** Race Condition LMS-11 — muss sauber geloest werden; Claude waehlt Loesung.
- **D-04:** FLAC als Standard-Fallback (konsistent mit bestehendem `getFormatForURL => 'flc'`).
- **D-05:** Globales Bitrate-Setting (96/160/320 kbps) in Phase 4. Per-Player in Phase 6.
- **D-06:** Volume Normalization Timing — Claude entscheidet (Phase 4 oder 6).
- **D-07:** Audio Cache Strategie — Claude entscheidet.
- **D-08:** Neuer "Streaming"-Abschnitt auf bestehender Settings-Seite (basic.html), unterhalb Auth.
- **D-09:** Kontext-Queueing: ganzes Album/Playlist in Queue, Playback startet beim angetippten Track. Fallback: ab angeklicktem Track.
- **D-10:** Kontextlose Tracks (Suche, Kueerzlich gehoert, Top Tracks) — Claude entscheidet.
- **D-11:** Gapless nice-to-have, nicht kritisch. Prioritaet: zuverlaessiges Playback.

### Claude's Discretion

- **D-01:** Pipeline-Auswahl: updateTranscodingTable EMPFOHLEN (Begruendung: Abschnitt Architecture Patterns)
- **D-02:** B&O OGG-Support: FLAC als Fallback empfohlen (Begruendung: Abschnitt Format-Support-Matrix)
- **D-03:** Race Condition: Per-Player-Schluessel im commandTable (Begruendung: Abschnitt Pitfall 1)
- **D-06:** Volume Normalization: Phase 4 als globaler Toggle (Begruendung: Abschnitt Architecture Patterns)
- **D-07:** Audio Cache: In Phase 4 immer deaktiviert (`--disable-audio-cache`); kein UI-Toggle
- **D-10:** Kontextlose Tracks: `playall => 1` auf alle sichtbaren Items (Suchergebnisse, Top Tracks, Kuerzlich gehoert)
- **D-11:** Gapless: Mit `--single-track` nicht erreichbar fuer FLAC/OGG; PCM kann gapless wenn Sample-Parameter gleich — aber nicht prioritaet

### Deferred Ideas (OUT OF SCOPE)

Keine — Discussion blieb im Phase-Scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| STR-01 | Single-track playback via librespot `--single-track` mode writing PCM to stdout | librespot 0.8.0 unterstuetzt `--single-track`; schreibt PCM S16LE nach stdout per Pipe-Backend [VERIFIED: CLAUDE.md] |
| STR-02 | FLAC transcoding pipeline als Default (`spt -> flc` via custom-convert.conf) | custom-convert.conf Eintrag vorhanden; muss dynamische Bitrate erhalten [VERIFIED: codebase] |
| STR-03 | PCM passthrough pipeline fuer faehige Player | `son pcm * *` Eintrag vorhanden [VERIFIED: codebase] |
| STR-04 | MP3 transcoding pipeline als Legacy-Fallback | `son mp3 * *` mit lame-Pipe vorhanden [VERIFIED: codebase] |
| STR-05 | OGG-Direct passthrough fuer native-OGG-Player | `son ogg * *` mit `--passthrough` vorhanden; Spieler-Erkennung muss in `formatOverride` eingebaut werden [VERIFIED: codebase] |
| STR-06 | Bitrate selection (96/160/320 kbps) via Plugin Settings | Pref `bitrate` bereits initialisiert; Settings-UI vorhanden; Injection via `updateTranscodingTable` [VERIFIED: codebase] |
| STR-07 | Seeking via `--start-position` in Transcoding Pipeline | `{START=--start-position %s}` Kapabilitaet vorhanden; `canSeek`/`canTranscodeSeek`/`getSeekData` in ProtocolHandler.pm implementiert [VERIFIED: codebase] |
| STR-08 | Volume Normalization optional | librespot-Flag `--enable-volume-normalisation`; Toggle in `updateTranscodingTable` via Regex-Injektion [VERIFIED: CLAUDE.md] |
| STR-09 | Gapless Playback | Mit `--single-track` (separater Prozess pro Track) nicht garantiert; PCM-Modus erlaubt LMS-seitiges Gapless wenn Parameter identisch — aber Nice-to-have |
| STR-10 | Hourly cleanup orphaned librespot processes | `Slim::Utils::Timers::setTimer` mit KILL_PROCESS_INTERVAL=3600; `pkill -f` auf Unix [VERIFIED: Spotty prior art] |
| STR-11 | Audio Cache management | In Phase 4 immer `--disable-audio-cache`; kein UI-Toggle (Claude's Discretion D-07) |
| LMS-11 | Transcoding table updated per-track, nicht global | `formatOverride` wird per Song/Player aufgerufen; `updateTranscodingTable($client)` injiziert Player-spezifische Werte [VERIFIED: LMS Song.pm source] |
</phase_requirements>

---

## Summary

Phase 4 dreht sich um das Zusammenspiel von drei Schichten: dem LMS Transcoding-System (`custom-convert.conf` + `TranscodingHelper`), dem librespot-Binary im `--single-track`-Modus, und dem OPML-Queueing-System in `XMLBrowser.pm`.

**Kernbefund 1 — updateTranscodingTable ist der richtige Ansatz:** `Song.pm::open()` ruft `formatOverride($song)` auf, um das Output-Format zu bestimmen. Das Format wird dann in `getConvertCommand2` verwendet, um den passenden `commandTable`-Eintrag zu finden. Der Witz: `commandTable` ist ein globaler Hash, der direkt per Referenz veraenderbar ist. Spotty nutzt `formatOverride` als Trigger, um per `updateTranscodingTable` alle `spt-*` Eintraege in diesem Hash dynamisch umzuschreiben (Bitrate, Cache-Pfad, Volume-Normalisation). Dieser Ansatz ist nachgewiesen und robust.

**Kernbefund 2 — playall fuer Kontext-Queueing:** LMS's `XMLBrowser.pm` unterstuetzt `playall => 1` als Flag auf Audio-Items. Wenn ein Track mit `playall => 1` ausgewaehlt wird, reiht LMS automatisch ALLE Items mit `playall => 1` aus dem gleichen Feed in die Queue ein und startet beim ausgewaehlten Index. Das ist der korrekte Mechanismus — kein manuelles `playlist loadtracks` notwendig. `_albumTrackItem` und `_trackItem` benoetigen nur dieses Flag.

**Kernbefund 3 — Race Condition (LMS-11):** `commandTable` ist global. Wenn zwei Player gleichzeitig streamen, koennte `updateTranscodingTable(Player_A)` die Eintraege aendern, waehrend `Player_B` ebenfalls startet. Loesung: Per-Player-Eintraege im `commandTable` via Player-spezifischen Profil-Schluessel (`son-flc-*-$CLIENTID$`). Alternativ (und einfacher): das Schreiben in `commandTable` per `formatOverride` ist synchron im LMS-Event-Loop — ein echter paralleler Race ist unmoeglich da LMS single-threaded ist. Das Risiko ist nur eine Sequenz-Collision: Player A laedt Track, Player B laedt Track waehrend A's formatOverride noch aktiv ist. Da LMS non-preemptiv ist, sind die Aufrufe serialisiert. Race Condition-Risiko ist damit GERING — aber fuer korrekte Implementierung empfiehlt sich dennoch der Player-spezifische Name-Ansatz (Spotty-Vorbild).

**Primaere Empfehlung:** `formatOverride` als Trigger + `updateTranscodingTable($client)` fuer dynamische Injection von Bitrate/Cache/Normalisation. `playall => 1` fuer Kontext-Queueing. `pkill -f` Cleanup-Timer alle 3600 Sekunden.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Format-Auswahl pro Player | ProtocolHandler.pm | LMS TranscodingHelper | `formatOverride()` wird von `Song::open()` aufgerufen; gibt Format zurueck, LMS waehlt commandTable-Profil |
| Transcoding-Parameter Injection (Bitrate, Cache, Normalisation) | Plugin.pm `updateTranscodingTable()` | ProtocolHandler.pm (als Trigger) | Spotty-Pattern: formatOverride ruft updateTranscodingTable auf |
| Seeking | ProtocolHandler.pm | custom-convert.conf `T`-Kapabilitaet | `canTranscodeSeek` + `getSeekData` + `{START=--start-position %s}` |
| Kontext-Queueing | Plugin.pm `_trackItem`/`_albumTrackItem` | XMLBrowser.pm (Framework) | `playall => 1` Flag; XMLBrowser sammelt alle Items und sendet loadtracks |
| Orphaned-Process-Cleanup | Plugin.pm Timer | OS pkill | Slim::Utils::Timers::setTimer alle 3600s |
| Settings UI | Settings.pm + basic.html | Plugin.pm prefs | Neuer "Streaming"-Abschnitt; Bitrate + Volume Normalisation |
| Audio Format Capabilities per Player | LMS CapabilitiesHelper | Slim::Player::Squeezebox2/SqueezePlay | `$client->formats()` gibt OGG/FLAC/PCM/MP3 pro Player-Typ zurueck |

---

## Standard Stack

### Core — Keine neuen Abhaengigkeiten

Phase 4 benoetigt keine neuen Abhaengigkeiten. Alle benotigten Module sind bereits vorhanden:

| Modul | Version | Zweck | Status |
|-------|---------|-------|--------|
| `Slim::Player::TranscodingHelper` | LMS 8.0+ | `Conversions()` Hashref fuer commandTable | Bereits genutzt (CLAUDE.md) |
| `Slim::Player::CapabilitiesHelper` | LMS 8.0+ | `supportedFormats($client)` fuer Format-Detection | Noch nicht genutzt — Phase 4 fuegt Nutzung hinzu |
| `Slim::Utils::Timers` | LMS 8.0+ | setTimer/killTimers fuer Cleanup | Bereits in Plugin.pm verwendet |
| `Slim::Utils::Prefs` | LMS 8.0+ | Bitrate/Normalisation Prefs | Bereits in Plugin.pm |
| librespot | 0.8.0 | `--single-track`, `--passthrough`, `--start-position`, `--enable-volume-normalisation` | Bestehendes Binary |

**Installation:** Keine npm/pip/cargo Pakete — Phase 4 ist reiner Perl/Conf-Code.

---

## Package Legitimacy Audit

Keine externen Packages in dieser Phase. Abschnitt entfaellt.

---

## Architecture Patterns

### System Architecture Diagram

```
Track-Tap (OPML)
       |
       v
XMLBrowser.pm
  playall=1 -> loadtracks mit allen Items im Feed, startIndex=angeklickter Track
  on_select='play' -> nur diesen Track
       |
       v
Song::open()
  -> formatOverride($song) in ProtocolHandler.pm
       |
       v
updateTranscodingTable($client) in Plugin.pm
  Liest: bitrate pref, normalisePref, cacheDir, Helper-Binary
  Schreibt: commandTable{'son-flc-*-*'} etc. per Regex
       |
       v
TranscodingHelper::getConvertCommand2($song, 'flc', ['R','I'], ...)
  Waehlt: son-flc-*-* oder son-ogg-*-* etc. je nach formatOverride-Rueckgabe
       |
       v
custom-convert.conf Kommando
  [spoton] -n Squeezebox -c "$CACHEDIR" --single-track $URL$ --bitrate 320 ...
  | [flac] -cs ...
       |
       v
librespot --single-track spotify:track:XYZ -> stdout PCM
  | flac encoder -> stdout FLAC stream -> LMS -> Player
```

Seeking:
```
LMS Seek-Request
  -> Song::canSeek() -> canTranscodeSeek() == 2
  -> Song::getSeekData(newtime) -> { timeOffset => N }
  -> Song::open(seekdata)
     -> transcoder{'start'} = timeOffset
     -> tokenizeConvertCommand: $START$ = "--start-position N"
  -> librespot --start-position N --single-track ...
```

Orphaned Process Cleanup:
```
Plugin::initPlugin
  -> Slim::Utils::Timers::setTimer(3600s) -> _killOrphanedProcesses
     -> pkill -f spoton (Unix) / taskkill (Windows)
     -> reschedule 3600s
```

### Recommended Project Structure

Keine neuen Verzeichnisse. Aenderungen in bestehenden Dateien:

```
Plugins/SpotOn/
├── Plugin.pm               # updateTranscodingTable(), _killOrphanedProcesses(), _trackItem(playall), initPlugin(timer)
├── ProtocolHandler.pm      # formatOverride() dynamisch, contentType 'son', getFormatForURL
├── custom-convert.conf     # $BITRATE$-Platzhalter hinzufuegen; OGG-Eintrag pruefen
├── Settings.pm             # normalization pref hinzufuegen zu prefs()
├── HTML/.../settings/basic.html  # "Streaming"-Abschnitt: Bitrate-Dropdown + Normalisation-Toggle
└── strings.txt             # neue Strings fuer Streaming-Abschnitt
```

### Pattern 1: updateTranscodingTable (Spotty-Pattern, verifiziert)

**Was:** `formatOverride` dient als Trigger-Hook; die eigentliche Logik liegt in `updateTranscodingTable`, welche den globalen `commandTable` per Regex aendert.

**Wann nutzen:** Jedes Mal, wenn ein neuer Track fuer einen Player gestartet wird (formatOverride wird von `Song::open()` aufgerufen).

**Warum NICHT `${plugin.spoton.bitrate}$` im conf-Datei nutzen:** TranscodingHelper cached pref-file-Substitutionen in `%binaries` nach dem ersten Lesen (`if (!exists $binaries{$placeholder})`). Aenderungen an der Pref werden danach nicht mehr aufgenommen. Dagegen: Regex in updateTranscodingTable laeuft pro Track frisch.

```perl
# Source: Spotty Plugin.pm updateTranscodingTable (verified pattern) + LMS TranscodingHelper.pm
sub formatOverride {
    my ($class, $song) = @_;

    # Trigger dynamic injection into commandTable
    require Plugins::SpotOn::Plugin;
    Plugins::SpotOn::Plugin->updateTranscodingTable($song->master);

    # Bestimme Format fuer diesen Player
    my $client = $song->master;
    my @formats = Slim::Player::CapabilitiesHelper::supportedFormats($client);

    # OGG-Direct: Player unterstuetzt OGG nativ -> passthrough (guenstig fuer CPU)
    return 'ogg' if grep { $_ eq 'ogg' } @formats;

    # FLAC: Default fuer alle modernen Player (squeezelite, SB2+)
    return 'flc';
}

sub updateTranscodingTable {
    my ($class, $client) = @_;

    my $prefs   = preferences('plugin.spoton');
    my $bitrate = $prefs->get('bitrate') || 320;

    my $serverPrefs = preferences('server');
    my $cacheDir = catdir($serverPrefs->get('cachedir'), 'spoton');

    my $normalize = $client
        ? ($prefs->client($client)->get('normalisation') || 0)
        : 0;

    my ($helper) = Plugins::SpotOn::Helper->get();
    require File::Basename;
    my $helperName = $helper ? File::Basename::basename($helper) : 'spoton';

    my $commandTable = Slim::Player::TranscodingHelper::Conversions();
    foreach my $key (keys %$commandTable) {
        next unless $key =~ /^son-/ && $commandTable->{$key} =~ /single-track/;

        $commandTable->{$key} =~ s/-c "[^"]*"/-c "$cacheDir"/g;
        $commandTable->{$key} =~ s/--bitrate \d{2,3}/--bitrate $bitrate/;
        $commandTable->{$key} =~ s/\[spoton[^\]]*\]/[$helperName]/g;

        # Volume Normalisation: immer neu setzen
        $commandTable->{$key} =~ s/ --enable-volume-normalisation//g;
        $commandTable->{$key} =~ s/( -n )/ --enable-volume-normalisation $1/ if $normalize;
    }
}
```

### Pattern 2: playall-Flag fuer Kontext-Queueing

**Was:** `playall => 1` auf allen Audio-Items in einem Album- oder Playlist-Feed signalisiert XMLBrowser, beim Tap auf eines dieser Items ALLE Items in die Queue zu laden und beim angeklickten Track zu starten.

**Beweislage:** `XMLBrowser.pm` Zeile 660: `if ($method =~ /^(add|play)$/ && $subFeed->{'items'}->[$playIndex]->{playall}) { $method .= 'all'; }`. Zeile 729: `if (!$url || defined($playIndex) && !$item->{'playall'}) { $playIndex-- ...; next; }` — nur Items mit `playall => 1` werden eingereiht. Zeile 768: `$client->execute(['playlist', 'loadtracks', 'listref', \@urls, undef, $playIndex])` startet bei `$playIndex`.

**Einschraenkung:** `playall` funktioniert NUR, wenn alle Items schon im Feed-Array vorhanden sind (d.h. der Feed wurde vollstaendig geladen). Bei paginierten Feeds (z.B. grosse Playlists) werden nur die bereits geladenen Items eingereiht. Das ist das erwartete Verhalten (Fallback: ab angeklicktem Track bis Ende der geladenen Seite).

```perl
# Source: XMLBrowser.pm Lines 660, 729, 768 (verified in /usr/share/perl5/Slim/Control/XMLBrowser.pm)
# In _albumTrackItem() und _trackItem(): playall => 1 hinzufuegen
sub _albumTrackItem {
    my ($client, $track, $albumImages, $albumArtist) = @_;
    # ... bestehender Code ...
    my %item = (
        name      => ...,
        url       => 'spotify://' . $track->{uri},
        play      => 'spotify://' . $track->{uri},
        on_select => 'play',
        playall   => 1,    # NEU: Kontext-Queueing (D-09)
        ...
    );
    return \%item;
}
```

**Fuer _trackItem (kontextlose Tracks):** `playall => 1` ebenfalls setzen (D-10 Claude's Discretion). Bei Top Tracks, Kuerzlich Gehoert, Suchergebnissen werden damit alle sichtbaren Tracks einer Seite eingereiht. Das ist das natuerliche LMS-Verhalten und konsistent mit Spotty/Qobuz.

### Pattern 3: Orphaned Process Cleanup

**Was:** Ein Slim::Utils::Timers-basierter Timer laeuft stundlich und killt verbleibende librespot-Prozesse.

**Warum notwendig:** librespot `--single-track` terminiert nach Track-Ende, aber Edge Cases (Plugin-Reload, LMS-Crash waehrend Playback, rapid track-skipping) koennen Prozesse hinterlassen.

```perl
# Source: Spotty Plugin.pm killHangingProcesses (verified pattern)
use constant KILL_PROCESS_INTERVAL => 3600;    # 1 Stunde

sub initPlugin {
    my $class = shift;
    # ... bestehender Code ...

    # Orphaned-process-Cleanup-Timer starten (STR-10)
    unless (main::SCANNER) {
        Slim::Utils::Timers::killTimers($class, \&_killOrphanedProcesses);
        Slim::Utils::Timers::setTimer(
            $class,
            Time::HiRes::time() + KILL_PROCESS_INTERVAL,
            \&_killOrphanedProcesses
        );
    }
}

sub _killOrphanedProcesses {
    my ($class) = @_;

    Slim::Utils::Timers::killTimers($class, \&_killOrphanedProcesses);

    # Nur killen wenn kein Player gerade spielt
    my $isBusy = 0;
    for my $client (Slim::Player::Client::clients()) {
        if ($client->isPlaying()) {
            $isBusy = 1;
            last;
        }
    }

    unless ($isBusy) {
        my ($helper) = Plugins::SpotOn::Helper->get();
        if ($helper) {
            if (main::ISWINDOWS) {
                require File::Basename;
                my $name = File::Basename::basename($helper);
                system("taskkill /IM $name /F 1>nul 2>&1");
            } else {
                `pkill -f "$helper"`;
            }
        }
    }

    # Timer neu starten
    Slim::Utils::Timers::setTimer(
        $class,
        Time::HiRes::time() + KILL_PROCESS_INTERVAL,
        \&_killOrphanedProcesses
    );
}
```

### Pattern 4: Cache-Verzeichnis fuer librespot

**Was:** librespot benoetigt ein beschreibbares Verzeichnis fuer Credentials (`-c` Flag). Wir nutzen ein Unterverzeichnis im LMS-Cache.

```perl
# Source: Spotty AccountHelper.pm cacheFolder (verified pattern) + LMS prefs
use File::Spec::Functions qw(catdir);

sub _getLibrespotCacheDir {
    my $serverPrefs = preferences('server');
    my $cacheDir = catdir($serverPrefs->get('cachedir'), 'spoton');
    unless (-d $cacheDir) {
        require File::Path;
        File::Path::make_path($cacheDir);
    }
    return $cacheDir;
}
```

**Wichtig:** In Phase 4 verwenden wir ein einziges Cache-Verzeichnis fuer alle Accounts (kein Account-spezifisches Unterverzeichnis wie Spotty). Da Phase 4 nur `--single-track` nutzt (keine Connect-Daemons), ist ein gemeinsames Cache-Dir ausreichend.

### Anti-Patterns to Avoid

- **Pref-File-Substitution `${plugin.spoton.bitrate}$`:** TranscodingHelper cached das Ergebnis nach dem ersten Lesen — Pref-Aenderungen greifen nicht. Immer `updateTranscodingTable` nutzen.
- **Globales updateTranscodingTable ohne Client:** Wenn der Client `undef` ist, werden alle Player-spezifischen Settings (Normalisation) auf Default gesetzt. Immer mit `$song->master` aufrufen.
- **`formatOverride` ohne updateTranscodingTable-Aufruf:** `formatOverride` gibt das Format zurueck, aber das Kommando in `commandTable` hat noch die alten Parameter (Bitrate 320 hardcoded, falschen Cache-Pfad). Beide muessen zusammen laufen.
- **OGG ohne --passthrough:** Der `son ogg * *` Eintrag setzt `--passthrough`, welches rohes Ogg Vorbis von librespot liefert. Ohne `--passthrough` wuerde librespot PCM ausgeben, das Format-Label waere aber `ogg` — der Player wuerde die Daten falsch interpretieren.
- **Audio-Cache aktivieren:** Mit `--single-track` hat jeder Track einen eigenen Prozess. Audio-Cache ist hier kontraproduktiv: er wuerde versuchen, den einzelnen Track zu cachen, belegt Disk-Space und hat keinen Vorteil (keine Wiederholung im Single-Track-Modus). Daher: `--disable-audio-cache` immer gesetzt (D-07).

---

## Don't Hand-Roll

| Problem | Nicht bauen | Stattdessen | Warum |
|---------|-------------|-------------|-------|
| Format-Capabilities pro Player | Eigene Lookup-Tabelle | `Slim::Player::CapabilitiesHelper::supportedFormats($client)` | LMS kennt alle Player-Typen (Squeezebox2, SqueezePlay, squeezelite, SB Touch etc.) und ihre Capabilities; Player-Liste ist erweiterbar |
| Gapless-Stitching | Eigene PCM-Pufferung | LMS-natives Gapless via PCM + gleiche Sample-Parameter | Squeezebox1.pm implementiert Gapless fuer PCM wenn channels/samplesize/samplerate identisch; funktioniert automatisch |
| Prozess-Tracking | PID-Datei oder eigene PID-Tabelle | `pkill -f $helper` / `taskkill` | Robust gegen Restarts; keine State-Verwaltung noetig |
| Transcoding-Parameter pro Player | Separate Tabelle | Regex-Injection in `commandTable` (Spotty-Pattern) | commandTable ist bereits das kanonische LMS-Transcoding-Register |
| Queue-Management | Eigene Playlist-Build-Logik | `playall => 1` + XMLBrowser | XMLBrowser implementiert loadtracks mit startIndex — kein Plugin-Code noetig |

---

## Format-Support-Matrix (D-02)

### Player-Typen und Ihre Format-Capabilities

| Player-Typ | OGG | FLAC | MP3 | PCM | Quelle |
|------------|-----|------|-----|-----|--------|
| squeezelite (SqueezePlay-kompatibel) | ✓ | ✓ | ✓ | ✓ | `/usr/share/perl5/Slim/Player/SqueezePlay.pm:59` `myFormats => [qw(ogg flc aif pcm mp3)]` [VERIFIED: LMS source] |
| Squeezebox2 / SB Touch / SB Radio | ✓ | ✓ | ✓ | ✓ | `/usr/share/perl5/Slim/Player/Squeezebox2.pm` `return qw(wma ogg flc aif pcm mp3)` [VERIFIED: LMS source] |
| Squeezebox 1 / SLIMP3 | ✗ | ✗ | ✓ | ✓ | Legacy-Hardware; kein FLAC/OGG [ASSUMED] |
| UPnPBridge / B&O | UNBEKANNT | Whrsch. ✓ | ✓ | ? | [ASSUMED — siehe unten] |

### B&O / UPnPBridge OGG-Support (D-02)

**Befund:** UPnPBridge ist ein LMS-Plugin das UPnP/DLNA-Geraete als LMS-Player integriert. Es koennte OGG unterstuetzen oder nicht — das haengt vom Geraet ab, nicht von LMS. Der Squeezelite-interne UPnPRenderer und B&O-Geraete annoncieren ihre DLNA-Capabilities via `urn:schemas-upnp-org:service:AVTransport`. B&O-Geraete unterstuetzen typischerweise FLAC und MP3, OGG Vorbis selten. [ASSUMED — nicht via LMS-Quelle verifizierbar ohne laufendes System]

**Empfehlung (Claude's Discretion D-02):** `formatOverride` gibt OGG nur zurueck, wenn `Slim::Player::CapabilitiesHelper::supportedFormats($client)` explizit `ogg` enthaelt. UPnPBridge-Players melden sich ueber ihren LMS-Client-Typ — wenn sie OGG nicht in ihren Capabilities melden, faellt `formatOverride` automatisch auf FLAC zurueck. Das ist sicher und korrekt.

**Fazit:** Die Capabilities-Abfrage loest D-02 automatisch — kein hardcodiertes Wissen ueber B&O/UPnP noetig.

---

## Pipeline-Auswahl: updateTranscodingTable vs. formatOverride-basiert (D-01)

### Entscheidung: updateTranscodingTable (Spotty-Pattern) EMPFOHLEN

**Analyse:**

| Kriterium | updateTranscodingTable | Statische conf + formatOverride |
|-----------|----------------------|--------------------------------|
| Bitrate dynamisch | ✓ (Regex-Injection) | ✗ (pref-file gecached, Aenderungen greifen nicht) |
| Cache-Pfad dynamisch | ✓ | Nur via updateTranscodingTable |
| Volume-Normalisation | ✓ | ✗ |
| Race Condition LMS-11 | Kein Risiko (LMS single-threaded event loop) | Kein Risiko |
| Bewaehrtes Pattern | ✓ Spotty-Plugin seit Jahren | ✗ Kein Referenz-Plugin |
| Wartbarkeit | Mittel (Regex auf Strings) | Gut (deklarativ) |
| LMS-Versionsabhaengigkeit | Keine | Keine |

**Warum die Variante mit pref-file `${plugin.spoton.bitrate}$` nicht geht:** `TranscodingHelper::tokenizeConvertCommand2` (Zeile 641-654) cached pref-file-Werte in `%binaries` via `if (!exists $binaries{$placeholder})`. Der Wert wird nur beim ersten Aufruf gelesen und nie aktualisiert. [VERIFIED: `/usr/share/perl5/Slim/Player/TranscodingHelper.pm` Zeilen 641-655]

**Warum kein echter Race Condition bei updateTranscodingTable:** LMS ist single-threaded (Perl event loop via `Slim::Utils::Timers`). `Song::open()` / `formatOverride()` werden sequentiell abgearbeitet. Zwei Player koennen nicht wirklich gleichzeitig `updateTranscodingTable` aufrufen. [VERIFIED: LMS-Architektur]

---

## Volume Normalization: Phase 4 oder Phase 6 (D-06)

### Entscheidung: Phase 4 als globaler Toggle EMPFOHLEN

**Begruendung:** Volume Normalization ist ein librespot-Flag (`--enable-volume-normalisation`), das im gleichen `updateTranscodingTable`-Aufruf injiziert wird wie Bitrate. Die Implementierungskosten in Phase 4 sind minimal. Per-Player-Override (unterschiedliche Normalisation fuer verschiedene Player) ist Phase 6 (LMS-08). Ein globaler Toggle in Phase 4 liefert sofortigen Nutzen.

**Methode (Claude's Discretion):** `basic` — einfacher, weniger CPU-intensiv als `dynamic`. `--normalisation-method` Flag optional (librespot default ist `dynamic` laut CLAUDE.md). Spotty nutzt keinen Methoden-Override — wir auch nicht in Phase 4.

**Pref:** `normalization => 0` (Standard: aus) im `$prefs->init()`.

---

## Gapless Playback (D-11)

### Machbarkeit-Analyse

**Mit `--single-track`:** Jeder Track startet einen neuen librespot-Prozess. Zwischen zwei Prozessen gibt es eine kurze Luecke (Prozess-Start-Zeit ~100-500ms). FLAC und OGG haben diese Luecke immer — der neue Prozess muss hochfahren, sich mit Spotify verbinden und den ersten Audio-Frame liefern.

**PCM-Gapless:** `Squeezebox1.pm` (betrifft alle modernen Player) erlaubt Gapless wenn `streamformat` identisch und channels/samplesize/samplerate identisch sind (Zeilen 132-157, verifiziert). Fuer PCM (`son pcm * *`) koennte LMS gapless streamen, WENN beide Tracks gleiche Parameter haben (Spotify: immer 44100Hz, 2ch, S16LE — also konstant). Aber: der neue librespot-Prozess muss noch starten, was eine minimale Luecke erzeugt.

**Fazit:** Echtes Gapless ist mit `--single-track` (separater Prozess) nicht moglich. PCM reduziert die Luecke minimal. Da Gapless Nice-to-have ist (D-11), keine besondere Massnahme. Standard-FLAC-Pipeline ist akzeptabel. [ASSUMED — kein live-Test moeglich, aber aus Architektur-Analyse evident]

---

## Common Pitfalls

### Pitfall 1: Race Condition bei updateTranscodingTable (LMS-11)

**Was passiert:** Player A und Player B spielen gleichzeitig. `updateTranscodingTable(Player_A)` aendert `commandTable`, dann kommt `updateTranscodingTable(Player_B)` — beim naechsten Track von A sind wieder Bs Parameter aktiv.

**Warum es passiert:** `commandTable` ist global; updateTranscodingTable aendert ALLE `son-*` Eintraege ohne Player-Differenzierung.

**Wie vermeiden:** Da LMS single-threaded ist, ist ein echter gleichzeitiger Aufruf unmoeglich. Das Risiko besteht nur bei schnell aufeinanderfolgenden Track-Starts (Skip-Skip-Skip). Mitigation: `updateTranscodingTable` wird in `formatOverride` aufgerufen — also unmittelbar vor der Transcoding-Entscheidung. Kein Zeitfenster fuer einen anderen Player dazwischen (ausser Timer-Callbacks, aber die laufen nicht mitten in einem formatOverride-Aufruf).

**Empfehlung fuer robuste Implementierung:** Globale Bitrate + normalization Einstellungen sind ohnehin global (Phase 4). Pro-Player-Differenzierung (Normalisation per Player) kommt in Phase 6. Daher reicht das globale Pattern voellig.

**Warnsignal:** Test mit zwei Squeezeplays die gleichzeitig unterschiedliche Tracks starten.

### Pitfall 2: formatOverride gibt 'ogg' zurueck, aber commandTable hat kein 'son ogg'-Eintrag

**Was passiert:** `formatOverride` gibt `'ogg'` zurueck, `getConvertCommand2` findet keinen `son-ogg-*-*` Eintrag (z.B. `son ogg * *` ist disabled oder syntaktisch falsch). LMS loggt "Couldn't create command line" und der Track spielt nicht.

**Warum es passiert:** `custom-convert.conf` Syntaxfehler, oder OGG-Eintrag fehlt.

**Wie vermeiden:** `son ogg * *` Eintrag in `custom-convert.conf` verifizieren. `formatOverride` mit FLAC-Fallback schreiben wenn OGG-Eintrag nicht verfuegbar.

**Warnsignal:** LMS-Log zeigt "Couldn't create command line for ogg playback".

### Pitfall 3: --passthrough ohne passthrough-decoder Feature in librespot

**Was passiert:** `son ogg * *` nutzt `--passthrough`. Wenn das librespot-Binary ohne `passthrough-decoder` Feature kompiliert wurde, ignoriert es das Flag und gibt trotzdem PCM aus — das Format-Label ist aber `ogg`, der Player interpretiert die PCM-Daten als OGG und bricht ab.

**Wie vermeiden:** `--check` JSON-Output des Binary pruefen auf passthrough-Capability. In `Helper.pm::getCapability()` auslesen. [ASSUMED — passthrough ist nicht in CLAUDE.md als Pflicht-Feature gelistet]

**Warnsignal:** Audio-Glitches oder Stille bei OGG-Direct-Player.

### Pitfall 4: $CACHE$ wird nicht substituiert

**Was passiert:** `custom-convert.conf` hat `-c "$CACHE$"`. `updateTranscodingTable` ersetzt `$CACHE$` via Regex `s/-c ".*?"/-c "$cacheDir"/g`. Falls der Eintrag beim LMS-Start noch nicht das Muster `-c ".*?"` hat (z.B. nach Plugin-Neuinstallation ist der Eintrag noch original), funktioniert die Substitution nicht.

**Wie vermeiden:** `$CACHE$` als Literal-String in `custom-convert.conf` stehenlassen. `updateTranscodingTable` ersetzt zuerst einmalig beim Plugin-Start (aus `initPlugin`), dann pro Track via `formatOverride`. Alternative: `$CACHE$` im Regex durch einen spezifischeren Pattern ersetzen, z.B. `s/-c "[^"]*"/-c "$cacheDir"/g` (was bereits im obigen Code-Beispiel so ist).

**Warnsignal:** librespot meldet "cannot create cache directory" oder ahnliche Fehler.

### Pitfall 5: canSeek == 2 erfordert 'T'-Capability im commandTable-Eintrag

**Was passiert:** `Song::canDoSeek()` gibt `2` zurueck (Transcoder-Seeking), wenn sowohl `canTranscodeSeek()` als auch ein `getConvertCommand2(... ['T'], ...)` erfolgreich ist. Das setzt voraus, dass der commandTable-Eintrag das `T`-Flag hat (`# RT:{START=--start-position %s}`). Fehlt `T`, faellt canSeek auf `1` zurueck (direktes Seeking ohne Transcoder — bei Remote-Streams nicht implementiert) oder `0`.

**Wie vermeiden:** `custom-convert.conf` Kapabilitaets-Zeile muss `T` enthalten: `# RT:{START=--start-position %s}`. [VERIFIED: bestehender custom-convert.conf hat das korrekt]

### Pitfall 6: pkill killt Connect-Daemons (Phase 5-Vorbereitung)

**Was passiert:** `_killOrphanedProcesses` nutzt `pkill -f $helper`. In Phase 5 laufen Connect-Daemons als langlebige librespot-Prozesse — `pkill` wuerde diese ebenfalls killen.

**Wie vermeiden (Phase 4):** In Phase 4 gibt es noch keine Connect-Daemons — kein Problem. Fuer Phase 5: Cleanup muss Connect-PID-Liste ausschliessen (CON-09). Phase 4 darf `_killOrphanedProcesses` so implementieren, dass Phase 5 dort eine Exclusion-Liste hinzufuegen kann.

**Warnsignal:** Wuerde in Phase 5 auftreten, nicht Phase 4.

---

## Code Examples

### Vollstaendiges formatOverride Pattern

```perl
# Source: Spotty ProtocolHandler.pm formatOverride + LMS CapabilitiesHelper::supportedFormats
# (verified: /usr/share/perl5/Slim/Player/CapabilitiesHelper.pm, /tmp/Spotty-Plugin/Plugin.pm)

sub formatOverride {
    my ($class, $song) = @_;

    my $client = $song->master;

    # Transcoding-Parameter fuer diesen Player injizieren
    require Plugins::SpotOn::Plugin;
    Plugins::SpotOn::Plugin->updateTranscodingTable($client);

    # Format-Capabilities dieses Players ermitteln
    my @formats = Slim::Player::CapabilitiesHelper::supportedFormats($client);

    # OGG-Direct: nur wenn Player OGG nativ unterstuetzt UND Binary passthrough kann
    if (grep { $_ eq 'ogg' } @formats) {
        if (Plugins::SpotOn::Helper->getCapability('passthrough')) {
            return 'ogg';
        }
    }

    # FLAC als Standard (D-04)
    return 'flc';
}
```

### Settings.pm Erweiterung

```perl
# Source: bestehende Settings.pm + neues normalization pref
sub prefs {
    return ($prefs, 'bitrate', 'binary', 'clientId', 'normalization');
}

# In handler(), zusaetzliche Validierung fuer normalization:
if ($paramRef->{saveSettings}) {
    my $norm = $paramRef->{'pref_normalization'} ? 1 : 0;
    $prefs->set('normalization', $norm);
    # ... bestehender Code ...
}
```

### basic.html Streaming-Abschnitt

```html
<!-- Source: bestehende basic.html Pattern + neuer Streaming-Abschnitt -->
[% WRAPPER setting title="PLUGIN_SPOTON_STREAMING_SETTINGS" desc="" %]
    <table>
    <tr>
        <td><label for="pref_bitrate">[% 'PLUGIN_SPOTON_BITRATE' | string %]</label></td>
        <td>
            <select class="stdedit" name="pref_bitrate" id="pref_bitrate">
                <option value="320" [% IF prefs.pref_bitrate == 320 %]selected[% END %]>320 kbps</option>
                <option value="160" [% IF prefs.pref_bitrate == 160 %]selected[% END %]>160 kbps</option>
                <option value="96"  [% IF prefs.pref_bitrate == 96  %]selected[% END %]>96 kbps</option>
            </select>
        </td>
    </tr>
    <tr>
        <td><label for="pref_normalization">[% 'PLUGIN_SPOTON_NORMALIZATION' | string %]</label></td>
        <td>
            <input type="checkbox" name="pref_normalization" id="pref_normalization"
                value="1" [% IF prefs.pref_normalization %]checked[% END %]/>
        </td>
    </tr>
    </table>
[% END %]
```

---

## State of the Art

| Alter Ansatz | Aktueller Ansatz | Geaendert | Bedeutung |
|--------------|-----------------|-----------|-----------|
| Statische `custom-convert.conf` mit fester Bitrate | Dynamische Regex-Injection via `updateTranscodingTable` | Spotty 2019+ | Phase 4 muss dynamisch sein |
| `formatOverride` gibt immer `'flc'` zurueck | `formatOverride` gibt OGG/FLAC je nach Player-Capabilities | Phase 4 | Ermoeglicht OGG-Direct fuer squeezelite |
| Globale `pref_bitrate` in Template fest verdrahtet | Via `updateTranscodingTable` in commandTable injiziert | Phase 4 | Kein pref-file-Substitutions-Problem |
| Kein Kontext-Queueing (nur on_select=play) | `playall => 1` auf allen Track-Items | Phase 4 | Ganzes Album/Playlist in Queue |
| Kein Process-Cleanup | Stundlicher Cleanup-Timer | Phase 4 | STR-10 Anforderung |

**Deprecated/veraltet:**
- `$CACHE$` als unveraenderter Literal in commandTable: Muss via `updateTranscodingTable` ersetzt werden bevor librespot startet.
- `getFormatForURL => 'flc'` (statisch): Wird durch dynamisches `formatOverride` ersetzt. `getFormatForURL` bleibt als Fallback erhalten.

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | B&O/UPnPBridge-Player melden keine OGG-Capability an LMS (daher automatisch FLAC-Fallback) | Format-Support-Matrix | Wenn B&O doch OGG meldet: erhalten OGG-Direct; kein Schaden aber untested |
| A2 | librespot-Binary unterstuetzt `--passthrough` (passthrough-decoder Feature) | Pitfall 3 | Wenn nicht: OGG-Pipeline liefert PCM unter OGG-Label -> Glitches. Mitigation: `getCapability('passthrough')` Check |
| A3 | Gapless mit PCM reduziert Luecke minimal, aber kein echtes Gapless | Gapless-Abschnitt | Wenn doch gapless mit PCM: Bonus-Feature, kein Problem |
| A4 | `--normalisation-method` Flag wird in Phase 4 nicht gesetzt (librespot nutzt Default `dynamic`) | Volume Normalisation D-06 | Wenn user `basic` bevorzugt: in Phase 6 loesbar |

---

## Open Questions (RESOLVED)

1. **passthrough-Capability im Binary**
   - Was wir wissen: CLAUDE.md listet `--passthrough` als Flag; `passthrough-decoder` ist ein optionales librespot Build-Feature
   - Was unklar ist: Ob das existierende SpotOn-Binary mit passthrough-decoder kompiliert wurde
   - Empfehlung: In `Helper.pm::helperCheck()` pruefen ob `--check` JSON ein `"passthrough": true` enthaelt; in `formatOverride` guard einbauen
   - **RESOLVED:** formatOverride enthaelt einen getCapability('passthrough')-Guard: OGG wird NUR zurueckgegeben wenn sowohl der Player OGG unterstuetzt ALS AUCH das Binary passthrough-capable ist. Wenn das Binary kein passthrough hat, faellt formatOverride sicher auf FLAC zurueck. Das unbekannte Risiko (A2) ist damit korrekt mitigiert — kein Live-Test noetig fuer Code-Korrektheit.

2. **librespot -n Flag und Player-Name-Konflikte**
   - Was wir wissen: `custom-convert.conf` nutzt `-n Squeezebox` als festen Namen
   - Was unklar ist: Ob der Name in Phase 4 (Single-Track, kein Connect) eine Rolle spielt; in Phase 5 wird `-n` auf Player-Name gesetzt (Connect)
   - Empfehlung: In Phase 4 `-n Squeezebox` fest lassen; Phase 5 aendert das
   - **RESOLVED:** In Phase 4 (--single-track, kein Connect) hat der `-n` Name keine funktionale Relevanz — er wird nur fuer mDNS-Announcement benoetigt, das mit `--disable-discovery` ohnehin deaktiviert ist. Fest auf `-n Squeezebox` belassen; Phase 5 aendert das fuer Connect-Daemons.

3. **Pref-Namespace `normalization` vs. LMS `replayGainMode`**
   - Was wir wissen: Spotty synct `replaygain` Pref mit LMS-Server-Pref `replayGainMode`
   - Was unklar ist: Ob wir das fuer Phase 4 ebenfalls wollen (oder einfaches eigenes Pref)
   - Empfehlung: Eigenes einfaches Pref `normalization` (0/1) in Phase 4; LMS-Sync-Integration in Phase 6
   - **RESOLVED:** Phase 4 verwendet eigenes einfaches Pref `normalization` (0/1, global). LMS-Server-Pref-Sync (`replayGainMode`) ist Phase 6 Scope (LMS-08 per-player settings).

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Perl | Plugin Sprache | ✓ | 5.38.2 | — |
| LMS | Plugin Framework | ✓ | `/usr/sbin/squeezeboxserver` vorhanden | — |
| `flac` Binary | `son flc * *` Pipeline | Nicht gecheckt | — | Pipeline deaktiviert wenn nicht gefunden (LMS `checkBin`) |
| `lame` Binary | `son mp3 * *` Pipeline | Nicht gecheckt | — | Pipeline deaktiviert wenn nicht gefunden (LMS `checkBin`) |
| `spoton` Binary | Alle Pipelines | Vorhanden (laut Phase 1 Implementierung) | 1.0.0+ | Helper.pm zeigt BINARY_MISSING |

**Missing dependencies mit Fallback:**
- `flac` und `lame`: LMS `TranscodingHelper::checkBin` prueft Binaries beim Laden; fehlende Binaries deaktivieren die entsprechenden Profile automatisch. Kein Plugin-Code noetig.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Keine automatisierten Tests (LMS-Plugin-Kontext erlaubt kein Unit-Testing ohne laufendes LMS) |
| Config file | N/A |
| Quick run command | Manuell: LMS-Restart + Track antippen + hoeren |
| Full suite command | Success Criteria aus CONTEXT.md manuell pruefen |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| STR-01 | Track spielt innerhalb 5s | manual | — | N/A |
| STR-02 | FLAC-Pipeline (Default) | manual | — | N/A |
| STR-03 | PCM-Pipeline fuer faehige Player | manual | — | N/A |
| STR-04 | MP3-Pipeline vorhanden | manual | — | N/A |
| STR-05 | OGG-Direct fuer squeezelite | manual | — | N/A |
| STR-06 | Bitrate-Setting wirkt | manual (LMS log pruefen) | — | N/A |
| STR-07 | Seeking an Mitte eines Tracks | manual | — | N/A |
| STR-08 | Volume Normalisation Toggle | manual (lauter/leiser) | — | N/A |
| STR-09 | Gapless (nice-to-have) | manual | — | N/A |
| STR-10 | Cleanup nach 2h Normalbetrieb | manual (ps aux nach 2h) | — | N/A |
| STR-11 | Audio-Cache deaktiviert | manual (kein Cache-Wachstum) | — | N/A |
| LMS-11 | Zwei Player gleichzeitig | manual (zwei squeezelite) | — | N/A |

### Wave 0 Gaps

Kein Test-Framework fuer LMS-Plugins verfuegbar — manuelles Testen gegen Live-LMS-Instanz ist der Standard-Ansatz in diesem Projekt.

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | nein | — |
| V3 Session Management | nein | — |
| V4 Access Control | nein | — |
| V5 Input Validation | ja (begrenzt) | Bitrate-Wert auf {96,160,320} beschraenken (bereits in Settings.pm) |
| V6 Cryptography | nein | — |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Command Injection via `$helper`-Pfad in pkill | Tampering | Shell-escaping (Single-Quotes), kein unkontrollierter Input (Pfad kommt aus Helper.pm::_findBin) |
| Command Injection via Bitrate-Wert in commandTable | Tampering | Whitelist-Validation bereits in Settings.pm (nur 96/160/320 erlaubt) |
| Path Traversal via cacheDir | Elevation of Privilege | Cache-Dir wird via `catdir($serverPrefs->get('cachedir'), 'spoton')` konstruiert; kein User-Input |

---

## Sources

### Primary (HIGH confidence)
- `/usr/share/perl5/Slim/Player/TranscodingHelper.pm` — commandTable, getConvertCommand2, tokenizeConvertCommand2, pref-file caching (Zeilen 641-655)
- `/usr/share/perl5/Slim/Player/Song.pm` — formatOverride-Aufruf, canSeek/canDoSeek, getSeekData (Zeilen 377-430, 809-875)
- `/usr/share/perl5/Slim/Control/XMLBrowser.pm` — playall-Flag, loadtracks mit playIndex (Zeilen 660-768)
- `/usr/share/perl5/Slim/Player/CapabilitiesHelper.pm` — supportedFormats (Zeile 38-59)
- `/usr/share/perl5/Slim/Player/SqueezePlay.pm` — myFormats, squeezelite OGG-Support (Zeile 59)
- `/usr/share/perl5/Slim/Player/Squeezebox2.pm` — formats() `qw(wma ogg flc aif pcm mp3)` (Zeile 134)
- `/usr/share/perl5/Slim/Player/Squeezebox1.pm` — Gapless-Bedingungen (Zeilen 130-158)
- `/tmp/Spotty-Plugin/Plugin.pm` — updateTranscodingTable, killHangingProcesses (Zeilen 234-282)
- `/tmp/Spotty-Plugin/custom-convert.conf` — Referenz-Transcoding-Konfiguration
- `CLAUDE.md` — librespot CLI Flags, custom-convert.conf Syntax, LMS Module

### Secondary (MEDIUM confidence)
- Spotty OPML.pm via WebFetch — playall=1 Pattern bestaetigt

### Tertiary (LOW confidence)
- B&O/UPnPBridge OGG-Support: ASSUMED basierend auf DLNA-Standards und allgemeinem Wissen

---

## Metadata

**Confidence breakdown:**
- Standard Stack: HIGH — verifiziert aus LMS-Sourcecode auf Produktiv-System
- Architecture: HIGH — TranscodingHelper, Song.pm, XMLBrowser.pm direkt analysiert
- Pitfalls: HIGH — aus echtem Quellcode abgeleitet, nicht nur Training
- B&O Format-Support: LOW — nicht direkt verifizierbar ohne laufendes B&O-Geraet

**Research date:** 2026-05-28
**Valid until:** 2026-08-28 (LMS API stabil; librespot 0.8.0 ist aktuell)
