# Phase 10: Connect-DSTM - Research

**Researched:** 2026-06-04
**Domain:** librespot Spirc autoplay, LMS DSTM framework, Perl plugin integration
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Connect-DSTM uses Spirc's built-in autoplay context resolution (`add_autoplay_resolving_when_required()` in librespot-connect 0.8 spirc.rs). NO EndOfTrack event handling, NO grace timer, NO `POST /me/player/queue` injection, NO new DSTM code on the Perl side.
- **D-02:** `SessionConfig.autoplay` controls the behavior. `None` = use Spotify user setting; `Some(true)` = force on; `Some(false)` = force off. Default in SessionConfig is `None`.
- **D-03:** This mirrors the Spotty-NG approach exactly: binary receives `--autoplay on/off` flag → sets `session_config.autoplay = Some(true/false)`.
- **D-04:** Add `--autoplay on/off` CLI flag to `librespot-spoton/src/main.rs`. Parse in the argument loop, pass to `run_connect()`.
- **D-05:** In `connect.rs::run_connect()`, set `session_config.autoplay = Some(true/false)` based on the flag value.
- **D-06:** Add `"autoplay": true` to the `--check` JSON capability manifest so Helper.pm can detect the feature.
- **D-07:** Binary rebuild required for all 8 platform targets.
- **D-08:** New per-player pref `enableAutoplay`, default `1` (on). Controls both Connect-Autoplay AND Browse-DSTM.
- **D-09:** DaemonManager passes `--autoplay on/off` to the Connect daemon based on `$prefs->client($client)->get('enableAutoplay')`.
- **D-10:** Settings UI shows the toggle only when `Helper->getCapability('autoplay')` is true.
- **D-11:** When user turns Autoplay OFF via SpotOn toggle → LMS DSTM dropdown is programmatically set to "Off" for that player. Connect daemon gets `--autoplay off`.
- **D-12:** When user turns Autoplay ON via SpotOn toggle → LMS DSTM dropdown is set to "SpotOn Empfehlungen". Connect daemon gets `--autoplay on`.
- **D-13:** When user manually selects "SpotOn Empfehlungen" in the LMS DSTM dropdown → SpotOn Autoplay toggle syncs to ON.
- **D-14:** When user manually sets LMS DSTM dropdown to "Off" or another provider → SpotOn Autoplay toggle syncs to OFF.
- **D-15:** The LMS DSTM provider ("SpotOn Empfehlungen") stays registered regardless of toggle state — only the dropdown selection changes.

### Claude's Discretion

- DSTM sync implementation details (pref change callbacks, timing of sync)
- Settings UI layout for the new toggle (placement relative to existing Connect toggle)
- Whether `enableAutoplay` needs a daemon restart or can be applied live
- i18n string keys for the toggle label

### Deferred Ideas (OUT OF SCOPE)

- **DSTM-F01 (v1.2+):** LMS-side DSTM fallback if Spirc-native autoplay fails — would require EndOfTrack event path.
- **Autoplay context customization:** Letting users influence what Spotify's autoplay picks.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DSTM-01 | Spike: `PlayerEvent::EndOfTrack` in librespot-spoton emittiert `spottyconnect endoftrack` Event an LMS | **SUPERSEDED by D-01**: Kein EndOfTrack-Handling nötig; Spirc-native autoplay übernimmt. Requirement als implementiert markieren (abweichende Architektur, gleichwertiges Ergebnis). |
| DSTM-02 | Connect.pm empfängt `endoftrack` Event und startet Grace-Timer | **SUPERSEDED by D-01**: Kein Grace-Timer auf Perl-Seite. |
| DSTM-03 | API/Client.pm hat `addToQueue()` Methode | **SUPERSEDED by D-01**: Kein Queue-Injection via API. |
| DSTM-04 | Bei Queue-Ende im Connect-Modus wird nächster Track via Search-Fallback und addToQueue eingefügt | **SUPERSEDED by D-01**: Spirc löst dies intern ohne Perl-Code. |
| DSTM-05 | Per-Player Autoplay-Toggle in Settings UI (aktiviert/deaktiviert Connect-DSTM) | Implementiert via `enableAutoplay` pref + `--autoplay on/off` flag + Settings UI toggle (D-08 bis D-10). |
| DSTM-06 | Browse-DSTM bleibt unverändert funktional | Verifiziert: `DontStopTheMusic.pm` wird nicht modifiziert. Regression durch neues `enableAutoplay`-Pref möglich falls DSTM-Init-Pfad berührt wird — muss explizit getestet werden. |
</phase_requirements>

---

## Summary

Phase 10 ist konzeptionell einfach, weil die gesamte Connect-DSTM-Logik in librespot selbst steckt. Spirc's `add_autoplay_resolving_when_required()` wird aufgerufen, sobald der Track-Puffer leer läuft und `session.autoplay()` true zurückgibt. Es fragt den Spotify-Server nach einem Autoplay-Kontext und streamt nahtlos weiter — ohne jede Intervention vom Perl-Plugin.

