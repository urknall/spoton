# Phase 3: Browse + Navigation - Research

**Researched:** 2026-05-28
**Domain:** LMS OPMLBased Plugin / Spotify Web API Browse + Navigation
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Home, Suche, Library als drei Top-Level-Einträge nach dem Account-Switcher. Keine flache Struktur, kein Spotty-Klon.
- **D-02:** Home-Feed: Kürzlich gehört (`/me/player/recently-played`), Made For You (aus `/me/playlists` gefiltert), Top Tracks (`/me/top/tracks`). Jeder Eintrag öffnet eigene Liste.
- **D-03:** Library: Liked Songs (`/me/tracks`), Alben (`/me/albums`), Künstler (`/me/following?type=artist`), Playlists (`/me/playlists` ohne Made-For-You).
- **D-04:** Made-For-You-Erkennung via generisches Merkmal (kein Namens-Pattern). Research klärt zuverlässigste Methode.
- **D-05:** `time_range=medium_term` (6 Monate) fest für Home Top Tracks — kein Umschalter.
- **D-06:** Track-Tap setzt `spotify://{uri}` Play-Intent. Playback scheitert in Phase 3 (kein Transcoder), Infrastruktur bereit für Phase 4.
- **D-07:** Tracks bekommen Kontextnavigation: "Artist anzeigen" und "Album anzeigen" als Kontextmenü-Einträge.
- **D-08:** Kein "Alle abspielen" auf Album- oder Playlist-Ebene. Nur Einzeltrack-Auswahl.
- **D-09:** Artist-Detailseite: vier Sektionen — Alben, Singles, Compilations, Erscheint auf (via `GET /artists/{id}/albums` mit separaten Requests pro `include_groups` — kein kombinierter Request, wegen API-Paginierungsbug).
- **D-10:** Suche: Top-Ergebnis prominent oben (bestes erstes Ergebnis aus relevantestem Typ), dann Sub-Menüs pro Typ. Kategorien mit 0 Ergebnissen ausgeblendet.
- **D-11:** Dev-Mode-entfernte Endpoints (Artist Top Tracks, Related Artists, Browse Categories, New Releases) werden stillschweigend ausgelassen.
- **D-12:** LMS-internes OPMLBased-Pagination-Framework (index/quantity). Spotify API offset/limit wird gemappt.
- **D-13:** Album-Cover, Playlist-Bilder und Artist-Fotos als `image`-Feld in OPML-Items (raw Spotify CDN URLs).

### Claude's Discretion

- Suchergebnisse pro Kategorie: Research entscheidet (5 vs. 10 bei Dev Mode limit=10)
- Library-Sortierung: Research prüft ob Umschaltung sinnvoll oder "recently added" als feste Sortierung ausreicht
- Track-Metadaten-Format: Research prüft OPML line2/subtext Support und orientiert sich an Spotty/Qobuz-Praxis
- Spotify-API → LMS-Pagination-Mapping: Research klärt technische Details

### Deferred Ideas (OUT OF SCOPE)

Keine — Diskussion blieb im Phase-3-Scope.

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| NAV-01 | Top-level menu structure: Home, Search, Library | D-01: Items nach Account-Switcher in handleFeed(); Callback-Pattern aus OPML.pm bekannt |
| NAV-02 | Home feed: Recently Played, Made For You mixes, Top Tracks | D-02/D-04: `/me/player/recently-played`, `/me/playlists` mit owner.id-Filter, `/me/top/tracks?time_range=medium_term` |
| NAV-03 | Library: Liked Songs, Saved Albums, Followed Artists, User Playlists | D-03: Vier Endpunkte; `user-follow-read` Scope fehlt noch in TokenManager.pm |
| NAV-04 | Search: free-text, results categorized (Tracks, Albums, Artists, Playlists) | D-10: `/search?type=track,album,artist,playlist&limit=10`; Dev Mode limit=10 |
| NAV-05 | Artist detail page: Discography (Albums, Singles, Compilations) | D-09: Separate Requests pro include_groups wegen Paginierungsbug |
| NAV-06 | Album detail page: tracklist with track number, duration, featuring artists | `GET /albums/{id}/tracks`, OPML line1/line2-Felder |
| NAV-07 | Playlist detail page: paginated tracks, description, creator | `GET /playlists/{id}/items` (neue Pfadform seit Feb 2026) |
| NAV-08 | Liked Songs unconditional (no custom Client ID gating) | `/me/tracks` scope bereits in REQUIRED_SCOPES |
| NAV-09 | Library items sortable, recently added as default | API liefert `added_at`-Feld; Spotify-API sortiert standardmäßig nach added_at desc |
| NAV-10 | Endpoints removed in Dev Mode gracefully hidden, not errored | Client-Methoden nicht aufrufen; betroffene Menüpunkte einfach weglassen |
| NAV-11 | Search pagination handles limit=10 per request (Dev Mode) | `offset`-Parameter für Folgeseiten; LMS index/quantity Mapping |

</phase_requirements>

---

## Summary

