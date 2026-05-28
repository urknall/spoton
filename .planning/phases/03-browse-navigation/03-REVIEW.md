---
phase: 03-browse-navigation
reviewed: 2026-05-28T00:00:00Z
depth: standard
files_reviewed: 4
files_reviewed_list:
  - Plugins/SpotOn/API/Client.pm
  - Plugins/SpotOn/API/TokenManager.pm
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/strings.txt
findings:
  critical: 4
  warning: 4
  info: 2
  total: 10
status: issues_found
---

# Phase 03: Code Review Report

**Reviewed:** 2026-05-28
**Depth:** standard
**Files Reviewed:** 4
**Status:** issues_found

## Zusammenfassung

Phase 03 liefert die vollständige Browse/Search/Library-Navigationsschicht: 12 neue API-Methoden in `Client.pm`, erweiterte OAuth-Scopes in `TokenManager.pm`, 18 neue i18n-Strings, alle Feed-Handler und Item-Builder in `Plugin.pm`. Das Grundgerüst ist solide — die `_request()`-Pipeline ist korrekt, die Cache-TTL-Regeln decken alle neuen Pfade ab, und das PKCE-Flow bleibt unverändert korrekt.

Vier **BLOCKER** wurden gefunden, die sich auf dasselbe Muster zurückführen lassen: Pagination wurde in drei Feed-Handlern vergessen (hart kodierter `offset => 0`, fehlendes `total` im Callback) und der `_albumItem`-Builder übergibt keine Album-Bilder ans Passthrough, was dazu führt, dass Seite 2+ eines Albums ohne Artwork erscheint. Diese Defekte müssen vor dem Go-live behoben werden.

---

## Kritische Fehler (BLOCKER)

### CR-01: `_savedAlbumsFeed` ignoriert LMS-Pagination — nur erste 50 Alben werden je angezeigt

**File:** `Plugins/SpotOn/Plugin.pm:490–507`

**Issue:** `_savedAlbumsFeed` liest `$args->{index}` und `$args->{quantity}` nie aus. `offset` ist hart auf `0` kodiert, und der Callback gibt kein `total` zurück. Damit fehlt LMS die Information, um weitere Seiten anzufordern. Nutzer mit mehr als 50 gespeicherten Alben sehen nur die ersten 50 — für immer.

Das Gegenstück `_savedTracksFeed` (Zeilen 466–486) macht es korrekt: `$args->{index}` → `offset`, `$args->{quantity}` → `limit`, `total => $data->{total}` im Callback.

**Fix:**
```perl
sub _savedAlbumsFeed {
    my ($client, $callback, $args) = @_;

    my $offset = $args->{index}    || 0;
    my $qty    = $args->{quantity} || 200;
    my $limit  = $qty > 50 ? 50 : $qty;    # Spotify Library max = 50

    my $accountId = _getAccountId($client);

    Plugins::SpotOn::API::Client->getSavedAlbums($accountId, {
        offset => $offset,
        limit  => $limit,
    }, sub {
        my $data = shift;
        unless ($data) {
            $callback->({ items => [{ name => cstring($client, 'PLUGIN_SPOTON_NO_RESULTS'), type => 'textarea' }] });
            return;
        }
        my @items = map { _albumItem($client, $_->{album}) } @{ $data->{items} || [] };
        $callback->({ items => \@items, total => $data->{total} });
    });
}
```

---

### CR-02: `_searchTypeFeed` ignoriert LMS-Pagination — nur erste 10 Suchergebnisse je Typ sichtbar

**File:** `Plugins/SpotOn/Plugin.pm:653–702`

**Issue:** Der Kommentar auf Zeile 652 verspricht explizit „Maps LMS index/quantity to offset/limit", aber der Code liest `$args` nie aus. `offset` ist hart auf `0` kodiert (Zeile 665), und der Callback enthält kein `total` (Zeile 700). LMS kann keine weiteren Seiten laden. NAV-11 (Search-Pagination) ist damit de facto nicht implementiert.