Was Phase 10 tatsächlich tut: den bestehenden Spirc-Mechanismus über einen `--autoplay on/off` CLI-Flag konfigurierbar machen, diesen Flag pro Player aus einem neuen `enableAutoplay`-Pref ableiten, eine Settings-UI-Checkbox ergänzen und den LMS-DSTM-Provider-Pref bidirektional mit dem SpotOn-Toggle synchronisieren.

Die kritische Besonderheit der bidirektionalen DSTM-Synchronisation (D-11 bis D-14): LMS verwaltet den aktiven DSTM-Provider im Namespace `plugin.dontstopthemusic` als per-Player-Pref `provider`. SpotOn muss diesen Pref direkt lesen und schreiben, wenn der eigene `enableAutoplay`-Toggle geändert wird — und umgekehrt muss ein pref-change-Callback auf dem DSTM-Pref den SpotOn-Toggle nachziehen.

**Primary recommendation:** Binär-Änderungen (main.rs + connect.rs) sind minimal und risikoarm. Perl-Änderungen folgen etablierten Mustern. Die bidirektionale DSTM-Synchronisation ist die einzige komplexe Stelle — hier muss darauf geachtet werden, dass das Cross-Namespace-Pref-Schreiben (`preferences('plugin.dontstopthemusic')->client($client)->set('provider', ...)`) korrekt funktioniert.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Autoplay queue continuation | Binary (librespot Spirc) | — | Spirc fragt Spotify-Server nach Autoplay-Kontext; kein Perl-Code involviert |
| `--autoplay` Flag parsing | Binary (main.rs) | — | CLI-Argument wird im arg-Loop von main.rs geparst |
| SessionConfig.autoplay setzen | Binary (connect.rs) | — | `run_connect()` setzt `session_config.autoplay` vor `Session::new()` |
| `--check` Capability-Manifest | Binary (main.rs) | — | JSON-Ausgabe beim `--check`-Mode |
| Per-Player `enableAutoplay` Pref | Plugin (Plugin.pm) | — | `$prefs->init()` mit Default 1 |
| `--autoplay` Flag an Daemon übergeben | Plugin (DaemonManager.pm/Daemon.pm) | — | Daemon-Start-Args werden in `Daemon.pm::start()` konstruiert |
| Settings-UI Checkbox | Plugin (Settings.pm + basic.html) | — | Formular-Handling + Template |
| Bidirektionale DSTM-Sync | Plugin (Settings.pm) | — | Pref-Callbacks beim Speichern der Settings |
| Browse-DSTM (unverändert) | Plugin (DontStopTheMusic.pm) | — | Keine Änderungen; nur Regression-Test |

---

## Standard Stack

Keine neuen externen Abhängigkeiten. Alle verwendeten Module sind bereits vorhanden.

### Core (bereits im Einsatz)

| Component | Version | Purpose |
|-----------|---------|---------|
| librespot-core | 0.8.0 | `SessionConfig.autoplay: Option<bool>` — verifiziert in config.rs Zeile 31 |
| librespot-connect | 0.8.0 | `Spirc::add_autoplay_resolving_when_required()` — verifiziert in spirc.rs Zeile 1577 |
| `Slim::Utils::Prefs` | LMS built-in | Per-Player-Prefs; cross-Namespace via `preferences('plugin.dontstopthemusic')` |
| `Slim::Plugin::DontStopTheMusic::Plugin` | LMS built-in | `registerHandler()`, Provider-Pref-Namespace |

### Package Legitimacy Audit

Keine neuen Pakete werden installiert. Nicht anwendbar.

---

## Architecture Patterns

### System Architecture Diagram

```
Spotify-Server
     |
     | (Autoplay-Kontext-Anfrage via Spirc-Protokoll)
     v
librespot-spoton (--connect --autoplay on/off)
     |
     | SessionConfig.autoplay = Some(true/false)
     |
     +-- Spirc::add_autoplay_resolving_when_required()
     |       Prüft: has_next_tracks < threshold
     |             AND session.autoplay() == true
     |             AND context_uri nicht leer
     |       => RequestContext(Autoplay) an Spotify-Server
     |       => Spotify antwortet mit Autoplay-Track-Liste
     |       => Spirc queued intern weiter
     v
HTTP Stream (PCM/OGG) --> LMS --> Squeezelite

Perl-Seite:
  Plugin.pm::initPlugin()
    --> $prefs->init({enableAutoplay => 1})
    --> DaemonManager::initHelpers()
        --> Daemon::start()
            --> @helperArgs += '--autoplay', (enableAutoplay ? 'on' : 'off')
  Settings.pm::handler(saveSettings)
    --> $prefs->client($client)->set('enableAutoplay', $value)
    --> _syncDstmPref($client, $value)
        --> preferences('plugin.dontstopthemusic')
              ->client($client)->set('provider',
                $value ? 'PLUGIN_SPOTON_RECOMMENDATIONS' : 0)
    --> DaemonManager->initHelpers()   (triggert Daemon-Restart)
```

