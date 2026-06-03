---
phase: 06-polish-release-readiness
reviewed: 2026-06-03T12:45:00Z
depth: standard
files_reviewed: 8
files_reviewed_list:
  - Plugins/SpotOn/API/Client.pm
  - Plugins/SpotOn/DontStopTheMusic.pm
  - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
  - Plugins/SpotOn/Plugin.pm
  - Plugins/SpotOn/ProtocolHandler.pm
  - Plugins/SpotOn/Settings.pm
  - Plugins/SpotOn/strings.txt
  - repo.xml
findings:
  critical: 1
  warning: 5
  info: 3
  total: 9
status: issues_found
---

# Phase 6: Code Review Report

**Reviewed:** 2026-06-03T12:45:00Z
**Depth:** standard
**Files Reviewed:** 8
**Status:** issues_found

## Summary

Phase 6 (Polish + Release Readiness) umfasst Per-Player-Settings (Bitrate-Override, Format-Dropdown), DSTM-Auto-Play, Client-ID-Konsolidierung, Custom-Binary-Support-Verifikation, i18n-Erweiterung, Setup-Guide, Credits und repo.xml-Distribution-Template. Insgesamt solider Code mit konsistenten Patterns. Ein Critical Finding betrifft eine nicht umgesetzte Code-Verzweigung in ProtocolHandler.pm, die dazu fuehrt, dass OGG-Passthrough im Browse-Modus bei expliziter Wahl nicht funktioniert. Fuenf Warnings betreffen funktionale Korrektheitsprobleme und eine fehlende Sicherheitshaertung. Drei Info-Items betreffen Code-Qualitaet.

## Critical Issues

### CR-01: formatOverride() ignoriert streamFormat-Pref -- OGG-Passthrough im Browse-Modus funktionslos

**File:** `Plugins/SpotOn/ProtocolHandler.pm:53-69`
**Issue:** Die Variable `$fmt` wird aus dem per-Player `streamFormat`-Pref gelesen (Zeile 53-57), aber anschliessend nie verwendet. `formatOverride()` gibt im Browse-Modus immer `'son'` zurueck, unabhaengig vom eingestellten Format. Die 06-02-SUMMARY.md dokumentiert explizit: "Browse mode: returns `'ogg'` if `streamFormat eq 'ogg'` (OGG passthrough), `'son'` otherwise" -- diese Logik fehlt im Code.

**Auswirkung:** Wenn ein Benutzer "OGG Passthrough" im Format-Dropdown waehlt, gibt `formatOverride()` trotzdem `'son'` zurueck. LMS konstruiert dann den Transcoding-Key `son-<playerOutput>-*-*`. Zwar loescht `updateTranscodingTable()` die konkurrierenden Pipelines (son-pcm, son-flc, son-mp3), aber LMS sucht zuerst nach dem zum Player passenden Output-Format. Fuer Player die kein OGG nativ unterstuetzen, gibt es keinen passenden Pipeline-Eintrag -- Wiedergabe schlaegt fehl. Fuer OGG-faehige Player waehlt LMS `son-ogg-*-*`, was zufaellig funktioniert, aber nicht durch die dokumentierte Code-Logik gesteuert wird.

**Fix:**
```perl
sub formatOverride {
    my ($class, $song) = @_;

    my $client = $song->master;
    my $url = $song->track->url || '';

    require Plugins::SpotOn::Plugin;
    Plugins::SpotOn::Plugin->updateTranscodingTable($client);

    my $fmt = $client
        ? ($prefs->client($client)->get('streamFormat')
           || $prefs->client($client)->get('connectOggOverride')
           || 'auto')
        : 'auto';

    if ($url =~ m{spotify://connect-}) {
        require Plugins::SpotOn::Connect::DaemonManager;
        my $helper = Plugins::SpotOn::Connect::DaemonManager->helperForClient($client);
        if ($helper && $helper->_streamMode) {
            return 'soc';
        }
    }

    # D-11/D-12: OGG Passthrough fuer Browse-Modus wenn explizit gewaehlt
    return 'ogg' if $fmt eq 'ogg';

    return 'son';
}
```

## Warnings

### WR-01: _baseSonPipelines Snapshot restauriert nur fehlende Eintraege -- vorherige Modifikationen persistieren

