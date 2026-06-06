# Phase 11: Track History Metadata - Research

**Researched:** 2026-06-04
**Domain:** LMS ProtocolHandler metadata persistence, async re-fetch, Connect→Browse URL translation
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01:** Connect-Metadata wird im selben `spoton_meta_` Cache persistiert wie Browse-Metadata. `_fetchTrackMetadata` in Connect.pm speichert den Datensatz zusaetzlich unter `spoton_meta_` + md5(connect-URL) — enthaelt cover, title, artist, album, duration, bitrate, type UND die echte `spotify:track:ID` als `spotifyUri` Feld.

**D-02:** TTL fuer alle Track-Metadata (Browse + Connect) einheitlich 604800s (7 Tage). Ersetzt den bisherigen 3600s TTL. Spotify Artwork-URLs halten deutlich laenger als 7 Tage.

**D-03:** Async Re-Fetch mit Placeholder. `getMetadataFor` gibt bei Cache-Miss sofort Minimal-Metadata zurueck (generisches Icon, Title aus URL wenn parsbar), startet einen async API-Call via `SimpleAsyncHTTP`, cached das Ergebnis, und feuert `Slim::Control::Request::notifyFromArray($client, ['newmetadata'])` damit LMS die Anzeige aktualisiert.

**D-04:** Connect-Cache-Eintraege enthalten ein `spotifyUri` Feld (z.B. `spotify:track:ABC123`) fuer Re-Fetch nach Expiry. Bei Browse-URLs wird die Track-ID direkt aus dem URL extrahiert (`spotify://track:ID`).

**D-05:** Ein laufender Re-Fetch pro Track-URL — Debounce via Package-Hash (`%_pendingRefetch`). Verhindert Doppel-Fetches bei schnellem History-Durchblaettern und schont das Spotify Rate-Limit.

**D-06:** Connect-Tracks in der History werden transparent zu Browse-URLs uebersetzt. Wenn `getMetadataFor` fuer eine `spotify://connect-*` URL aufgerufen wird und ein `spotifyUri` im Cache liegt, wird die abspielbare Browse-URL verfuegbar gemacht.

**D-07:** Uebersetzung ist unsichtbar — kein visueller Unterschied zwischen ehemaligen Connect- und Browse-Tracks in der History. Type-String wird nicht als "Connect" markiert.

### Claude's Discretion

- Technische Integration der URL-Translation (via getMetadataFor Redirect, canDirectStream, oder anderer LMS-Mechanismus) — Hauptsache Connect-Tracks aus der History sind abspielbar via Browse-Pipeline
- Placeholder-Inhalt bei Cache-Miss (Title-Parsing aus URL, generisches Icon-Wahl)
- Exakte Struktur des `%_pendingRefetch` Debounce-Hash
- Ob der Browse-Cache-TTL-Bump (3600s → 604800s) auch DontStopTheMusic.pm betrifft

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| HIST-01 | Browse-mode tracks replayed from history show correct album artwork (not generic icon) | TTL-Bump auf 604800s + async Re-Fetch bei Cache-Miss in `getMetadataFor` |
| HIST-02 | Browse-mode tracks replayed from history show correct streaming format and bitrate in Songinfo | Bestehende `_typeString`/`_bitrateForClient` Pipeline greift wenn Cache-Eintrag vorhanden; Re-Fetch stellt Metadata wieder her |
| HIST-03 | Connect-mode tracks in history are translatable to Browse URLs and can be replayed | `spotifyUri` Feld im Connect-Cache-Eintrag ermoeglichen Uebersetzung in `spotify://track:ID` URL |
| HIST-04 | Cache-miss in getMetadataFor triggers async API re-fetch and populates metadata for expired entries | `%_pendingRefetch` Debounce + `API::Client->getTrack` + `notifyFromArray(['newmetadata'])` |

</phase_requirements>

---

## Summary

Phase 11 ist eine reine Perl-Phase ohne Binary-Aenderungen. Das Ziel ist, dass der LMS Track-Verlauf ("Was lief da eben?") fuer alle Spotify-Tracks korrekte Metadata anzeigt — auch nach Cache-Expiry und auch fuer Connect-Mode-Tracks.