Phase 3 baut die vollständige Inhaltsnavigation auf dem in Phase 2 gelegten Fundament. Der Code-Kernbefund: `Plugin.pm::handleFeed()` und `API/Client.pm::_request()` sind fertig — Phase 3 ergänzt Methoden in Client.pm, baut Feed-Handler in Plugin.pm aus und verdrahtet alles mit LMS OPMLBased-Pagination.

**Wichtigste Erkenntnisse:** (1) Die `GET /artists/{id}/albums`-API hat einen dokumentierten Paginierungsbug bei kombinierten `include_groups` — Lösung ist separate Requests pro Typ. (2) `user-follow-read` Scope fehlt in TokenManager.pm und muss ergänzt werden. (3) Playlist-Tracks-Endpunkt heißt seit Feb 2026 `/playlists/{id}/items`, nicht mehr `/tracks`. (4) Made-For-You-Erkennung via `owner.id eq 'spotify'` ist die zuverlässigste Methode; trifft sowohl algorithmische (Daily Mix, Discover Weekly) als auch editorielle Spotify-Playlists.

**Primary recommendation:** Alle API-Methoden in `Client.pm` ergänzen, dann Feed-Handler schichtweise aufbauen: Top-Level → Home-Feed → Library-Feed → Search-Feed → Detail-Seiten (Artist/Album/Playlist). Separate `Browse/` Module für Übersichtlichkeit erwägen.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| OPML-Menü-Rendering | LMS OPMLBased | — | LMS übernimmt Rendering vollständig; Plugin liefert nur items-Array |
| Spotify API-Calls | API/Client.pm | — | Zentrales HTTP-Egress-Point (API-01), kein direktes HTTP in Plugin.pm |
| Pagination-Mapping | Plugin.pm Feed-Callbacks | API/Client.pm | LMS liefert index/quantity; Plugin übersetzt zu Spotify offset/limit |
| Token-Verwaltung | API/TokenManager.pm | — | Bereits implementiert; Phase 3 nur Scope-Erweiterung |
| Artwork-URLs | Spotify CDN (direkt) | — | Raw URLs aus API-Response direkt als OPML `image`-Feld |
| Made-For-You-Filter | Plugin.pm | — | owner.id-Check bei Playlist-Iteration, kein API-Support nötig |
| Graceful-Hide (NAV-10) | Plugin.pm | — | Bedingte Menüpunkt-Erzeugung, kein Fehler-Handling in Client.pm |

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `Slim::Plugin::OPMLBased` | LMS 8.0+ | Menü-Framework (handleFeed, Pagination) | Das IS das Framework — kein Ersatz |
| `Slim::Networking::SimpleAsyncHTTP` | LMS 8.0+ | Alle API-Calls non-blocking | LMS ist single-threaded; blockierendes LWP ist verboten |
| `Slim::Utils::Cache` | LMS 8.0+ | Response-Caching mit TTL | Bereits in Client.pm implementiert mit domain-TTLs |
| `Slim::Control::XMLBrowser` | LMS 8.0+ | Verarbeitet OPMLBased-Feed-Callbacks | Intern durch OPMLBased genutzt; liefert index/quantity |
| `JSON::XS::VersionOneAndTwo` | LMS-Bundle | JSON encode/decode | Bereits verwendet in Client.pm |
| `URI::Escape` | LMS-Bundle | URL-Encoding für Query-Parameter | Bereits verwendet in Client.pm |

### API-Endpunkte (Phase 3)

| Methode | Endpunkt | Scope | Notes |
|---------|----------|-------|-------|
| `search()` | `GET /search` | — | `type=track,album,artist,playlist`, `limit=10` (Dev Mode Max) |
| `getRecentlyPlayed()` | `GET /me/player/recently-played` | `user-read-recently-played` | Cursor-Paginierung, max 50 |
| `getTopTracks()` | `GET /me/top/tracks` | `user-top-read` | `time_range=medium_term` |
| `getSavedTracks()` | `GET /me/tracks` | `user-library-read` | Offset-Paginierung, max 50/Seite |
| `getSavedAlbums()` | `GET /me/albums` | `user-library-read` | Offset-Paginierung, max 50/Seite |
| `getFollowedArtists()` | `GET /me/following?type=artist` | `user-follow-read` (**FEHLT in TokenManager!**) | Cursor-Paginierung, max 50 |
| `getUserPlaylists()` | `GET /me/playlists` | `playlist-read-private` | Offset-Paginierung |
| `getArtist()` | `GET /artists/{id}` | — | Einzelabruf |
| `getArtistAlbums()` | `GET /artists/{id}/albums` | — | Separate Requests pro include_groups (Paginierungsbug) |
| `getAlbum()` | `GET /albums/{id}` | — | Einzelabruf |
| `getAlbumTracks()` | `GET /albums/{id}/tracks` | — | Offset-Paginierung |
| `getPlaylistItems()` | `GET /playlists/{id}/items` | `playlist-read-private` | Umbenennung von `/tracks` seit Feb 2026 |

**Keine externen CPAN-Abhängigkeiten.** Alle benötigten Module sind in LMS gebündelt.

---

## Package Legitimacy Audit

> Nicht anwendbar — Phase 3 installiert keine externen Pakete. Alle Module sind LMS-interne Bundle-Module.

---

## Architecture Patterns

### System Architecture Diagram