**File:** `Plugins/SpotOn/Plugin.pm:1255-1263`
**Issue:** Der `our %_baseSonPipelines`-Snapshot wird einmal beim ersten Aufruf erstellt und speichert die unmodifizierten Originalwerte aus custom-convert.conf. Die Restaurierung (Zeile 1261-1263) verwendet `unless exists $commandTable->{$k}`, d.h. nur geloeschte Eintraege werden zurueckgesetzt. Eintraege die existieren aber durch einen vorherigen `updateTranscodingTable()`-Aufruf fuer Player A modifiziert wurden (z.B. `--bitrate 96`), behalten diese Modifikation als Ausgangspunkt fuer den naechsten Aufruf fuer Player B. Die Regex-Substitutionen (Zeile 1272-1276) ueberschreiben zwar korrekt die variablen Teile, aber dieses Pattern ist fragil: Wenn ein neuer Parameter hinzugefuegt wird und nicht per Regex korrigiert wird, bleibt der Wert des vorherigen Aufrufs bestehen.

**Fix:** Restaurierung durch vollstaendiges Ueberschreiben aller Snapshot-Eintraege statt nur fehlender:
```perl
for my $k (keys %_baseSonPipelines) {
    $commandTable->{$k} = $_baseSonPipelines{$k};  # Immer auf Original zuruecksetzen
}
```

### WR-02: DSTM _searchFallback verwendet offset bis 40 bei Dev-Mode-Limit von 10

**File:** `Plugins/SpotOn/DontStopTheMusic.pm:223`
**Issue:** `int(rand(40))` generiert Offsets von 0 bis 39. Der Search-Aufruf (Zeile 232) verwendet `limit => 10`. In Dev Mode ist die maximale Search-Antwort auf 10 Items begrenzt. Wenn `offset > total - limit` (z.B. ein Artist hat 30 Tracks und offset=35), liefert Spotify eine leere oder sehr kurze Ergebnisliste. In vielen Faellen hat ein Artist weniger als 50 Tracks in der Suche, sodass Offsets >30 haeufig leere Antworten produzieren, was den Fallback stumm scheitern laesst.

**Fix:** Offset auf einen konservativeren Bereich reduzieren, oder nach leerem Ergebnis einmal mit offset=0 wiederholen:
```perl
my $offset = int(rand(20));  # Konservativer Bereich fuer typische Ergebnismengen
```

### WR-03: Settings.pm setzt connectOggOverride weiterhin bei form-submit, obwohl basic.html den alten Dropdown entfernt hat

**File:** `Plugins/SpotOn/Settings.pm:151-155`
**Issue:** Settings.pm (Zeile 151-155) prueft `defined $paramRef->{'pref_connectOggOverride'}` und speichert den Wert. Aber basic.html enthaelt kein `<select name="pref_connectOggOverride">` mehr -- der alte Dropdown wurde durch `pref_streamFormat` ersetzt. Da der Parameter nie im Form gesendet wird, ist der Code-Block toter Code. Dies ist kein Fehler, aber der tote Code-Pfad suggeriert, dass der alte Dropdown noch existiert, was bei zukuenftiger Wartung Verwirrung stiftet.

Zusaetzlich setzt Settings.pm Zeile 204 weiterhin `$paramRef->{connectOggOverride}` als Template-Variable, obwohl basic.html diese Variable nicht mehr referenziert.

**Fix:** Den `pref_connectOggOverride`-Block (Zeile 151-155) und die Template-Variable (Zeile 204) entfernen oder mit einem Kommentar als Migrationsrest kennzeichnen.

### WR-04: canDirectStream blockiert pcm/flac/mp3 fuer Connect-Streams -- aber Connect soll immer DirectStream nutzen

**File:** `Plugins/SpotOn/ProtocolHandler.pm:87-98`
**Issue:** Der streamFormat-Check in `canDirectStream()` (Zeile 88-98) blockiert DirectStream fuer `pcm`, `flac` und `mp3`. Dies betrifft aber auch Connect-URLs (`spotify://connect-*`), obwohl der Kommentar in Zeile 63 von `formatOverride()` sagt: "Connect: always 'soc', independent of streamFormat". Die CONTEXT.md D-11 spezifiziert: "Bei FLAC/MP3: canDirectStream() gibt 0 zurueck". Die 06-RESEARCH.md Pattern 2 Tabelle zeigt fuer pcm: "canDirectStream: 0 (forciert)".