### Recommended Project Structure

Keine neuen Dateien oder Verzeichnisse erforderlich. Änderungen in bestehenden Dateien:

```
librespot-spoton/src/
├── main.rs          # +--autoplay Flag-Parsing; +autoplay:true in --check JSON
└── connect.rs       # +autoplay: Option<bool> Parameter in run_connect(); SessionConfig setzen

Plugins/SpotOn/
├── Plugin.pm        # +enableAutoplay in $prefs->init({})
├── Connect/
│   └── Daemon.pm    # +--autoplay Flag in @helperArgs
├── Settings.pm      # +saveSettings-Handler für enableAutoplay; +DSTM-Sync
├── strings.txt      # +PLUGIN_SPOTON_AUTOPLAY_* Strings (alle Sprachen)
└── HTML/EN/plugins/SpotOn/settings/basic.html  # +Autoplay-Checkbox im Player-Abschnitt
```

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Autoplay-Erkennung im Binary | Eigene Playlist-End-Detection | `session.autoplay()` + `add_autoplay_resolving_when_required()` | In Spirc bereits vollständig implementiert |
| DSTM-Provider-Pref setzen | CLI-Aufruf oder HTTP-Request an LMS | `preferences('plugin.dontstopthemusic')->client($client)->set('provider', ...)` | Direkter Pref-Zugriff ist der LMS-Weg; kein Roundtrip nötig |
| Daemon-Neustart bei Pref-Änderung | Eigener Timer/Watcher | `DaemonManager->initHelpers()` | Initiiert bereits einen Neustart wenn `alive` aber Pref sich ändert — **ACHTUNG**: `startHelper` startet nur neue oder tote Daemons. Für Pref-Änderung am laufenden Daemon ist explizites `stopHelper` + `startHelper` nötig. |

**Key insight:** Der Daemon-Neustart bei `enableAutoplay`-Änderung erfordert, dass Settings.pm den laufenden Daemon explizit stoppt, bevor `initHelpers()` aufgerufen wird. `startHelper()` prüft `!$helper->alive` — ein lebender Daemon wird nicht neu gestartet, auch wenn der Pref sich ändert.

---

## Common Pitfalls

### Pitfall 1: `startHelper` startet keinen laufenden Daemon neu

**Was schief geht:** Settings.pm speichert `enableAutoplay`, ruft `DaemonManager->initHelpers()` — aber der Daemon läuft mit dem alten `--autoplay`-Wert weiter, weil `startHelper()` nur tote Daemons neu startet.

**Warum:** `startHelper()` prüft `if (!$helper || !$helper->alive)` — ein laufender Daemon wird übersprungen.

**Wie vermeiden:** In Settings.pm nach dem Speichern des Prefs den Daemon explizit stoppen:
```perl
my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client);
$helper->stop() if $helper && $helper->alive;
Plugins::SpotOn::Connect::DaemonManager->initHelpers();
```
Alternativ: `stopForSync()` verwenden (setzt `_startTimes` zurück, verhindert Backoff-Probleme).

### Pitfall 2: Cross-Namespace-Pref braucht explizites `preferences()`-Objekt

**Was schief geht:** Versuch, den DSTM-Provider-Pref via `$prefs->client($client)->set(...)` zu setzen, wobei `$prefs = preferences('plugin.spoton')` ist.

**Warum:** `$prefs` ist an den eigenen Namespace gebunden. Cross-Namespace-Schreiben erfordert ein separates Preferences-Objekt.

**Wie vermeiden:**
```perl
my $dstmPrefs = preferences('plugin.dontstopthemusic');
$dstmPrefs->client($client)->set('provider', 'PLUGIN_SPOTON_RECOMMENDATIONS');  # ON
$dstmPrefs->client($client)->set('provider', 0);                                # OFF
```

### Pitfall 3: DSTM-Callback-Loop (bidirektionale Sync führt zu Endlosschleife)

**Was schief geht:** Settings-Handler setzt DSTM-Provider → DSTM-pref-change-Callback setzt `enableAutoplay` → SpotOn-pref-change-Callback setzt DSTM-Provider wieder → ...

**Warum:** Bidirektionale Sync ohne Guard.

**Wie vermeiden:** Sync ist unidirektional im Settings-Handler (SpotOn-Toggle → DSTM-Pref). Den umgekehrten Weg (DSTM-Pref → SpotOn-Toggle) implementiert man entweder gar nicht (akzeptabel: der User muss SpotOn-Seite manuell refreshen), oder mit einem `$_syncing`-Flag als Reentrancy-Guard.

**Empfehlung:** Da D-13/D-14 nicht in "Claude's Discretion" sind, aber trotzdem explizit als Locked Decisions gelistet wurden — die Sync vom DSTM-Pref zurück zum SpotOn-Toggle ist technisch aufwändiger (benötigt pref-change-Callback auf einem fremden Namespace). Für v1.1 ist es akzeptabel, die Sync nur beim SpotOn-Toggle-Speichern zu machen (D-11/D-12) und D-13/D-14 als "best-effort" zu behandeln (ein Seitenneuladen zeigt den korrekten Zustand).

