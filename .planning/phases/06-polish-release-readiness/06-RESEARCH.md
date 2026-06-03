# Phase 6: Polish + Release Readiness — Research

**Researched:** 2026-06-03
**Domain:** LMS Plugin Polish, DSTM, Per-Player Prefs, i18n, Binary Distribution, repo.xml
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Bitrate per Player = Global + Override. Globales Bitrate-Setting (320 Default) bleibt. Jeder Player kann optional einen eigenen Wert setzen (96/160/320). Ohne Override gilt der globale Wert. Pattern analog zu bestehendem OGG-Override.
- **D-02:** Volume Normalisierung bleibt global. Kein per-Player Override. Normalisierung muss auch für Connect-Modus gelten (Daemon.pm übergibt bereits `--enable-volume-normalisation`).
- **D-03:** Per-Player Settings-Seite = eine Sektion pro Player. Alle per-Player Prefs in einer Sektion untereinander. Player-Dropdown oben wählt den Player aus.
- **D-04:** Client-ID Konsolidierung — `SPOTON_DEFAULT_CLIENT_ID` wird in `Client.pm` definiert (einzige Stelle). `TokenManager.pm` importiert die Konstante von dort. SC-7 erfüllt.
- **D-05:** DSTM Primär: `recommendations`-Endpoint via bundled-Token (Herger-ID mit Extended Quota). Gleiche Seed-Logik wie Spotty-NG: bis zu 5 Seeds aus aktueller Playlist, Spotify-Tracks direkt als seed_tracks, Nicht-Spotify-Tracks via Search-Match. Fallback bei 404/403: Search-basiert.
- **D-06:** DSTM-Integration über LMS-Standard-Framework. SpotOn registriert sich als DSTM-Provider via `Slim::Plugin::DontStopTheMusic::Plugin->registerHandler()`. Aktivierung durch User in LMS Player-Settings → Don't Stop The Music → "SpotOn Empfehlungen".
- **D-07:** Setup Guide Platzierung — Claude entscheidet basierend auf LMS-Konventionen und Spotty-NG-Referenz.
- **D-08:** Credits-Text als dezenter Footer am Ende der Settings-Seite: "SpotOn nutzt librespot. Inspiriert von Hergers Spotty Plugin."
- **D-09:** Repository-Distribution via GitHub repo.xml. LMS-Nutzer fügen die URL in Settings → Plugins → Additional Repositories ein.
- **D-10:** i18n-Übersetzungen von Claude generiert für die volle LMS-Sprachpalette (EN, DE, FR, NL, IT, ES, SV, NO, DA, PL, CS). Community kann später korrigieren.
- **D-11:** Per-Player Format-Dropdown — einheitlich für Connect UND Browse/Single-Track. Erweitert den bestehenden OGG-Override-Dropdown: Auto / OGG (DirectStream) / PCM (DirectStream) / FLAC (transkodiert) / MP3 (transkodiert). Bei FLAC/MP3: `canDirectStream()` gibt 0 zurück.
- **D-12:** Deferred Item aus Phase 5: Per-Player OGG-Passthrough gilt jetzt für BEIDE Modi (Connect und Browse), nicht nur Connect.

### Claude's Discretion

- Setup Guide Platzierung und Detailtiefe (D-07): Settings-Seite oben vs. inline vs. eigene Section
- Binary-Build-Pipeline: Wie die Multi-Architektur-Binaries (x86_64, aarch64, armhf, i386) bereitgestellt werden
- repo.xml Struktur und Versionierung
- Security Review und Code Review Scope (SC-11): Welche Module und in welcher Tiefe
- DSTM Search-Fallback Details: Welche Search-Parameter, wie viele Results, Randomisierung

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| LMS-03 | i18n support (EN + DE minimum) via LMS strings mechanism | 65 Schlüssel vorhanden, nur EN+DE. Erweiterung auf 11 Sprachen dokumentiert. |
| LMS-06 | Multi-architecture binaries (x86_64, aarch64, armhf, i386) | Nur x86_64 vorhanden. Fehlende Binaries: aarch64, armhf, i386. Cross.toml + musl-Targets bekannt. |
| LMS-08 | Player-specific preferences (bitrate, normalization, Connect on/off) | Pattern mit `$prefs->client($client)->get/set()` bereits etabliert. Erweiterung für Bitrate-Override und Format-Dropdown dokumentiert. |
| LMS-09 | Don't Stop The Music integration for auto-play after playlist end | Spotty-NG DontStopTheMusic.pm als vollständige Referenz vorhanden. DSTM-API und Routing via bundled-Token dokumentiert. |
| LMS-10 | Custom binary support (user-provided binary override) | `Helper.pm` hat bereits `spoton-custom` als ersten Kandidaten in `_findBin()`. LMS-10 ist strukturell vorbereitet. Vollständige Implementierung noch ausstehend. |

</phase_requirements>

---

## Summary

Phase 6 ist eine Querschnitts-Phase: Sie vollendet mehrere bewusst zurückgestellte Features (per-Player Bitrate, DSTM, Normalisierung im Connect-Modus, Custom-Binary-Support), schließt Qualitätslücken (i18n, Security Review, Client-ID-Duplikat), und liefert das Plugin als verteilbare Einheit aus (repo.xml, Multi-Architektur-Binaries).