```
User-Aktion (LMS-Remote/Jive/Web-UI)
         |
         v
Slim::Control::XMLBrowser  ← OPMLBased-Dispatcher
         |
         | ($client, $callback, $args{index, quantity, search}, @passthrough)
         v
Plugin.pm::handleFeed()           -- Top-Level: [Account-Switcher | Home | Suche | Library]
  |            |            |
  v            v            v
_homeFeed()  _searchFeed()  _libraryFeed()
  |            |              |
  |            |    [Liked Songs | Alben | Künstler | Playlists]
  |    _searchResults($type)
  |            |
  v            v
API/Client.pm::_request()   ← Zentrales Egress-Point
         |
         | SimpleAsyncHTTP + Token-Injection
         v
  Spotify Web API v1
         |
         v (JSON response)
API/Client.pm::_onSuccess()  → Cache (TTL) → $callback->(items)
         |
         v
Feed-Callback baut @items-Array
         |
         v
$callback->({ items => \@items })
         |
         v
LMS rendert OPML-Menü
```

### Recommended Project Structure

```
Plugins/SpotOn/
├── Plugin.pm              # handleFeed() + Top-Level + _homeFeed/_libraryFeed/_searchFeed
├── API/
│   ├── Client.pm          # +11 neue Endpunkt-Methoden
│   └── TokenManager.pm    # +user-follow-read Scope
├── Browse/
│   ├── Artist.pm          # _artistFeed(), _artistAlbumsFeed() (optional Separation)
│   ├── Album.pm           # _albumFeed(), _albumTracksFeed()
│   └── Playlist.pm        # _playlistFeed(), _playlistItemsFeed()
└── strings.txt            # +~20 neue i18n-Strings
```

> **Hinweis:** Die Browse/-Separation ist optional — alles kann auch in Plugin.pm verbleiben (Spotty-Pattern). Empfehlung: Browse/-Untermodule für Lesbarkeit ab ~600 Zeilen Plugin.pm.

### Pattern 1: Feed-Callback mit LMS-Pagination-Mapping

**What:** LMS liefert `$args->{index}` und `$args->{quantity}`. Diese werden auf Spotify `offset`/`limit` gemappt.
**When to use:** Alle Listen-Endpunkte mit Offset-Pagination (Tracks, Alben, Künstler, Playlists).

```perl
# Source: XMLBrowser.pm + Spotty-Plugin OPML.pm (verified)
sub _savedTracksFeed {
    my ($client, $callback, $args) = @_;

    my $index    = $args->{index}    || 0;
    my $quantity = $args->{quantity} || 200;

    # Spotify API nutzt 0-basierte Offsets, LMS $args->{index} ist ebenfalls 0-basiert
    my $offset = $index;
    my $limit  = ($quantity > 50) ? 50 : $quantity;  # Spotify max = 50

    my $accountId = _getAccountId($client);

    Plugins::SpotOn::API::Client->getSavedTracks($accountId, {
        offset => $offset,
        limit  => $limit,
    }, sub {
        my $data = shift;
        my @items;
        for my $entry (@{ $data->{items} || [] }) {
            my $track = $entry->{track};
            push @items, _trackItem($client, $track);
        }
        $callback->({ items => \@items, total => $data->{total} });
    });
}
```

### Pattern 2: Cursor-Paginierung (Recently Played, Followed Artists)

**What:** Einige Endpunkte verwenden Cursor- statt Offset-Paginierung.
**When to use:** `GET /me/player/recently-played`, `GET /me/following?type=artist`

```perl
# Recently Played: cursor-basiert, max 50 Tracks total
# LMS-Pagination nicht anwendbar — Einmalabfrage, kein Offset
sub _recentlyPlayedFeed {
    my ($client, $callback, $args) = @_;

    Plugins::SpotOn::API::Client->getRecentlyPlayed(_getAccountId($client), {
        limit => 50,
    }, sub {
        my $data = shift;
        my @items = map { _trackItem($client, $_->{track}) }
                    @{ $data->{items} || [] };
        $callback->({ items => \@items });
    });
}
```

### Pattern 3: OPML Track-Item-Struktur

**What:** Wie ein Track-Item korrekt gebaut wird (line1, line2, play, on_select, image, duration).
**When to use:** Alle Track-Listen (Recently Played, Top Tracks, Liked Songs, Albumtracks, Playlisttracks).

```perl
# Source: Spotty-Plugin OPML.pm + XMLBrowser.pm item field analysis [VERIFIED via code review]
sub _trackItem {
    my ($client, $track) = @_;

    my $title    = $track->{name};
    my $artist   = join(', ', map { $_->{name} } @{ $track->{artists} || [] });
    my $album    = $track->{album}{name} // '';
    my $image    = _largestImage($track->{album}{images});
    my $duration = ($track->{duration_ms} || 0) / 1000;

    return {
        name      => "$title \x{2014} $artist",   # Fallback-Text (ältere Clients)
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

# Größtes verfügbares Bild aus images-Array wählen
sub _largestImage {
    my ($images) = @_;
    return '' unless ref $images eq 'ARRAY' && @{$images};
    my ($largest) = sort { ($b->{width} || 0) <=> ($a->{width} || 0) } @{$images};
    return $largest->{url} || '';
}
```

