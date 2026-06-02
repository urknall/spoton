---
phase: 04-single-track-streaming
type: code-review
depth: standard
status: findings
files_reviewed: 5
files_reviewed_list:
  - Plugins/SpotOn/ProtocolHandler.pm
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/Settings.pm
  - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
  - Plugins/SpotOn/strings.txt
findings_critical: 3
findings_warning: 4
findings_info: 2
date: 2026-05-28
---

# Phase 04: Code Review Report

**Reviewed:** 2026-05-28T00:00:00Z
**Depth:** standard
**Files Reviewed:** 5
**Status:** issues_found

## Summary

Phase 04 implementiert das Single-Track-Streaming-Backend (ProtocolHandler, updateTranscodingTable), Kontext-Queueing via `playall`, stündlichen Orphaned-Process-Cleanup und ein Settings-UI mit Normalisierungs-Toggle. Die Grundarchitektur ist solide und folgt dem dokumentierten Spotty-Pattern. Drei kritische Defekte wurden gefunden: ein fehlender `require`-Import in ProtocolHandler.pm führt zu einem garantierten Laufzeitfehler bei OGG-Playback, ein nicht importiertes LMS-Versions-Modul kann unter bestimmten Bedingungen abstürzen, und der `pkill`-Aufruf enthält keine Shell-Absicherung des Pfadarguments. Vier Warnungen betreffen Template-Ausgabe-Escaping, doppelten Template-Block, eine private Methodenreferenz und eine stille Fehlfunktion beim Normalisierungs-Flag-Injekt.

---

## Critical Issues

### CR-01: Fehlender `require` für Helper in ProtocolHandler.pm — garantierter Laufzeitfehler bei OGG

**File:** `Plugins/SpotOn/ProtocolHandler.pm:41`

**Issue:** `formatOverride` ruft `Plugins::SpotOn::Helper->getCapability('passthrough')` auf (Zeile 41), aber ProtocolHandler.pm enthält weder `use Plugins::SpotOn::Helper` noch `require Plugins::SpotOn::Helper`. Der `require`-Aufruf für `Plugin` auf Zeile 32 stellt sicher, dass Plugin.pm geladen ist, aber Helper.pm wird nirgends in ProtocolHandler.pm importiert. Da Helper.pm von Plugin.pm über `require Plugins::SpotOn::Helper; Plugins::SpotOn::Helper->init()` in `initPlugin` geladen wird, ist das Modul zur Laufzeit *üblicherweise* im `%INC`-Cache vorhanden — aber nur wenn `initPlugin` vor dem ersten `formatOverride`-Aufruf ausgeführt wurde. Wenn LMS das Transcoding-Subsystem vor der vollständigen Plugin-Initialisierung aktiviert (z.B. nach Reload während laufenden Tracks), schlägt der Aufruf mit `Can't locate object method "getCapability" via package "Plugins::SpotOn::Helper"` fehl. Außerdem ist es eine Verletzung der expliziten Abhängigkeitsdeklaration: ein Modul darf nicht stillschweigend auf den Ladezustand eines anderen Moduls angewiesen sein.

**Fix:**
```perl
# In ProtocolHandler.pm, unmittelbar vor dem getCapability()-Aufruf
if (grep { $_ eq 'ogg' } @formats) {
    require Plugins::SpotOn::Helper;
    if (Plugins::SpotOn::Helper->getCapability('passthrough')) {
        return 'ogg';
    }
}
```

Alternativ `require Plugins::SpotOn::Helper;` oben in `formatOverride` einfügen, analog zum vorhandenen `require Plugins::SpotOn::Plugin;` auf Zeile 32.

---

### CR-02: `Slim::Utils::Versions` in ProtocolHandler.pm nicht importiert

**File:** `Plugins/SpotOn/ProtocolHandler.pm:51-52`

**Issue:** `canSeek` und `canTranscodeSeek` rufen `Slim::Utils::Versions->compareVersions(...)` auf, aber `Slim::Utils::Versions` wird in ProtocolHandler.pm weder per `use` noch per `require` importiert. In der LMS-Praxis ist dieses Modul für gewöhnlich schon geladen, da LMS es intern nutzt — der Code funktioniert damit in einer laufenden LMS-Instanz. Das Problem ist: (a) Es ist keine korrekte explizite Abhängigkeit, was künftige Refactoring-Fehler provoziert, und (b) bei Unit-Tests oder isolierten Ladeoperationen (z.B. Syntax-Checker ohne full LMS) führt das direkt zu einem Fehler. Da `canSeek` eine Performance-kritische Methode ist (wird pro Track ausgewertet), ist der fehlende Import ein Zuverlässigkeitsrisiko.

