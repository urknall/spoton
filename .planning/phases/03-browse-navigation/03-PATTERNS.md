# Phase 3: Browse + Navigation - Pattern Map

**Mapped:** 2026-05-28
**Files analyzed:** 5 (3 modified, 2 optional new)
**Analogs found:** 5 / 5

---

## File Classification

| Neue/Geänderte Datei | Rolle | Datenfluss | Nächster Analog | Match-Qualität |
|----------------------|-------|-----------|-----------------|----------------|
| `Plugins/SpotOn/Plugin.pm` | controller (Feed-Dispatcher) | request-response + CRUD | `Plugins/SpotOn/Plugin.pm` (bestehendes `_accountSwitcherFeed`) | exact |
| `Plugins/SpotOn/API/Client.pm` | service (HTTP-Egress) | CRUD + request-response | `Plugins/SpotOn/API/Client.pm` (bestehendes `getMe`) | exact |
| `Plugins/SpotOn/API/TokenManager.pm` | service (Auth) | request-response | `Plugins/SpotOn/API/TokenManager.pm` (bestehendes `REQUIRED_SCOPES`) | exact |
| `Plugins/SpotOn/strings.txt` | config (i18n) | — | `Plugins/SpotOn/strings.txt` (bestehende Einträge) | exact |
| `Plugins/SpotOn/Browse/*.pm` (optional) | controller (Sub-Feed-Module) | request-response + CRUD | `Plugins/SpotOn/Plugin.pm` (`_accountSwitcherFeed`, `_switchAccount`) | role-match |

---

## Pattern Assignments

### `Plugins/SpotOn/Plugin.pm` — Erweiterung um Home/Suche/Library

**Analog:** `Plugins/SpotOn/Plugin.pm` (aktueller Stand)

**Imports-Pattern** (Zeilen 1–16):
```perl
package Plugins::SpotOn::Plugin;

use strict;
use warnings;

use base qw(Slim::Plugin::OPMLBased);

use Slim::Utils::Log;
use Slim::Utils::Prefs;
use Slim::Utils::Strings qw(string cstring);
use Slim::Utils::Timers;
use Slim::Utils::Cache;
use Time::HiRes;

my $prefs = preferences('plugin.spoton');
my $cache = Slim::Utils::Cache->new();
```

**Top-Level-Feed-Pattern** (Zeilen 94–133) — Erweiterung von `handleFeed()`:
```perl
sub handleFeed {
    my ($client, $callback, $args) = @_;

    # Binary-Check kommt zuerst (Kurzschluss)
    if ( !Plugins::SpotOn::Helper->get() ) {
        $callback->({ items => [{ name => cstring($client, 'PLUGIN_SPOTON_BINARY_MISSING'), type => 'textarea' }] });
        return;
    }

    my @items;

    # Rate-limit-Hinweis oben (bleibt unverändert)
    if ( $cache->get(Plugins::SpotOn::API::Client->RATE_LIMIT_CACHE_KEY()) ) {
        push @items, { name => cstring($client, 'PLUGIN_SPOTON_RATE_LIMIT_HINT'), type => 'textarea' };
    }

    # Account-Switcher (bleibt unverändert, Zeilen 118–131)
    my $activeName = Plugins::SpotOn::API::TokenManager->getActiveAccountName($client);
    if ($activeName) {
        push @items, {
            name => cstring($client, 'PLUGIN_SPOTON_ACTIVE_ACCOUNT', $activeName),
            url  => \&_accountSwitcherFeed,
            type => 'link',
        };
    } else {
        push @items, { name => cstring($client, 'PLUGIN_SPOTON_ACCOUNT_NONE'), type => 'textarea' };
    }

    # NEU (Phase 3): Home, Suche, Library als weitere Top-Level-Items einfügen
    # push @items, { name => cstring($client, 'PLUGIN_SPOTON_HOME'),    url => \&_homeFeed,    type => 'link' };
    # push @items, { name => cstring($client, 'PLUGIN_SPOTON_SEARCH'),  url => \&_searchFeed,  type => 'link', search => '' };
    # push @items, { name => cstring($client, 'PLUGIN_SPOTON_LIBRARY'), url => \&_libraryFeed, type => 'link' };

    $callback->({ items => \@items });
}
```