### Pattern 4: Artist-Alben mit separaten Requests (Paginierungsbug-Workaround)

**What:** `GET /artists/{id}/albums` mit kombiniertem `include_groups` hat einen API-Paginierungsbug — separate Requests pro Typ nötig.
**When to use:** Artist-Detailseite, vier Sektionen.

```perl
# Pitfall: Kombinierter Request (include_groups=album,single) bricht Paginierung
# Lösung: Separate Requests, clientseitig zusammenführen
# Source: Spotify Community Bug Report [CITED: community.spotify.com/t5/.../td-p/6113264]
sub _artistFeed {
    my ($client, $callback, $args, $passthrough) = @_;
    my $artistId  = $passthrough->[0]{artistId};
    my $accountId = _getAccountId($client);

    my @sections = (
        { group => 'album',      labelKey => 'PLUGIN_SPOTON_ALBUMS' },
        { group => 'single',     labelKey => 'PLUGIN_SPOTON_SINGLES' },
        { group => 'compilation',labelKey => 'PLUGIN_SPOTON_COMPILATIONS' },
        { group => 'appears_on', labelKey => 'PLUGIN_SPOTON_APPEARS_ON' },
    );

    # Sequentielle Requests (je eine HTTP-Anfrage pro Gruppe)
    # Ergebnisse in Sektions-Items zusammenführen
    # ... (vollständige Implementierung in Plan)
}
```

### Pattern 5: Made-For-You-Erkennung

**What:** Spotify-generierte Playlists (Daily Mix, Discover Weekly, Release Radar) haben `owner.id eq 'spotify'` — zuverlässigster programmatischer Filter.
**When to use:** `/me/playlists`-Response beim Aufbau von Home-Feed (Made For You) und Library-Feed (User Playlists ohne Spotify-Playlists).

```perl
# Source: Spotty-Plugin code analysis + Spotify Community docs [CITED]
# owner.id == 'spotify' für algorithmische (Daily Mix, Discover Weekly) UND
# editorielle (RapCaviar, Today's Top Hits) Spotify-Playlists
sub _isMadeForYou {
    my ($playlist) = @_;
    return ($playlist->{owner}{id} // '') eq 'spotify';
}

# Home: nur Made-For-You
# Library-Playlists: alles AUSSER Made-For-You
my @mfy     = grep { _isMadeForYou($_) } @{$playlists};
my @user    = grep { !_isMadeForYou($_) } @{$playlists};
```

### Pattern 6: Search-Feed mit Sub-Kategorien

**What:** Suche liefert alle Typen in einem Request; UI trennt in Sektionen mit 0-Ergebnis-Ausblendung.
**When to use:** NAV-04, NAV-11.

```perl
# Dev Mode: limit=10 ist Maximum pro Typ
# Top-Ergebnis: erster Track des ersten Typs mit Ergebnissen (kein dediziertes API-Feld)
# Source: XMLBrowser.pm pattern + Spotty-Plugin OPML.pm analysis [VERIFIED]
sub _searchFeed {
    my ($client, $callback, $args) = @_;
    my $query = $args->{search} || '';

    Plugins::SpotOn::API::Client->search(_getAccountId($client), {
        q     => $query,
        type  => 'track,album,artist,playlist',
        limit => 10,
    }, sub {
        my $data = shift;
        my @items;

        # Top-Ergebnis (erster Track) prominent oben
        if (my $topTrack = $data->{tracks}{items}[0]) {
            push @items, {
                name  => cstring($client, 'PLUGIN_SPOTON_TOP_RESULT'),
                type  => 'outline',
                items => [ _trackItem($client, $topTrack) ],
            };
        }

        # Sub-Menüs pro Typ (nur wenn Ergebnisse vorhanden)
        my @tracks  = @{ $data->{tracks}{items}  || [] };
        my @albums  = @{ $data->{albums}{items}  || [] };
        my @artists = @{ $data->{artists}{items} || [] };
        my @plists  = @{ $data->{playlists}{items} || [] };

        push @items, _searchSection($client, 'PLUGIN_SPOTON_TRACKS',    \@tracks,  \&_trackItem)
            if @tracks;
        push @items, _searchSection($client, 'PLUGIN_SPOTON_ALBUMS',    \@albums,  \&_albumItem)
            if @albums;
        push @items, _searchSection($client, 'PLUGIN_SPOTON_ARTISTS',   \@artists, \&_artistItem)
            if @artists;
        push @items, _searchSection($client, 'PLUGIN_SPOTON_PLAYLISTS', \@plists,  \&_playlistItem)
            if @plists;

        $callback->({ items => \@items });
    });
}
```

### Anti-Patterns to Avoid

- **Kombinierter include_groups-Request für Artist-Alben:** Kaputte Paginierung im Spotify API. Immer separate Requests pro Gruppe.
- **Namens-basiertes Made-For-You-Matching:** `if ($name =~ /Daily Mix/i)` ist sprachabhängig (z.B. "Mix der Woche" auf Deutsch) und fragil. `owner.id eq 'spotify'` verwenden.
- **Blockierendes LWP in Feed-Callbacks:** LMS ist single-threaded. Immer SimpleAsyncHTTP nutzen.
- **Manuelles "Mehr laden"-Item:** LMS OPMLBased hat eingebautes Pagination-Framework. Niemals manuell implementieren.
- **Verwenden von `/playlists/{id}/tracks`-Pfad:** Seit Feb 2026 veraltet — `/playlists/{id}/items` verwenden.
- **Batch-Endpunkte verwenden:** `GET /tracks` (mehrere IDs) im Dev Mode entfernt. Einzelabrufe oder gecachte Metadata.