**Fix:**
```perl
# In der use-Sektion von ProtocolHandler.pm hinzufügen:
use Slim::Utils::Versions;
```

---

### CR-03: Shell-Injection-Risiko bei `pkill -f "$helper"` (unescapeter Pfad)

**File:** `Plugins/SpotOn/Plugin.pm:135`

**Issue:** Der Backtick-Befehl `` `pkill -f "$helper"` `` bettet `$helper` direkt in einen Shell-Befehl ein. Wenn `$helper` Shell-Sonderzeichen enthält (Leerzeichen, Klammern, Sternchen, Backticks, `$(...)` etc.), bricht der Befehl entweder ab oder führt unbeabsichtigte Shell-Operationen durch. Im Gegensatz dazu verwendet `Helper.pm::helperCheck()` korrekt single-quote-Escaping:

```perl
(my $safe = $candidate) =~ s/'/'\\''/g;
my $checkCmd = sprintf("'%s' -n 'SpotOn' --check", $safe);
```

Dieser Schutz fehlt in `_killOrphanedProcesses`. Die Bedrohung ist zwar eingeschränkt — `$helper` kommt aus `Helper->get()` welches das Binary über `_findBin` ermittelt —, aber Installationspfade mit Leerzeichen (z.B. `/home/user/My Apps/spoton`) sind plausibel und würden den `pkill`-Aufruf brechen (stille Fehlfunktion). Auf Systemen, bei denen der Helper-Pfad von außen beeinflusst werden kann, ist das ein Injektionsvektor.

Zusätzlich: Der Windows-Pfad `system("taskkill /IM $name /F 1>nul 2>&1")` hat das gleiche Problem — `$name` (via `basename($helper)`) wird ohne Quoting eingesetzt.

**Fix:**
```perl
# Unix: single-quote-Escaping wie in helperCheck()
(my $safeHelper = $helper) =~ s/'/'\\''/g;
`pkill -f '$safeHelper'`;

# Windows: Name in Anführungszeichen einschließen
my $name = basename($helper);
$name =~ s/"//g;    # doppelte Anführungszeichen aus dem Namen entfernen
system(qq{taskkill /IM "$name" /F 1>nul 2>&1});
```

---

## Warnings

### WR-01: Fehlender HTML-Escape bei `helperMissing` in basic.html

**File:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html:4`

**Issue:** `[% helperMissing %]` gibt den Wert ohne `| html`-Filter aus. Der Wert kommt aus `string('PLUGIN_SPOTON_BINARY_MISSING')` in Settings.pm, also aus einem bekannten, kontrollierten String — kein direkter XSS-Pfad. Aber alle anderen Ausgaben im Template nutzen konsistent `| html` (z.B. Zeilen 9, 33, 45). Das Template-Pattern sollte konsequent sein: jede Ausgabe einer Perl-Variable muss escapet werden, unabhängig davon, ob der Wert als sicher eingestuft wird. Das gilt umso mehr, weil Settings-Daten in zukünftigen Versionen aus anderen Quellen kommen könnten.

**Fix:**
```
[% helperMissing | html %]
```

---

### WR-02: Doppelter Binary-Status-Block in basic.html — redundante Fehlermeldung

**File:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html:3-13`

**Issue:** Die Template-Datei enthält zwei separate Blöcke, die beide `title="PLUGIN_SPOTON_BINARY_STATUS"` verwenden und beide eine Fehlermeldung für fehlende Binaries rendern:

- Block 1 (Zeilen 3-5): Rendert rot formatierten `helperMissing`-Text, wenn `helperMissing` gesetzt ist.
- Block 2 (Zeilen 7-13): Rendert `binaryPath` (grün, mit Version) oder die gleiche Fehlermeldung `PLUGIN_SPOTON_BINARY_MISSING` (rot), abhängig von `binaryPath`.

Wenn das Binary fehlt, erscheinen beide Blöcke mit nahezu identischem Inhalt übereinander. Block 1 ist redundant zu Block 2. Es scheint, als hätte Block 2 Block 1 ersetzen sollen, aber Block 1 wurde nicht entfernt.

**Fix:** Block 1 (Zeilen 3-5) entfernen. Block 2 ist selbstständig und vollständig.

---

### WR-03: Aufruf einer privaten Methode `_buildRedirectUri` aus einer anderen Klasse

**File:** `Plugins/SpotOn/Settings.pm:124`

**Issue:** `Plugins::SpotOn::API::TokenManager->_buildRedirectUri()` ruft eine Methode mit Unterstrich-Präfix (Perl-Konvention für "private") aus einer fremden Klasse direkt auf. Das ist eine fragile Kopplung: Wenn `TokenManager` die interne Implementierung von `_buildRedirectUri` ändert, umbenennt oder die Logik in eine andere Hilfsmethode verschiebt, bricht Settings.pm stumm. Außerdem signalisiert der Unterstrich-Präfix, dass diese Methode nicht Teil der öffentlichen API ist.