**Sub-Feed-Pattern mit passthrough** (Zeilen 137–159) — Vorlage für alle neuen Feed-Handler:
```perl
sub _accountSwitcherFeed {
    my ($client, $callback, $args) = @_;
    # ... baut @items Array ...
    for my $id (sort keys %{$accounts}) {
        push @items, {
            name        => $name . ($isActive ? ' *' : ''),
            url         => \&_switchAccount,
            passthrough => [{ accountId => $id }],   # <-- Parameter-Weitergabe an Sub-Feed
            type        => 'link',
            nextWindow  => 'refreshOrigin',
        };
    }
    $callback->({ items => \@items });
}
```

**Callback-Signatur mit passthrough** (Zeilen 162–178) — Vorlage für Feeds, die Parameter empfangen:
```perl
sub _switchAccount {
    my ($client, $callback, $args, $passthrough) = @_;
    # $passthrough->[0]{accountId} -- Zugriff auf weitergegebene Parameter
    my $accountId = $passthrough && $passthrough->[0] ? $passthrough->[0]{accountId} : undef;
    # ...
    $callback->({
        items      => [{ name => 'OK', type => 'textarea', showBriefly => 1 }],
        nextWindow => 'refreshOrigin',
    });
}
```

**Account-ID aus Client-Kontext ermitteln** (Zeilen 141–143):
```perl
my $activeId = $prefs->client($client)->get('activeAccount')
            || $prefs->get('activeAccount')
            || '';
```

---

### `Plugins/SpotOn/API/Client.pm` — Erweiterung um 11 neue Endpunkt-Methoden

**Analog:** `Plugins/SpotOn/API/Client.pm` (bestehendes `getMe`)

**Imports-Pattern** (Zeilen 1–14):
```perl
package Plugins::SpotOn::API::Client;

use strict;
use warnings;

use JSON::XS::VersionOneAndTwo;
use URI::Escape qw(uri_escape);

use Slim::Networking::SimpleAsyncHTTP;
use Slim::Utils::Cache;
use Slim::Utils::Log;
use Slim::Utils::Prefs;
use Slim::Utils::Timers;
use Time::HiRes;
```

**Endpunkt-Methoden-Pattern** (Zeilen 44–51) — Vorlage für alle neuen Methoden:
```perl
# getMe($class, $accountId, $cb)
# Fetches the current user profile (/me).
# $cb->($result) on success; $cb->(undef, $err) on failure.
sub getMe {
    my ($class, $accountId, $cb) = @_;
    $class->_request('get', 'me', { _accountId => $accountId, _noCache => 1 }, $cb);
}

# Vorlage für neue Methoden mit Offset-Pagination:
sub getSavedTracks {
    my ($class, $accountId, $params, $cb) = @_;
    $class->_request('get', 'me/tracks', {
        _accountId => $accountId,
        offset     => $params->{offset} || 0,
        limit      => $params->{limit}  || 50,
    }, $cb);
}

# Vorlage für neue Methoden mit Pfad-Parametern:
sub getArtist {
    my ($class, $accountId, $artistId, $cb) = @_;
    $class->_request('get', "artists/$artistId", { _accountId => $accountId }, $cb);
}
```