**Fix:**
```perl
sub _searchTypeFeed {
    my ($client, $callback, $args, $passthrough) = @_;

    my $query  = $passthrough->{query} // '';
    my $type   = $passthrough->{type}  // 'track';

    my $offset = $args->{index}    || 0;
    my $qty    = $args->{quantity} || 50;
    my $limit  = $qty > 10 ? 10 : $qty;    # Dev Mode max = 10

    my $accountId = _getAccountId($client);

    Plugins::SpotOn::API::Client->search($accountId, {
        q      => $query,
        type   => $type,
        limit  => $limit,
        offset => $offset,
    }, sub {
        my $data = shift;
        # ... (type-to-key mapping unchanged) ...
        $callback->({ items => \@items, total => $total });
    });
}
```

---

### CR-03: `_artistAlbumsFeed` ignoriert LMS-Pagination — Künstler mit mehr als 50 Einträgen werden abgeschnitten

**File:** `Plugins/SpotOn/Plugin.pm:750–774`

**Issue:** `offset` ist hart auf `0` kodiert (Zeile 760), kein `total` im Callback (Zeile 772). Prolific-Künstler (z. B. Jazz-Labels mit hunderten Compilations) werden bei 50 Einträgen abgeschnitten. NAV-05 verlangt eine korrekte Paginierung der Discography-Sektionen.

**Fix:**
```perl
sub _artistAlbumsFeed {
    my ($client, $callback, $args, $passthrough) = @_;

    my $artistId      = $passthrough->{artistId}      // '';
    my $includeGroups = $passthrough->{includeGroups} // 'album';

    my $offset = $args->{index}    || 0;
    my $qty    = $args->{quantity} || 200;
    my $limit  = $qty > 50 ? 50 : $qty;

    my $accountId = _getAccountId($client);

    Plugins::SpotOn::API::Client->getArtistAlbums($accountId, $artistId, {
        include_groups => $includeGroups,
        offset         => $offset,
        limit          => $limit,
    }, sub {
        my $data = shift;
        unless ($data) {
            $callback->({ items => [{ name => cstring($client, 'PLUGIN_SPOTON_NO_RESULTS'), type => 'textarea' }] });
            return;
        }
        my @items = map { _albumItem($client, $_) } @{ $data->{items} || [] };
        if (!@items) {
            push @items, { name => cstring($client, 'PLUGIN_SPOTON_NO_RESULTS'), type => 'textarea' };
        }
        $callback->({ items => \@items, total => $data->{total} // 0 });
    });
}
```

---

### CR-04: `_albumItem` übergibt `albumImages`/`albumArtist` nicht ans Passthrough — Seite 2+ eines Albums hat kein Artwork

**File:** `Plugins/SpotOn/Plugin.pm:283–298`

**Issue:** `_albumFeed` ist für Seite 2+ auf `$passthrough->{albumImages}` und `$passthrough->{albumArtist}` angewiesen (Zeilen 790–791), aber `_albumItem` befüllt das Passthrough nur mit `{ albumId => $album->{id} }`. Die Felder `albumImages` und `albumArtist` fehlen.

Ablauf: Nutzer öffnet Album → Seite 1 (index=0) → `getAlbum` liefert Metadaten inkl. Bilder. Seite 2 (index>0) → `getAlbumTracks` wird aufgerufen mit `$albumImages = undef` und `$albumArtist = ''` aus dem unvollständigen Passthrough → `_largestImage(undef)` → `''` → alle Tracks auf Seite 2+ erscheinen ohne Artwork.

Das Plan-Dokument (03-02-PLAN.md Zeile 200) beschreibt die Lösung explizit: `passthrough => [{ albumId => $id, albumImages => $album->{images}, albumArtist => $album->{artists}[0]{name} }]`.