---

## Don't Hand-Roll

| Problem | Nicht selbst bauen | Stattdessen | Warum |
|---------|-------------------|-------------|-------|
| Menü-Paginierung | "Mehr laden"-Item | LMS OPMLBased index/quantity | Eingebaut, automatisch von LMS-Clients unterstützt |
| Artwork-Resize | Eigene Resize-Logik | Raw Spotify CDN URLs (größtes Image wählen) | LMS-Clients cachen selbst; Spotty/Qobuz machen es gleich |
| JSON-Parsing | Eigener Parser | `JSON::XS::VersionOneAndTwo` | Bereits genutzt in Client.pm |
| URL-Encoding | Eigene Escape-Logik | `URI::Escape::uri_escape` | Bereits genutzt in Client.pm |
| Alphabetische Sortierung | Eigener Sort | API `added_at`-Feld (Spotify liefert desc by default) | Für Phase 3: `added_at`-Sort reicht aus (NAV-09) |
| Token-Verwaltung | Neuer Flow | Vorhandener TokenManager | Bereits vollständig implementiert |

---

## Common Pitfalls

### Pitfall 1: GET /artists/{id}/albums kombinierter include_groups bricht Paginierung

**What goes wrong:** Mit `include_groups=album,single` als kombiniertem Parameter liefert die API falsche `total`-Werte und das `next`-Feld zeigt auf weitere Seiten, die leer sind.
**Why it happens:** Dokumentierter Spotify API Bug — `total` bezieht sich auf alle Alben vor dem Filter, Paginierung springt über gefilterte Ergebnisse hinweg.
**How to avoid:** Vier separate Requests: `include_groups=album`, `include_groups=single`, `include_groups=compilation`, `include_groups=appears_on`. Ergebnisse clientseitig als Sektionen anzeigen.
**Warning signs:** Leere Albumlisten auf Seite 2+ bei Künstlern mit vielen Veröffentlichungen.

### Pitfall 2: Fehlender `user-follow-read` Scope

**What goes wrong:** `GET /me/following?type=artist` liefert HTTP 403 (Insufficient scope).
**Why it happens:** TokenManager.pm::REQUIRED_SCOPES enthält den Scope nicht (aktuelle Codeanalyse bestätigt).
**How to avoid:** `user-follow-read` zu REQUIRED_SCOPES in TokenManager.pm ergänzen. **Achtung:** Bereits authentifizierte User müssen sich re-authentifizieren (neuer Scope triggert PKCE-Flow neu). Dies ist ein Breaking Change für bestehende Sessions.
**Warning signs:** Library → Künstler zeigt keine Ergebnisse oder 403-Fehler.

### Pitfall 3: Playlist-Items-Pfad (Feb 2026 Umbenennung)

**What goes wrong:** `GET /playlists/{id}/tracks` funktioniert noch (deprecated), wird aber entfernt.
**Why it happens:** Spotify hat `/tracks` zu `/items` umbenannt. Alter Pfad noch aktiv aber deprecated.
**How to avoid:** Immer `GET /playlists/{id}/items` verwenden, niemals `/tracks`.
**Warning signs:** Deprecation-Warning in Response-Headern.

### Pitfall 4: Recently Played und Followed Artists — Cursor vs. Offset

**What goes wrong:** `offset`-Parameter auf `/me/player/recently-played` oder `/me/following` führt zu Fehlern; diese Endpunkte nutzen Cursor-Paginierung (`after`).
**Why it happens:** Zwei verschiedene Paginierungsmodelle in der Spotify API.
**How to avoid:** `recently-played` und `following` bekommen keine `offset`-Parameter — nur `limit` und optional `after`-Cursor.
**Warning signs:** HTTP 400 "Invalid parameter" auf Folgeseiten.

### Pitfall 5: Scope-Anforderung für Playlist-Inhalte (Feb 2026)

**What goes wrong:** `GET /playlists/{id}/items` liefert 403 für Playlists, die der User nicht besitzt oder an denen er nicht mitarbeitet.
**Why it happens:** Feb 2026: "Playlist contents are only returned for playlists the user owns or collaborates on."
**How to avoid:** Für Phase 3: User-eigene Playlists (aus `/me/playlists`) und Made-For-You-Playlists (Spotify-owned) sind betroffen. User-eigene Playlists funktionieren mit `playlist-read-private` Scope. Made-For-You-Playlists zeigen Tracks, da Spotify-owned Playlists via `/me/playlists` direkt zugänglich sind.
**Warning signs:** 403 bei Playlist-Track-Abfragen.

### Pitfall 6: Artwork-URLs ohne Größenauswahl