**_request()-Pipeline** (Zeilen 67–157) — NICHT verändern, nur über neue Methoden aufrufen:
```perl
# Pipeline-Schritte (Referenz, nicht kopieren):
# 1. Rate-limit-Flag-Check (Zeile 71–75)
# 2. Cache-Check via _cacheKey (Zeile 78–88)
# 3. Concurrency-Cap: max 3 parallel (Zeile 91–98)
# 4. inflightCount++ (Zeile 101)
# 5. Token-Injection via TokenManager->getToken (Zeile 106)
# 6. URL-Aufbau: API_BASE + "/" + $path + "?" + sortierte Query-Params (Zeile 119–128)
# 7. SimpleAsyncHTTP->get/$method (Zeile 148–156)
```

**Cache-TTL-Zuordnung** (Zeilen 230–250) — muss für neue Pfade erweitert werden:
```perl
sub _cacheTTL {
    my ($class, $path) = @_;
    return 0    if $path =~ /^me\/player/;        # Playback: nie cachen
    return 0    if $path eq 'me';                  # User-Profil: nie cachen
    return 60   if $path =~ /^me\/(?:tracks|albums)/;  # Library: 60s
    return 3600 if $path =~ /^(?:tracks|albums|artists)\//;  # Metadata: 1h
    return 300  if $path =~ /^(?:playlists|browse)\//;       # Playlists: 5min
    return 0;   # Default: kein Cache
}

# Neue Pfade ergänzen (Phase 3):
# me/following        -> Library (60s) -- Zeile 241 Regex erweitern
# me/top              -> Library (60s) -- dto.
# me/player/recently-played -> 0 (nie cachen — Playback-Zustand)
# search              -> 300s (Browse-Daten)
```

**Fehlerbehandlungs-Pattern** (Zeilen 191–224):
```perl
sub _onError {
    my ($class, $http, $error, $path, $params, $cb) = @_;
    $inflightCount--;
    my $code = ($http && $http->response) ? $http->response->code : 0;

    if ($code == 429) {
        # Retry-After cappen bei 300s (T-02-08)
        my $retryAfter = RATE_LIMIT_DEFAULT_BACKOFF;
        if ($http && $http->response) {
            my $headerVal = $http->response->header('Retry-After');
            $retryAfter = $headerVal if defined $headerVal && $headerVal =~ /^\d+$/;
        }
        $retryAfter = 300 if $retryAfter > 300;
        $cache->set(RATE_LIMIT_CACHE_KEY, 1, $retryAfter);
        $cb->(undef, { error => 'rate_limited', code => 429 });
        return;
    }

    if ($code == 401) {
        my $accountId = $params->{_accountId} // '';
        $cache->remove("spoton_token_$accountId") if $accountId;
        $cb->(undef, { error => 'unauthorized', code => 401 });
        return;
    }

    $log->error("Client: HTTP $code error for $path: $error");
    $cb->(undef, { error => $error, code => $code });
}
```

---

### `Plugins/SpotOn/API/TokenManager.pm` — Scope-Erweiterung

**Analog:** `Plugins/SpotOn/API/TokenManager.pm` (bestehendes `REQUIRED_SCOPES`)

**Scope-Konstante** (Zeilen 26–36) — muss um zwei Scopes erweitert werden:
```perl
use constant REQUIRED_SCOPES => join(' ', qw(
    user-read-playback-state
    user-modify-playback-state
    user-read-currently-playing
    user-read-recently-played
    user-read-private
    playlist-read-private
    playlist-read-collaborative    # NEU Phase 3 — für kollaborative Playlists
    user-library-read
    user-top-read
    user-follow-read               # NEU Phase 3 — für GET /me/following?type=artist (NAV-03)
    streaming
));
```

**Achtung:** Scope-Änderung invalidiert bestehende Tokens. Plan muss einen Token-Cache-Clear für alle Accounts beim Plugin-Start nach der Scope-Erweiterung einbauen.

---

### `Plugins/SpotOn/strings.txt` — i18n-Erweiterung

**Analog:** `Plugins/SpotOn/strings.txt` (bestehende Einträge)