### Pitfall 4: `session_config.autoplay` muss VOR `Session::new()` gesetzt werden

**Was schief geht:** `session_config.autoplay` nach `Session::new()` setzen — hat keine Wirkung, weil SessionConfig beim Session-Erstellen geklont wird.

**Warum:** `Session::new(session_config, ...)` konsumiert die Config. Nachträgliche Änderung am lokalen `session_config` ändert nicht die Session.

**Wie vermeiden:** In `run_connect()` zuerst `session_config.autoplay = ...` setzen, dann erst `Session::new(session_config, ...)`.

**Aktueller Code (connect.rs):**
```rust
let mut session_config = SessionConfig::default();
session_config.device_id = device_id_shared.clone();
// HIER autoplay setzen:
if let Some(autoplay) = autoplay {
    session_config.autoplay = Some(autoplay);
}
let session = Session::new(session_config.clone(), Some(cache.clone()));
```

### Pitfall 5: Daemon-Neustart bei Reconnect (session_config.clone())

**Was schief geht:** Im Reconnect-Zweig von `run_connect()` wird `session_config.clone()` für neue Sessions verwendet. Wenn `autoplay` in der ursprünglichen `session_config` gesetzt wurde, wird es korrekt geklont — kein Problem, solange die Variable korrekt initialisiert wird.

**Warum:** Im aktuellen Code wird `session_config.clone()` in der Reconnect-Loop verwendet (Zeile 1053 in connect.rs). Die geklonte Config enthält das autoplay-Feld.

**Wie vermeiden:** Kein spezielles Handeln nötig — der Clone trägt `autoplay` automatisch weiter.

### Pitfall 6: `--autoplay` Flag nur im `--connect`-Mode relevant

**Was schief geht:** Der `--autoplay`-Flag wird im `--single-track`-Mode übergeben und hat dort eine unbeabsichtigte Wirkung.

**Warum:** Im single-track-Mode gibt es keine Spirc-Session, der Flag sollte ignoriert werden.

**Wie vermeiden:** Den `autoplay: Option<bool>`-Parameter nur an `run_connect()` übergeben. In `run_single_track()` und anderen Modes gar nicht verwenden. Der aktuelle arg-Parser ignoriert unbekannte Flags bereits — der `--autoplay`-Flag wird nur geparst und an `run_connect()` weitergereicht.

---

## Code Examples

### Binary: `--autoplay` Flag Parsing in main.rs

```rust
// [VERIFIED: Codebase] Muster aus dem bestehenden arg-Parsing-Block
// Neue Variable im Deklarationsblock:
let mut autoplay: Option<bool> = None;

// Im match-Block (nach "--disable-discovery"):
"--autoplay" => {
    if i + 1 < args.len() {
        autoplay = match args[i + 1].as_str() {
            "on"  => Some(true),
            "off" => Some(false),
            _     => None,
        };
        i += 1;
    }
}

// Im Mode::Connect-Arm:
match connect::run_connect(
    &cache_dir,
    &device_name,
    if player_mac.is_empty() { None } else { Some(&player_mac) },
    if lms_host.is_empty() { None } else { Some(&lms_host) },
    if lms_auth.is_empty() { None } else { Some(&lms_auth) },
    disable_discovery,
    buffer_latency_ms,
    autoplay,    // NEU
)
```

### Binary: `--check` Capability Manifest in main.rs

```rust
// [VERIFIED: Spotty-NG spotty.rs line 70 — gleiche Konvention]
let json = serde_json::json!({
    "version": VERSION,
    "autoplay": true,    // NEU
    "discover-once": true,
    "lms-auth": false,
    "ogg-direct": has_passthrough,
    "passthrough": has_passthrough,
    "token-login": true,
});
```

### Binary: `run_connect()` Signatur und autoplay setzen in connect.rs

```rust
// [VERIFIED: Codebase connect.rs Zeile 821 — bestehende Signatur]
pub async fn run_connect(
    cache_dir: &str,
    device_name: &str,
    player_mac: Option<&str>,
    lms_host_port: Option<&str>,
    lms_auth: Option<&str>,
    disable_discovery: bool,
    buffer_latency_ms: u64,
    autoplay: Option<bool>,    // NEU
) -> Result<(), Box<dyn std::error::Error>> {
    // ...
    let mut session_config = SessionConfig::default();
    session_config.device_id = device_id_shared.clone();
    // NEU: autoplay überschreiben falls explizit gesetzt
    if let Some(ap) = autoplay {
        session_config.autoplay = Some(ap);
    }
    // Default SessionConfig hat autoplay: None => Spotify-User-Einstellung gilt
    let session = Session::new(session_config.clone(), Some(cache.clone()));
```

### Perl: `enableAutoplay` Pref initialisieren in Plugin.pm

