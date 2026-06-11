---
phase: 16-macos-universal-binary
reviewed: 2026-06-11T18:30:00Z
depth: standard
files_reviewed: 6
files_reviewed_list:
  - .github/workflows/build-librespot.yml
  - Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html
  - Plugins/SpotOn/Helper.pm
  - Plugins/SpotOn/Settings.pm
  - Plugins/SpotOn/strings.txt
  - README.md
findings:
  critical: 1
  warning: 2
  info: 1
  total: 4
status: issues_found
---

# Phase 16: Code Review Report

**Reviewed:** 2026-06-11T18:30:00Z
**Depth:** standard
**Files Reviewed:** 6
**Status:** issues_found

## Summary

Phase 16 führt den macOS Universal Binary CI-Build (PLT-01/PLT-02) sowie die Gatekeeper-UI und i18n-Strings (PLT-03) ein. Die Implementierung folgt den etablierten Patterns der Codebase korrekt. Ein kritischer Defekt im CI-Release-Job (doppelte Artifact-Namen) würde den ersten Release-Lauf auf einem `v*`-Tag zum Scheitern bringen. Zwei Warnungen betreffen einen toten Parameter und ein irreführendes UX-Detail. Ein Info-Fund weist auf Windows-Quoting-Lücken hin.

## Critical Issues

### CR-01: Release-Job schlägt fehl — doppelte Asset-Namen durch intermediäre Darwin-Artifacts

**File:** `.github/workflows/build-librespot.yml:204-215`

**Issue:** Der `release`-Job lädt mit `actions/download-artifact@v4` (ohne `name:`-Filter) alle Artifacts herunter — also nicht nur das Universal Binary `spoton-darwin`, sondern auch die beiden Zwischen-Artifacts `spoton-aarch64-darwin` und `spoton-x86_64-darwin` aus dem `build-macos`-Job. Alle drei enthalten eine Datei mit demselben Basename `spoton`. Der anschließende Upload-Glob `release-artifacts/**/spoton` matched alle drei, und `softprops/action-gh-release@v2` versucht, drei Assets mit identischem Namen `spoton` in dasselbe GitHub Release hochzuladen. Die GitHub API lehnt doppelte Asset-Namen mit HTTP 422 ab — der Release-Job schlägt beim ersten echten `v*`-Tag-Push fehl. Im besten Fall (stille Überschreibung) landet ein zufälliges der drei Darwin-Binaries im Release, nicht das Universal Binary.

**Fix:** Intermediäre Arch-Artifacts entweder gar nicht hochladen (Schritte in `build-macos` entfernen) oder im Download-Schritt explizit filtern. Einfachste Lösung: `Upload binary artifact`-Schritt aus dem `build-macos`-Job entfernen (die Artifacts werden nur für den `lipo`-Job benötigt, der sie direkt aus dem Build-Output zieht — oder alternativ per explizitem `name:`-Download abholt). Wenn die Artifacts für Debugging-Zwecke erhalten bleiben sollen, muss der Release-Download-Schritt auf benannte Artifacts gefiltert werden:

```yaml
# Option A: Intermediäre Arch-Artifacts aus build-macos nicht hochladen.
# In build-macos job: "Upload binary artifact"-Schritt entfernen.

# Option B: Im release-Job nur bekannte Artifact-Namen herunterladen statt alles.
# Statt eines einzelnen download-artifact-Schritts mit path: release-artifacts:
      - name: Download Linux/Windows artifacts
        uses: actions/download-artifact@v4
        with:
          pattern: spoton-*-linux
          path: release-artifacts
          merge-multiple: false

      - name: Download Windows artifact
        uses: actions/download-artifact@v4
        with:
          name: spoton-x86_64-win64
          path: release-artifacts/spoton-x86_64-win64

      - name: Download macOS Universal Binary
        uses: actions/download-artifact@v4
        with:
          name: spoton-darwin
          path: release-artifacts/spoton-darwin
```

## Warnings

### WR-01: `$customFirst`-Parameter in `_findBin` ist toter Code

**File:** `Plugins/SpotOn/Helper.pm:133`