**Eintrags-Pattern** (Zeilen 1–9) — immer Tab-Einrückung, immer EN+DE:
```
PLUGIN_SPOTON_HOME
	DE	Home
	EN	Home

PLUGIN_SPOTON_SEARCH
	DE	Suche
	EN	Search

PLUGIN_SPOTON_LIBRARY
	DE	Bibliothek
	EN	Library
```

**Neue Strings (vollständige Liste für Phase 3):**
```
PLUGIN_SPOTON_HOME               Home / Home
PLUGIN_SPOTON_SEARCH             Suche / Search
PLUGIN_SPOTON_LIBRARY            Bibliothek / Library
PLUGIN_SPOTON_RECENTLY_PLAYED    Kürzlich gehört / Recently Played
PLUGIN_SPOTON_MADE_FOR_YOU       Für dich gemacht / Made For You
PLUGIN_SPOTON_TOP_TRACKS         Top Tracks / Top Tracks
PLUGIN_SPOTON_LIKED_SONGS        Gefällt mir / Liked Songs
PLUGIN_SPOTON_ALBUMS             Alben / Albums
PLUGIN_SPOTON_ARTISTS            Künstler / Artists
PLUGIN_SPOTON_PLAYLISTS          Playlists / Playlists
PLUGIN_SPOTON_SINGLES            Singles / Singles
PLUGIN_SPOTON_COMPILATIONS       Compilations / Compilations
PLUGIN_SPOTON_APPEARS_ON         Erscheint auf / Appears On
PLUGIN_SPOTON_TOP_RESULT         Bestes Ergebnis / Top Result
PLUGIN_SPOTON_TRACKS             Tracks / Tracks
PLUGIN_SPOTON_NO_RESULTS         Keine Ergebnisse / No results
```

---

### `Plugins/SpotOn/Browse/*.pm` (optional — Artist.pm, Album.pm, Playlist.pm)

**Analog:** `Plugins/SpotOn/Plugin.pm` (Sub-Feed-Funktionen `_accountSwitcherFeed`, `_switchAccount`)

**Package-Header-Pattern** — falls Browse-Module erzeugt werden:
```perl
package Plugins::SpotOn::Browse::Artist;

use strict;
use warnings;

use Slim::Utils::Log;
use Slim::Utils::Prefs;
use Slim::Utils::Strings qw(cstring);

my $log   = logger('plugin.spoton');
my $prefs = preferences('plugin.spoton');

sub feed {
    my ($client, $callback, $args, $passthrough) = @_;
    my $artistId  = $passthrough->[0]{artistId};
    my $accountId = $prefs->client($client)->get('activeAccount')
                 || $prefs->get('activeAccount')
                 || '';
    # ... API-Call + @items aufbauen ...
    $callback->({ items => \@items });
}

1;
```

**Hinweis zur Modulstruktur:** Das Planner-Team entscheidet, ob Browse-Module erzeugt werden oder alles in Plugin.pm verbleibt. Spotty-Plugin-Referenz: alles in OPML.pm (~1200 Zeilen). Empfehlung laut RESEARCH.md: Separation ab ~600 Zeilen Plugin.pm.

---

## Shared Patterns

### LMS-Pagination-Mapping (NAV-11, NAV-12)

**Quelle:** Kombiniert aus XMLBrowser.pm-Analyse (RESEARCH.md Pattern 1) + bestehender Client.pm-Logik
**Anwenden auf:** Alle Listen-Feed-Callbacks mit Offset-Pagination (Liked Songs, Alben, Playlist-Items, Albumtracks)

```perl
# $args kommt von LMS XMLBrowser (Zeilen 194–198 aus RESEARCH.md Pattern 1)
sub _savedTracksFeed {
    my ($client, $callback, $args) = @_;
    my $offset = $args->{index}    || 0;
    my $qty    = $args->{quantity} || 200;
    my $limit  = $qty > 50 ? 50 : $qty;   # Spotify Library-Endpunkte: max 50/Seite
                                            # Suche: max 10 (Dev Mode)

    my $accountId = $prefs->client($client)->get('activeAccount')
                 || $prefs->get('activeAccount') || '';

    Plugins::SpotOn::API::Client->getSavedTracks($accountId, {
        offset => $offset,
        limit  => $limit,
    }, sub {
        my $data = shift;
        my @items;
        # ... Items aufbauen ...
        $callback->({ items => \@items, total => $data->{total} });
        # WICHTIG: total mitsenden -- LMS OPMLBased braucht es für korrekte Pagination
    });
}
```