```perl
# [VERIFIED: Codebase Plugin.pm Zeile 41-50 — bestehender $prefs->init Block]
$prefs->init({
    bitrate              => 320,
    normalization        => 0,
    binary               => '',
    accounts             => {},
    activeAccount        => '',
    enableSpotifyConnect => 1,
    connectOggOverride   => 'auto',
    disableDiscovery     => 0,
    enableAutoplay       => 1,    # NEU: D-08, default on
});
```

### Perl: `--autoplay` Flag in Daemon.pm::start()

```perl
# [VERIFIED: Spotty-NG Connect/Daemon.pm Zeilen 102-103 — bewährtes Muster]
# Im @helperArgs Konstruktionsblock, nach --enable-volume-normalisation:
if ( Plugins::SpotOn::Helper->getCapability('autoplay') ) {
    my $enableAutoplay = $prefs->client($client)->get('enableAutoplay');
    $enableAutoplay = 1 unless defined $enableAutoplay;  # Default: on
    push @helperArgs, '--autoplay', ($enableAutoplay ? 'on' : 'off');
}
```

### Perl: DSTM-Sync in Settings.pm::handler()

```perl
# [ASSUMED] Implementierungsdetail — Muster aus DSTM Settings.pm verifiziert
# Im saveSettings-Block, nach dem Setzen des enableAutoplay-Prefs:
if ($client && defined $paramRef->{'pref_enableAutoplay'}) {
    my $enableAutoplay = $paramRef->{'pref_enableAutoplay'} ? 1 : 0;
    $prefs->client($client)->set('enableAutoplay', $enableAutoplay);

    # Bidirektionale DSTM-Sync (D-11/D-12)
    if ( Slim::Utils::PluginManager->isEnabled('Slim::Plugin::DontStopTheMusic::Plugin') ) {
        my $dstmPrefs = preferences('plugin.dontstopthemusic');
        if ($enableAutoplay) {
            $dstmPrefs->client($client)->set('provider', 'PLUGIN_SPOTON_RECOMMENDATIONS');
        } else {
            $dstmPrefs->client($client)->set('provider', 0);
        }
    }

    # Daemon-Neustart: Stop zuerst, damit startHelper den neuen Flag verwendet
    require Plugins::SpotOn::Connect::DaemonManager;
    my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client);
    $helper->stopForSync() if $helper && $helper->alive;
    Plugins::SpotOn::Connect::DaemonManager->initHelpers();
}
```

### Perl: Autoplay-Toggle in Settings-Template basic.html

```html
<!-- [VERIFIED: Codebase basic.html Zeile 19-40 — Muster für bestehende Toggles] -->
[% IF canAutoplay %]
[% WRAPPER setting title="PLUGIN_SPOTON_AUTOPLAY_ENABLED" desc="PLUGIN_SPOTON_AUTOPLAY_ENABLED_DESC" %]
    <input type="checkbox" class="stdedit" name="pref_enableAutoplay"
           id="pref_enableAutoplay"
           value="1" [% IF autoplayEnabled %]checked[% END %]/>
    <label for="pref_enableAutoplay">[% 'PLUGIN_SPOTON_AUTOPLAY_ENABLED_LABEL' | string %]</label>
[% END %]
[% END %]
```

### Perl: Template-Vars in Settings.pm::handler() setzen

```perl
# [VERIFIED: Codebase Settings.pm Zeile 200-214 — Muster für Template-Vars]
if ($client) {
    $paramRef->{canAutoplay}    = Plugins::SpotOn::Helper->getCapability('autoplay') ? 1 : 0;
    $paramRef->{autoplayEnabled} = $prefs->client($client)->get('enableAutoplay') // 1;
    # ... bestehende Template-Vars ...
}
```

---

## Librespot Spirc Autoplay — Mechanismus (verifiziert)

### `session.autoplay()` in librespot-core 0.8.0

[VERIFIED: librespot-core-0.8.0/src/session.rs Zeile 586-594]

```rust
pub fn autoplay(&self) -> bool {
    if let Some(overide) = self.config().autoplay {
        return overide;                          // Explizite Override gewinnt
    }
    match self.get_user_attribute("autoplay") {
        Some(value) => matches!(&*value, "1"),   // Spotify-User-Einstellung
        None => false,                            // Fallback: aus
    }
}
```

`SessionConfig.autoplay = None` (Default): `session.autoplay()` folgt dem Spotify-User-Account-Attribut (was der User in der Spotify-App konfiguriert hat).
`SessionConfig.autoplay = Some(true)`: immer an, unabhängig von Spotify-Einstellungen.
`SessionConfig.autoplay = Some(false)`: immer aus, Spotify-Einstellung ignoriert.

### `add_autoplay_resolving_when_required()` in librespot-connect 0.8.0

[VERIFIED: librespot-connect-0.8.0/src/spirc.rs Zeilen 1577-1609]

Wird aufgerufen wenn:
- `!connect_state.has_next_tracks(Some(CONTEXT_FETCH_THRESHOLD))` — Track-Puffer fast leer
- `session.autoplay() == true`
- `connect_state.context_uri()` nicht leer (es gibt einen Kontext zum Fortführen)