**Fix:** Eine öffentliche Delegatmethode in TokenManager hinzufügen:
```perl
# In TokenManager.pm
sub buildRedirectUri {
    my ($class) = @_;
    return $class->_buildRedirectUri();
}
```
```perl
# In Settings.pm
$paramRef->{redirectUri} = Plugins::SpotOn::API::TokenManager->buildRedirectUri();
```

---

### WR-04: Normalisierungs-Flag-Injektion funktioniert nicht bei allen Kommandozeilen-Formaten

**File:** `Plugins/SpotOn/Plugin.pm:1077`

**Issue:** Der Regex `s/( -n )/ --enable-volume-normalisation $1/` sucht spezifisch nach dem Muster ` -n ` (Leerzeichen-Bindestrich-n-Leerzeichen). Wenn `-n` am Anfang des Kommandos steht (kein führendes Leerzeichen) oder die Kommandozeile `-n\t` (Tab statt Leerzeichen) enthält, schlägt der Match stumm fehl. Normalisierung wird dann nicht gesetzt, obwohl der Nutzer sie aktiviert hat — stille Fehlfunktion ohne Fehlermeldung.

In `custom-convert.conf` erscheint `-n Squeezebox` typischerweise nicht am Anfang der Zeile, sondern nach anderen Flags — daher ist der aktuelle Regex in der Praxis wahrscheinlich funktional. Das Problem: keine Fehler-/Warn-Ausgabe wenn der Regex nicht matcht, obwohl `$normalize == 1`.

**Fix:** Nach der Injection prüfen, ob der Flag gesetzt wurde, und im Fehlerfall loggen:
```perl
if ($normalize) {
    my $before = $commandTable->{$key};
    $commandTable->{$key} =~ s/( -n )/ --enable-volume-normalisation $1/;
    if ($commandTable->{$key} eq $before) {
        $log->warn("updateTranscodingTable: could not inject --enable-volume-normalisation for $key");
    }
}
```

---

## Info

### IN-01: `PLUGIN_SPOTON_STREAMING_SETTINGS` definiert aber nie verwendet

**File:** `Plugins/SpotOn/strings.txt:217-219`

**Issue:** Der String-Key `PLUGIN_SPOTON_STREAMING_SETTINGS` ist in strings.txt mit deutschen und englischen Übersetzungen definiert ("Streaming-Einstellungen" / "Streaming Settings"), wird aber in keiner der fünf reviewed Dateien referenziert — weder in basic.html noch in Plugin.pm oder Settings.pm. Das deutet darauf hin, dass ein geplanter "Streaming"-Wrapper-Abschnitt in basic.html (wie im Research-Dokument vorgesehen) nicht umgesetzt wurde: Bitrate und Normalisierung erscheinen als eigenständige `WRAPPER`-Blöcke anstatt unter einem gemeinsamen "Streaming"-Abschnitt.

**Fix:** Entweder den String in basic.html verwenden:
```
[% WRAPPER setting title="PLUGIN_SPOTON_STREAMING_SETTINGS" desc="" %]
    ...bitrate und normalization inputs...
[% END %]
```
Oder den String aus strings.txt entfernen, wenn der strukturelle Wrapper bewusst weggelassen wurde.

---

### IN-02: `on_select => 'play'` und `playall => 1` sind semantisch widersprüchlich

**File:** `Plugins/SpotOn/Plugin.pm:326-327` und `974-975`

**Issue:** In `_trackItem` und `_albumTrackItem` sind sowohl `on_select => 'play'` als auch `playall => 1` gesetzt. Das LMS-XMLBrowser-Framework interpretiert diese wie folgt: `on_select => 'play'` bedeutet "beim Antippen diesen Track abspielen"; `playall => 1` bedeutet "beim Antippen alle Items dieser Art in die Queue laden und an dieser Position starten". Gemäß XMLBrowser.pm Zeile 660 (aus dem Research-Dokument verifiziert): `if ($method =~ /^(add|play)$/ && $subFeed->{'items'}->[$playIndex]->{playall}) { $method .= 'all'; }` — das `playall`-Flag überschreibt das normale "play" in Richtung "playall". Die Kombination ist also funktional nicht falsch, aber `on_select => 'play'` ist dann redundant und möglicherweise irreführend. Das ist kein Bug, aber eine Unklarheit in der Bedeutung.

**Fix:** Kein zwingender Fix. Für Klarheit: Entweder `on_select` entfernen (wenn immer `playall`-Verhalten gewünscht ist) oder dokumentieren, warum beide Flags gesetzt werden.

---

_Reviewed: 2026-05-28_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