**Issue:** `_findBin` empfängt `$customFirst` als zweiten Parameter, nutzt ihn aber nie. Der `unshift @candidates, HELPER . '-custom'` auf Zeile 139 passiert bedingungslos, unabhängig davon ob `$customFirst` gesetzt ist oder nicht. Der Parameter ist bedeutungslos. Beim Aufrufer (Zeile 61) wird `'custom-first'` als expliziter Wert übergeben, was suggeriert, dass das Verhalten steuerbar sei — es ist aber hart kodiert.

**Fix:** Entweder den Parameter entfernen und den Aufruf bereinigen, oder die Logik korrekt implementieren:

```perl
# Option A: Parameter entfernen, da custom immer prependiert wird
sub _findBin {
    my ($checkerCb) = @_;  # $customFirst entfernt
    ...
    unshift @candidates, HELPER . '-custom';  # bleibt unbedingt
}

# Aufruf anpassen:
$helper = _findBin(sub { helperCheck(@_, \$check) });

# Option B: $customFirst tatsächlich als Guard verwenden
sub _findBin {
    my ($checkerCb, $customFirst) = @_;
    ...
    unshift @candidates, HELPER . '-custom' if $customFirst;
    ...
}
```

### WR-02: Gatekeeper-Hinweis erscheint bei jedem Binary-Missing auf macOS — nicht nur bei Gatekeeper-Blockierung

**File:** `Plugins/SpotOn/HTML/EN/plugins/SpotOn/settings/basic.html:81-85`

**Issue:** Der `[% IF isMac %]`-Block wird immer angezeigt, wenn das Binary auf macOS fehlt. Das schließt den Fall ein, dass das Plugin frisch installiert wurde und das Binary noch nie vorhanden war (z. B. weil der LMS Plugin Manager es noch nicht heruntergeladen hat, oder weil es auf einer nicht-unterstützten Architektur fehlt). Ein Nutzer, der gerade installiert und das Binary noch nie hatte, sieht den `xattr -d com.apple.quarantine`-Hinweis, der für ihn irreführend und nicht zutreffend ist. Gatekeeper blockiert nur Binaries, die bereits vorhanden, aber mit einem Quarantine-Attribut versehen sind.

**Fix:** Den Hinweis-Text oder die Logik anpassen, sodass er beide Fälle abdeckt, oder den Hinweis weniger alarmierend formulieren. Einfachste textliche Lösung: den Gatekeeper-Hinweis als nachrangige Möglichkeit formulieren statt als primäre Erklärung:

```
# strings.txt - PLUGIN_SPOTON_GATEKEEPER_HINT EN (Beispielformulierung):
"If the binary was downloaded manually and is blocked by macOS Gatekeeper, run: xattr -d com.apple.quarantine /path/to/spoton"
```

Alternativ: nur anzeigen, wenn das Binary nachweislich existiert aber nicht ausführbar ist (erfordert separaten Pfad-Check in Settings.pm — höherer Aufwand).

## Info

### IN-01: Windows-Shell-Quoting in `helperCheck` schützt nur gegen Anführungszeichen

**File:** `Plugins/SpotOn/Helper.pm:79-80`

**Issue:** Auf Windows wird der Pfad nur gegen doppelte Anführungszeichen (`"` → `""`) bereinigt, bevor er in einen Backtick-Befehl eingebettet wird. Andere cmd.exe-Metazeichen (`&`, `|`, `<`, `>`, `^`) werden nicht escaped. Da der Pfad aus `$prefs->get('binary')` stammt — also aus einem Formularfeld, das ausschließlich ein authentifizierter LMS-Administrator ausfüllen kann —, ist das Angriffspotenzial gering (keine Remote-Exploit-Möglichkeit). Dennoch könnte ein manipulierter Pfad wie `C:\tools\spoton" & net user` zu unerwarteter Befehlsausführung führen.

**Fix:** Für erhöhte Robustheit: Pfad auf gültige Zeichen prüfen (keine Shell-Metazeichen) oder `Win32::ShellQuote` verwenden. Pragmatischste Lösung für diesen Admin-Only-Kontext: Pfad-Validierung mit einem Regex-Guard vor der Shell-Ausführung:

```perl
# Vor helperCheck: Pfad-Validierung
if ($candidate !~ /\A[\w\s\.\-\\:\/]+\z/) {
    $log->warn("helperCheck: suspicious characters in binary path, skipping: $candidate");
    return 0;
}
```

---

_Reviewed: 2026-06-11T18:30:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