**What goes wrong:** Kleinste Bildgröße (64px) wird als Menü-Icon verwendet — sieht auf Retina-Displays unscharf aus.
**Why it happens:** Spotify liefert `images`-Array mit mehreren Größen (640px, 300px, 64px).
**How to avoid:** Immer `_largestImage()` nutzen — größtes Bild (höchste `width`) aus dem Array wählen. LMS-Clients skalieren selbst herunter.
**Warning signs:** Pixelige/unscharfe Artwork-Icons.

### Pitfall 7: LMS-Pagination-Mapping bei Cursor-APIs

**What goes wrong:** LMS übergibt `index=50, quantity=50` an einen Feed-Callback; der Code versucht `offset=50` an `/me/following` zu übergeben — API lehnt ab.
**Why it happens:** Cursor-APIs akzeptieren keinen `offset`, nur `after`-Cursor.
**How to avoid:** Für cursor-basierte Endpoints: Einmalabfrage mit `limit=50` (Recently Played, Top Tracks, gefolgte Künstler sind klein genug). Den `after`-Cursor für Folgeseiten im passthrough-Parameter speichern wenn nötig.
**Warning signs:** HTTP 400 bei index > 0 auf cursor-basierten Endpoints.

---

## Discretion-Entscheidungen

### Suchergebnisse pro Kategorie: 10 (Dev Mode Maximum)

**Empfehlung:** `limit=10` für alle Typen in einem einzigen Request. Dev Mode Maximum ist 10 — 5 wäre zu wenig für brauchbare Suchergebnisse. Der kombinierte Request (`type=track,album,artist,playlist`) liefert 40 Ergebnisse (10 pro Typ) in einem einzigen API-Call.

### Library-Sortierung: "Recently Added" als feste Sortierung

**Empfehlung:** Keine UI-Umschaltung in Phase 3. Spotify API liefert `added_at`-Feld in allen Library-Endpoints — Standardsortierung ist bereits nach `added_at` absteigend (neueste zuerst). Das entspricht dem Erwartungsmodell der meisten Nutzer. Alphabetische Sortierung via `Slim::Utils::Text::ignoreCaseArticles()` kann in Phase 6 (Polish) ergänzt werden.

### Track-Metadaten-Format: line1/line2 nutzen

**Empfehlung:** `line1` = Titel, `line2` = Künstler + Bullet + Album. XMLBrowser.pm unterstützt diese Felder explizit (verified). Spotty-Plugin nutzt identisches Pattern. Zusätzlich `name`-Feld als Fallback für ältere LMS-Clients ohne line1/line2-Unterstützung setzen.

---

## Code Examples

### Spotify-API → LMS-Pagination-Mapping

```perl
# Source: XMLBrowser.pm verified code analysis
# $args kommt von LMS XMLBrowser; index ist 0-basiert
my $offset = $args->{index}    || 0;
my $qty    = $args->{quantity} || 200;
my $limit  = $qty > 50 ? 50 : $qty;  # Spotify max/Seite = 50 für Library-Endpoints
                                       # Search: max = 10 (Dev Mode)

# Response zurück an LMS — total optional aber hilfreich für LMS-Pagination
$callback->({ items => \@items, total => $data->{total} });
```

### Account-ID aus Client ermitteln (etabliertes Projekt-Pattern)

```perl
# Source: Plugin.pm _accountSwitcherFeed() — etabliertes Pattern
sub _getAccountId {
    my ($client) = @_;
    return $prefs->client($client)->get('activeAccount')
        || $prefs->get('activeAccount')
        || '';
}
```

### Passthrough-Parameter für Sub-Feeds

```perl
# Source: Plugin.pm _accountSwitcherFeed() — etabliertes Pattern
push @items, {
    name        => $artist->{name},
    url         => \&_artistFeed,
    passthrough => [{ artistId => $artist->{id} }],
    image       => _largestImage($artist->{images}),
    type        => 'link',
};
```

### Graceful-Hide für Dev-Mode-entfernte Endpoints (NAV-10)

```perl
# Kein Fehler, kein Platzhalter — Menüpunkt einfach weglassen
# Source: D-11, NAV-10
# Artist Top Tracks, Related Artists, Browse Categories, New Releases
# werden in Phase 3 überhaupt nicht als Menüpunkte erzeugt.
# Keine Fehlerbehandlung nötig, da die Methoden nie aufgerufen werden.
```

### Scope-Ergänzung in TokenManager.pm

```perl
# MUSS ergänzt werden — fehlt aktuell (Phase-3-Blocker für NAV-03)
use constant REQUIRED_SCOPES => join(' ', qw(
    user-read-playback-state
    user-modify-playback-state
    user-read-currently-playing
    user-read-recently-played
    user-read-private
    playlist-read-private
    playlist-read-collaborative      # neu — für kollaborative Playlists
    user-library-read
    user-top-read
    user-follow-read                 # NEU — für GET /me/following?type=artist (NAV-03)
    streaming
));
```

---

## Runtime State Inventory

> Nicht anwendbar — Phase 3 ist reine Code-Erweiterung, keine Umbenennung/Migration.

---

## Environment Availability