**Das Kernproblem heute:** `getMetadataFor` in ProtocolHandler.pm gibt `{}` zurueck wenn der Cache-Eintrag abgelaufen ist (TTL 3600s). Das loest im Verlauf ein generisches Icon und keine Songinfo-Details aus. Fuer Connect-Tracks fehlt ausserdem die Cache-Persistenz komplett — `_fetchTrackMetadata` schreibt nur in `pluginData('info')` (ephemer, geht mit Song-Ende verloren), nicht in den persistenten Cache.

**Loesungsansatz** (aus CONTEXT.md, alle Entscheidungen locked):
1. TTL aller Track-Metadata-Caches von 3600s auf 604800s (7 Tage) erhoehen — in `_trackItem`, `_albumTrackItem` (Plugin.pm) und DontStopTheMusic.pm
2. `_fetchTrackMetadata` in Connect.pm: Cache-Persistenz unter `spoton_meta_` + md5(connect-URL) hinzufuegen, mit `spotifyUri` Feld
3. `getMetadataFor` in ProtocolHandler.pm: Async Re-Fetch bei Cache-Miss statt `{}`, mit Debounce via `%_pendingRefetch`
4. Connect→Browse URL-Translation: `getMetadataFor` erkennt expired Connect-URL, extrahiert `spotifyUri` aus Cache (falls noch vorhanden), liefert Browse-faehige Metadata zurueck

**Primary recommendation:** Alle vier Aenderungen in einem Plan, da sie eng zusammenhaengen und gemeinsam getestet werden muessen.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Metadata-Persistenz (Browse) | Plugin (Perl) | Cache (Slim::Utils::Cache) | `_trackItem`/`_albumTrackItem` befuellen Cache bei Menu-Navigation |
| Metadata-Persistenz (Connect) | Plugin (Perl) — Connect.pm | Cache | `_fetchTrackMetadata` muss Cache zusaetzlich zu pluginData befuellen |
| Cache-Miss Re-Fetch | Plugin (Perl) — ProtocolHandler.pm | API-Tier (Spotify Web API) | `getMetadataFor` ist der einzige Konsument; API-Call via `API::Client->getTrack` |
| Connect→Browse URL-Translation | Plugin (Perl) — ProtocolHandler.pm | — | Logik gehoert in getMetadataFor, nicht in die Playback-Pipeline |
| Debounce (Doppel-Fetch-Schutz) | Plugin (Perl) — ProtocolHandler.pm | — | Package-Var `%_pendingRefetch`, nicht LMS-seitig |

---

## Standard Stack

Kein neues Paket benoetigt. Alle eingesetzten Module sind LMS-intern oder bereits importiert.

### Core (bereits vorhanden)

| Module | Zweck in Phase 11 | Bereits importiert in |
|--------|------------------|----------------------|
| `Slim::Utils::Cache` | Cache get/set mit TTL | ProtocolHandler.pm, Plugin.pm, DontStopTheMusic.pm |
| `Digest::MD5 qw(md5_hex)` | Cache-Key-Bildung | ProtocolHandler.pm, Plugin.pm |
| `Slim::Control::Request` | `notifyFromArray` fuer newmetadata Event | Connect.pm (bereits genutzt, Zeile 869) |
| `Plugins::SpotOn::API::Client` | `getTrack($accountId, $trackId, $cb)` | Connect.pm (Zeile 803) |
| `Plugins::SpotOn::Plugin` | `_typeString`, `_bitrateForClient` | ProtocolHandler.pm (Zeile 297) |

### Neue Muster (kein neues Modul)

- `%_pendingRefetch` — Package-Level-Hash in ProtocolHandler.pm als Debounce-State
- On-demand `require Plugins::SpotOn::API::Client` in ProtocolHandler.pm (bereits Muster fuer andere requires dort)

---

## Package Legitimacy Audit

Keine externen Pakete werden installiert. Diese Phase aendert nur vorhandenen Perl-Code.