**Fix:**
```perl
sub _albumItem {
    my ($client, $album) = @_;
    my $firstArtist = ($album->{artists} && @{$album->{artists}})
        ? $album->{artists}[0]{name}
        : '';
    my $releaseDate = $album->{release_date} // '';
    my $line2 = $firstArtist . ($releaseDate ? " ($releaseDate)" : '');

    return {
        name        => $album->{name} // '',
        url         => \&_albumFeed,
        passthrough => [{ albumId      => $album->{id},
                          albumImages  => $album->{images},
                          albumArtist  => $firstArtist }],
        image       => _largestImage($album->{images}),
        line2       => $line2,
        type        => 'link',
    };
}
```

---

## Warnungen (WARNING)

### WR-01: `getArtist()` in `Client.pm` ist totes Code — nie aufgerufen

**File:** `Plugins/SpotOn/API/Client.pm:145–150`

**Issue:** `getArtist` ist implementiert und in der Plan-01-Acceptance-Liste aufgeführt, wird aber nirgends in `Plugin.pm` aufgerufen. `_artistFeed` zeigt vier statische Sections (Albums, Singles, Compilations, Appears On) ohne vorher `getArtist` aufzurufen, um Künstlername/-bild zu holen. Das ist kein Crash, aber toter Code erzeugt Wartungsaufwand und macht die API-Oberfläche irreführender.

**Fix:** Entweder `getArtist` in `_artistFeed` aufrufen, um den Künstlernamen als Feed-Titel zu setzen, oder `getArtist` aus `Client.pm` entfernen und aus dem Plan streichen, bis Phase 4/5 es tatsächlich braucht.

---

### WR-02: `_recentlyPlayedFeed` und `_savedTracksFeed` prüfen `$_->{track}` nicht auf `undef`

**File:** `Plugins/SpotOn/Plugin.pm:375` und `483`

**Issue:** `_playlistFeed` (Zeile 920–921) hat eine explizite `grep { defined $_->{track} }`-Guard, weil die Spotify API `null`-Track-Einträge für lokale Dateien zurückgibt. Dasselbe Risiko besteht potenziell bei Recently Played und Saved Tracks: wenn die API ein `null`-Track-Objekt liefert, wird `_trackItem($client, undef)` aufgerufen. Zeile 238 würde dann `undef->{name}` dereferenzieren und in Perl mit `Can't use string ("") as a HASH reference` crashen.

**Fix:**
```perl
# _recentlyPlayedFeed (Zeile 375):
my @items = map  { _trackItem($client, $_->{track}) }
            grep { defined $_->{track} }
            @{ $data->{items} || [] };

# _savedTracksFeed (Zeile 483):
my @items = map  { _trackItem($client, $_->{track}) }
            grep { defined $_->{track} }
            @{ $data->{items} || [] };
```

---

### WR-03: `TokenManager` enthält keine Scope-Mismatch-Erkennung — alte Tokens triggern kein Re-Auth

**File:** `Plugins/SpotOn/API/TokenManager.pm:26–38`

**Issue:** `REQUIRED_SCOPES` wird ausschließlich in `startOAuthFlow` (Zeile 155) verwendet, um die Auth-URL zu bauen. Es gibt keinen Code, der gespeicherte Token-Scopes mit `REQUIRED_SCOPES` vergleicht. Die Plan-01-Acceptance-Criteria verlangen explizit: *"TokenManager.pm contains scope mismatch detection logic"*. Die Implementierung fehlt vollständig.

Konsequenz: Nutzer mit alten Tokens (vor Phase 3, ohne `user-follow-read` und `playlist-read-collaborative`) erhalten auf `getFollowedArtists` einen `403`-Fehler, der stillschweigend als `NO_RESULTS`-Textarea angezeigt wird. Es gibt keinen Re-Auth-Prompt.