> Step 2.6: SKIPPED — Phase 3 benötigt keine externen Tools/Services über die LMS-Installation hinaus. Spotify API und bestehende TokenManager-Infrastruktur sind bereits in Phase 2 aufgebaut.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Perl unit tests (keine formale Framework-Konfiguration — LMS-Plugin-Convention) |
| Config file | none — LMS-Plugins nutzen kein separates Test-Framework |
| Quick run command | `perl -c Plugins/SpotOn/Plugin.pm && perl -c Plugins/SpotOn/API/Client.pm` (Syntax-Check) |
| Full suite command | Manuelle LMS-Integration: Plugin laden, Menüpunkte navigieren |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| NAV-01 | Top-Level-Menü zeigt Home/Suche/Library | manual | — | ❌ |
| NAV-02 | Home-Feed zeigt Recently Played + Made For You + Top Tracks | manual | — | ❌ |
| NAV-03 | Library zeigt Liked Songs/Alben/Künstler/Playlists | manual | — | ❌ |
| NAV-04 | Suche "Radiohead" liefert Tracks/Alben/Künstler/Playlists | manual | — | ❌ |
| NAV-05 | Artist-Seite zeigt Alben/Singles/Compilations | manual | — | ❌ |
| NAV-06 | Album-Seite zeigt Trackliste mit Nummer/Dauer/Features | manual | — | ❌ |
| NAV-07 | Playlist-Seite zeigt paginierte Tracks | manual | — | ❌ |
| NAV-08 | Liked Songs ohne Einschränkung zugänglich | manual | — | ❌ |
| NAV-09 | Library-Items nach "recently added" sortiert | manual | — | ❌ |
| NAV-10 | Dev-Mode-entfernte Endpoints fehlen stillschweigend | manual | — | ❌ |
| NAV-11 | Suche mit offset-Paginierung funktioniert | manual | — | ❌ |
| Syntax | Alle .pm-Dateien kompilieren fehlerfrei | automated | `perl -c Plugins/SpotOn/Plugin.pm` | ✅ |

### Wave 0 Gaps

- [ ] Syntax-Check-Skript für alle neuen .pm-Dateien
- [ ] LMS-Instanz für manuelle Integrationstests (beschrieben in VERIFICATION.md)

### Sampling Rate

- **Per task commit:** `perl -c <file>.pm` — Syntax-Check der geänderten Datei
- **Per wave merge:** Manuelle Navigation aller neuen Menüpunkte
- **Phase gate:** Alle 11 NAV-Anforderungen manuell verifiziert via LMS-Web-UI und Jive-Remote

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | nein | Token-Handling bereits in Phase 2 implementiert |
| V3 Session Management | nein | TokenManager.pm bereits vorhanden |
| V4 Access Control | ja | Scope-Prüfung: `user-follow-read` muss vor erstem API-Call vorhanden sein |
| V5 Input Validation | ja | Suchquery aus LMS `$args->{search}` via `uri_escape()` escapen (bereits in Client.pm vorhanden) |
| V6 Cryptography | nein | Kein neuer Krypto-Code in Phase 3 |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Suchquery-Injection via API-URL | Tampering | `URI::Escape::uri_escape()` auf `$args->{search}` — bereits in `_request()` implementiert |
| Spotify CDN-URLs in image-Feld | Information Disclosure | Raw URLs aus Spotify-Response — keine User-Input-URLs; kein Risiko |
| Stale Token nach Scope-Erweiterung | Elevation of Privilege | Re-Auth nach Scope-Änderung erzwingen (PKCE-Flow neu starten) |

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `GET /playlists/{id}/tracks` | `GET /playlists/{id}/items` | Feb 2026 | Alter Pfad deprecated, neuer Pfad verwenden |
| `GET /browse/categories` für Home-Feed | `GET /me/playlists` + owner.id-Filter | Nov 2024 / Feb 2026 | Browse-Kategorien entfernt; Made-For-You via Library |
| `GET /browse/new-releases` | Entfernt in Dev Mode | Feb 2026 | NAV-10: stillschweigend weglassen |
| Search `limit=50` | Search `limit=10` (Dev Mode Max) | Feb 2026 | Auswirkung auf Suchergebnis-Qualität; `offset`-Paginierung nötig |
| `GET /artists/{id}/top-tracks` | Entfernt in Dev Mode | Feb 2026 | NAV-10: Artist-Detail ohne Top-Tracks (D-09) |
| Batch `GET /tracks` (mehrere IDs) | Entfernt in Dev Mode | Feb 2026 | Einzelabrufe oder Cache nutzen |

**Deprecated/outdated:**
- `GET /playlists/{id}/tracks`: Deprecated, `/items` verwenden
- `GET /artists/{id}/top-tracks`: Im Dev Mode entfernt — nicht verwenden
- `GET /browse/featured-playlists`: Deprecated — nicht verwenden
- `implicit_grant` Auth-Flow: Deprecated — PKCE bereits in Projekt implementiert

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `owner.id eq 'spotify'` identifiziert zuverlässig alle Made-For-You-Playlists (Daily Mix, Discover Weekly, Release Radar) | Pattern 5 | Falsch-positive (editorielle Spotify-Playlists landen in Home); Workaround: für Library-Feed ist das trotzdem korrekt (User will keine Editorial-Playlists in "seine" Playlist-Liste) |
| A2 | LMS `$args->{index}` ist 0-basiert (identisch mit Spotify `offset`) | Pattern 1, Code Examples | Wenn 1-basiert: offset off-by-one, erste Seite fehlt Item 0 |
| A3 | `GET /me/following?type=artist` ist in Dev Mode noch verfügbar (Feb 2026 Changelog enthielt keine Entfernung dieses Endpoints) | Standard Stack | Wenn entfernt: NAV-03 Künstler-Library nicht realisierbar |
| A4 | LMS OPMLBased liefert `total`-Feld korrekt an Pagination-Framework zurück wenn `$callback->({ items => ..., total => N })` | Validation Architecture | Wenn ignoriert: Pagination zeigt falsche Seitenanzahl |