Der Codestand nach Phase 5.4 ist technisch funktionsfähig für die geplanten Erweiterungen. Die Per-Player-Prefs-Infrastruktur (`$prefs->client($client)->get/set`) ist für Connect, OGG-Override und Discovery-Toggle bereits etabliert und kann direkt für Bitrate-Override und das neue Format-Dropdown erweitert werden. `updateTranscodingTable()` ist der einzige Ort, der Bitrate in die Commandline injiziert — eine Erweiterung für per-Player-Bitrate ist ein chirurgischer Eingriff.

DSTM ist mit der Spotty-NG `DontStopTheMusic.pm` als vollständiger 165-Zeilen-Referenz gut dokumentiert. Die Routing-Infrastruktur (bundled-Token für `recommendations`) ist in `Client.pm` bereits eingebaut — `recommendations` steht in `@KNOWN_DEPRECATED_FAMILIES` und wird automatisch über den bundled-Token geleitet. Die einzige fehlende Komponente ist die `recommendations()`-Methode in `Client.pm` selbst sowie die neue Datei `DontStopTheMusic.pm`.

**Primary recommendation:** Zuerst die mechanischen Aufgaben (Client-ID-Konsolidierung, DSTM, per-Player-Bitrate) abarbeiten, dann die redaktionellen (i18n, Setup Guide, Credits), dann Distribution (repo.xml, Binaries), abschließend Security Review.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Per-Player Bitrate Override | Plugin (Settings.pm + Plugin.pm) | custom-convert.conf | Bitrate wird pro Track in updateTranscodingTable() injiziert; Settings.pm schreibt den Pref |
| Format-Dropdown (Unified) | Plugin (Settings.pm + ProtocolHandler.pm) | Plugin.pm::updateTranscodingTable | canDirectStream() und formatOverride() lesen den per-Player Pref; Settings.pm schreibt ihn |
| Volume Normalisation im Connect-Modus | Connect/Daemon.pm | Plugin.pm::updateTranscodingTable | Daemon.pm übergibt `--enable-volume-normalisation` an librespot-Prozess; Single-Track-Pipeline nutzt updateTranscodingTable |
| DSTM (Don't Stop The Music) | DontStopTheMusic.pm (neues Modul) | API/Client.pm::recommendations | DSTM-Handler wird bei initPlugin() registriert; recommendations() läuft über Client.pm |
| Client-ID Konsolidierung | API/Client.pm (Konstante) | API/TokenManager.pm (Import) | Eine Quelle der Wahrheit; TokenManager importiert statt duplikat zu pflegen |
| Custom Binary Support | Helper.pm | Settings.pm | Helper.pm::_findBin() prüft bereits `spoton-custom` zuerst; Settings.pm bietet Binary-Pfad-Eingabe |
| i18n | strings.txt | basic.html + *.pm via cstring() | LMS strings mechanism: strings.txt als einzige Quelle, cstring() für dynamische Strings |
| Setup Guide | basic.html | strings.txt | HTML-Template, Platztierung als erste Sektion oberhalb der Player-Settings |
| repo.xml Distribution | GitHub (raw URL) | install.xml | Zip-Archiv + SHA1 + repo.xml hostet auf GitHub; LMS fetcht die XML-Datei direkt |
| Multi-Architektur-Binaries | Bin/{arch}-linux/ | Cross.toml + cross-rs | Binaries in architekturspezifischen Verzeichnissen; Helper.pm::_findBin() wählt korrekte Arch |

---

## Standard Stack

### Core (alles bereits vorhanden — kein neues Dependency)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Slim::Plugin::DontStopTheMusic::Plugin | LMS 8+ | DSTM-Handler-Registration | LMS-Standard-API; `registerHandler()` ist der einzige Weg ins DSTM-Framework |
| Slim::Utils::PluginManager | LMS 8+ | DSTM-Availability-Check | `isEnabled()` vor Registration aufrufen (Spotty-NG Pattern) |
| Slim::Schema | LMS 8+ | Track-Object-Lookup für DSTM-Seed | Nötig wenn Track-ID ein RemoteTrack-Objekt ist (Spotty-NG Pattern, DontStopTheMusic.pm:58) |
| Slim::Plugin::DontStopTheMusic::Plugin::getMixableProperties | LMS 8+ | Seed-Track-Extraktion | Liefert bis zu N Track-Objekte aus aktueller Playlist |

### Keine neuen externen Pakete erforderlich

Alle benötigten Module sind LMS-Bundled oder bereits in SpotOn vorhanden. Die Phase installiert keine externen CPAN-Abhängigkeiten. [VERIFIED: CLAUDE.md §Constraints — "No external CPAN deps"]

---

## Package Legitimacy Audit

Keine neuen externen Pakete in dieser Phase. Alle verwendeten Module sind entweder LMS-gebundelt oder bereits im Projekt vorhanden. Dieser Abschnitt entfällt.

**Packages removed due to slopcheck [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

---

## Architecture Patterns

### System Architecture Diagram

```
DSTM-Flow:
  LMS Playlist End
       |
       v
  DontStopTheMusic::dontStopTheMusic($client, $cb)
       |
       v
  getMixableProperties($client, 5) --> max. 5 Seed-Tracks aus Playlist
       |
       +--[ Spotify-URIs vorhanden ]---> seed_tracks = [trackId, ...]
       |
       +--[ Nicht-Spotify-Tracks ]-----> Search: 'track "Title" artist:"Artist"'
       |                                    |
       |                          Kein Match: Search: artist:"Artist"
       |                                    |
       |                          seed_artists = [artistId, ...]
       |
       v
  Client.pm::recommendations(cb, {seed_tracks/artists, limit:25})
       |      [Routing: bundled-Token via @KNOWN_DEPRECATED_FAMILIES]
       v
  GET /recommendations?seed_tracks=...&limit=25
       |
  404/403: Search-Fallback ('artist:"SeedArtist"', randomisierter Offset)
       |
       v
  $cb->($client, ['spotify://track:XXX', ...])

Per-Player Format/Bitrate-Flow:
  ProtocolHandler::formatOverride($song)
       |
       v
  Plugin::updateTranscodingTable($client)
       |-- liest: $prefs->client($client)->get('bitrateOverride') || global bitrate
       |-- liest: $prefs->client($client)->get('streamFormat')   -- neu: auto/ogg/pcm/flac/mp3
       |
       v
  ProtocolHandler::canDirectStream($client, $url)
       |-- wenn streamFormat in ('flac','mp3'): return 0 --> LMS proxied via custom-convert.conf
       |-- sonst: bestehende Logik (Connect HTTP-URL oder 0)
       |
       v
  ProtocolHandler::formatOverride() return-Wert:
       - 'son' (pcm/flac/mp3 Browse)
       - 'soc' (Connect)
       - 'ogg' (OGG-Passthrough, wenn streamFormat='ogg')
```

### Recommended Project Structure

```
Plugins/SpotOn/
├── DontStopTheMusic.pm          # NEU: DSTM-Provider nach Spotty-NG Pattern
├── Plugin.pm                    # ERWEITERT: DSTM-Registration in initPlugin(), Prefs-Init
├── Settings.pm                  # ERWEITERT: bitrateOverride + streamFormat per-Player
├── ProtocolHandler.pm           # ERWEITERT: canDirectStream() + formatOverride() per-Player
├── Helper.pm                    # ERWEITERT: Custom-Binary-Support vollständig
├── strings.txt                  # ERWEITERT: 65 Keys × 11 Sprachen statt nur EN+DE
├── install.xml                  # AKTUALISIERT: Version, repo-URL
├── custom-convert.conf          # GEÄNDERT: $BITRATE$ Platzhalter statt hardcoded 320
├── API/
│   ├── Client.pm                # ERWEITERT: recommendations(), Client-ID-Konstante exportierbar
│   └── TokenManager.pm          # GEÄNDERT: SPOTON_DEFAULT_CLIENT_ID entfernen, aus Client.pm importieren
└── HTML/EN/plugins/SpotOn/settings/
    └── basic.html               # ERWEITERT: Setup Guide, Format-Dropdown, Bitrate-Override, Credits
```

### Pattern 1: Per-Player Pref lesen mit globalem Fallback

```perl
# Source: Settings.pm bestehend (D-01 Pattern analog OGG-Override)
# Neu: bitrateOverride — globaler Fallback wenn kein per-Player-Wert
sub _bitrateForClient {
    my ($class, $client) = @_;
    return $prefs->get('bitrate') || 320 unless $client;
    my $override = $prefs->client($client)->get('bitrateOverride');
    return ($override && $override =~ /^(?:96|160|320)$/) ? $override : ($prefs->get('bitrate') || 320);
}
```

**Wichtig:** Der Pref-Key muss sich vom globalen Key unterscheiden. Global: `bitrate`. Per-Player: `bitrateOverride`. Sonst überschreibt `$prefs->client($client)->get('bitrate')` den globalen Pref — das LMS-Prefs-System speichert per-Player-Werte unter demselben Namespace wie globale.

### Pattern 2: Format-Dropdown Unified (D-11)

Fünf Optionen ersetzen den bisherigen `connectOggOverride`-Dropdown:

| Pref-Wert | canDirectStream | formatOverride | Transcoding-Pipeline |
|-----------|----------------|----------------|---------------------|
| `auto` | wie bisher (Connect: HTTP-URL, Browse: 0) | wie bisher (soc/son) | Default |
| `ogg` | wie bisher | `ogg` (Connect+Browse) | son-ogg-*-* / soc-ogg-*-* |
| `pcm` | 0 (forciert) | `son` / `soc` | son-pcm / soc-pcm |
| `flac` | 0 (forciert) | `son` / `soc` | son-flc-*-* |
| `mp3` | 0 (forciert) | `son` / `soc` | son-mp3-*-* |

Der bisherige Pref-Key `connectOggOverride` wird auf den neuen Key `streamFormat` migriert. Bestehende Werte `auto`/`ogg`/`pcm` bleiben kompatibel. Neu hinzukommen: `flac`, `mp3`.

**Pref-Migration in Settings.pm:** Beim ersten Lesen des neuen Keys: wenn `streamFormat` leer/undefined und `connectOggOverride` vorhanden, den alten Wert als Startwert übernehmen.

### Pattern 3: DSTM-Registration in initPlugin()

```perl
# Source: Spotty-NG Plugin.pm Zeile 164-168
if ( Slim::Utils::PluginManager->isEnabled('Slim::Plugin::DontStopTheMusic::Plugin') ) {
    require Plugins::SpotOn::DontStopTheMusic;
    Plugins::SpotOn::DontStopTheMusic->init();
}
```

Der `Slim::Utils::Versions->compareVersions`-Check aus Spotty-NG ist für SpotOn nicht nötig (Minimum LMS 8.0 unterstützt DSTM bereits). [ASSUMED — LMS 8.0 DSTM-Support nicht explizit in LMS-Changelog verifiziert, aber Spotty-NG check war für LMS 7.9+ und SpotOn hat 8.0 als Floor]

### Pattern 4: DSTM recommendations() in Client.pm

Die `recommendations()`-Methode folgt dem gleichen `_request()`-Aufruf-Pattern wie alle anderen Methoden. Die `recommendations`-URL ist bereits in `@KNOWN_DEPRECATED_FAMILIES` gelistet — das Routing über den bundled-Token geschieht automatisch ohne zusätzlichen Code.

```perl
# Analog Spotty-NG API.pm::recommendations, vereinfacht für SpotOn-API-Architektur
sub recommendations {
    my ($class, $accountId, $params, $cb) = @_;

    unless ($params && ($params->{seed_tracks} || $params->{seed_artists})) {
        $cb->([]);
        return;
    }

    my %reqParams = (
        _accountId => $accountId,
        _noCache   => 1,                    # Empfehlungen nie cachen
        limit      => $params->{limit} // 25,
    );

    # seed_tracks/seed_artists: Array-Ref -> komma-getrennter String
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

**Kritisch:** Die Methode erhält `$accountId` als ersten Parameter — das ist das SpotOn-API-Pattern. Spotty-NG's `recommendations()` bekommt `$cb` zuerst (andere Calling Convention). Nicht 1:1 kopieren.

### Pattern 5: DSTM Search-Fallback

Der Fallback bei 404/403 verwendet eine einfache Artist-Search mit Zufalls-Offset:

```perl
# In DontStopTheMusic.pm::_searchFallback($client, $seedArtist, $cb)
my $offset = int(rand(40));  # Randomisierung verhindert immer denselben ersten Track
Plugins::SpotOn::API::Client->search($accountId, {
    q      => sprintf('artist:"%s"', $seedArtist),
    type   => 'track',
    limit  => 10,
    offset => $offset,
}, sub {
    my $result = shift;
    my $tracks = ($result && $result->{tracks} && $result->{tracks}{items}) ? $result->{tracks}{items} : [];
    $cb->($client, [ map { 'spotify://' . ($_->{uri} =~ /(track:[a-z0-9]+)/i)[0] } @$tracks ]);
});
```

### Pattern 6: custom-convert.conf Bitrate-Platzhalter

Aktuell ist `--bitrate 320` hardcoded in `custom-convert.conf`. `updateTranscodingTable()` überschreibt das per Regex. Das Muster funktioniert — aber der Platzhalter sollte einen erkennbaren Wert haben:

```
son pcm * *
	# RT:{START=--start-position %s}
	[spoton] -n Squeezebox -c "$CACHE$" --single-track $URL$ --bitrate 320 --disable-discovery --disable-audio-cache $START$
```

Der Wert `320` im File dient als Regex-Target für `updateTranscodingTable()` (`s/--bitrate \d+/--bitrate $bitrate/`). Das Pattern erfordert KEINE Änderung der conf-Datei selbst — die per-Player-Bitrate wird ausschließlich via `updateTranscodingTable()` injiziert.

### Pattern 7: repo.xml für GitHub-Hosting

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
      <desc lang="DE">Spotify-Plugin für Lyrion Music Server</desc>
    </plugin>
  </plugins>
</extensions>
```

Die Datei liegt als `repo.xml` im GitHub-Repo. GitHub Raw URL: `https://raw.githubusercontent.com/USER/spoton/main/repo.xml`. Diese URL fügt der LMS-Nutzer in Settings → Plugins → Additional Repositories ein. [VERIFIED: lyrion.org/reference/repository-dev/]

**SHA1-Berechnung:** `sha1sum SpotOn-1.0.0.zip` — muss jedes Release neu berechnet werden.

**Version im Zip-Namen:** LMS cached Zip-Dateien; der Versionsstring im Dateinamen ist zwingend, sonst werden Upgrades nicht erkannt. [VERIFIED: lyrion.org/reference/repository-dev/]

### Pattern 8: Client-ID-Konsolidierung (D-04)

`TokenManager.pm` Zeile 23 definiert derzeit eine zweite Kopie der Konstante:
```perl
use constant SPOTON_DEFAULT_CLIENT_ID => '93aac68fb06348598c1e67734dfaceee';
```

Diese Zeile wird gelöscht. Stattdessen Import aus `Client.pm`:
```perl
use Plugins::SpotOn::API::Client qw(SPOTON_DEFAULT_CLIENT_ID);
```

In `Client.pm` muss die Konstante exportierbar gemacht werden:
```perl
use Exporter 'import';
our @EXPORT_OK = qw(SPOTON_DEFAULT_CLIENT_ID);
```

**Vorsicht:** Perl-Konstanten via `use constant` sind per default nicht exportierbar. `Exporter` muss explizit eingebunden werden.

### Pattern 9: Setup Guide Platzierung (D-07 — Claude's Discretion)

Empfehlung: Eigene Sektion am **Anfang** der Settings-Seite, vor den Player-Settings, nur sichtbar wenn KEIN Account konfiguriert ist (analog zum bestehenden ZeroConf-Block) ODER permanent als ausklappbare Sektion.

Da Spotty-NG keine dedizierte Setup Guide Sektion hat (Setup läuft über ZeroConf-Discovery), orientiert sich SpotOn am LMS-Konventions-Pattern für neue Nutzer: Prominenter Block oben, der nach erstem Account-Setup verschwindet oder kollabiert.

```html
[% IF NOT accounts.keys.size %]
<div style="background:#f8f8f8; border:1px solid #ddd; padding:16px; margin-bottom:20px; border-radius:4px">
  <h3>[% 'PLUGIN_SPOTON_SETUP_GUIDE_TITLE' | string %]</h3>
  <!-- Schritt 1: Spotify Developer App -->
  <!-- Schritt 2: Client-ID eintragen -->
  <!-- Schritt 3: Spotify App verbinden -->
</div>
[% END %]
```

Wenn Account vorhanden: Credits-Footer erscheint am Ende jeder Settings-Seite.

### Anti-Patterns to Avoid

- **`$prefs->client($client)->get('bitrate')` als Override-Key verwenden:** Kollidiert mit dem globalen `bitrate`-Pref-Schlüssel. Separaten Key `bitrateOverride` verwenden.
- **DSTM-Handler vor `Slim::Plugin::DontStopTheMusic::Plugin` registrieren:** Crash wenn Plugin nicht installiert. Immer `isEnabled()` prüfen.
- **`connectOggOverride` Pref beibehalten parallel zu `streamFormat`:** Führt zu Widersprüchen. Migration bei erster Lese-Operation durchführen, alten Key dann löschen oder ignorieren.
- **`recommendations()` in `DontStopTheMusic.pm` direkt aufrufen ohne `$accountId`:** SpotOn's `Client.pm` erfordert `$accountId`. Spotty-NG-Code kann nicht 1:1 übernommen werden — Calling Convention unterscheidet sich.
- **repo.xml ohne Versionsnummer im Zip-URL:** LMS nutzt Cache und erkennt Upgrades nur wenn sich der URL ändert.
- **i18n-Texte mit Sonderzeichen nicht UTF-8-kodieren:** strings.txt muss UTF-8 sein. LMS erwartet das.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| DSTM-Framework | Eigener Playlist-Monitoring-Timer | `Slim::Plugin::DontStopTheMusic::Plugin::registerHandler()` | LMS-Framework kümmert sich um Playlist-Ende-Erkennung, Player-State, Timing |
| DSTM-Seed-Extraktion | Direktes Playlist-Parsing | `getMixableProperties($client, 5)` | Framework liefert normalisierten Track-Array mit artist/title/id |
| i18n-Loading | Eigenes Strings-System | LMS strings.txt Mechanismus + `cstring()` | LMS lädt strings.txt automatisch; `cstring($client, 'KEY')` liefert player-locale-String |
| Binary-Auswahl | Eigene Arch-Detection | `Slim::Utils::Misc::findbin()` + `Plugins::SpotOn::Helper::_findBin()` | LMS findet Binaries in Plugin Bin/-Verzeichnissen automatisch per arch-Prefix |
| Checksummen-Verifikation | Eigene SHA1-Prüfung | LMS-Repository-Manager | LMS verifiziert SHA1 beim Download automatisch |

---

## Runtime State Inventory

Diese Phase ist kein Rename/Refactor. Kein Runtime State betroffen.

**Gespeicherte Daten:** Pref-Key-Migration `connectOggOverride` → `streamFormat`: Bestehende Werte (`auto`/`ogg`/`pcm`) sind kompatibel — kein Datenmigrations-Task nötig, da die neuen Werte eine Obermenge der alten sind.

**Live-Service-Config:** Keine.
**OS-registrierter State:** Keine.
**Secrets/Env-Vars:** Keine.
**Build-Artefakte:** Fehlende Arch-Binaries (aarch64, armhf, i386) müssen gebaut werden.

---

## Common Pitfalls

### Pitfall 1: DSTM Calling Convention Mismatch (Spotty-NG vs SpotOn)

**Was schiefgeht:** Spotty-NG `DontStopTheMusic.pm` verwendet `$spotty->recommendations($cb, $params)` — die API-Instanz ist `$spotty`, nicht die Klasse. SpotOn's `Client.pm` ist eine Klasse mit `$accountId` als erstem Parameter.
**Warum:** Unterschiedliche API-Architekturen: Spotty-NG instanziiert eine `API`-Instanz pro Account; SpotOn verwendet eine statische Klasse mit explizitem `$accountId`.
**Vermeidung:** In `DontStopTheMusic.pm` muss `Plugins::SpotOn::API::Client->recommendations($accountId, $params, $cb)` statt `$spotty->recommendations($cb, $params)` aufgerufen werden. `$accountId` aus `$prefs->get('activeAccount')` holen.
**Warnsignal:** "Can't call method 'recommendations' on unblessed reference" oder falsche Token-Verwendung.

### Pitfall 2: Per-Player Pref Key Kollision mit globalem Pref

**Was schiefgeht:** `$prefs->client($client)->get('bitrate')` überschreibt nicht den globalen `bitrate`-Pref, sondern liest/schreibt den player-spezifischen. Aber wenn `$prefs->init({bitrate => 320})` aufgerufen wird, setzen alle Player den Wert 320 — weil der init-Default für Client-Prefs separat initialisiert werden muss.
**Warum:** LMS Prefs-System speichert globale und per-Player-Werte im gleichen Namespace. `$prefs->init()` initialisiert nur den globalen Wert.
**Vermeidung:** Separaten Key `bitrateOverride` mit leerem Default verwenden. Fallback-Logik: wenn `bitrateOverride` leer/undefined, globalen `bitrate`-Pref lesen.
**Warnsignal:** Alle Player zeigen plötzlich denselben Bitrate-Override.

### Pitfall 3: canDirectStream() Rückgabe mit Format-Dropdown

**Was schiefgeht:** Bei `streamFormat = 'flac'` oder `'mp3'` muss `canDirectStream()` explizit `0` zurückgeben — sonst versucht LMS DirectStream mit einer FLAC-URL, was fehlschlägt.
**Warum:** LMS fragt `canDirectStream()` vor dem Transcoding-Pipeline-Auswahl ab. Gibt die Methode eine HTTP-URL zurück, umgeht LMS das Transcoding komplett.
**Vermeidung:** In `canDirectStream()` per-Player `streamFormat` prüfen; bei `flac`/`mp3` sofort `0` zurückgeben.
**Warnsignal:** Stille oder Fehler bei FLAC/MP3-Transcode-Playback obwohl Format-Dropdown gesetzt ist.

### Pitfall 4: Exporter für Perl-Konstanten

**Was schiefgeht:** `use constant SPOTON_DEFAULT_CLIENT_ID => '...'` und dann `use Plugins::SpotOn::API::Client qw(SPOTON_DEFAULT_CLIENT_ID)` funktioniert nicht ohne explizites `Exporter`-Setup in `Client.pm`.
**Warum:** Perl-Konstanten via `use constant` sind Sub-Referenzen, die nur exportiert werden wenn `@EXPORT_OK` sie auflistet und `Exporter` eingebunden ist.
**Vermeidung:** In `Client.pm` hinzufügen: `use Exporter 'import'; our @EXPORT_OK = qw(SPOTON_DEFAULT_CLIENT_ID);`
**Warnsignal:** "Undefined subroutine &Plugins::SpotOn::API::Client::SPOTON_DEFAULT_CLIENT_ID"

### Pitfall 5: repo.xml SHA1 muss nach jedem Release neu berechnet werden

**Was schiefgeht:** SHA1 in repo.xml veraltet → LMS verweigert Installation mit Checksum-Fehler.
**Warum:** LMS verifiziert SHA1 beim Plugin-Download gegen den Wert in repo.xml.
**Vermeidung:** Release-Prozess dokumentiert: `sha1sum SpotOn-X.Y.Z.zip` → SHA1 in repo.xml eintragen, committen, Release-Tag setzen.
**Warnsignal:** "Checksum mismatch" beim Plugin-Install in LMS.

### Pitfall 6: i18n-Strings mit Sonderzeichen

**Was schiefgeht:** strings.txt in Latin-1 oder mit falschem Encoding → Umlaute etc. werden falsch angezeigt oder LMS schmeisst Encoding-Fehler.
**Warum:** LMS erwartet UTF-8.
**Vermeidung:** Editor auf UTF-8 konfigurieren; nach dem Schreiben mit `file strings.txt` prüfen ("UTF-8 Unicode text").
**Warnsignal:** "?" oder "â€" in der Settings-Seite statt Umlauten/Akzenten.

### Pitfall 7: DSTM-Handler-Schlüssel muss in strings.txt stehen

**Was schiefgeht:** `registerHandler('PLUGIN_SPOTON_RECOMMENDATIONS', ...)` — wenn der Schlüssel `PLUGIN_SPOTON_RECOMMENDATIONS` nicht in strings.txt steht, zeigt LMS den Rohen Schlüsselstring in der DSTM-Auswahl-UI.
**Warum:** LMS resolved den Handler-Namen als String-Key für die UI.
**Vermeidung:** `PLUGIN_SPOTON_RECOMMENDATIONS` in strings.txt für alle 11 Sprachen definieren.

---

## Code Examples

### DSTM-Modul (adaptiert von Spotty-NG für SpotOn-Calling-Convention)

```perl
# Source: /home/sti/spotty-ng/Spotty-Plugin/DontStopTheMusic.pm (adaptiert)
# Wesentliche Änderungen: $spotty->... → Client->...($accountId, ...)

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

sub init {
    Slim::Plugin::DontStopTheMusic::Plugin->registerHandler(
        'PLUGIN_SPOTON_RECOMMENDATIONS',
        \&dontStopTheMusic
    );
}

sub dontStopTheMusic {
    my ($client, $cb) = @_;

    my $seedTracks = Slim::Plugin::DontStopTheMusic::Plugin->getMixableProperties($client, 5);

    unless ($seedTracks && ref $seedTracks && @$seedTracks) {
        $cb->($client);
        return;
    }

    my $accountId = $prefs->get('activeAccount') || '';
    unless ($accountId) {
        $cb->($client);
        return;
    }

    require Plugins::SpotOn::API::Client;
    # ... Seed-Logik analog Spotty-NG ...
}
```

### Format-Dropdown in Settings.pm (handler)

```perl
# Per-player streamFormat (D-11) — ersetzt connectOggOverride (D-05, Phase 5)
if ($client && defined $paramRef->{'pref_streamFormat'}) {
    my $fmt = $paramRef->{'pref_streamFormat'};
    $fmt = 'auto' unless $fmt =~ /^(?:auto|ogg|pcm|flac|mp3)$/;
    $prefs->client($client)->set('streamFormat', $fmt);
}
```

### updateTranscodingTable() Erweiterung für per-Player Bitrate

```perl
# Bestehend: globale Bitrate
my $bitrate = $prefs->get('bitrate') || 320;

# NEU: per-Player Override, wenn gesetzt
if ($client) {
    my $override = $prefs->client($client)->get('bitrateOverride');
    $bitrate = $override if $override && $override =~ /^(?:96|160|320)$/;
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Spotty `connectOggOverride` (3 Werte) | SpotOn `streamFormat` (5 Werte) | Phase 6 | Unified Format-Dropdown für Connect und Browse |
| SPOTON_DEFAULT_CLIENT_ID in beiden Client.pm + TokenManager.pm | Nur in Client.pm, Export via Exporter | Phase 6 | SC-7 erfüllt, Single Source of Truth |
| Binaries nur für x86_64 | Binaries für x86_64, aarch64, armhf, i386 | Phase 6 | LMS-06 erfüllt |
| strings.txt nur EN+DE | strings.txt mit 11 Sprachen | Phase 6 | LMS-03 vollständig erfüllt |

**Deprecated/outdated:**
- `connectOggOverride` Pref-Key: Wird durch `streamFormat` ersetzt. Bestehende Werte sind kompatibel (Obermenge). Altkey kann nach Migration ignoriert werden (nicht löschen — rückwärtskompatibel lassen).

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | LMS 8.0 unterstützt `Slim::Plugin::DontStopTheMusic::Plugin` | Architecture Patterns, Pattern 3 | DSTM-Registration schlägt fehl bei alten LMS-Versionen — kein echtes Risiko da SpotOn 8.0 als Floor hat |
| A2 | `recommendations`-Endpoint ist via Herger-Bundled-ID verfügbar (Extended Quota) | Architecture Patterns, Pattern 5 (Fallback) | DSTM fällt auf Search-Fallback zurück — funktional, aber weniger Relevanz der Empfehlungen |
| A3 | GitHub Raw URL für repo.xml ist von LMS-Instanzen aus dem Internet erreichbar | Pattern 7 | Interne LMS-Instanzen ohne Internet-Zugang können Plugin nicht via Repository installieren |

---

## Open Questions

1. **Normalisierung im Connect-Modus**
   - Was wir wissen: `Daemon.pm` soll `--enable-volume-normalisation` übergeben wenn der globale Pref gesetzt ist. Das ist in CONTEXT.md D-02 als Anforderung gelistet ("muss auch für Connect-Modus gelten").
   - Was unklar ist: Ob `Daemon.pm` das bereits tut (CONTEXT.md sagt "Daemon.pm übergibt bereits") oder ob es noch fehlt.
   - Empfehlung: `Connect/Daemon.pm` oder `Connect/DaemonManager.pm` nach `normalization`-Pref-Lese-Code suchen. Wenn vorhanden: nur verifizieren. Wenn fehlend: als Task hinzufügen.

2. **Binary-Build-Pipeline**
   - Was wir wissen: Cross.toml + cross-rs ist vorhanden. Das aarch64-musl-Target wurde bereits gebaut (Target-Verzeichnis vorhanden). Die anderen Targets (armhf, i386) sind noch nicht gebaut.
   - Was unklar ist: Ob ein CI-basierter Build (GitHub Actions) für die Release-Pipeline geplant ist oder ob der User manuell baut.
   - Empfehlung: Dokumentierten Build-Befehl als Task anlegen. GitHub Actions für Multi-Arch-Builds ist für Early Adopter noch nicht zwingend — manueller Build via `cross build --target armv7-unknown-linux-musleabihf` reicht.

3. **Security Review Scope**
   - Was wir wissen: "Full security review" ist als Success Criterion gelistet. Keine konkreten Funde bekannt.
   - Was unklar ist: Welche Module und welche Tiefe (Code Review nur, oder auch Penetration-Test-Szenarien?).
   - Empfehlung: Code Review aller Module mit User-Input-Verarbeitung: Settings.pm (Form-Inputs), TokenManager.pm (Credentials), Helper.pm (Binary-Path), API/Client.pm (URL-Konstruktion). Priorität: Shell-Injection-Vektoren (binary path, bitrate injection) und Pref-Validierung.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| cross (Rust cross-compilation) | LMS-06 Multi-Arch-Binaries | Nicht verifiziert | — | Manueller Build auf ARM-Hardware |
| sha1sum | repo.xml SHA1-Berechnung | ✓ (Standard-Linux-Tool) | coreutils | `openssl dgst -sha1` |
| LMS 8.0+ | DSTM-Framework | ✓ (installiert) | 9.2.0-dev | — |

---

## Validation Architecture

Die Phase hat keine automatisierbaren Tests im LMS-Plugin-Kontext. Validierung erfolgt manuell gegen die 12 Success Criteria.

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | Bemerkung |
|--------|----------|-----------|-------------------|-----------|
| LMS-03 | Alle UI-Strings erscheinen in der eingestellten Browser-Sprache | manual | — | LMS auf DE/FR stellen, Settings-Seite laden |
| LMS-06 | Binaries auf x86_64, aarch64, armhf, i386 passen `--check` | manual | `./Bin/x86_64-linux/spoton -n test --check` | Für jede Arch ausführen |
| LMS-08 | Bitrate 96 auf Player A, 320 auf Player B → beide streamen korrekt | manual | — | Zwei-Player-Setup, netcap/log prüfen |
| LMS-09 | Playlist endet → DSTM queued nächsten Track automatisch | manual | — | DontStopTheMusic in LMS Player-Settings aktivieren |
| LMS-10 | `spoton-custom` Binary in Bin-Pfad → Plugin verwendet es | manual | Helper.pm::get() log prüfen | Custom-Binary in Pfad legen, LMS neu starten |

### Wave 0 Gaps

- [ ] `DontStopTheMusic.pm` — neue Datei, muss erstellt werden
- [ ] Test für Exporter-Setup in `Client.pm` — sicherstellen dass Konstante importierbar ist

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | ja (Token-Handling) | TokenManager.pm — Keymaster-Flow |
| V4 Access Control | nein | Plugin läuft im LMS-Kontext, kein eigenes Auth |
| V5 Input Validation | ja | Settings.pm: bitrate-Override, binary-Pfad, clientId |
| V6 Cryptography | nein | Kein eigenes Crypto; LMS/Keymaster handelt TLS |

### Known Threat Patterns for LMS Plugin Stack

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Shell-Injection via binary path | Tampering | `Helper.pm::helperCheck()` hat bereits Shell-Safe-Quoting: `(my $safe = $candidate) =~ s/'/'\\''/g` |
| Bitrate-Parameter-Injection | Tampering | Settings.pm validiert bereits: `unless $valid_bitrates{$bitrate}` — per-Player muss gleiche Validierung haben |
| ClientId-Injection in CLI-Flags | Tampering | Settings.pm sanitiert bereits: `$id =~ s/[^a-zA-Z0-9]//g; substr($id, 0, 32)` — gilt für globale ID; per-Player benötigt gleiche Validierung wenn zukünftig per-Player ClientId eingeführt wird |
| Path-Traversal via removeAccount | Tampering | Settings.pm prüft bereits: `$removeId =~ /\A[0-9a-f]{8}\z/` |

**Security Review Fokus für Phase 6:**
- Per-Player Bitrate-Override: Regex-Validierung vor Injection in `updateTranscodingTable()`
- Custom-Binary-Pfad: Shell-Quoting bereits vorhanden; prüfen ob absolute Pfad-Traversal möglich ist
- DSTM-Seed-Daten: Kommen von `getMixableProperties()` — LMS-intern, keine externe Injection-Gefahr
- repo.xml URL-Feld: Nur SHA1-verifikation durch LMS; kein aktiver Trust-Vektor für SpotOn selbst

---

## Sources

### Primary (HIGH confidence)
- `/home/sti/spotty-ng/Spotty-Plugin/DontStopTheMusic.pm` — Vollständige DSTM-Referenzimplementierung, lokal gelesen [VERIFIED: lokale Datei]
- `/home/sti/spotty-ng/Spotty-Plugin/Plugin.pm` Zeile 164-168 — DSTM-Registration Pattern, lokal gelesen [VERIFIED: lokale Datei]
- `/home/sti/spotty-ng/Spotty-Plugin/API.pm` Zeile 1308-1349 — recommendations() Implementierung, lokal gelesen [VERIFIED: lokale Datei]
- `/home/sti/spoton/Plugins/SpotOn/` — gesamte bestehende Codebase [VERIFIED: lokale Dateien]
- `https://lyrion.org/reference/repository-dev/` — repo.xml Format-Spezifikation [VERIFIED: offizielle LMS-Dokumentation]
- `CLAUDE.md` §LMS Plugin API — Prefs, i18n, DontStopTheMusic [VERIFIED: Projektdatei]

### Secondary (MEDIUM confidence)
- `https://github.com/LMS-Community/lms-plugin-repository/blob/master/extensions.xml` — Beispiel-Plugin-Einträge [CITED: offizielles LMS-Repository]
- `/home/sti/spotty-ng/Spotty-Plugin/strings.txt` — Sprachpalette und Format-Referenz [VERIFIED: lokale Datei]

---

## Metadata

**Confidence breakdown:**
- DSTM-Implementation: HIGH — vollständige Referenz aus Spotty-NG, klare Adaptation für SpotOn-API
- Per-Player-Prefs: HIGH — Pattern bereits etabliert in Settings.pm, chirurgische Erweiterung
- Format-Dropdown Unified: HIGH — Logik klar aus CONTEXT.md D-11, Code-Pattern bekannt
- Client-ID-Konsolidierung: HIGH — mechanisch, zwei bekannte Stellen
- i18n: HIGH — Format bekannt, 65 Keys vorhanden, 11 Sprachen erforderlich
- repo.xml: HIGH — offizielle Dokumentation gelesen, Format klar
- Multi-Arch-Binaries: MEDIUM — Build-Toolchain (cross) vorhanden aber nicht verifiziert ob alle Targets funktionieren
- Security Review: MEDIUM — bekannte Vektoren analysiert, Tiefe des Reviews noch offen

**Research date:** 2026-06-03
**Valid until:** 2026-07-03 (stabile LMS-API, 30 Tage Gültigkeit)