### Cursor-Paginierung (Recently Played, Followed Artists)

**Quelle:** RESEARCH.md Pattern 2 + Spotify API Docs
**Anwenden auf:** `_recentlyPlayedFeed()`, `_followedArtistsFeed()`

```perl
# Cursor-APIs: KEIN offset-Parameter — nur limit (und ggf. after-Cursor)
sub _recentlyPlayedFeed {
    my ($client, $callback, $args) = @_;
    # LMS-index wird ignoriert — Einmalabfrage mit limit=50
    Plugins::SpotOn::API::Client->getRecentlyPlayed($accountId, { limit => 50 }, sub {
        my $data = shift;
        my @items = map { _trackItem($client, $_->{track}) } @{ $data->{items} || [] };
        $callback->({ items => \@items });
        # KEIN total — LMS zeigt alle Items ohne Paginierung
    });
}
```

### OPML Track-Item-Struktur (NAV-06, D-07)

**Quelle:** RESEARCH.md Pattern 3 (verifiziert gegen Spotty-Plugin OPML.pm + XMLBrowser.pm)
**Anwenden auf:** Alle Track-Listen (Recently Played, Top Tracks, Liked Songs, Albumtracks, Playlisttracks)

```perl
sub _trackItem {
    my ($client, $track) = @_;
    my $title    = $track->{name};
    my $artist   = join(', ', map { $_->{name} } @{ $track->{artists} || [] });
    my $album    = $track->{album}{name} // '';
    my $image    = _largestImage($track->{album}{images});
    my $duration = ($track->{duration_ms} || 0) / 1000;

    return {
        name      => "$title \x{2014} $artist",   # Fallback für ältere LMS-Clients
        line1     => $title,
        line2     => $artist . ($album ? " \x{2022} $album" : ''),
        url       => 'spotify://' . $track->{uri},
        play      => 'spotify://' . $track->{uri},
        on_select => 'play',
        image     => $image,
        duration  => $duration,
        type      => 'audio',
    };
}
```

### OPML Link-Item-Struktur (Alben, Künstler, Playlists)

**Quelle:** Plugin.pm `_accountSwitcherFeed()` Zeilen 148–156 + RESEARCH.md Code Examples
**Anwenden auf:** Alle navigierbaren Container-Items

```perl
# Album-Item (navigiert zu Album-Detailseite):
push @items, {
    name        => $album->{name},
    url         => \&_albumFeed,
    passthrough => [{ albumId => $album->{id} }],
    image       => _largestImage($album->{images}),
    type        => 'link',
    line2       => $album->{artists}[0]{name} . ' (' . ($album->{release_date} // '') . ')',
};

# Artist-Item:
push @items, {
    name        => $artist->{name},
    url         => \&_artistFeed,
    passthrough => [{ artistId => $artist->{id} }],
    image       => _largestImage($artist->{images}),
    type        => 'link',
};
```

### Artwork-Auswahl

**Quelle:** RESEARCH.md Pattern 3 (`_largestImage`) + Anti-Patterns (Pitfall 6)
**Anwenden auf:** ALLE Items mit Spotify-`images`-Array (Tracks, Alben, Playlists, Künstler)

```perl
sub _largestImage {
    my ($images) = @_;
    return '' unless ref $images eq 'ARRAY' && @{$images};
    my ($largest) = sort { ($b->{width} || 0) <=> ($a->{width} || 0) } @{$images};
    return $largest->{url} || '';
}
```