**Hinweis:** A1 ist das größte Risiko — empirisch zu testen mit dem echten Account des Entwicklers beim ersten Ausführen.

---

## Open Questions

1. **Re-Auth-Trigger bei Scope-Erweiterung** (RESOLVED)
   - What we know: `user-follow-read` fehlt in REQUIRED_SCOPES. Hinzufügen ändert die Auth-URL.
   - What's unclear: (RESOLVED) Plan 01 Task 2 addresses this: TokenManager checks scope mismatch on token refresh. Plan 01 Task 2 acceptance_criteria now includes grep verification for scope-check logic.
   - Recommendation: Im Plan einen expliziten Schritt einbauen: nach Scope-Erweiterung Token-Cache für alle Accounts invalidieren (einmalig beim Plugin-Start).

2. **Playlist-Items für Spotify-owned Playlists (Made-For-You)** (ACKNOWLEDGED — empirical test at runtime)
   - What we know: Feb 2026: Items nur für Playlists, die der User besitzt oder mitarbeitet.
   - What's unclear: Zählt "Discover Weekly" (owner = spotify) als für den User zugänglich via `/playlists/{id}/items`? Der User "folgt" diesen Playlists.
   - Recommendation: Empirisch testen. Plan 03 Task 1 includes explicit fallback: if getPlaylistItems returns 403 (undef $data), the error callback shows NO_RESULTS textarea. Made-For-You home feed shows playlist list from getUserPlaylists (no drill-down blocked). Playlist detail drill-down may fail gracefully.

3. **Top-Ergebnis bei Suche (D-10)** (RESOLVED)
   - What we know: Kein `best_match`-Feld in der Spotify Search API.
   - What's unclear: (RESOLVED) Plan 03 implements first track from tracks.items[0] as Top Result — the most commonly searched object type. No best_match field needed.
   - Recommendation: Ersten Track aus `tracks.items[0]` als Top-Ergebnis nehmen — Tracks sind das häufigste gesuchte Objekt.

---

## Sources

### Primary (HIGH confidence)

- Existierende Codebase: `Plugin.pm`, `API/Client.pm`, `API/TokenManager.pm` — analysiert
- XMLBrowser.pm Quellcode: `https://raw.githubusercontent.com/LMS-Community/slimserver/public/9.1/Slim/Control/XMLBrowser.pm` — item fields, callback signature, index/quantity
- Spotify Feb 2026 Changelog: `https://developer.spotify.com/documentation/web-api/references/changes/february-2026` — endpoint-Status
- Spotify Feb 2026 Migration Guide: `https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide`
- Spotify GET /me/following Reference: `https://developer.spotify.com/documentation/web-api/reference/get-followed`
- Spotify GET /artists/{id}/albums Reference: `https://developer.spotify.com/documentation/web-api/reference/get-an-artists-albums`
- Spotify Search Reference: `https://developer.spotify.com/documentation/web-api/reference/search`

### Secondary (MEDIUM confidence)

- Spotty-Plugin OPML.pm: `https://github.com/michaelherger/Spotty-Plugin/blob/master/OPML.pm` — item structure patterns, Made-For-You detection
- Spotify Community Bug Report (include_groups pagination): `https://community.spotify.com/t5/Spotify-for-Developers/Web-API-Pagination-for-Get-Artist-s-Albums-broken/td-p/6113264`
- Music Assistant Issue #5360: `https://github.com/music-assistant/support/issues/5360` — Dev Mode API restrictions behavior

### Tertiary (LOW confidence)

- Made-For-You `owner.id = 'spotify'` Detection: aus Spotty-Plugin-Code-Analyse + Community-Forum-Diskussionen — empirisch zu verifizieren

---

## Metadata

**Confidence breakdown:**

- Standard Stack: HIGH — alle Endpunkte gegen offizielle Spotify-Doku verifiziert; CLAUDE.md enthält aktuellen Endpoint-Status-Table
- Architecture: HIGH — auf bestehendem, ausgeführtem Code in Phase 2 aufgebaut; LMS-Patterns aus XMLBrowser.pm direkt verifiziert
- Pitfalls: HIGH (Paginierungsbug, Scope-Missing) / MEDIUM (Made-For-You detection)
- Pagination-Mapping: HIGH — XMLBrowser.pm Quellcode direkt analysiert

**Research date:** 2026-05-28
**Valid until:** 2026-08-28 (stabil — Spotify API-Änderungen im Dev Mode sind dokumentiert; nächste bekannte Änderungswelle nicht angekündigt)