Allerdings ist die Frage, ob dieses Verhalten fuer Connect-Mode tatsaechlich gewollt ist. Wenn ein User `streamFormat=flac` waehlt und Connect laeuft, wuerde `canDirectStream` 0 zurueckgeben, was LMS zwingt die `soc-*` Pipeline zu verwenden statt DirectStream. Da `soc pcm * *` nur den Passthrough-Eintrag (`-`) hat, wuerde das tatsaechlich funktionieren. Aber `soc-flc-*-*` und `soc-mp3-*-*` existieren nicht in custom-convert.conf -- LMS wuerde keine passende Pipeline finden.

Die STREAM_FORMAT_DESC Strings sagen explizit: "Affects Browse mode only. Connect always uses PCM DirectStream." Aber der Code erzwingt Transcoding auch fuer Connect.

**Fix:** Den streamFormat-Check nur fuer Browse-URLs ausfuehren:
```perl
# Per-player streamFormat: pcm/flac/mp3 force transcoding — no DirectStream (D-11)
# Only for Browse mode — Connect always uses DirectStream (STREAM_FORMAT_DESC)
if ($url !~ m{spotify://connect-}) {
    my $fmt = $prefs->client($client)->get('streamFormat')
           || $prefs->client($client)->get('connectOggOverride')
           || 'auto';
    if ($fmt =~ /^(?:pcm|flac|mp3)$/) {
        main::INFOLOG && $log->is_info && $log->info(
            "canDirectStream: 0 (streamFormat=$fmt forces transcoding)"
        );
        return 0;
    }
}
```

### WR-05: target="_blank" Links ohne rel="noopener" in basic.html

**File:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html:9,96`
**Issue:** Zwei `<a target="_blank">` Links (developer.spotify.com/dashboard) fehlt das `rel="noopener noreferrer"` Attribut. In aelteren Browsern kann die geoeffnete Seite ueber `window.opener` auf das LMS-Settings-Fenster zugreifen. Da die Links zu einer externen Drittanbieter-Domain fuehren (developer.spotify.com), ist dies ein Haertungsproblem.

**Fix:**
```html
<a href="https://developer.spotify.com/dashboard" target="_blank" rel="noopener noreferrer">
```

## Info

### IN-01: Unused import Slim::Utils::Cache in DontStopTheMusic.pm Modul-Scope

**File:** `Plugins/SpotOn/DontStopTheMusic.pm:10,16`
**Issue:** `Slim::Utils::Cache` wird importiert und `$cache` initialisiert. Es wird tatsaechlich verwendet (Zeile 262 in `_cacheAndExtractUris`), daher ist es kein ungenutzter Import. Allerdings wird `Slim::Schema` (Zeile 9) nur fuer den RemoteTrack-Lookup verwendet (Zeile 67). Falls ein DSTM-Aufruf nie RemoteTrack-IDs (negative IDs) enthaelt, wird Slim::Schema unnoetig geladen. Da `use` compile-time ist und Slim::Schema in LMS ohnehin immer geladen ist, ist dies jedoch kein praktisches Problem.

**Fix:** Kein Fix noetig -- informativ.

### IN-02: repo.xml enthaelt Platzhalter-SHA1 und Platzhalter-URL

**File:** `repo.xml:28-29`
**Issue:** `PLACEHOLDER_SHA1_PHASE_6_1` und `PLACEHOLDER_URL_PHASE_6_1` als Attributwerte. Dies ist dokumentiert und beabsichtigt (Phase 6.1), aber wenn repo.xml versehentlich als Repository-URL verwendet wird, schlaegt die Installation fehl. Es gibt keinen Laufzeit-Check oder Warnung.

**Fix:** Kein Code-Fix noetig -- das Dokument in repo.xml selbst warnt davor. Sicherstellen, dass die README/Installation-Docs explizit darauf hinweisen, dass Phase 6.1 zuerst abgeschlossen werden muss.

### IN-03: TT-Vergleich mit == statt eq fuer String-Werte in basic.html

**File:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html:42-46,52-55`
**Issue:** Template-Toolkit-Vergleiche wie `[% IF streamFormat == 'auto' %]` verwenden `==` statt `eq`. In Template Toolkit ist `==` der numerische Vergleichsoperator; fuer String-Vergleiche sollte `eq` verwendet werden. In der Praxis funktioniert `==` fuer nicht-numerische Strings in TT trotzdem korrekt (TT behandelt `==` als universellen Vergleich, anders als Perl), aber die Verwendung von `eq` waere idiomatisch korrekter.

**Fix:** Optional -- TT-spezifisch kein Bug. Fuer Konsistenz mit Perl-Idiom:
```
[% IF streamFormat eq 'auto' %]
```

---

_Reviewed: 2026-06-03T12:45:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