**Fix:** Im `refreshToken`-Callback oder in `_storeTokens` die gewährten Scopes aus der API-Antwort (`$result->{scope}`) speichern und beim nächsten `getToken`-Aufruf mit `REQUIRED_SCOPES` vergleichen. Fehlende Scopes → `$cb->(undef)` mit speziellem Error-Code `scope_insufficient` → `_onError` triggert den Re-Auth-Flow.

---

### WR-04: `Retry-After: 0` aus der Spotify API deaktiviert den Rate-Limit-Schutz

**File:** `Plugins/SpotOn/API/Client.pm:347–354`

**Issue:** Wenn Spotify mit `429` und `Retry-After: 0` antwortet (technisch valide, bedeutet „sofort neu versuchen"), überschreibt Zeile 350 den Default-Wert mit `0`. Die Deckelung auf 300 (Zeile 352) greift nicht (`0 > 300` ist falsch). Der Cache-Eintrag `RATE_LIMIT_CACHE_KEY` wird mit TTL `0` gesetzt, was bei den meisten Cache-Implementierungen bedeutet „sofort ablaufen" oder „keine Ablaufzeit" — in beiden Fällen ist der Schutz wirkungslos.

**Fix:**
```perl
# Minimum-Backoff erzwingen, nachdem der Header-Wert gelesen wurde:
my $retryAfter = RATE_LIMIT_DEFAULT_BACKOFF;
if ($http && $http->response) {
    my $headerVal = $http->response->header('Retry-After');
    if (defined $headerVal && $headerVal =~ /^\d+$/) {
        $retryAfter = $headerVal > 0 ? $headerVal : RATE_LIMIT_DEFAULT_BACKOFF;
    }
}
$retryAfter = 300 if $retryAfter > 300;
```

---

## Info

### IN-01: Suchergebnis-Zähler in `_searchFeed` ist nicht lokalisiert

**File:** `Plugins/SpotOn/Plugin.pm:611, 620, 629, 638`

**Issue:** Die `line2`-Felder der Suchkategorien verwenden die hartkodierten englischen Strings `"$tracksTotal results"`, `"$albumsTotal results"` usw. Für deutschsprachige Nutzer erscheint also „100 results" statt „100 Ergebnisse". Die Phase-01-Strings decken diesen Fall nicht ab.

**Fix:** Neuen i18n-Key `PLUGIN_SPOTON_SEARCH_RESULTS_COUNT` (DE: `%s Ergebnisse`, EN: `%s results`) in `strings.txt` hinzufügen und mit `cstring($client, 'PLUGIN_SPOTON_SEARCH_RESULTS_COUNT', $tracksTotal)` aufrufen. Alternativ ist `line2` optional und kann weggelassen werden, um das Problem zu vermeiden.

---

### IN-02: `_userPlaylistsFeed` hat keine Pagination — Nutzer mit >50 Playlists verlieren Einträge

**File:** `Plugins/SpotOn/Plugin.pm:533–552`

**Issue:** `offset` ist hart auf `0` kodiert und `total` fehlt im Callback — identisches Muster wie die Blocker CR-01/CR-02/CR-03. Der Unterschied: die Made-For-You-Filterung macht eine korrekte Pagination semantisch schwierig (API-`total` enthält gefilterte Einträge), was im Plan als „akzeptierte Einschränkung" dokumentiert ist.

Trotzdem: Die vollständige Paginierung ist implementierbar (offset aus `$args->{index}`, total aus API mit Hinweis auf die Ungenauigkeit). In Phase 3 trifft dies nur Nutzer mit mehr als 50 Playlists. Als Info-Level eingestuft, weil der Plan die Einschränkung explizit dokumentiert.

**Fix (wenn gewünscht):**
```perl
my $offset = $args->{index}    || 0;
my $qty    = $args->{quantity} || 200;
my $limit  = $qty > 50 ? 50 : $qty;
# offset und limit an getUserPlaylists übergeben, total im Callback setzen
```

---

_Reviewed: 2026-05-28_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