**Packages removed due to slopcheck [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

---

## Architecture Patterns

### System Architecture Diagram

```
LMS Track History / NowPlaying Request
          |
          v
  getMetadataFor(class, client, url)
     [ProtocolHandler.pm line 268]
          |
   +------+---------------------+
   |                            |
   | url =~ connect-            | url =~ track:ID
   |                            |
   v                            v
pluginData('info')          Cache lookup
  [ephemer, only          spoton_meta_ + md5(url)
   during Connect              |
   session]                    |
   |                    +------+------+
   |                    |             |
   |                  HIT           MISS
   |                    |             |
   |               return meta   %_pendingRefetch
   |             + _typeString   check (Debounce)
   |             + _bitrateFor-      |
   |               Client      already    new
   |                           pending    fetch
   |                              |         |
   |                           return   API::Client
   |                          minimal   ->getTrack()
   |                          placeholder   |
   |                                        v
   |                               callback: set cache
   |                               notifyFromArray
   |                               ['newmetadata']
   |
   +-- Connect URL + spotifyUri in expired cache:
         translate to spotify://track:ID
         return Browse-pipeline metadata
```

### Recommended Project Structure

Keine strukturellen Aenderungen. Alle Aenderungen erfolgen in bestehenden Dateien:

```
Plugins/SpotOn/
├── ProtocolHandler.pm   # getMetadataFor(): async re-fetch, %_pendingRefetch, Connect→Browse translate
├── Connect.pm           # _fetchTrackMetadata(): cache persist + spotifyUri field
├── Plugin.pm            # _trackItem(), _albumTrackItem(): TTL 3600 -> 604800
└── DontStopTheMusic.pm  # cache set line 265: TTL 3600 -> 604800 (Claude's Discretion)
```

### Pattern 1: Cache-Miss Async Re-Fetch mit Debounce

[VERIFIED: codebase inspection] Dieses Muster ist eine Erweiterung des bestehenden Connect.pm `_fetchTrackMetadata` Musters (Zeilen 808-877).

```perl
# In ProtocolHandler.pm — package-level debounce hash
my %_pendingRefetch;

sub getMetadataFor {
    my ($class, $client, $url) = @_;

    # ... bestehende Connect pluginData-Logik ...
    # ... bestehende Cache-Lookup-Logik (canonical URL) ...

    # Cache-Miss: bestehende return {} Stelle wird ersetzt
    unless ($meta) {
        _asyncRefetch($class, $client, $url);
        return _placeholderMeta($url);  # sofort zurueck, kein Blockieren
    }

    # ... bestehende return $meta Logik ...
}

sub _asyncRefetch {
    my ($class, $client, $url) = @_;

    # D-05: Debounce — nur ein laufender Fetch pro URL
    return if $url && $_pendingRefetch{$url};

    # Track-ID ermitteln: Browse-URL direkt, Connect-URL via spotifyUri
    my $trackId;
    if ($url =~ m{spotify://track:([A-Za-z0-9]+)}) {
        $trackId = $1;
    } elsif ($url =~ m{spotify://connect-}) {
        my $connectMeta = $cache->get('spoton_meta_' . md5_hex($url));
        $trackId = $connectMeta->{spotifyUri} if $connectMeta;
        $trackId =~ s{^spotify:track:}{} if $trackId;  # strip uri prefix
    }
    return unless $trackId;

    my $accountId = $prefs->client($client)->get('activeAccount')
                 || $prefs->get('activeAccount')
                 || '';

    $_pendingRefetch{$url} = 1;
    require Plugins::SpotOn::API::Client;
    Plugins::SpotOn::API::Client->getTrack($accountId, $trackId, sub {
        my ($trackInfo) = @_;
        delete $_pendingRefetch{$url};
        return unless $trackInfo && $trackInfo->{name};

        # Cache befuellen (analog zu _trackItem in Plugin.pm)
        require Plugins::SpotOn::Plugin;
        my $artist = join(', ', map { $_->{name} } @{ $trackInfo->{artists} || [] });
        my $cover  = _largestImage(($trackInfo->{album} || {})->{images}) || IMG_TRACK;
        my $cacheUrl = ($url =~ m{spotify://connect-})
            ? 'spotify://track:' . $trackId  # canonical Browse-URL fuer Connect-History
            : $url;
        $cache->set('spoton_meta_' . md5_hex($cacheUrl), {
            title    => $trackInfo->{name},
            artist   => $artist,
            album    => ($trackInfo->{album} || {})->{name} || '',
            duration => ($trackInfo->{duration_ms} || 0) / 1000,
            cover    => $cover,
            icon     => $cover,
        }, 604800);  # D-02: 7 Tage

        # LMS Anzeige aktualisieren
        if ($client) {
            Slim::Control::Request::notifyFromArray($client, ['newmetadata']);
        }
    });
}
```

**Hinweis:** `IMG_TRACK` ist in Connect.pm definiert als `'/html/images/cover.png'`. Fuer ProtocolHandler.pm muss entweder das Constant importiert oder der String direkt verwendet werden.

### Pattern 2: Connect-Cache-Persistenz mit spotifyUri Feld

[VERIFIED: codebase inspection] Erweiterung von `_fetchTrackMetadata` in Connect.pm (Zeilen 847-857).

```perl
# NACH der bestehenden pluginData(info => {...}) Zeile in _fetchTrackMetadata:
my $connectUrl = $song->streamUrl || '';
if ($connectUrl) {
    require Slim::Utils::Cache;
    my $cacheObj = Slim::Utils::Cache->new();
    $cacheObj->set('spoton_meta_' . md5_hex($connectUrl), {
        title      => $title,
        artist     => $artist,
        album      => $album,
        duration   => $duration,
        cover      => $cover,
        icon       => $cover,
        bitrate    => $bitrate . 'k',
        type       => $type_str,
        spotifyUri => $trackInfo->{uri},  # D-01: z.B. "spotify:track:ABC123"
    }, 604800);  # D-02: 7 Tage
}
```

`md5_hex` ist bereits via `use Digest::MD5 qw(md5_hex)` in Connect.pm importiert.

### Pattern 3: Connect→Browse URL-Translation

[VERIFIED: codebase inspection] D-06/D-07: Connect-History-Tracks transparent zu Browse-URLs uebersetzen.

Der zentrale Mechanismus: `getMetadataFor` gibt fuer eine `spotify://connect-*` URL (aus der History) Browse-faehige Metadata zurueck, wenn `spotifyUri` im Cache liegt. Die abspielbare `spotify://track:ID` URL kann als `play`-Feld in der Metadata mitgeliefert werden — LMS kann das direkt nutzen.

```perl
# In getMetadataFor, Connect-URL-Zweig (nach pluginData-Check):
if ($url && $url =~ m{spotify://connect-}) {
    # 1. pluginData (aktive Session) — bestehend
    ...
    # 2. Persistierter Cache-Eintrag (History nach Session-Ende)
    my $connMeta = $cache->get('spoton_meta_' . md5_hex($url));
    if ($connMeta && $connMeta->{spotifyUri}) {
        my ($trackId) = $connMeta->{spotifyUri} =~ /^spotify:track:([A-Za-z0-9]+)$/;
        if ($trackId) {
            my $browseUrl = "spotify://track:$trackId";
            # Metadata mit Browse-URL zurueckliefern (D-07: kein "Connect"-Label)
            require Plugins::SpotOn::Plugin;
            return {
                %$connMeta,
                type    => Plugins::SpotOn::Plugin->_typeString($client, 'Browse'),
                bitrate => Plugins::SpotOn::Plugin->_bitrateForClient($client) . 'k',
                play    => $browseUrl,  # D-06: abspielbare Browse-URL
            };
        }
    }
}
```

### Anti-Patterns to Avoid

- **Blocking in getMetadataFor:** LMS ist single-threaded. NIEMALS `SimpleSyncHTTP` oder direkte API-Calls in `getMetadataFor` — immer async mit Callback und sofortiger Placeholder-Rueckgabe.
- **Double-Fetch ohne Debounce:** History-Browse kann getMetadataFor mehrfach pro Track in kurzer Zeit aufrufen. Ohne `%_pendingRefetch` wuerden viele parallele API-Calls gestartet.
- **Cache-Key Inkonsistenz:** Browse-URLs sind `spotify://track:ID`. Connect-URLs sind `spotify://connect-<ts>`. Beim Re-Fetch fuer Connect-History muss der Result-Cache unter der Browse-URL gespeichert werden (nicht unter der Connect-URL), damit zukunftige History-Eintraege den gleichen Track direkt treffen.
- **spotifyUri nicht ins pluginData schreiben:** Das `spotifyUri` Feld gehoert in den persistenten Cache, nicht in `pluginData('info')`. pluginData ist ephemer und geht beim naechsten Track verloren.
- **TTL-Bump vergessen in DontStopTheMusic.pm:** Zeile 265 hat ebenfalls `3600` — Vergessen wuerde zu inkonsistenten Expiry-Zeiten fuehren (DSTM-Tracks verfallt nach 1h, normal Browse-Tracks nach 7 Tagen).

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Warum |
|---------|-------------|-------------|-------|
| Async HTTP in getMetadataFor | Eigenes HTTP-Handling | `API::Client->getTrack` (nutzt bereits SimpleAsyncHTTP intern) | Pattern etabliert in Connect.pm, Zeilen 808-877 |
| Cache-Expiry erkennen | Eigene TTL-Buchhaltung | `$cache->get` gibt undef bei Expiry — das ist die etablierte Schnittstelle | Slim::Utils::Cache verwaltet TTL intern |
| LMS Display-Update nach async Fetch | Timer/Poll-Loop | `Slim::Control::Request::notifyFromArray($client, ['newmetadata'])` | Exakt dasselbe Pattern wie Connect.pm Zeile 869 |
| Debounce via Timers | Slim::Utils::Timers | Package-Hash `%_pendingRefetch` | Einfacher, kein Timer-Overhead, wird im Callback geloescht |

---

## Common Pitfalls

### Pitfall 1: IMG_TRACK Constant nicht verfuegbar in ProtocolHandler.pm

**What goes wrong:** `IMG_TRACK` ist in Connect.pm als `use constant IMG_TRACK => '/html/images/cover.png'` definiert. ProtocolHandler.pm hat dieses Constant nicht.

**Why it happens:** Fehlende cross-module Constant-Definition.

**How to avoid:** Entweder `use constant IMG_TRACK => '/html/images/cover.png'` in ProtocolHandler.pm hinzufuegen, oder den String direkt als Literal verwenden. Der String ist kurz und aendert sich nicht.

**Warning signs:** `Bareword "IMG_TRACK" not allowed` Syntax-Fehler beim `perl -c` Check.

### Pitfall 2: `$client` ist undef wenn LMS getMetadataFor fuer History aufruft

**What goes wrong:** LMS kann `getMetadataFor` ohne aktiven Client aufrufen (z.B. beim Laden der History-Liste). In diesem Fall kann kein `notifyFromArray` gefeuert werden.

**Why it happens:** History-Items koennen abgefragt werden, waehrend kein Player aktiv ist.

**How to avoid:** Vor `notifyFromArray` und vor `_bitrateForClient($client)` immer `if ($client)` prufen. Der Async-Callback muss den `$client` per Closure einfangen — wenn der Client zu diesem Zeitpunkt undef war, darf kein notify gefeuert werden.

**Warning signs:** "Can't call method on undef" Fehler im Log.

### Pitfall 3: Cache-Key-Diskrepanz zwischen Connect-URL und Browse-URL

**What goes wrong:** Connect-History speichert unter `spoton_meta_` + md5(connect-URL). Der Re-Fetch schreibt aber unter `spoton_meta_` + md5(browse-URL). Beim naechsten Aufruf von `getMetadataFor` fuer die gleiche Connect-URL wird der Re-Fetch Result nicht gefunden.

**Why it happens:** Zwei verschiedene URLs fuer denselben Track.

**How to avoid:** Fuer Connect-History Tracks gibt es zwei Cache-Eintraege:
1. Unter `connect-URL` (gesetzt von `_fetchTrackMetadata`): enthaelt `spotifyUri` Feld, ermoeglicht Translation
2. Unter `browse-URL` (gesetzt vom Re-Fetch-Callback): enthaelt vollstaendige Metadata fuer direkten Browse-Zugriff

Beide sind noetig und erganzen sich.

### Pitfall 4: Stale `%_pendingRefetch` Eintrag nach API-Fehler

**What goes wrong:** Wenn der `getTrack` API-Call mit `undef` oder leerem `$trackInfo` zurueckkommt (Netzwerkfehler, Rate-Limit), wird `delete $_pendingRefetch{$url}` moeglicherweise nicht aufgerufen.

**Why it happens:** Fehlerpfad im Callback wird nicht beruecksichtigt.

**How to avoid:** `delete $_pendingRefetch{$url}` IMMER als erstes im Callback aufrufen, unabhaengig vom Ergebnis — bevor irgendwelche `return unless` Guards.

### Pitfall 5: URL-Normalisierung wird im Re-Fetch-Callback vergessen

**What goes wrong:** `getMetadataFor` normalisiert `spotify:track:ID` zu `spotify://track:ID` fuer den Cache-Lookup. Der Re-Fetch-Callback muss den gleichen kanonischen URL als Cache-Key verwenden.

**Why it happens:** Zwei Code-Pfade, eine URL-Form.

**How to avoid:** Die gleiche Normalisierungs-Logik aus `getMetadataFor` in `_asyncRefetch` wiederverwenden, oder eine Hilfsfunktion `_canonicalUrl($url)` extrahieren.

---

## Code Examples

### Bestehende notifyFromArray-Nutzung (Referenz-Pattern)

```perl
# Source: Plugins/SpotOn/Connect.pm line 869
# Fire newmetadata notification so LMS refreshes Now Playing
Slim::Control::Request::notifyFromArray($client, ['newmetadata']);
```

### Bestehende getTrack-Nutzung (Referenz-Pattern)

```perl
# Source: Plugins/SpotOn/Connect.pm lines 808-877
Plugins::SpotOn::API::Client->getTrack($accountId, $trackId, sub {
    my ($trackInfo) = @_;
    return unless $trackInfo && $trackInfo->{name};
    # ... Metadata verarbeiten ...
});
```

### Bestehender Cache-Set mit TTL (Referenz-Pattern)

```perl
# Source: Plugins/SpotOn/Plugin.pm line 398 (_trackItem)
$cache->set('spoton_meta_' . md5_hex($spotify_url), {
    title    => $title,
    artist   => $artist,
    album    => $album,
    duration => $duration,
    cover    => $image,
    icon     => $image,
    bitrate  => __PACKAGE__->_bitrateForClient($client) . 'k',
    type     => __PACKAGE__->_typeString($client, 'Browse'),
}, 3600);  # Phase 11: wird auf 604800 erhoehen
```

### Placeholder-Meta bei Cache-Miss

```perl
# Source: ASSUMED — neues Pattern, angelehnt an Connect.pm IMG_TRACK Fallback
sub _placeholderMeta {
    my ($url) = @_;
    my $title = '';
    if ($url =~ m{spotify://track:([A-Za-z0-9]+)}) {
        $title = "Loading...";  # oder leer lassen — Claude's Discretion
    }
    return {
        cover => '/html/images/cover.png',  # generisches LMS-Icon
        icon  => '/html/images/cover.png',
        title => $title,
    };
}
```

---

## State of the Art

| Alter Ansatz | Aktueller Ansatz (nach Phase 11) | Impact |
|-------------|----------------------------------|--------|
| Cache-Miss → `return {}` | Cache-Miss → Placeholder + async Re-Fetch | HIST-04: History zeigt nach kurzer Verzoegerung korrekte Metadata |
| Connect-Metadata nur in pluginData (ephemer) | Connect-Metadata auch im Cache mit `spotifyUri` Feld | HIST-01/HIST-03: Connect-History-Tracks zeigen Artwork, sind abspielbar |
| TTL 3600s (1 Stunde) | TTL 604800s (7 Tage) | HIST-01/HIST-02: Metadata ueberlebt mehrtaegige History |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `img_track = '/html/images/cover.png'` ist der korrekte generische Fallback-Artwork-Pfad in ProtocolHandler.pm | Code Examples (Placeholder) | Visueller Fallback zeigt broken image — einfach zu korrigieren |
| A2 | LMS ruft `getMetadataFor` fuer History-Items auf (nicht nur fuer aktiv spielende Tracks) | Architecture Patterns | Wenn LMS History-Metadata anders bezieht, greift der Re-Fetch-Mechanismus nicht — Verhalten muss im UAT verifiziert werden |
| A3 | `play`-Feld in Metadata-Hash wird von LMS fuer History-Replay genutzt | Pattern 3 (Connect→Browse Translation) | Connect-History-Tracks koennen moeglicherweise nicht direkt replay'd werden — alternativer Mechanismus erforderlich |

---

## Open Questions (RESOLVED)

1. **Mechanism fuer Connect→Browse Replay (D-06, Claude's Discretion)** — RESOLVED: Plan implements `play` field approach (Claude's Discretion); Assumption A3 risk accepted and verified at UAT in Plan 02 Task 2 checkpoint step 7.
   - Was wir wissen: `getMetadataFor` kann Browse-Metadata mit `play`-Feld zurueckliefern
   - Was unklar ist: Ob LMS das `play`-Feld aus getMetadataFor fuer History-Replay nutzt, oder ob ein anderer Mechanismus (z.B. canDirectStream, URL-Rewrite) noetig ist
   - Recommendation: Placeholder-Test im UAT: Connect-Track im History anklicken, pruefen ob LMS die Browse-URL direkt oder via Redirect nutzt. Falls `play`-Feld nicht reicht, als Alternative pruefen ob LMS `canHandleURI` fuer connect-URLs die URL substituieren kann.

2. **DontStopTheMusic.pm TTL-Bump (Claude's Discretion)** — RESOLVED: DontStopTheMusic.pm TTL bump included in Plan 01 Task 2 action B.
   - Was wir wissen: DSTM.pm setzt Cache an Zeile 265 mit TTL 3600s, mit `$client = undef` (kein per-player Kontext)
   - Was unklar ist: Ob DSTM-Tracks in der History relevant fuer Phase 11 sind (DSTM laeuft waehrend einer Session, History danach)
   - Recommendation: TTL-Bump konsistenzhalber auf 604800s setzen — kein Risiko, vereinheitlicht das Verhalten.

---

## Environment Availability

Keine externen Abhaengigkeiten. Reine Perl-Aenderungen in bestehenden Dateien. Alle benoetigten Module sind LMS-intern.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Perl 5.38 | Plugin | ✓ | 5.38.2 | — |
| Test::More | Test-Suite | ✓ | System | — |
| Slim::Utils::Cache | Metadata-Persistenz | ✓ (LMS-intern) | — | — |
| Slim::Control::Request | newmetadata Notify | ✓ (LMS-intern) | — | — |
| Plugins::SpotOn::API::Client | getTrack | ✓ (vorhanden) | — | — |

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Test::More (Perl built-in) |
| Config file | none — test files in `t/` run directly |
| Quick run command | `perl t/05_perl_syntax.t && perl t/11_track_history.t` |
| Full suite command | `prove t/` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| HIST-01 | Browse-Track nach Cache-Expiry zeigt Cover | unit | `perl t/11_track_history.t` | ❌ Wave 0 |
| HIST-02 | Browse-Track nach Cache-Expiry zeigt Format+Bitrate | unit | `perl t/11_track_history.t` | ❌ Wave 0 |
| HIST-03 | Connect-Track aus History ist als Browse-URL abspielbar | unit | `perl t/11_track_history.t` | ❌ Wave 0 |
| HIST-04 | Cache-Miss loest async Re-Fetch aus + notifyFromArray | unit | `perl t/11_track_history.t` | ❌ Wave 0 |
| TTL-Check | alle cache->set Aufrufe verwenden 604800 | grep-gate | `perl t/11_track_history.t` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `perl t/05_perl_syntax.t`
- **Per wave merge:** `prove t/`
- **Phase gate:** `prove t/` vollstaendig gruen vor `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `t/11_track_history.t` — abdeckend: HIST-01 bis HIST-04, TTL-Grep-Gate, Debounce-Check
- [ ] ProtocolHandler.pm muss fuer Unit-Test isoliert ladbar sein (LMS-Stub-Pattern aus `t/10_stream_metadata.t` wiederverwenden)

---

## Security Domain

Phase 11 aendert ausschliesslich Metadata-Caching-Logik und asynchrone API-Fetches. Keine neuen Angriffsvektoren.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | — |
| V3 Session Management | no | — |
| V4 Access Control | no | — |
| V5 Input Validation | yes (minimal) | Track-ID aus URL via Regex extrahiert — nur `[A-Za-z0-9]+` akzeptiert |
| V6 Cryptography | no | — |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Malformed Spotify-URL in History | Tampering | Regex-Extraktion `m{spotify://track:([A-Za-z0-9]+)}` begrenzt Track-ID auf alphanumerische Zeichen — keine Injection moeglich |
| URL im Cache-Key | Tampering | md5_hex(url) — Cache-Key ist kein ausfuehrbarer String, kein Risiko |

---

## Sources

### Primary (HIGH confidence)

- Codebase: `Plugins/SpotOn/ProtocolHandler.pm` — vollstaendige Analyse der getMetadataFor-Implementierung (Zeilen 261-304) [VERIFIED: codebase inspection]
- Codebase: `Plugins/SpotOn/Connect.pm` — vollstaendige Analyse von `_fetchTrackMetadata` (Zeilen 794-878) [VERIFIED: codebase inspection]
- Codebase: `Plugins/SpotOn/Plugin.pm` — `_trackItem` (Zeilen 358-424), `_albumTrackItem` (Zeilen 1098-1180), `_typeString` (Zeilen 1352-1372), `_bitrateForClient` (Zeilen 1338-1350) [VERIFIED: codebase inspection]
- Codebase: `Plugins/SpotOn/DontStopTheMusic.pm` — Cache-Set an Zeile 265 mit TTL 3600s [VERIFIED: codebase inspection]
- Codebase: `t/10_stream_metadata.t` — Stub-Pattern fuer Unit-Tests (ProtocolHandler-Isolation) [VERIFIED: codebase inspection]
- Phase 11 CONTEXT.md — alle locked Decisions D-01 bis D-07 [VERIFIED: planning artifact]

### Secondary (MEDIUM confidence)

- Phase 9 CONTEXT.md — etablierte `_typeString`/`_bitrateForClient` Patterns und deren Integration in `getMetadataFor` [CITED: .planning/phases/09-stream-metadata/09-CONTEXT.md]

### Tertiary (LOW confidence)

- A2: LMS ruft getMetadataFor fuer History-Items auf — basiert auf LMS Plugin API Kenntnis, nicht verifiziert durch LMS-Source-Inspektion in dieser Session [ASSUMED]
- A3: `play`-Feld in Metadata-Hash fuer History-Replay nutzbar [ASSUMED]

---

## Metadata

**Confidence breakdown:**
- Standard Stack: HIGH — alle Module bestehen bereits, keine neuen Abhaengigkeiten
- Architecture: HIGH — alle vier Aenderungspunkte exakt identifiziert mit Zeilennummern, Muster aus Phase 9/10 direkt uebertragbar
- Pitfalls: HIGH — aus Codebase-Analyse abgeleitet (Constant-Scope, undef-Client, Cache-Key-Diskrepanz)
- Test-Strategy: HIGH — etabliertes Stub-Pattern aus t/10_stream_metadata.t direkt wiederverwendbar

**Research date:** 2026-06-04
**Valid until:** 2026-07-04 (stabiles LMS Plugin API, keine externen Abhaengigkeiten)