### Made-For-You-Filter (D-04)

**Quelle:** RESEARCH.md Pattern 5
**Anwenden auf:** `/me/playlists`-Response in `_homeFeed()` und `_libraryFeed()`

```perl
sub _isMadeForYou {
    my ($playlist) = @_;
    return ($playlist->{owner}{id} // '') eq 'spotify';
}

# Home: nur Made-For-You-Playlists (Discover Weekly, Daily Mix, etc.)
my @mfy  = grep {  _isMadeForYou($_) } @{$playlists};
# Library: alle Playlists AUSSER Made-For-You
my @user = grep { !_isMadeForYou($_) } @{$playlists};
```

### Graceful Hide für entfernte Endpoints (NAV-10, D-11)

**Quelle:** RESEARCH.md Code Examples + D-11
**Anwenden auf:** Artist Top Tracks, Related Artists, Browse Categories, New Releases

```perl
# KEIN Fehler, KEIN Platzhalter-Item — Menüpunkt einfach nicht erzeugen.
# Betroffene Endpoints: GET /artists/{id}/top-tracks, /artists/{id}/related-artists,
# /browse/categories, /browse/new-releases, GET /tracks (batch)
# Diese Methoden werden in Client.pm überhaupt nicht implementiert.
```

### Logging-Pattern

**Quelle:** Client.pm Zeilen 146, 169, 209, 223
**Anwenden auf:** Alle neuen Feed-Handler und Client-Methoden

```perl
my $log = logger('plugin.spoton');

# Infolog nur wenn INFOLOG-Flag gesetzt (Performance):
main::INFOLOG && $log->info("Client: $method $path");

# Fehler immer loggen:
$log->error("Client: HTTP $code error for $path: $error");

# KRITISCH: Niemals Token-Werte loggen (T-02-10):
# FALSCH: $log->info("Token: $token");
# RICHTIG: $log->info("Token refreshed for account $accountId");
```

---

## Kein Analog gefunden

Alle Phase-3-Dateien haben starke Analogs in der bestehenden Codebase. Kein Datei ohne Vorlage.

| Datei | Rolle | Datenfluss | Hinweis |
|-------|-------|-----------|---------|
| (keine) | — | — | Alle Muster bereits in Phase 2 etabliert |

---

## Kritische Implementierungshinweise (aus RESEARCH.md Pitfalls)

| Pitfall | Betroffene Datei | Vermeidungsmuster |
|---------|-----------------|-------------------|
| Kombinierter `include_groups`-Request bricht Paginierung | Client.pm `getArtistAlbums()` | Vier separate `_request()`-Aufrufe pro `include_groups`-Wert |
| Fehlender `user-follow-read` Scope | TokenManager.pm | Scope in `REQUIRED_SCOPES` ergänzen + Token-Cache-Clear beim Start |
| Alter Playlist-Items-Pfad (`/tracks` statt `/items`) | Client.pm `getPlaylistItems()` | Immer `playlists/{id}/items` verwenden |
| Cursor vs. Offset: Recently Played + Followed Artists | Plugin.pm Feed-Callbacks | Kein `offset` — nur `limit` (und ggf. `after`-Cursor) an diese Endpoints |
| Kleinstes Artwork aus `images`-Array | Plugin.pm Item-Builder | Immer `_largestImage()` nutzen |
| LMS-Pagination auf Cursor-Endpoints anwenden | Plugin.pm `_recentlyPlayedFeed()` | `$args->{index}` ignorieren; Einmalabfrage mit `limit=50` |

---

## Metadata

**Analog-Suchbereich:** `/home/sti/spoton/Plugins/SpotOn/`
**Gescannte Dateien:** 7 (Plugin.pm, API/Client.pm, API/TokenManager.pm, ProtocolHandler.pm, Settings.pm, Settings/Callback.pm, strings.txt)
**Pattern-Extraction-Datum:** 2026-05-28