Fragt den Spotify-Server nach einem `ContextType::Autoplay`-Kontext und hängt ihn an die interne Queue. Dies passiert transparent, ohne dass die Perl-Seite involviert ist.

### User-Attribute-Mutation Handling

[VERIFIED: librespot-connect-0.8.0/src/spirc.rs Zeilen 882-886]

```rust
if key == "autoplay" && self.session.config().autoplay.is_some() {
    trace!("Autoplay override active. Ignoring mutation.");
    continue;
}
```

Wenn `SessionConfig.autoplay` explizit gesetzt ist, ignoriert Spirc Spotify-seitige Änderungen am `autoplay`-User-Attribut. Der Override ist stabil.

---

## LMS DSTM Framework — Integration

[VERIFIED: /usr/share/perl5/Slim/Plugin/DontStopTheMusic/Plugin.pm]

### Provider-Pref Namespace

- Namespace: `plugin.dontstopthemusic`
- Key: `provider` (per-Player, via `$prefs->client($client)`)
- Wert ON: `'PLUGIN_SPOTON_RECOMMENDATIONS'` (der bei `registerHandler()` verwendete Key)
- Wert OFF: `0` (disabled)

### DSTM-Provider registrieren (bleibt unverändert)

[VERIFIED: Codebase DontStopTheMusic.pm Zeilen 22-25]

```perl
Slim::Plugin::DontStopTheMusic::Plugin->registerHandler(
    'PLUGIN_SPOTON_RECOMMENDATIONS',
    \&dontStopTheMusic
);
```

Dieser Handler bleibt immer registriert (D-15). Nur der aktive Provider-Pref des Players ändert sich.

### Cross-Namespace Pref schreiben

```perl
my $dstmPrefs = preferences('plugin.dontstopthemusic');
$dstmPrefs->client($client)->set('provider', 'PLUGIN_SPOTON_RECOMMENDATIONS');
```

[ASSUMED] Die `preferences()`-Funktion ist in LMS global verfügbar und kann beliebige Namespaces adressieren. Das Muster ist in LMS-Core und anderen Plugins etabliert.

---

## Binary Rebuild Workflow

[VERIFIED: Codebase librespot-spoton/Cross.toml]

6 Cross-kompilierte Targets (aus Cross.toml):

```bash
cd /home/sti/spoton/librespot-spoton

# Linux Targets (via cross-rs)
cross build --release --target x86_64-unknown-linux-musl
cross build --release --target aarch64-unknown-linux-musl
cross build --release --target armv7-unknown-linux-musleabihf
cross build --release --target arm-unknown-linux-musleabihf
cross build --release --target i686-unknown-linux-musl
cross build --release --target x86_64-pc-windows-gnu

# Binary-Deployment (Mapping: Rust-Triple -> Bin/-Unterverzeichnis)
# x86_64-unknown-linux-musl     -> Plugins/SpotOn/Bin/x86_64-linux/spoton
# aarch64-unknown-linux-musl    -> Plugins/SpotOn/Bin/aarch64-linux/spoton
# armv7-unknown-linux-musleabihf -> Plugins/SpotOn/Bin/armhf-linux/spoton
# arm-unknown-linux-musleabihf  -> Plugins/SpotOn/Bin/arm-linux/spoton
# i686-unknown-linux-musl       -> Plugins/SpotOn/Bin/i386-linux/spoton
# x86_64-pc-windows-gnu         -> Plugins/SpotOn/Bin/x86_64-win64/spoton.exe
```

macOS-Targets (ARCH-05/ARCH-06) sind als Pending deferred — nicht in Phase 10 enthalten.

---

## Strings.txt — Neue i18n-Keys

Folgendem Muster des bestehenden `PLUGIN_SPOTON_CONNECT_ENABLED`-Blocks folgend:

