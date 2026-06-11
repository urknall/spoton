# Phase 15: Like Button - Research

**Researched:** 2026-06-11
**Domain:** LMS trackInfoMenu pattern, Spotify Unified Library API (Feb 2026), SimpleAsyncHTTP JSON body, scope handling
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Like/Unlike wird über den `trackInfoMenu()`-Hook registriert (`Slim::Menu::TrackInfo->registerInfoProvider`)
- **D-02:** NICHT im `items`-Array von `_trackItem()`. Das `items`-Array bleibt für Navigation.
- **D-03:** Dynamisches Label: "Like" wenn nicht geliked, "Unlike" wenn geliked
- **D-04:** Nur für Tracks (type=tracks). Kein Album-Like in dieser Phase
- **D-05:** Labels in strings.txt für alle 11 Sprachen (PLUGIN_SPOTON_LIKE / PLUGIN_SPOTON_UNLIKE)
- **D-06:** On-demand State-Check: `GET /me/library/contains` erst beim Öffnen des Info-Menüs, kein Pre-fetch
- **D-07:** 60s TTL-Cache für Liked-State. Wiederholtes Öffnen innerhalb einer Minute = kein API-Call
- **D-08:** Sofortige Cache-Invalidierung nach Like/Unlike. Kein optimistisches Update
- **D-09:** Erfolg: `showBriefly => 1` + `nextWindow => 'grandparent'`
- **D-10:** API-Fehler: Fehlermeldung via showBriefly. Bei 403: Hinweis auf fehlende Berechtigung. User bleibt im Menü
- **D-11:** Bei 429: Standard-Retry via Client.pm, kein spezielles Handling
- **D-12:** Save: `PUT /me/library` mit `uris=spotify:track:ID` (neuer unified Endpoint)
- **D-13:** Remove: `DELETE /me/library` mit `uris=spotify:track:ID`
- **D-14:** Check: `GET /me/library/contains` mit `uris=spotify:track:ID`
- **D-15:** Token-Flavor: "own" (me/*-Endpoints)

### Claude's Discretion

- Scope-Upgrade-Mechanismus klären
- Genaue trackInfoMenu-Registrierung
- Cache-Key-Format für Liked-State
- Request-Body-Format für /me/library Endpoints

### Deferred Ideas (OUT OF SCOPE)

- Album-Like (PUT /me/library mit type=albums)
- Connect Now-Playing Like (LIB-06)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| LIB-01 | User can save currently playing or browsed track to Liked Songs via context menu action | trackInfoMenu-Hook, PUT /me/library endpoint, Client.pm saveTracks method |
| LIB-02 | User can remove track from Liked Songs via context menu action (Unlike) | trackInfoMenu-Hook, DELETE /me/library endpoint, Client.pm removeTracks method |
| LIB-03 | Track context menu shows current liked state — displays "Like" or "Unlike" based on library check | QobuzManageFavorites-Pattern, GET /me/library/contains, 60s TTL Cache |
| LIB-04 | Liked state check uses `GET /me/library/contains` without adding noticeable delay to menu rendering | On-demand (D-06), async via SimpleAsyncHTTP, 60s TTL cache hit = zero delay |
| LIB-05 | Plugin requests `user-library-modify` and `user-library-read` scopes; upgrade triggers one-time re-auth via cacheSchemaVersion bump | Scope-Analyse: librespot ZeroConf already includes both scopes — no bump needed in most cases; graceful 403 fallback as safety net |
</phase_requirements>

---

## Summary

Phase 15 implementiert den Like-Button für SpotOn. Die drei technischen Kernbereiche sind: (1) Integration in das LMS `trackInfoMenu`-System nach Qobuz-Vorbild, (2) die drei neuen Spotify Unified Library Endpoints (PUT/DELETE/GET /me/library), und (3) die Anpassung von `Client.pm`, um JSON-Bodys bei PUT/DELETE zu senden.

Die wichtigste Entdeckung betrifft die Scope-Frage (LIB-05): Die librespot-Binary des Projekts (`spotty-ng/librespot/src/main.rs`) deklariert `user-library-modify` und `user-library-read` explizit in `OAUTH_SCOPES`. Da ZeroConf-Credentials bereits unter diesem Scope-Set ausgestellt werden, ist ein cacheSchemaVersion-Bump für bestehende Nutzer **nicht zwingend erforderlich**. Die login5-Token, die SpotOn via `--get-token` abruft, enthalten diese Scopes bereits. Ein cacheSchemaVersion-Bump (2 → 3) ist trotzdem als Sicherheitsnetz sinnvoll: Er löscht Token-Caches und erzwingt einen Neuabruf — im Gegensatz zu einem Re-Auth-Flow ist das transparent für den Nutzer.

Das zweite kritische Finding betrifft die API-Requests: `PUT /me/library` und `DELETE /me/library` akzeptieren die Track-URI als **Query-Parameter** (`uris=spotify%3Atrack%3AID`), nicht als JSON-Body. `GET /me/library/contains` ebenfalls als Query-Parameter. Damit kann das bestehende `_request`-System unverändert genutzt werden — kein JSON-Body-Support nötig.

**Primary recommendation:** trackInfoMenu-Pattern exakt nach Qobuz implementieren. `_request` ohne Modifikation nutzen (URI als Query-Param). cacheSchemaVersion 2 → 3 bumpen als transparentes Sicherheitsnetz, kein expliziter Re-Auth-Dialog nötig.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Like/Unlike-Menüeintrag anzeigen | Frontend (LMS Plugin) | — | trackInfoMenu-Hook ist Plugin-seitig registriert |
| Liked-State prüfen | API-Schicht (Client.pm) | Cache (Slim::Utils::Cache) | State-Check via GET /me/library/contains, 60s TTL |
| Track in Library speichern | API-Schicht (Client.pm) | — | PUT /me/library, me/* → own-Token |
| Track aus Library entfernen | API-Schicht (Client.pm) | — | DELETE /me/library, me/* → own-Token |
| Cache-Invalidierung nach Aktion | Plugin.pm Callback | Client.pm | Direkter Cache-Delete nach erfolgreicher Aktion |
| Scope-Handling | Binary (librespot) | Plugin.pm (cacheSchemaVersion) | Scopes via login5 aus gespeicherten Credentials |

---

## Standard Stack

### Core (keine neuen Abhängigkeiten)

| Modul | Version | Zweck | Warum Standard |
|-------|---------|-------|----------------|
| `Slim::Menu::TrackInfo` | LMS 8.0+ | trackInfoMenu-Hook-Registrierung | Etabliertes LMS-Pattern, Qobuz-Referenz |
| `Slim::Utils::Cache` | LMS 8.0+ | 60s Liked-State-Cache | Bereits in Plugin.pm und Client.pm initialisiert |
| `Plugins::SpotOn::API::Client` | — | saveTracks / removeTracks / checkTracks | Bestehender zentraler API-Dispatcher |

**Installation:** Keine neuen Packages. Alle benötigten Module sind bereits vorhanden.

---

## Package Legitimacy Audit

> Diese Phase installiert keine neuen externen Packages. Abschnitt entfällt.

---

## Architecture Patterns

### System Architecture Diagram

```
User öffnet Track-Info-Menü
         │
         ▼
trackInfoMenu($client, $url, $track, $remoteMeta)
         │ extrahiert Track-URI aus $remoteMeta->{uri} oder via ProtocolHandler::crackUrl
         │
         ▼
SpotOnManageLike($client, $cb, $params, $args)  ← Menü-Callback
         │
         ├─► Cache-Lookup: spoton_liked_{accountId}_{trackId}   TTL=60s
         │         │ HIT                      MISS
         │         │                           │
         │         ▼                           ▼
         │    isFavorite                Client->checkTracks()
         │                               GET /me/library/contains
         │                               ?uris=spotify:track:ID
         │                               Response: [true|false]
         │
         ▼
{name => 'Like'|'Unlike', url => \&SpotOnLike|\&SpotOnUnlike, passthrough => [{trackUri}]}
         │
User wählt Aktion
         │
   ┌─────┴─────┐
   ▼           ▼
SpotOnLike   SpotOnUnlike
Client->     Client->
saveTracks   removeTracks
PUT          DELETE
/me/library  /me/library
?uris=...    ?uris=...
   │           │
   └─────┬─────┘
         │ Cache löschen: spoton_liked_{accountId}_{trackId}
         ▼
showBriefly + nextWindow => 'grandparent'
```

### Recommended Project Structure (Änderungen)

```
Plugins/SpotOn/
├── API/
│   └── Client.pm          # +3 neue Methoden: saveTracks, removeTracks, checkTracks
│                          # +1 Anpassung _cacheTTL: me/library → 60s (optional, da manueller Cache)
└── Plugin.pm              # +registerInfoProvider in initPlugin()
                           # +3 neue Subs: trackInfoMenu, SpotOnManageLike, SpotOnLike/SpotOnUnlike
                           # SPOTON_CACHE_VERSION: 2 → 3
strings.txt                # +PLUGIN_SPOTON_LIKE, +PLUGIN_SPOTON_UNLIKE, +PLUGIN_SPOTON_LIKE_ERROR
                           # +PLUGIN_SPOTON_LIKED, +PLUGIN_SPOTON_UNLIKED  (11 Sprachen je)
```

### Pattern 1: trackInfoMenu-Registrierung (Qobuz-Pattern)

**Was:** Hook ins LMS TrackInfo-System — erscheint in jedem Track-Kontextmenü.
**Wann nutzen:** Für Track-Aktionen, die nicht zur Navigation gehören.

```perl
# Source: github.com/LMS-Community/plugin-Qobuz/blob/master/Plugin.pm (VERIFIED)
# In initPlugin():
Slim::Menu::TrackInfo->registerInfoProvider( spotonTrackInfo => (
    func => \&trackInfoMenu,
) );
```

### Pattern 2: trackInfoMenu Callback-Signatur

```perl
# Source: Qobuz Plugin.pm (VERIFIED via WebFetch)
sub trackInfoMenu {
    my ($client, $url, $track, $remoteMeta, $tags) = @_;

    # Track-ID aus URI extrahieren
    # $remoteMeta->{uri} enthält 'spotify:track:ID' für SpotOn-Tracks
    my $trackUri = $remoteMeta ? $remoteMeta->{uri} : undef;
    return unless $trackUri && $trackUri =~ /^spotify:track:/;

    my $items = [];
    push @$items, {
        name        => cstring($client, 'PLUGIN_SPOTON_MANAGE_LIKE'),
        url         => \&SpotOnManageLike,
        passthrough => [{ trackUri => $trackUri, accountId => _getAccountId($client) }],
    };

    return { items => $items };
    # WICHTIG: KEIN _objInfoHandler wie Qobuz — SpotOn hat kein Label-System
}
```

### Pattern 3: ManageLike Callback (State-Check + dynamisches Label)

```perl
# Source: Qobuz QobuzManageFavorites() Pattern (VERIFIED via WebFetch)
sub SpotOnManageLike {
    my ($client, $cb, $params, $args) = @_;

    my $trackUri  = $args->{trackUri};
    my $accountId = $args->{accountId};

    # Cache-Key: stabil, pro Account + Track
    (my $trackId) = $trackUri =~ /^spotify:track:(.+)$/;
    my $cacheKey = "spoton_liked_${accountId}_${trackId}";

    my $buildMenu = sub {
        my ($isLiked) = @_;
        $cb->({ items => [{
            name        => cstring($client, $isLiked ? 'PLUGIN_SPOTON_UNLIKE' : 'PLUGIN_SPOTON_LIKE'),
            url         => $isLiked ? \&SpotOnUnlike : \&SpotOnLike,
            passthrough => [{ trackUri => $trackUri, accountId => $accountId, cacheKey => $cacheKey }],
            nextWindow  => 'grandparent',
        }] });
    };

    # D-07: Cache-Lookup zuerst
    my $cached = $cache->get($cacheKey);
    if (defined $cached) {
        $buildMenu->($cached);
        return;
    }

    # D-06: On-demand API-Call
    Plugins::SpotOn::API::Client->checkTracks($accountId, [$trackUri], sub {
        my ($result, $err) = @_;
        my $isLiked = ($result && ref $result eq 'ARRAY' && $result->[0]) ? 1 : 0;
        $cache->set($cacheKey, $isLiked, 60);  # D-07: 60s TTL
        $buildMenu->($isLiked);
    });
}
```

### Pattern 4: Like/Unlike Aktions-Callbacks

```perl
# Source: Qobuz QobuzAddFavorite/QobuzDeleteFavorite Pattern (VERIFIED via WebFetch)
sub SpotOnLike {
    my ($client, $cb, $params, $args) = @_;
    my $trackUri  = $args->{trackUri};
    my $accountId = $args->{accountId};
    my $cacheKey  = $args->{cacheKey};

    Plugins::SpotOn::API::Client->saveTracks($accountId, [$trackUri], sub {
        my ($result, $err) = @_;
        if ($err && $err->{code} != 200) {
            my $msg = ($err->{code} == 403)
                ? cstring($client, 'PLUGIN_SPOTON_LIKE_ERROR_SCOPE')
                : cstring($client, 'PLUGIN_SPOTON_LIKE_ERROR');
            $cb->({ items => [{ name => $msg, showBriefly => 1 }] });
            return;
        }
        $cache->remove($cacheKey);  # D-08: Sofortige Invalidierung
        $cb->({ items => [{ name => cstring($client, 'PLUGIN_SPOTON_LIKED'), showBriefly => 1, nextWindow => 'grandparent' }] });
    });
}
```

### Pattern 5: Client.pm — neue API-Methoden

Die drei neuen Methoden nutzen **Query-Parameter**, kein JSON-Body (verifiziert via offizielle Spotify-Doku):

```perl
# PUT /me/library?uris=spotify:track:ID — Query-Parameter, kein Body
# Source: developer.spotify.com/documentation/web-api/reference/save-library-items (CITED)
sub saveTracks {
    my ($class, $accountId, $uris, $cb) = @_;
    $class->_request('put', 'me/library', {
        _accountId => $accountId,
        _noCache   => 1,
        uris       => join(',', @{$uris || []}),
    }, $cb);
}

# DELETE /me/library?uris=spotify:track:ID — Query-Parameter bestätigt
# Source: developer.spotify.com/documentation/web-api/reference/remove-library-items (CITED)
sub removeTracks {
    my ($class, $accountId, $uris, $cb) = @_;
    $class->_request('delete', 'me/library', {
        _accountId => $accountId,
        _noCache   => 1,
        uris       => join(',', @{$uris || []}),
    }, $cb);
}

# GET /me/library/contains?uris=spotify:track:ID — Response: [true|false]
# Source: developer.spotify.com/documentation/web-api/reference/check-library-contains (VERIFIED)
sub checkTracks {
    my ($class, $accountId, $uris, $cb) = @_;
    $class->_request('get', 'me/library/contains', {
        _accountId => $accountId,
        _noCache   => 1,   # Caching läuft über Plugin.pm (D-07), nicht über _cacheTTL
        uris       => join(',', @{$uris || []}),
    }, $cb);
}
```

**WICHTIG:** Die `_request`-Methode sendet alle Nicht-`_`-Parameter als Query-String. PUT/DELETE mit Query-Params funktioniert mit den neuen `/me/library`-Endpoints korrekt. Der `Content-Length: 0`-Header, den `_request` für PUT/POST setzt, ist für Query-Param-basierte Calls unproblematisch.

### Pattern 6: DELETE via SimpleAsyncHTTP

```perl
# SimpleAsyncHTTP unterstützt delete() — analog zu get/put/post
# Source: github.com/LMS-Community/slimserver/.../SimpleHTTP/Base.pm (VERIFIED via WebFetch)
# sub delete { shift->_createHTTPRequest( DELETE => @_ ) }
# Wird in _request als $http->$method($url, @headers) aufgerufen — funktioniert für 'delete'
```

### Anti-Patterns to Avoid

- **Kein JSON-Body für PUT/DELETE Library:** Die Endpoints akzeptieren Query-Params. Kein Umbau von `_request` nötig.
- **Kein Pre-fetch bei Track-Listing:** D-06 verbietet Liked-State-Check beim Rendern der Trackliste. Nur on-demand beim Öffnen des Info-Menüs.
- **Kein optimistisches Update:** D-08: Cache immer löschen nach Aktion, nie spekulativ setzen.
- **Kein `favorites_url` auf Track-Items:** D-02: `favorites_url` bleibt für Album/Playlist-Items. Track-Items bekommen keinen `favorites_url`-Key.
- **`me/library/contains` liegt außerhalb aller bestehenden `_cacheTTL`-Regeln:** Der Standard-TTL wäre 0. Daher `_noCache => 1` + manuelles Cache-Set in SpotOnManageLike verwenden.

---

## Don't Hand-Roll

| Problem | Nicht selbst bauen | Stattdessen | Warum |
|---------|-------------------|-------------|-------|
| Track-Kontextmenü | Eigenes OPML-Menü-Item in `_trackItem()` | `Slim::Menu::TrackInfo->registerInfoProvider` | LMS-Standard, automatisch in alle Track-Ansichten injiziert |
| i18n-Labels | Hardcodierte Strings | `cstring($client, 'KEY')` + strings.txt | Alle 11 LMS-Sprachen, bereits überall genutzt |
| HTTP-Requests | Eigener HTTP-Client | `Client.pm->_request()` mit Rate-Limiting | Rate-Limit, Dual-Token, Cache, Concurrency Cap bereits implementiert |

---

## Specifics: Scope-Handling (LIB-05)

### Befund (VERIFIED via Codebase)

Die Datei `/home/sti/spotty-ng/librespot/src/main.rs` deklariert:

```rust
static OAUTH_SCOPES: &[&str] = &[
    ...
    "user-library-modify",
    "user-library-read",
    ...
];
```

[VERIFIED: /home/sti/spotty-ng/librespot/src/main.rs Zeile 210-211]

### Was das bedeutet

Die librespot-Binary, die SpotOn verwendet, fordert `user-library-modify` und `user-library-read` bereits beim ZeroConf-Verbindungsaufbau. Login5-Token, die via `--get-token` abgerufen werden, spiegeln die autorisierten Scopes der gespeicherten Credentials wider.

**Für Nutzer, die SpotOn vor Phase 15 installiert haben:** Ihre Credentials wurden unter einem Scope-Set erstellt, das `user-library-modify` und `user-library-read` bereits enthält. Ein neuer Token-Abruf (der automatisch beim nächsten Cache-Miss passiert) liefert diese Scopes.

### Empfohlener Mechanismus (cacheSchemaVersion-Bump)

Ein Bump `SPOTON_CACHE_VERSION` 2 → 3 ist das richtige Werkzeug für LIB-05, aber aus einem anderen Grund als erwartet: Er löscht **gecachte Tokens** aus dem LMS-Cache, sodass beim nächsten Start frische Tokens mit den aktuellen Scopes abgerufen werden. Das ist transparent — kein Re-Auth-Dialog, kein ZeroConf-Reconnect.

**Was der Bump NICHT macht:** Er löscht keine librespot-Credentials aus dem Dateisystem (`~/.cache/spoton/`). Die Credentials sind separat und enthalten bereits alle Scopes.

**Graceful 403-Fallback:** Falls ein Nutzer trotzdem einen 403 auf `/me/library` bekommt (z.B. sehr alte Credentials vor der OAUTH_SCOPES-Erweiterung), zeigt D-10 die Fehlermeldung `PLUGIN_SPOTON_LIKE_ERROR_SCOPE`. Kein Silent-Fail. Der Nutzer kann dann manuell re-authentifizieren via ZeroConf (normaler Settings-Flow).

**cacheSchemaVersion-Mechanismus:**
- `SPOTON_CACHE_VERSION`: 2 → 3
- `Plugin.pm initPlugin()`: bestehende Guard-Logik (Zeile 56-59) greift automatisch
- `TokenManager.pm`: `$cache = Slim::Utils::Cache->new('spoton', 2)` → 3 (Token-Cache-Namespace)
- **Keine neue strings.txt-Message nötig** — der Bump ist transparent

[ASSUMED] Ob sehr alte Installations-Credentials (vor OAUTH_SCOPES-Erweiterung) tatsächlich `user-library-modify` enthalten, ist nicht mit 100% Sicherheit verifizierbar ohne Live-Test. Der 403-Fallback (D-10) deckt diesen Fall ab.

---

## Specifics: API-Endpoint-Details (VERIFIED/CITED)

### GET /me/library/contains
- **Parameter:** `uris` (Query-String, comma-separated Spotify URIs)
- **Format:** `?uris=spotify%3Atrack%3AID`
- **Max:** 40 URIs (Phase 15 verwendet stets genau 1)
- **Response:** `[true]` oder `[false]` (Array of booleans)
- **Scope:** `user-library-read`
- [VERIFIED: developer.spotify.com/documentation/web-api/reference/check-library-contains]

### PUT /me/library
- **Parameter:** `uris` (Query-String oder JSON-Body — Query-Param ist einfacher)
- **Format:** `?uris=spotify%3Atrack%3AID`
- **Response:** 200 OK, leerer Body
- **Scope:** `user-library-modify`
- [CITED: developer.spotify.com/documentation/web-api/reference/save-library-items]

### DELETE /me/library
- **Parameter:** `uris` (Query-String)
- **Format:** `?uris=spotify%3Atrack%3AID`
- **Response:** 200 OK, leerer Body
- **Scope:** `user-library-modify`
- [CITED: developer.spotify.com/documentation/web-api/reference/remove-library-items]

### Cache-Key-Format (D-07, Claude's Discretion)

Empfehlung: `spoton_liked_{accountId}_{trackId}`

Begründung:
- `trackId` = Spotify-Track-ID (aus URI extrahiert, z.B. `4iV5W9uYEdYUVa79Axb7Rh`)
- `accountId` = bestehende Konvention für per-Account-Keys (CR-01 in Client.pm)
- Kurz, lesbar, keine Sonderzeichen im Key

Beispiel: `spoton_liked_abc123_4iV5W9uYEdYUVa79Axb7Rh`

---

## Specifics: SimpleAsyncHTTP DELETE-Support (VERIFIED)

`Slim::Networking::SimpleHTTP::Base` definiert:
```perl
sub delete { shift->_createHTTPRequest( DELETE => @_ ) }
```
[VERIFIED: github.com/LMS-Community/slimserver public/8.5 SimpleHTTP/Base.pm via WebFetch]

`Client.pm` ruft `$http->$method($url, @headers)` auf — `$method` = `'delete'` funktioniert identisch zu `'get'`, `'put'`, `'post'`.

---

## Common Pitfalls

### Pitfall 1: trackInfoMenu wird für ALLE Tracks aufgerufen

**Was schiefläuft:** `trackInfoMenu()` wird für jede Track-URL aufgerufen, auch für Nicht-SpotOn-Tracks (lokale Musik, Radio, andere Plugins).
**Warum:** `registerInfoProvider` ist global — LMS ruft den Handler für alle Tracks auf.
**Vermeidung:** Zu Beginn von `trackInfoMenu()` prüfen: `return unless $url =~ /^spoton:\/\//` oder `return unless $remoteMeta && $remoteMeta->{uri} =~ /^spotify:track:/`.
**Warning-Zeichen:** Like-Button erscheint in nicht-Spotify-Track-Menüs.

### Pitfall 2: _cacheTTL gibt 0 für me/library/contains

**Was schiefläuft:** `checkTracks()` wird mit `_noCache => 0` aufgerufen — `_request` versucht zu cachen, aber `_cacheTTL('me/library/contains')` gibt 0 zurück, also wird nie gecacht.
**Warum:** `_cacheTTL` kennt `me/library` noch nicht (nur `me/tracks`, `me/albums`, etc.).
**Vermeidung:** `_noCache => 1` in `checkTracks()` setzen, Caching manuell in `SpotOnManageLike` via `$cache->set($cacheKey, $isLiked, 60)` regeln.

### Pitfall 3: Content-Length: 0 bei PUT /me/library

**Was schiefläuft:** `_request` setzt für PUT `Content-Length: 0` (bestehende Logik aus D-04). Das ist für Query-Param-basierte PUT-Calls kein Problem, da kein Body gesendet wird. Wäre ein Body vorhanden, wäre Content-Length falsch.
**Warum relevant:** Die Entscheidung, Query-Params statt JSON-Body zu nutzen, vermeidet dieses Problem vollständig.
**Vermeidung:** Query-Params verwenden (wie entschieden). Kein `_request`-Umbau nötig.

### Pitfall 4: remoteMeta->{uri} kann fehlen

**Was schiefläuft:** Bei manchen Track-Quellen (z.B. DSTM, Connect-Now-Playing) ist `$remoteMeta` undef oder enthält keine `uri`.
**Warum:** trackInfoMenu wird mit verschiedenen Track-Herkunften aufgerufen.
**Vermeidung:** Defensive Guards: `return unless $remoteMeta && $remoteMeta->{uri} && $remoteMeta->{uri} =~ /^spotify:track:/`.

### Pitfall 5: nextWindow-Wert

**Was schiefläuft:** `nextWindow => 'parent'` navigiert nur eine Ebene zurück (bleibt im trackInfo-Menü). Für Like/Unlike-Aktionen wollen wir zwei Ebenen zurück.
**Warum:** Qobuz nutzt `nextWindow => 'grandparent'` für Add/Delete-Aktionen, aber die SpotOnManageLike-Callback muss `nextWindow` auf dem Item-Level, nicht auf Response-Level setzen.
**Vermeidung:** `nextWindow => 'grandparent'` auf dem Aktions-Item setzen (nicht auf der ManageLike-Response selbst). Qobuz-Pattern exakt folgen.

### Pitfall 6: Leere API-Response bei PUT/DELETE (200 OK, kein JSON)

**Was schiefläuft:** `_request`'s success-callback ruft `from_json($http->content)` auf — bei leerem Body schlägt `from_json('')` fehl.
**Warum:** PUT /me/library und DELETE /me/library geben 200 OK mit leerem Body zurück.
**Vermeidung:** In `saveTracks`/`removeTracks` einen Response-Wrapper nutzen, der leere Antwort als Erfolg interpretiert: Im Callback `$result || {}` annehmen; leerer Body → `$result` ist undef, aber `$err` ist auch undef → Erfolg. **Alternativ:** `_request` erweitern um leere-Body-Toleranz — aber das ist möglicherweise schon der Fall wenn JSON-Parse-Error bei leerem Body als `undef` zurückkommt. Dies muss im Plan verifiziert werden.

---

## Code Examples

### Vollständiges trackInfoMenu-Registrierungsbeispiel

```perl
# Source: Qobuz Plugin.pm Zeile ~131 (VERIFIED via WebFetch)
# In initPlugin(), nach SUPER::initPlugin():
Slim::Menu::TrackInfo->registerInfoProvider( spotonTrackInfo => (
    func => \&trackInfoMenu,
) );
```

### strings.txt-Format (bestehende Konvention)

```
PLUGIN_SPOTON_LIKE
    CS  Líbit se
    DA  Synes om
    DE  Gefällt mir
    EN  Like
    ES  Me gusta
    FR  J'aime
    IT  Mi piace
    NL  Vind ik leuk
    NO  Lik
    PL  Lubię to
    SV  Gilla

PLUGIN_SPOTON_UNLIKE
    CS  Přestat se líbit
    DA  Synes ikke om
    DE  Gefällt mir nicht mehr
    EN  Unlike
    ES  Ya no me gusta
    FR  Je n'aime plus
    IT  Non mi piace più
    NL  Vind ik niet meer leuk
    NO  Fjern like
    PL  Nie lubię
    SV  Ogilla
```

---

## State of the Art

| Alter Ansatz | Aktueller Ansatz | Wann geändert | Impact |
|-------------|-----------------|--------------|--------|
| `PUT /me/tracks?ids=...` | `PUT /me/library?uris=spotify:track:ID` | Feb 2026 | Neue Endpoints für Phase 15 zwingend |
| `DELETE /me/tracks?ids=...` | `DELETE /me/library?uris=spotify:track:ID` | Feb 2026 | — |
| `GET /me/tracks/contains?ids=...` | `GET /me/library/contains?uris=...` | Feb 2026 | URI-Format statt ID-Format |

**Deprecated/veraltet:**
- `PUT /me/tracks` (Deprecated): Ersetzt durch `PUT /me/library`
- `DELETE /me/tracks` (Deprecated): Ersetzt durch `DELETE /me/library`
- `GET /me/tracks/contains` (Deprecated): Ersetzt durch `GET /me/library/contains`

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Bestehende ZeroConf-Credentials enthalten `user-library-modify` (weil OAUTH_SCOPES in der Binary diesen Scope schon immer enthielt) | Scope-Handling | Nutzer bekommen 403 beim ersten Like — abgefangen durch D-10 403-Fallback |
| A2 | `PUT /me/library` mit Query-Param `uris` funktioniert ohne JSON-Body | API-Endpoints | Würde HTTP 400 zurückgeben; Fallback: JSON-Body in `_request` nachrüsten |
| A3 | `$http->delete($url, @headers)` in SimpleAsyncHTTP funktioniert analog zu `get`/`put` | SimpleAsyncHTTP | HTTP-Methode wird nicht gesendet; Test in Wave 0 nötig |
| A4 | Leerer 200-Body von PUT/DELETE /me/library führt nicht zu einem unbehandelten Fehler in `_request` | Code-Flow | Stille Fehler beim Like/Unlike; explizite Behandlung im Callback als Fallback |

---

## Open Questions (RESOLVED)

1. **Verhält sich `_request` bei leerem 200-Body korrekt?**
   - RESOLVED: Empty-Body-Guard in `_doFlavouredRequest` — leerer Content → `$result = undef, $err = undef` (kein parse_error). Plan 15-01 Task 1 implementiert den Guard.

2. **Sind Fehlerstrings für 11 Sprachen realistisch in Phase 15?**
   - RESOLVED: `PLUGIN_SPOTON_LIKE_ERROR` und `PLUGIN_SPOTON_LIKE_ERROR_SCOPE` nur EN-only. `PLUGIN_SPOTON_LIKE` und `PLUGIN_SPOTON_UNLIKE` in allen 11 Sprachen. LMS-Standard EN-Fallback greift für fehlende Sprachen. Plan 15-01 Task 2 setzt dies um.

---

## Environment Availability

> Keine neuen externen Abhängigkeiten. Abschnitt entfällt — alle benötigten LMS-Module sind vorhanden.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Test::More (Perl built-in) |
| Config file | t/ directory, keine prove.ini |
| Quick run command | `prove t/08_api_client.t t/05_perl_syntax.t` |
| Full suite command | `prove t/` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| LIB-01 | saveTracks sendet PUT /me/library?uris=... | unit | `prove t/08_api_client.t` | ❌ Wave 0 |
| LIB-02 | removeTracks sendet DELETE /me/library?uris=... | unit | `prove t/08_api_client.t` | ❌ Wave 0 |
| LIB-03 | checkTracks gibt [true]/[false] korrekt aus | unit | `prove t/08_api_client.t` | ❌ Wave 0 |
| LIB-04 | checkTracks nutzt Cache (kein 2. API-Call bei Cache-Hit) | unit | `prove t/08_api_client.t` | ❌ Wave 0 |
| LIB-05 | SPOTON_CACHE_VERSION = 3 in Plugin.pm | unit | `prove t/05_perl_syntax.t` | ✅ (Syntax-Check) |
| LIB-05 | cacheSchemaVersion-Guard setzt Version auf 3 | unit | `prove t/08_api_client.t` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `prove t/05_perl_syntax.t t/08_api_client.t`
- **Per wave merge:** `prove t/`
- **Phase gate:** Full suite green vor `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `t/08_api_client.t` — neue Tests für saveTracks, removeTracks, checkTracks, Cache-Invalidierung
- [ ] Kein neues Test-File nötig — Erweiterung von `08_api_client.t` ausreichend

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | nein | — |
| V3 Session Management | nein | — |
| V4 Access Control | ja | me/* → own-Token-Guard (bereits in Client.pm D-05) |
| V5 Input Validation | ja | Track-URI-Format prüfen vor API-Call (`spotify:track:[A-Za-z0-9]+`) |
| V6 Cryptography | nein | — |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| URI-Injection via $remoteMeta->{uri} | Tampering | Regex-Validation: `$uri =~ /^spotify:track:[A-Za-z0-9]+$/` vor API-Call |
| Token-Leakage in Logs | Info Disclosure | Bestehende Konvention: nie Token loggen (T-02-10 in Client.pm) |

---

## Sources

### Primary (HIGH confidence)
- `/home/sti/spotty-ng/librespot/src/main.rs` — OAUTH_SCOPES-Liste mit user-library-modify/read [VERIFIED: Codebase]
- `/home/sti/spotty-ng/librespot/core/src/token.rs` — Token-Provider Scope-Handling [VERIFIED: Codebase]
- `https://developer.spotify.com/documentation/web-api/reference/check-library-contains` — GET /me/library/contains uris-Parameter, Response [VERIFIED via WebFetch]
- `https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide` — Unified Library Endpoints, uris-Array-Format [CITED]
- `https://github.com/LMS-Community/plugin-Qobuz/blob/master/Plugin.pm` — trackInfoMenu, QobuzManageFavorites, QobuzAddFavorite/DeleteFavorite vollständige Implementierung [VERIFIED via WebFetch]
- `https://github.com/LMS-Community/slimserver/.../SimpleHTTP/Base.pm` — put/post/get/delete Signaturen, Body-Handling [VERIFIED via WebFetch]
- `/home/sti/spoton/Plugins/SpotOn/API/Client.pm` — _request, _cacheTTL, Dual-Token [VERIFIED: Codebase]
- `/home/sti/spoton/Plugins/SpotOn/Plugin.pm` — initPlugin, _trackItem, passthrough-Pattern [VERIFIED: Codebase]

### Secondary (MEDIUM confidence)
- `https://developer.spotify.com/documentation/web-api/reference/save-library-items` — PUT /me/library Query-Param-Format [CITED]
- `https://developer.spotify.com/documentation/web-api/reference/remove-library-items` — DELETE /me/library Query-Param-Format [CITED]

### Tertiary (LOW confidence)
- WebSearch: Spotify unified library endpoint uris format — Bestätigung, dass uris-Param verwendet wird [nicht unabhängig verifiziert via offizielle Referenz-Doku]

---

## Metadata

**Confidence breakdown:**
- trackInfoMenu-Pattern: HIGH — vollständige Qobuz-Implementierung verifiziert
- API-Endpoints: HIGH für GET /me/library/contains (Doku direkt), MEDIUM für PUT/DELETE (Migration Guide + Suche)
- Scope-Handling: HIGH — Binary-Source direkt gelesen
- SimpleAsyncHTTP DELETE: HIGH — Base-Class-Quelle verifiziert
- JSON-Body-Handling: HIGH — Base-Class-Quelle verifiziert; ABER Entscheidung für Query-Params macht das irrelevant

**Research date:** 2026-06-11
**Valid until:** 2026-09-11 (stabile Spotify-Endpoints, stabile LMS-API)