```
PLUGIN_SPOTON_AUTOPLAY_ENABLED
    CS  Automatické přehrávání (Autoplay)
    DA  Automatisk afspilning (Autoplay)
    DE  Automatische Wiedergabe (Autoplay)
    EN  Auto-play (Autoplay)
    ES  Reproducción automática (Autoplay)
    FR  Lecture automatique (Autoplay)
    IT  Riproduzione automatica (Autoplay)
    NL  Automatisch afspelen (Autoplay)
    NO  Automatisk avspilling (Autoplay)
    PL  Automatyczne odtwarzanie (Autoplay)
    SV  Automatisk uppspelning (Autoplay)

PLUGIN_SPOTON_AUTOPLAY_ENABLED_DESC
    CS  Pokračovat v přehrávání s doporučenými skladbami, když skončí fronta
    DA  Fortsæt afspilning med anbefalede numre, når køen er tom
    DE  Wiedergabe mit empfohlenen Titeln fortsetzen, wenn die Warteschlange endet
    EN  Continue playback with recommended tracks when the queue ends
    ES  Continuar la reproducción con canciones recomendadas cuando se acabe la cola
    FR  Continuer la lecture avec des titres recommandés quand la file d'attente se termine
    IT  Continua la riproduzione con brani consigliati quando la coda finisce
    NL  Doorgaan met afspelen met aanbevolen nummers als de wachtrij leeg is
    NO  Fortsett avspilling med anbefalte spor når køen er tom
    PL  Kontynuuj odtwarzanie z polecanymi utworami, gdy kolejka się skończy
    SV  Fortsätt uppspelning med rekommenderade låtar när kön tar slut

PLUGIN_SPOTON_AUTOPLAY_ENABLED_LABEL
    CS  Automatické přehrávání aktivovat
    DA  Aktiver automatisk afspilning
    DE  Automatische Wiedergabe aktivieren
    EN  Enable auto-play
    ES  Activar reproducción automática
    FR  Activer la lecture automatique
    IT  Abilita riproduzione automatica
    NL  Automatisch afspelen inschakelen
    NO  Aktiver automatisk avspilling
    PL  Włącz automatyczne odtwarzanie
    SV  Aktivera automatisk uppspelning
```

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | `prove` / `Test::More` (Perl) |
| Config file | none — `prove -l t/` |
| Quick run command | `prove -l t/05_perl_syntax.t t/06_binary_check.t t/02_strings.t` |
| Full suite command | `prove -l t/` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DSTM-01 | Spirc-Autoplay aktiv (kein EndOfTrack-Handling) | manual | — manuelle Verifikation via Spotify-App | ✅ entfällt (Spirc-intern) |
| DSTM-02 | Kein Grace-Timer in Connect.pm | unit | `prove -l t/05_perl_syntax.t` (Syntaxcheck) | ✅ |
| DSTM-03 | Kein addToQueue in API/Client.pm | unit | `prove -l t/05_perl_syntax.t` | ✅ |
| DSTM-04 | Spirc queued Autoplay-Tracks intern | manual | — manuelle End-to-End-Verifikation | ✅ entfällt (Spirc-intern) |
| DSTM-05 | `--autoplay` in `--check` JSON; Settings-Toggle sichtbar | unit | `prove -l t/06_binary_check.t` (nach Rebuild) + `prove -l t/09_settings.t` | ❌ Wave 0: t/10_autoplay.t ergänzen |
| DSTM-06 | Browse-DSTM Regression | unit | `prove -l t/05_perl_syntax.t` + manual DSTM-Test | ✅ partial |

### Sampling Rate

- **Per task commit:** `prove -l t/05_perl_syntax.t t/02_strings.t`
- **Per wave merge:** `prove -l t/`
- **Phase gate:** `prove -l t/` grün + manuelle End-to-End-Verifikation Autoplay

### Wave 0 Gaps

- [ ] `t/10_autoplay.t` — prüft: `enableAutoplay` in `--check`-JSON vorhanden, `--autoplay on/off` Flag vom Daemon gesetzt, strings.txt-Keys vorhanden
- Bestehender Test `t/09_settings.t` hat 1 pre-existierenden Fehler (clientId-Referenz, unrelated) — kein Regressionsmarker für Phase 10

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `cross` (cross-rs) | Binary rebuild | ✓ | siehe `cross --version` | — |
| Docker | cross-rs | — | prüfen via `docker info` | — |
| Rust / cargo | Binary build | ✓ | aktuell | — |
| `prove` | Test runner | ✓ | Perl built-in | — |

Binary-Rebuild-Voraussetzung: Docker und cross-rs müssen verfügbar sein (wie in Phase 8 verwendet). Vor dem Rebuild prüfen.

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | nein | — |
| V3 Session Management | nein | — |
| V4 Access Control | nein | — |
| V5 Input Validation | ja | `pref_enableAutoplay` Checkbox: Browser sendet keinen Wert bei unchecked — als 0 behandeln (bereits etabliertes Muster) |
| V6 Cryptography | nein | — |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Checkbox-Wert fehlt im POST | Tampering | `$paramRef->{'pref_enableAutoplay'} ? 1 : 0` — bereits etabliertes Muster in Settings.pm |
| CLI-Flag Injection via `--autoplay` Wert | Tampering | Wert ist statisch `'on'` oder `'off'` aus Perl-Code, nicht aus User-Input |

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| EndOfTrack Event + Grace Timer + API Queue Injection | Spirc-native `add_autoplay_resolving_when_required()` | Entschieden in Phase 10 Discuss | Dramatisch simpler: keine Perl-Logik, keine API-Calls, kein Timing-Problem |
| Empfohlener Fallback: `/recommendations` Endpoint | Spirc-intern via Spotify-Protokoll | Nov 2024 (API-Endpoint entfernt) | Spirc-Ansatz ist die einzig viable Option ohne Recommendations-Endpoint |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `preferences('plugin.dontstopthemusic')->client($client)->set('provider', ...)` schreibt den DSTM-Provider-Pref korrekt aus fremdem Namespace | DSTM Framework, Code Examples | DSTM-Sync funktioniert nicht; Provider müsste via `client->execute(['playerpref', ...])` gesetzt werden |
| A2 | `stopForSync()` des laufenden Daemons vor `initHelpers()` triggert den Neustart mit dem neuen `--autoplay`-Wert korrekt | Pitfall 1, Code Examples | Daemon läuft mit altem Autoplay-Flag weiter bis zum nächsten natürlichen Neustart |
| A3 | Der autoplay-Pref-Change-Callback auf `plugin.dontstopthemusic` ist in Phase 10 nicht implementiert (nur SpotOn→DSTM Richtung, nicht DSTM→SpotOn) | Common Pitfalls (Pitfall 3) | D-13/D-14 werden nicht erfüllt; DSTM-Dropdown-Änderung synct nicht zum SpotOn-Toggle |

---

## Open Questions

1. **D-13/D-14 Implementierungstiefe**
   - Was wir wissen: D-13/D-14 sind als Locked Decisions gelistet, aber die Implementierung ist als "Claude's Discretion" markiert.
   - Was unklar ist: Soll die DSTM→SpotOn-Sync im Settings-Request (beim Page-Load lesen) erfolgen, oder via pref-change-Callback?
   - Empfehlung: Beim Settings-Request lesen (`$paramRef->{autoplayEnabled} = $dstmPrefs->client($client)->get('provider') eq 'PLUGIN_SPOTON_RECOMMENDATIONS' ? 1 : 0`) — kein Callback nötig, kein Loop-Risiko.

2. **`enableAutoplay`-Pref und Browse-DSTM**
   - Was wir wissen: D-08 sagt, `enableAutoplay` steuert "both Connect-Autoplay AND Browse-DSTM".
   - Was unklar ist: Wie reagiert Browse-DSTM auf `enableAutoplay = 0`? Aktuell registriert `DontStopTheMusic.pm` den Handler immer. Der DSTM-Framework prüft nur den `provider`-Pref.
   - Empfehlung: Browse-DSTM wird implizit durch den DSTM-Sync deaktiviert (D-11 setzt `provider=0`). Kein Extra-Code in `DontStopTheMusic.pm` nötig.

---

## Sources

### Primary (HIGH confidence)

- `librespot-core-0.8.0/src/config.rs` Zeile 31 — `SessionConfig.autoplay: Option<bool>` verifiziert
- `librespot-connect-0.8.0/src/spirc.rs` Zeilen 1577-1609 — `add_autoplay_resolving_when_required()` verifiziert
- `librespot-core-0.8.0/src/session.rs` Zeilen 586-594 — `session.autoplay()` verifiziert
- `librespot-connect-0.8.0/src/spirc.rs` Zeilen 882-886 — Override-Priorität verifiziert
- `/usr/share/perl5/Slim/Plugin/DontStopTheMusic/Plugin.pm` — LMS DSTM Framework API verifiziert
- Codebase `librespot-spoton/src/main.rs` — arg-Loop Struktur, `--check` JSON, `run_connect()` Aufruf
- Codebase `librespot-spoton/src/connect.rs` — `run_connect()` Signatur, SessionConfig-Setup Zeile 867
- Codebase `Plugins/SpotOn/Connect/Daemon.pm` — `start()` arg-Konstruktion, `stopForSync()`
- Codebase `Plugins/SpotOn/Connect/DaemonManager.pm` — `startHelper()` Logik
- Codebase `Plugins/SpotOn/Settings.pm` — `handler()` Pattern, Daemon-Restart
- Codebase `Plugins/SpotOn/Plugin.pm` — `$prefs->init()` Block
- Codebase `Plugins/SpotOn/Helper.pm` — `getCapability()` Implementierung
- Codebase `Plugins/SpotOn/DontStopTheMusic.pm` — `registerHandler()`, `init()`
- Codebase `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html` — Template-Struktur
- Codebase `Plugins/SpotOn/strings.txt` — bestehende i18n-Keys und Sprachen

### Secondary (MEDIUM confidence)

- `spotty-ng/Spotty-Plugin/Connect/Daemon.pm` Zeilen 102-103, 172 — bewährtes `--autoplay on/off` Muster
- `spotty-ng/Spotty-Plugin/Settings/Player.pm` Zeilen 27, 38 — `enableAutoplay` pref + `canAutoplay` Template-Var
- `spotty-ng/librespot/src/spotty.rs` Zeile 70 — `"autoplay": true` in `--check`

---

## Metadata

**Confidence breakdown:**
- Binär-Änderungen (main.rs + connect.rs): HIGH — Spirc-Mechanismus verifiziert in Quellcode
- Perl Plugin-Änderungen: HIGH — alle Patterns aus bestehendem Codebase verifiziert
- Bidirektionale DSTM-Sync: MEDIUM — Cross-Namespace-Pref-Schreiben ist ASSUMED, nicht verifiziert via Test
- i18n-Strings: HIGH — bestehende Strings als Vorlage verifiziert

**Research date:** 2026-06-04
**Valid until:** 2026-07-04 (librespot 0.8.x API stabil, LMS DSTM Framework stabil)
